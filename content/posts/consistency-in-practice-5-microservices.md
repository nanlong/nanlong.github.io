---
title: "从单机到分布式：一致性实战（五）服务拆分：跨服务事务一致性"
date: 2025-12-11T10:40:00+08:00
draft: false
tags: ["一致性", "分布式事务", "TCC", "Saga", "微服务", "Rust"]
categories: ["技术"]
description: "当一个业务操作需要调用多个服务时，如何保证要么全部成功，要么全部失败？本文详解 TCC、Saga、本地消息表等方案的实现与权衡"
series: ["从单机到分布式：一致性实战"]
---

## 从这里开始

分库分表解决了数据量问题后，小明的二手书平台继续快速发展。团队也从最初的 3 人扩展到了 20 人。

单体应用的问题开始显现：
- 代码库越来越大，改一个小功能都要担心影响其他模块
- 每次发布都是全量部署，风险高
- 不同模块的技术栈被绑定在一起，无法独立演进

技术负责人建议："是时候拆分服务了。"

团队经过讨论，决定把系统拆分成多个服务：

- **用户服务**：管理用户账户和余额
- **库存服务**：管理书籍库存
- **订单服务**：处理订单逻辑

拆分后，架构清晰了很多。但第一个需求就让小明犯了难：

> 用户下单时，需要：1）从用户账户扣款，2）扣减库存，3）创建订单

在单体应用时代，这三个操作可以放在同一个数据库事务里：

```rust
// 单体时代：一个事务搞定
async fn place_order(pool: &PgPool, order: OrderRequest) -> Result<Order, Error> {
    let mut tx = pool.begin().await?;

    // 1. 扣款
    sqlx::query!("UPDATE users SET balance = balance - $1 WHERE id = $2", order.amount, order.user_id)
        .execute(&mut *tx).await?;

    // 2. 扣库存
    sqlx::query!("UPDATE books SET stock = stock - 1 WHERE id = $1", order.book_id)
        .execute(&mut *tx).await?;

    // 3. 创建订单
    let order = sqlx::query_as!(Order,
        "INSERT INTO orders (user_id, book_id, amount) VALUES ($1, $2, $3) RETURNING *",
        order.user_id, order.book_id, order.amount
    ).fetch_one(&mut *tx).await?;

    tx.commit().await?;
    Ok(order)
}
```

但现在，用户余额在用户服务的数据库，库存在库存服务的数据库，订单在订单服务的数据库。**三个操作跨越三个独立的数据库，无法在同一个事务中完成。**

如果扣款成功了，但扣库存失败了，怎么办？钱扣了，货没发，用户肯定要投诉。

这就是**跨服务事务一致性**问题。

## 问题的本质

分布式事务难在哪里？让我们看一个简化的场景：

```
订单服务                    库存服务
    │                          │
    │──── 1. 扣库存请求 ──────>│
    │                          │ (扣库存成功)
    │<─── 2. 扣库存响应 ───────│
    │                          │
    │ (创建订单...)            │
    │ (本地事务失败!)          │
    │                          │
    │ 现在怎么办？             │
    │ 库存已经扣了，但订单没创建│
```

问题的根源是：

1. **没有全局事务协调者**：每个服务只能控制自己的本地事务
2. **网络不可靠**：调用可能超时、失败、或成功但响应丢失
3. **部分失败**：一个操作成功，另一个失败

传统解决方案是**两阶段提交（2PC）**，但它有致命缺陷：
- 同步阻塞，性能差
- 单点故障（协调者挂了整个事务卡住）
- 极端情况下仍可能数据不一致

因此，现代分布式系统更倾向于使用**柔性事务**方案，接受最终一致性。

## 方案一：TCC（Try-Confirm-Cancel）

TCC 的核心思想是把每个操作拆分成三个阶段：

- **Try**：预留资源（冻结库存、冻结余额）
- **Confirm**：确认提交（真正扣减）
- **Cancel**：取消释放（解冻）

