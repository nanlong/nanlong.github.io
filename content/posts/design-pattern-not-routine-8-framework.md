---
title: "模式不是套路（八）：框架设计——把控制权还给用户"
date: 2025-12-10T11:10:00+08:00
draft: false
tags: ["Rust", "设计模式", "框架设计", "模板方法", "钩子模式"]
categories: ["技术"]
description: "写框架不是写业务代码——你定义骨架，用户填充细节。模板方法和钩子模式，让框架既有约束力又有灵活性，用户用得舒服，你也好维护。"
series: ["模式不是套路"]
---

写框架不是写业务代码——你定义骨架，用户填充细节。模板方法和钩子模式，让框架既有约束力又有灵活性，用户用得舒服，你也好维护。

<!--more-->

## 问题：框架的两难

你在设计一个 Web 框架的请求处理流程：

```rust
// 版本 1：完全封闭
pub fn handle_request(request: Request) -> Response {
    // 1. 解析请求
    let parsed = parse_request(request);

    // 2. 验证权限
    if !check_auth(&parsed) {
        return Response::unauthorized();
    }

    // 3. 执行业务逻辑（这里应该调用用户代码）
    let result = ???;  // 用户代码怎么介入？

    // 4. 构建响应
    build_response(result)
}
```

用户没法介入，这不是框架，是库。

```rust
// 版本 2：完全开放
pub trait RequestHandler {
    fn handle(&self, request: Request) -> Response;
}

// 用户实现
impl RequestHandler for MyHandler {
    fn handle(&self, request: Request) -> Response {
        // 用户需要自己处理所有事情
        let parsed = parse_request(request);  // 重复
        if !check_auth(&parsed) {             // 重复
            return Response::unauthorized();
        }
        let result = self.do_business_logic(parsed);
        build_response(result)                // 重复
    }
}
```

用户需要重复写很多样板代码，这是框架的失职。

**框架的艺术在于：定义好骨架，让用户只关注业务逻辑。**

## 破局：模板方法模式

**模板方法模式**的核心是：**定义算法的骨架，将某些步骤延迟到子类**。

### 基础实现

```rust
// 框架定义：请求处理的骨架
pub trait RequestHandler {
    // 模板方法：定义处理流程，不可覆盖
    fn handle(&self, request: Request) -> Response {
        // 1. 解析请求
        let context = self.parse_request(request);

        // 2. 前置检查
        if let Err(e) = self.before_handle(&context) {
            return self.on_error(e);
        }

        // 3. 权限验证
        if let Err(e) = self.check_auth(&context) {
            return self.on_auth_error(e);
        }

        // 4. 执行业务逻辑（用户实现）
        match self.do_handle(&context) {
            Ok(result) => self.build_response(result),
            Err(e) => self.on_error(e),
        }
    }

    // 抽象方法：必须由用户实现
    fn do_handle(&self, context: &RequestContext) -> Result<Value, Error>;

    // 可选钩子：有默认实现，用户可覆盖
    fn parse_request(&self, request: Request) -> RequestContext {
        RequestContext::from_request(request)
    }

    fn before_handle(&self, _context: &RequestContext) -> Result<(), Error> {
        Ok(())  // 默认不做任何事
    }

    fn check_auth(&self, context: &RequestContext) -> Result<(), AuthError> {
        // 默认检查 token
        if context.token.is_none() {
            return Err(AuthError::NoToken);
        }
        Ok(())
    }

    fn build_response(&self, result: Value) -> Response {
        Response::json(result)
    }

    fn on_error(&self, error: Error) -> Response {
        Response::internal_error(error.to_string())
    }

    fn on_auth_error(&self, error: AuthError) -> Response {
        Response::unauthorized()
    }
}
```

用户实现变得简洁：

