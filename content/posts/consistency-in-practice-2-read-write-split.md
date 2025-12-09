---
title: "从单机到分布式：一致性实战（二）读写分离：副本一致性"
date: 2025-12-11T10:10:00+08:00
draft: false
tags: ["一致性", "PostgreSQL", "Rust", "读写分离", "主从复制"]
categories: ["技术"]
description: "用户刚改完昵称，刷新页面发现还是旧的。这不是 Bug，这是主从延迟。读写分离带来的第一个一致性问题，你准备好了吗？"
series: ["从单机到分布式：一致性实战"]
---

用户刚改完昵称，刷新页面发现还是旧的。这不是 Bug，这是主从延迟。读写分离带来的第一个一致性问题，你准备好了吗？

<!--more-->

## 前情回顾

上一篇我们看到，小明的二手书平台在单机时代过得很幸福。一个 PostgreSQL 实例，一个事务搞定一切。

但好景不长。日活破万后，数据库 CPU 飙到 100%。小明不得不做出改变。

## 读写分离：看起来很美

小明的第一反应是：加机器。

分析了一下业务，发现 **读请求是写请求的 10 倍**。用户大部分时间在浏览书籍、查看订单，只有少数时间在下单、修改资料。

于是小明做了一个经典的架构升级：**读写分离**。

```text
┌─────────────────────────────────────────────────────────────────┐
│                           用户请求                               │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Rust + Axum                              │
│                                                                  │
│              写请求                        读请求                 │
│                │                             │                   │
└────────────────┼─────────────────────────────┼───────────────────┘
                 │                             │
                 ▼                             ▼
┌────────────────────────┐        ┌────────────────────────┐
│      PostgreSQL        │        │      PostgreSQL        │
│        主库            │ ─────→ │        从库            │
│       (写入)           │  复制   │       (只读)           │
└────────────────────────┘        └────────────────────────┘
```

**看起来很完美**：

- 写请求走主库，保证数据安全
- 读请求走从库，分担压力
- 主库的数据自动同步到从库

小明部署上线，CPU 立刻降到 30%。**问题解决了？**

## Bug 还是 Feature？

上线第二天，客服收到投诉：

> "我刚改了昵称，刷新页面还是旧的！你们系统有 Bug 吧？"

小明查了半天代码，没找到问题。数据库里的昵称明明是新的啊？

然后他意识到了：**用户的写请求走了主库，但刷新页面的读请求走了从库。**

主从复制有延迟。可能是 10 毫秒，可能是 1 秒，极端情况下可能是几秒。

```text
时间线：
───────────────────────────────────────────────────────────────→

T0: 用户修改昵称
    │
    ▼
T1: 主库更新成功，返回"修改成功"
    │
    │ ← 主从复制中...（延迟 500ms）
    │
T2: 用户刷新页面，从从库读取
    │
    ▼
T3: 返回旧昵称！（从库还没同步完）
```

**这不是 Bug，这是分布式系统的常态。**

> **核心问题**：主从复制是异步的，写入主库后，数据不会立即出现在从库。在这个时间窗口内，读从库会得到旧数据。

## 方案全景：五种选择

面对主从延迟，有五种解决方案：

```text
┌─────────────────────────────────────────────────────────────────┐
│                     读写分离一致性方案                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  方案1：强制读主          → 最简单，但失去读写分离的意义         │
│  方案2：写后读主（时间窗口）→ 平衡一致性和性能                   │
│  方案3：写后读主（会话级） → 同一请求内保证一致                  │
│  方案4：同步复制          → 数据库层面保证，但牺牲写性能         │
│  方案5：业务层容忍        → 零成本，但用户体验差                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

让我们逐一分析。

## 方案一：强制读主

**最简单粗暴的方案：所有读请求都走主库。**

```rust
pub struct PrimaryOnlyRepo {
    primary: PgPool,
    _replica: PgPool,  // 从库闲置
}

