---
title: "算法不是面试题（四）：布隆过滤器——用 1% 误判换 90% 内存"
date: 2025-12-08T10:30:00+08:00
draft: false
tags: ["算法", "布隆过滤器", "缓存穿透", "黑名单"]
categories: ["技术"]
description: "10 亿个 ID，判断某个 ID 是否存在。用 HashSet？内存爆炸。用布隆过滤器，1GB 搞定，代价是 1% 的误判率。这笔账怎么算？"
series: ["算法不是面试题"]
---

10 亿个 ID，判断某个 ID 是否存在。用 HashSet？内存爆炸。用布隆过滤器，1GB 搞定，代价是 1% 的误判率。这笔账怎么算？

<!--more-->

## 场景：缓存穿透

电商系统，商品详情页的经典架构：

```text
请求 → 缓存（Redis） → 数据库（MySQL）

正常流程：
  1. 查缓存，命中 → 返回
  2. 查缓存，未命中 → 查数据库 → 写缓存 → 返回
```

**攻击来了**：

```text
攻击者用不存在的商品 ID 疯狂请求：
  商品 ID = -1, -2, -3, ... (都不存在)

每个请求：
  1. 查缓存 → 未命中
  2. 查数据库 → 不存在
  3. 不写缓存（因为没数据）

下次请求：
  继续穿透到数据库
```

**结果**：缓存形同虚设，数据库被打爆。这就是**缓存穿透**。

### 朴素防御：缓存空值

```rust
async fn get_product(id: i64) -> Option<Product> {
    // 查缓存
    if let Some(cached) = cache.get(&id).await {
        return cached;  // 可能是 Some(product) 或 None
    }

    // 查数据库
    let product = db.find_product(id).await;

    // 即使不存在也缓存（空值）
    cache.set(&id, product.clone(), Duration::from_secs(60)).await;

    product
}
```

**问题**：攻击者可以用海量不同的 ID，每个 ID 都缓存一个空值，内存爆炸。

### 布隆过滤器防御

```text
思路：
  在缓存前面加一层"守门员"
  守门员知道哪些 ID 存在于数据库中
  不存在的 ID 直接拒绝，不查缓存也不查数据库
```

```rust
async fn get_product_with_bloom(id: i64) -> Option<Product> {
    // 布隆过滤器判断
    if !bloom_filter.may_contain(id) {
        return None;  // 肯定不存在，直接返回
    }

    // 可能存在，继续查缓存和数据库
    // ...
}
```

## 布隆过滤器原理

### 核心思想

用一个位数组和多个哈希函数，表示一个集合。

```text
位数组：[0, 0, 0, 0, 0, 0, 0, 0, 0, 0]  (长度 10)
哈希函数：h1, h2, h3

插入元素 "apple"：
  h1("apple") % 10 = 2
  h2("apple") % 10 = 5
  h3("apple") % 10 = 8

  把位置 2, 5, 8 设为 1：
  [0, 0, 1, 0, 0, 1, 0, 0, 1, 0]

查询元素 "apple"：
  检查位置 2, 5, 8 是否都是 1
  都是 1 → 可能存在

查询元素 "banana"：
  h1("banana") % 10 = 1
  h2("banana") % 10 = 5
  h3("banana") % 10 = 7

  检查位置 1, 5, 7
  位置 1 是 0 → 肯定不存在
```

### 关键特性

| 查询结果 | 含义 |
|----------|------|
| 返回"不存在" | **100% 准确**，肯定不在集合中 |
| 返回"存在" | **可能误判**，可能在也可能不在 |

**为什么会误判**：不同元素的哈希位置可能重叠。

```text
插入 "apple"：位置 2, 5, 8
插入 "orange"：位置 3, 5, 9

查询 "grape"：
  h1("grape") = 2  ← apple 设的
  h2("grape") = 3  ← orange 设的
  h3("grape") = 9  ← orange 设的

  三个位置都是 1，返回"存在"
  但实际上 grape 没有插入过 → 误判！
```

### 数学公式

```text
假设：
  m = 位数组长度
  n = 已插入元素数量
  k = 哈希函数数量

误判率 p ≈ (1 - e^(-kn/m))^k

最优 k = (m/n) × ln(2) ≈ 0.7 × (m/n)
```

**实用参数**：

| 误判率 | m/n (每元素位数) | 哈希函数数 k |
|--------|-----------------|-------------|
| 10% | 4.8 | 3 |
| 1% | 9.6 | 7 |
| 0.1% | 14.4 | 10 |

**例子**：10 亿个元素，1% 误判率

```text
m = 10 亿 × 9.6 = 96 亿位 = 12 亿字节 ≈ 1.2 GB

对比 HashSet：
  每个元素至少 8 字节（64位 ID）
  10 亿 × 8 = 80 亿字节 ≈ 8 GB
  还不算哈希表的额外开销

布隆过滤器节省 ~85% 内存
```

## Rust 实现

### 基础版本

