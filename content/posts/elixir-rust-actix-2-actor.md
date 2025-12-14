---
title: "Elixir 到 Rust Actix（二）：Actor 模型——GenServer vs Actix Actor"
date: 2025-12-14T21:00:00+08:00
draft: false
tags: ["Elixir", "Rust", "Actix", "Actor", "GenServer", "并发"]
categories: ["技术"]
description: "GenServer 是 Elixir 开发者的老朋友，三个回调函数就能搞定一切。Actix Actor 是它在 Rust 世界的'表亲'，但这个表亲要求你先定义消息类型、实现 Handler trait、处理 Result……为什么同样是 Actor，一个简洁优雅，一个'仪式感'满满？"
series: ["Elixir 到 Rust Actix"]
---

GenServer 是 Elixir 开发者的老朋友，三个回调函数就能搞定一切。Actix Actor 是它在 Rust 世界的"表亲"，但这个表亲要求你先定义消息类型、实现 Handler trait、处理 Result……为什么同样是 Actor，一个简洁优雅，一个"仪式感"满满？

<!--more-->

## 前情回顾

在上一篇中，我们看到了 Elixir 和 Rust Actix 的四大哲学差异：

- **运行时**：BEAM VM vs 原生编译
- **内存**：GC vs 所有权
- **类型**：动态 vs 静态
- **错误**：Let it crash vs 编译期消灭

今天我们深入第一个具体主题：**Actor 的实现方式**。

## 从一个计数器说起

让我们用最经典的例子——计数器——来对比两种 Actor 的写法。

### Elixir GenServer：三步搞定

```elixir
defmodule Counter do
  use GenServer

  # 客户端 API
  def start_link(initial \\ 0) do
    GenServer.start_link(__MODULE__, initial, name: __MODULE__)
  end

  def increment, do: GenServer.cast(__MODULE__, :increment)
  def decrement, do: GenServer.cast(__MODULE__, :decrement)
  def get, do: GenServer.call(__MODULE__, :get)

  # 服务端回调
  @impl true
  def init(initial), do: {:ok, initial}

  @impl true
  def handle_cast(:increment, count), do: {:noreply, count + 1}
  def handle_cast(:decrement, count), do: {:noreply, count - 1}

  @impl true
  def handle_call(:get, _from, count), do: {:reply, count, count}
end
```

使用起来更简单：

```elixir
{:ok, _pid} = Counter.start_link(0)
Counter.increment()
Counter.increment()
Counter.get()  # => 2
```

**总共不到 20 行代码**，清晰明了。

### Rust Actix：仪式感满满

同样的计数器，在 Actix 里是这样的：

```rust
use actix::prelude::*;

// 第一步：定义 Actor
struct Counter {
    count: i32,
}

impl Actor for Counter {
    type Context = Context<Self>;
}

// 第二步：定义消息类型
#[derive(Message)]
#[rtype(result = "()")]
struct Increment;

#[derive(Message)]
#[rtype(result = "()")]
struct Decrement;

#[derive(Message)]
#[rtype(result = "i32")]
struct Get;

// 第三步：为每种消息实现 Handler
impl Handler<Increment> for Counter {
    type Result = ();

    fn handle(&mut self, _msg: Increment, _ctx: &mut Context<Self>) -> Self::Result {
        self.count += 1;
    }
}

impl Handler<Decrement> for Counter {
    type Result = ();

    fn handle(&mut self, _msg: Decrement, _ctx: &mut Context<Self>) -> Self::Result {
        self.count -= 1;
    }
}

impl Handler<Get> for Counter {
    type Result = i32;

    fn handle(&mut self, _msg: Get, _ctx: &mut Context<Self>) -> Self::Result {
        self.count
    }
}
```

使用时：

```rust
#[actix::main]
async fn main() {
    let addr = Counter { count: 0 }.start();

    addr.do_send(Increment);
    addr.do_send(Increment);

    let count = addr.send(Get).await.unwrap();
    println!("Count: {}", count);  // => 2
}
```

**代码量翻了三倍**。但先别急着下结论。

## 为什么 Actix 这么"啰嗦"？

这不是 Actix 设计者的恶趣味，而是 **Rust 类型系统的必然要求**。

