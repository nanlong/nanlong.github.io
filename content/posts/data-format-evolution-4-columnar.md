---
title: "数据格式演进史（四）：列式革命，当数据以亿行计"
date: 2025-12-07T12:30:00+08:00
draft: false
tags: ["数据格式", "Parquet", "Arrow", "列式存储", "大数据", "数据湖"]
categories: ["技术"]
description: "你的 SQL 只查 2 列，数据库却扫描了全部 200 列。当数据量达到 TB 级别，这种浪费足以让查询从秒级变成小时级。"
series: ["数据格式演进史"]
---

你的 SQL 只查 2 列，数据库却扫描了全部 200 列。当数据量达到 TB 级别，这种浪费足以让查询从秒级变成小时级。

<!--more-->

## 前情回顾

在前三篇中，我们看到了数据格式如何为**传输效率**优化：

- **JSON → Protobuf**：文本到二进制，体积减半
- **Protobuf → FlatBuffers**：零拷贝，序列化时间趋近于零

但这些格式有一个共同点：**它们都是按行组织数据的**。

这在大多数场景下完全够用。但当你的数据量达到 TB 甚至 PB 级别时，另一个维度的优化变得至关重要：**按列存储**。

## 一个真实的噩梦

2017 年，Uber 的数据团队面临一个棘手的问题：

> **分析师写了一个简单查询，却要等 4 小时才能返回结果。**

查询是这样的：

```sql
SELECT city, SUM(fare)
FROM trips
WHERE date = '2017-01-01'
GROUP BY city;
```

就这么简单——统计某天各城市的订单金额。

问题在于，`trips` 表有 **200 多列**：乘客信息、司机信息、路线细节、支付信息、评价数据……

但这个查询只需要 3 列：`city`、`fare`、`date`。

**系统却不得不读取全部 200 列。**

为什么？

## 行式存储：直觉的陷阱

大多数数据库和文件格式采用**行式存储**（Row-oriented）：

```text
行式存储（像 JSON、CSV、Protobuf）：

记录1: [id=1, name="Alice", city="NYC", fare=25.00, rating=5, ...]
记录2: [id=2, name="Bob", city="LA", fare=18.50, rating=4, ...]
记录3: [id=3, name="Charlie", city="NYC", fare=32.00, rating=5, ...]
...
```

**每条记录的所有字段连续存放。**

这很直觉，就像人事档案：每个员工一个文件夹，里面放着他所有的资料。

### 行式存储的优点

| 场景 | 为什么行式存储好 |
|------|-----------------|
| 插入一条记录 | 直接追加到末尾 |
| 查询单条记录 | 一次定位，读取所有字段 |
| 更新一条记录 | 原地修改 |

**OLTP 场景（在线事务处理）的完美选择。**

### 行式存储的问题

但分析场景完全不同：

```sql
SELECT city, SUM(fare) FROM trips WHERE date = '2017-01-01';
```

**只需要 3 列，却要读取 200 列。**

```text
读取路径：

记录1: [id, name, city✓, fare✓, rating, payment, route, ...200列] → 只用2个
记录2: [id, name, city✓, fare✓, rating, payment, route, ...200列] → 只用2个
记录3: [id, name, city✓, fare✓, rating, payment, route, ...200列] → 只用2个
× 1亿条
```

**99% 的 I/O 被浪费了。**

## 列式存储：换个角度

**列式存储**（Column-oriented）把相同字段的数据放在一起：

```text
列式存储：

id列:     [1, 2, 3, 4, 5, ...]
name列:   ["Alice", "Bob", "Charlie", ...]
city列:   ["NYC", "LA", "NYC", ...]     ← 只读这个
fare列:   [25.00, 18.50, 32.00, ...]    ← 只读这个
rating列: [5, 4, 5, ...]
...
```

**每一列独立存储。**

现在查询 `city` 和 `fare`：

