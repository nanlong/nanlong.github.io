---
title: "模式不是套路（五）：状态流转——订单不只是 CRUD"
date: 2025-12-10T10:40:00+08:00
draft: false
tags: ["Rust", "设计模式", "状态机", "状态模式", "命令模式", "备忘录模式"]
categories: ["技术"]
description: "订单状态用字符串存，状态转换靠 if-else？当业务规则越来越复杂，这套玩法迟早崩盘。看状态模式、命令模式、备忘录模式如何驯服状态机这头野兽。"
series: ["模式不是套路"]
---

订单状态用字符串存，状态转换靠 if-else？当业务规则越来越复杂，这套玩法迟早崩盘。看状态模式、命令模式、备忘录模式如何驯服状态机这头野兽。

<!--more-->

## 问题：一个失控的订单系统

电商订单，看起来很简单：

```rust
struct Order {
    id: String,
    status: String,  // "created", "paid", "shipped", "delivered", "cancelled"
    // ...
}

impl Order {
    fn pay(&mut self) {
        self.status = "paid".to_string();
    }

    fn ship(&mut self) {
        self.status = "shipped".to_string();
    }

    fn cancel(&mut self) {
        self.status = "cancelled".to_string();
    }
}
```

然后业务规则来了：

```text
规则 1：只有"已创建"状态可以支付
规则 2：只有"已支付"状态可以发货
规则 3：只有"已创建"或"已支付"状态可以取消
规则 4：取消已支付订单需要退款
规则 5：发货后 7 天内可以申请退货
规则 6：退货需要先寄回商品，确认收到后才能退款
规则 7：部分退款的订单状态是...
```

代码变成了这样：

```rust
impl Order {
    fn pay(&mut self) -> Result<(), OrderError> {
        if self.status != "created" {
            return Err(OrderError::InvalidTransition {
                from: self.status.clone(),
                action: "pay".to_string(),
            });
        }
        self.status = "paid".to_string();
        Ok(())
    }

    fn ship(&mut self) -> Result<(), OrderError> {
        if self.status != "paid" {
            return Err(OrderError::InvalidTransition {
                from: self.status.clone(),
                action: "ship".to_string(),
            });
        }
        self.status = "shipped".to_string();
        Ok(())
    }

    fn cancel(&mut self) -> Result<(), OrderError> {
        match self.status.as_str() {
            "created" => {
                self.status = "cancelled".to_string();
                Ok(())
            }
            "paid" => {
                // 需要退款
                self.process_refund()?;
                self.status = "cancelled".to_string();
                Ok(())
            }
            "shipped" => {
                // 发货后不能直接取消，需要走退货流程
                Err(OrderError::CannotCancel("Order already shipped, please use return flow".to_string()))
            }
            _ => Err(OrderError::InvalidTransition {
                from: self.status.clone(),
                action: "cancel".to_string(),
            }),
        }
    }

    // 还有十几个类似的方法...
}
```

问题显而易见：

```text
1. 状态是字符串，拼错了编译器不会报错
2. 状态转换规则散落在各个方法中
3. 新增状态需要改很多地方
4. 无法在编译期保证状态转换的正确性
5. 测试困难——需要构造各种状态组合
```

## 第一次改进：用枚举替代字符串

最简单的改进：**用 enum 定义状态**。

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum OrderStatus {
    Created,
    Paid { paid_at: DateTime<Utc>, amount: Decimal },
    Shipped { shipped_at: DateTime<Utc>, tracking_number: String },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String, refunded: bool },
    Returning { return_reason: String, return_tracking: Option<String> },
    Returned { refund_amount: Decimal },
}

struct Order {
    id: OrderId,
    status: OrderStatus,
    items: Vec<OrderItem>,
    created_at: DateTime<Utc>,
}
```

每个状态可以携带不同的数据。编译器会强制你处理所有状态：

```rust
impl Order {
    fn can_cancel(&self) -> bool {
        matches!(self.status, OrderStatus::Created | OrderStatus::Paid { .. })
    }

