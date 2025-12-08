---
title: '数据库演进史（三）：嵌入式数据库——藏在你身边的数据库'
date: 2025-12-09T10:20:00+08:00
draft: false
tags: ["数据库", "SQLite", "DuckDB", "嵌入式"]
categories: ["技术"]
description: "你的手机里有它，你的浏览器里有它，你的桌面应用里有它。这个数据库全球部署量超过万亿，但你可能从未注意过它。"
series: ["数据库演进史"]
---

问一个问题：**世界上部署量最大的数据库是什么？**

不是 MySQL，不是 PostgreSQL，不是 Oracle。

答案是 **SQLite**——一个你可能从未直接使用过，但每天都在间接使用的数据库。

<!--more-->

## 什么是嵌入式数据库？

传统数据库是**客户端-服务器**架构：

```text
传统数据库（PostgreSQL、MySQL）：

┌─────────────┐         ┌─────────────┐
│  应用程序   │ ──TCP──→ │  数据库服务  │
│  (客户端)   │ ←─────── │   (服务器)  │
└─────────────┘         └─────────────┘
                              │
                        ┌─────▼─────┐
                        │  数据文件  │
                        └───────────┘

特点：
- 需要独立的服务器进程
- 通过网络协议通信
- 需要安装、配置、维护
- 支持多客户端并发访问
```

嵌入式数据库是**进程内**架构：

```text
嵌入式数据库（SQLite、DuckDB）：

┌─────────────────────────────┐
│         应用程序             │
│  ┌─────────────────────┐    │
│  │   数据库引擎（库）    │    │
│  │   - 直接函数调用      │    │
│  │   - 无网络开销        │    │
│  └──────────┬──────────┘    │
└─────────────┼───────────────┘
              │
        ┌─────▼─────┐
        │  数据文件  │  ← 就是一个普通文件
        └───────────┘

特点：
- 没有服务器进程
- 直接链接到应用程序
- 零配置，开箱即用
- 数据库就是一个文件
```

## SQLite：无处不在的数据库

### 惊人的普及度

```text
SQLite 在哪里？

📱 每部手机
   - iOS：系统应用、通讯录、短信、Safari
   - Android：系统设置、联系人、短信、几乎所有 App

💻 每台电脑
   - macOS：Mail、Photos、Safari、Spotlight
   - Windows：Edge、Skype、Windows 10+ 系统组件
   - Linux：Firefox、Chrome、GNOME 应用

🌐 每个浏览器
   - Chrome、Firefox、Safari、Edge
   - IndexedDB 底层实现
   - 浏览器历史、书签、缓存

📺 每个智能设备
   - 智能电视、机顶盒
   - 车载系统
   - 智能家居设备

保守估计：全球有超过 1 万亿个 SQLite 数据库在运行
```

### 为什么选择 SQLite？

```text
SQLite 的设计哲学：

1. 零配置
   - 不需要安装
   - 不需要配置文件
   - 不需要管理员

2. 自包含
   - 整个数据库就是一个文件
   - 没有外部依赖
   - 跨平台直接复制

3. 事务安全
   - 完整的 ACID 支持
   - 断电不会损坏数据
   - 原子提交

4. 足够快
   - 简单查询比 C/S 数据库更快（无网络开销）
   - 适合读多写少的场景
```

### 基本使用

```sql
-- SQLite 特色：动态类型
CREATE TABLE users (
    id INTEGER PRIMARY KEY,  -- 自动成为 rowid 别名
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    created_at TEXT DEFAULT (datetime('now'))
);

-- 插入数据
INSERT INTO users (name, email) VALUES ('张三', 'zhang@example.com');

-- 查询
SELECT * FROM users WHERE name LIKE '张%';

-- 实用函数
SELECT
    date('now') AS today,
    datetime('now', 'localtime') AS local_time,
    json_extract('{"name": "test"}', '$.name') AS json_value;
```

### Rust 中使用 SQLite

