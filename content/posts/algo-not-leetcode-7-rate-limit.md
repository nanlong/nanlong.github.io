---
title: "算法不是面试题（七）：限流算法——保护系统的三道防线"
date: 2025-12-08T11:00:00+08:00
draft: false
tags: ["算法", "限流", "令牌桶", "漏桶", "滑动窗口"]
categories: ["技术"]
description: "秒杀活动，10 万人同时涌入。不限流？服务器直接躺平。限流算法是系统的最后一道防线：令牌桶、漏桶、滑动窗口，各有各的用法。"
series: ["算法不是面试题"]
---

秒杀活动，10 万人同时涌入。不限流？服务器直接躺平。限流算法是系统的最后一道防线：令牌桶、漏桶、滑动窗口，各有各的用法。

<!--more-->

## 为什么需要限流？

```text
场景：
  - 系统容量：1000 QPS
  - 正常流量：800 QPS
  - 突发流量：10000 QPS（促销/攻击）

不限流：
  10000 请求涌入 → 服务器过载 → 响应变慢 → 超时 → 重试 → 更多请求 → 雪崩

限流后：
  前 1000 请求正常处理
  剩余 9000 请求立即拒绝（返回 429 Too Many Requests）
  系统保持健康
```

**限流的核心价值**：宁可丢弃部分请求，也不要让系统崩溃。

## 算法一：固定窗口

最简单的限流：每秒只允许 N 个请求。

```rust
use std::time::{Duration, Instant};

struct FixedWindowLimiter {
    max_requests: usize,
    window_size: Duration,
    window_start: Instant,
    current_count: usize,
}

impl FixedWindowLimiter {
    fn new(max_requests: usize, window_size: Duration) -> Self {
        Self {
            max_requests,
            window_size,
            window_start: Instant::now(),
            current_count: 0,
        }
    }

    fn allow(&mut self) -> bool {
        let now = Instant::now();

        // 新窗口
        if now.duration_since(self.window_start) >= self.window_size {
            self.window_start = now;
            self.current_count = 0;
        }

        if self.current_count < self.max_requests {
            self.current_count += 1;
            true
        } else {
            false
        }
    }
}
```

### 问题：临界点突发

```text
限制：每秒 100 个请求

时间线：
  0:00.900 - 0:00.999：100 个请求（允许）
  0:01.000 - 0:01.100：100 个请求（新窗口，允许）

在 200ms 内通过了 200 个请求！
超过了"每秒 100"的预期
```

## 算法二：滑动窗口

把固定窗口切成更小的格子，滑动计算。

```text
固定窗口：|-------- 1秒 --------|-------- 1秒 --------|
          [      100 个      ]  [      100 个      ]

滑动窗口：把 1 秒分成 10 个格子，每格 100ms
          [10][10][10][10][10][10][10][10][10][10]

任何连续 10 个格子的总和 <= 100
```

```rust
struct SlidingWindowLimiter {
    max_requests: usize,
    window_size: Duration,
    slot_count: usize,
    slots: Vec<usize>,
    current_slot: usize,
    last_update: Instant,
}

impl SlidingWindowLimiter {
    fn new(max_requests: usize, window_size: Duration, slot_count: usize) -> Self {
        Self {
            max_requests,
            window_size,
            slot_count,
            slots: vec![0; slot_count],
            current_slot: 0,
            last_update: Instant::now(),
        }
    }

    fn allow(&mut self) -> bool {
        self.advance_time();

        let total: usize = self.slots.iter().sum();
        if total < self.max_requests {
            self.slots[self.current_slot] += 1;
            true
        } else {
            false
        }
    }

    fn advance_time(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_update);
        let slot_duration = self.window_size / self.slot_count as u32;

        let slots_to_advance = (elapsed.as_millis() / slot_duration.as_millis()) as usize;

        for _ in 0..slots_to_advance.min(self.slot_count) {
            self.current_slot = (self.current_slot + 1) % self.slot_count;
            self.slots[self.current_slot] = 0;
        }

        if slots_to_advance > 0 {
            self.last_update = now;
        }
    }
}
```

**优点**：比固定窗口更平滑。

