---
title: "Elixir 到 Rust Actix（七）：状态管理——从 Agent/ETS 到 Rust 的选择"
date: 2025-12-15T02:00:00+08:00
draft: false
tags: ["Elixir", "Rust", "Actix", "状态管理", "ETS", "Agent", "并发"]
categories: ["技术"]
description: "在 Elixir 里，状态管理有很多选择：Agent 简单直接，ETS 高性能共享，GenServer 灵活强大。在 Rust 里呢？Arc<Mutex<T>>？DashMap？Actor 状态？每种方案的适用场景是什么？本篇带你梳理清楚。"
series: ["Elixir 到 Rust Actix"]
---

在 Elixir 里，状态管理有很多选择：Agent 简单直接，ETS 高性能共享，GenServer 灵活强大。在 Rust 里呢？`Arc<Mutex<T>>`？DashMap？Actor 状态？每种方案的适用场景是什么？本篇带你梳理清楚。

<!--more-->

## 前情回顾

这是系列的最后一篇。前六篇我们对比了：

1. **哲学差异**：两种 Actor 的根本分歧
2. **Actor 实现**：GenServer vs Actix Actor
3. **并发原语**：spawn vs tokio::spawn
4. **消息传递**：send/receive vs Message/Handler
5. **容错机制**：Let it crash vs 编译期消灭
6. **模式匹配**：语法相似，能力不同

今天聚焦**状态管理**——并发系统的核心挑战。

## Elixir 的状态管理工具箱

### Agent：最简单的状态容器

```elixir
# 启动 Agent
{:ok, agent} = Agent.start_link(fn -> %{} end)

# 读取状态
state = Agent.get(agent, fn s -> s end)

# 更新状态
Agent.update(agent, fn s -> Map.put(s, :count, 1) end)

# 读取并更新
Agent.get_and_update(agent, fn s ->
  {s.count, Map.update(s, :count, 1, &(&1 + 1))}
end)
```

**适用场景**：简单状态存储，不需要复杂逻辑。

### ETS：高性能共享表

```elixir
# 创建表
table = :ets.new(:my_cache, [:set, :public])

# 插入
:ets.insert(table, {:key, "value"})

# 查询
case :ets.lookup(table, :key) do
  [{:key, value}] -> {:ok, value}
  [] -> :error
end

# 原子更新计数器
:ets.update_counter(table, :visits, 1)
```

**适用场景**：高频读写、多进程共享、需要原子操作。

### GenServer：灵活的有状态进程

```elixir
defmodule Counter do
  use GenServer

  def start_link(initial) do
    GenServer.start_link(__MODULE__, initial, name: __MODULE__)
  end

  def increment, do: GenServer.call(__MODULE__, :increment)
  def get, do: GenServer.call(__MODULE__, :get)

  @impl true
  def init(initial), do: {:ok, initial}

  @impl true
  def handle_call(:increment, _from, count), do: {:reply, count + 1, count + 1}
  def handle_call(:get, _from, count), do: {:reply, count, count}
end
```

**适用场景**：需要复杂逻辑、生命周期管理、监督树集成。

### 对比表

| 工具 | 复杂度 | 性能 | 适用场景 |
|------|--------|------|---------|
| Agent | 最简单 | 一般 | 简单状态存储 |
| ETS | 中等 | 最高 | 高频读写、缓存 |
| GenServer | 较复杂 | 一般 | 复杂逻辑、需要监督 |

## Rust 的状态管理方案

### Arc<Mutex<T>>：最基础的方案

```rust
use std::sync::{Arc, Mutex};

// 创建共享状态
let state = Arc::new(Mutex::new(HashMap::new()));

// 在多个任务中共享
let state_clone = Arc::clone(&state);
tokio::spawn(async move {
    let mut guard = state_clone.lock().unwrap();
    guard.insert("key", "value");
});

// 读取
let guard = state.lock().unwrap();
println!("{:?}", guard.get("key"));
```

