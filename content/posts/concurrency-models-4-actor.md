---
title: "并发模型演进史（四）：Actor 模型——不要共享，要传递"
date: 2025-12-08T22:30:00+08:00
draft: false
tags: ["并发", "Actor", "Erlang", "Elixir", "Akka", "消息传递"]
categories: ["技术"]
description: "如果共享状态是万恶之源，那就别共享。Actor 模型的哲学很简单：每个 Actor 都是一座孤岛，唯一的沟通方式是发消息。这个 1973 年的理论，在 Erlang 手里变成了电信级的可靠性。"
series: ["并发模型演进史"]
---

如果共享状态是万恶之源，那就别共享。Actor 模型的哲学很简单：每个 Actor 都是一座孤岛，唯一的沟通方式是发消息。这个 1973 年的理论，在 Erlang 手里变成了电信级的可靠性。

<!--more-->

## 从一个问题说起

回顾前几篇的内容：

| 模型 | 解决了什么 | 没解决什么 |
|------|-----------|-----------|
| 线程 + 锁 | 共享数据保护 | 死锁、复杂性 |
| 协程 | 高并发 I/O | 共享状态问题 |

**核心问题始终存在**：多个执行流访问共享状态。

```text
传统思路：
  共享状态 + 锁保护 = 安全

  线程 A ───┐
            ├──► 共享变量 ◄── 锁
  线程 B ───┘

  问题：锁的粒度、死锁、性能

Actor 思路：
  不共享 + 消息传递 = 安全

  Actor A ──消息──► Actor B ──消息──► Actor C

  根本没有共享，锁的问题就不存在
```

**这就是 Actor 模型的核心洞察**：与其在共享上加锁，不如一开始就不共享。

## Actor 模型：1973 年的远见

Actor 模型由 Carl Hewitt 在 1973 年提出。当时这只是一个理论模型，直到 Erlang 让它真正落地。

### Actor 的定义

一个 Actor 是一个计算单元，它可以：

```text
Actor = 邮箱 + 状态 + 行为

  ┌────────────────────────────────┐
  │           Actor                │
  │  ┌────────────────────────┐   │
  │  │       邮箱 (Mailbox)    │   │  ← 接收消息
  │  │  [msg1, msg2, msg3]    │   │
  │  └──────────┬─────────────┘   │
  │             │                  │
  │             ▼                  │
  │  ┌────────────────────────┐   │
  │  │     状态 (State)        │   │  ← 私有数据
  │  │   { count: 42 }        │   │
  │  └────────────────────────┘   │
  │             │                  │
  │             ▼                  │
  │  ┌────────────────────────┐   │
  │  │     行为 (Behavior)     │   │  ← 处理消息的逻辑
  │  │  receive msg -> ...    │   │
  │  └────────────────────────┘   │
  └────────────────────────────────┘

Actor 收到消息后可以：
  1. 改变自己的状态
  2. 发送消息给其他 Actor
  3. 创建新的 Actor
```

### 关键规则

```text
Actor 模型的三条铁律：

1. 状态私有
   Actor 的状态只有它自己能访问
   其他 Actor 看不到、摸不着

2. 只能通过消息通信
   想要 Actor B 的数据？发消息问它
   想让 Actor B 做事？发消息告诉它

3. 消息异步处理
   发消息后不等待
   对方什么时候处理、怎么回复，发送方不管
```

### 类比：Actor 就是人

```text
把 Actor 想象成公司里的人：

传统并发（共享内存 + 锁）：
  所有人围着一块白板干活
  每次要写字，先大喊「我要用白板」
  写完再喊「白板用完了」
  效率低，还经常撞车

Actor 模型（消息传递）：
  每个人有自己的办公桌（私有状态）
  沟通只用便签/邮件（消息）
  收到便签，按顺序处理
  不用抢白板，不会撞车
```

## Erlang：让 Actor 成真

Erlang 是 Actor 模型最成功的实现。它诞生于爱立信实验室，目标是构建电信系统——那是对可靠性要求最高的领域。

### Erlang 的世界观

