---
title: '认证演进史（一）：Session——服务器说"我记得你"'
date: 2025-12-08T23:10:00+08:00
draft: false
tags: ["认证", "Session", "Cookie", "Web安全"]
categories: ["技术"]
description: "你登录了网站，刷新页面，还是登录状态。这背后是什么魔法？答案是：服务器在偷偷记着你。"
series: ["认证演进史"]
---

你登录了网站，刷新页面，还是登录状态。点击"我的订单"，不用再输密码。这背后是什么魔法？

答案是：**服务器在偷偷记着你。**

<!--more-->

## HTTP 的失忆症

HTTP 协议有一个"缺陷"：**它没有记忆。**

每一次请求，对服务器来说都是全新的陌生人。就像一个患有短期失忆症的店员：

```text
顾客：我要买这本书。
店员：好的，请付款。
顾客：（付完款）我刚才买的书呢？
店员：您是哪位？您买过书吗？
```

**这就是 HTTP 的本质**：无状态（Stateless）。

```text
请求 1: GET /index.html
        → 服务器返回页面，然后忘记你

请求 2: GET /my-orders
        → 服务器：你谁啊？请先登录
```

你可能会问：那我每次请求都带上用户名密码不就行了？

**理论上可以，但这太疯狂了：**

- 每个请求都要传密码 → 密码暴露风险成倍增加
- 每个请求都要查数据库验证 → 性能灾难
- 用户体验极差 → 没人愿意用

我们需要一种方式，让服务器"记住"你已经登录过了。

## Session：服务器的记事本

**Session 的本质**：服务器开了一个小本本，记下"这个人已经验证过身份了"。

```text
登录流程：

1. 你：输入用户名密码，提交登录
2. 服务器：验证通过！我给你发一张"通行证"
3. 服务器：同时在我的小本本上记一笔："通行证 ABC123 = 用户张三"
4. 你：收到通行证，存在浏览器里
5. 之后每次请求：自动出示通行证
6. 服务器：查小本本，哦是张三，放行
```

这张"通行证"就是 **Session ID**，服务器的"小本本"就是 **Session 存储**。

### Session ID 的传递：Cookie

浏览器怎么"自动"带上 Session ID？答案是 **Cookie**。

```text
登录成功后，服务器返回：

HTTP/1.1 200 OK
Set-Cookie: session_id=abc123; HttpOnly; Secure; Path=/

浏览器看到 Set-Cookie，自动保存。
之后每次请求同一网站，自动带上：

GET /my-orders HTTP/1.1
Cookie: session_id=abc123
```

**类比**：Cookie 就像酒店的房卡。入住时前台给你房卡（Set-Cookie），之后你每次进出酒店都刷这张卡（自动带 Cookie），前台一刷就知道你是 302 房的客人。

### 服务器如何存储 Session？

服务器的"小本本"可以有多种形式：

```text
方案 1：内存存储（单机）
  ┌─────────────────────────────┐
  │  服务器内存                  │
  │  abc123 → {user: "张三", ...} │
  │  def456 → {user: "李四", ...} │
  └─────────────────────────────┘

  优点：快
  缺点：服务器重启就没了，无法多机共享

方案 2：数据库存储
  ┌─────────────────────────────┐
  │  sessions 表                 │
  │  id | user_id | data | expires │
  └─────────────────────────────┘

  优点：持久化，可共享
  缺点：每次请求都要查数据库

方案 3：Redis 存储（推荐）
  ┌─────────────────────────────┐
  │  Redis                       │
  │  session:abc123 → {...}      │
  │  TTL: 30 分钟自动过期        │
  └─────────────────────────────┘

  优点：快 + 持久化 + 原生支持过期 + 可集群共享
```

### Rust 实现示例

用 `axum` + `tower-sessions` 实现 Session 认证：

