---
title: "算法不是面试题（六）：堆与优先队列——调度器的核心"
date: 2025-12-08T10:50:00+08:00
draft: false
tags: ["算法", "堆", "优先队列", "延迟任务", "Top-K"]
categories: ["技术"]
description: "定时任务怎么实现？Top-K 怎么高效计算？答案都指向同一个数据结构：堆。它是任务调度器、排行榜、事件驱动系统的核心。"
series: ["算法不是面试题"]
---

定时任务怎么实现？Top-K 怎么高效计算？答案都指向同一个数据结构：堆。它是任务调度器、排行榜、事件驱动系统的核心。

<!--more-->

## 堆是什么？

堆是一种特殊的**完全二叉树**，满足：

```text
最小堆：父节点 <= 所有子节点
最大堆：父节点 >= 所有子节点

最小堆示例：
        1
       / \
      3   2
     / \
    5   4

数组表示：[1, 3, 2, 5, 4]
父节点 i 的子节点：2i+1, 2i+2
子节点 i 的父节点：(i-1)/2
```

**核心操作**：

| 操作 | 复杂度 | 说明 |
|------|--------|------|
| peek | O(1) | 查看堆顶（最小/最大元素） |
| push | O(log n) | 插入元素 |
| pop | O(log n) | 删除堆顶 |
| heapify | O(n) | 从数组构建堆 |

## 场景一：延迟任务

订单 30 分钟未支付自动取消，怎么实现？

### 朴素方案：定时扫描

```rust
// 每分钟扫描一次
async fn scan_expired_orders() {
    loop {
        let now = Utc::now();
        let expired = db
            .query("SELECT * FROM orders WHERE status = 'pending' AND expire_at < ?", now)
            .await;

        for order in expired {
            cancel_order(order.id).await;
        }

        sleep(Duration::from_secs(60)).await;
    }
}
```

**问题**：

```text
1. 精度差：最多延迟 1 分钟执行
2. 数据库压力：每分钟全表扫描
3. 处理延迟：如果订单多，一次扫描可能处理不完
```

### 堆方案：延迟队列

```rust
use std::collections::BinaryHeap;
use std::cmp::{Ordering, Reverse};
use chrono::Utc;
use std::time::Duration;

#[derive(Eq, PartialEq)]
struct DelayedTask {
    execute_at: i64,  // 执行时间戳
    order_id: String,
}

impl Ord for DelayedTask {
    fn cmp(&self, other: &Self) -> Ordering {
        // 最小堆：最早执行的在堆顶
        other.execute_at.cmp(&self.execute_at)
    }
}

impl PartialOrd for DelayedTask {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

struct DelayQueue {
    heap: BinaryHeap<DelayedTask>,
}

impl DelayQueue {
    fn new() -> Self {
        Self {
            heap: BinaryHeap::new(),
        }
    }

    fn push(&mut self, task: DelayedTask) {
        self.heap.push(task);
    }

    fn pop_ready(&mut self) -> Vec<DelayedTask> {
        let now = Utc::now().timestamp();
        let mut ready = Vec::new();

        while let Some(task) = self.heap.peek() {
            if task.execute_at <= now {
                ready.push(self.heap.pop().unwrap());
            } else {
                break;
            }
        }

        ready
    }

    fn next_delay(&self) -> Option<Duration> {
        self.heap.peek().map(|task| {
            let now = Utc::now().timestamp();
            let wait = (task.execute_at - now).max(0) as u64;
            Duration::from_secs(wait)
        })
    }
}
```

### 使用方式

```rust
use std::sync::Arc;
use tokio::sync::Mutex;
use tokio::time::sleep;

async fn delay_queue_worker(queue: Arc<Mutex<DelayQueue>>) {
    loop {
        let delay = {
            let q = queue.lock().await;
            q.next_delay()
        };

        // 等待到下一个任务的执行时间
        if let Some(d) = delay {
            sleep(d).await;
        } else {
            sleep(Duration::from_secs(1)).await;
            continue;
        }

        // 处理到期任务
        let tasks = {
            let mut q = queue.lock().await;
            q.pop_ready()
        };

        for task in tasks {
            cancel_order(&task.order_id).await;
        }
    }
}
```