```rust
// 用户只需要关注业务逻辑
struct GetUserHandler {
    user_repo: Arc<UserRepository>,
}

impl RequestHandler for GetUserHandler {
    fn do_handle(&self, context: &RequestContext) -> Result<Value, Error> {
        let user_id = context.path_param("id")?;
        let user = self.user_repo.find_by_id(user_id)?;
        Ok(serde_json::to_value(user)?)
    }
}

// 需要自定义权限检查的场景
struct AdminOnlyHandler {
    // ...
}

impl RequestHandler for AdminOnlyHandler {
    fn do_handle(&self, context: &RequestContext) -> Result<Value, Error> {
        // 业务逻辑
        Ok(Value::Null)
    }

    // 覆盖默认的权限检查
    fn check_auth(&self, context: &RequestContext) -> Result<(), AuthError> {
        let token = context.token.as_ref().ok_or(AuthError::NoToken)?;
        if !token.is_admin() {
            return Err(AuthError::Forbidden);
        }
        Ok(())
    }
}

// 公开接口，不需要权限检查
struct PublicHandler {
    // ...
}

impl RequestHandler for PublicHandler {
    fn do_handle(&self, context: &RequestContext) -> Result<Value, Error> {
        Ok(Value::String("Hello, public!".to_string()))
    }

    fn check_auth(&self, _context: &RequestContext) -> Result<(), AuthError> {
        Ok(())  // 跳过权限检查
    }
}
```

### Rust 的礼物：trait 默认实现

Rust 的 trait 天然支持模板方法：

```rust
trait RequestHandler {
    // 有默认实现的方法 = 钩子
    fn before_handle(&self, _ctx: &RequestContext) -> Result<(), Error> {
        Ok(())
    }

    // 无默认实现的方法 = 抽象方法
    fn do_handle(&self, ctx: &RequestContext) -> Result<Value, Error>;

    // 调用其他方法的方法 = 模板方法
    fn handle(&self, request: Request) -> Response {
        let ctx = RequestContext::from(request);
        if let Err(e) = self.before_handle(&ctx) {
            return Response::error(e);
        }
        match self.do_handle(&ctx) {
            Ok(result) => Response::json(result),
            Err(e) => Response::error(e),
        }
    }
}
```

比 Java 的抽象类更灵活：

- **多重继承**：一个类型可以实现多个 trait
- **组合优于继承**：用 trait 组合而不是类继承
- **零成本抽象**：静态分发时没有虚函数开销

## 演进：钩子模式

有时候模板方法不够灵活——用户想在**任意位置**插入自定义逻辑。

**钩子模式**提供了更细粒度的扩展点。

### 生命周期钩子

```rust
// 框架定义生命周期钩子
pub trait LifecycleHooks: Send + Sync {
    fn on_start(&self) -> Result<(), Error> { Ok(()) }
    fn on_request_start(&self, _ctx: &mut RequestContext) -> Result<(), Error> { Ok(()) }
    fn on_request_end(&self, _ctx: &RequestContext, _response: &Response) { }
    fn on_error(&self, _error: &Error) { }
    fn on_shutdown(&self) -> Result<(), Error> { Ok(()) }
}

// 框架调用钩子
pub struct Server {
    hooks: Vec<Box<dyn LifecycleHooks>>,
    // ...
}

impl Server {
    pub fn add_hooks(&mut self, hooks: impl LifecycleHooks + 'static) {
        self.hooks.push(Box::new(hooks));
    }

    pub async fn run(&self) -> Result<(), Error> {
        // 调用启动钩子
        for hook in &self.hooks {
            hook.on_start()?;
        }

        // 处理请求
        while let Some(request) = self.accept().await {
            let mut ctx = RequestContext::from(request);

            // 调用请求开始钩子
            for hook in &self.hooks {
                hook.on_request_start(&mut ctx)?;
            }

            // 处理请求
            let response = self.handle_request(&ctx).await;

            // 调用请求结束钩子
            for hook in &self.hooks {
                hook.on_request_end(&ctx, &response);
            }

            self.send_response(response).await;
        }

        // 调用关闭钩子
        for hook in &self.hooks {
            hook.on_shutdown()?;
        }

        Ok(())
    }
}
```

用户可以注册多个钩子：

```rust
// 日志钩子
struct LoggingHooks;

impl LifecycleHooks for LoggingHooks {
    fn on_request_start(&self, ctx: &mut RequestContext) -> Result<(), Error> {
        ctx.set("request_start", Instant::now());
        println!("[{}] {} {}", Utc::now(), ctx.method, ctx.path);
        Ok(())
    }

    fn on_request_end(&self, ctx: &RequestContext, response: &Response) {
        let start: Instant = ctx.get("request_start").unwrap();
        println!("[{}] {} {} -> {} ({:?})",
            Utc::now(), ctx.method, ctx.path, response.status, start.elapsed());
    }
}

// 监控钩子
struct MetricsHooks {
    metrics: Arc<Metrics>,
}

impl LifecycleHooks for MetricsHooks {
    fn on_request_start(&self, _ctx: &mut RequestContext) -> Result<(), Error> {
        self.metrics.increment("requests_total");
        Ok(())
    }

    fn on_error(&self, _error: &Error) {
        self.metrics.increment("errors_total");
    }
}

// 注册钩子
server.add_hooks(LoggingHooks);
server.add_hooks(MetricsHooks::new(metrics));
```

