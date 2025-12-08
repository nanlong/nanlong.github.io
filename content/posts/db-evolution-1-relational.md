---
title: '数据库演进史（一）：关系型数据库——SQL 一统江湖的时代'
date: 2025-12-09T10:00:00+08:00
draft: false
tags: ["数据库", "PostgreSQL", "MySQL", "SQL", "关系型数据库"]
categories: ["技术"]
description: "1970 年，一篇论文改变了整个软件行业。50 年后，我们还在用它定义的模型存储数据。"
series: ["数据库演进史"]
---

1970 年，IBM 研究员 Edgar F. Codd 发表了一篇论文：《A Relational Model of Data for Large Shared Data Banks》。

这篇论文没有写任何代码，却定义了接下来 50 年数据库的基本形态。

**为什么一个 50 年前的模型，至今仍是主流？**

<!--more-->

## 混乱的前关系时代

在 Codd 之前，程序员是怎么存数据的？

```text
层次模型（Hierarchical Model）：

                    公司
                   /    \
               部门A    部门B
              /    \       \
          员工1   员工2    员工3

问题：员工能属于多个部门吗？
答案：不能，或者复制一份数据。
```

层次模型就像一棵树，数据只能有一个"父节点"。想表达"多对多"关系？抱歉，做不到。

网状模型稍微好一点，允许多对多，但代价是——**程序员必须知道数据的物理存储方式**。

```text
网状模型查询伪代码：

FIND FIRST 员工
WHILE NOT END
    IF 员工.部门指针 = 目标部门
        PRINT 员工.姓名
    FIND NEXT 员工 VIA 部门链表
```

注意这段代码：`VIA 部门链表`。程序员必须知道数据是用链表存的，必须手动遍历指针。

**如果有一天 DBA 决定换一种存储结构？所有代码重写。**

## Codd 的革命：数据独立性

Codd 的核心洞察：**数据的逻辑结构应该与物理存储分离**。

```text
Codd 的关系模型：

┌─────────────────────────────────────────┐
│  逻辑层：表、行、列                       │
│  SELECT * FROM employees WHERE dept='A' │
├─────────────────────────────────────────┤
│  物理层：B-Tree？哈希？堆？               │
│  程序员不需要知道！                       │
└─────────────────────────────────────────┘
```

这就像是——

> 你去图书馆借书，只需要说"我要《百年孤独》"。
>
> 你不需要知道这本书在第三排书架第五层，不需要知道图书馆用什么编码系统。
>
> 图书馆换了一套分类方式？没关系，你的借书方式不变。

**这就是"数据独立性"**——应用程序与数据存储解耦。

## 关系模型的核心：表、行、关系

关系模型用三个简单概念建模整个世界：

```text
表（Relation）：一类事物的集合
行（Tuple）：一个具体的事物
列（Attribute）：事物的属性

┌─────────────────────────────────────┐
│              employees               │
├────────┬──────────┬─────────────────┤
│   id   │   name   │   department    │
├────────┼──────────┼─────────────────┤
│   1    │  张三    │   工程部         │
│   2    │  李四    │   产品部         │
│   3    │  王五    │   工程部         │
└────────┴──────────┴─────────────────┘
```

多对多关系？用**关联表**：

```text
员工 ←→ 项目（多对多）

┌──────────────────┐     ┌──────────────────┐
│    employees     │     │     projects     │
├────────┬─────────┤     ├────────┬─────────┤
│   id   │  name   │     │   id   │  name   │
├────────┼─────────┤     ├────────┼─────────┤
│   1    │  张三   │     │   1    │ 项目A   │
│   2    │  李四   │     │   2    │ 项目B   │
└────────┴─────────┘     └────────┴─────────┘

            ┌──────────────────────┐
            │  employee_projects   │
            ├─────────────┬────────┤
            │ employee_id │ proj_id│
            ├─────────────┼────────┤
            │      1      │   1    │  张三参与项目A
            │      1      │   2    │  张三参与项目B
            │      2      │   1    │  李四参与项目A
            └─────────────┴────────┘
```

