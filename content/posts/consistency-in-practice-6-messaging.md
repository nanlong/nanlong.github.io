---
title: "从单机到分布式：一致性实战（六）消息驱动：最终一致性"
date: 2025-12-11T10:50:00+08:00
draft: false
tags: ["一致性", "消息队列", "幂等", "Kafka", "Rust"]
categories: ["技术"]
description: "消息队列如何保证不丢不重？消费者如何保证幂等？本文详解消息投递语义、幂等设计、顺序保证等核心问题"
series: ["从单机到分布式：一致性实战"]
---

## 从这里开始

上一章，小明使用本地消息表解决了跨服务事务问题。随着系统演进，消息队列已经成为服务间通信的核心基础设施。

但新的问题接踵而至。一天早上，运营同学报告了一个奇怪的现象："有个用户的订单被重复扣款了三次！"

小明查看日志，发现消息确实发送了三次。原因是消息发送后，在收到 broker 确认之前网络超时了，发送端以为失败了就重试。

"那我加个去重逻辑？"小明想。但很快又发现另一个问题：有些消息根本没发出去就丢了。

**消息队列不是银弹，它本身也有一致性问题需要解决。**

## 消息投递的三种语义

消息队列的投递保证分为三种：

### At-Most-Once（最多一次）

消息发送后不等待确认，可能丢失，但不会重复。

```rust
// At-Most-Once：发了就不管
async fn send_at_most_once(producer: &KafkaProducer, message: &Message) {
    // 发送后立即返回，不等待确认
    producer.send_fire_and_forget(message).await;
    // 消息可能丢失，但代码继续执行
}
```

**适用场景**：日志收集、监控数据等允许丢失的场景

### At-Least-Once（至少一次）

消息发送后等待确认，失败则重试，保证不丢但可能重复。

```rust
// At-Least-Once：确认收到才算成功
async fn send_at_least_once(
    producer: &KafkaProducer,
    message: &Message,
) -> Result<(), Error> {
    loop {
        match producer.send(message).await {
            Ok(()) => return Ok(()),
            Err(e) if e.is_retriable() => {
                tracing::warn!("Send failed, retrying: {}", e);
                tokio::time::sleep(Duration::from_millis(100)).await;
                // 继续重试，可能导致重复
            }
            Err(e) => return Err(e),
        }
    }
}
```

**这是最常用的语义**，因为重复可以通过消费端幂等来解决，但丢失无法恢复。

### Exactly-Once（恰好一次）

消息既不丢失也不重复，是最理想的状态。

**但真正的 Exactly-Once 在分布式系统中几乎不可能实现**。所谓的 "Exactly-Once" 实际上是 "At-Least-Once + 幂等消费"。

```rust
// "Exactly-Once" = At-Least-Once 发送 + 幂等消费
async fn send_exactly_once(
    producer: &KafkaProducer,
    message: &Message,
) -> Result<(), Error> {
    // 消息必须有唯一 ID
    let message_id = message.id;

    // At-Least-Once 发送
    producer.send_with_retry(message).await?;

    Ok(())
}

async fn consume_exactly_once(
    pool: &PgPool,
    message: &Message,
) -> Result<(), Error> {
    // 幂等消费（见下文）
    todo!()
}
```

## 生产端：确保消息发出

### 问题：本地事务与消息发送的一致性

```rust
// 这段代码有问题
async fn create_order_wrong(pool: &PgPool, producer: &KafkaProducer, order: Order) -> Result<(), Error> {
    // 1. 写数据库
    sqlx::query!("INSERT INTO orders (id, user_id) VALUES ($1, $2)", order.id, order.user_id)
        .execute(pool)
        .await?;

    // 2. 发消息
    // 如果这里失败了，订单已创建但消息没发出
    producer.send(&OrderCreatedEvent::from(&order)).await?;

    Ok(())
}
```

### 解决方案一：本地消息表（Transactional Outbox）

我们在上一章已经详细介绍过，这里再强调关键点：

