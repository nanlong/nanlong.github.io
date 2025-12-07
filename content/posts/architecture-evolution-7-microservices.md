---
title: "架构演进史（七）：微服务与云原生，分布式的代价"
date: 2025-12-08T17:00:00+08:00
draft: false
tags: ["架构", "微服务", "云原生", "Kubernetes", "分布式系统", "Docker"]
categories: ["技术"]
description: "「我们拆成了 50 个微服务！」CTO 自豪地说。半年后，我们花了 60% 的时间处理服务间通信、分布式事务和部署问题。代码量没减少，复杂度却指数级增长。"
series: ["架构演进史"]
---

「我们拆成了 50 个微服务！」CTO 自豪地说。半年后，我们花了 60% 的时间处理服务间通信、分布式事务和部署问题。代码量没减少，复杂度却指数级增长。

<!--more-->

## 前情回顾

在前六篇中，我们构建了一个模块化的单体应用：

- 清晰的业务边界（Context）
- 丰富的领域模型（DDD）
- 良好的依赖方向（六边形架构）
- 可拆分的模块结构（模块化单体）

现在，假设我们真的需要拆分了。让我们看看微服务世界里会遇到什么。

## 分布式系统的八个谬误

1994 年，Sun Microsystems 的工程师们总结了「分布式计算的八个谬误」（最初由 Peter Deutsch 提出前七条，James Gosling 后来补充了第八条）。30 年后，这些谬误依然在收割新手。

### 谬误 1：网络是可靠的

**你以为**：调用另一个服务，就像调用本地函数。

**现实**：

```text
Service A ──────► Service B
            │
            │ 可能发生：
            ├─ 网络超时
            ├─ 连接被拒绝
            ├─ 服务不可用
            ├─ 响应丢失
            └─ 重复请求
```

**代码层面的变化**：

```rust
// 单体时代：简单调用
let user = user_service.get_user(user_id)?;

// 微服务时代：需要处理各种异常
let user = retry(3, || async {
    match http_client.get(&format!("{}/users/{}", user_service_url, user_id))
        .timeout(Duration::from_secs(5))
        .send()
        .await
    {
        Ok(resp) if resp.status().is_success() => {
            Ok(resp.json::<User>().await?)
        }
        Ok(resp) => {
            Err(ServiceError::HttpError(resp.status()))
        }
        Err(e) if e.is_timeout() => {
            Err(ServiceError::Timeout)
        }
        Err(e) => {
            Err(ServiceError::NetworkError(e))
        }
    }
}).await?;
```

**原来简洁的调用，现在需要处理各种网络异常。** 虽然代码量的增加因封装程度而异（可以用库抽象），但错误处理的复杂度是真实存在的。

### 谬误 2：延迟为零

**你以为**：服务调用很快。

**现实**：

| 操作 | 延迟 |
|------|------|
| 本地函数调用 | ~10 纳秒 |
| 同机房 HTTP 调用 | ~1 毫秒 |
| 跨机房 HTTP 调用 | ~10-100 毫秒 |
| 跨地域 HTTP 调用 | ~100-500 毫秒 |

**一个请求链路**：

```text
用户 → Gateway → Auth → Orders → Inventory → Payment
         ↑         ↑       ↑         ↑          ↑
        10ms     20ms    30ms      40ms       50ms

总延迟 = 10 + 20 + 30 + 40 + 50 = 150ms
```

单体里同样的逻辑可能只需要 10ms。

### 谬误 3：带宽是无限的

**你以为**：传多少数据都没关系。

**现实**：

```json
// 微服务 A 返回用户信息
{
  "id": "user_123",
  "name": "Alice",
  "email": "alice@example.com",
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-02T00:00:00Z",
  "profile": { ... },      // 其实不需要
  "preferences": { ... },  // 其实不需要
  "history": [ ... ]       // 其实不需要
}
```

返回了 10KB 数据，其实只需要 `name`。

乘以每秒 10000 次调用 = 每秒 100MB 不必要的网络流量。

### 谬误 4：网络是安全的

**你以为**：内网环境，不需要额外安全措施。

**现实**：
- 服务间通信可能被窃听
- 恶意服务可能冒充合法服务
- 零信任网络：即使在内网也不能信任

