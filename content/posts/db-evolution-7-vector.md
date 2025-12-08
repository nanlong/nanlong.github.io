---
title: '数据库演进史（七）：向量数据库——AI 时代的新基建'
date: 2025-12-09T11:00:00+08:00
draft: false
tags: ["数据库", "向量数据库", "pgvector", "PostgreSQL", "AI", "RAG"]
categories: ["技术"]
description: "当你需要找的不是'等于'而是'相似'时，传统数据库彻底无能为力了。"
series: ["数据库演进史"]
---

"找出所有与这张图片相似的商品。"

"搜索语义上相关的文档，而不只是关键词匹配。"

"推荐用户可能喜欢的内容。"

这些需求有什么共同点？它们都需要找**"相似"**而不是**"相等"**的数据。传统数据库在这里彻底无能为力了。

<!--more-->

## 从精确匹配到相似搜索

传统数据库擅长的是**精确匹配**：

```sql
-- 这个，数据库很擅长
SELECT * FROM users WHERE email = 'alice@example.com';

-- 这个，也行
SELECT * FROM products WHERE name LIKE '%手机%';

-- 但这个？
-- "找出所有和这张猫的图片相似的图片"
-- "找出语义上和这段文字意思相近的文档"
-- 传统数据库：？？？
```

问题的核心是：**如何衡量"相似"？**

### 向量：万物的数学表示

答案是把一切都转换成**向量**（一组数字）：

```text
向量化（Embedding）：

"可爱的小猫" → [0.12, -0.45, 0.78, ..., 0.33]  (1536 维)
"adorable kitten" → [0.11, -0.43, 0.79, ..., 0.35]  (1536 维)
"enterprise software" → [-0.82, 0.15, -0.23, ..., 0.91]  (1536 维)

语义相近的内容，向量也相近
语义不同的内容，向量差异大
```

这就是**嵌入向量（Embedding）**的魔力。大语言模型和专门的嵌入模型能把任何东西——文本、图片、音频——转换成固定长度的向量。

```text
向量的来源：

文本嵌入：
  OpenAI text-embedding-3-small → 1536 维
  BERT → 768 维
  BGE → 1024 维

图片嵌入：
  CLIP → 512 维
  ResNet → 2048 维

音频嵌入：
  Whisper → 512 维
```

### 相似度计算

两个向量有多"相似"？常用的度量方式：

```text
1. 余弦相似度（Cosine Similarity）
   衡量方向的相似性，忽略长度
   范围：-1 到 1，1 表示完全相同方向

   适合：文本语义搜索

2. 欧几里得距离（L2 Distance）
   空间中的直线距离
   范围：0 到无穷，0 表示完全相同

   适合：图像搜索

3. 内积（Inner Product / Dot Product）
   综合考虑方向和长度

   适合：推荐系统（向量已归一化时等价于余弦相似度）
```

```text
可视化：

向量 A = [1, 0]
向量 B = [0.9, 0.1]  ← 方向相近
向量 C = [-1, 0]     ← 方向相反

        B
       ↗
A ────→

←──── C

A 和 B 的余弦相似度 ≈ 0.99（很相似）
A 和 C 的余弦相似度 = -1（完全相反）
```

## 暴力搜索的困境

最直接的方法是和所有向量逐一比较：

```text
暴力搜索（Brute Force）：

查询向量 q，数据库有 N 个向量

for i in 1..N:
    distance[i] = compute_distance(q, vectors[i])

return top_k_smallest(distance)

时间复杂度：O(N × D)
  N = 向量数量
  D = 向量维度
```

**问题**：当 N = 1 亿，D = 1536 时，每次搜索需要计算 1536 亿次浮点运算。即使用 GPU 加速，延迟也无法接受。

```text
类比传统数据库：

精确查找：
  有索引 → O(log N)
  无索引 → O(N)

向量搜索：
  需要专门的向量索引
  否则每次都是全表扫描
```

## 向量索引：用精度换速度

向量索引的核心思想：**放弃 100% 精确，换取数量级的速度提升。**

### IVF（倒排文件索引）

