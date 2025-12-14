---
title: "Elixir 到 Rust Actix（四）：消息传递——从 send 到 Handler"
date: 2025-12-14T23:00:00+08:00
draft: false
tags: ["Elixir", "Rust", "Actix", "消息传递", "类型系统"]
categories: ["技术"]
description: "Elixir 的 send 和 receive 简洁优雅——pid ! message，收到就处理，没收到就等。Actix 的消息系统？先定义消息类型，再声明返回类型，然后实现 Handler……为什么同样是发消息，一个三行代码，一个要写一堆 struct 和 impl？"
series: ["Elixir 到 Rust Actix"]
---

Elixir 的 `send` 和 `receive` 简洁优雅——`pid ! message`，收到就处理，没收到就等。Actix 的消息系统？先定义消息类型，再声明返回类型，然后实现 Handler……为什么同样是发消息，一个三行代码，一个要写一堆 struct 和 impl？

<!--more-->

## 前情回顾

前几篇我们对比了：

- **第一篇**：哲学差异（动态 vs 静态，Let it crash vs 编译期消灭）
- **第二篇**：Actor 实现（GenServer vs Actix Actor）
- **第三篇**：并发原语（spawn vs tokio::spawn）

今天深入**消息传递**——Actor 模型的核心。

## Elixir：消息传递的极简主义

### 最基本的 send/receive

```elixir
# 发送消息
send(pid, {:hello, "world"})

# 接收消息
receive do
  {:hello, name} -> IO.puts("Hello, #{name}!")
  {:bye, name} -> IO.puts("Goodbye, #{name}!")
after
  5000 -> IO.puts("No message in 5 seconds")
end
```

就这么简单。发送方用 `send`，接收方用 `receive` + 模式匹配。

### 消息可以是任何东西

```elixir
# atom
send(pid, :ping)

# tuple
send(pid, {:user, 123, "Alice"})

# map
send(pid, %{action: :create, data: %{name: "Bob"}})

# 甚至是函数
send(pid, fn -> IO.puts("I'm a function!") end)
```

**没有类型限制，没有格式要求。** 发送方想发什么就发什么，接收方用模式匹配处理。

### GenServer 中的消息

```elixir
# 发送（客户端）
GenServer.call(pid, {:get, :user, 123})
GenServer.cast(pid, {:delete, :user, 123})

# 接收（服务端）
def handle_call({:get, :user, id}, _from, state) do
  user = Map.get(state.users, id)
  {:reply, user, state}
end

def handle_cast({:delete, :user, id}, state) do
  {:noreply, %{state | users: Map.delete(state.users, id)}}
end
```

**消息格式完全由约定决定**，没有编译期检查。

## Actix：类型安全的消息传递

### 消息必须是类型

在 Actix 中，每种消息都必须：

```rust
// 1. 定义为一个类型
struct GetUser {
    id: u64,
}

// 2. 实现 Message trait，声明返回类型
impl Message for GetUser {
    type Result = Option<User>;
}

// 或者用 derive 宏简化
#[derive(Message)]
#[rtype(result = "Option<User>")]
struct GetUser {
    id: u64,
}
```

### 处理消息

```rust
// 为每种消息实现 Handler
impl Handler<GetUser> for UserActor {
    type Result = Option<User>;

    fn handle(&mut self, msg: GetUser, _ctx: &mut Context<Self>) -> Self::Result {
        self.users.get(&msg.id).cloned()
    }
}
```

### 发送消息

```rust
// 同步发送（等待回复）
let user = addr.send(GetUser { id: 123 }).await?;

// 异步发送（不等待）
addr.do_send(DeleteUser { id: 123 });
```

## 为什么差异这么大？

### 类比：寄快递

```text
Elixir 的消息传递：寄便签
  ┌────────────────────────────────────────┐
  │  寄件人："这是给你的"                  │
  │                                        │
  │  便签纸上写着：                         │
  │    {:get, :user, 123}                  │
  │                                        │
  │  收件人：看到内容，理解意思，处理       │
  │                                        │
  │  优点：简单、灵活                       │
  │  风险：写错了运行时才发现               │
  └────────────────────────────────────────┘

Actix 的消息传递：寄正式文件
  ┌────────────────────────────────────────┐
  │  寄件人：填写正式表格                   │
  │                                        │
  │  表格格式：                            │
  │    struct GetUser { id: u64 }          │
  │    预期回执：Option<User>              │
  │                                        │
  │  收件人：核对表格格式，处理，返回回执   │
  │                                        │
  │  优点：格式明确，错误编译期发现         │
  │  代价：每种表格都要先设计              │
  └────────────────────────────────────────┘
```

