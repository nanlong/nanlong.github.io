---
title: '数据库演进史（五）：分析型数据库——当查询变成分析'
date: 2025-12-09T10:40:00+08:00
draft: false
tags: ["数据库", "ClickHouse", "Elasticsearch", "OLAP", "搜索引擎"]
categories: ["技术"]
description: "你的 SQL 查询从毫秒级变成了分钟级。不是数据库的问题，是你用错了数据库。"
series: ["数据库演进史"]
---

老板说："给我一个报表，统计过去一年每个渠道的用户增长趋势。"

你写了一个 SQL，跑了 10 分钟还没出结果。PostgreSQL 表示很委屈——这不是它擅长的事。

**有些查询，需要专门的数据库。**

<!--more-->

## OLTP vs OLAP：两种截然不同的负载

数据库世界有两种截然不同的工作模式：

```text
OLTP（联机事务处理）：
┌─────────────────────────────────────────┐
│  特点                                    │
│  - 大量小事务（INSERT/UPDATE/DELETE）    │
│  - 查询返回少量行                        │
│  - 需要强一致性和事务                    │
│  - 延迟敏感（毫秒级）                    │
├─────────────────────────────────────────┤
│  典型场景                                │
│  - 下单、支付、注册                      │
│  - "用户 A 买了商品 B"                   │
├─────────────────────────────────────────┤
│  代表数据库                              │
│  PostgreSQL, MySQL, TiDB                │
└─────────────────────────────────────────┘

OLAP（联机分析处理）：
┌─────────────────────────────────────────┐
│  特点                                    │
│  - 少量大查询（复杂 SELECT）             │
│  - 查询扫描大量行，返回聚合结果          │
│  - 可以接受最终一致性                    │
│  - 吞吐敏感（秒级可接受）                │
├─────────────────────────────────────────┤
│  典型场景                                │
│  - 报表、仪表盘、数据分析                │
│  - "过去一年各渠道 GMV 趋势"             │
├─────────────────────────────────────────┤
│  代表数据库                              │
│  ClickHouse, Doris, Snowflake           │
└─────────────────────────────────────────┘
```

**为什么 OLTP 数据库不适合 OLAP？**

```text
场景：分析 1 亿条订单的月度销售趋势

PostgreSQL（行存储）的执行方式：
┌────────┬────────┬────────┬────────┬────────┐
│ row 1  │ id     │ user   │ amount │ date   │ ← 读取整行
├────────┼────────┼────────┼────────┼────────┤
│ row 2  │ id     │ user   │ amount │ date   │ ← 读取整行
├────────┼────────┼────────┼────────┼────────┤
│ ...    │ ...    │ ...    │ ...    │ ...    │
└────────┴────────┴────────┴────────┴────────┘

你只需要 amount 和 date 两列
但必须读取所有列 → 90% 的 IO 是浪费的

ClickHouse（列存储）的执行方式：
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│   id   │ │  user  │ │ amount │ │  date  │
├────────┤ ├────────┤ ├────────┤ ├────────┤
│   1    │ │ alice  │ │  100   │ │ 2024-1 │
│   2    │ │  bob   │ │  200   │ │ 2024-1 │
│  ...   │ │  ...   │ │  ...   │ │  ...   │
└────────┘ └────────┘ └────────┘ └────────┘
    ↑           ↑          ↑          ↑
  不读取      不读取     只读这列   只读这列

只读取需要的列 → IO 减少 80%+
```

## ClickHouse：列式存储的性能怪兽

ClickHouse 由俄罗斯 Yandex 开发，是目前最快的开源 OLAP 数据库之一。

### 为什么这么快？

```text
ClickHouse 的性能秘诀：

1. 列式存储
   - 只读取需要的列
   - 同一列数据类型相同，压缩率极高（10:1 常见）

2. 向量化执行
   - 批量处理数据（而非逐行）
   - 充分利用 CPU SIMD 指令

3. 稀疏索引
   - 不是每行都建索引
   - 每 8192 行建一个索引条目
   - 索引小，全部放内存

4. 数据分区 + 排序
   - 按时间分区，查询只扫描相关分区
   - 分区内数据有序，范围查询极快
```

### 基本使用

