---
title: "并发模型演进史（七）：实战选型——没有银弹，只有场景"
date: 2025-12-08T23:00:00+08:00
draft: false
tags: ["并发", "架构设计", "技术选型", "最佳实践"]
categories: ["技术"]
description: "学了六种并发模型，该用哪个？答案是：看场景。没有最好的模型，只有最适合的。这篇文章给你一个决策框架，帮你在实际项目中做出选择。"
series: ["并发模型演进史"]
---

学了六种并发模型，该用哪个？答案是：看场景。没有最好的模型，只有最适合的。这篇文章给你一个决策框架，帮你在实际项目中做出选择。

<!--more-->

## 全系列回顾

先把前六篇的核心内容汇总：

| 模型 | 核心思想 | 优势 | 劣势 |
|------|----------|------|------|
| 线程 + 锁 | 共享内存，锁保护 | 直观，控制力强 | 死锁，复杂度高 |
| 协程 | 用户态轻量线程 | 高并发，低开销 | 不解决共享问题 |
| Actor | 隔离实体，消息传递 | 故障隔离，易扩展 | 消息开销，调试难 |
| CSP | Channel 为中心 | 组合性好，同步点清晰 | 需要设计数据流 |
| Rust 所有权 | 编译期检查 | 无运行时开销，安全 | 学习曲线陡峭 |

**注意**：Rust 所有权严格来说不是并发模型，而是**跨模型的安全机制**。Rust 使用上述所有模型（线程+锁、Channel/CSP、async 协程），但通过所有权系统在编译期保证它们的安全性。

**一句话总结**：

```text
线程 + 锁：传统但危险，需要老司机
协程：让并发更轻量，但根本问题还在
Actor：不共享就没问题，但有通信开销
CSP：用管道串起一切，数据流清晰
Rust：编译器帮你检查，但要学新思维
```

## 决策框架

### 第一步：确定约束条件

在选择并发模型之前，先问几个问题：

```text
1. 语言约束
   └── 团队用什么语言？
       Go → CSP (goroutine + channel)
       Erlang/Elixir → Actor
       Rust → 锁/Channel/async，所有权保证安全
       Java → 线程 + 锁 / Akka
       Python → asyncio / 多进程

2. 性能约束
   └── 对延迟敏感吗？
       └── 极致延迟 → 线程 + 锁 / Rust
       └── 一般场景 → 协程 / Actor / CSP

3. 可靠性约束
   └── 需要故障隔离吗？
       └── 是 → Actor (Erlang/Elixir)
       └── 否 → 其他都行

4. 团队约束
   └── 团队熟悉什么？
       └── 熟悉的模型比完美的模型更重要
```

### 第二步：识别问题类型

不同问题适合不同模型：

| 问题类型 | 特征 | 推荐模型 |
|----------|------|----------|
| I/O 密集 | 大量网络/磁盘操作 | 协程 / CSP |
| CPU 密集 | 计算为主 | 线程池 |
| 高并发连接 | 10K+ 连接 | 协程 / Actor |
| 有状态服务 | 每个实体有独立状态 | Actor |
| 数据管道 | 数据流经多个处理阶段 | CSP |
| 共享缓存 | 多线程读写同一数据 | 线程 + 锁 / Rust |
| 分布式系统 | 跨机器通信 | Actor |

### 第三步：评估权衡

```text
并发模型的权衡三角：

                性能
                 /\
                /  \
               /    \
              /      \
             /________\
         安全性      简单性

- 线程 + 锁：性能好，但安全性和简单性差
- Actor：安全性好，简单性中等，但有性能开销
- Rust：性能和安全性都好，但简单性差
- CSP/协程：三者平衡
```

## 场景实战

### 场景一：Web API 服务

```text
特征：
  - 大量短连接
  - I/O 密集（数据库、缓存、外部 API）
  - 请求间基本独立

推荐：协程 + CSP

Go 实现：
  每个请求一个 goroutine
  用 Channel 做任务分发
  共享状态用 sync.Map 或 Redis

Python 实现：
  async/await + asyncio
  aiohttp 或 FastAPI
  共享状态用 Redis

Rust 实现：
  tokio + axum/actix-web
  async/await
  共享状态用 Arc<RwLock<T>> 或 Redis
```

代码示例（Rust + Axum）：