```rust
use bitvec::prelude::*;
use std::hash::{Hash, Hasher};
use std::collections::hash_map::DefaultHasher;

pub struct BloomFilter {
    bits: BitVec,
    hash_count: usize,
}

impl BloomFilter {
    pub fn new(size: usize, hash_count: usize) -> Self {
        Self {
            bits: bitvec![0; size],
            hash_count,
        }
    }

    pub fn insert<T: Hash>(&mut self, item: &T) {
        for i in 0..self.hash_count {
            let idx = self.hash_index(item, i);
            self.bits.set(idx, true);
        }
    }

    pub fn may_contain<T: Hash>(&self, item: &T) -> bool {
        for i in 0..self.hash_count {
            let idx = self.hash_index(item, i);
            if !self.bits[idx] {
                return false;  // 肯定不存在
            }
        }
        true  // 可能存在
    }

    fn hash_index<T: Hash>(&self, item: &T, seed: usize) -> usize {
        let mut hasher = DefaultHasher::new();
        item.hash(&mut hasher);
        seed.hash(&mut hasher);
        (hasher.finish() as usize) % self.bits.len()
    }
}
```

### 使用示例

```rust
fn main() {
    // 100 万位，7 个哈希函数，预期 1% 误判率
    let mut bloom = BloomFilter::new(1_000_000 * 10, 7);

    // 插入已知存在的 ID
    for id in valid_product_ids() {
        bloom.insert(&id);
    }

    // 查询
    let test_id = 12345i64;
    if bloom.may_contain(&test_id) {
        println!("可能存在，继续查数据库");
    } else {
        println!("肯定不存在，直接返回");
    }
}
```

### 生产级实现

实际项目用 `fastbloom` crate（高性能、活跃维护、支持并发）：

```rust
use fastbloom::BloomFilter;

// 指定误判率，自动计算最优参数
let mut bloom = BloomFilter::with_false_pos(0.01)  // 1% 误判率
    .expected_items(10_000_000);                    // 1000 万元素

bloom.insert(&"user:123");
assert!(bloom.contains(&"user:123"));  // true（可能存在）
assert!(!bloom.contains(&"user:999")); // false（肯定不存在）
```

## 场景二：黑名单过滤

风控系统需要过滤恶意用户/IP/设备。

```text
需求：
  - 10 亿个黑名单 ID
  - 每秒 10 万次查询
  - 查询延迟 < 1ms
```

### 为什么不用数据库？

```text
MySQL 查询：~1-10ms
Redis 查询：~0.5-2ms
布隆过滤器：~0.001ms（纯内存位操作）
```

布隆过滤器快 1000 倍。

### 架构设计

```text
请求 → 布隆过滤器 → 精确验证
         ↓
    "肯定不在黑名单" → 放行
         ↓
    "可能在黑名单" → 查 Redis 确认
```

```rust
async fn check_blacklist(user_id: &str) -> bool {
    // 第一层：布隆过滤器
    if !bloom_filter.may_contain(user_id) {
        return false;  // 肯定不在黑名单
    }

    // 第二层：Redis 精确确认（只有 1% 的请求会走到这里）
    redis.exists(&format!("blacklist:{}", user_id)).await
}
```

**效果**：99% 的正常请求在布隆过滤器层就返回了，Redis 压力降低 99%。

## 布隆过滤器的局限

### 1. 不能删除

```text
为什么不能删除？

假设 apple 和 orange 都设置了位置 5
删除 apple 时，能把位置 5 设为 0 吗？

不能！orange 也用了位置 5
设为 0 会导致 orange 也查不到
```

**解决方案**：Counting Bloom Filter

```rust
use std::hash::{Hash, Hasher};
use std::collections::hash_map::DefaultHasher;

// 每个位置存计数而不是 0/1
struct CountingBloomFilter {
    counts: Vec<u8>,  // 每个位置是一个计数器
    hash_count: usize,
}

impl CountingBloomFilter {
    fn new(size: usize, hash_count: usize) -> Self {
        Self {
            counts: vec![0; size],
            hash_count,
        }
    }

    fn insert<T: Hash>(&mut self, item: &T) {
        for i in 0..self.hash_count {
            let idx = self.hash_index(item, i);
            self.counts[idx] = self.counts[idx].saturating_add(1);
        }
    }

    fn remove<T: Hash>(&mut self, item: &T) {
        for i in 0..self.hash_count {
            let idx = self.hash_index(item, i);
            self.counts[idx] = self.counts[idx].saturating_sub(1);
        }
    }

    fn may_contain<T: Hash>(&self, item: &T) -> bool {
        for i in 0..self.hash_count {
            let idx = self.hash_index(item, i);
            if self.counts[idx] == 0 {
                return false;
            }
        }
        true
    }

    fn hash_index<T: Hash>(&self, item: &T, seed: usize) -> usize {
        let mut hasher = DefaultHasher::new();
        item.hash(&mut hasher);
        seed.hash(&mut hasher);
        (hasher.finish() as usize) % self.counts.len()
    }
}
```

**代价**：内存增加 8 倍（每位置 1 字节 vs 1 位）。

### 2. 无法获取元素

布隆过滤器只能告诉你"在不在"，不能告诉你"有哪些"。

