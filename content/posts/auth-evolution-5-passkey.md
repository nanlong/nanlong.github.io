---
title: "认证演进史（五）：Passkey——告别密码"
date: 2025-12-08T23:50:00+08:00
draft: false
tags: ["认证", "Passkey", "WebAuthn", "FIDO2", "无密码"]
categories: ["技术"]
description: "密码的问题不在于'密码不够复杂'，而在于'密码这个概念本身'。Passkey 用公钥密码学，从根本上消灭密码。"
series: ["认证演进史"]
---

我们在密码安全系列讲过各种保护密码的方法：加盐哈希、慢哈希、Argon2...

但所有这些努力都在解决一个问题：**密码泄露后怎么办。**

有没有可能从根本上解决问题？**让密码不存在？**

这就是 **Passkey**。

<!--more-->

## 密码的根本问题

### 密码为什么不安全？

```text
密码的本质：一个"共享的秘密"

注册时：
  你 → 服务器："我的密码是 123456"
  服务器存下来（哈希后）

登录时：
  你 → 服务器："我的密码是 123456"
  服务器验证：哈希值匹配？

问题：
  这个秘密必须"传输"和"存储"
  传输时可能被窃听
  存储时可能被拖库
```

### 密码的攻击面

```text
┌────────────────────────────────────────────────────────────┐
│                    密码的攻击面                              │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  1. 钓鱼攻击                                                │
│     假网站骗你输入密码                                       │
│     → 密码被盗                                              │
│                                                            │
│  2. 暴力破解                                                │
│     不断尝试密码组合                                         │
│     → 弱密码被破解                                          │
│                                                            │
│  3. 撞库攻击                                                │
│     用其他网站泄露的密码尝试                                  │
│     → 密码复用被利用                                         │
│                                                            │
│  4. 数据库泄露                                              │
│     服务器被入侵                                             │
│     → 密码哈希被盗，可离线破解                               │
│                                                            │
│  5. 社会工程学                                              │
│     骗你说出密码                                             │
│     → 人是最弱的环节                                         │
│                                                            │
│  6. 键盘记录 / 屏幕录制                                     │
│     恶意软件捕获输入                                         │
│     → 密码被窃取                                            │
│                                                            │
└────────────────────────────────────────────────────────────┘

根本原因：密码是"知识因素"
  只要"知道"密码就能登录
  而"知道"这件事，无法被独占
```

## 公钥密码学：反转思路

### 核心思想

```text
传统密码（对称）：
  你和服务器共享同一个秘密
  谁有这个秘密，谁就能登录

公钥密码学（非对称）：
  你有私钥，服务器有公钥
  私钥永远不离开你的设备
  服务器用公钥验证你的身份

关键区别：
  私钥不需要传输
  私钥不需要让服务器知道
  即使服务器被攻破，私钥也是安全的
```

### 认证流程

```text
┌─────────────────────────────────────────────────────────────┐
│              公钥认证流程                                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  注册：                                                      │
│    1. 设备生成密钥对（私钥 + 公钥）                           │
│    2. 私钥存在设备的安全区域                                  │
│    3. 公钥发给服务器保存                                      │
│                                                             │
│  登录：                                                      │
│    1. 服务器发送随机挑战（Challenge）                         │
│    2. 设备用私钥签名挑战                                      │
│    3. 服务器用公钥验证签名                                    │
│    4. 签名有效 → 认证成功                                    │
│                                                             │
│  安全性：                                                    │
│    ✓ 私钥从不离开设备                                        │
│    ✓ 服务器不存储任何敏感信息（公钥是公开的）                  │
│    ✓ 每次挑战不同，无法重放                                   │
│    ✓ 没有密码可以钓鱼                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## WebAuthn 和 FIDO2

**Passkey 不是凭空出现的，它基于成熟的标准。**

### 标准族谱

```text
FIDO（Fast IDentity Online）联盟制定了无密码认证标准

FIDO2 = WebAuthn + CTAP

