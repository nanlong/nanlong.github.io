---
title: "并发模型演进史（三）：协程——用户态的轻量级线程"
date: 2025-12-08T22:20:00+08:00
draft: false
tags: ["并发", "协程", "async/await", "goroutine", "异步编程"]
categories: ["技术"]
description: "线程太重，一个服务器撑不了几万个连接。于是我们发明了协程——一种假装是线程的函数。它轻到可以创建百万个，但代价是：你得重新学一套编程模型。"
series: ["并发模型演进史"]
---

线程太重，一个服务器撑不了几万个连接。于是我们发明了协程——一种假装是线程的函数。它轻到可以创建百万个，但代价是：你得重新学一套编程模型。

<!--more-->

## C10K 问题：线程的极限

1999 年，Dan Kegel 提出了一个问题：**如何让一台服务器同时处理 10000 个客户端连接？**

这就是著名的 **C10K 问题**。

```text
传统模型：一个连接一个线程

  客户端 1 ──── 线程 1
  客户端 2 ──── 线程 2
  客户端 3 ──── 线程 3
  ...
  客户端 10000 ──── 线程 10000

问题：
  - 每个线程占用 1-8MB 栈空间
  - 10000 个线程 = 10-80GB 内存
  - 线程切换开销巨大
  - 操作系统调度器崩溃
```

**线程不是免费的。** 每个线程都有成本：

| 资源 | 消耗 | 说明 |
|------|------|------|
| 栈空间 | 1-8MB | 默认栈大小，大部分用不完 |
| 内核对象 | ~1KB | 线程控制块 |
| 上下文切换 | ~1-10μs | 保存/恢复寄存器、刷新缓存 |
| 调度开销 | O(n) | 线程越多，调度越慢 |

**1999 年的服务器**：512MB 内存，能开几百个线程就不错了。

**于是人们开始思考**：能不能有一种更轻量的「线程」？

## 协程：用户态的魔法

协程（Coroutine）的核心思想很简单：

> **既然线程切换这么贵，那我们自己切换，不麻烦操作系统。**

```text
线程切换（内核态）：
  1. 陷入内核（系统调用）
  2. 保存当前线程状态
  3. 调度器选择下一个线程
  4. 恢复下一个线程状态
  5. 返回用户态

  代价：~1000-10000 CPU 周期（取决于 OS 和硬件）

协程切换（用户态）：
  1. 保存当前协程状态（几个寄存器）
  2. 恢复下一个协程状态
  完了。

  代价：~100 CPU 周期（取决于实现）
```

**通常快一到两个数量级**（具体取决于操作系统、硬件、协程实现方式）。

### 协程 vs 线程：类比

```text
线程 = 餐厅服务员
  - 由餐厅经理（OS）调度
  - 每个服务员有自己的工具车（栈空间）
  - 换人服务要走完整交接流程

协程 = 一个服务员分饰多角
  - 自己决定什么时候换角色
  - 共用一套工具
  - 换角色只需要换个帽子
```

### 协程的本质

协程本质上是**可以暂停和恢复的函数**。

```text
普通函数：
  调用 → 执行完 → 返回
  中间不能停

协程函数：
  调用 → 执行一部分 → 暂停（yield）→ 被恢复 → 继续执行 → 暂停 → ... → 返回
  中间可以停很多次
```

用代码说：

```python
# Python 生成器协程
def count_up():
    i = 0
    while True:
        yield i      # 暂停，返回 i
        i += 1       # 恢复后从这里继续

counter = count_up()
print(next(counter))  # 0
print(next(counter))  # 1
print(next(counter))  # 2
```

**关键**：`yield` 暂停函数，但保留了所有局部变量的状态。下次恢复时，从 `yield` 的下一行继续执行。

## 两种协程：有栈 vs 无栈

协程实现分两大流派：

### 有栈协程（Stackful）

每个协程有独立的调用栈，可以在任意位置暂停。

```text
有栈协程：
  ┌─────────────┐
  │ 协程 A 的栈  │  ← 独立栈空间（通常 2-8KB）
  │  - 局部变量  │
  │  - 返回地址  │
  │  - 调用链    │
  └─────────────┘

  ┌─────────────┐
  │ 协程 B 的栈  │
  │  - 局部变量  │
  │  - 返回地址  │
  │  - 调用链    │
  └─────────────┘

代表：Go goroutine, Lua coroutine
```

**优点**：
- 可以在任何函数深度暂停
- 编程模型直观，和线程相似
- 不需要 async/await 语法

**缺点**：
- 每个协程需要预分配栈空间
- 栈空间管理复杂（可能溢出或浪费）

### 无栈协程（Stackless）

