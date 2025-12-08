---
title: "并发模型演进史（二）：线程与锁，最直接也最危险"
date: 2025-12-08T22:10:00+08:00
draft: false
tags: ["并发", "多线程", "锁", "死锁", "互斥锁"]
categories: ["技术"]
description: "锁是治疗竞态条件的药，但剂量不对会变成毒。粒度太粗，性能完蛋；粒度太细，死锁等着你。欢迎来到锁的地狱。"
series: ["并发模型演进史"]
---

锁是治疗竞态条件的药，但剂量不对会变成毒。粒度太粗，性能完蛋；粒度太细，死锁等着你。欢迎来到锁的地狱。

<!--more-->

## 前情回顾

上一篇我们看到了并发的三大噩梦：

- **竞态条件**：两个线程同时改一个变量，结果不可预测
- **死锁**：互相等待，谁也动不了
- **内存可见性**：我改了，你看不到

今天我们来看人类的第一个解决方案：**线程 + 锁**。

## 线程：操作系统的礼物

操作系统提供了一个抽象：**线程**。

```text
进程 vs 线程：

进程（Process）：
  ┌─────────────────────────────┐
  │  代码  │  数据  │  堆  │  栈  │
  └─────────────────────────────┘
  独立的内存空间，进程间不共享

线程（Thread）：
  ┌─────────────────────────────────────┐
  │          进程的内存空间              │
  │  ┌─────┐  ┌─────┐  ┌─────┐         │
  │  │栈 1 │  │栈 2 │  │栈 3 │         │
  │  │线程1│  │线程2│  │线程3│  共享堆  │
  │  └─────┘  └─────┘  └─────┘         │
  └─────────────────────────────────────┘
  共享内存空间，各有独立的栈
```

**线程的好处**：

- 比进程轻量（创建、销毁、切换更快）
- 共享内存，通信方便
- 能利用多核 CPU

**线程的坏处**：

- 共享内存……这正是问题所在

## 锁：最直观的解决方案

既然问题是「多个线程同时访问共享状态」，那最直接的想法就是：**一次只让一个线程访问**。

这就是**互斥锁（Mutex）**的思想。

```text
类比：公共厕所

没有锁：
  两个人同时冲进去，尴尬

有锁：
  门上有锁，一个人进去锁门
  另一个人只能在外面等
  里面的人出来，下一个人才能进

互斥锁 = 厕所门锁
临界区 = 厕所里面
```

### 互斥锁的使用

```rust
use std::sync::Mutex;

static COUNTER: Mutex<i32> = Mutex::new(0);

fn increment() {
    let mut count = COUNTER.lock().unwrap();  // 进厕所，锁门
    *count += 1;                               // 临界区：独享
    // lock guard 离开作用域自动释放锁        // 开门，出来
}
```

现在两个线程同时调 increment()，count 一定是 2。

**为什么？** 因为 `lock()` 返回的 `MutexGuard` 保证了：同一时刻只有一个线程能访问被保护的数据。

**Rust 的优势**：锁和数据绑定在一起（`Mutex<i32>`），你无法在不获取锁的情况下访问数据。而 Go/Java/C++ 中锁和数据是分离的，忘记加锁编译器不会报错。

```text
时间线（有锁）：
  T1: 线程 A 调用 lock()，获取锁 ✓
  T2: 线程 B 调用 lock()，等待...
  T3: 线程 A 读取 count = 0
  T4: 线程 A 写入 count = 1
  T5: 线程 A 的 guard 离开作用域，释放锁
  T6: 线程 B 获取锁 ✓
  T7: 线程 B 读取 count = 1
  T8: 线程 B 写入 count = 2
  T9: 线程 B 的 guard 离开作用域，释放锁

结果：count = 2 ✓
```

**问题解决了？** 没那么简单。

## 锁的艺术：粒度之争

### 粗粒度锁：简单但慢

```rust
use std::sync::Mutex;

struct Account {
    // 一把锁保护所有数据
    data: Mutex<AccountData>,
}

struct AccountData {
    balance: i64,
    // ... 很多其他字段
}

impl Account {
    fn do_anything(&self) {
        let _data = self.data.lock().unwrap();
        // 所有操作都在锁里
    }
}
```

