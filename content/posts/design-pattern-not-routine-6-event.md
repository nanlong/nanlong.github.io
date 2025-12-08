---
title: "模式不是套路（六）：事件驱动——模块间的优雅对话"
date: 2025-12-10T10:50:00+08:00
draft: false
tags: ["Rust", "设计模式", "事件驱动", "观察者模式", "中介者模式"]
categories: ["技术"]
description: "订单模块要通知库存、积分、物流、消息推送……调用链越来越长，模块耦合越来越紧。观察者模式和中介者模式，让模块间的对话从'打电话'变成'发微信'。"
series: ["模式不是套路"]
---

订单模块要通知库存、积分、物流、消息推送……调用链越来越长，模块耦合越来越紧。观察者模式和中介者模式，让模块间的对话从"打电话"变成"发微信"。

<!--more-->

## 问题：一个纠缠不清的系统

电商系统，订单支付成功后：

```rust
impl OrderService {
    fn pay_order(&self, order_id: &str) -> Result<(), Error> {
        // 1. 更新订单状态
        let order = self.order_repo.get(order_id)?;
        order.pay()?;
        self.order_repo.save(&order)?;

        // 2. 扣减库存
        self.inventory_service.deduct(&order.items)?;

        // 3. 发放积分
        self.points_service.award(order.user_id, order.total_amount)?;

        // 4. 发送通知
        self.notification_service.send_payment_success(&order)?;

        // 5. 更新用户统计
        self.user_stats_service.record_purchase(&order)?;

        // 6. 触发推荐系统更新
        self.recommendation_service.update_preferences(&order)?;

        // 7. 记录审计日志
        self.audit_service.log_payment(&order)?;

        Ok(())
    }
}
```

问题显而易见：

```text
1. OrderService 依赖了 7 个其他服务
2. 任何一个服务失败，整个支付流程失败
3. 新增功能（比如发优惠券）要改 OrderService
4. 测试困难——需要 mock 7 个依赖
5. 性能问题——所有操作串行执行
```

这种代码叫做**"神对象"**（God Object）——一个类知道所有事情，做所有事情。

## 破局：观察者模式

**观察者模式**的核心是：**定义对象间一对多的依赖关系，当一个对象状态改变时，所有依赖者都会收到通知**。

### 基础实现

```rust
// 事件定义
#[derive(Debug, Clone)]
pub enum OrderEvent {
    Created { order_id: String, user_id: String },
    Paid { order_id: String, amount: Decimal, user_id: String },
    Shipped { order_id: String, tracking_number: String },
    Delivered { order_id: String },
    Cancelled { order_id: String, reason: String },
}

// 观察者 trait
pub trait OrderObserver: Send + Sync {
    fn on_event(&self, event: &OrderEvent);
}

// 被观察者（发布者）
pub struct OrderEventPublisher {
    observers: Vec<Arc<dyn OrderObserver>>,
}

impl OrderEventPublisher {
    pub fn new() -> Self {
        Self { observers: Vec::new() }
    }

    pub fn subscribe(&mut self, observer: Arc<dyn OrderObserver>) {
        self.observers.push(observer);
    }

    pub fn publish(&self, event: OrderEvent) {
        for observer in &self.observers {
            observer.on_event(&event);
        }
    }
}
```

现在各个服务实现观察者：