## 消息类型设计对比

### Elixir：约定式消息

```elixir
# 约定：操作类型用 atom，数据用 tuple
{:create, :user, %{name: "Alice", age: 30}}
{:update, :user, 123, %{name: "Bob"}}
{:delete, :user, 123}
{:get, :user, 123}

# 或者用 map
%{action: :create, resource: :user, data: %{...}}
```

团队约定好格式，依靠纪律和文档维护一致性。

### Actix：类型式消息

```rust
// 每种操作一个类型
#[derive(Message)]
#[rtype(result = "Result<User, DbError>")]
struct CreateUser {
    name: String,
    age: u32,
}

#[derive(Message)]
#[rtype(result = "Result<(), DbError>")]
struct UpdateUser {
    id: u64,
    name: Option<String>,
    age: Option<u32>,
}

#[derive(Message)]
#[rtype(result = "()")]
struct DeleteUser {
    id: u64,
}

#[derive(Message)]
#[rtype(result = "Option<User>")]
struct GetUser {
    id: u64,
}
```

编译器强制格式一致，IDE 提供自动补全。

## 返回值处理

### Elixir：返回值在运行时确定

```elixir
# 发送
result = GenServer.call(pid, {:get, :user, 123})

# result 的类型是什么？
# 看代码：可能是 {:ok, user}，可能是 {:error, reason}，可能是 nil
# Dialyzer 可以帮忙，但不是强制的
```

### Actix：返回值在编译期确定

```rust
// 消息定义时就声明了返回类型
#[derive(Message)]
#[rtype(result = "Option<User>")]
struct GetUser { id: u64 }

// 发送时，编译器知道返回类型
let result: Option<User> = addr.send(GetUser { id: 123 }).await?;
//          ↑ 类型是确定的
```

## 邮箱与背压

### Elixir：无界邮箱

```text
Elixir 进程邮箱：
  ┌─────────────────────────────────────┐
  │  进程邮箱（无大小限制）              │
  │  ┌─────┬─────┬─────┬─────┬─────┐   │
  │  │msg1 │msg2 │msg3 │ ... │msgN │   │
  │  └─────┴─────┴─────┴─────┴─────┘   │
  │                                     │
  │  消息会一直堆积，直到：              │
  │  • 被处理                           │
  │  • 进程崩溃                         │
  │  • 内存耗尽                         │
  └─────────────────────────────────────┘
```

如果生产者比消费者快，邮箱会无限增长：

```elixir
# 危险：快速发送大量消息
for i <- 1..1_000_000 do
  send(pid, {:process, i})
end
# 如果处理速度跟不上，邮箱会爆炸
```

### Actix：可配置邮箱

```rust
// 默认邮箱大小是 16
// 可以自定义
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        // 设置邮箱容量
        ctx.set_mailbox_capacity(1000);
    }
}
```

邮箱满时的行为：

```rust
// send() 返回 Future，邮箱满时会等待
let result = addr.send(MyMessage).await;

// do_send() 在邮箱满时会丢弃消息
addr.do_send(MyMessage);  // 可能丢失！

// try_send() 邮箱满时返回错误
match addr.try_send(MyMessage) {
    Ok(()) => println!("Sent"),
    Err(SendError::Full(_)) => println!("Mailbox full!"),
    Err(SendError::Closed(_)) => println!("Actor stopped!"),
}
```

## 请求-响应模式

### Elixir GenServer.call 的原理

```elixir
# GenServer.call 的简化实现原理
def call(pid, message, timeout \\ 5000) do
  ref = make_ref()  # 创建唯一引用
  send(pid, {:"$gen_call", {self(), ref}, message})

  receive do
    {^ref, response} -> response  # ^ 确保是同一个 ref
  after
    timeout -> exit(:timeout)
  end
end
```

特点：
- 使用唯一 ref 匹配响应
- 内置超时机制
- 阻塞等待