### 类比：餐厅点餐

```text
Elixir 餐厅（GenServer）：
  顾客："来一份 :increment"
  服务员："好的"（不检查菜单，直接喊后厨）
  后厨：收到！（模式匹配处理）

  优点：灵活、快速
  风险：点了不存在的菜，运行时才发现

Rust 餐厅（Actix）：
  顾客："来一份 Increment"
  服务员："请出示您的 Increment 订单（类型）"
  顾客：递上订单
  服务员：核对订单格式、返回类型……OK！
  后厨：收到！

  优点：不会点错菜，返回类型明确
  代价：每道菜都要有正式的订单格式
```

**Actix 的"仪式感"换来的是编译期的类型安全。**

### 具体来说

在 Elixir 中，你可以发送任意消息：

```elixir
GenServer.cast(pid, :increment)
GenServer.cast(pid, {:set, 100})
GenServer.cast(pid, "hello")
GenServer.cast(pid, %{anything: "goes"})
```

如果发送了 Actor 不处理的消息？**运行时才知道**（可能崩溃，可能被忽略）。

在 Actix 中，每种消息都必须：

1. 定义为一个类型（struct）
2. 实现 `Message` trait，声明返回类型
3. Actor 必须实现对应的 `Handler`

```rust
// 如果你发送 Actor 没有 Handler 的消息
addr.send(UnknownMessage);
// 编译错误！Handler<UnknownMessage> 未实现
```

**编译器帮你拦截了错误。**

## 结构对比：骨架的差异

让我们拆解两种 Actor 的骨架：

```text
┌─────────────────────────────────────────────────────────────────┐
│                     GenServer (Elixir)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  use GenServer          ← 引入行为                              │
│                                                                 │
│  init/1                 ← 初始化，返回 {:ok, state}             │
│                                                                 │
│  handle_call/3          ← 同步调用，需要回复                    │
│                         ← 参数：(消息, 来源, 状态)               │
│                         ← 返回：{:reply, 回复, 新状态}          │
│                                                                 │
│  handle_cast/2          ← 异步调用，不需要回复                  │
│                         ← 参数：(消息, 状态)                    │
│                         ← 返回：{:noreply, 新状态}              │
│                                                                 │
│  handle_info/2          ← 处理其他消息（如 :DOWN）              │
│                                                                 │
│  terminate/2            ← 终止回调                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     Actix Actor (Rust)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  struct MyActor { ... } ← 定义 Actor，包含状态                  │
│                                                                 │
│  impl Actor for MyActor ← 实现 Actor trait                      │
│    type Context         ← 指定上下文类型                        │
│    fn started()         ← 启动回调                              │
│    fn stopping()        ← 停止中回调                            │
│    fn stopped()         ← 已停止回调                            │
│                                                                 │
│  #[derive(Message)]     ← 定义消息类型                          │
│  #[rtype(result = T)]   ← 声明返回类型                          │
│                                                                 │
│  impl Handler<Msg>      ← 为每种消息实现处理器                  │
│    type Result          ← 返回类型                              │
│    fn handle()          ← 处理逻辑                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 关键差异

| 方面 | GenServer | Actix Actor |
|------|-----------|-------------|
| **消息定义** | 不需要，用 atom/tuple 即可 | 必须定义类型 + Message trait |
| **消息处理** | 一个 handle_call/cast 函数，模式匹配 | 每种消息一个 Handler impl |
| **返回类型** | 运行时确定 | 编译期确定（rtype） |
| **状态存储** | 回调函数的参数 | Actor struct 的字段 |
| **同步/异步** | call vs cast | send vs do_send |

## 生命周期：从生到死

### GenServer 的生命周期

```text
           start_link/start
                 │
                 ▼
           ┌─────────┐
           │  init   │ ← 初始化状态
           └────┬────┘
                │ {:ok, state}
                ▼
           ┌─────────┐
           │ running │ ← 处理消息循环
           └────┬────┘
                │ stop / crash / exit
                ▼
           ┌─────────┐
           │terminate│ ← 清理（可选）
           └─────────┘

回调函数：
  init/1        → 返回 {:ok, state} 或 {:stop, reason}
  terminate/2   → 清理资源（进程正常退出时调用）
