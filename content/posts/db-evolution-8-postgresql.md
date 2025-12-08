---
title: '数据库演进史（八）：PostgreSQL——一个数据库的文艺复兴'
date: 2025-12-09T11:10:00+08:00
draft: false
tags: ["数据库", "PostgreSQL", "架构"]
categories: ["技术"]
description: "当其他数据库在解决特定问题时，PostgreSQL 选择了一条不同的路：让自己成为一个可以被扩展的平台。"
series: ["数据库演进史"]
---

回顾这个系列：

- 关系型数据库解决了数据一致性
- NoSQL 解决了扩展性和灵活性
- 嵌入式数据库解决了零部署需求
- 分布式数据库解决了单机瓶颈
- 分析型数据库解决了海量数据分析
- 时序数据库解决了时间序列场景
- 向量数据库解决了语义搜索

每个问题都催生出新的数据库类型。而 PostgreSQL 选择了一条不同的路：**成为一个可以被无限扩展的平台**。

<!--more-->

## 从 Ingres 到 Postgres

PostgreSQL 的历史要追溯到 1970 年代：

```text
时间线：

1973  Ingres 项目启动（UC Berkeley）
      Michael Stonebraker 领导
      "Interactive Graphics and Retrieval System"

1985  Postgres 项目启动
      Post-Ingres，"后 Ingres 时代"
      目标：支持复杂数据类型和关系

1995  添加 SQL 支持，改名 Postgres95

1996  正式更名 PostgreSQL
      开源社区接管开发

2024  PostgreSQL 17
      经过 30+ 年进化
```

**关键设计哲学**：

```text
1. 可扩展性优先

   PostgreSQL 不只是一个数据库
   它是一个「数据库构建平台」

2. 标准兼容

   严格遵循 SQL 标准
   同时允许扩展

3. 社区驱动

   没有单一商业公司控制
   社区投票决定方向
```

## 扩展机制：PostgreSQL 的超能力

PostgreSQL 最独特的地方是它的**扩展机制**：

```text
扩展可以做什么：

┌─────────────────────────────────────────────────┐
│                  PostgreSQL                      │
├─────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────┐  │
│  │   数据类型   │  │    索引     │  │   函数   │ │
│  │  自定义类型  │  │  自定义索引  │  │ 自定义函数│ │
│  └─────────────┘  └─────────────┘  └─────────┘  │
│                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────┐  │
│  │   操作符    │  │   外部数据   │  │  语言    │ │
│  │  自定义操作符│  │ Foreign Data │  │PL/Python │ │
│  └─────────────┘  └─────────────┘  └─────────┘  │
└─────────────────────────────────────────────────┘

一行命令安装扩展：
CREATE EXTENSION postgis;     -- 地理信息
CREATE EXTENSION timescaledb; -- 时序数据
CREATE EXTENSION vector;      -- 向量搜索
CREATE EXTENSION citus;       -- 分布式
```

这就是为什么 PostgreSQL 能"变成"各种数据库：

```text
PostgreSQL + PostGIS      = 地理信息数据库
PostgreSQL + TimescaleDB  = 时序数据库
PostgreSQL + pgvector     = 向量数据库
PostgreSQL + Citus        = 分布式数据库
PostgreSQL + JSONB        = 文档数据库
```

## 一个数据库，多种能力

让我们看看 PostgreSQL 如何在一个数据库里满足各种需求。

### 关系型：核心能力

```sql
-- 标准 SQL，严格的数据完整性
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    total_amount DECIMAL(10,2) NOT NULL CHECK (total_amount > 0),
    status TEXT NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT valid_status CHECK (status IN ('pending', 'paid', 'shipped', 'completed'))
);

-- 复杂事务
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 'alice';
    UPDATE accounts SET balance = balance + 100 WHERE id = 'bob';
    INSERT INTO transfers (from_id, to_id, amount) VALUES ('alice', 'bob', 100);
COMMIT;
```

### 文档数据库：JSONB

```sql
-- 灵活的 JSON 存储
CREATE TABLE products (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL,
    attributes JSONB  -- 不同商品有不同属性
);

-- 存储任意结构的数据
INSERT INTO products (id, name, attributes) VALUES
    (gen_random_uuid(), 'iPhone 15', '{
        "brand": "Apple",
        "storage": "256GB",
        "color": "Black",
        "5g": true
    }'),
    (gen_random_uuid(), 'MacBook Pro', '{
        "brand": "Apple",
        "cpu": "M3 Pro",
        "ram": "18GB",
        "ports": ["thunderbolt", "hdmi", "sdcard"]
    }');

-- JSON 路径查询
SELECT name, attributes->>'brand' AS brand
FROM products
WHERE attributes @> '{"5g": true}';

-- GIN 索引加速 JSON 查询
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);
```

