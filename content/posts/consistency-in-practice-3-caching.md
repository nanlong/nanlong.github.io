---
title: "从单机到分布式：一致性实战（三）引入缓存：多存储一致性"
date: 2025-12-11T10:20:00+08:00
draft: false
tags: ["一致性", "缓存", "Redis", "PostgreSQL", "Rust"]
categories: ["技术"]
description: "当数据同时存在于数据库和缓存中，如何保证它们的一致性？本文详解 Cache-Aside、延迟双删、CDC 等六种方案的实现与权衡"
series: ["从单机到分布式：一致性实战"]
---

## 从这里开始

小明的二手书平台在读写分离上线后，数据库终于稳定了。但好景不长，运营同学兴奋地告诉他："我们要搞一次促销活动！"

活动当天，流量暴涨。虽然有了从库分担读请求，但热门书籍的详情页依然让数据库不堪重负。监控显示，相同书籍的查询每秒重复了上万次。

"这些数据明明很少变化，为什么每次都要查数据库？"小明决定引入 Redis 缓存。

代码很快写好了：

```rust
async fn get_book(pool: &PgPool, redis: &RedisClient, book_id: i64) -> Result<Book> {
    // 先查缓存
    if let Some(cached) = redis.get(&format!("book:{}", book_id)).await? {
        return Ok(serde_json::from_str(&cached)?);
    }

    // 缓存未命中，查数据库
    let book = sqlx::query_as!(Book, "SELECT * FROM books WHERE id = $1", book_id)
        .fetch_one(pool)
        .await?;

    // 写入缓存
    redis.set_ex(&format!("book:{}", book_id), &serde_json::to_string(&book)?, 3600).await?;

    Ok(book)
}
```

促销当天，一切看起来都很顺利。直到客服开始收到投诉："我明明改了价格，为什么页面显示的还是旧的？"

小明查看日志，发现问题了：卖家更新了价格，数据库确实变了，但缓存里还是旧数据。用户看到的是缓存中的过期信息。

**数据同时存在于两个地方，它们之间失去了同步**——这就是缓存一致性问题。

## 问题的本质

缓存一致性问题的根源在于：**数据库和缓存是两个独立的存储系统，对它们的操作无法在同一个事务中完成**。

无论你以什么顺序操作，都可能出问题：

**先更新数据库，再更新缓存**：

```
时刻1: 请求A 更新数据库 price=100
时刻2: 请求B 更新数据库 price=200
时刻3: 请求B 更新缓存 price=200
时刻4: 请求A 更新缓存 price=100  // 并发导致缓存最终是旧值！
```

**先更新缓存，再更新数据库**：

```
时刻1: 请求A 更新缓存 price=100
时刻2: 请求A 更新数据库失败，回滚
时刻3: 缓存中 price=100，但数据库还是旧值  // 脏数据！
```

**先删除缓存，再更新数据库**：

```
时刻1: 请求A 删除缓存
时刻2: 请求B 读取，缓存未命中，从数据库读取旧值
时刻3: 请求B 将旧值写入缓存
时刻4: 请求A 更新数据库  // 缓存中又是旧值！
```

**先更新数据库，再删除缓存**（最常用）：

```
时刻1: 请求A 读取，缓存未命中
时刻2: 请求A 从数据库读取 price=100
时刻3: 请求B 更新数据库 price=200
时刻4: 请求B 删除缓存
时刻5: 请求A 将 price=100 写入缓存  // 又是旧值！
```

看起来无论怎么做都不对？别急，让我们看看业界是如何解决的。

## 方案一：Cache-Aside（旁路缓存）

这是最经典、最广泛使用的方案。核心思想是：**读时填充缓存，写时只删除缓存**。

### 为什么"删除"而不是"更新"？

删除缓存的好处是：
1. **避免并发写冲突**：删除是幂等的，多次删除和一次删除效果相同
2. **简化逻辑**：不需要计算新值，让下一次读请求自然地填充正确的值
3. **避免无效更新**：如果数据更新后根本没人读，更新缓存就是浪费

### 实现