impl PrimaryOnlyRepo {
    pub async fn get_user(&self, user_id: i64) -> Result<User> {
        // 直接读主库，保证强一致
        sqlx::query_as!(
            User,
            r#"SELECT * FROM users WHERE id = $1"#,
            user_id
        )
        .fetch_one(&self.primary)
        .await
        .map_err(Into::into)
    }
}
```

**优点**：
- 实现最简单
- 一致性最强
- 不用担心主从延迟

**缺点**：
- 主库压力没有减轻
- 读写分离形同虚设
- 浪费了从库资源

**适用场景**：读写比接近 1:1，或者主库性能足够。

**结论**：这不是解决方案，这是放弃治疗。

## 方案二：写后读主（时间窗口）

**核心思想**：用户写入后的一段时间内，强制从主库读取。

```rust
use dashmap::DashMap;
use std::time::{Duration, Instant};

pub struct TimeWindowRouter {
    primary: PgPool,
    replica: PgPool,
    // 记录最近写入的 key 和时间
    recent_writes: DashMap<String, Instant>,
    // 写后强制读主的时间窗口
    window: Duration,
}

impl TimeWindowRouter {
    pub fn new(primary: PgPool, replica: PgPool, window_secs: u64) -> Self {
        Self {
            primary,
            replica,
            recent_writes: DashMap::new(),
            window: Duration::from_secs(window_secs),
        }
    }

    /// 标记写入
    pub fn mark_write(&self, key: &str) {
        self.recent_writes.insert(key.to_string(), Instant::now());
    }

    /// 获取读连接：判断走主还是从
    pub fn read_pool(&self, key: &str) -> &PgPool {
        if let Some(write_time) = self.recent_writes.get(key) {
            if write_time.elapsed() < self.window {
                return &self.primary;  // 刚写过，读主库
            }
        }
        &self.replica  // 否则读从库
    }

    /// 定期清理过期记录
    pub fn cleanup(&self) {
        self.recent_writes.retain(|_, v| v.elapsed() < self.window * 2);
    }
}
```

使用示例：

```rust
pub async fn update_user_nickname(
    router: &TimeWindowRouter,
    user_id: i64,
    nickname: &str,
) -> Result<()> {
    let key = format!("user:{}", user_id);

    // 写入主库
    sqlx::query!(
        r#"UPDATE users SET nickname = $1 WHERE id = $2"#,
        nickname,
        user_id
    )
    .execute(&router.primary)
    .await?;

    // 标记刚写入
    router.mark_write(&key);

    Ok(())
}

pub async fn get_user(router: &TimeWindowRouter, user_id: i64) -> Result<User> {
    let key = format!("user:{}", user_id);

    // 自动判断走主还是从
    let pool = router.read_pool(&key);

    sqlx::query_as!(
        User,
        r#"SELECT * FROM users WHERE id = $1"#,
        user_id
    )
    .fetch_one(pool)
    .await
    .map_err(Into::into)
}
```

**优点**：
- 兼顾一致性和性能
- 大部分读请求仍然走从库
- 只有刚写入的数据走主库

**缺点**：
- 需要维护写入记录
- 时间窗口难以精确设置
- 多实例部署时需要共享状态（如 Redis）

**时间窗口设置建议**：

| 主从延迟 | 建议窗口 |
|---------|---------|
| < 100ms | 1-2 秒 |
| 100ms - 1s | 3-5 秒 |
| > 1s | 考虑同步复制 |

**适用场景**：主从延迟可控，需要平衡一致性和性能。

## 方案三：写后读主（会话级）

**核心思想**：同一个 HTTP 请求内，写过的数据从主库读。

```rust
use std::collections::HashSet;
use std::sync::Arc;
use tokio::sync::RwLock;

/// 会话级上下文：追踪当前请求内写过的表
#[derive(Default)]
pub struct SessionContext {
    wrote_tables: RwLock<HashSet<String>>,
}

impl SessionContext {
    pub async fn mark_write(&self, table: &str) {
        self.wrote_tables.write().await.insert(table.to_string());
    }

    pub async fn should_read_primary(&self, table: &str) -> bool {
        self.wrote_tables.read().await.contains(table)
    }
}
```

结合 Axum 中间件使用：

```rust
use axum::{
    extract::Extension,
    middleware::{self, Next},
    response::Response,
};

/// 中间件：每个请求创建新的 SessionContext
pub async fn session_middleware<B>(
    mut req: axum::http::Request<B>,
    next: Next<B>,
) -> Response {
    req.extensions_mut()
        .insert(Arc::new(SessionContext::default()));
    next.run(req).await
}

