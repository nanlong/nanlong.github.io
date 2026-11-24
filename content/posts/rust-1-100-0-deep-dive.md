---
title: "Rust 1.100 深度解析：never 类型全面稳定，Vec 和 Box 支持自定义分配器"
date: 2026-11-18T00:00:00+08:00
draft: false
tags: ["Rust", "类型系统", "分配器", "标准库", "Cargo", "版本更新"]
categories: ["技术"]
description: "Rust 1.100 稳定了可在任意类型位置书写的 never 类型 `!` 与核心 `Allocator` API，将 `Infallible` 统一为 `!` 的别名，并新增 `DropGuard`、`conjure_zst`、`extern \"custom\"` 以及 Cargo 原生 lint、依赖冷却期和新构建目录布局。本文结合官方发布说明与 1.99.0 对照验证，拆解这些变化的语义、工程取舍和升级注意点。"
---

Rust 1.100.0 于 2026 年 11 月 12 日发布。这个整数版本落地了两项推进多年的底层能力：代表“永不产生值”的 never 类型 `!` 可以写在任意类型位置，`std::convert::Infallible` 正式成为 `!` 的别名；标准库同步稳定了 `Allocator` trait 的核心部分，现在可以为单个 `Vec` 或 `Box` 指定自定义内存分配器，不必再局限于全局分配器。

