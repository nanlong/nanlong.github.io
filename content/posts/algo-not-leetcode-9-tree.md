---
title: "算法不是面试题（九）：树结构——层级数据的优雅解法"
date: 2025-12-08T11:20:00+08:00
draft: false
tags: ["算法", "树", "层级数据", "邻接表", "闭包表"]
categories: ["技术"]
description: "组织架构、分类目录、评论回复——这些层级数据怎么存？邻接表、路径枚举、闭包表、嵌套集，四种方案各有优劣。"
series: ["算法不是面试题"]
---

组织架构、分类目录、评论回复——这些层级数据怎么存？邻接表、路径枚举、闭包表、嵌套集，四种方案各有优劣。

<!--more-->

## 层级数据的挑战

树形结构无处不在：

```text
组织架构：
  CEO
  ├── CTO
  │   ├── 后端组长
  │   │   └── 后端工程师
  │   └── 前端组长
  └── CFO
      └── 财务专员

商品分类：
  电子产品
  ├── 手机
  │   ├── 苹果
  │   └── 安卓
  └── 电脑
      ├── 笔记本
      └── 台式机

评论回复：
  评论 1
  ├── 回复 1.1
  │   └── 回复 1.1.1
  └── 回复 1.2
```

**核心问题**：关系型数据库是"表"，天然适合扁平数据，怎么存树？

## 方案一：邻接表

最直观的方案：每行记录自己的父节点。

```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    parent_id INTEGER REFERENCES categories(id)
);

-- 数据
-- id | name       | parent_id
-- 1  | 电子产品   | NULL
-- 2  | 手机       | 1
-- 3  | 苹果       | 2
-- 4  | 安卓       | 2
-- 5  | 电脑       | 1
```

### 操作实现

```rust
use anyhow::Result;
use sqlx::PgPool;

#[derive(Debug, sqlx::FromRow)]
struct Category {
    id: i32,
    name: String,
    parent_id: Option<i32>,
}

// 获取直接子节点
async fn get_children(pool: &PgPool, parent_id: i32) -> Result<Vec<Category>> {
    sqlx::query_as!(
        Category,
        "SELECT * FROM categories WHERE parent_id = $1",
        parent_id
    )
    .fetch_all(pool)
    .await
}

// 获取父节点
async fn get_parent(pool: &PgPool, id: i32) -> Result<Option<Category>> {
    sqlx::query_as!(
        Category,
        r#"
        SELECT p.* FROM categories c
        JOIN categories p ON c.parent_id = p.id
        WHERE c.id = $1
        "#,
        id
    )
    .fetch_optional(pool)
    .await
}
```

### 获取所有祖先（难点）

```rust
// 递归 CTE（PostgreSQL 8.4+）
async fn get_ancestors(pool: &PgPool, id: i32) -> Result<Vec<Category>> {
    sqlx::query_as!(
        Category,
        r#"
        WITH RECURSIVE ancestors AS (
            -- 起点
            SELECT * FROM categories WHERE id = $1
            UNION ALL
            -- 递归
            SELECT c.* FROM categories c
            JOIN ancestors a ON c.id = a.parent_id
        )
        SELECT * FROM ancestors WHERE id != $1
        "#,
        id
    )
    .fetch_all(pool)
    .await
}
```

### 获取所有后代

```rust
async fn get_descendants(pool: &PgPool, id: i32) -> Result<Vec<Category>> {
    sqlx::query_as!(
        Category,
        r#"
        WITH RECURSIVE descendants AS (
            SELECT * FROM categories WHERE parent_id = $1
            UNION ALL
            SELECT c.* FROM categories c
            JOIN descendants d ON c.parent_id = d.id
        )
        SELECT * FROM descendants
        "#,
        id
    )
    .fetch_all(pool)
    .await
}
```

### 优缺点

| 优点 | 缺点 |
|------|------|
| 结构简单 | 查子树需要递归 |
| 插入/移动简单 | 深层嵌套性能差 |
| 存储空间小 | 不支持递归的数据库很痛苦 |

**适用场景**：层级不深、主要查直接子节点的场景。

