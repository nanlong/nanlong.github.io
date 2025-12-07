---
title: "架构演进史（五）：边界的艺术，六边形与洋葱"
date: 2025-12-08T16:40:00+08:00
draft: false
tags: ["架构", "六边形架构", "洋葱架构", "Clean Architecture", "依赖倒置", "Rust"]
categories: ["技术"]
description: "有一天我突然意识到：我的领域层里有 sqlx::PgPool。这意味着，如果我想换数据库，要改领域层。领域层不是应该只有业务逻辑吗？"
series: ["架构演进史"]
---

有一天我突然意识到：我的领域层里有 `sqlx::PgPool`。这意味着，如果我想换数据库，要改领域层。领域层不是应该只有业务逻辑吗？

<!--more-->

## 前情回顾

在前四篇中，我们经历了：

- **混沌时代**：代码没有边界
- **MVC 时代**：技术分层
- **Context 时代**：业务分层
- **DDD 时代**：领域建模

DDD 教会我们用业务语言组织代码。但有一个问题：**领域层真的独立吗？**

## 一个尴尬的发现

看看这段「领域层」的代码：

```rust
// domain/payslip/repository.rs
use sqlx::PgPool;

pub struct PayslipRepository {
    pool: PgPool,  // 😱 领域层依赖了 sqlx！
}

impl PayslipRepository {
    pub async fn find_by_id(&self, id: PayslipId) -> Result<Payslip> {
        sqlx::query_as!(
            PayslipRow,
            "SELECT * FROM payslips WHERE id = $1",
            id.as_uuid()
        )
        .fetch_one(&self.pool)
        .await
        .map(|row| row.into())
    }
}
```

**问题**：
- 领域层依赖了 `sqlx`（数据库框架）
- 如果换 MongoDB，要改领域层
- 领域层没法独立测试（需要真实数据库）

这不是 DDD 承诺的「领域层独立」！

## 依赖的方向

让我画一下常见的依赖关系：

### 传统分层架构

```text
┌─────────────────────────┐
│    Interface Layer      │ (HTTP Handler)
└───────────┬─────────────┘
            │ 依赖
            ▼
┌─────────────────────────┐
│   Application Layer     │ (Service)
└───────────┬─────────────┘
            │ 依赖
            ▼
┌─────────────────────────┐
│     Domain Layer        │ (Entity, Repository)
└───────────┬─────────────┘
            │ 依赖
            ▼
┌─────────────────────────┐
│  Infrastructure Layer   │ (Database, Cache)
└─────────────────────────┘
```

**问题**：Domain Layer 依赖了 Infrastructure Layer。

业务逻辑依赖了数据库细节。换数据库？改业务代码。

### 正确的依赖方向

```text
┌─────────────────────────┐
│    Interface Layer      │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│   Application Layer     │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│     Domain Layer        │ ◄──── 核心！不依赖任何外部
└───────────▲─────────────┘
            │
            │ 依赖倒置
            │
┌─────────────────────────┐
│  Infrastructure Layer   │
└─────────────────────────┘
```

**关键**：Infrastructure 依赖 Domain 和 Application，而不是反过来。

这就是「依赖倒置原则」（Dependency Inversion Principle）。

## 六边形架构

六边形架构（Hexagonal Architecture）也叫「端口与适配器」架构（Ports and Adapters），由 Alistair Cockburn 在 2005 年提出。

### 核心思想

> **应用程序应该通过端口与外部世界交互，而不是直接依赖外部实现。**

```text
                    ┌──────────────┐
        ┌───────────│   HTTP API   │
        │           └──────────────┘
        │
        ▼
┌───────────────┐         ┌─────────────────────────┐
│  Web Adapter  │────────>│                         │
└───────────────┘         │                         │
                          │       Application       │
┌───────────────┐         │          Core           │
│  CLI Adapter  │────────>│                         │
└───────────────┘         │   ┌─────────────────┐   │
                          │   │  Domain Model   │   │
┌───────────────┐         │   └─────────────────┘   │
│  Test Adapter │────────>│                         │
└───────────────┘         └───────────┬─────────────┘
                                      │
                          ┌───────────▼───────────┐
                          │   Repository Port     │
                          └───────────┬───────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
     ┌────────▼────────┐   ┌─────────▼────────┐   ┌─────────▼────────┐
     │  PostgreSQL     │   │    MongoDB       │   │   In-Memory      │
     │  Adapter        │   │    Adapter       │   │   Adapter        │
     └─────────────────┘   └──────────────────┘   └──────────────────┘
```

