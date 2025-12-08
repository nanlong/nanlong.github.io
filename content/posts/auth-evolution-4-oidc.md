---
title: "认证演进史（四）：OpenID Connect——标准化的身份层"
date: 2025-12-08T23:40:00+08:00
draft: false
tags: ["认证", "OIDC", "OpenID Connect", "JWT", "Web安全"]
categories: ["技术"]
description: "OAuth 2.0 能授权，但不能认证。OpenID Connect 在 OAuth 基础上，加了一个简单的东西：ID Token。"
series: ["认证演进史"]
---

上一篇我们说过：**OAuth 2.0 是授权协议，不是认证协议。**

用 OAuth 做登录有安全风险。那正确的做法是什么？

答案是：**OpenID Connect（OIDC）**。

<!--more-->

## 从 OAuth 到 OIDC

### OAuth 的缺失

```text
OAuth 2.0 能告诉你：
  ✓ 用户允许你访问他的资源
  ✓ 这是访问资源的令牌

OAuth 2.0 不能告诉你：
  ✗ 用户是谁
  ✗ 用户什么时候登录的
  ✗ 这个令牌是不是颁发给你的
```

### OIDC 的补充

```text
OpenID Connect = OAuth 2.0 + 身份层

OIDC 额外告诉你：
  ✓ 用户的唯一标识（sub）
  ✓ 令牌颁发给谁（aud）
  ✓ 什么时候颁发的（iat）
  ✓ 什么时候过期（exp）
  ✓ 用户如何认证的（acr/amr）
```

## ID Token：身份的证明

**OIDC 最核心的概念就是 ID Token。**

### ID Token 是什么？

```text
ID Token = 一个 JWT，包含用户身份信息

┌────────────────────────────────────────────────────────────┐
│                       ID Token                              │
├────────────────────────────────────────────────────────────┤
│  Header:  { "alg": "RS256", "typ": "JWT" }                 │
├────────────────────────────────────────────────────────────┤
│  Payload:                                                   │
│  {                                                          │
│    "iss": "https://accounts.google.com",  // 谁颁发的       │
│    "sub": "110169484474386276334",        // 用户唯一 ID    │
│    "aud": "your-client-id",               // 颁发给谁       │
│    "exp": 1699900000,                     // 过期时间       │
│    "iat": 1699896400,                     // 颁发时间       │
│    "nonce": "abc123",                     // 防重放攻击     │
│    "name": "张三",                         // 可选：用户名   │
│    "email": "zhangsan@example.com",       // 可选：邮箱     │
│    "picture": "https://..."               // 可选：头像     │
│  }                                                          │
├────────────────────────────────────────────────────────────┤
│  Signature: RS256 签名                                      │
└────────────────────────────────────────────────────────────┘
```

### ID Token vs Access Token

```text
┌──────────────────┬───────────────────────┬──────────────────────┐
│                  │ ID Token              │ Access Token         │
├──────────────────┼───────────────────────┼──────────────────────┤
│ 用途              │ 证明用户身份           │ 访问资源              │
│ 给谁看            │ Client（你的 App）     │ Resource Server      │
│ 内容              │ 用户信息               │ 权限范围              │
│ 格式              │ 必须是 JWT             │ 可以是任意格式        │
│ 验证方             │ Client 验证           │ Resource Server 验证 │
└──────────────────┴───────────────────────┴──────────────────────┘

类比：

ID Token = 身份证
  "我是张三，这是公安局（iss）给你（aud）开的证明"

Access Token = 门禁卡
  "持此卡可进入 3 楼会议室"
```

## OIDC 认证流程

OIDC 基于 OAuth 2.0 的授权码模式，只是多了 `openid` scope 和 ID Token。

### 完整流程

```text
┌──────┐     ┌──────┐     ┌────────────┐     ┌──────────────┐
│ 用户  │     │ App  │     │ OIDC 提供商 │     │ UserInfo    │
└──┬───┘     └──┬───┘     └─────┬──────┘     └──────┬───────┘
   │            │               │                   │
   │ 1.点击登录  │               │                   │
   │───────────>│               │                   │
   │            │               │                   │
   │            │ 2.重定向（scope 包含 openid）       │
   │<───────────│──────────────>│                   │
   │            │               │                   │
   │ 3.用户登录并同意             │                   │
   │───────────────────────────>│                   │
   │            │               │                   │
   │            │ 4.返回授权码   │                   │
   │<───────────│<──────────────│                   │
   │            │               │                   │
   │            │ 5.换 Token    │                   │
   │            │──────────────>│                   │
   │            │               │                   │
   │            │ 6.返回 Access Token + ID Token    │
   │            │<──────────────│                   │
   │            │               │                   │
   │            │ 7.验证 ID Token                   │
   │            │──┐            │                   │
   │            │<─┘            │                   │
   │            │               │                   │
   │            │ 8.(可选) 获取更多用户信息          │
   │            │───────────────│──────────────────>│
   │            │               │                   │
```