```rust
// 库存服务
struct InventoryObserver {
    inventory_service: Arc<InventoryService>,
}

impl OrderObserver for InventoryObserver {
    fn on_event(&self, event: &OrderEvent) {
        if let OrderEvent::Paid { order_id, .. } = event {
            if let Err(e) = self.inventory_service.deduct_for_order(order_id) {
                log::error!("Failed to deduct inventory: {}", e);
            }
        }
    }
}

// 积分服务
struct PointsObserver {
    points_service: Arc<PointsService>,
}

impl OrderObserver for PointsObserver {
    fn on_event(&self, event: &OrderEvent) {
        if let OrderEvent::Paid { user_id, amount, .. } = event {
            // 消费 1 元 = 1 积分
            let points = amount.to_u64().unwrap_or(0);
            if let Err(e) = self.points_service.award(user_id, points) {
                log::error!("Failed to award points: {}", e);
            }
        }
    }
}

// 通知服务
struct NotificationObserver {
    notification_service: Arc<NotificationService>,
}

impl OrderObserver for NotificationObserver {
    fn on_event(&self, event: &OrderEvent) {
        match event {
            OrderEvent::Paid { order_id, .. } => {
                self.notification_service.send("payment_success", order_id);
            }
            OrderEvent::Shipped { order_id, tracking_number } => {
                self.notification_service.send_shipping_notice(order_id, tracking_number);
            }
            _ => {}
        }
    }
}
```

OrderService 变得简洁：

```rust
impl OrderService {
    fn pay_order(&self, order_id: &str) -> Result<(), Error> {
        // 1. 核心业务：更新订单状态
        let order = self.order_repo.get(order_id)?;
        order.pay()?;
        self.order_repo.save(&order)?;

        // 2. 发布事件，由观察者处理后续逻辑
        self.event_publisher.publish(OrderEvent::Paid {
            order_id: order_id.to_string(),
            amount: order.total_amount,
            user_id: order.user_id.clone(),
        });

        Ok(())
    }
}
```

### Rust 的礼物：Channel 实现异步观察者

用 `tokio::sync::broadcast` 实现发布-订阅：

```rust
use tokio::sync::broadcast;

pub struct EventBus {
    sender: broadcast::Sender<OrderEvent>,
}

impl EventBus {
    pub fn new(capacity: usize) -> Self {
        let (sender, _) = broadcast::channel(capacity);
        Self { sender }
    }

    pub fn publish(&self, event: OrderEvent) {
        // 忽略发送失败（没有订阅者时）
        let _ = self.sender.send(event);
    }

    pub fn subscribe(&self) -> broadcast::Receiver<OrderEvent> {
        self.sender.subscribe()
    }
}

// 订阅者作为独立任务运行
async fn start_inventory_subscriber(event_bus: &EventBus, inventory_service: Arc<InventoryService>) {
    let mut receiver = event_bus.subscribe();

    tokio::spawn(async move {
        loop {
            match receiver.recv().await {
                Ok(OrderEvent::Paid { order_id, .. }) => {
                    if let Err(e) = inventory_service.deduct_for_order(&order_id).await {
                        log::error!("Inventory deduction failed: {}", e);
                    }
                }
                Ok(_) => {} // 忽略其他事件
                Err(broadcast::error::RecvError::Lagged(n)) => {
                    log::warn!("Missed {} events", n);
                }
                Err(broadcast::error::RecvError::Closed) => break,
            }
        }
    });
}
```

### 更进一步：类型化事件通道

让不同类型的事件有独立的通道：

```rust
use std::any::{Any, TypeId};
use std::collections::HashMap;
use tokio::sync::broadcast;

pub struct TypedEventBus {
    channels: HashMap<TypeId, Box<dyn Any + Send + Sync>>,
}

impl TypedEventBus {
    pub fn new() -> Self {
        Self { channels: HashMap::new() }
    }

    pub fn register<E: Clone + Send + 'static>(&mut self, capacity: usize) {
        let (tx, _) = broadcast::channel::<E>(capacity);
        self.channels.insert(TypeId::of::<E>(), Box::new(tx));
    }

    pub fn publish<E: Clone + Send + 'static>(&self, event: E) {
        if let Some(channel) = self.channels.get(&TypeId::of::<E>()) {
            if let Some(tx) = channel.downcast_ref::<broadcast::Sender<E>>() {
                let _ = tx.send(event);
            }
        }
    }

    pub fn subscribe<E: Clone + Send + 'static>(&self) -> Option<broadcast::Receiver<E>> {
        self.channels
            .get(&TypeId::of::<E>())
            .and_then(|channel| channel.downcast_ref::<broadcast::Sender<E>>())
            .map(|tx| tx.subscribe())
    }
}

// 使用
let mut bus = TypedEventBus::new();
bus.register::<OrderPaidEvent>(100);
bus.register::<OrderShippedEvent>(100);

// 只订阅关心的事件类型
let mut paid_events = bus.subscribe::<OrderPaidEvent>().unwrap();
```