┌────────────────────────────────────────────────────────────┐
│                      FIDO2 架构                             │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  WebAuthn（W3C 标准）                                       │
│  └─ 浏览器/应用 与 服务器 之间的 API                          │
│  └─ JavaScript API：navigator.credentials.create/get       │
│                                                            │
│  CTAP（Client to Authenticator Protocol）                  │
│  └─ 浏览器/应用 与 认证器 之间的协议                          │
│  └─ 认证器：指纹、Face ID、YubiKey、手机...                  │
│                                                            │
└────────────────────────────────────────────────────────────┘

Passkey = FIDO2 的用户友好品牌名
```

### 认证器类型

```text
┌─────────────────┬────────────────────────────────────────────┐
│ 类型             │ 说明                                        │
├─────────────────┼────────────────────────────────────────────┤
│ 平台认证器       │ 内置于设备                                   │
│ (Platform)      │ 如：MacBook 的 Touch ID，iPhone 的 Face ID  │
│                 │ 优点：方便，用户已有                          │
│                 │ 缺点：换设备需要迁移                          │
├─────────────────┼────────────────────────────────────────────┤
│ 漫游认证器       │ 独立设备                                     │
│ (Roaming)       │ 如：YubiKey，手机作为认证器                  │
│                 │ 优点：跨设备使用                              │
│                 │ 缺点：需要携带，可能丢失                      │
├─────────────────┼────────────────────────────────────────────┤
│ 同步 Passkey    │ 云端同步的 Passkey                           │
│ (Synced)        │ 如：iCloud Keychain，Google Password Manager│
│                 │ 优点：跨设备无缝使用                          │
│                 │ 缺点：依赖云服务商的安全性                    │
└─────────────────┴────────────────────────────────────────────┘
```

## WebAuthn 流程详解

### 注册流程（创建 Passkey）

```text
┌──────┐        ┌─────────┐        ┌────────┐        ┌────────┐
│ 用户  │        │ 浏览器   │        │ 认证器  │        │ 服务器  │
└──┬───┘        └────┬────┘        └───┬────┘        └───┬────┘
   │                 │                 │                 │
   │ 1. 点击注册      │                 │                 │
   │────────────────>│                 │                 │
   │                 │                 │                 │
   │                 │ 2. 请求注册选项                    │
   │                 │────────────────────────────────>│
   │                 │                 │                 │
   │                 │ 3. 返回 challenge + 配置          │
   │                 │<────────────────────────────────│
   │                 │                 │                 │
   │                 │ 4. 调用 WebAuthn API              │
   │                 │───────────────>│                 │
   │                 │                 │                 │
   │ 5. 用户验证（指纹/Face ID）        │                 │
   │<─────────────────────────────────│                 │
   │────────────────────────────────>│                 │
   │                 │                 │                 │
   │                 │ 6. 返回公钥 + 签名                │
   │                 │<───────────────│                 │
   │                 │                 │                 │
   │                 │ 7. 发送凭证到服务器                │
   │                 │────────────────────────────────>│
   │                 │                 │                 │
   │                 │ 8. 验证并存储公钥                  │
   │                 │<────────────────────────────────│
   │                 │                 │                 │
```

### 前端 JavaScript 代码

```javascript
// 注册 Passkey
async function registerPasskey() {
    // 1. 从服务器获取注册选项
    const optionsResponse = await fetch('/api/webauthn/register/options', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ username: 'alice' })
    });
    const options = await optionsResponse.json();

    // 2. 转换为 WebAuthn 格式
    options.challenge = base64ToArrayBuffer(options.challenge);
    options.user.id = base64ToArrayBuffer(options.user.id);

    // 3. 调用 WebAuthn API 创建凭证
    const credential = await navigator.credentials.create({
        publicKey: options
    });

    // 4. 发送凭证到服务器
    const attestationResponse = {
        id: credential.id,
        rawId: arrayBufferToBase64(credential.rawId),
        type: credential.type,
        response: {
            clientDataJSON: arrayBufferToBase64(
                credential.response.clientDataJSON
            ),
            attestationObject: arrayBufferToBase64(
                credential.response.attestationObject
            )
        }
    };

    await fetch('/api/webauthn/register/verify', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(attestationResponse)
    });
}