### 端口（Port）

端口是应用程序的**接口定义**。它定义了「我需要什么能力」，但不关心「谁来提供」。

```rust
// 领域层定义端口（接口）
// domain/ports/repository.rs

#[async_trait]
pub trait PayslipRepository: Send + Sync {
    async fn find_by_id(&self, id: PayslipId) -> Result<Option<Payslip>>;
    async fn save(&self, payslip: &Payslip) -> Result<()>;
    async fn find_by_employee(&self, employee_id: EmployeeId) -> Result<Vec<Payslip>>;
}
```

领域层**只定义接口**，不实现。这是一个「端口」。

### 适配器（Adapter）

适配器是端口的**具体实现**。它把外部系统「适配」成应用程序需要的接口。

```rust
// 基础设施层实现适配器
// infrastructure/persistence/postgres_payslip_repository.rs

pub struct PostgresPayslipRepository {
    pool: PgPool,
}

#[async_trait]
impl PayslipRepository for PostgresPayslipRepository {
    async fn find_by_id(&self, id: PayslipId) -> Result<Option<Payslip>> {
        let row = sqlx::query_as!(
            PayslipRow,
            "SELECT * FROM payslips WHERE id = $1",
            id.as_uuid()
        )
        .fetch_optional(&self.pool)
        .await?;

        Ok(row.map(Into::into))
    }

    async fn save(&self, payslip: &Payslip) -> Result<()> {
        // ...
    }

    async fn find_by_employee(&self, employee_id: EmployeeId) -> Result<Vec<Payslip>> {
        // ...
    }
}
```

### 好处

**1. 领域层完全独立**

领域层只依赖自己定义的接口，不知道外部世界的存在。

```rust
// domain/payslip/service.rs
pub struct PayslipService {
    repository: Arc<dyn PayslipRepository>,  // 只依赖接口
}

impl PayslipService {
    pub async fn get_payslip(&self, id: PayslipId) -> Result<Payslip> {
        self.repository
            .find_by_id(id)
            .await?
            .ok_or(DomainError::PayslipNotFound)
    }
}
```

**2. 可以轻松替换实现**

```rust
// 生产环境用 PostgreSQL
let repo = PostgresPayslipRepository::new(pg_pool);

// 测试环境用内存实现
let repo = InMemoryPayslipRepository::new();

// 都满足 PayslipRepository 接口
let service = PayslipService::new(Arc::new(repo));
```

**3. 测试变得简单**

```rust
#[tokio::test]
async fn test_get_payslip() {
    // 使用内存实现，不需要真实数据库
    let mut repo = InMemoryPayslipRepository::new();
    repo.add(Payslip::new(...));

    let service = PayslipService::new(Arc::new(repo));

    let result = service.get_payslip(payslip_id).await;
    assert!(result.is_ok());
}
```

## 洋葱架构

洋葱架构（Onion Architecture）由 Jeffrey Palermo 在 2008 年提出，是六边形架构的变体。

### 核心思想

> **像洋葱一样，一层包一层。内层不知道外层的存在。**

```text
        ┌──────────────────────────────────────────────┐
        │              Infrastructure                  │
        │  ┌────────────────────────────────────────┐  │
        │  │          Application Services          │  │
        │  │  ┌──────────────────────────────────┐  │  │
        │  │  │        Domain Services           │  │  │
        │  │  │  ┌────────────────────────────┐  │  │  │
        │  │  │  │      Domain Model          │  │  │  │
        │  │  │  │  ┌──────────────────────┐  │  │  │  │
        │  │  │  │  │      Entities        │  │  │  │  │
        │  │  │  │  └──────────────────────┘  │  │  │  │
        │  │  │  └────────────────────────────┘  │  │  │
        │  │  └──────────────────────────────────┘  │  │
        │  └────────────────────────────────────────┘  │
        └──────────────────────────────────────────────┘
```

