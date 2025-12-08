---
title: "模式不是套路（七）：外部集成——与第三方和平共处"
date: 2025-12-10T11:00:00+08:00
draft: false
tags: ["Rust", "设计模式", "外部集成", "适配器模式", "外观模式", "代理模式"]
categories: ["技术"]
description: "第三方 SDK 接口丑陋，文档稀烂，还时不时改版？适配器、外观、代理三种模式，帮你在混乱的外部世界和整洁的内部代码之间建起防火墙。"
series: ["模式不是套路"]
---

第三方 SDK 接口丑陋，文档稀烂，还时不时改版？适配器、外观、代理三种模式，帮你在混乱的外部世界和整洁的内部代码之间建起防火墙。

<!--more-->

## 问题：被第三方绑架的代码

你的系统需要发短信。选了某云的短信服务：

```rust
// 某云短信 SDK
pub struct AliyunSmsClient {
    access_key_id: String,
    access_key_secret: String,
    region: String,
}

impl AliyunSmsClient {
    pub fn send_sms(
        &self,
        phone_numbers: &str,      // 逗号分隔的手机号
        sign_name: &str,          // 签名
        template_code: &str,      // 模板代码
        template_param: &str,     // JSON 格式的模板参数
    ) -> Result<AliyunSmsResponse, AliyunError> {
        // ...
    }
}

pub struct AliyunSmsResponse {
    pub request_id: String,
    pub code: String,
    pub message: String,
    pub biz_id: Option<String>,
}
```

业务代码直接调用：

```rust
impl NotificationService {
    pub fn send_verification_code(&self, phone: &str, code: &str) -> Result<(), Error> {
        let response = self.sms_client.send_sms(
            phone,
            "某某公司",
            "SMS_123456789",
            &format!(r#"{{"code":"{}"}}"#, code),
        )?;

        if response.code != "OK" {
            return Err(Error::SmsFailed(response.message));
        }

        Ok(())
    }

    pub fn send_order_notification(&self, phone: &str, order_id: &str) -> Result<(), Error> {
        let response = self.sms_client.send_sms(
            phone,
            "某某公司",
            "SMS_987654321",
            &format!(r#"{{"orderId":"{}"}}"#, order_id),
        )?;

        if response.code != "OK" {
            return Err(Error::SmsFailed(response.message));
        }

        Ok(())
    }
}
```

然后问题来了：

```text
场景 1：老板说某云太贵，换成腾讯云
  - 接口完全不同
  - 要改几十处调用代码

场景 2：海外用户用不了国内短信
  - 要接入 Twilio
  - 同一个方法要判断用哪个服务商

场景 3：测试时不想真发短信
  - Mock 很困难，因为代码和 SDK 耦合

场景 4：某云 SDK 升级了
  - 接口签名变了
  - 全部调用点都要改
```

## 破局：适配器模式

**适配器模式**的核心是：**将一个类的接口转换成客户希望的另一个接口**。

### 定义统一接口

```rust
// 我们想要的接口
#[async_trait]
pub trait SmsService: Send + Sync {
    async fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError>;
}

#[derive(Debug, Clone)]
pub struct SmsRequest {
    pub phone: String,
    pub template: SmsTemplate,
}

#[derive(Debug, Clone)]
pub enum SmsTemplate {
    VerificationCode { code: String },
    OrderNotification { order_id: String },
    ShippingNotice { tracking_number: String },
    // 业务模板，而不是平台模板代码
}

#[derive(Debug, Clone)]
pub struct SmsResponse {
    pub message_id: String,
    pub success: bool,
}

#[derive(Debug)]
pub enum SmsError {
    InvalidPhone,
    RateLimited,
    ServiceUnavailable,
    Unknown(String),
}
```

### 适配各个平台