### 回调钩子

用闭包提供更灵活的钩子：

```rust
pub struct ServerBuilder {
    on_start: Option<Box<dyn Fn() -> Result<(), Error> + Send + Sync>>,
    on_request: Option<Box<dyn Fn(&mut RequestContext) -> Result<(), Error> + Send + Sync>>,
    on_response: Option<Box<dyn Fn(&RequestContext, &mut Response) + Send + Sync>>,
    on_error: Option<Box<dyn Fn(&Error) + Send + Sync>>,
}

impl ServerBuilder {
    pub fn new() -> Self {
        Self {
            on_start: None,
            on_request: None,
            on_response: None,
            on_error: None,
        }
    }

    pub fn on_start<F>(mut self, f: F) -> Self
    where
        F: Fn() -> Result<(), Error> + Send + Sync + 'static,
    {
        self.on_start = Some(Box::new(f));
        self
    }

    pub fn on_request<F>(mut self, f: F) -> Self
    where
        F: Fn(&mut RequestContext) -> Result<(), Error> + Send + Sync + 'static,
    {
        self.on_request = Some(Box::new(f));
        self
    }

    pub fn on_response<F>(mut self, f: F) -> Self
    where
        F: Fn(&RequestContext, &mut Response) + Send + Sync + 'static,
    {
        self.on_response = Some(Box::new(f));
        self
    }

    pub fn build(self) -> Server {
        Server {
            on_start: self.on_start,
            on_request: self.on_request,
            on_response: self.on_response,
            on_error: self.on_error,
        }
    }
}

// 使用
let server = ServerBuilder::new()
    .on_start(|| {
        println!("Server starting...");
        Ok(())
    })
    .on_request(|ctx| {
        ctx.set("start_time", Instant::now());
        Ok(())
    })
    .on_response(|ctx, response| {
        response.headers.insert("X-Request-Id", ctx.request_id.clone());
    })
    .build();
```

### 中间件模式

中间件是钩子的一种特殊形式，可以**包装**整个处理流程：

```rust
// 中间件定义
pub trait Middleware: Send + Sync {
    fn handle<'a>(
        &'a self,
        ctx: RequestContext,
        next: Next<'a>,
    ) -> Pin<Box<dyn Future<Output = Response> + Send + 'a>>;
}

// Next 代表后续的处理链
pub struct Next<'a> {
    inner: &'a dyn Fn(RequestContext) -> Pin<Box<dyn Future<Output = Response> + Send>>,
}

impl<'a> Next<'a> {
    pub async fn run(self, ctx: RequestContext) -> Response {
        (self.inner)(ctx).await
    }
}

// 认证中间件
struct AuthMiddleware {
    auth_service: Arc<AuthService>,
}

impl Middleware for AuthMiddleware {
    fn handle<'a>(
        &'a self,
        mut ctx: RequestContext,
        next: Next<'a>,
    ) -> Pin<Box<dyn Future<Output = Response> + Send + 'a>> {
        Box::pin(async move {
            // 前置：验证 token
            match self.auth_service.verify_token(&ctx).await {
                Ok(user) => {
                    ctx.set("current_user", user);
                    // 调用下一个中间件/处理器
                    next.run(ctx).await
                }
                Err(_) => Response::unauthorized(),
            }
        })
    }
}

// 日志中间件
struct LogMiddleware;

impl Middleware for LogMiddleware {
    fn handle<'a>(
        &'a self,
        ctx: RequestContext,
        next: Next<'a>,
    ) -> Pin<Box<dyn Future<Output = Response> + Send + 'a>> {
        Box::pin(async move {
            let start = Instant::now();
            let method = ctx.method.clone();
            let path = ctx.path.clone();

            // 调用下一个
            let response = next.run(ctx).await;

            // 后置：记录日志
            println!("{} {} -> {} ({:?})",
                method, path, response.status, start.elapsed());

            response
        })
    }
}

// 组装中间件链
let app = App::new()
    .middleware(LogMiddleware)           // 最外层
    .middleware(AuthMiddleware::new(auth))
    .middleware(RateLimitMiddleware::new(100))
    .route("/users", get_users_handler)  // 最内层
    .route("/orders", get_orders_handler);
```