**规则**：
- 依赖方向：只能从外向内
- 内层不知道外层存在
- 越往里，越稳定、越核心

### 各层职责

| 层 | 职责 | 示例 |
|----|------|------|
| Entities | 业务实体、值对象 | `Payslip`, `Money`, `PayPeriod` |
| Domain Model | 聚合根、领域逻辑 | `PayslipAggregate`, `PayslipService` |
| Domain Services | 跨聚合的领域逻辑 | `SalaryCalculator` |
| Application Services | 用例编排、事务 | `CreatePayslipHandler` |
| Infrastructure | 外部依赖 | Database, HTTP, Message Queue |

## Clean Architecture

Clean Architecture 是 Robert C. Martin（Uncle Bob）在 2012 年提出的，综合了六边形、洋葱等架构的思想。

### 核心思想

> **框架是细节，数据库是细节，Web 是细节。业务规则是核心。**

```text
┌────────────────────────────────────────────────────────────┐
│                    Frameworks & Drivers                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │               Interface Adapters                     │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │             Application Business Rules         │  │  │
│  │  │  ┌──────────────────────────────────────────┐  │  │  │
│  │  │  │       Enterprise Business Rules          │  │  │  │
│  │  │  │                                          │  │  │  │
│  │  │  │              Entities                    │  │  │  │
│  │  │  │                                          │  │  │  │
│  │  │  └──────────────────────────────────────────┘  │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### 依赖规则

> **源代码依赖只能指向内层。内层不知道外层的任何信息。**

这意味着：
- Entity 不知道 Use Case 的存在
- Use Case 不知道 Controller 的存在
- Controller 不知道哪个框架在驱动它

## 实战：Rust 项目的架构

让我展示我的 Rust 项目如何实现这些原则。

### 目录结构

```text
src/
├── domain/                      # 领域层（最内层）
│   ├── mod.rs
│   ├── entities/                # 实体和值对象
│   │   ├── payslip.rs
│   │   └── money.rs
│   ├── aggregates/              # 聚合根
│   │   └── payslip_aggregate.rs
│   ├── services/                # 领域服务
│   │   └── salary_calculator.rs
│   └── ports/                   # 端口定义（接口）
│       ├── repository.rs
│       └── event_publisher.rs
│
├── application/                 # 应用层
│   ├── mod.rs
│   ├── commands/                # 命令
│   │   └── create_payslip.rs
│   ├── queries/                 # 查询
│   │   └── get_payslip.rs
│   └── handlers/                # 命令/查询处理器
│       ├── command_handler.rs
│       └── query_handler.rs
│
├── infrastructure/              # 基础设施层（最外层）
│   ├── mod.rs
│   ├── persistence/             # 数据库适配器
│   │   ├── postgres_repository.rs
│   │   └── in_memory_repository.rs
│   └── messaging/               # 消息适配器
│       └── kafka_publisher.rs
│
└── interface/                   # 接口层
    ├── mod.rs
    └── api/                     # HTTP 适配器
        ├── handlers.rs
        └── dto.rs
```

### 领域层：定义端口

使用 `nanlong/ddd` 库，仓储端口通过 `AggregateRepository` trait 定义：

```rust
// domain/ports/repository.rs

use ddd_domain::persist::AggregateRepository;
use ddd_domain::domain_event::{EventContext, EventEnvelope};
use ddd_domain::error::DomainError;
use async_trait::async_trait;

// 使用库提供的 AggregateRepository trait
// 这个 trait 定义了聚合的加载和保存接口
#[async_trait]
pub trait AggregateRepository<A: Aggregate>: Send + Sync {
    async fn load(&self, aggregate_id: &A::Id) -> Result<Option<A>, DomainError>;

