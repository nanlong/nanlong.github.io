---
title: "当单机 PostgreSQL 撑不住时：Citus 分布式实战"
date: 2025-12-11T12:00:00+08:00
draft: false
tags: ["PostgreSQL", "Citus", "分布式数据库", "分片", "Rust"]
categories: ["技术"]
description: "数据量暴增，单机 PostgreSQL 快撑不住了？Citus 让你用熟悉的 SQL 操作分布式数据库，代码几乎不用改"
---

## 你的数据库快爆了

凌晨 3 点，报警短信把你吵醒：**数据库 CPU 100%，响应超时**。

你揉着眼睛打开监控，订单表已经 2 亿行，每次查询都在全表扫描。加索引？早加过了。分表？业务代码要大改。

DBA 说："该分库分表了。"

你打开《一致性实战（四）》，看到分片路由、跨分片查询聚合、Saga 事务补偿... 头皮发麻。

**有没有更简单的方案？**

有。让我介绍 Citus——PostgreSQL 的分布式扩展。它让分片对应用**几乎透明**，大部分 SQL 不用改，ORM 和 sqlx 可以直接用。

## Citus 是什么

想象你有一个图书馆，书越来越多，一个管理员忙不过来了。

**传统方案**（手动分片）：
> 你把书分到 4 个房间，每个房间一个管理员。但你得记住每本书在哪个房间，跨房间借书要自己协调。

**Citus 方案**：
> 还是 4 个房间 4 个管理员，但前台有一个**总调度**。你只跟总调度说话，它自动知道书在哪，跨房间的事它帮你协调。

这就是 Citus 的架构：

```text
┌─────────────────────────────────────────────────────────┐
│                    应用程序                              │
│              (sqlx, Diesel, 任何 PG 客户端)              │
└─────────────────────┬───────────────────────────────────┘
                      │ 普通 PostgreSQL 协议
                      ▼
┌─────────────────────────────────────────────────────────┐
│                 Coordinator (协调器)                     │
│            看起来就是一个普通 PostgreSQL                  │
│         • 解析 SQL                                       │
│         • 路由到正确的分片                                │
│         • 聚合跨分片结果                                  │
└───────┬─────────────┬─────────────┬─────────────────────┘
        │             │             │
        ▼             ▼             ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │ Worker1 │   │ Worker2 │   │ Worker3 │
   │ (分片0-3)│   │ (分片4-7)│   │(分片8-11)│
   └─────────┘   └─────────┘   └─────────┘
```

**关键点**：应用只连接 Coordinator，它看起来、用起来就是一个普通 PostgreSQL。

## 五分钟上手

### 1. 启动 Citus 集群

用 Docker Compose 快速启动一个本地集群：

```yaml
# docker-compose.yml
services:
  coordinator:
    image: citusdata/citus:12.1
    container_name: citus_coordinator
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: mydb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - citus-network

  worker1:
    image: citusdata/citus:12.1
    container_name: citus_worker1
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - citus-network

  worker2:
    image: citusdata/citus:12.1
    container_name: citus_worker2
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - citus-network

networks:
  citus-network:
    driver: bridge
```

启动集群并等待健康检查通过：

```bash
docker-compose up -d
docker-compose ps  # 确认所有服务状态为 healthy
```

### 2. 注册 Worker 节点

连接到 Coordinator，注册 Worker：

```bash
# 连接到 coordinator（使用刚才配置的数据库）
psql -h localhost -U postgres -d mydb
```

```sql
-- 启用 Citus 扩展
CREATE EXTENSION IF NOT EXISTS citus;

-- 注册 worker 节点（使用 Docker 容器名）
SELECT citus_add_node('citus_worker1', 5432);
SELECT citus_add_node('citus_worker2', 5432);

-- 验证集群状态
SELECT * FROM citus_get_active_worker_nodes();
```

如果看到两个 worker 节点，说明集群已就绪。

### 3. 创建分布式表

这是 Citus 的核心——**一行命令让表变成分布式**：

```sql
-- 创建普通表（和以前一样）
CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    book_id BIGINT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, id)  -- 分片键必须在主键中
);

-- 魔法发生：将表按 user_id 分片
SELECT create_distributed_table('orders', 'user_id');
```

**就这样**。从现在开始，你的 SQL 和以前一样写，Citus 自动处理分片路由。

### 4. 像单机一样使用