## 进阶：策略注入

有时候用户需要替换框架的**核心策略**，而不只是在特定点插入代码。

```rust
// 框架定义可替换的策略
pub trait ErrorStrategy: Send + Sync {
    fn handle(&self, error: Error) -> Response;
}

pub trait SerializationStrategy: Send + Sync {
    fn serialize(&self, value: &Value) -> Result<Vec<u8>, Error>;
    fn content_type(&self) -> &str;
}

pub trait RoutingStrategy: Send + Sync {
    fn match_route(&self, path: &str, method: &str) -> Option<RouteMatch>;
}

// 框架使用策略
pub struct Framework {
    error_strategy: Box<dyn ErrorStrategy>,
    serialization_strategy: Box<dyn SerializationStrategy>,
    routing_strategy: Box<dyn RoutingStrategy>,
}

impl Framework {
    pub fn builder() -> FrameworkBuilder {
        FrameworkBuilder::default()
    }
}

// 默认实现
struct DefaultErrorStrategy;

impl ErrorStrategy for DefaultErrorStrategy {
    fn handle(&self, error: Error) -> Response {
        Response::json(json!({
            "error": error.to_string()
        })).status(500)
    }
}

struct JsonSerializationStrategy;

impl SerializationStrategy for JsonSerializationStrategy {
    fn serialize(&self, value: &Value) -> Result<Vec<u8>, Error> {
        Ok(serde_json::to_vec(value)?)
    }

    fn content_type(&self) -> &str {
        "application/json"
    }
}

// Builder 允许用户替换策略
#[derive(Default)]
pub struct FrameworkBuilder {
    error_strategy: Option<Box<dyn ErrorStrategy>>,
    serialization_strategy: Option<Box<dyn SerializationStrategy>>,
    routing_strategy: Option<Box<dyn RoutingStrategy>>,
}

impl FrameworkBuilder {
    pub fn error_strategy(mut self, strategy: impl ErrorStrategy + 'static) -> Self {
        self.error_strategy = Some(Box::new(strategy));
        self
    }

    pub fn serialization_strategy(mut self, strategy: impl SerializationStrategy + 'static) -> Self {
        self.serialization_strategy = Some(Box::new(strategy));
        self
    }

    pub fn build(self) -> Framework {
        Framework {
            error_strategy: self.error_strategy
                .unwrap_or_else(|| Box::new(DefaultErrorStrategy)),
            serialization_strategy: self.serialization_strategy
                .unwrap_or_else(|| Box::new(JsonSerializationStrategy)),
            routing_strategy: self.routing_strategy
                .unwrap_or_else(|| Box::new(DefaultRoutingStrategy)),
        }
    }
}

// 用户自定义策略
struct ProblemDetailsErrorStrategy;

impl ErrorStrategy for ProblemDetailsErrorStrategy {
    fn handle(&self, error: Error) -> Response {
        // 使用 RFC 7807 Problem Details 格式
        Response::json(json!({
            "type": "about:blank",
            "title": "Internal Server Error",
            "status": 500,
            "detail": error.to_string(),
            "instance": "/errors/internal"
        }))
        .status(500)
        .header("Content-Type", "application/problem+json")
    }
}

// 使用自定义策略
let framework = Framework::builder()
    .error_strategy(ProblemDetailsErrorStrategy)
    .build();
```

## 陷阱：别掉进这些坑

### 陷阱一：模板方法太僵化

```rust
// ❌ 流程完全固定，用户没有选择
trait Handler {
    fn handle(&self, req: Request) -> Response {
        let ctx = self.parse(req);          // 必须
        self.validate(&ctx)?;               // 必须
        self.authorize(&ctx)?;              // 必须
        let result = self.execute(&ctx)?;   // 必须
        self.audit(&ctx, &result);          // 必须
        self.respond(result)                // 必须
    }

    fn execute(&self, ctx: &Context) -> Result<Value, Error>;
    // ... 其他方法
}

// 用户：我不需要 audit，怎么跳过？
```