```rust
use redis::AsyncCommands;
use sqlx::PgPool;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Book {
    pub id: i64,
    pub title: String,
    pub price: i64,  // 以分为单位
    pub stock: i32,
    pub updated_at: chrono::DateTime<chrono::Utc>,
}

pub struct BookService {
    pool: PgPool,
    redis: redis::Client,
}

impl BookService {
    /// 读取书籍信息（Cache-Aside 模式）
    pub async fn get_book(&self, book_id: i64) -> Result<Book, Error> {
        let cache_key = format!("book:{}", book_id);
        let mut conn = self.redis.get_multiplexed_async_connection().await?;

        // 1. 先查缓存
        let cached: Option<String> = conn.get(&cache_key).await?;
        if let Some(json) = cached {
            return Ok(serde_json::from_str(&json)?);
        }

        // 2. 缓存未命中，查数据库
        let book = sqlx::query_as!(Book, "SELECT * FROM books WHERE id = $1", book_id)
            .fetch_one(&self.pool)
            .await?;

        // 3. 填充缓存（设置过期时间作为兜底）
        let json = serde_json::to_string(&book)?;
        conn.set_ex(&cache_key, &json, 3600).await?;

        Ok(book)
    }

    /// 更新书籍价格（先更新数据库，再删除缓存）
    pub async fn update_price(&self, book_id: i64, new_price: i64) -> Result<(), Error> {
        // 1. 更新数据库
        sqlx::query!(
            "UPDATE books SET price = $1, updated_at = NOW() WHERE id = $2",
            new_price,
            book_id
        )
        .execute(&self.pool)
        .await?;

        // 2. 删除缓存
        let cache_key = format!("book:{}", book_id);
        let mut conn = self.redis.get_multiplexed_async_connection().await?;
        conn.del(&cache_key).await?;

        Ok(())
    }
}
```

### 仍然存在的问题

前面我们分析过，"先更新数据库，再删除缓存"在极端并发下仍可能出现不一致：

```
时刻1: 读请求发现缓存未命中
时刻2: 读请求查询数据库得到 price=100
时刻3: 写请求更新数据库 price=200
时刻4: 写请求删除缓存
时刻5: 读请求将 price=100 写入缓存  // 旧值！
```

但这个场景发生的概率非常低，因为：
1. 需要读请求的"查库→写缓存"正好跨过写请求的"更新→删缓存"
2. 通常读操作比写操作快得多，这个时间窗口很小
3. 即使发生，缓存有过期时间兜底，不会永远不一致

**Cache-Aside 是工程上的最佳实践，因为它在复杂度和一致性之间取得了很好的平衡。**

### 增强：失败重试

删除缓存可能失败（网络抖动、Redis 故障），需要重试机制：

```rust
impl BookService {
    /// 带重试的缓存删除
    async fn delete_cache_with_retry(&self, cache_key: &str, max_retries: u32) -> Result<(), Error> {
        if max_retries == 0 {
            return Err(anyhow::anyhow!("max_retries must be > 0").into());
        }

        let mut conn = self.redis.get_multiplexed_async_connection().await?;
        let mut last_error = None;

        for attempt in 0..max_retries {
            match conn.del::<_, i64>(cache_key).await {
                Ok(_) => return Ok(()),
                Err(e) => {
                    tracing::warn!(
                        "Failed to delete cache key {}, attempt {}/{}: {}",
                        cache_key, attempt + 1, max_retries, e
                    );
                    last_error = Some(e);

                    if attempt < max_retries - 1 {
                        tokio::time::sleep(Duration::from_millis(100 * (attempt as u64 + 1))).await;
                    }
                }
            }
        }

        Err(last_error.expect("loop ran at least once").into())
    }

    /// 如果重试都失败，发送到消息队列异步删除
    pub async fn update_price_robust(&self, book_id: i64, new_price: i64) -> Result<(), Error> {
        // 1. 更新数据库
        sqlx::query!(
            "UPDATE books SET price = $1, updated_at = NOW() WHERE id = $2",
            new_price,
            book_id
        )
        .execute(&self.pool)
        .await?;

        // 2. 尝试删除缓存
        let cache_key = format!("book:{}", book_id);
        if let Err(e) = self.delete_cache_with_retry(&cache_key, 3).await {
            // 3. 重试失败，发送到消息队列
            tracing::error!("All cache delete retries failed: {}", e);
            self.send_cache_invalidation_message(&cache_key).await?;
        }

        Ok(())
    }
}
```

