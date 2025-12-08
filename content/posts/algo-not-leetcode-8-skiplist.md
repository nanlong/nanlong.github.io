---
title: "算法不是面试题（八）：跳表——Redis 排行榜的秘密"
date: 2025-12-08T11:10:00+08:00
draft: false
tags: ["算法", "跳表", "Redis", "排行榜", "有序集合"]
categories: ["技术"]
description: "游戏排行榜、积分榜、延迟队列——这些功能背后，Redis 用的不是红黑树，而是跳表。为什么？因为跳表更简单、更实用。"
series: ["算法不是面试题"]
---

游戏排行榜、积分榜、延迟队列——这些功能背后，Redis 用的不是红黑树，而是跳表。为什么？因为跳表更简单、更实用。

<!--more-->

## 为什么不用红黑树？

有序数据结构的选择：

| 数据结构 | 查找 | 插入 | 删除 | 范围查询 |
|----------|------|------|------|----------|
| 数组 | O(log n) | O(n) | O(n) | O(k) |
| 链表 | O(n) | O(1) | O(1) | O(n) |
| 红黑树 | O(log n) | O(log n) | O(log n) | O(k + log n) |
| 跳表 | O(log n) | O(log n) | O(log n) | O(k + log n) |

红黑树和跳表复杂度一样，为什么 Redis 选跳表？

```text
Redis 作者 antirez 的解释：

1. 实现简单
   红黑树：插入删除要旋转，几百行代码
   跳表：不需要旋转，几十行代码

2. 范围查询友好
   红黑树：需要中序遍历
   跳表：底层就是链表，直接向后遍历

3. 内存效率
   红黑树：每个节点 3 个指针（左右父）
   跳表：平均 1.33 个指针（概率决定）

4. 并发友好
   红黑树：旋转涉及多个节点
   跳表：局部修改，更容易实现无锁
```

## 跳表的原理

### 从链表优化说起

链表查找是 O(n)，怎么加速？

```text
普通链表：
  1 → 3 → 5 → 7 → 9 → 11 → 13 → 15

找 13，需要遍历 7 个节点
```

**核心思想**：加索引！

```text
Level 2:  1 ─────────────────→ 9 ─────────────→ NULL
          ↓                    ↓
Level 1:  1 ────→ 5 ────→ 9 ────→ 13 ────→ NULL
          ↓      ↓      ↓      ↓
Level 0:  1 → 3 → 5 → 7 → 9 → 11 → 13 → 15 → NULL
```

查找 13：

```text
1. 从最高层（Level 2）开始
2. 1 的下一个是 9，9 < 13，移动到 9
3. 9 的下一个是 NULL，下降到 Level 1
4. 9 的下一个是 13，找到！

只访问了 3 个节点（1, 9, 13）
```

### 数学分析

```text
如果每一层的节点数是下一层的 1/2：
  Level 0: n 个节点
  Level 1: n/2 个节点
  Level 2: n/4 个节点
  ...
  Level k: n/2^k 个节点

层数 = log₂(n)
每层最多走 2 步（然后下降）
总步数 ≈ 2 × log₂(n) = O(log n)
```

### 如何决定节点高度？

**不用精确控制，用概率！**

```rust
fn random_level() -> usize {
    let mut level = 1;
    // 每次有 50% 概率升一层
    while rand::random::<f64>() < 0.5 && level < MAX_LEVEL {
        level += 1;
    }
    level
}
```

**期望**：

- 50% 的节点高度为 1
- 25% 的节点高度为 2
- 12.5% 的节点高度为 3
- ...

平均每个节点 2 个指针（1 + 1/2 + 1/4 + ... = 2）。

## Rust 实现

### 基础结构

```rust
use std::cmp::Ordering;
use std::ptr::NonNull;

const MAX_LEVEL: usize = 16;
const P: f64 = 0.5;

struct Node<K, V> {
    key: K,
    value: V,
    forward: Vec<Option<NonNull<Node<K, V>>>>,  // 每层的下一个节点
}

pub struct SkipList<K, V> {
    head: Box<Node<K, V>>,
    level: usize,  // 当前最高层
    len: usize,
}

impl<K: Ord + Default, V: Default> SkipList<K, V> {
    pub fn new() -> Self {
        Self {
            head: Box::new(Node {
                key: K::default(),
                value: V::default(),
                forward: vec![None; MAX_LEVEL],
            }),
            level: 1,
            len: 0,
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }
}
```