## 演进：中介者模式

当模块之间的交互变得复杂时，观察者模式可能不够用：

```text
场景：限时秒杀

1. 用户下单 → 检查库存 → 锁定库存 → 创建订单
2. 支付超时 → 释放库存 → 取消订单
3. 支付成功 → 扣减库存 → 发放优惠券 → 通知
4. 库存不足 → 加入等待队列 → 等待释放 → 重试下单

模块之间需要复杂的协调
```

**中介者模式**的核心是：**用一个中介对象封装一系列对象的交互**。

```rust
// 中介者定义各模块的交互逻辑
pub struct FlashSaleMediator {
    order_service: Arc<OrderService>,
    inventory_service: Arc<InventoryService>,
    payment_service: Arc<PaymentService>,
    coupon_service: Arc<CouponService>,
    notification_service: Arc<NotificationService>,
    wait_queue: Arc<WaitQueue>,
}

impl FlashSaleMediator {
    // 下单流程
    pub async fn place_order(&self, request: PlaceOrderRequest) -> Result<Order, FlashSaleError> {
        // 1. 尝试锁定库存
        let lock_result = self.inventory_service
            .try_lock(&request.product_id, request.quantity)
            .await;

        match lock_result {
            Ok(lock) => {
                // 2. 创建订单
                let order = self.order_service
                    .create_flash_sale_order(&request, lock.id)
                    .await?;

                // 3. 设置支付超时
                self.schedule_payment_timeout(&order).await;

                Ok(order)
            }
            Err(InventoryError::InsufficientStock) => {
                // 库存不足，加入等待队列
                self.wait_queue.enqueue(request.clone()).await?;
                Err(FlashSaleError::AddedToWaitList)
            }
            Err(e) => Err(e.into()),
        }
    }

    // 支付成功
    pub async fn on_payment_success(&self, order_id: &str) -> Result<(), FlashSaleError> {
        let order = self.order_service.get(order_id).await?;

        // 1. 确认扣减库存（从锁定变为已扣减）
        self.inventory_service
            .confirm_deduction(&order.inventory_lock_id)
            .await?;

        // 2. 更新订单状态
        self.order_service.mark_paid(order_id).await?;

        // 3. 发放优惠券
        if let Some(coupon_rule) = self.get_coupon_rule(&order) {
            self.coupon_service
                .issue(&order.user_id, &coupon_rule)
                .await?;
        }

        // 4. 发送通知
        self.notification_service
            .send_flash_sale_success(&order)
            .await?;

        Ok(())
    }

    // 支付超时
    pub async fn on_payment_timeout(&self, order_id: &str) -> Result<(), FlashSaleError> {
        let order = self.order_service.get(order_id).await?;

        // 1. 释放库存锁定
        self.inventory_service
            .release_lock(&order.inventory_lock_id)
            .await?;

        // 2. 取消订单
        self.order_service
            .cancel(order_id, "Payment timeout")
            .await?;

        // 3. 通知等待队列
        self.notify_wait_queue(&order.product_id).await;

        // 4. 通知用户
        self.notification_service
            .send_payment_timeout(&order)
            .await?;

        Ok(())
    }

    // 库存释放后通知等待队列
    async fn notify_wait_queue(&self, product_id: &str) {
        if let Some(waiting) = self.wait_queue.dequeue(product_id).await {
            // 尝试为等待用户下单
            tokio::spawn({
                let mediator = self.clone();
                async move {
                    if let Err(e) = mediator.place_order(waiting).await {
                        log::warn!("Failed to place order from wait queue: {}", e);
                    }
                }
            });
        }
    }

    async fn schedule_payment_timeout(&self, order: &Order) {
        let order_id = order.id.clone();
        let mediator = self.clone();
        let timeout = Duration::from_secs(900);  // 15 分钟

        tokio::spawn(async move {
            tokio::time::sleep(timeout).await;

            // 检查是否已支付
            if let Ok(order) = mediator.order_service.get(&order_id).await {
                if matches!(order.status, OrderStatus::Created) {
                    let _ = mediator.on_payment_timeout(&order_id).await;
                }
            }
        });
    }
}
```