**需要**：
- mTLS（双向 TLS）
- 服务认证
- 请求签名
- 访问控制

### 谬误 5：拓扑结构不变

**你以为**：服务地址是固定的。

**现实**：
- 服务会扩缩容
- 服务会迁移
- 服务会故障转移

**需要**：服务发现（Consul、etcd、Kubernetes Service）

```yaml
# Kubernetes 里，Service 地址是动态的
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 8080
```

### 谬误 6：只有一个管理员

**你以为**：我们团队管理所有服务。

**现实**：
- 不同团队管理不同服务
- 不同服务有不同的发布节奏
- 版本不兼容问题频发

### 谬误 7：传输成本为零

**你以为**：网络传输不花钱。

**现实**：
- 云服务商按流量收费
- 跨区域传输更贵
- 序列化/反序列化消耗 CPU

### 谬误 8：网络是同构的

**你以为**：所有服务用同样的协议。

**现实**：
- 有的服务用 REST
- 有的服务用 gRPC
- 有的服务用 GraphQL
- 有的服务用消息队列

## 分布式事务：最难的问题

微服务最痛苦的问题之一：**跨服务事务**。

### 单体时代

```rust
async fn create_order(cmd: CreateOrderCommand) -> Result<OrderId> {
    let tx = db.begin().await?;

    // 所有操作在一个事务里
    let order = create_order_record(&tx, &cmd).await?;
    decrease_inventory(&tx, &cmd.items).await?;
    create_payment(&tx, order.id, cmd.amount).await?;

    tx.commit().await?;  // 要么全成功，要么全失败
    Ok(order.id)
}
```

简单、可靠、原子性。

### 微服务时代

```text
                      ┌──────────────┐
                      │   Orders     │
                      │   Service    │
                      └──────┬───────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌────────────────┐ ┌────────────────┐ ┌────────────────┐
│  Inventory     │ │   Payment      │ │  Notification  │
│  Service       │ │   Service      │ │  Service       │
└────────────────┘ └────────────────┘ └────────────────┘
```

**问题**：如果 Payment 失败了，怎么回滚 Inventory？

### 方案 1：两阶段提交（2PC）

```text
Phase 1: Prepare（准备）
协调者 → 所有参与者: "准备好了吗？"
所有参与者 → 协调者: "准备好了" / "没准备好"

Phase 2: Commit/Abort（提交/回滚）
如果都准备好了 → 协调者: "提交！"
如果有人没准备好 → 协调者: "回滚！"
```

**问题**：
- 性能差（需要多次网络往返）
- 协调者单点故障
- 参与者可能长时间锁定资源

### 方案 2：Saga 模式

Saga 是一系列本地事务，每个事务都有对应的**补偿操作**。

```text
正向流程：
创建订单 → 扣减库存 → 创建支付 → 发送通知
    T1        T2          T3         T4

补偿流程（如果 T3 失败）：
                    T3 失败
                       │
                       ▼
              恢复库存 (C2) ← 取消订单 (C1)
```

**代码实现**：

```rust
// 编排式 Saga
async fn create_order_saga(cmd: CreateOrderCommand) -> Result<OrderId> {
    // T1: 创建订单（状态：待确认）
    let order = orders_service.create_order(&cmd).await?;

    // T2: 扣减库存
    match inventory_service.decrease_stock(&cmd.items).await {
        Err(e) => {
            // C1: 补偿 - 取消订单
            orders_service.cancel_order(order.id).await?;
            return Err(e);
        }
        Ok(_) => {}
    }

    // T3: 创建支付
    match payment_service.create_payment(order.id, cmd.amount).await {
        Err(e) => {
            // C2: 补偿 - 恢复库存
            inventory_service.restore_stock(&cmd.items).await?;
            // C1: 补偿 - 取消订单
            orders_service.cancel_order(order.id).await?;
            return Err(e);
        }
        Ok(_) => {}
    }

    // T4: 确认订单
    orders_service.confirm_order(order.id).await?;

    Ok(order.id)
}
```

**问题**：
- 代码复杂（要写正向和补偿两套逻辑）
- 补偿可能失败（怎么办？重试？人工介入？）
- 中间状态对用户可见（用户可能看到「已扣款但订单取消」）

