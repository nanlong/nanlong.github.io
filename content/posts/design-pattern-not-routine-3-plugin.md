---
title: "模式不是套路（三）：插件架构——让系统长出翅膀"
date: 2025-12-10T10:20:00+08:00
draft: false
tags: ["Rust", "设计模式", "插件架构", "工厂模式", "桥接模式"]
categories: ["技术"]
description: "核心功能写死，扩展功能靠改代码？当系统需要支持第三方扩展时，工厂模式和桥接模式就该登场了。看 Rust 如何用 trait 和动态库实现优雅的插件系统。"
series: ["模式不是套路"]
---

核心功能写死，扩展功能靠改代码？当系统需要支持第三方扩展时，工厂模式和桥接模式就该登场了。看 Rust 如何用 trait 和动态库实现优雅的插件系统。

<!--more-->

## 问题：一个不断膨胀的系统

你在做一个数据处理平台。最初只支持 CSV 文件：

```rust
fn process_file(path: &str) -> Result<DataFrame, Error> {
    let content = std::fs::read_to_string(path)?;
    parse_csv(&content)
}
```

很快，产品经理来了：

```text
第一周："我们需要支持 JSON"
第二周："能不能加上 Excel？"
第一个月："合作方发的是 Parquet 格式"
第二个月："有些老系统还在用 XML"
第三个月："客户要对接他们自己的数据格式"
```

代码开始变成这样：

```rust
fn process_file(path: &str) -> Result<DataFrame, Error> {
    let extension = Path::new(path).extension()
        .and_then(|e| e.to_str())
        .unwrap_or("");

    match extension {
        "csv" => parse_csv(path),
        "json" => parse_json(path),
        "xlsx" => parse_excel(path),
        "parquet" => parse_parquet(path),
        "xml" => parse_xml(path),
        _ => Err(Error::UnsupportedFormat),
    }
}
```

问题越来越明显：

```text
1. 每加一种格式，都要改核心代码
2. 所有格式的依赖都要编译进去（xlsx 库很大！）
3. 客户想加自定义格式？让他们 fork 代码？
4. 不同格式的解析逻辑混在一起，维护困难
```

**你需要的是一个插件系统**：核心平台稳定，格式支持可以动态扩展。

## 朴素方案：硬编码工厂

最直接的改进是把创建逻辑抽出来：

```rust
trait DataParser: Send + Sync {
    fn parse(&self, content: &[u8]) -> Result<DataFrame, Error>;
    fn supported_extensions(&self) -> &[&str];
}

struct CsvParser;
impl DataParser for CsvParser {
    fn parse(&self, content: &[u8]) -> Result<DataFrame, Error> {
        // CSV 解析逻辑
        todo!()
    }
    fn supported_extensions(&self) -> &[&str] { &["csv"] }
}

struct JsonParser;
impl DataParser for JsonParser {
    fn parse(&self, content: &[u8]) -> Result<DataFrame, Error> {
        // JSON 解析逻辑
        todo!()
    }
    fn supported_extensions(&self) -> &[&str] { &["json"] }
}

// 工厂函数
fn create_parser(extension: &str) -> Option<Box<dyn DataParser>> {
    match extension {
        "csv" => Some(Box::new(CsvParser)),
        "json" => Some(Box::new(JsonParser)),
        // 每加一种格式，还是要改这里
        _ => None,
    }
}
```

这就是**工厂方法模式**的雏形：**定义一个创建对象的接口，让子类决定实例化哪一个类**。

但问题没有根本解决——`create_parser` 函数还是要硬编码所有格式。

## 第一次演进：注册式工厂

让解析器自己注册：

```rust
use std::collections::HashMap;
use std::sync::RwLock;

type ParserFactory = fn() -> Box<dyn DataParser>;

static REGISTRY: LazyLock<RwLock<HashMap<String, ParserFactory>>> =
    LazyLock::new(|| RwLock::new(HashMap::new()));

pub fn register_parser(extension: &str, factory: ParserFactory) {
    REGISTRY.write().unwrap()
        .insert(extension.to_string(), factory);
}

pub fn create_parser(extension: &str) -> Option<Box<dyn DataParser>> {
    REGISTRY.read().unwrap()
        .get(extension)
        .map(|factory| factory())
}
```

每个解析器模块负责自己注册：