**优势**：

| 指标 | 定时扫描 | 延迟队列 |
|------|----------|----------|
| 精度 | 分钟级 | 毫秒级 |
| CPU | 持续消耗 | 按需唤醒 |
| 数据库 | 每分钟扫描 | 无查询 |
| 复杂度 | O(n) | O(log n) |

## 场景二：Top-K 问题

找出访问量最高的 100 个 URL，数据量 10 亿。

### 朴素方案：全排序

```rust
fn top_k_naive(urls: &[(String, u64)], k: usize) -> Vec<(String, u64)> {
    let mut sorted = urls.to_vec();
    sorted.sort_by(|a, b| b.1.cmp(&a.1));  // 按访问量降序
    sorted.truncate(k);
    sorted
}
```

**问题**：O(n log n)，10 亿数据要排序几分钟。

### 堆方案：最小堆

```text
思路：
  维护一个大小为 K 的最小堆
  遍历所有元素，如果比堆顶大，替换堆顶
  最后堆里就是 Top-K

为什么用最小堆？
  堆顶是 K 个元素中最小的
  新元素比堆顶大 → 应该进入 Top-K → 替换堆顶
  新元素比堆顶小 → 不够格 → 跳过
```

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

fn top_k_heap(urls: impl Iterator<Item = (String, u64)>, k: usize) -> Vec<(String, u64)> {
    // 最小堆（用 Reverse 包装）
    let mut heap: BinaryHeap<Reverse<(u64, String)>> = BinaryHeap::with_capacity(k + 1);

    for (url, count) in urls {
        if heap.len() < k {
            heap.push(Reverse((count, url)));
        } else if let Some(Reverse((min_count, _))) = heap.peek() {
            if count > *min_count {
                heap.pop();
                heap.push(Reverse((count, url)));
            }
        }
    }

    heap.into_iter()
        .map(|Reverse((count, url))| (url, count))
        .collect()
}
```

**复杂度**：

| 方案 | 时间 | 空间 |
|------|------|------|
| 全排序 | O(n log n) | O(n) |
| 最小堆 | O(n log k) | O(k) |

K=100，n=10 亿：

```text
全排序：10 亿 × log(10 亿) ≈ 300 亿次操作
最小堆：10 亿 × log(100) ≈ 70 亿次操作

快 4 倍，且内存只需要存 100 个元素
```

### 流式 Top-K

数据持续到来，实时更新 Top-K：

```rust
use std::collections::{BinaryHeap, HashMap};
use std::cmp::Reverse;

struct StreamingTopK {
    heap: BinaryHeap<Reverse<(u64, String)>>,
    k: usize,
    // 用于快速查找元素是否在堆中
    in_heap: HashMap<String, u64>,
}

impl StreamingTopK {
    fn new(k: usize) -> Self {
        Self {
            heap: BinaryHeap::with_capacity(k + 1),
            k,
            in_heap: HashMap::new(),
        }
    }

    fn rebuild_heap(&mut self) {
        let items: Vec<_> = self.in_heap.iter()
            .map(|(url, &count)| Reverse((count, url.clone())))
            .collect();
        self.heap = BinaryHeap::from(items);
    }

