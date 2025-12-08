---
title: "分布式一致性演进史（七）：实战篇，方案选型与落地"
date: 2025-12-08T21:00:00+08:00
draft: false
tags: ["分布式系统", "分布式锁", "幂等性", "最终一致性", "实战"]
categories: ["技术"]
description: "理论学了一堆，项目里怎么用？这篇我们聊聊真实场景：库存扣减怎么防超卖、分布式锁怎么才安全、跨服务调用怎么保证一致性。"
series: ["分布式一致性演进史"]
---

理论学了一堆，项目里怎么用？这篇我们聊聊真实场景：库存扣减怎么防超卖、分布式锁怎么才安全、跨服务调用怎么保证一致性。

<!--more-->

## 场景一：库存扣减

**问题**：秒杀场景，1000 人同时抢 100 件商品，怎么防止超卖？

### 错误做法：先查后扣

```python
# ❌ 错误：存在竞态条件
stock = db.query("SELECT stock FROM products WHERE id = ?", product_id)
if stock > 0:
    db.execute("UPDATE products SET stock = stock - 1 WHERE id = ?", product_id)
```

**问题**：两个请求同时查到 stock=1，都认为可以扣减，结果扣成 -1。

### 正确做法一：乐观锁

```python
# ✅ 乐观锁：用版本号或条件更新
result = db.execute("""
    UPDATE products
    SET stock = stock - 1
    WHERE id = ? AND stock > 0
""", product_id)

if result.affected_rows == 0:
    raise OutOfStockError()
```

**原理**：把「检查」和「更新」合并成一个原子操作。数据库保证 UPDATE 的原子性。

**适用场景**：冲突概率低、可以接受重试。

### 正确做法二：Redis 预扣减

```python
# ✅ Redis 原子操作
remaining = redis.decr(f"stock:{product_id}")

if remaining < 0:
    # 扣超了，回滚
    redis.incr(f"stock:{product_id}")
    raise OutOfStockError()

# Redis 扣减成功，异步同步到数据库
mq.send("stock_deducted", {"product_id": product_id, "quantity": 1})
```

**原理**：Redis 单线程，DECR 是原子操作。先在 Redis 扣减，再异步同步到数据库。

**适用场景**：高并发秒杀，数据库扛不住。

**注意**：Redis 和数据库之间是最终一致，需要对账机制。

### 正确做法三：分布式锁

```python
# ✅ 分布式锁（适合库存充足时）
lock = redis.lock(f"product:{product_id}", timeout=5)
if lock.acquire():
    try:
        stock = db.query("SELECT stock FROM products WHERE id = ?", product_id)
        if stock > 0:
            db.execute("UPDATE products SET stock = stock - 1 WHERE id = ?", product_id)
        else:
            raise OutOfStockError()
    finally:
        lock.release()
```

**适用场景**：并发不高，需要复杂业务逻辑。

**不适合**：高并发秒杀，锁会成为瓶颈。

### 方案对比

| 方案 | 一致性 | 性能 | 复杂度 | 适用场景 |
|------|--------|------|--------|----------|
| 乐观锁 | 强 | 中 | 低 | 常规库存 |
| Redis 预扣减 | 最终 | 高 | 中 | 秒杀场景 |
| 分布式锁 | 强 | 低 | 中 | 复杂业务 |

## 场景二：分布式锁

**问题**：多个服务实例同时执行定时任务，怎么保证只有一个执行？

### 错误做法：简单 SETNX

```python
# ❌ 错误：锁可能永远不释放
if redis.setnx("job:daily_report", "locked"):
    try:
        run_daily_report()
    finally:
        redis.delete("job:daily_report")
```

**问题**：如果程序在 `run_daily_report()` 中途崩溃，锁永远不会释放。

### 改进做法：带过期时间

```python
# ⚠️ 有坑：非原子操作
if redis.setnx("job:daily_report", "locked"):
    redis.expire("job:daily_report", 300)  # 5分钟过期
    try:
        run_daily_report()
    finally:
        redis.delete("job:daily_report")
```

**问题**：SETNX 和 EXPIRE 不是原子的，中间崩溃还是会死锁。

### 正确做法：SET NX EX

