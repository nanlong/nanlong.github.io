---
title: "数据格式演进史（三）：零拷贝，当序列化本身也嫌慢"
date: 2025-12-07T12:00:00+08:00
draft: false
tags: ["数据格式", "FlatBuffers", "Cap'n Proto", "零拷贝", "游戏开发", "高频交易"]
categories: ["技术"]
description: "游戏引擎每帧只有 16 毫秒，高频交易系统每微秒都是金钱。在这些场景下，Protobuf 的序列化开销也成了不可接受的奢侈。"
series: ["数据格式演进史"]
---

游戏引擎每帧只有 16 毫秒，高频交易系统每微秒都是金钱。在这些场景下，Protobuf 的序列化开销也成了不可接受的奢侈。

<!--more-->

## 前情回顾

在上一篇中，我们看到了二进制格式如何打败 JSON：

- **Protobuf**：体积小 40-60%，速度快 3-5 倍
- **Thrift**：完整的 RPC 框架
- **gRPC**：HTTP/2 + Protobuf，现代微服务标配

但是，即使是 Protobuf，在某些极端场景下也显得「太慢」了。

## 16 毫秒的预算

假设你在开发一个 60 FPS 的游戏。

每一帧的时间预算：**1000ms ÷ 60 = 16.67ms**

这 16 毫秒要做什么？

| 任务 | 时间预算 |
|------|---------|
| 物理模拟 | 3-4 ms |
| AI 计算 | 2-3 ms |
| 渲染准备 | 4-5 ms |
| 网络通信 | 1-2 ms |
| 音频处理 | 1 ms |
| 其他 | 剩余 |

网络通信只有 1-2 毫秒的预算。

如果你的游戏需要同步 100 个玩家的状态，每个玩家有位置、速度、动画状态等数据，Protobuf 的序列化/反序列化可能就要吃掉这 1-2 毫秒。

**有没有办法让序列化时间接近于零？**

## 序列化到底在做什么？

让我们拆解一下 Protobuf 序列化的过程：

```
原始数据（内存中的结构体）
    ↓
[序列化] 遍历字段，编码成字节流
    ↓
字节流（可以传输/存储）
    ↓
[反序列化] 解析字节流，构建对象
    ↓
目标数据（内存中的结构体）
```

**每一步都需要 CPU 时间和内存分配。**

一个简化的对比：

| 操作 | Protobuf | 理想情况 |
|------|----------|---------|
| 序列化 | 遍历 + 编码 | 直接发送内存 |
| 反序列化 | 解析 + 构建对象 | 直接读取内存 |
| 内存分配 | 每次都要分配 | 零分配 |

**如果能直接把内存里的数据发出去，跳过序列化，该多好？**

## 零拷贝的核心思想

**传统方式**：

```
用户数据 → [序列化] → 字节缓冲区 → [网络发送] → 字节缓冲区 → [反序列化] → 用户数据
           复制1次              复制2次              复制3次
```

**零拷贝方式**：

```
用户数据（本身就是可发送的格式） → [网络发送] → 直接使用，无需转换
                     0次复制                    0次复制
```

**关键洞察**：如果我们设计的数据格式，让它在内存中的布局和传输格式完全一样，就不需要序列化了。

## Cap'n Proto：Protobuf 作者的反思

**Cap'n Proto** 由 Kenton Varda 创建——他正是 Protobuf v2 的主要作者。

在 Google 工作多年后，他意识到 Protobuf 的根本问题：

> **「Protobuf 设计时，序列化的开销被认为是可以接受的。但在某些场景，这个假设不成立。」**

### Cap'n Proto 的设计原则

1. **内存格式 = 传输格式**
2. **随机访问**：可以读取任意字段，不需要解析整个消息
3. **时间旅行**：可以在不同版本的数据间自由切换

### 实际例子

定义一个 Schema：

```capnp
struct Person {
  name @0 :Text;
  age @1 :UInt32;
  email @2 :Text;
}
```

使用时：

```cpp
// 创建消息（实际上是直接在内存缓冲区中构建）
capnp::MallocMessageBuilder message;
Person::Builder person = message.initRoot<Person>();
person.setName("Alice");
person.setAge(30);
person.setEmail("alice@example.com");

// 「序列化」—— 其实只是获取内存指针
auto bytes = messageToFlatArray(message);

// 发送 bytes...

// 「反序列化」—— 其实只是做一次指针转换
auto reader = capnp::FlatArrayMessageReader(bytes);
Person::Reader personReader = reader.getRoot<Person>();

// 直接读取，没有任何解析！
std::cout << personReader.getName().cStr();  // 直接指向缓冲区中的数据
```

**神奇之处**：`getName()` 返回的字符串直接指向接收缓冲区，没有任何复制！