## 方案二：路径枚举

存储从根到当前节点的完整路径。

```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    path VARCHAR(500)  -- 路径，如 "1/2/3"
);

-- id | name     | path
-- 1  | 电子产品 | 1
-- 2  | 手机     | 1/2
-- 3  | 苹果     | 1/2/3
-- 4  | 安卓     | 1/2/4
-- 5  | 电脑     | 1/5
```

### 操作实现

```rust
use anyhow::Result;
use sqlx::PgPool;

#[derive(Debug, sqlx::FromRow)]
struct Category {
    id: i32,
    name: String,
    path: String,
}

// 获取所有祖先
async fn get_ancestors(pool: &PgPool, path: &str) -> Result<Vec<Category>> {
    let ancestor_ids: Vec<i32> = path
        .split('/')
        .filter_map(|s| s.parse().ok())
        .collect();

    sqlx::query_as!(
        Category,
        "SELECT * FROM categories WHERE id = ANY($1)",
        &ancestor_ids
    )
    .fetch_all(pool)
    .await
}

// 获取所有后代
async fn get_descendants(pool: &PgPool, path: &str) -> Result<Vec<Category>> {
    sqlx::query_as!(
        Category,
        "SELECT * FROM categories WHERE path LIKE $1",
        format!("{}/%", path)  // 前缀匹配
    )
    .fetch_all(pool)
    .await
}

// 判断是否是祖先
fn is_ancestor(ancestor_path: &str, descendant_path: &str) -> bool {
    descendant_path.starts_with(&format!("{}/", ancestor_path))
}
```

### 插入和移动

```rust
// 插入节点
async fn insert(pool: &PgPool, name: &str, parent_path: &str) -> Result<Category> {
    let result = sqlx::query_as!(
        Category,
        r#"
        INSERT INTO categories (name, path)
        VALUES ($1, '')
        RETURNING *
        "#,
        name
    )
    .fetch_one(pool)
    .await?;

    // 更新路径
    let new_path = format!("{}/{}", parent_path, result.id);
    sqlx::query!(
        "UPDATE categories SET path = $1 WHERE id = $2",
        new_path,
        result.id
    )
    .execute(pool)
    .await?;

    Ok(result)
}

// 移动子树（需要更新所有后代的路径）
async fn move_subtree(
    pool: &PgPool,
    node_id: i32,
    old_path: &str,
    new_parent_path: &str,
) -> Result<()> {
    let new_path = format!("{}/{}", new_parent_path, node_id);

    // 更新所有后代的路径
    sqlx::query!(
        r#"
        UPDATE categories
        SET path = REPLACE(path, $1, $2)
        WHERE path LIKE $3
        "#,
        old_path,
        new_path,
        format!("{}%", old_path)
    )
    .execute(pool)
    .await?;

    Ok(())
}
```

### 优缺点

| 优点 | 缺点 |
|------|------|
| 祖先/后代查询快 | 移动节点要更新很多行 |
| 不需要递归 | 路径长度有限制 |
| 层级一目了然 | 依赖字符串操作 |

**适用场景**：层级固定、很少移动节点的场景（如商品分类）。

## 方案三：闭包表

专门用一张表存储所有的祖先-后代关系。

```sql
-- 节点表
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

-- 闭包表
CREATE TABLE category_closure (
    ancestor_id INTEGER REFERENCES categories(id),
    descendant_id INTEGER REFERENCES categories(id),
    depth INTEGER,
    PRIMARY KEY (ancestor_id, descendant_id)
);
```

```text
树结构：
  1 (电子产品)
  ├── 2 (手机)
  │   └── 3 (苹果)
  └── 4 (电脑)

闭包表：
ancestor | descendant | depth
1        | 1          | 0     (自己到自己)
1        | 2          | 1
1        | 3          | 2
1        | 4          | 1
2        | 2          | 0
2        | 3          | 1
3        | 3          | 0
4        | 4          | 0
```

### 操作实现

