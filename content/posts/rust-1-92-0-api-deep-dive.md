---
title: "Rust 1.92.0 新 API 深度解析"
date: 2025-12-15T00:00:00+08:00
draft: false
tags: ["Rust", "API", "并发", "智能指针", "版本更新"]
categories: ["技术"]
description: "Rust 1.92.0 带来了一批实用的新 API：RwLockWriteGuard::downgrade 终于稳定、零初始化智能指针、NonZero 的 div_ceil……这些 API 解决了哪些痛点？对 Rust 生态有什么影响？背后的设计哲学是什么？"
---

Rust 1.92.0 于 2025 年 12 月 11 日发布。这个版本没有惊天动地的新特性，但带来了一批**等了很久**的实用 API。作为一个写了几年 Rust 的开发者，看到这些 API 终于稳定，感觉就像是"终于不用自己造轮子了"。

本文将深入分析这些新 API 的实战价值、生态影响和设计哲学。

<!--more-->

## 总览：这个版本稳定了什么？

```text
┌─────────────────────────────────────────────────────────────────┐
│                    Rust 1.92.0 新稳定 API                        │
├─────────────────────────────────────────────────────────────────┤
│  并发相关                                                        │
│  • RwLockWriteGuard::downgrade                                  │
├─────────────────────────────────────────────────────────────────┤
│  智能指针零初始化                                                │
│  • Box::new_zeroed / new_zeroed_slice                           │
│  • Rc::new_zeroed / new_zeroed_slice                            │
│  • Arc::new_zeroed / new_zeroed_slice                           │
├─────────────────────────────────────────────────────────────────┤
│  数值计算                                                        │
│  • NonZero<u{N}>::div_ceil                                      │
├─────────────────────────────────────────────────────────────────┤
│  集合操作                                                        │
│  • btree_map::Entry::insert_entry                               │
│  • btree_map::VacantEntry::insert_entry                         │
├─────────────────────────────────────────────────────────────────┤
│  FFI / 调试                                                      │
│  • Location::file_as_c_str                                      │
├─────────────────────────────────────────────────────────────────┤
│  过程宏                                                          │
│  • TokenStream: Extend<Group/Literal/Punct/Ident>               │
├─────────────────────────────────────────────────────────────────┤
│  常量化                                                          │
│  • <[_]>::rotate_left / rotate_right (const)                    │
└─────────────────────────────────────────────────────────────────┘
```

让我们逐个深入。

## 一、RwLockWriteGuard::downgrade：等了九年的 API

### 这是什么？

```rust
impl<T: ?Sized> RwLockWriteGuard<'_, T> {
    pub fn downgrade(self) -> RwLockReadGuard<'_, T>;
}
```

简单说：**把写锁降级为读锁，而不释放锁**。

### 痛点：之前怎么做？

在 1.92.0 之前，如果你持有写锁，想切换到读锁，只能这样：

```rust
use std::sync::RwLock;

let lock = RwLock::new(42);

// 获取写锁，修改数据
{
    let mut write_guard = lock.write().unwrap();
    *write_guard = 100;
}  // 写锁释放

// 重新获取读锁
let read_guard = lock.read().unwrap();
println!("Value: {}", *read_guard);
```

**问题在哪？** 在 `}` 释放写锁和 `lock.read()` 获取读锁之间，存在一个**时间窗口**。其他线程可能在这个窗口里抢到写锁，修改数据。

这在某些场景下是致命的：

```rust
// 场景：初始化缓存
fn get_or_init(cache: &RwLock<Option<ExpensiveData>>) -> ExpensiveData {
    // 先尝试读
    {
        let read = cache.read().unwrap();
        if let Some(data) = &*read {
            return data.clone();
        }
    }  // 读锁释放

    // 没有数据，获取写锁初始化
    let mut write = cache.write().unwrap();

    // 问题：其他线程可能在我们释放读锁后、获取写锁前，已经初始化了！
    // 必须再次检查（双重检查锁定）
    if let Some(data) = &*write {
        return data.clone();
    }

    let data = expensive_computation();
    *write = Some(data.clone());
    data
    // 写锁释放后，其他线程才能读
}
```

这个模式叫**双重检查锁定**（Double-Checked Locking），代码繁琐且容易出错。

### downgrade 解决什么问题？