```rust
use rusqlite::{Connection, Result, params};
use anyhow::Context;

#[derive(Debug)]
struct User {
    id: i64,
    name: String,
    email: Option<String>,
}

struct UserRepository {
    conn: Connection,
}

impl UserRepository {
    /// 打开或创建数据库
    fn open(path: &str) -> Result<Self> {
        let conn = Connection::open(path)?;

        // 启用外键约束（SQLite 默认关闭）
        conn.execute_batch("PRAGMA foreign_keys = ON;")?;

        // 使用 WAL 模式提升并发性能
        conn.execute_batch("PRAGMA journal_mode = WAL;")?;

        Ok(Self { conn })
    }

    /// 初始化表结构
    fn init(&self) -> Result<()> {
        self.conn.execute_batch(
            "CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY,
                name TEXT NOT NULL,
                email TEXT UNIQUE,
                created_at TEXT DEFAULT (datetime('now'))
            );"
        )?;
        Ok(())
    }

    /// 创建用户
    fn create(&self, name: &str, email: Option<&str>) -> Result<i64> {
        self.conn.execute(
            "INSERT INTO users (name, email) VALUES (?1, ?2)",
            params![name, email],
        )?;
        Ok(self.conn.last_insert_rowid())
    }

    /// 查询用户
    fn find_by_id(&self, id: i64) -> Result<Option<User>> {
        let mut stmt = self.conn.prepare(
            "SELECT id, name, email FROM users WHERE id = ?1"
        )?;

        let user = stmt.query_row(params![id], |row| {
            Ok(User {
                id: row.get(0)?,
                name: row.get(1)?,
                email: row.get(2)?,
            })
        }).optional()?;

        Ok(user)
    }

    /// 事务示例：转移 email 到另一个用户
    fn transfer_email(&mut self, from_id: i64, to_id: i64) -> Result<()> {
        let tx = self.conn.transaction()?;

        // 先获取原始 email
        let email: Option<String> = tx.query_row(
            "SELECT email FROM users WHERE id = ?1",
            params![from_id],
            |row| row.get(0),
        )?;

        // 设置目标用户的 email
        tx.execute(
            "UPDATE users SET email = ?1 WHERE id = ?2",
            params![email, to_id],
        )?;

        // 清空原用户的 email
        tx.execute(
            "UPDATE users SET email = NULL WHERE id = ?1",
            params![from_id],
        )?;

        tx.commit()?;
        Ok(())
    }
}
```

### SQLite 的局限

```text
SQLite 不适合：

1. 高并发写入
   - 写锁是库级别的（整个数据库）
   - 不适合多用户 Web 应用

2. 大数据量
   - 理论上限 281 TB
   - 实际上超过几十 GB 就该考虑其他方案

3. 网络访问
   - 设计上就是本地数据库
   - 不要把 SQLite 文件放在 NFS 上

4. 复杂查询
   - 没有存储过程
   - 窗口函数支持有限（较新版本已改善）
```

## DuckDB：分析界的 SQLite

如果说 SQLite 是 OLTP 的嵌入式之王，那 DuckDB 就是 **OLAP 的嵌入式之王**。

### DuckDB 是什么？

```text
DuckDB 的定位：

SQLite 之于 PostgreSQL = DuckDB 之于 ClickHouse

┌─────────────────────────────────────────────────┐
│                   OLTP                           │
│  PostgreSQL（服务器）    SQLite（嵌入式）         │
├─────────────────────────────────────────────────┤
│                   OLAP                           │
│  ClickHouse（服务器）    DuckDB（嵌入式）         │
└─────────────────────────────────────────────────┘

DuckDB 特点：
- 零依赖，单文件
- 列式存储，向量化执行
- 可以直接查询 CSV、Parquet、JSON
- 支持完整的 SQL（包括窗口函数）
```

### 为什么 DuckDB 火了？

```text
2020 年之前的数据分析：

数据科学家："我有一个 2GB 的 CSV 文件要分析"

方案 A：Python + Pandas
  - 加载到内存 → 内存不够
  - 处理大文件很慢

方案 B：导入数据库
  - 安装 PostgreSQL → 太麻烦
  - 导入数据 → 太慢
  - 分析完还要清理 → 太烦

方案 C：上云
  - 配置 BigQuery/Snowflake → 要钱
  - 学习新语法 → 要时间

2020 年之后：

数据科学家："我有一个 2GB 的 CSV 文件要分析"

DuckDB：
  pip install duckdb
  >>> import duckdb
  >>> duckdb.sql("SELECT * FROM 'data.csv' WHERE amount > 1000")

  就这样，完事了。
```