解决方案：**提供跳过机制**

```rust
// ✅ 允许跳过某些步骤
trait Handler {
    fn handle(&self, req: Request) -> Response {
        let ctx = self.parse(req);

        if self.should_validate() {
            self.validate(&ctx)?;
        }

        if self.should_authorize() {
            self.authorize(&ctx)?;
        }

        let result = self.execute(&ctx)?;

        if self.should_audit() {
            self.audit(&ctx, &result);
        }

        self.respond(result)
    }

    // 控制开关，默认全开
    fn should_validate(&self) -> bool { true }
    fn should_authorize(&self) -> bool { true }
    fn should_audit(&self) -> bool { true }

    fn execute(&self, ctx: &Context) -> Result<Value, Error>;
}
```

### 陷阱二：钩子太多太乱

```rust
// ❌ 钩子泛滥
trait Hooks {
    fn before_parse(&self) { }
    fn after_parse(&self) { }
    fn before_validate(&self) { }
    fn after_validate(&self) { }
    fn before_authorize(&self) { }
    fn after_authorize(&self) { }
    fn before_execute(&self) { }
    fn after_execute(&self) { }
    fn before_respond(&self) { }
    fn after_respond(&self) { }
    // 用户：这些钩子有什么区别？我该用哪个？
}
```

解决方案：**只保留关键钩子**

```rust
// ✅ 只提供最重要的钩子
trait Hooks {
    fn on_request(&self, ctx: &mut Context) -> Result<(), Error> { Ok(()) }
    fn on_response(&self, ctx: &Context, response: &mut Response) { }
    fn on_error(&self, error: &Error) -> Option<Response> { None }
}
```

### 陷阱三：中间件顺序不直观

```rust
// ❌ 中间件顺序让人困惑
app.use(A);  // 先执行还是后执行？
app.use(B);
app.use(C);

// 执行顺序是 A -> B -> C -> handler -> C -> B -> A
// 还是 C -> B -> A -> handler -> A -> B -> C?
```

解决方案：**明确文档 + 调试工具**

```rust
// ✅ 方案 1：命名明确
app.use_before(auth);    // 在 handler 之前执行
app.use_after(logging);  // 在 handler 之后执行

// ✅ 方案 2：提供调试输出
#[cfg(debug_assertions)]
impl App {
    pub fn print_middleware_order(&self) {
        println!("Middleware execution order:");
        for (i, m) in self.middlewares.iter().enumerate() {
            println!("  {}. {}", i + 1, m.name());
        }
    }
}
```

### 陷阱四：扩展点不够用

```rust
// ❌ 用户需要的扩展点不存在
// "我想在数据库事务提交后执行某些逻辑"
// 框架：没有这个钩子
```

解决方案：**事件系统作为补充**

```rust
// ✅ 除了预定义钩子，还支持自定义事件
pub struct Framework {
    event_bus: EventBus,
    // ...
}

impl Framework {
    pub fn emit<E: Event>(&self, event: E) {
        self.event_bus.emit(event);
    }

    pub fn on<E: Event>(&self, handler: impl Fn(&E) + Send + Sync + 'static) {
        self.event_bus.on::<E>(handler);
    }
}

// 用户可以监听任意事件
framework.on::<TransactionCommitted>(|event| {
    // 事务提交后的逻辑
});
```

## 完整示例

