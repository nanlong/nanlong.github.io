---
title: "数据格式演进史（五）：API 范式之争，REST、GraphQL、gRPC"
date: 2025-12-07T14:00:00+08:00
draft: false
tags: ["API", "REST", "GraphQL", "gRPC", "微服务", "架构"]
categories: ["技术"]
description: "2012 年，Facebook 的移动端工程师们面临一个难题：News Feed 页面需要调用几十个 REST 接口才能渲染完成。他们的解决方案，后来改变了整个行业。"
series: ["数据格式演进史"]
---

2012 年，Facebook 的移动端工程师们面临一个难题：News Feed 页面需要调用几十个 REST 接口才能渲染完成。他们的解决方案，后来改变了整个行业。

<!--more-->

## 不只是格式的问题

在前四篇中，我们讨论了各种数据格式：

- **XML → JSON**：从复杂到简单
- **JSON → Protobuf**：从文本到二进制
- **Protobuf → FlatBuffers**：从序列化到零拷贝
- **INI → YAML → TOML**：配置文件的演进

但格式只是载体。真正影响开发效率的是：**API 如何设计？客户端如何获取数据？**

这就是「API 范式」的问题。

## REST：资源导向的时代

### REST 的核心理念

2000 年，Roy Fielding 在他的博士论文中提出了 REST（Representational State Transfer）。

核心思想：**把一切都看作「资源」，用 HTTP 动词操作资源。**

```http
GET    /users/123      # 获取用户
POST   /users          # 创建用户
PUT    /users/123      # 更新用户
DELETE /users/123      # 删除用户
```

**URL 表示资源，HTTP 方法表示操作。**

### REST 的优点

| 优点 | 说明 |
|------|------|
| 简单直观 | URL 即资源，动词即操作 |
| 无状态 | 每个请求包含所有必要信息 |
| 可缓存 | HTTP 缓存机制直接可用 |
| 统一接口 | 所有资源用相同方式访问 |
| 广泛支持 | 浏览器、curl、任何 HTTP 客户端都能用 |

### REST 成为事实标准

2010 年代，REST + JSON 成为 Web API 的默认选择：

- Twitter API
- GitHub API
- Stripe API
- 几乎所有 SaaS 产品

**如果你在 2010 年代设计 API，REST 是唯一合理的选择。**

## REST 的裂痕

随着应用变复杂，REST 的问题开始显现。

### 问题一：Over-fetching（过度获取）

假设你要显示用户卡片，只需要 `name` 和 `avatar`：

```http
GET /users/123
```

返回：

```json
{
  "id": 123,
  "name": "Alice",
  "avatar": "https://...",
  "email": "alice@example.com",
  "phone": "123-456-7890",
  "address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "zip": "94102"
  },
  "created_at": "2024-01-01",
  "updated_at": "2024-01-15",
  "preferences": { ... },
  "billing_info": { ... }
}
```

**你只要 2 个字段，服务器返回了 15 个。**

浪费带宽，尤其在移动网络上。

### 问题二：Under-fetching（获取不足）

现在你要显示用户的帖子列表，每个帖子带作者信息和评论数：

**REST 方式**：

```text
1. GET /users/123                    # 获取用户
2. GET /users/123/posts              # 获取帖子列表
3. GET /posts/456/comments/count     # 获取帖子1的评论数
4. GET /posts/789/comments/count     # 获取帖子2的评论数
5. GET /posts/101/comments/count     # 获取帖子3的评论数
...
```

**N+1 问题**：获取 N 个帖子需要 N+1 次请求。

在移动网络上，每次请求都有延迟。10 个请求可能要 2-3 秒。

### 问题三：版本地狱

你的 API 需要迭代。新版本加了字段，老版本要保持兼容：

```text
/api/v1/users/123
/api/v2/users/123
/api/v3/users/123
```

**三个版本，三套代码，三倍维护成本。**

### 问题四：文档与实现脱节

Swagger/OpenAPI 文档要和代码同步更新。

但总有人忘记更新文档，或者文档和实际行为不一致。

## Facebook 的困境

2012 年，Facebook 决定重写移动端应用。他们遇到了所有这些问题的放大版：

> **News Feed 页面需要调用几十个 REST 接口。**

每个帖子可能有：
- 发布者信息
- 点赞列表（只需要前几个人）
- 评论列表（只需要前几条）
- 分享信息
- 图片/视频
- 相关推荐
- ...

用 REST 实现：

```text
GET /me/feed
GET /posts/1/author
GET /posts/1/likes?limit=3
GET /posts/1/comments?limit=2
GET /posts/2/author
GET /posts/2/likes?limit=3
...
```

**一个页面，几十个请求。** 在 3G 网络上，这是灾难。

## 2015：GraphQL 的诞生

