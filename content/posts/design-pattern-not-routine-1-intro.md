---
title: "模式不是套路（一）：Rust 让设计模式返璞归真"
date: 2025-12-10T10:00:00+08:00
draft: false
tags: ["Rust", "设计模式", "软件架构", "工程实践"]
categories: ["技术"]
description: "背了 23 种设计模式，写代码时却一个都想不起来？不是你的问题，是这些模式本来就不该死记硬背。在 Rust 中，很多经典模式已经融入了语言本身。"
series: ["模式不是套路"]
---

背了 23 种设计模式，写代码时却一个都想不起来？不是你的问题，是这些模式本来就不该死记硬背。在 Rust 中，很多经典模式已经融入了语言本身。

<!--more-->

## 一个尴尬的面试

"请用代码实现一个单例模式。"

面试官的问题很标准。候选人也很标准地写出了答案：

```java
public class Singleton {
    private static volatile Singleton instance;
    private static final Object lock = new Object();

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (lock) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

"很好，你知道为什么要用 `volatile` 吗？"

候选人开始背诵：指令重排序、happens-before、内存屏障……

面试结束，候选人拿到了 offer。三个月后，他在代码评审中看到了这样的 Rust 代码：

```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<Config> = OnceLock::new();

fn get_config() -> &'static Config {
    CONFIG.get_or_init(|| Config::load())
}
```

**"这……这算单例吗？"**

当然算。而且比他面试时写的那个更安全、更简洁。只是 Rust 把单例模式的复杂性，藏在了 `OnceLock` 的实现里。

这一刻，他突然意识到：**他背了三年的设计模式，可能学错了方向。**

## 设计模式的本质是什么？

1994 年，四位作者（Gang of Four）出版了《设计模式》。这本书总结了 23 种面向对象编程中反复出现的解决方案。

三十年过去了，这本书依然是经典。但很多人学设计模式的方式是错的：

```text
错误的学法：
  1. 背诵 23 种模式的名字和结构
  2. 背诵每种模式的 UML 图
  3. 背诵每种模式的"标准实现"
  4. 面试时默写出来

正确的学法：
  1. 理解每种模式要解决什么问题
  2. 理解模式背后的设计原则
  3. 在实际场景中识别问题
  4. 用合适的方式解决，不拘泥于形式
```

设计模式不是"套路"，而是"语言"。它给了我们一套词汇，让我们能够讨论设计决策：

```text
不用模式语言："我们需要一个东西，能够根据不同的情况创建不同类型的对象，
              而且调用方不需要知道具体创建了什么类型。"

用模式语言："这里需要一个工厂。"
```

**模式是沟通工具，不是实现模板。**

## Rust 改变了什么？

GoF 的设计模式诞生于 C++ 和 Smalltalk 时代。那时的语言有一些共同的特点：

```text
1990 年代主流语言的特点：
  - 面向对象（类、继承、多态）
  - 手动内存管理或 GC
  - 没有泛型（或很弱）
  - 没有 trait/interface 的 default 实现
  - 没有 algebraic data types
```

很多设计模式，其实是在弥补语言的不足：

| 模式 | 要解决的问题 | Rust 的解决方案 |
|------|------------|----------------|
| 单例 | 全局唯一实例 + 线程安全 | `OnceLock` / `LazyLock` |
| 迭代器 | 统一遍历接口 | `Iterator` trait（语言内置） |
| 策略 | 算法可替换 | trait object / 泛型 / 闭包 |
| 状态 | 状态转换的类型安全 | 类型状态模式（编译期检查） |
| 观察者 | 解耦事件源和监听者 | channel / callback |
| 装饰器 | 动态添加功能 | `Deref` trait / newtype |
| 命令 | 封装操作 | `FnOnce` / `enum` |

这不是说设计模式在 Rust 中没用了。而是说，**很多模式的"形"变了，但"神"还在**。

## 一个例子：策略模式的三种形态

策略模式的核心是：**让算法可以独立于使用它的客户端变化**。

在 Java 中，标准实现长这样：

```java
// 定义策略接口
interface PricingStrategy {
    double calculatePrice(double basePrice);
}

// 具体策略：正常价格
class NormalPricing implements PricingStrategy {
    public double calculatePrice(double basePrice) {
        return basePrice;
    }
}

// 具体策略：VIP 折扣
class VipPricing implements PricingStrategy {
    public double calculatePrice(double basePrice) {
        return basePrice * 0.8;
    }
}

