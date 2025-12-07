---
title: "架构演进史（六）：单体的边界，何时该拆？"
date: 2025-12-08T16:50:00+08:00
draft: false
tags: ["架构", "单体", "模块化单体", "微服务", "重构"]
categories: ["技术"]
description: "「我们要上微服务！」技术总监在周会上宣布。我看了看我们 5 人的团队，又看了看那个 10 万行的单体应用，心里默默叹了口气。"
series: ["架构演进史"]
---

「我们要上微服务！」技术总监在周会上宣布。我看了看我们 5 人的团队，又看了看那个 10 万行的单体应用，心里默默叹了口气。

<!--more-->

## 前情回顾

在前五篇中，我们构建了一个架构良好的单体应用：

- **MVC** 带来了技术分层
- **Context** 带来了业务分层
- **DDD** 带来了领域建模
- **六边形架构** 带来了依赖倒置

现在，我们有了一个结构清晰、测试良好的单体应用。

但问题来了：**这个单体能一直用下去吗？什么时候需要拆？**

## 单体的污名

不知从何时起，「单体」似乎成了贬义词：

- 「我们是单体架构」= 「我们很 Low」
- 「我们要上微服务」= 「我们很先进」

**这种观点值得商榷。**

让我们看看一些知名公司的架构选择：

| 公司 | 架构 | 规模 |
|------|------|------|
| Shopify | 单体（模块化） | 每秒 80,000+ 请求 |
| Basecamp | 单体 | 数百万用户 |
| StackOverflow | 单体 | 全球 Top 50 网站 |
| GitHub（早期） | 单体 | 百万开发者 |

**单体不是问题，大泥球才是问题。**

## 什么时候不该拆？

在讨论「何时该拆」之前，先说说「何时不该拆」。

### 情况 1：「单体太慢了」

**错误诊断。**

单体慢，通常是因为：
- 数据库查询没优化
- 没有缓存
- 代码算法效率低
- 服务器配置不够

这些问题，拆成微服务**一个都不会解决**。反而会增加网络延迟。

**正确做法**：先优化，后考虑拆分。

### 情况 2：「代码太乱了」

**更错误的诊断。**

如果你的单体是大泥球，拆成微服务只会变成**分布式大泥球**。

问题会变得更糟：
- 原来改一个文件，现在改三个服务
- 原来调试一个进程，现在要追踪十个
- 原来事务回滚很简单，现在要处理分布式事务

**正确做法**：先重构，划清模块边界，再考虑拆分。

### 情况 3：「微服务是趋势」

**最错误的诊断。**

技术选型应该基于**实际约束**，不是行业热度。

问自己：
- 我们的团队有多大？5 人团队管 20 个微服务？
- 我们的运维能力如何？有 K8s 经验吗？
- 我们的业务有多复杂？真的需要独立扩展吗？

**正确做法**：根据团队规模和业务需求决定，不是根据 Hype Cycle。

## 什么时候该拆？

拆分单体的合理理由：

### 理由 1：团队规模扩大

**康威定律**：系统设计反映组织结构。

当团队从 5 人变成 50 人，单体会遇到问题：
- 代码冲突频繁
- 部署需要协调
- 每个人都要了解整个系统

**信号**：
- 合并代码时经常冲突
- 发布需要「锁仓」协调
- 新人上手时间超过 3 个月

### 理由 2：独立扩展需求

如果系统中有明显的热点：
- 订单系统需要 10 台服务器
- 用户系统只需要 2 台服务器
- 报表系统需要特殊的硬件配置

单体意味着**所有功能一起扩展**，资源浪费。

### 理由 3：独立发布需求

不同业务有不同的发布节奏：
- 营销活动每天发布
- 核心交易一周发布一次
- 财务系统一个月发布一次

单体意味着**所有功能一起发布**，高频影响低频。

### 理由 4：技术栈多样性