```rust
use anyhow::Result;
use sqlx::PgPool;

#[derive(Debug, sqlx::FromRow)]
struct Category {
    id: i32,
    name: String,
}

// 获取所有祖先
async fn get_ancestors(pool: &PgPool, id: i32) -> Result<Vec<Category>> {
    sqlx::query_as!(
        Category,
        r#"
        SELECT c.* FROM categories c
        JOIN category_closure cc ON c.id = cc.ancestor_id
        WHERE cc.descendant_id = $1 AND cc.depth > 0
        ORDER BY cc.depth DESC
        "#,
        id
    )
    .fetch_all(pool)
    .await
}

// 获取所有后代
async fn get_descendants(pool: &PgPool, id: i32) -> Result<Vec<Category>> {
    sqlx::query_as!(
        Category,
        r#"
        SELECT c.* FROM categories c
        JOIN category_closure cc ON c.id = cc.descendant_id
        WHERE cc.ancestor_id = $1 AND cc.depth > 0
        ORDER BY cc.depth
        "#,
        id
    )
    .fetch_all(pool)
    .await
}

// 获取直接子节点
async fn get_children(pool: &PgPool, id: i32) -> Result<Vec<Category>> {
    sqlx::query_as!(
        Category,
        r#"
        SELECT c.* FROM categories c
        JOIN category_closure cc ON c.id = cc.descendant_id
        WHERE cc.ancestor_id = $1 AND cc.depth = 1
        "#,
        id
    )
    .fetch_all(pool)
    .await
}
```

### 插入节点

```rust
async fn insert(pool: &PgPool, name: &str, parent_id: i32) -> Result<Category> {
    let mut tx = pool.begin().await?;

    // 插入节点
    let node = sqlx::query_as!(
        Category,
        "INSERT INTO categories (name) VALUES ($1) RETURNING *",
        name
    )
    .fetch_one(&mut *tx)
    .await?;

    // 插入闭包关系
    // 1. 自己到自己
    // 2. 复制父节点的所有祖先关系，depth + 1
    sqlx::query!(
        r#"
        INSERT INTO category_closure (ancestor_id, descendant_id, depth)
        SELECT ancestor_id, $1, depth + 1
        FROM category_closure
        WHERE descendant_id = $2
        UNION ALL
        SELECT $1, $1, 0
        "#,
        node.id,
        parent_id
    )
    .execute(&mut *tx)
    .await?;

    tx.commit().await?;
    Ok(node)
}
```

### 删除子树

```rust
use anyhow::Result;
use sqlx::PgPool;

async fn delete_subtree(pool: &PgPool, id: i32) -> Result<()> {
    let mut tx = pool.begin().await?;

    // 找到所有后代
    let descendant_ids: Vec<i32> = sqlx::query_scalar!(
        "SELECT descendant_id FROM category_closure WHERE ancestor_id = $1",
        id
    )
    .fetch_all(&mut *tx)
    .await?
    .into_iter()
    .flatten()
    .collect();

    // 删除闭包关系
    sqlx::query!(
        "DELETE FROM category_closure WHERE descendant_id = ANY($1)",
        &descendant_ids
    )
    .execute(&mut *tx)
    .await?;

    // 删除节点
    sqlx::query!(
        "DELETE FROM categories WHERE id = ANY($1)",
        &descendant_ids
    )
    .execute(&mut *tx)
    .await?;

    tx.commit().await?;
    Ok(())
}
```

### 优缺点

| 优点 | 缺点 |
|------|------|
| 祖先/后代查询 O(1) | 存储空间大（O(n²) 最坏） |
| 支持任意深度 | 插入要写多行 |
| 查询简单直观 | 移动节点复杂 |

**适用场景**：频繁查询祖先/后代、深度不确定的场景。

## 方案四：嵌套集

用左右值表示节点在树中的位置。

```text
树结构：
       1 (电子产品)
      / \
     2   3
   (手机) (电脑)
    /
   4
  (苹果)

遍历顺序（深度优先）：
  进入电子产品 (left=1)
    进入手机 (left=2)
      进入苹果 (left=3)
      离开苹果 (right=4)
    离开手机 (right=5)
    进入电脑 (left=6)
    离开电脑 (right=7)
  离开电子产品 (right=8)

存储：
id | name     | left | right
1  | 电子产品 | 1    | 8
2  | 手机     | 2    | 5
3  | 电脑     | 6    | 7
4  | 苹果     | 3    | 4
```