### 中介者 vs 观察者

| 特性 | 观察者模式 | 中介者模式 |
|------|-----------|-----------|
| 通信方向 | 一对多（发布-订阅） | 多对多（协调） |
| 耦合 | 观察者不知道彼此 | 中介者知道所有同事 |
| 适用场景 | 事件通知 | 复杂交互协调 |
| 复杂度 | 简单 | 可能成为 God Object |

实践中经常**组合使用**：

```rust
pub struct FlashSaleMediator {
    // ...
    event_bus: Arc<EventBus>,  // 用于非关键通知
}

impl FlashSaleMediator {
    pub async fn on_payment_success(&self, order_id: &str) -> Result<(), FlashSaleError> {
        // 关键流程：中介者协调
        self.inventory_service.confirm_deduction(...).await?;
        self.order_service.mark_paid(order_id).await?;

        // 非关键通知：发布事件
        self.event_bus.publish(OrderEvent::Paid {
            order_id: order_id.to_string(),
            // ...
        });

        Ok(())
    }
}
```

## 进阶：领域事件与事件溯源

在 DDD（领域驱动设计）中，**领域事件**是一等公民：

```rust
// 领域事件：带有业务语义
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DomainEvent {
    pub id: Uuid,
    pub aggregate_id: String,
    pub aggregate_type: String,
    pub event_type: String,
    pub payload: serde_json::Value,
    pub occurred_at: DateTime<Utc>,
    pub version: u64,
}

// 聚合根产生事件
pub trait AggregateRoot {
    fn id(&self) -> &str;
    fn version(&self) -> u64;
    fn take_events(&mut self) -> Vec<DomainEvent>;
}

impl Order {
    pub fn pay(&mut self, payment_id: String, amount: Decimal) -> Result<(), OrderError> {
        // 业务验证
        if !matches!(self.status, OrderStatus::Created) {
            return Err(OrderError::InvalidTransition);
        }

        // 状态变更
        self.status = OrderStatus::Paid {
            paid_at: Utc::now(),
            amount,
        };
        self.payment_id = Some(payment_id.clone());

        // 记录事件
        self.pending_events.push(DomainEvent {
            id: Uuid::new_v4(),
            aggregate_id: self.id.clone(),
            aggregate_type: "Order".to_string(),
            event_type: "OrderPaid".to_string(),
            payload: serde_json::json!({
                "payment_id": payment_id,
                "amount": amount.to_string(),
            }),
            occurred_at: Utc::now(),
            version: self.version + 1,
        });

        Ok(())
    }
}

// 事件处理器
pub trait EventHandler: Send + Sync {
    fn event_type(&self) -> &str;
    fn handle(&self, event: &DomainEvent) -> Result<(), EventHandlerError>;
}

// 事件分发器
pub struct EventDispatcher {
    handlers: HashMap<String, Vec<Arc<dyn EventHandler>>>,
}

impl EventDispatcher {
    pub fn register(&mut self, handler: Arc<dyn EventHandler>) {
        self.handlers
            .entry(handler.event_type().to_string())
            .or_default()
            .push(handler);
    }

    pub fn dispatch(&self, event: &DomainEvent) {
        if let Some(handlers) = self.handlers.get(&event.event_type) {
            for handler in handlers {
                if let Err(e) = handler.handle(event) {
                    log::error!("Event handler failed: {}", e);
                    // 可以加入重试队列
                }
            }
        }
    }
}
```

## 陷阱：别掉进这些坑