`downgrade` **不是**用来简化双重检查锁定的。它解决的是另一个问题：**修改数据后，需要长时间持有锁读取，同时希望其他线程也能并发读**。

看这个场景：

```rust
use std::sync::{RwLock, RwLockWriteGuard};

// 场景：更新配置后，需要基于新配置执行耗时的只读计算
fn update_and_compute(config: &RwLock<Config>) -> ComputeResult {
    // 1. 获取写锁，更新配置
    let mut write = config.write().unwrap();
    write.apply_updates();

    // 问题来了：接下来要基于新配置做耗时计算
    // 如果继续持有写锁，其他线程完全阻塞
    // 如果释放写锁再获取读锁，存在时间窗口

    // 2. 降级为读锁：原子操作，没有时间窗口
    let read = RwLockWriteGuard::downgrade(write);

    // 3. 耗时的只读计算（此时其他线程可以并发读！）
    expensive_read_only_computation(&read)
}
```

**关键点**：
- `downgrade` 是原子操作，锁从未被释放
- 降级后，其他线程可以立即获取读锁并发读取
- 如果不降级，其他线程必须等待整个函数完成

### 对比：不使用 downgrade 的两种选择

```rust
// 选项 A：继续持有写锁
fn update_and_compute_v1(config: &RwLock<Config>) -> ComputeResult {
    let mut write = config.write().unwrap();
    write.apply_updates();
    expensive_computation(&write)  // 其他线程完全阻塞 100ms
}

// 选项 B：释放写锁，重新获取读锁
fn update_and_compute_v2(config: &RwLock<Config>) -> ComputeResult {
    {
        let mut write = config.write().unwrap();
        write.apply_updates();
    }  // 写锁释放

    // 时间窗口：其他线程可能在这里修改配置！

    let read = config.read().unwrap();
    expensive_computation(&read)  // 读到的可能不是我们刚更新的配置
}

// 选项 C：使用 downgrade（最佳）
fn update_and_compute_v3(config: &RwLock<Config>) -> ComputeResult {
    let mut write = config.write().unwrap();
    write.apply_updates();
    let read = RwLockWriteGuard::downgrade(write);  // 原子降级
    expensive_computation(&read)  // 其他线程可以并发读
}
```

### 生态影响

这个 API 对以下场景特别有价值：

1. **配置热更新**：更新配置后，基于新配置执行长时间的只读操作，同时让其他线程可以读取新配置
2. **COW（Copy-on-Write）数据结构**：修改后需要长时间读取，降级让其他读者不必等待
3. **缓存预热**：写入缓存后，立即基于缓存数据进行计算，同时允许其他线程读取

一些第三方库（如 `parking_lot`）早就提供了这个功能，现在标准库终于跟上了。

### 设计哲学

为什么这个 API 等了这么久？因为 Rust 标准库对 API 的稳定非常谨慎。`downgrade` 涉及：

- **语义清晰性**：降级后原来的写锁失效，这通过 `self` 所有权转移来保证
- **平台兼容性**：不同操作系统的 RwLock 实现不同，需要确保所有平台都能正确实现
- **与 `upgrade` 的对称性**：Rust 故意**没有**提供 `upgrade`（读锁升级为写锁），因为这会导致死锁

```rust
// 为什么没有 upgrade？假设两个线程都持有读锁并尝试升级：
// 线程 A: 持有读锁，等待所有读锁释放以升级为写锁
// 线程 B: 持有读锁，等待所有读锁释放以升级为写锁
// 结果：死锁！双方都在等对方释放读锁
```

`downgrade` 没有这个问题，因为写锁本身就是独占的。

## 二、零初始化智能指针：性能优化的新选择

### 新增的 API

```rust
// Box
impl<T> Box<T> {
    pub fn new_zeroed() -> Box<MaybeUninit<T>>;
}
impl<T> Box<[T]> {
    pub fn new_zeroed_slice(len: usize) -> Box<[MaybeUninit<T>]>;
}

// Rc 和 Arc 同理
impl<T> Rc<T> {
    pub fn new_zeroed() -> Rc<MaybeUninit<T>>;
}
impl<T> Arc<T> {
    pub fn new_zeroed() -> Arc<MaybeUninit<T>>;
}
```

### 这解决什么问题？

传统的 `Box::new(value)` 流程是：

1. 在栈上创建 `value`
2. 在堆上分配内存
3. 把 `value` 从栈复制到堆

对于大型数据结构，第 3 步的复制开销可能很大：