    fn next_actions(&self) -> Vec<OrderAction> {
        match &self.status {
            OrderStatus::Created => vec![OrderAction::Pay, OrderAction::Cancel],
            OrderStatus::Paid { .. } => vec![OrderAction::Ship, OrderAction::Cancel],
            OrderStatus::Shipped { .. } => vec![OrderAction::Deliver, OrderAction::Return],
            OrderStatus::Delivered { .. } => vec![OrderAction::Return],
            OrderStatus::Cancelled { .. } => vec![],
            OrderStatus::Returning { .. } => vec![OrderAction::ConfirmReturn],
            OrderStatus::Returned { .. } => vec![],
        }
    }
}
```

如果新增一个状态但忘了处理，编译器会报错：

```rust
// 新增了 PartiallyRefunded 状态
pub enum OrderStatus {
    // ...
    PartiallyRefunded { refund_amount: Decimal },
}

// 编译器报错：non-exhaustive patterns: `PartiallyRefunded { .. }` not covered
fn next_actions(&self) -> Vec<OrderAction> {
    match &self.status {
        // 必须处理新状态
    }
}
```

## 第二次改进：状态模式

**状态模式**的核心是：**让对象在内部状态改变时改变它的行为**。

传统的实现方式是每个状态一个类：

```rust
trait OrderState: Send + Sync {
    fn pay(&self, order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError>;
    fn ship(&self, order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError>;
    fn cancel(&self, order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError>;
    fn deliver(&self, order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError>;

    fn status_name(&self) -> &str;
    fn can_cancel(&self) -> bool { false }
}

struct CreatedState;

impl OrderState for CreatedState {
    fn pay(&self, order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError> {
        // 支付逻辑
        order.paid_at = Some(Utc::now());
        Ok(Box::new(PaidState))
    }

    fn ship(&self, _order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError> {
        Err(OrderError::InvalidTransition("Cannot ship unpaid order".into()))
    }

    fn cancel(&self, _order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError> {
        Ok(Box::new(CancelledState { refunded: false }))
    }

    fn deliver(&self, _order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError> {
        Err(OrderError::InvalidTransition("Cannot deliver unpaid order".into()))
    }

    fn status_name(&self) -> &str { "created" }
    fn can_cancel(&self) -> bool { true }
}

struct PaidState;

impl OrderState for PaidState {
    fn pay(&self, _order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError> {
        Err(OrderError::InvalidTransition("Already paid".into()))
    }

    fn ship(&self, order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError> {
        order.shipped_at = Some(Utc::now());
        Ok(Box::new(ShippedState))
    }

    fn cancel(&self, order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError> {
        // 需要退款
        order.refund()?;
        Ok(Box::new(CancelledState { refunded: true }))
    }

    fn deliver(&self, _order: &mut OrderData) -> Result<Box<dyn OrderState>, OrderError> {
        Err(OrderError::InvalidTransition("Cannot deliver before shipping".into()))
    }

    fn status_name(&self) -> &str { "paid" }
    fn can_cancel(&self) -> bool { true }
}
```

### Rust 的礼物：类型状态模式

Rust 可以在**编译期**强制状态转换规则：

```rust
use std::marker::PhantomData;

// 状态类型（零大小）
struct Created;
struct Paid;
struct Shipped;
struct Delivered;
struct Cancelled;

// 订单泛型，状态作为类型参数
struct Order<S> {
    id: OrderId,
    items: Vec<OrderItem>,
    created_at: DateTime<Utc>,
    _state: PhantomData<S>,
}

// 只有 Created 状态的订单可以支付
impl Order<Created> {
    fn pay(self, payment: Payment) -> Result<Order<Paid>, PaymentError> {
        // 支付逻辑
        Ok(Order {
            id: self.id,
            items: self.items,
            created_at: self.created_at,
            _state: PhantomData,
        })
    }

    fn cancel(self, reason: String) -> Order<Cancelled> {
        Order {
            id: self.id,
            items: self.items,
            created_at: self.created_at,
            _state: PhantomData,
        }
    }
}

// 只有 Paid 状态的订单可以发货
impl Order<Paid> {
    fn ship(self, tracking: String) -> Order<Shipped> {
        Order {
            id: self.id,
            items: self.items,
            created_at: self.created_at,
            _state: PhantomData,
        }
    }

    fn cancel(self, reason: String) -> Result<Order<Cancelled>, RefundError> {
        // 退款逻辑
        Ok(Order {
            id: self.id,
            items: self.items,
            created_at: self.created_at,
            _state: PhantomData,
        })
    }
}

// 只有 Shipped 状态的订单可以确认收货
impl Order<Shipped> {
    fn deliver(self) -> Order<Delivered> {
        Order {
            id: self.id,
            items: self.items,
            created_at: self.created_at,
            _state: PhantomData,
        }
    }
    // 注意：Shipped 状态没有 cancel 方法！
}
```

使用时，编译器会检查状态转换：

```rust
// ✅ 正确的状态流转
let order = Order::<Created>::new(items);
let order = order.pay(payment)?;      // Created -> Paid
let order = order.ship(tracking);      // Paid -> Shipped
let order = order.deliver();           // Shipped -> Delivered

// ❌ 编译错误：Order<Created> 没有 ship 方法
let order = Order::<Created>::new(items);
let order = order.ship(tracking);  // 编译失败！

// ❌ 编译错误：Order<Shipped> 没有 cancel 方法
let order = Order::<Shipped>::new(items);
let order = order.cancel("changed mind");  // 编译失败！
```

**编译期就能发现非法的状态转换**，这是其他语言很难做到的。

### 类型状态的局限

类型状态很强大，但有局限：

```text
1. 状态类型必须在编译期确定
   - 不能根据运行时数据决定状态类型
   - 数据库加载的订单，状态是运行时确定的

2. 不同状态的订单是不同类型
   - Vec<Order<Created>> 和 Vec<Order<Paid>> 不能混用
   - 需要用 enum 包装或 trait object
```

实践中，类型状态适合**内部状态机**，而**持久化的业务对象**用 enum 更实际：

```rust
// 内部状态机：编译期检查
struct ConnectionStateMachine<S> { /* 状态数据 */ }

// 业务对象：运行时状态
struct Order {
    status: OrderStatus,  // enum
    // ...
}
```

## 第三次改进：命令模式

业务继续复杂化：

```text
需求 1：支持操作撤销（订单取消后可以恢复）
需求 2：支持操作重做
需求 3：记录完整的操作历史
需求 4：支持批量操作
```

**命令模式**的核心是：**将请求封装成对象，从而可以参数化、排队、记录日志、支持撤销**。

```rust
// 命令 trait
trait OrderCommand: Send + Sync {
    fn execute(&self, order: &mut Order) -> Result<(), OrderError>;
    fn undo(&self, order: &mut Order) -> Result<(), OrderError>;
    fn description(&self) -> String;
}

// 支付命令
struct PayCommand {
    payment_id: String,
    amount: Decimal,
    paid_at: DateTime<Utc>,
}

impl PayCommand {
    fn new(payment_id: String, amount: Decimal) -> Self {
        Self {
            payment_id,
            amount,
            paid_at: Utc::now(),
        }
    }
}

impl OrderCommand for PayCommand {
    fn execute(&self, order: &mut Order) -> Result<(), OrderError> {
        if !matches!(order.status, OrderStatus::Created) {
            return Err(OrderError::InvalidTransition("Can only pay created orders".into()));
        }

        order.status = OrderStatus::Paid {
            paid_at: self.paid_at,
            amount: self.amount,
        };
        order.payment_id = Some(self.payment_id.clone());

        Ok(())
    }

    fn undo(&self, order: &mut Order) -> Result<(), OrderError> {
        // 撤销支付 = 退款 + 恢复状态
        if !matches!(order.status, OrderStatus::Paid { .. }) {
            return Err(OrderError::InvalidTransition("Can only undo paid orders".into()));
        }

        // 调用退款服务
        // refund_service.refund(&self.payment_id)?;

        order.status = OrderStatus::Created;
        order.payment_id = None;

        Ok(())
    }

    fn description(&self) -> String {
        format!("Pay {} (payment_id: {})", self.amount, self.payment_id)
    }
}

// 发货命令
struct ShipCommand {
    tracking_number: String,
    shipped_at: DateTime<Utc>,
}

impl ShipCommand {
    fn new(tracking_number: String) -> Self {
        Self {
            tracking_number,
            shipped_at: Utc::now(),
        }
    }
}

impl OrderCommand for ShipCommand {
    fn execute(&self, order: &mut Order) -> Result<(), OrderError> {
        if !matches!(order.status, OrderStatus::Paid { .. }) {
            return Err(OrderError::InvalidTransition("Can only ship paid orders".into()));
        }

        order.status = OrderStatus::Shipped {
            shipped_at: self.shipped_at,
            tracking_number: self.tracking_number.clone(),
        };

        Ok(())
    }

    fn undo(&self, order: &mut Order) -> Result<(), OrderError> {
        // 撤销发货（比如物流拦截成功）
        if let OrderStatus::Shipped { .. } = &order.status {
            // 从之前的状态恢复
            order.status = OrderStatus::Paid {
                paid_at: Utc::now(), // 需要从历史记录恢复
                amount: Decimal::ZERO,
            };
            Ok(())
        } else {
            Err(OrderError::InvalidTransition("Can only undo shipped orders".into()))
        }
    }

    fn description(&self) -> String {
        format!("Ship with tracking {}", self.tracking_number)
    }
}
```

### 命令执行器

```rust
struct OrderCommandExecutor {
    history: Vec<Box<dyn OrderCommand>>,
    undo_stack: Vec<Box<dyn OrderCommand>>,
}

impl OrderCommandExecutor {
    fn new() -> Self {
        Self {
            history: Vec::new(),
            undo_stack: Vec::new(),
        }
    }

    fn execute(&mut self, command: Box<dyn OrderCommand>, order: &mut Order) -> Result<(), OrderError> {
        command.execute(order)?;
        self.history.push(command);
        self.undo_stack.clear();  // 执行新命令后清空重做栈
        Ok(())
    }

    fn undo(&mut self, order: &mut Order) -> Result<(), OrderError> {
        let command = self.history.pop()
            .ok_or(OrderError::NothingToUndo)?;

        command.undo(order)?;
        self.undo_stack.push(command);

        Ok(())
    }

    fn redo(&mut self, order: &mut Order) -> Result<(), OrderError> {
        let command = self.undo_stack.pop()
            .ok_or(OrderError::NothingToRedo)?;

        command.execute(order)?;
        self.history.push(command);

        Ok(())
    }

    fn history(&self) -> impl Iterator<Item = &dyn OrderCommand> {
        self.history.iter().map(|c| c.as_ref())
    }
}
```

### Rust 的礼物：用 enum 实现命令

如果命令集合是固定的，用 enum 更简洁：

```rust
#[derive(Debug, Clone)]
pub enum OrderCommand {
    Pay { payment_id: String, amount: Decimal },
    Ship { tracking_number: String },
    Deliver,
    Cancel { reason: String },
    Return { return_reason: String },
}

impl OrderCommand {
    pub fn execute(&self, order: &mut Order) -> Result<OrderEvent, OrderError> {
        match self {
            OrderCommand::Pay { payment_id, amount } => {
                // 支付逻辑
                Ok(OrderEvent::Paid {
                    payment_id: payment_id.clone(),
                    amount: *amount,
                    paid_at: Utc::now(),
                })
            }
            OrderCommand::Ship { tracking_number } => {
                // 发货逻辑
                Ok(OrderEvent::Shipped {
                    tracking_number: tracking_number.clone(),
                    shipped_at: Utc::now(),
                })
            }
            // ...
        }
    }
}
```

## 第四次改进：备忘录模式

需要**保存和恢复订单的完整状态**，比如：

```text
- 回滚到任意历史版本
- 查看某个时间点的订单状态
- 审计追踪
```

**备忘录模式**的核心是：**在不破坏封装性的前提下，捕获对象的内部状态并保存**。

```rust
// 备忘录：订单状态的快照
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderMemento {
    pub version: u64,
    pub created_at: DateTime<Utc>,
    pub status: OrderStatus,
    pub items: Vec<OrderItem>,
    pub payment_id: Option<String>,
    pub shipping_info: Option<ShippingInfo>,
    // 不包含瞬时状态
}

impl Order {
    // 创建快照
    pub fn create_memento(&self) -> OrderMemento {
        OrderMemento {
            version: self.version,
            created_at: Utc::now(),
            status: self.status.clone(),
            items: self.items.clone(),
            payment_id: self.payment_id.clone(),
            shipping_info: self.shipping_info.clone(),
        }
    }

    // 从快照恢复
    pub fn restore_from_memento(&mut self, memento: &OrderMemento) {
        self.version = memento.version;
        self.status = memento.status.clone();
        self.items = memento.items.clone();
        self.payment_id = memento.payment_id.clone();
        self.shipping_info = memento.shipping_info.clone();
    }
}

// 管理者：管理备忘录的历史
struct OrderHistory {
    snapshots: Vec<OrderMemento>,
    max_snapshots: usize,
}

impl OrderHistory {
    fn new(max_snapshots: usize) -> Self {
        Self {
            snapshots: Vec::new(),
            max_snapshots,
        }
    }

    fn save(&mut self, memento: OrderMemento) {
        self.snapshots.push(memento);

        // 限制历史记录数量
        if self.snapshots.len() > self.max_snapshots {
            self.snapshots.remove(0);
        }
    }

    fn get(&self, version: u64) -> Option<&OrderMemento> {
        self.snapshots.iter().find(|m| m.version == version)
    }

    fn get_at_time(&self, time: DateTime<Utc>) -> Option<&OrderMemento> {
        self.snapshots.iter()
            .filter(|m| m.created_at <= time)
            .last()
    }

    fn latest(&self) -> Option<&OrderMemento> {
        self.snapshots.last()
    }
}
```

### 结合事件溯源

备忘录模式和事件溯源（Event Sourcing）很相似。更进一步，可以只存储事件，按需重建状态：

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum OrderEvent {
    Created { items: Vec<OrderItem> },
    Paid { payment_id: String, amount: Decimal, paid_at: DateTime<Utc> },
    Shipped { tracking_number: String, shipped_at: DateTime<Utc> },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String, refunded: bool },
}

impl Order {
    // 应用单个事件
    pub fn apply(&mut self, event: &OrderEvent) {
        match event {
            OrderEvent::Created { items } => {
                self.items = items.clone();
                self.status = OrderStatus::Created;
            }
            OrderEvent::Paid { payment_id, amount, paid_at } => {
                self.payment_id = Some(payment_id.clone());
                self.status = OrderStatus::Paid {
                    paid_at: *paid_at,
                    amount: *amount,
                };
            }
            OrderEvent::Shipped { tracking_number, shipped_at } => {
                self.status = OrderStatus::Shipped {
                    shipped_at: *shipped_at,
                    tracking_number: tracking_number.clone(),
                };
            }
            OrderEvent::Delivered { delivered_at } => {
                self.status = OrderStatus::Delivered {
                    delivered_at: *delivered_at,
                };
            }
            OrderEvent::Cancelled { reason, refunded } => {
                self.status = OrderStatus::Cancelled {
                    reason: reason.clone(),
                    refunded: *refunded,
                };
            }
        }
        self.version += 1;
    }

    // 从事件流重建
    pub fn from_events(events: &[OrderEvent]) -> Self {
        let mut order = Order::default();
        for event in events {
            order.apply(event);
        }
        order
    }

    // 重建到指定版本
    pub fn rebuild_to_version(events: &[OrderEvent], target_version: u64) -> Self {
        let mut order = Order::default();
        for event in events.iter().take(target_version as usize) {
            order.apply(event);
        }
        order
    }
}
```

## 陷阱：别掉进这些坑

### 陷阱一：状态爆炸

```rust
// ❌ 状态太细，组合爆炸
enum OrderStatus {
    CreatedPendingPayment,
    CreatedPaymentProcessing,
    CreatedPaymentFailed,
    PaidPendingShipment,
    PaidShipmentProcessing,
    // ... 几十种状态
}
```

解决方案：**分离正交的状态维度**

```rust
// ✅ 分离核心状态和子状态
struct Order {
    status: OrderStatus,            // 核心状态：Created, Paid, Shipped...
    payment_status: PaymentStatus,  // 支付子状态：Pending, Processing, Completed, Failed
    shipping_status: ShippingStatus, // 物流子状态
}

enum OrderStatus {
    Created,
    Paid,
    Shipped,
    Delivered,
    Cancelled,
}

enum PaymentStatus {
    NotStarted,
    Pending,
    Processing,
    Completed,
    Failed { reason: String },
}
```

### 陷阱二：命令的副作用

```rust
// ❌ 命令执行有外部副作用，撤销很难
impl OrderCommand for PayCommand {
    fn execute(&self, order: &mut Order) -> Result<(), OrderError> {
        // 调用外部支付网关
        payment_gateway.charge(self.amount)?;  // 外部副作用！
        order.status = OrderStatus::Paid {
            paid_at: Utc::now(),
            amount: self.amount,
        };
        Ok(())
    }

    fn undo(&self, order: &mut Order) -> Result<(), OrderError> {
        // 退款可能失败，怎么办？
        payment_gateway.refund(self.payment_id)?;  // 可能失败！
        order.status = OrderStatus::Created;
        Ok(())
    }
}
```

解决方案：**分离命令和副作用**

```rust
// ✅ 命令只改变内部状态，返回需要执行的副作用
enum SideEffect {
    ChargePayment { payment_id: String, amount: Decimal },
    RefundPayment { payment_id: String },
    SendEmail { to: String, template: String },
    NotifyWarehouse { order_id: OrderId },
}

impl OrderCommand {
    fn execute(&self, order: &mut Order) -> Result<Vec<SideEffect>, OrderError> {
        match self {
            OrderCommand::Pay { payment_id, amount } => {
                order.status = OrderStatus::Paid {
                    paid_at: Utc::now(),
                    amount: *amount,
                };
                Ok(vec![
                    SideEffect::ChargePayment {
                        payment_id: payment_id.clone(),
                        amount: *amount,
                    },
                    SideEffect::SendEmail {
                        to: order.customer_email.clone(),
                        template: "payment_success".to_string(),
                    },
                ])
            }
            // ...
        }
    }
}

// 外部执行副作用
async fn execute_side_effects(effects: Vec<SideEffect>) -> Result<(), Error> {
    for effect in effects {
        match effect {
            SideEffect::ChargePayment { payment_id, amount } => {
                payment_gateway.charge(&payment_id, amount).await?;
            }
            // ...
        }
    }
    Ok(())
}
```

### 陷阱三：备忘录太大

```rust
// ❌ 每次保存完整状态
struct OrderMemento {
    // 包含所有字段的完整副本
    // 如果有大对象（如图片、文件），内存爆炸
}
```

解决方案：

```rust
// ✅ 方案 1：增量快照
struct IncrementalMemento {
    base_version: u64,
    changes: Vec<FieldChange>,
}

// ✅ 方案 2：只存事件，按需重建
struct EventStore {
    events: Vec<(u64, OrderEvent)>,  // (version, event)
}

impl EventStore {
    fn rebuild(&self, target_version: u64) -> Order {
        Order::from_events(
            &self.events.iter()
                .filter(|(v, _)| *v <= target_version)
                .map(|(_, e)| e.clone())
                .collect::<Vec<_>>()
        )
    }
}

// ✅ 方案 3：快照 + 事件
// 定期保存完整快照，快照之后只存事件
struct OptimizedStore {
    snapshots: BTreeMap<u64, OrderMemento>,  // 每 100 个版本一个快照
    events: Vec<(u64, OrderEvent)>,
}

impl OptimizedStore {
    fn rebuild(&self, target_version: u64) -> Order {
        // 找到最近的快照
        let (snapshot_version, snapshot) = self.snapshots
            .range(..=target_version)
            .last()
            .map(|(v, s)| (*v, s.clone()))
            .unwrap_or((0, OrderMemento::default()));

        // 从快照开始，应用后续事件
        let mut order = Order::from_memento(&snapshot);
        for (v, event) in &self.events {
            if *v > snapshot_version && *v <= target_version {
                order.apply(event);
            }
        }
        order
    }
}
```

## 完整示例

```rust
use chrono::{DateTime, Utc};
use rust_decimal::Decimal;
use serde::{Deserialize, Serialize};

// ==================== 核心类型 ====================

pub type OrderId = String;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Order {
    pub id: OrderId,
    pub version: u64,
    pub status: OrderStatus,
    pub items: Vec<OrderItem>,
    pub payment_id: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderItem {
    pub product_id: String,
    pub quantity: u32,
    pub price: Decimal,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum OrderStatus {
    Created,
    Paid { paid_at: DateTime<Utc>, amount: Decimal },
    Shipped { shipped_at: DateTime<Utc>, tracking_number: String },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String, refunded: bool },
}

// ==================== 命令 ====================

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum OrderCommand {
    Pay { payment_id: String, amount: Decimal },
    Ship { tracking_number: String },
    Deliver,
    Cancel { reason: String },
}

// ==================== 事件 ====================

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum OrderEvent {
    Created { order_id: OrderId, items: Vec<OrderItem> },
    Paid { payment_id: String, amount: Decimal, paid_at: DateTime<Utc> },
    Shipped { tracking_number: String, shipped_at: DateTime<Utc> },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String, refunded: bool },
}

// ==================== 错误 ====================

#[derive(Debug)]
pub enum OrderError {
    InvalidTransition(String),
    ValidationFailed(String),
}

// ==================== Order 实现 ====================

impl Order {
    pub fn new(id: OrderId, items: Vec<OrderItem>) -> Self {
        Self {
            id,
            version: 0,
            status: OrderStatus::Created,
            items,
            payment_id: None,
            created_at: Utc::now(),
        }
    }

    pub fn execute(&mut self, command: OrderCommand) -> Result<OrderEvent, OrderError> {
        let event = match (&self.status, command) {
            // Created -> Paid
            (OrderStatus::Created, OrderCommand::Pay { payment_id, amount }) => {
                OrderEvent::Paid {
                    payment_id,
                    amount,
                    paid_at: Utc::now(),
                }
            }

            // Created -> Cancelled
            (OrderStatus::Created, OrderCommand::Cancel { reason }) => {
                OrderEvent::Cancelled { reason, refunded: false }
            }

            // Paid -> Shipped
            (OrderStatus::Paid { .. }, OrderCommand::Ship { tracking_number }) => {
                OrderEvent::Shipped {
                    tracking_number,
                    shipped_at: Utc::now(),
                }
            }

            // Paid -> Cancelled (需要退款)
            (OrderStatus::Paid { .. }, OrderCommand::Cancel { reason }) => {
                OrderEvent::Cancelled { reason, refunded: true }
            }

            // Shipped -> Delivered
            (OrderStatus::Shipped { .. }, OrderCommand::Deliver) => {
                OrderEvent::Delivered { delivered_at: Utc::now() }
            }

            // 非法转换
            (status, command) => {
                return Err(OrderError::InvalidTransition(
                    format!("Cannot {:?} when status is {:?}", command, status)
                ));
            }
        };

        self.apply(&event);
        Ok(event)
    }

    pub fn apply(&mut self, event: &OrderEvent) {
        match event {
            OrderEvent::Created { items, .. } => {
                self.items = items.clone();
                self.status = OrderStatus::Created;
            }
            OrderEvent::Paid { payment_id, amount, paid_at } => {
                self.payment_id = Some(payment_id.clone());
                self.status = OrderStatus::Paid {
                    paid_at: *paid_at,
                    amount: *amount,
                };
            }
            OrderEvent::Shipped { tracking_number, shipped_at } => {
                self.status = OrderStatus::Shipped {
                    shipped_at: *shipped_at,
                    tracking_number: tracking_number.clone(),
                };
            }
            OrderEvent::Delivered { delivered_at } => {
                self.status = OrderStatus::Delivered { delivered_at: *delivered_at };
            }
            OrderEvent::Cancelled { reason, refunded } => {
                self.status = OrderStatus::Cancelled {
                    reason: reason.clone(),
                    refunded: *refunded,
                };
            }
        }
        self.version += 1;
    }

    pub fn can_execute(&self, command: &OrderCommand) -> bool {
        match (&self.status, command) {
            (OrderStatus::Created, OrderCommand::Pay { .. }) => true,
            (OrderStatus::Created, OrderCommand::Cancel { .. }) => true,
            (OrderStatus::Paid { .. }, OrderCommand::Ship { .. }) => true,
            (OrderStatus::Paid { .. }, OrderCommand::Cancel { .. }) => true,
            (OrderStatus::Shipped { .. }, OrderCommand::Deliver) => true,
            _ => false,
        }
    }

    pub fn available_commands(&self) -> Vec<&'static str> {
        match &self.status {
            OrderStatus::Created => vec!["pay", "cancel"],
            OrderStatus::Paid { .. } => vec!["ship", "cancel"],
            OrderStatus::Shipped { .. } => vec!["deliver"],
            OrderStatus::Delivered { .. } => vec![],
            OrderStatus::Cancelled { .. } => vec![],
        }
    }
}

// ==================== 事件存储 ====================

pub struct EventStore {
    events: Vec<OrderEvent>,
}

impl EventStore {
    pub fn new() -> Self {
        Self { events: Vec::new() }
    }

    pub fn append(&mut self, event: OrderEvent) {
        self.events.push(event);
    }

    pub fn rebuild(&self) -> Option<Order> {
        if self.events.is_empty() {
            return None;
        }

        let mut order = match &self.events[0] {
            OrderEvent::Created { order_id, items } => {
                Order::new(order_id.clone(), items.clone())
            }
            _ => return None,
        };

        for event in self.events.iter().skip(1) {
            order.apply(event);
        }

        Some(order)
    }

    pub fn rebuild_to_version(&self, target: u64) -> Option<Order> {
        if self.events.is_empty() {
            return None;
        }

        let mut order = match &self.events[0] {
            OrderEvent::Created { order_id, items } => {
                Order::new(order_id.clone(), items.clone())
            }
            _ => return None,
        };

        for event in self.events.iter().skip(1).take(target as usize) {
            order.apply(event);
        }

        Some(order)
    }
}

// ==================== 使用示例 ====================

fn main() {
    let items = vec![
        OrderItem {
            product_id: "PROD-001".to_string(),
            quantity: 2,
            price: Decimal::new(9900, 2),  // 99.00
        },
    ];

    let mut order = Order::new("ORD-001".to_string(), items);
    let mut store = EventStore::new();

    // 记录创建事件
    store.append(OrderEvent::Created {
        order_id: order.id.clone(),
        items: order.items.clone(),
    });

    println!("Available commands: {:?}", order.available_commands());

    // 支付
    let event = order.execute(OrderCommand::Pay {
        payment_id: "PAY-001".to_string(),
        amount: Decimal::new(9900, 2),
    }).unwrap();
    store.append(event);

    println!("After payment: {:?}", order.status);
    println!("Available commands: {:?}", order.available_commands());

    // 发货
    let event = order.execute(OrderCommand::Ship {
        tracking_number: "SF123456".to_string(),
    }).unwrap();
    store.append(event);

    println!("After shipping: {:?}", order.status);

    // 回溯到支付后的状态
    if let Some(order_v1) = store.rebuild_to_version(1) {
        println!("Order at version 1: {:?}", order_v1.status);
    }
}
```

## 小结

状态流转场景让我们用到了三种模式：

| 模式 | 解决的问题 | Rust 实现 |
|------|-----------|----------|
| 状态模式 | 状态相关的行为封装 | enum + match / 类型状态 |
| 命令模式 | 操作的封装与撤销 | enum 命令 / trait 命令 |
| 备忘录模式 | 状态的保存与恢复 | 快照 / 事件溯源 |

关键洞见：

1. **用 enum 定义状态**，编译器强制处理所有情况
2. **类型状态**可以在编译期检查状态转换，但有适用范围
3. **命令模式**实现撤销时，要分离命令和副作用
4. **事件溯源**是备忘录的升级版，支持任意时间点重建

下一篇，我们看模块间如何通信：**事件驱动——模块间的优雅对话**。观察者模式和中介者模式将帮助你解耦复杂的系统。

---

> **系列导航**
>
> - 第一篇：Rust 让设计模式返璞归真
> - 第二篇：配置系统——从硬编码到热加载
> - 第三篇：插件架构——让系统长出翅膀
> - 第四篇：数据管道——像乐高一样组装逻辑
> - **第五篇：状态流转——订单不只是 CRUD**（本文）
> - 第六篇：事件驱动——模块间的优雅对话
> - 第七篇：外部集成——与第三方和平共处
> - 第八篇：框架设计——把控制权还给用户
> - 第九篇：树形世界——递归结构的驯服术
> - 第十篇：规则引擎——让业务人员写代码
> - 第十一篇：终篇——反模式与断舍离