### 陷阱一：事件风暴

```rust
// ❌ 事件太细，产生风暴
impl Order {
    fn update_address(&mut self, address: Address) {
        self.address = address.clone();
        self.events.push(OrderAddressUpdated { address });
    }

    fn update_phone(&mut self, phone: String) {
        self.phone = phone.clone();
        self.events.push(OrderPhoneUpdated { phone });
    }

    // 用户修改收货信息时，产生大量事件
}
```

解决方案：**合并相关事件**

```rust
// ✅ 用业务事件而非数据变更事件
impl Order {
    fn update_shipping_info(&mut self, info: ShippingInfo) {
        self.address = info.address;
        self.phone = info.phone;
        self.receiver_name = info.receiver_name;

        // 一个业务事件，而不是三个数据变更事件
        self.events.push(OrderShippingInfoUpdated { info });
    }
}
```

### 陷阱二：循环依赖

```rust
// ❌ 危险：事件处理产生新事件，可能循环
impl OrderEventHandler {
    fn on_order_cancelled(&self, event: &OrderCancelledEvent) {
        // 退款
        self.refund_service.refund(event.order_id)?;
        // 退款服务发布 RefundCompleted 事件
        // RefundCompleted 处理器又更新订单状态
        // 订单状态变更又发布事件...
    }
}
```

解决方案：

```rust
// ✅ 方案 1：明确区分命令和事件
// 命令可以触发新命令，事件不应该触发命令

// ✅ 方案 2：事件去重
pub struct EventProcessor {
    processed: HashSet<Uuid>,
}

impl EventProcessor {
    fn process(&mut self, event: &DomainEvent) {
        if self.processed.contains(&event.id) {
            return;  // 已处理过
        }
        self.processed.insert(event.id);
        // 处理事件
    }
}

// ✅ 方案 3：设置最大传播深度
pub struct EventContext {
    depth: u32,
    max_depth: u32,
}

impl EventContext {
    fn can_propagate(&self) -> bool {
        self.depth < self.max_depth
    }

    fn child(&self) -> Self {
        Self {
            depth: self.depth + 1,
            max_depth: self.max_depth,
        }
    }
}
```

### 陷阱三：事件顺序依赖

```rust
// ❌ 危险：假设事件按发布顺序处理
// 在分布式系统中，这个假设不成立

// 事件 A: 订单创建
// 事件 B: 订单支付
// 处理器可能先收到 B 再收到 A
```

解决方案：

```rust
// ✅ 方案 1：事件带版本号
pub struct OrderEvent {
    version: u64,  // 严格递增
    // ...
}

impl EventHandler {
    fn handle(&self, event: &OrderEvent) {
        let current_version = self.get_aggregate_version(&event.aggregate_id);
        if event.version != current_version + 1 {
            // 乱序，加入等待队列
            self.pending_queue.push(event);
            return;
        }
        // 按序处理
    }
}

// ✅ 方案 2：幂等处理
impl EventHandler {
    fn handle(&self, event: &OrderEvent) {
        // 无论处理多少次，结果一样
        // 比如：设置状态为 Paid，而不是"增加"什么
    }
}
```

### 陷阱四：中介者变成 God Object

```rust
// ❌ 中介者承担太多职责
pub struct GodMediator {
    // 50 个依赖
    order_service: Arc<OrderService>,
    inventory_service: Arc<InventoryService>,
    payment_service: Arc<PaymentService>,
    // ... 47 more
}

impl GodMediator {
    // 100 个方法
}
```

解决方案：**拆分中介者**

```rust
// ✅ 按业务场景拆分
pub struct OrderMediator { /* ... */ }        // 订单相关协调
pub struct FlashSaleMediator { /* ... */ }    // 秒杀相关协调
pub struct RefundMediator { /* ... */ }       // 退款相关协调

// 或者使用 Saga 模式处理分布式事务
pub struct PaymentSaga {
    steps: Vec<SagaStep>,
}
```