```rust
// 问题：这个数组先在栈上创建，再复制到堆上
let big_array = Box::new([0u8; 1024 * 1024]);  // 1MB 的复制！

// 更糟：如果栈空间不够，直接栈溢出
let huge_array = Box::new([0u8; 10 * 1024 * 1024]);  // 10MB，可能栈溢出
```

### 零初始化的优势

```rust
use std::mem::MaybeUninit;

// 直接在堆上分配并零初始化，没有栈上的中间副本
let big_array: Box<[u8; 1024 * 1024]> = unsafe {
    Box::new_zeroed().assume_init()
};

// 对于切片
let dynamic_array: Box<[u8]> = unsafe {
    Box::new_zeroed_slice(1024 * 1024)
        .assume_init()  // 注意：这个方法在切片上不直接可用
        // 实际需要其他方式，见下文
};
```

**为什么返回 `MaybeUninit`？** 因为"零初始化"对某些类型来说**不是有效的初始状态**：

```rust
// 零初始化对这些类型是无效的：
// - NonZero<T>：零不是有效值
// - 引用：null 不是有效引用
// - 某些枚举：零可能不是有效的判别值

// MaybeUninit 告诉编译器："我知道这块内存可能还没准备好"
```

### 实战用法

```rust
use std::sync::Arc;
use std::mem::MaybeUninit;

// 场景：创建一个大型共享缓冲区
fn create_shared_buffer(size: usize) -> Arc<[u8]> {
    // 1. 创建零初始化的 MaybeUninit 切片
    let uninit: Arc<[MaybeUninit<u8>]> = Arc::new_zeroed_slice(size);

    // 2. 对于 u8，零初始化是完全有效的，可以安全转换
    // 注意：这需要一些 unsafe 魔法
    unsafe {
        // 利用 MaybeUninit<u8> 和 u8 有相同的内存布局
        Arc::from_raw(Arc::into_raw(uninit) as *const [u8])
    }
}

// 使用
let buffer = create_shared_buffer(1024 * 1024);  // 1MB 共享缓冲区
```

### 性能对比

```text
场景：分配 10MB 缓冲区

Box::new([0u8; 10MB])
  1. 栈上分配 10MB（可能栈溢出）
  2. 堆上分配 10MB
  3. 复制 10MB
  总计：20MB 内存操作 + 复制开销

Box::new_zeroed() + assume_init()
  1. 堆上分配 10MB
  2. 操作系统零初始化（通常是 COW，接近零成本）
  总计：10MB 内存操作，无复制
```

### 设计哲学

这组 API 体现了 Rust 的一个核心理念：**零成本抽象**。

- `MaybeUninit` 在编译后**完全消失**，没有运行时开销
- 通过类型系统强制程序员处理"未初始化"状态
- 提供 `unsafe` 逃生舱口，但把责任明确交给程序员

这也是为什么 API 返回 `MaybeUninit` 而不是直接返回零初始化的值——Rust 宁可让你多写几行代码，也不愿意让你无意中创建无效状态。

## 三、NonZero::div_ceil：小而美的补全

### 新增的 API

```rust
impl NonZero<u8> {  // 以及 u16, u32, u64, u128, usize
    pub const fn div_ceil(self, rhs: Self) -> Self;
}
```

**除法向上取整**，且保证结果也是 `NonZero`。

### 为什么需要这个？

`div_ceil`（向上取整除法）是一个常见操作：

```rust
// 场景：计算需要多少个页面来存储 n 字节
fn pages_needed(bytes: usize, page_size: usize) -> usize {
    (bytes + page_size - 1) / page_size  // 传统写法，容易溢出
    // 或
    bytes.div_ceil(page_size)  // 标准库方法，1.73.0 稳定
}
```

但对于 `NonZero` 类型，之前没有这个方法：

```rust
use std::num::NonZero;

let items = NonZero::new(100).unwrap();
let batch_size = NonZero::new(30).unwrap();

// 之前：需要转换
let batches = NonZero::new(items.get().div_ceil(batch_size.get())).unwrap();

// 现在：直接调用
let batches = items.div_ceil(batch_size);  // 返回 NonZero<usize>
```

### 为什么返回值也是 NonZero？

数学保证：如果被除数和除数都是正整数，向上取整的结果**至少是 1**。

