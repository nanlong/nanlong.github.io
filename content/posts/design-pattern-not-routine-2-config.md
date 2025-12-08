---
title: "模式不是套路（二）：配置系统——从硬编码到热加载"
date: 2025-12-10T10:10:00+08:00
draft: false
tags: ["Rust", "设计模式", "配置管理", "单例模式", "建造者模式"]
categories: ["技术"]
description: "配置文件谁都会写，但写出一个好用的配置系统却不简单。从硬编码到配置文件，从启动加载到热更新，看单例、建造者、原型三种模式如何在配置系统中自然浮现。"
series: ["模式不是套路"]
---

配置文件谁都会写，但写出一个好用的配置系统却不简单。从硬编码到配置文件，从启动加载到热更新，看单例、建造者、原型三种模式如何在配置系统中自然浮现。

<!--more-->

## 问题：一个"简单"的需求

"把数据库连接信息从代码里抽出来。"

产品经理觉得这是个五分钟的活儿。毕竟，不就是把字符串挪个地方吗？

```rust
// 之前：硬编码
const DB_HOST: &str = "localhost";
const DB_PORT: u16 = 5432;
const DB_USER: &str = "admin";
const DB_PASS: &str = "123456";  // 是的，密码就这么写在代码里

// 之后：配置文件
// config.toml
// [database]
// host = "localhost"
// port = 5432
```

开发者改完了，测试通过，上线。

然后问题开始了：

```text
第一天：
  "生产环境的配置呢？" —— 没考虑多环境

第一周：
  "配置改了要重启？不能热加载吗？" —— 没考虑运行时更新

第一个月：
  "微服务那么多，每个都要改配置文件？" —— 没考虑配置中心

第一个季度：
  "敏感信息怎么管理？" —— 没考虑安全性
```

一个"简单"的配置系统，逐渐变成了一个复杂的工程问题。

## 朴素实现：能跑就行？

先看最直接的实现方式：

```rust
use std::fs;
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Config {
    database: DatabaseConfig,
    server: ServerConfig,
}

#[derive(Debug, Deserialize)]
struct DatabaseConfig {
    host: String,
    port: u16,
    username: String,
    password: String,
}

#[derive(Debug, Deserialize)]
struct ServerConfig {
    host: String,
    port: u16,
}

fn load_config() -> Config {
    let content = fs::read_to_string("config.toml")
        .expect("Failed to read config file");
    toml::from_str(&content)
        .expect("Failed to parse config")
}

fn main() {
    let config = load_config();
    println!("Server: {}:{}", config.server.host, config.server.port);
}
```

这个实现能用，但有几个明显的问题：

```text
问题 1：每次调用都重新读文件
  - 性能浪费
  - 文件被删除会崩溃

问题 2：配置在哪里都能被修改
  - fn somewhere(config: &mut Config) 可以随便改

问题 3：没有默认值
  - 配置文件必须写全，漏一个字段就崩

问题 4：没有校验
  - port = -1 也能通过解析
```

## 第一次演进：全局配置（单例的雏形）

最直接的改进是：**启动时加载一次，之后全局共享**。

```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<Config> = OnceLock::new();

pub fn init_config(path: &str) -> Result<(), ConfigError> {
    let content = std::fs::read_to_string(path)
        .map_err(|e| ConfigError::ReadFile(e.to_string()))?;

    let config: Config = toml::from_str(&content)
        .map_err(|e| ConfigError::Parse(e.to_string()))?;

    CONFIG.set(config)
        .map_err(|_| ConfigError::AlreadyInitialized)
}

pub fn get_config() -> &'static Config {
    CONFIG.get().expect("Config not initialized")
}

#[derive(Debug)]
pub enum ConfigError {
    ReadFile(String),
    Parse(String),
    AlreadyInitialized,
}
```

这就是**单例模式**的本质：**确保一个类只有一个实例，并提供全局访问点**。

在 Rust 中，`OnceLock` 帮我们处理了两个棘手的问题：

1. **线程安全**：多线程同时初始化时，只有一个会成功
2. **生命周期**：`'static` 生命周期，程序结束前一直有效

但这个实现还有改进空间：

```rust
// 问题：如果忘了初始化，程序会 panic
fn main() {
    // 忘了调用 init_config()
    let config = get_config();  // panic!
}
```

更优雅的方式是使用 `LazyLock`：