## FlatBuffers：Google 游戏团队的答案

几乎同一时期（2014 年），Google 的游戏团队发布了 **FlatBuffers**。

### 设计背景

FlatBuffers 诞生于 Google 的 Fun Propulsion Labs（游戏技术团队），专门为 Android 游戏优化。

**移动游戏的特殊约束**：

| 约束 | 说明 |
|------|------|
| 内存有限 | 手机内存不如 PC |
| GC 敏感 | Java/C# 的垃圾回收会造成卡顿 |
| 电量珍贵 | CPU 工作越多，耗电越快 |
| 热量管理 | 持续高负载会导致降频 |

FlatBuffers 的目标：**零内存分配，零解析开销**。

### Schema 定义

```fbs
table Person {
  name: string;
  age: int;
  email: string;
}

root_type Person;
```

### 使用示例

```cpp
// 创建 FlatBuffer
flatbuffers::FlatBufferBuilder builder;
auto name = builder.CreateString("Alice");
auto email = builder.CreateString("alice@example.com");
auto person = CreatePerson(builder, name, 30, email);
builder.Finish(person);

// 获取二进制数据
auto buf = builder.GetBufferPointer();
auto size = builder.GetSize();

// 发送 buf...

// 接收后直接使用，无需解析！
auto person = GetPerson(received_buf);
std::cout << person->name()->c_str();  // 直接访问缓冲区
std::cout << person->age();
```

### FlatBuffers 的内存布局

```
+--------+--------+--------+--------+
| vtable | offset | offset | data   |
|  ptr   | to name| to age | ...    |
+--------+--------+--------+--------+
                    ↓
            +-------+-------+
            | len=5 | Alice |
            +-------+-------+
```

- **vtable**：虚表，记录每个字段的偏移量
- **offset**：指向实际数据的偏移
- **data**：实际数据（字符串、数组等）

关键：**所有引用都是偏移量，不是指针**。这意味着整个缓冲区可以直接 memcpy，在任何地址都能工作。

## 性能对比

### 序列化速度（每秒操作数）

| 格式 | 序列化 | 反序列化 | 访问单个字段 |
|------|--------|---------|-------------|
| JSON | 100 万 | 80 万 | 50 万 |
| Protobuf | 500 万 | 400 万 | N/A（需要先反序列化） |
| FlatBuffers | 1000 万 | ∞（无需反序列化） | 5000 万 |
| Cap'n Proto | 1000 万 | ∞（无需反序列化） | 5000 万 |

*注：数据为量级估算，实际性能因数据结构和硬件而异*

### 内存分配

| 格式 | 序列化时 | 反序列化时 |
|------|---------|-----------|
| JSON | 多次分配 | 多次分配 |
| Protobuf | 较少分配 | 每个消息一次 |
| FlatBuffers | 一次预分配 | 零分配 |
| Cap'n Proto | 一次预分配 | 零分配 |

**零分配意味着**：
- 没有 GC 压力（对 Java/C#/Go 很重要）
- 没有内存碎片
- 性能更可预测

## Cap'n Proto vs FlatBuffers

| 特性 | Cap'n Proto | FlatBuffers |
|------|-------------|-------------|
| 创建者 | Kenton Varda（Protobuf 作者） | Google 游戏团队 |
| 对齐方式 | 8 字节对齐 | 按类型对齐（更紧凑） |
| 语言支持 | C++, Rust, Go, Python... | 更广泛（包括 C#, Java） |
| RPC 支持 | 内置 | 需要额外库 |
| 生态成熟度 | 中等 | 较高（Google 背书） |
| 体积 | 稍大（8 字节对齐） | 更小 |

**选择建议**：
- 需要 RPC 支持 → Cap'n Proto
- 游戏开发 / 需要 C# → FlatBuffers
- 需要最广泛的语言支持 → FlatBuffers

## 零拷贝的代价

天下没有免费的午餐。零拷贝格式也有明显的缺点：

### 1. 内存对齐开销

为了实现零拷贝，数据需要按特定边界对齐：

```
Cap'n Proto（8 字节对齐）：
+-------+-------+-------+-------+-------+-------+-------+-------+
| data  | pad   | pad   | pad   | pad   | pad   | pad   | pad   |
+-------+-------+-------+-------+-------+-------+-------+-------+
  1 字节实际数据 + 7 字节填充

FlatBuffers（按类型对齐）：
+-------+-------+-------+-------+
| data  | pad   | pad   | pad   |
+-------+-------+-------+-------+
  1 字节实际数据 + 3 字节填充（如果后续是 4 字节类型）
```

**对于小消息，填充可能比数据还多。**

### 2. 编程复杂度

**Protobuf**（简单）：
```cpp
Person person;
person.set_name("Alice");
person.set_age(30);
std::string data = person.SerializeAsString();
```