不同业务有不同的技术需求：
- 实时计算用 Rust
- 机器学习用 Python
- 前台服务用 Node.js

单体通常意味着**单一技术栈**。

### 决策框架

```text
                     是
是否有扩展/发布瓶颈？──────────────► 考虑拆分
         │
         │ 否
         ▼
                     是
团队是否超过 10 人？ ──────────────► 考虑拆分
         │
         │ 否
         ▼
                     是
代码是否可以独立修改？─────────────► 保持单体
         │
         │ 否
         ▼
    先重构单体，再决定
```

## 模块化单体：最佳实践

在拆成微服务之前，有一个中间方案：**模块化单体（Modular Monolith）**。

### 什么是模块化单体？

> **单体部署，模块化开发。** 代码在一个进程里，但内部有清晰的模块边界。

```text
┌─────────────────────────────────────────────────┐
│                    单体应用                      │
│                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────┐  │
│  │   Accounts  │  │   Orders    │  │ Catalog │  │
│  │   Module    │  │   Module    │  │ Module  │  │
│  └──────┬──────┘  └──────┬──────┘  └────┬────┘  │
│         │                │              │       │
│         └────────────────┼──────────────┘       │
│                          │                      │
│                  ┌───────▼───────┐              │
│                  │  Shared Core  │              │
│                  └───────────────┘              │
└─────────────────────────────────────────────────┘
```

### 模块化单体的规则

**规则 1：模块间只能通过公开 API 通信**

```rust
// ❌ 错误：直接访问另一个模块的内部
mod orders {
    pub fn create_order(user_id: UserId) {
        // 直接访问 accounts 模块的内部
        let user = crate::accounts::repository::find_user(user_id);
    }
}

// ✅ 正确：通过公开 API 访问
mod orders {
    pub fn create_order(user_id: UserId, accounts: &dyn AccountsApi) {
        // 通过接口访问
        let user = accounts.get_user(user_id)?;
    }
}
```

**规则 2：模块间不共享数据库表**

```sql
-- ❌ 错误：orders 模块直接 JOIN accounts 的表
SELECT o.*, u.name
FROM orders.orders o
JOIN accounts.users u ON o.user_id = u.id;

-- ✅ 正确：orders 存储必要的冗余数据，或通过 API 查询
SELECT o.*, o.user_name
FROM orders.orders o;
```

**规则 3：模块可以异步通信**

即使在单体内，也可以使用事件驱动：

```rust
// 订单模块发布事件
pub async fn create_order(&self, cmd: CreateOrderCommand) -> Result<OrderId> {
    let order = Order::create(cmd)?;
    self.repository.save(&order).await?;

    // 发布事件（进程内）
    self.event_bus.publish(OrderCreated {
        order_id: order.id,
        user_id: order.user_id,
        total: order.total,
    }).await?;

    Ok(order.id)
}

// 库存模块订阅事件
pub async fn handle_order_created(&self, event: OrderCreated) -> Result<()> {
    for item in event.items {
        self.decrease_stock(item.product_id, item.quantity).await?;
    }
    Ok(())
}
```

**为什么这样做？**

- **现在**：事件在进程内同步处理
- **将来**：如果拆分，事件通过消息队列异步处理

代码改动最小。

### 目录结构示例

```text
src/
├── modules/
│   ├── accounts/           # 用户模块
│   │   ├── mod.rs
│   │   ├── api.rs          # 公开 API
│   │   ├── domain/
│   │   ├── application/
│   │   └── infrastructure/
│   │
│   ├── orders/             # 订单模块
│   │   ├── mod.rs
│   │   ├── api.rs          # 公开 API
│   │   ├── domain/
│   │   ├── application/
│   │   └── infrastructure/
│   │
│   └── catalog/            # 商品模块
│       ├── mod.rs
│       ├── api.rs
│       └── ...
│
├── shared/                 # 共享内核
│   ├── events.rs           # 领域事件定义
│   ├── types.rs            # 共享类型
│   └── utils.rs
│
└── main.rs                 # 组装和启动
```