```erlang
%% 创建一个 Actor（在 Erlang 里叫 process）
Pid = spawn(fun() ->
    loop(initial_state)
end).

%% Actor 的主循环
loop(State) ->
    receive
        {get, From} ->
            From ! {ok, State},
            loop(State);
        {set, NewState} ->
            loop(NewState);
        stop ->
            ok  % 退出循环，进程结束
    end.

%% 发消息
Pid ! {set, new_value}.
Pid ! {get, self()}.
```

**几个关键点**：

1. **进程极轻量**：Erlang 进程不是 OS 进程，是虚拟机管理的 Actor，创建成本约 300 字节
2. **海量并发**：一台机器可以跑几百万个进程
3. **进程隔离**：一个进程崩溃不影响其他进程

### "Let It Crash"：拥抱失败

Erlang 最反直觉的设计哲学：**让它崩溃**。

```text
传统思维：
  - 预防所有可能的错误
  - 写大量防御代码
  - try-catch 包裹一切
  - 目标：程序永远不崩溃

Erlang 思维：
  - 错误会发生，接受它
  - 进程崩了就崩了
  - 让别人（监督者）来处理善后
  - 目标：系统整体可靠，允许局部失败
```

**为什么这样做？**

```text
现实世界的错误：

1. 可预见的错误（invalid input, network timeout）
   → 正常处理

2. 不可预见的错误（hardware failure, cosmic ray）
   → 没法防御，只能接受

传统做法：假装 2 不存在，在 1 上做文章
Erlang 做法：承认 2 存在，设计系统容忍它
```

### 监督树：从崩溃中恢复

Erlang 用监督树（Supervision Tree）管理进程生命周期：

```text
监督树结构：
              ┌───────────┐
              │ Application│
              │ Supervisor │
              └─────┬─────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
   ┌────────┐  ┌────────┐  ┌────────┐
   │ Worker │  │ Worker │  │  Sub   │
   │   A    │  │   B    │  │Supervisor
   └────────┘  └────────┘  └───┬────┘
                               │
                        ┌──────┴──────┐
                        │             │
                        ▼             ▼
                   ┌────────┐   ┌────────┐
                   │ Worker │   │ Worker │
                   │   C    │   │   D    │
                   └────────┘   └────────┘

监督策略：
  one_for_one:  C 崩了只重启 C
  one_for_all:  C 崩了重启 C 和 D
  rest_for_one: C 崩了重启 C 和它之后的 D
```

**真实场景**：

```text
Web 服务器处理请求：

          ┌──────────────┐
          │  Supervisor  │
          └──────┬───────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
    ▼            ▼            ▼
┌───────┐   ┌───────┐   ┌───────┐
│ Conn1 │   │ Conn2 │   │ Conn3 │  ← 每个连接一个 Worker
└───────┘   └───────┘   └───────┘

Conn2 处理请求时崩溃了：
  1. Supervisor 收到通知
  2. 按策略重启 Conn2
  3. Conn1, Conn3 完全不受影响
  4. 用户可能只看到一次请求失败

传统服务器：一个线程崩溃，可能整个进程崩溃
Erlang：一个进程崩溃，其他进程无感知
```

### 传说中的九个九

Erlang 最著名的案例是爱立信的 AXD301 交换机。常被引用的数据：

```text
AXD301 的可靠性：99.9999999%（九个九）

这个数字怎么理解：
  - 如果按字面意思，每年宕机 31 毫秒
  - 但这个数字的具体含义有争议
  - 有人认为是"计划内维护不计入"的统计
  - 也有人质疑统计方法

更有意义的是 Erlang 实现高可用的方式：
  - 200 万行 Erlang 代码
  - 运行时热更新（不停机升级）
  - 故障自动恢复（监督树）
  - 节点间自动故障转移
```

**无论具体数字如何，AXD301 确实证明了 Erlang/Actor 模型在电信级系统中的可靠性。**

## Elixir：现代化的 Erlang

Elixir 是运行在 Erlang VM 上的现代语言，继承了 Erlang 的 Actor 模型，但语法更友好。

### 基本用法