## 方案二：延迟双删

延迟双删是对 Cache-Aside 的增强，专门解决"先删后更新"的并发问题。

### 核心思想

```
1. 删除缓存
2. 更新数据库
3. 延迟一段时间
4. 再次删除缓存
```

第二次删除的目的是：**清理掉在步骤2期间被其他请求填充的旧缓存**。

### 实现

```rust
impl BookService {
    /// 延迟双删
    pub async fn update_price_double_delete(
        &self,
        book_id: i64,
        new_price: i64,
    ) -> Result<(), Error> {
        let cache_key = format!("book:{}", book_id);
        let mut conn = self.redis.get_multiplexed_async_connection().await?;

        // 1. 第一次删除缓存
        conn.del(&cache_key).await?;

        // 2. 更新数据库
        sqlx::query!(
            "UPDATE books SET price = $1, updated_at = NOW() WHERE id = $2",
            new_price,
            book_id
        )
        .execute(&self.pool)
        .await?;

        // 3. 异步延迟删除（不阻塞主流程）
        let redis_client = self.redis.clone();
        let key = cache_key.clone();
        tokio::spawn(async move {
            // 延迟时间 > 读请求的耗时（通常设为 500ms ~ 1s）
            tokio::time::sleep(Duration::from_millis(500)).await;

            if let Ok(mut conn) = redis_client.get_multiplexed_async_connection().await {
                let _ = conn.del::<_, i64>(&key).await;
            }
        });

        Ok(())
    }
}
```

### 延迟时间如何确定？

延迟时间需要大于"读请求从数据库读取到写入缓存"的时间。可以通过监控来确定：

```rust
/// 监控读请求耗时，动态调整延迟时间
pub struct AdaptiveDelayCalculator {
    read_latencies: Arc<Mutex<VecDeque<Duration>>>,
    window_size: usize,
}

impl AdaptiveDelayCalculator {
    pub fn record_read_latency(&self, latency: Duration) {
        // Mutex::lock 只在 poisoned 时返回 Err（持有锁的线程 panic），这里选择 panic 是合理的
        let mut latencies = self.read_latencies.lock().expect("latencies mutex poisoned");
        if latencies.len() >= self.window_size {
            latencies.pop_front();
        }
        latencies.push_back(latency);
    }

    /// 返回 P99 延迟的 2 倍作为安全边际
    pub fn get_delay(&self) -> Duration {
        let latencies = self.read_latencies.lock().expect("latencies mutex poisoned");
        if latencies.is_empty() {
            return Duration::from_millis(500);  // 默认值
        }

        let mut sorted: Vec<_> = latencies.iter().cloned().collect();
        sorted.sort();

        let p99_index = (sorted.len() as f64 * 0.99) as usize;
        let p99 = sorted.get(p99_index).cloned().unwrap_or(Duration::from_millis(100));

        p99 * 2
    }
}
```

### 局限性

- **增加了写请求的延迟**：虽然异步执行，但增加了系统复杂度
- **延迟时间不好确定**：设短了可能没效果，设长了浪费资源
- **不能完全保证一致性**：极端情况下仍可能失败

## 方案三：Read-Through / Write-Through

这种模式将缓存作为唯一的数据访问入口，由缓存层自动处理数据库的读写。

### 概念

- **Read-Through**：读请求始终走缓存，未命中时由缓存层加载数据
- **Write-Through**：写请求写入缓存，由缓存层同步写入数据库

### 实现

