---
title: "并发模型演进史（六）：Rust 的第三条路——编译期消灭数据竞争"
date: 2025-12-08T22:50:00+08:00
draft: false
tags: ["并发", "Rust", "所有权", "Send", "Sync", "无畏并发"]
categories: ["技术"]
description: "其他语言在运行时防止数据竞争，Rust 说：太晚了。它把检查前移到编译期——写出数据竞争的代码，根本编译不过。这听起来很激进，但 Rust 证明了它是可行的。"
series: ["并发模型演进史"]
---

其他语言在运行时防止数据竞争，Rust 说：太晚了。它把检查前移到编译期——写出数据竞争的代码，根本编译不过。这听起来很激进，但 Rust 证明了它是可行的。

<!--more-->

## 回顾：并发的核心困难

前几篇我们看了各种并发模型：

| 模型 | 策略 | 安全保证方式 |
|------|------|--------------|
| 线程 + 锁 | 允许共享，用锁保护 | 程序员自律 + 运行时检测工具 |
| Actor | 禁止共享，只传消息 | 架构约束（但语言不强制） |
| CSP | 通过 Channel 通信 | 架构约束（但语言不强制） |

**共同点**：安全性依赖于程序员**正确使用**这些模型。写错了，编译器不会阻止你。

```text
这些模型的问题：

  线程 + 锁：
    忘记加锁 → 数据竞争 → 运行时未定义行为
    锁的顺序错 → 死锁 → 程序卡死

  Actor / CSP：
    不小心共享了可变状态 → 数据竞争（Go 允许这样做）
    消息中发送了共享引用 → 潜在风险

  共同问题：
    Bug 只有在运行时才能发现
    可能跑 1000 次才出一次
    测试很难覆盖所有并发场景
```

**Rust 的思路**：能不能在编译期就发现这些问题？

## Rust 的核心武器：所有权

### 所有权规则

Rust 的所有权系统是它一切魔法的基础：

```text
三条铁律：
  1. 每个值有且只有一个所有者（owner）
  2. 当所有者离开作用域，值被销毁
  3. 同一时间，要么有一个可变引用（&mut），要么有多个不可变引用（&）
```

```rust
// 规则 1 和 2：所有权转移
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // 所有权转移给 s2

    // println!("{}", s1);  // 编译错误！s1 已经无效
    println!("{}", s2);     // OK
}  // s2 离开作用域，字符串被释放

// 规则 3：借用规则
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;     // 不可变借用，OK
    let r2 = &s;     // 再来一个，OK
    // let r3 = &mut s; // 编译错误！已有不可变借用

    println!("{} {}", r1, r2);

    let r4 = &mut s;  // 现在可以了，r1 r2 不再使用
    r4.push_str(" world");
}
```

### 这和并发有什么关系？

**关键洞察**：数据竞争的本质是什么？

```text
数据竞争（Data Race）发生的条件：
  1. 多个线程访问同一数据
  2. 至少一个是写操作
  3. 没有同步机制

换个说法：
  1. 多个访问者
  2. 至少一个 &mut
  3. 没有协调

而 Rust 的借用规则正好禁止了这个：
  - 要么多个 &（都是读，安全）
  - 要么一个 &mut（只有一个写，安全）
  - 不允许 & 和 &mut 共存

如果把这个规则跨线程执行，数据竞争就不可能发生！
```

## Send 和 Sync：跨线程的守门人

Rust 用两个 trait 控制类型能否跨线程使用：

### Send：可以发送到其他线程

```rust
// Send 的含义：
// 这个类型的值可以安全地从一个线程转移到另一个线程

// 大多数类型是 Send 的
let v: Vec<i32> = vec![1, 2, 3];
std::thread::spawn(move || {
    println!("{:?}", v);  // v 被 move 到新线程，OK
});

// Rc<T> 不是 Send（引用计数非线程安全）
use std::rc::Rc;
let rc = Rc::new(42);
// std::thread::spawn(move || {
//     println!("{}", rc);  // 编译错误！Rc 不是 Send
// });
```

### Sync：可以被多线程共享引用

```rust
// Sync 的含义：
// 这个类型的引用（&T）可以安全地在多线程间共享

// 如果 T 是 Sync，那 &T 是 Send
// 即：如果类型可以安全共享引用，那引用可以发送到其他线程

// i32 是 Sync
let x: i32 = 42;
let r: &i32 = &x;
// r 可以在多线程共享

// Cell<T> 不是 Sync（内部可变性，非线程安全）
use std::cell::Cell;
let cell = Cell::new(42);
// &cell 不能在多线程共享
```

