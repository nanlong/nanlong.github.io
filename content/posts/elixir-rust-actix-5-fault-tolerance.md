---
title: "Elixir 到 Rust Actix（五）：容错机制——Let it crash vs 不让你 crash"
date: 2025-12-15T00:00:00+08:00
draft: false
tags: ["Elixir", "Rust", "Actix", "容错", "Supervisor", "错误处理"]
categories: ["技术"]
description: "Elixir 说'崩就崩，有人管'——监督树会重启你。Rust 说'我不让你崩'——编译器会阻止你。两种截然不同的哲学，一个拥抱失败，一个预防失败。什么时候该用哪种？"
series: ["Elixir 到 Rust Actix"]
---

Elixir 说"崩就崩，有人管"——监督树会重启你。Rust 说"我不让你崩"——编译器会阻止你。两种截然不同的哲学，一个拥抱失败，一个预防失败。什么时候该用哪种？

<!--more-->

## 前情回顾

前四篇我们对比了：

- **哲学差异**：BEAM vs Native，GC vs 所有权
- **Actor 实现**：GenServer vs Actix Actor
- **并发原语**：spawn vs tokio::spawn
- **消息传递**：send/receive vs Message/Handler

今天深入**容错机制**——这是两者哲学差异最深刻的体现。

## 两种哲学的根本分歧

### Elixir："拥抱失败"

```text
Elixir/Erlang 的世界观：

  现实：
    • 硬件会故障
    • 网络会中断
    • 代码有 bug
    • 未知错误总会发生

  结论：
    既然错误不可避免，那就设计系统来容忍错误

  策略：
    • 进程隔离（一个崩不影响其他）
    • 监督树（崩了自动重启）
    • Let it crash（不过度防御）
```

### Rust："预防失败"

```text
Rust 的世界观：

  现实：
    • 很多错误是可预防的
    • 类型错误可以编译期发现
    • 内存错误可以静态分析
    • 空指针可以用类型消灭

  结论：
    能在编译期消灭的错误，就不要留到运行时

  策略：
    • 类型系统（编译期检查）
    • 所有权系统（消灭内存错误）
    • Result/Option（强制处理错误）
```

## Elixir 的监督树

### 监督树是什么？

```text
监督树结构：

              ┌───────────────┐
              │  Application  │
              │   Supervisor  │
              └───────┬───────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Worker  │  │ Worker  │  │  Sub    │
   │   A     │  │   B     │  │Supervisor
   └─────────┘  └─────────┘  └────┬────┘
                                  │
                           ┌──────┴──────┐
                           │             │
                           ▼             ▼
                      ┌─────────┐  ┌─────────┐
                      │ Worker  │  │ Worker  │
                      │   C     │  │   D     │
                      └─────────┘  └─────────┘
```

### 代码实现

```elixir
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      {MyApp.DatabaseWorker, []},
      {MyApp.CacheWorker, []},
      {MyApp.WebSupervisor, []}
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end

defmodule MyApp.WebSupervisor do
  use Supervisor

  def start_link(arg) do
    Supervisor.start_link(__MODULE__, arg, name: __MODULE__)
  end

  def init(_arg) do
    children = [
      {MyApp.RequestHandler, []},
      {MyApp.WebSocketHandler, []}
    ]

    Supervisor.init(children, strategy: :rest_for_one)
  end
end
```

### 重启策略

```text
┌─────────────────────────────────────────────────────────────────┐
│                      重启策略                                    │
├──────────────────┬──────────────────────────────────────────────┤
│  one_for_one     │  谁崩重启谁                                   │
│                  │  A 崩了 → 只重启 A                            │
├──────────────────┼──────────────────────────────────────────────┤
│  one_for_all     │  一个崩全部重启                               │
│                  │  A 崩了 → 重启 A, B, C 全部                   │
├──────────────────┼──────────────────────────────────────────────┤
│  rest_for_one    │  崩的和后面的都重启                           │
│                  │  B 崩了 → 重启 B, C（A 不动）                 │
└──────────────────┴──────────────────────────────────────────────┘
```

### Let it crash 实战

```elixir
defmodule MyApp.Worker do
  use GenServer

  def handle_call({:divide, a, b}, _from, state) do
    # 不做防御，直接算
    result = a / b
    {:reply, result, state}
  end
end

# 调用方
case GenServer.call(worker, {:divide, 10, 0}) do
  result -> IO.puts("Result: #{result}")
end
# Worker 崩溃了？没关系，Supervisor 会重启它
```