### 时序图

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│  订单服务 │     │ 用户服务 │     │ 库存服务 │     │ 协调者   │
└────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘
     │               │               │               │
     │ ──────────────────── 开始事务 ─────────────────> │
     │               │               │               │
     │ <──────────────────── 事务ID ──────────────────- │
     │               │               │               │
     │ ── Try 冻结余额 ─>│               │               │
     │ <─── 冻结成功 ────│               │               │
     │               │               │               │
     │ ─────── Try 冻结库存 ──────────>│               │
     │ <────── 冻结成功 ──────────────-│               │
     │               │               │               │
     │ ─────────────────── 所有 Try 成功 ─────────────> │
     │               │               │               │
     │               │ <── Confirm ────────────────── │
     │               │               │ <── Confirm ── │
     │               │               │               │
     │ <────────────────── 事务完成 ─────────────────── │
```

### 实现

首先定义 TCC 接口：

```rust
use async_trait::async_trait;
use uuid::Uuid;

/// TCC 参与者接口
#[async_trait]
pub trait TccParticipant: Send + Sync {
    type TryRequest;
    type TryResult;

    /// Try 阶段：预留资源
    async fn try_action(&self, tx_id: Uuid, req: Self::TryRequest) -> Result<Self::TryResult, Error>;

    /// Confirm 阶段：确认提交
    async fn confirm(&self, tx_id: Uuid) -> Result<(), Error>;

    /// Cancel 阶段：取消回滚
    async fn cancel(&self, tx_id: Uuid) -> Result<(), Error>;
}
```

实现用户服务的 TCC：

```rust
pub struct UserTccService {
    pool: PgPool,
}

#[derive(Debug, Clone)]
pub struct FreezeBalanceRequest {
    pub user_id: i64,
    pub amount: i64,
}

#[async_trait]
impl TccParticipant for UserTccService {
    type TryRequest = FreezeBalanceRequest;
    type TryResult = ();

    async fn try_action(&self, tx_id: Uuid, req: Self::TryRequest) -> Result<(), Error> {
        // 检查余额并冻结
        let result = sqlx::query!(
            r#"
            UPDATE users
            SET balance = balance - $1,
                frozen_balance = frozen_balance + $1
            WHERE id = $2 AND balance >= $1
            "#,
            req.amount,
            req.user_id
        )
        .execute(&self.pool)
        .await?;

        if result.rows_affected() == 0 {
            return Err(Error::InsufficientBalance);
        }

        // 记录 TCC 事务状态
        sqlx::query!(
            "INSERT INTO tcc_transactions (tx_id, user_id, amount, status) VALUES ($1, $2, $3, 'tried')",
            tx_id,
            req.user_id,
            req.amount
        )
        .execute(&self.pool)
        .await?;

        Ok(())
    }

    async fn confirm(&self, tx_id: Uuid) -> Result<(), Error> {
        let mut tx = self.pool.begin().await?;

        // 获取事务记录
        let record = sqlx::query!(
            "SELECT user_id, amount FROM tcc_transactions WHERE tx_id = $1 AND status = 'tried' FOR UPDATE",
            tx_id
        )
        .fetch_optional(&mut *tx)
        .await?
        .ok_or(Error::TransactionNotFound)?;

        // 扣减冻结金额（真正扣款）
        sqlx::query!(
            "UPDATE users SET frozen_balance = frozen_balance - $1 WHERE id = $2",
            record.amount,
            record.user_id
        )
        .execute(&mut *tx)
        .await?;

        // 更新事务状态
        sqlx::query!(
            "UPDATE tcc_transactions SET status = 'confirmed' WHERE tx_id = $1",
            tx_id
        )
        .execute(&mut *tx)
        .await?;

        tx.commit().await?;
        Ok(())
    }