```elixir
# 定义一个 GenServer（通用 Actor）
defmodule Counter do
  use GenServer

  # 客户端 API
  def start_link(initial) do
    GenServer.start_link(__MODULE__, initial, name: __MODULE__)
  end

  def increment do
    GenServer.cast(__MODULE__, :increment)
  end

  def get do
    GenServer.call(__MODULE__, :get)
  end

  # 服务器回调
  @impl true
  def init(initial), do: {:ok, initial}

  @impl true
  def handle_cast(:increment, state) do
    {:noreply, state + 1}
  end

  @impl true
  def handle_call(:get, _from, state) do
    {:reply, state, state}
  end
end

# 使用
{:ok, _pid} = Counter.start_link(0)
Counter.increment()
Counter.increment()
Counter.get()  # => 2
```

### Phoenix：Actor 驱动的 Web 框架

Phoenix 是 Elixir 的 Web 框架，用 Actor 模型处理实时通信：

```text
Phoenix Channels 架构：

  浏览器 ─── WebSocket ───► Channel Process
                              │
                              ▼
                         ┌─────────┐
                         │  Topic  │ "room:lobby"
                         │ PubSub  │
                         └────┬────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
         ┌────────┐     ┌────────┐     ┌────────┐
         │ User A │     │ User B │     │ User C │
         └────────┘     └────────┘     └────────┘

每个 WebSocket 连接 = 一个 Channel Process
消息通过 PubSub 广播
天然支持百万连接
```

## Akka：JVM 上的 Actor

Java/Scala 世界用 Akka 实现 Actor 模型：

```scala
// 定义 Actor
class Counter extends Actor {
  private var count = 0

  def receive: Receive = {
    case Increment =>
      count += 1
    case Get =>
      sender() ! count
  }
}

// 创建和使用
val system = ActorSystem("MySystem")
val counter = system.actorOf(Props[Counter], "counter")

counter ! Increment
counter ! Increment
counter ! Get  // 会收到 2
```

### Akka 的特色

```text
1. 位置透明
   ActorRef 可以指向本地或远程 Actor
   发消息的代码不用改

   localActor ! message    // 本地
   remoteActor ! message   // 远程（可能在另一台机器）
   // 代码完全一样

2. Actor 层次
   和 Erlang 类似的监督树
   父 Actor 监督子 Actor

3. Akka Cluster
   Actor 可以跨节点分布
   自动处理节点加入/离开
```

## Actor 模型的优缺点

### 优点

| 优点 | 说明 |
|------|------|
| 无锁并发 | 没有共享状态，不需要锁 |
| 故障隔离 | 一个 Actor 崩溃不影响其他 |
| 可扩展 | 从单机到集群，编程模型不变 |
| 易于理解 | 消息传递比锁更直观 |
| 热更新 | 可以在运行时替换 Actor 代码 |

### 缺点

| 缺点 | 说明 |
|------|------|
| 消息开销 | 每次通信都要创建消息对象 |
| 调试困难 | 异步消息难以追踪 |
| 邮箱溢出 | 消息积压可能耗尽内存 |
| 死锁仍可能 | A 等 B 回复，B 等 A 回复 |
| 思维转换 | 需要从「调用」转换到「消息」 |

### Actor 死锁

是的，Actor 模型也能死锁：

```text
Actor 死锁场景：

  Actor A                    Actor B
     │                          │
     │── 请求消息 ──►           │
     │   (等待 B 回复)          │
     │                          │
     │           ◄── 请求消息 ──│
     │              (等待 A 回复)│
     │                          │
     ▼                          ▼
  等 B 回复...              等 A 回复...

两边都在等，都不能继续
```

**解决方案**：
- 避免同步请求（用异步消息 + 回调）
- 设置超时
- 单向消息流设计

## Actor vs 锁：什么时候用什么

```text
选择 Actor：
  ✓ 高并发，大量独立任务
  ✓ 需要故障隔离
  ✓ 分布式系统
  ✓ 实时通信（聊天、游戏）
  ✓ IoT、设备管理

选择传统锁：
  ✓ 性能极致敏感
  ✓ 简单共享状态
  ✓ 单机、少量线程
  ✓ 已有代码库（改造成本高）
```

### 性能对比