这种设计看起来"多了一张表"，但换来的是**无限的灵活性**——任何关系都能表达。

## SQL：数据库的通用语言

有了关系模型，还需要一种方式来操作它。这就是 SQL（Structured Query Language）。

```sql
-- 不需要知道数据怎么存储
-- 只需要描述"我要什么"
SELECT e.name, p.name as project
FROM employees e
JOIN employee_projects ep ON e.id = ep.employee_id
JOIN projects p ON ep.proj_id = p.id
WHERE e.department = '工程部';
```

SQL 是**声明式**的：你说"要什么"，数据库决定"怎么拿"。

这与层次/网状模型的**命令式**形成鲜明对比：

```text
命令式（网状模型）：
"从员工表开始，遍历部门指针，找到工程部，
 然后遍历项目链表，收集所有项目名..."

声明式（SQL）：
"给我工程部员工参与的所有项目"
```

**数据库优化器会自动选择最优执行计划**。表从 100 行变成 1 亿行？SQL 语句不用改，优化器会自动调整策略。

## ACID：关系型数据库的护城河

关系型数据库不只是"用表存数据"，它的杀手锏是**事务保证**：

```text
A - Atomicity（原子性）
    转账：扣款 + 入账，要么都成功，要么都失败
    不会出现"钱扣了但没到账"

C - Consistency（一致性）
    数据永远满足约束
    余额不能为负、外键必须存在...

I - Isolation（隔离性）
    并发事务互不干扰
    你读数据时，别人的修改不会影响你

D - Durability（持久性）
    一旦提交，断电也不丢
    写入磁盘/WAL 日志
```

**这四个字母，是 50 年来无数系统的基石**。

银行转账、电商下单、机票预订——任何涉及"钱"或"库存"的场景，ACID 都是刚需。

## MySQL vs PostgreSQL：两条路线

关系型数据库有很多实现，但两个名字最响亮：MySQL 和 PostgreSQL。

它们代表了两种截然不同的哲学：

```text
MySQL（1995）：
┌─────────────────────────────────────────┐
│  "够用就好"的实用主义                     │
├─────────────────────────────────────────┤
│  ✓ 简单、轻量、快速上手                   │
│  ✓ LAMP 栈标配（Linux+Apache+MySQL+PHP） │
│  ✓ 早期互联网的默认选择                   │
│  ✗ 功能相对简单                          │
│  ✗ 早期不支持事务（MyISAM 引擎）          │
└─────────────────────────────────────────┘

PostgreSQL（1996）：
┌─────────────────────────────────────────┐
│  "正确性优先"的学院派                     │
├─────────────────────────────────────────┤
│  ✓ 功能最完整的开源数据库                 │
│  ✓ 严格遵循 SQL 标准                     │
│  ✓ 强大的扩展系统                        │
│  ✗ 早期"重"，配置复杂                    │
│  ✗ 学习曲线陡峭                          │
└─────────────────────────────────────────┘
```

**MySQL 像是本田思域**：省油、好开、满街都是、修车便宜。

**PostgreSQL 像是沃尔沃**：安全、可靠、功能全，但你得花时间了解它。

2010 年之前，MySQL 因为"简单"赢了。

2010 年之后，PostgreSQL 因为"强大"逆袭了。

为什么？我们稍后会看到。

## Rust + PostgreSQL 实战

让我们用 Rust 和 sqlx 来体验关系型数据库的魅力。

首先是表结构：

```sql
-- 先创建部门表（被引用的表）
CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    budget DECIMAL(15, 2) NOT NULL
);

-- 再创建员工表（引用部门表）
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    department_id INTEGER REFERENCES departments(id),
    salary DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 创建索引
CREATE INDEX idx_employees_department ON employees(department_id);
```

