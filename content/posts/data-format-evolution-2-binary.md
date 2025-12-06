---
title: "数据格式演进史（二）：二进制觉醒，当 JSON 不够快"
date: 2025-12-07T11:00:00+08:00
draft: false
tags: ["数据格式", "Protobuf", "Thrift", "gRPC", "MessagePack", "二进制"]
categories: ["技术"]
description: "一家公司发现，他们 30% 的服务器 CPU 时间花在 JSON 序列化上。当每秒处理百万请求时，「人类可读」变成了奢侈品。"
series: ["数据格式演进史"]
---

一家公司发现，他们 30% 的服务器 CPU 时间花在 JSON 序列化上。当每秒处理百万请求时，「人类可读」变成了奢侈品。

<!--more-->

## 前情回顾

在上一篇中，我们看到了 JSON 如何打败 XML：

- **简单**：10 分钟上手
- **轻量**：没有结束标签，体积小
- **通用**：所有语言都支持

但 JSON 有一个隐藏的成本：**它是文本格式**。

## 一个真实的故事

2010 年，某大型互联网公司的工程师在做性能分析时发现了一个惊人的事实：

> **30% 的 CPU 时间花在 JSON 序列化和反序列化上。**

这意味着：
- 每 3 台服务器，有 1 台在「翻译」数据
- 每年数百万美元的服务器成本，有 30% 花在格式转换上

为什么 JSON 这么慢？

## JSON 的性能问题

### 问题一：文本 vs 二进制

看看数字 `12345` 在不同格式下的表示：

| 格式 | 表示方式 | 字节数 |
|------|---------|--------|
| JSON（文本） | `"12345"` | 5 字节（5 个 ASCII 字符） |
| 二进制（int32） | `0x00003039` | 4 字节 |
| 二进制（varint） | `0xB960` | 2 字节 |

**数字越大，差距越明显**：

| 数字 | JSON 字节数 | 二进制 varint 字节数 | 节省 |
|------|-------------|---------------------|------|
| 127 | 3 | 1 | 67% |
| 16383 | 5 | 2 | 60% |
| 2097151 | 7 | 3 | 57% |

### 问题二：字段名重复

JSON 是「自描述」的，每条记录都带着字段名：

```json
[
    {"user_id": 1, "name": "Alice", "email": "alice@example.com"},
    {"user_id": 2, "name": "Bob", "email": "bob@example.com"},
    {"user_id": 3, "name": "Charlie", "email": "charlie@example.com"}
]
```

`"user_id"`、`"name"`、`"email"` 重复了 3 次。如果有 100 万条记录呢？

**字段名本身可能比数据还大。**

### 问题三：解析成本

JSON 解析器需要：
1. 逐字符扫描
2. 识别字符串边界（找引号）
3. 转义字符处理（`\"`, `\\`）
4. 数字转换（字符串 → 数值）
5. Unicode 处理

而二进制格式可以直接「读取」内存，几乎不需要转换。

## Google 的答案：Protocol Buffers

2008 年，Google 开源了 **Protocol Buffers**（简称 Protobuf）。实际上，它在 Google 内部已经使用了近 10 年。

### 核心思想：用 Schema 换效率

Protobuf 的设计哲学：

> **既然发送方和接收方都知道数据结构，为什么每次传输都要带上字段名？**

就像两个人约定好：「第一个数字是用户 ID，第二个是年龄」，之后只传 `1, 30` 就够了，不用每次都说「用户 ID 是 1，年龄是 30」。

### 定义 Schema

Protobuf 用 `.proto` 文件定义数据结构：

```protobuf
syntax = "proto3";

message User {
    int32 user_id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
}
```

注意那些数字 `1`, `2`, `3`, `4`——它们是**字段编号**，在传输时代替字段名。

### 编码后的样子

同样的数据，JSON vs Protobuf：

**JSON**（67 字节）：
```json
{"user_id":12345,"name":"Alice","email":"alice@example.com","age":30}
```