```rust
use std::sync::LazyLock;

static CONFIG: LazyLock<Config> = LazyLock::new(|| {
    let path = std::env::var("CONFIG_PATH")
        .unwrap_or_else(|_| "config.toml".to_string());

    let content = std::fs::read_to_string(&path)
        .unwrap_or_else(|e| panic!("Failed to read config: {}", e));

    toml::from_str(&content)
        .unwrap_or_else(|e| panic!("Failed to parse config: {}", e))
});

pub fn config() -> &'static Config {
    &CONFIG
}
```

`LazyLock` 的特点是：**第一次访问时才初始化**。这避免了"忘记初始化"的问题。

## 第二次演进：建造者模式

配置越来越复杂：

```toml
[server]
host = "0.0.0.0"
port = 8080
max_connections = 1000
timeout_seconds = 30
enable_https = true
cert_path = "/etc/ssl/cert.pem"
key_path = "/etc/ssl/key.pem"

[database]
host = "localhost"
port = 5432
username = "admin"
password = "secret"
max_pool_size = 20
min_pool_size = 5
connection_timeout_seconds = 10
idle_timeout_seconds = 300

[cache]
enabled = true
host = "localhost"
port = 6379
ttl_seconds = 3600

[logging]
level = "info"
format = "json"
output = "stdout"
```

问题来了：

```text
1. 很多字段有合理的默认值，不应该强制用户配置
2. 有些字段之间有依赖关系（enable_https = true 时，cert_path 必填）
3. 配置的构建过程越来越复杂
```

这时候需要**建造者模式**：**将复杂对象的构建与表示分离**。

```rust
#[derive(Debug, Clone)]
pub struct ServerConfig {
    host: String,
    port: u16,
    max_connections: usize,
    timeout: Duration,
    tls: Option<TlsConfig>,
}

#[derive(Debug, Clone)]
pub struct TlsConfig {
    cert_path: PathBuf,
    key_path: PathBuf,
}

#[derive(Default)]
pub struct ServerConfigBuilder {
    host: Option<String>,
    port: Option<u16>,
    max_connections: Option<usize>,
    timeout: Option<Duration>,
    tls: Option<TlsConfig>,
}

impl ServerConfigBuilder {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }

    pub fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }

    pub fn max_connections(mut self, max: usize) -> Self {
        self.max_connections = Some(max);
        self
    }

    pub fn timeout(mut self, timeout: Duration) -> Self {
        self.timeout = Some(timeout);
        self
    }

    pub fn with_tls(mut self, cert_path: impl Into<PathBuf>, key_path: impl Into<PathBuf>) -> Self {
        self.tls = Some(TlsConfig {
            cert_path: cert_path.into(),
            key_path: key_path.into(),
        });
        self
    }

    pub fn build(self) -> Result<ServerConfig, ConfigError> {
        Ok(ServerConfig {
            host: self.host.unwrap_or_else(|| "0.0.0.0".to_string()),
            port: self.port.unwrap_or(8080),
            max_connections: self.max_connections.unwrap_or(1000),
            timeout: self.timeout.unwrap_or(Duration::from_secs(30)),
            tls: self.tls,
        })
    }
}
```

使用起来非常流畅：

```rust
let config = ServerConfigBuilder::new()
    .host("localhost")
    .port(3000)
    .max_connections(500)
    .with_tls("/etc/ssl/cert.pem", "/etc/ssl/key.pem")
    .build()?;
```

### Rust 的礼物：derive_builder

手写 Builder 很繁琐。Rust 生态有 `derive_builder` crate：

```rust
use derive_builder::Builder;

#[derive(Debug, Clone, Builder)]
#[builder(setter(into), build_fn(validate = "Self::validate"))]
pub struct ServerConfig {
    #[builder(default = r#""0.0.0.0".to_string()"#)]
    host: String,

    #[builder(default = "8080")]
    port: u16,

    #[builder(default = "1000")]
    max_connections: usize,

    #[builder(default = "Duration::from_secs(30)")]
    timeout: Duration,

    #[builder(default)]
    tls: Option<TlsConfig>,
}

impl ServerConfigBuilder {
    fn validate(&self) -> Result<(), String> {
        if let Some(port) = self.port {
            if port == 0 {
                return Err("Port cannot be 0".to_string());
            }
        }
        Ok(())
    }
}
```

一行 `#[derive(Builder)]`，自动生成所有 setter 方法。

### 建造者模式的变体：类型状态

Rust 的类型系统可以在编译期强制构建顺序：