```rust
use axum::{routing::get, Router, Json};
use tokio::join;

async fn get_data() -> Json<MergedResult> {
    // 并发获取多个数据源
    let (db_result, cache_result, api_result) = join!(
        fetch_from_db(),
        fetch_from_cache(),
        fetch_from_api()
    );

    Json(merge(db_result, cache_result, api_result))
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/api/data", get(get_data));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**Rust 的优势**：`join!` 宏在编译期检查所有 Future 的类型，比手动管理 Channel 更安全。

### 场景二：实时聊天系统

```text
特征：
  - 大量长连接（WebSocket）
  - 每个用户有独立状态
  - 需要广播和点对点消息
  - 用户上下线频繁

推荐：Actor 模型

Elixir/Phoenix 实现：
  每个用户连接一个 GenServer
  房间是一个 GenServer
  用 PubSub 做消息广播
  崩溃自动重启

Go 实现：
  每个连接一个 goroutine
  用 Channel 模拟 Actor
  自己实现监督逻辑
```

代码示例（Elixir）：

```elixir
defmodule ChatRoom do
  use GenServer

  def start_link(room_id) do
    GenServer.start_link(__MODULE__, room_id, name: via(room_id))
  end

  def join(room_id, user_pid) do
    GenServer.cast(via(room_id), {:join, user_pid})
  end

  def broadcast(room_id, message) do
    GenServer.cast(via(room_id), {:broadcast, message})
  end

  # 回调
  def init(room_id) do
    {:ok, %{room_id: room_id, users: MapSet.new()}}
  end

  def handle_cast({:join, user_pid}, state) do
    Process.monitor(user_pid)  # 监控用户进程
    {:noreply, %{state | users: MapSet.put(state.users, user_pid)}}
  end

  def handle_cast({:broadcast, message}, state) do
    Enum.each(state.users, fn pid ->
      send(pid, {:message, message})
    end)
    {:noreply, state}
  end

  def handle_info({:DOWN, _, :process, pid, _}, state) do
    # 用户断开，自动清理
    {:noreply, %{state | users: MapSet.delete(state.users, pid)}}
  end
end
```

### 场景三：数据处理管道

```text
特征：
  - 数据流经多个处理阶段
  - 每个阶段可以并行
  - 需要背压控制

推荐：CSP（Go Channel）

典型结构：
  读取 → 解析 → 转换 → 聚合 → 写入

  每个阶段是一组 goroutine
  阶段间用 Channel 连接
  用缓冲 Channel 控制背压
```

代码示例（Rust + Rayon）：

```rust
use rayon::prelude::*;
use std::fs::File;
use std::io::{BufRead, BufReader};

fn pipeline() {
    let file = File::open("input.txt").unwrap();
    let reader = BufReader::new(file);

    // rayon 自动处理并行：读取 → 解析 → 转换 → 写入
    reader
        .lines()
        .filter_map(Result::ok)
        .par_bridge()              // 转为并行迭代器
        .map(|line| parse(&line))  // 并行解析
        .map(|record| transform(record))  // 并行转换
        .for_each(|record| write(record)); // 并行写入
}
```

**Rust 的优势**：Rayon 自动管理线程池和任务调度，代码更简洁，且编译器保证无数据竞争。

如果需要更细粒度的控制，可以用 `crossbeam-channel`：

```rust
use crossbeam_channel::bounded;
use std::thread;

fn pipeline_with_channels() {
    let (tx_lines, rx_lines) = bounded::<String>(100);
    let (tx_parsed, rx_parsed) = bounded::<Record>(100);

    // 阶段 1：读取
    thread::spawn(move || {
        let file = File::open("input.txt").unwrap();
        for line in BufReader::new(file).lines().filter_map(Result::ok) {
            tx_lines.send(line).unwrap();
        }
    });

    // 阶段 2：解析（3 个 worker）
    for _ in 0..3 {
        let rx = rx_lines.clone();
        let tx = tx_parsed.clone();
        thread::spawn(move || {
            for line in rx {
                tx.send(parse(&line)).unwrap();
            }
        });
    }
    drop(tx_parsed); // 关闭发送端

    // 阶段 3：消费
    for record in rx_parsed {
        write(transform(record));
    }
}
```

### 场景四：游戏服务器

```text
特征：
  - 高频率状态更新（每帧）
  - 低延迟要求（< 16ms）
  - 复杂的状态同步
  - 需要物理模拟

