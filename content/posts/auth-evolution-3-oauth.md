---
title: "认证演进史（三）：OAuth 2.0——让别人帮你认证"
date: 2025-12-08T23:30:00+08:00
draft: false
tags: ["认证", "OAuth", "授权", "Web安全"]
categories: ["技术"]
description: "你想让用户用微信登录你的 App，但你不想（也不应该）知道他的微信密码。怎么办？"
series: ["认证演进史"]
---

你想让用户用微信登录你的 App，但你不想（也不应该）知道他的微信密码。怎么办？

这就是 **OAuth 2.0** 要解决的问题。

<!--more-->

## 为什么需要第三方登录？

### 用户的痛点

```text
场景：你发现了一个新 App

传统方式：
1. 注册账号（又要想一个密码...）
2. 验证邮箱（打开邮箱，找邮件，点链接...）
3. 填写资料（头像、昵称、手机号...）
4. 终于可以用了（已经过去 10 分钟）

第三方登录：
1. 点击"微信登录"
2. 微信弹窗确认
3. 完成（3 秒）
```

**用户不想再记一个密码。**

### 开发者的痛点

```text
自己做认证系统：
- 密码存储安全（上一个系列讲的）
- 忘记密码流程
- 账号安全（异地登录检测、二次验证）
- 防止暴力破解
- 防止撞库攻击
- ...

用第三方登录：
- 让微信/Google 帮你搞定这些
- 你只需要拿到用户身份
```

**开发者不想重复造轮子。**

### 但有一个问题

```text
危险的方式：

用户：我想用微信登录你的 App
App：好的，请把微信密码给我
用户：???

这不行：
1. 用户不信任你（凭什么给你密码）
2. App 权力过大（拿到密码什么都能干）
3. 无法撤销（密码给出去了就收不回来）
```

我们需要一种方式：**让用户授权 App 获取他的信息，但不需要给出密码。**

## OAuth 2.0 的核心思想

**OAuth 的本质**：不是认证（Authentication），而是**授权（Authorization）**。

```text
类比：酒店门卡

传统钥匙（密码）：
- 拥有者可以进入所有房间
- 无法限制权限
- 无法远程撤销

酒店门卡（OAuth）：
- 只能进入指定房间（权限范围）
- 有有效期（过期时间）
- 可以随时作废（撤销授权）
- 不暴露主钥匙（用户密码）
```

### 四个角色

```text
┌─────────────────────────────────────────────────────────────┐
│                      OAuth 2.0 角色                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Resource Owner（资源拥有者）                                 │
│  └─ 就是用户本人，拥有微信账号里的数据                         │
│                                                             │
│  Client（客户端）                                            │
│  └─ 就是你的 App，想要访问用户在微信的数据                     │
│                                                             │
│  Authorization Server（授权服务器）                          │
│  └─ 微信的授权服务，负责验证用户身份、颁发令牌                  │
│                                                             │
│  Resource Server（资源服务器）                               │
│  └─ 微信的 API 服务器，存储用户头像、昵称等数据                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**类比理解**：

| 角色 | 类比 | 说明 |
|------|------|------|
| Resource Owner | 房主 | 房子是他的，他说了算 |
| Client | 快递员 | 想进门放个包裹 |
| Authorization Server | 物业前台 | 核实房主身份，发放临时通行证 |
| Resource Server | 门禁系统 | 验证通行证，决定是否放行 |

## 授权码模式（Authorization Code）

这是**最安全**、**最常用**的模式，几乎所有"微信登录"、"Google 登录"都用它。

### 完整流程

```text
┌──────┐     ┌──────┐     ┌────────────┐     ┌──────────────┐
│ 用户  │     │ App  │     │ 授权服务器  │     │ 资源服务器    │
└──┬───┘     └──┬───┘     └─────┬──────┘     └──────┬───────┘
   │            │               │                   │
   │ 1.点击登录  │               │                   │
   │───────────>│               │                   │
   │            │               │                   │
   │            │ 2.重定向到授权页面                  │
   │<───────────│──────────────>│                   │
   │            │               │                   │
   │ 3.用户登录并同意授权         │                   │
   │───────────────────────────>│                   │
   │            │               │                   │
   │            │ 4.返回授权码   │                   │
   │<───────────│<──────────────│                   │
   │            │               │                   │
   │            │ 5.用授权码换令牌                   │
   │            │──────────────>│                   │
   │            │               │                   │
   │            │ 6.返回 Access Token               │
   │            │<──────────────│                   │
   │            │               │                   │
   │            │ 7.用令牌请求数据                   │
   │            │───────────────│──────────────────>│
   │            │               │                   │
   │            │ 8.返回用户数据                     │
   │            │<──────────────│───────────────────│
   │            │               │                   │