```rust
// 类型状态：强制必填字段
pub struct ServerConfigBuilder<HostSet, PortSet> {
    host: Option<String>,
    port: Option<u16>,
    max_connections: usize,
    _marker: PhantomData<(HostSet, PortSet)>,
}

pub struct NotSet;
pub struct Set;

impl ServerConfigBuilder<NotSet, NotSet> {
    pub fn new() -> Self {
        Self {
            host: None,
            port: None,
            max_connections: 1000,
            _marker: PhantomData,
        }
    }
}

impl<P> ServerConfigBuilder<NotSet, P> {
    pub fn host(self, host: impl Into<String>) -> ServerConfigBuilder<Set, P> {
        ServerConfigBuilder {
            host: Some(host.into()),
            port: self.port,
            max_connections: self.max_connections,
            _marker: PhantomData,
        }
    }
}

impl<H> ServerConfigBuilder<H, NotSet> {
    pub fn port(self, port: u16) -> ServerConfigBuilder<H, Set> {
        ServerConfigBuilder {
            host: self.host,
            port: Some(port),
            max_connections: self.max_connections,
            _marker: PhantomData,
        }
    }
}

// 只有当 host 和 port 都设置后，才能调用 build
impl ServerConfigBuilder<Set, Set> {
    pub fn build(self) -> ServerConfig {
        ServerConfig {
            host: self.host.unwrap(),
            port: self.port.unwrap(),
            max_connections: self.max_connections,
            timeout: Duration::from_secs(30),
            tls: None,
        }
    }
}
```

```rust
// ✅ 编译通过
let config = ServerConfigBuilder::new()
    .host("localhost")
    .port(8080)
    .build();

// ❌ 编译失败：没有设置 port，不能调用 build
let config = ServerConfigBuilder::new()
    .host("localhost")
    .build();  // 编译错误！
```

编译期就能发现错误，比运行时 panic 好得多。

## 第三次演进：配置继承（原型模式）

系统有多个环境：开发、测试、预发、生产。

```text
90% 的配置是一样的，只有少数字段不同：
  - 开发：数据库连本地，日志 debug 级别
  - 测试：数据库连测试服务器，日志 info 级别
  - 生产：数据库连集群，日志 warn 级别，开启 HTTPS
```

最笨的方法是写四份完整的配置文件。但这样维护成本太高，改一个公共配置要改四处。

**原型模式**的思路是：**基于现有对象创建新对象**。

```rust
#[derive(Debug, Clone)]
pub struct Config {
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub logging: LoggingConfig,
}

impl Config {
    /// 从基础配置派生新配置
    pub fn derive(&self) -> ConfigDeriver {
        ConfigDeriver {
            base: self.clone(),
        }
    }
}

pub struct ConfigDeriver {
    base: Config,
}

impl ConfigDeriver {
    pub fn with_database_host(mut self, host: impl Into<String>) -> Self {
        self.base.database.host = host.into();
        self
    }

    pub fn with_log_level(mut self, level: LogLevel) -> Self {
        self.base.logging.level = level;
        self
    }

    pub fn with_server_port(mut self, port: u16) -> Self {
        self.base.server.port = port;
        self
    }

    pub fn build(self) -> Config {
        self.base
    }
}
```

使用方式：

```rust
// 基础配置
let base_config = Config::load("config/base.toml")?;

// 派生开发环境配置
let dev_config = base_config.derive()
    .with_database_host("localhost")
    .with_log_level(LogLevel::Debug)
    .build();

// 派生生产环境配置
let prod_config = base_config.derive()
    .with_database_host("db.prod.internal")
    .with_log_level(LogLevel::Warn)
    .with_server_port(443)
    .build();
```

### 配置文件层叠

更实用的做法是让配置文件支持继承：

```rust
use std::path::Path;
use config::{Config as ConfigBuilder, File, Environment};

pub fn load_config(env: &str) -> Result<Config, ConfigError> {
    let builder = ConfigBuilder::builder()
        // 1. 先加载基础配置
        .add_source(File::with_name("config/base"))
        // 2. 再加载环境特定配置（覆盖基础配置）
        .add_source(File::with_name(&format!("config/{}", env)).required(false))
        // 3. 最后用环境变量覆盖（最高优先级）
        .add_source(Environment::with_prefix("APP").separator("__"));

    let config = builder.build()?;
    config.try_deserialize()
}
```

配置文件结构：

```text
config/
├── base.toml        # 基础配置
├── development.toml # 开发环境覆盖
├── staging.toml     # 预发环境覆盖
└── production.toml  # 生产环境覆盖
```

