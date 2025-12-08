---
title: "算法不是面试题（三）：哈希表——缓存设计的基石"
date: 2025-12-08T10:20:00+08:00
draft: false
tags: ["算法", "哈希表", "缓存", "LRU", "幂等"]
categories: ["技术"]
description: "HashMap 谁不会用？但你知道它背后的哈希函数、冲突处理、扩容机制吗？这些细节决定了你的缓存是高效还是灾难。"
series: ["算法不是面试题"]
---

HashMap 谁不会用？但你知道它背后的哈希函数、冲突处理、扩容机制吗？这些细节决定了你的缓存是高效还是灾难。

<!--more-->

## 哈希表的本质

哈希表做的事情很简单：**把 key 映射到数组下标**。

```text
key → hash(key) → index → bucket[index]

例如：
  "user:123" → hash("user:123") → 42 → bucket[42]
```

**为什么快**：数组按下标访问是 O(1)。

**问题**：不同的 key 可能映射到同一个下标（哈希冲突）。

## 哈希冲突的处理

### 方案一：链地址法

每个桶是一个链表，冲突的元素挂在链表上。

```text
bucket[0] → null
bucket[1] → (key1, val1) → (key2, val2) → null
bucket[2] → (key3, val3) → null
...
```

```rust
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

struct HashMapChain<K, V> {
    buckets: Vec<Vec<(K, V)>>,
    len: usize,
}

impl<K: Hash + Eq, V> HashMapChain<K, V> {
    fn get(&self, key: &K) -> Option<&V> {
        let idx = self.bucket_index(key);
        self.buckets[idx]
            .iter()
            .find(|(k, _)| k == key)
            .map(|(_, v)| v)
    }

    fn bucket_index(&self, key: &K) -> usize {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        (hasher.finish() as usize) % self.buckets.len()
    }
}
```

**优点**：实现简单，删除方便。

**缺点**：链表对缓存不友好，极端情况退化为 O(n)。

### 方案二：开放寻址法

冲突时，在数组中找下一个空位。

```text
插入 key1 → hash = 5 → bucket[5] 空 → 放入
插入 key2 → hash = 5 → bucket[5] 满 → 尝试 bucket[6] → 放入
```

```rust
use std::hash::Hash;

struct HashMapOpen<K, V> {
    buckets: Vec<Option<(K, V)>>,
    len: usize,
}

impl<K: Hash + Eq, V> HashMapOpen<K, V> {
    fn get(&self, key: &K) -> Option<&V> {
        let start = self.bucket_index(key);
        let capacity = self.buckets.len();

        for i in 0..capacity {
            let idx = (start + i) % capacity;
            match &self.buckets[idx] {
                Some((k, v)) if k == key => return Some(v),
                None => return None,  // 找到空位说明不存在
                _ => continue,        // 继续探测
            }
        }
        None
    }
}
```

**优点**：缓存友好，内存紧凑。

**缺点**：删除复杂（需要墓碑标记），负载因子不能太高。

### Rust 的 HashMap

Rust 标准库的 HashMap 使用 **SwissTable**（Google 发明）：

```text
SwissTable 的核心：
  1. 每个桶存储一个 1 字节的元数据（hash 的高 7 位 + 状态位）
  2. 元数据连续存储，可以用 SIMD 并行比较
  3. 一次比较 16 个桶的元数据

查找过程：
  1. 计算 hash
  2. 用 SIMD 比较 16 个元数据
  3. 找到匹配的再比较完整 key
```

**结果**：比传统开放寻址快 2-3 倍。

## 场景一：本地缓存

最简单的缓存：用 HashMap 存键值对。

```rust
use std::collections::HashMap;
use std::hash::Hash;
use std::time::{Duration, Instant};

struct CacheEntry<V> {
    value: V,
    expires_at: Instant,
}

struct LocalCache<K, V> {
    data: HashMap<K, CacheEntry<V>>,
    default_ttl: Duration,
}

impl<K: Hash + Eq, V> LocalCache<K, V> {
    fn get(&self, key: &K) -> Option<&V> {
        self.data.get(key).and_then(|entry| {
            if Instant::now() < entry.expires_at {
                Some(&entry.value)
            } else {
                None  // 已过期
            }
        })
    }

    fn set(&mut self, key: K, value: V) {
        let entry = CacheEntry {
            value,
            expires_at: Instant::now() + self.default_ttl,
        };
        self.data.insert(key, entry);
    }
}
```

**问题**：过期的 entry 不会自动清理，内存会一直涨。

### 惰性清理

```rust
impl<K: Hash + Eq + Clone, V> LocalCache<K, V> {
    fn get_and_cleanup(&mut self, key: &K) -> Option<&V> {
        // 先检查是否过期
        let expired = self.data.get(key)
            .map(|e| Instant::now() >= e.expires_at)
            .unwrap_or(false);

        if expired {
            self.data.remove(key);
            return None;
        }

        self.data.get(key).map(|e| &e.value)
    }

    // 定期清理过期 entry
    fn cleanup(&mut self) {
        let now = Instant::now();
        self.data.retain(|_, entry| entry.expires_at > now);
    }
}
```

