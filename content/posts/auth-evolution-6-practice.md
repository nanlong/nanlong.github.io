---
title: "认证演进史（六）：实战——设计一个现代认证系统"
date: 2025-12-09T00:00:00+08:00
draft: false
tags: ["认证", "系统设计", "安全", "实战"]
categories: ["技术"]
description: "学了这么多认证技术，实际项目该怎么选？这篇给你一个完整的决策框架和实现方案。"
series: ["认证演进史"]
---

前五篇我们学习了 Session、JWT、OAuth 2.0、OIDC、Passkey。

你可能会问：**实际项目该用哪个？**

答案是：**通常不止用一个。**

这篇我们来设计一个真实的现代认证系统。

<!--more-->

## 需求分析

### 假设场景

```text
你要为一个 SaaS 产品设计认证系统：

产品特点：
  - Web 端 + 移动端
  - 面向企业用户
  - 需要团队协作功能
  - 可能需要对接企业 SSO

用户规模：
  - 初期：1 万用户
  - 目标：100 万用户

安全要求：
  - 符合行业安全标准
  - 支持双因素认证
  - 审计日志
```

### 功能清单

```text
核心功能：
  □ 用户名/密码登录
  □ 邮箱验证
  □ 第三方登录（Google、GitHub）
  □ Passkey 支持
  □ 双因素认证（2FA）
  □ 单点登录（SSO）对接

安全功能：
  □ 登录异常检测
  □ 设备管理
  □ 会话管理
  □ 密码策略
  □ 审计日志

用户体验：
  □ "记住我"功能
  □ 密码重置
  □ 账号恢复
  □ 多设备同步
```

## 架构设计

### 整体架构

```text
┌─────────────────────────────────────────────────────────────────┐
│                         客户端                                   │
├─────────────┬─────────────┬─────────────┬─────────────────────────┤
│  Web App    │  iOS App    │ Android App │     第三方应用          │
└──────┬──────┴──────┬──────┴──────┬──────┴───────────┬────────────┘
       │             │             │                   │
       └─────────────┴──────┬──────┴───────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway                               │
│              (限流、认证、路由、日志)                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   Auth Service  │ │  User Service   │ │ Business Service│
│                 │ │                 │ │                 │
│ - 登录/登出     │ │ - 用户信息      │ │ - 业务逻辑      │
│ - Token 管理    │ │ - 权限管理      │ │                 │
│ - 2FA           │ │ - 团队管理      │ │                 │
│ - SSO           │ │                 │ │                 │
└────────┬────────┘ └────────┬────────┘ └─────────────────┘
         │                   │
         │                   │
         ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                        数据存储层                                 │
├─────────────────┬─────────────────┬─────────────────────────────┤
│    PostgreSQL   │      Redis      │           S3               │
│   (用户/凭证)   │  (Session/缓存) │       (头像/文件)           │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

### 认证流程选择

```text
┌────────────────────┬──────────────────────────────────────────────┐
│ 场景                │ 方案                                          │
├────────────────────┼──────────────────────────────────────────────┤
│ Web 端登录          │ Session（HttpOnly Cookie）+ CSRF Token       │
│                    │ 理由：最安全，浏览器原生支持                    │
├────────────────────┼──────────────────────────────────────────────┤
│ 移动端登录          │ JWT（Access Token + Refresh Token）           │
│                    │ 理由：无 Cookie，需要自己管理 Token            │
├────────────────────┼──────────────────────────────────────────────┤
│ 第三方应用对接      │ OAuth 2.0 / OIDC                             │
│                    │ 理由：标准化，安全边界清晰                      │
├────────────────────┼──────────────────────────────────────────────┤
│ 内部服务通信        │ JWT（服务间）或 mTLS                          │
│                    │ 理由：无用户参与，机器对机器认证                │
└────────────────────┴──────────────────────────────────────────────┘
```

## 核心实现

### 数据模型

```rust
use sqlx::FromRow;
use uuid::Uuid;
use chrono::{DateTime, Utc};

// 用户表
#[derive(FromRow)]
struct User {
    id: Uuid,
    email: String,
    email_verified: bool,
    password_hash: Option<String>,  // Passkey-only 用户可能没有密码
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
}

