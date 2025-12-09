---
title: "从单机到分布式：一致性实战（四）数据分片：跨分片事务一致性"
date: 2025-12-11T10:30:00+08:00
draft: false
tags: ["一致性", "分库分表", "分片", "PostgreSQL", "Citus", "Rust"]
categories: ["技术"]
description: "当数据被分散到多个数据库时，如何保证跨分片查询的一致性和跨分片事务的原子性？本文详解分片策略与跨分片事务的实现方案"
series: ["从单机到分布式：一致性实战"]
---

## 从这里开始

经过缓存优化后，小明的二手书平台性能有了很大提升。但新的问题出现了：**数据量太大，单个数据库快撑不住了**。

订单表已经有上亿条数据，每次查询都很慢，添加索引也无济于事。磁盘空间也快满了，单机存储已经到达瓶颈。

DBA 看了监控后说："是时候分库分表了。"

小明决定按 `user_id` 将订单数据分到 4 个数据库分片中：

```
user_id % 4 = 0 → shard_0
user_id % 4 = 1 → shard_1
user_id % 4 = 2 → shard_2
user_id % 4 = 3 → shard_3
```

分片后，单个用户的订单查询飞快了。但很快遇到了新问题：

**问题一：跨分片查询**

"给我查最近 7 天所有用户的订单总额。"运营说。

这意味着要查询所有 4 个分片，然后合并结果。

**问题二：跨分片事务**

"用户 A（在 shard_0）想把一本书转让给用户 B（在 shard_1）。"

这涉及两个分片的数据修改，如何保证原子性？

**问题三：分片键变更**

"用户要改手机号，但我们是按手机号分片的..."

分片键变更意味着数据要迁移到另一个分片。

这就是数据分片带来的一致性挑战。

## 分片基础

### 分片路由

首先实现一个分片路由器：

```rust
use sqlx::PgPool;
use std::collections::HashMap;
use std::sync::Arc;

/// 分片路由器
pub struct ShardRouter {
    shards: Vec<Arc<PgPool>>,
    shard_count: usize,
}

impl ShardRouter {
    pub fn new(shard_pools: Vec<PgPool>) -> Self {
        let shard_count = shard_pools.len();
        Self {
            shards: shard_pools.into_iter().map(Arc::new).collect(),
            shard_count,
        }
    }

    /// 根据分片键获取分片
    pub fn get_shard(&self, shard_key: i64) -> &PgPool {
        let shard_index = (shard_key as usize) % self.shard_count;
        &self.shards[shard_index]
    }

    /// 获取所有分片（用于跨分片查询）
    pub fn all_shards(&self) -> &[Arc<PgPool>] {
        &self.shards
    }

    /// 根据分片键计算分片索引
    pub fn shard_index(&self, shard_key: i64) -> usize {
        (shard_key as usize) % self.shard_count
    }
}
```

### 基本的分片读写

```rust
pub struct ShardedOrderRepository {
    router: ShardRouter,
}

impl ShardedOrderRepository {
    /// 写入订单（路由到对应分片）
    pub async fn create_order(&self, order: &Order) -> Result<(), Error> {
        let pool = self.router.get_shard(order.user_id);

        sqlx::query!(
            r#"
            INSERT INTO orders (id, user_id, book_id, amount, status, created_at)
            VALUES ($1, $2, $3, $4, $5, NOW())
            "#,
            order.id,
            order.user_id,
            order.book_id,
            order.amount,
            order.status,
        )
        .execute(pool)
        .await?;

        Ok(())
    }

    /// 查询用户订单（单分片查询）
    pub async fn get_user_orders(&self, user_id: i64) -> Result<Vec<Order>, Error> {
        let pool = self.router.get_shard(user_id);

        let orders = sqlx::query_as!(Order,
            "SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC",
            user_id
        )
        .fetch_all(pool)
        .await?;

        Ok(orders)
    }
}
```

## 问题一：跨分片查询

当查询条件不包含分片键时，需要查询所有分片。

### 方案一：并行查询所有分片