然后是 Rust 代码：

```rust
use sqlx::postgres::PgPoolOptions;
use sqlx::FromRow;
use rust_decimal::Decimal;
use chrono::{DateTime, Utc};

#[derive(Debug, FromRow)]
struct Employee {
    id: i32,
    name: String,
    email: String,
    department_id: Option<i32>,
    salary: Decimal,
    created_at: DateTime<Utc>,
}

async fn demo() -> Result<(), sqlx::Error> {
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect("postgres://user:pass@localhost/mydb")
        .await?;

    // 事务示例：给员工调薪（同时更新部门预算）
    let mut tx = pool.begin().await?;

    let raise = Decimal::new(5000, 0);

    sqlx::query!(
        "UPDATE employees SET salary = salary + $1 WHERE id = $2",
        raise,
        1
    )
    .execute(&mut *tx)
    .await?;

    sqlx::query!(
        "UPDATE departments SET budget = budget - $1 WHERE id = $2",
        raise,
        1
    )
    .execute(&mut *tx)
    .await?;

    tx.commit().await?;

    Ok(())
}
```

注意 `pool.begin()` 和 `tx.commit()`——这就是 ACID 的 A（原子性）。如果中间任何一步失败，整个事务回滚，数据保持一致。

## 查询优化：EXPLAIN 的艺术

关系型数据库的另一个优势是**成熟的查询优化器**。

```sql
EXPLAIN ANALYZE
SELECT e.name, d.name as department
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE d.name = '工程部';
```

输出可能是：

```text
Hash Join  (cost=1.04..2.09 rows=1 width=64) (actual time=0.025..0.027 rows=3 loops=1)
  Hash Cond: (e.department_id = d.id)
  ->  Seq Scan on employees e  (cost=0.00..1.03 rows=3 width=36)
  ->  Hash  (cost=1.03..1.03 rows=1 width=36)
        ->  Seq Scan on departments d  (cost=0.00..1.03 rows=1 width=36)
              Filter: (name = '工程部')
Planning Time: 0.089 ms
Execution Time: 0.048 ms
```

这告诉你：
- 数据库选择了 **Hash Join**
- 先扫描 departments 表（因为有过滤条件）
- 建立哈希表
- 然后扫描 employees 表进行匹配

**当数据量变大，你可能需要**：

```sql
-- 添加索引
CREATE INDEX idx_departments_name ON departments(name);

-- 再次 EXPLAIN，你会看到
Index Scan using idx_departments_name on departments d
```

查询时间从 O(n) 变成 O(log n)。

## 关系型的天花板

故事讲到这里，关系型数据库简直完美。

**但 2000 年代后期，麻烦来了。**

```text
2007 年的 Facebook：
- 5000 万用户
- 每天 10 亿 SQL 查询
- MySQL 单机扛不住了

怎么办？
```

传统方案：**买更贵的服务器**（垂直扩展）。

```text
服务器配置演进：
4 核 → 8 核 → 16 核 → 32 核 → ...
16GB → 64GB → 256GB → 1TB → ...

问题：
1. 成本指数级增长
2. 总有天花板（单机极限）
3. 单点故障
```

另一个方案：**加更多服务器**（水平扩展）。

但关系型数据库天生不适合水平扩展：

```text
问题 1：分布式事务

用户 A 在服务器 1
用户 B 在服务器 2

A 给 B 转账，怎么保证 ACID？
两台机器要协调——复杂、慢、容易出错

问题 2：JOIN 跨服务器

SELECT * FROM orders o
JOIN users u ON o.user_id = u.id

如果 orders 在服务器 1，users 在服务器 2
这个 JOIN 怎么执行？

问题 3：Schema 变更

ALTER TABLE users ADD COLUMN avatar VARCHAR(255);

1 亿行数据，这条语句可能锁表几小时
互联网公司无法接受
```

**关系型数据库为一致性（Consistency）优化，牺牲了可扩展性（Scalability）。**