```

### 为什么需要"授权码"这个中间步骤？

```text
问：为什么不直接返回 Access Token？

直接返回 Token 的问题：

1. 用户被重定向回 App 时，URL 是这样的：
   https://your-app.com/callback?token=abc123

2. 这个 URL 会出现在：
   - 浏览器历史记录
   - 服务器访问日志
   - Referer 头
   - ...

3. Token 泄露了！

使用授权码的好处：

1. 授权码只能用一次，用完即失效
2. 授权码换 Token 是服务器到服务器，不经过浏览器
3. 换 Token 时需要 client_secret，攻击者没有
```

### Rust 实现示例

用 `axum` 实现 OAuth 2.0 客户端：

```rust
use axum::{
    Router,
    routing::get,
    response::Redirect,
    extract::Query,
};
use serde::{Deserialize, Serialize};
use reqwest::Client;

// 配置
const CLIENT_ID: &str = "your_client_id";
const CLIENT_SECRET: &str = "your_client_secret";
const REDIRECT_URI: &str = "https://your-app.com/callback";
const AUTH_URL: &str = "https://oauth.example.com/authorize";
const TOKEN_URL: &str = "https://oauth.example.com/token";

// 步骤 1：重定向到授权页面
async fn login() -> Redirect {
    let auth_url = format!(
        "{}?response_type=code&client_id={}&redirect_uri={}&scope=read:user&state={}",
        AUTH_URL,
        CLIENT_ID,
        urlencoding::encode(REDIRECT_URI),
        generate_state() // 防 CSRF
    );
    Redirect::to(&auth_url)
}

#[derive(Deserialize)]
struct CallbackParams {
    code: String,
    state: String,
}

#[derive(Deserialize)]
struct TokenResponse {
    access_token: String,
    token_type: String,
    expires_in: u64,
    refresh_token: Option<String>,
}

// 步骤 2：处理回调，用授权码换令牌
async fn callback(Query(params): Query<CallbackParams>) -> String {
    // 验证 state（防 CSRF）
    if !verify_state(&params.state) {
        return "Invalid state".to_string();
    }

    // 用授权码换 Access Token
    let client = Client::new();
    let token_response = client
        .post(TOKEN_URL)
        .form(&[
            ("grant_type", "authorization_code"),
            ("code", &params.code),
            ("redirect_uri", REDIRECT_URI),
            ("client_id", CLIENT_ID),
            ("client_secret", CLIENT_SECRET),
        ])
        .send()
        .await
        .unwrap()
        .json::<TokenResponse>()
        .await
        .unwrap();

    // 用 Token 获取用户信息
    let user_info = client
        .get("https://api.example.com/user")
        .bearer_auth(&token_response.access_token)
        .send()
        .await
        .unwrap()
        .text()
        .await
        .unwrap();

    format!("登录成功！用户信息：{}", user_info)
}

fn generate_state() -> String {
    // 生成随机字符串，存入 Session
    uuid::Uuid::new_v4().to_string()
}

fn verify_state(state: &str) -> bool {
    // 验证 state 与 Session 中存储的是否一致
    true // 简化示例
}
```

## PKCE：移动端的救星

### 移动端的困境

```text
问题：移动 App 无法安全存储 client_secret

原因：
1. App 会被反编译
2. 代码里的任何字符串都可能被提取
3. client_secret 泄露 = 任何人都能冒充你的 App

传统方案：
- "那就不用 client_secret"
- 这叫"隐式模式"（Implicit Flow）
- 但这模式不安全，已被废弃
```

### PKCE（Proof Key for Code Exchange）

```text
PKCE 的思路：每次授权生成一个临时的"密钥对"

1. 客户端生成随机字符串 code_verifier
2. 计算 code_challenge = SHA256(code_verifier)
3. 授权请求带上 code_challenge
4. 换 Token 时带上 code_verifier
5. 授权服务器验证：SHA256(code_verifier) == code_challenge
```

### 流程图

```text
┌──────────────────────────────────────────────────────────────┐
│                     PKCE 流程                                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. App 生成 code_verifier（随机字符串）                       │
│     code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p..."        │
│                                                              │
│  2. 计算 code_challenge                                       │
│     code_challenge = BASE64URL(SHA256(code_verifier))        │
│                    = "E9Melhoa2OwvFrEMTJguCHaoeK1t..."        │
│                                                              │
│  3. 授权请求                                                  │
│     /authorize?                                              │
│       response_type=code&                                    │
│       client_id=xxx&                                         │
│       code_challenge=E9Melhoa...&                            │
│       code_challenge_method=S256                             │
│                                                              │
│  4. 换 Token 请求                                             │
│     POST /token                                              │
│       grant_type=authorization_code&                         │
│       code=授权码&                                            │
│       code_verifier=dBjftJeZ4CVP...  ← 原始值                 │
│                                                              │
│  5. 服务器验证                                                │
│     SHA256(code_verifier) == 之前的 code_challenge ?          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Rust 实现 PKCE