// 使用策略
class Order {
    private PricingStrategy pricing;

    public void setPricing(PricingStrategy pricing) {
        this.pricing = pricing;
    }

    public double getPrice(double basePrice) {
        return pricing.calculatePrice(basePrice);
    }
}
```

在 Rust 中，同样的问题有三种解法：

### 形态一：trait object（动态分发）

```rust
trait PricingStrategy {
    fn calculate_price(&self, base_price: f64) -> f64;
}

struct NormalPricing;
impl PricingStrategy for NormalPricing {
    fn calculate_price(&self, base_price: f64) -> f64 {
        base_price
    }
}

struct VipPricing {
    rate: f64, // 折扣率，0.8 表示八折
}

impl VipPricing {
    fn new(rate: f64) -> Self {
        Self { rate }
    }
}

impl PricingStrategy for VipPricing {
    fn calculate_price(&self, base_price: f64) -> f64 {
        base_price * self.rate
    }
}

struct Order {
    pricing: Box<dyn PricingStrategy>,
}

impl Order {
    fn new(pricing: Box<dyn PricingStrategy>) -> Self {
        Self { pricing }
    }

    fn get_price(&self, base_price: f64) -> f64 {
        self.pricing.calculate_price(base_price)
    }
}
```

### 形态二：泛型（静态分发）

```rust
struct Order<P: PricingStrategy> {
    pricing: P,
}

impl<P: PricingStrategy> Order<P> {
    fn new(pricing: P) -> Self {
        Self { pricing }
    }

    fn get_price(&self, base_price: f64) -> f64 {
        self.pricing.calculate_price(base_price)
    }
}

// 编译时确定具体类型，零运行时开销
let order = Order::new(VipPricing::new(0.8));
```

### 形态三：闭包（最简洁）

```rust
struct Order {
    pricing: Box<dyn Fn(f64) -> f64>,
}

impl Order {
    fn new(pricing: impl Fn(f64) -> f64 + 'static) -> Self {
        Self { pricing: Box::new(pricing) }
    }

    fn get_price(&self, base_price: f64) -> f64 {
        (self.pricing)(base_price)
    }
}

// 使用闭包作为策略
let normal_order = Order::new(|price| price);
let vip_order = Order::new(|price| price * 0.8);
let special_order = Order::new(|price| if price > 100.0 { price * 0.7 } else { price * 0.9 });
```

三种形态，同一个"神"：**让定价逻辑可以灵活替换**。

你不需要记住"策略模式的 UML 图"，你需要理解的是：**当你发现有多种算法需要互换时，把算法抽象出来**。至于用 trait、泛型还是闭包，取决于具体场景。

## 本系列的设计理念

这个系列不会按照 GoF 的分类（创建型、结构型、行为型）来讲解模式。因为那种分类方式很学术，但不实用。

我们会按照**业务场景**来组织：

```text
第二篇：配置系统——从硬编码到热加载
  ├── 单例模式：全局配置如何安全共享
  ├── 建造者模式：复杂配置对象的构建
  └── 原型模式：配置的深拷贝与继承

第三篇：插件架构——让系统长出翅膀
  ├── 工厂方法：插件的注册与创建
  ├── 抽象工厂：插件家族的管理
  └── 桥接模式：抽象与实现的分离

第四篇：数据管道——像乐高一样组装逻辑
  ├── 装饰器模式：给数据处理加 buff
  ├── 责任链模式：请求的流水线处理
  └── 策略模式：可插拔的处理算法

第五篇：状态流转——订单不只是 CRUD
  ├── 状态模式：状态机的优雅实现
  ├── 命令模式：操作的封装与撤销
  └── 备忘录模式：状态的快照与恢复

第六篇：事件驱动——模块间的优雅对话
  ├── 观察者模式：发布订阅的本质
  └── 中介者模式：解耦复杂的交互

第七篇：外部集成——与第三方和平共处
  ├── 适配器模式：接口的转换
  ├── 外观模式：复杂子系统的简化
  └── 代理模式：访问的控制与增强

第八篇：框架设计——把控制权还给用户
  ├── 模板方法：定义算法骨架
  └── 钩子模式：可选的扩展点

第九篇：树形世界——递归结构的驯服术
  ├── 组合模式：统一处理树形结构
  ├── 访问者模式：分离数据与操作
  ├── 迭代器模式：遍历的抽象
  └── 享元模式：共享细粒度对象

第十篇：规则引擎——让业务人员写代码
  ├── 解释器模式：DSL 的实现
  └── 策略模式的进阶：规则的动态组合