### 查找

```rust
impl<K: Ord, V> SkipList<K, V> {
    pub fn get(&self, key: &K) -> Option<&V> {
        let mut current = &*self.head;

        // 从最高层向下搜索
        for i in (0..self.level).rev() {
            while let Some(next) = current.forward[i] {
                let next_node = unsafe { next.as_ref() };
                match next_node.key.cmp(key) {
                    Ordering::Less => current = next_node,
                    Ordering::Equal => return Some(&next_node.value),
                    Ordering::Greater => break,
                }
            }
        }

        None
    }
}
```

### 插入

```rust
impl<K: Ord, V> SkipList<K, V> {
    pub fn insert(&mut self, key: K, value: V) {
        // 记录每层的前驱节点
        let mut update: Vec<*mut Node<K, V>> = vec![std::ptr::null_mut(); MAX_LEVEL];
        let mut current = &mut *self.head as *mut Node<K, V>;

        // 找到插入位置
        for i in (0..self.level).rev() {
            unsafe {
                while let Some(next) = (*current).forward[i] {
                    if next.as_ref().key < key {
                        current = next.as_ptr();
                    } else {
                        break;
                    }
                }
            }
            update[i] = current;
        }

        // 随机决定新节点高度
        let new_level = self.random_level();

        // 如果新节点更高，更新 update
        if new_level > self.level {
            for i in self.level..new_level {
                update[i] = &mut *self.head;
            }
            self.level = new_level;
        }

        // 创建新节点
        let mut new_node = Box::new(Node {
            key,
            value,
            forward: vec![None; new_level],
        });

        // 更新指针
        let new_node_ptr = Box::into_raw(new_node);
        for i in 0..new_level {
            unsafe {
                (*new_node_ptr).forward[i] = (*update[i]).forward[i];
                (*update[i]).forward[i] = Some(NonNull::new_unchecked(new_node_ptr));
            }
        }

        self.len += 1;
    }

    fn random_level(&self) -> usize {
        let mut level = 1;
        while rand::random::<f64>() < P && level < MAX_LEVEL {
            level += 1;
        }
        level
    }
}
```

### 范围查询

```rust
impl<K: Ord, V> SkipList<K, V> {
    pub fn range(&self, start: &K, end: &K) -> Vec<(&K, &V)> {
        let mut result = Vec::new();

        // 找到 >= start 的第一个节点
        let mut current = &*self.head;
        for i in (0..self.level).rev() {
            while let Some(next) = current.forward[i] {
                let next_node = unsafe { next.as_ref() };
                if next_node.key < *start {
                    current = next_node;
                } else {
                    break;
                }
            }
        }

        // 在底层遍历
        let mut node = current.forward[0];
        while let Some(ptr) = node {
            let n = unsafe { ptr.as_ref() };
            if n.key > *end {
                break;
            }
            result.push((&n.key, &n.value));
            node = n.forward[0];
        }

        result
    }
}
```

## 场景：Redis 排行榜

Redis 的 ZSET（有序集合）底层就是跳表。

### 基本操作

```text
ZADD leaderboard 100 "player1"   # 添加/更新分数
ZADD leaderboard 200 "player2"
ZADD leaderboard 150 "player3"

ZRANK leaderboard "player1"       # 排名（从低到高）→ 0
ZREVRANK leaderboard "player1"    # 排名（从高到低）→ 2

ZRANGE leaderboard 0 9            # 前 10 名
ZRANGEBYSCORE leaderboard 100 200 # 分数在 100-200 的玩家
```

### Rust 封装

