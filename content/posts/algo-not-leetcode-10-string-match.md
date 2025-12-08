---
title: "算法不是面试题（十）：字符串匹配——敏感词过滤的正确姿势"
date: 2025-12-08T11:30:00+08:00
draft: false
tags: ["算法", "字符串匹配", "AC自动机", "敏感词过滤", "Trie"]
categories: ["技术"]
description: "10000 个敏感词，一篇文章逐个检查？太慢了。AC 自动机让你一次扫描匹配所有敏感词，复杂度从 O(n×m) 降到 O(n)。"
series: ["算法不是面试题"]
---

10000 个敏感词，一篇文章逐个检查？太慢了。AC 自动机让你一次扫描匹配所有敏感词，复杂度从 O(n×m) 降到 O(n)。

<!--more-->

## 问题：敏感词过滤

社交平台、电商评论、直播弹幕——都需要敏感词过滤。

```text
敏感词库：["色情", "赌博", "毒品", "枪支", ...]  // 可能上万个
用户输入：一篇几千字的文章

需求：
  1. 检测是否包含敏感词
  2. 把敏感词替换成 ***
  3. 实时处理，延迟要低
```

### 朴素方案

```rust
fn contains_sensitive(text: &str, words: &[&str]) -> Vec<&str> {
    let mut found = Vec::new();
    for word in words {
        if text.contains(word) {
            found.push(*word);
        }
    }
    found
}
```

**复杂度**：O(n × m × k)

- n = 文本长度
- m = 敏感词数量
- k = 敏感词平均长度

10000 个敏感词 × 10000 字文章 × 5 字平均长度 = **5 亿次操作**

太慢了。

## Trie 树：多模式匹配的基础

### 核心思想

把所有敏感词构建成一棵前缀树。

```text
敏感词：["he", "she", "his", "hers"]

Trie 结构：
        root
       /    \
      h      s
     / \      \
    e   i      h
    |   |       \
   [he] s       e
        |        \
       [his]    [she]
                  \
                   r
                   |
                   s
                   |
                 [hers]

[] 表示单词结束
```

### Rust 实现

```rust
use std::collections::HashMap;

#[derive(Default)]
struct TrieNode {
    children: HashMap<char, TrieNode>,
    is_end: bool,
    word: Option<String>,
}

struct Trie {
    root: TrieNode,
}

impl Trie {
    fn new() -> Self {
        Self { root: TrieNode::default() }
    }

    fn insert(&mut self, word: &str) {
        let mut node = &mut self.root;
        for c in word.chars() {
            node = node.children.entry(c).or_default();
        }
        node.is_end = true;
        node.word = Some(word.to_string());
    }

    fn search(&self, text: &str) -> Vec<String> {
        let mut found = Vec::new();
        let chars: Vec<char> = text.chars().collect();

        // 从每个位置开始尝试匹配
        for i in 0..chars.len() {
            let mut node = &self.root;
            for j in i..chars.len() {
                match node.children.get(&chars[j]) {
                    Some(next) => {
                        node = next;
                        if node.is_end {
                            if let Some(word) = &node.word {
                                found.push(word.clone());
                            }
                        }
                    }
                    None => break,
                }
            }
        }

        found
    }
}
```

**复杂度**：O(n × k)，比朴素方案好，但还不够。

## AC 自动机：一次扫描匹配所有

### 核心思想

在 Trie 基础上，加上**失败指针**（fail pointer）。

```text
当匹配失败时，不回到起点，而是跳到"次优匹配位置"。

例如：
  正在匹配 "she"
  已匹配 "sh"，下一个字符是 "x"（不匹配 "e"）

  朴素：回到起点，从 "h" 重新开始
  AC 自动机：跳到 "h" 节点（因为 "sh" 以 "h" 结尾）

  这样就不会错过 "his"（如果后面是 "is"）
```

### 可视化