```rust
// csv_parser.rs
pub fn init() {
    register_parser("csv", || Box::new(CsvParser));
}

// json_parser.rs
pub fn init() {
    register_parser("json", || Box::new(JsonParser));
}

// main.rs
fn main() {
    csv_parser::init();
    json_parser::init();
    // 现在可以使用了
}
```

更进一步，用 Rust 的 `inventory` crate 实现自动注册：

```rust
use inventory;

pub struct ParserRegistration {
    pub extension: &'static str,
    pub factory: fn() -> Box<dyn DataParser>,
}

inventory::collect!(ParserRegistration);

pub fn create_parser(extension: &str) -> Option<Box<dyn DataParser>> {
    for reg in inventory::iter::<ParserRegistration> {
        if reg.extension == extension {
            return Some((reg.factory)());
        }
    }
    None
}

// csv_parser.rs
inventory::submit! {
    ParserRegistration {
        extension: "csv",
        factory: || Box::new(CsvParser),
    }
}
```

`inventory` 利用链接器在编译时收集所有注册项，不需要手动调用 `init()`。

## 第二次演进：抽象工厂

系统继续演进，需求变了：

```text
不仅要支持不同的文件格式，还要支持不同的存储后端：
  - 本地文件系统
  - S3
  - HDFS
  - FTP
```

现在有两个维度的变化：**格式** × **存储**。

如果用之前的方式，要写 `格式数 × 存储数` 个解析器。

这时候需要**抽象工厂模式**：**提供一个接口，用于创建相关或依赖对象的家族**。

```rust
// 存储抽象
trait Storage: Send + Sync {
    fn read(&self, path: &str) -> Result<Vec<u8>, Error>;
    fn write(&self, path: &str, data: &[u8]) -> Result<(), Error>;
    fn list(&self, prefix: &str) -> Result<Vec<String>, Error>;
}

// 解析器抽象
trait Parser: Send + Sync {
    fn parse(&self, data: &[u8]) -> Result<DataFrame, Error>;
    fn serialize(&self, df: &DataFrame) -> Result<Vec<u8>, Error>;
}

// 抽象工厂：创建一组相关对象
trait DataSourceFactory: Send + Sync {
    fn create_storage(&self) -> Box<dyn Storage>;
    fn create_parser(&self) -> Box<dyn Parser>;
    fn name(&self) -> &str;
}

// 具体工厂：本地 CSV
struct LocalCsvFactory;

impl DataSourceFactory for LocalCsvFactory {
    fn create_storage(&self) -> Box<dyn Storage> {
        Box::new(LocalFileStorage::new())
    }

    fn create_parser(&self) -> Box<dyn Parser> {
        Box::new(CsvParser::new())
    }

    fn name(&self) -> &str { "local-csv" }
}

// 具体工厂：S3 Parquet
struct S3ParquetFactory {
    bucket: String,
    region: String,
}

impl DataSourceFactory for S3ParquetFactory {
    fn create_storage(&self) -> Box<dyn Storage> {
        Box::new(S3Storage::new(&self.bucket, &self.region))
    }

    fn create_parser(&self) -> Box<dyn Parser> {
        Box::new(ParquetParser::new())
    }

    fn name(&self) -> &str { "s3-parquet" }
}
```

客户端代码不需要知道具体用的是什么存储和格式：

```rust
struct DataProcessor {
    factory: Box<dyn DataSourceFactory>,
}

impl DataProcessor {
    fn new(factory: Box<dyn DataSourceFactory>) -> Self {
        Self { factory }
    }

    fn process(&self, path: &str) -> Result<DataFrame, Error> {
        let storage = self.factory.create_storage();
        let parser = self.factory.create_parser();

        let data = storage.read(path)?;
        parser.parse(&data)
    }
}

// 使用
let processor = DataProcessor::new(Box::new(S3ParquetFactory {
    bucket: "my-bucket".to_string(),
    region: "us-west-2".to_string(),
}));

let df = processor.process("data/sales.parquet")?;
```

## 第三次演进：桥接模式

抽象工厂有个问题：**每种组合都要写一个工厂类**。

如果有 5 种存储 × 5 种格式，需要 25 个工厂类。这不现实。

**桥接模式**的思路是：**将抽象与实现分离，使它们可以独立变化**。