协程没有独立的栈，状态保存在堆上的状态机中。

```text
无栈协程：
  async fn fetch_data() {
      let a = step1().await;    // 暂停点 1
      let b = step2(a).await;   // 暂停点 2
      process(a, b)
  }

  编译成状态机：
  ┌─────────────────────────────┐
  │ FetchDataFuture             │
  │  state: enum { S1, S2, S3 } │  ← 当前状态
  │  a: Option<A>               │  ← 跨 await 的变量
  │  b: Option<B>               │
  └─────────────────────────────┘

代表：Rust async/await, JavaScript Promise, Python asyncio
```

**优点**：
- 状态机大小精确（编译器计算）
- 无栈空间浪费
- 可以零成本抽象

**缺点**：
- 只能在 `await` 点暂停
- async 会「传染」（调用 async 函数的也得是 async）
- 编程模型需要适应

### 对比表

| 特性 | 有栈协程 | 无栈协程 |
|------|----------|----------|
| 栈空间 | 每协程 2-8KB | 精确按需 |
| 暂停位置 | 任意 | 仅 await 点 |
| 语法 | 普通函数 | async/await |
| 传染性 | 无 | 有（async 传染） |
| 性能 | 好 | 更好 |
| 代表 | Go | JS, Rust (async) |

**注意**：Rust 出现在无栈协程列，是因为它的 async/await 采用了无栈协程实现。但 async 只是 Rust 并发工具箱中的一个选项，Rust 同样支持（且广泛使用）传统的锁和 Channel，并用所有权系统保证它们的安全性。

## Go 的选择：Goroutine

Go 语言选择了有栈协程，并把它发挥到了极致。

### Goroutine 的设计

```go
// 创建一个 goroutine，就这么简单
go func() {
    // 这里是另一个执行流
    fmt.Println("Hello from goroutine")
}()
```

**Goroutine 的特点**：

```text
1. 极小的初始栈（Go 1.4+ 为 2KB，可动态增长）
   普通线程：1-8MB（取决于 OS 配置）
   goroutine：2KB 起步（早期版本是 4KB/8KB）
   差距：数百到数千倍

2. 可以创建百万个
   func main() {
       for i := 0; i < 1_000_000; i++ {
           go func() {
               time.Sleep(time.Hour)
           }()
       }
   }
   // 用不了多少内存

3. M:N 调度
   M 个 goroutine 映射到 N 个 OS 线程
   Go runtime 负责调度
```

### GMP 调度模型

Go 的调度器是它的核心竞争力：

```text
GMP 模型：
  G = Goroutine（协程）
  M = Machine（OS 线程）
  P = Processor（逻辑处理器）

  ┌─────────────────────────────────┐
  │           Go Runtime            │
  │  ┌─────┐  ┌─────┐  ┌─────┐     │
  │  │  P  │  │  P  │  │  P  │     │  ← P 的数量 = GOMAXPROCS
  │  └──┬──┘  └──┬──┘  └──┬──┘     │
  │     │        │        │         │
  │  ┌──┴──┐  ┌──┴──┐  ┌──┴──┐     │
  │  │  M  │  │  M  │  │  M  │     │  ← M 绑定到 OS 线程
  │  └─────┘  └─────┘  └─────┘     │
  └─────────────────────────────────┘

每个 P 有一个本地队列：
  P1: [G1, G2, G3, G4]
  P2: [G5, G6]
  P3: [G7, G8, G9]

  还有一个全局队列：
  Global: [G10, G11, G12, ...]
```

**调度触发时机**：

1. **go 关键字**：创建新 goroutine
2. **channel 操作**：阻塞时让出
3. **系统调用**：阻塞时让出
4. **GC**：需要暂停时
5. **主动让出**：`runtime.Gosched()`

### Go 调度的精妙之处

```text
work stealing（工作窃取）：

  P1 的队列空了：[空]
  P2 的队列满：[G1, G2, G3, G4]

  P1 会从 P2 偷一半任务：
  P1: [G3, G4]
  P2: [G1, G2]

  负载自动均衡

系统调用处理：

  G1 要做系统调用（比如读文件）
    │
    ├─ M1 带着 G1 去做系统调用
    │
    └─ P1 找另一个 M2 继续执行其他 G
       不让 P1 闲着

  系统调用结束后：
    M1 尝试获取空闲的 P
    如果没有，G1 放入全局队列
```

**这就是为什么 Go 适合 I/O 密集型应用**：goroutine 阻塞时，OS 线程不阻塞。

## Rust 的异步方案：async/await

Rust 在**异步 I/O** 场景选择了无栈协程，追求零成本抽象。