```text
IVF (Inverted File) 原理：

1. 聚类：把向量空间分成若干区域（聚类中心）

   ┌─────────────────────────────┐
   │    •    C1    •             │
   │  • • •      • • •           │
   │         C2      C3          │
   │       • • •   • • • •       │
   │                             │
   └─────────────────────────────┘

2. 搜索时：先找最近的几个聚类中心，只在这些区域内搜索

   查询 q 最近的中心是 C2
   → 只搜索 C2 区域内的向量
   → 搜索范围从 N 缩小到 N/num_clusters
```

### HNSW（分层可导航小世界图）

这是目前最流行的向量索引：

```text
HNSW (Hierarchical Navigable Small World) 原理：

多层跳表式结构：

Layer 2:  A ──────────────────── D
          │                      │
Layer 1:  A ──── B ──────── C ── D
          │      │          │    │
Layer 0:  A ─ B ─ C ─ E ─ F ─ G ─ D

搜索过程：
1. 从顶层开始，粗粒度定位
2. 逐层下降，细粒度搜索
3. 最底层得到精确结果

优点：
- 搜索复杂度 O(log N)
- 召回率高（通常 > 95%）
- 不需要训练
```

### 索引对比

```text
┌─────────┬─────────────┬───────────┬──────────────┐
│  索引   │  构建速度   │  查询速度  │   内存占用   │
├─────────┼─────────────┼───────────┼──────────────┤
│ Flat    │  即时      │  最慢      │   最小       │
│ IVF     │  需要训练  │  中等      │   中等       │
│ HNSW    │  较慢      │  最快      │   较大       │
└─────────┴─────────────┴───────────┴──────────────┘

选择建议：
- 数据量小（< 10万）：Flat 就够了
- 数据量中（10万-1000万）：HNSW
- 数据量大（> 1000万）：IVF + 量化压缩
```

## 专用向量数据库

### Milvus：云原生向量数据库

```text
Milvus 架构：

┌────────────────────────────────────────────┐
│                  Milvus                     │
├────────────────────────────────────────────┤
│  Query Node    │  Data Node   │  Index Node │
│  (查询)        │  (存储)      │  (索引)     │
├────────────────────────────────────────────┤
│              Meta Store (etcd)              │
│              Object Storage (S3/MinIO)      │
└────────────────────────────────────────────┘

特点：
- 分布式，水平扩展
- 支持多种索引类型
- 云原生设计
```

### Pinecone：全托管向量服务

```text
Pinecone：

优点：
- 全托管，零运维
- 开箱即用
- 性能优秀

缺点：
- 只有云服务，无法自部署
- 价格较高
- 数据在第三方
```

### Qdrant：Rust 编写的高性能方案

```text
Qdrant：

- Rust 编写，性能优秀
- 支持丰富的过滤条件
- 提供云服务和自部署
- 活跃的开源社区
```

## pgvector：PostgreSQL 的向量扩展

又是 PostgreSQL！没错，PostgreSQL 通过 `pgvector` 扩展也支持向量搜索了。

### 基本使用

```sql
-- 安装扩展
CREATE EXTENSION vector;

-- 创建包含向量的表
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    embedding vector(1536)  -- 1536 维向量
);

-- 插入数据
INSERT INTO documents (title, content, embedding)
VALUES (
    'PostgreSQL简介',
    'PostgreSQL 是一个强大的开源关系数据库...',
    '[0.12, -0.45, 0.78, ...]'::vector
);
```

### 向量搜索

```sql
-- 余弦相似度搜索（<=> 操作符）
SELECT title, content,
       1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 10;

-- L2 距离搜索（<-> 操作符）
SELECT title, embedding <-> '[0.1, 0.2, ...]'::vector AS distance
FROM documents
ORDER BY embedding <-> '[0.1, 0.2, ...]'::vector
LIMIT 10;

-- 内积搜索（<#> 操作符，注意是负的）
SELECT title, (embedding <#> '[0.1, 0.2, ...]'::vector) * -1 AS inner_product
FROM documents
ORDER BY embedding <#> '[0.1, 0.2, ...]'::vector
LIMIT 10;
```

### 创建向量索引

```sql
-- HNSW 索引（推荐，性能最好）
CREATE INDEX ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- IVFFlat 索引（构建更快，适合频繁更新的场景）
CREATE INDEX ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- 索引操作符说明：
-- vector_cosine_ops: 余弦距离
-- vector_l2_ops: 欧几里得距离
-- vector_ip_ops: 内积
```

