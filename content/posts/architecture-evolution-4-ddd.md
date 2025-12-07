---
title: "架构演进史（四）：DDD 觉醒，让代码说业务的语言"
date: 2025-12-08T16:30:00+08:00
draft: false
tags: ["架构", "DDD", "领域驱动设计", "Rust", "Event Sourcing", "CQRS"]
categories: ["技术"]
description: "当我第一次把「工资单」从一个数据库表变成一个有行为的聚合根时，突然明白了 DDD 的意义：代码不再是「数据库的搬运工」，而是「业务规则的执行者」。"
series: ["架构演进史"]
---

当我第一次把「工资单」从一个数据库表变成一个有行为的聚合根时，突然明白了 DDD 的意义：代码不再是「数据库的搬运工」，而是「业务规则的执行者」。

<!--more-->

## 前情回顾

在前三篇中，我们经历了：

- **混沌时代**：代码没有边界
- **MVC 时代**：技术分层，但业务逻辑散落
- **Context 时代**：业务分层，但建模还是贫血的

Context 教会我们按业务边界组织代码。但代码内部呢？

看看一个典型的「贫血模型」：

```rust
struct Order {
    id: OrderId,
    user_id: UserId,
    items: Vec<OrderItem>,
    total_amount: Decimal,
    status: OrderStatus,
    created_at: DateTime<Utc>,
}
```

这个 `Order` 只有**数据**，没有**行为**。所有业务逻辑都在 Service 里：

```rust
impl OrderService {
    fn create_order(&self, user_id: UserId, items: Vec<OrderItem>) -> Result<Order> {
        // 验证用户...
        // 验证库存...
        // 计算金额...
        // 应用折扣...
        // 创建订单...
    }

    fn cancel_order(&self, order_id: OrderId) -> Result<()> {
        // 检查状态...
        // 更新状态...
        // 恢复库存...
        // 退款...
    }
}
```

**问题**：业务规则散落在 Service 里，`Order` 只是一个数据容器。

这就是 Eric Evans 所说的「贫血领域模型」（Anemic Domain Model）——虽然用了类和对象，但 Model 只是数据容器，业务逻辑全在 Service 里，**失去了 OOP 封装数据和行为的核心优势**。

## DDD 是什么？

DDD（Domain-Driven Design，领域驱动设计）是 Eric Evans 在 2003 年提出的。

它的核心思想：

> **软件的核心复杂性在于业务领域。把业务逻辑放在领域模型中，让代码说业务的语言。**

### 一个类比：公司 vs 外包

**贫血模型**像什么？像一个完全依赖外包的公司：

- 公司只有数据（员工花名册、财务报表）
- 所有事情都交给外包（Service）做
- 外包公司知道这家公司的所有业务规则

**问题**：外包公司换了，业务就乱了。因为业务知识不在公司内部。

**DDD 模型**像什么？像一个有核心能力的公司：

- 公司有自己的部门（聚合根）
- 每个部门知道自己的业务规则（领域逻辑）
- 外部只能通过正式渠道（API）与公司交互

**好处**：换合作伙伴、换系统，核心业务不受影响。

## DDD 的核心概念

让我用一个**工资单系统**来解释 DDD 的核心概念。这是我在 Rust 项目中实际实践的领域。

### 实体（Entity）

**实体**是有身份标识的对象。两个实体即使属性完全相同，只要 ID 不同，就是不同的实体。

```rust
struct Employee {
    id: EmployeeId,        // 身份标识
    name: String,
    department: Department,
    base_salary: Money,
}
```

**员工 A** 和 **员工 B** 即使同名同薪，也是不同的人。因为他们有不同的 `EmployeeId`。

### 值对象（Value Object）

**值对象**没有身份标识。它只由属性值定义。两个值对象如果属性相同，就是相等的。

```rust
#[derive(Clone, PartialEq)]
struct Money {
    amount: Decimal,
    currency: Currency,
}

impl Money {
    fn add(&self, other: &Money) -> Result<Money> {
        if self.currency != other.currency {
            return Err(DomainError::CurrencyMismatch);
        }
        Ok(Money {
            amount: self.amount + other.amount,
            currency: self.currency,
        })
    }
}
```