### 关键差异点

```text
与纯 OAuth 的区别：

1. scope 必须包含 "openid"
   scope=openid profile email

2. Token 响应包含 ID Token
   {
     "access_token": "...",
     "token_type": "Bearer",
     "id_token": "eyJhbGciOiJSUzI1NiI...",  ← 这是新的
     "expires_in": 3600
   }

3. 必须验证 ID Token
   - 验证签名
   - 验证 aud == 你的 client_id
   - 验证 iss == 预期的提供商
   - 验证 exp > 当前时间
```

## Rust 实现 OIDC 客户端

```rust
use jsonwebtoken::{decode, decode_header, DecodingKey, Validation, Algorithm};
use serde::{Deserialize, Serialize};
use reqwest::Client;

// aud 字段可能是字符串或数组
#[derive(Debug, Deserialize)]
#[serde(untagged)]
enum StringOrArray {
    String(String),
    Array(Vec<String>),
}

// ID Token 的 Claims
#[derive(Debug, Deserialize)]
struct IdTokenClaims {
    iss: String,           // 颁发者
    sub: String,           // 用户唯一标识
    aud: StringOrArray,    // 受众（可能是单个 client_id 或数组）
    exp: usize,            // 过期时间
    iat: usize,            // 颁发时间
    nonce: Option<String>, // 防重放
    name: Option<String>,
    email: Option<String>,
    picture: Option<String>,
}

// OIDC Discovery 文档
#[derive(Debug, Deserialize)]
struct OidcDiscovery {
    issuer: String,
    authorization_endpoint: String,
    token_endpoint: String,
    userinfo_endpoint: String,
    jwks_uri: String,
}

// JWK（JSON Web Key）
#[derive(Debug, Deserialize)]
struct Jwks {
    keys: Vec<Jwk>,
}

#[derive(Debug, Deserialize)]
struct Jwk {
    kid: String,
    kty: String,
    n: String,  // RSA modulus
    e: String,  // RSA exponent
}

const CLIENT_ID: &str = "your-client-id";
const ISSUER: &str = "https://accounts.google.com";

async fn verify_id_token(id_token: &str) -> Result<IdTokenClaims, String> {
    // 1. 获取 OIDC Discovery 文档
    let discovery_url = format!("{}/.well-known/openid-configuration", ISSUER);
    let client = Client::new();
    let discovery: OidcDiscovery = client
        .get(&discovery_url)
        .send()
        .await
        .map_err(|e| e.to_string())?
        .json()
        .await
        .map_err(|e| e.to_string())?;

    // 2. 获取 JWKS（公钥集）
    let jwks: Jwks = client
        .get(&discovery.jwks_uri)
        .send()
        .await
        .map_err(|e| e.to_string())?
        .json()
        .await
        .map_err(|e| e.to_string())?;

    // 3. 解析 JWT Header 获取 kid
    let header = decode_header(id_token).map_err(|e| e.to_string())?;
    let kid = header.kid.ok_or("Missing kid in header")?;

    // 4. 找到对应的公钥
    let jwk = jwks
        .keys
        .iter()
        .find(|k| k.kid == kid)
        .ok_or("Key not found")?;

    // 5. 构建验证配置
    let mut validation = Validation::new(Algorithm::RS256);
    validation.set_audience(&[CLIENT_ID]);
    validation.set_issuer(&[ISSUER]);

    // 6. 验证并解码 ID Token
    // 注意：实际使用时需要 base64 解码 n 和 e，这里简化处理
    let decoding_key = DecodingKey::from_rsa_raw_components(
        &base64_decode(&jwk.n)?,
        &base64_decode(&jwk.e)?,
    ).map_err(|e| e.to_string())?;

    let token_data = decode::<IdTokenClaims>(id_token, &decoding_key, &validation)
        .map_err(|e| e.to_string())?;

    Ok(token_data.claims)
}

// 使用示例
async fn handle_oidc_callback(id_token: &str) -> Result<String, String> {
    let claims = verify_id_token(id_token).await?;

    // 现在你可以安全地使用用户信息了
    Ok(format!(
        "欢迎 {}！你的用户 ID 是 {}",
        claims.name.unwrap_or_else(|| "用户".to_string()),
        claims.sub
    ))
}
```

## OIDC Discovery：自动化配置

**OIDC 提供商必须提供一个标准化的配置端点。**

```text
Discovery URL 格式：
{issuer}/.well-known/openid-configuration

例如：
https://accounts.google.com/.well-known/openid-configuration
```

### Discovery 文档内容