```rust
// 存储实现（独立变化）
trait Storage: Send + Sync {
    fn read(&self, path: &str) -> Result<Vec<u8>, Error>;
    fn write(&self, path: &str, data: &[u8]) -> Result<(), Error>;
}

struct LocalStorage;
struct S3Storage { bucket: String, region: String }
struct HdfsStorage { namenode: String }

impl Storage for LocalStorage { /* ... */ }
impl Storage for S3Storage { /* ... */ }
impl Storage for HdfsStorage { /* ... */ }

// 格式实现（独立变化）
trait Parser: Send + Sync {
    fn parse(&self, data: &[u8]) -> Result<DataFrame, Error>;
    fn serialize(&self, df: &DataFrame) -> Result<Vec<u8>, Error>;
}

struct CsvParser;
struct JsonParser;
struct ParquetParser;

impl Parser for CsvParser { /* ... */ }
impl Parser for JsonParser { /* ... */ }
impl Parser for ParquetParser { /* ... */ }

// 桥接：组合存储和格式
struct DataSource {
    storage: Box<dyn Storage>,
    parser: Box<dyn Parser>,
}

impl DataSource {
    fn new(storage: impl Storage + 'static, parser: impl Parser + 'static) -> Self {
        Self {
            storage: Box::new(storage),
            parser: Box::new(parser),
        }
    }

    fn read(&self, path: &str) -> Result<DataFrame, Error> {
        let data = self.storage.read(path)?;
        self.parser.parse(&data)
    }

    fn write(&self, path: &str, df: &DataFrame) -> Result<(), Error> {
        let data = self.parser.serialize(df)?;
        self.storage.write(path, &data)
    }
}
```

现在可以自由组合：

```rust
// 本地 CSV
let source = DataSource::new(LocalStorage, CsvParser);

// S3 Parquet
let source = DataSource::new(
    S3Storage { bucket: "data".into(), region: "us-west-2".into() },
    ParquetParser,
);

// HDFS JSON（新组合，不需要新工厂类）
let source = DataSource::new(
    HdfsStorage { namenode: "hdfs://namenode:9000".into() },
    JsonParser,
);
```

5 种存储 + 5 种格式 = 10 个实现类，而不是 25 个工厂类。

### Rust 的礼物：泛型桥接

用泛型可以避免动态分发的开销：

```rust
struct DataSource<S: Storage, P: Parser> {
    storage: S,
    parser: P,
}

impl<S: Storage, P: Parser> DataSource<S, P> {
    fn new(storage: S, parser: P) -> Self {
        Self { storage, parser }
    }

    fn read(&self, path: &str) -> Result<DataFrame, Error> {
        let data = self.storage.read(path)?;
        self.parser.parse(&data)
    }
}

// 编译时确定类型，零运行时开销
let source: DataSource<LocalStorage, CsvParser> = DataSource::new(LocalStorage, CsvParser);
```

需要动态选择时再用 trait object：

```rust
fn create_source(storage_type: &str, format: &str) -> Box<dyn DataSourceTrait> {
    let storage: Box<dyn Storage> = match storage_type {
        "local" => Box::new(LocalStorage),
        "s3" => Box::new(S3Storage::default()),
        _ => panic!("Unknown storage"),
    };

    let parser: Box<dyn Parser> = match format {
        "csv" => Box::new(CsvParser),
        "json" => Box::new(JsonParser),
        _ => panic!("Unknown format"),
    };

    Box::new(DynamicDataSource { storage, parser })
}
```

## 第四次演进：动态加载插件

终极需求：**让第三方在不修改主程序的情况下添加新格式**。

这需要动态库（`.so`/`.dll`/`.dylib`）支持。

### 定义插件接口

```rust
// plugin_api/src/lib.rs
// 这个 crate 会被主程序和所有插件依赖

pub trait Plugin: Send + Sync {
    fn name(&self) -> &str;
    fn version(&self) -> &str;
}

pub trait ParserPlugin: Plugin {
    fn supported_extensions(&self) -> Vec<String>;
    fn parse(&self, data: &[u8]) -> Result<DataFrame, PluginError>;
    fn serialize(&self, df: &DataFrame) -> Result<Vec<u8>, PluginError>;
}

#[derive(Debug)]
pub struct PluginError {
    pub message: String,
}

// 插件必须导出这个函数
pub type CreateParserFn = unsafe extern "C" fn() -> *mut dyn ParserPlugin;

#[macro_export]
macro_rules! declare_plugin {
    ($plugin_type:ty, $constructor:expr) => {
        #[no_mangle]
        pub extern "C" fn _create_parser() -> *mut dyn $crate::ParserPlugin {
            let plugin = $constructor;
            let boxed: Box<dyn $crate::ParserPlugin> = Box::new(plugin);
            Box::into_raw(boxed)
        }
    };
}
```

