---
title: '数据库演进史（六）：时序数据库——时间维度的专家'
date: 2025-12-09T10:50:00+08:00
draft: false
tags: ["数据库", "时序数据库", "TimescaleDB", "PostgreSQL", "监控"]
categories: ["技术"]
description: "你的服务器每秒产生上千条指标数据，传统数据库已经喘不过气。是时候让时间成为第一公民了。"
series: ["数据库演进史"]
---

你的服务器每秒产生上千条监控指标。用户行为日志像瀑布一样涌入。IoT 设备 7×24 小时不停地汇报传感器数据。

传统数据库处理这些数据时，已经开始喘不过气了。

<!--more-->

## 时序数据的特殊性

时序数据（Time Series Data）有三个显著特点：

```text
特点一：写多读少

  传统业务数据              时序数据
  ┌─────────────┐          ┌─────────────┐
  │ 读：80%     │          │ 读：20%     │
  │ 写：20%     │          │ 写：80%     │
  └─────────────┘          └─────────────┘

  用户反复查询订单          传感器不停写入数据
  偶尔下单                  偶尔查看图表

特点二：数据有时效性

  ┌───────────────────────────────────────┐
  │ 热数据（最近 24 小时）                 │  ← 频繁查询
  │ 温数据（最近 7 天）                    │  ← 偶尔查询
  │ 冷数据（更早）                         │  ← 基本不查，可压缩/删除
  └───────────────────────────────────────┘

  没人关心三年前某一秒的 CPU 使用率

特点三：时间范围查询为主

  常见查询模式：
  - "最近 1 小时的请求量"
  - "过去 7 天的平均响应时间"
  - "本月每天的销售额趋势"

  而不是：
  - "查找 ID = 12345 的那条记录"
```

这些特点让传统数据库力不从心：

```text
传统数据库的困境：

1. B+ 树索引不适合时间序列
   - 数据按时间顺序写入，B+ 树不断分裂
   - 写入性能随数据量增长而下降

2. 没有原生的时间聚合
   - "每 5 分钟的平均值" 需要复杂 SQL
   - 性能差，写起来也痛苦

3. 存储效率低
   - 相邻时间点的数据高度相似
   - 通用存储无法利用这个特性压缩
```

## 专用时序数据库的崛起

### InfluxDB：时序数据库的代表

InfluxDB 是最流行的开源时序数据库之一：

```text
InfluxDB 的数据模型：

measurement: cpu_usage          ← 类似"表名"
tags: host=server01, region=us  ← 索引字段（低基数）
fields: value=0.64, temp=72     ← 数据字段（高基数）
timestamp: 1699574400000000000  ← 纳秒精度时间戳

写入示例（Line Protocol）：
cpu_usage,host=server01,region=us value=0.64,temp=72 1699574400000000000
```

查询使用 Flux 语言（InfluxDB 2.x）：

```flux
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu_usage")
  |> filter(fn: (r) => r.host == "server01")
  |> aggregateWindow(every: 5m, fn: mean)
```

**InfluxDB 的问题**：

```text
1. 自创查询语言
   - SQL？不支持（或有限支持）
   - 学习成本高，生态工具少

2. 集群版闭源
   - 开源版本是单机
   - 高可用需要付费

3. 高基数问题
   - tag 如果有太多不同值（如 user_id）
   - 性能急剧下降
```

### Prometheus：监控领域的标准

Prometheus 是云原生监控的事实标准：

```text
Prometheus 架构：

┌─────────────┐    Pull    ┌─────────────┐
│ Application │ ◄───────── │ Prometheus  │
│ /metrics    │            │   Server    │
└─────────────┘            └──────┬──────┘
                                  │
                           ┌──────▼──────┐
                           │    TSDB     │
                           │ (本地存储)   │
                           └─────────────┘
```

PromQL 查询：

```promql
# 最近 5 分钟的平均 CPU 使用率
avg(rate(cpu_usage_seconds_total[5m])) by (instance)

# 请求错误率
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))
```