100 元 = 100 元。我们不关心「这是哪个 100 元」。

**值对象的好处**：
- 不可变（Immutable），天然线程安全
- 可以自由复制、共享
- 业务规则内聚（比如货币转换）

### 聚合根（Aggregate Root）

**聚合**是一组相关对象的集合，**聚合根**是聚合的入口。

外部只能通过聚合根访问聚合内的对象。

```rust
// 工资单是聚合根
struct Payslip {
    id: PayslipId,
    employee_id: EmployeeId,
    pay_period: PayPeriod,
    base_salary: Money,
    adjustments: Vec<SalaryAdjustment>,  // 聚合内部的实体
    deductions: Vec<Deduction>,          // 聚合内部的实体
    status: PayslipStatus,
}

// 工资调整项是聚合内部的实体
struct SalaryAdjustment {
    id: AdjustmentId,
    adjustment_type: AdjustmentType,
    amount: Money,
    reason: String,
}
```

**规则**：
- 外部不能直接创建 `SalaryAdjustment`
- 必须通过 `Payslip` 的方法来添加调整项
- `Payslip` 负责维护内部一致性

```rust
impl Payslip {
    pub fn add_adjustment(&mut self, adj_type: AdjustmentType, amount: Money, reason: String) -> Result<()> {
        // 验证业务规则
        if self.status != PayslipStatus::Draft {
            return Err(DomainError::CannotModifySubmittedPayslip);
        }

        if adj_type == AdjustmentType::Bonus && amount > self.base_salary {
            return Err(DomainError::BonusExceedsBaseSalary);
        }

        self.adjustments.push(SalaryAdjustment::new(adj_type, amount, reason));
        Ok(())
    }
}
```

**类比**：聚合根就像「家庭代表」。外人要联系家庭成员，必须先联系户主。户主负责维护家庭内部的秩序。

### 领域事件（Domain Event）

**领域事件**记录了领域中发生的重要事情。事件使用**过去时态**命名（`Created`、`Approved`），因为它们记录的是**已经发生的事实**。

使用 `ddd-macros` 库，领域事件通过宏来定义：

```rust
use ddd_macros::domain_event;

/// 工资单领域事件
/// 每个事件记录一个已完成的状态变更，遵循事件溯源模式
#[domain_event]
pub enum PayslipEvent {
    /// 工资单已创建
    Created {
        employee_id: EmployeeId,
        pay_period: PayPeriod,
        base_salary: Decimal,
        allowance: Decimal,
    },
    /// 调整项已设置（完整替换现有列表）
    AdjustmentsSet {
        adjustments: Vec<AdjustmentItem>,
    },
    /// 工资单已提交审批（单元变体自动注入 id 和 aggregate_version）
    Submitted,
    /// 工资单已审批通过
    Approved {
        approver_note: Option<String>,
    },
    /// 工资单已被拒绝
    Rejected {
        rejection_reason: String,
    },
    /// 工资已发放
    Paid {
        paid_at: DateTime<Utc>,
        payment_method: String,
    },
}
```

`#[domain_event]` 宏会自动为每个变体添加 `id`（事件唯一标识）和 `aggregate_version`（聚合版本）字段，用于事件溯源和乐观锁。

**领域事件的价值**：
- **解耦**：其他系统可以订阅事件，不需要直接调用
- **审计**：事件序列就是完整的历史记录
- **扩展**：新增功能只需要监听现有事件

## Event Sourcing：用事件记录一切

Event Sourcing（事件溯源）是 DDD 的一个重要模式：**不存储当前状态，只存储导致状态变化的事件**。

### 传统方式：存储状态

```text
payslips 表
| id | employee_id | base_salary | total_adjustments | status    |
|----|-------------|-------------|-------------------|-----------|
| 1  | emp_001     | 10000.00    | 2000.00           | approved  |
```

问题：
- 不知道工资单经历了什么
- 不知道谁在什么时候做了什么修改
- 回滚需要备份