### 规则

```text
节点 A 是节点 B 的祖先 ⟺ A.left < B.left AND A.right > B.right

子节点数量 = (right - left - 1) / 2
```

### 操作实现

```rust
use anyhow::Result;
use sqlx::PgPool;

#[derive(Debug, sqlx::FromRow)]
struct Category {
    id: i32,
    name: String,
    left: i32,
    right: i32,
}

// 获取所有后代
async fn get_descendants(pool: &PgPool, id: i32) -> Result<Vec<Category>> {
    sqlx::query_as!(
        Category,
        r#"
        SELECT c2.* FROM categories c1, categories c2
        WHERE c1.id = $1
          AND c2.left > c1.left
          AND c2.right < c1.right
        ORDER BY c2.left
        "#,
        id
    )
    .fetch_all(pool)
    .await
}

// 获取所有祖先
async fn get_ancestors(pool: &PgPool, id: i32) -> Result<Vec<Category>> {
    sqlx::query_as!(
        Category,
        r#"
        SELECT c2.* FROM categories c1, categories c2
        WHERE c1.id = $1
          AND c2.left < c1.left
          AND c2.right > c1.right
        ORDER BY c2.left
        "#,
        id
    )
    .fetch_all(pool)
    .await
}
```

### 插入节点（复杂！）

```rust
async fn insert(pool: &PgPool, name: &str, parent_id: i32) -> Result<Category> {
    let mut tx = pool.begin().await?;

    // 获取父节点的 right 值
    let parent = sqlx::query_as!(
        Category,
        "SELECT * FROM categories WHERE id = $1",
        parent_id
    )
    .fetch_one(&mut *tx)
    .await?;

    let insert_at = parent.right;

    // 为新节点腾出空间：所有 >= insert_at 的值 +2
    sqlx::query!(
        "UPDATE categories SET right = right + 2 WHERE right >= $1",
        insert_at
    )
    .execute(&mut *tx)
    .await?;

    sqlx::query!(
        "UPDATE categories SET left = left + 2 WHERE left > $1",
        insert_at
    )
    .execute(&mut *tx)
    .await?;

    // 插入新节点
    let node = sqlx::query_as!(
        Category,
        r#"
        INSERT INTO categories (name, left, right)
        VALUES ($1, $2, $3)
        RETURNING *
        "#,
        name,
        insert_at,
        insert_at + 1
    )
    .fetch_one(&mut *tx)
    .await?;

    tx.commit().await?;
    Ok(node)
}
```

### 优缺点

| 优点 | 缺点 |
|------|------|
| 查询极快（无 JOIN） | 插入/删除要更新很多行 |
| 存储空间小 | 并发困难 |
| 子树统计方便 | 调试困难 |

**适用场景**：读多写少、需要高性能查询的场景。

## 方案对比

| 方案 | 查子节点 | 查所有后代 | 查祖先 | 插入 | 删除 | 移动 |
|------|----------|------------|--------|------|------|------|
| 邻接表 | O(n) | O(n)递归 | O(d)递归 | O(1) | O(n) | O(1) |
| 路径枚举 | O(n) | O(n) | O(1) | O(1) | O(n) | O(n) |
| 闭包表 | O(1) | O(1) | O(1) | O(d) | O(n) | O(n²) |
| 嵌套集 | O(n) | O(1) | O(1) | O(n) | O(n) | O(n) |

n = 节点数，d = 深度

### 选择建议

| 场景 | 推荐方案 |
|------|----------|
| 简单层级、写多读少 | 邻接表 |
| 层级固定、路径重要 | 路径枚举 |
| 频繁查祖先/后代 | 闭包表 |
| 读多写少、需要统计 | 嵌套集 |
| 不确定 | 邻接表（最简单） |

## 实战：评论系统

评论系统的特点：

- 层级不深（通常 3-5 层）
- 读多写多
- 需要分页

