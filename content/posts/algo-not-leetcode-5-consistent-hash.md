---
title: "算法不是面试题（五）：一致性哈希——分布式系统的路由表"
date: 2025-12-08T10:40:00+08:00
draft: false
tags: ["算法", "一致性哈希", "分布式", "缓存", "负载均衡"]
categories: ["技术"]
description: "4 台缓存服务器扩到 8 台，缓存全部失效？这不是玄学，是你用了错误的路由算法。一致性哈希让你扩容时数据迁移量最小化。"
series: ["算法不是面试题"]
---

4 台缓存服务器扩到 8 台，缓存全部失效？这不是玄学，是你用了错误的路由算法。一致性哈希让扩容时的数据迁移量最小化。

<!--more-->

## 问题：扩容的噩梦

你有一个缓存集群，4 台服务器：

```text
路由算法：hash(key) % 4

key = "user:12345"
hash("user:12345") = 87654321
87654321 % 4 = 1 → 路由到服务器 1
```

系统运行良好，缓存命中率 95%。

**然后，流量涨了，你要扩容到 8 台。**

```text
扩容后：hash(key) % 8

key = "user:12345"
hash("user:12345") = 87654321
87654321 % 8 = 1 → 还是服务器 1 ✓

key = "user:67890"
hash("user:67890") = 12345678
扩容前：12345678 % 4 = 2 → 服务器 2
扩容后：12345678 % 8 = 6 → 服务器 6 ✗
```

**问题**：大部分 key 被路由到了不同的服务器，缓存全部失效。

```text
数学分析：
  原来 N 台，扩容到 M 台
  一个 key 路由不变的概率 = N/M

  4 → 8：只有 4/8 = 50% 的 key 路由不变
  实际上更糟，因为 hash % 4 = x 的 key
  在 hash % 8 下只有一半还是 x

  结论：约 75% 的缓存失效
```

**后果**：

```text
扩容瞬间：
  - 缓存命中率从 95% 降到 ~25%
  - 大量请求穿透到数据库
  - 数据库被打爆
  - 系统崩溃

这就是开篇提到的 Knight Capital 式灾难的技术版本
```

## 一致性哈希：优雅的解决方案

### 核心思想

把服务器和 key 都映射到一个**环**上。

```text
想象一个圆环，刻度从 0 到 2^32-1

             0
            /|\
           / | \
          /  |  \
     Server A    Server B
        /    |    \
       /     |     \
      /      |      \
  2^32-1 ----+---- 2^16
             |
         Server C

key 的路由规则：
  1. 计算 hash(key)
  2. 在环上找到这个位置
  3. 顺时针找到第一个服务器
```

### 视觉化

```text
环：0 --------- A --------- B --------- C --------- 0
              (2^10)     (2^20)      (2^30)

key1: hash = 500    → 顺时针找到 A
key2: hash = 2^15   → 顺时针找到 B
key3: hash = 2^25   → 顺时针找到 C
key4: hash = 2^31   → 顺时针找到 A（绕回来）
```

### 扩容时发生什么

```text
添加服务器 D（位于 A 和 B 之间）：

原来：0 --- A --------- B --- C --- 0
现在：0 --- A --- D --- B --- C --- 0

只有原本路由到 B、但位置在 D 之前的 key 会受影响
其他 key 的路由完全不变！

影响范围 ≈ 1/(N+1)（N+1 是扩容后的服务器数量）
```

## Rust 实现

### 基础版本

```rust
use std::collections::BTreeMap;
use std::hash::{Hash, Hasher};
use std::collections::hash_map::DefaultHasher;

pub struct ConsistentHash<N> {
    ring: BTreeMap<u64, N>,  // 哈希值 → 节点
}

impl<N: Clone> ConsistentHash<N> {
    pub fn new() -> Self {
        Self { ring: BTreeMap::new() }
    }

    pub fn add_node(&mut self, node: N, node_id: &str) {
        let hash = self.hash(node_id);
        self.ring.insert(hash, node);
    }

    pub fn remove_node(&mut self, node_id: &str) {
        let hash = self.hash(node_id);
        self.ring.remove(&hash);
    }

    pub fn get_node(&self, key: &str) -> Option<&N> {
        if self.ring.is_empty() {
            return None;
        }

        let hash = self.hash(key);

        // 找到第一个 >= hash 的节点
        self.ring
            .range(hash..)
            .next()
            .or_else(|| self.ring.iter().next())  // 绕回到开头
            .map(|(_, node)| node)
    }

    fn hash(&self, key: &str) -> u64 {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        hasher.finish()
    }
}
```

