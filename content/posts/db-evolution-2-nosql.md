---
title: '数据库演进史（二）：NoSQL 运动——打破关系的枷锁'
date: 2025-12-09T10:10:00+08:00
draft: false
tags: ["数据库", "NoSQL", "MongoDB", "Redis", "Cassandra"]
categories: ["技术"]
description: "2009 年，程序员们开始质疑：我们真的需要 Schema 吗？我们真的需要事务吗？一场数据库革命开始了。"
series: ["数据库演进史"]
---

## 前情回顾

上一篇我们看到了关系型数据库的辉煌与局限：

- **优点**：ACID 事务、SQL 标准、数据一致性
- **问题**：单机天花板、水平扩展困难、Schema 僵化

当 Facebook 有 5 亿用户、Google 每天处理 200 亿网页时，他们面临一个选择：

**要么花天价买超级计算机，要么——重新发明数据库。**

他们选择了后者。

<!--more-->

## Google 的三篇论文

2003-2006 年，Google 发表了三篇改变世界的论文：

```text
2003: The Google File System (GFS)
      → 分布式文件系统，数据切成块，分散存储

2004: MapReduce
      → 分布式计算框架，把计算拆分到数据所在的机器

2006: Bigtable
      → 分布式数据库，稀疏、多维、有序的 Map
```

这三篇论文的共同主题是：**用一堆便宜的机器，代替一台昂贵的超级计算机**。

```text
传统思路：
买更好的服务器 → 垂直扩展 → 总有天花板

Google 思路：
买更多便宜服务器 → 水平扩展 → 理论上无限
```

Bigtable 论文的一段话点燃了 NoSQL 运动：

> "Bigtable 不支持完整的关系数据模型。"

翻译成人话：**我们放弃了 SQL、放弃了事务、放弃了 Schema——然后我们赢了。**

## CAP：一个被误解的定理

NoSQL 运动中，有一个定理被反复提及：**CAP 定理**。

```text
Eric Brewer（2000）：

分布式系统不可能同时满足以下三点：

C - Consistency（一致性）
    所有节点同一时刻看到相同数据

A - Availability（可用性）
    每个请求都能得到响应（不是错误）

P - Partition tolerance（分区容错）
    网络分区时系统继续工作
```

**CAP 说的是：三选二，你只能放弃一个。**

这个定理被 NoSQL 阵营用来论证：

> "关系型数据库选择了 CA，所以不能水平扩展。"
> "我们选择 AP 或 CP，所以能无限扩展。"

**但这是对 CAP 的误读。**

```text
误解：你必须永久放弃一个

事实：
1. 网络分区（P）在真实世界必然发生
2. 所以你其实是在 C 和 A 之间选择
3. 而且是在"分区发生时"选择，不是永久放弃
4. 分区恢复后，可以重新同步

更实用的问题：
"当网络抖动时，你要返回旧数据还是返回错误？"
```

但无论如何，CAP 成了 NoSQL 的理论武器。

## 四大 NoSQL 流派

NoSQL（Not Only SQL）不是一个产品，而是一场运动。它分化出四个主要流派：

```text
┌─────────────────────────────────────────────────────────────────┐
│                     NoSQL 四大流派                               │
├───────────────┬──────────────┬──────────────────────────────────┤
│     类型       │     代表      │            适用场景              │
├───────────────┼──────────────┼──────────────────────────────────┤
│   键值存储     │    Redis     │  缓存、会话、计数器、排行榜       │
│   文档数据库   │   MongoDB    │  灵活 Schema、内容管理、目录      │
│   列族存储     │  Cassandra   │  时序数据、日志、大规模写入       │
│   图数据库     │    Neo4j     │  社交网络、推荐系统、知识图谱     │
└───────────────┴──────────────┴──────────────────────────────────┘
```

让我们逐一了解。

## Redis：不只是缓存

很多人以为 Redis 只是"缓存"，这低估了它。

**Redis 是一个"数据结构服务器"**。

```text
Redis 支持的数据结构：

String  → "hello"
List    → [a, b, c, d]
Set     → {a, b, c}（无序去重）
Hash    → {name: "张三", age: 25}
ZSet    → {(a, 1.0), (b, 2.0)}（带分数的有序集合）
Stream  → 消息队列
```

