---
title: "算法不是面试题（十一）：实战选型——没有银弹，只有场景"
date: 2025-12-08T11:40:00+08:00
draft: false
tags: ["算法", "系统设计", "技术选型", "实战"]
categories: ["技术"]
description: "学了这么多算法，遇到实际问题该用哪个？这篇文章用真实案例告诉你：算法选型不是考试，而是在约束条件下找最优解。"
series: ["算法不是面试题"]
---

学了这么多算法，遇到实际问题该用哪个？这篇文章用真实案例告诉你：算法选型不是考试，而是在约束条件下找最优解。

<!--more-->

## 选型框架

面对一个问题，按这个顺序思考：

```text
1. 问题是什么？
   - 输入什么，输出什么
   - 核心操作是什么（查找、排序、去重...）

2. 约束是什么？
   - 数据规模（100 还是 1 亿）
   - 性能要求（毫秒级还是可以秒级）
   - 内存限制（单机还是分布式）
   - 准确性（能否容忍误判）

3. 权衡是什么？
   - 时间 vs 空间
   - 准确性 vs 性能
   - 复杂度 vs 可维护性

4. 先用最简单的方案，性能不够再优化
```

## 案例一：用户在线状态

### 需求

```text
IM 系统，1 亿用户
需求：查询任意用户是否在线
QPS：10 万
```

### 分析

```text
核心操作：判断元素是否在集合中
数据规模：1 亿
读写比：读多写少（在线状态变化不频繁）
准确性：需要精确
```

### 方案对比

| 方案 | 空间 | 查询 | 更新 | 准确性 |
|------|------|------|------|--------|
| HashMap | 8GB+ | O(1) | O(1) | 精确 |
| 布隆过滤器 | 120MB | O(k) | O(k) | 有误判 |
| Redis Bitmap | 12.5MB | O(1) | O(1) | 精确 |

### 选择：Redis Bitmap

```rust
use redis::AsyncCommands;

type Redis = redis::Client;

// 用户 ID 作为偏移量
// 在线 = 1，离线 = 0

async fn set_online(redis: &Redis, user_id: u64) -> redis::RedisResult<()> {
    let mut conn = redis.get_async_connection().await?;
    conn.setbit("online_users", user_id as usize, true).await
}

async fn is_online(redis: &Redis, user_id: u64) -> redis::RedisResult<bool> {
    let mut conn = redis.get_async_connection().await?;
    conn.getbit("online_users", user_id as usize).await
}

async fn count_online(redis: &Redis) -> redis::RedisResult<u64> {
    let mut conn = redis.get_async_connection().await?;
    conn.bitcount("online_users").await
}
```

**为什么**：
- 空间极小：1 亿用户只需 12.5MB
- 操作 O(1)
- 支持统计在线人数
- Redis 原生支持

## 案例二：搜索框自动补全

### 需求

```text
电商搜索框，输入前缀，返回前 10 个匹配商品
商品数：1000 万
响应时间：< 50ms
```

### 分析

```text
核心操作：前缀匹配
数据规模：1000 万
排序需求：按热度排序
```

### 方案对比

| 方案 | 查询复杂度 | 空间 | 特点 |
|------|-----------|------|------|
| 数据库 LIKE | O(n) | - | 太慢 |
| Trie | O(k) | 大 | 返回所有匹配，需再排序 |
| Redis ZSET | O(log n) | 中 | 需要特殊存储 |
| Elasticsearch | O(1)* | 大 | 专业搜索引擎 |

### 选择：Trie + 堆

