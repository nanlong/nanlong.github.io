---
title: "认证演进史（二）：JWT——把状态装进令牌"
date: 2025-12-08T23:20:00+08:00
draft: false
tags: ["认证", "JWT", "Token", "无状态"]
categories: ["技术"]
description: "Session 让服务器变得'有状态'，这在分布式时代是个麻烦。JWT 说：让令牌自己携带信息，服务器不用存任何东西。"
series: ["认证演进史"]
---

上一篇我们讲了 Session：服务器用一个"小本本"记住你是谁。

但这个小本本带来了新问题：**服务器有状态了**。在微服务和水平扩展的时代，这是个大麻烦。

有没有办法让服务器**什么都不存**，却还能验证你的身份？

<!--more-->

## 无状态的诱惑

想象一下，如果服务器不需要存储任何会话信息：

```text
传统 Session：
  用户 → 服务器 A（要查 Redis）
       → 服务器 B（要查 Redis）
       → 服务器 C（要查 Redis）
       所有人都依赖 Redis，Redis 挂了全完蛋

无状态认证：
  用户 → 服务器 A（自己验证）
       → 服务器 B（自己验证）
       → 服务器 C（自己验证）
       服务器之间互不依赖，随便加机器
```

**无状态的好处**：
- 无限水平扩展（加机器就行）
- 没有共享存储的瓶颈
- 服务器可以随时重启
- 微服务架构天然友好

**问题**：不存任何东西，怎么知道这个请求来自谁？

答案是：**让令牌自己携带信息**。

## JWT 的结构

JWT（JSON Web Token）是一个自包含的令牌，它把用户信息"打包"在令牌里。

```text
一个真实的 JWT：

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IuW8oOS4iSIsImlhdCI6MTUxNjIzOTAyMn0.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

看起来像乱码？其实是三部分用点号连接：

Header.Payload.Signature
头部.载荷.签名
```

### 第一部分：Header（头部）

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Base64URL 编码后：`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9`

头部声明了两件事：
- `alg`：签名算法（这里是 HMAC-SHA256）
- `typ`：令牌类型（JWT）

### 第二部分：Payload（载荷）

```json
{
  "sub": "1234567890",
  "name": "张三",
  "role": "admin",
  "iat": 1516239022,
  "exp": 1516242622
}
```

Base64URL 编码后：`eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IuW8oOS4iSIsImlhdCI6MTUxNjIzOTAyMn0`

载荷包含实际数据（称为 Claims）：
- `sub`：Subject，用户标识
- `iat`：Issued At，签发时间
- `exp`：Expiration，过期时间
- 还可以加自定义字段：`name`、`role` 等

### 第三部分：Signature（签名）

```text
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret_key
)
```

签名是对前两部分的"数字签章"，保证内容没被篡改。

**类比**：

> JWT 就像一张**支票**。
>
> - Payload 是支票内容（金额、收款人）
> - Signature 是银行的印章
>
> 任何人都能看到支票内容，但只有银行能盖这个章。
> 如果有人改了金额，章就对不上了。

### 重要提醒：Base64 不是加密！

```text
Base64URL 编码：
  {"name":"张三"} → eyJuYW1lIjoi5byg5LiJIn0

这不是加密，只是编码！任何人都能解码：
  eyJuYW1lIjoi5byg5LiJIn0 → {"name":"张三"}

所以：绝对不要在 JWT 里放密码、信用卡号等敏感信息！
```

## 签名算法选择

JWT 支持多种签名算法，选择很重要：

### HS256：对称签名

```text
原理：签发和验证用同一个密钥（HMAC）

签发：服务器用 secret 签名
验证：服务器用 secret 验证

优点：简单、快
缺点：密钥需要保密，不能分享给第三方验证
适用：单体应用、内部服务
```

### RS256：非对称签名