**Protobuf**（约 30 字节）：
```
08 B9 60 12 05 41 6C 69 63 65 1A 11 61 6C 69 63 65 40 65 78 61 6D 70 6C 65 2E 63 6F 6D 20 1E
```

**体积减少 55%！**

### 字段编号的智慧

为什么用数字而不是字段名？

1. **更短**：`1` 只需要几个比特，`user_id` 需要 7 字节
2. **兼容性**：可以改字段名而不破坏协议
3. **无歧义**：避免大小写、拼写问题

**黄金法则**：字段编号一旦使用，永远不要改变。

### 向后兼容的艺术

Protobuf 设计的一个精妙之处：**新老版本可以互相通信**。

**场景**：你有一个 `User` 消息，现在要加一个 `phone` 字段。

```protobuf
message User {
    int32 user_id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    string phone = 5;  // 新增字段
}
```

- **新代码读取老数据**：`phone` 字段会是默认值（空字符串）
- **老代码读取新数据**：`phone` 字段会被忽略（不会报错）

**对比 JSON**：如果接收方代码没准备好处理新字段，可能会崩溃。

### Schema 演进的规则

| 操作 | 是否安全 | 说明 |
|------|---------|------|
| 添加可选字段 | ✅ 安全 | 老代码会忽略新字段 |
| 删除可选字段 | ⚠️ 注意 | 不要复用已删除的字段编号 |
| 改字段名 | ✅ 安全 | 传输用的是编号，不是名称 |
| 改字段编号 | ❌ 危险 | 会导致数据错误解析 |
| 改字段类型 | ❌ 危险 | 可能导致数据损坏 |

## Facebook 的答案：Thrift

几乎同一时期（2007 年），Facebook 开源了 **Thrift**。它的野心更大：

> **不只是序列化格式，还是完整的 RPC 框架。**

### Thrift vs Protobuf

| 特性 | Protobuf | Thrift |
|------|----------|--------|
| 序列化格式 | 1 种 | 多种（Binary、Compact、JSON...） |
| RPC 框架 | 需要配合 gRPC | 内置 |
| 语言支持 | 广泛 | 更广泛 |
| 社区活跃度 | 极高 | 高 |
| 学习曲线 | 较低 | 中等 |

Thrift 更像一个「全家桶」，Protobuf 更像一个「专注的工具」。

### 谁在用什么？

| 公司 | 选择 | 原因 |
|------|------|------|
| Google | Protobuf + gRPC | 自家产品 |
| Facebook | Thrift | 自家产品 |
| Netflix | Protobuf | 性能 |
| Uber | Thrift → Protobuf | 迁移中 |
| Airbnb | Thrift | 历史原因 |

## gRPC：Protobuf 的最佳搭档

2015 年，Google 开源了 **gRPC**——基于 HTTP/2 和 Protobuf 的 RPC 框架。

### 为什么需要 gRPC？

Protobuf 解决了「数据怎么编码」，但没解决「数据怎么传输」。

传统 REST API 的问题：

```
POST /api/users/create
Content-Type: application/json

{"name": "Alice", "email": "alice@example.com"}
```

- 每次请求都要建立连接
- 没有类型检查（服务端不知道客户端发的对不对）
- 只能请求-响应，不能双向流

### gRPC 的核心特性

**1. 强类型契约**

```protobuf
service UserService {
    rpc CreateUser(CreateUserRequest) returns (User);
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (stream User);  // 流式返回
}
```

客户端代码是自动生成的，类型安全。

**2. 基于 HTTP/2**

| 特性 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 连接复用 | 每个请求一个连接 | 多个请求共享连接 |
| 头部压缩 | 无 | 有（HPACK） |
| 双向流 | 不支持 | 支持 |
| 服务器推送 | 不支持 | 支持 |