Facebook 的解决方案是 **GraphQL**：

> **让客户端定义需要什么数据，服务器精确返回。**

### GraphQL 的核心思想

```graphql
# 客户端请求
query {
  user(id: 123) {
    name
    avatar
    posts(first: 3) {
      title
      commentsCount
      likes(first: 5) {
        user {
          name
        }
      }
    }
  }
}
```

**服务器响应**：
```json
{
  "data": {
    "user": {
      "name": "Alice",
      "avatar": "https://...",
      "posts": [
        {
          "title": "Hello World",
          "commentsCount": 42,
          "likes": [
            { "user": { "name": "Bob" } },
            { "user": { "name": "Charlie" } }
          ]
        }
      ]
    }
  }
}
```

**一个请求，精确获取所需数据。**

### GraphQL 解决的问题

| REST 问题 | GraphQL 方案 |
|-----------|-------------|
| Over-fetching | 客户端指定字段 |
| Under-fetching | 一次请求获取所有数据 |
| N+1 问题 | 嵌套查询 |
| 版本地狱 | 字段级别废弃，无需版本号 |
| 文档脱节 | Schema 即文档 |

### GraphQL 的 Schema

```graphql
type User {
  id: ID!
  name: String!
  email: String
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
  commentsCount: Int!
}

type Query {
  user(id: ID!): User
  posts(first: Int, after: String): PostConnection!
}

type Mutation {
  createPost(input: CreatePostInput!): Post!
  deletePost(id: ID!): Boolean!
}
```

**Schema 是强类型的契约**：
- 编译时类型检查
- 自动生成文档
- IDE 自动补全

### 谁在用 GraphQL？

| 公司 | 使用场景 |
|------|---------|
| Facebook | 移动端 API |
| GitHub | API v4 |
| Shopify | 商家 API |
| Twitter | 部分功能 |
| Airbnb | 移动端 |
| Netflix | 部分服务 |

## GraphQL 的代价

GraphQL 不是银弹。它也有明显的缺点：

### 1. 复杂度转移到服务端

客户端的灵活性意味着服务端的复杂性：

```graphql
# 恶意查询：可能导致服务器崩溃
query EvilQuery {
  posts(first: 1000) {
    author {
      posts(first: 1000) {
        author {
          posts(first: 1000) {
            # 无限嵌套...
          }
        }
      }
    }
  }
}
```

需要实现：
- 查询深度限制
- 查询复杂度分析
- 速率限制
- 超时控制

### 2. 缓存困难

REST 可以直接用 HTTP 缓存：

```http
GET /users/123
Cache-Control: max-age=3600
```

GraphQL 所有请求都是 POST 到同一个端点，HTTP 缓存失效。

需要自己实现客户端缓存（Apollo、Relay）。

### 3. 文件上传麻烦

GraphQL 是 JSON over HTTP，不直接支持文件上传。

需要额外的 multipart 规范或单独的上传接口。

### 4. 学习曲线

团队需要学习：
- GraphQL 语法
- Schema 设计
- Resolver 实现
- N+1 问题（DataLoader）
- 缓存策略

## gRPC：微服务的选择

在 GraphQL 专注于「客户端灵活性」的同时，另一个方向是「服务间通信效率」。

### 微服务的挑战

当你有几十上百个微服务时：

```text
用户服务 ←→ 订单服务 ←→ 支付服务 ←→ 库存服务 ←→ ...
```

每个服务间的调用都有开销。REST + JSON 在这里不够高效。

### gRPC 的核心特性

**1. 基于 HTTP/2**

| HTTP/1.1 | HTTP/2 |
|----------|--------|
| 每请求一个连接 | 多路复用 |
| 文本协议头 | 二进制 + 压缩 |
| 无服务器推送 | 支持推送 |