/// Handler 示例
pub async fn update_and_get_user(
    Extension(ctx): Extension<Arc<SessionContext>>,
    Extension(primary): Extension<PgPool>,
    Extension(replica): Extension<PgPool>,
    user_id: i64,
    nickname: String,
) -> Result<User> {
    // 写入主库
    sqlx::query!(
        r#"UPDATE users SET nickname = $1 WHERE id = $2"#,
        nickname,
        user_id
    )
    .execute(&primary)
    .await?;

    // 标记写入了 users 表
    ctx.mark_write("users").await;

    // 读取时判断
    let pool = if ctx.should_read_primary("users").await {
        &primary
    } else {
        &replica
    };

    sqlx::query_as!(
        User,
        r#"SELECT * FROM users WHERE id = $1"#,
        user_id
    )
    .fetch_one(pool)
    .await
    .map_err(Into::into)
}
```

**优点**：
- 精确控制，只影响当前请求
- 不需要持久化写入记录
- 请求结束自动清理

**缺点**：
- 只解决同一请求内的问题
- 用户刷新页面（新请求）仍可能读到旧数据
- 需要改造所有数据访问代码

**适用场景**：单次请求内有"写后读"的场景，如更新后返回最新数据。

## 方案四：同步复制

**核心思想**：让数据库保证写入主库后，从库也同步完成。

这是 PostgreSQL 的原生能力：

```sql
-- postgresql.conf 配置
-- 同步复制：至少等待一个从库确认
synchronous_commit = on
synchronous_standby_names = 'replica1'
```

在 Rust 中，你不需要改任何代码：

```rust
// 当 synchronous_commit = on 时
// 这条写入会等待从库确认后才返回
sqlx::query!(
    r#"UPDATE users SET nickname = $1 WHERE id = $2"#,
    nickname,
    user_id
)
.execute(pool)
.await?;

// 此时从库已经有最新数据了
```

**对于不重要的写入，可以临时关闭同步等待**：

```rust
pub async fn log_user_action(pool: &PgPool, user_id: i64, action: &str) -> Result<()> {
    // 本次事务不等待同步复制
    sqlx::query("SET LOCAL synchronous_commit = off")
        .execute(pool)
        .await?;

    sqlx::query!(
        r#"INSERT INTO user_logs (user_id, action, created_at) VALUES ($1, $2, NOW())"#,
        user_id,
        action
    )
    .execute(pool)
    .await?;

    Ok(())
}
```

**优点**：
- 应用层最简单，无需改代码
- 强一致，从库保证有最新数据
- 数据库原生支持，稳定可靠

**缺点**：
- 写入延迟增加（要等从库确认）
- 从库故障会影响主库写入
- 不适合写密集场景

**同步模式对比**：

| 模式 | 配置 | 一致性 | 性能 | 可用性 |
|------|------|--------|------|--------|
| 异步复制 | `synchronous_commit = off` | 弱 | 高 | 高 |
| 本地提交 | `synchronous_commit = local` | 中 | 中 | 高 |
| 同步复制 | `synchronous_commit = on` | 强 | 低 | 中 |
| 远程写入 | `synchronous_commit = remote_write` | 较强 | 中 | 中 |

**适用场景**：金融交易、对一致性要求极高的场景。

## 方案五：业务层容忍

**最简单的方案：什么都不做，在 UI 上告诉用户。**

```rust
pub async fn update_user_nickname(
    pool: &PgPool,
    user_id: i64,
    nickname: &str,
) -> Result<UpdateResponse> {
    sqlx::query!(
        r#"UPDATE users SET nickname = $1 WHERE id = $2"#,
        nickname,
        user_id
    )
    .execute(pool)
    .await?;

    Ok(UpdateResponse {
        success: true,
        message: "修改成功！数据同步可能需要几秒钟。".to_string(),
    })
}
```

**优点**：
- 零开发成本
- 不影响系统性能
- 符合最终一致性的哲学

**缺点**：
- 用户体验差
- 只适合非关键业务
- 可能引起用户困惑

**适用场景**：非关键业务，如浏览历史、阅读记录。

## 方案对比与选型

| 方案 | 一致性 | 性能影响 | 实现复杂度 | 适用场景 |
|------|--------|----------|------------|----------|
| 强制读主 | 强 | 高 | 低 | 不推荐 |
| 写后读主（时间窗口） | 较强 | 低 | 中 | **通用推荐** |
| 写后读主（会话级） | 较强 | 低 | 中 | 单请求内读写 |
| 同步复制 | 强 | 中 | 低 | 金融场景 |
| 业务层容忍 | 弱 | 无 | 无 | 非关键业务 |

### 决策流程

```text
                    ┌─────────────────────┐
                    │   一致性要求高吗？   │
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                  │
              ▼                                  ▼
         ┌────────┐                        ┌────────┐
         │   是   │                        │   否   │
         └────┬───┘                        └────┬───┘
              │                                  │
              ▼                                  ▼
    ┌─────────────────────┐            ┌─────────────────────┐
    │  DBA 能配置同步复制？ │            │    业务层容忍       │
    └──────────┬──────────┘            │  （提示用户延迟）    │
               │                        └─────────────────────┘
    ┌──────────┴──────────┐
    │                      │
    ▼                      ▼