```
1 / 1 = 1  (向上取整)
1 / 100 = 1  (向上取整)
99 / 100 = 1  (向上取整)
100 / 100 = 1
```

所以返回 `NonZero` 是类型安全的——**不可能返回零**。

### 生态影响

`NonZero` 类型在以下场景广泛使用：

1. **内存布局优化**：`Option<NonZero<u32>>` 和 `u32` 大小相同
2. **类型安全**：避免除零错误
3. **FFI**：很多 C API 用非零值表示有效句柄

这个 API 让 `NonZero` 在数值计算场景更加实用。

## 四、BTreeMap::Entry::insert_entry：补齐最后一块拼图

### 新增的 API

```rust
impl<'a, K: Ord, V> Entry<'a, K, V> {
    pub fn insert_entry(self, value: V) -> OccupiedEntry<'a, K, V>;
}

impl<'a, K: Ord, V> VacantEntry<'a, K, V> {
    pub fn insert_entry(self, value: V) -> OccupiedEntry<'a, K, V>;
}
```

### 与现有 API 的区别

```rust
use std::collections::BTreeMap;

let mut map = BTreeMap::new();

// 现有的 insert 方法：返回 &mut V
let value_ref: &mut i32 = map.entry("key").or_insert(42);

// 新的 insert_entry 方法：返回 OccupiedEntry
let entry = map.entry("key2").insert_entry(100);
// 现在可以：
let key: &str = entry.key();      // 获取键的引用
let value: &mut i32 = entry.get_mut();  // 获取值的可变引用
entry.remove();                    // 移除整个条目
```

### 使用场景

```rust
// 场景：插入后需要同时操作键和值
fn insert_and_log<K: Ord + std::fmt::Debug, V: std::fmt::Debug>(
    map: &mut BTreeMap<K, V>,
    key: K,
    value: V,
) {
    let entry = map.entry(key).insert_entry(value);
    println!("Inserted: {:?} => {:?}", entry.key(), entry.get());
}

// 场景：条件性移除刚插入的值
fn insert_if_valid<K: Ord, V>(
    map: &mut BTreeMap<K, V>,
    key: K,
    value: V,
    is_valid: impl FnOnce(&K, &V) -> bool,
) -> Option<V> {
    let entry = map.entry(key).insert_entry(value);
    if !is_valid(entry.key(), entry.get()) {
        Some(entry.remove())  // 返回被移除的值
    } else {
        None
    }
}
```

### 设计哲学

这个 API 体现了 Rust 集合 API 的一个原则：**Entry API 应该是完备的**。

`Entry` 模式是 Rust 集合的标志性设计，它解决了"查找后插入"的效率问题：

```rust
// 低效：两次查找
if !map.contains_key(&key) {
    map.insert(key, compute_value());
}

// 高效：一次查找
map.entry(key).or_insert_with(|| compute_value());
```

`insert_entry` 补齐了 Entry API 的最后一块：**插入后继续操作条目**。

## 五、Location::file_as_c_str：FFI 友好的调试信息

### 新增的 API

```rust
impl<'a> Location<'a> {
    pub fn file_as_c_str(&self) -> &'a CStr;
}
```

### 这是什么？

`std::panic::Location` 用于获取代码位置信息（文件名、行号、列号），常用于 `#[track_caller]`：

```rust
#[track_caller]
fn my_unwrap<T>(opt: Option<T>) -> T {
    match opt {
        Some(v) => v,
        None => {
            let loc = std::panic::Location::caller();
            panic!("Called unwrap on None at {}:{}:{}",
                   loc.file(), loc.line(), loc.column());
        }
    }
}
```

### 为什么需要 C 字符串版本？

当你需要把位置信息传给 C 代码时：

```rust
// FFI 场景：调用 C 的日志库
extern "C" {
    fn c_log(file: *const c_char, line: u32, message: *const c_char);
}

#[track_caller]
fn log_from_rust(message: &str) {
    let loc = std::panic::Location::caller();
    let message_cstr = CString::new(message).unwrap();

    // 之前：需要额外分配
    let file_cstr = CString::new(loc.file()).unwrap();

    // 现在：零分配
    let file_cstr = loc.file_as_c_str();

    unsafe {
        c_log(file_cstr.as_ptr(), loc.line(), message_cstr.as_ptr());
    }
}
```

### 设计哲学

这个 API 很小，但体现了 Rust 的**零成本互操作**理念：