### 结合传统查询

这是 pgvector 最大的优势——可以和 SQL 无缝结合：

```sql
-- 向量搜索 + 传统过滤
SELECT title, content
FROM documents
WHERE category = 'technology'           -- 传统过滤
  AND created_at > '2024-01-01'         -- 传统过滤
ORDER BY embedding <=> $1::vector       -- 向量排序
LIMIT 10;

-- 向量搜索 + JOIN
SELECT d.title, c.name AS category
FROM documents d
JOIN categories c ON d.category_id = c.id
WHERE d.embedding <=> $1::vector < 0.5  -- 相似度阈值
ORDER BY d.embedding <=> $1::vector
LIMIT 10;

-- 向量搜索 + 全文搜索混合
SELECT title,
       ts_rank(to_tsvector(content), to_tsquery('postgresql')) AS text_rank,
       1 - (embedding <=> $1::vector) AS vector_similarity
FROM documents
WHERE to_tsvector(content) @@ to_tsquery('postgresql')
ORDER BY vector_similarity DESC
LIMIT 10;
```

### Rust 实战：RAG 应用

RAG（Retrieval-Augmented Generation）是 AI 应用的热门架构：

```rust
use sqlx::{PgPool, postgres::PgPoolOptions};
use pgvector::Vector;
use anyhow::Result;

#[derive(Debug, sqlx::FromRow)]
struct Document {
    id: i32,
    title: String,
    content: String,
    similarity: f64,
}

struct KnowledgeBase {
    pool: PgPool,
}

impl KnowledgeBase {
    async fn new(database_url: &str) -> Result<Self> {
        let pool = PgPoolOptions::new()
            .max_connections(5)
            .connect(database_url)
            .await?;

        Ok(Self { pool })
    }

    /// 添加文档到知识库
    async fn add_document(
        &self,
        title: &str,
        content: &str,
        embedding: &[f32],
    ) -> Result<i32> {
        let vector = Vector::from(embedding.to_vec());

        let id = sqlx::query_scalar!(
            r#"
            INSERT INTO documents (title, content, embedding)
            VALUES ($1, $2, $3)
            RETURNING id
            "#,
            title,
            content,
            vector as Vector
        )
        .fetch_one(&self.pool)
        .await?;

        Ok(id)
    }

    /// 语义搜索：找到最相关的文档
    async fn search(
        &self,
        query_embedding: &[f32],
        limit: i32,
    ) -> Result<Vec<Document>> {
        let vector = Vector::from(query_embedding.to_vec());

        let docs = sqlx::query_as!(
            Document,
            r#"
            SELECT
                id,
                title,
                content,
                1 - (embedding <=> $1::vector) AS "similarity!"
            FROM documents
            ORDER BY embedding <=> $1::vector
            LIMIT $2
            "#,
            vector as Vector,
            limit
        )
        .fetch_all(&self.pool)
        .await?;

        Ok(docs)
    }

    /// 带过滤条件的混合搜索
    async fn hybrid_search(
        &self,
        query_embedding: &[f32],
        keyword: &str,
        category: &str,
        limit: i32,
    ) -> Result<Vec<Document>> {
        let vector = Vector::from(query_embedding.to_vec());

        // 注意：中文全文搜索需要安装 zhparser 或 pg_jieba 扩展
        // 这里使用 simple 配置作为示例
        let docs = sqlx::query_as!(
            Document,
            r#"
            SELECT
                d.id,
                d.title,
                d.content,
                1 - (d.embedding <=> $1::vector) AS "similarity!"
            FROM documents d
            WHERE d.category = $2
              AND to_tsvector('simple', d.content) @@ plainto_tsquery('simple', $3)
            ORDER BY d.embedding <=> $1::vector
            LIMIT $4
            "#,
            vector as Vector,
            category,
            keyword,
            limit
        )
        .fetch_all(&self.pool)
        .await?;

        Ok(docs)
    }
}

/// RAG 流程示例
async fn rag_query(
    kb: &KnowledgeBase,
    user_question: &str,
    embedding_client: &impl EmbeddingClient,
    llm_client: &impl LLMClient,
) -> Result<String> {
    // 1. 将问题转换为向量
    let question_embedding = embedding_client
        .embed(user_question)
        .await?;

    // 2. 检索相关文档
    let relevant_docs = kb
        .search(&question_embedding, 5)
        .await?;

    // 3. 构建上下文
    let context = relevant_docs
        .iter()
        .map(|d| format!("标题：{}\n内容：{}", d.title, d.content))
        .collect::<Vec<_>>()
        .join("\n\n---\n\n");

    // 4. 调用 LLM 生成回答
    let prompt = format!(
        "基于以下参考资料回答问题。\n\n\
         参考资料：\n{}\n\n\
         问题：{}\n\n\
         回答：",
        context, user_question
    );

    let answer = llm_client.generate(&prompt).await?;

    Ok(answer)
}

// 特征定义
#[async_trait::async_trait]
trait EmbeddingClient {
    async fn embed(&self, text: &str) -> Result<Vec<f32>>;
}

#[async_trait::async_trait]
trait LLMClient {
    async fn generate(&self, prompt: &str) -> Result<String>;
}
```