```sql
-- 插入（自动路由到正确分片）
INSERT INTO orders (user_id, book_id, amount)
VALUES (123, 456, 99.99);

-- 单用户查询（单分片，毫秒级）
SELECT * FROM orders WHERE user_id = 123;

-- 跨分片聚合（Citus 自动并行查询所有分片）
SELECT COUNT(*), SUM(amount)
FROM orders
WHERE created_at > '2024-01-01';

-- JOIN（共置表之间的 JOIN 在单分片完成，详见下文 Co-location）
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.user_id = 123;
```

## 三种表类型

Citus 有三种表类型，选对类型是性能的关键：

### 分布式表（Distributed Table）

大表按分片键分散到各 Worker：

```sql
-- 订单表：按 user_id 分片（每个用户的订单在同一分片）
SELECT create_distributed_table('orders', 'user_id');

-- 用户表：按 id 分片（与 orders.user_id 对齐）
SELECT create_distributed_table('users', 'id');
```

**什么时候用**：数据量大、需要水平扩展的表。

### 引用表（Reference Table）

小表**完整复制**到每个 Worker：

```sql
-- 分类表：数据量小，所有节点都需要
SELECT create_reference_table('categories');

-- 配置表：同理
SELECT create_reference_table('system_config');
```

**什么时候用**：
- 数据量小（< 10 万行）
- 经常被 JOIN
- 很少更新

**好处**：JOIN 引用表时，数据在本地，不需要跨节点传输。

### 本地表（Local Table）

只存在于 Coordinator，不分布：

```sql
-- 普通 CREATE TABLE，不调用 create_distributed_table
CREATE TABLE admin_logs (
    id SERIAL PRIMARY KEY,
    action TEXT,
    created_at TIMESTAMP
);
```

**什么时候用**：管理类数据、不需要扩展的小表。

### 三种表对比

| 类型 | 数据位置 | 适用场景 | 示例 |
|------|---------|---------|------|
| 分布式表 | 分散在各 Worker | 大表、主业务表 | orders, users, products |
| 引用表 | 复制到所有节点 | 小表、频繁 JOIN | categories, regions, config |
| 本地表 | 仅在 Coordinator | 管理数据 | admin_logs, migrations |

## 分片键：最重要的决策

分片键选错，Citus 救不了你。

### 好的分片键

```sql
-- 多租户 SaaS：按 tenant_id 分片
SELECT create_distributed_table('tenants', 'id');
SELECT create_distributed_table('users', 'tenant_id', colocate_with => 'tenants');
SELECT create_distributed_table('orders', 'tenant_id', colocate_with => 'tenants');

-- 好处：同一租户的所有数据在同一分片，查询是本地的
SELECT * FROM orders o
JOIN users u ON o.tenant_id = u.tenant_id AND o.user_id = u.id
WHERE o.tenant_id = 1001;  -- 单分片完成！
```

### 差的分片键

```sql
-- 按 order_id 分片
SELECT create_distributed_table('orders', 'id');

-- 问题：查用户订单要扫描所有分片！
SELECT * FROM orders WHERE user_id = 123;  -- 跨所有分片
```

### 分片键选择原则

| 原则 | 说明 | 示例 |
|------|------|------|
| **高频过滤条件** | 大部分查询都按这个字段过滤 | user_id, tenant_id |
| **JOIN 关联字段** | 相关表用相同分片键 | orders.user_id = users.id |
| **高基数** | 值要足够多，分布均匀 | user_id（百万级）vs status（3 个值）|
| **不常变更** | 分片键变更 = 数据迁移 | user_id（稳定）vs phone（可能换）|

### Co-location：让 JOIN 变快

Co-location（共置）是 Citus 性能优化的关键。**同一分片键值的数据在同一物理节点**，JOIN 才能在本地完成。

**重要**：Co-location 需要显式配置，不是自动的！

```sql
-- 第一个表：创建时指定分片数（默认 32）
SELECT create_distributed_table('users', 'id');

-- 后续表：使用 colocate_with 参数确保共置
SELECT create_distributed_table('orders', 'user_id', colocate_with => 'users');
SELECT create_distributed_table('order_items', 'user_id', colocate_with => 'users');
```

共置后，以下 JOIN 完全在单分片内完成：

```sql
SELECT u.name, o.id, oi.product_name
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id AND o.user_id = oi.user_id
WHERE u.id = 123;  -- 单分片查询，性能极好
```