```sql
-- 创建表（注意引擎选择）
CREATE TABLE events (
    event_date Date,
    event_time DateTime,
    user_id UInt64,
    event_type String,
    properties String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)  -- 按月分区
ORDER BY (event_type, user_id, event_time);  -- 排序键

-- 插入数据
INSERT INTO events VALUES
    ('2024-01-15', '2024-01-15 10:30:00', 1001, 'click', '{"page": "home"}'),
    ('2024-01-15', '2024-01-15 10:31:00', 1002, 'purchase', '{"amount": 99.9}');

-- 分析查询：过去 30 天每天的事件量
SELECT
    event_date,
    event_type,
    count() AS cnt
FROM events
WHERE event_date >= today() - 30
GROUP BY event_date, event_type
ORDER BY event_date, cnt DESC;

-- 漏斗分析：用户从浏览到购买的转化
SELECT
    level,
    count() AS users
FROM (
    SELECT
        user_id,
        windowFunnel(86400)(event_time, event_type = 'view', event_type = 'cart', event_type = 'purchase') AS level
    FROM events
    WHERE event_date >= '2024-01-01'
    GROUP BY user_id
)
GROUP BY level
ORDER BY level;
```

### ClickHouse 的适用场景

```text
非常适合：
✓ 日志分析（访问日志、应用日志）
✓ 用户行为分析（埋点数据）
✓ 实时报表和仪表盘
✓ 时序指标聚合
✓ A/B 测试分析

不适合：
✗ 事务处理（没有事务）
✗ 频繁更新/删除（Mutation 很慢）
✗ 点查询（按主键查单行）
✗ 高并发小查询（更适合少量大查询）
```

### ClickHouse vs PostgreSQL

```text
场景：1 亿行数据，统计每天的销售额

PostgreSQL：
  SELECT date, SUM(amount) FROM orders GROUP BY date;
  执行时间：60-120 秒

ClickHouse：
  SELECT date, SUM(amount) FROM orders GROUP BY date;
  执行时间：0.5-2 秒

性能差距：50-100 倍

为什么？
- PostgreSQL 读取所有列 → ClickHouse 只读 2 列
- PostgreSQL 逐行处理 → ClickHouse 向量化批处理
- PostgreSQL 通用优化 → ClickHouse 专为聚合优化
```

## Elasticsearch：搜索引擎的事实标准

如果说 ClickHouse 是分析之王，Elasticsearch 就是搜索之王。

### 倒排索引：搜索的核心

```text
传统数据库（B-Tree 索引）：

"找出内容包含 'PostgreSQL' 的文章"

articles 表：
┌────┬────────────────────────────────────┐
│ id │ content                            │
├────┼────────────────────────────────────┤
│ 1  │ PostgreSQL is a powerful database  │
│ 2  │ MySQL is popular                   │
│ 3  │ PostgreSQL supports JSONB          │
└────┴────────────────────────────────────┘

执行方式：全表扫描，逐行检查是否包含关键词
时间复杂度：O(N × L)，N=行数，L=平均文本长度

Elasticsearch（倒排索引）：

倒排索引：
┌─────────────┬────────────┐
│    词项     │   文档列表  │
├─────────────┼────────────┤
│ postgresql  │ [1, 3]     │
│ powerful    │ [1]        │
│ database    │ [1]        │
│ mysql       │ [2]        │
│ popular     │ [2]        │
│ supports    │ [3]        │
│ jsonb       │ [3]        │
└─────────────┴────────────┘

执行方式：直接查找 "postgresql" → 返回 [1, 3]
时间复杂度：O(1) 哈希查找
```

### 基本使用

```json
// 创建索引（类似创建表）
PUT /articles
{
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "ik_max_word" },
      "content": { "type": "text", "analyzer": "ik_max_word" },
      "author": { "type": "keyword" },
      "publish_date": { "type": "date" },
      "tags": { "type": "keyword" }
    }
  }
}

// 索引文档（插入数据）
POST /articles/_doc
{
  "title": "PostgreSQL 性能优化指南",
  "content": "本文介绍 PostgreSQL 数据库的性能优化技巧...",
  "author": "张三",
  "publish_date": "2024-01-15",
  "tags": ["数据库", "PostgreSQL", "性能优化"]
}

// 全文搜索
GET /articles/_search
{
  "query": {
    "multi_match": {
      "query": "PostgreSQL 性能",
      "fields": ["title^2", "content"]  // title 权重更高
    }
  },
  "highlight": {
    "fields": {
      "title": {},
      "content": {}
    }
  }
}

// 聚合分析：按作者统计文章数
GET /articles/_search
{
  "size": 0,
  "aggs": {
    "by_author": {
      "terms": { "field": "author" }
    }
  }
}
```

### ELK 生态：日志分析标配