```text
Trie + 失败指针：

        root ←─────────────────┐
       /    \                  │
      h      s ─────fail────→ root
     / \      \
    e   i      h ──fail──→ h
    |   |       \
   [he] s       e ──fail──→ e([he])
        |        \
       [his]    [she]
                  \
                   r ──fail──→ root
                   |
                   s
                   |
                 [hers] ──fail──→ s

失败指针指向：当前路径的最长真后缀在 Trie 中的位置
```

### Rust 实现

```rust
use std::collections::{HashMap, VecDeque};

struct AcNode {
    children: HashMap<char, usize>,  // 子节点索引
    fail: usize,                      // 失败指针
    output: Vec<String>,              // 匹配的词（可能多个）
}

pub struct AhoCorasick {
    nodes: Vec<AcNode>,
}

impl AhoCorasick {
    pub fn new(patterns: &[&str]) -> Self {
        let mut ac = Self {
            nodes: vec![AcNode {
                children: HashMap::new(),
                fail: 0,
                output: Vec::new(),
            }],
        };

        // 构建 Trie
        for pattern in patterns {
            ac.insert(pattern);
        }

        // 构建失败指针
        ac.build_fail();

        ac
    }

    fn insert(&mut self, pattern: &str) {
        let mut current = 0;

        for c in pattern.chars() {
            let next = if let Some(&idx) = self.nodes[current].children.get(&c) {
                idx
            } else {
                let idx = self.nodes.len();
                self.nodes.push(AcNode {
                    children: HashMap::new(),
                    fail: 0,
                    output: Vec::new(),
                });
                self.nodes[current].children.insert(c, idx);
                idx
            };
            current = next;
        }

        self.nodes[current].output.push(pattern.to_string());
    }

    fn build_fail(&mut self) {
        let mut queue = VecDeque::new();

        // 第一层的失败指针都指向 root
        for &child_idx in self.nodes[0].children.values() {
            self.nodes[child_idx].fail = 0;
            queue.push_back(child_idx);
        }

        // BFS 构建
        while let Some(current) = queue.pop_front() {
            let children: Vec<(char, usize)> = self.nodes[current]
                .children
                .iter()
                .map(|(&c, &idx)| (c, idx))
                .collect();

            for (c, child_idx) in children {
                queue.push_back(child_idx);

                // 找失败指针
                let mut fail = self.nodes[current].fail;
                loop {
                    if let Some(&next) = self.nodes[fail].children.get(&c) {
                        self.nodes[child_idx].fail = next;
                        break;
                    }
                    if fail == 0 {
                        self.nodes[child_idx].fail = 0;
                        break;
                    }
                    fail = self.nodes[fail].fail;
                }

                // 合并输出
                let fail_idx = self.nodes[child_idx].fail;
                let fail_output = self.nodes[fail_idx].output.clone();
                self.nodes[child_idx].output.extend(fail_output);
            }
        }
    }

    pub fn search(&self, text: &str) -> Vec<Match> {
        let mut matches = Vec::new();
        let mut current = 0;

        for (pos, c) in text.char_indices() {
            // 沿着失败指针找到能匹配的节点
            loop {
                if let Some(&next) = self.nodes[current].children.get(&c) {
                    current = next;
                    break;
                }
                if current == 0 {
                    break;
                }
                current = self.nodes[current].fail;
            }

            // 收集匹配
            for word in &self.nodes[current].output {
                let word_byte_len = word.len();
                matches.push(Match {
                    word: word.clone(),
                    end: pos + c.len_utf8(),
                    start: pos + c.len_utf8() - word_byte_len,
                });
            }
        }

        matches
    }
}

pub struct Match {
    pub word: String,
    pub start: usize,
    pub end: usize,
}
```

### 复杂度

| 操作 | 复杂度 |
|------|--------|
| 构建 | O(Σ|pattern|) |
| 搜索 | O(|text| + |matches|) |

**搜索复杂度与模式数量无关！** 这就是 AC 自动机的威力。

## 实战：敏感词过滤服务