当数据量和并发量超过单机极限，你要么接受"一致性降级"，要么寻找新方案。

2009 年，一场名为"NoSQL"的运动开始了。

## 核心认知

> **关系型数据库的本质**：用严格的结构（表、约束、事务）换取数据的可靠性和一致性。

**50 年屹立不倒的原因**：
- 数据独立性——应用与存储解耦
- SQL 标准——一次学习，到处使用
- ACID 事务——金融级可靠性
- 成熟生态——工具、人才、最佳实践

**适用场景**：
- 数据一致性要求高（金融、电商、ERP）
- 关系复杂（需要 JOIN）
- 数据量在单机可承受范围（< 1TB）
- 团队熟悉 SQL

**不适用场景**：
- 超大规模（> 10 亿行）
- Schema 频繁变化
- 需要极致水平扩展
- 非结构化数据为主

## 引出下一篇

关系型数据库的问题在于：**它为"正确"优化，而不是为"规模"优化**。

2009 年，当 Facebook、Google、Amazon 的数据量超过任何单机能承受的极限时，一批工程师开始思考：

> "我们真的需要 ACID 吗？"
> "我们真的需要 Schema 吗？"
> "我们真的需要 SQL 吗？"

他们的答案是：**不一定**。

于是 MongoDB、Cassandra、Redis 横空出世。

这就是下一篇要讲的：**NoSQL 运动——打破关系的枷锁**。

---

## 常见问题

### Q：PostgreSQL 和 MySQL 选哪个？

**A：2025 年，优先选 PostgreSQL。**

| 维度 | MySQL | PostgreSQL |
|------|-------|------------|
| 功能完整性 | 基础 | 最完整 |
| JSON 支持 | 有 | JSONB 更强 |
| 扩展生态 | 少 | 极其丰富 |
| 地理信息 | 需插件 | PostGIS |
| 全文搜索 | 基础 | 内置强大 |
| 学习资料 | 多 | 也很多了 |

除非你在维护遗留 MySQL 系统，否则新项目推荐 PostgreSQL。

### Q：单表多少行需要考虑分库分表？

**A：没有绝对标准，但有经验值。**

```text
PostgreSQL 单表：
- < 1000 万行：随便玩
- 1000 万 - 1 亿行：需要关注索引和查询优化
- 1 - 10 亿行：考虑分区表
- > 10 亿行：考虑分库分表或换方案
```

**关键不是行数，而是**：
- 查询延迟是否可接受
- 单机 IO/CPU 是否打满
- 备份恢复时间是否可接受

### Q：ORM 还是原生 SQL？

**A：看团队和场景。**

```text
ORM（如 Diesel、SeaORM）：
- 类型安全
- 防 SQL 注入
- 但复杂查询写起来别扭

原生 SQL + 类型检查（如 sqlx）：
- 灵活
- 性能好
- sqlx 编译时检查 SQL 正确性

推荐：sqlx（兼顾灵活和安全）
```

---

*下一篇：[NoSQL 运动——打破关系的枷锁](/posts/db-evolution-2-nosql/)*

*本系列：*
1. *关系型数据库——SQL 一统江湖的时代（本篇）*
2. *[NoSQL 运动——打破关系的枷锁](/posts/db-evolution-2-nosql/)*
3. *[嵌入式数据库——藏在你身边的数据库](/posts/db-evolution-3-embedded/)*
4. *[分布式数据库——当单机不够用](/posts/db-evolution-4-distributed/)*
5. *[分析型数据库——当查询变成分析](/posts/db-evolution-5-olap-search/)*
6. *[时序数据库——时间维度的专家](/posts/db-evolution-6-timeseries/)*
7. *[向量数据库——AI 时代的新基建](/posts/db-evolution-7-vector/)*
8. *[PostgreSQL——一个数据库的文艺复兴](/posts/db-evolution-8-postgresql/)*