### 实现一个插件

```rust
// yaml_plugin/src/lib.rs
use plugin_api::{ParserPlugin, Plugin, PluginError, declare_plugin};

pub struct YamlParser;

impl Plugin for YamlParser {
    fn name(&self) -> &str { "YAML Parser" }
    fn version(&self) -> &str { "1.0.0" }
}

impl ParserPlugin for YamlParser {
    fn supported_extensions(&self) -> Vec<String> {
        vec!["yaml".to_string(), "yml".to_string()]
    }

    fn parse(&self, data: &[u8]) -> Result<DataFrame, PluginError> {
        let content = std::str::from_utf8(data)
            .map_err(|e| PluginError { message: e.to_string() })?;
        // YAML 解析逻辑
        todo!()
    }

    fn serialize(&self, df: &DataFrame) -> Result<Vec<u8>, PluginError> {
        // YAML 序列化逻辑
        todo!()
    }
}

declare_plugin!(YamlParser, YamlParser);
```

```toml
# yaml_plugin/Cargo.toml
[lib]
crate-type = ["cdylib"]  # 编译为动态库
```

### 主程序加载插件

```rust
// main_app/src/plugin_loader.rs
use libloading::{Library, Symbol};
use plugin_api::{ParserPlugin, CreateParserFn};
use std::path::Path;
use std::collections::HashMap;
use std::sync::Arc;

pub struct PluginManager {
    libraries: Vec<Library>,  // 保持库加载状态
    parsers: HashMap<String, Arc<dyn ParserPlugin>>,
}

impl PluginManager {
    pub fn new() -> Self {
        Self {
            libraries: Vec::new(),
            parsers: HashMap::new(),
        }
    }

    pub fn load_plugin(&mut self, path: &Path) -> Result<(), PluginLoadError> {
        unsafe {
            // 加载动态库
            let lib = Library::new(path)
                .map_err(|e| PluginLoadError::LoadFailed(e.to_string()))?;

            // 获取创建函数
            let create_fn: Symbol<CreateParserFn> = lib.get(b"_create_parser")
                .map_err(|e| PluginLoadError::SymbolNotFound(e.to_string()))?;

            // 创建插件实例
            let raw = create_fn();
            let plugin: Box<dyn ParserPlugin> = Box::from_raw(raw);
            let plugin = Arc::from(plugin);

            // 注册支持的扩展名
            for ext in plugin.supported_extensions() {
                self.parsers.insert(ext, Arc::clone(&plugin));
            }

            println!("Loaded plugin: {} v{}", plugin.name(), plugin.version());

            // 保持库加载状态
            self.libraries.push(lib);
        }

        Ok(())
    }

    pub fn load_plugins_from_dir(&mut self, dir: &Path) -> Result<(), PluginLoadError> {
        let extension = if cfg!(target_os = "windows") {
            "dll"
        } else if cfg!(target_os = "macos") {
            "dylib"
        } else {
            "so"
        };

        for entry in std::fs::read_dir(dir)? {
            let entry = entry?;
            let path = entry.path();

            if path.extension().map_or(false, |e| e == extension) {
                if let Err(e) = self.load_plugin(&path) {
                    eprintln!("Failed to load plugin {:?}: {:?}", path, e);
                }
            }
        }

        Ok(())
    }

    pub fn get_parser(&self, extension: &str) -> Option<Arc<dyn ParserPlugin>> {
        self.parsers.get(extension).cloned()
    }
}

#[derive(Debug)]
pub enum PluginLoadError {
    LoadFailed(String),
    SymbolNotFound(String),
    IoError(std::io::Error),
}

impl From<std::io::Error> for PluginLoadError {
    fn from(e: std::io::Error) -> Self {
        PluginLoadError::IoError(e)
    }
}
```