```text
ELK Stack（现在叫 Elastic Stack）：

┌─────────────────────────────────────────────────────────┐
│                      Kibana                              │
│              （可视化仪表盘）                              │
└─────────────────────────────────────────────────────────┘
                           ↑
┌─────────────────────────────────────────────────────────┐
│                    Elasticsearch                         │
│              （存储 + 搜索 + 分析）                        │
└─────────────────────────────────────────────────────────┘
                           ↑
┌─────────────────────────────────────────────────────────┐
│                      Logstash                            │
│              （数据采集 + 转换）                           │
└─────────────────────────────────────────────────────────┘
                           ↑
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  服务器   │ │  容器    │ │  应用    │ │   ...    │
│   日志   │ │   日志   │ │   日志   │ │          │
└──────────┘ └──────────┘ └──────────┘ └──────────┘

典型用途：
- 服务器日志集中管理
- 应用错误追踪
- 安全事件分析
- APM（应用性能监控）
```

### Elasticsearch 的适用场景

```text
非常适合：
✓ 全文搜索（网站搜索、商品搜索）
✓ 日志分析（ELK Stack）
✓ 实时监控和告警
✓ 地理位置搜索

不适合：
✗ 事务处理（没有 ACID）
✗ 复杂关联查询（没有 JOIN）
✗ 精确数值计算（浮点精度问题）
✗ 作为主数据库（数据安全性不如传统数据库）
```

### Elasticsearch vs PostgreSQL 全文搜索

```text
PostgreSQL 内置全文搜索：
  优点：
  - 无需额外组件
  - 支持事务
  - 可以和关系数据 JOIN

  缺点：
  - 中文需要额外扩展（zhparser）
  - 功能相对简单
  - 大数据量性能不如 ES

Elasticsearch：
  优点：
  - 功能丰富（高亮、建议、拼音等）
  - 分布式，可水平扩展
  - 生态完善（Kibana、Logstash）
  - 中文支持好（IK 分词器）

  缺点：
  - 额外维护一套系统
  - 需要数据同步
  - 资源消耗大

选择建议：
- 简单搜索 + 数据量小 → PostgreSQL
- 复杂搜索 + 数据量大 → Elasticsearch
- 日志分析 → Elasticsearch（事实标准）
```

## Rust 实战：日志分析系统

```rust
use reqwest::Client;
use serde::{Deserialize, Serialize};
use serde_json::json;
use anyhow::Result;

#[derive(Debug, Serialize, Deserialize)]
struct LogEntry {
    timestamp: String,
    level: String,
    service: String,
    message: String,
    trace_id: Option<String>,
}

struct LogAnalyzer {
    client: Client,
    es_url: String,
}

impl LogAnalyzer {
    fn new(es_url: &str) -> Self {
        Self {
            client: Client::new(),
            es_url: es_url.to_string(),
        }
    }

    /// 批量写入日志到 Elasticsearch
    async fn bulk_index(&self, logs: &[LogEntry]) -> Result<()> {
        let mut body = String::new();

        for log in logs {
            // Bulk API 格式：每条数据两行（action + document）
            body.push_str(&json!({"index": {"_index": "logs"}}).to_string());
            body.push('\n');
            body.push_str(&serde_json::to_string(log)?);
            body.push('\n');
        }

        self.client
            .post(format!("{}/_bulk", self.es_url))
            .header("Content-Type", "application/x-ndjson")
            .body(body)
            .send()
            .await?;

        Ok(())
    }

    /// 搜索错误日志
    async fn search_errors(
        &self,
        service: &str,
        hours: i32,
    ) -> Result<Vec<LogEntry>> {
        let query = json!({
            "query": {
                "bool": {
                    "must": [
                        { "term": { "service": service } },
                        { "term": { "level": "ERROR" } },
                        {
                            "range": {
                                "timestamp": {
                                    "gte": format!("now-{}h", hours)
                                }
                            }
                        }
                    ]
                }
            },
            "sort": [{ "timestamp": "desc" }],
            "size": 100
        });

        let response: SearchResponse = self.client
            .post(format!("{}/logs/_search", self.es_url))
            .json(&query)
            .send()
            .await?
            .json()
            .await?;

        Ok(response.hits.hits.into_iter().map(|h| h._source).collect())
    }

    /// 统计每个服务的错误数量
    async fn error_stats(&self) -> Result<Vec<ServiceErrorCount>> {
        let query = json!({
            "size": 0,
            "query": {
                "bool": {
                    "must": [
                        { "term": { "level": "ERROR" } },
                        { "range": { "timestamp": { "gte": "now-24h" } } }
                    ]
                }
            },
            "aggs": {
                "by_service": {
                    "terms": { "field": "service" }
                }
            }
        });

        let response: AggResponse = self.client
            .post(format!("{}/logs/_search", self.es_url))
            .json(&query)
            .send()
            .await?
            .json()
            .await?;

        Ok(response.aggregations.by_service.buckets
            .into_iter()
            .map(|b| ServiceErrorCount {
                service: b.key,
                count: b.doc_count,
            })
            .collect())
    }
}

#[derive(Debug, Deserialize)]
struct SearchResponse {
    hits: Hits,
}

#[derive(Debug, Deserialize)]
struct Hits {
    hits: Vec<Hit>,
}

#[derive(Debug, Deserialize)]
struct Hit {
    _source: LogEntry,
}

#[derive(Debug, Deserialize)]
struct AggResponse {
    aggregations: Aggregations,
}

#[derive(Debug, Deserialize)]
struct Aggregations {
    by_service: BucketAgg,
}

#[derive(Debug, Deserialize)]
struct BucketAgg {
    buckets: Vec<Bucket>,
}

#[derive(Debug, Deserialize)]
struct Bucket {
    key: String,
    doc_count: i64,
}

#[derive(Debug)]
struct ServiceErrorCount {
    service: String,
    count: i64,
}
```