```rust
impl ShardedOrderRepository {
    /// 跨分片查询：查询所有分片并合并结果
    pub async fn get_all_orders_in_period(
        &self,
        start_date: chrono::DateTime<chrono::Utc>,
        end_date: chrono::DateTime<chrono::Utc>,
        limit: i64,
    ) -> Result<Vec<Order>, Error> {
        let shards = self.router.all_shards();

        // 并行查询所有分片
        let futures: Vec<_> = shards
            .iter()
            .map(|pool| {
                let pool = pool.clone();
                let start = start_date;
                let end = end_date;

                async move {
                    sqlx::query_as!(Order,
                        r#"
                        SELECT * FROM orders
                        WHERE created_at >= $1 AND created_at < $2
                        ORDER BY created_at DESC
                        LIMIT $3
                        "#,
                        start,
                        end,
                        limit,
                    )
                    .fetch_all(pool.as_ref())
                    .await
                }
            })
            .collect();

        let results = futures::future::try_join_all(futures).await?;

        // 合并结果
        let mut all_orders: Vec<Order> = results.into_iter().flatten().collect();

        // 全局排序
        all_orders.sort_by(|a, b| b.created_at.cmp(&a.created_at));

        // 截取 limit
        all_orders.truncate(limit as usize);

        Ok(all_orders)
    }
}
```

### 方案二：索引表（全局索引）

创建一个中心化的索引表，存储分片位置信息：

```rust
/// 全局订单索引
pub struct GlobalOrderIndex {
    index_pool: PgPool,  // 索引库
    router: ShardRouter,  // 分片路由
}

impl GlobalOrderIndex {
    /// 创建订单时同时写入索引
    pub async fn create_order_with_index(&self, order: &Order) -> Result<(), Error> {
        // 1. 写入分片数据
        let shard_pool = self.router.get_shard(order.user_id);
        sqlx::query!(
            "INSERT INTO orders (id, user_id, book_id, amount, status) VALUES ($1, $2, $3, $4, $5)",
            order.id, order.user_id, order.book_id, order.amount, order.status
        )
        .execute(shard_pool)
        .await?;

        // 2. 写入全局索引
        let shard_index = self.router.shard_index(order.user_id) as i32;
        sqlx::query!(
            r#"
            INSERT INTO order_index (order_id, user_id, book_id, shard_index, created_at)
            VALUES ($1, $2, $3, $4, NOW())
            "#,
            order.id,
            order.user_id,
            order.book_id,
            shard_index,
        )
        .execute(&self.index_pool)
        .await?;

        Ok(())
    }

    /// 按书籍查询订单（使用全局索引）
    pub async fn get_orders_by_book(&self, book_id: i64) -> Result<Vec<Order>, Error> {
        // 1. 从索引表查询订单位置
        let index_records = sqlx::query!(
            "SELECT order_id, shard_index FROM order_index WHERE book_id = $1",
            book_id
        )
        .fetch_all(&self.index_pool)
        .await?;

        // 2. 按分片分组
        let mut shard_orders: HashMap<i32, Vec<i64>> = HashMap::new();
        for record in index_records {
            shard_orders
                .entry(record.shard_index)
                .or_default()
                .push(record.order_id);
        }

        // 3. 并行查询各分片
        let futures: Vec<_> = shard_orders
            .into_iter()
            .map(|(shard_index, order_ids)| {
                let pool = self.router.all_shards()[shard_index as usize].clone();

                async move {
                    sqlx::query_as!(Order,
                        "SELECT * FROM orders WHERE id = ANY($1)",
                        &order_ids
                    )
                    .fetch_all(pool.as_ref())
                    .await
                }
            })
            .collect();

        let results = futures::future::try_join_all(futures).await?;
        let orders: Vec<Order> = results.into_iter().flatten().collect();

        Ok(orders)
    }
}
```

### 方案三：CQRS（命令查询分离）

将写入和查询分离，写入走分片，查询走专门的查询库：