### pgvector 的局限

```text
pgvector 的限制：

1. 性能天花板
   - 单机性能有限
   - 超过 1000 万向量开始吃力
   - 无法像 Milvus 那样分布式扩展

2. 索引构建
   - HNSW 索引构建较慢
   - 大数据集需要足够内存

3. 高级功能缺失
   - 没有量化压缩（减少内存占用）
   - 没有 GPU 加速支持

适用场景：
✓ 向量数量 < 1000 万
✓ 已有 PostgreSQL 技术栈
✓ 需要向量与业务数据关联
✓ 对成本敏感（不想引入新组件）

不适用：
✗ 超大规模向量搜索（> 1 亿）
✗ 极低延迟要求（< 10ms）
✗ 需要分布式架构
```

## 向量数据库选型

```text
选 pgvector 如果：
  ✓ 数据量适中（< 1000 万向量）
  ✓ 已有 PostgreSQL
  ✓ 需要 SQL 能力和事务
  ✓ 成本敏感

选 Qdrant 如果：
  ✓ 需要高性能
  ✓ 可以自部署
  ✓ 需要丰富的过滤能力
  ✓ Rust 技术栈

选 Milvus 如果：
  ✓ 超大规模数据
  ✓ 需要分布式
  ✓ 已有 Kubernetes 基础设施

选 Pinecone 如果：
  ✓ 全托管，零运维
  ✓ 快速上线
  ✓ 预算充足
```

## 核心认知

> **向量数据库的本质**：把"相似"这个模糊的概念，转化为向量空间中"距离"这个精确的数学问题。

**AI 时代的数据库趋势**：

```text
传统应用：
  用户请求 → 精确查询 → 返回结果
  "给我 ID=123 的订单"

AI 应用：
  用户请求 → 语义理解 → 向量搜索 → 上下文构建 → LLM 生成
  "帮我找找类似这个风格的产品"

数据库需要同时支持：
- 精确查询（关系模型）
- 模糊匹配（全文搜索）
- 语义搜索（向量搜索）

PostgreSQL + pgvector 提供了一站式解决方案
```

下一篇是本系列的终章。我们将回顾 PostgreSQL 如何通过扩展机制，一步步成为能够应对几乎所有场景的"瑞士军刀"——从关系型到文档、从时序到向量，一个数据库的文艺复兴。

---

*上一篇：[时序数据库——时间维度的专家](/posts/db-evolution-6-timeseries/)*

*下一篇：[PostgreSQL——一个数据库的文艺复兴](/posts/db-evolution-8-postgresql/)*

*本系列：*
1. *[关系型数据库——SQL 一统江湖的时代](/posts/db-evolution-1-relational/)*
2. *[NoSQL 运动——打破关系的枷锁](/posts/db-evolution-2-nosql/)*
3. *[嵌入式数据库——藏在你身边的数据库](/posts/db-evolution-3-embedded/)*
4. *[分布式数据库——当单机不够用](/posts/db-evolution-4-distributed/)*
5. *[分析型数据库——当查询变成分析](/posts/db-evolution-5-olap-search/)*
6. *[时序数据库——时间维度的专家](/posts/db-evolution-6-timeseries/)*
7. *向量数据库——AI 时代的新基建（本篇）*
8. *[PostgreSQL——一个数据库的文艺复兴](/posts/db-evolution-8-postgresql/)*