```rust
use async_trait::async_trait;

/// 缓存存储抽象
#[async_trait]
pub trait CacheStore: Send + Sync {
    async fn get(&self, key: &str) -> Result<Option<String>, Error>;
    async fn set(&self, key: &str, value: &str, ttl_secs: u64) -> Result<(), Error>;
    async fn del(&self, key: &str) -> Result<(), Error>;
}

/// 数据库存储抽象
#[async_trait]
pub trait DatabaseStore: Send + Sync {
    async fn load(&self, id: i64) -> Result<Option<Book>, Error>;
    async fn save(&self, book: &Book) -> Result<(), Error>;
}

/// Read-Through / Write-Through 缓存
pub struct ThroughCache<C: CacheStore, D: DatabaseStore> {
    cache: C,
    database: D,
    ttl_secs: u64,
}

impl<C: CacheStore, D: DatabaseStore> ThroughCache<C, D> {
    /// Read-Through: 始终从缓存读取，未命中自动加载
    pub async fn get(&self, id: i64) -> Result<Option<Book>, Error> {
        let key = format!("book:{}", id);

        // 1. 查缓存
        if let Some(json) = self.cache.get(&key).await? {
            return Ok(Some(serde_json::from_str(&json)?));
        }

        // 2. 未命中，从数据库加载
        let book = match self.database.load(id).await? {
            Some(b) => b,
            None => return Ok(None),
        };

        // 3. 写入缓存
        let json = serde_json::to_string(&book)?;
        self.cache.set(&key, &json, self.ttl_secs).await?;

        Ok(Some(book))
    }

    /// Write-Through: 同时写入缓存和数据库
    pub async fn save(&self, book: &Book) -> Result<(), Error> {
        let key = format!("book:{}", book.id);

        // 1. 先写数据库（保证数据持久化）
        self.database.save(book).await?;

        // 2. 再更新缓存
        let json = serde_json::to_string(&book)?;
        self.cache.set(&key, &json, self.ttl_secs).await?;

        Ok(())
    }
}
```

### 问题

Write-Through 有严重的并发问题：

```
时刻1: 请求A 写数据库 price=100
时刻2: 请求B 写数据库 price=200
时刻3: 请求B 写缓存 price=200
时刻4: 请求A 写缓存 price=100  // 缓存和数据库不一致！
```

**因此，Write-Through 必须配合分布式锁或版本号才能安全使用。**

## 方案四：Write-Behind（异步写回）

Write-Behind 是一种激进的优化策略：**写请求只更新缓存，异步批量写入数据库**。

### 实现

```rust
use tokio::sync::mpsc;

pub struct WriteBehindCache {
    cache: redis::Client,
    write_queue: mpsc::Sender<WriteOperation>,
}

#[derive(Debug, Clone)]
struct WriteOperation {
    book_id: i64,
    book_data: Book,
}

impl WriteBehindCache {
    pub fn new(cache: redis::Client, pool: PgPool) -> Self {
        let (tx, rx) = mpsc::channel(10000);

        // 启动后台写入线程
        tokio::spawn(Self::background_writer(rx, pool));

        Self { cache, write_queue: tx }
    }

    /// 只写缓存，异步落库
    pub async fn update(&self, book: Book) -> Result<(), Error> {
        let key = format!("book:{}", book.id);
        let mut conn = self.cache.get_multiplexed_async_connection().await?;

        // 1. 写入缓存
        let json = serde_json::to_string(&book)?;
        conn.set_ex(&key, &json, 3600).await?;

        // 2. 发送到写入队列
        self.write_queue.send(WriteOperation {
            book_id: book.id,
            book_data: book,
        }).await?;

        Ok(())
    }

    /// 后台批量写入数据库
    async fn background_writer(mut rx: mpsc::Receiver<WriteOperation>, pool: PgPool) {
        let mut batch = Vec::new();
        let mut interval = tokio::time::interval(Duration::from_millis(100));

        loop {
            tokio::select! {
                Some(op) = rx.recv() => {
                    batch.push(op);

                    // 攒够 100 条或队列空了就写入
                    if batch.len() >= 100 {
                        Self::flush_batch(&pool, &mut batch).await;
                    }
                }
                _ = interval.tick() => {
                    // 定时刷新
                    if !batch.is_empty() {
                        Self::flush_batch(&pool, &mut batch).await;
                    }
                }
            }
        }
    }

    async fn flush_batch(pool: &PgPool, batch: &mut Vec<WriteOperation>) {
        // 使用 UPSERT 批量写入
        let mut tx = match pool.begin().await {
            Ok(tx) => tx,
            Err(e) => {
                tracing::error!("Failed to start transaction: {}", e);
                return;
            }
        };

        for op in batch.drain(..) {
            if let Err(e) = sqlx::query!(
                r#"
                INSERT INTO books (id, title, price, stock, updated_at)
                VALUES ($1, $2, $3, $4, $5)
                ON CONFLICT (id) DO UPDATE SET
                    title = EXCLUDED.title,
                    price = EXCLUDED.price,
                    stock = EXCLUDED.stock,
                    updated_at = EXCLUDED.updated_at
                "#,
                op.book_data.id,
                op.book_data.title,
                op.book_data.price,
                op.book_data.stock,
                op.book_data.updated_at,
            )
            .execute(&mut *tx)
            .await {
                tracing::error!("Failed to write book {}: {}", op.book_id, e);
            }
        }

        if let Err(e) = tx.commit().await {
            tracing::error!("Failed to commit batch: {}", e);
        }
    }
}
```