### Actix send 的原理

```rust
// addr.send(msg) 返回一个 Request Future
let request: Request<MyActor, MyMessage> = addr.send(MyMessage);

// await 时真正发送并等待
let result: Result<Response, MailboxError> = request.await;
```

更多控制：

```rust
// 设置超时
let result = addr
    .send(MyMessage)
    .timeout(Duration::from_secs(5))
    .await;

// 处理各种错误
match result {
    Ok(response) => println!("Got: {:?}", response),
    Err(MailboxError::Closed) => println!("Actor stopped"),
    Err(MailboxError::Timeout) => println!("Timeout"),
}
```

## 实战：消息路由器

实现一个根据消息类型路由到不同处理器的系统。

### Elixir 实现

```elixir
defmodule Router do
  use GenServer

  def start_link(handlers) do
    GenServer.start_link(__MODULE__, handlers, name: __MODULE__)
  end

  def route(message) do
    GenServer.call(__MODULE__, {:route, message})
  end

  @impl true
  def init(handlers), do: {:ok, handlers}

  @impl true
  def handle_call({:route, message}, _from, handlers) do
    # 根据消息类型分发
    handler = case message do
      {:user, _} -> handlers[:user]
      {:order, _} -> handlers[:order]
      {:payment, _} -> handlers[:payment]
      _ -> handlers[:default]
    end

    result = GenServer.call(handler, message)
    {:reply, result, handlers}
  end
end

# 使用
Router.route({:user, :get, 123})
Router.route({:order, :create, %{items: [...]}})
```

### Actix 实现

```rust
use actix::prelude::*;
use std::collections::HashMap;

// 路由器 Actor
struct Router {
    handlers: HashMap<String, Recipient<RouteMessage>>,
}

impl Actor for Router {
    type Context = Context<Self>;
}

// 通用路由消息
#[derive(Message, Clone)]
#[rtype(result = "Result<String, RouterError>")]
struct RouteMessage {
    msg_type: String,
    payload: String,
}

#[derive(Debug)]
enum RouterError {
    NoHandler,
    HandlerError(String),
}

impl Handler<RouteMessage> for Router {
    type Result = ResponseFuture<Result<String, RouterError>>;

    fn handle(&mut self, msg: RouteMessage, _ctx: &mut Context<Self>) -> Self::Result {
        let handler = self.handlers.get(&msg.msg_type).cloned();

        Box::pin(async move {
            match handler {
                Some(recipient) => {
                    recipient
                        .send(msg)
                        .await
                        .map_err(|e| RouterError::HandlerError(e.to_string()))?
                }
                None => Err(RouterError::NoHandler),
            }
        })
    }
}

// 用户处理器
struct UserHandler;

impl Actor for UserHandler {
    type Context = Context<Self>;
}

impl Handler<RouteMessage> for UserHandler {
    type Result = Result<String, RouterError>;

    fn handle(&mut self, msg: RouteMessage, _ctx: &mut Context<Self>) -> Self::Result {
        Ok(format!("UserHandler processed: {}", msg.payload))
    }
}
```

## 消息模式对比

### 单向消息（Fire-and-Forget）

```elixir
# Elixir
send(pid, :do_something)
GenServer.cast(pid, :do_something)
```

```rust
// Actix
addr.do_send(DoSomething);
```

### 请求-响应

```elixir
# Elixir
result = GenServer.call(pid, :get_data)
```

```rust
// Actix
let result = addr.send(GetData).await?;
```

### 广播

```elixir
# Elixir：使用 Registry 或 pg
Registry.dispatch(MyRegistry, "topic", fn entries ->
  for {pid, _} <- entries, do: send(pid, :broadcast_msg)
end)
```

```rust
// Actix：使用 Recipient 列表
for recipient in &self.subscribers {
    recipient.do_send(BroadcastMsg);
}
```

## 常见问题

### Q：Actix 为什么要定义那么多消息类型？

**A：类型安全的代价和收益。**

```rust
// 假如 Actix 允许任意消息（像 Elixir 那样）
addr.send(anything);  // 返回什么类型？编译器不知道

// 现在的方式
#[derive(Message)]
#[rtype(result = "i32")]
struct Add(i32, i32);

addr.send(Add(1, 2)).await  // 返回 Result<i32, _>，编译器知道
```