┌────────┐           ┌────────┐
│  能    │           │  不能  │
└────┬───┘           └────┬───┘
     │                     │
     ▼                     ▼
┌─────────────┐    ┌──────────────────┐
│  同步复制   │    │  写后读主         │
│  （推荐）   │    │  时间窗口 3-5 秒  │
└─────────────┘    └──────────────────┘
```

## 完整实现：多实例友好的写后读主

在生产环境中，你的服务通常有多个实例。写入记录需要共享，这时候用 Redis：

```rust
use redis::AsyncCommands;

pub struct RedisBackedRouter {
    primary: PgPool,
    replica: PgPool,
    redis: redis::Client,
    window_secs: u64,
}

impl RedisBackedRouter {
    pub async fn mark_write(&self, key: &str) -> Result<()> {
        let mut conn = self.redis.get_multiplexed_async_connection().await?;
        let redis_key = format!("write_marker:{}", key);

        // 设置标记，自动过期
        conn.set_ex::<_, _, ()>(&redis_key, "1", self.window_secs).await?;

        Ok(())
    }

    pub async fn read_pool(&self, key: &str) -> Result<&PgPool> {
        let mut conn = self.redis.get_multiplexed_async_connection().await?;
        let redis_key = format!("write_marker:{}", key);

        let exists: bool = conn.exists(&redis_key).await?;

        if exists {
            Ok(&self.primary)
        } else {
            Ok(&self.replica)
        }
    }
}
```

**使用示例**：

```rust
pub struct UserService {
    router: RedisBackedRouter,
}

impl UserService {
    pub async fn update_nickname(&self, user_id: i64, nickname: &str) -> Result<()> {
        let key = format!("user:{}", user_id);

        sqlx::query!(
            r#"UPDATE users SET nickname = $1 WHERE id = $2"#,
            nickname,
            user_id
        )
        .execute(&self.router.primary)
        .await?;

        self.router.mark_write(&key).await?;

        Ok(())
    }

    pub async fn get_user(&self, user_id: i64) -> Result<User> {
        let key = format!("user:{}", user_id);
        let pool = self.router.read_pool(&key).await?;

        sqlx::query_as!(
            User,
            r#"SELECT * FROM users WHERE id = $1"#,
            user_id
        )
        .fetch_one(pool)
        .await
        .map_err(Into::into)
    }
}
```

## 监控主从延迟

无论选择哪种方案，监控主从延迟都是必须的：

```rust
use prometheus::{Histogram, register_histogram};

lazy_static::lazy_static! {
    static ref REPLICATION_LAG: Histogram = register_histogram!(
        "pg_replication_lag_seconds",
        "PostgreSQL replication lag in seconds"
    ).unwrap();
}