```rust
// 阿里云适配器
pub struct AliyunSmsAdapter {
    client: AliyunSmsClient,
    sign_name: String,
    templates: HashMap<String, String>,  // 业务模板 -> 平台模板代码
}

impl AliyunSmsAdapter {
    pub fn new(config: AliyunConfig) -> Self {
        let client = AliyunSmsClient::new(
            &config.access_key_id,
            &config.access_key_secret,
            &config.region,
        );

        let mut templates = HashMap::new();
        templates.insert("verification_code".to_string(), "SMS_123456789".to_string());
        templates.insert("order_notification".to_string(), "SMS_987654321".to_string());

        Self { client, sign_name: config.sign_name, templates }
    }

    fn get_template_code(&self, template: &SmsTemplate) -> Option<&str> {
        let key = match template {
            SmsTemplate::VerificationCode { .. } => "verification_code",
            SmsTemplate::OrderNotification { .. } => "order_notification",
            SmsTemplate::ShippingNotice { .. } => "shipping_notice",
        };
        self.templates.get(key).map(|s| s.as_str())
    }

    fn build_params(&self, template: &SmsTemplate) -> String {
        match template {
            SmsTemplate::VerificationCode { code } => {
                format!(r#"{{"code":"{}"}}"#, code)
            }
            SmsTemplate::OrderNotification { order_id } => {
                format!(r#"{{"orderId":"{}"}}"#, order_id)
            }
            SmsTemplate::ShippingNotice { tracking_number } => {
                format!(r#"{{"trackingNumber":"{}"}}"#, tracking_number)
            }
        }
    }
}

#[async_trait]
impl SmsService for AliyunSmsAdapter {
    async fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError> {
        let template_code = self.get_template_code(&request.template)
            .ok_or(SmsError::Unknown("Unknown template".to_string()))?;

        let params = self.build_params(&request.template);

        let response = self.client
            .send_sms(&request.phone, &self.sign_name, template_code, &params)
            .map_err(|e| SmsError::Unknown(e.to_string()))?;

        if response.code == "OK" {
            Ok(SmsResponse {
                message_id: response.biz_id.unwrap_or_default(),
                success: true,
            })
        } else if response.code == "isv.BUSINESS_LIMIT_CONTROL" {
            Err(SmsError::RateLimited)
        } else {
            Err(SmsError::Unknown(response.message))
        }
    }
}

// 腾讯云适配器
pub struct TencentSmsAdapter {
    client: TencentSmsClient,
    app_id: String,
    templates: HashMap<String, u32>,
}

#[async_trait]
impl SmsService for TencentSmsAdapter {
    async fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError> {
        // 腾讯云的接口完全不同，但对外暴露相同的 SmsService trait
        let template_id = self.get_template_id(&request.template)?;
        let params = self.build_params(&request.template);

        let response = self.client
            .send_sms(&self.app_id, template_id, &[&request.phone], &params)
            .await
            .map_err(|e| SmsError::Unknown(e.to_string()))?;

        // 转换响应格式
        Ok(SmsResponse {
            message_id: response.serial_no,
            success: response.result == 0,
        })
    }
}

// Twilio 适配器（国际短信）
pub struct TwilioSmsAdapter {
    client: TwilioClient,
}

#[async_trait]
impl SmsService for TwilioSmsAdapter {
    async fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError> {
        // Twilio 使用纯文本而不是模板
        let body = self.render_message(&request.template);

        let response = self.client
            .send_message(&request.phone, &body)
            .await
            .map_err(|e| SmsError::Unknown(e.to_string()))?;

        Ok(SmsResponse {
            message_id: response.sid,
            success: true,
        })
    }
}
```

### 业务代码干净了

```rust
impl NotificationService {
    pub fn new(sms_service: Arc<dyn SmsService>) -> Self {
        Self { sms_service }
    }

    pub async fn send_verification_code(&self, phone: &str, code: &str) -> Result<(), Error> {
        self.sms_service.send(SmsRequest {
            phone: phone.to_string(),
            template: SmsTemplate::VerificationCode { code: code.to_string() },
        }).await?;

        Ok(())
    }
}

// 切换服务商只需要换适配器
let sms: Arc<dyn SmsService> = if config.sms_provider == "aliyun" {
    Arc::new(AliyunSmsAdapter::new(config.aliyun))
} else {
    Arc::new(TencentSmsAdapter::new(config.tencent))
};

let notification_service = NotificationService::new(sms);
```

## 演进：外观模式

系统需要对接多个外部服务来完成一个业务功能：

```text
用户实名认证流程：
1. 调用身份证 OCR 服务识别证件
2. 调用人脸活体检测服务
3. 调用公安数据库验证身份信息
4. 调用银行卡四要素验证
5. 调用黑名单服务检查
```