### 时序数据：TimescaleDB

```sql
-- 启用 TimescaleDB
CREATE EXTENSION timescaledb;

-- 创建时序表
CREATE TABLE sensor_data (
    time TIMESTAMPTZ NOT NULL,
    sensor_id TEXT NOT NULL,
    temperature DOUBLE PRECISION,
    humidity DOUBLE PRECISION
);

SELECT create_hypertable('sensor_data', 'time');

-- 时间聚合查询
SELECT
    time_bucket('1 hour', time) AS hour,
    sensor_id,
    AVG(temperature) AS avg_temp
FROM sensor_data
WHERE time > NOW() - INTERVAL '7 days'
GROUP BY hour, sensor_id
ORDER BY hour DESC;
```

### 向量搜索：pgvector

```sql
-- 启用向量扩展
CREATE EXTENSION vector;

-- 存储文档和嵌入向量
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)
);

-- 语义搜索
SELECT content,
       1 - (embedding <=> $1::vector) AS similarity
FROM documents
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

### 全文搜索：内置能力

```sql
-- 全文搜索（英文示例）
-- 中文需要安装 zhparser 或 pg_jieba 扩展
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    search_vector tsvector GENERATED ALWAYS AS (
        to_tsvector('english', title || ' ' || content)
    ) STORED
);

CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);

-- 搜索
SELECT title, ts_rank(search_vector, query) AS rank
FROM articles, plainto_tsquery('english', 'PostgreSQL performance') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### 地理信息：PostGIS

```sql
-- 启用地理扩展
CREATE EXTENSION postgis;

-- 存储位置信息
CREATE TABLE stores (
    id SERIAL PRIMARY KEY,
    name TEXT,
    location GEOGRAPHY(POINT)
);

-- 找出 5 公里内的门店
SELECT name, ST_Distance(location, ST_MakePoint(121.4, 31.2)::geography) AS distance
FROM stores
WHERE ST_DWithin(location, ST_MakePoint(121.4, 31.2)::geography, 5000)
ORDER BY distance;
```

## 混合查询：真正的威力

PostgreSQL 的真正威力在于**混合查询**——在一个查询里组合多种能力：

```sql
-- 场景：电商搜索
-- 需求：找到"附近 10 公里内、评分 > 4.0、与用户搜索语义相关"的商家

SELECT
    s.name,
    s.rating,
    ST_Distance(s.location, $user_location::geography) AS distance,
    1 - (s.description_embedding <=> $query_embedding::vector) AS relevance
FROM stores s
WHERE
    -- 地理位置过滤（PostGIS）
    ST_DWithin(s.location, $user_location::geography, 10000)
    -- 评分过滤（标准 SQL）
    AND s.rating >= 4.0
    -- 分类过滤（JSONB）
    AND s.attributes @> '{"category": "restaurant"}'
    -- 语义相关性（pgvector）
    AND s.description_embedding <=> $query_embedding::vector < 0.5
ORDER BY
    relevance DESC,
    distance ASC
LIMIT 20;
```

**这个查询同时使用了**：
- 标准 SQL（过滤、排序）
- PostGIS（地理位置）
- JSONB（灵活属性）
- pgvector（语义搜索）

换成其他方案，你需要：
1. 在 Elasticsearch 搜索文本
2. 在 MongoDB 查询 JSON 属性
3. 在 Redis 查询地理位置
4. 在 Pinecone 查询向量
5. 在应用层合并结果

**复杂度差异巨大。**

## Rust + SQLx：类型安全的数据库访问