### 直接查询文件

DuckDB 最惊艳的功能是**直接查询文件**，无需导入：

```sql
-- 直接查询 CSV
SELECT * FROM 'sales_2024.csv' LIMIT 10;

-- 直接查询 Parquet（列式文件格式）
SELECT
    region,
    SUM(amount) AS total
FROM 'sales/*.parquet'  -- 支持通配符！
GROUP BY region;

-- 直接查询 JSON
SELECT * FROM read_json_auto('events.json');

-- 甚至可以查询远程文件
SELECT * FROM 'https://example.com/data.parquet';

-- 多文件联合查询
SELECT
    a.user_id,
    a.order_amount,
    b.user_name
FROM 'orders.csv' a
JOIN 'users.parquet' b ON a.user_id = b.id;
```

### Rust 中使用 DuckDB

```rust
use duckdb::{Connection, Result, params};
use anyhow::Context;

struct DataAnalyzer {
    conn: Connection,
}

impl DataAnalyzer {
    /// 创建内存数据库（临时分析）
    fn in_memory() -> Result<Self> {
        let conn = Connection::open_in_memory()?;
        Ok(Self { conn })
    }

    /// 创建持久化数据库
    fn open(path: &str) -> Result<Self> {
        let conn = Connection::open(path)?;
        Ok(Self { conn })
    }

    /// 直接分析 CSV 文件
    fn analyze_csv(&self, csv_path: &str) -> Result<Vec<SalesSummary>> {
        let mut stmt = self.conn.prepare(&format!(
            "SELECT
                region,
                COUNT(*) AS order_count,
                SUM(amount) AS total_amount,
                AVG(amount) AS avg_amount
            FROM '{}'
            GROUP BY region
            ORDER BY total_amount DESC",
            csv_path
        ))?;

        let results = stmt.query_map([], |row| {
            Ok(SalesSummary {
                region: row.get(0)?,
                order_count: row.get(1)?,
                total_amount: row.get(2)?,
                avg_amount: row.get(3)?,
            })
        })?;

        results.collect()
    }

    /// 复杂分析：窗口函数
    fn ranking_analysis(&self, parquet_path: &str) -> Result<Vec<RankingResult>> {
        let mut stmt = self.conn.prepare(&format!(
            "SELECT
                product_id,
                product_name,
                sales,
                RANK() OVER (ORDER BY sales DESC) AS rank,
                sales * 100.0 / SUM(sales) OVER () AS percentage
            FROM '{}'
            QUALIFY rank <= 10",  -- DuckDB 特有语法：过滤窗口函数结果
            parquet_path
        ))?;

        let results = stmt.query_map([], |row| {
            Ok(RankingResult {
                product_id: row.get(0)?,
                product_name: row.get(1)?,
                sales: row.get(2)?,
                rank: row.get(3)?,
                percentage: row.get(4)?,
            })
        })?;

        results.collect()
    }

    /// 导出分析结果
    fn export_to_parquet(&self, query: &str, output_path: &str) -> Result<()> {
        self.conn.execute_batch(&format!(
            "COPY ({}) TO '{}' (FORMAT PARQUET)",
            query, output_path
        ))?;
        Ok(())
    }
}

#[derive(Debug)]
struct SalesSummary {
    region: String,
    order_count: i64,
    total_amount: f64,
    avg_amount: f64,
}

#[derive(Debug)]
struct RankingResult {
    product_id: i64,
    product_name: String,
    sales: f64,
    rank: i64,
    percentage: f64,
}
```

### DuckDB vs Pandas

```text
场景：分析一个 10GB 的 CSV 文件

Pandas：
  import pandas as pd
  df = pd.read_csv('big_file.csv')  # 💥 内存不够

  # 即使够，也要全部加载到内存
  # 处理速度慢

DuckDB：
  import duckdb
  result = duckdb.sql("""
      SELECT region, SUM(amount)
      FROM 'big_file.csv'
      GROUP BY region
  """)

  # 流式处理，不需要全部加载
  # 列式扫描，只读取需要的列
  # 向量化执行，速度快 10-100 倍
```