使用：

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut plugins = PluginManager::new();

    // 从 plugins 目录加载所有插件
    plugins.load_plugins_from_dir(Path::new("./plugins"))?;

    // 使用插件
    if let Some(parser) = plugins.get_parser("yaml") {
        let data = std::fs::read("config.yaml")?;
        let df = parser.parse(&data)?;
        println!("Parsed {} rows", df.len());
    }

    Ok(())
}
```

### 安全考虑

动态加载插件有安全风险。在生产环境中：

```rust
impl PluginManager {
    pub fn load_plugin_verified(&mut self, path: &Path) -> Result<(), PluginLoadError> {
        // 1. 验证签名
        let signature_path = path.with_extension("sig");
        if !verify_signature(path, &signature_path)? {
            return Err(PluginLoadError::InvalidSignature);
        }

        // 2. 检查版本兼容性
        let metadata = read_plugin_metadata(path)?;
        if !is_compatible(&metadata) {
            return Err(PluginLoadError::IncompatibleVersion);
        }

        // 3. 沙箱加载（如果平台支持）
        self.load_plugin(path)
    }
}
```

## 陷阱：别掉进这些坑

### 陷阱一：过度抽象

```rust
// ❌ 太多层次，难以理解
trait StorageProvider {
    fn create_storage(&self) -> Box<dyn Storage>;
}

trait StorageProviderFactory {
    fn create_provider(&self) -> Box<dyn StorageProvider>;
}

trait StorageProviderFactoryRegistry {
    fn get_factory(&self, name: &str) -> Box<dyn StorageProviderFactory>;
}
```

如果你的抽象层次超过 3 层，很可能是过度设计了。

```rust
// ✅ 保持简单
fn get_storage(storage_type: &str) -> Box<dyn Storage> {
    match storage_type {
        "local" => Box::new(LocalStorage),
        "s3" => Box::new(S3Storage::from_env()),
        _ => panic!("Unknown storage type"),
    }
}
```

### 陷阱二：插件间的隐式依赖

```rust
// ❌ 插件 A 依赖插件 B 的内部实现
// plugin_a.rs
fn process(&self, data: &[u8]) -> Result<DataFrame, Error> {
    // 假设 plugin_b 已经加载并处理过数据
    let cached = GLOBAL_CACHE.get("plugin_b_result");  // 危险！
    // ...
}
```

插件应该是独立的：

```rust
// ✅ 通过明确的接口传递依赖
trait ParserPlugin {
    fn parse(&self, data: &[u8], context: &PluginContext) -> Result<DataFrame, Error>;
}

struct PluginContext {
    pub config: Arc<Config>,
    pub cache: Arc<dyn Cache>,
    pub logger: Arc<dyn Logger>,
}
```

### 陷阱三：版本兼容性

```rust
// v1.0 的接口
trait ParserPlugin {
    fn parse(&self, data: &[u8]) -> Result<DataFrame, Error>;
}

// v2.0 添加了新方法
trait ParserPlugin {
    fn parse(&self, data: &[u8]) -> Result<DataFrame, Error>;
    fn parse_streaming(&self, reader: &mut dyn Read) -> Result<DataFrame, Error>;  // 新增
}
```

旧插件没实现 `parse_streaming`，加载时会崩溃。

解决方案：

```rust
// 方案 1：默认实现
trait ParserPlugin {
    fn parse(&self, data: &[u8]) -> Result<DataFrame, Error>;

    fn parse_streaming(&self, reader: &mut dyn Read) -> Result<DataFrame, Error> {
        // 默认实现：读取全部数据后调用 parse
        let mut data = Vec::new();
        reader.read_to_end(&mut data)?;
        self.parse(&data)
    }
}

// 方案 2：能力查询
trait ParserPlugin {
    fn capabilities(&self) -> Capabilities;
    fn parse(&self, data: &[u8]) -> Result<DataFrame, Error>;
    fn parse_streaming(&self, reader: &mut dyn Read) -> Result<DataFrame, Error>;
}

struct Capabilities {
    pub supports_streaming: bool,
    pub supports_schema_inference: bool,
}
```

### 陷阱四：动态库的内存安全

```rust
// ❌ 危险：String 跨 FFI 边界
extern "C" fn create_parser(name: *const c_char) -> *mut dyn ParserPlugin {
    let name = unsafe { CStr::from_ptr(name) };  // 可能崩溃
    // ...
}

