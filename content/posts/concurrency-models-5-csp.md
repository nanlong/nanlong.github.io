---
title: "并发模型演进史（五）：CSP 模型——Channel 是一等公民"
date: 2025-12-08T22:40:00+08:00
draft: false
tags: ["并发", "CSP", "Channel", "Go", "goroutine", "消息传递"]
categories: ["技术"]
description: "Actor 说：我是实体，给我发消息。CSP 说：不，Channel 才是主角，进程只是 Channel 两端的工人。Go 语言选择了 CSP，并且用一句话总结了它的哲学：不要通过共享内存来通信，而要通过通信来共享内存。"
series: ["并发模型演进史"]
---

Actor 说：我是实体，给我发消息。CSP 说：不，Channel 才是主角，进程只是 Channel 两端的工人。Go 语言选择了 CSP，并且用一句话总结了它的哲学：**不要通过共享内存来通信，而要通过通信来共享内存**。

<!--more-->

## CSP：与 Actor 同时代的理论

CSP（Communicating Sequential Processes）由 Tony Hoare 在 1978 年提出，比 Actor 模型（1973 年）晚 5 年。

虽然 Actor 先出现，但两者都诞生于 1970 年代，各自走了不同的路。

### CSP vs Actor：核心差异

```text
Actor 模型：
  以 Actor（实体）为中心
  消息发给 Actor
  Actor 有身份、有邮箱

  ┌────────┐   消息    ┌────────┐
  │ Actor A│ ────────► │ Actor B│
  └────────┘           └────────┘
  发给 B 这个实体

CSP 模型：
  以 Channel（管道）为中心
  消息通过 Channel 传递
  进程是匿名的

  ┌────────┐           ┌────────┐
  │Process │ ──► ch ──►│Process │
  └────────┘           └────────┘
  发到 Channel，谁收谁收
```

**一个类比**：

```text
Actor = 给人发邮件
  - 你知道对方是谁
  - 发给 bob@company.com
  - Bob 的邮箱是他的身份的一部分

CSP = 往传送带上放东西
  - 你不知道谁会收
  - 只知道放到「零件传送带」上
  - 传送带另一端是谁不重要
```

### 设计哲学对比

| 维度 | Actor | CSP |
|------|-------|-----|
| 核心概念 | Actor（实体） | Channel（管道） |
| 消息目标 | 发给特定 Actor | 发到 Channel |
| 身份 | Actor 有身份 | 进程匿名 |
| 邮箱 | 每个 Actor 一个 | 无（Channel 是共享的） |
| 同步 | 异步（发了就走） | 可同步（阻塞等待） |
| 代表语言 | Erlang, Elixir | Go, Clojure |

## Go 的 CSP 实现

Go 是 CSP 模型最成功的实践者。它的口号：

> **Don't communicate by sharing memory; share memory by communicating.**
>
> 不要通过共享内存来通信；而要通过通信来共享内存。

### Channel 基础

```go
// 创建 Channel
ch := make(chan int)     // 无缓冲 Channel
ch := make(chan int, 10) // 带缓冲 Channel

// 发送
ch <- 42

// 接收
value := <-ch

// 关闭
close(ch)
```

**无缓冲 vs 带缓冲**：

```text
无缓冲 Channel：
  发送方：ch <- value
  接收方：value := <-ch

  发送和接收必须同时准备好
  否则会阻塞

  类比：面对面交接
  你伸手递东西，对方必须伸手接
  一方没准备好，另一方就得等着

带缓冲 Channel：
  ch := make(chan int, 3)

  ┌─────────────────────┐
  │ [  ] [  ] [  ]      │  容量 3
  └─────────────────────┘

  发送方可以发 3 个，不用等接收
  满了才阻塞

  类比：往传送带上放东西
  传送带没满就能放
  满了得等工人取走
```

### Channel 是一等公民

在 Go 里，Channel 是值，可以传递：