```rust
pub struct TransactionalOutbox {
    pool: PgPool,
}

impl TransactionalOutbox {
    /// 在同一个事务中写入业务数据和消息
    pub async fn execute_with_message<F, T>(
        &self,
        business_logic: F,
        message: OutboxMessage,
    ) -> Result<T, Error>
    where
        F: FnOnce(&mut sqlx::Transaction<'_, sqlx::Postgres>) -> BoxFuture<'_, Result<T, Error>>,
    {
        let mut tx = self.pool.begin().await?;

        // 1. 执行业务逻辑
        let result = business_logic(&mut tx).await?;

        // 2. 写入消息表
        sqlx::query!(
            r#"
            INSERT INTO outbox (id, topic, key, payload, created_at)
            VALUES ($1, $2, $3, $4, NOW())
            "#,
            message.id,
            message.topic,
            message.key,
            message.payload,
        )
        .execute(&mut *tx)
        .await?;

        // 3. 原子提交
        tx.commit().await?;

        Ok(result)
    }
}
```

### 解决方案二：事务消息（半消息）

某些消息队列（如 RocketMQ）支持事务消息：

```
1. 发送半消息（消费者不可见）
2. 执行本地事务
3. 根据本地事务结果提交或回滚半消息
```

用 Rust 模拟这个流程：

```rust
pub struct TransactionalMessageProducer {
    producer: KafkaProducer,
    pool: PgPool,
}

impl TransactionalMessageProducer {
    pub async fn send_transactional<F, T>(
        &self,
        message: Message,
        local_transaction: F,
    ) -> Result<T, Error>
    where
        F: FnOnce() -> BoxFuture<'static, Result<T, Error>>,
    {
        let tx_id = Uuid::new_v4();

        // 1. 发送半消息（标记为 PREPARING）
        let half_message = message.clone().with_header("tx_status", "PREPARING");
        self.producer.send(&half_message).await?;

        // 记录事务状态
        sqlx::query!(
            "INSERT INTO message_transactions (tx_id, message_id, status) VALUES ($1, $2, 'preparing')",
            tx_id,
            message.id
        )
        .execute(&self.pool)
        .await?;

        // 2. 执行本地事务
        let result = local_transaction().await;

        // 3. 根据结果提交或回滚
        match result {
            Ok(value) => {
                // 提交：发送确认消息
                let commit_message = message.with_header("tx_status", "COMMITTED");
                self.producer.send(&commit_message).await?;

                sqlx::query!(
                    "UPDATE message_transactions SET status = 'committed' WHERE tx_id = $1",
                    tx_id
                )
                .execute(&self.pool)
                .await?;

                Ok(value)
            }
            Err(e) => {
                // 回滚：发送取消消息
                let rollback_message = message.with_header("tx_status", "ROLLBACK");
                self.producer.send(&rollback_message).await?;

                sqlx::query!(
                    "UPDATE message_transactions SET status = 'rolled_back' WHERE tx_id = $1",
                    tx_id
                )
                .execute(&self.pool)
                .await?;

                Err(e)
            }
        }
    }
}
```

### 消息表轮询 vs CDC

**轮询（Polling）**：定时扫描消息表，发送待处理消息

```rust
pub struct OutboxPoller {
    pool: PgPool,
    producer: KafkaProducer,
}

impl OutboxPoller {
    pub async fn poll_and_send(&self) -> Result<u64, Error> {
        let messages = sqlx::query_as!(OutboxMessage,
            r#"
            SELECT * FROM outbox
            WHERE status = 'pending'
            ORDER BY created_at
            LIMIT 100
            FOR UPDATE SKIP LOCKED
            "#
        )
        .fetch_all(&self.pool)
        .await?;

        let mut sent_count = 0u64;

        for msg in messages {
            match self.producer.send(&msg.to_kafka_message()).await {
                Ok(()) => {
                    sqlx::query!(
                        "UPDATE outbox SET status = 'sent', sent_at = NOW() WHERE id = $1",
                        msg.id
                    )
                    .execute(&self.pool)
                    .await?;
                    sent_count += 1;
                }
                Err(e) => {
                    tracing::error!("Failed to send message {}: {}", msg.id, e);
                }
            }
        }

        Ok(sent_count)
    }
}
```

**CDC（Change Data Capture）**：监听数据库变更日志，实时捕获新消息