```rust
use sqlx::{PgPool, postgres::PgPoolOptions, FromRow};
use pgvector::Vector;
use anyhow::Result;

/// 商家实体，展示 PostgreSQL 的多种数据类型
#[derive(Debug, FromRow)]
struct Store {
    id: i32,
    name: String,
    rating: f64,
    // JSONB 字段
    attributes: serde_json::Value,
    // 地理位置（简化处理）
    latitude: f64,
    longitude: f64,
    // 语义搜索相关性
    relevance: Option<f64>,
}

struct StoreRepository {
    pool: PgPool,
}

impl StoreRepository {
    async fn new(database_url: &str) -> Result<Self> {
        let pool = PgPoolOptions::new()
            .max_connections(5)
            .connect(database_url)
            .await?;

        Ok(Self { pool })
    }

    /// 混合搜索：位置 + 评分 + 分类 + 语义
    async fn hybrid_search(
        &self,
        user_lat: f64,
        user_lng: f64,
        radius_meters: f64,
        min_rating: f64,
        category: &str,
        query_embedding: &[f32],
        limit: i32,
    ) -> Result<Vec<Store>> {
        let embedding = Vector::from(query_embedding.to_vec());

        let stores = sqlx::query_as!(
            Store,
            r#"
            SELECT
                id,
                name,
                rating,
                attributes,
                ST_Y(location::geometry) AS "latitude!",
                ST_X(location::geometry) AS "longitude!",
                1 - (description_embedding <=> $6::vector) AS relevance
            FROM stores
            WHERE
                ST_DWithin(
                    location,
                    ST_SetSRID(ST_MakePoint($2, $1), 4326)::geography,
                    $3
                )
                AND rating >= $4
                AND attributes @> jsonb_build_object('category', $5::text)
            ORDER BY
                relevance DESC,
                ST_Distance(
                    location,
                    ST_SetSRID(ST_MakePoint($2, $1), 4326)::geography
                ) ASC
            LIMIT $7
            "#,
            user_lat,
            user_lng,
            radius_meters,
            min_rating,
            category,
            embedding as Vector,
            limit
        )
        .fetch_all(&self.pool)
        .await?;

        Ok(stores)
    }

    /// 事务示例：创建订单
    async fn create_order(&self, user_id: i32, items: Vec<OrderItem>) -> Result<i32> {
        let mut tx = self.pool.begin().await?;

        // 计算总金额
        let total: f64 = items.iter().map(|i| i.price * i.quantity as f64).sum();

        // 创建订单
        let order_id = sqlx::query_scalar!(
            r#"
            INSERT INTO orders (user_id, total_amount, status)
            VALUES ($1, $2, 'pending')
            RETURNING id
            "#,
            user_id,
            total
        )
        .fetch_one(&mut *tx)
        .await?;

        // 创建订单项
        for item in items {
            sqlx::query!(
                r#"
                INSERT INTO order_items (order_id, product_id, quantity, price)
                VALUES ($1, $2, $3, $4)
                "#,
                order_id,
                item.product_id,
                item.quantity,
                item.price
            )
            .execute(&mut *tx)
            .await?;

            // 更新库存
            let affected = sqlx::query!(
                r#"
                UPDATE products
                SET stock = stock - $1
                WHERE id = $2 AND stock >= $1
                "#,
                item.quantity,
                item.product_id
            )
            .execute(&mut *tx)
            .await?
            .rows_affected();

            if affected == 0 {
                tx.rollback().await?;
                anyhow::bail!("库存不足: product_id={}", item.product_id);
            }
        }

        tx.commit().await?;
        Ok(order_id)
    }
}

struct OrderItem {
    product_id: i32,
    quantity: i32,
    price: f64,
}
```

## PostgreSQL 的局限性

PostgreSQL 不是银弹，它也有局限：

```text
1. 单机写入瓶颈

   PostgreSQL 主要是单机数据库
   写入性能有上限（约 10-50K TPS，取决于硬件）
   解决方案：Citus、读写分离

2. 超大规模数据

   单表超过 10 亿行，性能下降明显
   需要分区或分布式方案

3. 学习曲线

   功能太多，完全掌握需要时间
   扩展之间可能有兼容性问题

4. 运维复杂度

   扩展越多，运维越复杂
   备份恢复需要考虑所有扩展
```

## PostgreSQL vs 专用数据库