**关键思想**：不是不处理错误，而是把错误处理**上移**到监督者层面。

## Rust 的错误处理

### Result 类型：强制处理

```rust
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err("Division by zero".to_string())
    } else {
        Ok(a / b)
    }
}

fn main() {
    let result = divide(10, 0);

    // 你必须处理 Result，否则编译警告/错误
    match result {
        Ok(value) => println!("Result: {}", value),
        Err(e) => println!("Error: {}", e),
    }

    // 或者用 ? 传播
    fn caller() -> Result<(), String> {
        let value = divide(10, 0)?;  // 错误会向上传播
        println!("Result: {}", value);
        Ok(())
    }
}
```

### Option 类型：消灭 null

```rust
fn find_user(id: u64) -> Option<User> {
    // 要么返回 Some(user)，要么返回 None
    // 不存在 null pointer exception
}

fn main() {
    match find_user(123) {
        Some(user) => println!("Found: {}", user.name),
        None => println!("User not found"),
    }

    // 或者用 if let
    if let Some(user) = find_user(123) {
        println!("Found: {}", user.name);
    }
}
```

### panic! 是最后手段

```rust
fn must_have_user(id: u64) -> User {
    find_user(id).expect("User must exist")
    // 如果 None，程序 panic
    // 这应该只在"不可能发生"的情况下使用
}
```

## Actix 中的错误处理

### Handler 的 Result 类型

```rust
#[derive(Message)]
#[rtype(result = "Result<User, DbError>")]
struct GetUser {
    id: u64,
}

impl Handler<GetUser> for UserActor {
    type Result = Result<User, DbError>;

    fn handle(&mut self, msg: GetUser, _ctx: &mut Context<Self>) -> Self::Result {
        self.db
            .get_user(msg.id)
            .ok_or(DbError::NotFound)
    }
}

// 调用方
async fn get_user(addr: Addr<UserActor>, id: u64) -> Result<User, AppError> {
    let result = addr.send(GetUser { id }).await??;
    //                                       ^^ 两个 ?
    //                     第一个处理 MailboxError
    //                     第二个处理 DbError
    Ok(result)
}
```

### Actor 崩溃的处理

Actix Actor panic 时：

```rust
impl Handler<DangerousMsg> for MyActor {
    type Result = ();

    fn handle(&mut self, _msg: DangerousMsg, _ctx: &mut Context<Self>) {
        panic!("Oops!");  // Actor 会停止
    }
}

// 发送方
let result = addr.send(DangerousMsg).await;
match result {
    Ok(_) => println!("Success"),
    Err(MailboxError::Closed) => println!("Actor died!"),
    Err(MailboxError::Timeout) => println!("Timeout"),
}
```

### 模拟监督树

Actix 没有内置监督树，但可以实现类似模式：

```rust
struct Supervisor {
    workers: Vec<Addr<Worker>>,
}

impl Actor for Supervisor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        // 定期检查 workers 状态
        ctx.run_interval(Duration::from_secs(1), |act, _ctx| {
            // 先收集需要重启的索引（避免借用冲突）
            let to_restart: Vec<usize> = act.workers
                .iter()
                .enumerate()
                .filter(|(_, addr)| !addr.connected())
                .map(|(i, _)| i)
                .collect();

            // 然后重启它们
            for i in to_restart {
                let new_worker = Worker::new().start();
                act.workers[i] = new_worker;
                println!("Restarted worker {}", i);
            }
        });
    }
}

// 或者用第三方库如 bastion
```

## 错误传播对比

### Elixir：进程链接和监控

```elixir
# 链接：一个崩，另一个也崩
pid = spawn_link(fn -> raise "oops" end)
# 当前进程也会崩溃（除非 trap_exit）

# 监控：一个崩，另一个收到通知
{pid, ref} = spawn_monitor(fn -> raise "oops" end)
receive do
  {:DOWN, ^ref, :process, ^pid, reason} ->
    IO.puts("Process died: #{inspect(reason)}")
end
# 当前进程不受影响

# trap_exit：捕获退出信号
Process.flag(:trap_exit, true)
pid = spawn_link(fn -> raise "oops" end)
receive do
  {:EXIT, ^pid, reason} ->
    IO.puts("Caught exit: #{inspect(reason)}")
end
```