### 风险

- **数据可能丢失**：缓存宕机时，未落库的数据会丢失
- **无法提供强一致性读**：数据库可能滞后于缓存

**Write-Behind 只适合对数据丢失容忍度较高的场景**，如：
- 阅读计数、点赞数
- 用户行为日志
- 实时性要求不高的统计数据

## 方案五：订阅 PostgreSQL 逻辑复制

前面的方案都需要业务代码主动维护缓存，有没有办法让数据库变更自动同步到缓存？

PostgreSQL 的逻辑复制（Logical Replication）可以做到：**订阅数据库的变更流，自动更新缓存**。

### 架构

```
Application ─────────────────────────────────> PostgreSQL
                                                    │
                                                    │ WAL (逻辑复制)
                                                    ▼
                                              CDC Consumer
                                                    │
                                                    │ 更新/删除
                                                    ▼
                                                  Redis
```

### 配置 PostgreSQL

```sql
-- 1. 修改 postgresql.conf
-- wal_level = logical

-- 2. 创建发布
CREATE PUBLICATION book_changes FOR TABLE books;

-- 3. 创建复制槽
SELECT pg_create_logical_replication_slot('book_cache_slot', 'pgoutput');
```

### 实现 CDC Consumer

```rust
use postgres_protocol::message::backend::LogicalReplicationMessage;
use tokio_postgres::replication::LogicalReplicationStream;

pub struct CdcConsumer {
    redis: redis::Client,
}

impl CdcConsumer {
    pub async fn start(&self, connection_string: &str) -> Result<(), Error> {
        let (client, connection) = tokio_postgres::connect(
            &format!("{} replication=database", connection_string),
            tokio_postgres::NoTls,
        ).await?;

        tokio::spawn(async move {
            if let Err(e) = connection.await {
                tracing::error!("Connection error: {}", e);
            }
        });

        // 开始复制
        let query = r#"
            START_REPLICATION SLOT book_cache_slot LOGICAL 0/0
            (proto_version '1', publication_names 'book_changes')
        "#;

        let stream = client
            .copy_both_simple::<bytes::Bytes>(query)
            .await?;

        let mut stream = LogicalReplicationStream::new(stream);

        while let Some(msg) = stream.next().await {
            match msg? {
                LogicalReplicationMessage::Begin(_) => {}
                LogicalReplicationMessage::Commit(_) => {}
                LogicalReplicationMessage::Insert(insert) => {
                    self.handle_insert(&insert).await?;
                }
                LogicalReplicationMessage::Update(update) => {
                    self.handle_update(&update).await?;
                }
                LogicalReplicationMessage::Delete(delete) => {
                    self.handle_delete(&delete).await?;
                }
                _ => {}
            }
        }

        Ok(())
    }

    async fn handle_insert(&self, insert: &InsertBody) -> Result<(), Error> {
        // 新插入的数据不需要删缓存，让下次读取时自然填充
        let book_id = self.extract_book_id(insert.tuple())?;
        tracing::debug!("New book inserted: {}", book_id);
        Ok(())
    }

    async fn handle_update(&self, update: &UpdateBody) -> Result<(), Error> {
        // 解析新数据
        let book_id = self.extract_book_id(update.new_tuple())?;

        // 删除缓存（让下次读取时重新加载）
        let mut conn = self.redis.get_multiplexed_async_connection().await?;
        conn.del(&format!("book:{}", book_id)).await?;

        tracing::info!("Invalidated cache for book {}", book_id);
        Ok(())
    }

    async fn handle_delete(&self, delete: &DeleteBody) -> Result<(), Error> {
        let book_id = self.extract_book_id(delete.key_tuple())?;

        let mut conn = self.redis.get_multiplexed_async_connection().await?;
        conn.del(&format!("book:{}", book_id)).await?;

        Ok(())
    }

    /// 从 tuple 数据中提取 book_id（假设 id 是第一个字段）
    fn extract_book_id(&self, tuple: Option<&TupleData>) -> Result<i64, Error> {
        let tuple = tuple.ok_or_else(|| anyhow::anyhow!("Missing tuple data"))?;
        // TupleData 包含列值，第一列是 id
        // 实际解析逻辑取决于 pgoutput 格式
        let id_bytes = tuple.values().first()
            .ok_or_else(|| anyhow::anyhow!("No columns in tuple"))?;
        // 简化示例：实际需要根据列类型解析
        let id_str = std::str::from_utf8(id_bytes)?;
        Ok(id_str.parse()?)
    }
}
```