- 文件名字符串本来就以 null 结尾存储在二进制中
- `file_as_c_str` 只是暴露了这个事实，没有任何运行时开销
- 对于 FFI 密集型代码，这些小优化累积起来很可观

## 六、TokenStream 扩展：过程宏的人体工程学改进

### 新增的 API

```rust
impl Extend<Group> for TokenStream { ... }
impl Extend<Ident> for TokenStream { ... }
impl Extend<Literal> for TokenStream { ... }
impl Extend<Punct> for TokenStream { ... }
```

### 之前怎么做？

```rust
use proc_macro::{TokenStream, TokenTree, Ident, Literal, Punct, Spacing};

fn build_token_stream() -> TokenStream {
    let mut tokens = TokenStream::new();

    // 之前：必须先转换为 TokenTree
    tokens.extend(std::iter::once(TokenTree::Ident(
        Ident::new("hello", Span::call_site())
    )));
    tokens.extend(std::iter::once(TokenTree::Punct(
        Punct::new(',', Spacing::Alone)
    )));
    tokens.extend(std::iter::once(TokenTree::Literal(
        Literal::string("world")
    )));

    tokens
}
```

### 现在：更直接

```rust
fn build_token_stream() -> TokenStream {
    let mut tokens = TokenStream::new();

    // 现在：直接 extend
    tokens.extend([Ident::new("hello", Span::call_site())]);
    tokens.extend([Punct::new(',', Spacing::Alone)]);
    tokens.extend([Literal::string("world")]);

    tokens
}
```

### 生态影响

过程宏是 Rust 生态的重要组成部分（`serde`、`tokio`、`diesel` 等都依赖它）。这个改进让手写过程宏更加愉快。

当然，大多数人会使用 `quote!` 宏，但对于需要动态构建 token 的场景，这个改进很有价值。

## 七、切片旋转的常量化

### 新增的能力

```rust
// 现在可以在 const 上下文中使用
const ROTATED: [i32; 5] = {
    let mut arr = [1, 2, 3, 4, 5];
    arr.rotate_left(2);
    arr
};
// ROTATED = [3, 4, 5, 1, 2]
```

### 使用场景

```rust
// 编译期生成查找表
const LOOKUP_TABLE: [u8; 256] = {
    let mut table = [0u8; 256];
    // ... 初始化 ...
    table.rotate_left(128);  // 调整布局
    table
};

// 编译期字符串处理
const fn rotate_string(s: &[u8; 5], n: usize) -> [u8; 5] {
    let mut result = *s;
    result.rotate_left(n);
    result
}

const HELLO: [u8; 5] = *b"hello";
const LLOHE: [u8; 5] = rotate_string(&HELLO, 2);
```

### 设计哲学

Rust 正在逐步扩展 `const fn` 的能力边界。每个版本都会有一些函数被"常量化"，这是一个渐进的过程：

1. 编译期计算减少运行时开销
2. 更多的编译期验证
3. 支持更复杂的 const 泛型

## 总结：这个版本值得升级吗？

**值得。** 虽然没有革命性的新特性，但这些 API 都是**实打实解决痛点**的：

| API | 解决的痛点 | 受益场景 |
|-----|-----------|---------|
| `RwLockWriteGuard::downgrade` | 锁降级的时间窗口 | 缓存、配置热更新 |
| `Box/Rc/Arc::new_zeroed` | 大数据结构的栈溢出和复制开销 | 内存密集型应用 |
| `NonZero::div_ceil` | NonZero 数值计算不便 | 分页、批处理计算 |
| `Entry::insert_entry` | 插入后无法操作条目 | 复杂的 Map 操作 |
| `Location::file_as_c_str` | FFI 场景的额外分配 | C 互操作 |
| `TokenStream::extend` | 过程宏编写繁琐 | 宏开发 |
| `rotate_*` const 化 | 编译期数组操作受限 | 查找表、编译期计算 |

升级命令：

```bash
rustup update stable
```

---

**Rust 的演进哲学**：不追求大版本的惊艳，而是每六周稳定地补齐一些 API、修复一些痛点。这种"无聊"的演进方式，恰恰是生产级语言应该有的样子。

每一个新稳定的 API 背后，都是社区多年的讨论、RFC、实现、测试。`RwLockWriteGuard::downgrade` 从提出到稳定用了九年——这就是 Rust 对 API 稳定性的承诺。