```toml
# config/base.toml
[server]
host = "0.0.0.0"
port = 8080
max_connections = 1000

[database]
port = 5432
max_pool_size = 20

[logging]
level = "info"
format = "json"
```

```toml
# config/production.toml
# 只需要写需要覆盖的部分
[database]
host = "db.prod.internal"
max_pool_size = 100

[logging]
level = "warn"
```

## 第四次演进：热加载

需求升级了：**配置改了不想重启服务**。

这需要引入可变性。但 `OnceLock` 只能写一次，我们需要 `RwLock`：

```rust
use std::sync::RwLock;
use notify::{Watcher, RecursiveMode, Event};

static CONFIG: LazyLock<RwLock<Config>> = LazyLock::new(|| {
    let config = load_config_from_file().expect("Failed to load initial config");
    RwLock::new(config)
});

pub fn config() -> std::sync::RwLockReadGuard<'static, Config> {
    CONFIG.read().unwrap()
}

pub fn reload_config() -> Result<(), ConfigError> {
    let new_config = load_config_from_file()?;
    let mut config = CONFIG.write().unwrap();
    *config = new_config;
    Ok(())
}

/// 启动配置文件监听
pub fn watch_config(path: &str) -> Result<(), notify::Error> {
    let path = path.to_string();

    std::thread::spawn(move || {
        let (tx, rx) = std::sync::mpsc::channel();

        let mut watcher = notify::recommended_watcher(move |res: Result<Event, _>| {
            if let Ok(event) = res {
                if event.kind.is_modify() {
                    let _ = tx.send(());
                }
            }
        }).unwrap();

        watcher.watch(Path::new(&path), RecursiveMode::NonRecursive).unwrap();

        for _ in rx {
            // 文件变化，重新加载
            if let Err(e) = reload_config() {
                eprintln!("Failed to reload config: {:?}", e);
            } else {
                println!("Config reloaded successfully");
            }
        }
    });

    Ok(())
}
```

但这里有个陷阱：**配置变更的原子性**。

```rust
// 危险：配置变更不是原子的
fn handle_request() {
    let db_host = config().database.host.clone();
    // ← 配置可能在这里被重新加载
    let db_port = config().database.port;
    // db_host 和 db_port 可能来自不同版本的配置！
}

// 安全：一次性获取配置快照
fn handle_request() {
    let cfg = config();
    let db_host = &cfg.database.host;
    let db_port = cfg.database.port;
    // 同一个配置版本
}
```

更好的方案是使用 `Arc`：

```rust
use std::sync::Arc;
use arc_swap::ArcSwap;

static CONFIG: LazyLock<ArcSwap<Config>> = LazyLock::new(|| {
    let config = load_config_from_file().expect("Failed to load initial config");
    ArcSwap::from_pointee(config)
});

pub fn config() -> Arc<Config> {
    CONFIG.load_full()
}

pub fn reload_config() -> Result<(), ConfigError> {
    let new_config = load_config_from_file()?;
    CONFIG.store(Arc::new(new_config));
    Ok(())
}
```

`arc_swap` crate 提供的 `ArcSwap` 是专门为这种场景设计的：

- **读取几乎无锁**：`load_full()` 返回 `Arc` 的克隆，非常快
- **写入原子**：`store()` 原子地替换整个配置
- **无读写竞争**：读者拿到的是快照，不受后续更新影响

## 陷阱：别掉进这些坑

### 陷阱一：配置的全局可变性

```rust
// ❌ 危险：任何地方都能改配置
pub static CONFIG: LazyLock<RwLock<Config>> = ...;

fn somewhere() {
    CONFIG.write().unwrap().server.port = 0;  // 谁干的？！
}
```

解决方案：只暴露只读接口

```rust
// ✅ 安全：只能读，不能写
mod config {
    static CONFIG: LazyLock<ArcSwap<Config>> = ...;

    pub fn get() -> Arc<Config> {
        CONFIG.load_full()
    }

    // reload 是模块内部函数，外部无法调用
    pub(crate) fn reload() -> Result<(), ConfigError> { ... }
}
```

### 陷阱二：配置验证放错位置

```rust
// ❌ 错误：使用时才验证
fn connect_database() {
    let config = config();
    if config.database.port == 0 {
        panic!("Invalid port");  // 太晚了！
    }
}

// ✅ 正确：加载时就验证
fn load_config() -> Result<Config, ConfigError> {
    let raw: RawConfig = toml::from_str(&content)?;
    validate(&raw)?;  // 启动时就发现问题
    Ok(raw.into())
}

fn validate(config: &RawConfig) -> Result<(), ConfigError> {
    if config.database.port == 0 {
        return Err(ConfigError::InvalidPort);
    }
    if config.server.timeout_seconds == 0 {
        return Err(ConfigError::InvalidTimeout);
    }
    Ok(())
}
```

