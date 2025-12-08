---
title: "模式不是套路（十一）：终篇——反模式与断舍离"
date: 2025-12-10T11:40:00+08:00
draft: false
tags: ["Rust", "设计模式", "反模式", "过度设计", "重构"]
categories: ["技术"]
description: "设计模式系列终篇，探讨过度设计的症状、模式滥用的案例，以及在 Rust 中什么时候不需要模式。"
series: ["模式不是套路"]
---

> "模式不是目的，而是手段。当手段成为目的，代码便开始腐烂。"

走完了这趟设计模式的旅程，我们学会了用 Rust 的特性让模式返璞归真。但还有一个更重要的问题：**什么时候不该用模式？**

这篇终章，我们来聊聊设计模式的阴暗面——过度设计、模式滥用，以及断舍离的智慧。

## 过度设计的七宗罪

### 第一罪：未来幻想症

```rust
// ❌ 为"可能的需求"做准备
trait PaymentProcessor {
    fn process(&self, amount: f64) -> Result<(), Error>;
    fn refund(&self, transaction_id: &str) -> Result<(), Error>;
    fn partial_refund(&self, transaction_id: &str, amount: f64) -> Result<(), Error>;
    fn recurring(&self, schedule: &Schedule) -> Result<(), Error>;
    fn batch_process(&self, payments: Vec<Payment>) -> Result<(), Error>;
    fn validate_card(&self, card: &Card) -> Result<bool, Error>;
    fn tokenize(&self, card: &Card) -> Result<Token, Error>;
    fn dispute(&self, transaction_id: &str, reason: &str) -> Result<(), Error>;
    // 还有 20 个方法...
}

// ✅ 只实现当前需要的
trait PaymentProcessor {
    fn process(&self, amount: f64) -> Result<(), Error>;
}

// 需要时再扩展
trait RefundablePayment: PaymentProcessor {
    fn refund(&self, transaction_id: &str) -> Result<(), Error>;
}
```

**症状**：接口定义了大量"将来可能用到"的方法，但实现者只用其中两三个。

**Rust 的解药**：trait 组合。需要什么就定义什么，通过 trait 继承和组合按需扩展。

### 第二罪：抽象层叠床架屋

```rust
// ❌ 抽象套抽象
trait Repository<T> {
    fn save(&self, entity: T) -> Result<(), Error>;
}

trait UserRepository: Repository<User> {
    fn find_by_email(&self, email: &str) -> Result<Option<User>, Error>;
}

trait UserRepositoryFactory {
    fn create(&self) -> Box<dyn UserRepository>;
}

trait UserRepositoryFactoryProvider {
    fn get_factory(&self) -> Box<dyn UserRepositoryFactory>;
}

// ✅ 直接了当
struct UserRepository {
    pool: PgPool,
}

impl UserRepository {
    pub fn save(&self, user: &User) -> Result<(), Error> {
        // 直接实现
    }

    pub fn find_by_email(&self, email: &str) -> Result<Option<User>, Error> {
        // 直接实现
    }
}
```

**症状**：为了"灵活性"层层包装，但整个系统只有一种实现。

**诊断方法**：如果你的 Factory 只返回一种类型，你不需要 Factory。

### 第三罪：配置恐惧症

```rust
// ❌ 把所有东西都变成配置
struct ServerConfig {
    host: String,
    port: u16,
    max_connections: usize,
    timeout_ms: u64,
    retry_count: u32,
    retry_delay_ms: u64,
    buffer_size: usize,
    thread_pool_size: usize,
    enable_compression: bool,
    compression_level: u32,
    enable_logging: bool,
    log_level: String,
    log_format: String,
    enable_metrics: bool,
    metrics_port: u16,
    // 还有 50 个配置项...
}

// ✅ 合理的默认值 + 必要的配置
struct ServerConfig {
    host: String,
    port: u16,
}

impl Default for ServerConfig {
    fn default() -> Self {
        Self {
            host: "127.0.0.1".to_string(),
            port: 8080,
        }
    }
}
```

**症状**：开发者恐惧硬编码，把所有常量都变成配置项，结果配置文件比代码还难维护。

**断舍离原则**：
- 不会变的值 → 常量
- 环境相关的值 → 环境变量
- 运行时可能调整的值 → 配置文件
- 其他 → 合理的默认值

### 第四罪：模式强迫症