```rust
use axum::{
    Router,
    routing::{get, post},
    response::Json,
    Extension,
};
use tower_sessions::{Session, SessionManagerLayer, MemoryStore};
use serde::{Deserialize, Serialize};

#[derive(Serialize)]
struct User {
    id: u64,
    name: String,
}

async fn login(session: Session) -> Json<&'static str> {
    // 验证用户名密码后...
    session.insert("user_id", 42).await.unwrap();
    session.insert("username", "张三").await.unwrap();
    Json("登录成功")
}

async fn get_profile(session: Session) -> Json<Option<User>> {
    // 从 Session 获取用户信息
    let user_id: Option<u64> = session.get("user_id").await.unwrap();
    let username: Option<String> = session.get("username").await.unwrap();

    match (user_id, username) {
        (Some(id), Some(name)) => Json(Some(User { id, name })),
        _ => Json(None), // 未登录
    }
}

async fn logout(session: Session) -> Json<&'static str> {
    session.delete().await.unwrap();
    Json("已登出")
}

#[tokio::main]
async fn main() {
    let session_store = MemoryStore::default();
    let session_layer = SessionManagerLayer::new(session_store);

    let app = Router::new()
        .route("/login", post(login))
        .route("/profile", get(get_profile))
        .route("/logout", post(logout))
        .layer(session_layer);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**代码要点**：
- `Session` 通过中间件自动注入
- `session.insert()` 存数据，`session.get()` 取数据
- `session.delete()` 销毁整个 Session（登出）

## Session 的安全问题

Session 看起来简单，但安全问题不少。

### 问题一：Session 固定攻击

```text
攻击流程：

1. 攻击者访问网站，获得 Session ID = evil123
2. 攻击者诱导受害者使用这个 ID 登录
   （比如发送链接 https://bank.com/?sessionid=evil123）
3. 受害者用 evil123 登录成功
4. 攻击者用 evil123 访问，发现已经登录了！
```

**防护**：登录成功后，必须**换一个新的 Session ID**。

```rust
async fn login(session: Session) -> Json<&'static str> {
    // 验证密码成功后...

    // 关键：重新生成 Session ID！
    session.cycle_id().await.unwrap();

    session.insert("user_id", 42).await.unwrap();
    Json("登录成功")
}
```

### 问题二：Session 劫持

如果攻击者偷到了你的 Session ID，他就能冒充你。

```text
偷 Session ID 的方式：

1. 网络窃听（HTTP 明文传输）
   → 防护：强制 HTTPS

2. XSS 攻击（JavaScript 读取 Cookie）
   → 防护：Cookie 设置 HttpOnly

3. 中间人攻击
   → 防护：Cookie 设置 Secure（只在 HTTPS 传输）
```

**正确的 Cookie 设置**：

```text
Set-Cookie: session_id=abc123;
            HttpOnly;    ← JS 无法读取
            Secure;      ← 只在 HTTPS 传输
            SameSite=Lax; ← 防止 CSRF
            Path=/;
            Max-Age=1800  ← 30 分钟过期
```

### 问题三：CSRF 攻击

CSRF（跨站请求伪造）利用了浏览器"自动带 Cookie"的特性。

```text
攻击场景：

1. 你登录了银行网站，Cookie 里有 Session ID
2. 你访问了恶意网站，页面里有：
   <img src="https://bank.com/transfer?to=hacker&amount=10000">
3. 浏览器自动发请求，自动带上银行的 Cookie
4. 银行服务器以为是你在转账！
```

**防护方案**：

```text
方案 1：SameSite Cookie
  Set-Cookie: session_id=abc123; SameSite=Strict

  Strict：只有从同一网站发起的请求才带 Cookie
  Lax：GET 请求可以带，POST 不行（推荐）

方案 2：CSRF Token
  服务器生成随机 Token，嵌入表单
  提交时验证 Token
  攻击者无法获取 Token，请求会失败
```

## Session 的局限

Session 解决了"记住用户"的问题，但它自己也有问题：

### 问题一：服务器有状态

```text
单机时代：没问题
  用户 → 服务器 A（内存里有 Session）