### 陷阱三：敏感信息明文存储

```toml
# ❌ 危险：密码明文
[database]
password = "super_secret_123"
```

解决方案：

```rust
// 方案 1：环境变量
// config.toml
[database]
password = "${DB_PASSWORD}"  # 占位符

// 方案 2：密钥管理服务
// config.toml
[database]
password_secret = "aws:secretsmanager:prod/db/password"

// 代码中解析
impl Config {
    async fn resolve_secrets(&mut self) -> Result<(), Error> {
        if let Some(secret_key) = &self.database.password_secret {
            self.database.password = fetch_secret(secret_key).await?;
        }
        Ok(())
    }
}
```

### 陷阱四：测试中的全局状态

```rust
// ❌ 测试相互影响
#[test]
fn test_a() {
    init_config("config_a.toml");
    assert_eq!(config().server.port, 8080);
}

#[test]
fn test_b() {
    init_config("config_b.toml");  // 可能失败，因为已经初始化过了
}
```

解决方案：测试使用依赖注入

```rust
// ✅ 测试友好的设计
struct App {
    config: Arc<Config>,
}

impl App {
    fn new(config: Config) -> Self {
        Self { config: Arc::new(config) }
    }
}

#[test]
fn test_app() {
    let config = Config::builder()
        .server_port(8080)
        .build()
        .unwrap();

    let app = App::new(config);
    // 每个测试有独立的配置
}
```

## 完整实现

把上面的演进整合起来：