    async fn cancel(&self, tx_id: Uuid) -> Result<(), Error> {
        let mut tx = self.pool.begin().await?;

        // 获取事务记录
        let record = sqlx::query!(
            "SELECT user_id, amount FROM tcc_transactions WHERE tx_id = $1 AND status = 'tried' FOR UPDATE",
            tx_id
        )
        .fetch_optional(&mut *tx)
        .await?;

        // 幂等性：如果记录不存在或已处理，直接返回成功
        let record = match record {
            Some(r) => r,
            None => return Ok(()),
        };

        // 解冻余额
        sqlx::query!(
            "UPDATE users SET balance = balance + $1, frozen_balance = frozen_balance - $1 WHERE id = $2",
            record.amount,
            record.user_id
        )
        .execute(&mut *tx)
        .await?;

        // 更新事务状态
        sqlx::query!(
            "UPDATE tcc_transactions SET status = 'cancelled' WHERE tx_id = $1",
            tx_id
        )
        .execute(&mut *tx)
        .await?;

        tx.commit().await?;
        Ok(())
    }
}
```

TCC 协调者：

```rust
pub struct TccCoordinator {
    user_service: Arc<dyn TccParticipant<TryRequest = FreezeBalanceRequest, TryResult = ()>>,
    inventory_service: Arc<dyn TccParticipant<TryRequest = FreezeStockRequest, TryResult = ()>>,
}

impl TccCoordinator {
    pub async fn place_order(&self, order: OrderRequest) -> Result<Uuid, Error> {
        let tx_id = Uuid::new_v4();
        let mut tried_participants: Vec<&str> = Vec::new();

        // Try 阶段
        let try_result = async {
            // Try 冻结余额
            self.user_service.try_action(tx_id, FreezeBalanceRequest {
                user_id: order.user_id,
                amount: order.amount,
            }).await?;
            tried_participants.push("user");

            // Try 冻结库存
            self.inventory_service.try_action(tx_id, FreezeStockRequest {
                book_id: order.book_id,
                quantity: 1,
            }).await?;
            tried_participants.push("inventory");

            Ok::<_, Error>(())
        }.await;

        match try_result {
            Ok(()) => {
                // 所有 Try 成功，执行 Confirm
                self.confirm_all(tx_id).await?;
                Ok(tx_id)
            }
            Err(e) => {
                // 有 Try 失败，执行 Cancel
                self.cancel_all(tx_id, &tried_participants).await;
                Err(e)
            }
        }
    }

    async fn confirm_all(&self, tx_id: Uuid) -> Result<(), Error> {
        // 并行执行 Confirm（失败需要重试）
        let (user_result, inventory_result) = tokio::join!(
            self.user_service.confirm(tx_id),
            self.inventory_service.confirm(tx_id),
        );

        user_result?;
        inventory_result?;
        Ok(())
    }

    async fn cancel_all(&self, tx_id: Uuid, participants: &[&str]) {
        // Cancel 必须保证最终成功，失败要重试
        for participant in participants {
            let result = match *participant {
                "user" => self.user_service.cancel(tx_id).await,
                "inventory" => self.inventory_service.cancel(tx_id).await,
                _ => Ok(()),
            };

            if let Err(e) = result {
                tracing::error!("Cancel failed for {}, will retry: {}", participant, e);
                // 发送到重试队列
            }
        }
    }
}
```

### TCC 的关键点

1. **空回滚**：Cancel 被调用时，Try 可能根本没执行过（网络超时），需要处理
2. **幂等性**：Confirm 和 Cancel 可能被多次调用，必须幂等
3. **悬挂事务**：Try 在 Cancel 之后到达，需要拒绝

```rust
// 处理空回滚和悬挂事务
async fn try_action(&self, tx_id: Uuid, req: Self::TryRequest) -> Result<(), Error> {
    // 检查是否已经 Cancel 过（防悬挂）
    let existing = sqlx::query!(
        "SELECT status FROM tcc_transactions WHERE tx_id = $1",
        tx_id
    )
    .fetch_optional(&self.pool)
    .await?;

    if let Some(record) = existing {
        if record.status == "cancelled" {
            // 已经取消，拒绝 Try（防悬挂）
            return Err(Error::TransactionAlreadyCancelled);
        }
    }

    // 正常执行 Try...
}