```rust
// 使用 Debezium + Kafka Connect 的架构
// 这里展示消费 CDC 事件并转发到目标 topic 的逻辑

pub struct CdcOutboxConsumer {
    source_consumer: KafkaConsumer,  // 消费 Debezium 的 CDC topic
    target_producer: KafkaProducer,   // 发送到业务 topic
}

impl CdcOutboxConsumer {
    pub async fn process_cdc_event(&self, event: CdcEvent) -> Result<(), Error> {
        // CDC 事件包含：before, after, operation (c/u/d)
        if event.operation != "c" {
            return Ok(());  // 只处理 INSERT
        }

        let outbox_record: OutboxRecord = serde_json::from_value(event.after)?;

        // 转发到目标 topic
        self.target_producer.send(&ProducerRecord {
            topic: &outbox_record.topic,
            key: outbox_record.key.as_deref(),
            payload: &outbox_record.payload,
        }).await?;

        Ok(())
    }
}
```

**对比**：

| 方案 | 延迟 | 复杂度 | 数据库压力 |
|------|------|--------|-----------|
| 轮询 | 秒级 | 低 | 高（频繁查询） |
| CDC | 毫秒级 | 高 | 低（只读 WAL） |

## 消费端：幂等设计

消息可能重复，消费端必须保证幂等性。

### 方案一：唯一约束

利用数据库唯一约束自动去重：

```rust
pub async fn consume_with_unique_constraint(
    pool: &PgPool,
    message: &OrderCreatedMessage,
) -> Result<(), Error> {
    // 利用 message_id 的唯一约束
    let result = sqlx::query!(
        r#"
        INSERT INTO processed_orders (message_id, order_id, user_id, amount)
        VALUES ($1, $2, $3, $4)
        ON CONFLICT (message_id) DO NOTHING
        "#,
        message.message_id,
        message.order_id,
        message.user_id,
        message.amount,
    )
    .execute(pool)
    .await?;

    if result.rows_affected() == 0 {
        tracing::info!("Message {} already processed, skipping", message.message_id);
    }

    Ok(())
}
```

### 方案二：消费记录表

专门维护一张消费记录表：

```rust
pub struct IdempotentConsumer {
    pool: PgPool,
}

impl IdempotentConsumer {
    pub async fn consume<F, T>(&self, message_id: &str, handler: F) -> Result<Option<T>, Error>
    where
        F: FnOnce() -> BoxFuture<'static, Result<T, Error>>,
    {
        let mut tx = self.pool.begin().await?;

        // 1. 检查是否已消费（加锁）
        let existing = sqlx::query!(
            "SELECT id FROM consumed_messages WHERE message_id = $1 FOR UPDATE",
            message_id
        )
        .fetch_optional(&mut *tx)
        .await?;

        if existing.is_some() {
            tracing::info!("Message {} already consumed", message_id);
            return Ok(None);
        }

        // 2. 执行业务逻辑
        let result = handler().await?;

        // 3. 记录已消费
        sqlx::query!(
            "INSERT INTO consumed_messages (message_id, consumed_at) VALUES ($1, NOW())",
            message_id
        )
        .execute(&mut *tx)
        .await?;

        tx.commit().await?;
        Ok(Some(result))
    }
}
```

### 方案三：业务幂等

设计业务逻辑本身就是幂等的：

```rust
// 场景：扣减库存
pub async fn deduct_stock_idempotent(
    pool: &PgPool,
    order_id: i64,
    book_id: i64,
    quantity: i32,
) -> Result<(), Error> {
    // 使用 order_id 作为幂等键
    // 同一个订单只扣一次库存
    let result = sqlx::query!(
        r#"
        UPDATE books
        SET stock = stock - $1
        WHERE id = $2
          AND stock >= $1
          AND NOT EXISTS (
              SELECT 1 FROM stock_deductions
              WHERE order_id = $3
          )
        "#,
        quantity,
        book_id,
        order_id,
    )
    .execute(pool)
    .await?;

    if result.rows_affected() > 0 {
        // 记录扣减记录
        sqlx::query!(
            "INSERT INTO stock_deductions (order_id, book_id, quantity) VALUES ($1, $2, $3)",
            order_id,
            book_id,
            quantity,
        )
        .execute(pool)
        .await?;
    }

    Ok(())
}
```

### 方案四：版本号/乐观锁

使用版本号确保操作的顺序性：

```rust
pub async fn update_with_version(
    pool: &PgPool,
    book_id: i64,
    new_price: i64,
    expected_version: i64,
) -> Result<bool, Error> {
    let result = sqlx::query!(
        r#"
        UPDATE books
        SET price = $1, version = version + 1
        WHERE id = $2 AND version = $3
        "#,
        new_price,
        book_id,
        expected_version,
    )
    .execute(pool)
    .await?;

    Ok(result.rows_affected() > 0)
}
```

