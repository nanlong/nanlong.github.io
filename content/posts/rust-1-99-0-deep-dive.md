---
title: "Rust 1.99 深度解析：用 Rust 定义 C 变参函数，从裸指针读取布局"
date: 2026-10-04T00:00:00+08:00
draft: false
tags: ["Rust", "FFI", "Cargo", "标准库", "版本更新"]
categories: ["技术"]
description: "Rust 1.99 稳定了 C 变参函数定义、裸指针布局查询和一组 Box/Vec 所有权转换 API，Cargo 新增 debug profile 并在 CI 中默认关闭增量编译。本文结合官方资料与 1.98.1 对照实测，说明这些变化的准确语义、使用限制和升级时要注意的地方。"
---

Rust 1.99.0 于 2026 年 10 月 1 日发布。这次最值得细看的变化在 FFI 一侧：用 `extern "C"` 或 `extern "C-unwind"` 声明的 C 变参函数，现在可以直接用 Rust 编写，参数列表 `...` 的类型 `VaList` 与目标平台的 `va_list` 在布局和 ABI 上一致。另一项是三个读取布局的函数：`size_of_val_raw`、`align_of_val_raw` 和 `Layout::for_value_raw`，它们明确了在裸指针上查询大小和对齐的安全条件。

标准库还稳定了 `Box<[T; N]>` 的按值迭代、`Box`/`Vec` 与 `NonNull` 之间的转换、有损 UTF-8 解码的拥有型版本、`VecDeque::retain_back` 和按路径设置文件时间戳。`Box::leak` 的文档则改为不建议再把泄漏的内存收回。Cargo 新增了内置的 `debug` profile，在 CI 中默认关闭增量编译，并允许 2024 edition 的成员覆盖继承来的 `default-features`。完整清单见 [Rust 1.99.0 发布说明](https://doc.rust-lang.org/stable/releases.html#version-1990-2026-10-01)。

本文中的可运行示例均使用下面这套正式版工具链编译和运行：

```text
rustc 1.99.0 (b940084d7 2026-09-28)
cargo 1.99.0 (5f94df478 2026-08-27)
host: aarch64-apple-darwin
LLVM version: 23.1.1
```

需要说明“旧版本不接受、新版本接受”的地方，使用 Rust 1.98.1 做对照。所有实测都在这一台 aarch64 macOS 机器上完成；涉及其他平台的说法（例如 `VaList` 在各目标上与 `va_list` 的 ABI 一致），以官方文档为依据。文中会分开标注三类证据：Rust 官方文档、标准库源码，以及本文在上述工具链上的实际结果。

<!--more-->

## 一张表看懂 Rust 1.99

| 变化 | 1.99 中可以确认的行为 |
|------|-----------------------|
| C 变参函数定义 | `unsafe extern "C" fn f(args: ...)` 可以用 Rust 编写，`args` 的类型是 `VaList` |
| 裸指针布局查询 | `size_of_val_raw` 等三个函数稳定，也可以在 `const` 中调用 |
| `Box::leak` 文档 | 不再建议把泄漏的内存重新组装成 `Box` 释放，改用 `into_raw` / `into_non_null` |
| `Box` / `Vec` 与 `NonNull` | `Box::into_non_null`、`Vec::into_parts` 及对应的重建函数稳定 |
| `Box<[T; N]>` 迭代 | `for x in boxed_array` 可以按值迭代，`into_iter()` 的返回类型随之改变 |
| 有损 UTF-8 | `String::from_utf8_lossy_owned` 与 `FromUtf8Error::into_utf8_lossy` 直接产出 `String` |
| 其他 API | `VecDeque::retain_back`、`fs::set_times`、`fs::set_times_nofollow` |
| 语言与 lint | `UnsafeCell` 内容可以不经 `get` 访问；`#[attr] mod foo;` 可用；新增或加强若干 lint |
| Cargo | 新增 `debug` profile；CI 中默认关闭增量编译；2024 edition 成员可关闭继承的默认特性 |

下面从 C 变参函数讲起。

## 一、C 变参函数：可以用 Rust 来写了

### 1. 以前只能调用，现在可以定义

Rust 很早就能声明并调用外部的 C 变参函数，比如 `libc::printf`。但如果要反过来，用 Rust 实现一个让 C 调用的变参函数，在 1.98 及之前的稳定版中没有直接的写法，通常要用 C 写一层包装，或者改用 nightly 的 `c_variadic` 特性。

1.99 把这项特性带入了稳定版。写法是把最后一个参数声明成 `name: ...`，在函数体内通过它逐个读取参数：

```rust
use std::ffi::{CStr, VaList, c_char, c_int};

unsafe extern "C" {
    fn vsnprintf(buf: *mut c_char, size: usize, format: *const c_char, args: VaList) -> c_int;
}

/// # Safety
/// 调用者必须在 `count` 之后再传入至少 `count` 个 `c_int`。
#[unsafe(no_mangle)]
pub unsafe extern "C" fn sum_ints(count: c_int, mut args: ...) -> c_int {
    let mut total = 0;
    for _ in 0..count {
        total += unsafe { args.next_arg::<c_int>() };
    }
    total
}

/// # Safety
/// 可变参数必须与 `format` 中的格式说明符一一对应。
pub unsafe extern "C" fn log_c(format: *const c_char, args: ...) {
    let mut buf = [0 as c_char; 64];
    // `args` 的类型就是 VaList，可以直接交给 C 的 v* 系列函数。
    unsafe { vsnprintf(buf.as_mut_ptr(), buf.len(), format, args) };
    let line = unsafe { CStr::from_ptr(buf.as_ptr()) };
    println!("[log] {}", line.to_string_lossy());
}

fn main() {
    let total = unsafe { sum_ints(3, 10, 20, 12) };
    println!("sum_ints = {total}");

    unsafe { log_c(c"%s has %d items, avg %.2f".as_ptr(), c"cart".as_ptr(), 3, 7.5f64) };
}
```

Rust 1.99.0 的运行结果：

```text
sum_ints = 42
[log] cart has 3 items, avg 7.50
```

同一份源码交给 Rust 1.98.1，会报两类 `E0658`：`C-variadic functions are unstable`，以及 `use of unstable library feature c_variadic`。

`log_c` 演示了这项特性在实际项目中更常见的用途：Rust 函数接收一组可变参数，再原样转交给 C 库里以 `v` 开头的函数（`vprintf`、`vsnprintf`，以及不少 C 日志库提供的 `v` 系列接口）。[`VaList` 文档](https://doc.rust-lang.org/1.99.0/core/ffi/struct.VaList.html) 写明，它在布局和 ABI 上与平台的 `va_list` 完全对应，所以可以直接出现在外部函数的参数里。

### 2. C 也能直接调用

把 `sum_ints` 单独编译为静态库，再写一段 C 代码调用它：

```c
#include <stdio.h>

int sum_ints(int count, ...);

int main(void) {
    printf("from C: %d\n", sum_ints(4, 1, 2, 3, 36));
    return 0;
}
```

```text
$ rustc --edition 2024 --crate-type staticlib -O sum_lib.rs -o libsum.a
$ cc main.c libsum.a -o from_c && ./from_c
from C: 42
```

C 编译器看到的就是一个普通的 `int sum_ints(int, ...)`，不需要知道它由 Rust 实现。

### 3. 能读哪些类型？

`next_arg::<T>()` 要求 `T: VaArgSafe`。在常见的 32 位和 64 位目标上，这个 trait 的列表很短：`i32`、`u32`、`i64`、`u64`、`isize`、`usize`、`f64`，以及任意 `*const T` / `*mut T`。`c_int`、`c_long`、`c_double` 等 C 类型别名都落在这个范围内。部分平台上的标准库源码也为 `i128`、`u128` 实现了它，但这两个实现仍受未稳定的 `c_variadic_int128` 特性限制：本文在 1.99.0 上读取 `i128`，得到 `use of unstable library feature c_variadic_int128`。

`i8`、`i16`、`f32` 不在其中，原因来自 C 的默认参数提升规则：比 `int` 窄的整数在传入变参时会被提升为 `int` 或 `unsigned int`，`float` 会被提升为 `double`。（在 `int` 只有 16 位的 AVR、MSP430 上，`i16`、`u16` 本身就是 `int` 的宽度，因此在列表中。）[`VaArgSafe` 文档](https://doc.rust-lang.org/1.99.0/core/ffi/trait.VaArgSafe.html) 明确说，为这些会被提升的类型实现该 trait 是错误的。本文尝试读取 `i8` 和 `f32`，1.99.0 都在编译期拒绝：

```text
error[E0277]: the trait bound `i8: VaArgSafe` is not satisfied
  |     let _a = unsafe { args.next_arg::<i8>() };
  |                                       ^^ the nightly-only, unstable trait `VaArgSafe` is not implemented for `i8`
```

报错里的 “nightly-only, unstable trait” 还说明了一件事：`VaList` 和 `next_arg` 已经稳定，但 `VaArgSafe` 这个 trait 本身仍未稳定。本文实际编写 `fn read<T: VaArgSafe>(ap: &mut VaList<'_>) -> T`，1.99.0 报 `use of unstable library feature c_variadic_va_arg_safe`。在稳定版上，只能在调用 `next_arg` 时写出具体类型，不能编写对 `VaArgSafe` 泛型的辅助函数。

### 4. 使用限制

- 变参函数必须标为 `unsafe`。本文去掉 `unsafe` 后，编译器报 `functions with a C variable argument list must be unsafe`。原因很直接：参数个数和类型都由调用者决定，函数自身无法检查。
- [`next_arg` 的安全条件](https://doc.rust-lang.org/1.99.0/core/ffi/struct.VaList.html#method.next_arg) 包括：确实还有下一个参数；实际类型与读取类型兼容（同一类型、同宽度整数，或兼容的指针类型）；整数值在两种类型中都能表示。读多了、读错类型都是未定义行为，编译器不会为此报错。
- `VaList` 实现了 `Clone`（对应 C 的 `va_copy`）和 `Drop`（对应 `va_end`），但没有实现 `Copy`。标准库源码中的注释解释，这是为了给未来某些可能需要分配内存的目标平台留出余地。
- 公告列出的稳定范围是 `"C"` 和 `"C-unwind"` 两种 ABI。本文在 aarch64-apple-darwin 上把 ABI 换成 `"system"`、`"efiapi"`、`"sysv64"`、`"aapcs"`，1.99.0 都报 `` `...` is not supported ``。发布说明另外提到，非 `"C"` ABI 的变参函数可以用 `#[unsafe(naked)]` 函数加内联汇编定义，这属于更底层的用法，本文没有展开。

## 二、裸指针布局：`size_of_val_raw` 终于有了稳定的安全约定

`mem::size_of_val` 接收 `&T`，对切片和 trait 对象这类动态大小类型，它会读取引用里的元数据计算实际大小。问题在于，有些场景手里只有裸指针，而且这时还不能（或不应该）创建引用：内存尚未初始化，或者正在为一个动态大小类型分配空间。

1.99 稳定了三个接收 `*const T` 的版本：

- [`mem::size_of_val_raw`](https://doc.rust-lang.org/1.99.0/core/mem/fn.size_of_val_raw.html)
- [`mem::align_of_val_raw`](https://doc.rust-lang.org/1.99.0/core/mem/fn.align_of_val_raw.html)
- [`Layout::for_value_raw`](https://doc.rust-lang.org/1.99.0/core/alloc/struct.Layout.html#method.for_value_raw)

```rust
use std::alloc::Layout;
use std::fmt::Debug;
use std::mem::{align_of_val_raw, size_of_val_raw};
use std::ptr::{self, NonNull};

#[allow(dead_code)]
struct Packet {
    kind: u16,
    payload: [u32],
}

fn main() {
    // 一个从未分配过内存的切片指针：地址悬空，但长度元数据是 5。
    let dangling = NonNull::<u32>::dangling().as_ptr();
    let slice: *const [u32] = ptr::slice_from_raw_parts(dangling, 5);
    let size = unsafe { size_of_val_raw(slice) };
    let align = unsafe { align_of_val_raw(slice) };
    println!("[u32] x5: size={size}, align={align}");

    // 自定义 DST：把切片指针的元数据原样“借”给 Packet。
    let packet = slice as *const Packet;
    let layout = unsafe { Layout::for_value_raw(packet) };
    println!("Packet with 5 u32: {layout:?}");

    // trait 对象：大小和对齐从指针携带的 vtable 中读取。
    let value = 7u64;
    let object: *const dyn Debug = &value;
    println!("dyn Debug -> u64: size={}, align={}", unsafe { size_of_val_raw(object) }, unsafe {
        align_of_val_raw(object)
    });

    // 这些函数也是 const fn。
    const SIZE: usize = unsafe { size_of_val_raw(ptr::slice_from_raw_parts(ptr::null::<u16>(), 3)) };
    println!("const [u16] x3: {SIZE}");
}
```

Rust 1.99.0 的输出：

```text
[u32] x5: size=20, align=4
Packet with 5 u32: Layout { size: 24, align: 4 (1 << 2) }
dyn Debug -> u64: size=8, align=8
const [u16] x3: 6
```

`Packet` 的结果是 24 字节，可以这样理解：`kind` 占 2 字节，为满足 `u32` 对齐补 2 字节，再加 5 个 `u32` 共 20 字节。默认的 `repr(Rust)` 并不保证字段顺序和填充方式，这里是本文工具链上的实际结果；需要固定布局时应加 `#[repr(C)]`。不管采用哪种布局，`for_value_raw` 给出的都是编译器实际使用的大小和对齐，这正是为一个“头部 + 变长尾部”的结构分配内存前需要的数字，而计算过程中没有出现任何引用。

Rust 1.98.1 对同一份源码报 8 处 `use of unstable library feature layout_for_ptr`，以及一处 `size_of_val_raw is not yet stable as a const fn`。

这几个函数仍然是 `unsafe fn`。[`size_of_val_raw` 文档](https://doc.rust-lang.org/1.99.0/core/mem/fn.size_of_val_raw.html) 把条件分成两层：

- 如果指针可以安全地重新借用为 `&T`，调用总是安全的；`T: Sized` 时也总是安全的。
- 否则，对尾部为切片、`str` 或 trait 对象的类型，整个值（静态前缀加动态尾部）的大小必须能放进 `isize`。切片尾部长度为 0 时总是安全的。

这些条件不要求指针地址可以解引用，只要求元数据算出的大小合理。对 trait 对象来说，大小和对齐要从 vtable 中读取，所以指针携带的 vtable 本身必须有效。

## 三、`Box` 与 `Vec`：泄漏和收回各走各的路

### 1. `Box::leak` 的文档改了口径

发布公告专门用一节说明：1.99 没有改变语言语义，但更新了 [`Box::leak` 的文档](https://doc.rust-lang.org/1.99.0/std/boxed/struct.Box.html#method.leak)，不再建议先 `leak` 拿到 `&'static mut T`，之后又用 `Box::from_raw` 把它收回释放。

新文档的说法是：`leak` 适合真正要活到程序结束的数据；如果内存最终要释放，应该改用 `Box::into_raw` 或 `Box::into_non_null`。从 `leak` 返回的可变引用重新组装 `Box`，只有在分配器为 `Global` 时才可能做到，而且属于“灰色地带”，很多看起来无害的写法都是未定义行为。公告给出的原因是，这类代码与现有以及未来可能出现的编译器优化存在冲突，自定义分配器稳定之后问题会更明显。这条建议同样适用于标准库里其他名为 `leak` 的函数。

### 2. 用 `NonNull` 交出和收回所有权

同一版本还稳定了四个函数，`Box::leak` 新文档推荐的 `into_non_null` 就在其中：

```rust
use std::ptr::NonNull;

fn main() {
    // Box 与 NonNull 互转：所有权交给调用者，之后再完整收回。
    let boxed = Box::new(String::from("config"));
    let ptr: NonNull<String> = Box::into_non_null(boxed);
    unsafe { (*ptr.as_ptr()).push_str(".toml") };
    let boxed = unsafe { Box::from_non_null(ptr) };
    println!("box: {boxed}");

    // Vec 拆成 (指针, 长度, 容量)，再按同样的三元组重建。
    let mut v = Vec::with_capacity(8);
    v.extend([1u32, 2, 3]);
    let (ptr, len, cap) = v.into_parts();
    println!("parts: len={len}, cap={cap}");
    let rebuilt = unsafe { Vec::from_parts(ptr, len, cap) };
    println!("rebuilt: {rebuilt:?}");
}
```

```text
box: config.toml
parts: len=3, cap=8
rebuilt: [1, 2, 3]
```

它们与已有的 `Box::into_raw` / `from_raw`、`Vec::into_raw_parts` 是同一套约定，区别是指针类型为 `NonNull<T>`。很多数据结构内部本来就用 `NonNull` 保存指针，以前要在 `*mut T` 和 `NonNull` 之间来回转换，现在可以直接对接。`Box::into_non_null` 和 `into_raw` 一样是关联函数，要写成 `Box::into_non_null(b)`；`Vec::into_parts` 标了 `#[must_use]`，丢掉返回值会泄漏内存。

## 四、`Box<[T; N]>` 可以按值迭代

1.99 为 `Box<[T; N]>`、`&Box<[T; N]>` 和 `&mut Box<[T; N]>` 实现了 `IntoIterator`：

```rust
fn main() {
    let names: Box<[String; 3]> = Box::new(["a".into(), "b".into(), "c".into()]);

    for name in &names {
        print!("{name} ");
    }
    println!();

    // 按值迭代：拿到的是 String，不是 &String。
    let owned: Vec<String> = names.into_iter().map(|s| s + "!").collect();
    println!("{owned:?}");
}
```

```text
a b c 
["a!", "b!", "c!"]
```

Rust 1.98.1 编译这段代码时，`for name in &names` 报 `E0277`：`&Box<[String; 3]> is not an iterator`。`for name in names` 同样不能编译。

不过本文对照时发现，`names.into_iter()` 这种**方法调用**在 1.98.1 上可以编译，结果也是按值产出 `String`。区别在返回类型：

| 工具链 | `names.into_iter()` 的类型 |
|--------|----------------------------|
| 1.98.1 | `core::array::iter::IntoIter<String, 3>` |
| 1.99.0 | `alloc::boxed::iter::BoxedArrayIntoIter<String, 3>` |

从返回类型看，1.98.1 的方法解析是通过解引用把整个数组从 `Box` 中移出来，再调用 `[T; N]` 的 `into_iter`。1.99 有了 `Box` 自己的实现，方法解析优先选中它。按标准库源码，新实现先把 `Box<[T; N]>` 转成 `Box<[T]>`，再转成 `Vec` 迭代，不会把数组复制到栈上。

这个变化会让少数代码无法编译：如果代码显式写出了旧类型，例如 `let it: std::array::IntoIter<String, 3> = names.into_iter();`，1.98.1 可以通过，1.99.0 报 `E0308`。新类型可以通过 `std::boxed::BoxedArrayIntoIter` 引用。发布说明的兼容性章节没有列出这一点，属于本文的实测结果。

## 五、其他新 API

### 1. 有损 UTF-8 解码直接得到 `String`

本节前三个小节的代码片段摘自同一个 `main`（返回类型为 `std::io::Result<()>`），这里只省略了 `use` 语句和函数框架。

`String::from_utf8_lossy` 接收 `&[u8]`，返回 `Cow<str>`。如果手里本来就是 `Vec<u8>`，还想要一个 `String`，以前要多绕一步。1.99 稳定了两个拥有型版本：

```rust
// 1. 有损 UTF-8：直接消费 Vec<u8>
let valid = b"hello".to_vec();
let addr = valid.as_ptr();
let s = String::from_utf8_lossy_owned(valid);
println!("valid: {s:?}, same buffer: {}", s.as_ptr() == addr);

let broken = b"temp \xF0\x90\x80C".to_vec();
println!("broken: {:?}", String::from_utf8_lossy_owned(broken));

// 先严格校验，失败后再降级，不必重新检查已经通过的前缀
let text = match String::from_utf8(b"id=\xFF42".to_vec()) {
    Ok(text) => text,
    Err(err) => {
        println!("valid_up_to = {}", err.utf8_error().valid_up_to());
        err.into_utf8_lossy()
    }
};
println!("recovered: {text:?}");
```

```text
valid: "hello", same buffer: true
broken: "temp �C"
valid_up_to = 3
recovered: "id=�42"
```

输出中的 `�` 是 Unicode 替换字符 U+FFFD，不是乱码。`\xF0\x90\x80` 是一个缺了最后一个字节的 4 字节序列，整体只被替换成一个 `�`；`\xFF` 在 UTF-8 中永远不合法，同样被替换成一个 `�`。

本文观察到，输入本身合法时，返回的 `String` 复用了原来的缓冲区。但[文档](https://doc.rust-lang.org/1.99.0/std/string/struct.String.html#method.from_utf8_lossy_owned)明确写着“不保证复用原 `Vec` 的分配”，代码不应依赖这一点。`into_utf8_lossy` 的文档则说明，它不需要重新检查已经通过校验的前缀，适合“先严格解析，失败后再容错”的流程。

### 2. `VecDeque::retain_back`

```rust
// 2. VecDeque::retain_back：只保留末尾 N 个
let mut recent: VecDeque<u32> = (1..=6).collect();
recent.retain_back(3);
println!("retain_back(3): {recent:?}");
recent.retain_back(10);
println!("retain_back(10): {recent:?}");
```

```text
retain_back(3): [4, 5, 6]
retain_back(10): [4, 5, 6]
```

它与 `truncate` 方向相反：`truncate(n)` 保留前 n 个，`retain_back(n)` 保留后 n 个，丢弃并析构前面的元素。`n` 大于等于长度时什么也不做。维护“最近 N 条记录”时，以前要循环 `pop_front` 或用 `drain(..len - n)`，现在一行即可。它带有 `#[doc(alias = "truncate_front")]`，在文档里搜这个名字也能找到。

### 3. 按路径设置文件时间戳

此前用标准库修改文件时间，需要先打开文件，再调用 `File::set_times`。1.99 增加了按路径操作的 `fs::set_times` 和 `fs::set_times_nofollow`：

```rust
// 3. fs::set_times：按路径设置时间戳，不必先打开文件
let path = "times-demo.txt";
fs::write(path, "x")?;
let past = SystemTime::UNIX_EPOCH + Duration::from_secs(1_700_000_000);
fs::set_times(path, FileTimes::new().set_modified(past).set_accessed(past))?;
let modified = fs::metadata(path)?.modified()?;
println!("mtime = {:?}", modified.duration_since(SystemTime::UNIX_EPOCH).unwrap());
```

```text
mtime = 1700000000s
```

两者的区别在符号链接：按文档，`set_times` 会跟随链接，修改目标文件；`set_times_nofollow` 修改链接本身。本文在 macOS 上创建了一个指向 `target.txt` 的符号链接，用 `set_times_nofollow` 把链接的修改时间设为 1600000000，结果链接本身的 `symlink_metadata` 读到 1600000000，目标文件的修改时间没有被改成这个值。按[文档](https://doc.rust-lang.org/1.99.0/std/fs/fn.set_times_nofollow.html)，它在 Unix 上对应带 `AT_SYMLINK_NOFOLLOW` 的 `utimensat`，在 Apple 平台上对应带 `FSOPT_NOFOLLOW` 的 `setattrlist`，在 Windows 上对应 `SetFileTime`。

### 4. 补充上一篇：`NumBuffer` 现在可以从 `std` 导入

上一篇提到，1.98 稳定的 `NumBuffer` 只能写成 `core::fmt::NumBuffer`，`std::fmt::NumBuffer` 会报错。1.99 在 `alloc` 和 `std` 中重新导出了它：

```rust
use std::fmt::NumBuffer;

fn main() {
    let mut buf = NumBuffer::new();
    println!("{}", 1999_u32.format_into(&mut buf));
}
```

这段代码在 1.98.1 上报 `E0432: unresolved import std::fmt::NumBuffer`，在 1.99.0 上输出 `1999`。

此外，`StepBy<I>` 在 `I: FusedIterator` 时也实现了 `FusedIterator`；`mem::transmute_copy` 的源类型参数放宽为 `Src: ?Sized`，可以直接从 `&[u8]` 这样的切片复制出一个值。`transmute_copy` 的尺寸检查也变了，这一点放在第八节说明。

## 六、语言与编译器：unsafe 规则说清楚了，诊断更准了

### 1. `UnsafeCell` 的内容可以不经 `get` 访问

这是一项规则层面的变化。1.98 及之前的 [`UnsafeCell` 文档](https://doc.rust-lang.org/1.98.0/core/cell/struct.UnsafeCell.html)写着，从共享的 `UnsafeCell<T>` 取得 `*mut T` 的唯一合法方式是 `.get()` 或 `.raw_get()`，直接把 `&UnsafeCell<T>` 转成 `*mut T` 是未定义行为。1.99 把这条改成了保证：既然 `UnsafeCell<T>` 与 `T` 布局相同，直接做指针转换也可以访问内部数据，`&mut` 的唯一性要求保持不变。

`invalid_reference_casting` lint 也随之调整。下面的函数仿照新版 [`UnsafeCell` 文档](https://doc.rust-lang.org/1.99.0/core/cell/struct.UnsafeCell.html)中的示例：

```rust
use std::cell::UnsafeCell;

/// # Safety
/// 在返回的引用存活期间，调用者不能再通过 `cell` 或它的任何别名访问其中的数据。
unsafe fn get_mut_unchecked<T>(cell: &UnsafeCell<T>) -> &mut T {
    let ptr = cell as *const UnsafeCell<T> as *mut T;
    unsafe { &mut *ptr }
}

fn main() {
    let cell = UnsafeCell::new(1);
    let value = unsafe { get_mut_unchecked(&cell) };
    *value += 41;
    println!("{}", cell.into_inner());
}
```

Rust 1.98.1 直接拒绝编译，报 `casting &T to &mut T is undefined behavior`（`invalid_reference_casting` 默认是 deny）。Rust 1.99.0 编译通过，输出 `42`。

这项保证放宽的是“怎样拿到指针”，没有放宽别名规则：同一时刻仍然不能存在两个指向同一数据的 `&mut`，也不能在 `&mut` 存活期间通过其他指针访问数据。

### 2. 新的 `raw_borrows_via_references` lint

`&mut x as *mut T` 会先创建一个 `&mut` 引用，再转成裸指针。这个中间引用同样要满足别名规则，而 `&raw mut x` 不会创建引用。1.99 新增了一个默认允许的 lint，用来找出这类写法：

```rust
#![warn(raw_borrows_via_references)]

fn main() {
    let mut counter = 0u32;
    let ptr = &mut counter as *mut u32;
    unsafe { *ptr += 1 };
    println!("{counter}");
}
```

```text
warning: creating an intermediate reference implies aliasing requirements even when immediately cast to a raw pointers
 --> raw_borrow.rs:5:15
  |
5 |     let ptr = &mut counter as *mut u32;
  |               ^^^^^^^^^^^^^^^^^^^^^^^^
...
help: consider using `&raw mut` for a safer and more explicit raw pointer
  |
5 -     let ptr = &mut counter as *mut u32;
5 +     let ptr = &raw mut counter;
```

它默认是 `allow`，需要手动开启。对含有较多 unsafe 代码的项目，可以在 crate 根部加上 `#![warn(raw_borrows_via_references)]` 跑一遍。

### 3. `#[my_macro] mod foo;` 可以用了

以前，只要过程宏的输入中出现引用外部文件的 `mod foo;`，稳定版就会报 `E0658: file modules in proc macro input are unstable`。这既包括 `#[my_macro] mod foo;`，也包括属性宏或 derive 宏作用的条目内部嵌套的 `mod foo;`。

本文写了一个原样返回输入、并在编译期打印输入源码的属性宏，分别作用于两种写法：

```rust
#[macros::inspect]
mod handlers;

#[macros::inspect]
mod api {
    pub mod routes;
}
```

1.98.1 对两处都报上面的 `E0658`。1.99.0 正常编译运行，宏打印的输入是：

```text
inspect: received `mod handlers;`
inspect: received `mod api { pub mod routes; }`
```

可以看到，宏收到的只有模块声明本身，不包含 `handlers.rs` 或 `api/routes.rs` 的内容；程序运行时这两个文件中的函数都能正常调用，说明它们是在宏展开之后才按常规规则加载的。这项稳定解除了语法位置上的限制，但外部文件里的代码不会出现在宏的输入中。

### 4. 方法名建议优先使用 `doc(alias)`

调用不存在的方法时，rustc 会建议一个名字相近的方法。1.99 起，如果某个方法通过 `#[doc(alias = "...")]` 声明了与调用名完全相同的别名，编译器会优先推荐它，而不是按拼写相似度挑选。

```rust
pub struct Queue {
    items: Vec<u32>,
}

impl Queue {
    #[doc(alias = "enqueue")]
    pub fn push_back(&mut self, item: u32) {
        self.items.push(item);
    }

    pub fn dequeue(&mut self) -> Option<u32> {
        self.items.pop()
    }
}

fn main() {
    let mut q = Queue { items: Vec::new() };
    q.enqueue(1);
    println!("{:?}", q.dequeue());
}
```

对 `q.enqueue(1)`，1.98.1 给出的提示是 `there is a method dequeue with a similar name, but with different arguments`；1.99.0 给出 `there is a method push_back with a similar name`，并附上把 `enqueue` 改成 `push_back` 的修改建议。库作者如果知道用户常用另一个名字找某个方法，加一个 `doc(alias)` 就能同时改善 rustdoc 搜索和编译器提示。

### 5. 其他编译器变化

- `unconditional_panic` 现在能发现常量泛型参数为 0 的 `as_chunks`、`as_rchunks`、`array_windows` 等调用。这个 lint 默认是 deny，见第八节。
- `unreachable_cfg_select_predicates` 被加入 `unused` lint 组。
- `invalid_runtime_symbol_definitions` 和 `suspicious_runtime_symbol_definitions` 覆盖了更多 POSIX 符号。这组检查在 1.98 中引入，上一篇有介绍。
- 在 x86 上，`asm!` 可以通过向量寄存器传递 128 位整数。
- 所有 gnu 和 musl 目标启用了 `static_position_independent_executables`。
- `riscv64-unknown-linux-musl` 升级为带主机工具的 Tier 2 平台。
- rustdoc 改进了 trait 实现的过滤方式。发布说明给出的数据是平均提速 20%，部分真实 crate 最多提速 40%。本文没有复测这组数据。
- 编译器后端升级到 LLVM 23。

## 七、Cargo：新的 `debug` profile 与 CI 默认行为

Cargo 1.99 有三项会影响日常使用的变化。本机的 `~/.cargo/config.toml` 设置了 `incremental = false`，而 Cargo 会读取当前目录各级父目录中的配置，所以 `debug` profile 和增量编译的测试都放在用户主目录之外的临时目录中，并使用单独的 `CARGO_HOME`。

### 1. 内置的 `debug` profile

```text
$ cargo +1.98.1 build --profile debug
error: profile name `debug` is reserved
To configure the default development profile, use the name `dev` as in [profile.dev]

$ cargo +1.99.0 build --profile debug
    Finished `debug` profile [unoptimized + debuginfo] target(s) in 0.22s
```

[Cargo 文档](https://doc.rust-lang.org/nightly/cargo/reference/profiles.html#debug-1)对它的定义只有一行 `inherits = "dev"`，用于“构建要配合调试器运行的二进制”，`cargo install --debug` 也会使用它。[Cargo 更新日志](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-199-2026-10-01)说明了动机：这是为将来调整 `dev` profile 做准备，让 `dev` 不必再兼顾调试，可以换成更适合快速迭代的默认值。目前两者没有区别。

在 1.99 中，`[profile.debug]` 也可以自定义了。本文写入 `opt-level = 1` 后，构建结果显示为 `debug profile [optimized + debuginfo]`；1.98.1 对同一份清单报错，指出 `debug` 是保留名称。本文观察到，`debug` profile 的产物和 `dev` 一样放在 `target/debug` 下。

`cargo build` 仍然默认使用 `dev`。如果团队有专门配合调试器的流程，可以开始改用 `--profile debug`，并把这类设置放进 `[profile.debug]`。这样以后 `dev` 的默认值变化时，调试流程不会受到影响。

### 2. CI 中默认关闭增量编译

Cargo 1.99 会检查 `CI` 环境变量。检测到 CI 时，默认不启用增量编译。[Cargo 配置文档](https://doc.rust-lang.org/nightly/cargo/reference/config.html#buildincremental)对 `build.incremental` 的说明是：设置了 `CI` 的环境中，如果没有显式配置，默认值相当于 `CARGO_BUILD_INCREMENTAL=false`；否则沿用 profile 中的值。本文用 `cargo build -v` 检查 rustc 命令行里是否出现 `-C incremental=`：

| 环境 | 1.98.1 | 1.99.0 |
|------|--------|--------|
| 未设置 `CI` | 启用 | 启用 |
| `CI=true` | 启用 | 关闭 |
| `CI=true` 且 `CARGO_INCREMENTAL=1` | 启用 | 启用 |

本文另外得到三个文档没有写明的观察结果：

- `CI=false`、`CI=0` 和 `CI=`（空值）都会让 1.99.0 关闭增量编译。看起来 Cargo 只判断变量是否存在，不看它的值。
- 只设置 `TF_BUILD=True`（Azure Pipelines 使用的变量）也会关闭增量编译；只设置 `GITHUB_ACTIONS=true` 不会。不过 GitHub Actions 本身会设置 `CI=true`。
- `--release` 构建本来就默认不启用增量编译，这一点没有变化。

在 CI 中，构建目录通常不会保留到下一次运行，增量编译的缓存派不上用场，反而要多写一份磁盘数据。如果你的 CI 确实缓存了 `target` 目录并依赖增量编译，需要注意重新开启的方式：

| 在 `CI=true` 下的设置 | 1.99.0 是否启用增量编译 |
|------------------------|------------------------|
| `CARGO_INCREMENTAL=1` | 启用 |
| `CARGO_BUILD_INCREMENTAL=true` 或 `--config build.incremental=true` | 启用 |
| `[profile.dev]` 中写 `incremental = true`（或 `CARGO_PROFILE_DEV_INCREMENTAL=true`） | **不启用** |

最后一行容易踩坑：profile 里的 `incremental` 优先级低于 `build.incremental`，而 CI 环境下的新默认值正是作用在 `build.incremental` 上，所以只改 profile 不起作用。

### 3. 2024 edition 成员可以覆盖继承的 `default-features`

工作区经常这样声明共享依赖：

```toml
# 工作区根目录 Cargo.toml
[workspace.dependencies]
featured = { path = "featured" }

# 成员 app/Cargo.toml（edition = "2024"）
[dependencies]
featured = { workspace = true, default-features = false }
```

本文测试用的 `featured` 默认启用 `fancy` 特性，三种情况的结果如下：

- Cargo 1.98.1 直接报错：`default-features = false cannot override workspace's default-features`。
- Cargo 1.99.0，成员使用 2024 edition：构建通过，`featured` 的默认特性被关闭，程序输出 `featured mode = plain`。
- Cargo 1.99.0，成员使用 2021 edition：构建通过，但 Cargo 给出警告，并忽略 `default-features = false`，程序输出 `featured mode = fancy`。

```text
warning: app/Cargo.toml: `default-features` is ignored for featured, since `default-features` was not specified for `workspace.dependencies.featured`; overriding workspace `default-features` to false requires Rust 1.99+ and the 2024 edition
```

这里要注意 Cargo 的特性合并规则：特性是在整个依赖图上取并集的。如果工作区里另一个成员启用了 `featured` 的默认特性，在同一次构建中，`app` 这一侧写 `default-features = false` 也关不掉。这项改动来自 [RFC 3945](https://github.com/rust-lang/rfcs/pull/3945)。

另外，Cargo 1.99 会对 `[lints]` 表中带连字符的 lint 名给出警告。本文在 `[lints.rust]` 中写 `unexpected-cfgs = "warn"`，1.98.1 没有提示，1.99.0 给出 `` `lints.rust.unexpected-cfgs` is deprecated in favor of `lints.rust.unexpected_cfgs` and will not work in a future edition ``。

## 八、升级前需要确认的兼容性变化

### 1. 可能导致编译失败的变化

| 变化 | 本文实测 |
|------|----------|
| `no_mangle_generic_items` 升级为硬错误 | 对泛型函数加 `#[unsafe(no_mangle)]`，1.98.1 是警告，1.99.0 报错；加 `#[allow(no_mangle_generic_items)]` 也无效，只会多一条 lint 已被移除的警告 |
| `unconditional_panic` 覆盖常量泛型为 0 的调用 | `data.as_chunks::<0>()` 在 1.98.1 能编译，运行时 panic，信息为 `chunk size must be non-zero`；1.99.0 编译期报 `this operation will panic at runtime` |
| `Box<[T; N]>::into_iter()` 返回类型改变 | 显式标注为 `std::array::IntoIter<T, N>` 的代码在 1.99.0 报 `E0308`（见第四节） |

发布说明还列出了其他几项：不再对指向 `extern static` 的引用做常量提升；`let` 模式推断出的类型现在一定会参与类型检查；常量求值遇到值的类型中含有泛型参数时会中止；文档注释的代码块中，没有作用对象的属性会报错；稳定版过去在宏调用上意外接受了 `#[repr(simd)]`，这个问题已修复；`Pin::new_unchecked` 的安全约定也有细微调整；关联常量类型中省略的生命周期现在推断为 `'static`；此外还有 PowerPC 内联汇编、Hermit 文件系统和 doctest 过滤等平台或工具层面的修正。这些变化本文没有逐一复现。

### 2. 新的警告

- **旧的整数模块被完全弃用**：`std::i32::MAX`、`std::u8::MIN` 这类写法在 1.99.0 中会给出 `deprecated` 警告，应改为 `i32::MAX`、`u8::MIN`。1.98.1 对同样的代码没有警告。
- **跨 crate 宏的尾随分号**：如果宏展开为 `42;`，又在表达式位置使用（`let x = mac::answer!();`），以前只有同一个 crate 定义的宏会触发警告。1.99 拆出了 `semicolon_in_expressions_from_non_local_macros`，对其他 crate 的宏也会警告，它属于 `future_incompatible` 组。本文用两个 crate 测试，1.98.1 没有警告，1.99.0 有。发布说明特别提醒：遇到这条警告时，应该向提供该宏的 crate 报告，而不是在自己的代码里屏蔽。

### 3. `transmute_copy` 的尺寸检查只在调试构建中生效

关于尺寸检查，发布说明只写了 `transmute_copy` 改用“不会展开的 panic”。本文对照后发现，变化不止于此。下面的代码让目标类型比源类型大，这本身就是错误用法：

```rust
fn main() {
    let small = 1u16;
    let big: u64 = unsafe { std::mem::transmute_copy(&small) };
    println!("{big}");
}
```

| 工具链与构建方式 | 结果 |
|------------------|------|
| 1.98.1，调试构建或 `-O` | 都会 panic：`cannot transmute_copy if Dst is larger than Src` |
| 1.99.0，调试构建 | panic：`unsafe precondition(s) violated: ...`，进程以退出码 134 终止 |
| 1.99.0，`-O` | 不做检查，程序打印出一个无意义的数（这是未定义行为，结果不可预测） |

按 1.99.0 源码，这项检查改成了 `assert_unsafe_precondition!`，只在启用调试断言时生效，失败时直接终止进程。本文把调用放进 `catch_unwind`：1.98.1 能捕获这个 panic；1.99.0 无法捕获，进程直接以退出码 134 结束，报错信息还写明 `This Undefined Behavior check is optional, and cannot be relied on for safety`。

这并不违背已有的承诺：[1.98.0 的 `transmute_copy` 文档](https://doc.rust-lang.org/1.98.0/core/mem/fn.transmute_copy.html)已经写明，`Dst` 比 `Src` 大时会触发未定义行为，panic 只是实现上的附带检查。但如果代码曾经依赖这个 panic 在发布版中兜底，升级后它会变成静默的未定义行为。

### 4. 已经耗尽的 `RangeInclusive`

为了让 `a..=b` 的迭代在一些场景下优化得更好，标准库修改了 `RangeInclusive` 的迭代实现。一个后果是，迭代耗尽后，它的 `start()`、`end()` 返回值可能不同，用作切片下标时的行为也可能不同。发布说明认为这些行为原本就没有稳定性保证，因此不算破坏性变更。本文的对照结果是：

```rust
let mut r = 1..=3;
for _ in r.by_ref() {}
println!("{r:?} start={} end={} is_empty={}", r.start(), r.end(), r.is_empty());
```

```text
1.98.1: 3..=3 (exhausted) start=3 end=3 is_empty=true
1.99.0: 4..=3 start=4 end=3 is_empty=true
```

`is_empty()` 的结果一致。如果代码在迭代之后还读取 `start()` 或 `end()`，需要检查一下。

### 5. 补充上一篇：1.98.0 的误编译已在 1.98.1 修复

[上一篇](/posts/rust-1-98-0-deep-dive/)提到，Rust 1.98.0 存在一项尚未修复的 vtable 误编译问题（[#161441](https://github.com/rust-lang/rust/issues/161441)）。根据 RELEASES.md，Rust 1.98.1 已于 2026 年 9 月 3 日发布，唯一的修复就是这一项。仍停留在 1.98.0 的项目，至少应升级到 1.98.1。

## 九、怎样验证自己的项目？

```bash
rustup update stable
rustc -vV          # 确认 release 为 1.99.0
cargo build --all-targets
cargo test
```

重点检查以下几项：

1. 搜索 `std::i32::`、`std::u64::` 等旧模块路径，以及依赖中触发 `semicolon_in_expressions_from_non_local_macros` 的宏。
2. 搜索泛型函数上的 `no_mangle`，以及显式写出 `array::IntoIter` 类型、又对 `Box<[T; N]>` 调用 `into_iter()` 的代码。
3. 如果 CI 依赖增量编译缓存，检查 `CI` 环境变量下的构建时间是否变化；需要保留增量编译时，用 `CARGO_INCREMENTAL=1` 或 `build.incremental`，不要只改 profile。
4. 搜索 `transmute_copy`，确认没有代码依赖它在尺寸不匹配时 panic。
5. 对含有 unsafe 代码的 crate，可以临时开启 `#![warn(raw_borrows_via_references)]` 检查一遍。
6. 使用 `Box::leak` 后又用 `Box::from_raw` 收回的代码，改为 `Box::into_raw` 或 `Box::into_non_null`。

## 本文做过哪些验证？

| 验证项 | 方法 | 结果 |
|--------|------|------|
| C 变参函数 | 1.99.0 编译运行；静态库由 C 调用；同一源码交给 1.98.1 | 1.99.0 输出正确，C 端得到 42；1.98.1 报 `E0658` |
| `VaArgSafe` 边界 | 读取 `i8` / `f32` / `i128`；对 `VaArgSafe` 写泛型约束；去掉 `unsafe` | 前两者报 `E0277`，`i128` 和泛型约束报未稳定特性，去掉 `unsafe` 报错 |
| 裸指针布局 | 悬空切片指针、自定义 DST、trait 对象、`const` 求值 | 结果为 20/4、24/4、8/8、6；1.98.1 报未稳定 |
| `Box`/`Vec` 与 `NonNull` | 转出、修改、重建 | 内容与容量保持一致 |
| `Box<[T; N]>` 迭代 | `for` 循环、`into_iter()` 返回类型、显式类型标注 | 1.98.1 的 `for` 失败；返回类型改变；显式标注旧类型在 1.99.0 报 `E0308` |
| 有损 UTF-8、`retain_back` | 合法与非法输入；`n` 小于和大于长度 | 输出符合文档 |
| `std::fmt::NumBuffer` | 同一源码交给 1.98.1 与 1.99.0 | 1.98.1 报 `E0432`，1.99.0 输出 1999 |
| `set_times` / `set_times_nofollow` | 在 macOS 上修改普通文件和符号链接的时间 | 前者修改目标文件，后者只修改链接 |
| `UnsafeCell` 指针转换 | 同一源码交给 1.98.1 与 1.99.0 | 1.98.1 被 `invalid_reference_casting` 拒绝，1.99.0 输出 42 |
| 新 lint 与诊断 | `raw_borrows_via_references`、`doc(alias)` 建议、`as_chunks::<0>` | 与正文描述一致 |
| `#[attr] mod foo;` | 本地属性宏作用于 `mod foo;` 和含嵌套 `mod` 的内联模块 | 1.98.1 报 `E0658`；1.99.0 通过，宏只收到模块声明 |
| Cargo | `debug` profile、`CI`/`TF_BUILD` 变量与增量编译、重新开启增量编译的几种方式、`default-features` 覆盖（2021/2024 edition） | 见第七节表格与输出 |
| `transmute_copy` | 尺寸不匹配时分别做调试构建、`-O` 构建和 `catch_unwind`；从 `&[u8]` 复制 | 见第八节 |
| 兼容性 | `no_mangle` 泛型、旧整数模块、跨 crate 宏分号、`RangeInclusive` 耗尽状态 | 见第八节 |
| 其他变参 ABI | 在 aarch64-apple-darwin 上尝试 `system`、`efiapi`、`sysv64`、`aapcs` | 1.99.0 均不支持 |

## 写在最后

C 变参函数的稳定，补上了 Rust 与 C 互操作中长期缺少的一环：过去稳定版只能在 Rust 中调用变参函数，现在也可以定义。但它的安全边界和 C 一样：参数个数和类型只能靠调用双方的约定保证，`next_arg` 读错了，编译器不会报错，运行时也不保证有任何提示。`size_of_val_raw` 这几个函数也是如此，它们把“什么时候可以在裸指针上算布局”写成了明确的条件，调用者仍要自己满足这些条件。

`Box::leak` 文档的修改，以及 `UnsafeCell` 保证的放宽，都属于规则层面的澄清。代码可能一行都不用改，但原来处在灰色地带的写法，现在有了明确的结论：前者不建议再用，后者有了正式保证，但别名规则依然适用。Cargo 的三项变化影响面更广，尤其是 CI 中的增量编译默认值，升级后值得看一眼构建时间。

## 参考资料

- [Rust 1.99.0 官方发布公告](https://blog.rust-lang.org/2026/10/01/Rust-1.99.0/)
- [Rust 1.99.0 详细发布说明](https://doc.rust-lang.org/stable/releases.html#version-1990-2026-10-01)
- [Cargo 1.99 更新日志](https://doc.rust-lang.org/nightly/cargo/CHANGELOG.html#cargo-199-2026-10-01)
- [Rust Reference：C 变参函数](https://doc.rust-lang.org/reference/items/functions.html#c-variadic-functions)
- [`core::ffi::VaList` 标准库文档](https://doc.rust-lang.org/1.99.0/core/ffi/struct.VaList.html)
- [`core::ffi::VaArgSafe` 标准库文档](https://doc.rust-lang.org/1.99.0/core/ffi/trait.VaArgSafe.html)
- [`mem::size_of_val_raw` 标准库文档](https://doc.rust-lang.org/1.99.0/core/mem/fn.size_of_val_raw.html)
- [`Layout::for_value_raw` 标准库文档](https://doc.rust-lang.org/1.99.0/core/alloc/struct.Layout.html#method.for_value_raw)
- [`Box::leak` 标准库文档](https://doc.rust-lang.org/1.99.0/std/boxed/struct.Box.html#method.leak)
- [`Vec::into_parts` 标准库文档](https://doc.rust-lang.org/1.99.0/std/vec/struct.Vec.html#method.into_parts)
- [`String::from_utf8_lossy_owned` 标准库文档](https://doc.rust-lang.org/1.99.0/std/string/struct.String.html#method.from_utf8_lossy_owned)
- [`fs::set_times_nofollow` 标准库文档](https://doc.rust-lang.org/1.99.0/std/fs/fn.set_times_nofollow.html)
- [`mem::transmute_copy` 1.98.0 文档](https://doc.rust-lang.org/1.98.0/core/mem/fn.transmute_copy.html)
- [`UnsafeCell` 标准库文档](https://doc.rust-lang.org/1.99.0/core/cell/struct.UnsafeCell.html)
- [`UnsafeCell` 保证变更 PR #159730](https://github.com/rust-lang/rust/pull/159730)
- [Cargo profile 文档：`debug`](https://doc.rust-lang.org/nightly/cargo/reference/profiles.html#debug-1)
- [Cargo 配置文档：`build.incremental`](https://doc.rust-lang.org/nightly/cargo/reference/config.html#buildincremental)
- [RFC 3945：覆盖继承的 `default-features`](https://github.com/rust-lang/rfcs/pull/3945)
- [Rust 1.98.0 误编译问题 #161441](https://github.com/rust-lang/rust/issues/161441)