### Event Sourcing：存储事件

```text
payslip_events 表
| event_id | payslip_id | event_type      | event_data                              | occurred_at |
|----------|------------|-----------------|------------------------------------------|-------------|
| 1        | ps_001     | Created         | {employee_id: emp_001, base: 10000}     | 2025-01-01  |
| 2        | ps_001     | AdjustmentAdded | {type: bonus, amount: 1500}             | 2025-01-02  |
| 3        | ps_001     | AdjustmentAdded | {type: overtime, amount: 500}           | 2025-01-03  |
| 4        | ps_001     | Submitted       | {submitted_by: user_001}                | 2025-01-04  |
| 5        | ps_001     | Approved        | {approved_by: user_002}                 | 2025-01-05  |
```

**重建状态**：按顺序回放事件，就能得到当前状态。

```rust
impl Aggregate for Payslip {
    const TYPE: &'static str = "payslip";
    type Command = PayslipCommand;
    type Event = PayslipEvent;
    type Error = DomainError;

    fn execute(&self, command: Self::Command) -> Result<Vec<Self::Event>, Self::Error> {
        // 命令处理逻辑，产生事件（见下文）
        todo!()
    }

    fn apply(&mut self, event: &Self::Event) {
        match event {
            PayslipEvent::Created { aggregate_version, base_salary, .. } => {
                self.base_salary = base_salary.clone();
                self.status = PayslipStatus::Draft;
                self.version = *aggregate_version;
            }
            PayslipEvent::AdjustmentAdded { aggregate_version, amount, .. } => {
                self.total_adjustments = self.total_adjustments.add(amount).unwrap();
                self.version = *aggregate_version;
            }
            PayslipEvent::Submitted { aggregate_version, .. } => {
                self.status = PayslipStatus::Submitted;
                self.version = *aggregate_version;
            }
            PayslipEvent::Approved { aggregate_version, .. } => {
                self.status = PayslipStatus::Approved;
                self.version = *aggregate_version;
            }
        }
    }
}
```

注意：`apply` 方法接收事件的引用（`&Self::Event`），并且需要更新聚合的版本号。

**Event Sourcing 的好处**：
- **完整历史**：任何时间点的状态都能重建
- **审计友好**：天然的审计日志
- **调试利器**：回放事件重现问题
- **时间旅行**：可以查询「上个月的状态」

## CQRS：读写分离

CQRS（Command Query Responsibility Segregation）把读和写分开。

### 为什么要分开？

Event Sourcing 重建状态需要回放所有事件，查询可能很慢。

比如「查询所有待审批的工资单」：
1. 加载所有工资单的事件
2. 重建每个工资单的状态
3. 过滤出待审批的

**太慢了！**

### CQRS 的解决方案

```text
                  ┌─────────────┐
   Command ──────>│  聚合根     │──────> Event Store
                  │ (写模型)    │
                  └─────────────┘
                         │
                         ▼ 事件
                  ┌─────────────┐
                  │  投影/视图  │
                  │ (读模型)    │
                  └─────────────┘
                         │
                         ▼
   Query ─────────────────────────────> 读数据库
```

**写模型**（Command Side）：
- 接收命令（CreatePayslip、AddAdjustment）
- 执行业务规则验证
- 产生领域事件
- 存储事件

**读模型**（Query Side）：
- 订阅领域事件
- 更新查询优化的视图
- 快速响应查询