```rust
pub struct CqrsOrderService {
    command_router: ShardRouter,       // 写入用分片路由
    query_pool: PgPool,                // 查询用汇总库
    event_publisher: EventPublisher,   // 事件发布器
}

impl CqrsOrderService {
    /// 写入：发送到分片
    pub async fn create_order(&self, order: &Order) -> Result<(), Error> {
        let pool = self.command_router.get_shard(order.user_id);

        // 写入分片
        sqlx::query!(
            "INSERT INTO orders (id, user_id, book_id, amount, status) VALUES ($1, $2, $3, $4, $5)",
            order.id, order.user_id, order.book_id, order.amount, order.status
        )
        .execute(pool)
        .await?;

        // 发布事件，异步同步到查询库
        self.event_publisher.publish(OrderCreatedEvent {
            order: order.clone(),
        }).await?;

        Ok(())
    }

    /// 查询：从汇总库读取
    pub async fn query_orders(&self, query: OrderQuery) -> Result<Vec<Order>, Error> {
        let orders = sqlx::query_as!(Order,
            r#"
            SELECT * FROM orders_view
            WHERE ($1::bigint IS NULL OR user_id = $1)
              AND ($2::bigint IS NULL OR book_id = $2)
              AND ($3::timestamptz IS NULL OR created_at >= $3)
            ORDER BY created_at DESC
            LIMIT $4
            "#,
            query.user_id,
            query.book_id,
            query.start_date,
            query.limit,
        )
        .fetch_all(&self.query_pool)
        .await?;

        Ok(orders)
    }
}

/// 事件处理器：同步数据到查询库
pub struct QueryModelUpdater {
    query_pool: PgPool,
}

impl QueryModelUpdater {
    pub async fn handle_order_created(&self, event: OrderCreatedEvent) -> Result<(), Error> {
        sqlx::query!(
            r#"
            INSERT INTO orders_view (id, user_id, book_id, amount, status, created_at)
            VALUES ($1, $2, $3, $4, $5, $6)
            ON CONFLICT (id) DO UPDATE SET
                status = EXCLUDED.status,
                updated_at = NOW()
            "#,
            event.order.id,
            event.order.user_id,
            event.order.book_id,
            event.order.amount,
            event.order.status,
            event.order.created_at,
        )
        .execute(&self.query_pool)
        .await?;

        Ok(())
    }
}
```

### 跨分片查询方案比较

| 方案 | 一致性 | 实时性 | 查询性能 | 写入性能 | 复杂度 |
|------|--------|--------|---------|---------|--------|
| 并行查询所有分片 | 强一致 | 实时 | 差 | 好 | 低 |
| 全局索引表 | 最终一致 | 近实时 | 中 | 中 | 中 |
| CQRS | 最终一致 | 延迟 | 好 | 好 | 高 |

## 问题二：跨分片事务

当一个操作需要修改多个分片的数据时，如何保证原子性？

### 场景：用户转账

用户 A（shard_0）向用户 B（shard_1）转账：

```rust
// 这段代码有问题！
async fn transfer_wrong(
    router: &ShardRouter,
    from_user: i64,
    to_user: i64,
    amount: i64,
) -> Result<(), Error> {
    let from_pool = router.get_shard(from_user);
    let to_pool = router.get_shard(to_user);

    // 1. 扣减 A 的余额
    sqlx::query!("UPDATE users SET balance = balance - $1 WHERE id = $2", amount, from_user)
        .execute(from_pool)
        .await?;

    // 2. 增加 B 的余额
    // 如果这里失败了，A 的钱扣了，B 没收到！
    sqlx::query!("UPDATE users SET balance = balance + $1 WHERE id = $2", amount, to_user)
        .execute(to_pool)
        .await?;

    Ok(())
}
```

### 方案一：应用层 Saga

使用 Saga 模式处理跨分片事务：