```text
读取路径：

city列:  ["NYC", "LA", "NYC", ...] ← 读取
fare列:  [25.00, 18.50, 32.00, ...]  ← 读取
其他198列: 完全不碰！
```

**只读需要的列，I/O 直接减少 99%！**

### 类比：档案柜 vs 电子表格

**行式存储 = 人事档案柜**：
- 每个员工一个文件夹
- 查某个人的所有信息：很快
- 统计"所有人的工资"：要翻遍每个文件夹

**列式存储 = Excel 表格**：
- 每一列是一个属性
- 统计"所有人的工资"：直接看 F 列
- 查某个人的所有信息：要横向收集多列

## 列式存储的额外红利

除了减少 I/O，列式存储还有两个意想不到的好处：

### 1. 压缩率飙升

相同列的数据类型相同、取值相似：

```text
city列: ["NYC", "NYC", "NYC", "LA", "LA", "NYC", "NYC", ...]
```

**连续重复的值！** 字典编码 + 游程编码可以极大压缩：

```text
原始: NYC, NYC, NYC, LA, LA, NYC, NYC
字典: {0: "NYC", 1: "LA"}
编码: 0×3, 1×2, 0×2  → 极小！
```

**列式存储的压缩率通常是行式存储的 3-10 倍。**

| 数据类型 | 行式压缩率 | 列式压缩率 |
|---------|-----------|-----------|
| 字符串（低基数） | 2-3x | 10-20x |
| 整数 | 1.5-2x | 5-10x |
| 时间戳 | 2x | 8-15x |

### 2. SIMD 向量化

现代 CPU 有 SIMD 指令（单指令多数据），可以一次处理多个值：

```text
行式存储：
  加载 record1.fare → 累加
  加载 record2.fare → 累加  ← 每次只处理1个
  加载 record3.fare → 累加

列式存储：
  加载 fare[0:8] → SIMD 一次累加8个！
  加载 fare[8:16] → SIMD 一次累加8个！
```

**CPU 利用率大幅提升。**

## Apache Parquet：大数据的基石

2013 年，Twitter 和 Cloudera 联合发布了 **Apache Parquet**——列式存储的事实标准。

### 为什么 Parquet 成为标准？

| 特性 | 说明 |
|------|------|
| 列式存储 | 只读需要的列 |
| 高压缩率 | 支持 Snappy、Gzip、ZSTD |
| 嵌套数据 | 支持 JSON 风格的嵌套结构 |
| Schema 演进 | 可以安全添加列 |
| 语言无关 | Java、Python、C++、Rust... |
| 生态广泛 | Spark、Hive、Presto、DuckDB... |

### Parquet 文件结构

```text
┌─────────────────────────────────────┐
│           文件元数据                 │
├─────────────────────────────────────┤
│  Row Group 1                        │
│  ├── Column Chunk: id               │
│  │   └── Page → Page → Page         │
│  ├── Column Chunk: name             │
│  │   └── Page → Page → Page         │
│  ├── Column Chunk: city             │
│  │   └── Page → Page → Page         │
│  └── ...                            │
├─────────────────────────────────────┤
│  Row Group 2                        │
│  └── ...                            │
├─────────────────────────────────────┤
│           Footer（Schema + 统计）    │
└─────────────────────────────────────┘
```

**关键设计**：
- **Row Group**：一批行（通常 50MB-1GB），支持并行处理
- **Column Chunk**：一个 Row Group 内某列的数据
- **Page**：最小 I/O 单位（通常 1MB）
- **Footer**：包含 Schema 和每列的统计信息（最大值、最小值）

### 谓词下推：跳过不需要的数据

Footer 中的统计信息支持**谓词下推**（Predicate Pushdown）：

```sql
SELECT * FROM trips WHERE fare > 100;
```

Parquet 可以直接跳过统计信息显示 `max(fare) < 100` 的 Row Group！

```text
Row Group 1: fare min=5, max=50     → 跳过！
Row Group 2: fare min=20, max=200   → 需要读取
Row Group 3: fare min=10, max=80    → 跳过！
```

