---
title: "数据格式演进史（六）：LLM 时代，TOON 与格式的未来"
date: 2025-12-07T15:00:00+08:00
draft: false
tags: ["数据格式", "TOON", "LLM", "AI", "Token", "未来"]
categories: ["技术"]
description: "当 AI 成为数据的消费者，一个新的稀缺资源出现了：Token。JSON 的每一个大括号、引号、冒号，都在消耗你的钱。"
series: ["数据格式演进史"]
---

当 AI 成为数据的消费者，一个新的稀缺资源出现了：Token。JSON 的每一个大括号、引号、冒号，都在消耗你的钱。

<!--more-->

## 系列回顾

在这个系列中，我们看到了数据格式 40 年的演进：

| 时代 | 驱动力 | 代表格式 |
|------|--------|---------|
| 1990s | 需要结构化 | XML |
| 2000s | 需要简单 | JSON |
| 2010s | 需要高效 | Protobuf |
| 2010s | 需要极致 | FlatBuffers |
| 2020s | 需要灵活 | GraphQL |

每一次演进都由**新的约束条件**驱动。

2023 年后，一个全新的约束条件出现了：**Token**。

## 新时代，新稀缺资源

在 LLM 时代，成本不再按字节计算，而是按 **Token** 计算。

### 什么是 Token？

Token 是 LLM 处理文本的基本单位。大致可以理解为：

- 英文：约 4 个字符 = 1 个 Token
- 中文：约 1-2 个字符 = 1 个 Token
- 代码/标点：每个符号可能是独立的 Token

```text
"Hello, World!" → ["Hello", ",", " World", "!"] → 4 Tokens
```

### Token 就是金钱

以 GPT-4 为例（2024 年价格）：

| 类型 | 价格 |
|------|------|
| 输入 | $0.03 / 1K Tokens |
| 输出 | $0.06 / 1K Tokens |

听起来不多？算一下：

| 场景 | Token 数 | 每次成本 | 每天 10 万次 |
|------|---------|---------|-------------|
| 简单查询 | 500 | $0.015 | $1,500 |
| 复杂对话 | 5,000 | $0.15 | $15,000 |
| 带上下文 | 50,000 | $1.50 | $150,000 |

**Token 数直接决定了 AI 应用的运营成本。**

### 上下文窗口的限制

除了成本，还有**容量限制**：

| 模型 | 上下文窗口 |
|------|-----------|
| GPT-3.5 | 16K Tokens |
| GPT-4 | 128K Tokens |
| Claude 3 | 200K Tokens |

听起来很大？但如果你要让 AI 分析一个代码库，或者处理大量结构化数据，很快就会撞到天花板。

## JSON 在 LLM 时代的问题

JSON 是为**人类可读**设计的，不是为 **Token 效率**设计的。

### 结构符号的开销

```json
{
    "user_id": 12345,
    "name": "Alice",
    "email": "alice@example.com"
}
```

让我们数一下 Token：

| 内容 | Token 数 | 是否必要？ |
|------|---------|-----------|
| `{`, `}` | 2 | 结构开销 |
| `"user_id":` | 2-3 | 字段名 |
| `"name":` | 2 | 字段名 |
| `"email":` | 2 | 字段名 |
| `12345` | 1-2 | 实际数据 |
| `"Alice"` | 1 | 实际数据 |
| `"alice@example.com"` | 3-4 | 实际数据 |

**大约 50% 的 Token 是结构开销，不是实际数据！**

### 更极端的例子

假设你要发送一个用户列表：

```json
[
    {"id": 1, "name": "Alice", "age": 30},
    {"id": 2, "name": "Bob", "age": 25},
    {"id": 3, "name": "Charlie", "age": 35}
]
```

每个用户都重复 `"id"`, `"name"`, `"age"`。100 个用户，这些字段名就重复了 100 次。

**JSON 的自描述特性，在 LLM 场景下成了累赘。**

## TOON：Token 导向的格式

2025 年 11 月，**TOON**（Token-Oriented Object Notation）正式发布，专门为 LLM 场景设计。

### 设计目标

> **在保持可读性的同时，最大限度减少 Token 消耗。**

### TOON 的核心语法