### 使用示例

```rust
fn main() {
    let mut ring = ConsistentHash::new();

    // 添加服务器
    ring.add_node("192.168.1.1:6379", "redis-1");
    ring.add_node("192.168.1.2:6379", "redis-2");
    ring.add_node("192.168.1.3:6379", "redis-3");

    // 路由 key
    let server = ring.get_node("user:12345").unwrap();
    println!("user:12345 路由到: {}", server);

    // 添加新服务器
    ring.add_node("192.168.1.4:6379", "redis-4");

    // 大部分 key 的路由不变
    let server = ring.get_node("user:12345").unwrap();
    println!("扩容后，user:12345 路由到: {}", server);
}
```

## 问题：数据倾斜

基础版本有个问题：服务器在环上分布可能不均匀。

```text
极端情况：
  Server A: hash = 100
  Server B: hash = 200
  Server C: hash = 2^64 - 100

  A 负责：100 ~ 200（100 的范围）
  B 负责：200 ~ 2^64-100（几乎整个环！）
  C 负责：2^64-100 ~ 100（200 的范围）

  B 承担 99% 的请求
```

### 解决方案：虚拟节点

每个物理节点映射到多个虚拟节点：

```text
物理节点 A → 虚拟节点 A#1, A#2, A#3, ..., A#100
物理节点 B → 虚拟节点 B#1, B#2, B#3, ..., B#100

虚拟节点越多，分布越均匀
```

```rust
use std::collections::BTreeMap;
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

pub struct ConsistentHashWithVirtual<N> {
    ring: BTreeMap<u64, N>,
    virtual_nodes: usize,
}

impl<N: Clone> ConsistentHashWithVirtual<N> {
    pub fn new(virtual_nodes: usize) -> Self {
        Self {
            ring: BTreeMap::new(),
            virtual_nodes,
        }
    }

    pub fn add_node(&mut self, node: N, node_id: &str) {
        for i in 0..self.virtual_nodes {
            let virtual_id = format!("{}#{}", node_id, i);
            let hash = self.hash(&virtual_id);
            self.ring.insert(hash, node.clone());
        }
    }

    pub fn remove_node(&mut self, node_id: &str) {
        for i in 0..self.virtual_nodes {
            let virtual_id = format!("{}#{}", node_id, i);
            let hash = self.hash(&virtual_id);
            self.ring.remove(&hash);
        }
    }

    pub fn get_node(&self, key: &str) -> Option<&N> {
        if self.ring.is_empty() {
            return None;
        }
        let hash = self.hash(key);
        self.ring
            .range(hash..)
            .next()
            .or_else(|| self.ring.iter().next())
            .map(|(_, node)| node)
    }

    fn hash(&self, key: &str) -> u64 {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        hasher.finish()
    }
}
```

### 虚拟节点数量

```text
虚拟节点数量的选择：

太少（如 10）：分布不均匀
太多（如 10000）：内存占用大，查找慢

推荐：100-200 个虚拟节点

实际测试（1000 万 key，4 台服务器）：
  虚拟节点数 | 最大/最小负载比
  10        | 3.2
  50        | 1.5
  100       | 1.2
  200       | 1.1
```

## 场景：分布式缓存

### 架构

```text
Client → 一致性哈希路由 → Redis 集群
                          ├── Redis 1
                          ├── Redis 2
                          ├── Redis 3
                          └── Redis 4
```

### 实现