**Co-location 的条件**：
- 分片键类型相同（都是 BIGINT）
- 分片数相同
- 使用 `colocate_with` 参数或在同一 colocation group

**验证共置状态**：

```sql
SELECT logicalrelid, colocationid
FROM pg_dist_partition
WHERE logicalrelid IN ('users'::regclass, 'orders'::regclass);
-- colocationid 相同表示已共置
```

**注意**：`order_items` 表的主键应该是 `(user_id, order_id, id)`，分片键必须出现在所有唯一约束中。

## 跨分片查询：Citus 的魔法

当查询需要多个分片时，Citus 自动处理：

```sql
-- 统计所有用户的订单总额
SELECT user_id, SUM(amount) as total
FROM orders
WHERE created_at > '2024-01-01'
GROUP BY user_id
ORDER BY total DESC
LIMIT 10;
```

Citus 会：
1. 将查询发送到所有 Worker
2. 每个 Worker 计算本地结果
3. Coordinator 聚合最终结果

**但是**，跨分片查询比单分片查询慢。如果你的应用 90% 的查询都跨分片，说明**分片键选错了**。

### 监控跨分片查询

```sql
-- 查看查询执行计划
EXPLAIN SELECT * FROM orders WHERE user_id = 123;
-- 显示：单分片查询

EXPLAIN SELECT * FROM orders WHERE created_at > '2024-01-01';
-- 显示：跨所有分片查询
```

## 跨分片事务

Citus 支持跨分片事务，使用 **2PC（两阶段提交）**：

```sql
BEGIN;

-- 用户 A（分片 1）转账给用户 B（分片 2）
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;

COMMIT;  -- Citus 自动用 2PC 保证原子性
```

**但是**，2PC 有性能开销。如果可以，尽量设计成单分片事务。

### 什么时候需要应用层 Saga？

即使用了 Citus，以下场景仍需要应用层处理：

| 场景 | Citus 2PC | 应用层 Saga |
|------|-----------|------------|
| 跨分片转账 | ✅ 可以 | 也可以 |
| 跨服务调用（如调用支付网关）| ❌ 不行 | ✅ 必须 |
| 需要补偿逻辑 | ❌ 不行 | ✅ 必须 |
| 长事务（> 几秒）| ❌ 性能差 | ✅ 推荐 |

**结论**：Citus 2PC 适合纯数据库操作；涉及外部服务或需要补偿逻辑时，仍需 Saga。

## 在线扩容

业务增长，需要加节点？Citus 让这变得简单：

```sql
-- 1. 添加新 Worker（如果是 Docker 环境，使用容器名）
SELECT citus_add_node('citus_worker3', 5432);

-- 2. 重新平衡分片（在线进行，不停服务）
SELECT rebalance_table_shards();

-- 3. 查看分片分布
SELECT * FROM citus_shards;
```

Citus 会自动将部分分片从旧节点迁移到新节点，整个过程**应用无感知**。

## Rust + sqlx 集成

Citus 对应用透明，sqlx 代码几乎不用改：

```rust
use chrono::{DateTime, Utc};
use rust_decimal::Decimal;
use sqlx::{FromRow, PgPool};

/// 订单实体
#[derive(Debug, FromRow)]
pub struct Order {
    pub id: i64,
    pub user_id: i64,
    pub book_id: i64,
    pub amount: Decimal,
    pub status: String,
    pub created_at: DateTime<Utc>,
}

pub struct OrderRepository {
    pool: PgPool,  // 连接到 Citus Coordinator
}

impl OrderRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }

    /// 创建订单（自动路由到正确分片）
    pub async fn create_order(
        &self,
        user_id: i64,
        book_id: i64,
        amount: Decimal,
    ) -> Result<i64, sqlx::Error> {
        let record = sqlx::query_scalar!(
            r#"
            INSERT INTO orders (user_id, book_id, amount)
            VALUES ($1, $2, $3)
            RETURNING id
            "#,
            user_id,
            book_id,
            amount,
        )
        .fetch_one(&self.pool)
        .await?;

        Ok(record)
    }

    /// 查询用户订单（单分片查询）
    pub async fn get_user_orders(&self, user_id: i64) -> Result<Vec<Order>, sqlx::Error> {
        sqlx::query_as!(
            Order,
            r#"
            SELECT id, user_id, book_id, amount, status, created_at
            FROM orders
            WHERE user_id = $1
            ORDER BY created_at DESC
            "#,
            user_id
        )
        .fetch_all(&self.pool)
        .await
    }

    /// 统计订单总额（跨分片聚合）
    pub async fn get_total_amount_since(
        &self,
        since: DateTime<Utc>,
    ) -> Result<Decimal, sqlx::Error> {
        let total = sqlx::query_scalar!(
            r#"SELECT COALESCE(SUM(amount), 0) as "total!" FROM orders WHERE created_at > $1"#,
            since
        )
        .fetch_one(&self.pool)
        .await?;

        Ok(total)
    }
}
```