此外，这一版还补齐了一批高频工程能力：标准库新增作用域清理守卫、零大小类型构造、智能指针与切片便捷接口；编译器稳定了自定义调用约定和采样 PGO；Cargo 引入了原生 lint 系统、新中间构建目录布局与依赖冷却期，并调整了 `cargo install` 的默认锁文件策略。完整更新见 [Rust 1.100.0 发布说明](https://doc.rust-lang.org/1.100.0/releases.html#version-11000-2026-11-12)。

文中可运行示例均使用以下正式版工具链编译运行，旧版对照使用 `rustc 1.99.0 (b940084d7 2026-09-28)` 与 `cargo 1.99.0 (5f94df478 2026-08-27)`。除专门对比 edition 的示例外，均按 Rust 2021（`edition = "2021"`）编译；涉及其他平台或底层内存模型的保证，以官方文档为准。

```text
rustc 1.100.0 (0ef1bc386 2026-11-09)
cargo 1.100.0 (3acc79c0c 2026-11-05)
host: aarch64-apple-darwin
LLVM version: 23.1.5
```

<!--more-->

## 一张表看懂 Rust 1.100

| 模块 | 核心变化 | 直接影响 |
|------|----------|----------|
| 类型系统 | never 类型 `!` 全面稳定 | 可写在任意类型位置，`Infallible` 变为 `!` 的别名，旧 edition 回退规则统一切到 `!` |
| 内存分配 | 核心 `Allocator` API 稳定 | `Vec` 和 `Box` 支持自定义分配器；`Arc::new_in`、`String` 等尚未稳定 |
| `core::mem` | `DropGuard` 与 `conjure_zst` | 闭包式作用域回滚免写模板代码；有值 ZST 获得专用 `unsafe` 构造入口 |
| 标准库 | 一组高频小接口补齐 | `Box::take`/`map`、`Rc`/`Arc::map`、`UnsafeCell` 三方法、`trim_prefix`/`suffix`、`Vec::from_fn` |
| 语言与编译器 | `extern "custom"`、SPGO 与新 lint | 稳定自定义 ABI 和 `-Cprofile-sample-use`；orphanck 启用新求解器；`drop_in_place` 接入无效果检查 |
| Cargo | 原生 lint、新布局与冷却期 | 默认检查 `unused_dependencies`；中间产物改按构建步骤分目录；支持 `global-min-publish-age`；`cargo install` 默认用包内锁文件 |
| 兼容性 | 回退、重载与转义调整 | 未约束泛型 `?` 回退变 `!`；`fn() -> !` 与 `fn() -> Infallible` 冲突；`escape_debug` 转义组合字符；移除 `box` 模式 |

下面从历时十年的 never 类型讲起。

## 一、never 类型 `!`：十年后终于能写在任意类型位置

### 1. `!` 与 `Infallible` 合二为一

在 Rust 里，`!`（never type）代表“不可能产生任何值”的类型。`return`、`break`、`continue` 以及 `std::process::exit` 这类打断控制流的表达式，类型都是 `!`。因为不可能真的拿到一个 `!` 类型的值，编译器允许把它隐式转换为任意目标类型（never-to-any coercion），这也是 `match` 一条分支写 `None => return`、另一条分支返回 `u32` 时能通过类型检查的原因。

过去稳定版只允许把 `!` 写在函数返回类型（`fn stop() -> !`）等少数位置。要在泛型参数里表达“错误分支绝不会发生”，只能用标准库提供的空枚举 `std::convert::Infallible`。1.100 放开了位置限制，`!` 可以出现在任意类型实参中，`Infallible` 的定义也改为 `pub type Infallible = !`。标准库同时稳定了 `impl AsRef<T> for !` 和 `impl AsMut<T> for !`，方便泛型代码统一约束错误类型。

重点看 `match result` 中的 `Err(never) => never` 分支：

```rust
use std::convert::Infallible;

fn convert<T, U>(value: T) -> U
where
    U: TryFrom<T, Error = Infallible>,
{
    let result: Result<U, !> = U::try_from(value);
    match result {
        Ok(output) => output,
        Err(never) => never,
    }
}

fn require_str_ref(err: &!) -> &str {
    err.as_ref()
}

fn main() {
    let value: u16 = convert(42_u8);
    println!("converted = {value}");

    let ok: Result<&str, !> = Ok("ready");
    let text = match ok {
        Ok(s) => s,
        Err(ref never) => require_str_ref(never),
    };
    println!("text = {text}");
}
```

运行输出：

```text
converted = 42
text = ready
```

这里能看清 `!` 相对旧版 `Infallible` 真正多出来的能力：

- 在 Rust 1.99.0 中，虽然不可反驳模式 `let Ok(output) = result;` 配合 `Result<U, Infallible>` 本来就能写，但如果写成 `Err(never) => never`，编译器会报 `E0308: expected type parameter U, found enum Infallible`，必须补一层空匹配 `match never {}`。
- 在 Rust 1.100.0 中，哪怕左侧类型标注仍写 `Result<U, Infallible>`，`Err(never) => never` 也能直接编译通过——因为 `Infallible` 已是 `!` 的别名，自动具备 never-to-any 隐式转换能力。
- 如果把上例原样交给 Rust 1.99.0，在 `Result<U, !>` 和 `err: &!` 处会直接报 `E0658: the ! type is experimental`。

### 2. 为什么这扇门卡了十年？

[Rust 1.100.0 官方公告](https://blog.rust-lang.org/2026/11/12/Rust-1.100.0/)提到，never 类型从提案到稳定历时十年，此前五次尝试均未完成。核心难点不在语法上放行 `!`，而在于必须把生态中早已广泛使用的 `Infallible` 统一到 `!` 上，避免标准库和第三方 crate 长期分裂成两套“不可失败”的错误类型。

完成这一步有个绕不开的前提：统一旧 edition 的 never 类型回退规则。当一段涉及 `!` 隐式转换的泛型表达式始终没有被上下文约束类型时，早期 edition 会把它回退成 `()`，而 Rust 2024 已改为回退成 `!`。1.100 将 Rust 2015、2018、2021 的回退规则全部切到 `!`。即便源码里从未写过 `!`，只要存在被丢弃返回值的泛型 `?` 调用，升级后类型推断结果也可能变化（详见第七节实测）。

### 3. 库作者的 MSRV 边界

> **兼容提示**：需要继续支持 Rust 1.99 或更低 MSRV 的库，公开接口可以保持写 `Result<T, Infallible>`。下游升级到 1.100 后，它与 `Result<T, !>` 自动互通；反过来，一旦在源码的类型参数位置直接写 `!`，最低支持版本就必须提高到 1.100。

## 二、`Allocator` API：`Vec` 和 `Box` 支持自定义分配器

### 1. 为单个容器指定分配策略

此前在稳定版 Rust 中，只能通过 `#[global_allocator]` 在进程级别替换全局分配器；如果想让某个缓冲区使用系统分配器、内存池或带上限的分配策略，要么启用 nightly 的 `allocator_api`，要么换用第三方容器。

Rust 1.100 稳定了 `Allocator` 体系的核心：

- 核心类型：`std::alloc::Allocator`、`Global`、`System`；
- 转发实现：为 `Global`、`System`、`&A`、`&mut A`、`Box<T, A>`、`Rc<T, A>`、`Arc<T, A>` 实现 `Allocator`；
- 容器入口：`Vec<T, A>` 与 `Box<T, A>` 的 `new_in`、`with_capacity_in`、`allocator` 等方法。

下面的 `MaxBlock` 分配器拒绝单次超过指定字节数的内存请求，合法请求则委托给 `System`，再把它同时借给一个 `Vec` 和一个 `Box`：

```rust
use std::alloc::{AllocError, Allocator, Layout, System};
use std::ptr::NonNull;

struct MaxBlock(usize);

// SAFETY: 成功的分配和对应的释放都委托给 System；
// 移动 MaxBlock 不影响已分配内存块的有效性。
unsafe impl Allocator for MaxBlock {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        if layout.size() > self.0 {
            return Err(AllocError);
        }
        System.allocate(layout)
    }

    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout) {
        // SAFETY: 调用方保证 ptr 和 layout 匹配原分配，且内存来自 System。
        unsafe { System.deallocate(ptr, layout) }
    }
}

fn main() {
    let limit = MaxBlock(16);
    let mut bytes = Vec::<u8, _>::new_in(&limit);

    let too_large = bytes.try_reserve_exact(32);
    assert!(too_large.is_err());
    println!("reserve 32 bytes err = {}", too_large.is_err());

    bytes.try_reserve_exact(8).expect("8 bytes within limit");
    bytes.extend_from_slice(&[1, 2, 3]);
    let block = Box::new_in([7_u8; 16], &limit);
    println!("bytes = {bytes:?}, boxed len = {}", block.len());
}
```

运行输出：

```text
reserve 32 bytes err = true
bytes = [1, 2, 3], boxed len = 16
```

同一份代码用 Rust 1.99.0 编译，会报 `E0658: use of unstable library feature allocator_api`。

使用自定义分配器时有两点核心机制值得留意：

- **共享分配器句柄**：`new_in(&limit)` 成立，是因为标准库为 `&A` 实现了 `Allocator`，允许多个局部容器共享同一个分配器。需要跨线程或长期共享有状态的分配器时，可以用 `Arc<MyAlloc>`（标准库同样为 `Arc<T, A>` 实现了 `Allocator`）。
- **`unsafe trait` 安全约定**：按 [`Allocator` 文档](https://doc.rust-lang.org/1.100.0/std/alloc/trait.Allocator.html)，返回的内存块必须满足请求的 `Layout` 大小和对齐要求（允许大于请求值），且移动或克隆分配器句柄不得使已分配内存失效。上例检查的是单次分配的 `layout.size()`，所以 `bytes` 申请 8 字节后再由 `Box` 申请 16 字节数组可以同时成功。

### 2. 拆解为裸指针时，分配器必须成对流转

[上一篇（Rust 1.99）](https://zhuanlan.zhihu.com/p/2090105221263390225)介绍过 `Box::into_non_null` 和 `Vec::into_parts`。带自定义分配器的容器拆解后，指针必须与对应分配器成对流转，不能丢掉分配器信息。

1.100 稳定了全套带分配器的拆解与重建接口：`Box::{into_raw_with_allocator, from_raw_in, into_non_null_with_allocator, from_non_null_in}`，以及 `Vec::{into_raw_parts_with_allocator, from_raw_parts_in, into_parts_with_allocator, from_parts_in}`。

```rust
use std::alloc::System;

fn main() {
    // Box：拆出 NonNull 与分配器，修改后再配对重建
    let boxed = Box::new_in(String::from("cfg"), System);
    let (ptr, alloc) = Box::into_non_null_with_allocator(boxed);
    unsafe { (*ptr.as_ptr()).push_str(".toml") };
    let rebuilt_box = unsafe { Box::from_non_null_in(ptr, alloc) };
    println!("box = {rebuilt_box}");

    // Vec：拆出 (ptr, len, cap, alloc) 四元组再重建
    let mut vec = Vec::with_capacity_in(8, System);
    vec.extend([10_u32, 20, 30]);
    let (ptr, len, cap, alloc) = vec.into_parts_with_allocator();
    let rebuilt_vec = unsafe { Vec::from_parts_in(ptr, len, cap, alloc) };
    println!("vec = {rebuilt_vec:?}, cap = {}", rebuilt_vec.capacity());
}
```

运行输出：

```text
box = cfg.toml
vec = [10, 20, 30], cap = 8
```

> **安全边界**：[`Global` 文档](https://doc.rust-lang.org/1.100.0/std/alloc/struct.Global.html) 明确指出，即便程序通过 `#[global_allocator]` 把 `System` 注册为全局分配器，也不能把通过 `Global` 分配的内存交给 `System` 释放，两者在契约上并不等价。

### 3. 接口设计取舍与当前稳定范围

引入自定义分配器后，最先遇到的工程问题是**类型签名变化**：`Vec<u8, MaxBlock>` 和普通 `Vec<u8>`（即 `Vec<u8, Global>`）是两个不同的类型，不能直接互传。实际写接口时可以按三层处理：

1. **只读或就地读写元素**：继续接收 `&[u8]` 或 `&mut [u8]`，完全不感知分配器。
2. **需要接管或扩容容器**：将函数或结构体泛型化为 `Vec<T, A>`（`A: Allocator`），保持零运行时开销。
3. **不想让分配器类型传染到外层签名**：`Allocator` 支持动态分发（dyn-compatible），可以用 `Vec<u8, &dyn Allocator>` 统一抹平具体分配器类型，代价是一层虚表分发开销。验证如下：

```rust
use std::alloc::{Allocator, System};

fn main() {
    let alloc: &dyn Allocator = &System;
    let mut bytes: Vec<u8, &dyn Allocator> = Vec::new_in(alloc);
    bytes.push(1);
    println!("bytes = {bytes:?}");
}
```

运行输出：

```text
bytes = [1]
```

另外需要注意，1.100 只稳定了 `Vec` 和 `Box` 两个容器的分配器入口：

- `Arc::new_in(42, System)` 在 1.100.0 上仍报 `E0658: use of unstable library feature allocator_ext`（tracking issue `#163177`）。也就是说，`Arc<MyAlloc>` 可以**充当分配器**传给 `Vec` 或 `Box`，但 `Arc` 自身节点的内存来源还不能用稳定接口定制。
- `String::new_in(System)` 会报 `E0599`，标准库目前尚未提供该方法；`VecDeque`、`HashMap`、`BTreeMap` 等集合的支持仍待后续版本推进。

## 三、`DropGuard` 与 `conjure_zst`：两个填补空白的 `core::mem` 工具

### 1. `DropGuard`：用闭包实现事务式回滚

Rust 的 `Drop` 保证变量离开作用域时必定执行清理，无论退出路径是正常返回、`?` 提前返回还是 panic 栈展开。但如果在函数内部只是想临时写一段“失败时回滚”的清理逻辑，专门定义一个结构体再去实现 `Drop`、`Deref`、`DerefMut` 过于繁琐，很多项目为此引入了 `scopeguard`。

1.100 在标准库加入了 `core::mem::DropGuard`（同步重导出到 `std::mem::DropGuard`）。它包裹一个值和清理闭包，自身实现了 `Deref` 与 `DerefMut`：操作过程中像平时一样读写内部值；中途失败离开作用域时自动触发闭包；全部成功后调用 `DropGuard::dismiss(guard)` 取消清理并取回原值。

下面模拟一种单字节长度前缀的二进制帧编码，要求一批帧原子地追加到现有缓冲区，只要任意一帧超过 255 字节就撤销整批写入：

```rust
use std::mem::DropGuard;
use std::num::TryFromIntError;

fn append_frames(output: &mut Vec<u8>, frames: &[&[u8]]) -> Result<(), TryFromIntError> {
    // 后续只向末尾追加，失败时截回原长度即可撤销本批写入。
    let original_len = output.len();
    let mut guarded = DropGuard::new(output, |buf| buf.truncate(original_len));

    for frame in frames {
        let len = u8::try_from(frame.len())?;
        guarded.push(len);
        guarded.extend_from_slice(frame);
    }

    DropGuard::dismiss(guarded);
    Ok(())
}

fn main() {
    let mut output = vec![0xaa];
    append_frames(&mut output, &[b"hi", b"!"]).unwrap();
    println!("committed = {output:?}");

    let before = output.clone();
    let oversized = [0_u8; 256];
    let err = append_frames(&mut output, &[b"ok", &oversized]);
    println!("oversized err = {}, rolled back = {}", err.is_err(), output == before);
}
```

运行输出：

```text
committed = [170, 2, 104, 105, 1, 33]
oversized err = true, rolled back = true
```

用 Rust 1.99.0 编译同一份代码，会报 `E0658: use of unstable library feature drop_guard`。

使用 `DropGuard` 时需留意三点：

1. **必须用关联函数调用 `dismiss`**：写法是 `DropGuard::dismiss(guarded)`，不是 `guarded.dismiss()`。官方公告说明，这是为了避免 `DerefMut` 透传时与被包裹类型自身的同名方法冲突。
2. **回滚范围取决于清理闭包**：上例只向末尾追加，所以 `truncate` 能恢复原有内容，但中途扩容增加的容量会保留。
3. **遵循普通析构边界**：在 `panic = "abort"` 下进程直接终止，或者守卫被 `mem::forget` 泄漏时，清理闭包不会运行。

### 2. `conjure_zst`：有值零大小类型的专用构造方法

编写底层数据结构或泛型抽象时，有时确定泛型参数 `T` 是零大小类型（ZST），却没有 `T: Default` 约束。过去常见做法包括 `mem::transmute(())`、`MaybeUninit::uninit().assume_init()` 或 `mem::zeroed()`，语义不够直接。1.100 提供了专用的 `unsafe` 入口 `core::mem::conjure_zst::<T>()`：

```rust
use std::mem::conjure_zst;

#[derive(Debug, PartialEq, Eq)]
struct Marker;

fn main() {
    // SAFETY: ()、Marker 和 [u32; 0] 都是有值且无额外安全不变量的 ZST。
    let unit: () = unsafe { conjure_zst() };
    let marker: Marker = unsafe { conjure_zst() };
    let empty: [u32; 0] = unsafe { conjure_zst() };
    println!("unit={unit:?}, marker={marker:?}, empty={empty:?}");
}
```

运行输出：

```text
unit=(), marker=Marker, empty=[]
```

Rust 1.99.0 编译会报 `E0658: use of unstable library feature mem_conjure_zst`。

调用 `conjure_zst` 要把**函数自带的尺寸检查**和**调用者负责的安全前提**区分清楚：

- **非零大小类型会直接 panic**：在 1.100.0 中调用 `conjure_zst::<u8>()`，运行时 panic 并输出 `mem::conjure_zst invoked on type u8, which is not zero-sized`；若写在 `const` 常量求值里，编译器会报 `E0080: evaluation panicked: mem::conjure_zst invoked on a non-zero-sized type`。
- **有值性与库不变量由调用者保证**：[`conjure_zst` 文档](https://doc.rust-lang.org/1.100.0/core/mem/fn.conjure_zst.html) 规定两条安全前提：第一，`T` 必须是有值的（inhabited），构造 `!` 或无变体枚举属于未定义行为；第二，不能破坏类型的安全不变量。例如 PyO3 的 `Python<'py>` 是零大小标记令牌，代表当前线程持有 Python GIL，凭空造出令牌传给安全接口就会绕过持锁前提。

## 四、其他实用标准库新 API

### 1. `Box::take` 与 `Box` / `Rc` / `Arc` 的 `map`

1.100 为智能指针新增了一组堆内存复用和值转换的关联函数：

- `Box::take(boxed)` 取出堆上的值 `T`，同时返回指向原未初始化槽位的 `Box<MaybeUninit<T>, A>`，配合 `Box::write` 可复用同一块堆内存。
- `Box::map`、`Rc::map`、`Arc::map` 将内部值转换为新类型并装回同类指针，条件允许时复用原分配（文档不保证地址不变，代码不应依赖指针地址相等）。

```rust
use std::rc::Rc;
use std::sync::Arc;

fn main() {
    // Box::take 取出值并保留未初始化槽位，随后用 Box::write 复用
    let boxed = Box::new(10_u32);
    let (val, uninit) = Box::take(boxed);
    let reused = Box::write(uninit, val + 5);
    println!("reused = {reused}");

    // Box::map 按值接收 T；Rc::map 和 Arc::map 按共享引用接收 &T
    let mapped_box = Box::map(Box::new(3_u16), u32::from);
    let mapped_rc = Rc::map(Rc::new(String::from("rust")), |s| s.len());
    let mapped_arc = Arc::map(Arc::new([1_i32, 2, 3]), |arr| arr.iter().sum::<i32>());
    println!("box={mapped_box}, rc={mapped_rc}, arc={mapped_arc}");
}
```

运行输出：

```text
reused = 15
box=3, rc=4, arc=6
```

留意闭包参数的区别：`Box` 独占内部值，因此 `Box::map` 按值传入 `T`；`Rc` 与 `Arc` 可能存在其他强引用，因此 `Rc::map` 和 `Arc::map` 传入的是 `&T`。与 `DropGuard::dismiss` 同理，这四个函数都设计为关联函数以避免方法名冲突。用 Rust 1.99.0 编译上例时，`Box::take(boxed)` 会先触发 `unstable_name_collisions` 警告并试图解析到 `Iterator::take`，随后因 `u32` 不是迭代器而失败。

### 2. `UnsafeCell` 的三个 `const unsafe` 访问方法

[Rust 1.99 篇](https://zhuanlan.zhihu.com/p/2090105221263390225)提到，1.99 在规则层面明确了访问 `UnsafeCell` 内部数据的指针转换保证。1.100 接着稳定了 `UnsafeCell::replace`、`as_ref_unchecked` 和 `as_mut_unchecked`，且三者均为 `const unsafe fn`，可用于常量求值：

```rust
use std::cell::UnsafeCell;

const REPLACED: u32 = {
    let cell = UnsafeCell::new(10);
    // SAFETY: 局部独占访问，无其他引用存活。
    let old = unsafe { cell.replace(42) };
    let current = unsafe { *cell.as_ref_unchecked() };
    old + current
};

fn main() {
    let cell = UnsafeCell::new(String::from("hello"));
    // SAFETY: 此作用域内没有其他指向内部值的引用。
    unsafe {
        cell.as_mut_unchecked().push_str(" 1.100");
        println!("cell = {}", cell.as_ref_unchecked());
    }
    let prev = unsafe { cell.replace(String::from("reset")) };
    println!("prev = {prev}, now = {}, const = {REPLACED}", cell.into_inner());
}
```

运行输出：

```text
cell = hello 1.100
prev = hello 1.100, now = reset, const = 52
```

Rust 1.99.0 会报 `E0658: use of unstable library feature unsafe_cell_access`。这几个方法省去了手写 `&*cell.get()` 与 `ptr::replace(cell.get(), val)` 的样板代码，但别名规则不变：调用 `as_mut_unchecked` 或 `replace` 时不得有其他指向内部值的引用存活，调用 `as_ref_unchecked` 时不得存在活跃可变引用或并发写入。

### 3. `trim_prefix`、`Vec::from_fn` 与日常接口补齐

其余稳定 API 覆盖了一批日常高频操作：

```rust
use std::borrow::Cow;
use std::path::Path;
use std::sync::Once;

fn main() {
    // 1. 字符串、切片与路径去可选前后缀
    println!("str prefix    = {:?}", "refs/heads/main".trim_prefix("refs/heads/"));
    println!("str keep      = {:?}", "main".trim_prefix("refs/heads/"));
    println!("str suffix    = {:?}", "archive.tar.gz".trim_suffix(".gz"));
    println!("slice trim    = {:?}", [0, 0, 1, 2].trim_prefix(&[0][..]));
    println!("path trim     = {:?}", Path::new("/etc/nginx/nginx.conf").trim_prefix("/etc"));
    println!("path boundary = {:?}", Path::new("/etc_backup/a").trim_prefix("/etc"));

    // 2. 按下标生成向量与布尔翻转
    let squares = Vec::from_fn(5, |i| i * i);
    let mut enabled = false;
    enabled.toggle();
    println!("squares = {squares:?}, enabled = {enabled}");

    // 3. Vec / 切片直接与 Cow<[T]> 比较，以及 Once::default()
    let cow: Cow<'_, [i32]> = Cow::Borrowed(&[1, 2, 3]);
    let vec = vec![1, 2, 3];
    let slice: &[i32] = &[1, 2, 3];
    println!("vec==cow: {}, slice==cow: {}", vec == cow, slice == cow);

    let once = Once::default();
    let mut ran = false;
    once.call_once(|| ran = true);
    println!("once ran = {ran}");
}
```

运行输出：

```text
str prefix    = "main"
str keep      = "main"
str suffix    = "archive.tar"
slice trim    = [0, 1, 2]
path trim     = "nginx/nginx.conf"
path boundary = "/etc_backup/a"
squares = [0, 1, 4, 9, 16], enabled = true
vec==cow: true, slice==cow: true
once ran = true
```

几处使用细节：

- **`trim_prefix` vs `strip_prefix`**：前者最多剥离一次，不匹配时原样返回，省去 `.unwrap_or(s)`；需要区分“是否命中”时仍用返回 `Option` 的 `strip_prefix`。
- **`Path::trim_prefix` 按路径组件匹配**：面对 `"/etc_backup/a"` 时不会切出 `"_backup/a"`，而是按组件边界原样返回。注意标准库这次未提供 `Path::trim_suffix`。
- **其他平台与错误映射**：Unix 的 `ENOTSUP` 现在映射到 `std::io::ErrorKind::Unsupported`；Windows 新增 `ChildExt::main_thread_handle` 与 `CommandExt::show_window`。

## 五、语言、编译器与 Rustdoc 变化

### 1. `extern "custom"`：自定义调用约定进入稳定版

系统编程、内核中断入口、协程上下文切换或 JIT 跳板函数，常需要自定义寄存器传参规则。此前在稳定版上写这类 `#[unsafe(naked)]` 函数，只能借用 `extern "C"` 占位，容易误导阅读者以为它能按 C ABI 直接调用。

Rust 1.100 稳定了 `extern "custom"`（[PR #158504](https://github.com/rust-lang/rust/pull/158504)）。按 [Rust Reference 规范](https://doc.rust-lang.org/1.100.0/reference/items/functions.html#extern-custom)，它表示未知的自定义 ABI，**只能通过内联汇编调用**，并强制满足四条签名约束：

1. 必须标记 `unsafe`；
2. 不能声明任何参数；
3. 返回类型必须省略或直接写作 `()`（语法级检查，不能用 `type Unit = ();` 别名替代）；
4. 函数定义必须标注 `#[unsafe(naked)]`。

下面是一个 `aarch64` 架构的示例，约定通过 `x0` 寄存器传入参数并返回加 1 后的结果：

```rust
use core::arch::{asm, naked_asm};

/// 自定义调用约定：输入和返回值都在 x0 寄存器中，不修改其他寄存器。
#[unsafe(naked)]
unsafe extern "custom" fn add_one_x0() {
    naked_asm!(
        "add x0, x0, #1",
        "ret",
    )
}

fn main() {
    let mut x: u64 = 41;
    // SAFETY: 严格按 add_one_x0 的自定义约定通过 x0 传参并取回结果。
    unsafe {
        asm!(
            "bl {sym}",
            sym = sym add_one_x0,
            inout("x0") x,
            out("lr") _,
        );
    }
    println!("result = {x}");
}
```

运行输出：

```text
result = 42
```

Rust 1.99.0 编译会报 `E0658: the extern "custom" ABI is experimental and subject to change`。在 1.100.0 中，如果尝试用普通 Rust 语法直接调用 `unsafe { add_one_x0(); }`，编译器也会立刻拒绝并提示 `functions with the "custom" ABI cannot be called`，从语言层堵住误用风险。

### 2. 四组新 lint 与诊断改进

1.100 在 `rustc` 和 `rustdoc` 中补充了多条贴近排错的检查。

**`drop_in_place` 接入三项无效果 drop 检查。** `dropping_references`、`dropping_copy_types`（默认 warn）和 `undropped_manually_drops`（**默认 deny**）现在都会检查 `std::ptr::drop_in_place`。把 `*mut ManuallyDrop<String>` 直接传给 `drop_in_place`，1.99.0 静默通过，1.100.0 直接拒绝编译：

```text
error: calls to `drop_in_place` with a pointer to a `std::mem::ManuallyDrop` instead of the inner value does nothing
   |
14 |         drop_in_place(p_md);
   |         ^^^^^^^^^^^^^^----^
   |                       |
   |                       argument has type `*mut ManuallyDrop<String>`
   |
   = note: `#[deny(undropped_manually_drops)]` on by default
help: use `std::mem::ManuallyDrop::drop` to drop the inner value
```

如果旧 unsafe 代码误把 `ManuallyDrop` 自身的指针传给了 `drop_in_place`，升级到 1.100 会立刻暴露问题。

其余几项 lint 调整同样值得关注：

- **重复 `#[repr(..)]` 警告**：同一结构体重复写 `#[repr(C)]`，1.100.0 触发默认开启的 `repeated_reprs` 警告（归属 `unused` 组），1.99.0 无提示。
- **`aligned_fields_in_packed` 改为默认 deny 的 lint**：在 `#[repr(packed)]` 中嵌套 `#[repr(align(N))]` 字段，1.99.0 是不可压制的硬错误 `E0588`；1.100.0 将其转为可配置的默认 deny lint。显式加 `#[allow(aligned_fields_in_packed)]` 后可编译通过（含 1 字节 `u8` 和 `#[repr(align(8))]` 的 `u64` 封装时，实测尺寸为 9 字节），但会破坏内部对齐假设，除极特殊底层兼容需求外不应放行。
- **rustdoc 新增 `invalid_markdown_table`**：文档注释里的 Markdown 表格某行列数超出表头时，1.100.0 的 `rustdoc` 默认发出 `warning: table row has too many columns`，提示用 `\|` 转义竖线。
- **减少 `#[expect(dead_code)]` 误报**：外层项已触发 `dead_code` 诊断时，其内部嵌套的 `#[expect(dead_code)]` 不再误报 `unfulfilled_lint_expectations`。

### 3. 编译器后端、内存模型与平台支持

以下几项底层演进据官方发布说明和 rustc 文档整理，未在特定目标上单独复现：

- **稳定 `-Cprofile-sample-use`**：支持基于采样的性能优化（Sample-based PGO / AutoFDO），接收 `.prof` 文件。相比传统插桩 PGO，可直接利用生产环境的硬件性能计数器（如 Linux `perf`）采集数据，无需部署带插桩开销的二进制。
- **孤儿规则检查启用下一代 trait 求解器**：在 orphanck 阶段默认切换至 next-generation trait solver，推进编译器核心求解器替换。
- **`volatile` 内存模型澄清**：允许对非 Rust 抽象机（non-AM）内存的 `volatile` 访问触发硬件 trap，更贴合嵌入式与内核 MMIO 场景。
- **平台与工具更新**：`wasm32-wasip3` 晋升 Tier 2；`i686-pc-windows-gnu` 调整为不带主机工具的 Tier 2；新增 Tier 3 目标 `powerpc64-sony-ps3`。标准库 Unicode 升级至 `18.0.0`；`libtest` 允许重复传入 `--test-threads` 和 `--color`，后传参数覆盖先传参数。

## 六、Cargo 的四项工程化升级

为避免受全局 Cargo 配置干扰，下面所有 Cargo 示例均在独立的临时目录中显式指定 `CARGO_HOME` 和 `--target-dir` 执行。

### 1. Cargo 原生 lint 与未使用依赖检查

rustc 虽早有 `unused_crate_dependencies`，但在多目标 Cargo 包里极易误报：一个依赖写在 `[dependencies]` 里，单独编译某个 bin 时，rustc 看不到其他 bin 是否用到了它。

Cargo 1.100 推出了由 `[lints.cargo]`（或 `[workspace.lints.cargo]`）控制的原生 lint 系统。按 [Cargo Lints 文档](https://doc.rust-lang.org/1.100.0/cargo/reference/lints.html)，首批内置 lint 分为三档：

| 默认级别 | 内置 lint |
|----------|-----------|
| `allow` | `manual_readme`、`non_kebab_case_{bins,features,packages}`、`non_snake_case_{features,packages}` |
| `warn` | `unused_dependencies`、`unused_workspace_dependencies`、`unused_workspace_package_fields`、`redundant_homepage`、`missing_lints_inheritance`、`blanket_hint_mostly_unused`、`unknown_lints` |
| `deny` | `text_direction_codepoint_in_comment`、`text_direction_codepoint_in_literal` |

最直接影响日常开发的是 `cargo::unused_dependencies`。在一个包含两个二进制目标（`draw` 使用 `ink`，`inspect` 不使用）且同时声明了未使用依赖 `spare-ink` 和未用开发依赖 `test-ink` 的包中，Cargo 1.100.0 表现如下：

- `cargo check --bin inspect`：不报警告（未构建全部目标，无法排除其他目标使用依赖的可能）。
- `cargo check`：只报 `spare-ink`，不误报 `ink`，并直接定位到 `Cargo.toml` 声明行：

```text
warning: unused dependency `spare-ink`
 --> Cargo.toml:8:1
  |
8 | spare-ink = { path = "../spare-ink" }
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: `cargo::unused_dependencies` is set to `warn` by default
help: consider removing the dependency on `spare-ink`
```

- `cargo check --all-targets`：仍然只报 `spare-ink`。官方文档说明，该 lint 目前不检查 `[dev-dependencies]`。在 `[lints.cargo]` 中设置 `unused_dependencies = "deny"` 后，`cargo check` 会因 `spare-ink` 直接失败。Cargo 1.99.0 下均无此提示。

> **清理前务必检查特性合并**：被 `unused_dependencies` 警告的那一行未必能直接删除，它可能在工作区里替另一个成员打开了某个 Cargo feature。

在 `resolver = "2"` 的工作区中，`worker` 通过 `render` 间接依赖 `palette`（默认关闭 `color` 特性），同工作区的 `app` 源码中未引用 `palette`，仅在 `app/Cargo.toml` 中声明了：

```toml
# app/Cargo.toml
[dependencies]
palette = { path = "../palette", features = ["color"] }
```

运行 `cargo check --workspace` 时，Cargo 1.100.0 对 `app` 发出 `unused dependency palette` 警告。但对比实际构建结果，删除这行声明会直接改变 `worker` 的运行行为：

| 构建命令与清单状态 | `worker` 输出 |
|--------------------|---------------|
| 保留 `app` 依赖，`cargo build --workspace` | `color` |
| 保留 `app` 依赖，`cargo build -p worker` | `plain` |
| 删除 `app` 依赖，`cargo build --workspace` | `plain` |

这是因为同一次构建会合并所有参与成员为同一依赖开启的 feature。用 `cargo tree --workspace -e features -i palette` 可以直接追查来源：

```text
palette v0.1.0 (...)
├── palette feature "color"
│   └── app v0.1.0 (...)
│       └── app feature "default" (command-line)
└── palette feature "default" (command-line)
    ├── app v0.1.0 (...) (*)
    └── render v0.1.0 (...)
        └── render feature "default" (command-line)
            └── worker v0.1.0 (...)
                └── worker feature "default" (command-line)
```

清理被警告的依赖前，建议先运行 `cargo tree -e features -i <crate>`，再按实际发布所用的构建范围重新验证。

### 2. 中间构建目录换了新布局

同一包启用不同 `--features` 时，Cargo 通过配置哈希区分中间产物。旧布局把哈希拼在文件名里（如 `deps/layout_demo-<哈希>`），但部分平台要求固定文件名，导致哈希无法覆盖全部文件，多 Git worktree 共享构建目录时更容易冲突。

Cargo 1.100 改为按构建步骤组织目录，把配置哈希移到父目录名中，为后续的缓存垃圾回收、跨工作区去重和细粒度锁铺路。分别用默认配置与 `--features blue` 构建 `layout-demo`，在 `aarch64-apple-darwin` 上可见中间可执行文件路径变化如下（`<哈希>` 为配置哈希）：

- Cargo 1.99.0：`target/debug/deps/layout_demo-<哈希>`
- Cargo 1.100.0：`target/debug/build/layout-demo/<哈希>/out/layout_demo`

中间路径一直属于内部实现，没有稳定性承诺。打包或测试脚本应改为解析 Cargo 的 JSON 构建消息，从 `compiler-artifact` 读取 `executable`（查库产物则读 `filenames`）：

```bash
cargo build --message-format=json |
  jq -r '
    select(.reason == "compiler-artifact")
    | select(.target.name == "layout-demo")
    | .executable // empty'
```

在默认构建和 `--features blue` 下分别执行这条管道，取到的路径均为最终产物目录下的 `debug/layout-demo`，直接执行分别输出 `blue=false` 和 `blue=true`。

### 3. 依赖冷却期：`global-min-publish-age`

恶意版本或被劫持的 crate 通常在发布后几小时到几天内被发现并撤回。基于 [RFC 3923](https://rust-lang.github.io/rfcs/3923-cargo-min-publish-age.html)，Cargo 1.100 稳定了发布冷却期机制。在项目或用户的 `.cargo/config.toml` 中写入：

```toml
[registry]
global-min-publish-age = "3 days"
```

在临时搭建的本地稀疏索引仓库中，放入 `tint-demo` 的两个版本：`1.0.0` 发布时间设为 10 天前，`1.1.0` 设为 1 小时前，项目声明依赖 `"1.0"`。

- 冷却期为 0 时，`cargo run` 直接解析并运行 `1.1.0`。
- 开启 3 天冷却期后重新生成 `Cargo.lock`，Cargo 跳过过新的 `1.1.0` 并给出清晰提示：

```text
Locking 1 package to highest compatible version as of 3 days ago
 Adding tint-demo v1.0.0 (registry `local`) (available: v1.1.0, published 1 hour ago)
```

如果新版本包含紧急安全修复、不能等待冷却期，可以用环境变量为单条 `cargo update` 放行：

```bash
CARGO_RESOLVER_INCOMPATIBLE_PUBLISH_AGE=allow \
  cargo update -p tint-demo --precise 1.1.0
```

终端输出：

```text
Updating tint-demo v1.0.0 (registry `local`) -> v1.1.0 (published 1 hour ago, minimum age 3 days)
```

随后去掉环境变量、保留 3 天配置再运行 `cargo run --locked`，程序依然运行 `1.1.0`——冷却期只过滤准备新选入的版本，已写入 `Cargo.lock` 的版本不受影响。

使用冷却期时需注意三条边界：

1. 依赖仓库索引必须记录 `pubtime` 字段，缺失 `pubtime` 的版本不会被拦下。
2. 按官方公告，`cargo install` 对被安装工具及其依赖均忽略冷却期，固定版本仍应显式写 `cargo install foo@1.2.3`。
3. 冷却期能压缩刚发布即中招的窗口，但不能替代依赖审查。

### 4. `cargo install` 默认使用包内 `Cargo.lock`

从 Cargo 1.100 起，通过 `cargo install` 安装仓库中的命令行工具时，只要 `.crate` 包内带有 `Cargo.lock`，Cargo 默认直接使用这份锁文件，确保用户装到的二进制与发布时测试的依赖版本一致；`cargo install --locked` 成为不改变解析结果的兼容选项（若发布包缺少 `Cargo.lock`，传 `--locked` 会给出警告）。

在上面的本地稀疏仓库中发布两个工具：`tint-cli 1.0.0` 附带将 `tint-demo` 锁在 `1.0.0` 的 `Cargo.lock`，`tint-cli-nolock 1.0.0` 不带锁文件，仓库同时存在 `tint-demo 1.1.0`。两版 Cargo 分别安装到独立目录后，程序打印的 `tint-demo` 版本如下：

| 安装工具链 | `tint-cli`（默认） | `tint-cli`（`--locked`） | `tint-cli-nolock`（默认） | `tint-cli-nolock`（`--locked`） |
|------------|--------------------|--------------------------|---------------------------|---------------------------------|
| Cargo 1.99.0 | `1.1.0` | `1.0.0` | `1.1.0` | `1.1.0`（警告缺锁文件） |
| Cargo 1.100.0 | `1.0.0` | `1.0.0` | `1.1.0` | `1.1.0`（警告缺锁文件） |

这条规则仅适用于 `cargo install`。在普通项目中，`cargo build --locked` 和 `cargo test --locked` 依然要求存在有效且无需改动的 `Cargo.lock`。

## 七、升级前需要确认的兼容性变化

### 1. 旧 edition 的 never 类型回退改为 `!`

这是 1.100 最容易触发旧项目编译变化的改动。只要旧代码调用了返回泛型 `Result<T, E>` 的函数、用 `?` 传播错误并丢弃成功值，就会命中 never 类型回退。下面这段 `fallback.rs` 将推断出的 `T` 的类型名放进 `Err` 中输出：

```rust
fn inferred<T>() -> Result<T, &'static str> {
    Err(std::any::type_name::<T>())
}

fn outer() -> Result<(), &'static str> {
    inferred()?;
    Ok(())
}

fn main() {
    println!("{}", outer().unwrap_err());
}
```

用 Rust 1.99.0 和 1.100.0 在四个 edition 下分别编译运行（仅用 `type_name` 观察推断结果，不依赖字符串格式稳定性）：

| 编译器 | Rust 2015 | Rust 2018 | Rust 2021 | Rust 2024 |
|--------|-----------|-----------|-----------|-----------|
| 1.99.0 | `()` | `()` | `()` | `!` |
| 1.100.0 | `!` | `!` | `!` | `!` |

为什么一个 `?` 会牵涉到 `!`？展开来看，`inferred()?` 的成功分支产出泛型 `T`，错误分支从 `outer` 提前 `return`（类型为 `!`），编译器需要将 `!` 隐式转换到另一分支的类型 `T`。但分号丢弃了成功值，下一行的 `Ok(())` 只约束了 `outer` 自身的返回类型，等推断结束时 `T` 仍未确定，于是落入回退类型：1.99.0 在 2024 之前的 edition 回退为 `()`，1.100.0 则统一回退为 `!`。

若 `inferred` 对 `T` 带有 trait 约束（例如 `T: Default`），在 Rust 2021 下：

- Rust 1.99.0 默认已给出未来兼容错误 `this function depends on never type fallback being ()`；
- Rust 1.100.0 直接报 `E0277: the trait bound !: Default is not satisfied`，并在 help 中提示可能意在传入 `()`。

修复方式在两个版本通用：显式写上 `inferred::<()>()?;`。若报错落在第三方依赖中，先尝试 `cargo update`，官方团队已提前向受影响的公共 crate 提交了兼容修复。

### 2. `fn() -> !` 与 `fn() -> Infallible` 的 trait 实现冲突

因为 `fn() -> !` 早已在稳定版合法，旧代码可以为两种函数指针分别实现同一个本地 trait：

```rust
use std::convert::Infallible;

trait Marker {}
impl Marker for fn() -> ! {}
impl Marker for fn() -> Infallible {}
```

Rust 1.99.0 允许这段代码通过；Rust 1.100.0 报 `E0119: conflicting implementations of trait Marker for type fn() -> !`。删除重复的一份实现即可同时覆盖两种写法。

### 3. `Box` 的部分 trait 实现泛化到任意分配器

配合自定义分配器稳定，标准库将原本仅针对 `Box<T>`（即 `Box<T, Global>`）的部分 trait 实现泛化到了 `Box<T, A>`（[PR #161946](https://github.com/rust-lang/rust/pull/161946)），例如 `std::io::Write`。向 `Box::new_in(Vec::<u8>::new(), System)` 调用 `Write::write_all(b"ok")` 在 1.100.0 上可直接写入 `[111, 107]`（`System` 管理外层 `Box`，内层 `Vec` 仍用 `Global`）。发布说明提醒，极少数依赖原有窄实现范围做类型推断的代码，升级后可能需要补写类型标注。

### 4. 其他行为与语法调整

- **`str::escape_debug` 总是转义字形扩展字符**：[PR #158303](https://github.com/rust-lang/rust/pull/158303) 调整了对组合重音等字形扩展字符的处理。对 `"e\u{0301}"` 调用 `escape_debug()`，1.99.0 输出 `é`，1.100.0 输出 `e\u{301}`。涉及精确文本断言的快照测试需留意核对。
- **语法层彻底移除 `box` 模式**：[PR #156749](https://github.com/rust-lang/rust/pull/156749) 移除了长期未稳定的 `let box x = b;` 语法。1.99.0 报 `E0658: box pattern syntax is experimental`，1.100.0 直接报 `error: box patterns have been removed`。
- **类型与宏规则收紧**（据发布说明整理，未单独复现）：
  - 闭包在 `if let` 中捕获变量更精确（[#154210](https://github.com/rust-lang/rust/pull/154210)），可能暴露原本被整块捕获掩盖的借用错误；
  - 更多位置检查 trait 对象的 dyn 兼容性（[#154992](https://github.com/rust-lang/rust/pull/154992)）；
  - `let ... else` 尊重 `macro_rules!` 中 `expr` 元变量分组（[#158515](https://github.com/rust-lang/rust/pull/158515)）；
  - `ambiguous_glob_imported_traits` lint 在略多场景触发（[#161805](https://github.com/rust-lang/rust/pull/161805)）；
  - `Iterator::{min, max}(_by_key)` 内部更换了调用的 `Ord` 方法（[#160203](https://github.com/rust-lang/rust/pull/160203)），遵守 `Ord` 一致性契约的类型语义不变。
- **目标平台与 ABI 调整**（据发布说明整理，未单独复现）：
  - x86 上 `"Rust"` ABI 在目标恒定支持 SSE 时改用 SSE 寄存器；x86 软浮点目标禁止开启 SSE/AVX；
  - ARM 目标拒绝会改变 ABI 的 `-Ctarget-feature`/`-Ctarget-cpu`；部分 ARMv7-A 硬浮点目标修正了误把 `d32` 列入基线的问题（受影响时可显式传 `-Ctarget-feature=+d32`）；
  - `windows-gnullvm` 改为总是静态链接 `libunwind`；WASI SDK 从 33 升至 34。

### 5. 补充上一篇：Rust 1.99.1 定向修复

在 1.99.0 与 1.100.0 之间，官方于 2026 年 10 月 15 日发布了 [Rust 1.99.1](https://doc.rust-lang.org/1.100.0/releases.html#version-1991-2026-10-15)。据 `RELEASES.md`，该补丁包含三项修复：修正 x86_64 上 FFI `bool` 返回值误编译（[#163911](https://github.com/rust-lang/rust/issues/163911)）、修复 UEFI/嵌入式目标缺失 `wcslen` 符号（[#163614](https://github.com/rust-lang/rust/issues/163614)），以及为 macOS 上的 `rust-lld` 补上 `libLLVM` rpath（[#163947](https://github.com/rust-lang/rust/issues/163947)）。如果团队暂缓升级到 1.100.0，仍停留在 [Rust 1.99.0](https://zhuanlan.zhihu.com/p/2090105221263390225) 的项目建议至少先升至 1.99.1。

## 八、怎样验证自己的项目？

对已经提交 `Cargo.lock` 的工作区，先保留原 edition 与锁文件，单独切换编译器跑一遍完整检查：

```bash
rustup toolchain install 1.100.0 --profile minimal
rustc +1.100.0 -vV
cargo +1.100.0 check --workspace --all-targets --locked
cargo +1.100.0 test --workspace --locked
```

升级排查可按顺序核对六项：

1. **泛型 `?` 报 `E0277`（`!` 未实现某 trait）**：确认是否丢弃了成功值，必要时补写 `::<()>`；若在外部依赖中，先 `cargo update`。
2. **函数指针报 `E0119`**：检查是否同时为 `fn() -> !` 和 `fn() -> Infallible` 实现了同一个 trait。
3. **unsafe 代码触发 `undropped_manually_drops`**：检查是否把 `*mut ManuallyDrop<T>` 传给了 `ptr::drop_in_place`，改为 `ManuallyDrop::drop`。
4. **新增 `cargo::unused_dependencies` 警告**：删行前先跑 `cargo tree --workspace -e features -i <依赖名>`，确认没有替其他成员打开特性。
5. **打包脚本找不到产物**：检查是否硬编码了 `target/<profile>/deps/...`，改用 `--message-format=json` 读取 `executable` 或 `filenames`。
6. **文档与快照测试**：跑一遍 `cargo +1.100.0 doc --no-deps`，顺手确认 `invalid_markdown_table` 警告与涉及组合字符的 `escape_debug` 断言。

## 本文做过哪些验证？

| 验证项 | 验证方式 | 结果 |
|--------|----------|------|
| never 类型与 `Infallible` | 1.99.0 / 1.100.0 对照 `Result<U, !>`、`Err(never) => never`、`AsRef` for `!` | 1.100.0 全部通过；1.99.0 分别报 `E0658` 与 `E0308` |
| `Allocator` 与 `Vec`/`Box` | `MaxBlock` 共享、`&dyn Allocator`、拆解重建四元组、`Arc::new_in`、`String::new_in` | `Vec`/`Box` 与 `&dyn Allocator` 通过；`Arc::new_in` 报 `allocator_ext`，`String::new_in` 报 `E0599` |
| `DropGuard` 与 `conjure_zst` | 帧写入提交与回滚；ZST 构造、`u8` 运行时与 `const` 求值 | 提交/回滚正确；`u8` 运行时 panic、`const` 报 `E0080` |
| 其他标准库 API | 运行 `Box::take`/`map`、`Rc`/`Arc::map`、`UnsafeCell` 三方法、`trim_prefix`/`suffix`、`Vec::from_fn` 等 | 1.100.0 全部符合文档；1.99.0 报未稳定特性或名称冲突 |
| `extern "custom"` | `aarch64` 下 `#[unsafe(naked)]` + `asm!` 调用；测试直接 Rust 调用 | 1.100.0 输出 `42`，直接调用被拒；1.99.0 报 `E0658` |
| 新语言与 rustdoc lint | 对照 `drop_in_place` 三种指针、重复 `#[repr(C)]`、`aligned_fields_in_packed`、`invalid_markdown_table` | 1.100.0 按预期报 warn/deny；`aligned_fields_in_packed` 加 `allow` 后尺寸为 9 字节 |
| Cargo `unused_dependencies` | 多 bin 包对照不同目标参数；4 成员 workspace 删掉未直接使用但开特性的依赖 | 仅全目标构建报未使用普通依赖；删掉 `app` 依赖后 `worker` 输出由 `color` 变 `plain` |
| 构建布局与 JSON 消息 | 1.99.0 / 1.100.0 对比默认与 `--features blue` 目录，用 `jq` 解析 `executable` | 中间路径由 `debug/deps/...-<哈希>` 变为 `debug/build/.../<哈希>/out/...`；JSON 路径运行正常 |
| 冷却期与 `cargo install` | 本地稀疏索引（10 天前 `1.0.0` vs 1 小时前 `1.1.0`）测 0/3 天与放行；两版 Cargo 测含锁/无锁安装 | 3 天冷却期选 `1.0.0`，放行后可锁定 `1.1.0`；1.100.0 默认用包内锁文件装出 `1.0.0` |
| 兼容性对照 | 4 edition × 2 版本运行 `fallback.rs`；对照重叠 impl、`Box<Vec<u8>, System>`、`escape_debug`、`box` 模式 | 1.100.0 全 edition 回退 `!`；重叠 impl 报 `E0119`；`escape_debug` 由 `é` 变 `e\u{301}` |

## 写在最后

回看 Rust 1.100.0，它把几条推进了很久的路线真正接上了。`!` 能在任意类型位置书写并与 `Infallible` 融为一体，让类型系统里“绝不会失败”的表达告别了双轨并存，代价是把旧 edition 的回退规则一次性拉齐。`Allocator` API 从最常用的 `Vec` 和 `Box` 起步，虽然其他集合还要等后续版本，但核心 trait、动态分发支持以及“指针与分配器成对流转”的契约已经定型。

Cargo 的四项更新同样务实：原生 lint 把未使用依赖检查拉回到了正确层级，新构建目录为后续缓存能力腾出了空间，依赖冷却期与 `cargo install` 默认锁文件则直接补上了工程流程里的两处缺口。对多数团队来说，这次升级既值得尽早验证，也适合借机顺手清查一次工作区里的依赖、特性和打包脚本。

## 参考资料

- [Rust 1.100.0 官方发布公告](https://blog.rust-lang.org/2026/11/12/Rust-1.100.0/)
- [Rust 1.100.0 详细发布说明](https://doc.rust-lang.org/1.100.0/releases.html#version-11000-2026-11-12)
- [Cargo 1.100 更新日志](https://doc.rust-lang.org/1.100.0/cargo/CHANGELOG.html#cargo-1100-2026-11-12)
- [Rust Reference：never type 与隐式转换](https://doc.rust-lang.org/1.100.0/reference/types/never.html)
- [Edition Guide：never type 回退规则](https://doc.rust-lang.org/edition-guide/rust-2024/never-type-fallback.html)
- [`std::convert::Infallible` 标准库文档（1.100.0）](https://doc.rust-lang.org/1.100.0/std/convert/type.Infallible.html)
- [`std::alloc::Allocator` 标准库文档（1.100.0）](https://doc.rust-lang.org/1.100.0/std/alloc/trait.Allocator.html)
- [`std::alloc::Global` 标准库文档（1.100.0）](https://doc.rust-lang.org/1.100.0/std/alloc/struct.Global.html)
- [`core::mem::DropGuard` 标准库文档（1.100.0）](https://doc.rust-lang.org/1.100.0/core/mem/struct.DropGuard.html)
- [`core::mem::conjure_zst` 标准库文档（1.100.0）](https://doc.rust-lang.org/1.100.0/core/mem/fn.conjure_zst.html)
- [`std::boxed::Box` 标准库文档（1.100.0）](https://doc.rust-lang.org/1.100.0/std/boxed/struct.Box.html)
- [`std::cell::UnsafeCell` 标准库文档（1.100.0）](https://doc.rust-lang.org/1.100.0/std/cell/struct.UnsafeCell.html)
- [Rust Reference：`extern "custom"` ABI](https://doc.rust-lang.org/1.100.0/reference/items/functions.html#extern-custom)
- [rustc 代码生成选项：`-Cprofile-sample-use`](https://doc.rust-lang.org/1.100.0/rustc/codegen-options/index.html#profile-sample-use)
- [Cargo Lints 参考文档（1.100.0）](https://doc.rust-lang.org/1.100.0/cargo/reference/lints.html)
- [Cargo 依赖解析与发布冷却期（1.100.0）](https://doc.rust-lang.org/1.100.0/cargo/reference/resolver.html#publish-age)
- [RFC 3923：Cargo 最小发布冷却期](https://rust-lang.github.io/rfcs/3923-cargo-min-publish-age.html)
- [Cargo 构建缓存与目录布局（1.100.0）](https://doc.rust-lang.org/1.100.0/cargo/reference/build-cache.html)
- [Cargo JSON 构建消息（1.100.0）](https://doc.rust-lang.org/1.100.0/cargo/reference/external-tools.html#json-messages)
- [`cargo install` 锁文件处理说明（1.100.0）](https://doc.rust-lang.org/1.100.0/cargo/commands/cargo-install.html#dealing-with-the-lockfile)
- [`Box` trait 实现泛化 PR #161946](https://github.com/rust-lang/rust/pull/161946)