**2. 使用 Protobuf**

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
  rpc CreateUser(CreateUserRequest) returns (User);
}
```

**3. 四种通信模式**

| 模式 | 场景 |
|------|------|
| Unary | 普通请求-响应 |
| Server Streaming | 日志流、实时数据 |
| Client Streaming | 文件上传 |
| Bidirectional | 聊天、游戏 |

### gRPC 的优势

| 维度 | REST | gRPC |
|------|------|------|
| 传输效率 | JSON 文本 | Protobuf 二进制 |
| 类型安全 | 无（或 OpenAPI） | 强类型（编译时检查） |
| 代码生成 | 可选 | 标准流程 |
| 流式传输 | 困难 | 原生支持 |
| 双向通信 | 需要 WebSocket | 原生支持 |

### gRPC 的局限

**1. 浏览器支持差**

浏览器不能直接调用 gRPC。需要 gRPC-Web 或网关转换。

**2. 不可读**

调试时看到的是二进制，不如 JSON 直观。

**3. 需要代码生成**

改了 proto 文件，所有客户端都要重新生成代码。

## 三种范式的对比

### 设计哲学

| 范式 | 核心思想 |
|------|---------|
| REST | 资源导向，HTTP 即接口 |
| GraphQL | 客户端驱动，精确获取 |
| gRPC | 强类型契约，高效传输 |

### 适用场景

| 场景 | 推荐 |
|------|------|
| 公开 API（第三方开发者） | REST（最通用） |
| 移动端 + 复杂数据需求 | GraphQL |
| 微服务内部通信 | gRPC |
| 实时双向通信 | gRPC 或 WebSocket |
| 简单 CRUD 应用 | REST |

### 混合使用

现实中，很多公司同时使用多种范式：

```text
                    ┌─────────────┐
                    │   浏览器     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  GraphQL    │  ← 对外 API
                    │  Gateway    │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼────┐       ┌─────▼────┐       ┌────▼────┐
   │ 用户服务 │◄─────►│ 订单服务  │◄─────►│ 支付服务 │
   └─────────┘ gRPC  └──────────┘ gRPC  └─────────┘
                           ↑
              微服务内部用 gRPC
```

- **对外**：GraphQL（灵活性）或 REST（通用性）
- **对内**：gRPC（效率）

## 如何选择？

### 决策树

```text
你在构建什么？
│
├── 公开 API（第三方使用）
│   └── REST（最通用，文档丰富）
│
├── 移动端应用
│   ├── 简单数据需求 → REST
│   └── 复杂数据需求 → GraphQL
│
├── 微服务内部
│   ├── 需要流式 → gRPC
│   └── 简单调用 → gRPC 或 REST 都行
│
├── 实时应用（聊天、游戏）
│   └── gRPC 或 WebSocket
│
└── 不确定
    └── 从 REST 开始，按需演进
```

### 务实建议

| 建议 | 原因 |
|------|------|
| 从 REST 开始 | 最简单，团队最熟悉 |
| 需要再换 | 过早优化是万恶之源 |
| 可以混用 | 不是非此即彼 |
| 工具链优先 | 选团队熟悉的 |

## 常见问题

### Q：GraphQL 会取代 REST 吗？

**A：不会。** 它们解决不同问题。

- REST 更简单、更通用
- GraphQL 更灵活、更复杂
- 大多数应用不需要 GraphQL 的灵活性

### Q：gRPC 适合公开 API 吗？

**A：通常不适合。**

- 浏览器支持差
- 需要代码生成
- 调试不直观
- 第三方开发者学习成本高

公开 API 用 REST 或 GraphQL，内部服务用 gRPC。

### Q：能同时用 GraphQL 和 gRPC 吗？

**A：可以，很常见。**

GraphQL 作为 BFF（Backend for Frontend），对接多个 gRPC 微服务。

## 总结

**API 范式演进**：

```text
SOAP/XML（复杂、严谨）
    ↓ 需要更简单
REST/JSON（简单、通用）
    ↓ 需要更灵活
GraphQL（灵活、强类型）
    ↓
    └──── 同时 ────┐
                   ↓
gRPC（高效、流式）—— 专注微服务
```

**核心教训**：

> **没有最好的 API 范式，只有最适合的。**
>
> - 公开 API 用 REST
> - 复杂前端用 GraphQL
> - 微服务用 gRPC
> - 可以混合使用

**选择时问自己**：
1. 谁是 API 的消费者？
2. 数据需求有多复杂？
3. 团队熟悉什么？
4. 性能要求如何？

---

到目前为止，我们讨论的所有格式都有一个共同的假设：**数据的消费者是程序**。

但 2023 年后，一个新的消费者出现了：**大语言模型（LLM）**。LLM 不看字节数，它看 **Token 数**。

下一篇，我们来看一个全新的维度：**TOON——为 LLM 优化的数据格式**。

---

*上一篇：[配置文件简史：从混沌到秩序](/posts/data-format-evolution-4-config/)*

*下一篇：[LLM 时代：TOON 与格式的未来](/posts/data-format-evolution-6-llm-toon/)*

*本系列：*
1. *[从 XML 到 JSON，复杂之死](/posts/data-format-evolution-1-xml-to-json/)*
2. *[二进制觉醒：当 JSON 不够快](/posts/data-format-evolution-2-binary/)*
3. *[零拷贝：当序列化本身也嫌慢](/posts/data-format-evolution-3-zero-copy/)*
4. *[配置文件简史：从混沌到秩序](/posts/data-format-evolution-4-config/)*
5. *API 范式之争：REST、GraphQL、gRPC（本篇）*
6. *[LLM 时代：TOON 与格式的未来](/posts/data-format-evolution-6-llm-toon/)*
