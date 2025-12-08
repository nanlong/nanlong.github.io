---
title: "模式不是套路（四）：数据管道——像乐高一样组装逻辑"
date: 2025-12-10T10:30:00+08:00
draft: false
tags: ["Rust", "设计模式", "数据处理", "装饰器模式", "责任链模式", "策略模式"]
categories: ["技术"]
description: "数据处理逻辑写成一坨，每次改需求都要动主流程？装饰器、责任链、策略三种模式，教你把处理逻辑拆成可组合的积木块。"
series: ["模式不是套路"]
---

数据处理逻辑写成一坨，每次改需求都要动主流程？装饰器、责任链、策略三种模式，教你把处理逻辑拆成可组合的积木块。

<!--more-->

## 问题：一个不断膨胀的处理函数

你在写一个日志分析系统。最初的需求很简单：

```rust
fn process_log(line: &str) -> Option<LogEntry> {
    let entry = parse_log(line)?;
    Some(entry)
}
```

然后需求来了：

```text
第一周："过滤掉 DEBUG 级别的日志"
第二周："IP 地址要脱敏"
第三周："时间戳要转成北京时间"
第四周："错误日志要额外标记"
第五周："支持采样，只处理 10% 的日志"
第六周："添加处理耗时统计"
```

代码变成了这样：

```rust
fn process_log(line: &str, config: &Config) -> Option<LogEntry> {
    // 采样
    if config.sample_rate < 1.0 {
        if rand::random::<f64>() > config.sample_rate {
            return None;
        }
    }

    let start = Instant::now();

    // 解析
    let mut entry = parse_log(line)?;

    // 过滤
    if config.min_level > entry.level {
        return None;
    }

    // IP 脱敏
    if config.mask_ip {
        entry.ip = mask_ip(&entry.ip);
    }

    // 时区转换
    if let Some(tz) = &config.target_timezone {
        entry.timestamp = convert_timezone(entry.timestamp, tz);
    }

    // 错误标记
    if entry.level >= LogLevel::Error {
        entry.tags.push("needs_attention".to_string());
    }

    // 耗时统计
    if config.collect_metrics {
        METRICS.record_processing_time(start.elapsed());
    }

    Some(entry)
}
```

问题显而易见：

```text
1. 函数越来越长，难以理解
2. 每次加功能都要改这个函数
3. 功能之间耦合，无法单独测试
4. 不同场景需要不同的处理组合，但代码是硬编码的
```

## 破局：把处理步骤拆开

核心思路：**把一个大函数拆成多个可组合的小步骤**。

### 方案一：责任链模式

**责任链模式**的核心是：**将请求沿着处理者链传递，直到某个处理者处理它**。

但在数据管道场景，我们做个变体：每个处理者都处理数据，然后传给下一个。

```rust
trait LogProcessor: Send + Sync {
    fn process(&self, entry: LogEntry) -> Option<LogEntry>;
}

// 过滤器：按日志级别过滤
struct LevelFilter {
    min_level: LogLevel,
}

impl LevelFilter {
    fn new(min_level: LogLevel) -> Self {
        Self { min_level }
    }
}

impl LogProcessor for LevelFilter {
    fn process(&self, entry: LogEntry) -> Option<LogEntry> {
        if entry.level >= self.min_level {
            Some(entry)
        } else {
            None  // 过滤掉
        }
    }
}

// IP 脱敏
struct IpMasker;

impl LogProcessor for IpMasker {
    fn process(&self, mut entry: LogEntry) -> Option<LogEntry> {
        entry.ip = mask_ip(&entry.ip);
        Some(entry)
    }
}

// 时区转换
struct TimezoneConverter {
    target_tz: chrono_tz::Tz,
}

impl TimezoneConverter {
    fn new(tz: chrono_tz::Tz) -> Self {
        Self { target_tz: tz }
    }
}

impl LogProcessor for TimezoneConverter {
    fn process(&self, mut entry: LogEntry) -> Option<LogEntry> {
        if let Some(ts) = entry.timestamp {
            // 转换时区后格式化为字符串存储，或保持 UTC 存储
            entry.timestamp = Some(ts.with_timezone(&self.target_tz).to_utc());
        }
        Some(entry)
    }
}

// 管道：串联多个处理器
struct Pipeline {
    processors: Vec<Box<dyn LogProcessor>>,
}

impl Pipeline {
    fn new() -> Self {
        Self { processors: Vec::new() }
    }

    fn add(mut self, processor: impl LogProcessor + 'static) -> Self {
        self.processors.push(Box::new(processor));
        self
    }

    fn process(&self, mut entry: LogEntry) -> Option<LogEntry> {
        for processor in &self.processors {
            entry = processor.process(entry)?;
        }
        Some(entry)
    }
}
```