```text
发一条消息 vs 获取一把锁（粗略估计，仅供参考）：

  消息传递（取决于实现）：
    - 创建消息对象（可能涉及堆分配）
    - 放入邮箱（队列操作）
    - 调度接收方处理
    代价：数百到数千 ns（Erlang、Akka 等各不相同）

  获取无竞争的锁：
    - 一个原子操作（CAS）
    代价：~25 ns

  单次操作开销：消息传递通常更高

但这只是单次操作的成本，实际场景复杂得多：
  - 锁有竞争时代价可能飙升到微秒级
  - 消息传递不会阻塞发送方
  - Actor 更容易扩展到多机
  - 消息传递避免了锁带来的心智负担

结论：不要只看单次操作成本，要看整体架构收益。
```

## 实战：用 Actor 思维设计系统

### 示例：在线游戏服务器

```text
传统设计（共享状态）：
  所有玩家数据在一个大 HashMap
  每次操作都要加锁

  ┌────────────────────────┐
  │   Global Game State    │
  │  {player1: data1,      │ ◄── 所有线程竞争这把锁
  │   player2: data2, ...} │
  └────────────────────────┘

Actor 设计：
  每个玩家一个 Actor
  房间是 Actor
  匹配系统是 Actor

  ┌─────────────┐
  │ Matchmaker  │ ◄── 匹配 Actor
  └──────┬──────┘
         │
  ┌──────┴──────┐
  ▼             ▼
┌────────┐  ┌────────┐
│ Room 1 │  │ Room 2 │ ◄── 房间 Actor
└───┬────┘  └───┬────┘
    │           │
┌───┴───┐   ┌───┴───┐
▼   ▼   ▼   ▼   ▼   ▼
P1  P2  P3  P4  P5  P6  ◄── 玩家 Actor
```

**Actor 设计的好处**：
- 玩家 A 的操作不会阻塞玩家 B
- 一个房间崩溃不影响其他房间
- 可以把房间 Actor 分布到不同服务器

### 示例：银行转账

```text
传统做法（两把锁）：
  lock(accountA);
  lock(accountB);  // 可能死锁
  transfer();
  unlock(accountB);
  unlock(accountA);

Actor 做法（消息协调）：

  TransferActor ──► AccountA: deduct(100)
                              │
                              ▼
                    AccountA ──► TransferActor: ok
                                                │
                                                ▼
                    TransferActor ──► AccountB: add(100)
                                                      │
                                                      ▼
                                        AccountB ──► TransferActor: ok

  或者用 Saga 模式：
    1. 冻结 A 的 100
    2. 转入 B
    3. 如果失败，解冻 A
```

## 总结

**Actor 模型的核心思想**：

```text
1. 隔离
   每个 Actor 是独立的计算单元
   状态私有，不共享

2. 通信
   只能通过消息传递
   异步、非阻塞

3. 容错
   故障隔离 + 监督树
   让崩溃成为可恢复的事件
```

**Actor 模型的定位**：

| 对比 | 线程 + 锁 | Actor |
|------|----------|-------|
| 共享状态 | 允许，用锁保护 | 禁止，用消息传递 |
| 同步方式 | 锁、条件变量 | 消息发送/接收 |
| 错误处理 | try-catch | let it crash + 监督 |
| 扩展性 | 单机 | 单机到集群 |
| 心智模型 | 共享内存 | 独立实体通信 |

**记住**：Actor 不是银弹。它解决了共享状态问题，但引入了消息传递的开销和复杂性。

下一篇，我们看另一种消息传递模型——CSP（Communicating Sequential Processes）。它和 Actor 有什么不同？Go 语言为什么选择 CSP 而不是 Actor？

---

*上一篇：[协程：用户态的轻量级线程](/posts/concurrency-models-3-coroutines/)*

*下一篇：[CSP 模型：Channel 是一等公民](/posts/concurrency-models-5-csp/)*

*本系列：*
1. *[为什么并发这么难？](/posts/concurrency-models-1-why-hard/)*
2. *[线程与锁：最直接也最危险](/posts/concurrency-models-2-threads-locks/)*
3. *[协程：用户态的轻量级线程](/posts/concurrency-models-3-coroutines/)*
4. *Actor 模型：不要共享，要传递（本篇）*
5. *[CSP 模型：Channel 是一等公民](/posts/concurrency-models-5-csp/)*
6. *[Rust 的第三条路：编译期消灭数据竞争](/posts/concurrency-models-6-rust/)*
7. *[实战选型：没有银弹，只有场景](/posts/concurrency-models-7-practice/)*