```rust
use std::cmp::Reverse;
use std::collections::{BinaryHeap, HashMap};

#[derive(Default)]
struct TrieNode {
    children: HashMap<char, TrieNode>,
    score: Option<u64>,  // 词的热度分数
}

struct Trie {
    root: TrieNode,
}

impl Trie {
    fn new() -> Self {
        Self { root: TrieNode::default() }
    }

    fn insert(&mut self, word: &str, score: u64) {
        let mut node = &mut self.root;
        for c in word.chars() {
            node = node.children.entry(c).or_default();
        }
        node.score = Some(score);
    }

    fn find_prefix_node(&self, prefix: &str) -> Option<&TrieNode> {
        let mut node = &self.root;
        for c in prefix.chars() {
            node = node.children.get(&c)?;
        }
        Some(node)
    }
}

#[derive(Eq, PartialEq)]
struct ScoredWord {
    word: String,
    score: u64,
}

impl Ord for ScoredWord {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        self.score.cmp(&other.score)
    }
}

impl PartialOrd for ScoredWord {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

struct AutoComplete {
    trie: Trie,
}

impl AutoComplete {
    fn new() -> Self {
        Self { trie: Trie::new() }
    }

    fn add_word(&mut self, word: &str, score: u64) {
        self.trie.insert(word, score);
    }

    fn suggest(&self, prefix: &str, limit: usize) -> Vec<String> {
        // 找到前缀节点
        let node = match self.trie.find_prefix_node(prefix) {
            Some(n) => n,
            None => return vec![],
        };

        // 用小顶堆维护 Top-K
        let mut heap = BinaryHeap::with_capacity(limit + 1);

        self.collect_words(node, prefix, &mut heap, limit);

        heap.into_sorted_vec()
            .into_iter()
            .map(|item| item.0.word)
            .collect()
    }

    fn collect_words(
        &self,
        node: &TrieNode,
        current: &str,
        heap: &mut BinaryHeap<Reverse<ScoredWord>>,
        limit: usize,
    ) {
        if let Some(score) = node.score {
            heap.push(Reverse(ScoredWord {
                word: current.to_string(),
                score,
            }));
            if heap.len() > limit {
                heap.pop();
            }
        }

        for (c, child) in &node.children {
            let next = format!("{}{}", current, c);
            self.collect_words(child, &next, heap, limit);
        }
    }
}
```

**为什么**：
- Trie 前缀匹配 O(k)，k 是前缀长度
- 堆维护 Top-K，不需要全排序
- 内存可控

**进阶**：如果需要更多特性（拼音、纠错），用 Elasticsearch。

## 案例三：订单号生成

### 需求

```text
分布式系统，每秒 10 万订单
要求：
  - 全局唯一
  - 趋势递增（便于数据库索引）
  - 包含业务信息（时间、机器）
```

### 方案对比

| 方案 | 唯一性 | 有序性 | 长度 | 依赖 |
|------|--------|--------|------|------|
| UUID v4 | 是 | 否 | 36字符 | 无 |
| 自增 ID | 是 | 是 | 可变 | 数据库 |
| Redis INCR | 是 | 是 | 可变 | Redis |
| Snowflake | 是 | 是 | 19位数字 | 时钟 |

### 选择：Snowflake

```text
Snowflake ID 结构（64 位）：
  1 位：符号位（0）
  41 位：时间戳（毫秒，可用 69 年）
  10 位：机器 ID（1024 台机器）
  12 位：序列号（每毫秒 4096 个）
```

```rust
use std::time::{SystemTime, UNIX_EPOCH};

pub struct Snowflake {
    epoch: i64,           // 起始时间戳
    machine_id: i64,      // 机器 ID
    sequence: i64,        // 序列号
    last_timestamp: i64,  // 上次生成时间
}

impl Snowflake {
    pub fn new(machine_id: i64) -> Self {
        // 自定义纪元：2020-01-01 00:00:00 UTC
        let epoch = 1577836800000;
        Self {
            epoch,
            machine_id: machine_id & 0x3FF,  // 10 位
            sequence: 0,
            last_timestamp: 0,
        }
    }

    pub fn next_id(&mut self) -> i64 {
        let mut timestamp = self.current_millis();

        if timestamp == self.last_timestamp {
            self.sequence = (self.sequence + 1) & 0xFFF;  // 12 位
            if self.sequence == 0 {
                // 当前毫秒用完，等下一毫秒
                timestamp = self.wait_next_millis();
            }
        } else {
            self.sequence = 0;
        }

        self.last_timestamp = timestamp;

        // 组装 ID
        ((timestamp - self.epoch) << 22)
            | (self.machine_id << 12)
            | self.sequence
    }

    fn current_millis(&self) -> i64 {
        SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_millis() as i64
    }

    fn wait_next_millis(&self) -> i64 {
        let mut timestamp = self.current_millis();
        while timestamp <= self.last_timestamp {
            timestamp = self.current_millis();
        }
        timestamp
    }
}
```

**为什么**：
- 纯本地生成，无网络依赖
- 趋势递增，数据库友好
- 高性能：单机每秒 400 万

**生产级替代**：如果不想自己实现，可以使用活跃维护的第三方库：
- `snowflaked` - 经典 Snowflake 实现，API 简洁
- `ulid` - 128 位，字典序可排序，无需机器 ID 配置

## 案例四：实时排行榜

### 需求

```text
游戏排行榜
玩家数：1000 万
更新频率：每秒 1 万次分数变化
查询：Top 100、某玩家排名、周围玩家
```

### 分析

