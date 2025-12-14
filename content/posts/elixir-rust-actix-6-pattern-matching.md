---
title: "Elixir 到 Rust Actix（六）：模式匹配——熟悉的语法，不同的能力"
date: 2025-12-14T23:20:00+08:00
draft: false
tags: ["Elixir", "Rust", "模式匹配", "类型系统"]
categories: ["技术"]
description: "Rust 的模式匹配看起来和 Elixir 很像——都有 match，都能解构，都能用 guard。但用起来，总觉得差点什么。Elixir 的 pin 操作符呢？Rust 为什么不能匹配任意类型？两者的模式匹配，看似相似，实则不同。"
series: ["Elixir 到 Rust Actix"]
---

Rust 的模式匹配看起来和 Elixir 很像——都有 match，都能解构，都能用 guard。但用起来，总觉得差点什么。Elixir 的 pin 操作符呢？Rust 为什么不能匹配任意类型？两者的模式匹配，看似相似，实则不同。

<!--more-->

## 前情回顾

前五篇我们对比了架构层面的差异：

- **哲学**、**Actor**、**并发**、**消息**、**容错**

今天来看一个语言层面的特性：**模式匹配**。

## 基础语法对比

### 最简单的 match

Elixir：

```elixir
case status do
  :ok -> "Success"
  :error -> "Failed"
  _ -> "Unknown"
end
```

Rust：

```rust
match status {
    Status::Ok => "Success",
    Status::Error => "Failed",
    _ => "Unknown",
}
```

看起来很像对吧？但细节差异很大。

### 元组解构

Elixir：

```elixir
case result do
  {:ok, value} -> "Got: #{value}"
  {:error, reason} -> "Error: #{reason}"
end
```

Rust：

```rust
match result {
    Ok(value) => format!("Got: {}", value),
    Err(reason) => format!("Error: {}", reason),
}
```

### 列表/数组解构

Elixir：

```elixir
case list do
  [] -> "Empty"
  [a, b, c] -> "Exactly three: #{a}, #{b}, #{c}"
  [head | tail] -> "Head: #{head}, Tail: #{inspect(tail)}"
end
```

Rust：

```rust
match slice {
    [] => "Empty".to_string(),
    [a, b, c] => format!("Exactly three: {}, {}, {}", a, b, c),
    [head, tail @ ..] => format!("Head: {}, Tail: {:?}", head, tail),
}
```

**注意**：
- Rust 的 `@ ..` 类似于 Elixir 的 `|`，但只能用于数组/切片，不能用于 Vec
- **模式匹配顺序很重要**：`[head, tail @ ..]` 会匹配所有非空切片，必须放在更具体的模式之后

## 函数参数的模式匹配

### Elixir：函数头模式匹配

```elixir
defmodule Calculator do
  def calculate({:add, a, b}), do: a + b
  def calculate({:sub, a, b}), do: a - b
  def calculate({:mul, a, b}), do: a * b
  def calculate({:div, _, 0}), do: {:error, :division_by_zero}
  def calculate({:div, a, b}), do: a / b
  def calculate(_), do: {:error, :unknown_operation}
end
```

调用时自动匹配：

```elixir
Calculator.calculate({:add, 1, 2})  # => 3
Calculator.calculate({:div, 10, 0}) # => {:error, :division_by_zero}
```

### Rust：需要显式 match

```rust
enum Operation {
    Add(i32, i32),
    Sub(i32, i32),
    Mul(i32, i32),
    Div(i32, i32),
}

fn calculate(op: Operation) -> Result<i32, &'static str> {
    match op {
        Operation::Add(a, b) => Ok(a + b),
        Operation::Sub(a, b) => Ok(a - b),
        Operation::Mul(a, b) => Ok(a * b),
        Operation::Div(_, 0) => Err("division by zero"),
        Operation::Div(a, b) => Ok(a / b),
    }
}
```

**Rust 没有函数头模式匹配**，需要在函数体内用 match。

## Guard 子句

### Elixir：when 子句