// 登录
async function loginWithPasskey() {
    // 1. 获取认证选项
    const optionsResponse = await fetch('/api/webauthn/login/options', {
        method: 'POST'
    });
    const options = await optionsResponse.json();

    options.challenge = base64ToArrayBuffer(options.challenge);
    options.allowCredentials = options.allowCredentials.map(cred => ({
        ...cred,
        id: base64ToArrayBuffer(cred.id)
    }));

    // 2. 获取凭证
    const assertion = await navigator.credentials.get({
        publicKey: options
    });

    // 3. 发送给服务器验证
    const assertionResponse = {
        id: assertion.id,
        rawId: arrayBufferToBase64(assertion.rawId),
        type: assertion.type,
        response: {
            clientDataJSON: arrayBufferToBase64(
                assertion.response.clientDataJSON
            ),
            authenticatorData: arrayBufferToBase64(
                assertion.response.authenticatorData
            ),
            signature: arrayBufferToBase64(
                assertion.response.signature
            )
        }
    };

    await fetch('/api/webauthn/login/verify', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(assertionResponse)
    });
}
```

### Rust 后端实现

使用 `webauthn-rs` 库：

```rust
use webauthn_rs::prelude::*;
use axum::{Router, routing::post, Json, Extension};
use std::sync::Arc;
use tokio::sync::RwLock;
use std::collections::HashMap;

// 用户凭证存储（实际应用中应该用数据库）
type CredentialStore = Arc<RwLock<HashMap<String, Vec<Passkey>>>>;

// 注册会话存储
type RegistrationStore = Arc<RwLock<HashMap<String, PasskeyRegistration>>>;

// 认证会话存储
type AuthenticationStore = Arc<RwLock<HashMap<String, PasskeyAuthentication>>>;

struct AppState {
    webauthn: Webauthn,
    credentials: CredentialStore,
    registrations: RegistrationStore,
    authentications: AuthenticationStore,
}

// 注册步骤 1：生成选项
async fn register_start(
    Extension(state): Extension<Arc<AppState>>,
    Json(req): Json<RegisterRequest>,
) -> Json<CreationChallengeResponse> {
    let user_id = Uuid::new_v4();

    // 生成注册挑战
    let (ccr, reg_state) = state
        .webauthn
        .start_passkey_registration(
            user_id,
            &req.username,
            &req.username,
            None, // 不排除已有凭证
        )
        .expect("Failed to start registration");

    // 保存注册状态
    state.registrations
        .write()
        .await
        .insert(req.username.clone(), reg_state);

    Json(ccr)
}

// 注册步骤 2：验证并存储
async fn register_finish(
    Extension(state): Extension<Arc<AppState>>,
    Json(req): Json<RegisterFinishRequest>,
) -> Json<RegisterResponse> {
    // 获取注册状态
    let reg_state = state.registrations
        .write()
        .await
        .remove(&req.username)
        .expect("Registration not found");

    // 验证并获取 Passkey
    let passkey = state
        .webauthn
        .finish_passkey_registration(&req.credential, &reg_state)
        .expect("Failed to finish registration");

    // 存储凭证
    state.credentials
        .write()
        .await
        .entry(req.username.clone())
        .or_default()
        .push(passkey);

    Json(RegisterResponse { success: true })
}

