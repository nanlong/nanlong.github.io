---
title: "从单机到分布式：一致性实战（一）单机时代：ACID 的庇护"
date: 2025-12-11T10:00:00+08:00
draft: false
tags: ["一致性", "PostgreSQL", "Rust", "事务", "ACID"]
categories: ["技术"]
description: "当你的系统只有一台机器、一个数据库时，一致性问题几乎不存在。数据库事务是你的保护伞。但这种幸福，注定是短暂的。"
series: ["从单机到分布式：一致性实战"]
---

当你的系统只有一台机器、一个数据库时，一致性问题几乎不存在。数据库事务是你的保护伞。但这种幸福，注定是短暂的。

<!--more-->

## 一个创业项目的开始

2024 年初，小明辞职创业，做一个二手书交易平台。

技术选型很简单：Rust + Axum + PostgreSQL，一台 4 核 8G 的云服务器。日活用户？乐观估计 100 人。

**这是最幸福的时光。**

因为在这个阶段，小明写出的每一行代码都不需要考虑"分布式一致性"这个词。用户下单、扣库存、扣余额——一个数据库事务搞定一切。

```rust
pub async fn create_order(
    pool: &PgPool,
    buyer_id: i64,
    book_id: i64,
    price: Decimal,
) -> Result<Order> {
    let mut tx = pool.begin().await?;

    // 1. 创建订单
    let order = sqlx::query_as!(
        Order,
        r#"INSERT INTO orders (buyer_id, book_id, price, status)
           VALUES ($1, $2, $3, 'created')
           RETURNING *"#,
        buyer_id, book_id, price
    )
    .fetch_one(&mut *tx)
    .await?;

    // 2. 扣减库存（标记书籍为已售）
    let rows = sqlx::query!(
        r#"UPDATE books SET status = 'sold' WHERE id = $1 AND status = 'available'"#,
        book_id
    )
    .execute(&mut *tx)
    .await?
    .rows_affected();

    if rows == 0 {
        return Err(Error::BookNotAvailable);
    }

    // 3. 扣减买家余额
    let rows = sqlx::query!(
        r#"UPDATE accounts SET balance = balance - $1 WHERE user_id = $2 AND balance >= $1"#,
        price, buyer_id
    )
    .execute(&mut *tx)
    .await?
    .rows_affected();

    if rows == 0 {
        return Err(Error::InsufficientBalance);
    }

    // 4. 增加卖家余额
    let rows = sqlx::query!(
        r#"UPDATE accounts SET balance = balance + $1 WHERE user_id = (
            SELECT seller_id FROM books WHERE id = $2
        )"#,
        price, book_id
    )
    .execute(&mut *tx)
    .await?
    .rows_affected();

    if rows == 0 {
        return Err(Error::SellerAccountNotFound);
    }

    // 提交事务
    tx.commit().await?;

    Ok(order)
}
```

这段代码做了四件事：创建订单、标记书籍已售、扣买家钱、加卖家钱。

**任何一步失败，整个事务回滚，就像什么都没发生过。**

这就是数据库事务的魔力。

## ACID：你的保护伞

PostgreSQL 的事务提供了 ACID 保证，这四个字母就像一把保护伞，让你不用操心"一致性"这个词：

| 特性 | 含义 | 类比 |
|------|------|------|
| **A**tomicity（原子性） | 要么全成功，要么全失败 | 要么结婚，要么分手，没有"半婚"状态 |
| **C**onsistency（一致性） | 事务前后，数据库约束都满足 | 身份证号不能重复，外键必须存在 |
| **I**solation（隔离性） | 并发事务互不干扰 | 你改你的，我改我的 |
| **D**urability（持久性） | 提交后，断电也不丢 | 白纸黑字，赖不掉 |

> **注**：ACID 中的 Consistency 指数据库层面的约束一致性（如主键、外键、唯一约束、CHECK 约束等），而非业务层面的一致性（如"转账后总金额不变"）。业务一致性需要通过正确的应用逻辑来保证。

> **核心观点**：单机时代，ACID 帮你解决了一致性问题。你只需要把操作包在 `BEGIN` 和 `COMMIT` 之间，数据库会保证一切。

## 隔离级别：可以调节的保护力度

PostgreSQL 支持四种隔离级别，保护力度从低到高：

```text
保护力度：低 ──────────────────────────────────→ 高
         READ UNCOMMITTED → READ COMMITTED → REPEATABLE READ → SERIALIZABLE
                              ↑
                              │
                         PostgreSQL 默认
```

### 各隔离级别对比

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 性能 |
|---------|------|-----------|------|------|
| READ UNCOMMITTED | 不可能* | 可能 | 可能 | 高 |
| READ COMMITTED | 不可能 | 可能 | 可能 | 高 |
| REPEATABLE READ | 不可能 | 不可能 | 不可能* | 中 |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 | 低 |