```rust
// ❌ 为了用模式而用模式
// "这里应该用策略模式！"
trait AdditionStrategy {
    fn add(&self, a: i32, b: i32) -> i32;
}

struct SimpleAddition;
impl AdditionStrategy for SimpleAddition {
    fn add(&self, a: i32, b: i32) -> i32 {
        a + b
    }
}

struct Calculator {
    strategy: Box<dyn AdditionStrategy>,
}

// ✅ 简单的事情简单做
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

**症状**：看到任何逻辑都想套个模式，哪怕只是简单的加法。

**自检问题**：
- 这个抽象解决了什么问题？
- 如果不用模式，代码会有什么问题？
- 三个月后，新人能看懂这段代码吗？

### 第五罪：继承瘾

```rust
// ❌ Java 思维的继承链（在 Rust 中强行模拟）
trait Animal {
    fn breathe(&self);
}

trait Mammal: Animal {
    fn give_birth(&self);
}

trait Canine: Mammal {
    fn bark(&self);
}

trait DomesticCanine: Canine {
    fn obey_command(&self);
}

struct Dog;

impl Animal for Dog { fn breathe(&self) {} }
impl Mammal for Dog { fn give_birth(&self) {} }
impl Canine for Dog { fn bark(&self) {} }
impl DomesticCanine for Dog { fn obey_command(&self) {} }

// ✅ 组合优于继承
struct Dog {
    name: String,
}

impl Dog {
    pub fn bark(&self) {
        println!("{} says woof!", self.name);
    }

    pub fn obey(&self, command: &str) {
        println!("{} obeys: {}", self.name, command);
    }
}
```

**症状**：强行在 Rust 中模拟 Java/C++ 的继承体系。

**Rust 哲学**：组合优于继承。Rust 没有类继承是故意的设计。

### 第六罪：DRY 洁癖

```rust
// ❌ 过度追求 DRY，制造错误的抽象
fn process_entity<T: Entity>(entity: T, config: &Config) -> Result<(), Error>
where
    T: Validate + Transform + Persist + Notify + Audit,
{
    entity.validate()?;
    let transformed = entity.transform(config)?;
    transformed.persist()?;
    transformed.notify()?;
    transformed.audit()?;
    Ok(())
}

// User 和 Order 被强行统一，但它们的处理逻辑完全不同
process_entity(user, &config)?;
process_entity(order, &config)?;

// ✅ 允许适度重复
fn process_user(user: User) -> Result<(), Error> {
    validate_user(&user)?;
    let user = enrich_user(user);
    save_user(&user)?;
    send_welcome_email(&user)?;
    Ok(())
}

fn process_order(order: Order) -> Result<(), Error> {
    validate_order(&order)?;
    let order = calculate_total(order);
    save_order(&order)?;
    notify_warehouse(&order)?;
    Ok(())
}
```

**症状**：看到两段代码有相似结构就想抽象，但它们的业务含义完全不同。

**断舍离智慧**：
- 重复两次 → 观察
- 重复三次 → 考虑抽象
- 业务含义不同的重复 → 保持重复

### 第七罪：性能幻觉

```rust
// ❌ "对象池一定更快"
struct ConnectionPool {
    connections: Vec<Connection>,
    // 复杂的池化逻辑...
}

// ✅ 先测量，再优化
// 很多时候，现代数据库驱动已经内置了连接池
// 或者连接创建成本并没有想象中高
let client = Client::connect(&config).await?;
```

**症状**：没有性能数据支撑，凭直觉添加"优化"代码。

**正确做法**：
1. 先写最简单的实现
2. 测量性能
3. 找到瓶颈
4. 针对性优化

## Rust 中不需要的模式

### 1. 单例模式——大多数时候

```rust
// ❌ 复杂的单例实现
static INSTANCE: OnceLock<Mutex<Config>> = OnceLock::new();

fn get_config() -> &'static Mutex<Config> {
    INSTANCE.get_or_init(|| Mutex::new(Config::load()))
}

// ✅ 直接传递依赖
struct App {
    config: Config,
    // ...
}

fn main() {
    let config = Config::load();
    let app = App::new(config);
    app.run();
}
```

**什么时候真的需要单例**：
- 全局日志器（但建议用 `tracing`）
- 全局配置（但考虑依赖注入）
- 硬件资源抽象（如 GPIO 引脚）

### 2. 迭代器模式——语言内置

```rust
// ❌ 手动实现迭代器容器
struct NumberIterator {
    current: i32,
    end: i32,
}

impl NumberIterator {
    fn new(end: i32) -> Self {
        Self { current: 0, end }
    }

    fn has_next(&self) -> bool {
        self.current < self.end
    }