async fn cancel(&self, tx_id: Uuid) -> Result<(), Error> {
    let existing = sqlx::query!(
        "SELECT status FROM tcc_transactions WHERE tx_id = $1",
        tx_id
    )
    .fetch_optional(&self.pool)
    .await?;

    match existing {
        None => {
            // 空回滚：Try 没执行过，记录一个已取消状态（防悬挂）
            sqlx::query!(
                "INSERT INTO tcc_transactions (tx_id, status) VALUES ($1, 'cancelled')",
                tx_id
            )
            .execute(&self.pool)
            .await?;
            Ok(())
        }
        Some(record) if record.status == "confirmed" || record.status == "cancelled" => {
            // 幂等：已确认或已取消，直接返回
            Ok(())
        }
        Some(_) => {
            // 正常取消流程...
        }
    }
}
```

### TCC 的局限性

- **侵入性强**：每个服务都要实现 Try/Confirm/Cancel 三个接口
- **开发成本高**：需要设计"冻结"状态，改造数据模型
- **锁定资源**：Try 阶段冻结的资源在 Confirm 前一直被占用

## 方案二：Saga

Saga 的核心思想是：**把长事务拆分成多个本地事务，每个本地事务都有对应的补偿操作**。

与 TCC 不同，Saga 没有 Try 阶段，每个步骤直接执行真实操作，失败时执行补偿。

### 两种执行方式

**编排式（Choreography）**：每个服务监听事件，自行决定下一步

```
订单服务发起 ─────> 用户服务监听并扣款
                           │
                           ▼ 发送"扣款成功"事件
                    库存服务监听并扣库存
                           │
                           ▼ 发送"扣库存成功"事件
                    订单服务监听并创建订单
```

**协调式（Orchestration）**：由中央协调者控制流程

```
协调者 ──> 调用用户服务扣款
协调者 <── 扣款成功
协调者 ──> 调用库存服务扣库存
协调者 <── 扣库存成功
协调者 ──> 调用订单服务创建订单
协调者 <── 创建成功
```

### 协调式 Saga 实现

```rust
use serde::{Deserialize, Serialize};

/// Saga 步骤定义
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SagaStep {
    pub name: String,
    pub status: StepStatus,
}

#[derive(Debug, Clone, Copy, PartialEq, Serialize, Deserialize)]
pub enum StepStatus {
    Pending,
    Completed,
    Failed,
    Compensated,
}

/// Saga 执行状态
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SagaState {
    pub saga_id: Uuid,
    pub steps: Vec<SagaStep>,
    pub current_step: usize,
    pub status: SagaStatus,
    pub data: serde_json::Value,  // 存储中间数据
}

#[derive(Debug, Clone, Copy, PartialEq, Serialize, Deserialize)]
pub enum SagaStatus {
    Running,
    Completed,
    Compensating,
    Failed,
}

/// Saga 协调者
pub struct OrderSagaOrchestrator {
    pool: PgPool,
    user_client: UserServiceClient,
    inventory_client: InventoryServiceClient,
    order_client: OrderServiceClient,
}

impl OrderSagaOrchestrator {
    pub async fn execute(&self, order: OrderRequest) -> Result<Uuid, Error> {
        let saga_id = Uuid::new_v4();

        // 初始化 Saga 状态
        let mut state = SagaState {
            saga_id,
            steps: vec![
                SagaStep { name: "deduct_balance".into(), status: StepStatus::Pending },
                SagaStep { name: "deduct_stock".into(), status: StepStatus::Pending },
                SagaStep { name: "create_order".into(), status: StepStatus::Pending },
            ],
            current_step: 0,
            status: SagaStatus::Running,
            data: serde_json::to_value(&order)?,
        };

        self.save_state(&state).await?;

        // 执行 Saga
        match self.run_forward(&mut state).await {
            Ok(()) => {
                state.status = SagaStatus::Completed;
                self.save_state(&state).await?;
                Ok(saga_id)
            }
            Err(e) => {
                tracing::error!("Saga {} failed at step {}: {}", saga_id, state.current_step, e);
                self.run_compensation(&mut state).await;
                Err(e)
            }
        }
    }