pub async fn check_replication_lag(primary: &PgPool, replica: &PgPool) -> Result<f64> {
    // 方法：写入时间戳到主库，从从库读取，计算差值
    let now = chrono::Utc::now();

    sqlx::query!(
        r#"INSERT INTO heartbeat (id, ts) VALUES (1, $1)
           ON CONFLICT (id) DO UPDATE SET ts = $1"#,
        now
    )
    .execute(primary)
    .await?;

    // 等待一小段时间让复制有机会完成
    tokio::time::sleep(std::time::Duration::from_millis(100)).await;

    let replica_ts: chrono::DateTime<chrono::Utc> = sqlx::query_scalar!(
        r#"SELECT ts FROM heartbeat WHERE id = 1"#
    )
    .fetch_one(replica)
    .await?
    .ok_or_else(|| anyhow::anyhow!("Heartbeat timestamp not found"))?;

    let lag = (now - replica_ts).num_milliseconds() as f64 / 1000.0;

    REPLICATION_LAG.observe(lag);

    Ok(lag)
}
```

**告警阈值建议**：

| 延迟 | 级别 | 动作 |
|------|------|------|
| < 100ms | 正常 | 无 |
| 100ms - 1s | 警告 | 观察 |
| > 1s | 严重 | 检查从库状态 |
| > 5s | 紧急 | 考虑切换或降级 |

## 常见问题

### Q：如果从库挂了怎么办？

**A：自动 fallback 到主库。**

```rust
impl RedisBackedRouter {
    pub async fn read_pool_with_fallback(&self, key: &str) -> &PgPool {
        // 先尝试从从库读
        match self.check_replica_health().await {
            Ok(true) => {
                // 从库健康，走正常逻辑
                self.read_pool(key).await.unwrap_or(&self.primary)
            }
            _ => {
                // 从库不健康，fallback 到主库
                &self.primary
            }
        }
    }

    async fn check_replica_health(&self) -> Result<bool> {
        let result = sqlx::query("SELECT 1")
            .execute(&self.replica)
            .await;

        Ok(result.is_ok())
    }
}
```

### Q：多个从库怎么负载均衡？

**A：用连接池或中间件。**

```rust
// 简单的轮询
pub struct MultiReplicaRouter {
    primary: PgPool,
    replicas: Vec<PgPool>,
    counter: AtomicUsize,
}

impl MultiReplicaRouter {
    pub fn next_replica(&self) -> &PgPool {
        let idx = self.counter.fetch_add(1, Ordering::Relaxed) % self.replicas.len();
        &self.replicas[idx]
    }
}
```

### Q：写后读主会不会增加主库压力？

**A：会，但通常可控。**

假设写请求占 10%（读占 90%），时间窗口 3 秒，平均请求间隔 10 秒：
- 用户写入后 3 秒内的读请求会走主库
- 假设 30% 的读请求命中这个窗口
- 额外主库负载：90% × 30% ≈ 27% 的请求从从库转到主库

**实际场景下这个数字会更低**，因为：
1. 大部分用户只是浏览，不写入
2. 写入后用户不一定立即刷新
3. 可以只对关键业务（如用户资料）启用写后读主，其他业务仍走从库

## 小明的选择

经过权衡，小明选择了**方案二：写后读主（时间窗口）**：

- 时间窗口设为 3 秒
- 用 Redis 存储写入标记
- 监控主从延迟，延迟超过 1 秒告警

用户投诉没有了，主库 CPU 保持在 40%，从库分担了大部分读压力。

**但新的问题来了。**

小明发现，书籍列表页面仍然很慢。每次请求都要查数据库，即使数据根本没变。

> "要不要加个缓存？"

小明陷入了思考。他不知道的是，缓存会带来另一个一致性问题——**缓存与数据库不一致**。

下一篇，我们来聊这个坑。

---

*上一篇：[单机时代：ACID 的庇护](/posts/consistency-in-practice-1-single-node/)*
*下一篇：[引入缓存：多存储一致性](/posts/consistency-in-practice-3-caching/)*

*本系列：*
1. *[单机时代：ACID 的庇护](/posts/consistency-in-practice-1-single-node/)*
2. *读写分离：副本一致性（本篇）*
3. *[引入缓存：多存储一致性](/posts/consistency-in-practice-3-caching/)*
4. *[数据分片：跨分片事务一致性](/posts/consistency-in-practice-4-sharding/)*
5. *[服务拆分：跨服务事务一致性](/posts/consistency-in-practice-5-microservices/)*
6. *[消息驱动：最终一致性](/posts/consistency-in-practice-6-messaging/)*
7. *[总结：演进全景与选型指南](/posts/consistency-in-practice-7-summary/)*