    fn update(&mut self, url: String, count: u64) {
        // 如果已在堆中，需要更新
        if let Some(&old_count) = self.in_heap.get(&url) {
            if count > old_count {
                // 需要重建堆（或使用支持更新的数据结构）
                self.in_heap.insert(url.clone(), count);
                self.rebuild_heap();
            }
            return;
        }

        // 不在堆中
        if self.heap.len() < self.k {
            self.heap.push(Reverse((count, url.clone())));
            self.in_heap.insert(url, count);
        } else if let Some(Reverse((min_count, _))) = self.heap.peek() {
            if count > *min_count {
                let Reverse((_, old_url)) = self.heap.pop().unwrap();
                self.in_heap.remove(&old_url);
                self.heap.push(Reverse((count, url.clone())));
                self.in_heap.insert(url, count);
            }
        }
    }
}
```

## 场景三：合并有序流

K 个有序的数据流，合并成一个有序流。

```text
场景：
  - 多路归并排序
  - 合并多个数据库的查询结果
  - 日志聚合（多个 Pod 的日志按时间排序）
```

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

struct MergeIterator<I> {
    heap: BinaryHeap<Reverse<(i64, usize)>>,  // (值, 流索引)
    streams: Vec<I>,
}

impl<I: Iterator<Item = i64>> MergeIterator<I> {
    fn new(streams: Vec<I>) -> Self {
        let mut iter = Self {
            heap: BinaryHeap::new(),
            streams,
        };
        // 初始化：从每个流取第一个元素
        for (idx, stream) in iter.streams.iter_mut().enumerate() {
            if let Some(value) = stream.next() {
                iter.heap.push(Reverse((value, idx)));
            }
        }
        iter
    }
}

impl<I: Iterator<Item = i64>> Iterator for MergeIterator<I> {
    type Item = i64;

    fn next(&mut self) -> Option<i64> {
        // 取出最小的
        let Reverse((value, stream_idx)) = self.heap.pop()?;

        // 从该流补充下一个元素
        if let Some(next_value) = self.streams[stream_idx].next() {
            self.heap.push(Reverse((next_value, stream_idx)));
        }

        Some(value)
    }
}
```

**复杂度**：每次取最小值 O(log k)，总共 n 个元素，O(n log k)。

## 场景四：事件驱动调度

Tokio 的定时器就是用堆实现的：

```text
Timer Wheel 简化版：

1. 维护一个最小堆，存储所有定时器
2. 事件循环：
   a. 查看堆顶的到期时间
   b. epoll_wait 等待到那个时间（或有 IO 事件）
   c. 处理到期的定时器
   d. 重复
```

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;
use std::time::{Duration, Instant};

type TaskId = u64;

struct TimerHeap {
    heap: BinaryHeap<Reverse<(Instant, TaskId)>>,
}

impl TimerHeap {
    fn new() -> Self {
        Self {
            heap: BinaryHeap::new(),
        }
    }

    fn schedule(&mut self, task_id: TaskId, deadline: Instant) {
        self.heap.push(Reverse((deadline, task_id)));
    }

    fn next_timeout(&self) -> Option<Duration> {
        self.heap.peek().map(|Reverse((deadline, _))| {
            deadline.saturating_duration_since(Instant::now())
        })
    }

    fn poll_expired(&mut self) -> Vec<TaskId> {
        let now = Instant::now();
        let mut expired = Vec::new();

        while let Some(Reverse((deadline, _))) = self.heap.peek() {
            if *deadline <= now {
                let Reverse((_, task_id)) = self.heap.pop().unwrap();
                expired.push(task_id);
            } else {
                break;
            }
        }

        expired
    }
}
```

## 堆的变体

### 1. 索引堆

支持根据 key 更新优先级：

```rust
use std::collections::HashMap;
use std::hash::Hash;

struct IndexedHeap<K, V> {
    heap: Vec<(K, V)>,
    index: HashMap<K, usize>,  // key → 在堆中的位置
}

impl<K: Hash + Eq + Clone, V: Ord> IndexedHeap<K, V> {
    fn new() -> Self {
        Self {
            heap: Vec::new(),
            index: HashMap::new(),
        }
    }

    fn push(&mut self, key: K, value: V) {
        let pos = self.heap.len();
        self.index.insert(key.clone(), pos);
        self.heap.push((key, value));
        self.sift_up(pos);
    }