```rust
use base64::{engine::general_purpose::URL_SAFE_NO_PAD, Engine};
use rand::Rng;
use sha2::{Digest, Sha256};

struct PkceChallenge {
    verifier: String,
    challenge: String,
}

fn generate_pkce() -> PkceChallenge {
    // 生成 43-128 字符的随机字符串
    let verifier: String = rand::thread_rng()
        .sample_iter(&rand::distributions::Alphanumeric)
        .take(64)
        .map(char::from)
        .collect();

    // 计算 SHA256 并 Base64URL 编码
    let mut hasher = Sha256::new();
    hasher.update(verifier.as_bytes());
    let challenge = URL_SAFE_NO_PAD.encode(hasher.finalize());

    PkceChallenge { verifier, challenge }
}

// 使用示例
fn start_auth() {
    let pkce = generate_pkce();

    // 保存 verifier（后面换 Token 要用）
    save_to_session(&pkce.verifier);

    // 授权 URL
    let auth_url = format!(
        "{}?response_type=code&client_id={}&code_challenge={}&code_challenge_method=S256",
        AUTH_URL, CLIENT_ID, pkce.challenge
    );
}
```

## 其他授权模式

```text
┌────────────────────────────────────────────────────────────────────┐
│                        OAuth 2.0 授权模式对比                       │
├──────────────────┬─────────────────────────────────────────────────┤
│ 模式              │ 说明                                            │
├──────────────────┼─────────────────────────────────────────────────┤
│ 授权码模式        │ 最安全，适合有后端的 Web 应用                     │
│ (Authorization   │ 前端获取 code，后端换 token                       │
│  Code)           │                                                  │
├──────────────────┼─────────────────────────────────────────────────┤
│ 授权码 + PKCE    │ 授权码模式的增强版                                │
│                  │ 适合移动端、SPA 等无法安全存储密钥的场景           │
├──────────────────┼─────────────────────────────────────────────────┤
│ 客户端凭证模式    │ 用于服务器到服务器                                │
│ (Client          │ 没有用户参与，App 用自己的身份访问                 │
│  Credentials)    │ 例：微服务之间的 API 调用                         │
├──────────────────┼─────────────────────────────────────────────────┤
│ 隐式模式          │ ⚠️ 已废弃，不要使用                               │
│ (Implicit)       │ Token 直接返回给浏览器，不安全                    │
├──────────────────┼─────────────────────────────────────────────────┤
│ 密码模式          │ ⚠️ 仅限高度信任的第一方应用                       │
│ (Password)       │ 用户把密码给 App，App 代为认证                    │
│                  │ 例：同公司的官方 App                              │
└──────────────────┴─────────────────────────────────────────────────┘
```

## OAuth ≠ 认证

**这是最容易混淆的点。**

```text
OAuth 2.0 是授权协议，不是认证协议！

授权（Authorization）：
  "你可以访问我的照片"
  → 回答的是"能做什么"

认证（Authentication）：
  "你是张三"
  → 回答的是"你是谁"
```

### 用 OAuth 做"登录"的问题

```text
场景：用 OAuth 实现"微信登录"

你拿到了 Access Token，然后呢？

错误做法：
  "有 Token 就说明用户登录了"
  → 但你不知道这个 Token 是不是给你的！

攻击场景：
  1. 攻击者在恶意 App 里用微信登录
  2. 恶意 App 拿到攻击者的 Access Token
  3. 恶意 App 把这个 Token 发给你的后端
  4. 你的后端用这个 Token 查到攻击者的微信信息
  5. 你以为是正常登录，创建了账号
  6. 但这个 Token 不是"颁发给你的 App"的！
```

### 正确的做法

```text
方案 1：调用 /userinfo 并验证

  用 Token 调用授权服务器的 /userinfo 接口
  验证返回的用户 ID
  确保 Token 是有效的

方案 2：使用 OpenID Connect（推荐）

  这是下一篇的主题
  专门为认证设计的协议
  在 OAuth 2.0 基础上增加身份层
```

## 安全最佳实践

### 1. 始终验证 state 参数

```rust
// 生成 state
fn generate_state() -> String {
    let state = uuid::Uuid::new_v4().to_string();
    // 存入 Session 或 Redis
    session.insert("oauth_state", &state);
    state
}

// 验证 state
fn verify_state(received: &str, session: &Session) -> bool {
    let stored = session.get::<String>("oauth_state");
    match stored {
        Some(s) if s == received => {
            session.remove("oauth_state"); // 用后即删
            true
        }
        _ => false,
    }
}
```