### 优点

- **业务代码无侵入**：不需要修改任何业务代码
- **不会遗漏**：任何途径的数据变更都会被捕获
- **支持跨语言**：多个服务可以共享同一份缓存

### 局限性

- **有延迟**：从数据库变更到缓存更新有秒级延迟
- **运维复杂**：需要维护 CDC 组件
- **存在积压风险**：消费速度跟不上生产速度会导致 WAL 积压

## 方案六：PostgreSQL LISTEN/NOTIFY

如果觉得逻辑复制太重，PostgreSQL 还提供了轻量级的通知机制。

### 实现

```rust
/// 在更新时发送通知
pub async fn update_price_with_notify(
    pool: &PgPool,
    book_id: i64,
    new_price: i64,
) -> Result<(), Error> {
    // 使用事务确保更新和通知的原子性
    let mut tx = pool.begin().await?;

    // 1. 更新数据
    sqlx::query!(
        "UPDATE books SET price = $1, updated_at = NOW() WHERE id = $2",
        new_price,
        book_id
    )
    .execute(&mut *tx)
    .await?;

    // 2. 发送通知（pg_notify 会在事务提交时发出）
    sqlx::query!("SELECT pg_notify('book_changes', $1::text)", book_id)
        .execute(&mut *tx)
        .await?;

    tx.commit().await?;

    Ok(())
}

/// 监听通知并更新缓存
pub struct NotifyListener {
    pool: PgPool,
    redis: redis::Client,
}

impl NotifyListener {
    pub async fn start(&self) -> Result<(), Error> {
        let mut listener = sqlx::postgres::PgListener::connect_with(&self.pool).await?;
        listener.listen("book_changes").await?;

        tracing::info!("Started listening for book_changes");

        loop {
            let notification = listener.recv().await?;
            let book_id: i64 = notification.payload().parse()?;

            // 删除缓存
            let mut conn = self.redis.get_multiplexed_async_connection().await?;
            conn.del(&format!("book:{}", book_id)).await?;

            tracing::info!("Invalidated cache for book {} via NOTIFY", book_id);
        }
    }
}
```

### 使用触发器自动发送通知

```sql
-- 创建触发器函数
CREATE OR REPLACE FUNCTION notify_book_change()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('book_changes', NEW.id::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 创建触发器
CREATE TRIGGER book_change_trigger
    AFTER INSERT OR UPDATE ON books
    FOR EACH ROW
    EXECUTE FUNCTION notify_book_change();
```

### 局限性

- **单机限制**：LISTEN/NOTIFY 只在单个数据库连接上工作
- **消息可能丢失**：如果监听者断开，期间的通知会丢失
- **无持久化**：不像逻辑复制，通知没有持久化和重放能力

## 方案比较

| 方案 | 一致性保证 | 实现复杂度 | 侵入性 | 延迟 | 适用场景 |
|------|-----------|-----------|--------|------|---------|
| Cache-Aside | 最终一致 | 低 | 中 | 低 | 通用场景，首选方案 |
| 延迟双删 | 最终一致（更强） | 中 | 中 | 中 | 对一致性要求较高的读场景 |
| Write-Through | 最终一致 | 中 | 高 | 低 | 需要严格控制缓存的场景 |
| Write-Behind | 弱一致 | 高 | 高 | 最低 | 容忍数据丢失的高写入场景 |
| 逻辑复制 CDC | 最终一致 | 高 | 无 | 秒级 | 多服务共享缓存，存量系统 |
| LISTEN/NOTIFY | 最终一致 | 低 | 低 | 低 | 单机或简单集群场景 |