多机时代：麻烦了
  用户 → 负载均衡 → 服务器 A（有 Session）
                  → 服务器 B（没有 Session！）

  请求 1 去了 A，登录成功
  请求 2 去了 B，Session 不在，要求重新登录
```

**解决方案**：

| 方案 | 原理 | 缺点 |
|------|------|------|
| 粘性会话 | 同一用户固定分配到同一服务器 | 负载不均，服务器挂了就丢失 |
| Session 复制 | 所有服务器同步 Session | 同步开销大，延迟问题 |
| 集中存储（Redis） | 所有服务器连同一个 Redis | 引入新依赖，Redis 成为瓶颈 |

### 问题二：跨域困难

```text
前后端分离场景：

前端：https://app.example.com
后端：https://api.example.com

Cookie 默认不能跨域发送！
需要配置 CORS + credentials
复杂且容易出错
```

### 问题三：移动端支持

```text
原生 App 没有浏览器的 Cookie 机制
需要手动存储和发送 Session ID
实现起来别扭
```

### 问题四：每次请求都要查存储

```text
用户量大了：
  - 每个请求都要查 Redis（虽然 Redis 很快）
  - Session 数据越来越大
  - 相比无状态方案，多了一次网络往返
```

## 核心认知

> **Session 的本质**：用服务器的存储，换取无状态协议上的"有状态"体验。

**优点**：
- 概念简单，容易理解
- 服务器完全控制，可随时踢人下线
- 浏览器原生支持，无需额外代码

**缺点**：
- 服务器有状态，水平扩展困难
- 跨域场景复杂
- 存储压力随用户数线性增长

**适用场景**：
- 传统服务端渲染的 Web 应用
- 用户量不大（< 100 万）
- 单体架构或简单集群

## 引出下一篇

Session 的核心问题是：**服务器要存东西**。

能不能让服务器**完全不存储**，却还能验证用户身份？

答案是：把用户信息**装进令牌里**，让令牌自己证明自己。

这就是下一篇要讲的 **JWT（JSON Web Token）**。

---

## 常见问题

### Q：Session ID 要多长？

**A：至少 128 位随机数。**

Session ID 必须不可预测。如果攻击者能猜到别人的 Session ID，就能冒充他人。

```rust
// 好：使用密码学安全的随机数生成器
use rand::Rng;
let session_id: String = rand::thread_rng()
    .sample_iter(&rand::distributions::Alphanumeric)
    .take(32)  // 32 字符 = 约 190 位熵
    .map(char::from)
    .collect();

// 坏：使用时间戳或自增 ID
let session_id = format!("session_{}", timestamp); // 可预测！
```

### Q：Session 应该多久过期？

**A：看场景，但不要太长。**

| 场景 | 建议过期时间 |
|------|-------------|
| 银行等高安全 | 5-15 分钟无操作过期 |
| 普通网站 | 30 分钟-2 小时 |
| "记住我"功能 | 7-30 天（使用单独的 Remember Token） |

### Q：用户改密码后，旧 Session 怎么办？

**A：必须全部失效。**

```rust
// 方法 1：存储密码版本号
struct Session {
    user_id: u64,
    password_version: u32,  // 用户改密码时 +1
}

// 验证时检查版本号是否匹配

// 方法 2：存储所有 Session ID，改密码时批量删除
// Redis: 用 Set 存储用户的所有 Session ID
```

---

*下一篇：[JWT——把状态装进令牌](/posts/auth-evolution-2-jwt/)*

*本系列：*
1. *Session——服务器说"我记得你"（本篇）*
2. *[JWT——把状态装进令牌](/posts/auth-evolution-2-jwt/)*
3. *[OAuth 2.0——让别人帮你认证](/posts/auth-evolution-3-oauth/)*
4. *[OpenID Connect——标准化的身份层](/posts/auth-evolution-4-oidc/)*
5. *[Passkey——告别密码](/posts/auth-evolution-5-passkey/)*
6. *[实战：设计一个现代认证系统](/posts/auth-evolution-6-practice/)*