## 消息顺序保证

某些场景需要保证消息的处理顺序，例如：

- 订单状态变更：created → paid → shipped
- 账户流水：先扣款后退款

### 问题分析

消息乱序的原因：
1. 生产端并发发送到不同分区
2. 消费端并发处理
3. 重试导致的顺序颠倒

### 解决方案一：单分区顺序

将同一业务实体的消息发送到同一个分区：

```rust
pub struct OrderedProducer {
    producer: KafkaProducer,
}

impl OrderedProducer {
    pub async fn send_ordered(&self, message: &OrderEvent) -> Result<(), Error> {
        // 使用 order_id 作为 partition key
        // 同一订单的所有消息会发送到同一分区
        let partition_key = message.order_id.to_string();

        self.producer.send(ProducerRecord {
            topic: "order-events",
            key: Some(&partition_key),
            partition: None,  // 让 Kafka 根据 key 选择分区
            payload: &serde_json::to_vec(message)?,
        }).await
    }
}
```

### 解决方案二：单线程消费

每个分区只用一个线程消费：

```rust
pub struct OrderedConsumer {
    consumer: KafkaConsumer,
    handler: Arc<dyn MessageHandler>,
}

impl OrderedConsumer {
    pub async fn start(&self) {
        // 获取分配的分区
        let partitions = self.consumer.assignment().await;

        // 每个分区一个处理协程，但串行处理消息
        let handles: Vec<_> = partitions
            .into_iter()
            .map(|partition| {
                let consumer = self.consumer.clone();
                let handler = self.handler.clone();

                tokio::spawn(async move {
                    loop {
                        // 从该分区拉取消息
                        let messages = consumer.poll_partition(partition, Duration::from_millis(100)).await;

                        // 串行处理，保证顺序
                        for msg in messages {
                            if let Err(e) = handler.handle(&msg).await {
                                tracing::error!("Failed to handle message: {}", e);
                                // 错误处理：重试或发送到死信队列
                            }
                        }
                    }
                })
            })
            .collect();

        futures::future::join_all(handles).await;
    }
}
```

### 解决方案三：序列号检查

在消费端检查序列号，乱序则等待或拒绝：

```rust
pub struct SequenceChecker {
    pool: PgPool,
}

impl SequenceChecker {
    /// 检查并更新序列号
    pub async fn check_and_update(
        &self,
        entity_id: &str,
        sequence: i64,
    ) -> Result<SequenceCheckResult, Error> {
        let mut tx = self.pool.begin().await?;

        let current = sqlx::query!(
            "SELECT last_sequence FROM entity_sequences WHERE entity_id = $1 FOR UPDATE",
            entity_id
        )
        .fetch_optional(&mut *tx)
        .await?;

        let expected = current.map(|r| r.last_sequence).unwrap_or(0) + 1;

        if sequence < expected {
            // 重复消息
            return Ok(SequenceCheckResult::Duplicate);
        }

        if sequence > expected {
            // 消息乱序，需要等待
            return Ok(SequenceCheckResult::OutOfOrder { expected });
        }

        // 序列号正确，更新
        sqlx::query!(
            r#"
            INSERT INTO entity_sequences (entity_id, last_sequence)
            VALUES ($1, $2)
            ON CONFLICT (entity_id) DO UPDATE SET last_sequence = $2
            "#,
            entity_id,
            sequence,
        )
        .execute(&mut *tx)
        .await?;

        tx.commit().await?;
        Ok(SequenceCheckResult::Ok)
    }
}

pub enum SequenceCheckResult {
    Ok,
    Duplicate,
    OutOfOrder { expected: i64 },
}
```

## 失败处理：死信队列

消息处理失败后，不能无限重试，需要有兜底机制。