```rust
// 写模型：实现 Aggregate trait
impl Aggregate for Payslip {
    const TYPE: &'static str = "payslip";
    type Command = PayslipCommand;
    type Event = PayslipEvent;
    type Error = DomainError;

    fn execute(&self, command: Self::Command) -> Result<Vec<Self::Event>, Self::Error> {
        match command {
            PayslipCommand::Create { employee_id, pay_period, base_salary } => {
                if self.version().is_created() {
                    return Err(DomainError::invalid_state("payslip already created"));
                }
                // 业务规则验证...
                Ok(vec![PayslipEvent::Created {
                    id: Ulid::new().to_string(),
                    aggregate_version: self.version().next(),
                    employee_id,
                    pay_period,
                    base_salary,
                }])
            }
            PayslipCommand::AddAdjustment { adjustment_type, amount, reason } => {
                if self.status != PayslipStatus::Draft {
                    return Err(DomainError::invalid_state("cannot modify submitted payslip"));
                }
                Ok(vec![PayslipEvent::AdjustmentAdded {
                    id: Ulid::new().to_string(),
                    aggregate_version: self.version().next(),
                    adjustment_type,
                    amount,
                    reason,
                }])
            }
        }
    }

    fn apply(&mut self, event: &Self::Event) {
        // 应用事件更新状态...
    }
}

// 读模型：更新视图
impl PayslipProjection {
    pub fn apply(&mut self, event: &PayslipEvent) {
        match event {
            PayslipEvent::Created { employee_id, .. } => {
                // 插入到查询表
                self.insert_payslip_view(employee_id, ...);
            }
            PayslipEvent::Submitted { .. } => {
                // 更新状态
                self.update_status("submitted");
            }
            _ => {}
        }
    }
}
```

## 我的 Rust DDD 实践

在我的 Rust 项目中，目录结构是这样的：

```text
src/
├── domain/                    # 领域层
│   └── payslip/
│       ├── mod.rs
│       ├── aggregate.rs       # 聚合根
│       ├── commands.rs        # 命令
│       ├── events.rs          # 领域事件
│       ├── entities.rs        # 实体
│       └── value_objects.rs   # 值对象
├── application/               # 应用层
│   └── payslip/
│       ├── command_handler.rs # 命令处理器
│       └── query_handler.rs   # 查询处理器
├── infrastructure/            # 基础设施层
│   └── persistence/
│       ├── event_store.rs     # 事件存储
│       └── projections.rs     # 读模型投影
└── interface/                 # 接口层
    └── api/
        └── payslip_handler.rs # HTTP 处理器
```

### 领域层：纯粹的业务逻辑

使用 `nanlong/ddd` 库，聚合根通过宏和 trait 来定义：