    /// 正向执行
    async fn run_forward(&self, state: &mut SagaState) -> Result<(), Error> {
        let order: OrderRequest = serde_json::from_value(state.data.clone())?;

        while state.current_step < state.steps.len() {
            let step_name = &state.steps[state.current_step].name;

            let result = match step_name.as_str() {
                "deduct_balance" => {
                    self.user_client.deduct_balance(
                        state.saga_id,
                        order.user_id,
                        order.amount,
                    ).await
                }
                "deduct_stock" => {
                    self.inventory_client.deduct_stock(
                        state.saga_id,
                        order.book_id,
                        1,
                    ).await
                }
                "create_order" => {
                    self.order_client.create_order(
                        state.saga_id,
                        order.clone(),
                    ).await
                }
                _ => Err(Error::UnknownStep(step_name.clone())),
            };

            match result {
                Ok(()) => {
                    state.steps[state.current_step].status = StepStatus::Completed;
                    state.current_step += 1;
                    self.save_state(state).await?;
                }
                Err(e) => {
                    state.steps[state.current_step].status = StepStatus::Failed;
                    state.status = SagaStatus::Compensating;
                    self.save_state(state).await?;
                    return Err(e);
                }
            }
        }

        Ok(())
    }

    /// 反向补偿
    async fn run_compensation(&self, state: &mut SagaState) {
        // 从当前步骤往回补偿
        for i in (0..state.current_step).rev() {
            if state.steps[i].status != StepStatus::Completed {
                continue;
            }

            let step_name = &state.steps[i].name;

            // 解析订单数据，如果失败则跳过该步骤（理论上不应发生）
            let order: OrderRequest = match serde_json::from_value(state.data.clone()) {
                Ok(o) => o,
                Err(e) => {
                    tracing::error!("Failed to parse order data for saga {}: {}", state.saga_id, e);
                    continue;
                }
            };

            let result = match step_name.as_str() {
                "deduct_balance" => {
                    self.user_client.refund_balance(
                        state.saga_id,
                        order.user_id,
                        order.amount,
                    ).await
                }
                "deduct_stock" => {
                    self.inventory_client.restore_stock(
                        state.saga_id,
                        order.book_id,
                        1,
                    ).await
                }
                "create_order" => {
                    self.order_client.cancel_order(state.saga_id).await
                }
                _ => Ok(()),
            };

            match result {
                Ok(()) => {
                    state.steps[i].status = StepStatus::Compensated;
                    self.save_state(state).await.ok();
                }
                Err(e) => {
                    // 补偿失败，记录日志，等待人工介入或重试
                    tracing::error!(
                        "Compensation failed for saga {} step {}: {}",
                        state.saga_id, step_name, e
                    );
                }
            }
        }

        state.status = SagaStatus::Failed;
        self.save_state(state).await.ok();
    }

    async fn save_state(&self, state: &SagaState) -> Result<(), Error> {
        sqlx::query!(
            r#"
            INSERT INTO saga_states (saga_id, state_json, updated_at)
            VALUES ($1, $2, NOW())
            ON CONFLICT (saga_id) DO UPDATE SET
                state_json = EXCLUDED.state_json,
                updated_at = NOW()
            "#,
            state.saga_id,
            serde_json::to_value(state)?,
        )
        .execute(&self.pool)
        .await?;

        Ok(())
    }
}
```

### 服务端实现补偿操作

```rust
impl UserService {
    /// 扣款（正向操作）
    pub async fn deduct_balance(
        &self,
        saga_id: Uuid,
        user_id: i64,
        amount: i64,
    ) -> Result<(), Error> {
        let mut tx = self.pool.begin().await?;

        // 幂等检查
        let existing = sqlx::query!(
            "SELECT id FROM saga_operations WHERE saga_id = $1 AND operation = 'deduct'",
            saga_id
        )
        .fetch_optional(&mut *tx)
        .await?;

        if existing.is_some() {
            return Ok(());  // 已执行过
        }

        // 扣款
        let result = sqlx::query!(
            "UPDATE users SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
            amount,
            user_id
        )
        .execute(&mut *tx)
        .await?;

        if result.rows_affected() == 0 {
            return Err(Error::InsufficientBalance);
        }

        // 记录操作（用于幂等和补偿）
        sqlx::query!(
            "INSERT INTO saga_operations (saga_id, operation, user_id, amount) VALUES ($1, 'deduct', $2, $3)",
            saga_id,
            user_id,
            amount
        )
        .execute(&mut *tx)
        .await?;

        tx.commit().await?;
        Ok(())
    }