**说明**：
- 使用 `"total!"` 语法告诉 sqlx 该字段非空（因为有 `COALESCE`）
- 显式列出 SELECT 字段，避免 `SELECT *` 带来的类型推断问题

**关键点**：代码和单机 PostgreSQL 完全一样。唯一的区别是连接字符串指向 Coordinator。

### 连接池配置

```rust
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(100)  // 根据 Coordinator 配置调整
    .connect("postgres://user:pass@coordinator:5432/mydb")
    .await?;
```

## sqlx 数据库迁移

使用 sqlx 迁移时，需要处理 Citus 特有的表分布式化操作。好消息是：**大部分 DDL 自动传播，只有建表时需要特殊处理**。

### 迁移文件结构

```text
migrations/
├── 20241201000001_create_users.sql      ← 建表 + 分布式化
├── 20241201000002_create_orders.sql     ← 建表 + 分布式化
├── 20241201000003_add_user_phone.sql    ← 普通 ALTER，自动传播
└── 20241201000004_add_order_index.sql   ← 普通 INDEX，自动传播
```

### 建表迁移（需要分布式化）

```sql
-- migrations/20241201000001_create_users.sql

-- 创建表（分片键必须在主键中）
CREATE TABLE users (
    id BIGSERIAL,
    tenant_id BIGINT NOT NULL,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (tenant_id, id)
);

CREATE INDEX idx_users_email ON users(email);

-- 分布式化（兼容无 Citus 的开发环境）
DO $$
BEGIN
    IF EXISTS (SELECT 1 FROM pg_extension WHERE extname = 'citus') THEN
        IF NOT EXISTS (
            SELECT 1 FROM pg_catalog.pg_dist_partition
            WHERE logicalrelid = 'users'::regclass
        ) THEN
            PERFORM create_distributed_table('users', 'tenant_id');
        END IF;
    END IF;
END $$;
```

**关键点**：
- `IF EXISTS (SELECT 1 FROM pg_extension WHERE extname = 'citus')` — 兼容无 Citus 的本地开发环境
- `IF NOT EXISTS ... pg_dist_partition` — 幂等检查，避免重复分布式化报错

### 修改表迁移（无需特殊处理）

后续的 `ALTER TABLE`、`CREATE INDEX` 等 DDL **自动传播到所有分片**，无需 Citus 特殊处理：

```sql
-- migrations/20241201000003_add_user_phone.sql

-- 普通 SQL，Citus 自动传播到所有分片
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
```

```sql
-- migrations/20241201000004_add_order_index.sql

-- 普通索引，Citus 在所有分片上创建
CREATE INDEX idx_orders_created ON orders(created_at);
```

**注意**：`CREATE INDEX CONCURRENTLY` 不能在事务中执行，而 sqlx 默认在事务中运行迁移。如果需要不锁表创建索引，应该在迁移外手动执行。

### 引用表迁移

小表使用 `create_reference_table`：

```sql
-- migrations/20241201000005_create_categories.sql

CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

DO $$
BEGIN
    IF EXISTS (SELECT 1 FROM pg_extension WHERE extname = 'citus') THEN
        IF NOT EXISTS (
            SELECT 1 FROM pg_catalog.pg_dist_partition
            WHERE logicalrelid = 'categories'::regclass
        ) THEN
            PERFORM create_reference_table('categories');
        END IF;
    END IF;
END $$;
```

### 运行迁移

```rust
use sqlx::postgres::PgPoolOptions;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let pool = PgPoolOptions::new()
        .connect("postgres://user:pass@coordinator:5432/mydb")
        .await?;

    // 迁移在 Coordinator 上执行，DDL 自动传播到 Workers
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await?;

    Ok(())
}
```

### 什么时候需要 Citus 特殊处理？