```rust
pub struct TransferSaga {
    router: ShardRouter,
}

impl TransferSaga {
    pub async fn transfer(
        &self,
        from_user: i64,
        to_user: i64,
        amount: i64,
    ) -> Result<Uuid, Error> {
        let saga_id = Uuid::new_v4();
        let mut completed_steps: Vec<&str> = Vec::new();

        // 步骤1：冻结转出方余额
        let result = self.freeze_balance(saga_id, from_user, amount).await;
        if let Err(e) = result {
            return Err(e);
        }
        completed_steps.push("freeze");

        // 步骤2：增加接收方余额
        let result = self.credit_balance(saga_id, to_user, amount).await;
        if let Err(e) = result {
            self.compensate(&completed_steps, saga_id, from_user, to_user, amount).await;
            return Err(e);
        }
        completed_steps.push("credit");

        // 步骤3：确认扣减转出方余额
        let result = self.confirm_debit(saga_id, from_user, amount).await;
        if let Err(e) = result {
            self.compensate(&completed_steps, saga_id, from_user, to_user, amount).await;
            return Err(e);
        }

        Ok(saga_id)
    }

    async fn freeze_balance(&self, saga_id: Uuid, user_id: i64, amount: i64) -> Result<(), Error> {
        let pool = self.router.get_shard(user_id);

        let result = sqlx::query!(
            r#"
            UPDATE users
            SET balance = balance - $1, frozen_balance = frozen_balance + $1
            WHERE id = $2 AND balance >= $1
            "#,
            amount,
            user_id
        )
        .execute(pool)
        .await?;

        if result.rows_affected() == 0 {
            return Err(Error::InsufficientBalance);
        }

        // 记录 Saga 状态
        sqlx::query!(
            "INSERT INTO saga_logs (saga_id, step, user_id, amount) VALUES ($1, 'freeze', $2, $3)",
            saga_id, user_id, amount
        )
        .execute(pool)
        .await?;

        Ok(())
    }

    async fn credit_balance(&self, saga_id: Uuid, user_id: i64, amount: i64) -> Result<(), Error> {
        let pool = self.router.get_shard(user_id);

        sqlx::query!(
            "UPDATE users SET balance = balance + $1 WHERE id = $2",
            amount,
            user_id
        )
        .execute(pool)
        .await?;

        sqlx::query!(
            "INSERT INTO saga_logs (saga_id, step, user_id, amount) VALUES ($1, 'credit', $2, $3)",
            saga_id, user_id, amount
        )
        .execute(pool)
        .await?;

        Ok(())
    }

    async fn confirm_debit(&self, saga_id: Uuid, user_id: i64, amount: i64) -> Result<(), Error> {
        let pool = self.router.get_shard(user_id);

        // 解冻并确认扣减
        sqlx::query!(
            "UPDATE users SET frozen_balance = frozen_balance - $1 WHERE id = $2",
            amount,
            user_id
        )
        .execute(pool)
        .await?;

        sqlx::query!(
            "UPDATE saga_logs SET status = 'confirmed' WHERE saga_id = $1 AND step = 'freeze'",
            saga_id
        )
        .execute(pool)
        .await?;

        Ok(())
    }

    async fn compensate(
        &self,
        completed_steps: &[&str],
        saga_id: Uuid,
        from_user: i64,
        to_user: i64,
        amount: i64,
    ) {
        for step in completed_steps.iter().rev() {
            let result = match *step {
                "freeze" => self.unfreeze_balance(saga_id, from_user, amount).await,
                "credit" => self.debit_balance(saga_id, to_user, amount).await,
                _ => Ok(()),
            };

            if let Err(e) = result {
                tracing::error!("Compensation failed for step {}: {}", step, e);
                // 记录待人工处理
            }
        }
    }

    async fn unfreeze_balance(&self, saga_id: Uuid, user_id: i64, amount: i64) -> Result<(), Error> {
        let pool = self.router.get_shard(user_id);

        sqlx::query!(
            "UPDATE users SET balance = balance + $1, frozen_balance = frozen_balance - $1 WHERE id = $2",
            amount, user_id
        )
        .execute(pool)
        .await?;

        // 记录补偿操作
        sqlx::query!(
            "UPDATE saga_logs SET status = 'compensated' WHERE saga_id = $1 AND step = 'freeze'",
            saga_id
        )
        .execute(pool)
        .await?;

        Ok(())
    }

    async fn debit_balance(&self, saga_id: Uuid, user_id: i64, amount: i64) -> Result<(), Error> {
        let pool = self.router.get_shard(user_id);

        sqlx::query!(
            "UPDATE users SET balance = balance - $1 WHERE id = $2",
            amount, user_id
        )
        .execute(pool)
        .await?;

        // 记录补偿操作
        sqlx::query!(
            "UPDATE saga_logs SET status = 'compensated' WHERE saga_id = $1 AND step = 'credit'",
            saga_id
        )
        .execute(pool)
        .await?;

        Ok(())
    }
}
```

### 方案二：两阶段提交（2PC）

虽然 2PC 有性能问题，但某些场景下仍然有用：