```rust
use std::sync::Arc;
use std::path::{Path, PathBuf};
use std::time::Duration;
use arc_swap::ArcSwap;
use serde::Deserialize;
use notify::{Watcher, RecursiveMode, Event, EventKind};

// ==================== 配置结构 ====================

#[derive(Debug, Clone, Deserialize)]
pub struct Config {
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub logging: LoggingConfig,
}

#[derive(Debug, Clone, Deserialize)]
pub struct ServerConfig {
    #[serde(default = "default_host")]
    pub host: String,
    #[serde(default = "default_port")]
    pub port: u16,
    #[serde(default = "default_max_connections")]
    pub max_connections: usize,
    #[serde(default = "default_timeout")]
    #[serde(with = "humantime_serde")] // 支持 "30s", "5m" 这样的格式
    pub timeout: Duration,
}

#[derive(Debug, Clone, Deserialize)]
pub struct DatabaseConfig {
    pub host: String,
    pub port: u16,
    pub username: String,
    #[serde(default)]
    pub password: Option<String>,
    #[serde(default)]
    pub password_env: Option<String>,
    #[serde(default = "default_pool_size")]
    pub max_pool_size: u32,
}

#[derive(Debug, Clone, Deserialize)]
pub struct LoggingConfig {
    #[serde(default = "default_log_level")]
    pub level: String,
    #[serde(default = "default_log_format")]
    pub format: String,
}

// 默认值函数
fn default_host() -> String { "0.0.0.0".to_string() }
fn default_port() -> u16 { 8080 }
fn default_max_connections() -> usize { 1000 }
fn default_timeout() -> Duration { Duration::from_secs(30) }
fn default_pool_size() -> u32 { 20 }
fn default_log_level() -> String { "info".to_string() }
fn default_log_format() -> String { "json".to_string() }

// ==================== 错误类型 ====================

#[derive(Debug)]
pub enum ConfigError {
    Io(std::io::Error),
    Parse(toml::de::Error),
    Validation(String),
    SecretResolution(String),
}

impl std::fmt::Display for ConfigError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ConfigError::Io(e) => write!(f, "IO error: {}", e),
            ConfigError::Parse(e) => write!(f, "Parse error: {}", e),
            ConfigError::Validation(msg) => write!(f, "Validation error: {}", msg),
            ConfigError::SecretResolution(msg) => write!(f, "Secret resolution error: {}", msg),
        }
    }
}

impl std::error::Error for ConfigError {}

// ==================== 配置管理器 ====================

pub struct ConfigManager {
    config: Arc<ArcSwap<Config>>,  // Arc 包装以支持跨线程共享
    config_path: PathBuf,
}

impl ConfigManager {
    pub fn new(path: impl AsRef<Path>) -> Result<Self, ConfigError> {
        let config = Self::load_from_path(path.as_ref())?;
        Ok(Self {
            config: Arc::new(ArcSwap::from_pointee(config)),
            config_path: path.as_ref().to_path_buf(),
        })
    }

    pub fn get(&self) -> Arc<Config> {
        self.config.load_full()
    }

    pub fn reload(&self) -> Result<(), ConfigError> {
        let new_config = Self::load_from_path(&self.config_path)?;
        self.config.store(Arc::new(new_config));
        Ok(())
    }

    fn load_from_path(path: &Path) -> Result<Config, ConfigError> {
        let content = std::fs::read_to_string(path)
            .map_err(ConfigError::Io)?;

        let mut config: Config = toml::from_str(&content)
            .map_err(ConfigError::Parse)?;

        // 解析环境变量中的密码
        if let Some(env_var) = &config.database.password_env {
            config.database.password = std::env::var(env_var).ok();
        }

        // 验证配置
        Self::validate(&config)?;

        Ok(config)
    }

    fn validate(config: &Config) -> Result<(), ConfigError> {
        if config.server.port == 0 {
            return Err(ConfigError::Validation("Server port cannot be 0".into()));
        }
        if config.database.host.is_empty() {
            return Err(ConfigError::Validation("Database host is required".into()));
        }
        Ok(())
    }

    pub fn watch(&self) -> Result<(), notify::Error> {
        let config_ref = Arc::clone(&self.config);
        let path = self.config_path.clone();

        std::thread::spawn(move || {
            let (tx, rx) = std::sync::mpsc::channel();

            let mut watcher = notify::recommended_watcher(move |res: Result<Event, _>| {
                if let Ok(event) = res {
                    if matches!(event.kind, EventKind::Modify(_)) {
                        let _ = tx.send(());
                    }
                }
            }).expect("Failed to create watcher");

            watcher.watch(&path, RecursiveMode::NonRecursive)
                .expect("Failed to watch config file");

            for _ in rx {
                match Self::load_from_path(&path) {
                    Ok(new_config) => {
                        config_ref.store(Arc::new(new_config));
                        println!("Config reloaded successfully");
                    }
                    Err(e) => {
                        eprintln!("Failed to reload config: {}", e);
                    }
                }
            }
        });

        Ok(())
    }
}

// ==================== 全局实例（可选） ====================

use std::sync::LazyLock;

static GLOBAL_CONFIG: LazyLock<ConfigManager> = LazyLock::new(|| {
    let path = std::env::var("CONFIG_PATH")
        .unwrap_or_else(|_| "config.toml".to_string());
    ConfigManager::new(&path).expect("Failed to initialize config")
});

pub fn config() -> Arc<Config> {
    GLOBAL_CONFIG.get()
}

pub fn reload_config() -> Result<(), ConfigError> {
    GLOBAL_CONFIG.reload()
}

pub fn watch_config() -> Result<(), notify::Error> {
    GLOBAL_CONFIG.watch()
}
```

## 小结

一个"简单"的配置系统，让我们自然地用到了三种设计模式：

| 模式 | 解决的问题 | Rust 实现 |
|------|-----------|----------|
| 单例 | 全局唯一配置实例 | `LazyLock` + `ArcSwap` |
| 建造者 | 复杂配置的构建 | Builder 类型 / `derive_builder` |
| 原型 | 配置的派生与继承 | `Clone` + 配置层叠 |

这些模式不是我们刻意去"用"的，而是在解决实际问题的过程中自然浮现的。

记住：**模式是解决方案的名字，不是必须遵循的模板**。

下一篇，我们看另一个常见场景：**插件架构**。当你的系统需要支持第三方扩展时，工厂模式和桥接模式就该登场了。

---

> **系列导航**
>
> - 第一篇：Rust 让设计模式返璞归真
> - **第二篇：配置系统——从硬编码到热加载**（本文）
> - 第三篇：插件架构——让系统长出翅膀
> - 第四篇：数据管道——像乐高一样组装逻辑
> - 第五篇：状态流转——订单不只是 CRUD
> - 第六篇：事件驱动——模块间的优雅对话
> - 第七篇：外部集成——与第三方和平共处
> - 第八篇：框架设计——把控制权还给用户
> - 第九篇：树形世界——递归结构的驯服术
> - 第十篇：规则引擎——让业务人员写代码
> - 第十一篇：终篇——反模式与断舍离