// 登录步骤 1：生成挑战
async fn login_start(
    Extension(state): Extension<Arc<AppState>>,
    Json(req): Json<LoginRequest>,
) -> Json<RequestChallengeResponse> {
    // 获取用户的凭证
    let credentials = state.credentials
        .read()
        .await
        .get(&req.username)
        .cloned()
        .unwrap_or_default();

    // 生成认证挑战
    let (rcr, auth_state) = state
        .webauthn
        .start_passkey_authentication(&credentials)
        .expect("Failed to start authentication");

    // 保存认证状态（关联用户名）
    state.authentications
        .write()
        .await
        .insert(req.username.clone(), auth_state);

    Json(rcr)
}

// 登录步骤 2：验证签名
async fn login_finish(
    Extension(state): Extension<Arc<AppState>>,
    Json(req): Json<LoginFinishRequest>,
) -> Json<LoginResponse> {
    // 获取认证状态
    let auth_state = state.authentications
        .write()
        .await
        .remove(&req.username)
        .expect("Authentication session not found");

    // 验证签名
    let auth_result = state
        .webauthn
        .finish_passkey_authentication(&req.credential, &auth_state)
        .expect("Authentication failed");

    // 更新凭证计数器（防止克隆攻击）
    // update_credential_counter(&auth_result);

    Json(LoginResponse {
        success: true,
        username: req.username,
    })
}
```

## Passkey 如何防止攻击

### 防钓鱼

```text
传统密码：
  用户可能在假网站输入密码
  假网站获得密码后可以在真网站使用

Passkey：
  认证器验证网站来源（origin）
  假网站 phishing.com 发起的认证
  签名会包含 phishing.com 作为 origin
  真网站 bank.com 验证时会拒绝
  因为 origin 不匹配！

技术实现：
  clientDataJSON 包含：
  {
    "type": "webauthn.get",
    "challenge": "...",
    "origin": "https://bank.com",  ← 认证器填入
    "crossOrigin": false
  }

  服务器验证 origin 是否是自己的域名
```

### 防重放

```text
每次认证，服务器发送随机 challenge

攻击者即使截获了签名，也无法重用：
  - 下次 challenge 不同
  - 签名是对 challenge 的签名
  - 旧签名对新 challenge 无效
```

### 防数据库泄露

```text
服务器存储的是公钥

公钥泄露的后果：
  - 公钥本来就是公开的
  - 攻击者无法用公钥登录
  - 攻击者无法从公钥推导私钥
  - 没有任何可以离线破解的东西
```

### 防中间人

```text
HTTPS 保护传输通道

即使中间人看到了通信内容：
  - 看到的是公钥和签名
  - 无法获得私钥
  - 无法伪造签名
```

## Passkey 的用户体验

### 注册流程

```text
传统注册：
  1. 想一个密码
  2. 想一个不太简单的密码
  3. 重复输入密码
  4. 记住这个密码（或者存到密码管理器）

Passkey 注册：
  1. 点击"创建 Passkey"
  2. 验证身份（Touch ID / Face ID）
  3. 完成
```

### 登录流程

```text
传统登录：
  1. 输入用户名
  2. 输入密码（还记得吗？）
  3. 可能需要 2FA
  4. 登录成功

Passkey 登录：
  1. 点击"用 Passkey 登录"
  2. 验证身份（Touch ID / Face ID）
  3. 登录成功
```

### 跨设备使用

```text
同步 Passkey（Apple/Google 生态）：
  - 自动同步到你的所有设备
  - 新设备登录 iCloud/Google 后即可使用
  - 无需手动迁移

跨平台登录（如用 iPhone 登录 Windows）：
  1. 网站显示二维码
  2. 用手机扫码
  3. 手机验证身份
  4. 电脑上登录成功
```

## 实施建议

### 渐进式迁移

```text
阶段 1：Passkey 作为可选
  - 保留密码登录
  - 允许用户添加 Passkey
  - 积累用户反馈

阶段 2：推广 Passkey
  - 登录时提示"添加 Passkey"
  - 强调安全性和便利性
  - 但不强制

阶段 3：Passkey 优先
  - 默认显示 Passkey 登录
  - 密码登录作为备选
  - 新用户默认用 Passkey