如果业务代码直接调用这 5 个服务，会非常复杂。

**外观模式**的核心是：**为子系统中的一组接口提供一个一致的界面**。

```rust
// 外观：统一的实名认证接口
pub struct KycFacade {
    ocr_service: Arc<dyn OcrService>,
    liveness_service: Arc<dyn LivenessService>,
    identity_service: Arc<dyn IdentityVerificationService>,
    bank_service: Arc<dyn BankCardService>,
    blacklist_service: Arc<dyn BlacklistService>,
}

impl KycFacade {
    // 简化的接口：一个方法完成所有验证
    pub async fn verify_identity(&self, request: KycRequest) -> Result<KycResult, KycError> {
        // 1. OCR 识别身份证
        let id_info = self.ocr_service
            .recognize_id_card(&request.id_card_front, &request.id_card_back)
            .await
            .map_err(|e| KycError::OcrFailed(e.to_string()))?;

        // 2. 人脸活体检测
        let liveness_result = self.liveness_service
            .verify_liveness(&request.face_video)
            .await
            .map_err(|e| KycError::LivenessFailed(e.to_string()))?;

        if !liveness_result.is_live {
            return Err(KycError::LivenessCheckFailed);
        }

        // 3. 人脸与身份证照片比对
        let face_match = self.liveness_service
            .compare_faces(&request.face_video, &id_info.photo)
            .await?;

        if face_match.similarity < 0.8 {
            return Err(KycError::FaceMismatch);
        }

        // 4. 身份信息核验
        let identity_valid = self.identity_service
            .verify(&id_info.name, &id_info.id_number)
            .await?;

        if !identity_valid {
            return Err(KycError::IdentityInvalid);
        }

        // 5. 银行卡验证（如果提供）
        if let Some(bank_card) = &request.bank_card {
            let bank_valid = self.bank_service
                .verify_four_elements(
                    &id_info.name,
                    &id_info.id_number,
                    &bank_card.card_number,
                    &bank_card.phone,
                )
                .await?;

            if !bank_valid {
                return Err(KycError::BankCardInvalid);
            }
        }

        // 6. 黑名单检查
        let is_blacklisted = self.blacklist_service
            .check(&id_info.id_number)
            .await?;

        if is_blacklisted {
            return Err(KycError::Blacklisted);
        }

        Ok(KycResult {
            name: id_info.name,
            id_number: id_info.id_number,
            verified: true,
            verified_at: Utc::now(),
        })
    }

    // 也可以提供简化版本
    pub async fn quick_verify(&self, name: &str, id_number: &str) -> Result<bool, KycError> {
        // 只做身份核验，不做人脸和银行卡
        self.identity_service.verify(name, id_number).await
    }
}

// 业务代码只需要调用外观
impl UserService {
    pub async fn register_with_kyc(&self, request: RegisterRequest) -> Result<User, Error> {
        // 一行代码完成复杂的认证流程
        let kyc_result = self.kyc_facade.verify_identity(KycRequest {
            id_card_front: request.id_card_front,
            id_card_back: request.id_card_back,
            face_video: request.face_video,
            bank_card: request.bank_card,
        }).await?;

        // 创建用户
        self.user_repo.create(User {
            name: kyc_result.name,
            id_number: kyc_result.id_number,
            kyc_verified: true,
            // ...
        }).await
    }
}
```

### 外观 vs 适配器

| 特性 | 适配器 | 外观 |
|------|--------|------|
| 目的 | 接口转换 | 简化接口 |
| 对象数量 | 通常是一个 | 通常是多个 |
| 改变接口 | 是 | 是 |
| 使用场景 | 兼容不同接口 | 隐藏复杂性 |

## 进阶：代理模式

有时候我们需要**控制对外部服务的访问**：

```text
- 缓存：避免重复调用
- 限流：防止打爆外部服务
- 熔断：服务不可用时快速失败
- 日志：记录所有调用
- 重试：失败自动重试
```

**代理模式**的核心是：**为其他对象提供一种代理以控制对这个对象的访问**。

### 缓存代理