```rust
use anyhow::Result;
use redis::{AsyncCommands, Commands};

struct Leaderboard {
    redis: redis::Client,
    key: String,
}

impl Leaderboard {
    fn new(redis: redis::Client, key: impl Into<String>) -> Self {
        Self {
            redis,
            key: key.into(),
        }
    }

    async fn update_score(&self, player: &str, score: f64) -> Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        conn.zadd(&self.key, player, score).await?;
        Ok(())
    }

    async fn get_rank(&self, player: &str) -> Result<Option<usize>> {
        let mut conn = self.redis.get_async_connection().await?;
        // ZREVRANK 返回从高到低的排名
        let rank: Option<usize> = conn.zrevrank(&self.key, player).await?;
        Ok(rank)
    }

    async fn get_top_n(&self, n: usize) -> Result<Vec<(String, f64)>> {
        let mut conn = self.redis.get_async_connection().await?;
        // ZREVRANGE 按分数从高到低
        let result: Vec<(String, f64)> = conn
            .zrevrange_withscores(&self.key, 0, (n - 1) as isize)
            .await?;
        Ok(result)
    }

    async fn get_around_player(&self, player: &str, count: usize) -> Result<Vec<(String, f64)>> {
        let mut conn = self.redis.get_async_connection().await?;

        // 先获取玩家排名
        let rank: Option<usize> = conn.zrevrank(&self.key, player).await?;
        let rank = rank.ok_or_else(|| anyhow::anyhow!("Player not found"))?;

        // 获取周围的玩家
        let start = rank.saturating_sub(count / 2);
        let end = start + count;

        let result: Vec<(String, f64)> = conn
            .zrevrange_withscores(&self.key, start as isize, end as isize)
            .await?;

        Ok(result)
    }
}
```

### 实时排行榜

```rust
impl Leaderboard {
    // 增量更新分数
    async fn incr_score(&self, player: &str, delta: f64) -> Result<f64> {
        let mut conn = self.redis.get_async_connection().await?;
        let new_score: f64 = conn.zincr(&self.key, player, delta).await?;
        Ok(new_score)
    }

    // 批量更新
    async fn batch_update(&self, updates: &[(String, f64)]) -> Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        let mut pipe = redis::pipe();

        for (player, score) in updates {
            pipe.zadd(&self.key, player.as_str(), *score);
        }

        pipe.query_async(&mut conn).await?;
        Ok(())
    }
}
```

## 场景：延迟队列

用 ZSET 实现延迟队列：score 是执行时间戳。

```rust
use anyhow::Result;
use chrono::Utc;
use redis::AsyncCommands;
use std::time::Duration;

struct DelayedQueue {
    redis: redis::Client,
    key: String,
}

impl DelayedQueue {
    fn new(redis: redis::Client, key: impl Into<String>) -> Self {
        Self {
            redis,
            key: key.into(),
        }
    }

    async fn push(&self, task_id: &str, execute_at: i64) -> Result<()> {
        let mut conn = self.redis.get_async_connection().await?;
        conn.zadd(&self.key, task_id, execute_at as f64).await?;
        Ok(())
    }

    async fn pop_ready(&self) -> Result<Vec<String>> {
        let mut conn = self.redis.get_async_connection().await?;
        let now = Utc::now().timestamp();

        // 获取所有到期的任务
        let tasks: Vec<String> = conn
            .zrangebyscore(&self.key, 0, now)
            .await?;

        if !tasks.is_empty() {
            // 删除已取出的任务
            conn.zrem(&self.key, &tasks).await?;
        }

        Ok(tasks)
    }

    async fn poll(&self) -> Result<Vec<String>> {
        loop {
            let tasks = self.pop_ready().await?;
            if !tasks.is_empty() {
                return Ok(tasks);
            }

            // 看下一个任务什么时候到期
            let mut conn = self.redis.get_async_connection().await?;
            let next: Option<(String, f64)> = conn
                .zrange_withscores(&self.key, 0, 0)
                .await?
                .into_iter()
                .next();

            if let Some((_, score)) = next {
                let wait = (score as i64 - Utc::now().timestamp()).max(0);
                tokio::time::sleep(Duration::from_secs(wait as u64)).await;
            } else {
                // 队列为空，等一会再检查
                tokio::time::sleep(Duration::from_secs(1)).await;
            }
        }
    }
}
```

## 跳表的优化

### 1. 层数优化

```rust
// Redis 的做法：P = 0.25 而不是 0.5
const P: f64 = 0.25;

// 这样平均每个节点 1.33 个指针，而不是 2 个
// 查找多走一点，但内存省了
```

### 2. 记录跨度