**问题**：
- Mutex 锁住整个数据结构
- 写操作会阻塞读操作
- 性能可能成为瓶颈

### RwLock：读写分离

```rust
use std::sync::{Arc, RwLock};

let state = Arc::new(RwLock::new(HashMap::new()));

// 读取（多个读可以并发）
{
    let guard = state.read().unwrap();
    println!("{:?}", guard.get("key"));
}

// 写入（独占访问）
{
    let mut guard = state.write().unwrap();
    guard.insert("key", "value");
}
```

**适用场景**：读多写少的场景。

### DashMap：并发 HashMap

```rust
// 需要在 Cargo.toml 中添加: dashmap = "5"
use dashmap::DashMap;

// 创建
let map: DashMap<String, String> = DashMap::new();

// 插入（无需手动锁）
map.insert("key".to_string(), "value".to_string());

// 读取
if let Some(value) = map.get("key") {
    println!("Got: {}", *value);
}

// 原子更新
map.entry("counter".to_string())
    .and_modify(|v| *v = format!("{}_updated", v))
    .or_insert("initial".to_string());
```

**优点**：分段锁，细粒度并发，API 类似 HashMap。

### Actix Actor 状态

```rust
struct CacheActor {
    data: HashMap<String, String>,
}

impl Actor for CacheActor {
    type Context = Context<Self>;
}

#[derive(Message)]
#[rtype(result = "Option<String>")]
struct Get(String);

#[derive(Message)]
#[rtype(result = "()")]
struct Set(String, String);

impl Handler<Get> for CacheActor {
    type Result = Option<String>;

    fn handle(&mut self, msg: Get, _ctx: &mut Context<Self>) -> Self::Result {
        self.data.get(&msg.0).cloned()
    }
}

impl Handler<Set> for CacheActor {
    type Result = ();

    fn handle(&mut self, msg: Set, _ctx: &mut Context<Self>) {
        self.data.insert(msg.0, msg.1);
    }
}
```

**优点**：
- 状态被 Actor 独占，无需锁
- 消息串行处理，保证一致性
- 可以添加复杂逻辑

## 对照表：Elixir vs Rust

```text
┌────────────────────────────────────────────────────────────────┐
│                    状态管理对照                                 │
├──────────────────┬─────────────────────────────────────────────┤
│     Elixir       │              Rust                          │
├──────────────────┼─────────────────────────────────────────────┤
│ Agent            │ Arc<Mutex<T>> / Arc<RwLock<T>>             │
│                  │ 简单封装，适合简单状态                      │
├──────────────────┼─────────────────────────────────────────────┤
│ ETS              │ DashMap / 内存数据库（sled）                │
│                  │ 高性能共享，细粒度锁                        │
├──────────────────┼─────────────────────────────────────────────┤
│ GenServer state  │ Actix Actor state                          │
│                  │ Actor 独占，消息串行访问                    │
├──────────────────┼─────────────────────────────────────────────┤
│ :persistent_term │ lazy_static / once_cell                    │
│                  │ 启动时初始化的只读配置                      │
├──────────────────┼─────────────────────────────────────────────┤
│ Process Dict     │ tokio::task_local!                         │
│                  │ 任务本地存储                                │
└──────────────────┴─────────────────────────────────────────────┘
```

## 深入对比：Agent vs Arc<Mutex<T>>

### Elixir Agent

```elixir
defmodule CounterAgent do
  def start_link(initial) do
    Agent.start_link(fn -> initial end, name: __MODULE__)
  end

  def increment do
    Agent.update(__MODULE__, &(&1 + 1))
  end

  def get do
    Agent.get(__MODULE__, & &1)
  end
end
```

**特点**：
- 状态在独立进程中
- 通过消息传递访问
- 进程崩溃状态丢失
- 进程隔离，无锁

### Rust Arc<Mutex<T>> 封装