```

### Actix Actor 的生命周期

```text
           Actor.start() / Actor.create()
                 │
                 ▼
           ┌─────────┐
           │ Started │ ← started() 回调
           └────┬────┘
                │
                ▼
           ┌─────────┐
           │ Running │ ← 处理消息
           └────┬────┘
                │ ctx.stop() / 所有地址丢弃
                ▼
           ┌─────────┐
           │Stopping │ ← stopping() 回调
           └────┬────┘
                │
                ▼
           ┌─────────┐
           │ Stopped │ ← stopped() 回调
           └─────────┘

回调函数：
  started()   → Actor 开始运行
  stopping()  → 即将停止（可以取消）
  stopped()   → 已停止，清理资源
```

### 代码对比：生命周期回调

Elixir：

```elixir
defmodule MyActor do
  use GenServer

  def init(arg) do
    IO.puts("Starting with #{inspect(arg)}")
    {:ok, arg}
  end

  def terminate(reason, state) do
    IO.puts("Terminating: #{inspect(reason)}")
    :ok
  end
end
```

Rust：

```rust
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, _ctx: &mut Self::Context) {
        println!("Actor started");
    }

    fn stopping(&mut self, _ctx: &mut Self::Context) -> Running {
        println!("Actor stopping");
        Running::Stop  // 或 Running::Continue 取消停止
    }

    fn stopped(&mut self, _ctx: &mut Self::Context) {
        println!("Actor stopped");
    }
}
```

> **注意**：Actix 的 `stopping()` 可以返回 `Running::Continue` 来阻止 Actor 停止。GenServer 没有这个机制。

## 同步 vs 异步：call/cast 与 send/do_send

这是两种 Actor 最常用的操作。

### GenServer：call 和 cast

```elixir
# 同步调用（等待回复）
result = GenServer.call(pid, {:get, key})
#        ↑ 阻塞直到收到回复

# 异步调用（不等待）
GenServer.cast(pid, {:set, key, value})
#         ↑ 立即返回，不关心结果
```

对应的处理：

```elixir
# handle_call 必须回复
def handle_call({:get, key}, _from, state) do
  {:reply, Map.get(state, key), state}
  #  ↑ 必须返回 :reply
end

# handle_cast 不需要回复
def handle_cast({:set, key, value}, state) do
  {:noreply, Map.put(state, key, value)}
  #  ↑ 返回 :noreply
end
```

### Actix：send 和 do_send

```rust
// 同步调用（返回 Future，需要 await）
let result = addr.send(GetMessage).await?;
//               ↑ 返回 Future，await 等待结果

// 异步调用（不等待）
addr.do_send(SetMessage { key, value });
//   ↑ 立即返回，忽略结果
```

**关键区别**：Actix 的 `send` 返回的是 `Future`，你需要 `.await` 才能获得结果。

```rust
// send 的完整类型
let future: Request<MyActor, GetMessage> = addr.send(GetMessage);
let result: Result<i32, MailboxError> = future.await;
//                 ↑ 消息的返回类型        ↑ 可能的邮箱错误
```

### 对比表

| 操作 | GenServer | Actix | 说明 |
|------|-----------|-------|------|
| 同步发送 | `call(pid, msg)` | `addr.send(msg).await` | 等待回复 |
| 异步发送 | `cast(pid, msg)` | `addr.do_send(msg)` | 不等待 |
| 超时控制 | `call(pid, msg, timeout)` | `addr.send(msg).timeout(dur)` | 设置超时 |
| 获取结果 | 直接返回 | `Result<T, MailboxError>` | Rust 需要处理错误 |

## 实战：会话管理器

让我们用一个稍微复杂的例子——会话管理器——来对比实际应用。

### 需求

- 创建会话，返回会话 ID
- 根据 ID 获取会话数据
- 更新会话
- 删除会话

### Elixir 实现

```elixir
defmodule SessionManager do
  use GenServer

  def start_link(_), do: GenServer.start_link(__MODULE__, %{}, name: __MODULE__)

  # 客户端 API
  def create(data), do: GenServer.call(__MODULE__, {:create, data})
  def get(id), do: GenServer.call(__MODULE__, {:get, id})
  def update(id, data), do: GenServer.call(__MODULE__, {:update, id, data})
  def delete(id), do: GenServer.cast(__MODULE__, {:delete, id})

  # 服务端回调
  @impl true
  def init(_), do: {:ok, %{}}

  @impl true
  def handle_call({:create, data}, _from, sessions) do
    id = generate_id()
    {:reply, {:ok, id}, Map.put(sessions, id, data)}
  end

  def handle_call({:get, id}, _from, sessions) do
    case Map.get(sessions, id) do
      nil -> {:reply, {:error, :not_found}, sessions}
      data -> {:reply, {:ok, data}, sessions}
    end
  end

  def handle_call({:update, id, data}, _from, sessions) do
    if Map.has_key?(sessions, id) do
      {:reply, :ok, Map.put(sessions, id, data)}
    else
      {:reply, {:error, :not_found}, sessions}
    end
  end

  @impl true
  def handle_cast({:delete, id}, sessions) do
    {:noreply, Map.delete(sessions, id)}
  end

  defp generate_id, do: :crypto.strong_rand_bytes(16) |> Base.encode16()