> **PostgreSQL 的特殊行为**：
> - *READ UNCOMMITTED 在 PostgreSQL 中会自动升级为 READ COMMITTED，因此脏读不可能发生
> - *REPEATABLE READ 使用快照隔离（Snapshot Isolation），实际上也防止了幻读
> - 这是 PostgreSQL 相比 SQL 标准的"加强版"实现，提供了更强的一致性保证

**什么是脏读、不可重复读、幻读？**

用银行转账的场景来理解：

**脏读**：你看到了别人还没提交的数据。

```text
事务 A：张三余额从 1000 改成 500（还没提交）
事务 B：读取张三余额 = 500 ← 脏读！
事务 A：回滚
结果：事务 B 读到了一个从未存在过的值
```

**不可重复读**：同一个事务内，两次读取结果不一样。

```text
事务 A：读取张三余额 = 1000
事务 B：张三余额改成 500，提交
事务 A：再次读取张三余额 = 500 ← 不可重复读！
```

**幻读**：同一个事务内，两次查询的行数不一样。

```text
事务 A：SELECT COUNT(*) FROM orders WHERE user_id = 1  → 10 条
事务 B：INSERT INTO orders (user_id, ...) VALUES (1, ...)，提交
事务 A：SELECT COUNT(*) FROM orders WHERE user_id = 1  → 11 条 ← 幻读！
```

### 如何选择隔离级别？

```rust
// 默认：READ COMMITTED，适合大多数场景
sqlx::query!("SELECT * FROM books WHERE id = $1", book_id)
    .fetch_one(pool)
    .await?;

// 需要可重复读：金融对账、报表统计
sqlx::query!("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
    .execute(&mut *tx)
    .await?;

// 需要严格一致性读：统计、审计（注意：会增加序列化失败风险）
sqlx::query!("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
    .execute(&mut *tx)
    .await?;
```

**实际建议**：

| 场景 | 推荐方案 |
|------|----------|
| 普通 CRUD | READ COMMITTED（默认） |
| 金融交易、对账 | REPEATABLE READ |
| 秒杀、库存扣减 | 悲观锁（FOR UPDATE）或乐观锁 |
| 统计报表、审计 | SERIALIZABLE（需处理重试） |

> **为什么秒杀不用 SERIALIZABLE？** SERIALIZABLE 在高并发写入时会产生大量序列化失败（PostgreSQL 错误码 40001），需要应用层重试。在秒杀这种极端并发场景，重试风暴会让系统雪崩。相比之下，`FOR UPDATE` 让事务排队等待，虽然有延迟但不会失败；乐观锁则通过版本号检测冲突，失败时由业务决定是否重试。

## 乐观锁 vs 悲观锁：另一种选择

除了隔离级别，还可以用锁来解决并发问题。

### 悲观锁：先占坑，再操作

```rust
// FOR UPDATE：锁定这一行，其他事务等着
let book = sqlx::query_as!(
    Book,
    r#"SELECT * FROM books WHERE id = $1 FOR UPDATE"#,
    book_id
)
.fetch_one(&mut *tx)
.await?;

// 现在可以安全地修改了
sqlx::query!(
    r#"UPDATE books SET status = 'sold' WHERE id = $1"#,
    book_id
)
.execute(&mut *tx)
.await?;
```

**类比**：去银行取钱，先拿号排队。轮到你之前，柜台被你独占。

**适用场景**：冲突概率高、不能接受重试。

### 乐观锁：先操作，冲突再说

```rust
// 用版本号实现乐观锁
let book = sqlx::query_as!(
    Book,
    r#"SELECT * FROM books WHERE id = $1"#,
    book_id
)
.fetch_one(pool)
.await?;

// 更新时检查版本号
let rows = sqlx::query!(
    r#"UPDATE books
       SET status = 'sold', version = version + 1
       WHERE id = $1 AND version = $2"#,
    book_id, book.version
)
.execute(pool)
.await?
.rows_affected();

if rows == 0 {
    return Err(Error::ConcurrentModification);
}
```

**类比**：Google Docs 协作编辑。大家同时编辑，提交时检查有没有冲突。

**适用场景**：冲突概率低、可以接受重试。

### 对比选型

| 维度 | 悲观锁 | 乐观锁 |
|------|--------|--------|
| 实现方式 | `SELECT ... FOR UPDATE` | 版本号 / 时间戳 |
| 并发性能 | 低（串行等待） | 高（无等待） |
| 冲突处理 | 等待 | 重试 |
| 适用场景 | 冲突多 | 冲突少 |
| 死锁风险 | 有 | 无 |

## 完整的单机架构

此时小明的系统架构非常简单：

```text
┌─────────────────────────────────────────────────────────┐
│                      用户请求                           │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    Nginx (反向代理)                      │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│               Rust + Axum (单实例)                       │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │              业务逻辑                            │   │
│   │  create_order() / get_books() / ...            │   │
│   └─────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   PostgreSQL (单实例)                    │
│                                                         │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│   │  users  │  │  books  │  │ orders  │  │accounts │   │
│   └─────────┘  └─────────┘  └─────────┘  └─────────┘   │
└─────────────────────────────────────────────────────────┘
```