Redis 的跳表还记录了每个指针的**跨度**（span），用于快速计算排名。

```rust
use std::ptr::NonNull;

struct RedisSkipListNode<V> {
    value: V,
    score: f64,
    forward: Vec<SkipListLevel<V>>,
}

struct SkipListLevel<V> {
    next: Option<NonNull<RedisSkipListNode<V>>>,
    span: usize,  // 跨越了多少个底层节点
}

struct RedisSkipList<V> {
    head: Box<RedisSkipListNode<V>>,
    level: usize,
}

impl<V> RedisSkipList<V> {
    // ZRANK 的实现：累加所有经过的 span
    fn get_rank(&self, score: f64) -> usize {
        let mut rank = 0;
        let mut current = &*self.head;

        for i in (0..self.level).rev() {
            while let Some(next) = current.forward[i].next {
                let next_node = unsafe { next.as_ref() };
                if next_node.score < score {
                    rank += current.forward[i].span;
                    current = next_node;
                } else {
                    break;
                }
            }
        }

        rank
    }
}
```

### 3. 后向指针

Redis 的跳表有**后向指针**，支持反向遍历（ZREVRANGE）。

```rust
struct RedisSkipListNode<V> {
    value: V,
    score: f64,
    forward: Vec<SkipListLevel>,
    backward: Option<NonNull<RedisSkipListNode<V>>>,  // 只在底层
}
```

## 跳表 vs 其他结构

| 特性 | 跳表 | 红黑树 | B+树 |
|------|------|--------|------|
| 实现复杂度 | 简单 | 复杂 | 中等 |
| 查找 | O(log n) | O(log n) | O(log n) |
| 范围查询 | 优秀 | 一般 | 优秀 |
| 并发 | 容易 | 困难 | 中等 |
| 内存局部性 | 差 | 差 | 好 |
| 磁盘友好 | 否 | 否 | 是 |

**选择建议**：

| 场景 | 推荐 |
|------|------|
| 内存有序集合 | 跳表 |
| 需要持久化 | B+树 |
| 标准库实现 | 红黑树（已有成熟实现） |
| 高并发 | 跳表（无锁版本） |

## 总结

| 操作 | 复杂度 | Redis 命令 |
|------|--------|-----------|
| 插入/更新 | O(log n) | ZADD |
| 删除 | O(log n) | ZREM |
| 按分数查找 | O(log n) | ZSCORE |
| 获取排名 | O(log n) | ZRANK |
| 范围查询 | O(log n + k) | ZRANGE |
| 按分数范围 | O(log n + k) | ZRANGEBYSCORE |

**核心认知**：

> 跳表的价值在于"简单实用"。它牺牲了一点点常数因子，换来了实现简单、范围查询友好、并发容易的优点。在需要有序数据结构的场景下，跳表是非常务实的选择。

---

*上一篇：[限流算法：保护系统的三道防线](/posts/algo-not-leetcode-7-rate-limit/)*

*下一篇：[树结构：层级数据的优雅解法](/posts/algo-not-leetcode-9-tree/)*

*本系列：*
1. *[算法是业务武器](/posts/algo-not-leetcode-1-intro/)*
2. *[排序与二分：被低估的基础功](/posts/algo-not-leetcode-2-sort-binary-search/)*
3. *[哈希表：缓存设计的基石](/posts/algo-not-leetcode-3-hashtable/)*
4. *[布隆过滤器：用 1% 误判换 90% 内存](/posts/algo-not-leetcode-4-bloom-filter/)*
5. *[一致性哈希：分布式系统的路由表](/posts/algo-not-leetcode-5-consistent-hash/)*
6. *[堆与优先队列：调度器的核心](/posts/algo-not-leetcode-6-heap/)*
7. *[限流算法：保护系统的三道防线](/posts/algo-not-leetcode-7-rate-limit/)*
8. *跳表：Redis 排行榜的秘密（本篇）*
9. *[树结构：层级数据的优雅解法](/posts/algo-not-leetcode-9-tree/)*
10. *[字符串匹配：敏感词过滤的正确姿势](/posts/algo-not-leetcode-10-string-match/)*
11. *[实战选型：没有银弹，只有场景](/posts/algo-not-leetcode-11-selection/)*