**FlatBuffers**（复杂）：
```cpp
flatbuffers::FlatBufferBuilder builder;
auto name = builder.CreateString("Alice");  // 必须先创建字符串
auto person = CreatePerson(builder, name, 30, email);  // 然后创建对象
builder.Finish(person);
auto buf = builder.GetBufferPointer();
```

必须按**自底向上**的顺序构建（先创建叶节点，再创建父节点）。

### 3. 不可变性

零拷贝缓冲区通常是**只读的**。如果要修改数据，需要创建新的缓冲区。

```cpp
// 不能这样做：
auto person = GetPerson(buf);
person->set_age(31);  // 错误！FlatBuffer 是只读的

// 必须这样做：
flatbuffers::FlatBufferBuilder builder;
// 完全重新构建...
```

### 4. 调试更困难

```
原始字节：
08 00 00 00 0C 00 00 00 10 00 00 00 05 00 00 00 41 6C 69 63 65 ...
```

没有工具，完全无法理解。

## 适用场景

### ✅ 适合零拷贝的场景

| 场景 | 原因 |
|------|------|
| 游戏引擎 | 每帧时间预算紧张 |
| 高频交易 | 微秒级延迟要求 |
| 嵌入式系统 | 资源受限，避免内存分配 |
| 大数据处理 | 海量数据，减少拷贝 |
| 内存映射文件 | 直接映射到内存使用 |

### ❌ 不适合零拷贝的场景

| 场景 | 原因 |
|------|------|
| 普通 Web API | 复杂度不值得 |
| 小消息高频场景 | 对齐开销可能抵消收益 |
| 需要修改数据 | 零拷贝通常是只读的 |
| 跨语言互操作优先 | Protobuf 生态更成熟 |

## 一个真实案例：游戏网络同步

假设一个 MOBA 游戏需要同步 10 个玩家的状态：

```fbs
table PlayerState {
  player_id: int;
  position: Vec3;
  rotation: float;
  health: int;
  mana: int;
  animation_state: int;
  skills_cooldown: [float];
}

table GameState {
  tick: ulong;
  players: [PlayerState];
}
```

每秒 60 次更新，每次更新 10 个玩家。

**性能对比**：

| 格式 | 每帧序列化时间 | 每秒 CPU 开销 |
|------|---------------|--------------|
| JSON | ~500 μs | ~30 ms |
| Protobuf | ~50 μs | ~3 ms |
| FlatBuffers | ~5 μs | ~0.3 ms |

**FlatBuffers 节省了 99% 的序列化 CPU 开销。**

## 总结

**零拷贝格式的核心思想**：

> **内存格式 = 传输格式 = 存储格式**
>
> 数据只存在一份，只复制一次（传输时），在接收端直接使用。

**适用性判断**：

| 问题 | 答案 |
|------|------|
| 序列化是性能瓶颈吗？ | 不是 → 用 Protobuf |
| 需要频繁修改数据吗？ | 是 → 用 Protobuf |
| 每微秒都重要吗？ | 是 → 考虑零拷贝 |
| 团队熟悉零拷贝吗？ | 否 → 考虑学习成本 |

**数据格式演进**：

```
JSON（人类可读，慢）
  ↓ 需要更快
Protobuf（二进制，快）
  ↓ 需要更快
FlatBuffers/Cap'n Proto（零拷贝，极快）
```

**最重要的是**：

> **99% 的应用不需要零拷贝优化。**
>
> 在你证明序列化是瓶颈之前，不要使用零拷贝格式。过早优化是万恶之源。

---

到目前为止，我们讨论的都是「传输格式」——程序之间交换数据用的。但程序还需要「配置文件」——人类编写、程序读取的格式。

下一篇，我们来看一条平行的演进线：**配置文件的演进，从 INI 到 YAML 到 TOML**。

---

*上一篇：[二进制觉醒：当 JSON 不够快](/posts/data-format-evolution-2-binary/)*

*下一篇：[配置文件简史：从混沌到秩序](/posts/data-format-evolution-4-config/)*

*本系列：*
1. *[从 XML 到 JSON，复杂之死](/posts/data-format-evolution-1-xml-to-json/)*
2. *[二进制觉醒：当 JSON 不够快](/posts/data-format-evolution-2-binary/)*
3. *零拷贝：当序列化本身也嫌慢（本篇）*
4. *[配置文件简史：从混沌到秩序](/posts/data-format-evolution-4-config/)*
5. *[API 范式之争：REST、GraphQL、gRPC](/posts/data-format-evolution-5-api-paradigm/)*
6. *[LLM 时代：TOON 与格式的未来](/posts/data-format-evolution-6-llm-toon/)*