每种数据结构都有对应的原子操作：

```bash
# 计数器（原子自增）
INCR page_views

# 分布式锁
SET lock:order:123 owner_id NX EX 30

# 排行榜（有序集合）
ZADD leaderboard 100 "player1"
ZADD leaderboard 200 "player2"
ZREVRANGE leaderboard 0 9 WITHSCORES

# 限流（滑动窗口）
ZADD rate_limit:user:1 1704067200 "req1"
ZREMRANGEBYSCORE rate_limit:user:1 0 1704063600
ZCARD rate_limit:user:1
```

**Rust 中使用 Redis**：

```rust
use redis::AsyncCommands;

async fn demo() -> redis::RedisResult<()> {
    let client = redis::Client::open("redis://127.0.0.1/")?;
    let mut con = client.get_multiplexed_async_connection().await?;

    // 缓存用户信息（带过期时间）
    con.set_ex("user:1:profile", r#"{"name":"张三"}"#, 3600).await?;

    // 原子自增
    let views: i64 = con.incr("article:1:views", 1).await?;

    // 排行榜
    con.zadd("leaderboard", "player1", 100.0).await?;
    let top10: Vec<(String, f64)> = con.zrevrange_withscores("leaderboard", 0, 9).await?;

    Ok(())
}
```

**Redis 的定位**：

```text
Redis 是什么：
- 内存数据库（速度极快）
- 数据结构服务器（不只是 KV）
- 单线程命令执行（无锁，简单可靠；6.0+ 支持多线程 IO）

Redis 不是什么：
- 不是持久化数据库的替代品（RDB/AOF 有丢数据风险）
- 不是大数据存储（内存有限）
- 不是关系型数据库（没有 JOIN、没有 ACID 事务*）

*Redis 有 MULTI/EXEC 事务，但不支持回滚，与传统 ACID 事务不同
```

## MongoDB：文档模型的自由与代价

MongoDB 是 NoSQL 运动中最"出圈"的产品。

它的核心卖点是：**不需要 Schema**。

```text
关系型数据库：
1. 先定义表结构（CREATE TABLE）
2. 数据必须符合结构
3. 修改结构需要 ALTER TABLE（痛苦）

MongoDB：
1. 直接插入 JSON
2. 每条数据结构可以不同
3. 加字段？直接加，不用改表
```

这太自由了！

```javascript
// MongoDB 插入数据
db.users.insertOne({
    name: "张三",
    age: 25,
    hobbies: ["篮球", "编程"],  // 数组
    address: {                   // 嵌套对象
        city: "北京",
        street: "朝阳路"
    }
});

// 每个文档结构可以不同
db.users.insertOne({
    name: "李四",
    age: 30,
    company: "某科技公司",      // 张三没有这个字段
    skills: {                   // 完全不同的嵌套结构
        programming: ["Rust", "Go"],
        languages: ["中文", "英语"]
    }
});
```

**但自由是有代价的。**

### MongoDB 的血泪教训

**教训一：没有 Schema 不代表不需要设计**

```text
场景：用户系统

第一版：
{name: "张三", email: "zhang@example.com"}

第二版（新需求）：
{name: "张三", email: ["zhang@example.com", "zs@work.com"]}

问题：email 从字符串变成了数组
旧数据怎么办？代码怎么兼容？

关系型数据库会强迫你在 ALTER TABLE 时想清楚
MongoDB 让你"先上线再说"——然后在凌晨 3 点修 bug
```

**教训二：嵌套 vs 引用的艰难选择**

```text
订单系统设计：

方案 A：嵌套（反范式）
{
    order_id: "123",
    user: {id: 1, name: "张三", address: "北京"},
    items: [{product: "手机", price: 5999}]
}

优点：一次查询拿到所有数据
缺点：用户改地址，所有历史订单要不要改？

方案 B：引用（类似关系型）
{
    order_id: "123",
    user_id: 1,
    item_ids: ["item_1", "item_2"]
}

优点：数据不冗余
缺点：需要多次查询（MongoDB 没有 JOIN*）

*MongoDB 3.2+ 有 $lookup，但性能远不如关系型 JOIN
```