```rust
/// 简化版 2PC 协调器
pub struct TwoPhaseCommitCoordinator {
    router: ShardRouter,
}

impl TwoPhaseCommitCoordinator {
    pub async fn transfer(
        &self,
        from_user: i64,
        to_user: i64,
        amount: i64,
    ) -> Result<(), Error> {
        let tx_id = Uuid::new_v4();

        // Phase 1: Prepare
        let prepare_results = self.prepare_all(tx_id, from_user, to_user, amount).await;

        if prepare_results.iter().all(|r| r.is_ok()) {
            // Phase 2: Commit
            self.commit_all(tx_id, from_user, to_user).await?;
        } else {
            // Phase 2: Rollback
            self.rollback_all(tx_id, from_user, to_user).await?;
            return Err(Error::TransactionFailed);
        }

        Ok(())
    }

    async fn prepare_all(
        &self,
        tx_id: Uuid,
        from_user: i64,
        to_user: i64,
        amount: i64,
    ) -> Vec<Result<(), Error>> {
        let from_pool = self.router.get_shard(from_user);
        let to_pool = self.router.get_shard(to_user);

        // 注意：以下使用 format! 构造 SQL 仅用于演示 PostgreSQL 的 PREPARE TRANSACTION 语法
        // 生产环境中应该：
        // 1. 使用原生连接执行多条语句，或
        // 2. 使用支持 XA 事务的连接池/驱动
        // 这里的参数都是 i64 和 Uuid 类型，不存在 SQL 注入风险

        let from_prepare = async {
            // 使用 PostgreSQL 的 PREPARE TRANSACTION
            sqlx::query(&format!(
                "BEGIN; \
                 UPDATE users SET balance = balance - {} WHERE id = {} AND balance >= {}; \
                 PREPARE TRANSACTION '{}';",
                amount, from_user, amount, tx_id
            ))
            .execute(from_pool)
            .await?;
            Ok::<_, Error>(())
        };

        let to_prepare = async {
            sqlx::query(&format!(
                "BEGIN; \
                 UPDATE users SET balance = balance + {} WHERE id = {}; \
                 PREPARE TRANSACTION '{}';",
                amount, to_user, tx_id
            ))
            .execute(to_pool)
            .await?;
            Ok::<_, Error>(())
        };

        let (from_result, to_result) = tokio::join!(from_prepare, to_prepare);
        vec![from_result, to_result]
    }

    async fn commit_all(
        &self,
        tx_id: Uuid,
        from_user: i64,
        to_user: i64,
    ) -> Result<(), Error> {
        let from_pool = self.router.get_shard(from_user);
        let to_pool = self.router.get_shard(to_user);

        let commit_sql = format!("COMMIT PREPARED '{}'", tx_id);

        tokio::try_join!(
            sqlx::query(&commit_sql).execute(from_pool),
            sqlx::query(&commit_sql).execute(to_pool),
        )?;

        Ok(())
    }

    async fn rollback_all(
        &self,
        tx_id: Uuid,
        from_user: i64,
        to_user: i64,
    ) -> Result<(), Error> {
        let from_pool = self.router.get_shard(from_user);
        let to_pool = self.router.get_shard(to_user);

        let rollback_sql = format!("ROLLBACK PREPARED '{}'", tx_id);

        // 忽略回滚错误（可能某些分片根本没 prepare 成功）
        let _ = sqlx::query(&rollback_sql).execute(from_pool).await;
        let _ = sqlx::query(&rollback_sql).execute(to_pool).await;

        Ok(())
    }
}
```

### 方案三：避免跨分片事务

最好的跨分片事务是没有跨分片事务。通过设计避免：