TOON 的核心思想是：**用声明式的结构头，替代重复的字段名**。

**JSON 格式**（16 tokens）：
```json
{"users":[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]}
```

**TOON 格式**（13 tokens）：
```toon
users[2]{id,name}:
 1,Alice
 2,Bob
```

**关键创新**：
- `[2]` 声明数组长度
- `{id,name}` 声明字段结构（只出现一次）
- 数据行用逗号分隔，无需重复字段名

**Token 节省：18.75%**

### 数组的表示

**原始数组**：
```toon
tags[3]: python,rust,go
```

等价于 JSON：`{"tags": ["python", "rust", "go"]}`

**对象数组（表格形式）**：

**JSON**：
```json
{
    "users": [
        {"id": 1, "name": "Alice", "active": true},
        {"id": 2, "name": "Bob", "active": false}
    ]
}
```

**TOON**：
```toon
users[2]{id,name,active}:
 1,Alice,true
 2,Bob,false
```

**省略了什么？**
- 所有大括号 `{}`、方括号 `[]`
- 所有引号 `""`
- 重复的字段名（`id`、`name`、`active` 只出现一次）
- 大量的冒号和逗号

### Token 节省效果

根据官方测试（使用 GPT-4 tokenizer）：

| 数据类型 | JSON Tokens | TOON Tokens | 节省 |
|---------|-------------|-------------|------|
| 简单对象（2字段） | 16 | 13 | 19% |
| 对象数组（10条） | ~120 | ~65 | 46% |
| 对象数组（100条） | ~1100 | ~350 | 68% |

**数据越规整、越重复，节省越多。**

## TOON 的完整语法

### 基本类型

```toon
# 字符串（简单值不需要引号）
name: Alice

# 包含特殊字符的字符串需要引号
message: "Hello, World!"

# 数字（标准十进制，不用科学计数法）
count: 1000000
price: 19.99

# 布尔值
active: true

# 空值
data: null
```

### 对象

```toon
user:
 name: Alice
 age: 30
```

等价于 JSON：
```json
{"user": {"name": "Alice", "age": 30}}
```

### 数组语法详解

**1. 原始值数组**：
```toon
colors[3]: red,green,blue
```

**2. 对象数组（表格形式）**：
```toon
items[2]{id,name,price}:
 1,Apple,1.50
 2,Banana,0.75
```

**3. 列表项形式**（用于非统一结构）：
```toon
mixed[3]:
 - simple value
 - key: value
 - another
```

### 混合结构示例

```toon
order:
 id: 12345
 customer:
  name: Alice
  email: alice@example.com
 items[2]{productId,qty,price}:
  101,2,29.99
  102,1,49.99
 tags[2]: urgent,premium
 total: 109.97
```

等价的 JSON 需要更多结构符号，Token 消耗显著增加。

### 分隔符选项

TOON 支持多种分隔符：

| 分隔符 | 示例 | 适用场景 |
|--------|------|---------|
| 逗号 `,` | `1,Alice,30` | 默认，紧凑 |
| 制表符 | `1→Alice→30` | 数据含逗号时 |
| 管道符 `\|` | `1\|Alice\|30` | 可读性优先 |

## 什么时候用 TOON？

### ✅ 适合的场景

| 场景 | 原因 |
|------|------|
| LLM 提示词工程 | 直接节省成本 |
| AI Agent 数据交换 | 上下文更大 |
| 大量结构化数据分析 | 表格格式高效 |
| 成本敏感的 AI 应用 | Token 就是钱 |

### ❌ 不适合的场景

| 场景 | 原因 |
|------|------|
| 程序间通信 | Protobuf/JSON 更成熟 |
| 配置文件 | TOML/YAML 生态更好 |
| 需要严格验证 | TOON Schema 还在发展 |
| 非 AI 场景 | 没有 Token 约束 |

## 格式演进的新维度

TOON 的出现揭示了一个有趣的趋势：

**数据格式的「消费者」在变化。**

| 时代 | 主要消费者 | 优化目标 |
|------|-----------|---------|
| XML 时代 | 程序 + 人类 | 可扩展性、严谨性 |
| JSON 时代 | 程序 + 人类 | 简单性、可读性 |
| Protobuf 时代 | 程序 | 效率、速度 |
| TOON 时代 | **AI** | Token 效率 |