使用时像搭积木：

```rust
let pipeline = Pipeline::new()
    .add(LevelFilter::new(LogLevel::Info))
    .add(IpMasker)
    .add(TimezoneConverter::new(chrono_tz::Asia::Shanghai));

// 处理日志
for line in log_lines {
    if let Some(entry) = parse_log(&line) {
        if let Some(processed) = pipeline.process(entry) {
            output.write(processed);
        }
    }
}
```

### Rust 的礼物：Iterator 适配器

Rust 标准库的 `Iterator` 就是责任链的完美实现：

```rust
let results: Vec<LogEntry> = log_lines
    .iter()
    .filter_map(|line| parse_log(line))           // 解析
    .filter(|e| e.level >= LogLevel::Info)        // 过滤
    .map(|mut e| { e.ip = mask_ip(&e.ip); e })    // 脱敏
    .map(|mut e| {                                 // 时区转换
        e.timestamp = e.timestamp.map(|ts| ts.with_timezone(&tz).to_utc());
        e
    })
    .collect();
```

每个 `.filter()` 和 `.map()` 都是一个处理步骤，可以自由组合。而且是**惰性求值**——只有在 `.collect()` 时才真正执行。

### 方案二：装饰器模式

**装饰器模式**的核心是：**动态地给对象添加额外的职责**。

与责任链不同，装饰器是**包装**关系，外层可以在调用内层前后做事情。

```rust
trait LogProcessor: Send + Sync {
    fn process(&self, entry: LogEntry) -> Option<LogEntry>;
}

// 基础处理器
struct BaseProcessor;

impl LogProcessor for BaseProcessor {
    fn process(&self, entry: LogEntry) -> Option<LogEntry> {
        Some(entry)
    }
}

// 装饰器：添加耗时统计
struct TimingDecorator<P: LogProcessor> {
    inner: P,
    name: String,
}

impl<P: LogProcessor> TimingDecorator<P> {
    fn new(inner: P, name: impl Into<String>) -> Self {
        Self { inner, name: name.into() }
    }
}

impl<P: LogProcessor> LogProcessor for TimingDecorator<P> {
    fn process(&self, entry: LogEntry) -> Option<LogEntry> {
        let start = Instant::now();
        let result = self.inner.process(entry);  // 调用内层
        let elapsed = start.elapsed();
        println!("[{}] took {:?}", self.name, elapsed);
        result
    }
}

// 装饰器：添加日志
struct LoggingDecorator<P: LogProcessor> {
    inner: P,
}

impl<P: LogProcessor> LoggingDecorator<P> {
    fn new(inner: P) -> Self {
        Self { inner }
    }
}

impl<P: LogProcessor> LogProcessor for LoggingDecorator<P> {
    fn process(&self, entry: LogEntry) -> Option<LogEntry> {
        println!("Processing entry: {:?}", entry.id);
        let result = self.inner.process(entry);
        if result.is_none() {
            println!("Entry was filtered out");
        }
        result
    }
}

// 装饰器：错误处理
struct ErrorHandlingDecorator<P: LogProcessor> {
    inner: P,
}

impl<P: LogProcessor> ErrorHandlingDecorator<P> {
    fn new(inner: P) -> Self {
        Self { inner }
    }
}

impl<P: LogProcessor> LogProcessor for ErrorHandlingDecorator<P> {
    fn process(&self, entry: LogEntry) -> Option<LogEntry> {
        // 用 catch_unwind 防止 panic 传播
        std::panic::catch_unwind(std::panic::AssertUnwindSafe(|| {
            self.inner.process(entry)
        }))
        .unwrap_or_else(|_| {
            eprintln!("Processor panicked!");
            None
        })
    }
}
```