```python
# ✅ 原子操作
lock_value = str(uuid.uuid4())  # 唯一标识
acquired = redis.set(
    "job:daily_report",
    lock_value,
    nx=True,  # 不存在才设置
    ex=300    # 5分钟过期
)

if acquired:
    try:
        run_daily_report()
    finally:
        # ⚠️ 注意：这里仍有竞态条件
        # GET 和 DELETE 之间可能被打断，见下方 Lua 脚本解决方案
        if redis.get("job:daily_report") == lock_value:
            redis.delete("job:daily_report")
```

**为什么要 lock_value？** 防止删除别人的锁：

```
时间线：
  T1: A 获取锁（过期时间 5s）
  T2: A 执行任务（耗时 6s）
  T3: 锁过期，自动释放
  T4: B 获取锁
  T5: A 执行完毕，删除锁 → 删掉了 B 的锁！
```

### 更安全：Lua 脚本

```python
# ✅✅ 最安全：用 Lua 保证原子性
UNLOCK_SCRIPT = """
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
"""

# 释放锁
redis.eval(UNLOCK_SCRIPT, keys=["job:daily_report"], args=[lock_value])
```

**原理**：Lua 脚本在 Redis 中原子执行，「检查」和「删除」不会被打断。

### Redlock：跨机房分布式锁

如果 Redis 是单点，挂了锁就失效。Redlock 算法用多个 Redis 实例：

```
Redlock 流程：
  1. 获取当前时间 T1
  2. 依次向 N 个 Redis 实例请求锁（相同的 key 和 value）
  3. 如果在 N/2+1 个实例上获取成功，且总耗时 < 锁过期时间
     → 获取锁成功
  4. 否则，向所有实例发送释放锁请求
```

**适用场景**：对锁可靠性要求极高。

**注意**：Redlock 有争议，Martin Kleppmann 写过文章质疑其正确性。对于大多数场景，单 Redis 实例 + 合理的过期时间就够了。

## 场景三：跨服务调用一致性

**问题**：订单服务调用库存服务和支付服务，怎么保证一致？

```
用户下单：
  1. 订单服务：创建订单
  2. 库存服务：扣减库存
  3. 支付服务：扣款

如果第 2 步成功，第 3 步失败怎么办？
```

### 方案一：本地消息表

```python
# 订单服务
def create_order(user_id, product_id, amount):
    with db.transaction():
        # 1. 创建订单
        order_id = db.insert("orders", {...})

        # 2. 写入本地消息表（同一个事务）
        db.insert("outbox", {
            "id": uuid4(),
            "topic": "order_created",
            "payload": {"order_id": order_id, "product_id": product_id},
            "status": "pending"
        })

    return order_id

# 后台任务：轮询发送消息
def send_pending_messages():
    messages = db.query("SELECT * FROM outbox WHERE status = 'pending'")
    for msg in messages:
        try:
            mq.send(msg["topic"], msg["payload"])
            db.update("outbox", {"status": "sent"}, id=msg["id"])
        except:
            pass  # 下次重试
```

**原理**：
- 业务操作和消息写入在同一个事务，保证原子性
- 后台任务保证消息最终发送出去
- 消费者需要实现幂等

### 方案二：TCC（Try-Confirm-Cancel）

```python
# 第一阶段：Try（预留资源）
def try_create_order(order_id, product_id, amount):
    # 订单服务：创建订单（状态：待确认）
    order_service.create_order(order_id, status="trying")

    # 库存服务：冻结库存
    inventory_service.freeze_stock(order_id, product_id, 1)

    # 支付服务：冻结余额
    payment_service.freeze_balance(order_id, amount)

# 第二阶段：Confirm（确认提交）
def confirm_order(order_id):
    order_service.confirm_order(order_id)
    inventory_service.deduct_frozen_stock(order_id)
    payment_service.deduct_frozen_balance(order_id)

# 第二阶段：Cancel（回滚）
def cancel_order(order_id):
    order_service.cancel_order(order_id)
    inventory_service.unfreeze_stock(order_id)
    payment_service.unfreeze_balance(order_id)
```

**原理**：
- Try 阶段预留资源，不真正扣减
- 所有 Try 成功后执行 Confirm
- 任何 Try 失败则执行 Cancel