// 用户凭证（Passkey）
#[derive(FromRow)]
struct Credential {
    id: Uuid,
    user_id: Uuid,
    credential_id: Vec<u8>,        // WebAuthn credential ID
    public_key: Vec<u8>,           // COSE 格式公钥
    counter: u32,                   // 签名计数器
    transports: Vec<String>,        // ["usb", "nfc", "ble", "internal"]
    created_at: DateTime<Utc>,
    last_used_at: Option<DateTime<Utc>>,
}

// 第三方身份关联
#[derive(FromRow)]
struct ExternalIdentity {
    id: Uuid,
    user_id: Uuid,
    provider: String,               // "google", "github"
    provider_user_id: String,       // 第三方用户 ID
    email: Option<String>,
    created_at: DateTime<Utc>,
}

// 会话表
#[derive(FromRow)]
struct Session {
    id: Uuid,
    user_id: Uuid,
    token_hash: String,             // Session Token 的哈希
    device_info: String,            // 设备信息 JSON
    ip_address: String,
    created_at: DateTime<Utc>,
    last_active_at: DateTime<Utc>,
    expires_at: DateTime<Utc>,
}

// Refresh Token
#[derive(FromRow)]
struct RefreshToken {
    id: Uuid,
    user_id: Uuid,
    token_hash: String,
    device_id: Option<Uuid>,
    created_at: DateTime<Utc>,
    expires_at: DateTime<Utc>,
    revoked_at: Option<DateTime<Utc>>,
}

// 2FA 设置
#[derive(FromRow)]
struct TwoFactorAuth {
    id: Uuid,
    user_id: Uuid,
    method: String,                 // "totp", "sms", "email"
    secret: Option<String>,         // TOTP 密钥（加密存储）
    phone: Option<String>,          // SMS 验证的手机号
    verified: bool,
    created_at: DateTime<Utc>,
}
```

### 多方式登录流程

```rust
use axum::{Router, routing::post, Json, Extension};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
#[serde(tag = "method")]
enum LoginRequest {
    #[serde(rename = "password")]
    Password { email: String, password: String },

    #[serde(rename = "passkey")]
    Passkey { credential: PublicKeyCredential },

    #[serde(rename = "oauth")]
    OAuth { provider: String, code: String, state: String },
}

#[derive(Serialize)]
struct LoginResponse {
    user: UserInfo,
    #[serde(skip_serializing_if = "Option::is_none")]
    requires_2fa: Option<TwoFactorChallenge>,
    #[serde(skip_serializing_if = "Option::is_none")]
    session_token: Option<String>,  // Web 端通过 Set-Cookie 返回
    #[serde(skip_serializing_if = "Option::is_none")]
    tokens: Option<TokenPair>,      // 移动端返回 Token
}

async fn login(
    Extension(state): Extension<Arc<AppState>>,
    Json(req): Json<LoginRequest>,
) -> Result<Json<LoginResponse>, AuthError> {
    // 第一步：验证身份
    let user = match req {
        LoginRequest::Password { email, password } => {
            verify_password(&state, &email, &password).await?
        }
        LoginRequest::Passkey { credential } => {
            verify_passkey(&state, &credential).await?
        }
        LoginRequest::OAuth { provider, code, state: oauth_state } => {
            verify_oauth(&state, &provider, &code, &oauth_state).await?
        }
    };

    // 第二步：检查是否需要 2FA
    if let Some(tfa) = get_user_2fa(&state, user.id).await? {
        let challenge = create_2fa_challenge(&state, user.id, &tfa).await?;
        return Ok(Json(LoginResponse {
            user: user.into(),
            requires_2fa: Some(challenge),
            session_token: None,
            tokens: None,
        }));
    }

    // 第三步：创建会话
    let session = create_session(&state, &user).await?;

    Ok(Json(LoginResponse {
        user: user.into(),
        requires_2fa: None,
        session_token: Some(session.token),
        tokens: None,
    }))
}
```

### JWT Token 实现

```rust
use jsonwebtoken::{encode, decode, Header, Validation, EncodingKey, DecodingKey};
use chrono::{Utc, Duration};

#[derive(Debug, Serialize, Deserialize)]
struct AccessTokenClaims {
    sub: String,           // 用户 ID
    email: String,
    roles: Vec<String>,
    exp: usize,
    iat: usize,
    jti: String,           // Token ID，用于撤销
}