装饰器可以层层包装：

```rust
// 从内到外：BaseProcessor -> Timing -> Logging -> ErrorHandling
let processor = ErrorHandlingDecorator::new(
    LoggingDecorator::new(
        TimingDecorator::new(
            BaseProcessor,
            "base"
        )
    )
);

// 调用时，执行顺序：
// 1. ErrorHandlingDecorator.process() 开始
// 2.   LoggingDecorator.process() 开始
// 3.     TimingDecorator.process() 开始
// 4.       BaseProcessor.process()
// 5.     TimingDecorator.process() 结束（记录耗时）
// 6.   LoggingDecorator.process() 结束（记录日志）
// 7. ErrorHandlingDecorator.process() 结束（捕获异常）
```

### Rust 的礼物：Deref 实现透明装饰

利用 `Deref` trait，装饰器可以"假装"成被装饰的对象：

```rust
use std::ops::Deref;

struct CachedProcessor<P> {
    inner: P,
    cache: HashMap<String, LogEntry>,
}

impl<P> Deref for CachedProcessor<P> {
    type Target = P;

    fn deref(&self) -> &Self::Target {
        &self.inner
    }
}

impl<P: LogProcessor> CachedProcessor<P> {
    fn process_cached(&mut self, entry: LogEntry) -> Option<LogEntry> {
        let key = entry.id.clone();

        if let Some(cached) = self.cache.get(&key) {
            return Some(cached.clone());
        }

        let result = self.inner.process(entry)?;
        self.cache.insert(key, result.clone());
        Some(result)
    }
}
```

### 方案三：策略模式

当需要**动态选择处理算法**时，用策略模式。

```rust
trait SamplingStrategy: Send + Sync {
    fn should_process(&self, entry: &LogEntry) -> bool;
}

// 固定比例采样
struct RatioSampling {
    rate: f64,
}

impl RatioSampling {
    fn new(rate: f64) -> Self {
        Self { rate: rate.clamp(0.0, 1.0) }
    }
}

impl SamplingStrategy for RatioSampling {
    fn should_process(&self, _entry: &LogEntry) -> bool {
        rand::random::<f64>() < self.rate
    }
}

// 基于内容的采样（错误日志全量，其他采样）
struct ContentBasedSampling {
    error_rate: f64,
    normal_rate: f64,
}

impl ContentBasedSampling {
    fn new(error_rate: f64, normal_rate: f64) -> Self {
        Self { error_rate, normal_rate }
    }
}

impl SamplingStrategy for ContentBasedSampling {
    fn should_process(&self, entry: &LogEntry) -> bool {
        let rate = if entry.level >= LogLevel::Error {
            self.error_rate
        } else {
            self.normal_rate
        };
        rand::random::<f64>() < rate
    }
}

// 时间窗口采样（每分钟最多 N 条）
struct WindowSampling {
    max_per_minute: usize,
    counter: std::sync::atomic::AtomicUsize,
    window_start: std::sync::RwLock<Instant>,
}

impl WindowSampling {
    fn new(max_per_minute: usize) -> Self {
        Self {
            max_per_minute,
            counter: std::sync::atomic::AtomicUsize::new(0),
            window_start: std::sync::RwLock::new(Instant::now()),
        }
    }
}

impl SamplingStrategy for WindowSampling {
    fn should_process(&self, _entry: &LogEntry) -> bool {
        // 检查是否需要重置窗口
        let mut window = self.window_start.write().unwrap();
        if window.elapsed() > Duration::from_secs(60) {
            *window = Instant::now();
            self.counter.store(0, Ordering::Relaxed);
        }
        drop(window);

        // 检查计数
        let count = self.counter.fetch_add(1, Ordering::Relaxed);
        count < self.max_per_minute
    }
}

// 使用策略的处理器
struct SamplingProcessor {
    strategy: Box<dyn SamplingStrategy>,
}

impl SamplingProcessor {
    fn new(strategy: impl SamplingStrategy + 'static) -> Self {
        Self { strategy: Box::new(strategy) }
    }
}

impl LogProcessor for SamplingProcessor {
    fn process(&self, entry: LogEntry) -> Option<LogEntry> {
        if self.strategy.should_process(&entry) {
            Some(entry)
        } else {
            None
        }
    }
}
```