// ✅ 安全：使用 C 兼容的类型
#[repr(C)]
pub struct PluginInfo {
    pub name: [c_char; 64],
    pub version: [c_char; 16],
}

extern "C" fn get_info() -> PluginInfo {
    // 返回固定大小的结构
}
```

**重要警告**：上述示例是简化版本，仅供理解概念。在生产环境中，**必须**使用 `abi_stable` crate。原因是 Rust 没有稳定的 ABI，不同编译器版本、不同编译选项生成的动态库，其 trait object 内存布局可能不同，直接跨 FFI 传递 trait object 是未定义行为。

`abi_stable` crate 通过以下方式解决这个问题：
- 定义了稳定的 vtable 布局
- 提供了 `#[sabi_trait]` 宏自动生成 FFI 安全的 trait
- 确保跨 Rust 版本的二进制兼容性

## 完整示例：可扩展的数据处理平台

```rust
use std::collections::HashMap;
use std::sync::Arc;
use std::path::Path;

// ==================== 核心 Trait ====================

pub trait Storage: Send + Sync {
    fn read(&self, path: &str) -> Result<Vec<u8>, StorageError>;
    fn write(&self, path: &str, data: &[u8]) -> Result<(), StorageError>;
    fn exists(&self, path: &str) -> Result<bool, StorageError>;
}

pub trait Parser: Send + Sync {
    fn name(&self) -> &str;
    fn extensions(&self) -> &[&str];
    fn parse(&self, data: &[u8]) -> Result<DataFrame, ParseError>;
    fn serialize(&self, df: &DataFrame) -> Result<Vec<u8>, ParseError>;
}

// ==================== 注册表 ====================

pub struct Registry {
    storages: HashMap<String, Arc<dyn Storage>>,
    parsers: HashMap<String, Arc<dyn Parser>>,
}

impl Registry {
    pub fn new() -> Self {
        Self {
            storages: HashMap::new(),
            parsers: HashMap::new(),
        }
    }

    pub fn register_storage(&mut self, name: &str, storage: impl Storage + 'static) {
        self.storages.insert(name.to_string(), Arc::new(storage));
    }

    pub fn register_parser(&mut self, parser: impl Parser + 'static) {
        let parser = Arc::new(parser);
        for ext in parser.extensions() {
            self.parsers.insert(ext.to_string(), Arc::clone(&parser));
        }
    }

    pub fn get_storage(&self, name: &str) -> Option<Arc<dyn Storage>> {
        self.storages.get(name).cloned()
    }

    pub fn get_parser(&self, extension: &str) -> Option<Arc<dyn Parser>> {
        self.parsers.get(extension).cloned()
    }
}

// ==================== 数据源（桥接） ====================

pub struct DataSource {
    storage: Arc<dyn Storage>,
    registry: Arc<Registry>,
}

impl DataSource {
    pub fn new(storage: Arc<dyn Storage>, registry: Arc<Registry>) -> Self {
        Self { storage, registry }
    }

    pub fn read(&self, path: &str) -> Result<DataFrame, DataSourceError> {
        // 根据扩展名选择解析器
        let ext = Path::new(path)
            .extension()
            .and_then(|e| e.to_str())
            .ok_or(DataSourceError::UnknownFormat)?;

        let parser = self.registry.get_parser(ext)
            .ok_or(DataSourceError::UnsupportedFormat(ext.to_string()))?;

        let data = self.storage.read(path)?;
        let df = parser.parse(&data)?;

        Ok(df)
    }

    pub fn write(&self, path: &str, df: &DataFrame) -> Result<(), DataSourceError> {
        let ext = Path::new(path)
            .extension()
            .and_then(|e| e.to_str())
            .ok_or(DataSourceError::UnknownFormat)?;

        let parser = self.registry.get_parser(ext)
            .ok_or(DataSourceError::UnsupportedFormat(ext.to_string()))?;

        let data = parser.serialize(df)?;
        self.storage.write(path, &data)?;

        Ok(())
    }
}

// ==================== 内置实现 ====================

pub struct LocalStorage {
    root: std::path::PathBuf,
}

impl LocalStorage {
    pub fn new(root: impl AsRef<Path>) -> Self {
        Self { root: root.as_ref().to_path_buf() }
    }
}

impl Storage for LocalStorage {
    fn read(&self, path: &str) -> Result<Vec<u8>, StorageError> {
        let full_path = self.root.join(path);
        std::fs::read(&full_path)
            .map_err(|e| StorageError::Io(e.to_string()))
    }

    fn write(&self, path: &str, data: &[u8]) -> Result<(), StorageError> {
        let full_path = self.root.join(path);
        if let Some(parent) = full_path.parent() {
            std::fs::create_dir_all(parent)
                .map_err(|e| StorageError::Io(e.to_string()))?;
        }
        std::fs::write(&full_path, data)
            .map_err(|e| StorageError::Io(e.to_string()))
    }

    fn exists(&self, path: &str) -> Result<bool, StorageError> {
        Ok(self.root.join(path).exists())
    }
}

pub struct CsvParser;

impl Parser for CsvParser {
    fn name(&self) -> &str { "CSV Parser" }
    fn extensions(&self) -> &[&str] { &["csv"] }

    fn parse(&self, data: &[u8]) -> Result<DataFrame, ParseError> {
        // CSV 解析逻辑
        todo!("Implement CSV parsing")
    }

    fn serialize(&self, df: &DataFrame) -> Result<Vec<u8>, ParseError> {
        // CSV 序列化逻辑
        todo!("Implement CSV serialization")
    }
}

// ==================== 使用示例 ====================

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建注册表
    let mut registry = Registry::new();

    // 注册存储后端
    registry.register_storage("local", LocalStorage::new("."));

    // 注册内置解析器
    registry.register_parser(CsvParser);

    // TODO: 从插件目录加载更多解析器
    // plugin_loader.load_plugins_from_dir("./plugins", &mut registry)?;

    let registry = Arc::new(registry);

    // 创建数据源
    let storage = registry.get_storage("local").unwrap();
    let source = DataSource::new(storage, Arc::clone(&registry));

    // 读取数据（自动根据扩展名选择解析器）
    // let df = source.read("data/sales.csv")?;

    Ok(())
}

// ==================== 错误类型 ====================

#[derive(Debug)]
pub enum StorageError {
    Io(String),
    NotFound(String),
}

#[derive(Debug)]
pub enum ParseError {
    InvalidFormat(String),
    Io(String),
}

#[derive(Debug)]
pub enum DataSourceError {
    Storage(StorageError),
    Parse(ParseError),
    UnknownFormat,
    UnsupportedFormat(String),
}

impl From<StorageError> for DataSourceError {
    fn from(e: StorageError) -> Self {
        DataSourceError::Storage(e)
    }
}

impl From<ParseError> for DataSourceError {
    fn from(e: ParseError) -> Self {
        DataSourceError::Parse(e)
    }
}

// 占位类型
pub struct DataFrame;
impl DataFrame {
    pub fn len(&self) -> usize { 0 }
}
```