**教训三：最终一致性的坑**

早期 MongoDB 默认配置下：

```text
1. 写入主节点，立即返回成功
2. 异步复制到从节点
3. 主节点挂了？可能丢数据！

真实案例：
某电商大促，订单写入"成功"
主节点宕机，从节点还没同步
恢复后：订单没了，但用户已付款
```

现在 MongoDB 可以配置 `writeConcern: "majority"`，但这会降低性能。

### PostgreSQL 的回应：JSONB

有趣的是，PostgreSQL 用另一种方式解决了"灵活 Schema"问题：

```sql
-- PostgreSQL：关系型 + 文档型
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,  -- 结构化字段
    profile JSONB                         -- 灵活字段
);

-- 插入数据
INSERT INTO users (email, profile) VALUES (
    'zhang@example.com',
    '{"name": "张三", "hobbies": ["篮球", "编程"], "address": {"city": "北京"}}'
);

-- 查询 JSON 字段
SELECT * FROM users WHERE profile->>'name' = '张三';

-- 查询嵌套字段
SELECT * FROM users WHERE profile->'address'->>'city' = '北京';

-- 创建 JSON 索引
CREATE INDEX idx_users_name ON users ((profile->>'name'));

-- 最强大的：GIN 索引支持任意字段查询
CREATE INDEX idx_users_profile ON users USING GIN (profile);
SELECT * FROM users WHERE profile @> '{"hobbies": ["篮球"]}';
```

**PostgreSQL JSONB 的优势**：

| 特性 | MongoDB | PostgreSQL JSONB |
|------|---------|------------------|
| 灵活 Schema | ✓ | ✓ |
| ACID 事务 | 有限支持 | 完整支持 |
| JOIN | $lookup（慢） | 原生 JOIN |
| 索引 | 有 | GIN 索引更强 |
| 学习成本 | 新语法 | 还是 SQL |

**什么时候选 MongoDB**：
- 数据 99% 是文档，几乎不需要 JOIN
- 团队已熟悉 MongoDB
- 需要地理位置查询（MongoDB 原生支持好）

**什么时候选 PostgreSQL JSONB**：
- 结构化数据 + 少量灵活字段
- 需要事务
- 需要 JOIN
- 已有 PostgreSQL 基础设施

## Cassandra：为写入而生

如果说 Redis 是"快"，MongoDB 是"灵活"，那 Cassandra 是"大"。

```text
Cassandra 的设计目标：

- 写入吞吐：每秒百万级
- 数据规模：PB 级
- 节点数量：成百上千
- 可用性：永不停机（无单点故障）
```

Cassandra 的架构很特别：

```text
传统主从架构：
┌────────┐
│ Master │ ← 所有写入
└────┬───┘
     │ 复制
┌────┴───┐
│ Slave  │ ← 只能读
└────────┘
问题：Master 是瓶颈和单点故障

Cassandra 环形架构：
      ┌─────┐
   ┌──┤ N1  ├──┐
   │  └─────┘  │
┌──┴──┐     ┌──┴──┐
│ N4  │     │ N2  │
└──┬──┘     └──┬──┘
   │  ┌─────┐  │
   └──┤ N3  ├──┘
      └─────┘
特点：
- 无主节点，任意节点可读写
- 数据按哈希分布到节点
- 自动复制（默认 3 份）
```

**Cassandra 的适用场景**：

```text
适合：
- 时序数据（日志、监控、IoT）
- 写多读少
- 简单查询（主键查询）
- 需要极高可用

不适合：
- 复杂查询（没有 JOIN）
- 需要事务
- 读多写少
- 数据量不大（杀鸡用牛刀）
```

## NoSQL 的反思

2009-2015 年，NoSQL 风头无两。

但随着时间推移，大家开始反思：

```text
我们放弃了什么？

1. 事务
   "最终一致性"听起来很酷
   直到你的订单系统出现超卖
   直到你的支付系统出现重复扣款

2. Schema
   "无 Schema"意味着自由
   也意味着数据质量失控
   也意味着代码里到处是防御性检查

3. SQL
   新语法学习成本
   每个数据库 API 都不一样
   复杂查询很难写

4. 生态
   成熟的工具链
   熟练的 DBA
   可靠的备份恢复
```