```rust
use anyhow::Result;
use chrono::{DateTime, Utc};
use sqlx::PgPool;

// 使用邻接表 + 物化路径的混合方案
#[derive(Debug, sqlx::FromRow)]
struct Comment {
    id: i64,
    content: String,
    parent_id: Option<i64>,
    path: String,           // 物化路径，用于排序
    depth: i32,             // 深度，用于缩进
    created_at: DateTime<Utc>,
}

// 获取帖子的所有评论（按层级排序）
async fn get_comments(pool: &PgPool, post_id: i64) -> Result<Vec<Comment>> {
    sqlx::query_as!(
        Comment,
        r#"
        SELECT * FROM comments
        WHERE post_id = $1
        ORDER BY path, created_at
        "#,
        post_id
    )
    .fetch_all(pool)
    .await
}

// 插入评论
async fn insert_comment(
    pool: &PgPool,
    post_id: i64,
    parent_id: Option<i64>,
    content: &str,
) -> Result<Comment> {
    let (parent_path, depth) = if let Some(pid) = parent_id {
        let parent = sqlx::query_as!(
            Comment,
            "SELECT * FROM comments WHERE id = $1",
            pid
        )
        .fetch_one(pool)
        .await?;
        (parent.path, parent.depth + 1)
    } else {
        (String::new(), 0)
    };

    let comment = sqlx::query_as!(
        Comment,
        r#"
        INSERT INTO comments (post_id, parent_id, content, path, depth)
        VALUES ($1, $2, $3, '', $4)
        RETURNING *
        "#,
        post_id,
        parent_id,
        content,
        depth
    )
    .fetch_one(pool)
    .await?;

    // 更新路径
    let new_path = if parent_path.is_empty() {
        format!("{:010}", comment.id)
    } else {
        format!("{}/{:010}", parent_path, comment.id)
    };

    sqlx::query!(
        "UPDATE comments SET path = $1 WHERE id = $2",
        new_path,
        comment.id
    )
    .execute(pool)
    .await?;

    Ok(comment)
}
```

## 总结

| 方案 | 核心思想 | 最佳场景 |
|------|----------|----------|
| 邻接表 | 只存父节点 | 简单、层级浅 |
| 路径枚举 | 存完整路径 | 层级固定、排序重要 |
| 闭包表 | 存所有关系 | 频繁查询祖先/后代 |
| 嵌套集 | 左右值编码 | 读多写少 |

**核心认知**：

> 树结构的存储没有银弹。选择方案要根据：读写比例、层级深度、查询模式、并发要求。大多数场景，从最简单的邻接表开始，遇到性能问题再考虑其他方案。

---

*上一篇：[跳表：Redis 排行榜的秘密](/posts/algo-not-leetcode-8-skiplist/)*

*下一篇：[字符串匹配：敏感词过滤的正确姿势](/posts/algo-not-leetcode-10-string-match/)*

*本系列：*
1. *[算法是业务武器](/posts/algo-not-leetcode-1-intro/)*
2. *[排序与二分：被低估的基础功](/posts/algo-not-leetcode-2-sort-binary-search/)*
3. *[哈希表：缓存设计的基石](/posts/algo-not-leetcode-3-hashtable/)*
4. *[布隆过滤器：用 1% 误判换 90% 内存](/posts/algo-not-leetcode-4-bloom-filter/)*
5. *[一致性哈希：分布式系统的路由表](/posts/algo-not-leetcode-5-consistent-hash/)*
6. *[堆与优先队列：调度器的核心](/posts/algo-not-leetcode-6-heap/)*
7. *[限流算法：保护系统的三道防线](/posts/algo-not-leetcode-7-rate-limit/)*
8. *[跳表：Redis 排行榜的秘密](/posts/algo-not-leetcode-8-skiplist/)*
9. *树结构：层级数据的优雅解法（本篇）*
10. *[字符串匹配：敏感词过滤的正确姿势](/posts/algo-not-leetcode-10-string-match/)*
11. *[实战选型：没有银弹，只有场景](/posts/algo-not-leetcode-11-selection/)*
