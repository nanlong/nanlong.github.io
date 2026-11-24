---
title: "Rust 1.98 深度解析：代数浮点运算可重排，整数格式化可复用缓冲区"
date: 2026-08-24T00:00:00+08:00
draft: false
tags: ["Rust", "浮点数", "性能", "标准库", "版本更新"]
categories: ["技术"]
description: "Rust 1.98 将代数浮点运算与 NumBuffer/format_into 带入稳定版，并新增借用片段定位、UTF-16 字节序解码、原子切片转换和 FFI 检查。本文结合官方资料与实际编译结果，说明这些变化的准确语义和使用限制。"
---

Rust 1.98.0 于 2026 年 8 月 20 日发布。这次更新中，最值得细看的两项新功能都与性能有关：`f32` 和 `f64` 新增代数运算方法，允许编译器重排指定的浮点运算；所有基础整数类型新增 `format_into`，可以把十进制文本写入调用者提供的 `NumBuffer`。

标准库还加入了借用片段定位、UTF-16 大小端解码、普通切片与原子切片之间的转换，并明确保证：`ManuallyDrop<Box<_>>` 中的 `Box` 被手动析构后，移动外层 `ManuallyDrop` 不再属于未定义行为。编译器则新增了两组检查，用来发现错误的 `c_void` 返回值和运行时符号定义。完整清单见 [Rust 1.98.0 发布说明](https://doc.rust-lang.org/stable/releases.html#version-1980-2026-08-20)。

本文中的可运行示例均使用下面这套正式版工具链重新编译和运行：

```text
rustc 1.98.0 (88d9e12ae 2026-08-18)
cargo 1.98.0 (797e8a9bc 2026-08-05)
host: aarch64-apple-darwin
LLVM version: 22.1.8
```

文中还会专门注明三类证据：Rust 官方文档、上游项目给出的测试数据，以及本文在上述工具链上的实际结果。三者不会混在一起表述。

<!--more-->

## 一张表看懂 Rust 1.98

| 变化 | 1.98 中可以确认的行为 |
|------|-----------------------|
| `algebraic_*` | 只允许编译器重排明确调用这些方法的浮点运算 |
| `NumBuffer` 与 `format_into` | 把基础整数格式化为十进制 `&str`，返回值借用调用者的缓冲区 |
| `substr_range` / `subslice_range` | 根据指针位置计算借用片段在原数据中的区间，不搜索内容 |
| UTF-16 大小端解码 | 直接从 `&[u8]` 解码 UTF-16LE 或 UTF-16BE |
| 原子切片转换 | 在对齐要求允许的目标上，把独占的普通切片临时视为原子切片 |
| `ManuallyDrop<Box<_>>` | 手动析构内部 `Box` 后，可以移动外层 `ManuallyDrop` |
| FFI 检查 | 警告 `c_void` 返回值，并拒绝一部分签名错误的运行时符号 |
| 生命周期转换 | 在动态大小类型转换时，可以缩短 `&mut` 的生命周期，即使它处于不变位置 |

下面从代数浮点运算讲起。

## 一、`algebraic_*`：允许编译器重排指定的浮点运算

### 为什么普通浮点加法不能随意重排？

浮点加法不满足结合律。对 `f32` 来说，`(a + b) + c` 和 `a + (b + c)` 可能在不同位置发生舍入，因此结果也可能不同。普通的 `a + b + c + d` 要按照左结合顺序计算；如果改用 `algebraic_add`，编译器便可以选择别的组合方式。这一点由 [Rust 1.98 官方公告](https://blog.rust-lang.org/2026/08/20/Rust-1.98.0/) 明确说明。

这项改动的直接用途是给编译器更多向量化机会。最初的 [API 变更提案](https://github.com/rust-lang/libs-team/issues/532) 提供了一组点积测试：在 i7-10875H 上，Rust 1.84.1 使用 `-C opt-level=3 -C target-feature=+avx2,+fma` 编译的普通版本耗时 `84 µs`；Rust 1.86.0-nightly 使用相同优化选项和代数内建函数的版本耗时 `10 µs`。这是提案作者在特定硬件和特定编译器上的测试，不是本文对 Rust 1.98 的通用性能结论。

### 同一组数据，为什么会得到不同结果？

Rust 1.98 为 `f32` 和 `f64` 稳定了五个方法：

```text
algebraic_add
algebraic_sub
algebraic_mul
algebraic_div
algebraic_rem
```

只有写成这些方法调用的运算才采用新的语义。下面的程序只放宽了累加时的加法：

```rust
#[inline(never)]
fn strict_sum(values: &[f32]) -> f32 {
    values.iter().copied().fold(0.0, |sum, value| sum + value)
}

#[inline(never)]
fn algebraic_sum(values: &[f32]) -> f32 {
    values
        .iter()
        .copied()
        .fold(0.0, |sum, value| sum.algebraic_add(value))
}

fn main() {
    let values: Vec<_> = [1.0e20_f32, 1.0, -1.0e20, 1.0]
        .into_iter()
        .cycle()
        .take(4_096)
        .collect();

    println!("strict sum    = {}", strict_sum(&values));
    println!("algebraic sum = {}", algebraic_sum(&values));
}
```

本文使用 `cargo run --release` 得到的结果是：

```text
strict sum    = 1
algebraic sum = 0
```

序列 `[1e20, 1, -1e20, 1]` 重复了 1024 次；如果按照数学上的实数精确计算，结果是 `2048`。普通版本按顺序累加，得到 `1`。代数版本允许编译器重排，在本次构建中得到 `0`。这次运行只能证明两种运算的语义不同，不能据此断定编译器采用了某一种固定的重排顺序。

### 标准库到底允许哪些变化？

[`f32` 的代数运算文档](https://doc.rust-lang.org/1.98.0/core/primitive.f32.html#algebraic-operators) 列出了这些可能的优化：

- 合并或重排一串运算；
- 在除法和乘以倒数之间转换；
- 忽略零的正负号；
- 采用能够缩短依赖链或便于向量化的计算顺序。

标准库不承诺这些运算具有固定精度。`NaN`、正负无穷和 `-0.0` 也可能出现与普通运算不同的结果。由于编译器可以选择不同的优化方式，同一组输入甚至可能在同一次程序运行中产生不同结果。

不过，代数运算本身不会造成未定义行为。官方文档同时强调，unsafe 代码不能依赖其返回值的任何数值性质来保证内存安全。

因此，是否采用 `algebraic_*` 应当由基准测试和误差测试决定。账务、共识、签名输入、确定性回放以及要求逐位一致的测试，都不适合使用这种没有固定结果保证的运算。这是本文根据上述语义给出的使用建议，不是编译器替调用者执行的限制。

## 二、`NumBuffer`：整数格式化不必先创建 `String`

Rust 1.98 为所有基础整数类型加入了 `format_into`。它接收 `&mut NumBuffer<Self>`，把带符号或不带符号的十进制文本写入缓冲区，然后返回一个借用该缓冲区的 `&str`。[`NumBuffer` 文档](https://doc.rust-lang.org/1.98.0/core/fmt/struct.NumBuffer.html) 还说明，缓冲区的大小按对应整数类型能够出现的最大十进制位数确定。

```rust
use core::fmt::NumBuffer;

fn main() {
    let mut buffer = NumBuffer::new();

    for value in [0_i64, -1_972, i64::MIN, i64::MAX] {
        println!("formatted = {}", value.format_into(&mut buffer));
    }
}
```

实际输出如下：

```text
formatted = 0
formatted = -1972
formatted = -9223372036854775808
formatted = 9223372036854775807
```

这个接口有三个需要注意的地方。

第一，稳定的类型路径是 `core::fmt::NumBuffer`。本文也实际尝试了 `std::fmt::NumBuffer`，Rust 1.98.0 会报 `E0433`，提示 `std::fmt` 中没有这个类型。因为正确入口位于 `core`，`no_std` 代码也可以使用它。

第二，返回的 `&str` 借用了 `NumBuffer`。如果上一次返回的文本仍在使用，代码就不能再次可变借用同一个缓冲区。下面这种写法在 Rust 1.98.0 中会报 `E0499`：

```rust
let mut buffer = core::fmt::NumBuffer::new();
let first = 42_u32.format_into(&mut buffer);
let second = 7_u32.format_into(&mut buffer); // E0499
println!("{first} {second}");
```

第三，`format_into` 提供的是基础整数的十进制格式化，不包含 `write!` 所支持的宽度、填充、进制等格式选项。官方公告引用的 `itoa-benchmark` 显示，它在那组测试中的表现与 `itoa` 接近。本文没有重新测量这项性能，因此不会把上游数据写成所有项目都能得到的结果。

## 三、`substr_range`：直接取得借用片段的位置

解析器经常先用 `split` 得到一个借用片段，随后还要记录它在原字符串中的位置。`str::substr_range` 和 `[T]::subslice_range` 不会按内容搜索，而是根据指针位置计算该片段在原数据中的区间。

```rust
fn main() {
    let line = "alpha, beta, alpha";
    let ranges: Vec<_> = line
        .split(", ")
        .map(|field| line.substr_range(field).expect("field comes from line"))
        .collect();

    println!("ranges = {ranges:?}");
    println!("inner  = {:?}", "<rust-1.98>".strip_circumfix('<', '>'));
}
```

本文的运行结果是：

```text
ranges = [0..5, 7..11, 13..18]
inner  = Some("rust-1.98")
```

两个内容相同的 `alpha` 得到了各自的位置，这也符合 [`str::substr_range` 文档](https://doc.rust-lang.org/1.98.0/std/primitive.str.html#method.substr_range) 中“不搜索字符串”的定义。

返回值使用新的 `core::range::Range<usize>`，而 `0..5` 目前仍会生成旧的 `core::ops::Range<usize>`。两者不是同一个类型，但可以双向 `From` 转换。1.98 还稳定了 `std::range::legacy`；在 `no_std` 中也可以使用 `core::range::legacy`。本文已在 1.98.0 中编译并测试这两条路径和双向转换。按照新范围类型的文档，未来的 edition 计划让 `start..end` 生成新类型，但 1.98 尚未改变现有语法。

`substr_range` 和 `subslice_range` 都有一个容易忽略的限制：如果传入的是另一个独立对象的零长度切片，而且它的地址恰好落在原数据的起点或终点，方法可能返回起点或末尾的空区间。它们适合处理确实从原数据借出的片段，不适合用来鉴别任意引用来自哪里。此外，`subslice_range` 用于零大小类型时会触发 panic。

同一版本稳定的 `strip_circumfix` 会同时检查并移除前缀与后缀。只匹配一端，或者前后两段发生重叠，都会返回 `None`。本文对成功、单端不匹配和重叠三种情况都做了测试。

## 四、UTF-16 与原子切片：少写两类手工转换

### 1. 直接从字节解码 UTF-16LE 和 UTF-16BE

`String::from_utf16le`、`String::from_utf16be` 以及各自的 `*_lossy` 版本都接收 `&[u8]`。调用者拿到带有明确字节序的 UTF-16 数据后，不必先把每两个字节拼成 `u16`。

```rust
use std::num::NonZeroU16;

fn main() {
    let utf16le = [0x52, 0x00, 0x75, 0x00, 0x73, 0x00, 0x74, 0x00];
    println!("utf16le = {}", String::from_utf16le(&utf16le).unwrap());

    let port = NonZeroU16::from_str_radix("1F90", 16).unwrap();
    println!("port = {port}");
}
```

实际输出为：

```text
utf16le = Rust
port = 8080
```

严格解码会拒绝奇数字节和没有配对的代理项；`*_lossy` 会用 `U+FFFD` 替换这些无效数据。本文分别测试了奇数字节和单独代理项。`NonZeroU16::from_str_radix` 则会同时完成进制解析和非零检查，传入字符串 `"0"` 时返回 `ParseIntError`。这些行为与 [`String::from_utf16le` 文档](https://doc.rust-lang.org/1.98.0/alloc/string/struct.String.html#method.from_utf16le) 及 `NonZero` 的实现一致。

### 2. 在普通切片与原子切片之间转换

原子整数、原子布尔和原子指针新增或稳定了 `from_mut`、`from_mut_slice` 和 `get_mut_slice`。下面的例子先创建普通数组，再把它临时作为原子切片交给四个作用域线程：

```rust
use std::sync::atomic::{AtomicU32, Ordering};

fn main() {
    let mut counters = [0_u32; 4];

    {
        let atomics = AtomicU32::from_mut_slice(&mut counters);
        std::thread::scope(|scope| {
            for (index, counter) in atomics.iter().enumerate() {
                scope.spawn(move || counter.store((index + 1) as u32, Ordering::Relaxed));
            }
        });
    }

    println!("counters = {counters:?}");
}
```

实际结果是：

```text
counters = [1, 2, 3, 4]
```

安全性来自独占的可变借用：转换发生时，`&mut [u32]` 保证没有其他代码同时访问这块数据；原子切片还在使用时，借用检查也不允许代码通过普通整数引用读写同一数组。[`AtomicU32::from_mut_slice` 文档](https://doc.rust-lang.org/1.98.0/core/sync/atomic/type.AtomicU32.html#method.from_mut_slice) 给出的示例同样在线程中使用 `Relaxed`，并在线程全部结束后读取普通数组。

这里的 `Relaxed` 只适用于这个例子：四个线程分别写入不同元素，普通读取又发生在线程全部结束之后。若原子变量还承担数据发布或状态同步的职责，仍需按实际并发关系选择内存顺序。另一个限制来自对齐要求：`from_mut` 和 `from_mut_slice` 只在原子类型与底层类型对齐相同的目标上提供，跨平台项目需要在对应目标上编译确认。

## 五、`ManuallyDrop<Box<_>>` 补上稳定保证

在 Rust 1.96 之前，下面这段代码被认定为未定义行为：

```rust
use std::mem::ManuallyDrop;

fn main() {
    let mut value = ManuallyDrop::new(Box::new(42));
    unsafe {
        ManuallyDrop::drop(&mut value);
    }
    let _moved = value;
}
```

问题不在于重复释放。旧规则认为，已经析构的 `Box` 不能再被移动；这项限制也传递给了外层的 `ManuallyDrop<Box<_>>`。因此，即使 `ManuallyDrop` 阻止了自动析构，最后一行仍然会移动一个内部 `Box` 已经失效的值。

编译器在 Rust 1.96 中修复了这个问题。Rust 1.98 又更新了 [`ManuallyDrop` 文档](https://doc.rust-lang.org/1.98.0/std/mem/struct.ManuallyDrop.html#pre-196-interaction-with-box)，正式保证手动析构内部 `Box` 后，移动外层 `ManuallyDrop` 不再属于未定义行为。本文也确认这段代码可以通过 Rust 1.98.0 编译。

这项保证只解决“析构后移动外层包装”这个问题。已经析构的内部值仍然不能再读取。标准库文档还明确指出：如果安全的公共接口可能让调用者访问一个已经析构的 `ManuallyDrop` 字段，那么为外层类型派生 `Debug`、`Clone`、`PartialEq`、`PartialOrd`、`Ord` 或 `Hash` 可能是不可靠的，因为这些派生实现会读取该字段。若只需要控制字段析构顺序，文档建议先调整字段声明顺序。

## 六、两类 FFI 错误可以更早发现

### 1. C 的 `void` 返回值在 Rust 中应写成 `()`

`core::ffi::c_void` 用来表示 `void*` 指向的数据类型，不是 C 函数的 `void` 返回值。Rust 1.98 新增了默认开启的 `c_void_returns` 警告：

```rust
#![allow(dead_code)]

use std::ffi::c_void;

unsafe extern "C" {
    fn legacy_callback() -> c_void;
}

fn main() {}
```

本文实际得到的诊断是：

```text
warning: declarations returning `c_void` are not compatible with C functions returning `void`
help: remove the return type to implicitly return `()`
```

诊断信息还明确说明：Rust 的 `()` 与 C 的 `void` 返回值对应，而 `c_void` 只应通过裸指针与 C 的 `void*` 兼容。

### 2. 自定义运行时符号必须使用正确签名

Rust 1.98 增加了 `invalid_runtime_symbol_definitions` 和 `suspicious_runtime_symbol_definitions`。前者默认拒绝编译，后者默认发出警告。目前检查的对象包括 `memcmp`、`memset`、`strlen` 等 `core` 运行时符号；[发布说明](https://doc.rust-lang.org/stable/releases.html#version-1980-2026-08-20) 还注明，后续版本会继续扩大检查范围。

```rust
#[unsafe(no_mangle)]
pub fn strlen() {}

fn main() {}
```

这段代码在 Rust 1.98.0 中无法编译。诊断同时给出了标准库需要的签名：

```text
error: invalid definition of the runtime `strlen` symbol used by the standard library
note: expected `unsafe extern "C" fn(*const i8) -> usize`
      found    `fn()`
```

这两项检查只覆盖编译器目前认识的运行时符号，不能证明其他 FFI 声明也都正确。因此，检查通过不能代替逐项核对 C 头文件、调用约定和数据布局。

## 七、生命周期转换：1.97.1 不接受，1.98.0 可以接受

Rust 1.98 允许在转换为动态大小类型时缩短 `&mut` 的生命周期，即使这个引用位于不变位置。官方发布说明给出的例子正是 `Cell<&'long mut i32>` 到 `Cell<&'short mut dyn Send>`：

```rust
use std::cell::Cell;

fn erase_and_shorten<'short, 'long: 'short>(
    value: Cell<&'long mut i32>,
) -> Cell<&'short mut dyn Send> {
    value
}
```

本文使用 Rust 1.98.0 编译成功。换成 Rust 1.97.1，编译器会指出 `Cell` 的类型参数 `T` 是不变的（invariant），因而无法满足返回值的生命周期要求。

1.98 新增的规则允许在 `i32` 转成 `dyn Send` 的同时，把独占借用的生命周期从 `'long` 缩短为 `'short`。它没有改变 `Cell` 的型变规则，也没有放松 `&mut` 的独占要求。发布说明还指出，`&mut` 转成共享引用以及共享引用之间的类似转换，此前已经允许缩短生命周期。

## 八、升级前需要确认的兼容性变化

### 1. Rust 1.98.0 有一项仍在处理的严重误编译问题

Rust 官方仓库中的 [#161441](https://github.com/rust-lang/rust/issues/161441) 报告称：同一份 Rama 代码使用 Rust 1.97.1 可以运行，使用 Rust 1.98.0 会生成首个方法槽位为空的虚函数表，safe Rust 通过该槽位动态分发后跳到空地址并触发 `SIGSEGV`。问题报告提供了固定提交和完整复现步骤。

截至 2026 年 8 月 24 日，问题仍为打开状态，标签包括 `P-critical`、`I-miscompile` 和 `regression-from-stable-to-stable`。对应的稳定版修复 PR [#161555](https://github.com/rust-lang/rust/pull/161555) 也尚未合并；其中已经有一项把版本更新为 1.98.1 的提交，但这不等于 1.98.1 已经发布。

因此，如果项目使用了与该报告相近的装箱异步服务、trait 对象和动态分发组合，保守做法是暂时保留 1.97.1，或者至少用 1.98.0 生成的实际程序跑完关键流程。这个建议针对的是已知问题所涉及的代码形态，不表示所有使用 Rust 1.98.0 的程序都会触发误编译。

### 2. 发布说明列出的源码兼容性变化

[Rust 1.98.0 详细发布说明](https://doc.rust-lang.org/stable/releases.html#version-1980-2026-08-20) 还列出了以下变化：

- `repr(transparent)` 对可忽略字段的判断更严格：`repr(C)` 类型、含有私有字段的外部类型和 `#[non_exhaustive]` 类型不再被视为布局显然无关；
- 更多有歧义的导入会报错，其中一部分 `ambiguous_glob_imports` 已从 lint 变成硬错误；
- Windows 的线程局部析构改用纤程本地存储（Fiber Local Storage，FLS）；
- Emscripten 目标统一使用 WebAssembly 异常处理 ABI，并删除 `-Zemscripten-wasm-eh=false`；
- rustfmt 现在会发现写在 `cfg_select!` 中的模块文件，因此可能格式化过去没有处理的文件。

平台支持方面，`thumbv7a-none-eabi`、`thumbv7a-none-eabihf`、`thumbv7r-none-eabi`、`thumbv7r-none-eabihf` 和 `thumbv8r-none-eabihf` 在 1.98 中升为 Tier 2；`powerpc64-unknown-linux-gnuelfv2` 与 `aarch64-unknown-linux-pauthtest` 则作为 Tier 3 目标加入。

## 九、怎样验证自己的项目？

先确认正在使用的精确版本：

```bash
rustc --version --verbose
cargo --version
```

然后再做与项目有关的检查：

1. 查看 #161441、修复 PR 和 Rust 官方发布公告，确认当前工具链是否已经包含修复。
2. 运行项目原有的检查、测试和发布模式构建；如果项目大量使用动态分发，还应运行实际程序，而不是只看 `cargo check`。
3. 搜索返回 `c_void` 的外部函数，以及使用 `no_mangle`、`export_name` 或 `link_name` 的运行时符号，逐项核对 ABI。
4. 对准备改成 `algebraic_*` 的代码分别测量速度和数值误差，不要根据上游的一组点积数据推断自己的结果。
5. 如果库在没有条件编译的代码中使用了本次稳定的新 API，就要把 MSRV 提高到 1.98。`{core,std}::derive` 是发布说明中特别列出的例外：它在 1.96 中已经被意外稳定，因此官方记录的 MSRV 是 1.96。
6. Windows、Emscripten 和嵌入式目标的行为不能用 macOS 上的测试代替，应在实际目标或相应 CI 环境中验证。

## 本文做过哪些验证？

| 项目 | 验证方式 | 结果 |
|------|----------|------|
| 文章中的正常示例 | Rust 1.98.0 下运行 `cargo test`、Clippy 和发布模式程序 | 5 项测试通过，Clippy 无警告，输出与文中一致 |
| `NumBuffer` 的类型路径 | 分别编译 `core::fmt::NumBuffer` 与 `std::fmt::NumBuffer` | 前者通过，后者报 `E0433` |
| `NumBuffer` 的重复借用 | 保留第一次返回的 `&str`，再次可变借用同一缓冲区 | 报 `E0499` |
| 新旧范围类型 | 编译 `core::range::legacy`、`std::range::legacy` 和双向 `From` | 全部通过 |
| UTF-16 与 `NonZero` 的错误分支 | 测试奇数字节、单独代理项和零值 | 严格解码与零值解析返回错误，容错解码返回 `U+FFFD` |
| 生命周期转换 | 同一源码分别交给 Rust 1.98.0 与 1.97.1 | 1.98.0 通过，1.97.1 因 `Cell<T>` 不变而失败 |
| `c_void_returns` | 编译返回 `c_void` 的外部函数声明 | 编译成功并出现预期警告 |
| 运行时符号检查 | 编译错误签名的 `strlen` | `invalid_runtime_symbol_definitions` 拒绝编译 |
| 1.98.0 误编译状态 | 查询 rust-lang/rust 的 issue 与修复 PR | 截至 2026-08-24，两者都未关闭或合并 |

## 写在最后

Rust 1.98 没有给整个 crate 增加一项笼统的“快速浮点”开关，而是让调用者逐个选择可以重排的运算；`NumBuffer` 也没有取代完整的格式化系统，而是提供一个只处理基础整数十进制文本的接口。其他新 API 同样有清楚的边界：片段区间依赖引用位置，原子切片转换依赖独占借用和目标平台的对齐要求，FFI lint 只检查编译器目前认识的错误。

这些功能可以单独评估，但是否用 Rust 1.98.0 编译生产程序，还应结合已知的误编译问题判断。等修复真正合并并由发布团队公布补丁版本之后，再核对精确版本，比只看 `stable` 这个名称更可靠。

## 参考资料

- [Rust 1.98.0 官方发布公告](https://blog.rust-lang.org/2026/08/20/Rust-1.98.0/)
- [Rust 1.98.0 详细发布说明](https://doc.rust-lang.org/stable/releases.html#version-1980-2026-08-20)
- [Rust `f32` 代数运算文档](https://doc.rust-lang.org/1.98.0/core/primitive.f32.html#algebraic-operators)
- [`algebraic_*` API 变更提案](https://github.com/rust-lang/libs-team/issues/532)
- [`NumBuffer` 标准库文档](https://doc.rust-lang.org/1.98.0/core/fmt/struct.NumBuffer.html)
- [`str::substr_range` 标准库文档](https://doc.rust-lang.org/1.98.0/std/primitive.str.html#method.substr_range)
- [`String::from_utf16le` 标准库文档](https://doc.rust-lang.org/1.98.0/alloc/string/struct.String.html#method.from_utf16le)
- [`AtomicU32::from_mut_slice` 标准库文档](https://doc.rust-lang.org/1.98.0/core/sync/atomic/type.AtomicU32.html#method.from_mut_slice)
- [`ManuallyDrop` 与 `Box` 的标准库保证](https://doc.rust-lang.org/1.98.0/std/mem/struct.ManuallyDrop.html#pre-196-interaction-with-box)
- [rustc 默认警告的 lint 列表](https://doc.rust-lang.org/1.98.0/rustc/lints/listing/warn-by-default.html#c-void-returns)
- [rustc 默认拒绝的 lint 列表](https://doc.rust-lang.org/1.98.0/rustc/lints/listing/deny-by-default.html#invalid-runtime-symbol-definitions)
- [Rust 1.98.0 误编译问题 #161441](https://github.com/rust-lang/rust/issues/161441)
- [Rust 1.98 稳定版修复 PR #161555](https://github.com/rust-lang/rust/pull/161555)