### 模块的公开 API

每个模块只暴露必要的接口：

```rust
// modules/accounts/api.rs

/// 账户模块的公开 API
#[async_trait]
pub trait AccountsApi: Send + Sync {
    /// 获取用户信息
    async fn get_user(&self, id: UserId) -> Result<Option<UserDto>>;

    /// 检查用户是否活跃
    async fn is_user_active(&self, id: UserId) -> Result<bool>;

    /// 验证用户权限
    async fn has_permission(&self, id: UserId, permission: Permission) -> Result<bool>;
}

// 实现
pub struct AccountsModule {
    repository: Arc<dyn UserRepository>,
}

#[async_trait]
impl AccountsApi for AccountsModule {
    async fn get_user(&self, id: UserId) -> Result<Option<UserDto>> {
        let user = self.repository.find_by_id(&id).await?;
        Ok(user.map(|u| UserDto::from(u)))
    }

    // ...
}
```

### 模块间的依赖

```rust
// modules/orders/api.rs

pub struct OrdersModule {
    repository: Arc<dyn OrderRepository>,
    accounts: Arc<dyn AccountsApi>,  // 依赖 accounts 的接口
    catalog: Arc<dyn CatalogApi>,    // 依赖 catalog 的接口
    event_bus: Arc<dyn EventBus>,
}

impl OrdersModule {
    pub async fn create_order(&self, cmd: CreateOrderCommand) -> Result<OrderId> {
        // 通过接口调用其他模块
        let user = self.accounts.get_user(cmd.user_id).await?
            .ok_or(OrderError::UserNotFound)?;

        let product = self.catalog.get_product(cmd.product_id).await?
            .ok_or(OrderError::ProductNotFound)?;

        // 业务逻辑...
        let order = Order::create(user.id, product, cmd.quantity)?;
        self.repository.save(&order).await?;

        Ok(order.id)
    }
}
```

## 从模块化单体到微服务

模块化单体的好处：**拆分变得简单**。

### 拆分前

```text
单体应用（一个进程）
├── Accounts Module
├── Orders Module
└── Catalog Module
```

### 拆分后

```text
Accounts Service（独立进程）
Orders Service（独立进程）
Catalog Service（独立进程）
```

**改动点**：

| 方面 | 拆分前 | 拆分后 |
|------|--------|--------|
| 模块间通信 | 进程内调用 | HTTP/gRPC |
| 事件传递 | 进程内事件总线 | Kafka/RabbitMQ |
| 数据库 | 共享数据库（不同 Schema） | 独立数据库 |
| 部署 | 单个二进制 | 多个容器 |

**因为模块边界已经清晰，拆分只是把「进程内通信」换成「进程间通信」。**

### 渐进式拆分

不需要一次性拆完。可以：

1. **第一步**：把访问量最大的模块拆出去
2. **第二步**：把需要独立扩展的模块拆出去
3. **第三步**：剩下的继续保持单体

```text
                    ┌──────────────┐
                    │   Gateway    │
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │   Orders    │ │  Accounts   │ │  Catalog    │
    │  Service    │ │  Service    │ │   单体内    │
    │  (独立)     │ │  (独立)     │ │             │
    └─────────────┘ └─────────────┘ └─────────────┘
```

这种「混合架构」是完全合理的。

## 实战：识别模块边界

如何在现有代码中识别模块边界？

### 方法 1：从业务流程出发

画出核心业务流程，看哪些步骤可以独立：

```text
用户下单流程：
1. 用户登录 ─────────────────► Accounts
2. 浏览商品 ─────────────────► Catalog
3. 添加购物车 ───────────────► Cart
4. 提交订单 ─────────────────► Orders
5. 支付 ─────────────────────► Payments
6. 发货 ─────────────────────► Shipping
```

每个步骤可能是一个模块。

### 方法 2：从数据出发

