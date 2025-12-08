---
title: "算法不是面试题（二）：排序与二分——被低估的基础功"
date: 2025-12-08T10:10:00+08:00
draft: false
tags: ["算法", "排序", "二分查找", "版本回滚"]
categories: ["技术"]
description: "排序和二分查找，入门级算法？别小看它们。版本回滚、数据对账、范围查询——这些高频业务场景，背后都是它们。"
series: ["算法不是面试题"]
---

排序和二分查找，入门级算法？别小看它们。版本回滚、数据对账、范围查询——这些高频业务场景，背后都是它们。

<!--more-->

## 场景一：版本回滚

你负责一个配置中心，需求很简单：

```text
1. 保存配置的历史版本
2. 支持回滚到任意历史版本
3. 查询某个时间点的配置
```

### 朴素实现

```rust
struct ConfigVersion {
    version: u64,
    timestamp: i64,
    content: String,
}

struct ConfigStore {
    versions: Vec<ConfigVersion>,
}

impl ConfigStore {
    // 查找某个时间点的配置
    fn get_at(&self, timestamp: i64) -> Option<&ConfigVersion> {
        // 遍历找到最后一个 <= timestamp 的版本
        let mut result = None;
        for v in &self.versions {
            if v.timestamp <= timestamp {
                result = Some(v);
            } else {
                break;
            }
        }
        result
    }
}
```

**问题**：每次查询都要遍历，O(n) 复杂度。

如果有 10000 个版本，每秒 1000 次查询，就是 1000 万次比较。

### 二分查找优化

版本是按时间顺序插入的，天然有序。有序数组 + 查找 = 二分。

```rust
impl ConfigStore {
    fn get_at(&self, timestamp: i64) -> Option<&ConfigVersion> {
        if self.versions.is_empty() {
            return None;
        }

        // 二分查找：找到最后一个 <= timestamp 的版本
        let mut left = 0;
        let mut right = self.versions.len();

        while left < right {
            let mid = left + (right - left) / 2;
            if self.versions[mid].timestamp <= timestamp {
                left = mid + 1;
            } else {
                right = mid;
            }
        }

        // left 是第一个 > timestamp 的位置
        // 所以 left - 1 是最后一个 <= timestamp 的位置
        if left > 0 {
            Some(&self.versions[left - 1])
        } else {
            None
        }
    }
}
```

**复杂度对比**：

| 方案 | 时间复杂度 | 10000 版本查询 |
|------|-----------|---------------|
| 遍历 | O(n) | ~10000 次比较 |
| 二分 | O(log n) | ~14 次比较 |

**提升 700 倍**，代码改动不到 20 行。

### 二分的变体

二分查找有多种变体，对应不同的业务需求：

```text
1. 找等于 target 的元素（精确匹配）
   场景：根据版本号查配置

2. 找第一个 >= target 的元素（下界）
   场景：找到生效的第一个版本

3. 找最后一个 <= target 的元素（上界）
   场景：找到某时间点的配置

4. 找第一个 > target 的元素
   场景：找到下一个版本
```

```rust
// 标准库提供了 partition_point，可以简化实现
impl ConfigStore {
    fn get_at(&self, timestamp: i64) -> Option<&ConfigVersion> {
        let idx = self.versions.partition_point(|v| v.timestamp <= timestamp);
        if idx > 0 {
            Some(&self.versions[idx - 1])
        } else {
            None
        }
    }
}
```

## 场景二：数据对账

金融系统常见需求：对比两个系统的数据差异。

```text
系统 A：100 万条交易记录
系统 B：100 万条交易记录

找出：
  - A 有 B 没有的
  - B 有 A 没有的
  - 两边都有但金额不一致的
```

### 朴素实现

```rust
fn reconcile_naive(a: &[Transaction], b: &[Transaction]) -> Vec<Difference> {
    let mut diffs = Vec::new();

    for tx_a in a {
        // 在 B 中查找
        let found = b.iter().find(|tx_b| tx_b.id == tx_a.id);
        match found {
            None => diffs.push(Difference::OnlyInA(tx_a.clone())),
            Some(tx_b) if tx_a.amount != tx_b.amount => {
                diffs.push(Difference::AmountMismatch(tx_a.clone(), tx_b.clone()));
            }
            _ => {}
        }
    }

    // 还要找 B 有 A 没有的...
    for tx_b in b {
        if !a.iter().any(|tx_a| tx_a.id == tx_b.id) {
            diffs.push(Difference::OnlyInB(tx_b.clone()));
        }
    }

    diffs
}
```

**时间复杂度**：O(n × m)

100 万 × 100 万 = 1 万亿次比较。**跑到天荒地老**。

### 排序 + 双指针

