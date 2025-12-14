---
title: "Elixir 到 Rust Actix（三）：进程与并发——spawn 的两种人生"
date: 2025-12-14T22:00:00+08:00
draft: false
tags: ["Elixir", "Rust", "Actix", "并发", "Tokio", "spawn", "Task"]
categories: ["技术"]
description: "在 Elixir 里，spawn 一个进程就像呼吸一样自然——300 字节，微秒级创建，随便造。在 Rust 里，你要考虑 Actor 还是 tokio::spawn，要不要 Send + 'static，生命周期够不够长……同样是'创建并发执行单元'，为什么体验差这么多？"
series: ["Elixir 到 Rust Actix"]
---

在 Elixir 里，`spawn` 一个进程就像呼吸一样自然——300 字节，微秒级创建，随便造。在 Rust 里，你要考虑 Actor 还是 `tokio::spawn`，要不要 `Send + 'static`，生命周期够不够长……同样是"创建并发执行单元"，为什么体验差这么多？

<!--more-->

## 前情回顾

在前两篇中，我们对比了：

- **第一篇**：两种 Actor 的哲学差异（运行时、内存、类型、错误处理）
- **第二篇**：GenServer 和 Actix Actor 的具体实现

今天我们聚焦**并发原语**：如何创建并发执行单元，以及它们的差异。

## Elixir：spawn 像呼吸一样自然

### 最简单的并发

在 Elixir 里创建一个并发任务：

```elixir
spawn(fn ->
  IO.puts("Hello from another process!")
end)
```

就这样。一行代码，一个新进程。

### spawn 的三种姿势

```elixir
# 1. 基础 spawn（不监控）
pid = spawn(fn -> do_something() end)

# 2. spawn_link（链接，一个崩另一个也崩）
pid = spawn_link(fn -> do_something() end)

# 3. spawn_monitor（监控，收到通知但不崩）
{pid, ref} = spawn_monitor(fn -> do_something() end)
```

### Task：更高级的抽象

```elixir
# 启动异步任务
task = Task.async(fn ->
  expensive_computation()
end)

# 等待结果
result = Task.await(task)

# 或者并行执行多个任务
results = Task.async_stream(urls, fn url ->
  fetch(url)
end)
|> Enum.to_list()
```

### 进程的成本

```text
Elixir 进程的开销：

  创建成本：~300 字节内存，~1-3 微秒
  切换成本：纳秒级（BEAM 调度器优化）
  数量上限：理论上数百万个

  类比：进程像便签纸
    - 随手撕一张写个备忘
    - 用完就扔
    - 便宜到不用考虑成本
```

这就是为什么在 Elixir 里，"每个连接一个进程"、"每个请求一个进程"是常见模式。

## Rust：并发需要思考

### 基础选项：tokio::spawn

在 Rust + Tokio 里创建异步任务：

```rust
tokio::spawn(async {
    println!("Hello from another task!");
});
```

看起来也很简单？等等，这有要求：

```rust
tokio::spawn(async move {
    // 闭包必须是 'static + Send
    // 即：不能借用外部非 'static 数据，且要能跨线程发送
});
```

### 当你想捕获外部变量时

```rust
let data = vec![1, 2, 3];

// 错误！data 被借用，不满足 'static
tokio::spawn(async {
    println!("{:?}", data);
});

// 正确：move 所有权
tokio::spawn(async move {
    println!("{:?}", data);
});
// 但现在 data 不能再用了！
```

### 对比：Elixir 的消息复制

```elixir
data = [1, 2, 3]

spawn(fn ->
  # data 被复制进新进程
  # 原进程的 data 不受影响
  IO.inspect(data)
end)

# 还能继续用 data
IO.inspect(data)
```

**Elixir 通过消息复制实现进程隔离，Rust 通过所有权系统保证内存安全。**

## 并发原语对照表

```text
┌────────────────────────────────────────────────────────────────┐
│                    并发原语对照                                 │
├──────────────────┬─────────────────┬───────────────────────────┤
│     Elixir       │    Rust/Tokio   │          说明             │
├──────────────────┼─────────────────┼───────────────────────────┤
│ spawn(fn)        │ tokio::spawn()  │ 创建异步任务               │
│ spawn_link(fn)   │ 无直接对应       │ 链接的进程（一崩都崩）     │
│ Task.async       │ tokio::spawn()  │ 异步任务                   │
│ Task.await       │ handle.await    │ 等待结果                   │
│ Task.async_stream│ futures::stream │ 并行流处理                 │
│ Process          │ Actor           │ 有状态的并发单元           │
│ Agent            │ Arc<Mutex<T>>   │ 简单状态容器               │
├──────────────────┴─────────────────┴───────────────────────────┤
│ Actix 特有                                                      │
├──────────────────┬─────────────────┬───────────────────────────┤
│ -                │ Actor.start()   │ 启动 Actor                 │
│ -                │ Arbiter::spawn  │ 在特定 Arbiter 上启动      │
│ -                │ ctx.spawn()     │ 在 Actor 上下文中启动任务  │
└──────────────────┴─────────────────┴───────────────────────────┘
```