```rust
use std::sync::{Arc, Mutex};

#[derive(Clone)]
struct Counter {
    inner: Arc<Mutex<i32>>,
}

impl Counter {
    fn new(initial: i32) -> Self {
        Self {
            inner: Arc::new(Mutex::new(initial)),
        }
    }

    fn increment(&self) {
        let mut guard = self.inner.lock().unwrap();
        *guard += 1;
    }

    fn get(&self) -> i32 {
        *self.inner.lock().unwrap()
    }
}
```

**特点**：
- 状态在堆上，通过引用计数共享
- 通过锁保护访问
- 锁 panic 可能导致问题
- 直接内存访问，有锁开销

### 关键差异

| 方面 | Elixir Agent | Rust Arc<Mutex<T>> |
|------|-------------|-------------------|
| 访问方式 | 消息传递 | 锁 |
| 并发模型 | 进程隔离 | 共享内存 |
| 错误处理 | 进程崩溃，可重启 | panic 可能毒化锁 |
| 性能 | 消息开销 | 锁竞争开销 |
| 复杂度 | 更简单 | 需要理解所有权 |

## 深入对比：ETS vs DashMap

### Elixir ETS

```elixir
defmodule Cache do
  def start do
    :ets.new(:cache, [:set, :public, :named_table])
  end

  def get(key) do
    case :ets.lookup(:cache, key) do
      [{^key, value}] -> {:ok, value}
      [] -> :error
    end
  end

  def put(key, value) do
    :ets.insert(:cache, {key, value})
  end

  def update_counter(key, increment) do
    :ets.update_counter(:cache, key, increment, {key, 0})
  end
end
```

### Rust DashMap

```rust
use dashmap::DashMap;
use std::sync::atomic::{AtomicI64, Ordering};

struct Cache {
    data: DashMap<String, String>,
    counters: DashMap<String, AtomicI64>,
}

impl Cache {
    fn new() -> Self {
        Self {
            data: DashMap::new(),
            counters: DashMap::new(),
        }
    }

    fn get(&self, key: &str) -> Option<String> {
        self.data.get(key).map(|v| v.value().clone())
    }

    fn put(&self, key: String, value: String) {
        self.data.insert(key, value);
    }

    fn update_counter(&self, key: &str, increment: i64) -> i64 {
        self.counters
            .entry(key.to_string())
            .or_insert(AtomicI64::new(0))
            .fetch_add(increment, Ordering::SeqCst)
    }
}
```

### 关键差异

| 方面 | ETS | DashMap |
|------|-----|---------|
| 存储位置 | BEAM 虚拟机堆 | 进程堆 |
| 并发机制 | 读写锁 + 细粒度锁 | 分段锁 |
| 原子操作 | 内置 `update_counter` | 需要 Atomic 类型 |
| 进程共享 | 天然支持 | 需要 Arc |
| 持久化 | DETS | sled/rocksdb |

## 实战：缓存服务

### Elixir 实现

```elixir
defmodule CacheService do
  use GenServer

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def get(key), do: GenServer.call(__MODULE__, {:get, key})
  def put(key, value, ttl \\ nil), do: GenServer.cast(__MODULE__, {:put, key, value, ttl})
  def delete(key), do: GenServer.cast(__MODULE__, {:delete, key})

  @impl true
  def init(_opts) do
    # 使用 ETS 存储实际数据
    table = :ets.new(:cache_data, [:set, :private])
    schedule_cleanup()
    {:ok, %{table: table}}
  end

  @impl true
  def handle_call({:get, key}, _from, state) do
    result = case :ets.lookup(state.table, key) do
      [{^key, value, expiry}] ->
        if expiry == nil or expiry > System.monotonic_time(:millisecond) do
          {:ok, value}
        else
          :ets.delete(state.table, key)
          :miss
        end
      [] -> :miss
    end
    {:reply, result, state}
  end

  @impl true
  def handle_cast({:put, key, value, ttl}, state) do
    expiry = if ttl, do: System.monotonic_time(:millisecond) + ttl, else: nil
    :ets.insert(state.table, {key, value, expiry})
    {:noreply, state}
  end

  def handle_cast({:delete, key}, state) do
    :ets.delete(state.table, key)
    {:noreply, state}
  end

  @impl true
  def handle_info(:cleanup, state) do
    now = System.monotonic_time(:millisecond)
    # 清理过期条目
    :ets.select_delete(state.table, [
      {{:_, :_, :"$1"}, [{:"/=", :"$1", nil}, {:<, :"$1", now}], [true]}
    ])
    schedule_cleanup()
    {:noreply, state}
  end

  defp schedule_cleanup, do: Process.send_after(self(), :cleanup, 60_000)
end
```