**缺点**：需要存储多个格子，内存占用更大。

## 算法三：漏桶

请求像水一样流入桶中，以固定速率流出。

```text
漏桶模型：
        请求 ↓
       ┌─────┐
       │     │ ← 桶（缓冲队列）
       │█████│
       │█████│
       └──┬──┘
          │ ← 固定速率流出
          ↓
        处理
```

```rust
use std::collections::VecDeque;
use std::time::{Duration, Instant};

struct Request {
    id: String,
    data: Vec<u8>,
}

struct LeakyBucket {
    capacity: usize,           // 桶容量
    rate: f64,                 // 每秒流出数量
    queue: VecDeque<Request>,  // 请求队列
    last_leak: Instant,        // 上次漏水时间
}

impl LeakyBucket {
    fn new(capacity: usize, rate: f64) -> Self {
        Self {
            capacity,
            rate,
            queue: VecDeque::new(),
            last_leak: Instant::now(),
        }
    }

    fn allow(&mut self, request: Request) -> bool {
        self.leak();

        if self.queue.len() < self.capacity {
            self.queue.push_back(request);
            true
        } else {
            false  // 桶满，拒绝
        }
    }

    fn leak(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_leak).as_secs_f64();
        let to_leak = (elapsed * self.rate) as usize;

        for _ in 0..to_leak.min(self.queue.len()) {
            self.queue.pop_front();
        }

        if to_leak > 0 {
            self.last_leak = now;
        }
    }

    fn process_next(&mut self) -> Option<Request> {
        self.leak();
        self.queue.pop_front()
    }
}
```

**特点**：

| 优点 | 缺点 |
|------|------|
| 输出速率恒定 | 不能处理突发 |
| 平滑流量 | 可能增加延迟 |
| 保护下游 | 桶满就丢弃 |

**适用场景**：需要严格控制处理速率的场景，如发送短信、调用第三方 API。

## 算法四：令牌桶

定期往桶里放令牌，每个请求消耗一个令牌。

```text
令牌桶模型：
    令牌生成 ↓
       ┌─────┐
       │ ○○○ │ ← 桶（存储令牌）
       │ ○○  │
       └─────┘
          │
    请求 →│ 取令牌
          │
          ↓
    有令牌 → 允许
    没令牌 → 拒绝
```

```rust
use std::time::{Duration, Instant};

struct TokenBucket {
    capacity: f64,      // 桶容量
    rate: f64,          // 每秒生成令牌数
    tokens: f64,        // 当前令牌数
    last_refill: Instant,
}

impl TokenBucket {
    fn new(capacity: f64, rate: f64) -> Self {
        Self {
            capacity,
            rate,
            tokens: capacity,  // 初始满
            last_refill: Instant::now(),
        }
    }

    fn allow(&mut self) -> bool {
        self.refill();

        if self.tokens >= 1.0 {
            self.tokens -= 1.0;
            true
        } else {
            false
        }
    }

    fn allow_n(&mut self, n: f64) -> bool {
        self.refill();

        if self.tokens >= n {
            self.tokens -= n;
            true
        } else {
            false
        }
    }

    fn refill(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        let new_tokens = elapsed * self.rate;

        self.tokens = (self.tokens + new_tokens).min(self.capacity);
        self.last_refill = now;
    }
}
```

**特点**：

| 优点 | 缺点 |
|------|------|
| 允许突发 | 突发可能压垮下游 |
| 平均速率受控 | 参数不好调 |
| 实现简单 | - |

**适用场景**：允许一定突发的场景，如 API 网关、用户请求限流。

## 漏桶 vs 令牌桶

```text
漏桶：
  - 输出速率恒定
  - 适合保护下游（下游处理能力固定）
  - 突发流量会排队或丢弃

令牌桶：
  - 允许突发（桶里有令牌就能用）
  - 适合应对突发（用户请求本来就有波动）
  - 平均速率受控
```

| 场景 | 推荐算法 |
|------|----------|
| 调用第三方 API（有 QPS 限制） | 漏桶 |
| 用户请求限流 | 令牌桶 |
| 消息队列消费 | 漏桶 |
| 接口熔断 | 令牌桶 |