推荐：线程 + 锁 / ECS 架构

原因：
  - 性能优先
  - 状态更新频繁
  - 需要细粒度控制

架构：
  主线程：游戏逻辑、物理模拟
  网络线程：处理 IO
  工作线程池：AI、寻路等

  线程间用无锁队列通信
  共享状态用细粒度锁或原子操作
```

代码示例（伪代码）：

```rust
// Rust ECS 风格
struct GameWorld {
    positions: Vec<Position>,
    velocities: Vec<Velocity>,
    healths: Vec<Health>,
}

impl GameWorld {
    fn update(&mut self, dt: f32) {
        // 物理系统：可以并行
        self.positions.par_iter_mut()
            .zip(self.velocities.par_iter())
            .for_each(|(pos, vel)| {
                pos.x += vel.x * dt;
                pos.y += vel.y * dt;
            });

        // 碰撞检测：可能需要同步
        self.check_collisions();

        // AI 系统：可以并行
        self.update_ai();
    }
}
```

### 场景五：IoT 设备管理

```text
特征：
  - 大量设备（百万级）
  - 每个设备独立状态
  - 设备会上下线
  - 需要故障隔离

推荐：Actor 模型（Erlang/Elixir）

原因：
  - 每个设备一个 Actor，天然映射
  - 设备崩溃不影响其他设备
  - 可以跨节点分布
  - 热更新（不停机升级）

架构：
  Device Actor：设备状态和通信
  Zone Actor：管理一组设备
  Gateway Actor：协议转换
  Supervisor：监督树管理生命周期
```

## 混合使用

真实系统往往混合多种模型：

```text
电商系统示例：

┌─────────────────────────────────────────────┐
│                  API 网关                    │
│            (Go + goroutine)                 │
└─────────────────┬───────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
    ▼             ▼             ▼
┌────────┐  ┌──────────┐  ┌──────────┐
│订单服务│  │ 商品服务 │  │ 用户服务 │
│ (Go)   │  │  (Go)    │  │  (Go)    │
└───┬────┘  └────┬─────┘  └────┬─────┘
    │            │             │
    ▼            ▼             ▼
┌────────┐  ┌──────────┐  ┌──────────┐
│ MySQL  │  │   ES     │  │  Redis   │
│        │  │ (搜索)   │  │ (缓存)   │
└────────┘  └──────────┘  └──────────┘

并发模型使用：
  - API 网关：goroutine + Channel（CSP）
  - 订单服务：goroutine + 分布式锁（防止超卖）
  - 商品服务：读多写少，RWMutex + 缓存
  - 用户服务：Session 用 Redis 原子操作
```

## 反模式

### 反模式一：过度并发

```rust
// 错误：为每个任务创建线程
for item in items {
    thread::spawn(move || process(item));  // 100 万个线程？
}
```

**问题**：
- 内存爆炸（每个线程 1-8MB 栈空间）
- 调度开销
- 数据库连接耗尽

**正确做法**：使用线程池控制并发度

```rust
use rayon::prelude::*;

// 方案 1：使用 rayon（自动管理线程池）
items.par_iter().for_each(|item| process(item));

// 方案 2：使用 tokio（异步任务池）
let semaphore = Arc::new(Semaphore::new(100));  // 限制并发数
for item in items {
    let permit = semaphore.clone().acquire_owned().await.unwrap();
    tokio::spawn(async move {
        process(item).await;
        drop(permit);  // 释放许可
    });
}
```

### 反模式二：锁粒度不当

```rust
// 错误（锁太粗）：
let mut cache = cache.lock().unwrap();
let data = fetch_from_network().await;  // I/O 期间持有锁！
cache.insert(key, data);
```

**问题**：
- 其他线程全部阻塞
- I/O 期间持有锁，浪费

**正确做法**：只锁关键区域

```rust
// 先做 I/O（不加锁）
let data = fetch_from_network().await;

// 只锁实际共享的数据访问
{
    let mut cache = cache.lock().unwrap();
    cache.insert(key, data);
} // 锁在这里立即释放
```

**注意**：在 async 代码中使用 `std::sync::Mutex` 时要特别小心——不要在持有锁时 `.await`！这会阻塞整个线程。如果需要跨 `.await` 持有锁，使用 `tokio::sync::Mutex`。

### 反模式三：滥用 Channel

```rust
// 错误：用 Channel 做简单的计数器
use std::sync::mpsc;