#[derive(Debug, Serialize, Deserialize)]
struct RefreshTokenClaims {
    sub: String,
    exp: usize,
    iat: usize,
    jti: String,
    device_id: Option<String>,
}

struct TokenService {
    access_key: EncodingKey,
    refresh_key: EncodingKey,
    access_ttl: Duration,
    refresh_ttl: Duration,
}

impl TokenService {
    fn create_access_token(&self, user: &User, roles: Vec<String>) -> Result<String, Error> {
        let now = Utc::now();
        let claims = AccessTokenClaims {
            sub: user.id.to_string(),
            email: user.email.clone(),
            roles,
            iat: now.timestamp() as usize,
            exp: (now + self.access_ttl).timestamp() as usize,
            jti: Uuid::new_v4().to_string(),
        };

        encode(&Header::default(), &claims, &self.access_key)
            .map_err(|e| Error::TokenCreation(e.to_string()))
    }

    fn create_refresh_token(&self, user: &User, device_id: Option<Uuid>) -> Result<String, Error> {
        let now = Utc::now();
        let claims = RefreshTokenClaims {
            sub: user.id.to_string(),
            iat: now.timestamp() as usize,
            exp: (now + self.refresh_ttl).timestamp() as usize,
            jti: Uuid::new_v4().to_string(),
            device_id: device_id.map(|d| d.to_string()),
        };

        encode(&Header::default(), &claims, &self.refresh_key)
            .map_err(|e| Error::TokenCreation(e.to_string()))
    }

    async fn refresh(&self, refresh_token: &str, db: &PgPool) -> Result<TokenPair, Error> {
        // 1. 验证 Refresh Token
        let claims = decode::<RefreshTokenClaims>(
            refresh_token,
            &DecodingKey::from_secret(/* ... */),
            &Validation::default(),
        )?.claims;

        // 2. 检查是否被撤销
        let is_revoked = sqlx::query_scalar!(
            "SELECT EXISTS(SELECT 1 FROM revoked_tokens WHERE jti = $1)",
            claims.jti
        )
        .fetch_one(db)
        .await?
        .unwrap_or(false);

        if is_revoked {
            return Err(Error::TokenRevoked);
        }

        // 3. 获取用户
        let user = get_user_by_id(db, &claims.sub).await?;

        // 4. 生成新的 Token 对
        let device_id = claims.device_id.and_then(|d| Uuid::parse_str(&d).ok());
        Ok(TokenPair {
            access_token: self.create_access_token(&user, get_user_roles(db, user.id).await?)?,
            refresh_token: self.create_refresh_token(&user, device_id)?,
        })
    }
}
```

### 2FA 实现

```rust
use totp_rs::{TOTP, Algorithm, Secret};

struct TwoFactorService {
    issuer: String,
}

impl TwoFactorService {
    // 生成 TOTP 密钥
    fn generate_secret(&self, user_email: &str) -> Result<TotpSetup, Error> {
        let secret = Secret::generate_secret();

        let totp = TOTP::new(
            Algorithm::SHA1,
            6,      // 6 位数字
            1,      // 允许 1 步偏移
            30,     // 30 秒一步
            secret.to_bytes().unwrap(),
            Some(self.issuer.clone()),
            user_email.to_string(),
        )?;

        Ok(TotpSetup {
            secret: secret.to_encoded().to_string(),
            qr_code_url: totp.get_url(),
        })
    }

    // 验证 TOTP 码
    fn verify_totp(&self, secret: &str, code: &str) -> Result<bool, Error> {
        let secret = Secret::Encoded(secret.to_string())
            .to_bytes()
            .map_err(|_| Error::InvalidSecret)?;

        let totp = TOTP::new(
            Algorithm::SHA1,
            6,
            1,
            30,
            secret,
            None,
            String::new(),
        )?;

        Ok(totp.check_current(code)?)
    }
}