```elixir
def describe(n) when n < 0, do: "negative"
def describe(n) when n == 0, do: "zero"
def describe(n) when n > 0, do: "positive"

case value do
  x when x > 100 -> "large"
  x when x > 10 -> "medium"
  x -> "small"
end
```

### Rust：match guard

```rust
fn describe(n: i32) -> &'static str {
    match n {
        n if n < 0 => "negative",
        0 => "zero",
        n if n > 0 => "positive",
        _ => unreachable!(),
    }
}

match value {
    x if x > 100 => "large",
    x if x > 10 => "medium",
    _ => "small",
}
```

语法略有不同（`when` vs `if`），但功能类似。

## Elixir 独有：Pin 操作符

这是 Elixir 开发者转 Rust 时最容易"想念"的特性。

### 什么是 Pin？

```elixir
x = 1

case {1, 2} do
  {^x, y} -> "x is 1, y is #{y}"  # ^x 表示"匹配 x 的值"，而不是"绑定到 x"
  {a, b} -> "a is #{a}, b is #{b}"
end
# => "x is 1, y is 2"
```

**没有 `^`**：

```elixir
x = 1

case {1, 2} do
  {x, y} -> "x is #{x}, y is #{y}"  # x 被重新绑定为 1
end
# => "x is 1, y is 2"
# 但这里的 x 是新绑定的，不是检查原来的 x
```

### Rust 没有 Pin 操作符

Rust 的模式匹配中，变量名总是绑定新值：

```rust
let x = 1;

match (1, 2) {
    (x, y) => println!("x is {}, y is {}", x, y),
    // x 被重新绑定，不是检查原来的 x
}
```

**要检查特定值，必须显式写出**：

```rust
let x = 1;

match (1, 2) {
    (1, y) => println!("first is 1, y is {}", y),  // 直接写 1
    (a, b) => println!("a is {}, b is {}", a, b),
}

// 或者用 guard
match (1, 2) {
    (a, b) if a == x => println!("a equals x, b is {}", b),
    (a, b) => println!("a is {}, b is {}", a, b),
}
```

## 结构体/Map 解构

### Elixir：灵活的 Map 匹配

```elixir
# Map 匹配只需要指定关心的 key
case user do
  %{name: name, age: age} when age >= 18 -> "Adult: #{name}"
  %{name: name} -> "Minor: #{name}"
  _ -> "Unknown"
end

# 可以匹配部分字段
%{name: name} = %{name: "Alice", age: 30, email: "alice@example.com"}
# name => "Alice"，其他字段被忽略
```

### Rust：严格的结构体匹配

```rust
struct User {
    name: String,
    age: u32,
    email: String,
}

// 必须处理所有字段，或用 .. 忽略
match user {
    User { name, age, .. } if age >= 18 => format!("Adult: {}", name),
    User { name, .. } => format!("Minor: {}", name),
}

// 解构也类似
let User { name, .. } = user;
```

**关键差异**：Rust 必须用 `..` 显式忽略字段，Elixir 默认忽略未提及的字段。

## 类型的影响

### Elixir：运行时类型检查

```elixir
def process(value) do
  case value do
    x when is_integer(x) -> "Integer: #{x}"
    x when is_binary(x) -> "String: #{x}"
    x when is_list(x) -> "List: #{inspect(x)}"
    _ -> "Unknown type"
  end
end

process(42)        # => "Integer: 42"
process("hello")   # => "String: hello"
process([1, 2, 3]) # => "List: [1, 2, 3]"
```

**同一个函数可以接受任意类型**。

### Rust：编译时类型确定

```rust
// 不能这样写 - value 必须有确定的类型
fn process(value: ???) {
    match value {
        // 不能匹配"任意类型"
    }
}

// 要处理多种类型，必须用枚举
enum Value {
    Integer(i64),
    Text(String),
    List(Vec<Value>),
}

fn process(value: Value) -> String {
    match value {
        Value::Integer(x) => format!("Integer: {}", x),
        Value::Text(s) => format!("String: {}", s),
        Value::List(l) => format!("List: {:?}", l),
    }
}
```