end
```

### Rust Actix 实现

```rust
use actix::prelude::*;
use std::collections::HashMap;
use uuid::Uuid;  // 需要在 Cargo.toml 中添加: uuid = { version = "1", features = ["v4"] }

// Actor 定义
struct SessionManager {
    sessions: HashMap<String, String>,
}

impl Actor for SessionManager {
    type Context = Context<Self>;
}

// 消息定义
#[derive(Message)]
#[rtype(result = "Result<String, SessionError>")]
struct Create {
    data: String,
}

#[derive(Message)]
#[rtype(result = "Result<String, SessionError>")]
struct Get {
    id: String,
}

#[derive(Message)]
#[rtype(result = "Result<(), SessionError>")]
struct Update {
    id: String,
    data: String,
}

#[derive(Message)]
#[rtype(result = "()")]
struct Delete {
    id: String,
}

// 错误类型
#[derive(Debug)]
enum SessionError {
    NotFound,
}

// Handler 实现
impl Handler<Create> for SessionManager {
    type Result = Result<String, SessionError>;

    fn handle(&mut self, msg: Create, _ctx: &mut Context<Self>) -> Self::Result {
        let id = Uuid::new_v4().to_string();
        self.sessions.insert(id.clone(), msg.data);
        Ok(id)
    }
}

impl Handler<Get> for SessionManager {
    type Result = Result<String, SessionError>;

    fn handle(&mut self, msg: Get, _ctx: &mut Context<Self>) -> Self::Result {
        self.sessions
            .get(&msg.id)
            .cloned()
            .ok_or(SessionError::NotFound)
    }
}

impl Handler<Update> for SessionManager {
    type Result = Result<(), SessionError>;

    fn handle(&mut self, msg: Update, _ctx: &mut Context<Self>) -> Self::Result {
        if self.sessions.contains_key(&msg.id) {
            self.sessions.insert(msg.id, msg.data);
            Ok(())
        } else {
            Err(SessionError::NotFound)
        }
    }
}

impl Handler<Delete> for SessionManager {
    type Result = ();

    fn handle(&mut self, msg: Delete, _ctx: &mut Context<Self>) {
        self.sessions.remove(&msg.id);
    }
}
```

### 代码量对比

| 指标 | Elixir | Rust Actix |
|------|--------|------------|
| 总行数 | ~40 行 | ~80 行 |
| Actor 定义 | 1 行 (`use GenServer`) | ~5 行 |
| 消息定义 | 0 行 | ~25 行 |
| 处理逻辑 | ~25 行 | ~40 行 |

**Rust 代码量约为 Elixir 的 2 倍**。但 Rust 版本有：

- 明确的消息类型（IDE 自动补全友好）
- 编译期类型检查
- 明确的错误类型

## 什么时候用哪个？

这不是"谁更好"的问题，而是"谁更适合"的问题。

### 选 Elixir GenServer 当：

```text
✓ 快速原型开发
✓ 消息类型变化频繁
✓ 团队熟悉动态语言
✓ 需要热更新
✓ 容错比类型安全更重要
```

### 选 Rust Actix 当：

```text
✓ 性能关键路径
✓ 需要编译期类型保证
✓ 消息协议相对稳定
✓ 需要和 C/C++ 代码互操作
✓ 团队愿意接受更长的编译时间换取运行时安全
```

## 常见问题

### Q：Actix 的 do_send 和 send 有什么区别？

**A：`do_send` 是"发完即忘"，`send` 需要等待回复。**

```rust
// do_send：不关心结果，也不等待
addr.do_send(MyMessage);