```rust
// domain/payslip/aggregate.rs
use ddd_domain::aggregate::Aggregate;
use ddd_domain::entity::Entity;
use ddd_domain::error::DomainError;
use ddd_domain::value_object::Version;
use ddd_macros::{entity, entity_id, domain_event};
use ulid::Ulid;

// 使用宏定义聚合根 ID
#[entity_id]
pub struct PayslipId(Ulid);

// 使用宏定义聚合根实体
#[entity(id = PayslipId)]
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct Payslip {
    base_salary: Money,
    adjustments: Vec<SalaryAdjustment>,
    status: PayslipStatus,
}

// 命令定义
#[derive(Debug)]
pub enum PayslipCommand {
    Create { employee_id: EmployeeId, pay_period: PayPeriod, base_salary: Money },
    AddAdjustment { adjustment_type: AdjustmentType, amount: Money, reason: String },
    Submit,
}

// 领域事件定义（使用过去时态命名）
#[domain_event]
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum PayslipEvent {
    Created {
        employee_id: EmployeeId,
        pay_period: PayPeriod,
        base_salary: Money,
    },
    AdjustmentAdded {
        adjustment_type: AdjustmentType,
        amount: Money,
        reason: String,
    },
    Submitted,  // 单元变体自动转换为 Submitted { id, aggregate_version }
}

// 实现 Aggregate trait
impl Aggregate for Payslip {
    const TYPE: &'static str = "payslip";
    type Command = PayslipCommand;
    type Event = PayslipEvent;
    type Error = DomainError;

    fn execute(&self, command: Self::Command) -> Result<Vec<Self::Event>, Self::Error> {
        match command {
            PayslipCommand::Create { employee_id, pay_period, base_salary } => {
                // 状态检查：不能重复创建
                if self.version().is_created() {
                    return Err(DomainError::invalid_state("payslip already created"));
                }
                // 业务规则验证
                if base_salary.is_negative() {
                    return Err(DomainError::invalid_value("base salary cannot be negative"));
                }
                // 宏自动注入 id 和 aggregate_version 字段
                Ok(vec![PayslipEvent::Created {
                    id: Ulid::new().to_string(),
                    aggregate_version: self.version().next(),
                    employee_id,
                    pay_period,
                    base_salary,
                }])
            }
            PayslipCommand::AddAdjustment { adjustment_type, amount, reason } => {
                // 状态检查
                if self.status != PayslipStatus::Draft {
                    return Err(DomainError::invalid_state("cannot modify submitted payslip"));
                }
                // 业务规则：奖金不能超过基本工资的 50%
                if adjustment_type == AdjustmentType::Bonus {
                    let max_bonus = self.base_salary.multiply(dec!(0.5));
                    if amount > max_bonus {
                        return Err(DomainError::invalid_command("bonus exceeds 50% of base salary"));
                    }
                }
                Ok(vec![PayslipEvent::AdjustmentAdded {
                    id: Ulid::new().to_string(),
                    aggregate_version: self.version().next(),
                    adjustment_type,
                    amount,
                    reason,
                }])
            }
            PayslipCommand::Submit => {
                if self.status != PayslipStatus::Draft {
                    return Err(DomainError::invalid_state("can only submit draft payslip"));
                }
                Ok(vec![PayslipEvent::Submitted {
                    id: Ulid::new().to_string(),
                    aggregate_version: self.version().next(),
                }])
            }
        }
    }

    fn apply(&mut self, event: &Self::Event) {
        match event {
            PayslipEvent::Created { aggregate_version, base_salary, .. } => {
                self.base_salary = base_salary.clone();
                self.status = PayslipStatus::Draft;
                self.version = *aggregate_version;
            }
            PayslipEvent::AdjustmentAdded { aggregate_version, adjustment_type, amount, reason, .. } => {
                self.adjustments.push(SalaryAdjustment {
                    adjustment_type: adjustment_type.clone(),
                    amount: amount.clone(),
                    reason: reason.clone(),
                });
                self.version = *aggregate_version;
            }
            PayslipEvent::Submitted { aggregate_version, .. } => {
                self.status = PayslipStatus::Submitted;
                self.version = *aggregate_version;
            }
        }
    }
}
```

**关键设计**：
- `execute()` 方法**只产生事件，不改变状态**
- `apply()` 方法**只更新状态，不做业务验证**
- 领域层**不依赖任何框架**（没有 Axum、没有 SQLx、没有 HTTP），纯粹的业务逻辑

### 应用层：协调和编排

应用层使用 `CommandHandler` 和 `QueryHandler` trait 组织代码，通过 `AggregateRoot` 编排器简化命令处理流程：

```rust
// application/payslip/service.rs
use async_trait::async_trait;
use ddd_application::{CommandHandler, QueryHandler, AppContext, AppError};
use ddd_domain::aggregate_root::AggregateRoot;

/// 工资单应用服务
/// 使用 bon::Builder 简化依赖注入
#[derive(bon::Builder)]
pub struct PayslipService {
    payslip_root: Arc<AggregateRoot<Payslip>>,
    payslip_provider: Arc<dyn PayslipProvider>,
    payslip_checker: Arc<dyn PayslipChecker>,
}

/// 提交工资单命令
pub struct SubmitPayslipCommand {
    pub payslip_id: PayslipId,
}

/// 实现命令处理器
#[async_trait]
impl CommandHandler<SubmitPayslipCommand> for PayslipService {
    async fn handle(&self, ctx: &AppContext, cmd: SubmitPayslipCommand) -> Result<(), AppError> {
        // 1. 业务检查（通过 Checker trait）
        if !self.payslip_checker.exists(&cmd.payslip_id).await? {
            return Err(AppError::not_found("payslip not found"));
        }

        // 2. 执行领域命令
        // AggregateRoot 自动处理：加载 → 执行 → 应用 → 持久化
        self.payslip_root
            .execute(
                &cmd.payslip_id,
                vec![PayslipCommand::Submit],
                ctx.event_context.clone(),
            )
            .await?;

        Ok(())
    }
}

/// 查询处理器
#[async_trait]
impl QueryHandler<GetPayslipQuery, PayslipView> for PayslipService {
    async fn handle(&self, _ctx: &AppContext, query: GetPayslipQuery) -> Result<PayslipView, AppError> {
        // 通过 Provider 查询读模型
        self.payslip_provider
            .get_by_id(&query.payslip_id)
            .await?
            .ok_or_else(|| AppError::not_found("payslip not found"))
    }
}
```