enum CounterMsg {
    Increment,
    Get(mpsc::Sender<i64>),
}

// 复杂的 Actor 式计数器实现...
```

**问题**：
- 代码更复杂
- 对于简单计数器，性能不如 atomic（取决于实现，差距可能很大）
- 但不要因此就放弃 Channel

**正确做法**：简单计数器用 atomic

```rust
use std::sync::atomic::{AtomicI64, Ordering};

static COUNTER: AtomicI64 = AtomicI64::new(0);

COUNTER.fetch_add(1, Ordering::Relaxed);
COUNTER.load(Ordering::Relaxed);
```

**Channel 适合的场景**：
- 任务分发
- 流水线
- 同步点
- 结果收集
- 需要解耦的场景

**选择原则**：
```text
简单共享数据 → AtomicXxx / Mutex
需要协调多个任务 → Channel
数据流处理 → Channel / Stream
```

### 反模式四：忽略取消

```rust
// 错误：任务永远不会退出
tokio::spawn(async {
    loop {
        do_work().await;
    }
});
```

**问题**：
- 任务永远不会退出
- 资源泄漏
- 无法优雅关闭

**正确做法**：使用 `CancellationToken` 或 `select!`

```rust
use tokio_util::sync::CancellationToken;

let token = CancellationToken::new();
let task_token = token.clone();

tokio::spawn(async move {
    loop {
        tokio::select! {
            _ = task_token.cancelled() => {
                println!("Task cancelled, cleaning up...");
                return;  // 优雅退出
            }
            _ = do_work() => {}
        }
    }
});

// 需要关闭时
token.cancel();
```

**或者使用 `AbortHandle`**：

```rust
let handle = tokio::spawn(async {
    loop { do_work().await; }
});

// 强制取消（不推荐，无法清理资源）
handle.abort();
```

## 选型检查清单

在做技术选型时，过一遍这个清单：

```text
□ 团队熟悉吗？
  └── 不熟悉的模型有学习成本

□ 语言支持吗？
  └── 不是所有语言都支持所有模型

□ 性能够用吗？
  └── 测量，不要猜

□ 调试友好吗？
  └── 出了问题能查得出来吗

□ 可以演进吗？
  └── 未来需求变了怎么办

□ 有成熟方案吗？
  └── 避免重新发明轮子

□ 社区活跃吗？
  └── 遇到问题能找到帮助吗
```

## 总结

**核心建议**：

```text
1. 先用简单的方案
   单线程能搞定？那就单线程
   不要过早优化

2. 选择团队熟悉的
   熟悉的模型比完美的模型更可靠
   学习成本是成本

3. 根据场景选择
   I/O 密集 → 协程
   有状态实体 → Actor
   数据管道 → CSP
   极致性能 → 线程 + 锁

4. 准备好混合使用
   真实系统往往不止一种模型
   在合适的地方用合适的工具

5. 重视可观测性
   无论选什么模型
   都要能监控、追踪、调试
```

**最后的话**：

> 并发编程是难的。不是因为某个模型不好，而是因为并发本身就复杂。
>
> 每种模型都是前人智慧的结晶，解决特定问题的特定方案。
>
> 没有银弹，只有场景。
>
> 选择适合你的，然后深入掌握它。

---

*本系列到此结束。感谢阅读！*

---

*上一篇：[Rust 的第三条路：编译期消灭数据竞争](/posts/concurrency-models-6-rust/)*

*本系列：*
1. *[为什么并发这么难？](/posts/concurrency-models-1-why-hard/)*
2. *[线程与锁：最直接也最危险](/posts/concurrency-models-2-threads-locks/)*
3. *[协程：用户态的轻量级线程](/posts/concurrency-models-3-coroutines/)*
4. *[Actor 模型：不要共享，要传递](/posts/concurrency-models-4-actor/)*
5. *[CSP 模型：Channel 是一等公民](/posts/concurrency-models-5-csp/)*
6. *[Rust 的第三条路：编译期消灭数据竞争](/posts/concurrency-models-6-rust/)*
7. *实战选型：没有银弹，只有场景（本篇）*