```rust
use std::collections::HashMap;
use std::sync::Arc;
use std::time::Instant;

// ==================== 核心类型 ====================

pub struct Request {
    pub method: String,
    pub path: String,
    pub headers: HashMap<String, String>,
    pub body: Vec<u8>,
}

pub struct Response {
    pub status: u16,
    pub headers: HashMap<String, String>,
    pub body: Vec<u8>,
}

impl Response {
    pub fn ok(body: impl Into<Vec<u8>>) -> Self {
        Self {
            status: 200,
            headers: HashMap::new(),
            body: body.into(),
        }
    }

    pub fn error(status: u16, message: &str) -> Self {
        Self {
            status,
            headers: HashMap::new(),
            body: message.as_bytes().to_vec(),
        }
    }
}

pub struct Context {
    pub request: Request,
    pub extensions: HashMap<String, Box<dyn std::any::Any + Send + Sync>>,
}

impl Context {
    pub fn new(request: Request) -> Self {
        Self {
            request,
            extensions: HashMap::new(),
        }
    }

    pub fn set<T: Send + Sync + 'static>(&mut self, key: &str, value: T) {
        self.extensions.insert(key.to_string(), Box::new(value));
    }

    pub fn get<T: 'static>(&self, key: &str) -> Option<&T> {
        self.extensions.get(key)?.downcast_ref()
    }
}

// ==================== 模板方法 ====================

pub trait Handler: Send + Sync {
    // 模板方法：定义请求处理流程
    fn handle(&self, request: Request) -> Response {
        let mut ctx = Context::new(request);

        // 1. 前置钩子
        if let Err(e) = self.before_handle(&mut ctx) {
            return self.on_error(&e);
        }

        // 2. 认证检查
        if self.requires_auth() {
            if let Err(e) = self.check_auth(&ctx) {
                return Response::error(401, "Unauthorized");
            }
        }

        // 3. 执行业务逻辑
        match self.do_handle(&ctx) {
            Ok(response) => {
                // 4. 后置钩子
                self.after_handle(&ctx, &response);
                response
            }
            Err(e) => self.on_error(&e),
        }
    }

    // 抽象方法：必须实现
    fn do_handle(&self, ctx: &Context) -> Result<Response, Box<dyn std::error::Error>>;

    // 钩子方法：有默认实现，可覆盖
    fn before_handle(&self, _ctx: &mut Context) -> Result<(), Box<dyn std::error::Error>> {
        Ok(())
    }

    fn after_handle(&self, _ctx: &Context, _response: &Response) {}

    fn requires_auth(&self) -> bool {
        true
    }

    fn check_auth(&self, ctx: &Context) -> Result<(), Box<dyn std::error::Error>> {
        if ctx.request.headers.get("Authorization").is_some() {
            Ok(())
        } else {
            Err("No auth token".into())
        }
    }

    fn on_error(&self, error: &dyn std::error::Error) -> Response {
        Response::error(500, &error.to_string())
    }
}

// ==================== 中间件 ====================

pub type Next = Box<dyn FnOnce(Context) -> Response + Send>;

pub trait Middleware: Send + Sync {
    fn handle(&self, ctx: Context, next: Next) -> Response;
}

// 日志中间件
pub struct LoggingMiddleware;

impl Middleware for LoggingMiddleware {
    fn handle(&self, mut ctx: Context, next: Next) -> Response {
        let start = Instant::now();
        let method = ctx.request.method.clone();
        let path = ctx.request.path.clone();

        ctx.set("request_start", start);

        let response = next(ctx);

        println!("[LOG] {} {} -> {} ({:?})",
            method, path, response.status, start.elapsed());

        response
    }
}

// 认证中间件
pub struct AuthMiddleware;

impl Middleware for AuthMiddleware {
    fn handle(&self, ctx: Context, next: Next) -> Response {
        if ctx.request.headers.get("Authorization").is_none() {
            return Response::error(401, "Unauthorized");
        }
        next(ctx)
    }
}

// ==================== 框架 ====================

pub struct App {
    middlewares: Vec<Arc<dyn Middleware>>,
    handlers: HashMap<String, Arc<dyn Handler>>,
}

impl App {
    pub fn new() -> Self {
        Self {
            middlewares: Vec::new(),
            handlers: HashMap::new(),
        }
    }

    pub fn middleware(mut self, m: impl Middleware + 'static) -> Self {
        self.middlewares.push(Arc::new(m));
        self
    }

    pub fn route(mut self, path: &str, handler: impl Handler + 'static) -> Self {
        self.handlers.insert(path.to_string(), Arc::new(handler));
        self
    }

    pub fn handle(&self, request: Request) -> Response {
        let path = request.path.clone();

        // 找到对应的 handler
        let handler = match self.handlers.get(&path) {
            Some(h) => Arc::clone(h),
            None => return Response::error(404, "Not Found"),
        };

        // 构建中间件链
        let handler_fn: Next = Box::new(move |ctx| handler.handle(ctx.request));

        let chain = self.middlewares.iter().rev().fold(handler_fn, |next, middleware| {
            let m = Arc::clone(middleware);
            Box::new(move |ctx| m.handle(ctx, next))
        });

        chain(Context::new(request))
    }
}

// ==================== 用户代码 ====================

struct HelloHandler;

impl Handler for HelloHandler {
    fn do_handle(&self, _ctx: &Context) -> Result<Response, Box<dyn std::error::Error>> {
        Ok(Response::ok("Hello, World!"))
    }

    fn requires_auth(&self) -> bool {
        false  // 公开接口，不需要认证
    }
}

struct UserHandler;

impl Handler for UserHandler {
    fn do_handle(&self, ctx: &Context) -> Result<Response, Box<dyn std::error::Error>> {
        Ok(Response::ok(format!("User info for path: {}", ctx.request.path)))
    }

    fn before_handle(&self, ctx: &mut Context) -> Result<(), Box<dyn std::error::Error>> {
        ctx.set("handler_name", "UserHandler".to_string());
        Ok(())
    }

    fn after_handle(&self, ctx: &Context, response: &Response) {
        if let Some(name) = ctx.get::<String>("handler_name") {
            println!("[UserHandler] Completed: {} -> {}", name, response.status);
        }
    }
}

// ==================== 使用示例 ====================

fn main() {
    let app = App::new()
        .middleware(LoggingMiddleware)
        .route("/hello", HelloHandler)
        .route("/user", UserHandler);

    // 模拟请求
    let request = Request {
        method: "GET".to_string(),
        path: "/hello".to_string(),
        headers: HashMap::new(),
        body: vec![],
    };

    let response = app.handle(request);
    println!("Response: {} - {:?}",
        response.status,
        String::from_utf8_lossy(&response.body));

    // 带认证的请求
    let mut headers = HashMap::new();
    headers.insert("Authorization".to_string(), "Bearer token".to_string());

    let request = Request {
        method: "GET".to_string(),
        path: "/user".to_string(),
        headers,
        body: vec![],
    };

    let response = app.handle(request);
    println!("Response: {} - {:?}",
        response.status,
        String::from_utf8_lossy(&response.body));
}
```