**3. 四种通信模式**

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| Unary | 一问一答 | 普通 API 调用 |
| Server Streaming | 一问多答 | 日志流、实时数据 |
| Client Streaming | 多问一答 | 文件上传 |
| Bidirectional | 多问多答 | 实时聊天、游戏 |

## 轻量级替代：MessagePack

如果你想要二进制的效率，但不想写 Schema，**MessagePack** 是个好选择。

### 设计理念

> **「它就是二进制的 JSON。」**

```python
import msgpack

data = {"name": "Alice", "age": 30}
packed = msgpack.packb(data)    # 打包
unpacked = msgpack.unpackb(packed)  # 解包
```

**不需要定义 Schema，直接把 JSON 对象二进制化。**

### JSON vs MessagePack vs Protobuf

| 特性 | JSON | MessagePack | Protobuf |
|------|------|-------------|----------|
| 需要 Schema | 否 | 否 | 是 |
| 人类可读 | 是 | 否 | 否 |
| 体积 | 大 | 中 | 小 |
| 解析速度 | 慢 | 快 | 最快 |
| 类型安全 | 无 | 弱 | 强 |
| 向后兼容 | 手动处理 | 手动处理 | 内置支持 |

**选择建议**：
- 动态语言 + 快速原型 → MessagePack
- 强类型 + 大规模系统 → Protobuf

## Apache Avro：大数据的选择

在大数据生态中，还有一个重要的格式：**Apache Avro**（2009）。

### 为什么 Hadoop 生态需要 Avro？

| 需求 | Avro 的解决方案 |
|------|----------------|
| Schema 演进 | Schema 随数据存储，版本兼容性好 |
| 动态类型 | 不需要代码生成，运行时解析 Schema |
| 分片友好 | 数据块可独立处理，适合 MapReduce |
| 压缩效率 | 支持多种压缩算法 |

**谁在用 Avro？**
- Apache Kafka（消息格式）
- Apache Spark（数据交换）
- Confluent Schema Registry

> **选择建议**：如果你在 Hadoop/Kafka 生态，Avro 是自然选择；否则 Protobuf 更通用。

## CBOR：为 IoT 设计

**CBOR**（Concise Binary Object Representation）是 IETF 标准化的二进制格式，专为资源受限的环境设计。

### 为什么 IoT 需要专门的格式？

| 约束 | 要求 |
|------|------|
| 内存 | 嵌入式设备可能只有几 KB RAM |
| 带宽 | 蜂窝网络流量按字节计费 |
| 电量 | 每次传输都消耗电池 |
| 算力 | CPU 很弱，复杂解析耗电 |

CBOR 的优势：
- 极小的解析器（可以在 1KB 代码内实现）
- 确定性编码（同样的数据总是产生同样的字节）
- 自描述（可以不用 Schema）

**使用者**：
- WebAuthn（浏览器无密码登录）
- COSE（CBOR Object Signing and Encryption）
- CoAP（受限应用协议）

## 调试的代价

二进制格式有一个明显的缺点：**不可读**。

```
08 B9 60 12 05 41 6C 69 63 65 1A 11 61 6C 69 63 65 40 65 78
```

这是什么？没有工具，你完全看不懂。

### 调试工具

| 格式 | 调试方案 |
|------|---------|
| Protobuf | `protoc --decode` 命令 |
| Thrift | TCompactProtocol + 自定义打印 |
| MessagePack | 转成 JSON 查看 |
| gRPC | grpcurl 工具 |

### 日志策略

常见做法：
1. **开发环境**：使用 JSON，方便调试
2. **生产环境**：使用二进制，提高性能
3. **关键日志**：记录时转成可读格式

## 实际对比数据

以一个典型的用户数据为例：

**数据内容**：
- user_id: 12345678
- name: "Alice Johnson"
- email: "alice.johnson@example.com"
- age: 30
- created_at: 1704067200 (Unix 时间戳)
- roles: ["admin", "user"]

**各格式体积对比**：