// send：返回 Future，需要 await 获取结果
let result = addr.send(MyMessage).await?;
```

类似于 GenServer 的 `cast` 和 `call`。

### Q：Actix Actor 的状态怎么初始化？

**A：在创建 Actor 时直接初始化，或使用 `create` 方法。**

```rust
// 方式 1：直接初始化
let addr = MyActor { count: 0, name: "test".into() }.start();

// 方式 2：使用 create（可以访问 Context）
let addr = MyActor::create(|ctx| {
    // 这里可以使用 ctx
    MyActor { count: 0, name: "test".into() }
});
```

### Q：GenServer 的 handle_info 在 Actix 里怎么实现？

**A：使用 `StreamHandler` 或在 `started()` 中设置消息接收。**

GenServer 的 `handle_info` 处理非标准消息（如进程监控消息）。Actix 中类似的功能需要通过其他 trait 实现：

```rust
impl StreamHandler<Item> for MyActor {
    fn handle(&mut self, item: Item, ctx: &mut Self::Context) {
        // 处理流消息
    }
}
```

## 总结

```text
GenServer vs Actix Actor：

┌─────────────────────────────────────────────────────────────────┐
│                       GenServer                                 │
│  "简洁优雅，信任运行时"                                          │
│                                                                 │
│  • 消息：atom/tuple，无需定义类型                                │
│  • 处理：模式匹配，一个函数处理多种消息                          │
│  • 风格：少量代码，快速实现                                      │
│  • 代价：类型错误在运行时发现                                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                       Actix Actor                               │
│  "仪式感满满，信任编译器"                                        │
│                                                                 │
│  • 消息：必须定义类型 + Message trait                            │
│  • 处理：每种消息一个 Handler                                    │
│  • 风格：更多代码，更多类型定义                                  │
│  • 收益：类型错误在编译期发现                                    │
└─────────────────────────────────────────────────────────────────┘
```

**核心映射表**：

| Elixir GenServer | Rust Actix | 说明 |
|------------------|------------|------|
| `use GenServer` | `impl Actor` | 声明为 Actor |
| `init/1` | `started()` | 初始化回调 |
| `handle_call/3` | `impl Handler<Msg>` + `send().await` | 同步消息 |
| `handle_cast/2` | `impl Handler<Msg>` + `do_send()` | 异步消息 |
| `terminate/2` | `stopped()` | 终止回调 |
| `GenServer.call` | `addr.send(msg).await` | 发送同步消息 |
| `GenServer.cast` | `addr.do_send(msg)` | 发送异步消息 |

下一篇，我们将深入探讨**进程与并发**——在 Elixir 里 `spawn` 一个进程像呼吸一样自然，在 Rust 里呢？

---

*上一篇：[同源不同路：两种 Actor 哲学的碰撞](/posts/elixir-rust-actix-1-overview/)*

*下一篇：[进程与并发：spawn 的两种人生](/posts/elixir-rust-actix-3-concurrency/)*

*本系列：*
1. *[同源不同路：两种 Actor 哲学的碰撞](/posts/elixir-rust-actix-1-overview/)*
2. *Actor 模型：GenServer vs Actix Actor（本篇）*
3. *[进程与并发：spawn 的两种人生](/posts/elixir-rust-actix-3-concurrency/)*
4. *[消息传递：从 send 到 Handler](/posts/elixir-rust-actix-4-message/)*
5. *[容错机制：Let it crash vs 不让你 crash](/posts/elixir-rust-actix-5-fault-tolerance/)*
6. *[模式匹配：熟悉的语法，不同的能力](/posts/elixir-rust-actix-6-pattern-matching/)*
7. *[状态管理：从 Agent/ETS 到 Rust 的选择](/posts/elixir-rust-actix-7-state/)*