**应用层的职责**：
- **Checker**：业务前置检查（存在性、权限等）
- **Provider**：查询数据（CQRS 的读模型）
- **AggregateRoot**：执行领域命令（CQRS 的写模型）
- **不包含业务逻辑**：业务规则在领域层的 `execute()` 中

### 基础设施层：技术实现

基础设施层实现领域层和应用层定义的 trait，提供具体的技术实现：

```rust
// infrastructure/postgres/payslip_repository.rs
use async_trait::async_trait;
use ddd_domain::persist::{AggregateRepository, EventRepository, serialize_events};
use ddd_domain::domain_event::{EventContext, EventEnvelope};

pub struct PgPayslipRepository {
    pool: Arc<PgPool>,
    event_repo: Arc<PgEventRepository>,
}

#[async_trait]
impl AggregateRepository<Payslip> for PgPayslipRepository {
    async fn load(&self, aggregate_id: &PayslipId) -> Result<Option<Payslip>, DomainError> {
        // 从快照表加载聚合状态（避免每次重放所有事件）
        let row = sqlx::query_as!(
            PayslipRow,
            "SELECT * FROM payslips WHERE id = $1",
            aggregate_id.to_string(),
        )
        .fetch_optional(self.pool.as_ref())
        .await?;

        Ok(row.map(|r| r.into()))
    }

    async fn save(
        &self,
        aggregate: &Payslip,
        events: Vec<PayslipEvent>,
        context: EventContext,
    ) -> Result<Vec<EventEnvelope<Payslip>>, DomainError> {
        // 1. 封装事件
        let envelopes: Vec<EventEnvelope<Payslip>> = events
            .into_iter()
            .map(|e| EventEnvelope::new(aggregate.id(), e, context.clone()))
            .collect();

        // 2. 乐观锁检查
        let expected_version = aggregate.version().value().saturating_sub(envelopes.len());
        let actual = self.get_current_version(aggregate.id()).await?;
        if actual != expected_version {
            return Err(DomainError::conflict(expected_version, actual));
        }

        // 3. 持久化事件
        let serialized = serialize_events(&envelopes)?;
        self.event_repo.save(serialized).await?;

        // 4. 更新聚合快照
        self.upsert_snapshot(aggregate).await?;

        Ok(envelopes)
    }
}

// infrastructure/postgres/payslip_provider.rs
/// 实现应用层定义的 Provider trait
#[async_trait]
impl PayslipProvider for PgPayslipProvider {
    async fn get_by_id(&self, id: &PayslipId) -> Result<Option<PayslipView>, AppError> {
        // 从读模型表查询
        let row = sqlx::query_as!(
            PayslipViewRow,
            r#"SELECT * FROM payslip_views WHERE id = $1"#,
            id.to_string(),
        )
        .fetch_optional(self.pool.as_ref())
        .await?;

        Ok(row.map(Into::into))
    }

    async fn list_pending(&self, page: PageRequest) -> Result<PageResponse<PayslipView>, AppError> {
        // 分页查询待审批的工资单
        let rows = sqlx::query_as!(
            PayslipViewRow,
            r#"SELECT * FROM payslip_views
               WHERE status = 'submitted'
               ORDER BY created_at DESC
               LIMIT $1 OFFSET $2"#,
            page.limit as i64,
            page.offset as i64,
        )
        .fetch_all(self.pool.as_ref())
        .await?;

        Ok(PageResponse::new(rows.into_iter().map(Into::into).collect()))
    }
}
```

**关键设计**：
- `load()` 从快照表加载聚合状态（避免每次重放所有事件）
- `save()` 先检查乐观锁版本，再持久化事件，最后更新快照
- `Provider` 实现读模型查询，支持分页和复杂过滤
- 乐观锁确保并发安全：`DomainError::conflict(expected, actual)` 提示调用者重试