**未来可能会有更多为 AI 优化的格式。**

### 三方博弈

现在的数据格式需要同时考虑三方：

```text
        人类可读
           ▲
          /|\
         / | \
        /  |  \
       /   |   \
      /    |    \
     ▼     ▼     ▼
程序高效 ◄────► AI 高效
```

- **人类可读**：JSON, YAML, TOML
- **程序高效**：Protobuf, FlatBuffers
- **AI 高效**：TOON（以及未来更多）

**最优解可能是：不同场景用不同格式。**

## 未来展望

### 趋势一：场景专用化

通用格式（JSON）的时代可能在终结。未来可能是：

- 公开 API → JSON（通用性）
- 微服务 → Protobuf（效率）
- 配置 → TOML（简单性）
- AI 交互 → TOON（Token 效率）

### 趋势二：智能转换

未来的工具可能自动选择格式：

```python
# 伪代码
data = load_data()

if target == "human":
    output = to_json(data)
elif target == "service":
    output = to_protobuf(data)
elif target == "llm":
    output = to_toon(data)
```

### 趋势三：AI 原生格式

TOON 可能只是开始。未来可能出现：

- 针对特定模型优化的格式
- 动态压缩（根据上下文调整）
- 语义压缩（AI 自动理解省略）

## 系列总结

### 40 年的演进规律

**驱动力一直是：新的约束条件。**

| 年代 | 新约束 | 响应 |
|------|--------|------|
| 1990s | 需要结构化、可扩展 | XML |
| 2000s | XML 太复杂 | JSON |
| 2010s | JSON 太慢 | Protobuf |
| 2010s | 序列化有开销 | FlatBuffers |
| 2015s | REST 太死板 | GraphQL |
| 2020s | Token 是稀缺资源 | TOON |

**每次演进都不是「更好」，而是「更适合新场景」。**

### 格式选择的本质

> **选择数据格式，本质上是在多个维度之间做权衡。**

| 维度 | 两端 |
|------|------|
| 可读性 | 人类可读 ←→ 机器可读 |
| 效率 | 表达力 ←→ 紧凑性 |
| 灵活性 | 无 Schema ←→ 强 Schema |
| 通用性 | 专用优化 ←→ 广泛支持 |

**没有最好的格式，只有最适合当前约束的格式。**

### 决策框架

```text
你的数据要做什么？
│
├── 人类编写和阅读 → YAML / TOML
│
├── 程序间通信
│   ├── 公开 API → JSON
│   ├── 内部微服务 → Protobuf / gRPC
│   └── 极致性能 → FlatBuffers
│
├── AI 交互 → TOON
│
└── 不确定 → 从 JSON 开始，按需演进
```

### 最重要的教训

> **技术选型的核心不是「什么最好」，而是「什么最适合」。**
>
> 每种格式都是在特定约束下的最优解。约束变了，最优解就变了。
>
> 保持开放，持续学习，是应对技术演进的唯一方式。

---

这个系列到此结束。感谢阅读。

如果你的团队正在选型数据格式，希望这个系列能给你一些启发。

如果你有任何问题或想法，欢迎留言讨论。

---

*上一篇：[API 范式之争：REST、GraphQL、gRPC](/posts/data-format-evolution-5-api-paradigm/)*

*本系列：*
1. *[从 XML 到 JSON，复杂之死](/posts/data-format-evolution-1-xml-to-json/)*
2. *[二进制觉醒：当 JSON 不够快](/posts/data-format-evolution-2-binary/)*
3. *[零拷贝：当序列化本身也嫌慢](/posts/data-format-evolution-3-zero-copy/)*
4. *[配置文件简史：从混沌到秩序](/posts/data-format-evolution-4-config/)*
5. *[API 范式之争：REST、GraphQL、gRPC](/posts/data-format-evolution-5-api-paradigm/)*
6. *LLM 时代：TOON 与格式的未来（本篇）*

*延伸阅读：*
- *[TOON 官方规范](https://github.com/toon-format/spec)*
- *[TOON Rust 实现](https://github.com/toon-format/toon-rust)*