```rust
pub struct ConsumerWithDlq {
    consumer: KafkaConsumer,
    dlq_producer: KafkaProducer,
    handler: Arc<dyn MessageHandler>,
    max_retries: u32,
}

impl ConsumerWithDlq {
    pub async fn consume(&self, message: Message) -> Result<(), Error> {
        let retry_count = message
            .headers
            .get("retry_count")
            .and_then(|v| v.parse().ok())
            .unwrap_or(0u32);

        match self.handler.handle(&message).await {
            Ok(()) => Ok(()),
            Err(e) => {
                if retry_count >= self.max_retries {
                    // 发送到死信队列
                    tracing::error!(
                        "Message {} exceeded max retries, sending to DLQ: {}",
                        message.id, e
                    );
                    self.send_to_dlq(&message, &e).await?;
                    Ok(())
                } else {
                    // 发送到重试队列
                    self.send_for_retry(&message, retry_count + 1).await?;
                    Ok(())
                }
            }
        }
    }

    async fn send_to_dlq(&self, message: &Message, error: &Error) -> Result<(), Error> {
        let dlq_message = Message {
            topic: format!("{}.dlq", message.topic),
            headers: {
                let mut h = message.headers.clone();
                h.insert("original_topic".into(), message.topic.clone());
                h.insert("error".into(), error.to_string());
                h.insert("failed_at".into(), chrono::Utc::now().to_rfc3339());
                h
            },
            ..message.clone()
        };

        self.dlq_producer.send(&dlq_message).await
    }

    async fn send_for_retry(&self, message: &Message, retry_count: u32) -> Result<(), Error> {
        let retry_message = Message {
            topic: format!("{}.retry", message.topic),
            headers: {
                let mut h = message.headers.clone();
                h.insert("retry_count".into(), retry_count.to_string());
                // 延迟重试（指数退避）
                let delay_ms = 1000 * (2_u64.pow(retry_count));
                h.insert("delay_ms".into(), delay_ms.to_string());
                h
            },
            ..message.clone()
        };

        self.dlq_producer.send(&retry_message).await
    }
}
```

### 死信队列处理流程

```rust
pub struct DlqProcessor {
    pool: PgPool,
}

impl DlqProcessor {
    /// 记录死信消息供人工处理
    pub async fn record_dead_letter(&self, message: &Message) -> Result<(), Error> {
        sqlx::query!(
            r#"
            INSERT INTO dead_letters (
                message_id, topic, payload, error, failed_at, status
            ) VALUES ($1, $2, $3, $4, $5, 'pending')
            "#,
            message.id,
            message.headers.get("original_topic"),
            serde_json::to_value(&message.payload)?,
            message.headers.get("error"),
            chrono::Utc::now(),
        )
        .execute(&self.pool)
        .await?;

        // 发送告警
        self.send_alert(&message).await?;

        Ok(())
    }

    /// 人工确认后重新投递
    pub async fn replay_dead_letter(&self, dead_letter_id: i64) -> Result<(), Error> {
        let record = sqlx::query!(
            "SELECT * FROM dead_letters WHERE id = $1 AND status = 'pending'",
            dead_letter_id
        )
        .fetch_one(&self.pool)
        .await?;

        // 重新发送到原 topic
        // ...

        // 更新状态
        sqlx::query!(
            "UPDATE dead_letters SET status = 'replayed', replayed_at = NOW() WHERE id = $1",
            dead_letter_id
        )
        .execute(&self.pool)
        .await?;

        Ok(())
    }
}
```

## 完整消费者示例

综合以上所有机制的完整实现：

```rust
pub struct RobustConsumer {
    consumer: KafkaConsumer,
    pool: PgPool,
    dlq_producer: KafkaProducer,
    max_retries: u32,
}

impl RobustConsumer {
    pub async fn start(&self) {
        loop {
            match self.consumer.poll(Duration::from_millis(100)).await {
                Ok(messages) => {
                    for message in messages {
                        if let Err(e) = self.process_message(message).await {
                            tracing::error!("Unrecoverable error: {}", e);
                        }
                    }
                }
                Err(e) => {
                    tracing::error!("Poll error: {}", e);
                    tokio::time::sleep(Duration::from_secs(1)).await;
                }
            }
        }
    }

    async fn process_message(&self, message: Message) -> Result<(), Error> {
        let message_id = &message.id;

        // 1. 幂等检查
        let already_processed = sqlx::query!(
            "SELECT id FROM consumed_messages WHERE message_id = $1",
            message_id
        )
        .fetch_optional(&self.pool)
        .await?
        .is_some();

        if already_processed {
            tracing::debug!("Message {} already processed, skipping", message_id);
            self.consumer.commit(&message).await?;
            return Ok(());
        }

        // 2. 尝试处理
        let retry_count = message
            .headers
            .get("retry_count")
            .and_then(|v| v.parse().ok())
            .unwrap_or(0u32);

        match self.handle_business_logic(&message).await {
            Ok(()) => {
                // 3. 记录已消费
                sqlx::query!(
                    "INSERT INTO consumed_messages (message_id, processed_at) VALUES ($1, NOW())",
                    message_id
                )
                .execute(&self.pool)
                .await?;

                // 4. 提交偏移量
                self.consumer.commit(&message).await?;
                Ok(())
            }
            Err(e) if retry_count < self.max_retries => {
                // 5. 发送到重试队列
                tracing::warn!("Message {} failed, will retry: {}", message_id, e);
                self.send_for_retry(&message, retry_count + 1).await?;
                self.consumer.commit(&message).await?;
                Ok(())
            }
            Err(e) => {
                // 6. 发送到死信队列
                tracing::error!("Message {} exceeded max retries: {}", message_id, e);
                self.send_to_dlq(&message, &e).await?;
                self.consumer.commit(&message).await?;
                Ok(())
            }
        }
    }

    async fn handle_business_logic(&self, message: &Message) -> Result<(), Error> {
        // 具体的业务逻辑
        todo!()
    }
}
```