策略可以在运行时切换：

```rust
// 开发环境：全量处理
let sampling = SamplingProcessor::new(RatioSampling::new(1.0));

// 生产环境：错误全量，其他 10%
let sampling = SamplingProcessor::new(
    ContentBasedSampling::new(1.0, 0.1)
);

// 高峰期：限流
let sampling = SamplingProcessor::new(
    WindowSampling::new(1000)  // 每分钟最多 1000 条
);
```

## 演进：组合三种模式

实际系统中，三种模式经常一起使用：

```rust
// 完整的日志处理管道
struct LogPipeline {
    // 采样策略（策略模式）
    sampler: Box<dyn SamplingStrategy>,
    // 处理链（责任链模式）
    processors: Vec<Box<dyn LogProcessor>>,
    // 装饰：统一的错误处理和监控（装饰器模式）
    metrics: Arc<Metrics>,
}

impl LogPipeline {
    fn builder() -> LogPipelineBuilder {
        LogPipelineBuilder::new()
    }

    fn process(&self, entry: LogEntry) -> Option<LogEntry> {
        // 采样
        if !self.sampler.should_process(&entry) {
            self.metrics.record_sampled_out();
            return None;
        }

        let start = Instant::now();

        // 处理链
        let mut current = Some(entry);
        for processor in &self.processors {
            current = current.and_then(|e| processor.process(e));
            if current.is_none() {
                self.metrics.record_filtered_out();
                return None;
            }
        }

        self.metrics.record_processing_time(start.elapsed());
        current
    }
}

// 建造者模式构建管道
struct LogPipelineBuilder {
    sampler: Option<Box<dyn SamplingStrategy>>,
    processors: Vec<Box<dyn LogProcessor>>,
    metrics: Arc<Metrics>,
}

impl LogPipelineBuilder {
    fn new() -> Self {
        Self {
            sampler: None,
            processors: Vec::new(),
            metrics: Arc::new(Metrics::new()),
        }
    }

    fn with_sampling(mut self, strategy: impl SamplingStrategy + 'static) -> Self {
        self.sampler = Some(Box::new(strategy));
        self
    }

    fn add_processor(mut self, processor: impl LogProcessor + 'static) -> Self {
        self.processors.push(Box::new(processor));
        self
    }

    fn with_metrics(mut self, metrics: Arc<Metrics>) -> Self {
        self.metrics = metrics;
        self
    }

    fn build(self) -> LogPipeline {
        LogPipeline {
            sampler: self.sampler.unwrap_or_else(|| Box::new(RatioSampling::new(1.0))),
            processors: self.processors,
            metrics: self.metrics,
        }
    }
}
```

使用示例：