    /// 退款（补偿操作）
    pub async fn refund_balance(
        &self,
        saga_id: Uuid,
        user_id: i64,
        amount: i64,
    ) -> Result<(), Error> {
        let mut tx = self.pool.begin().await?;

        // 幂等检查
        let existing = sqlx::query!(
            "SELECT id FROM saga_operations WHERE saga_id = $1 AND operation = 'refund'",
            saga_id
        )
        .fetch_optional(&mut *tx)
        .await?;

        if existing.is_some() {
            return Ok(());  // 已退款
        }

        // 检查是否有对应的扣款记录
        let deduct_record = sqlx::query!(
            "SELECT id FROM saga_operations WHERE saga_id = $1 AND operation = 'deduct'",
            saga_id
        )
        .fetch_optional(&mut *tx)
        .await?;

        if deduct_record.is_none() {
            // 空补偿：没扣过款，直接返回成功
            return Ok(());
        }

        // 退款
        sqlx::query!(
            "UPDATE users SET balance = balance + $1 WHERE id = $2",
            amount,
            user_id
        )
        .execute(&mut *tx)
        .await?;

        // 记录退款操作
        sqlx::query!(
            "INSERT INTO saga_operations (saga_id, operation, user_id, amount) VALUES ($1, 'refund', $2, $3)",
            saga_id,
            user_id,
            amount
        )
        .execute(&mut *tx)
        .await?;

        tx.commit().await?;
        Ok(())
    }
}
```

### Saga vs TCC

| 对比项 | TCC | Saga |
|-------|-----|------|
| 资源锁定 | Try 阶段锁定 | 不锁定，直接执行 |
| 隔离性 | 较好（资源被冻结） | 较差（中间状态可见） |
| 开发成本 | 高（三个接口） | 中（正向+补偿） |
| 适用场景 | 短事务、需要隔离性 | 长事务、可接受最终一致 |

## 方案三：本地消息表

本地消息表（Transactional Outbox）是一种可靠的异步方案：**在本地事务中同时写入业务数据和消息记录，然后异步发送消息**。

### 架构

```
订单服务                                          库存服务
   │                                                │
   │ 1. 开启本地事务                                  │
   │ 2. 创建订单                                     │
   │ 3. 写入消息表                                   │
   │ 4. 提交事务                                     │
   │                                                │
   │ 5. 后台任务扫描消息表                            │
   │ 6. 发送消息 ────────────────────────────────────>│
   │                                                │
   │                                    7. 扣减库存   │
   │                                    8. 返回成功   │
   │                                                │
   │ 9. 标记消息已发送                                │
```

### 实现

```rust
/// 消息表
#[derive(Debug, Clone, sqlx::FromRow)]
pub struct OutboxMessage {
    pub id: i64,
    pub aggregate_type: String,      // 聚合类型，如 "order"
    pub aggregate_id: String,        // 聚合 ID，如订单号
    pub message_type: String,        // 消息类型，如 "OrderCreated"
    pub payload: serde_json::Value,  // 消息内容
    pub status: MessageStatus,
    pub retry_count: i32,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub processed_at: Option<chrono::DateTime<chrono::Utc>>,
}

#[derive(Debug, Clone, Copy, PartialEq, sqlx::Type)]
#[sqlx(type_name = "message_status", rename_all = "snake_case")]
pub enum MessageStatus {
    Pending,
    Sent,
    Failed,
}