## 决策树

```
需要缓存一致性
    │
    ├─ 是新项目 / 能改代码吗？
    │   ├─ 是 → 选择 Cache-Aside
    │   │         │
    │   │         ├─ 写少读多，能接受极低概率不一致？ → 基础版 Cache-Aside
    │   │         └─ 需要更强一致性？ → 延迟双删
    │   │
    │   └─ 否 → 选择 CDC
    │             │
    │             ├─ 有 Kafka/消息队列基础设施？ → Debezium + Kafka
    │             └─ 想要轻量方案？ → LISTEN/NOTIFY + 触发器
    │
    └─ 特殊场景
        ├─ 超高写入，能容忍丢数据？ → Write-Behind
        └─ 需要缓存层完全托管？ → Read/Write-Through + 分布式锁
```

## 小明的选择

分析了所有方案后，小明决定：

1. **主方案**：Cache-Aside，简单可靠
2. **增强措施**：删除缓存失败时发送到消息队列重试
3. **兜底机制**：所有缓存设置 1 小时过期时间

```rust
impl BookService {
    pub async fn update_book(&self, book_id: i64, updates: BookUpdates) -> Result<(), Error> {
        // 1. 更新数据库
        sqlx::query!(
            "UPDATE books SET price = COALESCE($1, price), stock = COALESCE($2, stock), updated_at = NOW() WHERE id = $3",
            updates.price,
            updates.stock,
            book_id
        )
        .execute(&self.pool)
        .await?;

        // 2. 删除缓存（带重试）
        let cache_key = format!("book:{}", book_id);
        if let Err(e) = self.delete_cache_with_retry(&cache_key, 3).await {
            // 3. 重试失败，发到消息队列
            self.message_queue.send(CacheInvalidation { key: cache_key }).await?;
            tracing::warn!("Cache deletion failed, queued for retry: {}", e);
        }

        Ok(())
    }
}
```

"不追求完美，追求足够好。"小明总结道。

## 实战建议

1. **从 Cache-Aside 开始**：不要过度设计，这个方案已经够用
2. **永远设置过期时间**：作为一致性的最后兜底
3. **监控缓存命中率**：太低说明缓存策略有问题，太高可能隐藏了不一致
4. **区分数据类型**：
   - 强一致性数据（账户余额）：不缓存，或缓存时间极短
   - 最终一致性数据（商品详情）：正常缓存
   - 静态数据（系统配置）：长时间缓存
5. **做好降级**：Redis 故障时能直接读数据库

## 下一步

缓存问题解决后，小明的平台继续发展。用户量持续增长，数据库里的书籍和订单越来越多。

"单表已经有几千万条数据了，查询越来越慢。"DBA 在周会上提出警告。

小明查看监控，发现复杂查询动辄需要几秒，热门时段甚至超时。读写分离和缓存只能解决读的问题，写入和复杂查询的压力还是压在主库上。

"得分库分表了。"技术 leader 说，"把订单表按用户 ID 哈希分到多个库。"

但分库分表后，跨分片的查询怎么做？如果一个操作需要修改多个分片的数据，如何保证一致性？

这就是下一章的主题：**跨分片事务一致性**。

---

*上一篇：[读写分离：副本一致性](/posts/consistency-in-practice-2-read-write-split/)*
*下一篇：[数据分片：跨分片事务一致性](/posts/consistency-in-practice-4-sharding/)*

*本系列：*
1. *[单机时代：ACID 的庇护](/posts/consistency-in-practice-1-single-node/)*
2. *[读写分离：副本一致性](/posts/consistency-in-practice-2-read-write-split/)*
3. *引入缓存：多存储一致性（本篇）*
4. *[数据分片：跨分片事务一致性](/posts/consistency-in-practice-4-sharding/)*
5. *[服务拆分：跨服务事务一致性](/posts/consistency-in-practice-5-microservices/)*
6. *[消息驱动：最终一致性](/posts/consistency-in-practice-6-messaging/)*
7. *[总结：演进全景与选型指南](/posts/consistency-in-practice-7-summary/)*