| 操作 | 需要 DO $$ 检查？ |
|------|------------------|
| `CREATE TABLE` | ❌ 不需要 |
| `create_distributed_table()` | ✅ 需要 |
| `create_reference_table()` | ✅ 需要 |
| `ALTER TABLE ADD/DROP COLUMN` | ❌ 不需要（自动传播）|
| `CREATE/DROP INDEX` | ❌ 不需要（自动传播）|
| `ALTER TABLE ADD CONSTRAINT` | ❌ 不需要（自动传播）|

**最佳实践**：建表和分布式化写在同一个迁移文件，后续修改用普通 SQL。

## 从单机迁移到 Citus

已有单机 PostgreSQL，如何迁移？

### 步骤 1：评估表结构

```sql
-- 找出大表
SELECT relname, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC
LIMIT 10;
```

决定每个表的类型：
- 大表（> 100 万行）→ 分布式表
- 小表 + 频繁 JOIN → 引用表
- 其他 → 本地表

### 步骤 2：确定分片键

分析查询模式（需先启用 `pg_stat_statements` 扩展）：

```sql
-- 启用扩展（如果还没有）
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 查看最慢的查询
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

找出高频过滤条件，选为分片键。

### 步骤 3：调整主键

分片键必须是主键的一部分：

```sql
-- 原表（假设主键只有 id）
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    amount DECIMAL(10,2)
);

-- 改为复合主键（分片键 + 原主键）
ALTER TABLE orders DROP CONSTRAINT orders_pkey;
ALTER TABLE orders ADD PRIMARY KEY (user_id, id);
```

### 步骤 4：迁移数据

```sql
-- 在 Citus 集群中创建表结构（注意复合主键）
CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    book_id BIGINT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, id)
);

-- 将表转为分布式
SELECT create_distributed_table('orders', 'user_id');

-- 从旧库导出数据（仅数据，不含 schema）
-- 在旧库执行：
\copy orders TO '/path/to/orders.csv' WITH CSV HEADER;

-- 导入到 Citus（Citus 自动路由到正确分片）
\copy orders FROM '/path/to/orders.csv' WITH CSV HEADER;
```

**注意**：不要直接用 `pg_dump | psql`，因为它会尝试重建表结构，而我们需要先手动创建分布式表。

### 步骤 5：验证

```sql
-- 检查分片分布
SELECT * FROM citus_shards WHERE logicalrelid = 'orders'::regclass;

-- 测试查询
EXPLAIN SELECT * FROM orders WHERE user_id = 123;
```

## 监控与运维

### 关键指标

```sql
-- 查看分片大小分布（检查数据倾斜）
SELECT shardid, shard_size
FROM citus_shards
WHERE logicalrelid = 'orders'::regclass
ORDER BY shard_size DESC;

-- 查看节点负载
SELECT nodename, count(*) as shard_count
FROM citus_shards
GROUP BY nodename;

-- 查看活跃连接
SELECT * FROM citus_stat_activity;
```

### 常见问题

**数据倾斜**：某些分片数据量远大于其他分片

```sql
-- 检查分片大小
SELECT shardid, shard_size FROM citus_shards ORDER BY shard_size DESC;

-- 如果倾斜严重，考虑换分片键或增加分片数
SELECT alter_distributed_table('orders', shard_count := 64);
```

**跨分片查询太多**：说明分片键选错了

```sql
-- 监控跨分片查询比例
-- 使用 pg_stat_statements 分析
```

## 限制与注意事项

Citus 不是银弹，了解其限制有助于避免踩坑：

### SQL 功能限制

| 功能 | 支持情况 | 说明 |
|------|---------|------|
| 基本 CRUD | ✅ 完全支持 | |
| JOIN（共置表）| ✅ 完全支持 | 需要配置 co-location |
| JOIN（非共置表）| ⚠️ 部分支持 | 可能触发跨节点数据传输 |
| 子查询 | ⚠️ 部分支持 | 相关子查询有限制 |
| CTE (WITH) | ⚠️ 部分支持 | 不支持递归 CTE |
| 窗口函数 | ⚠️ 部分支持 | PARTITION BY 需包含分片键 |
| 外键约束 | ⚠️ 有限制 | 仅支持同一 colocation group 内的表 |
| 唯一约束 | ⚠️ 有限制 | **必须包含分片键** |
| TRUNCATE | ✅ 支持 | |
| COPY | ✅ 支持 | |

### 必须遵守的规则

**1. 主键和唯一约束必须包含分片键**

```sql
-- ❌ 错误：唯一约束不含分片键
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,  -- 错误！
    user_id BIGINT NOT NULL,
    email VARCHAR(255) UNIQUE  -- 错误！
);