看哪些数据经常一起访问，哪些很少一起访问：

```text
users ──────────┐
credentials ────┤ 经常一起访问 → Accounts 模块
permissions ────┘

products ───────┐
categories ─────┤ 经常一起访问 → Catalog 模块
inventory ──────┘

orders ─────────┐
order_items ────┤ 经常一起访问 → Orders 模块
payments ───────┘
```

### 方法 3：从团队出发

问：「谁负责这块代码？」

```text
Team A 负责用户系统 → Accounts 模块
Team B 负责商品系统 → Catalog 模块
Team C 负责交易系统 → Orders + Payments 模块
```

如果一个模块由多个团队维护，可能需要拆分。

## 常见陷阱

### 陷阱 1：数据库层面没拆干净

```sql
-- 模块内部的查询
SELECT * FROM orders WHERE user_id = ?;

-- 跨模块的 JOIN（危险！）
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id;
```

如果模块间还在 JOIN，拆成微服务时会很痛苦。

**解决方案**：
- 冗余必要数据（`orders` 表存 `user_name`）
- 通过 API 查询后在应用层组装

### 陷阱 2：循环依赖

```text
Accounts ─────► Orders
    ▲               │
    └───────────────┘
```

账户模块依赖订单模块，订单模块又依赖账户模块。

**解决方案**：
- 引入事件：一方发事件，另一方订阅
- 抽取共享服务：把互相依赖的部分抽出来

### 陷阱 3：过度拆分

5 个人的团队，拆了 20 个模块。

结果：
- 每个人负责 4 个模块，心智负担重
- 模块间通信复杂度高
- 没有足够的人维护基础设施

**原则**：模块数量不应该超过团队人数的 2 倍。

## 总结

**单体 vs 微服务的本质**：

| 方面 | 单体 | 微服务 |
|------|------|--------|
| 复杂度位置 | 代码内部 | 网络/基础设施 |
| 适合团队 | 小团队（< 10 人） | 大团队（> 30 人） |
| 事务处理 | 简单（本地事务） | 复杂（分布式事务） |
| 部署 | 简单（一个包） | 复杂（多个服务） |
| 扩展 | 整体扩展 | 独立扩展 |

**模块化单体的价值**：

> **享受单体的简单，又保留拆分的可能。**

- 部署简单（一个进程）
- 模块边界清晰（易于理解和维护）
- 未来可拆（边界已经划好）

**决策建议**：

```text
团队 < 10 人 && 业务清晰 → 模块化单体
团队 > 30 人 || 需要独立扩展 → 考虑微服务
介于两者之间 → 从模块化单体开始，按需拆分
```

**核心教训**：

> **如果你的单体是大泥球，拆成微服务只会变成分布式大泥球。**

先重构，划清边界，再考虑拆分。

下一篇，我们来看微服务和云原生——当你决定拆分后，会面临哪些新的挑战。

---

*上一篇：[边界的艺术：六边形与洋葱](/posts/architecture-evolution-5-hexagonal/)*

*下一篇：[微服务与云原生：分布式的代价](/posts/architecture-evolution-7-microservices/)*

*本系列：*
1. *[混沌时代：当代码没有架构](/posts/architecture-evolution-1-chaos/)*
2. *[MVC 启蒙：框架带来的秩序](/posts/architecture-evolution-2-mvc/)*
3. *[Context 之道：从技术分层到业务分层](/posts/architecture-evolution-3-context/)*
4. *[DDD 觉醒：让代码说业务的语言](/posts/architecture-evolution-4-ddd/)*
5. *[边界的艺术：六边形与洋葱](/posts/architecture-evolution-5-hexagonal/)*
6. *单体的边界：何时该拆？（本篇）*
7. *[微服务与云原生：分布式的代价](/posts/architecture-evolution-7-microservices/)*
8. *[没有银弹：架构决策的本质](/posts/architecture-evolution-8-no-silver-bullet/)*