**一把大锁保护所有东西。**

```text
优点：简单，不会死锁
缺点：性能差

场景：账户有 10 个方法，都用同一把锁
  - 查余额的人要等转账的人
  - 查流水的人要等查余额的人
  - 所有操作串行执行

  → 多线程变成了"排队单线程"
```

### 细粒度锁：快但危险

```rust
use std::sync::Mutex;

struct Account {
    balance: Mutex<i64>,      // 余额独立一把锁
    logs: Mutex<Vec<String>>, // 日志独立一把锁
}

impl Account {
    fn transfer(&self, amount: i64) {
        {
            let mut balance = self.balance.lock().unwrap();
            *balance -= amount;
        } // balance 锁在这里释放

        {
            let mut logs = self.logs.lock().unwrap();
            logs.push("transferred".to_string());
        } // logs 锁在这里释放
    }

    fn get_balance(&self) -> i64 {
        *self.balance.lock().unwrap()
    }
}
```

**每个字段一把锁。**

```text
优点：并发度高
  - 查余额和记日志可以同时进行
  - 不同操作互不阻塞

缺点：死锁风险
  - 如果 A 方法先锁 balanceMu 再锁 logMu
  - B 方法先锁 logMu 再锁 balanceMu
  - 💥 死锁
```

### 锁粒度的权衡

| 粒度 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 粗（一把大锁） | 简单、安全 | 性能差 | 低并发、简单场景 |
| 细（多把小锁） | 性能好 | 复杂、易死锁 | 高并发、性能敏感 |

**实践建议**：

> 从粗粒度开始，遇到性能问题再细化。
> 过早优化是万恶之源，过早细化锁是死锁之源。

## 死锁：锁的终极陷阱

### 死锁的四个必要条件

1978 年，Coffman 等人总结了死锁的四个必要条件：

| 条件 | 含义 | 类比 |
|------|------|------|
| 互斥 | 资源同时只能被一个线程持有 | 厕所只能进一个人 |
| 持有并等待 | 持有资源的同时等待其他资源 | 占着茅坑等卫生纸 |
| 不可剥夺 | 资源只能被持有者主动释放 | 不能把人从厕所里拖出来 |
| 循环等待 | 形成等待环路 | A 等 B，B 等 C，C 等 A |

**四个条件同时满足，死锁就会发生。**

打破任何一个条件，就能避免死锁。

### 实战：银行转账

最经典的死锁场景：A 给 B 转账的同时，B 给 A 转账。

```rust
use std::sync::Mutex;

struct Account {
    id: u64,
    balance: Mutex<i64>,
}

fn transfer(from: &Account, to: &Account, amount: i64) {
    let mut from_balance = from.balance.lock().unwrap();
    // 假设这里有一些耗时操作...
    let mut to_balance = to.balance.lock().unwrap();

    *from_balance -= amount;
    *to_balance += amount;
}
```

两个线程同时执行：

```text
线程 1: transfer(&A, &B, 100)
线程 2: transfer(&B, &A, 50)

时间线：
  T1: 线程 1 锁定 A ✓
  T2: 线程 2 锁定 B ✓
  T3: 线程 1 尝试锁定 B... 等待
  T4: 线程 2 尝试锁定 A... 等待

  💥 死锁
```

### 解决方案一：固定加锁顺序

**打破循环等待条件**：所有人按相同顺序加锁。

```rust
fn transfer(from: &Account, to: &Account, amount: i64) {
    // 按账户 ID 排序，小的先锁
    let (first, second) = if from.id < to.id {
        (from, to)
    } else {
        (to, from)
    };

    let mut first_balance = first.balance.lock().unwrap();
    let mut second_balance = second.balance.lock().unwrap();

    // 注意：这里需要判断哪个是 from 哪个是 to
    if from.id < to.id {
        *first_balance -= amount;
        *second_balance += amount;
    } else {
        *second_balance -= amount;
        *first_balance += amount;
    }
}
```