**连数据都不用读，直接从元数据判断。**

### Python 示例

```python
import pyarrow.parquet as pq

# 写入 Parquet
df.to_parquet("trips.parquet", compression="snappy")

# 只读取需要的列
table = pq.read_table("trips.parquet", columns=["city", "fare"])

# 带谓词过滤
table = pq.read_table(
    "trips.parquet",
    columns=["city", "fare"],
    filters=[("date", "=", "2017-01-01")]
)
```

## Apache Arrow：内存中的列式格式

Parquet 解决了**磁盘存储**的问题。但还有另一个效率黑洞：

> **Python 读 Parquet → 转成 Pandas → 传给 Spark → 又转格式 → 传给 R → 再转格式...**

每次跨系统传递数据，都要序列化、反序列化。

**2016 年，Apache Arrow 诞生了。**

### Arrow 的核心思想

> **定义一个标准的内存中列式格式，所有系统直接共享，不需要转换。**

```text
之前：
Pandas ──序列化──► 字节流 ──反序列化──► Spark ──序列化──► 字节流 ──反序列化──► R

之后：
Pandas ─┐
        │
Spark ──┼──► Arrow 内存格式（零拷贝共享）
        │
R ──────┘
```

### Arrow 的内存布局

```text
Arrow Array（Int64 类型）：

┌────────────────────────────────────────┐
│  Validity Bitmap: [1,1,1,0,1,1,...]   │  ← 标记 NULL
├────────────────────────────────────────┤
│  Values: [10, 20, 30, ?, 50, 60, ...]  │  ← 连续内存
└────────────────────────────────────────┘
         ↑
    内存对齐，支持 SIMD
```

**关键设计**：
- 连续内存，缓存友好
- 显式 NULL 位图
- 固定宽度类型无需偏移
- 内存对齐，支持向量化

### Arrow 的威力：跨语言零拷贝

```python
import pyarrow as pa

# Python 创建 Arrow 数组
arr = pa.array([1, 2, 3, 4, 5])

# 传给 Rust（零拷贝！）
# Rust 代码直接读取同一块内存

# 传给 R（零拷贝！）
# R 代码直接读取同一块内存
```

**没有序列化，没有内存复制，指针直接传递。**

### Parquet + Arrow = 完美组合

```text
磁盘                    内存                    计算
┌─────────┐            ┌─────────┐            ┌─────────┐
│ Parquet │ ──读取──►  │  Arrow  │ ──共享──►  │ Pandas  │
│  文件   │            │  Table  │            │ Spark   │
└─────────┘            └─────────┘            │ DuckDB  │
   列式                   列式                 └─────────┘
   压缩                   原生
```

- **Parquet**：磁盘存储，高压缩，持久化
- **Arrow**：内存计算，零拷贝，跨系统共享

## 性能对比：真实数据

以 1 亿行、200 列的数据为例：

### 存储体积

| 格式 | 体积 | 相对 CSV |
|------|------|---------|
| CSV | 100 GB | 100% |
| JSON | 150 GB | 150% |
| Parquet (Snappy) | 15 GB | 15% |
| Parquet (ZSTD) | 10 GB | 10% |

**Parquet 压缩到 CSV 的 1/10！**

### 查询时间（只查 2 列）

| 格式 | 读取时间 | I/O 量 |
|------|---------|--------|
| CSV | 10 分钟 | 100 GB |
| JSON | 15 分钟 | 150 GB |
| Parquet | 5 秒 | 1 GB |

**从分钟级到秒级！**

### 跨系统传输

| 场景 | 无 Arrow | 有 Arrow |
|------|---------|---------|
| Pandas → Spark | 30 秒 | < 1 秒 |
| Spark → R | 45 秒 | < 1 秒 |

**零拷贝的威力。**

## 什么时候用列式格式？

### ✅ 适合的场景