```rust
let metrics = Arc::new(Metrics::new());

let pipeline = LogPipeline::builder()
    .with_sampling(ContentBasedSampling::new(1.0, 0.1))
    .add_processor(LevelFilter::new(LogLevel::Info))
    .add_processor(IpMasker)
    .add_processor(TimezoneConverter::new(chrono_tz::Asia::Shanghai))
    .add_processor(ErrorTagger)
    .with_metrics(Arc::clone(&metrics))
    .build();

// 处理日志流
for line in log_stream {
    if let Some(entry) = parse_log(&line) {
        if let Some(processed) = pipeline.process(entry) {
            output.write(processed);
        }
    }
}

// 查看统计
println!("Processed: {}", metrics.processed_count());
println!("Filtered: {}", metrics.filtered_count());
println!("Avg time: {:?}", metrics.avg_processing_time());
```

## 进阶：异步管道

现代系统经常需要异步处理：

```rust
use async_trait::async_trait;

#[async_trait]
trait AsyncLogProcessor: Send + Sync {
    async fn process(&self, entry: LogEntry) -> Option<LogEntry>;
}

// 异步处理器：调用外部服务进行地理位置查询
struct GeoIpEnricher {
    client: reqwest::Client,
    api_url: String,
}

impl GeoIpEnricher {
    fn new(api_url: impl Into<String>) -> Self {
        Self {
            client: reqwest::Client::new(),
            api_url: api_url.into(),
        }
    }
}

#[async_trait]
impl AsyncLogProcessor for GeoIpEnricher {
    async fn process(&self, mut entry: LogEntry) -> Option<LogEntry> {
        // 调用 GeoIP 服务
        let url = format!("{}/{}", self.api_url, entry.ip);
        if let Ok(resp) = self.client.get(&url).send().await {
            if let Ok(geo) = resp.json::<GeoInfo>().await {
                entry.country = Some(geo.country);
                entry.city = Some(geo.city);
            }
        }
        Some(entry)
    }
}

// 异步管道
struct AsyncPipeline {
    processors: Vec<Box<dyn AsyncLogProcessor>>,
}

impl AsyncPipeline {
    async fn process(&self, mut entry: LogEntry) -> Option<LogEntry> {
        for processor in &self.processors {
            entry = processor.process(entry).await?;
        }
        Some(entry)
    }

    // 并行处理多条日志
    async fn process_batch(&self, entries: Vec<LogEntry>) -> Vec<LogEntry> {
        let futures: Vec<_> = entries
            .into_iter()
            .map(|e| self.process(e))
            .collect();

        futures::future::join_all(futures)
            .await
            .into_iter()
            .flatten()
            .collect()
    }
}
```

### 使用 Tower 构建中间件

Rust 生态的 `tower` crate 是构建异步服务管道的利器：

```rust
use tower::{Service, ServiceBuilder, ServiceExt};
use std::task::{Context, Poll};
use std::pin::Pin;
use std::future::Future;

// 定义服务
struct LogService;

impl Service<LogEntry> for LogService {
    type Response = LogEntry;
    type Error = ProcessError;
    type Future = Pin<Box<dyn Future<Output = Result<LogEntry, ProcessError>> + Send>>;

    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, entry: LogEntry) -> Self::Future {
        Box::pin(async move { Ok(entry) })
    }
}

// 使用 Tower 的中间件
let service = ServiceBuilder::new()
    .concurrency_limit(100)        // 并发限制
    .rate_limit(1000, Duration::from_secs(1))  // 限流
    .timeout(Duration::from_secs(5))           // 超时
    .service(LogService);
```

## 陷阱：别掉进这些坑

### 陷阱一：处理器之间的隐式依赖

```rust
// ❌ 危险：TimezoneConverter 依赖 entry.timestamp 已经被正确设置
let pipeline = Pipeline::new()
    .add(TimezoneConverter::new(tz))  // 如果时间戳是空的会怎样？
    .add(TimestampParser)             // 这个应该在前面！
```

解决方案：