### Rust：Result 和 ?

```rust
// ? 操作符传播错误
fn process_data(data: &str) -> Result<Output, Error> {
    let parsed = parse(data)?;         // 错误传播
    let validated = validate(parsed)?; // 错误传播
    let result = transform(validated)?; // 错误传播
    Ok(result)
}

// 调用链
fn main() -> Result<(), Error> {
    let data = read_file("input.txt")?;
    let output = process_data(&data)?;
    write_file("output.txt", &output)?;
    Ok(())
}
```

## 实战：带重试的服务

### Elixir 实现

```elixir
defmodule RetryWorker do
  use GenServer

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def do_work(data) do
    GenServer.call(__MODULE__, {:work, data})
  end

  @impl true
  def init(opts) do
    {:ok, %{max_retries: opts[:max_retries] || 3}}
  end

  @impl true
  def handle_call({:work, data}, _from, state) do
    result = do_with_retry(data, state.max_retries)
    {:reply, result, state}
  end

  defp do_with_retry(data, retries_left) when retries_left > 0 do
    case try_work(data) do
      {:ok, result} -> {:ok, result}
      {:error, _reason} ->
        Process.sleep(1000)  # 等待后重试
        do_with_retry(data, retries_left - 1)
    end
  end

  defp do_with_retry(_data, 0), do: {:error, :max_retries_exceeded}

  defp try_work(data) do
    # 模拟可能失败的操作
    if :rand.uniform() > 0.5 do
      {:ok, "Processed: #{data}"}
    else
      {:error, :random_failure}
    end
  end
end
```

### Rust Actix 实现

```rust
use actix::prelude::*;
use std::time::Duration;

struct RetryWorker {
    max_retries: u32,
}

impl Actor for RetryWorker {
    type Context = Context<Self>;
}

#[derive(Message)]
#[rtype(result = "Result<String, WorkerError>")]
struct DoWork {
    data: String,
}

#[derive(Debug)]
enum WorkerError {
    MaxRetriesExceeded,
    WorkFailed(String),
}

impl Handler<DoWork> for RetryWorker {
    type Result = ResponseFuture<Result<String, WorkerError>>;

    fn handle(&mut self, msg: DoWork, _ctx: &mut Context<Self>) -> Self::Result {
        let max_retries = self.max_retries;

        Box::pin(async move {
            do_with_retry(&msg.data, max_retries).await
        })
    }
}

async fn do_with_retry(data: &str, mut retries_left: u32) -> Result<String, WorkerError> {
    while retries_left > 0 {
        match try_work(data).await {
            Ok(result) => return Ok(result),
            Err(_) => {
                retries_left -= 1;
                if retries_left > 0 {
                    tokio::time::sleep(Duration::from_secs(1)).await;
                }
            }
        }
    }
    Err(WorkerError::MaxRetriesExceeded)
}

// 需要在 Cargo.toml 中添加: rand = "0.8"
async fn try_work(data: &str) -> Result<String, WorkerError> {
    // 模拟可能失败的操作
    if rand::random::<f64>() > 0.5 {
        Ok(format!("Processed: {}", data))
    } else {
        Err(WorkerError::WorkFailed("Random failure".into()))
    }
}
```

## 什么时候用哪种策略？

### 选择 "Let it crash"（Elixir 风格）：

```text
✓ 分布式系统（节点可能随时失联）
✓ 长期运行的服务（需要自我恢复）
✓ 错误难以预测或枚举
✓ 隔离比预防更重要
✓ 需要热更新
```

### 选择 "编译期消灭"（Rust 风格）：

```text
✓ 对正确性要求极高（金融、安全相关）
✓ 错误可以明确枚举
✓ 性能关键（不想有监督树的开销）
✓ 资源受限环境
✓ 需要静态分析和验证
```

### 两者可以结合

```text
实际项目中的混合策略：

  ┌─────────────────────────────────────────┐
  │             应用层                       │
  │  Elixir: 监督树管理整体架构             │
  │          Let it crash 处理未知错误      │
  └─────────────────────┬───────────────────┘
                        │
  ┌─────────────────────▼───────────────────┐
  │             核心层                       │
  │  Rust: 类型安全处理关键逻辑             │
  │        Result/Option 处理已知错误       │
  └─────────────────────────────────────────┘
```

## 常见问题

### Q：Rust 程序 panic 了怎么办？