// 2FA 验证端点
async fn verify_2fa(
    Extension(state): Extension<Arc<AppState>>,
    Json(req): Json<Verify2faRequest>,
) -> Result<Json<LoginResponse>, AuthError> {
    // 1. 获取 2FA challenge
    let challenge = get_2fa_challenge(&state, &req.challenge_id).await?;

    // 2. 验证代码
    let is_valid = match challenge.method.as_str() {
        "totp" => {
            let tfa = get_user_2fa(&state, challenge.user_id).await?
                .ok_or(AuthError::TwoFactorNotSetup)?;
            state.tfa_service.verify_totp(&tfa.secret.unwrap(), &req.code)?
        }
        "sms" | "email" => {
            verify_otp_code(&state, &challenge, &req.code).await?
        }
        _ => return Err(AuthError::InvalidMethod),
    };

    if !is_valid {
        return Err(AuthError::InvalidCode);
    }

    // 3. 创建会话
    let user = get_user_by_id(&state.db, challenge.user_id).await?;
    let session = create_session(&state, &user).await?;

    // 4. 清理 challenge
    delete_2fa_challenge(&state, &req.challenge_id).await?;

    Ok(Json(LoginResponse {
        user: user.into(),
        requires_2fa: None,
        session_token: Some(session.token),
        tokens: None,
    }))
}
```

### 安全中间件

```rust
use axum::{middleware::Next, http::Request, response::Response};

// 认证中间件
async fn auth_middleware<B>(
    Extension(state): Extension<Arc<AppState>>,
    mut request: Request<B>,
    next: Next<B>,
) -> Result<Response, AuthError> {
    // 尝试从 Cookie 获取 Session Token
    let session = if let Some(cookie) = request.headers().get("cookie") {
        extract_session_from_cookie(cookie, &state).await?
    } else if let Some(auth) = request.headers().get("authorization") {
        // 尝试从 Authorization 头获取 JWT
        extract_user_from_jwt(auth, &state).await?
    } else {
        return Err(AuthError::Unauthorized);
    };

    // 注入用户信息
    request.extensions_mut().insert(session);

    Ok(next.run(request).await)
}

// 限流中间件
async fn rate_limit_middleware<B>(
    Extension(state): Extension<Arc<AppState>>,
    request: Request<B>,
    next: Next<B>,
) -> Result<Response, RateLimitError> {
    let ip = extract_client_ip(&request);
    let endpoint = request.uri().path();

    // 登录端点有更严格的限制
    let limit = if endpoint.contains("/login") {
        RateLimit::new(5, Duration::minutes(1))   // 每分钟 5 次
    } else {
        RateLimit::new(100, Duration::minutes(1)) // 每分钟 100 次
    };

    let key = format!("rate_limit:{}:{}", ip, endpoint);
    let current = state.redis.incr(&key).await?;

    if current == 1 {
        state.redis.expire(&key, limit.window.num_seconds() as usize).await?;
    }

    if current > limit.max_requests {
        return Err(RateLimitError::TooManyRequests);
    }

    Ok(next.run(request).await)
}

// 审计日志中间件
async fn audit_middleware<B>(
    Extension(state): Extension<Arc<AppState>>,
    Extension(user): Extension<Option<AuthenticatedUser>>,
    request: Request<B>,
    next: Next<B>,
) -> Response {
    let method = request.method().clone();
    let path = request.uri().path().to_string();
    let ip = extract_client_ip(&request);
    let user_id = user.map(|u| u.id);

    let response = next.run(request).await;

    // 异步记录审计日志
    let status = response.status().as_u16();
    tokio::spawn(async move {
        log_audit_event(AuditEvent {
            user_id,
            action: format!("{} {}", method, path),
            ip_address: ip,
            status_code: status,
            timestamp: Utc::now(),
        }).await;
    });

    response
}
```

## OAuth/OIDC 集成

### Provider 抽象

```rust
#[async_trait]
trait OAuthProvider: Send + Sync {
    fn name(&self) -> &str;
    fn authorization_url(&self, state: &str, nonce: &str) -> String;
    async fn exchange_code(&self, code: &str) -> Result<TokenResponse, Error>;
    async fn get_user_info(&self, access_token: &str) -> Result<OAuthUser, Error>;
}

struct GoogleProvider {
    client_id: String,
    client_secret: String,
    redirect_uri: String,
}

#[async_trait]
impl OAuthProvider for GoogleProvider {
    fn name(&self) -> &str {
        "google"
    }