## 深入对比：Task 的差异

### Elixir Task

```elixir
# 启动任务
task = Task.async(fn ->
  :timer.sleep(1000)
  42
end)

# 做其他事情...

# 获取结果（默认 5 秒超时）
result = Task.await(task)  # => 42

# 自定义超时
result = Task.await(task, 10_000)  # 10 秒超时
```

任务崩溃时的行为：

```elixir
task = Task.async(fn ->
  raise "oops"
end)

# Task.await 会把异常重新抛出
Task.await(task)  # => ** (RuntimeError) oops
```

### Rust tokio::spawn

```rust
// 启动任务
let handle = tokio::spawn(async {
    tokio::time::sleep(Duration::from_secs(1)).await;
    42
});

// 做其他事情...

// 获取结果
let result = handle.await?;  // => 42
//                      ^ JoinError 需要处理
```

任务 panic 时的行为：

```rust
let handle = tokio::spawn(async {
    panic!("oops");
});

// await 返回 Err(JoinError)，不会传播 panic
match handle.await {
    Ok(value) => println!("Got: {}", value),
    Err(e) => println!("Task panicked: {:?}", e),
}
```

### 关键差异

| 方面 | Elixir Task | Rust tokio::spawn |
|------|-------------|-------------------|
| 错误传播 | 默认传播异常 | 返回 `Result<T, JoinError>` |
| 取消 | `Task.shutdown/2` | `handle.abort()` |
| 链接 | `Task.async_link` | 无直接对应 |
| 监控 | 进程监控 | 无直接对应 |

## Actix 的并发模型

Actix 提供了几种创建并发执行的方式：

### 1. 启动新 Actor

```rust
// 直接启动 Actor
let addr = MyActor.start();

// 使用 create 方法（可以在启动前访问 Context）
let addr = MyActor::create(|ctx| {
    // 可以在这里使用 ctx，例如获取地址
    let _self_addr = ctx.address();
    MyActor::default()
});
```

### 2. 在 Actor 上下文中 spawn

```rust
impl Handler<MyMessage> for MyActor {
    type Result = ();

    fn handle(&mut self, msg: MyMessage, ctx: &mut Context<Self>) {
        // 在 Actor 的上下文中启动异步任务
        ctx.spawn(async move {
            // 这个任务和 Actor 绑定
            // Actor 停止时，任务也会被取消
            do_async_work().await;
        }.into_actor(self));
    }
}
```

### 3. 使用 Arbiter

```rust
// Arbiter 是 Actix 的执行器单元
let arbiter = Arbiter::new();

// 在特定 Arbiter 上执行
arbiter.spawn(async {
    println!("Running on specific arbiter");
});
```

## 调度模型：抢占式 vs 协作式

这是一个根本性的差异。

### Elixir：抢占式调度

```text
BEAM 调度器：

  时间线 ─────────────────────────────────────►
           │      │      │      │      │
  进程 A   ██████ │      │ █████│      │ ████
  进程 B         ██████  │      │ █████│
  进程 C                 ██████ │      │ █████

  • 每个进程运行约 2000 reductions（约指令数）
  • 调度器强制切换，无需进程配合
  • 进程不能独占 CPU
```

**即使代码里写了死循环，也不会卡住整个系统：**

```elixir
spawn(fn ->
  loop = fn loop -> loop.(loop) end
  loop.(loop)  # 死循环
end)

# 其他进程照常运行，不受影响
```

### Rust/Tokio：协作式调度

```text
Tokio 调度器：

  时间线 ─────────────────────────────────────►
           │         │              │
  任务 A   ██████████ await ████████ await ███
  任务 B             ███████████████
  任务 C                            ██████████

  • 任务必须主动让出（await）
  • 如果不 await，任务会一直占用线程
  • 长时间计算需要手动让出
```

**如果忘记 await，会阻塞整个线程：**

```rust
tokio::spawn(async {
    // 危险！CPU 密集型计算没有 await 点
    loop {
        heavy_computation();
        // 没有 await，其他任务无法运行
    }
});
```

**正确做法**：