**重要澄清**：async/await 只是 Rust 并发工具箱的一部分，主要用于 I/O 密集型场景。对于共享状态的并发访问，Rust 程序员同样（甚至更多地）使用 `Mutex`、`RwLock`、`Channel` 等传统工具——只是这些工具在 Rust 中有所有权系统的保护。

### async/await 的本质

```rust
// 这个 async 函数
async fn fetch_user(id: u64) -> User {
    let data = fetch_from_db(id).await;
    let user = parse_user(data).await;
    user
}

// 编译器转换成这样的状态机
enum FetchUserFuture {
    Start { id: u64 },
    WaitingDb { id: u64, fetch_future: DbFuture },
    WaitingParse { data: Data, parse_future: ParseFuture },
    Done,
}

impl Future for FetchUserFuture {
    type Output = User;

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<User> {
        loop {
            match self.state {
                Start { id } => {
                    let future = fetch_from_db(id);
                    self.state = WaitingDb { id, fetch_future: future };
                }
                WaitingDb { fetch_future, .. } => {
                    match fetch_future.poll(cx) {
                        Poll::Pending => return Poll::Pending,
                        Poll::Ready(data) => {
                            let future = parse_user(data);
                            self.state = WaitingParse { data, parse_future: future };
                        }
                    }
                }
                // ...
            }
        }
    }
}
```

**关键点**：
- `async fn` 返回的不是结果，而是一个 `Future`
- `Future` 只有被 `poll` 时才执行
- `.await` 是暂停点，编译器在这里切分状态

### Rust async 的优势

```text
1. 零成本抽象
   async 函数编译后，和手写状态机一样高效
   没有运行时开销

2. 精确的内存占用
   编译器精确计算 Future 大小
   不需要预分配栈

3. 无 GC 配合所有权
   异步代码也受所有权规则保护
   编译期防止数据竞争

4. 可选的运行时
   std 不包含 async 运行时
   可以选择 tokio, async-std 等
   甚至可以在嵌入式环境用
```

### async 的传染性问题

```rust
// 一个 async 函数
async fn async_operation() -> i32 {
    42
}

// 调用它的函数也得是 async
async fn caller() -> i32 {
    async_operation().await
}

// 一路传染上去
async fn main_logic() -> i32 {
    caller().await
}

// 直到 main 或运行时边界
#[tokio::main]
async fn main() {
    let result = main_logic().await;
    println!("{}", result);
}
```

**传染性**：调用 async 函数的函数也必须是 async，一路传染到入口点。

这是无栈协程的固有限制，也是设计权衡。

## 各语言对比

### Go：隐式并发

```go
func main() {
    // 创建协程，用 go 关键字
    go doSomething()

    // channel 通信
    ch := make(chan int)
    go func() {
        ch <- 42
    }()
    result := <-ch
}
```

**特点**：
- 语法简单，隐藏了并发复杂性
- 有栈协程，任意位置可暂停
- 强制使用 channel 通信（CSP 风格）
- GC 管理内存

### Rust：显式并发

```rust
#[tokio::main]
async fn main() {
    // 创建异步任务
    let handle = tokio::spawn(async {
        do_something().await
    });

    // 等待完成
    let result = handle.await.unwrap();
}
```

**特点**：
- 显式 async/await，代码意图清晰
- 无栈协程，编译期确定开销
- 可选运行时（tokio, async-std）
- 所有权系统防止数据竞争

### Python：老语言的新特性

```python
import asyncio

async def main():
    # 创建任务
    task = asyncio.create_task(do_something())

    # 等待完成
    result = await task

asyncio.run(main())
```

**特点**：
- async/await 语法（Python 3.5+）
- 单线程事件循环
- 不适合 CPU 密集型（GIL 限制）
- 生态丰富（aiohttp, asyncpg 等）

### JavaScript：事件循环老玩家

```javascript
async function main() {
    // Promise 和 async/await
    const result = await fetch('/api/data');
    const data = await result.json();
    return data;
}

main().then(console.log);
```

**特点**：
- 单线程事件循环（天生异步）
- Promise 是语言核心
- async/await 是 Promise 的语法糖
- 适合 I/O 密集型（Web 服务）

### 对比表（仅比较异步/协程特性）

| 特性 | Go | Rust | Python | JavaScript |
|------|-----|------|--------|------------|
| 协程类型 | 有栈 | 无栈 | 无栈 | 无栈 |
| 异步语法 | go/channel | async/await | async/await | async/await |
| 异步运行时 | 内置 | 可选(tokio等) | 内置 | 内置 |
| 内存管理 | GC | 所有权 | GC | GC |
| 多线程 | 原生支持 | 原生支持 | GIL 限制 | Worker 线程 |
| 适合场景 | 通用 | 系统/Web | I/O 密集 | Web |