## 分布式限流

单机限流不够，需要全局限流。

### 方案一：Redis + Lua

```lua
-- 令牌桶限流的 Lua 脚本
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- 获取当前状态
local state = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(state[1]) or capacity
local last_refill = tonumber(state[2]) or now

-- 计算新令牌
local elapsed = now - last_refill
local new_tokens = elapsed * rate / 1000  -- 毫秒转秒
tokens = math.min(capacity, tokens + new_tokens)

-- 尝试获取令牌
local allowed = 0
if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
end

-- 更新状态
redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
redis.call('EXPIRE', key, 60)

return allowed
```

```rust
use chrono::Utc;

async fn distributed_rate_limit(redis: &Redis, key: &str, capacity: f64, rate: f64) -> bool {
    let now = Utc::now().timestamp_millis();
    let result: i32 = redis
        .eval(LUA_SCRIPT, &[key], &[capacity, rate, now, 1.0])
        .await
        .unwrap_or(0);
    result == 1
}
```

### 方案二：滑动窗口 + Redis ZSET

```rust
use chrono::Utc;

async fn sliding_window_limit(
    redis: &Redis,
    key: &str,
    max_requests: usize,
    window_ms: i64,
) -> bool {
    let now = Utc::now().timestamp_millis();
    let window_start = now - window_ms;

    // 移除窗口外的请求
    redis.zremrangebyscore(key, 0, window_start).await;

    // 统计当前窗口的请求数
    let count: usize = redis.zcard(key).await.unwrap_or(0);

    if count < max_requests {
        // 记录本次请求
        let request_id = uuid::Uuid::new_v4().to_string();
        redis.zadd(key, now, &request_id).await;
        redis.expire(key, (window_ms / 1000 + 1) as usize).await;
        true
    } else {
        false
    }
}
```

## 多维度限流

实际系统需要多维度限流：

```rust
use std::collections::HashMap;

struct MultiDimensionLimiter {
    // 全局限流
    global: TokenBucket,
    // 每用户限流
    per_user: HashMap<String, TokenBucket>,
    // 每 IP 限流
    per_ip: HashMap<String, TokenBucket>,
}

impl MultiDimensionLimiter {
    fn new(global_capacity: f64, global_rate: f64) -> Self {
        Self {
            global: TokenBucket::new(global_capacity, global_rate),
            per_user: HashMap::new(),
            per_ip: HashMap::new(),
        }
    }

    fn allow(&mut self, user_id: &str, ip: &str) -> LimitResult {
        // 检查全局限流
        if !self.global.allow() {
            return LimitResult::GlobalLimit;
        }

        // 检查用户限流
        let user_limiter = self.per_user
            .entry(user_id.to_string())
            .or_insert_with(|| TokenBucket::new(100.0, 10.0));

        if !user_limiter.allow() {
            return LimitResult::UserLimit;
        }

        // 检查 IP 限流
        let ip_limiter = self.per_ip
            .entry(ip.to_string())
            .or_insert_with(|| TokenBucket::new(1000.0, 100.0));

        if !ip_limiter.allow() {
            return LimitResult::IpLimit;
        }

        LimitResult::Allowed
    }
}

enum LimitResult {
    Allowed,
    GlobalLimit,
    UserLimit,
    IpLimit,
}
```

## 限流的配置策略

### 1. 预热

```rust
use std::time::{Duration, Instant};

struct WarmupTokenBucket {
    bucket: TokenBucket,
    warmup_duration: Duration,
    start_time: Instant,
}

impl WarmupTokenBucket {
    fn new(capacity: f64, rate: f64, warmup_duration: Duration) -> Self {
        Self {
            bucket: TokenBucket::new(capacity, rate),
            warmup_duration,
            start_time: Instant::now(),
        }
    }

    fn allow(&mut self) -> bool {
        let elapsed = self.start_time.elapsed();

        // 预热期间逐渐提升速率
        let warmup_factor = if elapsed < self.warmup_duration {
            elapsed.as_secs_f64() / self.warmup_duration.as_secs_f64()
        } else {
            1.0
        };

        // 调整有效速率
        let effective_tokens = self.bucket.tokens * warmup_factor;
        if effective_tokens >= 1.0 {
            self.bucket.tokens -= 1.0;
            true
        } else {
            false
        }
    }
}
```