/// 订单服务 - 创建订单并写入消息
impl OrderService {
    pub async fn create_order(&self, order: OrderRequest) -> Result<Order, Error> {
        let mut tx = self.pool.begin().await?;

        // 1. 创建订单
        let order = sqlx::query_as!(Order,
            r#"
            INSERT INTO orders (user_id, book_id, amount, status)
            VALUES ($1, $2, $3, 'pending')
            RETURNING *
            "#,
            order.user_id,
            order.book_id,
            order.amount,
        )
        .fetch_one(&mut *tx)
        .await?;

        // 2. 写入消息表（在同一个事务中）
        let message_payload = serde_json::json!({
            "order_id": order.id,
            "book_id": order.book_id,
            "quantity": 1,
        });

        sqlx::query!(
            r#"
            INSERT INTO outbox_messages
            (aggregate_type, aggregate_id, message_type, payload, status)
            VALUES ('order', $1, 'DeductStock', $2, 'pending')
            "#,
            order.id.to_string(),
            message_payload,
        )
        .execute(&mut *tx)
        .await?;

        // 3. 提交事务 - 订单和消息要么同时成功，要么同时失败
        tx.commit().await?;

        Ok(order)
    }
}

/// 消息发送器 - 后台任务
pub struct OutboxProcessor {
    pool: PgPool,
    message_sender: MessageSender,
}

impl OutboxProcessor {
    pub async fn start(&self) {
        let mut interval = tokio::time::interval(Duration::from_secs(1));

        loop {
            interval.tick().await;

            if let Err(e) = self.process_pending_messages().await {
                tracing::error!("Failed to process outbox messages: {}", e);
            }
        }
    }

    async fn process_pending_messages(&self) -> Result<(), Error> {
        // 获取待发送的消息（加锁防止并发处理）
        let messages = sqlx::query_as!(OutboxMessage,
            r#"
            SELECT id, aggregate_type, aggregate_id, message_type,
                   payload, status as "status: MessageStatus",
                   retry_count, created_at, processed_at
            FROM outbox_messages
            WHERE status = 'pending' AND retry_count < 5
            ORDER BY created_at
            LIMIT 100
            FOR UPDATE SKIP LOCKED
            "#
        )
        .fetch_all(&self.pool)
        .await?;

        for message in messages {
            match self.send_message(&message).await {
                Ok(()) => {
                    // 标记为已发送
                    sqlx::query!(
                        "UPDATE outbox_messages SET status = 'sent', processed_at = NOW() WHERE id = $1",
                        message.id
                    )
                    .execute(&self.pool)
                    .await?;
                }
                Err(e) => {
                    // 增加重试计数
                    tracing::error!("Failed to send message {}: {}", message.id, e);
                    sqlx::query!(
                        "UPDATE outbox_messages SET retry_count = retry_count + 1 WHERE id = $1",
                        message.id
                    )
                    .execute(&self.pool)
                    .await?;
                }
            }
        }

        Ok(())
    }

    async fn send_message(&self, message: &OutboxMessage) -> Result<(), Error> {
        self.message_sender.send(&message.message_type, &message.payload).await
    }
}
```

### 消费端幂等处理

```rust
impl InventoryService {
    pub async fn handle_deduct_stock(&self, message: &DeductStockMessage) -> Result<(), Error> {
        let mut tx = self.pool.begin().await?;

        // 幂等检查：是否已处理过这条消息
        let existing = sqlx::query!(
            "SELECT id FROM processed_messages WHERE message_id = $1",
            message.message_id
        )
        .fetch_optional(&mut *tx)
        .await?;

        if existing.is_some() {
            tracing::info!("Message {} already processed, skipping", message.message_id);
            return Ok(());
        }

        // 扣减库存
        let result = sqlx::query!(
            "UPDATE books SET stock = stock - $1 WHERE id = $2 AND stock >= $1",
            message.quantity,
            message.book_id
        )
        .execute(&mut *tx)
        .await?;

        if result.rows_affected() == 0 {
            // 库存不足，发送补偿消息
            self.send_stock_insufficient_message(message).await?;
            return Ok(());
        }

        // 记录已处理的消息
        sqlx::query!(
            "INSERT INTO processed_messages (message_id, processed_at) VALUES ($1, NOW())",
            message.message_id
        )
        .execute(&mut *tx)
        .await?;

        tx.commit().await?;
        Ok(())
    }
}
```

### 优点与局限

**优点**：
- 消息发送的可靠性有保障
- 业务解耦，异步处理
- 实现相对简单

**局限**：
- 有延迟，不是实时的
- 需要消费端保证幂等
- 消息顺序难以保证

## 方案比较

| 方案 | 一致性 | 侵入性 | 性能 | 复杂度 | 适用场景 |
|------|--------|--------|------|--------|---------|
| TCC | 强 | 高 | 中 | 高 | 短事务、金融场景 |
| Saga | 最终一致 | 中 | 高 | 中 | 长事务、可接受最终一致 |
| 本地消息表 | 最终一致 | 低 | 高 | 低 | 异步场景、跨系统集成 |

## 决策树

```
需要跨服务事务一致性
    │
    ├─ 对隔离性要求高吗？（中间状态不能被看到）
    │   ├─ 是 → TCC（资源预留）
    │   └─ 否 → 继续判断
    │
    ├─ 需要同步响应吗？
    │   ├─ 是 → Saga（协调式）
    │   └─ 否 → 本地消息表
    │
    └─ 想要低侵入吗？
        ├─ 是 → 本地消息表
        └─ 否 → TCC 或 Saga