现在无论谁先执行，加锁顺序都一样，不会死锁。

### 解决方案二：超时机制

**打破持有并等待条件**：等不到就放弃。

```rust
use std::sync::Mutex;

fn transfer(from: &Account, to: &Account, amount: i64) -> Result<(), &'static str> {
    // try_lock 立即返回，不阻塞
    let Ok(mut from_balance) = from.balance.try_lock() else {
        return Err("failed to lock from account");
    };

    let Ok(mut to_balance) = to.balance.try_lock() else {
        // 获取 to 失败，from 的锁会自动释放（RAII）
        return Err("failed to lock to account");
    };

    *from_balance -= amount;
    *to_balance += amount;
    Ok(())
}
```

**Rust 的优势**：RAII 保证了锁一定会被释放。即使 `try_lock` 失败提前返回，已获取的锁也会自动释放，不会泄漏。

**注意**：标准库的 `try_lock` 是非阻塞的，不支持超时。如果需要超时机制，可以使用 `parking_lot` crate 提供的 `try_lock_for`。

### 解决方案三：一次性获取所有锁

**打破持有并等待条件**：要么全拿到，要么全不拿。

```rust
use std::thread;
use std::time::Duration;
use rand::Rng;

fn transfer(from: &Account, to: &Account, amount: i64) {
    loop {
        // 尝试获取第一把锁
        let Ok(mut from_balance) = from.balance.try_lock() else {
            backoff();
            continue;
        };

        // 尝试获取第二把锁
        let Ok(mut to_balance) = to.balance.try_lock() else {
            // 获取失败，from_balance 会自动释放（RAII）
            backoff();
            continue;
        };

        // 全部获取成功，执行转账
        *from_balance -= amount;
        *to_balance += amount;
        return;
    }
}

fn backoff() {
    let mut rng = rand::thread_rng();
    let millis = rng.gen_range(0..10);
    thread::sleep(Duration::from_millis(millis));
}
```

**Rust 的优势**：当 `try_lock` 失败时，之前获取的锁会自动释放（RAII），不需要手动管理。

## 其他锁类型

互斥锁不是唯一的选择。

### 读写锁（RWMutex）

```text
场景：读多写少

互斥锁：
  读也要独占，10 个读者只能排队

读写锁：
  读者之间不互斥，可以同时读
  写者和所有人互斥

  ┌────────────────────────────────┐
  │  读者1  读者2  读者3  (同时)    │
  └────────────────────────────────┘
             ↓ 写者来了
  ┌────────────────────────────────┐
  │         写者 (独占)            │
  └────────────────────────────────┘
```

```rust
use std::sync::RwLock;
use std::collections::HashMap;

static DATA: RwLock<HashMap<String, String>> = RwLock::new(HashMap::new());

fn read(key: &str) -> Option<String> {
    let data = DATA.read().unwrap();  // 读锁，允许并发读
    data.get(key).cloned()
}

fn write(key: String, value: String) {
    let mut data = DATA.write().unwrap();  // 写锁，独占
    data.insert(key, value);
}
```

| 锁类型 | 读-读 | 读-写 | 写-写 |
|--------|-------|-------|-------|
| 互斥锁 | 互斥 | 互斥 | 互斥 |
| 读写锁 | 不互斥 | 互斥 | 互斥 |

### 自旋锁（Spinlock）

```text
普通锁：
  获取失败 → 进入睡眠 → 操作系统调度 → 被唤醒

自旋锁：
  获取失败 → 原地循环重试 → 获取成功

类比：
  普通锁 = 去休息室等叫号
  自旋锁 = 站在门口不停问"好了没"
```

**自旋锁适合**：临界区很短，锁很快就会释放。避免线程切换的开销。

**自旋锁不适合**：临界区很长。白白浪费 CPU。

## Java 的演进：从 synchronized 到 Lock

Java 的锁演进是理解锁复杂性的好例子。

### 早期：synchronized

```java
public synchronized void increment() {
    count++;
}
```

**问题**：
- 不能中断等待中的线程
- 不能设置超时
- 不能尝试获取锁