## 小结

构建一个插件系统，我们用到了三种模式：

| 模式 | 解决的问题 | Rust 实现 |
|------|-----------|----------|
| 工厂方法 | 对象创建的解耦 | trait + 注册表 / `inventory` |
| 抽象工厂 | 相关对象家族的创建 | trait 组合 |
| 桥接 | 抽象与实现的独立变化 | 泛型 + trait object |

关键洞见：

1. **注册式工厂**比硬编码 `match` 更灵活
2. **桥接模式**避免组合爆炸（M × N → M + N）
3. **动态加载**实现真正的运行时扩展
4. **保持简单**——不要为了"可能的扩展"过度设计

下一篇，我们看数据处理场景：**数据管道——像乐高一样组装逻辑**。装饰器、责任链、策略，这些模式如何帮你构建灵活的处理流程。

---

> **系列导航**
>
> - 第一篇：Rust 让设计模式返璞归真
> - 第二篇：配置系统——从硬编码到热加载
> - **第三篇：插件架构——让系统长出翅膀**（本文）
> - 第四篇：数据管道——像乐高一样组装逻辑
> - 第五篇：状态流转——订单不只是 CRUD
> - 第六篇：事件驱动——模块间的优雅对话
> - 第七篇：外部集成——与第三方和平共处
> - 第八篇：框架设计——把控制权还给用户
> - 第九篇：树形世界——递归结构的驯服术
> - 第十篇：规则引擎——让业务人员写代码
> - 第十一篇：终篇——反模式与断舍离