**优点**：资源锁定时间短。

**缺点**：
- 每个服务要实现 Try/Confirm/Cancel 三个接口
- 需要处理空回滚、幂等、悬挂等问题

### 方案三：Saga

```python
# Saga：每个步骤有对应的补偿操作
saga = [
    {"action": create_order, "compensate": cancel_order},
    {"action": deduct_stock, "compensate": restore_stock},
    {"action": deduct_balance, "compensate": refund_balance},
]

def execute_saga(saga, context):
    completed = []
    try:
        for step in saga:
            step["action"](context)
            completed.append(step)
    except Exception as e:
        # 逆序执行补偿
        for step in reversed(completed):
            step["compensate"](context)
        raise e
```

**原理**：
- 每个步骤有对应的补偿操作
- 失败时逆序执行补偿
- 最终达到一致状态

**Saga vs TCC**：

| 维度 | TCC | Saga |
|------|-----|------|
| 资源锁定 | 有（Try 阶段冻结） | 无 |
| 隔离性 | 较好 | 较差（可能看到中间状态） |
| 实现复杂度 | 高 | 中 |
| 适用场景 | 金融交易 | 长流程业务 |

### 方案对比

| 方案 | 一致性 | 性能 | 复杂度 | 适用场景 |
|------|--------|------|--------|----------|
| 本地消息表 | 最终 | 高 | 低 | 大多数场景 |
| TCC | 强 | 中 | 高 | 金融交易 |
| Saga | 最终 | 高 | 中 | 长流程业务 |

**我的建议**：先用本地消息表，真的需要强一致再考虑 TCC。

## 场景四：幂等性设计

**问题**：网络超时重试，怎么防止重复执行？

### 全局唯一 ID

```python
# 客户端生成请求 ID
request_id = str(uuid.uuid4())

def create_order(request_id, user_id, product_id):
    # 检查是否已处理
    existing = db.query(
        "SELECT * FROM orders WHERE request_id = ?",
        request_id
    )
    if existing:
        return existing  # 直接返回之前的结果

    # 正常创建订单
    order = db.insert("orders", {
        "request_id": request_id,
        "user_id": user_id,
        "product_id": product_id
    })
    return order
```

**原理**：用唯一 ID 标识请求，重复请求直接返回之前的结果。

### 状态机

```python
# 订单状态机：只允许合法转换
VALID_TRANSITIONS = {
    "created": ["paid", "cancelled"],
    "paid": ["shipped", "refunded"],
    "shipped": ["delivered", "returned"],
    "delivered": [],  # 终态
    "cancelled": [],  # 终态
}

def update_order_status(order_id, new_status):
    order = db.query("SELECT * FROM orders WHERE id = ?", order_id)

    if new_status not in VALID_TRANSITIONS[order["status"]]:
        raise InvalidTransitionError()

    db.update("orders", {"status": new_status}, id=order_id)
```

**原理**：状态只能单向流转，重复请求不会改变状态。

### 乐观锁 + 版本号

```python
def update_user_balance(user_id, amount, expected_version):
    result = db.execute("""
        UPDATE users
        SET balance = balance + ?, version = version + 1
        WHERE id = ? AND version = ?
    """, amount, user_id, expected_version)

    if result.affected_rows == 0:
        raise ConcurrentModificationError()
```

**原理**：版本号不匹配说明数据已被修改，拒绝重复操作。

## 场景五：配置中心一致性

**问题**：配置更新后，怎么保证所有服务实例都能及时获取？

### 方案一：定期轮询

```python
# 简单但有延迟
def poll_config():
    while True:
        config = config_center.get("app.config")
        update_local_config(config)
        time.sleep(30)  # 30 秒轮询一次
```

**问题**：最长 30 秒延迟，紧急配置变更不友好。

### 方案二：长轮询

```python
# 长轮询：配置变更立即返回
def watch_config():
    last_version = 0
    while True:
        # 阻塞等待，直到版本变化或超时
        result = config_center.watch("app.config", last_version, timeout=60)
        if result.changed:
            update_local_config(result.config)
            last_version = result.version
```

**原理**：服务端 hold 住请求，有变更立即返回。

### 方案三：etcd Watch