### 为什么这很重要？

```rust
// 尝试跨线程共享非 Sync 类型
use std::cell::RefCell;
use std::thread;

fn main() {
    let data = RefCell::new(42);

    let handle = thread::spawn(|| {
        // 尝试访问 data
        *data.borrow_mut() += 1;  // 编译错误！
    });
}

// 编译器错误信息：
// error[E0277]: `RefCell<i32>` cannot be shared between threads safely
//
// the trait `Sync` is not implemented for `RefCell<i32>`
```

**编译器在编译期就阻止了你犯错。**

## 线程安全的共享：Arc 和 Mutex

### Arc：原子引用计数

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    // Arc = Atomic Reference Counted
    // 线程安全的引用计数智能指针
    let data = Arc::new(vec![1, 2, 3]);

    let handles: Vec<_> = (0..3).map(|i| {
        let data = Arc::clone(&data);  // 克隆引用，不是克隆数据
        thread::spawn(move || {
            println!("Thread {}: {:?}", i, data);
        })
    }).collect();

    for h in handles {
        h.join().unwrap();
    }
}
```

**Arc vs Rc**：

| 特性 | Rc | Arc |
|------|-----|-----|
| 引用计数操作 | 普通整数操作 | 原子操作 |
| Send | 否 | 是 |
| Sync | 否 | 是 |
| 性能 | 更快 | 有原子操作开销 |
| 使用场景 | 单线程 | 多线程 |

### Mutex：互斥锁

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let handles: Vec<_> = (0..10).map(|_| {
        let counter = Arc::clone(&counter);
        thread::spawn(move || {
            // lock() 返回 MutexGuard，自动解锁
            let mut num = counter.lock().unwrap();
            *num += 1;
            // MutexGuard 离开作用域，自动解锁
        })
    }).collect();

    for h in handles {
        h.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap()); // 10
}
```

**Rust 的 Mutex 和其他语言的区别**：

```rust
// Rust：Mutex 保护的数据在 Mutex 里面
let m = Mutex::new(42);  // 数据被 Mutex 包裹
let data = m.lock().unwrap();  // 必须 lock 才能访问

// 其他语言（伪代码）：数据和锁是分开的
let data = 42;
let m = Mutex::new();
m.lock();
data += 1;  // 可能忘记加锁！
m.unlock();
```

**Rust 的设计让你无法在不加锁的情况下访问数据。**

### RwLock：读写锁

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3]));

    // 多个读者
    let readers: Vec<_> = (0..3).map(|i| {
        let data = Arc::clone(&data);
        thread::spawn(move || {
            let read_guard = data.read().unwrap();
            println!("Reader {}: {:?}", i, *read_guard);
        })
    }).collect();

    // 一个写者
    let writer = {
        let data = Arc::clone(&data);
        thread::spawn(move || {
            let mut write_guard = data.write().unwrap();
            write_guard.push(4);
            println!("Writer: {:?}", *write_guard);
        })
    };

    for r in readers {
        r.join().unwrap();
    }
    writer.join().unwrap();
}
```

## 无畏并发（Fearless Concurrency）

Rust 官方文档用「无畏并发」来描述 Rust 的并发编程体验：

```text
无畏并发的含义：

  传统并发：
    - 写代码时战战兢兢
    - 总担心忘记加锁
    - 总担心锁的顺序不对
    - 测试难以发现问题
    - 上线后惴惴不安

  Rust 并发：
    - 编译器帮你检查
    - 写错了编译不过
    - 能编译通过的代码，没有数据竞争
    - 可以放心大胆地写并发代码
```

### 一个例子：编译器救你于水火

```rust
use std::thread;

fn main() {
    let mut data = vec![1, 2, 3];

    // 尝试在多个线程中修改同一个数据
    let handle = thread::spawn(|| {
        data.push(4);  // 编译错误！
    });

    data.push(5);  // 主线程也想改

    handle.join().unwrap();
}