## 完整示例

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::{broadcast, RwLock};
use chrono::{DateTime, Utc};
use uuid::Uuid;

// ==================== 事件定义 ====================

#[derive(Debug, Clone)]
pub enum OrderEvent {
    Created {
        order_id: String,
        user_id: String,
        items: Vec<OrderItem>,
    },
    Paid {
        order_id: String,
        user_id: String,
        amount: u64,
        payment_id: String,
    },
    Shipped {
        order_id: String,
        tracking_number: String,
    },
    Cancelled {
        order_id: String,
        reason: String,
    },
}

#[derive(Debug, Clone)]
pub struct OrderItem {
    pub product_id: String,
    pub quantity: u32,
}

// ==================== 事件总线 ====================

pub struct EventBus {
    sender: broadcast::Sender<OrderEvent>,
}

impl EventBus {
    pub fn new(capacity: usize) -> Self {
        let (sender, _) = broadcast::channel(capacity);
        Self { sender }
    }

    pub fn publish(&self, event: OrderEvent) {
        let _ = self.sender.send(event);
    }

    pub fn subscribe(&self) -> broadcast::Receiver<OrderEvent> {
        self.sender.subscribe()
    }
}

// ==================== 订单服务 ====================

pub struct OrderService {
    orders: RwLock<HashMap<String, Order>>,
    event_bus: Arc<EventBus>,
}

pub struct Order {
    pub id: String,
    pub user_id: String,
    pub status: OrderStatus,
    pub items: Vec<OrderItem>,
}

pub enum OrderStatus {
    Created,
    Paid,
    Shipped,
    Cancelled,
}

impl OrderService {
    pub fn new(event_bus: Arc<EventBus>) -> Self {
        Self {
            orders: RwLock::new(HashMap::new()),
            event_bus,
        }
    }

    pub async fn create_order(&self, user_id: String, items: Vec<OrderItem>) -> String {
        let order_id = Uuid::new_v4().to_string();
        let order = Order {
            id: order_id.clone(),
            user_id: user_id.clone(),
            status: OrderStatus::Created,
            items: items.clone(),
        };

        self.orders.write().await.insert(order_id.clone(), order);

        // 发布事件
        self.event_bus.publish(OrderEvent::Created {
            order_id: order_id.clone(),
            user_id,
            items,
        });

        order_id
    }

    pub async fn pay_order(&self, order_id: &str, payment_id: String, amount: u64) -> Result<(), String> {
        let mut orders = self.orders.write().await;
        let order = orders.get_mut(order_id).ok_or("Order not found")?;

        if !matches!(order.status, OrderStatus::Created) {
            return Err("Order cannot be paid".to_string());
        }

        order.status = OrderStatus::Paid;

        // 发布事件
        self.event_bus.publish(OrderEvent::Paid {
            order_id: order_id.to_string(),
            user_id: order.user_id.clone(),
            amount,
            payment_id,
        });

        Ok(())
    }
}

// ==================== 库存观察者 ====================

pub struct InventoryObserver {
    inventory: RwLock<HashMap<String, u32>>,
}

impl InventoryObserver {
    pub fn new() -> Self {
        Self {
            inventory: RwLock::new(HashMap::new()),
        }
    }

    pub async fn start(self: Arc<Self>, event_bus: &EventBus) {
        let mut receiver = event_bus.subscribe();

        tokio::spawn(async move {
            loop {
                match receiver.recv().await {
                    Ok(OrderEvent::Paid { order_id, .. }) => {
                        println!("[Inventory] Processing paid order: {}", order_id);
                        // 扣减库存逻辑
                    }
                    Ok(OrderEvent::Cancelled { order_id, .. }) => {
                        println!("[Inventory] Restoring inventory for cancelled order: {}", order_id);
                        // 恢复库存逻辑
                    }
                    Ok(_) => {}
                    Err(broadcast::error::RecvError::Lagged(n)) => {
                        println!("[Inventory] Missed {} events", n);
                    }
                    Err(broadcast::error::RecvError::Closed) => break,
                }
            }
        });
    }
}