```rust
use std::cmp::Ordering;

fn reconcile_sorted(a: &mut [Transaction], b: &mut [Transaction]) -> Vec<Difference> {
    // 先排序
    a.sort_by_key(|tx| tx.id);
    b.sort_by_key(|tx| tx.id);

    let mut diffs = Vec::new();
    let mut i = 0;
    let mut j = 0;

    while i < a.len() && j < b.len() {
        match a[i].id.cmp(&b[j].id) {
            Ordering::Less => {
                // A 有 B 没有
                diffs.push(Difference::OnlyInA(a[i].clone()));
                i += 1;
            }
            Ordering::Greater => {
                // B 有 A 没有
                diffs.push(Difference::OnlyInB(b[j].clone()));
                j += 1;
            }
            Ordering::Equal => {
                // 两边都有，检查金额
                if a[i].amount != b[j].amount {
                    diffs.push(Difference::AmountMismatch(a[i].clone(), b[j].clone()));
                }
                i += 1;
                j += 1;
            }
        }
    }

    // 处理剩余
    while i < a.len() {
        diffs.push(Difference::OnlyInA(a[i].clone()));
        i += 1;
    }
    while j < b.len() {
        diffs.push(Difference::OnlyInB(b[j].clone()));
        j += 1;
    }

    diffs
}
```

**时间复杂度**：O(n log n + m log m + n + m) ≈ O(n log n)

| 方案 | 复杂度 | 100 万数据 |
|------|--------|-----------|
| 朴素 | O(n²) | ~1 万亿次 |
| 排序+双指针 | O(n log n) | ~2000 万次 |

**提升 5 万倍**。

### 为什么不用 HashSet？

```rust
use std::collections::HashSet;

fn reconcile_hashset(a: &[Transaction], b: &[Transaction]) -> Vec<Difference> {
    let set_a: HashSet<_> = a.iter().map(|tx| tx.id).collect();
    let set_b: HashSet<_> = b.iter().map(|tx| tx.id).collect();

    // 找差集
    let only_in_a: Vec<_> = set_a.difference(&set_b).collect();
    let only_in_b: Vec<_> = set_b.difference(&set_a).collect();
    // ...
}
```

HashSet 方案也是 O(n)，为什么还要排序？

```text
排序方案的优势：

1. 内存效率
   - HashSet 需要额外存储哈希桶
   - 排序是原地操作（或近似原地）

2. 缓存友好
   - 排序后的数组顺序访问，缓存命中率高
   - HashSet 随机访问，缓存不友好

3. 结果有序
   - 对账结果按 ID 排序，便于人工核查
   - HashSet 结果无序

4. 可分片处理
   - 排序后可以分段并行处理
   - 适合超大数据量的分布式对账
```

**实际经验**：数据量较小时（通常百万级以下），HashSet 因常数因子小而更快；数据量大且内存敏感时，排序 + 双指针更稳定。具体分界点取决于数据特征和硬件环境，建议实测。

## 排序算法的选择

不同场景需要不同的排序算法。

### 1. 小数据量：插入排序

```rust
fn insertion_sort<T: Ord>(arr: &mut [T]) {
    for i in 1..arr.len() {
        let mut j = i;
        while j > 0 && arr[j - 1] > arr[j] {
            arr.swap(j - 1, j);
            j -= 1;
        }
    }
}
```

**适用场景**：n < 50，或者数据几乎有序。

**为什么**：常数因子小，对缓存友好，几乎有序时接近 O(n)。

### 2. 通用场景：快速排序 / 归并排序

```rust
// Rust 的 sort_unstable 用的是 pdqsort（改进的快排）
arr.sort_unstable();

// Rust 的 sort 用的是 timsort（改进的归并）
arr.sort();
```

| 算法 | 平均 | 最坏 | 稳定性 | 内存 |
|------|------|------|--------|------|
| 快排 | O(n log n) | O(n²) | 不稳定 | O(log n) |
| 归并 | O(n log n) | O(n log n) | 稳定 | O(n) |
| Timsort | O(n log n) | O(n log n) | 稳定 | O(n) |

**稳定性**：相等元素的相对顺序是否保持。

```text
场景：先按金额排序，再按时间排序

如果排序不稳定：
  第二次排序可能打乱第一次的结果

如果排序稳定：
  时间相同的记录，金额顺序保持
```

### 3. 特殊场景：基数排序 / 计数排序

```rust
// 计数排序：适用于值域小的整数
fn counting_sort(arr: &mut [u32], max_val: usize) {
    let mut count = vec![0usize; max_val + 1];

    for &x in arr.iter() {
        count[x as usize] += 1;
    }

    let mut idx = 0;
    for (val, &cnt) in count.iter().enumerate() {
        for _ in 0..cnt {
            arr[idx] = val as u32;
            idx += 1;
        }
    }
}
```