```rust
/// 方案A：将相关数据放在同一个分片
/// 例如：按 transfer_id 分片，而不是按 user_id 分片

/// 方案B：使用消息驱动的最终一致性
pub struct EventuallyConsistentTransfer {
    router: ShardRouter,
    outbox: TransactionalOutbox,
}

impl EventuallyConsistentTransfer {
    pub async fn transfer(
        &self,
        from_user: i64,
        to_user: i64,
        amount: i64,
    ) -> Result<Uuid, Error> {
        let transfer_id = Uuid::new_v4();
        let from_pool = self.router.get_shard(from_user);

        let mut tx = from_pool.begin().await?;

        // 1. 扣减转出方余额
        let result = sqlx::query!(
            "UPDATE users SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
            amount,
            from_user
        )
        .execute(&mut *tx)
        .await?;

        if result.rows_affected() == 0 {
            return Err(Error::InsufficientBalance);
        }

        // 2. 记录转账事件
        sqlx::query!(
            r#"
            INSERT INTO transfer_events (id, from_user, to_user, amount, status)
            VALUES ($1, $2, $3, $4, 'pending')
            "#,
            transfer_id,
            from_user,
            to_user,
            amount,
        )
        .execute(&mut *tx)
        .await?;

        // 3. 写入 outbox 消息
        sqlx::query!(
            r#"
            INSERT INTO outbox (id, topic, payload)
            VALUES ($1, 'transfer-credits', $2)
            "#,
            Uuid::new_v4(),
            serde_json::json!({
                "transfer_id": transfer_id,
                "to_user": to_user,
                "amount": amount,
            }),
        )
        .execute(&mut *tx)
        .await?;

        tx.commit().await?;

        Ok(transfer_id)
    }
}

/// 消费者：处理转账入账
pub struct TransferCreditConsumer {
    router: ShardRouter,
}

impl TransferCreditConsumer {
    pub async fn handle_credit(&self, event: TransferCreditEvent) -> Result<(), Error> {
        let pool = self.router.get_shard(event.to_user);

        let mut tx = pool.begin().await?;

        // 幂等检查
        let existing = sqlx::query!(
            "SELECT id FROM processed_transfers WHERE transfer_id = $1",
            event.transfer_id
        )
        .fetch_optional(&mut *tx)
        .await?;

        if existing.is_some() {
            return Ok(());  // 已处理
        }

        // 入账
        sqlx::query!(
            "UPDATE users SET balance = balance + $1 WHERE id = $2",
            event.amount,
            event.to_user
        )
        .execute(&mut *tx)
        .await?;

        // 记录已处理
        sqlx::query!(
            "INSERT INTO processed_transfers (transfer_id, processed_at) VALUES ($1, NOW())",
            event.transfer_id
        )
        .execute(&mut *tx)
        .await?;

        tx.commit().await?;

        Ok(())
    }
}
```

## 问题三：分片键变更

当需要修改分片键时，数据需要从一个分片迁移到另一个分片。

### 双写迁移策略

```rust
pub struct ShardMigration {
    router: ShardRouter,
}

impl ShardMigration {
    /// 迁移用户到新分片
    pub async fn migrate_user(&self, user_id: i64, new_shard_key: i64) -> Result<(), Error> {
        let old_pool = self.router.get_shard(user_id);
        let new_pool = self.router.get_shard(new_shard_key);

        // 1. 在旧分片标记为迁移中
        sqlx::query!(
            "UPDATE users SET migration_status = 'migrating', new_shard_key = $1 WHERE id = $2",
            new_shard_key,
            user_id
        )
        .execute(old_pool)
        .await?;

        // 2. 复制数据到新分片
        let user_data = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", user_id)
            .fetch_one(old_pool)
            .await?;

        sqlx::query!(
            r#"
            INSERT INTO users (id, phone, balance, created_at)
            VALUES ($1, $2, $3, $4)
            ON CONFLICT (id) DO UPDATE SET
                phone = EXCLUDED.phone,
                balance = EXCLUDED.balance
            "#,
            user_data.id,
            user_data.phone,
            user_data.balance,
            user_data.created_at,
        )
        .execute(new_pool)
        .await?;

        // 3. 迁移用户的订单数据
        let orders = sqlx::query_as!(Order, "SELECT * FROM orders WHERE user_id = $1", user_id)
            .fetch_all(old_pool)
            .await?;

        for order in orders {
            sqlx::query!(
                r#"
                INSERT INTO orders (id, user_id, book_id, amount, status, created_at)
                VALUES ($1, $2, $3, $4, $5, $6)
                ON CONFLICT (id) DO NOTHING
                "#,
                order.id,
                order.user_id,
                order.book_id,
                order.amount,
                order.status,
                order.created_at,
            )
            .execute(new_pool)
            .await?;
        }

        // 4. 更新路由表
        sqlx::query!(
            "INSERT INTO shard_routing (user_id, shard_key) VALUES ($1, $2) ON CONFLICT (user_id) DO UPDATE SET shard_key = $2",
            user_id,
            new_shard_key
        )
        .execute(old_pool)  // 路由表在所有分片都有副本
        .await?;

        // 5. 标记迁移完成
        sqlx::query!(
            "UPDATE users SET migration_status = 'completed' WHERE id = $1",
            user_id
        )
        .execute(old_pool)
        .await?;

        // 6. 软删除旧数据（延迟清理）
        sqlx::query!(
            "UPDATE users SET deleted_at = NOW() WHERE id = $1",
            user_id
        )
        .execute(old_pool)
        .await?;

        Ok(())
    }
}

/// 支持迁移的路由器
pub struct MigrationAwareRouter {
    router: ShardRouter,
    routing_cache: Arc<Mutex<HashMap<i64, i64>>>,  // user_id -> shard_key 缓存
}

impl MigrationAwareRouter {
    pub async fn get_shard(&self, user_id: i64) -> &PgPool {
        // 先查缓存
        // Mutex::lock 只在 poisoned 时返回 Err，这里 panic 是合理的
        if let Some(&shard_key) = self.routing_cache.lock().expect("routing cache mutex poisoned").get(&user_id) {
            return self.router.get_shard(shard_key);
        }

        // 查路由表（任意分片都可以查）
        let any_pool = &self.router.all_shards()[0];
        let routing = sqlx::query!(
            "SELECT shard_key FROM shard_routing WHERE user_id = $1",
            user_id
        )
        .fetch_optional(any_pool.as_ref())
        .await
        .ok()
        .flatten();

        let shard_key = routing.map(|r| r.shard_key).unwrap_or(user_id);

        // 更新缓存
        self.routing_cache.lock().expect("routing cache mutex poisoned").insert(user_id, shard_key);

        self.router.get_shard(shard_key)
    }
}
```