### 方案 3：最终一致性

放弃强一致性，接受**最终一致性**。

```text
Orders Service:
1. 创建订单（状态：待处理）
2. 发布事件: OrderCreated

Inventory Service:
1. 订阅 OrderCreated
2. 扣减库存
3. 发布事件: InventoryDecreased

Payment Service:
1. 订阅 InventoryDecreased
2. 创建支付
3. 发布事件: PaymentCreated

Orders Service:
1. 订阅 PaymentCreated
2. 更新订单状态为「已确认」
```

**好处**：服务解耦，各自处理各自的事务。

**问题**：
- 调试困难（事件链路追踪）
- 数据不一致窗口期
- 需要处理消息丢失、重复

## 可观测性：你看得见你的系统吗？

单体时代，一个日志文件、一个监控面板就够了。

微服务时代，你需要：

### 1. 分布式追踪

```text
Trace ID: abc123

Gateway (10ms)
  └── Auth Service (20ms)
        └── User Service (15ms)
              └── Database (5ms)
  └── Orders Service (50ms)
        ├── Inventory Service (20ms)
        │     └── Database (10ms)
        └── Payment Service (30ms)
              └── External API (25ms)
```

工具：Jaeger、Zipkin、OpenTelemetry

### 2. 集中式日志

```text
所有服务的日志 → Fluentd/Filebeat → Elasticsearch → Kibana

然后你可以搜索：
"error AND service:orders AND trace_id:abc123"
```

工具：ELK Stack、Grafana Loki

### 3. 指标监控

```text
每个服务暴露：
- 请求量（QPS）
- 错误率
- 延迟（P50, P95, P99）
- 资源使用（CPU, 内存）
```

工具：Prometheus + Grafana

### 4. 告警

```yaml
# 当订单服务错误率超过 5% 时告警
- alert: OrderServiceHighErrorRate
  expr: rate(http_requests_total{service="orders", status="5xx"}[5m])
        / rate(http_requests_total{service="orders"}[5m]) > 0.05
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "订单服务错误率过高"
```

**没有可观测性的微服务 = 盲人摸象。**

## 容器与 Kubernetes

微服务需要一个运行环境。Docker 和 Kubernetes 成了标配。

### Docker：解决「在我机器上能跑」

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/orders-service /usr/local/bin/
CMD ["orders-service"]
```

**好处**：
- 环境一致性（开发、测试、生产用同样的镜像）
- 隔离性（不同服务的依赖不冲突）
- 可移植性（在哪都能跑）

### Kubernetes：解决「大规模管理容器」

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
spec:
  replicas: 3  # 运行 3 个副本
  selector:
    matchLabels:
      app: orders-service
  template:
    spec:
      containers:
      - name: orders-service
        image: myregistry/orders-service:v1.2.3
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
```

Kubernetes 提供：
- **自动扩缩容**：根据负载自动增减副本
- **自愈**：容器挂了自动重启
- **滚动更新**：无停机部署
- **服务发现**：自动注册和发现服务
- **负载均衡**：自动分发流量

### Kubernetes 的复杂度

但 Kubernetes 本身也很复杂：

| 概念 | 作用 |
|------|------|
| Pod | 最小部署单元 |
| Deployment | 管理 Pod 的副本 |
| Service | 服务发现和负载均衡 |
| Ingress | 外部流量入口 |
| ConfigMap | 配置管理 |
| Secret | 敏感信息管理 |
| PersistentVolume | 持久化存储 |
| NetworkPolicy | 网络安全策略 |
| ServiceAccount | 身份认证 |
| RBAC | 权限控制 |

**学会这些，需要时间投入。自建集群运维好这些，通常需要专职团队。** 当然，你也可以选择云厂商的托管 Kubernetes（如 EKS、GKE、AKS），把控制平面的运维交给云厂商。

> **Docker 让部署变简单，Kubernetes 在提供强大能力的同时也带来了学习曲线。** 是否值得取决于你的规模和需求。

## Serverless：更进一步

如果你不想管服务器，还有 Serverless：

```javascript
// AWS Lambda
exports.handler = async (event) => {
    const orderId = event.pathParameters.orderId;
    const order = await getOrder(orderId);
    return {
        statusCode: 200,
        body: JSON.stringify(order)
    };
};
```