```rust
// ✅ 方案 1：显式声明依赖
trait LogProcessor {
    fn requires(&self) -> &[&str] { &[] }  // 依赖的字段
    fn provides(&self) -> &[&str] { &[] }  // 填充的字段
    fn process(&self, entry: LogEntry) -> Option<LogEntry>;
}

impl Pipeline {
    fn validate(&self) -> Result<(), ValidationError> {
        let mut available: HashSet<&str> = HashSet::new();

        for processor in &self.processors {
            // 检查依赖是否满足
            for req in processor.requires() {
                if !available.contains(req) {
                    return Err(ValidationError::MissingDependency {
                        processor: processor.name(),
                        requires: req.to_string(),
                    });
                }
            }
            // 添加提供的字段
            available.extend(processor.provides());
        }

        Ok(())
    }
}

// ✅ 方案 2：防御性编程
impl LogProcessor for TimezoneConverter {
    fn process(&self, mut entry: LogEntry) -> Option<LogEntry> {
        // 如果时间戳为空，跳过转换而不是崩溃
        if let Some(ts) = entry.timestamp {
            entry.timestamp = Some(ts.with_timezone(&self.target_tz));
        }
        Some(entry)
    }
}
```

### 陷阱二：装饰器顺序错误

```rust
// ❌ 错误顺序：日志在错误处理之外
let processor = LoggingDecorator::new(
    ErrorHandlingDecorator::new(inner)
);
// 如果 inner panic，LoggingDecorator 也会 panic

// ✅ 正确顺序：错误处理在最外层
let processor = ErrorHandlingDecorator::new(
    LoggingDecorator::new(inner)
);
// ErrorHandlingDecorator 会捕获内部的所有 panic
```

### 陷阱三：策略的运行时切换不安全

```rust
// ❌ 危险：多线程环境下的策略切换
struct Pipeline {
    strategy: Box<dyn SamplingStrategy>,  // 不是线程安全的
}

impl Pipeline {
    fn set_strategy(&mut self, strategy: Box<dyn SamplingStrategy>) {
        self.strategy = strategy;  // 在处理过程中切换会怎样？
    }
}

// ✅ 安全：使用 Arc + Atomic 或 RwLock
struct Pipeline {
    strategy: Arc<ArcSwap<Box<dyn SamplingStrategy>>>,
}

impl Pipeline {
    fn set_strategy(&self, strategy: impl SamplingStrategy + 'static) {
        self.strategy.store(Arc::new(Box::new(strategy)));
    }
}
```

### 陷阱四：管道太长导致性能问题

```rust
// ❌ 每个处理器都是一次虚函数调用
let pipeline = Pipeline::new()
    .add(Step1)
    .add(Step2)
    // ... 50 个处理器
    .add(Step50);

// 处理一条日志需要 50 次虚函数调用
```

解决方案：

```rust
// ✅ 方案 1：合并相关的处理步骤
struct CompositeProcessor {
    // 把多个简单步骤合并成一个
}

// ✅ 方案 2：使用泛型避免动态分发
fn process<P1, P2, P3>(entry: LogEntry, p1: &P1, p2: &P2, p3: &P3) -> Option<LogEntry>
where
    P1: LogProcessor,
    P2: LogProcessor,
    P3: LogProcessor,
{
    let entry = p1.process(entry)?;
    let entry = p2.process(entry)?;
    p3.process(entry)
}

// ✅ 方案 3：用 enum 替代 trait object
enum Processor {
    LevelFilter(LevelFilter),
    IpMasker(IpMasker),
    TimezoneConverter(TimezoneConverter),
}

impl Processor {
    fn process(&self, entry: LogEntry) -> Option<LogEntry> {
        match self {
            Processor::LevelFilter(p) => p.process(entry),
            Processor::IpMasker(p) => p.process(entry),
            Processor::TimezoneConverter(p) => p.process(entry),
        }
    }
}
```

## 完整示例