### Java 5：Lock 接口

```java
Lock lock = new ReentrantLock();

public void increment() {
    if (lock.tryLock(1, TimeUnit.SECONDS)) {
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

**改进**：tryLock、超时、可中断。

### Java 6+：锁优化

JVM 对 synchronized 做了大量优化：

| 优化 | 说明 |
|------|------|
| 偏向锁 | 假设只有一个线程，几乎无开销 |
| 轻量级锁 | 通过 CAS 自旋，避免操作系统调用 |
| 重量级锁 | 真的有竞争时才用操作系统锁 |
| 锁消除 | 编译器发现不需要锁，直接去掉 |
| 锁粗化 | 多次加锁合并成一次 |

**教训**：不要过早优化。JVM 比你聪明。

## 常见问题

### Q：锁的性能开销大吗？

**A：看情况。**

- **无竞争的锁**：几乎没有开销（可能被优化掉）
- **轻度竞争**：自旋等待，微秒级
- **重度竞争**：线程切换，毫秒级

关键不是锁本身，而是**锁的竞争程度**。

### Q：用了锁就一定安全吗？

**A：不一定。**

常见错误：

```rust
// 错误：锁了但没保护到
fn get_and_increment() -> i32 {
    let v = {
        let count = COUNTER.lock().unwrap();
        *count
    }; // 锁在这里释放了！

    {
        let mut count = COUNTER.lock().unwrap();
        *count = v + 1;  // v 可能已经过时，其他线程可能已经修改了
    }

    v
}
```

整个读-改-写操作必须在一把锁里：

```rust
// 正确：整个操作在同一把锁里
fn get_and_increment() -> i32 {
    let mut count = COUNTER.lock().unwrap();
    let v = *count;
    *count = v + 1;
    v
}
```

### Q：能不能不用锁？

**A：有些场景可以。**

- **原子操作**：CPU 提供的原子指令（CAS）
- **无锁数据结构**：利用原子操作构建
- **Actor/CSP**：根本不共享内存

后面的文章会讲这些替代方案。

## 总结

**线程 + 锁的本质**：

> 允许共享内存，但用锁串行化访问。
> 把"并行问题"变成"排队问题"。

**优点**：

| 优点 | 说明 |
|------|------|
| 直观 | 符合人类思维，"独占"容易理解 |
| 灵活 | 可以精确控制哪些操作需要互斥 |
| 普遍 | 所有语言都支持 |

**缺点**：

| 缺点 | 说明 |
|------|------|
| 死锁 | 多把锁容易死锁 |
| 性能 | 高竞争时性能差 |
| 可组合性差 | 两个正确的锁操作组合起来可能出错 |
| 容易出错 | 忘记加锁、加错锁、锁错范围 |

**核心认知**：

> 锁是必要的恶。
> 它解决了竞态条件，但引入了新的复杂性。
> 锁越多，出错的可能性越大。

---

线程 + 锁的最大问题是：**成本太高**。

一个线程要 1MB 栈空间，创建销毁要微秒级开销。如果你要同时处理 10000 个连接，开 10000 个线程？内存先爆了。

有没有更轻量的方案？

下一篇，我们来看**协程**——用户态的轻量级线程。

---

*上一篇：[为什么并发这么难](/posts/concurrency-models-1-why-hard/)*

*下一篇：[协程：用户态的轻量级线程](/posts/concurrency-models-3-coroutines/)*

*本系列：*
1. *[为什么并发这么难](/posts/concurrency-models-1-why-hard/)*
2. *线程与锁：最直接也最危险（本篇）*
3. *[协程：用户态的轻量级线程](/posts/concurrency-models-3-coroutines/)*
4. *[Actor 模型：不要共享，要传递](/posts/concurrency-models-4-actor/)*
5. *[CSP 模型：Channel 是一等公民](/posts/concurrency-models-5-csp/)*
6. *[Rust 的第三条路：编译期消灭数据竞争](/posts/concurrency-models-6-rust/)*
7. *[实战选型：没有银弹，只有场景](/posts/concurrency-models-7-practice/)*