**这个架构的优点**：

1. **简单**：没有分布式，没有网络分区，没有数据同步
2. **一致性强**：数据库事务保证 ACID
3. **调试方便**：所有日志在一台机器上
4. **成本低**：一台服务器 + 一个数据库

**这个架构能支撑多大？**

| 指标 | 估算值 |
|------|--------|
| 日活用户 | 1,000 - 10,000 |
| QPS | 100 - 500 |
| 数据量 | 单表 < 1000 万行 |

对于一个初创项目，**足够了**。

## 单机时代的代码实践

让我们看一个更完整的例子：用户注册 + 初始化账户。

```rust
use sqlx::{PgPool, Postgres, Transaction};
use uuid::Uuid;

pub struct UserService {
    pool: PgPool,
}

impl UserService {
    /// 用户注册：创建用户 + 初始化账户余额
    pub async fn register(&self, username: &str, email: &str) -> Result<User> {
        let mut tx: Transaction<'_, Postgres> = self.pool.begin().await?;

        // 1. 检查用户名是否已存在
        let exists = sqlx::query_scalar!(
            r#"SELECT EXISTS(SELECT 1 FROM users WHERE username = $1)"#,
            username
        )
        .fetch_one(&mut *tx)
        .await?
        .unwrap_or(false);

        if exists {
            return Err(Error::UsernameExists);
        }

        // 2. 创建用户
        let user = sqlx::query_as!(
            User,
            r#"INSERT INTO users (id, username, email, created_at)
               VALUES ($1, $2, $3, NOW())
               RETURNING *"#,
            Uuid::new_v4(),
            username,
            email
        )
        .fetch_one(&mut *tx)
        .await?;

        // 3. 初始化账户（余额为 0）
        sqlx::query!(
            r#"INSERT INTO accounts (user_id, balance) VALUES ($1, 0)"#,
            user.id
        )
        .execute(&mut *tx)
        .await?;

        // 4. 记录注册日志
        sqlx::query!(
            r#"INSERT INTO user_logs (user_id, action, created_at)
               VALUES ($1, 'register', NOW())"#,
            user.id
        )
        .execute(&mut *tx)
        .await?;

        // 提交事务
        tx.commit().await?;

        Ok(user)
    }
}
```

这段代码做了四件事，全部在一个事务里。如果第 3 步失败（比如 `accounts` 表有问题），用户也不会被创建。

**这就是单机时代的幸福：你不需要考虑"用户创建了但账户没初始化"这种情况。**

## 常见问题

### Q：单机数据库会不会有并发问题？

**A：会，但数据库帮你处理了。**

PostgreSQL 的 MVCC（多版本并发控制）机制，让读写可以并发进行。你需要关心的只是选择合适的隔离级别和锁策略。

### Q：什么时候应该从单机升级？

**A：当你遇到以下任何一个问题时：**

1. **读压力大**：数据库 CPU 持续 80%+ → 考虑读写分离
2. **单点故障**：数据库挂了，整个系统挂 → 考虑主从复制
3. **数据量大**：单表超过 5000 万行 → 考虑分库分表
4. **代码臃肿**：单体应用改不动了 → 考虑服务拆分

**在遇到这些问题之前，不要过早优化。**

### Q：PostgreSQL vs MySQL，选哪个？

**A：对于新项目，推荐 PostgreSQL。**

| 维度 | PostgreSQL | MySQL |
|------|-----------|-------|
| SQL 标准兼容 | 更好 | 一般 |
| 复杂查询 | 更强 | 一般 |
| JSON 支持 | 原生 JSONB | JSON |
| 扩展性 | 丰富（PostGIS、Citus） | 有限 |
| 社区 | 活跃 | 活跃 |

当然，如果团队更熟悉 MySQL，用 MySQL 也完全可以。

## 幸福的终结

小明的二手书平台慢慢火了。

半年后，日活用户突破 1 万。某天晚上，数据库 CPU 飙到 100%，网站打不开了。

小明看着监控面板，意识到：**单机时代结束了。**

下一步，他需要引入读写分离——一台主库负责写，多台从库负责读。

但这也意味着，**一致性问题正式登场**。

下一篇，我们来聊聊读写分离带来的第一个坑：**主从延迟**。

---

*下一篇：[读写分离：副本一致性](/posts/consistency-in-practice-2-read-write-split/)*

*本系列：*
1. *单机时代：ACID 的庇护（本篇）*
2. *[读写分离：副本一致性](/posts/consistency-in-practice-2-read-write-split/)*
3. *[引入缓存：多存储一致性](/posts/consistency-in-practice-3-caching/)*
4. *[数据分片：跨分片事务一致性](/posts/consistency-in-practice-4-sharding/)*
5. *[服务拆分：跨服务事务一致性](/posts/consistency-in-practice-5-microservices/)*
6. *[消息驱动：最终一致性](/posts/consistency-in-practice-6-messaging/)*
7. *[总结：演进全景与选型指南](/posts/consistency-in-practice-7-summary/)*