### SQLite vs DuckDB

```text
┌─────────────┬────────────────────┬────────────────────┐
│    维度     │      SQLite        │      DuckDB        │
├─────────────┼────────────────────┼────────────────────┤
│  存储模型   │ 行式存储           │ 列式存储           │
│  优化目标   │ OLTP（事务）       │ OLAP（分析）       │
│  典型查询   │ 按 ID 查单行       │ 聚合百万行         │
│  文件查询   │ 不支持             │ CSV/Parquet/JSON   │
│  并发写入   │ 库级锁             │ 表级锁             │
│  成熟度     │ 20+ 年，极其稳定   │ 较新，快速迭代     │
│  生态       │ 极其丰富           │ 快速增长中         │
├─────────────┼────────────────────┼────────────────────┤
│  适用场景   │ App 本地存储       │ 数据分析、ETL      │
│            │ 配置文件           │ 数据科学           │
│            │ 缓存               │ 嵌入式 BI          │
└─────────────┴────────────────────┴────────────────────┘
```

## 嵌入式数据库的现代应用

```text
应用场景：

1. 桌面应用
   - 用户数据存储（SQLite）
   - 本地分析功能（DuckDB）
   - 离线缓存

2. 移动应用
   - 本地数据库（SQLite）
   - 离线优先架构

3. CLI 工具
   - 状态持久化
   - 历史记录
   - 缓存

4. 数据工程
   - ETL 中间处理（DuckDB）
   - 数据验证
   - 格式转换

5. 边缘计算
   - IoT 设备本地存储
   - 数据预聚合
```

## 选型指南

```text
选 SQLite 如果：
✓ 需要本地事务存储
✓ App 或桌面应用的数据层
✓ 读多写少
✓ 需要极致稳定性

选 DuckDB 如果：
✓ 需要本地数据分析
✓ 处理 CSV/Parquet 文件
✓ 需要复杂 SQL（窗口函数等）
✓ 数据科学/数据工程场景

两者结合：
SQLite 存储业务数据
DuckDB 分析 SQLite 数据或外部文件
```

## 核心认知

> **嵌入式数据库的本质**：把数据库能力变成一个库，消除部署和运维的复杂度。

**嵌入式数据库的哲学**：

```text
"最好的服务器是不需要服务器"

传统思维：
  数据库 = 需要专人运维的基础设施

嵌入式思维：
  数据库 = 应用程序的一部分
  部署应用 = 部署数据库
  备份文件 = 备份数据库

这种简单性在很多场景下是无价的
```

嵌入式数据库代表了"简单"这个方向的极致。但当数据量超过单机极限时，我们必须走向另一个方向——**分布式**。

NoSQL 解决了"水平扩展"问题，但它放弃了 SQL 和事务。有没有可能两全其美？2012 年，Google 发布了 Spanner 论文，证明了这是可能的。

下一篇，我们来看：**分布式数据库——当单机不够用**。

---

*上一篇：[NoSQL 运动——打破关系的枷锁](/posts/db-evolution-2-nosql/)*

*下一篇：[分布式数据库——当单机不够用](/posts/db-evolution-4-distributed/)*

*本系列：*
1. *[关系型数据库——SQL 一统江湖的时代](/posts/db-evolution-1-relational/)*
2. *[NoSQL 运动——打破关系的枷锁](/posts/db-evolution-2-nosql/)*
3. *嵌入式数据库——藏在你身边的数据库（本篇）*
4. *[分布式数据库——当单机不够用](/posts/db-evolution-4-distributed/)*
5. *[分析型数据库——当查询变成分析](/posts/db-evolution-5-olap-search/)*
6. *[时序数据库——时间维度的专家](/posts/db-evolution-6-timeseries/)*
7. *[向量数据库——AI 时代的新基建](/posts/db-evolution-7-vector/)*
8. *[PostgreSQL——一个数据库的文艺复兴](/posts/db-evolution-8-postgresql/)*