```text
原理：私钥签名，公钥验证（RSA + SHA256）

签发：认证服务用私钥签名
验证：任何服务用公钥验证（公钥可公开）

优点：公钥可以分享，第三方能验证
缺点：比 HS256 慢
适用：微服务、需要第三方验证的场景
```

### ES256：椭圆曲线

```text
原理：和 RS256 类似，但用椭圆曲线算法

优点：签名更短，速度更快
缺点：算法相对新，某些老库不支持
适用：对性能敏感的场景
```

**选择建议**：

| 场景 | 推荐算法 |
|------|----------|
| 单体应用 | HS256 |
| 微服务架构 | RS256 或 ES256 |
| 对外开放验证 | RS256 或 ES256 |
| 性能敏感 | ES256 |

## JWT 的使用流程

```text
登录流程：

1. 用户提交用户名密码
2. 服务器验证通过，生成 JWT
3. 将 JWT 返回给客户端
4. 客户端存储 JWT（localStorage/cookie）
5. 后续请求带上 JWT（通常在 Authorization 头）

请求验证：

GET /api/profile
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

服务器收到后：
1. 提取 JWT
2. 验证签名（用密钥）
3. 检查是否过期
4. 签名正确 + 未过期 = 信任载荷中的用户信息
```

### Rust 实现示例

使用 `jwt-simple` crate：

```rust
use jwt_simple::prelude::*;
use serde::{Deserialize, Serialize};

// 自定义 Claims 数据
#[derive(Debug, Serialize, Deserialize)]
struct UserClaims {
    user_id: String,
    name: String,
    role: String,
}

// 全局密钥（实际应用中应从配置读取）
lazy_static::lazy_static! {
    static ref JWT_KEY: HS256Key = HS256Key::generate();
}

fn create_token(user_id: &str, name: &str, role: &str) -> Result<String, jwt_simple::Error> {
    let custom = UserClaims {
        user_id: user_id.to_string(),
        name: name.to_string(),
        role: role.to_string(),
    };

    // 创建带自定义数据的 Claims，有效期 1 小时
    let claims = Claims::with_custom_claims(custom, Duration::from_hours(1))
        .with_issuer("my-app")
        .with_subject(user_id);

    JWT_KEY.authenticate(claims)
}

fn verify_token(token: &str) -> Result<JWTClaims<UserClaims>, jwt_simple::Error> {
    let options = VerificationOptions {
        allowed_issuers: Some(HashSet::from_strings(&["my-app"])),
        ..Default::default()
    };

    JWT_KEY.verify_token::<UserClaims>(token, Some(options))
}
```

在 `axum` 中使用中间件提取：

```rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode, header},
    response::IntoResponse,
    Json,
};

struct AuthUser {
    user_id: String,
    name: String,
    role: String,
}

#[async_trait::async_trait]
impl<S> FromRequestParts<S> for AuthUser
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _: &S) -> Result<Self, Self::Rejection> {
        // 从 Authorization 头提取 Token
        let auth_header = parts.headers
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .ok_or((StatusCode::UNAUTHORIZED, "Missing Authorization header"))?;

        // 去掉 "Bearer " 前缀
        let token = auth_header
            .strip_prefix("Bearer ")
            .ok_or((StatusCode::UNAUTHORIZED, "Invalid Authorization format"))?;

        // 验证 Token
        let claims = verify_token(token)
            .map_err(|_| (StatusCode::UNAUTHORIZED, "Invalid token"))?;

        // 从 claims.custom 获取自定义数据
        Ok(AuthUser {
            user_id: claims.custom.user_id,
            name: claims.custom.name,
            role: claims.custom.role,
        })
    }
}

// 在 handler 中使用
async fn get_profile(user: AuthUser) -> impl IntoResponse {
    Json(serde_json::json!({
        "id": user.user_id,
        "name": user.name,
        "role": user.role
    }))
}
```

## Access Token 与 Refresh Token

只用一个 Token 有问题：