```rust
pub struct CachingOcrProxy {
    inner: Arc<dyn OcrService>,
    cache: Arc<RwLock<HashMap<String, OcrResult>>>,
    ttl: Duration,
}

impl CachingOcrProxy {
    pub fn new(inner: Arc<dyn OcrService>, ttl: Duration) -> Self {
        Self {
            inner,
            cache: Arc::new(RwLock::new(HashMap::new())),
            ttl,
        }
    }

    fn cache_key(image: &[u8]) -> String {
        // 用图片哈希作为缓存 key
        use sha2::{Sha256, Digest};
        let hash = Sha256::digest(image);
        format!("{:x}", hash)
    }
}

#[async_trait]
impl OcrService for CachingOcrProxy {
    async fn recognize_id_card(&self, front: &[u8], back: &[u8]) -> Result<IdCardInfo, OcrError> {
        let key = Self::cache_key(front);

        // 检查缓存
        if let Some(cached) = self.cache.read().await.get(&key) {
            if cached.created_at.elapsed() < self.ttl {
                return Ok(cached.data.clone());
            }
        }

        // 缓存未命中，调用真实服务
        let result = self.inner.recognize_id_card(front, back).await?;

        // 存入缓存
        self.cache.write().await.insert(key, OcrResult {
            data: result.clone(),
            created_at: Instant::now(),
        });

        Ok(result)
    }
}
```

### 重试代理

```rust
pub struct RetryingProxy<S> {
    inner: S,
    max_retries: u32,
    backoff: Duration,
}

impl<S> RetryingProxy<S> {
    pub fn new(inner: S, max_retries: u32, backoff: Duration) -> Self {
        Self { inner, max_retries, backoff }
    }
}

#[async_trait]
impl<S: SmsService> SmsService for RetryingProxy<S> {
    async fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError> {
        let mut last_error = None;

        for attempt in 0..=self.max_retries {
            if attempt > 0 {
                let delay = self.backoff * attempt;
                tokio::time::sleep(delay).await;
            }

            match self.inner.send(request.clone()).await {
                Ok(response) => return Ok(response),
                Err(SmsError::RateLimited) => {
                    // 限流，等待后重试
                    last_error = Some(SmsError::RateLimited);
                }
                Err(SmsError::ServiceUnavailable) => {
                    // 服务不可用，重试
                    last_error = Some(SmsError::ServiceUnavailable);
                }
                Err(e) => {
                    // 其他错误，不重试
                    return Err(e);
                }
            }
        }

        Err(last_error.unwrap_or(SmsError::Unknown("Max retries exceeded".to_string())))
    }
}
```

### 熔断代理

```rust
use std::sync::atomic::{AtomicU32, AtomicU64, Ordering};

pub struct CircuitBreaker<S> {
    inner: S,
    failure_count: AtomicU32,
    last_failure_time: AtomicU64,
    threshold: u32,
    reset_timeout: Duration,
    state: RwLock<CircuitState>,
}

#[derive(Debug, Clone, Copy)]
enum CircuitState {
    Closed,      // 正常
    Open,        // 熔断
    HalfOpen,    // 半开（试探）
}

impl<S> CircuitBreaker<S> {
    pub fn new(inner: S, threshold: u32, reset_timeout: Duration) -> Self {
        Self {
            inner,
            failure_count: AtomicU32::new(0),
            last_failure_time: AtomicU64::new(0),
            threshold,
            reset_timeout,
            state: RwLock::new(CircuitState::Closed),
        }
    }

    fn record_success(&self) {
        self.failure_count.store(0, Ordering::Relaxed);
        *self.state.write().unwrap() = CircuitState::Closed;
    }

    fn record_failure(&self) {
        let count = self.failure_count.fetch_add(1, Ordering::Relaxed) + 1;
        self.last_failure_time.store(
            Utc::now().timestamp() as u64,
            Ordering::Relaxed
        );

        if count >= self.threshold {
            *self.state.write().unwrap() = CircuitState::Open;
        }
    }

    fn should_allow(&self) -> bool {
        let state = *self.state.read().unwrap();
        match state {
            CircuitState::Closed => true,
            CircuitState::Open => {
                // 检查是否应该进入半开状态
                let last_failure = self.last_failure_time.load(Ordering::Relaxed);
                let now = Utc::now().timestamp() as u64;
                if now - last_failure > self.reset_timeout.as_secs() {
                    *self.state.write().unwrap() = CircuitState::HalfOpen;
                    true
                } else {
                    false
                }
            }
            CircuitState::HalfOpen => true,
        }
    }
}

#[async_trait]
impl<S: SmsService> SmsService for CircuitBreaker<S> {
    async fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError> {
        if !self.should_allow() {
            return Err(SmsError::ServiceUnavailable);
        }

        match self.inner.send(request).await {
            Ok(response) => {
                self.record_success();
                Ok(response)
            }
            Err(e) => {
                self.record_failure();
                Err(e)
            }
        }
    }
}
```