```python
# etcd 的 Watch 机制
import etcd3

client = etcd3.client()

def watch_config():
    events, cancel = client.watch("/config/app")
    for event in events:
        if isinstance(event, etcd3.events.PutEvent):
            update_local_config(event.value)
```

**原理**：基于 gRPC 的流式推送，变更实时通知。

### 配置中心选型

| 方案 | 一致性 | 实时性 | 复杂度 |
|------|--------|--------|--------|
| 文件 + 轮询 | 最终 | 秒级 | 低 |
| Apollo | 强 | 秒级 | 中 |
| Nacos | 最终 | 秒级 | 中 |
| etcd | 强 | 毫秒级 | 中 |
| ZooKeeper | 强 | 毫秒级 | 高 |

## 常见踩坑

### 坑一：缓存与数据库不一致

```python
# ❌ 错误：先删缓存再更新数据库
def update_user(user_id, data):
    cache.delete(f"user:{user_id}")
    db.update("users", data, id=user_id)

# 问题：
# T1: A 删除缓存
# T2: B 读取缓存，miss
# T3: B 从数据库读取旧值，写入缓存
# T4: A 更新数据库
# 结果：缓存是旧值，数据库是新值
```

```python
# ✅ 较好：先更新数据库再删缓存
def update_user(user_id, data):
    db.update("users", data, id=user_id)
    cache.delete(f"user:{user_id}")
```

**⚠️ 仍有极端情况**：

```
T1: 线程 A 查询缓存 miss
T2: 线程 A 从数据库读取旧值 V1
T3: 线程 B 更新数据库为 V2
T4: 线程 B 删除缓存
T5: 线程 A 把旧值 V1 写入缓存  ← 脏数据！
```

**解决方案**：

```python
# ✅✅ 延迟双删
def update_user(user_id, data):
    cache.delete(f"user:{user_id}")        # 第一次删
    db.update("users", data, id=user_id)
    time.sleep(0.5)                         # 等待读请求完成
    cache.delete(f"user:{user_id}")        # 第二次删

# 或者：设置较短过期时间，允许短暂不一致
```

### 坑二：分布式锁续期

```python
# ❌ 错误：任务执行时间超过锁过期时间
lock = redis.lock("job", timeout=10)  # 10 秒过期
if lock.acquire():
    run_long_job()  # 执行了 30 秒
    lock.release()  # 锁早就过期了，释放的是别人的锁
```

```python
# ✅ 正确：后台线程续期
class RenewableLock:
    def __init__(self, redis, key, timeout=10):
        self.redis = redis
        self.key = key
        self.timeout = timeout
        self.value = str(uuid.uuid4())
        self._running = False  # 控制续期线程
        self.renew_thread = None

    def acquire(self):
        if self.redis.set(self.key, self.value, nx=True, ex=self.timeout):
            self._running = True
            self._start_renew_thread()
            return True
        return False

    def release(self):
        self._running = False  # 停止续期
        # 用 Lua 脚本原子删除（只删自己的锁）
        script = 'if redis.call("get",KEYS[1])==ARGV[1] then return redis.call("del",KEYS[1]) else return 0 end'
        self.redis.eval(script, keys=[self.key], args=[self.value])

    def _start_renew_thread(self):
        def renew():
            while self._running:  # 用标志变量控制
                time.sleep(self.timeout / 3)
                if self._running:  # 再次检查
                    self.redis.expire(self.key, self.timeout)
        self.renew_thread = threading.Thread(target=renew, daemon=True)
        self.renew_thread.start()
```

### 坑三：消息乱序

```python
# 问题：消息到达顺序可能和发送顺序不同
# 发送：创建订单 → 支付成功 → 发货
# 到达：支付成功 → 创建订单 → 发货  ← 创建订单还没处理，支付成功怎么办？
```

```python
# ✅ 解决方案一：消息带序号，乱序则等待或丢弃
def handle_message(msg):
    expected_seq = get_expected_seq(msg.order_id)
    if msg.seq < expected_seq:
        return  # 已处理，丢弃
    if msg.seq > expected_seq:
        queue_for_later(msg)  # 暂存，等待前序消息
        return
    process(msg)
    increment_expected_seq(msg.order_id)

# ✅ 解决方案二：用状态机保证幂等，乱序无所谓
def handle_payment_success(order_id):
    order = db.query("SELECT * FROM orders WHERE id = ?", order_id)
    if order is None:
        queue_for_later(msg)  # 订单还没创建，稍后重试
        return
    if order.status != "created":
        return  # 状态已变，幂等处理
    update_order_status(order_id, "paid")
```