**"NoSQL"悄悄变成了"Not Only SQL"。**

MongoDB 加上了事务（4.0+），加上了 Schema 验证。

Cassandra 有了 CQL（长得像 SQL）。

大家开始承认：**SQL 是个好东西，ACID 也是个好东西**。

## 核心认知

> **NoSQL 的本质**：用一致性换可扩展性，用灵活性换规范性。

**NoSQL 教会我们**：
- 没有银弹，只有权衡
- CAP 是选择题，不是判断题
- Schema 不是枷锁，是保护
- 最终一致性的"最终"可能很久

**选型建议**：

| 需求 | 推荐方案 |
|------|----------|
| 缓存、会话、计数器 | Redis |
| 灵活 Schema + 事务 | PostgreSQL JSONB |
| 纯文档、海量数据、无 JOIN | MongoDB |
| 时序日志、超大规模写入 | Cassandra |
| 社交关系、知识图谱 | Neo4j |
| 不确定 | 先用 PostgreSQL |

## 引出下一篇

到目前为止，我们讨论的数据库都有一个共同点：**它们都是独立运行的服务**。

你需要安装、配置、维护一个数据库进程。哪怕只是 Redis，也需要一个 redis-server。

但有一类数据库完全不同：**它不需要服务器，直接嵌入你的应用程序**。

你可能每天都在用它，却从未注意过——打开 Chrome，它用 SQLite 存储你的历史记录；发送微信，它用 SQLite 存储聊天记录；甚至你的 iPhone，每一个 App 都可能在用 SQLite。

下一篇，我们来看：**嵌入式数据库——藏在你身边的数据库**。

---

## 常见问题

### Q：Redis 和 Memcached 选哪个？

**A：2025 年，选 Redis。**

```text
Redis 优势：
- 数据结构丰富（不只是 KV）
- 持久化选项
- 主从复制、集群
- 活跃的社区和生态

Memcached 仅在以下场景有优势：
- 纯粹的 KV 缓存
- 已有大量 Memcached 基础设施
```

### Q：MongoDB 是不是已经过时了？

**A：没有，但它的定位变了。**

```text
2010 年的 MongoDB：
"我们不需要 Schema！不需要事务！"

2025 年的 MongoDB：
"我们有 Schema 验证！有完整事务！"
"但我们的文档模型和地理查询确实很强"

MongoDB 仍然适合：
- 内容管理系统
- 产品目录
- 游戏存档
- 地理位置应用

但对于大多数 CRUD 应用，PostgreSQL 更稳妥。
```

### Q：我应该用 Redis 做消息队列吗？

**A：简单场景可以，复杂场景用专业的。**

```text
Redis Streams 适合：
- 简单的任务队列
- 延迟要求低
- 消息丢失可接受

不适合：
- 金融级可靠性要求
- 复杂的消费者组
- 消息需要持久化保证

复杂场景用：Kafka、RabbitMQ、Pulsar
```

---

*上一篇：[关系型数据库——SQL 一统江湖的时代](/posts/db-evolution-1-relational/)*

*下一篇：[嵌入式数据库——藏在你身边的数据库](/posts/db-evolution-3-embedded/)*

*本系列：*
1. *[关系型数据库——SQL 一统江湖的时代](/posts/db-evolution-1-relational/)*
2. *NoSQL 运动——打破关系的枷锁（本篇）*
3. *[嵌入式数据库——藏在你身边的数据库](/posts/db-evolution-3-embedded/)*
4. *[分布式数据库——当单机不够用](/posts/db-evolution-4-distributed/)*
5. *[分析型数据库——当查询变成分析](/posts/db-evolution-5-olap-search/)*
6. *[时序数据库——时间维度的专家](/posts/db-evolution-6-timeseries/)*
7. *[向量数据库——AI 时代的新基建](/posts/db-evolution-7-vector/)*
8. *[PostgreSQL——一个数据库的文艺复兴](/posts/db-evolution-8-postgresql/)*