**适用场景**：值域有限的整数排序，如年龄（0-150）、分数（0-100）。

**时间复杂度**：O(n + k)，k 是值域大小。

## 实战：范围查询优化

场景：订单系统，按金额范围查询。

```rust
// 原始方案：每次查询都遍历
fn query_by_amount_range(orders: &[Order], min: f64, max: f64) -> Vec<&Order> {
    orders.iter().filter(|o| o.amount >= min && o.amount <= max).collect()
}
```

**优化方案**：维护一个按金额排序的索引。

```rust
struct OrderIndex {
    // 按金额排序的订单引用
    by_amount: Vec<usize>,  // 存储原数组的索引
}

impl OrderIndex {
    fn new(orders: &[Order]) -> Self {
        let mut by_amount: Vec<usize> = (0..orders.len()).collect();
        by_amount.sort_by(|&a, &b| {
            orders[a].amount.partial_cmp(&orders[b].amount).unwrap()
        });
        Self { by_amount }
    }

    fn query_range(&self, orders: &[Order], min: f64, max: f64) -> Vec<&Order> {
        // 二分找到起始位置
        let start = self.by_amount.partition_point(|&idx| orders[idx].amount < min);

        // 二分找到结束位置
        let end = self.by_amount.partition_point(|&idx| orders[idx].amount <= max);

        // 返回范围内的订单
        self.by_amount[start..end]
            .iter()
            .map(|&idx| &orders[idx])
            .collect()
    }
}
```

**复杂度**：

| 操作 | 原始方案 | 索引方案 |
|------|----------|----------|
| 构建 | - | O(n log n) |
| 查询 | O(n) | O(log n + k) |

k 是结果数量。如果频繁查询，索引方案完胜。

## 常见陷阱

### 1. 二分边界错误

```rust
// 错误：可能死循环
while left <= right {
    let mid = (left + right) / 2;  // 溢出风险！
    if arr[mid] < target {
        left = mid;  // 错误！应该是 mid + 1
    } else {
        right = mid;
    }
}

// 正确
while left < right {
    let mid = left + (right - left) / 2;  // 防溢出
    if arr[mid] < target {
        left = mid + 1;
    } else {
        right = mid;
    }
}
```

### 2. 排序稳定性

```rust
// 需要稳定排序时，用 sort() 而不是 sort_unstable()
orders.sort_by_key(|o| o.timestamp);  // 稳定
orders.sort_unstable_by_key(|o| o.timestamp);  // 不稳定，但更快
```

### 3. 浮点数比较

```rust
// 错误：浮点数不能直接用 sort()
let mut prices: Vec<f64> = vec![1.1, 2.2, 3.3];
// prices.sort();  // 编译错误！f64 不实现 Ord

// 正确
prices.sort_by(|a, b| a.partial_cmp(b).unwrap());
```

## 总结

| 场景 | 算法 | 复杂度 |
|------|------|--------|
| 有序数据查找 | 二分查找 | O(log n) |
| 数据对账 | 排序 + 双指针 | O(n log n) |
| 范围查询 | 排序索引 + 二分 | O(log n + k) |
| 小数据排序 | 插入排序 | O(n²) |
| 通用排序 | 快排/归并 | O(n log n) |

**核心认知**：

> 排序和二分是"基础设施"级别的算法。它们简单到你可能忽视，但强大到几乎所有复杂算法都依赖它们。

---

*上一篇：[算法是业务武器](/posts/algo-not-leetcode-1-intro/)*

*下一篇：[哈希表：缓存设计的基石](/posts/algo-not-leetcode-3-hashtable/)*

*本系列：*
1. *[算法是业务武器](/posts/algo-not-leetcode-1-intro/)*
2. *排序与二分：被低估的基础功（本篇）*
3. *[哈希表：缓存设计的基石](/posts/algo-not-leetcode-3-hashtable/)*
4. *[布隆过滤器：用 1% 误判换 90% 内存](/posts/algo-not-leetcode-4-bloom-filter/)*
5. *[一致性哈希：分布式系统的路由表](/posts/algo-not-leetcode-5-consistent-hash/)*
6. *[堆与优先队列：调度器的核心](/posts/algo-not-leetcode-6-heap/)*
7. *[限流算法：保护系统的三道防线](/posts/algo-not-leetcode-7-rate-limit/)*
8. *[跳表：Redis 排行榜的秘密](/posts/algo-not-leetcode-8-skiplist/)*
9. *[树结构：层级数据的优雅解法](/posts/algo-not-leetcode-9-tree/)*
10. *[字符串匹配：敏感词过滤的正确姿势](/posts/algo-not-leetcode-10-string-match/)*
11. *[实战选型：没有银弹，只有场景](/posts/algo-not-leetcode-11-selection/)*