### 组合代理

代理可以像装饰器一样层层包装：

```rust
// 从内到外：真实服务 -> 缓存 -> 重试 -> 熔断 -> 日志
let sms_service: Arc<dyn SmsService> = {
    let real = AliyunSmsAdapter::new(config);
    let with_retry = RetryingProxy::new(real, 3, Duration::from_secs(1));
    let with_circuit_breaker = CircuitBreaker::new(with_retry, 5, Duration::from_secs(60));
    let with_logging = LoggingProxy::new(with_circuit_breaker);
    Arc::new(with_logging)
};
```

### Rust 的礼物：tower 中间件

Rust 的 `tower` crate 专门用于构建这种分层服务：

```rust
use tower::{ServiceBuilder, ServiceExt, Service};
use tower::limit::RateLimitLayer;
use tower::retry::RetryLayer;
use tower::timeout::TimeoutLayer;

let sms_service = ServiceBuilder::new()
    .layer(TimeoutLayer::new(Duration::from_secs(10)))
    .layer(RateLimitLayer::new(100, Duration::from_secs(1)))
    .layer(RetryLayer::new(RetryPolicy::default()))
    .service(AliyunSmsService::new(config));
```

## 陷阱：别掉进这些坑

### 陷阱一：适配器泄漏

```rust
// ❌ 适配器暴露了底层实现的细节
pub trait SmsService {
    fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError>;

    // 这个方法暴露了"模板"概念，但不是所有服务商都有模板
    fn list_templates(&self) -> Vec<SmsTemplate>;

    // 这个暴露了阿里云特有的概念
    fn get_sign_names(&self) -> Vec<String>;
}
```

解决方案：**接口只暴露业务概念**

```rust
// ✅ 接口只包含通用的业务操作
pub trait SmsService {
    fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError>;
    fn get_balance(&self) -> Result<u64, SmsError>;  // 通用：查余额
    fn get_send_history(&self, phone: &str) -> Result<Vec<SmsRecord>, SmsError>;  // 通用：发送记录
}
```

### 陷阱二：外观变成 God Object

```rust
// ❌ 外观承担太多职责
pub struct EverythingFacade {
    // 20 个服务
    sms_service: Arc<dyn SmsService>,
    email_service: Arc<dyn EmailService>,
    push_service: Arc<dyn PushService>,
    payment_service: Arc<dyn PaymentService>,
    // ... 16 more

    // 50 个方法
    pub fn send_sms(...) { }
    pub fn send_email(...) { }
    pub fn process_payment(...) { }
    // ... 47 more
}
```

解决方案：**按业务领域拆分外观**

```rust
// ✅ 每个外观只负责一个领域
pub struct NotificationFacade { /* ... */ }  // 通知相关
pub struct PaymentFacade { /* ... */ }       // 支付相关
pub struct KycFacade { /* ... */ }           // 认证相关
pub struct LogisticsFacade { /* ... */ }     // 物流相关
```

### 陷阱三：代理链太长

```rust
// ❌ 代理套代理套代理...
let service = LoggingProxy::new(
    MetricsProxy::new(
        CachingProxy::new(
            RetryingProxy::new(
                CircuitBreakerProxy::new(
                    RateLimitingProxy::new(
                        TimeoutProxy::new(
                            RealService::new()
                        )
                    )
                )
            )
        )
    )
);
// 调试时根本不知道问题出在哪一层
```

解决方案：

```rust
// ✅ 方案 1：使用 Builder 模式，明确顺序
let service = ServiceBuilder::new()
    .with_logging()
    .with_metrics()
    .with_caching(Duration::from_secs(60))
    .with_retry(3)
    .with_circuit_breaker(5, Duration::from_secs(30))
    .with_rate_limit(100)
    .with_timeout(Duration::from_secs(10))
    .build(RealService::new());

// ✅ 方案 2：合并相关功能到一个代理
pub struct ResilientProxy<S> {
    inner: S,
    retry_config: RetryConfig,
    circuit_breaker: CircuitBreaker,
    rate_limiter: RateLimiter,
}
// 一个代理处理所有"弹性"相关的功能
```