| 格式 | 字节数 | 相对 JSON |
|------|--------|-----------|
| JSON | 145 | 100% |
| JSON（压缩后） | 120 | 83% |
| MessagePack | 95 | 66% |
| Protobuf | 62 | 43% |

**序列化速度（每秒操作数，越大越好）**：

| 格式 | 序列化 | 反序列化 |
|------|--------|---------|
| JSON | 100 万 | 80 万 |
| MessagePack | 300 万 | 250 万 |
| Protobuf | 500 万 | 400 万 |

*数据仅供参考，实际性能因实现和数据结构而异*

## 什么时候该切换到二进制格式？

**不需要切换的场景**：
- 请求量不大（每秒 < 1000）
- 主要瓶颈在数据库或业务逻辑
- 团队不熟悉，学习成本高
- 需要频繁调试线上数据

**值得考虑切换的场景**：
- 微服务间高频通信
- 移动端需要节省流量
- 性能分析显示序列化是瓶颈
- 需要强类型契约

**必须切换的场景**：
- 每秒处理 10 万+ 请求
- 带宽成本是重要考量
- 实时系统（游戏、交易）

## 常见问题

### Q：我该选 Protobuf 还是 Thrift？

**A：大多数情况选 Protobuf。**

- 如果你用 Google 技术栈（Go、gRPC、K8s）→ Protobuf
- 如果你的公司已经在用 Thrift → 继续用 Thrift
- 如果你需要更多传输协议选择 → Thrift

### Q：MessagePack 能替代 Protobuf 吗？

**A：看场景。**

- 动态类型语言 + 快速迭代 → MessagePack 够用
- 强类型 + 长期维护 + 多团队协作 → Protobuf 更好

### Q：迁移成本高吗？

**A：取决于系统规模。**

| 系统规模 | 迁移策略 |
|---------|---------|
| 新项目 | 直接用 Protobuf/gRPC |
| 小型系统 | 可以整体迁移 |
| 大型系统 | 渐进式，新服务用新格式，老服务慢慢迁移 |

## 总结

**JSON vs 二进制格式**

| 维度 | JSON | 二进制（Protobuf） |
|------|------|-------------------|
| 人类可读 | ✅ 是 | ❌ 否 |
| 体积 | 大 | 小（节省 40-60%） |
| 解析速度 | 慢 | 快（3-5 倍） |
| 类型安全 | 无 | 强 |
| Schema | 可选 | 必须 |
| 调试难度 | 低 | 高 |
| 适用场景 | Web API、配置 | 微服务、高性能 |

**核心教训**：

> **性能优化是有代价的。二进制格式用空间换时间，用可读性换效率。**
>
> 只有当性能真的成为瓶颈时，才值得付出这个代价。

---

但 Protobuf 也不是终点。

当每一微秒都很重要时——比如游戏引擎、高频交易——人们发现：**序列化本身也是开销**。能不能直接把内存里的数据发出去，不做任何转换？

下一篇，我们来看 **FlatBuffers 和 Cap'n Proto**：零拷贝的极致优化。

---

*上一篇：[从 XML 到 JSON，复杂之死](/posts/data-format-evolution-1-xml-to-json/)*

*下一篇：[零拷贝：当序列化本身也嫌慢](/posts/data-format-evolution-3-zero-copy/)*

*本系列：*
1. *[从 XML 到 JSON，复杂之死](/posts/data-format-evolution-1-xml-to-json/)*
2. *二进制觉醒：当 JSON 不够快（本篇）*
3. *[零拷贝：当序列化本身也嫌慢](/posts/data-format-evolution-3-zero-copy/)*
4. *[配置文件简史：从混沌到秩序](/posts/data-format-evolution-4-config/)*
5. *[API 范式之争：REST、GraphQL、gRPC](/posts/data-format-evolution-5-api-paradigm/)*
6. *[LLM 时代：TOON 与格式的未来](/posts/data-format-evolution-6-llm-toon/)*