    fn update(&mut self, key: K, new_value: V) {
        if let Some(&pos) = self.index.get(&key) {
            let old_value = &self.heap[pos].1;
            if new_value > *old_value {
                self.heap[pos].1 = new_value;
                self.sift_up(pos);
            } else {
                self.heap[pos].1 = new_value;
                self.sift_down(pos);
            }
        } else {
            self.push(key, new_value);
        }
    }

    fn sift_up(&mut self, mut pos: usize) {
        while pos > 0 {
            let parent = (pos - 1) / 2;
            if self.heap[pos].1 > self.heap[parent].1 {
                self.index.insert(self.heap[parent].0.clone(), pos);
                self.index.insert(self.heap[pos].0.clone(), parent);
                self.heap.swap(pos, parent);
                pos = parent;
            } else {
                break;
            }
        }
    }

    fn sift_down(&mut self, mut pos: usize) {
        let len = self.heap.len();
        loop {
            let left = 2 * pos + 1;
            let right = 2 * pos + 2;
            let mut largest = pos;

            if left < len && self.heap[left].1 > self.heap[largest].1 {
                largest = left;
            }
            if right < len && self.heap[right].1 > self.heap[largest].1 {
                largest = right;
            }

            if largest != pos {
                self.index.insert(self.heap[largest].0.clone(), pos);
                self.index.insert(self.heap[pos].0.clone(), largest);
                self.heap.swap(pos, largest);
                pos = largest;
            } else {
                break;
            }
        }
    }
}
```

**适用场景**：Dijkstra 最短路径、动态优先级调度。

### 2. 双端堆

同时支持取最大和最小：

```rust
// 可以用两个堆模拟
struct MinMaxHeap<T> {
    min_heap: BinaryHeap<Reverse<T>>,
    max_heap: BinaryHeap<T>,
    // 需要处理两个堆的同步...
}
```

**实际应用**：滑动窗口中位数。

### 3. 斐波那契堆

```text
复杂度对比：
           | 二叉堆    | 斐波那契堆
push       | O(log n)  | O(1) 均摊
peek       | O(1)      | O(1)
pop        | O(log n)  | O(log n) 均摊
decrease   | O(log n)  | O(1) 均摊

理论上更优，但实际中常数因子大，很少使用
```

## 实战：优先级任务队列

```rust
use std::collections::BinaryHeap;
use std::cmp::Ordering;
use chrono::Utc;

#[derive(Eq, PartialEq)]
struct Task {
    priority: u8,      // 0 = 最高优先级
    created_at: i64,   // 创建时间
    id: String,
    payload: String,
}

impl Ord for Task {
    fn cmp(&self, other: &Self) -> Ordering {
        // 先比较优先级（小的优先）
        // 优先级相同，比较创建时间（早的优先）
        match self.priority.cmp(&other.priority) {
            Ordering::Equal => other.created_at.cmp(&self.created_at),
            ord => ord.reverse(),  // BinaryHeap 是最大堆，所以反转
        }
    }
}

impl PartialOrd for Task {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

struct PriorityTaskQueue {
    heap: BinaryHeap<Task>,
}

impl PriorityTaskQueue {
    fn new() -> Self {
        Self {
            heap: BinaryHeap::new(),
        }
    }

    fn enqueue(&mut self, priority: u8, payload: String) -> String {
        let id = uuid::Uuid::new_v4().to_string();
        let task = Task {
            priority,
            created_at: Utc::now().timestamp(),
            id: id.clone(),
            payload,
        };
        self.heap.push(task);
        id
    }