```rust
use std::time::{Duration, Instant};
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};

// ==================== 核心类型 ====================

#[derive(Debug, Clone)]
pub struct LogEntry {
    pub id: String,
    pub timestamp: Option<chrono::DateTime<chrono::Utc>>,
    pub level: LogLevel,
    pub message: String,
    pub ip: String,
    pub tags: Vec<String>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum LogLevel {
    Debug,
    Info,
    Warn,
    Error,
}

// ==================== 处理器 Trait ====================

pub trait LogProcessor: Send + Sync {
    fn name(&self) -> &str;
    fn process(&self, entry: LogEntry) -> Option<LogEntry>;
}

// ==================== 具体处理器 ====================

pub struct LevelFilter {
    min_level: LogLevel,
}

impl LevelFilter {
    pub fn new(min_level: LogLevel) -> Self {
        Self { min_level }
    }
}

impl LogProcessor for LevelFilter {
    fn name(&self) -> &str { "LevelFilter" }

    fn process(&self, entry: LogEntry) -> Option<LogEntry> {
        if entry.level >= self.min_level {
            Some(entry)
        } else {
            None
        }
    }
}

pub struct IpMasker;

impl LogProcessor for IpMasker {
    fn name(&self) -> &str { "IpMasker" }

    fn process(&self, mut entry: LogEntry) -> Option<LogEntry> {
        // 192.168.1.100 -> 192.168.1.***
        if let Some(last_dot) = entry.ip.rfind('.') {
            entry.ip = format!("{}.***", &entry.ip[..last_dot]);
        }
        Some(entry)
    }
}

pub struct ErrorTagger;

impl LogProcessor for ErrorTagger {
    fn name(&self) -> &str { "ErrorTagger" }

    fn process(&self, mut entry: LogEntry) -> Option<LogEntry> {
        if entry.level >= LogLevel::Error {
            entry.tags.push("needs_attention".to_string());
        }
        Some(entry)
    }
}

// ==================== 采样策略 ====================

pub trait SamplingStrategy: Send + Sync {
    fn should_process(&self, entry: &LogEntry) -> bool;
}

pub struct RatioSampling {
    rate: f64,
}

impl RatioSampling {
    pub fn new(rate: f64) -> Self {
        Self { rate: rate.clamp(0.0, 1.0) }
    }
}

impl SamplingStrategy for RatioSampling {
    fn should_process(&self, _entry: &LogEntry) -> bool {
        rand::random::<f64>() < self.rate
    }
}

pub struct FullSampling;

impl SamplingStrategy for FullSampling {
    fn should_process(&self, _entry: &LogEntry) -> bool {
        true
    }
}

// ==================== 监控指标 ====================

pub struct Metrics {
    processed: AtomicUsize,
    filtered: AtomicUsize,
    sampled_out: AtomicUsize,
}

impl Metrics {
    pub fn new() -> Self {
        Self {
            processed: AtomicUsize::new(0),
            filtered: AtomicUsize::new(0),
            sampled_out: AtomicUsize::new(0),
        }
    }

    pub fn record_processed(&self) {
        self.processed.fetch_add(1, Ordering::Relaxed);
    }

    pub fn record_filtered(&self) {
        self.filtered.fetch_add(1, Ordering::Relaxed);
    }

    pub fn record_sampled_out(&self) {
        self.sampled_out.fetch_add(1, Ordering::Relaxed);
    }

    pub fn summary(&self) -> String {
        format!(
            "Processed: {}, Filtered: {}, Sampled out: {}",
            self.processed.load(Ordering::Relaxed),
            self.filtered.load(Ordering::Relaxed),
            self.sampled_out.load(Ordering::Relaxed),
        )
    }
}

// ==================== 管道 ====================

pub struct Pipeline {
    sampler: Box<dyn SamplingStrategy>,
    processors: Vec<Box<dyn LogProcessor>>,
    metrics: Arc<Metrics>,
}

impl Pipeline {
    pub fn builder() -> PipelineBuilder {
        PipelineBuilder::new()
    }

    pub fn process(&self, entry: LogEntry) -> Option<LogEntry> {
        // 采样
        if !self.sampler.should_process(&entry) {
            self.metrics.record_sampled_out();
            return None;
        }

        // 处理链
        let mut current = Some(entry);
        for processor in &self.processors {
            current = current.and_then(|e| processor.process(e));
            if current.is_none() {
                self.metrics.record_filtered();
                return None;
            }
        }

        self.metrics.record_processed();
        current
    }
}

pub struct PipelineBuilder {
    sampler: Option<Box<dyn SamplingStrategy>>,
    processors: Vec<Box<dyn LogProcessor>>,
    metrics: Option<Arc<Metrics>>,
}

impl PipelineBuilder {
    pub fn new() -> Self {
        Self {
            sampler: None,
            processors: Vec::new(),
            metrics: None,
        }
    }

    pub fn with_sampling(mut self, strategy: impl SamplingStrategy + 'static) -> Self {
        self.sampler = Some(Box::new(strategy));
        self
    }

    pub fn add(mut self, processor: impl LogProcessor + 'static) -> Self {
        self.processors.push(Box::new(processor));
        self
    }

    pub fn with_metrics(mut self, metrics: Arc<Metrics>) -> Self {
        self.metrics = Some(metrics);
        self
    }

    pub fn build(self) -> Pipeline {
        Pipeline {
            sampler: self.sampler.unwrap_or_else(|| Box::new(FullSampling)),
            processors: self.processors,
            metrics: self.metrics.unwrap_or_else(|| Arc::new(Metrics::new())),
        }
    }
}

// ==================== 使用示例 ====================

fn main() {
    let metrics = Arc::new(Metrics::new());

    let pipeline = Pipeline::builder()
        .with_sampling(RatioSampling::new(0.5))
        .add(LevelFilter::new(LogLevel::Info))
        .add(IpMasker)
        .add(ErrorTagger)
        .with_metrics(Arc::clone(&metrics))
        .build();

    // 模拟处理日志
    let entries = vec![
        LogEntry {
            id: "1".to_string(),
            timestamp: None,
            level: LogLevel::Debug,
            message: "Debug message".to_string(),
            ip: "192.168.1.100".to_string(),
            tags: vec![],
        },
        LogEntry {
            id: "2".to_string(),
            timestamp: None,
            level: LogLevel::Error,
            message: "Error occurred".to_string(),
            ip: "10.0.0.50".to_string(),
            tags: vec![],
        },
    ];

    for entry in entries {
        if let Some(processed) = pipeline.process(entry) {
            println!("Output: {:?}", processed);
        }
    }

    println!("{}", metrics.summary());
}
```