### 2. 优雅降级

```rust
enum RequestPriority {
    Critical,  // 关键请求（登录、支付）
    Normal,    // 普通请求
    Low,       // 低优先级（统计、日志）
}

impl MultiDimensionLimiter {
    fn allow_with_priority(&mut self, priority: RequestPriority) -> bool {
        let threshold = match priority {
            RequestPriority::Critical => 0.1,  // 只保留 10% 容量给关键请求
            RequestPriority::Normal => 0.5,
            RequestPriority::Low => 1.0,      // 必须有足够的余量
        };

        let current_usage = 1.0 - (self.global.tokens / self.global.capacity);

        if current_usage < threshold {
            self.global.allow()
        } else {
            priority == RequestPriority::Critical && self.global.allow()
        }
    }
}
```

## 常见陷阱

### 1. 限流粒度太粗

```text
错误：全局限流 1000 QPS
  → 一个大客户用掉 900 QPS
  → 其他客户只剩 100 QPS

正确：全局 1000 + 每客户 100
  → 每个客户最多 100 QPS
  → 保护公平性
```

### 2. 限流后的处理

```rust
// 错误：简单返回 429
fn handle_request(req: Request) -> Response {
    if !limiter.allow() {
        return Response::new(429);
    }
    // ...
}

// 正确：返回 Retry-After 头
fn handle_request(req: Request) -> Response {
    if !limiter.allow() {
        let retry_after = limiter.estimate_wait_time();
        return Response::builder()
            .status(429)
            .header("Retry-After", retry_after.as_secs())
            .body("Too Many Requests");
    }
    // ...
}
```

### 3. 突发导致雪崩

```text
场景：
  令牌桶容量 1000，速率 100/秒
  积攒了 1000 个令牌
  突发 1000 个请求同时通过
  下游服务扛不住

解决方案：
  1. 限制最大突发（capacity 不要太大）
  2. 下游也做限流
  3. 熔断机制
```

## 总结

| 算法 | 特点 | 适用场景 |
|------|------|----------|
| 固定窗口 | 简单，有临界问题 | 简单场景 |
| 滑动窗口 | 平滑，占内存 | 需要精确计数 |
| 漏桶 | 恒定速率 | 保护下游 |
| 令牌桶 | 允许突发 | 应对流量波动 |

**核心认知**：

> 限流不是目的，保护系统才是。限流算法要结合业务场景选择，还要考虑降级策略、监控告警、动态调整。算法只是第一步。

---

*上一篇：[堆与优先队列：调度器的核心](/posts/algo-not-leetcode-6-heap/)*

*下一篇：[跳表：Redis 排行榜的秘密](/posts/algo-not-leetcode-8-skiplist/)*

*本系列：*
1. *[算法是业务武器](/posts/algo-not-leetcode-1-intro/)*
2. *[排序与二分：被低估的基础功](/posts/algo-not-leetcode-2-sort-binary-search/)*
3. *[哈希表：缓存设计的基石](/posts/algo-not-leetcode-3-hashtable/)*
4. *[布隆过滤器：用 1% 误判换 90% 内存](/posts/algo-not-leetcode-4-bloom-filter/)*
5. *[一致性哈希：分布式系统的路由表](/posts/algo-not-leetcode-5-consistent-hash/)*
6. *[堆与优先队列：调度器的核心](/posts/algo-not-leetcode-6-heap/)*
7. *限流算法：保护系统的三道防线（本篇）*
8. *[跳表：Redis 排行榜的秘密](/posts/algo-not-leetcode-8-skiplist/)*
9. *[树结构：层级数据的优雅解法](/posts/algo-not-leetcode-9-tree/)*
10. *[字符串匹配：敏感词过滤的正确姿势](/posts/algo-not-leetcode-10-string-match/)*
11. *[实战选型：没有银弹，只有场景](/posts/algo-not-leetcode-11-selection/)*