```go
// Channel 作为参数
func worker(jobs <-chan int, results chan<- int) {
    for job := range jobs {
        results <- job * 2
    }
}

// Channel 作为返回值
func generator() <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; ; i++ {
            ch <- i
        }
    }()
    return ch
}

// Channel 的 Channel
func fanOut(in <-chan int, n int) []<-chan int {
    outs := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        ch := make(chan int)
        outs[i] = ch
        go func(c chan int) {  // 注意：通过参数传递避免闭包陷阱
            for v := range in {
                c <- v
            }
        }(ch)
    }
    return outs
}
```

### Select：多路复用

`select` 是 Go CSP 的精华，类似于 IO 多路复用的 select/epoll：

```go
select {
case v := <-ch1:
    // ch1 有数据
    fmt.Println("from ch1:", v)
case v := <-ch2:
    // ch2 有数据
    fmt.Println("from ch2:", v)
case ch3 <- value:
    // 成功发送到 ch3
    fmt.Println("sent to ch3")
case <-time.After(time.Second):
    // 超时
    fmt.Println("timeout")
default:
    // 所有 Channel 都没准备好
    fmt.Println("nothing ready")
}
```

**select 的行为**：
- 如果多个 case 同时就绪，随机选一个
- 如果没有 default，会阻塞直到某个 case 就绪
- 如果有 default，不阻塞

## CSP 模式大全

### 模式一：生产者-消费者

```text
最基本的模式：

  Producer ──► Channel ──► Consumer

  producer:
    for item := range items {
        ch <- item  // 生产
    }
    close(ch)

  consumer:
    for item := range ch {
        process(item)  // 消费
    }
```

```go
func producer(ch chan<- int) {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    close(ch)
}

func consumer(ch <-chan int) {
    for v := range ch {
        fmt.Println(v)
    }
}

func main() {
    ch := make(chan int)
    go producer(ch)
    consumer(ch)
}
```

### 模式二：Fan-Out（一对多）

```text
一个生产者，多个消费者：

                ┌──► Worker 1 ──► result
                │
  Producer ──►──┼──► Worker 2 ──► result
                │
                └──► Worker 3 ──► result

适用场景：
  - 任务分发
  - 负载均衡
  - 并行处理
```

```go
func fanOut(in <-chan int, workers int) []<-chan int {
    outs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        out := make(chan int)
        outs[i] = out
        go func(ch chan int) {  // 通过参数传递，避免闭包问题
            for v := range in {
                ch <- process(v)
            }
            close(ch)
        }(out)
    }
    return outs
}
```

**注意**：在 Go 1.22 之前，如果直接在闭包中引用循环变量 `out`，所有 goroutine 可能共享同一个值（最后一次迭代的值）。通过函数参数传递可以避免这个问题。Go 1.22+ 已修复此问题，但为了兼容性，建议显式传递。

### 模式三：Fan-In（多对一）

```text
多个生产者，一个消费者：

  Source 1 ──►──┐
                │
  Source 2 ──►──┼──► Collector
                │
  Source 3 ──►──┘

适用场景：
  - 结果收集
  - 日志聚合
  - 多数据源合并
```

```go
func fanIn(ins ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    for _, in := range ins {
        wg.Add(1)
        go func(ch <-chan int) {
            defer wg.Done()
            for v := range ch {
                out <- v
            }
        }(in)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

### 模式四：Pipeline（流水线）

```text
数据流过多个处理阶段：

  Source ──► Stage1 ──► Stage2 ──► Stage3 ──► Sink

  每个阶段：
    - 从输入 Channel 读
    - 处理
    - 写到输出 Channel

适用场景：
  - 数据处理管道
  - ETL 流程
  - 图像处理
```

```go
// 生成数字
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// 平方
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// 过滤偶数
func filterEven(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            if n%2 == 0 {
                out <- n
            }
        }
        close(out)
    }()
    return out
}

func main() {
    // 组装流水线
    // 1,2,3,4,5 → 平方 → 过滤偶数 → 输出
    for v := range filterEven(square(gen(1, 2, 3, 4, 5))) {
        fmt.Println(v) // 4, 16
    }
}
```

### 模式五：Worker Pool

```text
固定数量的 Worker 处理任务：

  ┌────────────────────────────────┐
  │           Task Queue           │
  │  [task1, task2, task3, ...]    │
  └───────────────┬────────────────┘
                  │
        ┌─────────┼─────────┐
        │         │         │
        ▼         ▼         ▼
   ┌────────┐ ┌────────┐ ┌────────┐
   │Worker 1│ │Worker 2│ │Worker 3│
   └───┬────┘ └───┬────┘ └───┬────┘
       │         │         │
       └─────────┼─────────┘
                 ▼
        ┌────────────────┐
        │  Result Queue  │
        └────────────────┘