```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "scopes_supported": ["openid", "email", "profile"],
  "response_types_supported": ["code", "token", "id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "claims_supported": ["sub", "name", "email", "picture", "locale"]
}
```

### 自动化的好处

```text
无需硬编码：
  ✓ 自动获取授权端点
  ✓ 自动获取 Token 端点
  ✓ 自动获取公钥位置
  ✓ 自动获取支持的算法

切换提供商只需改 issuer：
  Google:   https://accounts.google.com
  微软:     https://login.microsoftonline.com/{tenant}/v2.0
  Auth0:    https://{domain}.auth0.com
```

## 标准化的 scope 和 claims

### 标准 scope

```text
┌──────────────┬───────────────────────────────────────────┐
│ scope        │ 包含的 claims                             │
├──────────────┼───────────────────────────────────────────┤
│ openid       │ sub（必须）                               │
│ profile      │ name, family_name, given_name, picture,  │
│              │ nickname, preferred_username, gender,    │
│              │ birthdate, locale, updated_at            │
│ email        │ email, email_verified                    │
│ phone        │ phone_number, phone_number_verified      │
│ address      │ address（结构化地址）                     │
└──────────────┴───────────────────────────────────────────┘

请求示例：
scope=openid profile email
```

### 自定义 claims

```text
提供商可以添加自定义 claims：

Google:
  - hd: 企业域名（G Suite）
  - locale: 语言偏好

微信:
  - unionid: 跨应用用户标识
  - openid: 应用内用户标识
```

## nonce：防止重放攻击

### 为什么需要 nonce？

```text
重放攻击场景：

1. 攻击者截获了一个有效的 ID Token
2. 攻击者把这个 Token 发给你的后端
3. Token 签名有效，验证通过
4. 攻击者登录成功！

问题：你无法区分"新登录"和"重放的旧 Token"
```

### nonce 如何工作

```text
正常流程：

1. 用户点击登录
   → App 生成随机 nonce，存入 Session

2. 授权请求带上 nonce
   /authorize?...&nonce=random123

3. 提供商把 nonce 放入 ID Token
   { "sub": "...", "nonce": "random123", ... }

4. 回调时验证
   → ID Token 里的 nonce == Session 里的 nonce ?
   → 验证后从 Session 删除 nonce

重放攻击时：

1. 攻击者拿着旧的 ID Token
2. Token 里的 nonce = "old_random"
3. 受害者的 Session 里没有这个 nonce（或有不同的 nonce）
4. 验证失败！
```

### Rust 实现

```rust
use uuid::Uuid;

// 生成 nonce
fn generate_nonce(session: &mut Session) -> String {
    let nonce = Uuid::new_v4().to_string();
    session.insert("oidc_nonce", &nonce);
    nonce
}

// 验证 nonce
fn verify_nonce(id_token_nonce: Option<&str>, session: &mut Session) -> bool {
    let stored_nonce = session.get::<String>("oidc_nonce");

    match (id_token_nonce, stored_nonce) {
        (Some(token_nonce), Some(stored)) if token_nonce == stored => {
            session.remove("oidc_nonce"); // 用后即删
            true
        }
        _ => false,
    }
}

// 在验证 ID Token 时
async fn handle_callback(id_token: &str, session: &mut Session) -> Result<User, Error> {
    let claims = verify_id_token(id_token).await?;

    // 验证 nonce
    if !verify_nonce(claims.nonce.as_deref(), session) {
        return Err(Error::InvalidNonce);
    }

    // 继续处理...
    Ok(User::from(claims))
}
```

## OIDC vs OAuth：何时用哪个

```text
┌─────────────────────┬────────────────────────────────────────┐
│ 场景                 │ 推荐方案                               │
├─────────────────────┼────────────────────────────────────────┤
│ 第三方登录           │ OIDC                                   │
│ (微信登录、Google登录)│ 需要知道"用户是谁"                      │
├─────────────────────┼────────────────────────────────────────┤
│ 单点登录 (SSO)       │ OIDC                                   │
│                     │ 一次登录，多处使用                       │
├─────────────────────┼────────────────────────────────────────┤
│ 访问第三方 API       │ OAuth 2.0                              │
│ (不需要用户身份)      │ 只需要"能访问"，不关心"谁在访问"         │
├─────────────────────┼────────────────────────────────────────┤
│ 服务间通信           │ OAuth 2.0 (Client Credentials)        │
│                     │ 没有用户参与                            │
└─────────────────────┴────────────────────────────────────────┘
```

## 安全最佳实践

### 1. 完整验证 ID Token