阶段 4（可选）：密码淘汰
  - 只对高安全需求服务
  - 需要完善的账号恢复机制
```

### 账号恢复

```text
Passkey 最大的挑战：丢失设备怎么办？

方案 1：多设备注册
  - 鼓励用户在多个设备注册 Passkey
  - 一个设备丢失，其他设备仍可登录

方案 2：同步 Passkey
  - 使用 iCloud Keychain 或 Google Password Manager
  - 设备丢失不影响 Passkey

方案 3：备用恢复方式
  - 保留备用邮箱/手机验证
  - 恢复码（打印保存）
  - 人工客服验证

方案 4：硬件密钥备份
  - 注册一个 YubiKey 作为备份
  - 安全存放
```

## 核心认知

> **Passkey 的本质**：用公钥密码学替代"共享秘密"，从根本上消除密码的安全风险。

**为什么更安全**：
- 私钥从不传输，无法被窃取
- 公钥可以公开，泄露无影响
- 每次挑战不同，无法重放
- 绑定域名，防止钓鱼

**为什么更好用**：
- 无需记忆密码
- 生物识别一键登录
- 跨设备同步

**关键组件**：
- WebAuthn：浏览器 API 标准
- CTAP：与认证器通信协议
- 认证器：Touch ID、Face ID、YubiKey 等

## 引出下一篇

我们已经学习了认证的演进历史：

1. **Session**：服务器记住你
2. **JWT**：无状态令牌
3. **OAuth 2.0**：第三方授权
4. **OpenID Connect**：标准化身份层
5. **Passkey**：告别密码

但在实际系统中，你不会只用一种方案。

下一篇，我们来做一个**实战练习**：如何设计一个现代认证系统，把这些技术组合起来。

---

## 常见问题

### Q：Passkey 会完全取代密码吗？

**A：短期内不会，但长期是趋势。**

```text
Passkey 普及的障碍：
  - 用户教育成本
  - 旧设备不支持
  - 部分场景仍需密码（如设备加密）
  - 企业系统迁移周期长

预期时间线：
  2023-2025：早期采用者
  2025-2027：主流网站支持
  2027-2030：成为主要登录方式
  2030+：密码成为备选
```

### Q：同步 Passkey 安全吗？

**A：相对安全，但引入新的风险。**

```text
优点：
  - 私钥在云端加密存储
  - 用户不会因丢失设备而丢失 Passkey

风险：
  - 依赖云账号安全（Apple ID / Google 账号）
  - 云服务商有理论上的访问能力

对比：
  - 同步 Passkey：安全性略低，但实用性高
  - 设备绑定 Passkey：安全性最高，但丢设备就没了

建议：
  - 普通用户：同步 Passkey 足够
  - 高安全需求：设备绑定 + 硬件备份
```

### Q：我应该继续使用 YubiKey 吗？

**A：取决于你的安全需求。**

```text
YubiKey 的优势：
  - 物理隔离，最高安全等级
  - 不依赖云服务
  - 不依赖设备操作系统

YubiKey 的劣势：
  - 需要随身携带
  - 可能丢失
  - 价格不便宜

建议：
  - 普通用户：平台 Passkey（Touch ID/Face ID）足够
  - 开发者/管理员：考虑 YubiKey 作为备份
  - 高管/关键账号：强烈建议 YubiKey
```

---

*下一篇：[实战：设计一个现代认证系统](/posts/auth-evolution-6-practice/)*

*本系列：*
1. *[Session——服务器说"我记得你"](/posts/auth-evolution-1-session/)*
2. *[JWT——把状态装进令牌](/posts/auth-evolution-2-jwt/)*
3. *[OAuth 2.0——让别人帮你认证](/posts/auth-evolution-3-oauth/)*
4. *[OpenID Connect——标准化的身份层](/posts/auth-evolution-4-oidc/)*
5. *Passkey——告别密码（本篇）*
6. *[实战：设计一个现代认证系统](/posts/auth-evolution-6-practice/)*