## 小结

框架设计的核心是**控制反转**——框架调用用户代码，而不是用户调用框架。

| 模式 | 解决的问题 | Rust 实现 |
|------|-----------|----------|
| 模板方法 | 定义算法骨架 | trait 默认实现 |
| 钩子 | 可选的扩展点 | 有默认实现的 trait 方法 |
| 中间件 | 可组合的处理链 | 闭包 / trait object |
| 策略注入 | 可替换的核心行为 | Builder + trait |

关键洞见：

1. **模板方法定义流程**，用户只需实现关键步骤
2. **钩子提供灵活性**，但不能太多太乱
3. **中间件实现横切关注点**，注意执行顺序
4. **策略注入允许替换核心行为**，保持框架可扩展

下一篇，我们看如何处理树形结构：**树形世界——递归结构的驯服术**。组合、访问者、迭代器、享元，这些模式帮你优雅地处理层级数据。

---

*上一篇：[外部集成——与第三方和平共处](/posts/design-pattern-not-routine-7-integration/)*
*下一篇：[树形世界——递归结构的驯服术](/posts/design-pattern-not-routine-9-tree/)*

*本系列：*
1. *[Rust 让设计模式返璞归真](/posts/design-pattern-not-routine-1-intro/)*
2. *[配置系统——从硬编码到热加载](/posts/design-pattern-not-routine-2-config/)*
3. *[插件架构——让系统长出翅膀](/posts/design-pattern-not-routine-3-plugin/)*
4. *[数据管道——像乐高一样组装逻辑](/posts/design-pattern-not-routine-4-pipeline/)*
5. *[状态流转——订单不只是 CRUD](/posts/design-pattern-not-routine-5-state/)*
6. *[事件驱动——模块间的优雅对话](/posts/design-pattern-not-routine-6-event/)*
7. *[外部集成——与第三方和平共处](/posts/design-pattern-not-routine-7-integration/)*
8. *框架设计——把控制权还给用户（本篇）*
9. *[树形世界——递归结构的驯服术](/posts/design-pattern-not-routine-9-tree/)*
10. *[规则引擎——让业务人员写代码](/posts/design-pattern-not-routine-10-rule-engine/)*
11. *[终篇——反模式与断舍离](/posts/design-pattern-not-routine-11-antipattern/)*