    fn authorization_url(&self, state: &str, nonce: &str) -> String {
        format!(
            "https://accounts.google.com/o/oauth2/v2/auth?\
            client_id={}&\
            redirect_uri={}&\
            response_type=code&\
            scope=openid%20email%20profile&\
            state={}&\
            nonce={}",
            self.client_id,
            urlencoding::encode(&self.redirect_uri),
            state,
            nonce
        )
    }

    async fn exchange_code(&self, code: &str) -> Result<TokenResponse, Error> {
        let client = reqwest::Client::new();
        let response = client
            .post("https://oauth2.googleapis.com/token")
            .form(&[
                ("client_id", &self.client_id),
                ("client_secret", &self.client_secret),
                ("code", &code.to_string()),
                ("redirect_uri", &self.redirect_uri),
                ("grant_type", &"authorization_code".to_string()),
            ])
            .send()
            .await?
            .json::<TokenResponse>()
            .await?;

        Ok(response)
    }

    async fn get_user_info(&self, access_token: &str) -> Result<OAuthUser, Error> {
        let client = reqwest::Client::new();
        let response = client
            .get("https://www.googleapis.com/oauth2/v2/userinfo")
            .bearer_auth(access_token)
            .send()
            .await?
            .json::<GoogleUserInfo>()
            .await?;

        Ok(OAuthUser {
            provider: "google".to_string(),
            provider_user_id: response.id,
            email: Some(response.email),
            name: response.name,
            avatar: response.picture,
        })
    }
}
```

### 企业 SSO 支持

```rust
// SAML 和 OIDC 企业 SSO
struct EnterpriseSSO {
    db: PgPool,
    oidc_providers: HashMap<String, OidcConfig>,
    saml_providers: HashMap<String, SamlConfig>,
}

impl EnterpriseSSO {
    // 根据邮箱域名自动选择 SSO 提供商
    async fn get_provider_for_email(&self, email: &str) -> Option<SsoProvider> {
        let domain = email.split('@').last()?;

        // 查数据库：这个域名配置了哪个 SSO
        let config = sqlx::query_as!(
            SsoConfig,
            "SELECT * FROM enterprise_sso WHERE domain = $1",
            domain
        )
        .fetch_optional(&self.db)
        .await
        .ok()??;

        Some(SsoProvider {
            config_type: config.provider_type,
            metadata_url: config.metadata_url,
            client_id: config.client_id,
        })
    }
}

// 企业配置端点
async fn configure_enterprise_sso(
    Extension(state): Extension<Arc<AppState>>,
    Extension(user): Extension<AuthenticatedUser>,
    Json(req): Json<ConfigureSsoRequest>,
) -> Result<Json<SsoConfigResponse>, Error> {
    // 验证用户是组织管理员
    require_org_admin(&state, &user, &req.org_id).await?;

    // 验证 OIDC/SAML 配置有效
    validate_sso_config(&req).await?;

    // 保存配置
    let config = sqlx::query_as!(
        SsoConfig,
        r#"
        INSERT INTO enterprise_sso (org_id, domain, provider_type, metadata_url, client_id, client_secret)
        VALUES ($1, $2, $3, $4, $5, $6)
        RETURNING *
        "#,
        req.org_id,
        req.domain,
        req.provider_type,
        req.metadata_url,
        req.client_id,
        encrypt(&req.client_secret)
    )
    .fetch_one(&state.db)
    .await?;

    Ok(Json(config.into()))
}
```

## 安全加固

### 登录异常检测

```rust
struct SecurityService {
    redis: Redis,
    db: PgPool,
}

impl SecurityService {
    async fn check_login_risk(&self, user_id: Uuid, context: &LoginContext) -> RiskLevel {
        let mut score = 0;

        // 1. 检查 IP 是否异常
        if self.is_new_ip(user_id, &context.ip).await {
            score += 20;
        }

        // 2. 检查设备是否异常
        if self.is_new_device(user_id, &context.device_fingerprint).await {
            score += 30;
        }

        // 3. 检查地理位置
        if self.is_impossible_travel(user_id, &context.geo).await {
            score += 50;  // 不可能的旅行（短时间内跨大距离）
        }

        // 4. 检查失败尝试次数
        let failed_attempts = self.get_failed_attempts(user_id).await;
        if failed_attempts > 3 {
            score += failed_attempts * 10;
        }

        match score {
            0..=30 => RiskLevel::Low,
            31..=60 => RiskLevel::Medium,
            _ => RiskLevel::High,
        }
    }