**不验证 state 的后果**：CSRF 攻击，攻击者可以让你绑定到他的账号。

### 2. 始终使用 HTTPS

```text
HTTP 下的风险：

1. 授权码在 URL 里，可能被中间人截获
2. Token 在请求头里，可能被窃取
3. 所有 OAuth 流程都可能被劫持
```

### 3. 最小权限原则

```text
错误示范：
scope=read write delete admin  ← 要了所有权限

正确示范：
scope=read:user read:email     ← 只要需要的
```

### 4. 安全存储 Token

```text
Web 应用：
  - Access Token：内存或 HttpOnly Cookie
  - Refresh Token：HttpOnly + Secure Cookie

移动应用：
  - iOS：Keychain
  - Android：EncryptedSharedPreferences
```

### 5. 正确处理 Token 过期

```rust
async fn call_api_with_refresh(
    access_token: &str,
    refresh_token: &str,
) -> Result<Response, Error> {
    // 先尝试用 Access Token
    let response = api_client
        .get("/resource")
        .bearer_auth(access_token)
        .send()
        .await?;

    if response.status() == 401 {
        // Token 过期，刷新
        let new_tokens = refresh_access_token(refresh_token).await?;

        // 用新 Token 重试
        return api_client
            .get("/resource")
            .bearer_auth(&new_tokens.access_token)
            .send()
            .await;
    }

    Ok(response)
}
```

## 核心认知

> **OAuth 2.0 的本质**：让用户在不暴露密码的情况下，授权第三方应用访问他的资源。

**OAuth 是什么**：
- 授权框架（Authorization Framework）
- 解决"第三方应用访问用户数据"的问题
- 通过令牌代替密码

**OAuth 不是什么**：
- 不是认证协议（不能回答"你是谁"）
- 不是单点登录方案
- 不是安全的银弹

**关键安全点**：
- 使用授权码模式 + PKCE
- 验证 state 防 CSRF
- 最小权限 scope
- HTTPS 全程加密

## 引出下一篇

OAuth 2.0 解决了授权问题，但我们想做的是**登录**。

用 OAuth 做登录会遇到我们讲的问题：你不知道 Token 是不是颁发给你的。

**OpenID Connect（OIDC）** 在 OAuth 2.0 基础上，增加了标准化的身份层：

- **ID Token**：告诉你"用户是谁"
- **标准化的用户信息端点**
- **标准化的发现机制**

下一篇，我们来看 OIDC 如何让"第三方登录"变得安全且标准。

---

## 常见问题

### Q：为什么授权码只能用一次？

**A：防止重放攻击。**

如果授权码可以重复使用，攻击者截获后可以换取多个 Token。一次性使用确保即使泄露，攻击者也来不及利用。

### Q：Access Token 和 Refresh Token 为什么要分开？

**A：安全性和可用性的平衡。**

| Token 类型 | 有效期 | 用途 | 暴露风险 |
|-----------|--------|------|----------|
| Access Token | 短（分钟-小时） | 访问 API | 每次请求都带，风险高 |
| Refresh Token | 长（天-月） | 换新的 Access Token | 只在刷新时用，风险低 |

Access Token 短期有效，即使泄露影响也有限。Refresh Token 长期有效，但只在安全的后端使用。

### Q：state 参数具体防止什么攻击？

**A：CSRF（跨站请求伪造）。**

```text
攻击场景（没有 state）：

1. 攻击者在自己的浏览器完成 OAuth 到第 4 步
   拿到指向你的 App 的授权码 URL
2. 攻击者把这个 URL 发给受害者
3. 受害者点击，浏览器访问你的 App
4. 你的 App 用授权码换 Token
5. 受害者的账号绑定了攻击者的微信！

有 state 的情况：

1. 每次授权开始时生成随机 state
2. 回调时验证 state 是否匹配
3. 攻击者的 state 和受害者的不一样
4. 攻击失败
```

---

*下一篇：[OpenID Connect——标准化的身份层](/posts/auth-evolution-4-oidc/)*

*本系列：*
1. *[Session——服务器说"我记得你"](/posts/auth-evolution-1-session/)*
2. *[JWT——把状态装进令牌](/posts/auth-evolution-2-jwt/)*
3. *OAuth 2.0——让别人帮你认证（本篇）*
4. *[OpenID Connect——标准化的身份层](/posts/auth-evolution-4-oidc/)*
5. *[Passkey——告别密码](/posts/auth-evolution-5-passkey/)*
6. *[实战：设计一个现代认证系统](/posts/auth-evolution-6-practice/)*