// ==================== 积分观察者 ====================

pub struct PointsObserver;

impl PointsObserver {
    pub async fn start(event_bus: &EventBus) {
        let mut receiver = event_bus.subscribe();

        tokio::spawn(async move {
            loop {
                match receiver.recv().await {
                    Ok(OrderEvent::Paid { user_id, amount, .. }) => {
                        let points = amount / 100;  // 1 元 = 1 积分
                        println!("[Points] Awarding {} points to user {}", points, user_id);
                    }
                    Ok(_) => {}
                    Err(broadcast::error::RecvError::Lagged(_)) => {}
                    Err(broadcast::error::RecvError::Closed) => break,
                }
            }
        });
    }
}

// ==================== 通知观察者 ====================

pub struct NotificationObserver;

impl NotificationObserver {
    pub async fn start(event_bus: &EventBus) {
        let mut receiver = event_bus.subscribe();

        tokio::spawn(async move {
            loop {
                match receiver.recv().await {
                    Ok(OrderEvent::Created { order_id, user_id, .. }) => {
                        println!("[Notification] Sending order confirmation to user {}", user_id);
                    }
                    Ok(OrderEvent::Paid { order_id, user_id, .. }) => {
                        println!("[Notification] Sending payment success to user {}", user_id);
                    }
                    Ok(OrderEvent::Shipped { order_id, tracking_number }) => {
                        println!("[Notification] Sending shipping notice for order {}", order_id);
                    }
                    Ok(_) => {}
                    Err(_) => break,
                }
            }
        });
    }
}

// ==================== 使用示例 ====================

#[tokio::main]
async fn main() {
    // 创建事件总线
    let event_bus = Arc::new(EventBus::new(100));

    // 创建服务
    let order_service = Arc::new(OrderService::new(Arc::clone(&event_bus)));

    // 启动观察者
    let inventory_observer = Arc::new(InventoryObserver::new());
    inventory_observer.clone().start(&event_bus).await;
    PointsObserver::start(&event_bus).await;
    NotificationObserver::start(&event_bus).await;

    // 模拟业务流程
    let order_id = order_service.create_order(
        "user-001".to_string(),
        vec![OrderItem {
            product_id: "prod-001".to_string(),
            quantity: 2,
        }],
    ).await;

    println!("Created order: {}", order_id);

    // 稍等一下让观察者处理
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;

    // 支付
    order_service.pay_order(&order_id, "pay-001".to_string(), 9900).await.unwrap();

    // 等待观察者处理完成
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
}
```

## 小结

事件驱动架构中，两种模式各有适用场景：

| 模式 | 解决的问题 | Rust 实现 |
|------|-----------|----------|
| 观察者 | 一对多的事件通知 | channel / broadcast |
| 中介者 | 多对多的复杂协调 | 中心协调器 |

关键洞见：

1. **观察者实现解耦**，发布者不知道有谁在监听
2. **channel 是 Rust 的观察者模式**，天然支持异步
3. **中介者适合复杂流程**，但要防止变成 God Object
4. **两者经常组合使用**：关键流程用中介者，通知用观察者

下一篇，我们看如何与外部系统和平共处：**外部集成——与第三方和平共处**。适配器、外观、代理，这些模式帮你隔离外部依赖的变化。

---

> **系列导航**
>
> - 第一篇：Rust 让设计模式返璞归真
> - 第二篇：配置系统——从硬编码到热加载
> - 第三篇：插件架构——让系统长出翅膀
> - 第四篇：数据管道——像乐高一样组装逻辑
> - 第五篇：状态流转——订单不只是 CRUD
> - **第六篇：事件驱动——模块间的优雅对话**（本文）
> - 第七篇：外部集成——与第三方和平共处
> - 第八篇：框架设计——把控制权还给用户
> - 第九篇：树形世界——递归结构的驯服术
> - 第十篇：规则引擎——让业务人员写代码
> - 第十一篇：终篇——反模式与断舍离