    async fn save(
        &self,
        aggregate: &A,
        events: Vec<A::Event>,
        context: EventContext,
    ) -> Result<Vec<EventEnvelope<A>>, DomainError>;
}

// 如果需要额外的查询能力，可以定义扩展 trait
#[async_trait]
pub trait PayslipQueryRepository: Send + Sync {
    async fn find_pending_approval(&self) -> Result<Vec<PayslipView>, DomainError>;
    async fn find_by_employee(&self, employee_id: &EmployeeId) -> Result<Vec<PayslipView>, DomainError>;
}

/// 事件发布端口
#[async_trait]
pub trait EventPublisher: Send + Sync {
    async fn publish<A: Aggregate>(&self, events: &[EventEnvelope<A>]) -> Result<(), DomainError>;
}
```

**注意**：这些 trait 在领域层定义，但**实现在基础设施层**。`AggregateRepository` 来自 `ddd_domain::persist` 模块。

### 应用层：使用端口

应用层使用 `CommandHandler` trait 和 `AggregateRoot` 编排器：

```rust
// application/payslip/service.rs

use async_trait::async_trait;
use ddd_application::{CommandHandler, AppContext, AppError};
use ddd_domain::aggregate_root::AggregateRoot;
use std::sync::Arc;

/// 工资单应用服务
/// 使用 bon::Builder 简化依赖注入
#[derive(bon::Builder)]
pub struct PayslipService {
    payslip_root: Arc<AggregateRoot<Payslip>>,
    payslip_checker: Arc<dyn PayslipChecker>,  // 端口：检查器
    event_publisher: Arc<dyn EventPublisher>,  // 端口：事件发布
}

/// 创建工资单的应用层命令
pub struct CreatePayslipCommand {
    pub employee_id: EmployeeId,
    pub pay_period: PayPeriod,
    pub base_salary: Money,
}

/// 实现命令处理器
#[async_trait]
impl CommandHandler<CreatePayslipCommand, PayslipId> for PayslipService {
    async fn handle(&self, ctx: &AppContext, cmd: CreatePayslipCommand) -> Result<PayslipId, AppError> {
        let payslip_id = PayslipId::new();

        // AggregateRoot 自动处理：加载 → 执行 → 应用 → 持久化
        let events = self.payslip_root
            .execute(
                &payslip_id,
                vec![PayslipCommand::Create {
                    employee_id: cmd.employee_id,
                    pay_period: cmd.pay_period,
                    base_salary: cmd.base_salary,
                }],
                ctx.event_context.clone(),
            )
            .await?;

        // 发布领域事件
        self.event_publisher.publish(&events).await?;

        Ok(payslip_id)
    }
}
```

**注意**：应用层依赖端口（trait），不知道具体实现是 PostgreSQL 还是内存。

### 基础设施层：实现端口

实现 `AggregateRepository` trait，提供 PostgreSQL 适配器：

```rust
// infrastructure/postgres/pg_payslip_repo.rs

use ddd_domain::persist::{AggregateRepository, EventRepository, serialize_events};
use ddd_domain::domain_event::{EventContext, EventEnvelope};
use ddd_domain::error::DomainError;
use sqlx::PgPool;
use async_trait::async_trait;
use std::sync::Arc;

pub struct PgPayslipRepository {
    pool: Arc<PgPool>,
    event_repo: Arc<PgEventRepository>,
}

impl PgPayslipRepository {
    pub fn new(pool: Arc<PgPool>, event_repo: Arc<PgEventRepository>) -> Self {
        Self { pool, event_repo }
    }
}