## 小结

数据管道场景让我们自然地用到了三种模式：

| 模式 | 解决的问题 | Rust 实现 |
|------|-----------|----------|
| 责任链 | 处理步骤的串联 | trait + Vec / Iterator 适配器 |
| 装饰器 | 动态添加横切关注点 | 泛型包装类型 / Deref |
| 策略 | 算法的动态选择 | trait object / enum |

关键洞见：

1. **责任链**适合"流水线"场景，每个处理器专注一件事
2. **装饰器**适合"增强"场景，在不改变核心逻辑的情况下添加功能
3. **策略**适合"选择"场景，根据条件选择不同算法
4. **三者经常组合使用**，形成灵活的处理架构

下一篇，我们进入业务核心：**状态流转——订单不只是 CRUD**。看状态模式、命令模式、备忘录模式如何让状态机优雅又健壮。

---

> **系列导航**
>
> - 第一篇：Rust 让设计模式返璞归真
> - 第二篇：配置系统——从硬编码到热加载
> - 第三篇：插件架构——让系统长出翅膀
> - **第四篇：数据管道——像乐高一样组装逻辑**（本文）
> - 第五篇：状态流转——订单不只是 CRUD
> - 第六篇：事件驱动——模块间的优雅对话
> - 第七篇：外部集成——与第三方和平共处
> - 第八篇：框架设计——把控制权还给用户
> - 第九篇：树形世界——递归结构的驯服术
> - 第十篇：规则引擎——让业务人员写代码
> - 第十一篇：终篇——反模式与断舍离