```rust
tokio::spawn(async {
    loop {
        heavy_computation();
        tokio::task::yield_now().await;  // 主动让出
    }
});

// 或者用 spawn_blocking 处理 CPU 密集型任务
let result = tokio::task::spawn_blocking(|| {
    heavy_computation()
}).await?;
```

## 进程/任务通信

### Elixir：消息传递

```elixir
# 进程间通信只有一种方式：发消息
parent = self()

spawn(fn ->
  # 计算完成后发消息给父进程
  result = expensive_computation()
  send(parent, {:result, result})
end)

# 父进程接收
receive do
  {:result, value} -> IO.puts("Got: #{value}")
after
  5000 -> IO.puts("Timeout!")
end
```

### Rust：多种方式

```rust
// 方式 1：通过 channel
let (tx, mut rx) = tokio::sync::mpsc::channel(32);

tokio::spawn(async move {
    let result = expensive_computation().await;
    tx.send(result).await.unwrap();
});

let value = rx.recv().await.unwrap();

// 方式 2：通过 JoinHandle
let handle = tokio::spawn(async {
    expensive_computation().await
});

let value = handle.await?;

// 方式 3：通过 Actix 消息
addr.send(MyMessage).await?;
```

## 实战：并发任务处理器

让我们实现一个并发执行多个 URL 请求的处理器。

### Elixir 实现

```elixir
defmodule UrlFetcher do
  def fetch_all(urls) do
    urls
    |> Task.async_stream(&fetch_url/1, max_concurrency: 10)
    |> Enum.map(fn {:ok, result} -> result end)
  end

  defp fetch_url(url) do
    case HTTPoison.get(url) do
      {:ok, response} -> {:ok, url, response.body}
      {:error, reason} -> {:error, url, reason}
    end
  end
end

# 使用
urls = ["https://example1.com", "https://example2.com", ...]
results = UrlFetcher.fetch_all(urls)
```

简洁明了，`Task.async_stream` 自动处理并发控制。

### Rust Actix 实现

```rust
use actix::prelude::*;
use futures::stream::{self, StreamExt};  // futures = "0.3"
use reqwest::Client;  // reqwest = { version = "0.11", features = ["json"] }

struct UrlFetcher {
    client: Client,
}

impl Actor for UrlFetcher {
    type Context = Context<Self>;
}

#[derive(Message)]
#[rtype(result = "Vec<FetchResult>")]
struct FetchAll {
    urls: Vec<String>,
}

#[derive(Debug)]
enum FetchResult {
    Ok { url: String, body: String },
    Err { url: String, error: String },
}

impl Handler<FetchAll> for UrlFetcher {
    type Result = ResponseFuture<Vec<FetchResult>>;

    fn handle(&mut self, msg: FetchAll, _ctx: &mut Context<Self>) -> Self::Result {
        let client = self.client.clone();

        Box::pin(async move {
            stream::iter(msg.urls)
                .map(|url| {
                    let client = client.clone();
                    async move {
                        match client.get(&url).send().await {
                            Ok(resp) => match resp.text().await {
                                Ok(body) => FetchResult::Ok { url, body },
                                Err(e) => FetchResult::Err {
                                    url,
                                    error: e.to_string(),
                                },
                            },
                            Err(e) => FetchResult::Err {
                                url,
                                error: e.to_string(),
                            },
                        }
                    }
                })
                .buffer_unordered(10)  // 最多 10 个并发
                .collect()
                .await
        })
    }
}
```

**Rust 版本更长，但有明确的类型和错误处理。**

## 性能对比

```text
创建成本（粗略估计）：

┌─────────────────────────────────────────────────────────────┐
│  操作                    │  Elixir      │  Rust/Tokio       │
├─────────────────────────────────────────────────────────────┤
│  创建进程/任务           │  ~1-3 μs     │  ~0.1-0.5 μs      │
│  内存开销                │  ~300 B      │  ~几十到几百 B    │
│  上下文切换              │  ~ns 级      │  ~ns 级           │
│  最大并发数              │  数百万      │  取决于内存和设计 │
└─────────────────────────────────────────────────────────────┘

注意：
  • Elixir 进程包含更多元数据（邮箱、链接等）
  • Rust 任务更轻量，但功能也更少
  • 实际性能取决于具体场景
```

## 什么时候用什么？

### 用 Elixir spawn/Task：

```text
✓ 需要大量独立的并发单元（如每连接一个进程）
✓ 需要进程隔离（一个崩不影响其他）
✓ 需要进程监控和链接
✓ 并发逻辑复杂，需要 OTP 支持
```

### 用 Rust tokio::spawn：