**再次强调**：此表仅比较各语言的异步/协程特性。Rust 的并发能力远不止 async/await，同样支持 `thread::spawn`、`Mutex`、`mpsc::channel` 等传统并发原语。

## 协程的陷阱

### 陷阱一：阻塞调用

```rust
// 错误：在 async 中调用阻塞函数
async fn bad_example() {
    // 这会阻塞整个线程！
    std::thread::sleep(Duration::from_secs(1));
}

// 正确：使用异步版本
async fn good_example() {
    tokio::time::sleep(Duration::from_secs(1)).await;
}
```

**问题**：协程运行在少数几个 OS 线程上。一个协程阻塞了线程，其他协程都得等。

```text
tokio 默认配置：
  - 4 个 worker 线程（CPU 核心数）
  - 运行 10000 个 async 任务

如果一个任务阻塞了线程：
  - 只剩 3 个线程干活
  - 吞吐量下降 25%

如果 4 个任务同时阻塞：
  - 所有任务都卡住
  - 系统假死
```

**解决方案**：
- 使用异步版本的 API
- CPU 密集任务用 `spawn_blocking`
- 或者增加 worker 线程数

### 陷阱二：async 传染

```rust
// 你想在同步代码中调用 async 函数
fn sync_function() {
    // 这编译不过！
    // let result = async_function().await;

    // 需要用 block_on
    let result = tokio::runtime::Runtime::new()
        .unwrap()
        .block_on(async_function());
}
```

**问题**：整个调用链都得是 async，或者用 block_on 打断。

**现实**：大型代码库迁移到 async 是渐进的，边界处理很痛苦。

### 陷阱三：取消安全

```rust
async fn transfer(from: &Account, to: &Account, amount: u64) {
    from.deduct(amount).await;
    // 如果这里被取消...
    to.add(amount).await;  // 这行不会执行
    // 钱就丢了！
}
```

**问题**：async 任务可以被取消（比如超时），取消点在每个 `.await`。

**解决方案**：
- 设计幂等操作
- 使用事务
- 小心处理取消

## 什么时候用协程

### 适合的场景

| 场景 | 原因 |
|------|------|
| Web 服务 | 大量连接，I/O 为主 |
| API 网关 | 高并发，低计算 |
| 爬虫 | 大量 HTTP 请求 |
| 数据库代理 | 连接复用 |
| 即时通讯 | 大量长连接 |

**共同特点**：I/O 密集、高并发、连接多。

### 不适合的场景

| 场景 | 原因 |
|------|------|
| 科学计算 | CPU 密集，协程帮不上忙 |
| 视频编码 | CPU 密集 |
| 简单脚本 | 复杂度不值得 |
| 延迟敏感 | 调度有抖动 |

**共同特点**：CPU 密集、简单场景、极致延迟。

## 总结

**协程解决了什么问题？**

```text
线程的问题：
  - 创建成本高（MB 级栈空间）
  - 切换成本高（内核态切换）
  - 数量有限（几千个就很多了）

协程的解决方案：
  - 用户态切换，快 10-100 倍
  - 更小的栈或无栈，可创建百万个
  - 程序自己调度，更可控
```

**但协程不是银弹**：

| 问题 | 协程解决了吗 |
|------|-------------|
| 高并发 I/O | 解决 |
| 线程切换开销 | 解决 |
| 共享状态并发 | **没解决** |
| 竞态条件 | **没解决** |
| 死锁 | **没解决** |

**协程只是让并发更轻量，不是让并发更安全。**

要解决「共享状态」这个根本问题，需要从编程模型入手。

下一篇，我们看 Actor 模型——它的解决方案很激进：**直接禁止共享**。

---

*上一篇：[线程与锁：最直接也最危险](/posts/concurrency-models-2-threads-locks/)*

*下一篇：[Actor 模型：不要共享，要传递](/posts/concurrency-models-4-actor/)*

*本系列：*
1. *[为什么并发这么难？](/posts/concurrency-models-1-why-hard/)*
2. *[线程与锁：最直接也最危险](/posts/concurrency-models-2-threads-locks/)*
3. *协程：用户态的轻量级线程（本篇）*
4. *[Actor 模型：不要共享，要传递](/posts/concurrency-models-4-actor/)*
5. *[CSP 模型：Channel 是一等公民](/posts/concurrency-models-5-csp/)*
6. *[Rust 的第三条路：编译期消灭数据竞争](/posts/concurrency-models-6-rust/)*
7. *[实战选型：没有银弹，只有场景](/posts/concurrency-models-7-practice/)*