**Prometheus 的局限**：

```text
1. 专为监控设计
   - 不适合业务时序数据
   - 数据精度有限（默认保留 15 天）

2. 单机存储
   - 长期存储需要外部方案
   - Thanos、VictoriaMetrics 等

3. Pull 模型的限制
   - 短生命周期任务难以监控
   - 需要 Pushgateway 中转
```

### 时序数据库的共同困境

```text
┌─────────────────────────────────────────────────────────┐
│                    时序数据库的困境                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 又是一个新数据库                                      │
│     - 新的查询语言（Flux、PromQL）                        │
│     - 新的运维知识                                       │
│     - 新的备份恢复流程                                    │
│                                                         │
│  2. 与关系数据库割裂                                      │
│     - 指标数据在时序库                                    │
│     - 业务数据在 PostgreSQL                              │
│     - 关联查询？做梦吧                                    │
│                                                         │
│  3. SQL 技能无法复用                                     │
│     - 团队积累的 SQL 经验                                │
│     - 丰富的 SQL 工具生态                                │
│     - 全都用不上                                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## TimescaleDB：PostgreSQL 的时序扩展

TimescaleDB 的思路是：**既然 PostgreSQL 这么强大，为什么不直接扩展它？**

```text
TimescaleDB = PostgreSQL + 时序超能力

┌─────────────────────────────────────────┐
│              TimescaleDB                 │
├─────────────────────────────────────────┤
│  ┌─────────────────────────────────┐    │
│  │         Hypertable              │    │  ← 自动分区
│  │  ┌─────┬─────┬─────┬─────┐     │    │
│  │  │ 1月 │ 2月 │ 3月 │ ... │     │    │  ← 按时间分片
│  │  └─────┴─────┴─────┴─────┘     │    │
│  └─────────────────────────────────┘    │
│                                         │
│  时间聚合函数 │ 压缩 │ 保留策略 │ ...    │
├─────────────────────────────────────────┤
│              PostgreSQL                  │
│   SQL │ ACID │ 索引 │ JOIN │ 生态       │
└─────────────────────────────────────────┘
```

### Hypertable：透明的时间分区

```sql
-- 创建普通表
CREATE TABLE metrics (
    time        TIMESTAMPTZ NOT NULL,
    device_id   TEXT NOT NULL,
    temperature DOUBLE PRECISION,
    humidity    DOUBLE PRECISION
);

-- 一行命令，变成 hypertable
SELECT create_hypertable('metrics', 'time');

-- 就这样！现在它是时序表了
-- 内部按时间自动分片，但你感知不到
```

插入数据，和普通表完全一样：

```sql
INSERT INTO metrics (time, device_id, temperature, humidity)
VALUES
    (NOW(), 'sensor-001', 23.5, 65.2),
    (NOW(), 'sensor-002', 24.1, 63.8);
```

查询也是标准 SQL：

```sql
-- 最近 24 小时的数据
SELECT * FROM metrics
WHERE time > NOW() - INTERVAL '24 hours';

-- 按设备分组的平均温度
SELECT device_id, AVG(temperature)
FROM metrics
WHERE time > NOW() - INTERVAL '7 days'
GROUP BY device_id;
```

### time_bucket：时间聚合利器

这是 TimescaleDB 最实用的函数：

```sql
-- 每 5 分钟的平均温度
SELECT
    time_bucket('5 minutes', time) AS bucket,
    device_id,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp
FROM metrics
WHERE time > NOW() - INTERVAL '1 hour'
GROUP BY bucket, device_id
ORDER BY bucket DESC;