适用场景：
  - 控制并发数
  - 资源池
  - 限流
```

```go
func workerPool(numWorkers int, jobs <-chan int, results chan<- int) {
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for job := range jobs {
                fmt.Printf("Worker %d processing job %d\n", id, job)
                results <- job * 2
            }
        }(i)
    }

    wg.Wait()
    close(results)
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    // 启动 3 个 Worker
    go workerPool(3, jobs, results)

    // 发送 10 个任务
    for i := 0; i < 10; i++ {
        jobs <- i
    }
    close(jobs)

    // 收集结果
    for r := range results {
        fmt.Println(r)
    }
}
```

### 模式六：Timeout 和 Cancellation

```go
// 超时控制
func doWithTimeout(timeout time.Duration) (int, error) {
    result := make(chan int, 1)

    go func() {
        // 模拟耗时操作
        time.Sleep(2 * time.Second)
        result <- 42
    }()

    select {
    case r := <-result:
        return r, nil
    case <-time.After(timeout):
        return 0, errors.New("timeout")
    }
}

// 取消控制（使用 context）
func doWithCancel(ctx context.Context) (int, error) {
    result := make(chan int, 1)

    go func() {
        // 模拟耗时操作
        time.Sleep(2 * time.Second)
        select {
        case result <- 42:
        case <-ctx.Done():
        }
    }()

    select {
    case r := <-result:
        return r, nil
    case <-ctx.Done():
        return 0, ctx.Err()
    }
}
```

## CSP 的陷阱

### 陷阱一：Goroutine 泄漏

```go
// 错误示例：goroutine 泄漏
func bad() <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; ; i++ {
            ch <- i  // 如果没人接收，永远阻塞
        }
    }()
    return ch
}

func main() {
    ch := bad()
    fmt.Println(<-ch) // 只读一个
    // goroutine 永远阻塞在那里，泄漏了
}

// 正确做法：使用 context 控制
func good(ctx context.Context) <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for i := 0; ; i++ {
            select {
            case ch <- i:
            case <-ctx.Done():
                return  // 收到取消信号，退出
            }
        }
    }()
    return ch
}
```

### 陷阱二：Channel 关闭问题

```go
// 错误：向已关闭的 Channel 发送
ch := make(chan int)
close(ch)
ch <- 1  // panic: send on closed channel

// 错误：重复关闭
close(ch)
close(ch)  // panic: close of closed channel

// 正确做法：只让发送方关闭
func producer(ch chan<- int) {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    close(ch)  // 发送方关闭
}

func consumer(ch <-chan int) {
    for v := range ch {  // 安全地消费
        fmt.Println(v)
    }
}
```

### 陷阱三：死锁

```go
// 死锁示例 1：无缓冲 Channel 在同一个 goroutine 发送和接收
func main() {
    ch := make(chan int)
    ch <- 1     // 阻塞，等待接收
    <-ch        // 永远执行不到
}

// 死锁示例 2：循环依赖
func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    go func() {
        <-ch1   // 等 ch1
        ch2 <- 1
    }()

    go func() {
        <-ch2   // 等 ch2
        ch1 <- 1
    }()

    // 两个 goroutine 互相等待，死锁
    select {}
}
```

## CSP vs Actor：选择指南

```text
选择 CSP（Go）：
  ✓ 数据流处理
  ✓ 管道模式
  ✓ 需要同步点（无缓冲 Channel）
  ✓ 结构化并发
  ✓ 已经在用 Go

选择 Actor（Erlang/Elixir）：
  ✓ 大量独立实体（用户、设备、连接）
  ✓ 需要故障隔离
  ✓ 分布式系统
  ✓ 热更新需求
  ✓ 电信级可靠性

两者都可以：
  - Web 服务
  - 消息队列消费
  - 任务调度