**好处**：
- 按请求计费（没流量不花钱）
- 自动扩缩容（从 0 到 10000 并发）
- 不用管服务器

**问题**：
- 冷启动延迟（第一次请求可能要几秒）
- 供应商锁定
- 调试困难
- 复杂业务难以实现

## 微服务的真正价值

说了这么多「代价」，微服务到底有什么价值？

### 价值 1：团队自治

```text
Team A (用户组) ──────► User Service
                       - 独立开发
                       - 独立部署
                       - 独立运维

Team B (订单组) ──────► Order Service
                       - 独立开发
                       - 独立部署
                       - 独立运维
```

两个团队可以用不同的语言、不同的框架、不同的发布节奏。

**康威定律的逆向应用**：用架构来支持组织结构。

### 价值 2：独立扩展

```text
高峰期：
User Service:   2 个实例
Order Service: 20 个实例 ← 只扩展需要的
Payment Service: 5 个实例
```

不用为了订单的流量扩展整个系统。

### 价值 3：故障隔离

```text
Payment Service 挂了

单体：整个系统不可用
微服务：只有支付功能不可用，浏览、下单可以降级处理
```

### 价值 4：技术多样性

```text
User Service:   Go（高并发）
Order Service:  Rust（高性能）
ML Service:     Python（机器学习）
Admin Service:  Node.js（快速开发）
```

为每个问题选择最合适的工具。

## 决策框架

什么时候选择微服务？

```text
┌─────────────────────────────────────────────────────────┐
│                    决策矩阵                              │
├─────────────────┬───────────────┬───────────────────────┤
│     维度        │    单体       │       微服务          │
├─────────────────┼───────────────┼───────────────────────┤
│ 团队规模        │ < 20 人       │ > 50 人               │
│ 部署频率        │ 周/月         │ 每天/每小时           │
│ 扩展需求        │ 整体扩展      │ 独立扩展              │
│ 技术栈          │ 统一          │ 多样                  │
│ 故障隔离        │ 不需要        │ 需要                  │
│ 运维能力        │ 基础          │ DevOps 成熟           │
└─────────────────┴───────────────┴───────────────────────┘
```

**如果你的答案大多在「单体」列，不要选微服务。**

## 总结

**微服务的本质**：

> **把代码复杂度换成了基础设施复杂度。**

| 方面 | 单体 | 微服务 |
|------|------|--------|
| 代码复杂度 | 高（都在一起） | 低（分散各服务） |
| 基础设施复杂度 | 低 | 高 |
| 团队协作 | 需要协调 | 可以独立 |
| 运维负担 | 低 | 高 |
| 适合场景 | 小团队、简单业务 | 大团队、复杂业务 |

**分布式系统的代价**：
- 网络不可靠
- 延迟不可忽略
- 事务很难
- 可观测性是必需品
- 运维复杂度指数级增长

**核心教训**：

> **微服务不是目标，独立部署和团队自治才是。**

如果你的团队不需要独立部署，不需要独立扩展，不需要技术多样性——**不要用微服务**。

下一篇，我们来总结整个系列——架构决策的本质是什么？

---

*上一篇：[单体的边界：何时该拆？](/posts/architecture-evolution-6-modular-monolith/)*

*下一篇：[没有银弹：架构决策的本质](/posts/architecture-evolution-8-no-silver-bullet/)*

*本系列：*
1. *[混沌时代：当代码没有架构](/posts/architecture-evolution-1-chaos/)*
2. *[MVC 启蒙：框架带来的秩序](/posts/architecture-evolution-2-mvc/)*
3. *[Context 之道：从技术分层到业务分层](/posts/architecture-evolution-3-context/)*
4. *[DDD 觉醒：让代码说业务的语言](/posts/architecture-evolution-4-ddd/)*
5. *[边界的艺术：六边形与洋葱](/posts/architecture-evolution-5-hexagonal/)*
6. *[单体的边界：何时该拆？](/posts/architecture-evolution-6-modular-monolith/)*
7. *微服务与云原生：分布式的代价（本篇）*
8. *[没有银弹：架构决策的本质](/posts/architecture-evolution-8-no-silver-bullet/)*