```text
✓ 轻量级异步任务
✓ 不需要持久状态
✓ 性能关键路径
✓ 任务间需要共享数据（通过 Arc）
```

### 用 Actix Actor：

```text
✓ 需要持久状态的并发单元
✓ 需要类型安全的消息传递
✓ 需要生命周期管理
✓ 想要类似 Elixir 进程的抽象
```

## 常见问题

### Q：Rust 的 tokio::spawn 要求 Send + 'static，太烦了怎么办？

**A：理解它，然后拥抱它。**

```rust
// 问题：借用数据不能 spawn
let data = vec![1, 2, 3];
let data_ref = &data;

tokio::spawn(async {
    println!("{:?}", data_ref);  // 错误！
});

// 解决方案 1：move 所有权
tokio::spawn(async move {
    println!("{:?}", data);
});

// 解决方案 2：克隆
let data_clone = data.clone();
tokio::spawn(async move {
    println!("{:?}", data_clone);
});

// 解决方案 3：使用 Arc 共享
let data = Arc::new(vec![1, 2, 3]);
let data_clone = Arc::clone(&data);
tokio::spawn(async move {
    println!("{:?}", data_clone);
});
```

### Q：Elixir 的 spawn_link 在 Rust 里怎么实现？

**A：没有直接对应，需要自己实现监控逻辑。**

```rust
// 模拟 spawn_link 的效果
let handle = tokio::spawn(async {
    // 子任务
});

// 父任务监控子任务
tokio::select! {
    result = handle => {
        if result.is_err() {
            // 子任务失败，父任务也失败
            panic!("Child task failed");
        }
    }
}
```

在 Actix 中，可以通过监控 Actor 地址是否有效来实现类似效果。

### Q：CPU 密集型任务怎么处理？

**Elixir**：BEAM 调度器自动处理，但建议用 NIF 或 Port。

**Rust**：使用 `spawn_blocking`：

```rust
let result = tokio::task::spawn_blocking(|| {
    // CPU 密集型计算
    heavy_computation()
}).await?;
```

## 总结

```text
spawn 的两种人生：

┌─────────────────────────────────────────────────────────────────┐
│                   Elixir spawn                                  │
│  "随手创建，无忧无虑"                                            │
│                                                                 │
│  • 进程轻量（~300B），随便创建                                   │
│  • 抢占式调度，不用担心阻塞                                      │
│  • 消息复制，天然隔离                                            │
│  • spawn_link/monitor 提供链接和监控                            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   Rust spawn                                    │
│  "精心设计，万无一失"                                            │
│                                                                 │
│  • 任务更轻量，但功能也更少                                      │
│  • 协作式调度，需要主动让出                                      │
│  • 所有权系统保证内存安全                                        │
│  • 需要自己处理链接和监控逻辑                                    │
└─────────────────────────────────────────────────────────────────┘
```

**核心映射**：

| Elixir | Rust | 说明 |
|--------|------|------|
| `spawn(fn)` | `tokio::spawn(async {})` | 创建并发任务 |
| `spawn_link` | 无直接对应 | 链接的任务 |
| `Task.async/await` | `tokio::spawn` + `.await` | 异步任务 |
| `Task.async_stream` | `stream::iter().buffer_unordered()` | 并发流处理 |
| `Process` | `Actor` | 有状态并发单元 |
| 消息复制 | `move` 或 `Arc::clone` | 数据传递 |

下一篇，我们将深入**消息传递**——Elixir 的 `send` 和 `receive` 简洁优雅，Actix 为什么要定义那么多类型？

---

*上一篇：[Actor 模型：GenServer vs Actix Actor](/posts/elixir-rust-actix-2-actor/)*

*下一篇：[消息传递：从 send 到 Handler](/posts/elixir-rust-actix-4-message/)*

*本系列：*
1. *[同源不同路：两种 Actor 哲学的碰撞](/posts/elixir-rust-actix-1-overview/)*
2. *[Actor 模型：GenServer vs Actix Actor](/posts/elixir-rust-actix-2-actor/)*
3. *进程与并发：spawn 的两种人生（本篇）*
4. *[消息传递：从 send 到 Handler](/posts/elixir-rust-actix-4-message/)*
5. *[容错机制：Let it crash vs 不让你 crash](/posts/elixir-rust-actix-5-fault-tolerance/)*
6. *[模式匹配：熟悉的语法，不同的能力](/posts/elixir-rust-actix-6-pattern-matching/)*
7. *[状态管理：从 Agent/ETS 到 Rust 的选择](/posts/elixir-rust-actix-7-state/)*