// 编译器错误：
// error[E0373]: closure may outlive the current function,
// but it borrows `data`, which is owned by the current function
//
// error[E0499]: cannot borrow `data` as mutable more than once at a time
```

**在其他语言里**，这段代码可能编译通过。运行时会怎样？
- 可能正常工作（取决于时序）
- 可能数据损坏（静默错误，最危险）
- 可能崩溃（还算好，至少知道有问题）
- 可能跑 1000 次才出一次问题

**在 Rust 里，编译器直接拒绝。** 你不需要担心"可能"。

## 数据竞争 vs 竞态条件

Rust 能防止数据竞争，但不能防止所有竞态条件：

```text
数据竞争（Data Race）：
  多线程同时读写同一内存位置，无同步
  Rust 编译期阻止

竞态条件（Race Condition）：
  程序结果依赖于执行顺序
  Rust 无法阻止（这是逻辑问题）

例子：
  // 没有数据竞争，但有竞态条件
  let counter = Arc::new(Mutex::new(0));

  // 线程 A
  if *counter.lock().unwrap() == 0 {
      // 线程 B 可能在这里修改 counter
      *counter.lock().unwrap() += 1;
  }

  // 结果依赖于 A 和 B 的执行顺序
  // Rust 无法阻止这种逻辑错误
```

**Rust 的承诺**：

| 类型 | 能防止吗 | 说明 |
|------|----------|------|
| 数据竞争 | 是 | 编译期检查（safe Rust） |
| 内存不安全 | 是 | 编译期检查（safe Rust） |
| 死锁 | 否 | 逻辑问题，需设计避免 |
| 竞态条件 | 否 | 逻辑问题 |
| 资源泄漏 | 部分 | 循环引用可能泄漏 |

**关于 `unsafe`**：
- 上述保证仅限于 **safe Rust**（不使用 `unsafe` 块的代码）
- `unsafe` 代码需要程序员手动保证安全性，编译器不检查
- 标准库和很多 crate 内部使用 `unsafe`，但对外暴露安全的 API
- 如果只使用安全的 API，你仍然受到 Rust 的保护

## Channel：Rust 的 CSP

Rust 标准库也提供了 Channel：

```rust
use std::sync::mpsc;  // multiple producer, single consumer
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    // 生产者线程
    let producer = thread::spawn(move || {
        for i in 0..10 {
            tx.send(i).unwrap();
        }
    });

    // 消费者（主线程）
    for received in rx {
        println!("Got: {}", received);
    }

    producer.join().unwrap();
}
```

### 多生产者

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    for i in 0..3 {
        let tx = tx.clone();  // 克隆发送端
        thread::spawn(move || {
            tx.send(format!("from thread {}", i)).unwrap();
        });
    }

    drop(tx);  // 关闭原始发送端

    for msg in rx {
        println!("{}", msg);
    }
}
```

### 所有权转移保证安全

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let data = vec![1, 2, 3];
        tx.send(data).unwrap();  // data 的所有权转移
        // println!("{:?}", data);  // 编译错误！data 已经被 move
    });

    let received = rx.recv().unwrap();
    println!("{:?}", received);  // 现在 received 拥有数据
}
```

**Channel 发送会转移所有权，发送后原线程无法访问数据。这从根本上防止了数据竞争。**

## async/await：异步与所有权

Rust 的异步模型我们在协程篇讲过，这里看它如何和所有权配合：

```rust
use std::sync::Arc;
use tokio::sync::Mutex;  // 异步版本的 Mutex

async fn process(data: Arc<Mutex<Vec<i32>>>) {
    let mut guard = data.lock().await;
    guard.push(42);
    // guard 离开作用域自动解锁
}

#[tokio::main]
async fn main() {
    let data = Arc::new(Mutex::new(vec![]));

    let handles: Vec<_> = (0..10).map(|_| {
        let data = Arc::clone(&data);
        tokio::spawn(async move {
            process(data).await;
        })
    }).collect();

    for h in handles {
        h.await.unwrap();
    }

    println!("{:?}", *data.lock().await);
}
```

**注意**：
- 如果需要**跨 `.await` 持有锁**，必须用 `tokio::sync::Mutex`
- 如果锁的持有时间很短且**不跨越 `.await`**，可以用 `std::sync::Mutex`（性能更好）
- 用 `std::sync::Mutex` 并在持有锁时 `.await` 会阻塞整个线程，破坏异步的优势

## 实际使用模式

### 模式一：内部可变性

```rust
use std::sync::{Arc, Mutex};