让我们诚实地比较 PostgreSQL 与各个专用数据库：

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    PostgreSQL vs 专用数据库对比                          │
├───────────────┬───────────────────┬─────────────────────────────────────┤
│     场景       │  专用方案更好时    │      PostgreSQL 够用时              │
├───────────────┼───────────────────┼─────────────────────────────────────┤
│  OLAP 分析    │ > 10 亿行聚合查询  │ < 1 亿行，或可接受分钟级响应        │
│  (ClickHouse) │ 需要亚秒级响应     │ 使用物化视图预聚合                  │
│               │ 专职分析团队       │ 分析只是附属需求                    │
├───────────────┼───────────────────┼─────────────────────────────────────┤
│  全文搜索     │ 复杂搜索需求       │ 简单关键词搜索                      │
│ (Elasticsearch)│ 需要搜索建议/纠错 │ 不需要高级搜索功能                  │
│               │ 日志分析（ELK）    │ 可接受 GIN 索引性能                 │
├───────────────┼───────────────────┼─────────────────────────────────────┤
│  嵌入式       │ 移动端/桌面端      │ N/A（场景不同）                     │
│  (SQLite)     │ 零服务器需求       │ PostgreSQL 需要服务器               │
├───────────────┼───────────────────┼─────────────────────────────────────┤
│  本地分析     │ 处理本地大文件     │ N/A（场景不同）                     │
│  (DuckDB)     │ 数据科学工作流     │ 数据在 PostgreSQL 中就用 PG        │
├───────────────┼───────────────────┼─────────────────────────────────────┤
│  向量搜索     │ > 1000 万向量      │ < 1000 万向量                       │
│  (Milvus)     │ 需要分布式         │ pgvector 够用                       │
├───────────────┼───────────────────┼─────────────────────────────────────┤
│  时序数据     │ 专业监控平台       │ 业务时序 + 其他数据混合             │
│  (InfluxDB)   │ Prometheus 生态    │ TimescaleDB 扩展                    │
└───────────────┴───────────────────┴─────────────────────────────────────┘
```

**务实的选择策略**：

```text
第一步：问自己"PostgreSQL 能解决吗？"
  │
  ├─ 能，且性能可接受 → 用 PostgreSQL
  │
  └─ 不能，或性能差距太大 →
      │
      第二步：这个场景有多重要？
      │
      ├─ 核心场景 → 引入专用数据库
      │
      └─ 边缘场景 → 考虑妥协或延迟优化

例子：
- "需要日志分析" → 如果日志不多，PostgreSQL + 分区表
                 → 如果是专职日志平台，上 Elasticsearch
- "需要实时报表" → 如果查询不复杂，PostgreSQL + 物化视图
                 → 如果是 BI 平台，上 ClickHouse
```

## 选择 PostgreSQL 的理由

```text
什么时候选 PostgreSQL：

✓ 需要 ACID 事务保证
✓ 数据模型复杂（关系 + JSON + 向量 + 时序 + ...）
✓ 团队熟悉 SQL
✓ 希望减少技术栈复杂度
✓ 数据规模在单机或小集群可以处理的范围内
✓ 需要多种数据类型的混合查询

什么时候考虑其他方案：

✗ 超大规模 OLAP 分析（考虑 ClickHouse）
✗ 专业日志平台/复杂搜索（考虑 Elasticsearch）
✗ 移动端/桌面端/嵌入式场景（考虑 SQLite）
✗ 本地大文件分析（考虑 DuckDB）
✗ 超大规模向量搜索（考虑 Milvus）
✗ 极简单的缓存场景（用 Redis）
```

## 核心认知

> **PostgreSQL 的本质**：一个可以被无限扩展的数据库平台，通过扩展机制"变成"你需要的任何数据库。

**系列总结**：

```text
数据库演进的启示：

1. 没有万能的数据库
   每种数据库解决特定问题
   但 PostgreSQL 通过扩展覆盖了大多数场景

2. 复杂度是敌人
   多个数据库 = 多套运维、多种查询语言、数据同步问题
   能用一个数据库解决，就不要用多个

3. 选择的智慧
   80% 的场景：PostgreSQL 足够
   20% 的极端场景：选择专用数据库

4. 演进而非革命
   从简单开始（单表 PostgreSQL）
   根据需要逐步添加扩展
   只在必要时引入新数据库
```

---

*上一篇：[向量数据库——AI 时代的新基建](/posts/db-evolution-7-vector/)*

*回顾本系列：*
1. *[关系型数据库——SQL 一统江湖的时代](/posts/db-evolution-1-relational/)*：数据一致性的基石
2. *[NoSQL 运动——打破关系的枷锁](/posts/db-evolution-2-nosql/)*：扩展性和灵活性的探索
3. *[嵌入式数据库——藏在你身边的数据库](/posts/db-evolution-3-embedded/)*：零部署的本地数据库
4. *[分布式数据库——当单机不够用](/posts/db-evolution-4-distributed/)*：突破单机瓶颈
5. *[分析型数据库——当查询变成分析](/posts/db-evolution-5-olap-search/)*：海量数据的聚合分析
6. *[时序数据库——时间维度的专家](/posts/db-evolution-6-timeseries/)*：针对时间序列的优化
7. *[向量数据库——AI 时代的新基建](/posts/db-evolution-7-vector/)*：语义搜索的新范式
8. *PostgreSQL——一个数据库的文艺复兴（本篇）*：一个平台，无限可能