```text
核心操作：
  - 更新分数
  - 获取排名
  - 范围查询

特点：读写都很频繁
```

### 选择：Redis ZSET

```rust
use redis::AsyncCommands;

type Redis = redis::Client;

struct Leaderboard {
    redis: Redis,
    key: String,
}

impl Leaderboard {
    fn new(redis: Redis, key: impl Into<String>) -> Self {
        Self {
            redis,
            key: key.into(),
        }
    }

    // 更新分数：O(log n)
    async fn update_score(&self, player_id: &str, score: f64) -> redis::RedisResult<()> {
        let mut conn = self.redis.get_async_connection().await?;
        conn.zadd(&self.key, player_id, score).await
    }

    // 获取排名：O(log n)
    async fn get_rank(&self, player_id: &str) -> redis::RedisResult<Option<u64>> {
        let mut conn = self.redis.get_async_connection().await?;
        conn.zrevrank(&self.key, player_id).await
    }

    // Top N：O(log n + N)
    async fn top_n(&self, n: usize) -> redis::RedisResult<Vec<(String, f64)>> {
        let mut conn = self.redis.get_async_connection().await?;
        conn.zrevrange_withscores(&self.key, 0, (n - 1) as isize).await
    }

    // 周围玩家：O(log n + k)
    async fn around_player(&self, player_id: &str, count: usize) -> redis::RedisResult<Vec<(String, f64)>> {
        let rank = match self.get_rank(player_id).await? {
            Some(r) => r as isize,
            None => return Ok(vec![]),
        };

        let start = (rank - count as isize / 2).max(0);
        let end = rank + count as isize / 2;

        let mut conn = self.redis.get_async_connection().await?;
        conn.zrevrange_withscores(&self.key, start, end).await
    }
}
```

**为什么**：
- 底层是跳表，所有操作 O(log n)
- Redis 原生支持，无需自己实现
- 支持分布式

## 案例五：URL 去重

### 需求

```text
爬虫系统，判断 URL 是否已爬取
预期 URL 数：10 亿
内存限制：4GB
允许少量误判
```

### 分析

```text
核心操作：集合成员判断
数据规模：10 亿
准确性：允许误判（重爬一下不是大问题）
```

### 选择：布隆过滤器

```rust
// 10 亿 URL，1% 误判率
// 需要 ~1.2GB 内存

use fastbloom::BloomFilter;

struct UrlDedup {
    bloom: BloomFilter,
}

impl UrlDedup {
    fn new() -> Self {
        // 10 亿元素，1% 误判率
        let bloom = BloomFilter::with_false_pos(0.01)
            .expected_items(1_000_000_000);
        Self { bloom }
    }

    fn should_crawl(&mut self, url: &str) -> bool {
        if self.bloom.contains(&url) {
            false  // 可能已爬取
        } else {
            self.bloom.insert(&url);
            true   // 肯定没爬取
        }
    }
}
```

**为什么**：
- 1.2GB 存 10 亿 URL，HashSet 要 80GB+
- 误判只会导致重爬，不是致命问题
- 不会漏爬（假阴性率为 0）

## 案例六：限流器

### 需求

```text
API 网关限流
规则：
  - 全局：10000 QPS
  - 每用户：100 QPS
  - 允许短时突发
```

### 选择：令牌桶 + 多维度

```rust
use dashmap::DashMap;
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::{Duration, Instant};

struct TokenBucket {
    capacity: f64,
    rate: f64,
    tokens: f64,
    last_refill: Instant,
}

impl TokenBucket {
    fn new(capacity: f64, rate: f64) -> Self {
        Self {
            capacity,
            rate,
            tokens: capacity,
            last_refill: Instant::now(),
        }
    }

    fn try_acquire(&mut self) -> bool {
        self.refill();
        if self.tokens >= 1.0 {
            self.tokens -= 1.0;
            true
        } else {
            false
        }
    }

    fn refill(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.rate).min(self.capacity);
        self.last_refill = now;
    }
}

struct RateLimiter {
    global: TokenBucket,
    per_user: DashMap<String, TokenBucket>,
}

impl RateLimiter {
    fn new(global_capacity: f64, global_rate: f64) -> Self {
        Self {
            global: TokenBucket::new(global_capacity, global_rate),
            per_user: DashMap::new(),
        }
    }

    fn check(&mut self, user_id: &str) -> bool {
        // 全局检查
        if !self.global.try_acquire() {
            return false;
        }

        // 用户检查
        let mut user_bucket = self.per_user
            .entry(user_id.to_string())
            .or_insert_with(|| TokenBucket::new(100.0, 10.0));

        user_bucket.try_acquire()
    }
}
```