```rust
// 不可能实现
fn list_all_elements() -> Vec<T> {
    // 做不到！
    // 布隆过滤器不存储原始元素
}
```

### 3. 误判率不可控

插入越多，误判率越高。

```rust
// 插入超过预期数量
let mut bloom = BloomFilter::new(1_000_000, 7);  // 设计容量 10 万

for i in 0..1_000_000 {  // 实际插入 100 万
    bloom.insert(&i);
}

// 误判率远超 1%
```

**解决方案**：监控实际误判率，必要时重建。

## 变体：Cuckoo Filter

布隆过滤器的改进版，支持删除。

```text
核心思想：
  每个元素存储在两个候选位置之一
  删除时可以精确移除

优点：
  - 支持删除
  - 空间效率更高（相同误判率下）
  - 查询更快（只查 2 个位置）

缺点：
  - 插入可能失败（表满时）
  - 实现更复杂
```

```rust
// 使用 cuckoofilter crate
use cuckoofilter::CuckooFilter;

let mut cf = CuckooFilter::new();

cf.add(&"hello").unwrap();
assert!(cf.contains(&"hello"));

cf.delete(&"hello");
assert!(!cf.contains(&"hello"));
```

## 最佳实践

### 1. 参数计算

```rust
fn calculate_bloom_params(expected_items: usize, fp_rate: f64) -> (usize, usize) {
    // m = -n * ln(p) / (ln(2))^2
    let m = -(expected_items as f64 * fp_rate.ln()) / (2_f64.ln().powi(2));
    let m = m.ceil() as usize;

    // k = (m/n) * ln(2)
    let k = (m as f64 / expected_items as f64) * 2_f64.ln();
    let k = k.round() as usize;

    (m, k)
}

// 1000 万元素，1% 误判
let (bits, hashes) = calculate_bloom_params(10_000_000, 0.01);
// bits ≈ 96,000,000 (12MB), hashes = 7
```

### 2. 预热

```rust
async fn warm_up_bloom_filter(bloom: &mut BloomFilter, db: &Database) {
    // 启动时从数据库加载所有有效 ID
    let mut cursor = db.scan_all_ids().await;

    while let Some(batch) = cursor.next_batch().await {
        for id in batch {
            bloom.insert(&id);
        }
    }

    println!("Bloom filter warmed up");
}
```

### 3. 持久化

```rust
impl BloomFilter {
    fn to_bytes(&self) -> Vec<u8> {
        // 序列化位数组
        self.bits.as_raw_slice().to_vec()
    }

    fn from_bytes(bytes: &[u8], hash_count: usize) -> Self {
        let bits = BitVec::from_slice(bytes);
        Self { bits, hash_count }
    }
}

// 定期持久化到文件/Redis
async fn persist_bloom_filter(bloom: &BloomFilter, redis: &Redis) {
    let bytes = bloom.to_bytes();
    redis.set("bloom:products", &bytes).await;
}
```

## 总结

| 特性 | 布隆过滤器 | HashSet |
|------|-----------|---------|
| 空间 | O(n) 位 | O(n) 元素 |
| 查询 | O(k) | O(1) |
| 假阳性 | 有（可控） | 无 |
| 假阴性 | 无 | 无 |
| 删除 | 不支持 | 支持 |

| 适用场景 | 不适用场景 |
|----------|------------|
| 缓存穿透防护 | 需要精确判断 |
| 黑名单快速过滤 | 需要删除元素 |
| 去重（允许误判） | 数据量小 |
| 爬虫 URL 去重 | 误判代价高 |

**核心认知**：

> 布隆过滤器用概率换空间，用误判换效率。在"先快速排除大多数，再精确验证少数"的场景下，它是最佳选择。

---

*上一篇：[哈希表：缓存设计的基石](/posts/algo-not-leetcode-3-hashtable/)*

*下一篇：[一致性哈希：分布式系统的路由表](/posts/algo-not-leetcode-5-consistent-hash/)*

*本系列：*
1. *[算法是业务武器](/posts/algo-not-leetcode-1-intro/)*
2. *[排序与二分：被低估的基础功](/posts/algo-not-leetcode-2-sort-binary-search/)*
3. *[哈希表：缓存设计的基石](/posts/algo-not-leetcode-3-hashtable/)*
4. *布隆过滤器：用 1% 误判换 90% 内存（本篇）*
5. *[一致性哈希：分布式系统的路由表](/posts/algo-not-leetcode-5-consistent-hash/)*
6. *[堆与优先队列：调度器的核心](/posts/algo-not-leetcode-6-heap/)*
7. *[限流算法：保护系统的三道防线](/posts/algo-not-leetcode-7-rate-limit/)*
8. *[跳表：Redis 排行榜的秘密](/posts/algo-not-leetcode-8-skiplist/)*
9. *[树结构：层级数据的优雅解法](/posts/algo-not-leetcode-9-tree/)*
10. *[字符串匹配：敏感词过滤的正确姿势](/posts/algo-not-leetcode-10-string-match/)*
11. *[实战选型：没有银弹，只有场景](/posts/algo-not-leetcode-11-selection/)*