**工程实践**：后台线程定期调用 `cleanup()`，或者在 `set()` 时随机清理几个 entry。

## 场景二：LRU 缓存

缓存满了怎么办？淘汰**最久未使用**的。

```text
LRU（Least Recently Used）的操作：
  - get(key)：O(1) 获取，并把 key 移到"最近使用"
  - put(key, val)：O(1) 插入，满了就淘汰"最久未使用"
```

**核心数据结构**：HashMap + 双向链表。

```rust
use std::collections::HashMap;
use std::ptr::NonNull;

struct Node<K, V> {
    key: K,
    value: V,
    prev: Option<NonNull<Node<K, V>>>,
    next: Option<NonNull<Node<K, V>>>,
}

pub struct LRUCache<K, V> {
    map: HashMap<K, NonNull<Node<K, V>>>,
    head: Option<NonNull<Node<K, V>>>,  // 最近使用
    tail: Option<NonNull<Node<K, V>>>,  // 最久未使用
    capacity: usize,
}
```

```text
链表结构：
  head ←→ node1 ←→ node2 ←→ node3 ←→ tail
  (最新)                           (最旧)

get(key):
  1. HashMap 找到 node
  2. 把 node 移到 head

put(key, val):
  1. 如果 key 存在，更新值，移到 head
  2. 如果不存在：
     a. 如果满了，删除 tail，从 HashMap 删除
     b. 创建新 node，插入 head，加入 HashMap
```

**为什么是 O(1)**：

| 操作 | HashMap | 链表 | 总计 |
|------|---------|------|------|
| 查找 | O(1) | - | O(1) |
| 移动到头部 | - | O(1) | O(1) |
| 删除尾部 | O(1) | O(1) | O(1) |
| 插入头部 | O(1) | O(1) | O(1) |

### 生产级实现

实际项目中，直接用 `lru` crate：

```rust
use lru::LruCache;
use std::num::NonZeroUsize;

let mut cache = LruCache::new(NonZeroUsize::new(100).unwrap());

cache.put("key1", "value1");
cache.put("key2", "value2");

assert_eq!(cache.get(&"key1"), Some(&"value1"));
```

**注意**：标准库没有 LRU，因为安全的 Rust 实现双向链表很麻烦（需要 unsafe 或 Rc/RefCell）。

## 场景三：幂等控制

支付系统的核心需求：**同一笔请求只能成功一次**。

```text
场景：
  用户点击支付 → 网络超时 → 用户重试 → 又点了一次

如果没有幂等控制：
  可能扣款 3 次
```

### 基于哈希表的幂等

```rust
use std::collections::HashMap;
use std::time::{Duration, Instant};

enum IdempotentResult<T> {
    New,                    // 新请求
    Duplicate(T),           // 重复请求，返回之前的结果
}

struct IdempotentStore<T> {
    results: HashMap<String, (T, Instant)>,
    ttl: Duration,
}

impl<T: Clone> IdempotentStore<T> {
    fn check_or_create(&mut self, key: &str) -> IdempotentResult<T> {
        let now = Instant::now();

        // 检查是否存在且未过期
        if let Some((result, created_at)) = self.results.get(key) {
            if now.duration_since(*created_at) < self.ttl {
                return IdempotentResult::Duplicate(result.clone());
            }
        }

        IdempotentResult::New
    }

    fn save_result(&mut self, key: String, result: T) {
        self.results.insert(key, (result, Instant::now()));
    }
}
```

### 使用方式

```rust
async fn process_payment(req: PaymentRequest) -> Result<PaymentResult> {
    let idempotent_key = format!("pay:{}:{}", req.user_id, req.order_id);

    // 检查幂等
    match store.check_or_create(&idempotent_key) {
        IdempotentResult::Duplicate(result) => {
            return Ok(result);  // 直接返回之前的结果
        }
        IdempotentResult::New => {
            // 继续处理
        }
    }

    // 执行实际的支付逻辑
    let result = do_payment(&req).await?;

    // 保存结果
    store.save_result(idempotent_key, result.clone());

    Ok(result)
}
```

**分布式场景**：单机 HashMap 不够用，需要 Redis：

```rust
async fn check_idempotent_redis(key: &str, ttl_seconds: u64) -> Result<bool> {
    // SET key 1 NX EX ttl
    // NX = 只在 key 不存在时设置
    // 返回 true 表示是新请求
    let result: Option<String> = redis.set_nx_ex(key, "1", ttl_seconds).await?;
    Ok(result.is_some())
}
```

## 哈希函数的选择

不同场景需要不同的哈希函数。

### 1. 通用哈希（HashMap 默认）

```rust
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

fn hash_it<T: Hash>(value: &T) -> u64 {
    let mut hasher = DefaultHasher::new();
    value.hash(&mut hasher);
    hasher.finish()
}
```