    fn next(&mut self) -> i32 {
        let value = self.current;
        self.current += 1;
        value
    }
}

// ✅ 使用标准迭代器
let numbers: Vec<i32> = (0..10).collect();

// 或者为自定义类型实现 Iterator trait
impl Iterator for MyCollection {
    type Item = MyItem;

    fn next(&mut self) -> Option<Self::Item> {
        // 只需要实现这一个方法
    }
}
```

### 3. 空对象模式——Option 就够了

```rust
// ❌ 空对象模式
trait Logger {
    fn log(&self, message: &str);
}

struct RealLogger;
impl Logger for RealLogger {
    fn log(&self, message: &str) {
        println!("{}", message);
    }
}

struct NullLogger;
impl Logger for NullLogger {
    fn log(&self, _message: &str) {
        // 什么都不做
    }
}

// ✅ 用 Option
fn do_something(logger: Option<&dyn Logger>) {
    if let Some(logger) = logger {
        logger.log("doing something");
    }
    // 或者
    logger.map(|l| l.log("doing something"));
}
```

### 4. 观察者模式——channel 更 Rust

```rust
// ❌ 经典观察者（需要处理生命周期地狱）
struct Subject<'a> {
    observers: Vec<&'a dyn Observer>,
}

// ✅ 用 channel
use tokio::sync::broadcast;

let (tx, _) = broadcast::channel(100);

// 发布
tx.send(Event::UserCreated { id: 1 })?;

// 订阅
let mut rx = tx.subscribe();
while let Ok(event) = rx.recv().await {
    handle_event(event);
}
```

### 5. 原型模式——Clone trait

```rust
// ❌ 自定义原型接口
trait Prototype {
    fn clone_instance(&self) -> Box<dyn Prototype>;
}

// ✅ 直接用 Clone
#[derive(Clone)]
struct Document {
    title: String,
    content: String,
}

let doc2 = doc1.clone();
```

### 6. 访问者模式——enum + match

```rust
// ❌ 复杂的访问者模式
trait Visitor {
    fn visit_circle(&mut self, circle: &Circle);
    fn visit_rectangle(&mut self, rectangle: &Rectangle);
    fn visit_triangle(&mut self, triangle: &Triangle);
}

trait Shape {
    fn accept(&self, visitor: &mut dyn Visitor);
}

// ✅ enum + match
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { base: f64, height: f64 },
}

fn calculate_area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
        Shape::Rectangle { width, height } => width * height,
        Shape::Triangle { base, height } => 0.5 * base * height,
    }
}
```

**什么时候需要访问者**：
- 类型是开放的（不断有新类型加入）
- 操作是稳定的（很少添加新操作）
- 类型定义在外部 crate，你无法修改

## 断舍离的艺术

### 何时引入模式

```
引入模式的正确时机：

                    复杂度
                      ↑
                      │     ╭──────────────────╮
                      │    ╱                    ╲
                      │   ╱   过度设计区域        ╲
                      │  ╱    (模式成为负担)       ╲
                      │ ╱                          ╲
        ──────────────┼──────────────────────────────→ 需求变化频率
                      │╲                          ╱
                      │ ╲   欠设计区域            ╱
                      │  ╲  (代码开始腐烂)        ╱
                      │   ╲                    ╱
                      │    ╲__________________╱
                      │
                      │    ╔════════════════════╗
                      │    ║   最佳实践区域       ║
                      │    ║  (恰到好处的抽象)    ║
                      │    ╚════════════════════╝

引入模式的信号：
✓ 相同的修改需要改多处代码
✓ 新增功能需要大量条件判断
✓ 测试变得困难
✓ 代码审查时经常有人困惑
```

### 移除模式的时机

**信号 1：只有一个实现**

```rust
// 如果你的 trait 只有一个实现，考虑去掉它
trait UserService {
    fn create_user(&self, name: &str) -> Result<User, Error>;
}

struct UserServiceImpl;
impl UserService for UserServiceImpl { /* ... */ }

// 改为
struct UserService;
impl UserService {
    pub fn create_user(&self, name: &str) -> Result<User, Error> { /* ... */ }
}
```

**信号 2：抽象只是转发**

```rust
// 如果你的"facade"只是转发调用
struct Facade {
    service_a: ServiceA,
    service_b: ServiceB,
}

impl Facade {
    fn do_a(&self) { self.service_a.do_a(); }
    fn do_b(&self) { self.service_b.do_b(); }
}