```rust
fn validate_id_token(claims: &IdTokenClaims) -> Result<(), Error> {
    // 1. 验证 issuer
    if claims.iss != EXPECTED_ISSUER {
        return Err(Error::InvalidIssuer);
    }

    // 2. 验证 audience（可能是字符串或数组）
    let aud_valid = match &claims.aud {
        StringOrArray::String(s) => s == CLIENT_ID,
        StringOrArray::Array(arr) => arr.contains(&CLIENT_ID.to_string()),
    };
    if !aud_valid {
        return Err(Error::InvalidAudience);
    }

    // 3. 验证过期时间
    let now = chrono::Utc::now().timestamp() as usize;
    if claims.exp < now {
        return Err(Error::TokenExpired);
    }

    // 4. 验证颁发时间（可选，防止时钟偏差攻击）
    if claims.iat > now + 300 {  // 允许 5 分钟偏差
        return Err(Error::InvalidIssuedAt);
    }

    // 5. 验证 nonce（在前面处理）

    Ok(())
}
```

### 2. 使用 PKCE（即使是机密客户端）

```text
OAuth 2.1 草案已经要求所有客户端使用 PKCE

即使你有 client_secret，也建议使用 PKCE：
  - 多层防御
  - 防止未来的攻击向量
  - 符合最新安全标准
```

### 3. 定期轮换公钥

```text
提供商会定期轮换 JWKS 中的密钥

你的应用应该：
  - 缓存 JWKS（避免每次请求）
  - 设置合理的缓存时间（如 1 小时）
  - 当 kid 找不到时，刷新缓存
```

### 4. 正确处理用户标识

```rust
// 正确：使用 (iss, sub) 组合作为唯一标识
fn get_user_unique_id(claims: &IdTokenClaims) -> String {
    format!("{}|{}", claims.iss, claims.sub)
}

// 错误：只用 sub
// 不同提供商的 sub 可能冲突！
fn get_user_unique_id_wrong(claims: &IdTokenClaims) -> String {
    claims.sub.clone()  // 危险！
}
```

## 核心认知

> **OIDC 的本质**：在 OAuth 2.0 的授权流程上，增加一个标准化的身份层。

**一句话理解**：
- OAuth 2.0 回答"能做什么"
- OIDC 回答"你是谁"

**关键组件**：
- ID Token：用户身份的加密证明
- Discovery：自动化配置
- UserInfo：更多用户信息的端点
- 标准化的 scope 和 claims

**安全要点**：
- 验证签名、iss、aud、exp
- 使用 nonce 防重放
- 使用 state 防 CSRF
- (iss, sub) 组合作为用户唯一标识

## 引出下一篇

到目前为止，我们讨论的认证方式都有一个共同点：**依赖密码或密码的替代品（Token）**。

- Session：登录时输密码
- JWT：登录时输密码
- OAuth/OIDC：在第三方输密码

但密码有天然的安全问题：
- 可以被钓鱼
- 可以被暴力破解
- 可以被泄露

**有没有一种认证方式，从根本上不需要密码？**

下一篇，我们来看 **Passkey**——用公钥密码学彻底告别密码。

---

## 常见问题

### Q：ID Token 和 Access Token 能互相替代吗？

**A：绝对不能。**

```text
错误 1：用 Access Token 做身份验证
  - Access Token 是给资源服务器的
  - 可能不包含用户信息
  - 你不知道它是不是颁发给你的

错误 2：用 ID Token 访问 API
  - ID Token 是给 Client 的
  - 资源服务器不应该接受 ID Token
  - 违反职责分离原则
```

### Q：ID Token 应该存储在哪里？

**A：用完即弃，不需要存储。**

```text
ID Token 的用途：
  - 验证用户身份
  - 获取用户信息
  - 一次性使用

验证完成后：
  - 创建自己的 Session 或 JWT
  - 不需要保留 ID Token
  - ID Token 很快就会过期
```

### Q：如果提供商不支持 OIDC 怎么办？

**A：退而求其次，但要谨慎。**

```text
有些提供商只支持 OAuth 2.0（如某些国内平台）

安全措施：
1. 用 Access Token 调用 /userinfo 或类似端点
2. 验证返回的用户信息
3. 使用 state 防 CSRF
4. 在文档中标明这不是标准 OIDC

风险：
  - 没有 ID Token 的签名验证
  - 需要信任 userinfo 端点的 HTTPS
```

---

*下一篇：[Passkey——告别密码](/posts/auth-evolution-5-passkey/)*

*本系列：*
1. *[Session——服务器说"我记得你"](/posts/auth-evolution-1-session/)*
2. *[JWT——把状态装进令牌](/posts/auth-evolution-2-jwt/)*
3. *[OAuth 2.0——让别人帮你认证](/posts/auth-evolution-3-oauth/)*
4. *OpenID Connect——标准化的身份层（本篇）*
5. *[Passkey——告别密码](/posts/auth-evolution-5-passkey/)*
6. *[实战：设计一个现代认证系统](/posts/auth-evolution-6-practice/)*