```text
Token 有效期太短（如 5 分钟）：
  用户频繁被踢出，体验差

Token 有效期太长（如 7 天）：
  Token 泄露后，攻击者能用很久
```

**解决方案**：双 Token 机制

```text
Access Token：
  - 短期（15 分钟 - 1 小时）
  - 携带用户信息
  - 每个请求都带
  - 泄露影响有限

Refresh Token：
  - 长期（7 天 - 30 天）
  - 只用于获取新的 Access Token
  - 存储更安全（HttpOnly Cookie）
  - 可以在服务端撤销
```

**流程**：

```text
1. 登录成功：
   返回 access_token (15分钟) + refresh_token (7天)

2. 正常请求：
   带上 access_token

3. access_token 过期：
   用 refresh_token 请求 /refresh 接口
   获取新的 access_token

4. refresh_token 过期：
   用户需要重新登录

5. 用户改密码/被封禁：
   服务端使 refresh_token 失效
```

```rust
#[derive(Serialize)]
struct TokenPair {
    access_token: String,
    refresh_token: String,
    expires_in: u64,
}

async fn login(/* credentials */) -> Result<Json<TokenPair>, StatusCode> {
    // 验证用户...

    let access_token = create_token(&user_id, &name, &role)
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    let refresh_token = create_refresh_token(&user_id); // 存入数据库，7 天过期

    Ok(Json(TokenPair {
        access_token,
        refresh_token,
        expires_in: 3600, // 1 小时
    }))
}

async fn refresh(refresh_token: String) -> Result<Json<TokenPair>, StatusCode> {
    // 验证 refresh_token（从数据库查询）
    let user = validate_refresh_token(&refresh_token)
        .ok_or(StatusCode::UNAUTHORIZED)?;

    // 生成新的 token pair
    let access_token = create_token(&user.id, &user.name, &user.role)
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(TokenPair {
        access_token,
        refresh_token, // 可以返回同一个，或生成新的（滚动刷新）
        expires_in: 3600,
    }))
}
```

## JWT 的坑

### 坑一：无法主动失效

```text
Session：
  用户改密码 → 删除 Session → 立即失效

JWT：
  用户改密码 → Token 还在用户手里 → 在过期前仍然有效！
```

**解决方案**：

```text
方案 1：短过期时间
  Access Token 只有 15 分钟
  即使泄露，影响有限

方案 2：黑名单
  维护一个"已撤销 Token"列表
  验证时检查是否在黑名单
  （讽刺：又变成有状态了）

方案 3：版本号
  Token 里带 password_version
  用户改密码时 version +1
  验证时检查 version 是否匹配
```

### 坑二：Token 过大

```text
Session ID：32 字节

JWT：可能几百字节甚至更大
  - Header: ~36 字节
  - Payload: 取决于你塞了多少东西
  - Signature: ~43 字节（HS256）

如果在 Payload 里塞了权限列表：
  ["read:users", "write:users", "read:orders", ...]

Token 可能变成 1KB+
每个请求都带 1KB 的 Header
移动网络下很痛
```

**建议**：Payload 只放必要信息，详细权限查数据库。

### 坑三：alg:none 攻击

某些 JWT 库有个致命漏洞：

```json
// 攻击者构造的恶意 Token
{
  "alg": "none",  // 声明不需要签名
  "typ": "JWT"
}
```

如果服务端库信任这个声明，就不验证签名了！

**防护**：明确指定允许的算法，不要让 Token 自己说了算。

```rust
// jwt-simple 通过类型系统防止此攻击
// HS256Key 只能验证 HS256 签名，不接受其他算法
let key = HS256Key::generate();
let claims = key.verify_token::<NoCustomClaims>(&token, None)?;

// 库内部会检查 Token 的 alg 是否匹配密钥类型
// 如果 Token 声明 alg: none，验证会失败
```

### 坑四：密钥泄露