### Rust Actix 实现

```rust
use actix::prelude::*;
use dashmap::DashMap;
use std::time::{Duration, Instant};

struct CacheEntry {
    value: String,
    expiry: Option<Instant>,
}

struct CacheActor {
    data: DashMap<String, CacheEntry>,
}

impl Actor for CacheActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        // 定期清理
        ctx.run_interval(Duration::from_secs(60), |act, _ctx| {
            act.cleanup();
        });
    }
}

impl CacheActor {
    fn new() -> Self {
        Self {
            data: DashMap::new(),
        }
    }

    fn cleanup(&self) {
        let now = Instant::now();
        self.data.retain(|_, entry| {
            entry.expiry.map_or(true, |exp| exp > now)
        });
    }
}

// 消息定义
#[derive(Message)]
#[rtype(result = "Option<String>")]
struct Get(String);

#[derive(Message)]
#[rtype(result = "()")]
struct Put {
    key: String,
    value: String,
    ttl: Option<Duration>,
}

#[derive(Message)]
#[rtype(result = "()")]
struct Delete(String);

// Handler 实现
impl Handler<Get> for CacheActor {
    type Result = Option<String>;

    fn handle(&mut self, msg: Get, _ctx: &mut Context<Self>) -> Self::Result {
        // 先检查是否存在且有效
        let result = self.data.get(&msg.0).and_then(|entry| {
            if entry.expiry.map_or(true, |exp| exp > Instant::now()) {
                Some(entry.value.clone())
            } else {
                None
            }
        });

        // 如果返回 None 且 key 存在，说明已过期，需要删除
        if result.is_none() {
            self.data.remove(&msg.0);
        }

        result
    }
}

impl Handler<Put> for CacheActor {
    type Result = ();

    fn handle(&mut self, msg: Put, _ctx: &mut Context<Self>) {
        let expiry = msg.ttl.map(|ttl| Instant::now() + ttl);
        self.data.insert(msg.key, CacheEntry {
            value: msg.value,
            expiry,
        });
    }
}

impl Handler<Delete> for CacheActor {
    type Result = ();

    fn handle(&mut self, msg: Delete, _ctx: &mut Context<Self>) {
        self.data.remove(&msg.0);
    }
}
```

## 如何选择？

### Elixir 选择指南

```text
需要简单状态？
  └─► Agent

需要高性能、多进程共享？
  └─► ETS

需要复杂逻辑、生命周期管理？
  └─► GenServer

需要只读配置？
  └─► :persistent_term

需要持久化？
  └─► DETS / Mnesia
```

### Rust 选择指南

```text
简单状态、低并发？
  └─► Arc<Mutex<T>>

读多写少？
  └─► Arc<RwLock<T>>

高并发、细粒度访问？
  └─► DashMap

需要复杂逻辑、消息驱动？
  └─► Actix Actor

需要持久化？
  └─► sled / rocksdb

全局配置？
  └─► once_cell / lazy_static
```

## 常见问题

### Q：Rust 的 Mutex 会死锁吗？

**A：会，但可以避免。**

```rust
// 死锁示例
let a = Mutex::new(1);
let b = Mutex::new(2);

// 线程 1
let _guard_a = a.lock();
let _guard_b = b.lock();  // 等待 b

// 线程 2
let _guard_b = b.lock();
let _guard_a = a.lock();  // 等待 a，死锁！

// 避免：总是按相同顺序获取锁
// 或者使用 try_lock + 超时
```