## 方案比较

| 问题 | 解决方案 | 复杂度 | 可靠性 |
|------|---------|--------|--------|
| 生产端一致性 | 本地消息表 | 中 | 高 |
| 生产端一致性 | 事务消息 | 高 | 高 |
| 消费端幂等 | 唯一约束 | 低 | 中 |
| 消费端幂等 | 消费记录表 | 中 | 高 |
| 消费端幂等 | 业务幂等设计 | 高 | 高 |
| 顺序保证 | 单分区 | 低 | 中 |
| 顺序保证 | 单线程消费 | 低 | 中 |
| 顺序保证 | 序列号检查 | 高 | 高 |
| 失败处理 | 重试队列 + DLQ | 中 | 高 |

## 决策树

```
需要消息驱动
    │
    ├─ 生产端如何保证消息发出？
    │   ├─ 有消息队列事务消息支持？ → 使用事务消息
    │   └─ 没有？ → 本地消息表 + 轮询/CDC
    │
    ├─ 消费端如何保证幂等？
    │   ├─ 能改业务逻辑？ → 业务幂等设计
    │   ├─ 有唯一键？ → 唯一约束去重
    │   └─ 通用方案 → 消费记录表
    │
    ├─ 需要顺序保证吗？
    │   ├─ 是 → 单分区 + 单线程消费
    │   └─ 否 → 并发消费提高吞吐
    │
    └─ 失败如何处理？
        └─ 重试队列 + 死信队列 + 人工介入
```

## 小明的实践

小明梳理了消息系统的完整设计：

1. **生产端**：本地消息表 + 后台轮询，确保消息必达
2. **消费端**：消费记录表 + 唯一约束双重保证幂等
3. **顺序**：同一订单的消息使用 order_id 作为 partition key
4. **失败处理**：最多重试 3 次，超过后进入死信队列，发送钉钉告警

系统上线后运行稳定，再也没有出现重复扣款的问题。

## 下一步

至此，小明的系统已经从单机演进到了完整的分布式架构：

1. **单机** → 读写分离 → 缓存 → 分库分表 → 服务拆分 → **消息驱动**

每一步都遇到了新的一致性问题，也学会了对应的解决方案。

下一章，我们将回顾整个演进过程，总结各种一致性问题的解决方案，给出一份完整的**选型指南**。

---

*上一篇：[服务拆分：跨服务事务一致性](/posts/consistency-in-practice-5-microservices/)*
*下一篇：[总结：演进全景与选型指南](/posts/consistency-in-practice-7-summary/)*

*本系列：*
1. *[单机时代：ACID 的庇护](/posts/consistency-in-practice-1-single-node/)*
2. *[读写分离：副本一致性](/posts/consistency-in-practice-2-read-write-split/)*
3. *[引入缓存：多存储一致性](/posts/consistency-in-practice-3-caching/)*
4. *[数据分片：跨分片事务一致性](/posts/consistency-in-practice-4-sharding/)*
5. *[服务拆分：跨服务事务一致性](/posts/consistency-in-practice-5-microservices/)*
6. *消息驱动：最终一致性（本篇）*
7. *[总结：演进全景与选型指南](/posts/consistency-in-practice-7-summary/)*