// 配置结构，需要在多线程共享和修改
struct Config {
    value: i32,
}

fn main() {
    let config = Arc::new(Mutex::new(Config { value: 0 }));

    // 修改配置
    {
        let mut c = config.lock().unwrap();
        c.value = 42;
    }

    // 读取配置
    println!("{}", config.lock().unwrap().value);
}
```

### 模式二：读多写少用 RwLock

```rust
use std::collections::HashMap;
use std::sync::RwLock;

struct Cache {
    data: RwLock<HashMap<String, String>>,
}

impl Cache {
    fn get(&self, key: &str) -> Option<String> {
        self.data.read().unwrap().get(key).cloned()
    }

    fn set(&self, key: String, value: String) {
        self.data.write().unwrap().insert(key, value);
    }
}
```

### 模式三：无锁数据结构

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicUsize::new(0));

    let handles: Vec<_> = (0..10).map(|_| {
        let counter = Arc::clone(&counter);
        thread::spawn(move || {
            for _ in 0..1000 {
                counter.fetch_add(1, Ordering::SeqCst);
            }
        })
    }).collect();

    for h in handles {
        h.join().unwrap();
    }

    println!("{}", counter.load(Ordering::SeqCst)); // 10000
}
```

## 对比其他语言

```text
Go：
  - 运行时检测竞争（-race flag）
  - 这是一个非常强大的工具，能发现很多问题
  - 局限：只能检测测试实际执行到的代码路径
  - 生产环境通常不开启（有 2-20x 性能开销）
  - 设计哲学：信任程序员 + 工具辅助

Java：
  - synchronized 和 volatile
  - JVM 保证内存模型语义
  - 有成熟的并发库（java.util.concurrent）
  - 局限：编译器不阻止误用

C++：
  - 有 std::mutex、std::atomic 等工具
  - 有 ThreadSanitizer 等检测工具
  - 局限：编译器不阻止误用，误用导致未定义行为

Rust：
  - 编译期检查（Send/Sync trait）
  - 错了就编译不过
  - 没有运行时开销
  - 代价：学习曲线陡峭

各有权衡：
  Go 选择了简单 + 工具检测
  Rust 选择了编译期保证 + 复杂性
  Java/C++ 提供工具但不强制使用
```

## 总结

**Rust 的并发哲学**：

```text
核心思想：
  用类型系统和所有权规则
  在编译期消灭数据竞争

实现方式：
  1. 所有权：同一时间只有一个可变引用
  2. Send/Sync：控制类型能否跨线程
  3. 智能指针：Arc、Mutex、RwLock
  4. 编译器检查：写错了就不让编译
```

**和其他模型的关系**：

| 模型 | Rust 的态度 |
|------|-------------|
| 线程 + 锁 | 支持，但用类型系统保护 |
| 消息传递 | 支持，Channel + 所有权转移 |
| Actor | 可以实现（actix 框架） |
| async/await | 支持，无栈协程 + 所有权 |

**Rust 不是另一种并发模型，而是给所有模型加了一层编译期检查。**

**代价是什么？**

| 优点 | 代价 |
|------|------|
| 编译期发现错误 | 学习曲线陡峭 |
| 无运行时开销 | 编译时间长 |
| 无 GC | 需要理解所有权 |
| 细粒度控制 | 代码可能啰嗦 |

最后一篇，我们会把所有模型放在一起，看实际项目中如何选择——没有银弹，只有场景。

---

*上一篇：[CSP 模型：Channel 是一等公民](/posts/concurrency-models-5-csp/)*

*下一篇：[实战选型：没有银弹，只有场景](/posts/concurrency-models-7-practice/)*

*本系列：*
1. *[为什么并发这么难？](/posts/concurrency-models-1-why-hard/)*
2. *[线程与锁：最直接也最危险](/posts/concurrency-models-2-threads-locks/)*
3. *[协程：用户态的轻量级线程](/posts/concurrency-models-3-coroutines/)*
4. *[Actor 模型：不要共享，要传递](/posts/concurrency-models-4-actor/)*
5. *[CSP 模型：Channel 是一等公民](/posts/concurrency-models-5-csp/)*
6. *Rust 的第三条路：编译期消灭数据竞争（本篇）*
7. *[实战选型：没有银弹，只有场景](/posts/concurrency-models-7-practice/)*