```

### 实际对比

```text
场景：聊天服务器，管理 10 万用户

Actor 思路：
  每个用户一个 Actor
  发消息 = 给用户 Actor 发消息
  用户状态在 Actor 内部

  优点：用户级别隔离，一个用户崩了不影响别人
  缺点：Actor 数量多，管理开销

CSP 思路：
  每个用户一个 goroutine + Channel
  发消息 = 往用户的 Channel 发
  用户状态可以在 goroutine 内部

  优点：更轻量，百万 goroutine 没压力
  缺点：没有内置的监督机制

实际上：两种方式都能实现，选你熟悉的
```

## 代码对比

同一个功能，用两种模型实现：

### Actor 风格（Elixir）

```elixir
defmodule Counter do
  use GenServer

  def start_link(initial) do
    GenServer.start_link(__MODULE__, initial)
  end

  def increment(pid) do
    GenServer.cast(pid, :increment)
  end

  def get(pid) do
    GenServer.call(pid, :get)
  end

  # 回调
  def init(initial), do: {:ok, initial}

  def handle_cast(:increment, state) do
    {:noreply, state + 1}
  end

  def handle_call(:get, _from, state) do
    {:reply, state, state}
  end
end

# 使用
{:ok, counter} = Counter.start_link(0)
Counter.increment(counter)
Counter.get(counter)  # => 1
```

### CSP 风格（Go）

```go
type Counter struct {
    incr chan struct{}
    get  chan chan int
}

func NewCounter() *Counter {
    c := &Counter{
        incr: make(chan struct{}),
        get:  make(chan chan int),
    }
    go c.run()
    return c
}

func (c *Counter) run() {
    count := 0
    for {
        select {
        case <-c.incr:
            count++
        case reply := <-c.get:
            reply <- count
        }
    }
}

func (c *Counter) Increment() {
    c.incr <- struct{}{}
}

func (c *Counter) Get() int {
    reply := make(chan int)
    c.get <- reply
    return <-reply
}

// 使用
counter := NewCounter()
counter.Increment()
counter.Get()  // => 1
```

**注意差异**：
- Actor：消息发给 Actor（`pid`）
- CSP：消息发给 Channel（`c.incr`, `c.get`）

## 总结

**CSP 的核心思想**：

```text
1. Channel 是一等公民
   不是 Actor 的邮箱
   是独立的、可传递的通信原语

2. 进程是匿名的
   不关心谁在另一端
   只关心数据流向

3. 同步通信
   无缓冲 Channel 强制同步
   天然的同步点
```

**Go 的 CSP 哲学**：

| 原则 | 说明 |
|------|------|
| 通过通信共享 | 不要共享内存，用 Channel 传递数据 |
| Channel 优先 | 优先用 Channel，而不是锁 |
| goroutine 廉价 | 不要吝啬，该开就开 |
| select 多路复用 | 同时等待多个 Channel |

**记住**：

> CSP 和 Actor 都是消息传递模型。
>
> Actor 说：我是谁很重要，给我发消息。
>
> CSP 说：Channel 是主角，进程只是 Channel 两端的工人。

下一篇，我们看 Rust 的第三条路——它既不用 Actor，也不是纯 CSP，而是用**所有权系统**在编译期就消灭数据竞争。

---

*上一篇：[Actor 模型：不要共享，要传递](/posts/concurrency-models-4-actor/)*

*下一篇：[Rust 的第三条路：编译期消灭数据竞争](/posts/concurrency-models-6-rust/)*

*本系列：*
1. *[为什么并发这么难？](/posts/concurrency-models-1-why-hard/)*
2. *[线程与锁：最直接也最危险](/posts/concurrency-models-2-threads-locks/)*
3. *[协程：用户态的轻量级线程](/posts/concurrency-models-3-coroutines/)*
4. *[Actor 模型：不要共享，要传递](/posts/concurrency-models-4-actor/)*
5. *CSP 模型：Channel 是一等公民（本篇）*
6. *[Rust 的第三条路：编译期消灭数据竞争](/posts/concurrency-models-6-rust/)*
7. *[实战选型：没有银弹，只有场景](/posts/concurrency-models-7-practice/)*