## DDD 的价值与代价

### 价值

| 方面 | 贫血模型 | DDD |
|------|---------|-----|
| 业务规则 | 散落在 Service | 内聚在聚合根 |
| 可测试性 | 需要 mock 数据库 | 领域层纯函数 |
| 可读性 | 技术语言 | 业务语言 |
| 可维护性 | 改一处，动多处 | 改一处，影响可控 |
| 历史追溯 | 额外的审计表 | 事件天然记录 |

### 代价

| 方面 | 说明 |
|------|------|
| 学习曲线 | DDD 概念多，需要时间理解 |
| 代码量 | 比贫血模型多（事件、命令、聚合根） |
| 复杂度 | 简单 CRUD 没必要用 DDD |
| 最终一致性 | CQRS 的读模型有延迟 |

### 什么时候用 DDD？

**适合**：
- 业务逻辑复杂（很多 if-else、状态转换）
- 业务规则频繁变化
- 需要审计和历史追溯
- 团队有领域专家

**不适合**：
- 简单 CRUD
- 业务逻辑很少
- 快速原型
- 团队不熟悉 DDD

## 常见误区

### 误区 1：DDD = 微服务

**错误。** DDD 是一种设计方法，与部署架构无关。

我的 Rust 项目是**单体应用**，但用了 DDD。

单体 + DDD 是完全合理的选择。

### 误区 2：每个实体都要是聚合根

**错误。** 聚合根是有边界的。

- `Order` 是聚合根
- `OrderItem` 不是聚合根，它属于 `Order` 聚合

判断标准：这个对象能独立存在吗？需要独立的生命周期吗？

### 误区 3：Event Sourcing 是必须的

**错误。** Event Sourcing 是可选的。

你可以用 DDD 的聚合根、实体、值对象，但用传统方式存储状态。

Event Sourcing 在需要历史追溯、审计的场景更有价值。

## 总结

**DDD 的核心价值**：
- **业务逻辑内聚**：放在领域模型中，不散落各处
- **业务语言**：代码即文档，代码说业务的话
- **边界清晰**：聚合根是一致性边界
- **可测试**：领域层不依赖框架，纯函数测试

**从贫血到充血的转变**：

| 方面 | 贫血模型 | 充血模型（DDD） |
|------|---------|----------------|
| 模型 | 只有数据 | 数据 + 行为 |
| Service | 业务逻辑堆在这里 | 只做协调，不做业务判断 |
| 验证 | Service 里验证 | 聚合根内部验证 |
| 一致性 | 靠开发者自觉 | 靠聚合根保证 |

**核心教训**：

> **DDD 不是微服务的前置条件，而是好代码的必然选择。**

好的代码应该用业务语言描述业务规则。而 DDD 提供了一套完整的模式来实现这一点。

下一篇，我们来看六边形架构和洋葱架构——如何用「依赖倒置」让领域层真正独立。

---

*上一篇：[Context 之道：从技术分层到业务分层](/posts/architecture-evolution-3-context/)*

*下一篇：[边界的艺术：六边形与洋葱](/posts/architecture-evolution-5-hexagonal/)*

*本系列：*
1. *[混沌时代：当代码没有架构](/posts/architecture-evolution-1-chaos/)*
2. *[MVC 启蒙：框架带来的秩序](/posts/architecture-evolution-2-mvc/)*
3. *[Context 之道：从技术分层到业务分层](/posts/architecture-evolution-3-context/)*
4. *DDD 觉醒：让代码说业务的语言（本篇）*
5. *[边界的艺术：六边形与洋葱](/posts/architecture-evolution-5-hexagonal/)*
6. *[单体的边界：何时该拆？](/posts/architecture-evolution-6-modular-monolith/)*
7. *[微服务与云原生：分布式的代价](/posts/architecture-evolution-7-microservices/)*
8. *[没有银弹：架构决策的本质](/posts/architecture-evolution-8-no-silver-bullet/)*