-- 结果：
--        bucket        | device_id | avg_temp | max_temp | min_temp
-- ---------------------+-----------+----------+----------+----------
-- 2024-01-15 10:30:00 | sensor-001|    23.5  |    24.2  |    22.8
-- 2024-01-15 10:30:00 | sensor-002|    24.1  |    25.0  |    23.5
-- 2024-01-15 10:25:00 | sensor-001|    23.2  |    23.8  |    22.5
```

对比原生 SQL 的写法：

```sql
-- 没有 time_bucket，你得这样写
SELECT
    date_trunc('hour', time) +
    (EXTRACT(minute FROM time)::int / 5) * INTERVAL '5 minutes' AS bucket,
    device_id,
    AVG(temperature)
FROM metrics
GROUP BY 1, 2;

-- 可读性差，还容易出错
```

### 连续聚合：预计算的魔法

实时聚合大量数据很慢。TimescaleDB 的解决方案是预计算：

```sql
-- 创建连续聚合（类似物化视图，但自动更新）
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour,
    device_id,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp,
    COUNT(*) AS sample_count
FROM metrics
GROUP BY hour, device_id;

-- 设置自动刷新策略
SELECT add_continuous_aggregate_policy('metrics_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- 查询时自动使用预聚合数据，快得飞起
SELECT * FROM metrics_hourly
WHERE hour > NOW() - INTERVAL '7 days';
```

### 数据压缩：存储成本直降 90%

时序数据有个特点：相邻时间点的数据往往相似。TimescaleDB 利用这个特性进行列式压缩：

```sql
-- 启用压缩
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id'
);

-- 添加压缩策略：7 天前的数据自动压缩
SELECT add_compression_policy('metrics', INTERVAL '7 days');

-- 查看压缩效果
SELECT
    pg_size_pretty(before_compression_total_bytes) AS before,
    pg_size_pretty(after_compression_total_bytes) AS after,
    round(100 - (after_compression_total_bytes::numeric /
          before_compression_total_bytes::numeric) * 100, 1) AS ratio
FROM hypertable_compression_stats('metrics');

-- 结果：
-- before | after | ratio
-- -------+-------+-------
-- 10 GB  | 950 MB| 90.5%
```

### 数据保留策略

自动删除过期数据：

```sql
-- 只保留最近 30 天的数据
SELECT add_retention_policy('metrics', INTERVAL '30 days');

-- 查看当前策略
SELECT * FROM timescaledb_information.jobs
WHERE proc_name = 'policy_retention';
```

### Rust 实战：监控数据采集

```rust
use sqlx::{PgPool, postgres::PgPoolOptions};
use chrono::{DateTime, Utc};
use anyhow::Result;

#[derive(Debug)]
struct Metric {
    time: DateTime<Utc>,
    device_id: String,
    temperature: f64,
    humidity: f64,
}

struct MetricsRepository {
    pool: PgPool,
}

impl MetricsRepository {
    async fn new(database_url: &str) -> Result<Self> {
        let pool = PgPoolOptions::new()
            .max_connections(5)
            .connect(database_url)
            .await?;

        Ok(Self { pool })
    }

    /// 批量写入指标（时序数据的典型场景）
    async fn insert_batch(&self, metrics: &[Metric]) -> Result<()> {
        // 使用事务 + 批量插入实现高性能写入
        let mut tx = self.pool.begin().await?;

        for metric in metrics {
            sqlx::query!(
                r#"
                INSERT INTO metrics (time, device_id, temperature, humidity)
                VALUES ($1, $2, $3, $4)
                "#,
                metric.time,
                metric.device_id,
                metric.temperature,
                metric.humidity
            )
            .execute(&mut *tx)
            .await?;
        }

        tx.commit().await?;
        Ok(())
    }

    /// 查询最近 N 小时的聚合数据
    async fn get_hourly_stats(
        &self,
        device_id: &str,
        hours: i32,
    ) -> Result<Vec<HourlyStat>> {
        let stats = sqlx::query_as!(
            HourlyStat,
            r#"
            SELECT
                time_bucket('1 hour', time) AS "hour!",
                AVG(temperature) AS "avg_temp!",
                MAX(temperature) AS "max_temp!",
                MIN(temperature) AS "min_temp!",
                COUNT(*)::int AS "sample_count!"
            FROM metrics
            WHERE device_id = $1
              AND time > NOW() - make_interval(hours => $2)
            GROUP BY 1
            ORDER BY 1 DESC
            "#,
            device_id,
            hours
        )
        .fetch_all(&self.pool)
        .await?;

        Ok(stats)
    }