-- ✅ 正确：分片键在所有唯一约束中
CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    email VARCHAR(255),
    PRIMARY KEY (user_id, id),
    UNIQUE (user_id, email)
);
```

**2. 跨分片事务有性能开销**

跨分片事务需要额外的网络往返（2PC 协调），延迟通常是单分片事务的数倍。设计时应尽量让事务在单分片内完成。

**3. 分片键一旦选定难以更改**

更改分片键意味着重建整个表。选择前务必分析查询模式。

### 不适合 Citus 的场景

| 场景 | 原因 | 替代方案 |
|------|------|---------|
| 复杂分析查询（OLAP） | 跨分片聚合开销大 | 使用 ClickHouse、Doris |
| 图数据库场景 | 关系遍历跨分片 | 使用 Neo4j |
| 全文搜索 | 跨分片搜索效率低 | 使用 Elasticsearch |
| 无明确分片键的场景 | 所有查询都跨分片 | 考虑垂直拆分 |

## Citus vs 手动分片 vs NewSQL

| 特性 | 手动分片 | Citus | TiDB/CockroachDB |
|------|---------|-------|------------------|
| 应用改造 | 大量 | 极少 | 极少 |
| 学习成本 | 高 | 低（就是 PG）| 中（新产品）|
| 跨分片查询 | 手动聚合 | 自动 | 自动 |
| 跨分片事务 | Saga/2PC | 自动 2PC | 自动分布式事务 |
| PostgreSQL 生态 | ✅ | ✅ | ❌（兼容但不完全）|
| 扩容复杂度 | 高 | 低 | 低 |
| 运维成本 | 高 | 中 | 中 |
| 适用场景 | 精细控制、已有系统 | PG 用户、多租户 | 新项目、强一致需求 |

## 常见问题

### Q：Citus 免费吗？

**A：开源版完全免费**。企业版有额外功能（如多租户隔离、备份恢复），但开源版足够大多数场景。

### Q：性能损失大吗？

**A：单分片查询开销很小**（通常亚毫秒级）。跨分片查询有网络开销，但 Citus 会并行执行，对于大表通常比单机全表扫描快。实际性能取决于网络延迟、分片数量和查询复杂度。

### Q：能和现有 PostgreSQL 工具一起用吗？

**A：完全兼容**。pgAdmin、DBeaver、pg_dump、pg_restore、所有 ORM 都能用。

### Q：主键必须包含分片键？

**A：是的**。这是 Citus 保证数据本地性的方式。如果原表主键是 `id`，需要改成 `(user_id, id)`。

### Q：云服务支持吗？

**A：Azure 原生支持**（Azure Database for PostgreSQL - Hyperscale）。AWS/GCP 可以在 EC2/GCE 上自建，或使用 Citus Cloud（已整合到 Azure）。

## 总结

Citus 不是银弹，但它是 PostgreSQL 用户水平扩展的**最简单路径**：

| 你的情况 | 建议 |
|---------|------|
| 新项目，预期数据量大 | 直接用 Citus 起步 |
| 已有 PG，数据量暴增 | 迁移到 Citus |
| 需要完全控制分片逻辑 | 手动分片（参考一致性实战系列）|
| 不是 PostgreSQL 用户 | 考虑 TiDB（MySQL 兼容）|

**最后一点**：即使用了 Citus，理解分片原理仍然重要。它帮你选对分片键、设计好表结构、优化跨分片查询。

想深入了解分片原理？参考 [《一致性实战（四）数据分片》](/posts/consistency-in-practice-4-sharding/)。

想了解更多分布式数据库选型？参考 [《数据库演进史（四）分布式数据库》](/posts/db-evolution-4-distributed/)。

---

*相关文章：*
- *[一致性实战（四）数据分片：跨分片事务一致性](/posts/consistency-in-practice-4-sharding/)*
- *[数据库演进史（四）分布式数据库](/posts/db-evolution-4-distributed/)*
- *[数据库演进史（八）PostgreSQL](/posts/db-evolution-8-postgresql/)*