### 陷阱四：测试困难

```rust
// ❌ 代理依赖具体类型，难以 mock
pub struct CachingProxy {
    inner: AliyunSmsAdapter,  // 具体类型
    cache: HashMap<String, SmsResponse>,
}
```

解决方案：**依赖抽象**

```rust
// ✅ 依赖 trait，方便测试
pub struct CachingProxy<S: SmsService> {
    inner: S,
    cache: HashMap<String, SmsResponse>,
}

// 测试时用 mock
#[cfg(test)]
mod tests {
    struct MockSmsService;

    impl SmsService for MockSmsService {
        fn send(&self, _: SmsRequest) -> Result<SmsResponse, SmsError> {
            Ok(SmsResponse { message_id: "mock".to_string(), success: true })
        }
    }

    #[test]
    fn test_caching_proxy() {
        let proxy = CachingProxy::new(MockSmsService, Duration::from_secs(60));
        // 测试缓存逻辑
    }
}
```

## 完整示例

```rust
use async_trait::async_trait;
use std::collections::HashMap;
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::RwLock;

// ==================== 统一接口 ====================

#[async_trait]
pub trait SmsService: Send + Sync {
    async fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError>;
}

#[derive(Debug, Clone)]
pub struct SmsRequest {
    pub phone: String,
    pub template: SmsTemplate,
}

#[derive(Debug, Clone)]
pub enum SmsTemplate {
    VerificationCode { code: String },
    OrderNotification { order_id: String },
}

#[derive(Debug, Clone)]
pub struct SmsResponse {
    pub message_id: String,
    pub success: bool,
}

#[derive(Debug, Clone)]
pub enum SmsError {
    InvalidPhone,
    RateLimited,
    ServiceUnavailable,
    Unknown(String),
}

// ==================== 适配器 ====================

pub struct MockSmsAdapter;

#[async_trait]
impl SmsService for MockSmsAdapter {
    async fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError> {
        println!("[MockSms] Sending to {}: {:?}", request.phone, request.template);
        Ok(SmsResponse {
            message_id: format!("mock-{}", uuid::Uuid::new_v4()),
            success: true,
        })
    }
}

// ==================== 代理：缓存 ====================

pub struct CachingProxy<S> {
    inner: S,
    cache: Arc<RwLock<HashMap<String, CachedResponse>>>,
    ttl: Duration,
}

struct CachedResponse {
    response: SmsResponse,
    created_at: Instant,
}

impl<S> CachingProxy<S> {
    pub fn new(inner: S, ttl: Duration) -> Self {
        Self {
            inner,
            cache: Arc::new(RwLock::new(HashMap::new())),
            ttl,
        }
    }
}

#[async_trait]
impl<S: SmsService> SmsService for CachingProxy<S> {
    async fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError> {
        // 验证码不缓存
        if matches!(request.template, SmsTemplate::VerificationCode { .. }) {
            return self.inner.send(request).await;
        }

        let key = format!("{}:{:?}", request.phone, request.template);

        // 检查缓存
        {
            let cache = self.cache.read().await;
            if let Some(cached) = cache.get(&key) {
                if cached.created_at.elapsed() < self.ttl {
                    println!("[Cache] Hit for {}", key);
                    return Ok(cached.response.clone());
                }
            }
        }

        println!("[Cache] Miss for {}", key);
        let response = self.inner.send(request).await?;

        // 存入缓存
        self.cache.write().await.insert(key, CachedResponse {
            response: response.clone(),
            created_at: Instant::now(),
        });

        Ok(response)
    }
}

// ==================== 代理：日志 ====================

pub struct LoggingProxy<S> {
    inner: S,
}

impl<S> LoggingProxy<S> {
    pub fn new(inner: S) -> Self {
        Self { inner }
    }
}

#[async_trait]
impl<S: SmsService> SmsService for LoggingProxy<S> {
    async fn send(&self, request: SmsRequest) -> Result<SmsResponse, SmsError> {
        let start = Instant::now();
        println!("[Log] Sending SMS to {}", request.phone);

        let result = self.inner.send(request.clone()).await;

        match &result {
            Ok(resp) => {
                println!("[Log] SMS sent successfully in {:?}, message_id: {}",
                    start.elapsed(), resp.message_id);
            }
            Err(e) => {
                println!("[Log] SMS failed in {:?}: {:?}",
                    start.elapsed(), e);
            }
        }

        result
    }
}

// ==================== 外观 ====================

pub struct NotificationFacade {
    sms_service: Arc<dyn SmsService>,
}

impl NotificationFacade {
    pub fn new(sms_service: Arc<dyn SmsService>) -> Self {
        Self { sms_service }
    }

    pub async fn send_verification_code(&self, phone: &str, code: &str) -> Result<(), SmsError> {
        self.sms_service.send(SmsRequest {
            phone: phone.to_string(),
            template: SmsTemplate::VerificationCode { code: code.to_string() },
        }).await?;
        Ok(())
    }

    pub async fn notify_order(&self, phone: &str, order_id: &str) -> Result<(), SmsError> {
        self.sms_service.send(SmsRequest {
            phone: phone.to_string(),
            template: SmsTemplate::OrderNotification { order_id: order_id.to_string() },
        }).await?;
        Ok(())
    }
}

// ==================== 使用示例 ====================

#[tokio::main]
async fn main() {
    // 构建服务：Mock -> 缓存 -> 日志
    let sms_service: Arc<dyn SmsService> = {
        let mock = MockSmsAdapter;
        let with_cache = CachingProxy::new(mock, Duration::from_secs(60));
        let with_logging = LoggingProxy::new(with_cache);
        Arc::new(with_logging)
    };

    // 使用外观
    let notification = NotificationFacade::new(sms_service);

    // 发送验证码（不缓存）
    notification.send_verification_code("13800138000", "123456").await.unwrap();

    // 发送订单通知（会缓存）
    notification.notify_order("13800138000", "ORD-001").await.unwrap();

    // 再次发送同样的订单通知（命中缓存）
    notification.notify_order("13800138000", "ORD-001").await.unwrap();
}
```