## 分片策略选择

### 常见分片策略

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 取模（Hash） | 数据均匀分布 | 扩容困难 | 数据量可预估 |
| 范围（Range） | 扩容简单 | 可能热点 | 时间序列数据 |
| 一致性哈希 | 扩容影响小 | 实现复杂 | 需要动态扩容 |
| 目录（Directory） | 灵活 | 需要维护映射 | 复杂分片规则 |

### 一致性哈希实现

```rust
use std::collections::BTreeMap;
use std::hash::{Hash, Hasher};
use std::collections::hash_map::DefaultHasher;

pub struct ConsistentHash {
    ring: BTreeMap<u64, usize>,  // hash -> shard_index
    virtual_nodes: usize,
    shard_count: usize,
}

impl ConsistentHash {
    pub fn new(shard_count: usize, virtual_nodes: usize) -> Self {
        let mut ring = BTreeMap::new();

        for shard_index in 0..shard_count {
            for vn in 0..virtual_nodes {
                let key = format!("shard-{}-vn-{}", shard_index, vn);
                let hash = Self::hash(&key);
                ring.insert(hash, shard_index);
            }
        }

        Self { ring, virtual_nodes, shard_count }
    }

    pub fn get_shard(&self, key: &str) -> usize {
        let hash = Self::hash(key);

        // 找到第一个 >= hash 的节点
        self.ring
            .range(hash..)
            .next()
            .or_else(|| self.ring.iter().next())
            .map(|(_, &shard)| shard)
            .unwrap_or(0)
    }

    fn hash(key: &str) -> u64 {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        hasher.finish()
    }

    /// 添加新分片（只需迁移一小部分数据）
    pub fn add_shard(&mut self) -> Vec<(u64, usize, usize)> {
        let new_shard_index = self.shard_count;
        let mut migrations = Vec::new();

        for vn in 0..self.virtual_nodes {
            let key = format!("shard-{}-vn-{}", new_shard_index, vn);
            let new_hash = Self::hash(&key);

            // 找到需要迁移的范围
            if let Some((&next_hash, &old_shard)) = self.ring.range(new_hash..).next() {
                migrations.push((new_hash, old_shard, new_shard_index));
            }

            self.ring.insert(new_hash, new_shard_index);
        }

        self.shard_count += 1;
        migrations
    }
}
```

## 方案比较