    async fn is_impossible_travel(&self, user_id: Uuid, current_geo: &GeoLocation) -> bool {
        // 获取上次登录位置和时间
        let last_login = sqlx::query!(
            "SELECT ip_address, created_at FROM sessions
             WHERE user_id = $1 ORDER BY created_at DESC LIMIT 1",
            user_id
        )
        .fetch_optional(&self.db)
        .await
        .ok()
        .flatten();

        if let Some(last) = last_login {
            let last_geo = ip_to_geo(&last.ip_address).await;
            let distance = haversine_distance(&last_geo, current_geo);
            let time_diff = Utc::now() - last.created_at;

            // 假设最快飞机速度 1000 km/h
            let max_possible_distance = time_diff.num_hours() as f64 * 1000.0;

            return distance > max_possible_distance;
        }

        false
    }
}

// 根据风险等级采取措施
async fn handle_login_risk(
    risk: RiskLevel,
    user: &User,
    state: &AppState,
) -> Result<LoginAction, Error> {
    match risk {
        RiskLevel::Low => Ok(LoginAction::Allow),
        RiskLevel::Medium => {
            // 发送通知邮件
            send_login_alert(user, state).await?;
            Ok(LoginAction::AllowWithWarning)
        }
        RiskLevel::High => {
            // 要求额外验证
            Ok(LoginAction::Require2FA)
        }
    }
}
```

### 会话管理

```rust
// 用户可以查看和管理自己的会话
async fn list_sessions(
    Extension(state): Extension<Arc<AppState>>,
    Extension(user): Extension<AuthenticatedUser>,
) -> Result<Json<Vec<SessionInfo>>, Error> {
    let sessions = sqlx::query_as!(
        SessionInfo,
        r#"
        SELECT id, device_info, ip_address, created_at, last_active_at,
               (id = $2) as "is_current!"
        FROM sessions
        WHERE user_id = $1 AND expires_at > NOW()
        ORDER BY last_active_at DESC
        "#,
        user.id,
        user.session_id
    )
    .fetch_all(&state.db)
    .await?;

    Ok(Json(sessions))
}

// 撤销特定会话
async fn revoke_session(
    Extension(state): Extension<Arc<AppState>>,
    Extension(user): Extension<AuthenticatedUser>,
    Path(session_id): Path<Uuid>,
) -> Result<StatusCode, Error> {
    // 确保会话属于当前用户
    let affected = sqlx::query!(
        "DELETE FROM sessions WHERE id = $1 AND user_id = $2",
        session_id,
        user.id
    )
    .execute(&state.db)
    .await?
    .rows_affected();

    if affected == 0 {
        return Err(Error::NotFound);
    }

    Ok(StatusCode::NO_CONTENT)
}

// 撤销所有其他会话（"登出所有设备"）
async fn revoke_other_sessions(
    Extension(state): Extension<Arc<AppState>>,
    Extension(user): Extension<AuthenticatedUser>,
) -> Result<Json<RevokeResponse>, Error> {
    let affected = sqlx::query!(
        "DELETE FROM sessions WHERE user_id = $1 AND id != $2",
        user.id,
        user.session_id
    )
    .execute(&state.db)
    .await?
    .rows_affected();

    Ok(Json(RevokeResponse {
        revoked_count: affected as u32,
    }))
}
```

## 部署架构

### 高可用部署

```text
┌─────────────────────────────────────────────────────────────────┐
│                        CDN / WAF                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│                      Load Balancer                               │
│                    (AWS ALB / Nginx)                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  Auth Service   │ │  Auth Service   │ │  Auth Service   │
│   (Pod 1)       │ │   (Pod 2)       │ │   (Pod 3)       │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  PostgreSQL     │ │  Redis Cluster  │ │     Vault       │
│  (Primary +     │ │  (密钥存储、    │ │  (密钥管理)     │
│   Replica)      │ │   Session缓存)  │ │                 │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### 密钥管理