## 小结

与外部系统集成时，三种模式各有用途：

| 模式 | 解决的问题 | Rust 实现 |
|------|-----------|----------|
| 适配器 | 接口不兼容 | trait + 包装类型 |
| 外观 | 子系统太复杂 | 统一入口类 |
| 代理 | 控制访问 | 同接口的包装类型 |

关键洞见：

1. **适配器隔离变化**，第三方 SDK 改版只需要改适配器
2. **外观简化调用**，复杂流程一个方法搞定
3. **代理增强功能**，缓存/重试/熔断/日志层层叠加
4. **依赖抽象**而不是具体实现，方便测试和替换

下一篇，我们看框架如何把控制权交给用户：**框架设计——把控制权还给用户**。模板方法和钩子模式将帮你设计灵活可扩展的框架。

---

*上一篇：[事件驱动——模块间的优雅对话](/posts/design-pattern-not-routine-6-event/)*
*下一篇：[框架设计——把控制权还给用户](/posts/design-pattern-not-routine-8-framework/)*

*本系列：*
1. *[Rust 让设计模式返璞归真](/posts/design-pattern-not-routine-1-intro/)*
2. *[配置系统——从硬编码到热加载](/posts/design-pattern-not-routine-2-config/)*
3. *[插件架构——让系统长出翅膀](/posts/design-pattern-not-routine-3-plugin/)*
4. *[数据管道——像乐高一样组装逻辑](/posts/design-pattern-not-routine-4-pipeline/)*
5. *[状态流转——订单不只是 CRUD](/posts/design-pattern-not-routine-5-state/)*
6. *[事件驱动——模块间的优雅对话](/posts/design-pattern-not-routine-6-event/)*
7. *外部集成——与第三方和平共处（本篇）*
8. *[框架设计——把控制权还给用户](/posts/design-pattern-not-routine-8-framework/)*
9. *[树形世界——递归结构的驯服术](/posts/design-pattern-not-routine-9-tree/)*
10. *[规则引擎——让业务人员写代码](/posts/design-pattern-not-routine-10-rule-engine/)*
11. *[终篇——反模式与断舍离](/posts/design-pattern-not-routine-11-antipattern/)*