// 直接使用原始服务可能更清晰
```

**信号 3：没人能解释它存在的原因**

如果在代码审查中问"为什么要这样设计"，答案是"我也不知道，一直就是这样"——是时候重构了。

### 重构策略

**从简单开始，按需演进**：

```
第一阶段：直接实现
fn calculate_tax(amount: f64) -> f64 {
    amount * 0.1  // 10% 税率
}

第二阶段：当需要不同税率时
fn calculate_tax(amount: f64, rate: f64) -> f64 {
    amount * rate
}

第三阶段：当税率计算变复杂时
struct TaxCalculator {
    base_rate: f64,
    deductions: Vec<Deduction>,
}

impl TaxCalculator {
    fn calculate(&self, amount: f64) -> f64 {
        // 复杂逻辑
    }
}

第四阶段：当需要不同税制时（才考虑策略模式）
trait TaxStrategy {
    fn calculate(&self, amount: f64) -> f64;
}
```

## 本系列的设计哲学

回顾整个系列，我们传达的核心思想是：

### 1. Rust 的类型系统是最好的设计模式

```rust
// 编译器帮你检查状态转换
struct Order<S: OrderState> {
    state: PhantomData<S>,
    // ...
}

// 编译器帮你管理资源
struct Connection {
    // Drop 自动清理
}

// 编译器帮你保证线程安全
struct SharedState {
    data: Arc<RwLock<Data>>,
}
```

### 2. 组合优于继承，函数优于对象

```rust
// Rust 的迭代器适配器就是最好的例子
let result: Vec<_> = items
    .iter()
    .filter(|x| x.is_valid())
    .map(|x| x.transform())
    .collect();
```

### 3. 显式优于隐式

```rust
// Rust 强制你处理错误
let result = may_fail()?;

// Rust 强制你处理空值
if let Some(value) = maybe_value {
    // ...
}

// Rust 强制你标注可变性
let mut data = vec![];
data.push(1);
```

### 4. 约束产生自由

```rust
// 借用检查器的约束让你写出无数据竞争的代码
// 生命周期标注的约束让你写出无悬垂指针的代码
// 类型系统的约束让你在编译时发现大部分 bug
```

## 写在最后

设计模式是前人智慧的结晶，但它们诞生于特定的语言和时代背景。在 Rust 中：

- **有些模式变得多余**——因为语言特性已经解决了它们要解决的问题
- **有些模式变得简洁**——因为 enum、trait、生命周期让实现更优雅
- **有些模式依然重要**——因为它们解决的是架构层面的问题，与语言无关

最重要的不是记住 23 个模式的名字和实现，而是理解：

1. **每个模式要解决什么问题**
2. **在 Rust 中有没有更好的替代方案**
3. **什么时候该用，什么时候不该用**

软件设计的终极目标是**管理复杂性**。如果你的设计模式增加了复杂性而不是降低了它，那就是时候断舍离了。

## 系列回顾

1. [开篇：Rust 让设计模式返璞归真](@/posts/design-pattern-not-routine-1-intro.md)
2. [配置系统：从硬编码到热加载](@/posts/design-pattern-not-routine-2-config.md) — 单例、建造者、原型
3. [插件架构：让系统长出翅膀](@/posts/design-pattern-not-routine-3-plugin.md) — 工厂、抽象工厂、桥接
4. [数据管道：像乐高一样组装逻辑](@/posts/design-pattern-not-routine-4-pipeline.md) — 装饰器、责任链、策略
5. [状态流转：订单不只是 CRUD](@/posts/design-pattern-not-routine-5-state.md) — 状态、命令、备忘录
6. [事件驱动：模块间的优雅对话](@/posts/design-pattern-not-routine-6-event.md) — 观察者、中介者
7. [外部集成：与第三方和平共处](@/posts/design-pattern-not-routine-7-integration.md) — 适配器、外观、代理
8. [框架设计：把控制权还给用户](@/posts/design-pattern-not-routine-8-framework.md) — 模板方法、钩子
9. [树形世界：递归结构的驯服术](@/posts/design-pattern-not-routine-9-tree.md) — 组合、访问者、迭代器、享元
10. [规则引擎：让业务人员写代码](@/posts/design-pattern-not-routine-10-rule-engine.md) — 解释器
11. [终篇：反模式与断舍离](@/posts/design-pattern-not-routine-11-antipattern.md) — 本文

---

> "好的代码不是没有坏味道，而是在闻到坏味道时，有勇气说'这里需要改变'。"

感谢你读完这个系列。愿你的 Rust 代码，既有模式的优雅，又有断舍离的智慧。