**Rust 的模式匹配必须在已知类型上进行。**

## 穷尽性检查

### Rust：编译器强制穷尽

```rust
enum Status {
    Active,
    Inactive,
    Pending,
}

fn describe(status: Status) -> &'static str {
    match status {
        Status::Active => "active",
        Status::Inactive => "inactive",
        // 编译错误！没有处理 Pending
    }
}
```

这是 Rust 模式匹配的一大优势：**编译器保证你处理了所有情况**。

### Elixir：运行时失败

```elixir
def describe(status) do
  case status do
    :active -> "active"
    :inactive -> "inactive"
    # 如果传入 :pending，运行时抛出 CaseClauseError
  end
end
```

虽然 Dialyzer 可以帮忙发现，但不是强制的。

## 实战：消息路由器

让我们用模式匹配实现一个消息路由器。

### Elixir 实现

```elixir
defmodule MessageRouter do
  def route(message) do
    case message do
      # 用户相关消息
      {:user, :create, %{name: name}} when byte_size(name) > 0 ->
        {:ok, "Creating user: #{name}"}

      {:user, :get, id} when is_integer(id) and id > 0 ->
        {:ok, "Getting user: #{id}"}

      {:user, :delete, id} when is_integer(id) ->
        {:ok, "Deleting user: #{id}"}

      # 订单相关消息
      {:order, :create, %{items: items}} when length(items) > 0 ->
        {:ok, "Creating order with #{length(items)} items"}

      {:order, :cancel, order_id} ->
        {:ok, "Cancelling order: #{order_id}"}

      # 未知消息
      {type, action, _} ->
        {:error, "Unknown message: #{type}/#{action}"}

      _ ->
        {:error, "Invalid message format"}
    end
  end
end
```

### Rust 实现

```rust
enum UserAction {
    Create { name: String },
    Get { id: u64 },
    Delete { id: u64 },
}

enum OrderAction {
    Create { items: Vec<String> },
    Cancel { order_id: String },
}

enum Message {
    User(UserAction),
    Order(OrderAction),
}

fn route(message: Message) -> Result<String, String> {
    match message {
        // 用户相关消息
        Message::User(UserAction::Create { name }) if !name.is_empty() => {
            Ok(format!("Creating user: {}", name))
        }
        Message::User(UserAction::Create { .. }) => {
            Err("User name cannot be empty".to_string())
        }
        Message::User(UserAction::Get { id }) if id > 0 => {
            Ok(format!("Getting user: {}", id))
        }
        Message::User(UserAction::Get { .. }) => {
            Err("Invalid user id".to_string())
        }
        Message::User(UserAction::Delete { id }) => {
            Ok(format!("Deleting user: {}", id))
        }

        // 订单相关消息
        Message::Order(OrderAction::Create { items }) if !items.is_empty() => {
            Ok(format!("Creating order with {} items", items.len()))
        }
        Message::Order(OrderAction::Create { .. }) => {
            Err("Order must have at least one item".to_string())
        }
        Message::Order(OrderAction::Cancel { order_id }) => {
            Ok(format!("Cancelling order: {}", order_id))
        }
    }
}
```

### 对比

| 方面 | Elixir | Rust |
|------|--------|------|
| 代码量 | 更少 | 更多（需要定义类型） |
| 类型安全 | 运行时检查 | 编译时保证 |
| 灵活性 | 可以匹配任意结构 | 必须预定义类型 |
| 穷尽性 | 依赖程序员 | 编译器检查 |

## 常见问题

### Q：Rust 有类似 Elixir 的 `with` 语句吗？

**A：可以用 `?` 操作符 + 闭包模拟。**

Elixir：

```elixir
with {:ok, user} <- fetch_user(id),
     {:ok, profile} <- fetch_profile(user.profile_id),
     {:ok, posts} <- fetch_posts(user.id) do
  {:ok, %{user: user, profile: profile, posts: posts}}
else
  {:error, reason} -> {:error, reason}
end
```