```text
HS256 密钥泄露：
  攻击者能伪造任意用户的 Token
  所有已签发的 Token 都不可信
  需要更换密钥，所有用户重新登录

RS256 私钥泄露：
  同样严重
  公钥可以继续用，但私钥必须换
```

**建议**：
- 使用密钥管理服务（AWS KMS、HashiCorp Vault）
- 定期轮换密钥
- 不要把密钥写在代码里

## Session vs JWT：如何选择

| 维度 | Session | JWT |
|------|---------|-----|
| 状态 | 有状态（服务器存储） | 无状态（Token 自包含） |
| 扩展性 | 需要共享存储 | 天然支持水平扩展 |
| 撤销 | 删除即失效 | 需要额外机制 |
| 大小 | 只有 Session ID | 携带数据，较大 |
| 性能 | 每次要查存储 | 本地验证，无 IO |
| 跨域 | Cookie 有同源限制 | 随便放在哪 |
| 移动端 | Cookie 支持不好 | 直接用，无障碍 |

**选择建议**：

```text
用 Session：
  - 传统服务端渲染
  - 安全要求高，需要即时踢人
  - 架构简单，不需要水平扩展

用 JWT：
  - 前后端分离（SPA）
  - 微服务架构
  - 移动端 App
  - 需要跨域认证
  - 无状态 API
```

## 核心认知

> **JWT 的本质**：用计算（签名验证）换取存储（服务端无状态）。

**优点**：
- 服务端无状态，易于扩展
- 跨域友好
- 移动端友好
- 自包含，减少数据库查询

**缺点**：
- 无法主动撤销（或需要额外机制）
- Token 较大
- 密钥管理是关键

## 引出下一篇

JWT 解决了"服务器不想存东西"的问题。

但还有一个场景没解决：**用户想用 GitHub 账号登录你的网站**。

你不应该知道用户的 GitHub 密码，但你需要确认他确实是那个 GitHub 用户。

这就是 **OAuth 2.0** 要解决的问题：如何安全地"借用"别人的身份。

---

## 常见问题

### Q：JWT 应该存在哪里？

**A：看场景。**

| 存储位置 | 优点 | 缺点 | 风险等级 |
|----------|------|------|----------|
| localStorage | 方便访问 | 可被 XSS 窃取 | ⚠️ 不推荐 |
| sessionStorage | 标签页隔离 | 同样可被 XSS 窃取 | ⚠️ 不推荐 |
| HttpOnly Cookie | XSS 无法访问 | 需要 CSRF 防护 | ✅ 推荐 |
| 内存 | XSS 无法直接访问 | 刷新页面就没了 | ✅ 推荐（配合刷新机制） |

### Q：JWT 过期时间设多长？

**A：Access Token 短，Refresh Token 长。**

| Token 类型 | 建议时间 |
|------------|----------|
| Access Token | 15 分钟 - 1 小时 |
| Refresh Token | 7 天 - 30 天 |

### Q：能不能用 JWT 做 Session？

**A：可以，但要想清楚。**

把 JWT 存在 HttpOnly Cookie 里，每次请求自动带上，这其实就是用 JWT 模拟 Session。

好处是服务端无状态，坏处是失去了"即时踢人"的能力。

---

*上一篇：[Session——服务器说"我记得你"](/posts/auth-evolution-1-session/)*

*下一篇：[OAuth 2.0——让别人帮你认证](/posts/auth-evolution-3-oauth/)*

*本系列：*
1. *[Session——服务器说"我记得你"](/posts/auth-evolution-1-session/)*
2. *JWT——把状态装进令牌（本篇）*
3. *[OAuth 2.0——让别人帮你认证](/posts/auth-evolution-3-oauth/)*
4. *[OpenID Connect——标准化的身份层](/posts/auth-evolution-4-oidc/)*
5. *[Passkey——告别密码](/posts/auth-evolution-5-passkey/)*
6. *[实战：设计一个现代认证系统](/posts/auth-evolution-6-practice/)*