    /// 获取设备的最新读数
    async fn get_latest(&self, device_id: &str) -> Result<Option<Metric>> {
        // TimescaleDB 对这类查询有专门优化
        let metric = sqlx::query_as!(
            Metric,
            r#"
            SELECT time, device_id, temperature, humidity
            FROM metrics
            WHERE device_id = $1
            ORDER BY time DESC
            LIMIT 1
            "#,
            device_id
        )
        .fetch_optional(&self.pool)
        .await?;

        Ok(metric)
    }
}

#[derive(Debug)]
struct HourlyStat {
    hour: DateTime<Utc>,
    avg_temp: f64,
    max_temp: f64,
    min_temp: f64,
    sample_count: i32,
}
```

### 实际部署建议

```text
小规模（< 1TB 数据）：
  - 单节点 TimescaleDB 足够
  - 启用压缩，设置保留策略
  - 定期备份即可

中规模（1-10TB）：
  - 考虑读写分离
  - 使用连续聚合减少查询压力
  - 冷热数据分层存储

大规模（> 10TB）：
  - TimescaleDB 分布式版本
  - 或考虑 Citus + TimescaleDB 结合
  - 配合对象存储做归档
```

## 时序数据库选型指南

```text
选 InfluxDB 如果：
  ✓ 纯指标监控场景
  ✓ 团队熟悉 Flux 语言
  ✓ 可以接受单机版或付费集群版

选 Prometheus 如果：
  ✓ Kubernetes 环境
  ✓ 主要用于基础设施监控
  ✓ 配合 Grafana 使用

选 TimescaleDB 如果：
  ✓ 已有 PostgreSQL 技术栈
  ✓ 需要时序与业务数据关联查询
  ✓ 团队熟悉 SQL
  ✓ 需要 ACID 事务保证
```

## 核心认知

> **时序数据库的本质**：针对时间维度的数据访问模式，做存储和查询的深度优化。

**为什么 TimescaleDB 是个好选择**：

```text
1. 不增加技术栈复杂度
   - 还是 PostgreSQL
   - 还是 SQL
   - 还是原有的运维流程

2. 时序与业务数据打通
   - 可以 JOIN 业务表
   - 统一的事务保证
   - 一套备份恢复方案

3. 渐进式采用
   - 先当普通表用
   - 数据量大了再启用 hypertable
   - 按需开启压缩、聚合
```

下一篇，我们来看一个更新的领域：AI 时代的向量数据库。当你需要找"相似"而不是"相等"的数据时，传统索引就彻底无能为力了。

---

*上一篇：[分析型数据库——当查询变成分析](/posts/db-evolution-5-olap-search/)*

*下一篇：[向量数据库——AI 时代的新基建](/posts/db-evolution-7-vector/)*

*本系列：*
1. *[关系型数据库——SQL 一统江湖的时代](/posts/db-evolution-1-relational/)*
2. *[NoSQL 运动——打破关系的枷锁](/posts/db-evolution-2-nosql/)*
3. *[嵌入式数据库——藏在你身边的数据库](/posts/db-evolution-3-embedded/)*
4. *[分布式数据库——当单机不够用](/posts/db-evolution-4-distributed/)*
5. *[分析型数据库——当查询变成分析](/posts/db-evolution-5-olap-search/)*
6. *时序数据库——时间维度的专家（本篇）*
7. *[向量数据库——AI 时代的新基建](/posts/db-evolution-7-vector/)*
8. *[PostgreSQL——一个数据库的文艺复兴](/posts/db-evolution-8-postgresql/)*