**A：panic 应该是"不应该发生"的情况。**

```rust
// 不要这样：用 panic 处理正常错误
fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("Division by zero!");  // 不好
    }
    a / b
}

// 应该这样：用 Result 处理可预期的错误
fn divide(a: i32, b: i32) -> Result<i32, &'static str> {
    if b == 0 {
        Err("Division by zero")
    } else {
        Ok(a / b)
    }
}

// panic 只用于"不可能发生"的情况
fn get_first(vec: &[i32]) -> i32 {
    // 调用方保证 vec 非空
    vec.first().expect("vec should not be empty")
}
```

### Q：Elixir 的 Let it crash 不是不负责任吗？

**A：恰恰相反，这是更负责任的做法。**

```text
传统做法：
  try {
    doSomething();
  } catch (Exception e) {
    log(e);
    // 然后呢？系统状态可能已经不一致了
  }

Let it crash：
  进程崩溃
  → 状态被清理（进程有自己的堆）
  → Supervisor 重启一个干净的进程
  → 系统状态保证一致
```

**Let it crash 是"快速失败 + 干净恢复"的策略。**

### Q：Actix 如何实现类似 Elixir 的监督树？

**A：手动实现或使用第三方库。**

```rust
// 简单的监督模式
struct SupervisorActor {
    child_spec: fn() -> Addr<ChildActor>,
    child: Option<Addr<ChildActor>>,
}

impl Actor for SupervisorActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        self.start_child();
        self.watch_child(ctx);
    }
}

impl SupervisorActor {
    fn start_child(&mut self) {
        self.child = Some((self.child_spec)());
    }

    fn watch_child(&self, ctx: &mut Context<Self>) {
        ctx.run_interval(Duration::from_millis(100), |act, _ctx| {
            if let Some(ref child) = act.child {
                if !child.connected() {
                    println!("Child died, restarting...");
                    act.start_child();
                }
            }
        });
    }
}
```

## 总结

```text
容错的两种哲学：

┌─────────────────────────────────────────────────────────────────┐
│               Elixir: Let it crash                              │
│  "错误会发生，设计系统来容忍它"                                  │
│                                                                 │
│  • 进程隔离：一个崩不影响其他                                   │
│  • 监督树：自动重启失败的进程                                   │
│  • 快速失败：不过度防御，崩了重来                               │
│  • 适合：分布式系统、长期运行的服务                             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│               Rust: 编译期消灭                                   │
│  "能预防的错误，就不要留到运行时"                                │
│                                                                 │
│  • 类型系统：编译期发现类型错误                                  │
│  • Result/Option：强制处理错误                                   │
│  • 所有权：消灭内存错误                                          │
│  • 适合：正确性关键、性能关键的系统                              │
└─────────────────────────────────────────────────────────────────┘
```

**核心映射**：

| Elixir | Rust | 说明 |
|--------|------|------|
| Supervisor | 手动实现 / bastion | 监督进程 |
| spawn_link | 无直接对应 | 链接的进程 |
| spawn_monitor | 手动检查 connected() | 监控进程 |
| try/rescue | Result + ? | 错误处理 |
| raise/throw | panic! | 不可恢复错误 |
| :error tuple | Err variant | 可恢复错误 |

**两种哲学不是对立的，而是互补的。** 理解两者，才能在不同场景下做出正确选择。

---

*上一篇：[消息传递：从 send 到 Handler](/posts/elixir-rust-actix-4-message/)*

*下一篇：[模式匹配：熟悉的语法，不同的能力](/posts/elixir-rust-actix-6-pattern-matching/)*

*本系列：*
1. *[同源不同路：两种 Actor 哲学的碰撞](/posts/elixir-rust-actix-1-overview/)*
2. *[Actor 模型：GenServer vs Actix Actor](/posts/elixir-rust-actix-2-actor/)*
3. *[进程与并发：spawn 的两种人生](/posts/elixir-rust-actix-3-concurrency/)*
4. *[消息传递：从 send 到 Handler](/posts/elixir-rust-actix-4-message/)*
5. *容错机制：Let it crash vs 不让你 crash（本篇）*
6. *[模式匹配：熟悉的语法，不同的能力](/posts/elixir-rust-actix-6-pattern-matching/)*
7. *[状态管理：从 Agent/ETS 到 Rust 的选择](/posts/elixir-rust-actix-7-state/)*