| 场景 | 原因 |
|------|------|
| 数据分析/BI | 聚合查询只需少数列 |
| 数据湖 | 存储成本、查询效率 |
| 机器学习特征 | 读取特定特征列 |
| 日志分析 | 海量数据，少数维度 |
| 时序数据 | 高压缩率 |

### ❌ 不适合的场景

| 场景 | 原因 |
|------|------|
| OLTP 事务 | 需要频繁更新单行 |
| 实时写入 | Parquet 是批量写优化 |
| 小数据量 | 元数据开销不值得 |
| 需要全行读取 | 列式存储需要组装 |

## 生态系统

**谁在用 Parquet + Arrow？**

| 项目 | 使用方式 |
|------|---------|
| Apache Spark | 默认数据格式 |
| Pandas | 2.0 默认后端是 Arrow |
| DuckDB | 原生支持 Parquet |
| Polars | 基于 Arrow 构建 |
| Databricks | Delta Lake 基于 Parquet |
| Snowflake | 内部使用列式存储 |

**如果你做数据相关工作，这两个格式是必须掌握的。**

## 常见问题

### Q：Parquet 可以增量更新吗？

**A：不直接支持。** Parquet 是不可变格式，更新需要重写文件。

实际做法：
- 使用 Delta Lake / Iceberg / Hudi 提供事务层
- 或者按时间分区，只更新最新分区

### Q：Arrow 会占用很多内存吗？

**A：是的，Arrow 是非压缩的内存格式。**

但这是设计取舍：
- 压缩 = 计算开销
- Arrow 优化的是计算速度，不是内存占用
- 可以用 Arrow 的 streaming 模式分批处理

### Q：我应该用 Parquet 还是 ORC？

**A：大多数情况选 Parquet。**

- ORC 主要是 Hive 生态
- Parquet 生态更广（Spark、Pandas、DuckDB...）
- 性能差异不大

## 总结

**列式存储的核心洞察**：

> **分析查询通常只需要少数列，但行式存储强迫你读取所有列。**
>
> 把存储从"按行"改成"按列"，可以减少 90%+ 的 I/O。

**Parquet vs Arrow**：

| 维度 | Parquet | Arrow |
|------|---------|-------|
| 位置 | 磁盘 | 内存 |
| 压缩 | 高压缩 | 无压缩 |
| 用途 | 持久化存储 | 计算和传输 |
| 互操作 | 文件交换 | 零拷贝共享 |

**它们是互补的**：
- Parquet 负责高效存储
- Arrow 负责高效计算
- 组合使用 = 现代数据栈的标配

**数据格式的两条演进线**：

```text
传输优化线：
JSON → Protobuf → FlatBuffers（零拷贝传输）

分析优化线：
CSV → Parquet（磁盘列式） + Arrow（内存列式）
```

---

到目前为止，我们讨论的都是「程序间的数据交换」。但程序还需要「配置文件」——人类编写、程序读取的格式。

下一篇，我们来看另一条演进线：**配置文件的演进，从 INI 到 YAML 到 TOML**。

---

*上一篇：[零拷贝：当序列化本身也嫌慢](/posts/data-format-evolution-3-zero-copy/)*

*下一篇：[配置文件简史：从混沌到秩序](/posts/data-format-evolution-5-config/)*

*本系列：*
1. *[从 XML 到 JSON，复杂之死](/posts/data-format-evolution-1-xml-to-json/)*
2. *[二进制觉醒：当 JSON 不够快](/posts/data-format-evolution-2-binary/)*
3. *[零拷贝：当序列化本身也嫌慢](/posts/data-format-evolution-3-zero-copy/)*
4. *列式革命：当数据以亿行计（本篇）*
5. *[配置文件简史：从混沌到秩序](/posts/data-format-evolution-5-config/)*
6. *[API 范式之争：REST、GraphQL、gRPC](/posts/data-format-evolution-6-api-paradigm/)*
7. *[LLM 时代：TOON 与格式的未来](/posts/data-format-evolution-7-llm-toon/)*