| 问题 | 解决方案 | 一致性 | 性能 | 复杂度 |
|------|---------|--------|------|--------|
| 跨分片查询 | 并行查询 | 强一致 | 低 | 低 |
| 跨分片查询 | 全局索引 | 最终一致 | 中 | 中 |
| 跨分片查询 | CQRS | 最终一致 | 高 | 高 |
| 跨分片事务 | Saga | 最终一致 | 高 | 中 |
| 跨分片事务 | 2PC | 强一致 | 低 | 高 |
| 跨分片事务 | 消息驱动 | 最终一致 | 高 | 中 |
| 分片键变更 | 双写迁移 | 最终一致 | 中 | 高 |

## 决策树

```
需要数据分片
    │
    ├─ 选择分片键
    │   ├─ 查询主要按某字段？ → 按该字段分片
    │   ├─ 需要避免热点？ → 使用哈希分片
    │   └─ 需要范围查询？ → 使用范围分片
    │
    ├─ 跨分片查询方案
    │   ├─ 查询频率低？ → 并行查询所有分片
    │   ├─ 需要复杂查询？ → CQRS + 查询库
    │   └─ 需要实时性？ → 全局索引
    │
    └─ 跨分片事务方案
        ├─ 能避免跨分片？ → 调整分片策略
        ├─ 需要强一致？ → 2PC（慎用）
        └─ 可接受最终一致？ → Saga 或消息驱动
```

## 更优选择：透明分片

上面讲的都是**应用层分片**方案——需要在代码中实现分片路由、跨分片查询聚合、跨分片事务协调。这些方案适合已有系统的渐进式改造，或者需要精细控制的场景。

但如果是**新项目**，或者团队不想维护复杂的分片逻辑，有更简单的选择：

**PostgreSQL + Citus 扩展**

```sql
-- 安装 Citus 后，分片对应用完全透明
CREATE EXTENSION citus;

-- 将订单表声明为分布式表
SELECT create_distributed_table('orders', 'user_id');

-- 之后就像操作单库一样，Citus 自动处理分片路由、跨分片查询、跨分片事务
INSERT INTO orders (user_id, amount) VALUES (123, 99.99);
SELECT * FROM orders WHERE created_at > '2024-01-01';  -- 自动并行查询所有分片
```

Citus 的优势：
- **应用代码无需改造**：ORM、sqlx 等可以直接使用
- **自动跨分片查询**：Citus 协调器自动聚合结果
- **支持跨分片事务**：使用 2PC 自动保证原子性
- **在线扩容**：`citus_add_node` + `rebalance_table_shards` 自动重平衡

关于 Citus 的详细介绍，包括架构图、适用场景、与 TiDB/CockroachDB 的对比，参见：[《数据库演进史（四）分布式数据库》](/posts/db-evolution-4-distributed/)

## 小明的选择

小明最终的分片设计：

1. **分片策略**：按 user_id 哈希取模
2. **跨分片查询**：运营报表使用 CQRS，异步同步到查询库
3. **跨分片事务**：转账使用消息驱动的最终一致性
4. **扩容预案**：预留一致性哈希方案，需要时平滑扩容

分片上线后，数据库压力大大减轻，单用户查询速度提升了 10 倍。

## 下一步

数据分片解决了存储瓶颈，但小明的平台还在继续发展。随着功能越来越复杂，单体应用变得难以维护。

团队决定将系统拆分成多个独立服务：用户服务、库存服务、订单服务...

但拆分后，一个下单请求需要同时调用多个服务。如果某个服务调用失败了怎么办？

这就是下一章的主题：**跨服务事务一致性**。

---

*上一篇：[引入缓存：多存储一致性](/posts/consistency-in-practice-3-caching/)*
*下一篇：[服务拆分：跨服务事务一致性](/posts/consistency-in-practice-5-microservices/)*

*本系列：*
1. *[单机时代：ACID 的庇护](/posts/consistency-in-practice-1-single-node/)*
2. *[读写分离：副本一致性](/posts/consistency-in-practice-2-read-write-split/)*
3. *[引入缓存：多存储一致性](/posts/consistency-in-practice-3-caching/)*
4. *数据分片：跨分片事务一致性（本篇）*
5. *[服务拆分：跨服务事务一致性](/posts/consistency-in-practice-5-microservices/)*
6. *[消息驱动：最终一致性](/posts/consistency-in-practice-6-messaging/)*
7. *[总结：演进全景与选型指南](/posts/consistency-in-practice-7-summary/)*