#[async_trait]
impl AggregateRepository<Payslip> for PgPayslipRepository {
    async fn load(&self, aggregate_id: &PayslipId) -> Result<Option<Payslip>, DomainError> {
        // 从快照表加载（推荐，性能好）
        let row = sqlx::query_as!(
            PayslipRow,
            "SELECT * FROM payslips WHERE id = $1",
            aggregate_id.to_string()
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
```

同时可以提供一个内存实现用于测试：

```rust
// infrastructure/persistence/in_memory_repository.rs

#[derive(Default, Clone)]
pub struct InMemoryPayslipRepository {
    states: Arc<Mutex<HashMap<String, Payslip>>>,
    events: Arc<Mutex<HashMap<String, Vec<SerializedEvent>>>>,
}

#[async_trait]
impl AggregateRepository<Payslip> for InMemoryPayslipRepository {
    async fn load(&self, aggregate_id: &PayslipId) -> Result<Option<Payslip>, DomainError> {
        let states = self.states.lock().unwrap();
        Ok(states.get(&aggregate_id.to_string()).cloned())
    }

    async fn save(
        &self,
        aggregate: &Payslip,
        events: Vec<PayslipEvent>,
        context: EventContext,
    ) -> Result<Vec<EventEnvelope<Payslip>>, DomainError> {
        // 简化实现，直接保存状态
        let mut states = self.states.lock().unwrap();
        states.insert(aggregate.id().to_string(), aggregate.clone());

        let envelopes = events
            .into_iter()
            .map(|e| EventEnvelope::new(aggregate.id(), e, context.clone()))
            .collect();
        Ok(envelopes)
    }
}
```

### 接口层：验证参数，调用应用层

```rust
// interface/api/handlers.rs

use crate::application::handlers::CreatePayslipHandler;
use axum::{Json, extract::State};

pub async fn create_payslip(
    State(handler): State<Arc<CreatePayslipHandler>>,
    Json(request): Json<CreatePayslipRequest>,
) -> Result<Json<CreatePayslipResponse>, ApiError> {
    let cmd = CreatePayslipCommand::from(request);
    let id = handler.handle(cmd).await?;
    Ok(Json(CreatePayslipResponse { id: id.to_string() }))
}
```

### 依赖注入：在启动时组装一切

```rust
// main.rs

#[tokio::main]
async fn main() -> Result<()> {
    // 创建基础设施
    let pg_pool = Arc::new(PgPool::connect(&database_url).await?);
    let event_repo = Arc::new(PgEventRepository::new(pg_pool.clone()));

    // 创建适配器（实现端口 trait）
    let payslip_repo = Arc::new(PgPayslipRepository::new(pg_pool.clone(), event_repo.clone()));
    let payslip_checker: Arc<dyn PayslipChecker> = Arc::new(PgPayslipChecker::new(pg_pool.clone()));
    let event_publisher: Arc<dyn EventPublisher> = Arc::new(KafkaEventPublisher::new(&kafka_config)?);

    // 创建 AggregateRoot（编排器）
    let payslip_root = Arc::new(AggregateRoot::new(payslip_repo));

    // 创建应用服务（使用 bon::Builder 注入依赖）
    let payslip_service = Arc::new(
        PayslipService::builder()
            .payslip_root(payslip_root)
            .payslip_checker(payslip_checker)
            .event_publisher(event_publisher)
            .build()
    );

    // 创建 HTTP 路由
    let app = Router::new()
        .route("/payslips", post(create_payslip))
        .with_state(payslip_service);

    // 启动服务器
    let listener = tokio::net::TcpListener::bind(&addr).await?;
    axum::serve(listener, app).await?;

    Ok(())
}
```

测试时可以轻松替换为内存实现：

```rust
#[tokio::test]
async fn test_create_payslip() -> Result<()> {
    // 使用内存实现，不需要真实数据库
    let repo = Arc::new(InMemoryPayslipRepository::default());
    let checker = Arc::new(InMemoryPayslipChecker::default());
    let publisher = Arc::new(InMemoryEventPublisher::default());

    let payslip_root = Arc::new(AggregateRoot::new(repo));
    let service = PayslipService::builder()
        .payslip_root(payslip_root)
        .payslip_checker(checker)
        .event_publisher(publisher)
        .build();

    let cmd = CreatePayslipCommand {
        employee_id: EmployeeId::new(),
        pay_period: PayPeriod::new(2025, 1),
        base_salary: Money::new(dec!(10000)),
    };
    let ctx = AppContext::default();

    let result = service.handle(&ctx, cmd).await;
    assert!(result.is_ok());

    Ok(())
}
```

## 三种架构的对比

| 方面 | 六边形 | 洋葱 | Clean |
|------|--------|------|-------|
| 提出时间 | 2005 | 2008 | 2012 |
| 核心概念 | 端口与适配器 | 同心圆分层 | 依赖规则 |
| 强调重点 | 外部依赖隔离 | 层次清晰 | 业务规则独立 |
| 可视化 | 六边形 | 洋葱 | 同心圆 |

**本质上是同一件事**：让业务逻辑不依赖外部细节。

## 依赖倒置的代价

这种架构不是免费的。

### 代价 1：代码量增加

需要定义接口（trait）、实现适配器、依赖注入...

一个简单的 CRUD，代码量会明显增加（具体倍数因项目而异）。

### 代价 2：间接层增多

调用链变长：Handler → Service → Repository trait → Repository impl → Database

调试时需要跳转更多文件。

### 代价 3：学习曲线

团队需要理解依赖倒置、接口隔离等概念。

### 什么时候值得？

**值得**：
- 业务逻辑复杂
- 需要频繁替换外部依赖
- 需要高测试覆盖率
- 长期维护的项目

**不值得**：
- 简单 CRUD
- 快速原型
- 一次性脚本
- 外部依赖稳定不变

## 常见问题

### Q：小项目也要这样分层吗？

**A：不需要。**

过度设计和设计不足一样有害。

小项目可以简化：
- 不用定义接口，直接用具体类型
- 不用严格分层，放在同一个模块也行
- 等复杂度增加了再重构

### Q：如何判断依赖方向是否正确？

**A：问自己：「如果换掉 X，需要改领域层吗？」**

- 换数据库 → 不应该改领域层 ✅
- 换 Web 框架 → 不应该改领域层 ✅
- 换消息队列 → 不应该改领域层 ✅

如果需要改，说明依赖方向错了。

### Q：领域层可以用第三方库吗？

**A：看情况。**

- `uuid`：可以，这是通用工具
- `serde`：可以，序列化是通用需求
- `sqlx`：不行，这是数据库细节
- `axum`：不行，这是 Web 框架细节

原则：**无副作用的纯计算库可以用，有 I/O 的库需要谨慎评估。** 例如日志库通常可以接受，但数据库驱动应该在基础设施层。

## 总结

**六边形/洋葱/Clean 架构的核心价值**：

> **框架是细节，数据库是细节，Web 是细节。业务规则才是核心。**

**如何实现**：
1. 领域层定义端口（接口）
2. 基础设施层实现适配器
3. 依赖方向：外层依赖内层
4. 在启动时通过依赖注入组装

**核心教训**：

| 传统分层 | 依赖倒置 |
|---------|---------|
| 领域依赖数据库 | 数据库依赖领域 |
| 换数据库改业务代码 | 换数据库只改适配器 |
| 测试需要真实数据库 | 测试用内存实现 |

下一篇，我们来看「单体的边界」——当你的单体足够大，什么时候该拆？怎么拆？

---

*上一篇：[DDD 觉醒：让代码说业务的语言](/posts/architecture-evolution-4-ddd/)*

*下一篇：[单体的边界：何时该拆？](/posts/architecture-evolution-6-modular-monolith/)*

*本系列：*
1. *[混沌时代：当代码没有架构](/posts/architecture-evolution-1-chaos/)*
2. *[MVC 启蒙：框架带来的秩序](/posts/architecture-evolution-2-mvc/)*
3. *[Context 之道：从技术分层到业务分层](/posts/architecture-evolution-3-context/)*
4. *[DDD 觉醒：让代码说业务的语言](/posts/architecture-evolution-4-ddd/)*
5. *边界的艺术：六边形与洋葱（本篇）*
6. *[单体的边界：何时该拆？](/posts/architecture-evolution-6-modular-monolith/)*
7. *[微服务与云原生：分布式的代价](/posts/architecture-evolution-7-microservices/)*
8. *[没有银弹：架构决策的本质](/posts/architecture-evolution-8-no-silver-bullet/)*