```rust
// 使用 Vault 管理敏感密钥
struct KeyManager {
    vault_client: VaultClient,
}

impl KeyManager {
    async fn get_jwt_signing_key(&self) -> Result<Vec<u8>, Error> {
        let secret = self.vault_client
            .read_secret("secret/auth/jwt-signing-key")
            .await?;
        Ok(secret.data.key)
    }

    async fn rotate_jwt_key(&self) -> Result<(), Error> {
        // 1. 生成新密钥
        let new_key = generate_secure_key();

        // 2. 存储新密钥（版本化）
        self.vault_client
            .write_secret("secret/auth/jwt-signing-key", &new_key)
            .await?;

        // 3. 保留旧密钥一段时间（验证旧 Token）
        // Vault 自动保留历史版本

        Ok(())
    }
}
```

## 决策指南

### 如何选择认证方案

```text
┌─────────────────────────────────────────────────────────────────┐
│                      选择决策树                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1: 是单页应用(SPA)还是传统服务端渲染?                           │
│    ├─ 传统服务端 → Session + Cookie                             │
│    └─ SPA        → JWT 或 Session + Cookie（推荐后者）          │
│                                                                 │
│  Q2: 需要支持移动端吗?                                           │
│    ├─ 是 → JWT（移动端）+ Session（Web端）                       │
│    └─ 否 → Session 足够                                         │
│                                                                 │
│  Q3: 需要第三方登录吗?                                           │
│    ├─ 是 → OAuth 2.0 + OIDC                                    │
│    └─ 否 → 可以跳过                                             │
│                                                                 │
│  Q4: 目标用户是企业还是消费者?                                    │
│    ├─ 企业 → 考虑 SSO（SAML/OIDC）                              │
│    └─ 消费者 → 社交登录 + Passkey                               │
│                                                                 │
│  Q5: 安全要求有多高?                                             │
│    ├─ 金融级 → 2FA 必须 + Passkey 优先                          │
│    └─ 普通 → 2FA 可选                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 常见错误

```text
❌ 错误 1：在前端存储 Access Token 到 localStorage
   → 容易被 XSS 攻击窃取
   → 正确：HttpOnly Cookie 或 内存

❌ 错误 2：JWT 过期时间设太长
   → 7 天的 Access Token = 7 天的攻击窗口
   → 正确：Access Token 15分钟，Refresh Token 7天

❌ 错误 3：不验证 OAuth state 参数
   → CSRF 攻击
   → 正确：始终验证 state

❌ 错误 4：密码重置链接不过期
   → 正确：15-30 分钟过期，一次性使用

❌ 错误 5：2FA 恢复码不加密存储
   → 正确：与密码同等对待，使用强哈希
```

## 核心认知

> **现代认证系统不是选择一种技术，而是组合多种技术来满足不同场景。**

**关键原则**：

1. **分层防御**：密码 + 2FA + 设备绑定 + 异常检测
2. **最小权限**：Token 权限最小化，有效期最短化
3. **用户体验**：提供多种登录方式，让用户选择
4. **可观测性**：完善的日志和监控
5. **可恢复性**：完善的账号恢复机制

**技术选择**：

| 场景 | 推荐方案 |
|------|----------|
| Web 认证 | Session + HttpOnly Cookie |
| 移动端 | JWT + Refresh Token |
| 第三方登录 | OIDC |
| 企业 SSO | SAML 或 OIDC |
| 最高安全 | Passkey + 2FA |

## 系列总结

六篇文章，我们走过了认证技术的演进历程：

1. **Session**：最基础，服务器记住你
2. **JWT**：无状态，自包含的令牌
3. **OAuth 2.0**：授权框架，让第三方安全访问
4. **OpenID Connect**：标准化的身份层
5. **Passkey**：公钥密码学，告别密码
6. **实战**：组合使用，设计现代系统

认证技术还在演进。但无论技术怎么变，核心问题始终是：

**如何在安全和便利之间找到平衡。**

希望这个系列对你有帮助。

---

*本系列：*
1. *[Session——服务器说"我记得你"](/posts/auth-evolution-1-session/)*
2. *[JWT——把状态装进令牌](/posts/auth-evolution-2-jwt/)*
3. *[OAuth 2.0——让别人帮你认证](/posts/auth-evolution-3-oauth/)*
4. *[OpenID Connect——标准化的身份层](/posts/auth-evolution-4-oidc/)*
5. *[Passkey——告别密码](/posts/auth-evolution-5-passkey/)*
6. *实战：设计一个现代认证系统（本篇）*