```rust
pub struct SensitiveFilter {
    ac: AhoCorasick,
}

impl SensitiveFilter {
    pub fn new(words: &[&str]) -> Self {
        Self {
            ac: AhoCorasick::new(words),
        }
    }

    // 检测是否包含敏感词
    pub fn contains(&self, text: &str) -> bool {
        !self.ac.search(text).is_empty()
    }

    // 获取所有敏感词
    pub fn find_all(&self, text: &str) -> Vec<String> {
        self.ac
            .search(text)
            .into_iter()
            .map(|m| m.word)
            .collect()
    }

    // 替换敏感词
    pub fn replace(&self, text: &str) -> String {
        let matches = self.ac.search(text);
        if matches.is_empty() {
            return text.to_string();
        }

        // 按位置排序
        let mut sorted_matches = matches;
        sorted_matches.sort_by_key(|m| m.start);

        let mut result = String::new();
        let mut last_end = 0;

        for m in sorted_matches {
            // 处理重叠
            if m.start < last_end {
                continue;
            }

            // 添加未替换的部分（使用字节索引切片）
            result.push_str(&text[last_end..m.start]);
            // 添加替换（按字符数计算星号数量）
            result.push_str(&"*".repeat(m.word.chars().count()));
            last_end = m.end;
        }

        // 添加剩余部分
        result.push_str(&text[last_end..]);
        result
    }
}
```

### 使用示例

```rust
fn main() {
    let words = ["色情", "赌博", "毒品", "枪支"];
    let filter = SensitiveFilter::new(&words);

    let text = "这里有色情内容和赌博信息";

    println!("包含敏感词: {}", filter.contains(text));
    println!("敏感词列表: {:?}", filter.find_all(text));
    println!("替换后: {}", filter.replace(text));
    // 输出：这里有**内容和**信息
}
```

## 进阶：处理变体

敏感词会有各种变体：

```text
原词：色情
变体：色 情、色．情、s色q情、色*情、se情
```

### 方案一：预处理

```rust
fn normalize(text: &str) -> String {
    text.chars()
        .filter(|c| c.is_alphanumeric() || c.is_cjk())
        .flat_map(|c| c.to_lowercase())
        .collect()
}

trait CharExt {
    fn is_cjk(&self) -> bool;
}

impl CharExt for char {
    fn is_cjk(&self) -> bool {
        matches!(*self as u32, 0x4E00..=0x9FFF)
    }
}

impl SensitiveFilter {
    pub fn contains_normalized(&self, text: &str) -> bool {
        let normalized = normalize(text);
        self.contains(&normalized)
    }
}
```

### 方案二：模糊匹配

在 AC 自动机中添加跳过字符的能力：

```rust
use std::collections::HashSet;

impl AhoCorasick {
    pub fn search_fuzzy(&self, text: &str, skip_chars: &HashSet<char>) -> Vec<Match> {
        let mut matches = Vec::new();
        let mut current = 0;

        let chars: Vec<(usize, char)> = text.char_indices().collect();
        let mut i = 0;

        while i < chars.len() {
            let (pos, c) = chars[i];

            // 跳过干扰字符
            if skip_chars.contains(&c) {
                i += 1;
                continue;
            }

            // 正常匹配逻辑...
            // ...

            i += 1;
        }

        matches
    }
}
```

### 方案三：DFA 最小化

对于超大词库，可以将 AC 自动机转换为 DFA 并最小化：

```rust
// 使用 aho-corasick crate，它内部已经做了优化
use aho_corasick::AhoCorasick;

let patterns = &["色情", "赌博", "毒品"];
let ac = AhoCorasick::new(patterns).unwrap();

let text = "这里有色情内容";
for mat in ac.find_iter(text) {
    println!("找到: {} 在位置 {}-{}", &text[mat.start()..mat.end()], mat.start(), mat.end());
}
```

## 性能优化

### 1. 使用数组代替 HashMap

```rust
// 如果字符集有限（如只有小写字母）
struct AcNodeFast {
    children: [Option<usize>; 26],  // a-z
    fail: usize,
    output: Vec<String>,
}

fn char_to_index(c: char) -> Option<usize> {
    if c.is_ascii_lowercase() {
        Some((c as u8 - b'a') as usize)
    } else {
        None
    }
}
```

### 2. 双数组 Trie