第十一篇：终篇——反模式与断舍离
  ├── 过度设计的症状
  ├── 模式的滥用案例
  └── 什么时候不需要模式
```

每一篇文章都会遵循相同的结构：

1. **问题**：先看一个真实的业务困境
2. **朴素实现**：最直接的解法，以及它的问题
3. **破局**：引入模式，解决核心矛盾
4. **演进**：模式的组合与变体
5. **Rust 的礼物**：语言特性如何简化实现
6. **陷阱**：常见的误用和反模式
7. **小结**：提炼核心洞见

## Rust 的三件礼物

在进入具体场景之前，先了解 Rust 给设计模式带来的三个根本性变化：

### 礼物一：所有权让资源管理不再是问题

很多设计模式的复杂性来自于"谁负责释放资源"：

```text
传统问题：
  - 单例：谁负责销毁？
  - 观察者：被观察者销毁时，如何通知观察者？
  - 命令：命令对象的生命周期如何管理？
```

Rust 的所有权系统自动解决这些问题：

```rust
// 所有权转移，不需要手动管理
fn process(command: Box<dyn Command>) {
    command.execute();
    // command 在这里自动释放
}

// Arc + Weak 解决观察者的引用循环
struct Subject {
    observers: Vec<Weak<dyn Observer>>,
}
```

### 礼物二：枚举让状态建模更自然

很多行为模式的核心是"根据状态执行不同行为"：

```rust
// 状态模式：用 enum 替代继承体系
enum OrderState {
    Created { created_at: DateTime<Utc> },
    Paid { paid_at: DateTime<Utc>, amount: Decimal },
    Shipped { shipped_at: DateTime<Utc>, tracking: String },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String },
}

impl OrderState {
    fn can_cancel(&self) -> bool {
        matches!(self, OrderState::Created { .. } | OrderState::Paid { .. })
    }

    fn next_actions(&self) -> Vec<&'static str> {
        match self {
            OrderState::Created { .. } => vec!["pay", "cancel"],
            OrderState::Paid { .. } => vec!["ship", "cancel"],
            OrderState::Shipped { .. } => vec!["deliver"],
            OrderState::Delivered { .. } => vec!["return"],
            OrderState::Cancelled { .. } => vec![],
        }
    }
}
```

这比传统的状态模式（每个状态一个类）更紧凑，而且编译器会强制你处理所有状态。

### 礼物三：trait 让抽象更灵活

trait 比传统的接口/抽象类更强大：

```rust
// 可以为已有类型实现 trait（孤儿规则限制下）
impl MyTrait for Vec<String> {
    // 你的实现
}

// 可以有默认实现
trait Handler {
    fn handle(&self, request: &Request) -> Response;

    fn can_handle(&self, request: &Request) -> bool {
        true // 默认实现
    }
}

// 可以有关联类型
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// 可以组合多个 trait
fn process(item: impl Read + Write + Seek) {
    // 同时要求多种能力
}
```

## 在开始之前

学习设计模式的最大误区是：**为了使用模式而使用模式**。

```text
错误的思路：
  "这里应该用什么设计模式？"

正确的思路：
  "这里的问题是什么？有什么解法？"
  "这个解法和某个模式很像，可以借鉴它的经验。"
```

设计模式是工具，不是目标。

就像武功招式一样——初学者记招式，高手忘招式。**最好的代码，是让人看不出用了什么模式，但就是觉得清晰、灵活、好维护。**

下一篇，我们从一个最常见的场景开始：**配置系统**。

你会看到，一个"能用"的配置系统，如何一步步演进成一个"好用"的配置系统。在这个过程中，单例、建造者、原型这些模式会自然地浮现出来。

不是因为我们想用它们，而是因为我们需要它们。

---

> **系列导航**
>
> - **第一篇：Rust 让设计模式返璞归真**（本文）
> - 第二篇：配置系统——从硬编码到热加载
> - 第三篇：插件架构——让系统长出翅膀
> - 第四篇：数据管道——像乐高一样组装逻辑
> - 第五篇：状态流转——订单不只是 CRUD
> - 第六篇：事件驱动——模块间的优雅对话
> - 第七篇：外部集成——与第三方和平共处
> - 第八篇：框架设计——把控制权还给用户
> - 第九篇：树形世界——递归结构的驯服术
> - 第十篇：规则引擎——让业务人员写代码
> - 第十一篇：终篇——反模式与断舍离