Rust：

```rust
fn fetch_all(id: u64) -> Result<UserData, Error> {
    let user = fetch_user(id)?;
    let profile = fetch_profile(user.profile_id)?;
    let posts = fetch_posts(user.id)?;
    Ok(UserData { user, profile, posts })
}
```

`?` 操作符自动处理错误传播，比 `with` 更简洁。

### Q：如何在 Rust 中实现 Elixir 的函数头模式匹配？

**A：使用枚举 + match，或多个函数。**

```rust
// 方法 1：枚举 + match（推荐）
enum Command {
    Start,
    Stop,
    Status,
}

fn handle(cmd: Command) {
    match cmd {
        Command::Start => start(),
        Command::Stop => stop(),
        Command::Status => status(),
    }
}

// 方法 2：泛型 trait（复杂但灵活）
trait Handler<T> {
    fn handle(&self, input: T);
}
```

### Q：Rust 的 `if let` 和 `let else` 是什么？

**A：简化的模式匹配。**

```rust
// if let：只关心一种情况
if let Some(value) = optional {
    println!("Got: {}", value);
}

// let else (Rust 1.65+)：匹配失败时执行 else
let Some(value) = optional else {
    return Err("No value");
};
// value 在这里可用

// 等价于
let value = match optional {
    Some(v) => v,
    None => return Err("No value"),
};
```

## 总结

```text
模式匹配对比：

┌─────────────────────────────────────────────────────────────────┐
│                   Elixir 模式匹配                                │
│  "灵活动态，运行时检查"                                          │
│                                                                 │
│  • 可以匹配任意类型                                              │
│  • 函数头模式匹配                                                │
│  • Pin 操作符 (^)                                               │
│  • Map 部分匹配                                                  │
│  • 运行时类型检查                                                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   Rust 模式匹配                                  │
│  "类型安全，编译期检查"                                          │
│                                                                 │
│  • 必须在已知类型上匹配                                          │
│  • 必须在函数体内用 match                                        │
│  • 无 Pin 操作符                                                 │
│  • 结构体必须用 .. 忽略字段                                      │
│  • 编译期穷尽性检查                                              │
└─────────────────────────────────────────────────────────────────┘
```

**核心映射**：

| Elixir | Rust | 说明 |
|--------|------|------|
| `case x do` | `match x {}` | 基本模式匹配 |
| `when guard` | `if guard` | Guard 子句 |
| `^variable` | 无直接对应 | Pin 操作符 |
| `[h \| t]` | `[h, t @ ..]` | 列表解构 |
| `%{key: val}` | `Struct { key, .. }` | 结构体解构 |
| 函数头匹配 | `match` 在函数体内 | 多子句函数 |
| 运行时 `is_*` | 枚举类型 | 类型检查 |

**模式匹配是两种语言的核心特性，语法相似但哲学不同**：Elixir 追求灵活性，Rust 追求安全性。

---

*上一篇：[容错机制：Let it crash vs 不让你 crash](/posts/elixir-rust-actix-5-fault-tolerance/)*

*下一篇：[状态管理：从 Agent/ETS 到 Rust 的选择](/posts/elixir-rust-actix-7-state/)*

*本系列：*
1. *[同源不同路：两种 Actor 哲学的碰撞](/posts/elixir-rust-actix-1-overview/)*
2. *[Actor 模型：GenServer vs Actix Actor](/posts/elixir-rust-actix-2-actor/)*
3. *[进程与并发：spawn 的两种人生](/posts/elixir-rust-actix-3-concurrency/)*
4. *[消息传递：从 send 到 Handler](/posts/elixir-rust-actix-4-message/)*
5. *[容错机制：Let it crash vs 不让你 crash](/posts/elixir-rust-actix-5-fault-tolerance/)*
6. *模式匹配：熟悉的语法，不同的能力（本篇）*
7. *[状态管理：从 Agent/ETS 到 Rust 的选择](/posts/elixir-rust-actix-7-state/)*