### Q：ETS 的数据在进程崩溃后会丢失吗？

**A：取决于 ETS 表的配置。**

```elixir
# 默认：表归创建者所有，创建者崩溃表就销毁
:ets.new(:my_table, [])

# 使用 heir 选项：创建者崩溃后，表转移给继承者
:ets.new(:my_table, [{:heir, heir_pid, :some_data}])

# 或者用 named_table + 监督树确保表所有者重启
```

### Q：DashMap 和 RwLock<HashMap> 哪个更好？

**A：取决于访问模式。**

```text
DashMap 更好：
  • 高并发读写
  • 不同 key 的操作可以并发
  • 需要细粒度锁

RwLock<HashMap> 更好：
  • 读远多于写
  • 经常需要遍历整个 map
  • 内存占用敏感
```

## 总结

```text
状态管理的选择：

┌─────────────────────────────────────────────────────────────────┐
│                   Elixir 状态管理                                │
│  "进程隔离，消息传递"                                            │
│                                                                 │
│  Agent    → 简单状态，消息访问                                   │
│  ETS      → 高性能共享，细粒度锁                                 │
│  GenServer → 复杂逻辑，完整生命周期                              │
│  DETS     → 持久化到磁盘                                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   Rust 状态管理                                  │
│  "所有权 + 锁，编译期安全"                                       │
│                                                                 │
│  Arc<Mutex<T>> → 简单状态，独占锁                                │
│  Arc<RwLock<T>> → 读多写少，读写分离                             │
│  DashMap    → 高并发，分段锁                                     │
│  Actor 状态 → 复杂逻辑，消息驱动                                 │
│  sled       → 持久化到磁盘                                       │
└─────────────────────────────────────────────────────────────────┘
```

**核心映射**：

| Elixir | Rust | 说明 |
|--------|------|------|
| Agent | Arc<Mutex<T>> | 简单状态容器 |
| ETS | DashMap | 高性能共享表 |
| GenServer state | Actor state | 复杂逻辑状态 |
| :persistent_term | once_cell | 只读全局配置 |
| DETS | sled | 持久化存储 |
| Process dictionary | task_local! | 任务本地存储 |

## 系列总结

七篇文章，我们从**哲学差异**到**具体实现**，全面对比了 Elixir 和 Rust Actix 两种 Actor 实现。

```text
核心结论：

1. 同源不同路
   都是 Actor 模型，但哲学完全不同
   Elixir：运行时容错，Let it crash
   Rust：编译期安全，类型保证

2. 各有所长
   Elixir：高并发、容错、分布式、快速迭代
   Rust：性能、内存安全、类型安全、可预测性

3. 可以结合
   不是二选一，而是用对的工具做对的事
   Elixir 做网关和编排，Rust 做性能关键路径
```

感谢你读完这个系列！希望对你理解两种技术有所帮助。

---

*上一篇：[模式匹配：熟悉的语法，不同的能力](/posts/elixir-rust-actix-6-pattern-matching/)*

*本系列：*
1. *[同源不同路：两种 Actor 哲学的碰撞](/posts/elixir-rust-actix-1-overview/)*
2. *[Actor 模型：GenServer vs Actix Actor](/posts/elixir-rust-actix-2-actor/)*
3. *[进程与并发：spawn 的两种人生](/posts/elixir-rust-actix-3-concurrency/)*
4. *[消息传递：从 send 到 Handler](/posts/elixir-rust-actix-4-message/)*
5. *[容错机制：Let it crash vs 不让你 crash](/posts/elixir-rust-actix-5-fault-tolerance/)*
6. *[模式匹配：熟悉的语法，不同的能力](/posts/elixir-rust-actix-6-pattern-matching/)*
7. *状态管理：从 Agent/ETS 到 Rust 的选择（本篇）*