**代价**：更多样板代码。
**收益**：类型错误在编译期发现，IDE 支持更好。

### Q：如何在 Actix 中实现类似 Elixir 的 handle_info？

**A：使用 StreamHandler 或 Context::run_interval。**

```rust
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        // 定时消息（类似 Elixir 的 :timer.send_interval）
        ctx.run_interval(Duration::from_secs(1), |act, ctx| {
            // 每秒执行
        });
    }
}

// 处理外部流
impl StreamHandler<Item> for MyActor {
    fn handle(&mut self, item: Item, ctx: &mut Self::Context) {
        // 处理流消息
    }
}
```

### Q：消息太多，代码太啰嗦怎么办？

**A：使用枚举合并相关消息。**

```rust
// 不要这样：每个操作一个类型
struct GetUser { id: u64 }
struct CreateUser { name: String }
struct DeleteUser { id: u64 }

// 可以这样：用枚举
#[derive(Message)]
#[rtype(result = "UserResult")]
enum UserCommand {
    Get { id: u64 },
    Create { name: String },
    Delete { id: u64 },
}

enum UserResult {
    User(Option<User>),
    Created(u64),
    Deleted,
}

// 一个 Handler 处理所有
impl Handler<UserCommand> for UserActor {
    type Result = UserResult;

    fn handle(&mut self, msg: UserCommand, _ctx: &mut Context<Self>) -> Self::Result {
        match msg {
            UserCommand::Get { id } => UserResult::User(self.get(id)),
            UserCommand::Create { name } => UserResult::Created(self.create(name)),
            UserCommand::Delete { id } => { self.delete(id); UserResult::Deleted }
        }
    }
}
```

## 总结

```text
消息传递的两种哲学：

┌─────────────────────────────────────────────────────────────────┐
│                   Elixir send/receive                           │
│  "简单直接，信任约定"                                            │
│                                                                 │
│  • 消息可以是任何 term                                          │
│  • 模式匹配处理不同消息                                          │
│  • 无界邮箱，需要自己控制背压                                    │
│  • 灵活，但类型错误在运行时发现                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   Actix Message/Handler                         │
│  "类型安全，编译器保证"                                          │
│                                                                 │
│  • 消息必须定义类型 + Result 类型                                │
│  • 每种消息一个 Handler（或用枚举合并）                          │
│  • 可配置邮箱，内置背压机制                                      │
│  • 繁琐，但类型错误在编译期发现                                  │
└─────────────────────────────────────────────────────────────────┘
```

**核心映射**：

| Elixir | Actix | 说明 |
|--------|-------|------|
| `send(pid, msg)` | `addr.do_send(msg)` | 异步发送 |
| `GenServer.call` | `addr.send(msg).await` | 同步发送 |
| `GenServer.cast` | `addr.do_send(msg)` | 异步发送 |
| 模式匹配消息 | `impl Handler<Msg>` | 消息处理 |
| `{:ok, result}` | `Result<T, E>` | 返回类型 |
| 无界邮箱 | 可配置邮箱 | 背压控制 |

下一篇，我们将探讨两者最深层的哲学差异——**容错机制**：Elixir 说"崩就崩"，Rust 说"我不让你崩"。

---

*上一篇：[进程与并发：spawn 的两种人生](/posts/elixir-rust-actix-3-concurrency/)*

*下一篇：[容错机制：Let it crash vs 不让你 crash](/posts/elixir-rust-actix-5-fault-tolerance/)*

*本系列：*
1. *[同源不同路：两种 Actor 哲学的碰撞](/posts/elixir-rust-actix-1-overview/)*
2. *[Actor 模型：GenServer vs Actix Actor](/posts/elixir-rust-actix-2-actor/)*
3. *[进程与并发：spawn 的两种人生](/posts/elixir-rust-actix-3-concurrency/)*
4. *消息传递：从 send 到 Handler（本篇）*
5. *[容错机制：Let it crash vs 不让你 crash](/posts/elixir-rust-actix-5-fault-tolerance/)*
6. *[模式匹配：熟悉的语法，不同的能力](/posts/elixir-rust-actix-6-pattern-matching/)*
7. *[状态管理：从 Agent/ETS 到 Rust 的选择](/posts/elixir-rust-actix-7-state/)*