    fn dequeue(&mut self) -> Option<Task> {
        self.heap.pop()
    }
}
```

## 常见陷阱

### 1. BinaryHeap 是最大堆

```rust
// Rust 的 BinaryHeap 默认是最大堆
let mut heap = BinaryHeap::new();
heap.push(3);
heap.push(1);
heap.push(2);
assert_eq!(heap.pop(), Some(3));  // 最大的先出

// 要最小堆，用 Reverse 包装
use std::cmp::Reverse;
let mut min_heap = BinaryHeap::new();
min_heap.push(Reverse(3));
min_heap.push(Reverse(1));
assert_eq!(min_heap.pop(), Some(Reverse(1)));  // 最小的先出
```

### 2. 自定义类型要实现 Ord

```rust
#[derive(Eq, PartialEq)]
struct Item {
    value: i32,
}

// 错误：没有实现 Ord，不能放入 BinaryHeap
// let mut heap: BinaryHeap<Item> = BinaryHeap::new();

// 正确：实现 Ord
impl Ord for Item {
    fn cmp(&self, other: &Self) -> Ordering {
        self.value.cmp(&other.value)
    }
}

impl PartialOrd for Item {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}
```

### 3. 堆不支持随机删除

```rust
// 标准 BinaryHeap 不支持删除任意元素
// 需要懒删除：标记为删除，pop 时跳过

use std::collections::{BinaryHeap, HashSet};
use std::hash::Hash;

struct LazyHeap<T: Eq + Hash> {
    heap: BinaryHeap<T>,
    deleted: HashSet<T>,
}

impl<T: Ord + Eq + Hash + Clone> LazyHeap<T> {
    fn new() -> Self {
        Self {
            heap: BinaryHeap::new(),
            deleted: HashSet::new(),
        }
    }

    fn push(&mut self, item: T) {
        self.heap.push(item);
    }

    fn remove(&mut self, item: &T) {
        self.deleted.insert(item.clone());
    }

    fn pop(&mut self) -> Option<T> {
        while let Some(item) = self.heap.pop() {
            if !self.deleted.remove(&item) {
                return Some(item);
            }
        }
        None
    }
}
```

## 总结

| 场景 | 堆的作用 | 复杂度 |
|------|----------|--------|
| 延迟任务 | 按执行时间排序 | O(log n) 插入/删除 |
| Top-K | 维护最小的 K 个 | O(n log k) |
| 多路归并 | 取 K 路中最小 | O(n log k) |
| 优先级队列 | 按优先级出队 | O(log n) |
| 定时器 | 找最近到期的 | O(1) 查看，O(log n) 删除 |

**核心认知**：

> 堆的本质是"高效找极值"。任何需要反复获取最大/最小元素的场景，都应该想到堆。它不能做排序（那是排序算法的事），但能高效维护动态数据的极值。

---

*上一篇：[一致性哈希：分布式系统的路由表](/posts/algo-not-leetcode-5-consistent-hash/)*

*下一篇：[限流算法：保护系统的三道防线](/posts/algo-not-leetcode-7-rate-limit/)*

*本系列：*
1. *[算法是业务武器](/posts/algo-not-leetcode-1-intro/)*
2. *[排序与二分：被低估的基础功](/posts/algo-not-leetcode-2-sort-binary-search/)*
3. *[哈希表：缓存设计的基石](/posts/algo-not-leetcode-3-hashtable/)*
4. *[布隆过滤器：用 1% 误判换 90% 内存](/posts/algo-not-leetcode-4-bloom-filter/)*
5. *[一致性哈希：分布式系统的路由表](/posts/algo-not-leetcode-5-consistent-hash/)*
6. *堆与优先队列：调度器的核心（本篇）*
7. *[限流算法：保护系统的三道防线](/posts/algo-not-leetcode-7-rate-limit/)*
8. *[跳表：Redis 排行榜的秘密](/posts/algo-not-leetcode-8-skiplist/)*
9. *[树结构：层级数据的优雅解法](/posts/algo-not-leetcode-9-tree/)*
10. *[字符串匹配：敏感词过滤的正确姿势](/posts/algo-not-leetcode-10-string-match/)*
11. *[实战选型：没有银弹，只有场景](/posts/algo-not-leetcode-11-selection/)*