## 选型决策树

```
                    ┌─────────────────────┐
                    │   一致性要求高吗？   │
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                  │
              ▼                                  ▼
         ┌────────┐                        ┌────────┐
         │   是   │                        │   否   │
         └────┬───┘                        └────┬───┘
              │                                  │
              ▼                                  ▼
    ┌─────────────────────┐          ┌─────────────────────┐
    │  涉及金钱/库存吗？  │          │  最终一致性方案     │
    └──────────┬──────────┘          │  - 本地消息表       │
               │                     │  - 消息队列         │
    ┌──────────┴──────────┐          │  - 缓存 + 过期      │
    │                      │          └─────────────────────┘
    ▼                      ▼
┌────────┐           ┌────────┐
│   是   │           │   否   │
└────┬───┘           └────┬───┘
     │                     │
     ▼                     ▼
┌─────────────┐    ┌─────────────┐
│ TCC / Saga  │    │  分布式锁   │
│ 分布式事务  │    │  乐观锁     │
└─────────────┘    └─────────────┘
```

## 监控与验证

### 关键指标

| 指标 | 说明 | 报警阈值 |
|------|------|----------|
| 主从复制延迟 | MySQL/Redis 复制延迟 | > 1s |
| 消息积压 | 消息队列未消费消息数 | > 10000 |
| 锁等待时间 | 获取分布式锁的平均等待时间 | > 100ms |
| 事务耗时 P99 | 分布式事务 99 分位耗时 | > 500ms |
| 幂等拦截率 | 被幂等机制拦截的重复请求比例 | > 5% 需排查 |

### 数据对账

```python
# 定期对账：比对各系统数据
def daily_reconciliation():
    # 订单系统 vs 库存系统
    orders = order_db.query("SELECT product_id, SUM(quantity) FROM orders GROUP BY product_id")
    inventory = inventory_db.query("SELECT product_id, sold_quantity FROM products")

    for order in orders:
        inv = inventory.get(order.product_id)
        if order.total != inv.sold_quantity:
            alert(f"数据不一致: product={order.product_id}, order={order.total}, inventory={inv.sold_quantity}")
```

## 总结

**实战原则**：

| 原则 | 说明 |
|------|------|
| 能单机就别分布式 | 复杂度是成本 |
| 能最终一致就别强一致 | 强一致代价高 |
| 幂等是基础 | 任何接口都要考虑重试 |
| 监控对账不能少 | 问题早发现早解决 |

**常见场景速查**：

| 场景 | 推荐方案 |
|------|----------|
| 库存扣减 | 乐观锁 / Redis 原子操作 |
| 分布式锁 | Redis SET NX EX + Lua |
| 跨服务事务 | 本地消息表（首选） |
| 配置下发 | etcd Watch / 长轮询 |
| 防重复提交 | 唯一 ID + 状态机 |

---

至此，分布式一致性系列完结。

从理论（CAP、Paxos、CRDT）到实践（分布式锁、消息队列、数据对账），希望这个系列能帮你建立起完整的知识体系。

**记住**：没有银弹，只有权衡。理解问题本质，选择适合业务的方案。

---

*本系列完结：*
1. *[单机到分布式：一致性为何变难](/posts/distributed-consistency-1-why-hard/)*
2. *[2PC 与 CAP：理想的破灭](/posts/distributed-consistency-2-2pc-cap/)*
3. *[Paxos 与 Raft：让多数人达成共识](/posts/distributed-consistency-3-consensus/)*
4. *[最终一致性：不强求，但终会一致](/posts/distributed-consistency-4-eventual/)*
5. *[CRDT：无需协调的合并魔法](/posts/distributed-consistency-5-crdt/)*
6. *[现代方案：从 Spanner 到 TiDB](/posts/distributed-consistency-6-modern/)*
7. *实战篇：方案选型与落地（本篇）*