Rust 默认用 SipHash-1-3，特点是**抗 HashDoS 攻击**。

### 2. 快速哈希（性能优先）

```rust
use rustc_hash::FxHashMap;

// FxHash 比 SipHash 快 2-3 倍
let mut map: FxHashMap<String, i32> = FxHashMap::default();
```

**适用场景**：key 是可信的（不来自用户输入）。

### 3. 一致性哈希（分布式）

后面会专门讲，这里先跳过。

### HashDoS 攻击

```text
攻击原理：
  1. 攻击者构造大量哈希冲突的 key
  2. HashMap 退化为链表，O(n) 查找
  3. 服务器 CPU 被打满

防御：
  1. 用带随机种子的哈希函数（SipHash）
  2. 限制单个桶的长度
  3. 负载均衡 + 限流
```

Rust 的 HashMap 默认就是防 HashDoS 的，但如果你换成 FxHash，要确保 key 来源可信。

## 扩容机制

HashMap 不能无限往一个桶里塞，满了要扩容。

```text
负载因子 = 元素数量 / 桶数量

Rust HashMap 的默认策略：
  - 负载因子 > 0.875 时扩容
  - 扩容为原来的 2 倍
  - 所有元素重新哈希（rehash）
```

### 扩容的代价

```rust
// 假设有 100 万元素
let mut map = HashMap::new();
for i in 0..1_000_000 {
    map.insert(i, i);  // 可能触发多次扩容
}
```

```text
扩容过程：
  1. 分配新数组（2 倍大小）
  2. 遍历旧数组，重新计算每个 key 的位置
  3. 插入新数组
  4. 释放旧数组

100 万元素的 rehash 可能要几十毫秒
```

### 预分配容量

```rust
// 如果知道大概要存多少元素，预分配
let mut map = HashMap::with_capacity(1_000_000);
for i in 0..1_000_000 {
    map.insert(i, i);  // 不会触发扩容
}
```

**性能对比**：

| 方式 | 时间 |
|------|------|
| 默认容量 | ~200ms |
| 预分配 | ~80ms |

**提升 2.5 倍**。

## 常见陷阱

### 1. 可变 key

```rust
// 错误：用可变内容作为 key
let mut key = String::from("user:123");
map.insert(key.clone(), "value");

key.push_str(":extra");  // key 变了！
// 但 map 里的 key 还是旧的，get 不到
```

### 2. 浮点数作为 key

```rust
// 错误：f64 不能作为 HashMap 的 key
// let map: HashMap<f64, String> = HashMap::new();  // 编译错误

// 原因：NaN != NaN，违反 Eq 的约定
```

### 3. 自定义类型忘记实现 Hash

```rust
#[derive(Hash, Eq, PartialEq)]  // 三个都要
struct UserId(u64);

let mut map: HashMap<UserId, String> = HashMap::new();
```

## 总结

| 场景 | 方案 | 要点 |
|------|------|------|
| 本地缓存 | HashMap + TTL | 惰性清理/定期清理 |
| 有容量限制 | LRU Cache | HashMap + 双向链表 |
| 幂等控制 | HashMap/Redis | 设置过期时间 |
| 高性能 | FxHashMap | 确保 key 可信 |
| 大数据量 | 预分配容量 | 避免 rehash |

**核心认知**：

> 哈希表是最重要的数据结构之一，O(1) 的查找让它成为缓存、索引、去重的首选。但要用好它，你需要理解哈希函数、冲突处理、扩容机制这些底层细节。

---

*上一篇：[排序与二分：被低估的基础功](/posts/algo-not-leetcode-2-sort-binary-search/)*

*下一篇：[布隆过滤器：用 1% 误判换 90% 内存](/posts/algo-not-leetcode-4-bloom-filter/)*

*本系列：*
1. *[算法是业务武器](/posts/algo-not-leetcode-1-intro/)*
2. *[排序与二分：被低估的基础功](/posts/algo-not-leetcode-2-sort-binary-search/)*
3. *哈希表：缓存设计的基石（本篇）*
4. *[布隆过滤器：用 1% 误判换 90% 内存](/posts/algo-not-leetcode-4-bloom-filter/)*
5. *[一致性哈希：分布式系统的路由表](/posts/algo-not-leetcode-5-consistent-hash/)*
6. *[堆与优先队列：调度器的核心](/posts/algo-not-leetcode-6-heap/)*
7. *[限流算法：保护系统的三道防线](/posts/algo-not-leetcode-7-rate-limit/)*
8. *[跳表：Redis 排行榜的秘密](/posts/algo-not-leetcode-8-skiplist/)*
9. *[树结构：层级数据的优雅解法](/posts/algo-not-leetcode-9-tree/)*
10. *[字符串匹配：敏感词过滤的正确姿势](/posts/algo-not-leetcode-10-string-match/)*
11. *[实战选型：没有银弹，只有场景](/posts/algo-not-leetcode-11-selection/)*