```rust
// Double-Array Trie，空间和时间都更优
// 使用 daachorse crate
use daachorse::DoubleArrayAhoCorasick;

let patterns = &["色情", "赌博"];
let pma = DoubleArrayAhoCorasick::new(patterns).unwrap();

for m in pma.find_iter("色情内容") {
    println!("匹配: {:?}", m);
}
```

### 3. 并行处理

```rust
use rayon::prelude::*;

fn batch_filter(texts: &[String], filter: &SensitiveFilter) -> Vec<String> {
    texts
        .par_iter()
        .map(|text| filter.replace(text))
        .collect()
}
```

## 其他字符串算法

### KMP：单模式匹配

```rust
fn kmp_search(text: &str, pattern: &str) -> Vec<usize> {
    let text: Vec<char> = text.chars().collect();
    let pattern: Vec<char> = pattern.chars().collect();

    if pattern.is_empty() {
        return vec![];
    }

    // 构建 next 数组
    let mut next = vec![0usize; pattern.len()];
    let mut j = 0;
    for i in 1..pattern.len() {
        while j > 0 && pattern[i] != pattern[j] {
            j = next[j - 1];
        }
        if pattern[i] == pattern[j] {
            j += 1;
        }
        next[i] = j;
    }

    // 匹配
    let mut matches = Vec::new();
    let mut j = 0;
    for i in 0..text.len() {
        while j > 0 && text[i] != pattern[j] {
            j = next[j - 1];
        }
        if text[i] == pattern[j] {
            j += 1;
        }
        if j == pattern.len() {
            matches.push(i + 1 - j);
            j = next[j - 1];
        }
    }

    matches
}
```

### Rabin-Karp：基于哈希的匹配

```rust
fn rabin_karp(text: &str, pattern: &str) -> Vec<usize> {
    let text: Vec<char> = text.chars().collect();
    let pattern: Vec<char> = pattern.chars().collect();

    if pattern.len() > text.len() {
        return vec![];
    }

    const BASE: u64 = 256;
    const MOD: u64 = 1_000_000_007;

    // 计算 pattern 的哈希
    let mut pattern_hash = 0u64;
    let mut text_hash = 0u64;
    let mut power = 1u64;

    for i in 0..pattern.len() {
        pattern_hash = (pattern_hash * BASE + pattern[i] as u64) % MOD;
        text_hash = (text_hash * BASE + text[i] as u64) % MOD;
        if i > 0 {
            power = (power * BASE) % MOD;
        }
    }

    let mut matches = Vec::new();

    for i in 0..=(text.len() - pattern.len()) {
        if text_hash == pattern_hash && &text[i..i + pattern.len()] == &pattern[..] {
            matches.push(i);
        }

        // 滚动哈希
        if i + pattern.len() < text.len() {
            text_hash = (text_hash + MOD - (text[i] as u64 * power) % MOD) % MOD;
            text_hash = (text_hash * BASE + text[i + pattern.len()] as u64) % MOD;
        }
    }

    matches
}
```

## 总结

| 算法 | 复杂度 | 适用场景 |
|------|--------|----------|
| 朴素匹配 | O(n×m×k) | 少量短模式 |
| KMP | O(n+m) | 单模式匹配 |
| Rabin-Karp | O(n+m) | 单模式、多模式 |
| Trie | O(n×k) | 多模式前缀查询 |
| AC 自动机 | O(n+输出) | 多模式精确匹配 |

**核心认知**：

> AC 自动机是多模式匹配的终极武器。它把"对每个模式检查一遍"变成"扫描一遍文本匹配所有模式"，是敏感词过滤、病毒扫描、入侵检测等场景的标准解法。

---

*上一篇：[树结构：层级数据的优雅解法](/posts/algo-not-leetcode-9-tree/)*

*下一篇：[实战选型：没有银弹，只有场景](/posts/algo-not-leetcode-11-selection/)*

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
10. *字符串匹配：敏感词过滤的正确姿势（本篇）*
11. *[实战选型：没有银弹，只有场景](/posts/algo-not-leetcode-11-selection/)*