## 选型指南

```text
┌─────────────────────────────────────────────────────────────┐
│                     分析型数据库选型                          │
├───────────────┬─────────────────────────────────────────────┤
│     场景       │              推荐方案                        │
├───────────────┼─────────────────────────────────────────────┤
│ 日志分析       │ Elasticsearch（ELK 生态完善）                │
│ 全文搜索       │ Elasticsearch（功能最强）                    │
│ 实时报表       │ ClickHouse（速度最快）                       │
│ 用户行为分析   │ ClickHouse（漏斗、留存等函数）               │
│ BI 仪表盘     │ ClickHouse / Doris                          │
│ 简单搜索      │ PostgreSQL 全文搜索（够用就行）               │
│ 小数据量分析   │ PostgreSQL（别上牛刀杀鸡）                   │
└───────────────┴─────────────────────────────────────────────┘
```

```text
决策流程：

需要什么能力？
├─ 全文搜索为主 → Elasticsearch
├─ 聚合分析为主 → ClickHouse
└─ 两者都要 →
    │
    数据量多大？
    ├─ < 1000 万行 → PostgreSQL 可能够用
    └─ > 1000 万行 →
        │
        实时性要求？
        ├─ 秒级 → ClickHouse（分析）+ ES（搜索）
        └─ 分钟级 → 考虑数据仓库方案
```

## 核心认知

> **分析型数据库的本质**：用写入灵活性和事务能力，换取海量数据的查询性能。

**OLTP vs OLAP 的选择**：

```text
同一份数据，不同的使用方式：

交易系统（OLTP）：
  用户下单 → PostgreSQL → 毫秒级响应

分析系统（OLAP）：
  数据同步 → ClickHouse → 秒级分析报表

搜索系统（Search）：
  数据同步 → Elasticsearch → 毫秒级搜索

一个成熟的系统往往三者都有，各司其职
```

分析型数据库解决了"海量数据聚合"的问题。但还有一类数据，它的特点太明显，需要专门的优化：

- 数据按时间顺序写入
- 写入后很少修改
- 查询以时间范围为主
- 需要大量聚合计算

这就是**时序数据**——监控指标、IoT 传感器、金融行情。

下一篇，我们来看：**时序数据库——时间维度的专家**。

---

*上一篇：[分布式数据库——当单机不够用](/posts/db-evolution-4-distributed/)*

*下一篇：[时序数据库——时间维度的专家](/posts/db-evolution-6-timeseries/)*

*本系列：*
1. *[关系型数据库——SQL 一统江湖的时代](/posts/db-evolution-1-relational/)*
2. *[NoSQL 运动——打破关系的枷锁](/posts/db-evolution-2-nosql/)*
3. *[嵌入式数据库——藏在你身边的数据库](/posts/db-evolution-3-embedded/)*
4. *[分布式数据库——当单机不够用](/posts/db-evolution-4-distributed/)*
5. *分析型数据库——当查询变成分析（本篇）*
6. *[时序数据库——时间维度的专家](/posts/db-evolution-6-timeseries/)*
7. *[向量数据库——AI 时代的新基建](/posts/db-evolution-7-vector/)*
8. *[PostgreSQL——一个数据库的文艺复兴](/posts/db-evolution-8-postgresql/)*