**为什么**：
- 令牌桶允许突发
- 多维度保护公平性
- 本地计算，性能高

## 选型速查表

### 按问题类型

| 问题 | 首选方案 | 备选方案 |
|------|----------|----------|
| 查找元素 | HashMap | BTreeMap（需有序） |
| 判断存在（大数据） | 布隆过滤器 | Cuckoo Filter |
| 有序集合 | BTreeMap | 跳表 |
| 优先级处理 | 堆 | 有序数组（小数据） |
| 范围查询 | B+树 | 跳表 |
| 前缀匹配 | Trie | HashMap（暴力） |
| 多模式匹配 | AC 自动机 | 正则（少量模式） |
| 分布式路由 | 一致性哈希 | 范围分片 |

### 按场景

| 场景 | 推荐算法/数据结构 |
|------|------------------|
| 缓存 | HashMap + LRU |
| 排行榜 | Redis ZSET（跳表） |
| 延迟队列 | 堆 / Redis ZSET |
| 敏感词过滤 | AC 自动机 |
| 去重（大数据） | 布隆过滤器 |
| 限流 | 令牌桶 / 滑动窗口 |
| ID 生成 | Snowflake |
| 层级数据 | 邻接表 / 闭包表 |
| 数据对账 | 排序 + 双指针 |

### 按数据规模

| 规模 | 策略 |
|------|------|
| < 1000 | 随便选，都很快 |
| 1000 - 100万 | 注意复杂度，避免 O(n²) |
| 100万 - 1亿 | 选对数据结构很重要 |
| > 1亿 | 考虑分布式、近似算法 |

## 常见误区

### 1. 过早优化

```text
错误：数据量 100，花 2 天实现红黑树
正确：先用 Vec 排序，有问题再优化
```

### 2. 过度设计

```text
错误：为了"未来扩展"，现在就上分布式
正确：先单机，确认是瓶颈再分布式
```

### 3. 忽视常数因子

```text
理论：HashMap O(1) vs BTreeMap O(log n)
实际：1000 个元素，BTreeMap 可能更快（缓存友好）
```

### 4. 只看时间复杂度

```text
错误：只看时间复杂度，选了内存爆炸的方案
正确：时间、空间、可维护性综合考虑
```

## 系列总结

这个系列讲了 10 种算法/数据结构的业务应用：

| 篇目 | 算法 | 核心场景 |
|------|------|----------|
| 二 | 排序、二分 | 版本回滚、数据对账 |
| 三 | 哈希表 | 缓存、LRU、幂等 |
| 四 | 布隆过滤器 | 缓存穿透、去重 |
| 五 | 一致性哈希 | 分布式路由 |
| 六 | 堆 | 延迟任务、Top-K |
| 七 | 限流算法 | 流量控制 |
| 八 | 跳表 | 排行榜 |
| 九 | 树结构 | 层级数据 |
| 十 | AC 自动机 | 敏感词过滤 |

**核心认知**：

> 算法不是面试题，是解决业务问题的工具。选型的关键不是"哪个更高级"，而是"哪个更适合"。
>
> 最好的算法是：**能解决问题的最简单的那个**。

---

*上一篇：[字符串匹配：敏感词过滤的正确姿势](/posts/algo-not-leetcode-10-string-match/)*

*本系列：*
1. *[算法是业务武器](/posts/algo-not-leetcode-1-intro/)*
2. *[排序与二分：被低估的基础功](/posts/algo-not-leetcode-2-sort-binary-search/)*
3. *[哈希表：缓存设计的基石](/posts/algo-not-leetcode-3-hashtable/)*
4. *[布隆过滤器：用 1% 误判换 90% 内存](/posts/algo-not-leetcode-4-bloom-filter/)*
5. *[一致性哈希：分布式系统的路由表](/posts/algo-not-leetcode-5-consistent-hash/)*
6. *[堆与优先队列：调度器的核心](/posts/algo-not-leetcode-6-heap/)*
7. *[限流算法：保护系统的三道防线](/posts/algo-not-leetcode-7-rate-limit/)*
8. *[跳表：Redis 排行榜的秘密](/posts/algo-not-leetcode-8-skiplist/)*
9. *[树结构：层级数据的优雅解法](/posts/algo-not-leetcode-9-tree/)*
10. *[字符串匹配：敏感词过滤的正确姿势](/posts/algo-not-leetcode-10-string-match/)*
11. *实战选型：没有银弹，只有场景（本篇）*