```

## 小明的选择

小明分析了自己的场景：

1. 下单不需要实时响应（用户可以等几秒看结果）
2. 可以接受最终一致性（只要最终扣款和发货一致）
3. 不想大改现有代码

最终选择了**本地消息表 + Saga 补偿**的组合：

- 订单服务创建订单时，写入消息表
- 后台任务发送消息给库存服务和用户服务
- 如果某个步骤失败，由 Saga 协调者触发补偿

```rust
pub async fn place_order(&self, order: OrderRequest) -> Result<OrderId, Error> {
    let mut tx = self.pool.begin().await?;

    // 1. 创建订单（pending 状态）
    let order = sqlx::query_as!(Order,
        "INSERT INTO orders (user_id, book_id, amount, status) VALUES ($1, $2, $3, 'pending') RETURNING *",
        order.user_id, order.book_id, order.amount
    ).fetch_one(&mut *tx).await?;

    // 2. 写入消息（扣款 + 扣库存）
    sqlx::query!(
        "INSERT INTO outbox_messages (aggregate_id, message_type, payload) VALUES ($1, 'ProcessOrder', $2)",
        order.id.to_string(),
        serde_json::to_value(&order)?
    ).execute(&mut *tx).await?;

    tx.commit().await?;

    // 3. 返回订单 ID，用户可以轮询状态
    Ok(order.id)
}
```

## 实战建议

1. **首选本地消息表**：简单可靠，适合大多数场景
2. **Saga 补偿要幂等**：每个操作都可能被调用多次
3. **做好监控和告警**：分布式事务失败后需要人工介入
4. **设计好状态机**：订单状态流转要清晰（pending → paid → shipped → failed）
5. **考虑人工补偿**：极端情况下的兜底方案

## 下一步

跨服务事务解决后，小明发现系统中已经大量使用消息来实现服务间通信。

但消息队列本身也可能出问题：消息丢失了怎么办？消息重复消费了怎么办？消息顺序乱了怎么办？

这就是下一章的主题：**消息驱动与最终一致性**。

---

*上一篇：[数据分片：跨分片事务一致性](/posts/consistency-in-practice-4-sharding/)*
*下一篇：[消息驱动：最终一致性](/posts/consistency-in-practice-6-messaging/)*

*本系列：*
1. *[单机时代：ACID 的庇护](/posts/consistency-in-practice-1-single-node/)*
2. *[读写分离：副本一致性](/posts/consistency-in-practice-2-read-write-split/)*
3. *[引入缓存：多存储一致性](/posts/consistency-in-practice-3-caching/)*
4. *[数据分片：跨分片事务一致性](/posts/consistency-in-practice-4-sharding/)*
5. *服务拆分：跨服务事务一致性（本篇）*
6. *[消息驱动：最终一致性](/posts/consistency-in-practice-6-messaging/)*
7. *[总结：演进全景与选型指南](/posts/consistency-in-practice-7-summary/)*