```rust
use redis::Client;

pub struct CacheCluster {
    ring: ConsistentHashWithVirtual<Client>,
}

impl CacheCluster {
    pub fn new(nodes: Vec<&str>) -> Self {
        let mut ring = ConsistentHashWithVirtual::new(150);
        for node in nodes {
            let client = Client::open(format!("redis://{}", node)).unwrap();
            ring.add_node(client, node);
        }
        Self { ring }
    }

    pub async fn get(&self, key: &str) -> Option<String> {
        let client = self.ring.get_node(key)?;
        let mut conn = client.get_async_connection().await.ok()?;
        redis::cmd("GET").arg(key).query_async(&mut conn).await.ok()
    }

    pub async fn set(&self, key: &str, value: &str, ttl: usize) {
        if let Some(client) = self.ring.get_node(key) {
            if let Ok(mut conn) = client.get_async_connection().await {
                let _: () = redis::cmd("SETEX")
                    .arg(key)
                    .arg(ttl)
                    .arg(value)
                    .query_async(&mut conn)
                    .await
                    .unwrap_or(());
            }
        }
    }
}
```

### 扩容流程

```rust
impl CacheCluster {
    pub async fn add_node(&mut self, node: &str) {
        // 1. 添加到哈希环
        let client = Client::open(format!("redis://{}", node)).unwrap();
        self.ring.add_node(client, node);

        // 2. 数据迁移（可选，用于热点数据）
        // 新节点会自动承担一部分 key
        // 对于缓存场景，通常不需要主动迁移
        // 缓存 miss 后会自动写入正确的节点
    }

    pub async fn remove_node(&mut self, node: &str) {
        // 1. 标记节点为不可用
        // 2. 从哈希环移除
        self.ring.remove_node(node);

        // 被移除节点的 key 会自动路由到下一个节点
        // 缓存 miss 后重新填充
    }
}
```

## 场景：数据库分片

缓存可以容忍丢失，数据库不行。

```text
缓存扩容：数据丢了就重新从 DB 加载
数据库扩容：数据丢了就完了

数据库分片需要：
  1. 一致性哈希路由
  2. 数据迁移机制
```

### 带权重的一致性哈希

不同机器性能不同，需要按权重分配。

```rust
use std::collections::BTreeMap;
use std::hash::{Hash, Hasher};
use std::collections::hash_map::DefaultHasher;

pub struct WeightedConsistentHash<N> {
    ring: BTreeMap<u64, N>,
    base_virtual_nodes: usize,
}

impl<N: Clone> WeightedConsistentHash<N> {
    pub fn new(base_virtual_nodes: usize) -> Self {
        Self {
            ring: BTreeMap::new(),
            base_virtual_nodes,
        }
    }

    pub fn add_node(&mut self, node: N, node_id: &str, weight: usize) {
        // 权重越高，虚拟节点越多
        let virtual_count = self.base_virtual_nodes * weight;
        for i in 0..virtual_count {
            let virtual_id = format!("{}#{}", node_id, i);
            let hash = self.hash(&virtual_id);
            self.ring.insert(hash, node.clone());
        }
    }

    pub fn get_node(&self, key: &str) -> Option<&N> {
        if self.ring.is_empty() {
            return None;
        }
        let hash = self.hash(key);
        self.ring
            .range(hash..)
            .next()
            .or_else(|| self.ring.iter().next())
            .map(|(_, node)| node)
    }

    fn hash(&self, key: &str) -> u64 {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        hasher.finish()
    }
}

// 使用
let mut ring = WeightedConsistentHash::new(50);
ring.add_node("db-1", "db-1", 2);  // 2 倍性能
ring.add_node("db-2", "db-2", 1);  // 基准性能
ring.add_node("db-3", "db-3", 1);

// db-1 承担约 50% 的流量，db-2 和 db-3 各 25%
```

## 一致性哈希 vs 其他方案

| 方案 | 扩容影响 | 实现复杂度 | 适用场景 |
|------|----------|------------|----------|
| hash % N | ~100% 数据迁移 | 简单 | 节点数固定 |
| 一致性哈希 | ~1/N 数据迁移 | 中等 | 缓存、无状态服务 |
| 范围分片 | 可控 | 复杂 | 数据库 |
| 虚拟桶 | 可控 | 复杂 | 大规模分布式存储 |

### 虚拟桶方案

```text
一致性哈希的问题：
  - 虚拟节点多，内存占用大
  - 节点变更时，需要重新计算所有虚拟节点

虚拟桶方案：
  1. 固定数量的虚拟桶（如 1024 个）
  2. key → hash % 1024 → 桶
  3. 桶 → 物理节点（映射表）

  扩容时只需修改映射表，不需要重新哈希
```

```rust
use std::hash::{Hash, Hasher};
use std::collections::hash_map::DefaultHasher;

pub struct VirtualBucket {
    bucket_count: usize,
    bucket_to_node: Vec<String>,  // 桶 → 节点映射
}

impl VirtualBucket {
    pub fn new(bucket_count: usize, default_node: &str) -> Self {
        Self {
            bucket_count,
            bucket_to_node: vec![default_node.to_string(); bucket_count],
        }
    }

    pub fn get_node(&self, key: &str) -> &str {
        let bucket = self.hash(key) % self.bucket_count;
        &self.bucket_to_node[bucket]
    }

    pub fn migrate_bucket(&mut self, bucket: usize, new_node: &str) {
        self.bucket_to_node[bucket] = new_node.to_string();
    }

    fn hash(&self, key: &str) -> usize {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        hasher.finish() as usize
    }
}
```

## 常见陷阱

### 1. 哈希函数选择

```rust
// 不好：DefaultHasher 不稳定
// 同一个 key，不同进程可能得到不同的 hash
use std::collections::hash_map::DefaultHasher;

// 好：使用稳定的哈希函数
use xxhash_rust::xxh64::xxh64;

fn stable_hash(key: &str) -> u64 {
    xxh64(key.as_bytes(), 0)  // 种子固定为 0
}
```

### 2. 热点 key

```text
问题：
  某些 key 访问量特别大（热点）
  即使分布均匀，热点 key 所在的节点也会过载

解决方案：
  1. 热点 key 缓存在客户端
  2. 热点 key 复制到多个节点（读取时随机选择）
  3. 热点 key 使用单独的缓存层
```

### 3. 脑裂问题

```text
场景：
  网络分区导致不同客户端看到不同的节点列表
  同一个 key 被路由到不同节点
  数据不一致

解决方案：
  使用配置中心（如 etcd）同步节点列表
  确保所有客户端看到一致的视图
```

## 总结

| 特性 | 传统哈希 | 一致性哈希 |
|------|----------|------------|
| 扩容影响 | ~100% | ~1/N |
| 缩容影响 | ~100% | ~1/N |
| 实现复杂度 | O(1) | O(log N) |
| 负载均衡 | 取决于哈希函数 | 需要虚拟节点 |

**适用场景**：

| 场景 | 推荐方案 |
|------|----------|
| 缓存集群 | 一致性哈希 + 虚拟节点 |
| 数据库分片 | 虚拟桶 + 映射表 |
| 负载均衡 | 一致性哈希 + 权重 |
| 固定节点数 | 简单取模 |

**核心认知**：

> 一致性哈希的核心价值不是"分布均匀"（虚拟节点解决），而是"变化最小"——节点增减时，只有最少的数据需要迁移。这对于分布式系统的稳定性至关重要。

---

*上一篇：[布隆过滤器：用 1% 误判换 90% 内存](/posts/algo-not-leetcode-4-bloom-filter/)*

*下一篇：[堆与优先队列：调度器的核心](/posts/algo-not-leetcode-6-heap/)*

*本系列：*
1. *[算法是业务武器](/posts/algo-not-leetcode-1-intro/)*
2. *[排序与二分：被低估的基础功](/posts/algo-not-leetcode-2-sort-binary-search/)*
3. *[哈希表：缓存设计的基石](/posts/algo-not-leetcode-3-hashtable/)*
4. *[布隆过滤器：用 1% 误判换 90% 内存](/posts/algo-not-leetcode-4-bloom-filter/)*
5. *一致性哈希：分布式系统的路由表（本篇）*
6. *[堆与优先队列：调度器的核心](/posts/algo-not-leetcode-6-heap/)*
7. *[限流算法：保护系统的三道防线](/posts/algo-not-leetcode-7-rate-limit/)*
8. *[跳表：Redis 排行榜的秘密](/posts/algo-not-leetcode-8-skiplist/)*
9. *[树结构：层级数据的优雅解法](/posts/algo-not-leetcode-9-tree/)*
10. *[字符串匹配：敏感词过滤的正确姿势](/posts/algo-not-leetcode-10-string-match/)*
11. *[实战选型：没有银弹，只有场景](/posts/algo-not-leetcode-11-selection/)*
