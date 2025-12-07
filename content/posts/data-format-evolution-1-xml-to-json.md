---
title: "数据格式演进史（一）：从 XML 到 JSON，复杂之死"
date: 2025-12-07T10:00:00+08:00
draft: false
tags: ["数据格式", "XML", "JSON", "REST", "SOAP", "API"]
categories: ["技术"]
description: "2005 年，一个叫 Jesse James Garrett 的人写了一篇文章，彻底改变了 Web 开发的走向。XML 统治了企业级软件十年，却在短短几年内被一个「玩具般简单」的格式取代。"
series: ["数据格式演进史"]
---

2005 年，一个叫 Jesse James Garrett 的人写了一篇文章，彻底改变了 Web 开发的走向。XML 统治了企业级软件十年，却在短短几年内被一个「玩具般简单」的格式取代。

<!--more-->

## 史前时代：程序之间怎么说话？

在互联网的早期，如果两个程序想交换数据，它们面临一个根本性的问题：**数据长什么样？**

想象两个人用电报通信：

> 张三发：`王五|30|北京|工程师`
>
> 李四收到后困惑：这是「姓名|年龄|城市|职业」还是「姓名|工号|部门|职位」？

没有标准，就没有沟通。

早期的解决方案五花八门：

| 方案 | 问题 |
|------|------|
| CSV | 字段含义靠猜，嵌套结构无法表达 |
| 固定宽度文本 | 每个字段固定 N 个字符，浪费空间，难以扩展 |
| 自定义二进制 | 每个系统都不一样，互不兼容 |
| EDI | 金融/医疗专用，复杂得令人发指 |

人们需要一种**通用的、自描述的、可扩展的**数据格式。

## 1998：XML 的野望

XML（eXtensible Markup Language）带着雄心壮志登场。它的设计哲学是：

> **数据应该自我描述。任何人拿到数据，不需要额外文档就能理解它的结构。**

看看 XML 如何表达一个用户：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<user>
    <name>张三</name>
    <age>30</age>
    <address>
        <city>北京</city>
        <district>朝阳区</district>
    </address>
    <skills>
        <skill>Java</skill>
        <skill>Python</skill>
    </skills>
</user>
```

**优点显而易见**：
- ✅ 自描述：每个字段都有名字
- ✅ 可嵌套：地址可以包含子字段
- ✅ 可扩展：随时添加新字段
- ✅ 人类可读：打开就能看懂

XML 迅速成为企业级软件的标准。

## SOAP：XML 的巅峰与膨胀

有了 XML，人们开始构建基于它的整套生态系统。其中最重要的就是 **SOAP**（Simple Object Access Protocol）。

**讽刺的是，SOAP 里唯一不存在的就是「Simple」。**

看看一个 SOAP 请求长什么样：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
    xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:user="http://example.com/user">
    <soap:Header>
        <user:AuthToken>abc123</user:AuthToken>
    </soap:Header>
    <soap:Body>
        <user:GetUserRequest>
            <user:UserId>12345</user:UserId>
        </user:GetUserRequest>
    </soap:Body>
</soap:Envelope>
```

只是想获取一个用户，需要写这么多代码。

但这还不是全部。SOAP 生态还包括：

| 组件 | 作用 | 复杂度 |
|------|------|--------|
| WSDL | 描述服务接口 | 一个服务可能有上千行 WSDL |
| XSD | 定义数据类型 | 类型系统比编程语言还复杂 |
| WS-Security | 安全规范 | 单独学习需要几周 |
| WS-ReliableMessaging | 可靠传输 | 又一套复杂协议 |
| WS-Transaction | 分布式事务 | 复杂到很少有人敢用 |

**企业架构师们爱死了这套东西**——它「完整」、「规范」、「可验证」。

**开发者们恨透了这套东西**——写一个简单的 API 调用，要生成几百行代码。

## 一个真实的对比

假设我们要实现一个简单的功能：**获取用户信息**。

### SOAP 方案

**第一步**：写 WSDL 文件（约 100 行）
**第二步**：写 XSD 定义数据类型（约 50 行）
**第三步**：用工具生成客户端代码（约 500 行）
**第四步**：写业务逻辑（约 20 行）

**总计**：670+ 行代码，才能发出一个请求。

### REST + JSON 方案

```javascript
fetch('/api/users/12345')
  .then(res => res.json())
  .then(user => console.log(user.name));
```

**总计**：3 行。

**这就是为什么 JSON 能赢。**

## 2001：JSON 的诞生

JSON（JavaScript Object Notation）的发明者 Douglas Crockford 说过一句著名的话：

> **「我没有发明 JSON，我只是发现了它。它本来就存在于 JavaScript 中。」**

JSON 就是 JavaScript 的对象字面量语法：

```json
{
    "name": "张三",
    "age": 30,
    "address": {
        "city": "北京",
        "district": "朝阳区"
    },
    "skills": ["Java", "Python"]
}
```

把同样的数据用 XML 和 JSON 表示，对比一下：

| 对比项 | XML | JSON |
|--------|-----|------|
| 字符数 | 312 | 132 |
| 冗余度 | 高（结束标签重复） | 低 |
| 解析速度 | 慢 | 快 |
| 人类可读性 | 好 | 更好 |
| 嵌套支持 | 是 | 是 |
| 类型系统 | 复杂（XSD） | 简单（6种类型） |

## 2005：AJAX 革命

转折点发生在 2005 年。

Jesse James Garrett 发表了一篇文章《Ajax: A New Approach to Web Applications》，描述了 Google 在 Gmail 和 Google Maps 中使用的技术：

> **网页可以在不刷新的情况下，与服务器交换数据。**

这就是 AJAX（Asynchronous JavaScript and XML）。虽然名字里有 XML，但开发者们很快发现：**用 JSON 更爽**。

**为什么 JSON 在 AJAX 场景下完胜？**

浏览器里的 JavaScript 引擎天然支持 JSON：

```javascript
// 解析 JSON：一行代码
const user = JSON.parse(responseText);

// 解析 XML：十几行代码
const parser = new DOMParser();
const xmlDoc = parser.parseFromString(responseText, "text/xml");
const name = xmlDoc.getElementsByTagName("name")[0].textContent;
const age = xmlDoc.getElementsByTagName("age")[0].textContent;
// ... 继续写 ...
```

**JSON 是 JavaScript 的原生数据格式，XML 不是。**

## 为什么「简单」打败了「完美」？

这背后有一个深刻的道理：

> **在大多数场景下，开发者的时间比机器的时间更贵。**

### XML/SOAP 的设计假设

- 数据需要严格的类型验证
- 接口需要完整的契约描述
- 企业级应用需要可靠传输、事务支持
- 前期投入多，后期收益大

### 现实情况

- 大多数 API 不需要那么严格的验证
- 接口文档用 Markdown 写写就够了
- 很多应用根本不需要分布式事务
- 互联网时代讲究「快速迭代」，不是「完美设计」

**SOAP 是为「火箭工程师」设计的，JSON 是为「普通程序员」设计的。**

而普通程序员的数量，是火箭工程师的一万倍。

## JSON 的局限性

JSON 不是完美的。它也有明显的缺点：

### 1. 没有注释

```json
{
    "timeout": 30  // 这是超时时间（秒）—— 语法错误！JSON 不支持注释
}
```

Douglas Crockford 故意不加注释，因为他担心有人用注释来传递解析指令。

> **后来的解决方案**：JSON5（2012）允许注释、尾逗号等，常用于配置文件。但在 API 传输中，标准 JSON 仍是主流。

### 2. 没有日期类型

```json
{
    "created_at": "2025-01-01T10:00:00Z"  // 只能用字符串表示
}
```

日期是字符串还是时间戳？ISO 8601 还是 Unix 时间？全靠约定。

### 3. 数字精度问题

```json
{
    "id": 9007199254740993  // JavaScript 会丢失精度！
}
```

JavaScript 的 Number 类型是 64 位浮点数，超过 2^53 的整数会丢失精度。

### 4. 没有 Schema 验证

XML 有 XSD，可以在解析前验证数据结构。JSON 原生不支持，虽然后来有了 JSON Schema，但不如 XSD 普及。

## XML 并未死亡

尽管在 Web API 领域败下阵来，XML 仍然在很多场景活得很好：

| 场景 | 为什么用 XML |
|------|-------------|
| 银行/金融 | 监管要求、历史系统、严格验证 |
| 医疗（HL7） | 行业标准、生命攸关需要严格校验 |
| 文档格式（DOCX/XLSX） | 复杂结构、需要命名空间 |
| 配置文件（Maven/Android） | 工具链成熟、IDE 支持好 |
| SOAP 遗留系统 | 还在运行的老系统 |

**时至今日，仍有相当比例的企业 API 使用 SOAP/XML**——主要在银行、医疗和政府领域。这些系统承载着大量关键业务，迁移成本高昂。

## 安全：XML 的黑历史

XML 有一个臭名昭著的安全漏洞：**XXE（XML External Entity）攻击**。

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<user>
    <name>&xxe;</name>
</user>
```

这段 XML 可以让服务器读取本地文件，甚至发起内网请求！

**受害者名单**：
- 2012：Facebook
- 2014：Google
- 2017：Uber
- ……

JSON 没有这个问题——它太简单了，没有「实体引用」这种特性。

**有时候，简单本身就是安全。**

## 时间线回顾

| 年份 | 事件 |
|------|------|
| 1996 | XML 1.0 草案发布 |
| 1998 | XML-RPC 诞生（XML + RPC 的早期尝试） |
| 2000 | SOAP 1.1 发布，企业级 Web 服务开始流行 |
| 2001 | Douglas Crockford 发现并命名 JSON |
| 2002 | WSDL 1.1 发布，SOAP 生态完整 |
| 2005 | AJAX 革命，JSON 开始流行 |
| 2006 | JSON 被标准化为 RFC 4627 |
| 2010 | REST + JSON 成为新项目的默认选择 |
| 2015 | 约 90% 的新 Web API 使用 JSON |
| 2025 | JSON 统治 Web，XML 退守银行/医疗/配置 |

## 总结

**XML vs JSON 对比**

| 维度 | XML | JSON |
|------|-----|------|
| 设计哲学 | 完整、严谨、可验证 | 够用就好、简单至上 |
| 学习曲线 | 陡峭（WSDL/XSD/命名空间...） | 平缓（10 分钟上手） |
| 生态复杂度 | 高（WS-* 全家桶） | 低（就是个格式） |
| 解析速度 | 慢 | 快 |
| 冗余度 | 高 | 低 |
| 适合场景 | 企业级、强类型、严格校验 | Web/移动、快速开发 |

**核心教训**：

> **在技术选型时，不要追求「最强大」，要追求「够用且简单」。**
>
> 复杂度是有成本的。每一层抽象、每一个特性，都需要有人去理解、维护、调试。
>
> JSON 赢了，不是因为它更好，而是因为它**足够好，同时足够简单**。

---

但 JSON 也不是终点。

当微服务架构兴起，当每秒百万请求成为常态，当移动端对流量斤斤计较——JSON 的文本格式开始显得「太大」「太慢」了。

下一篇，我们来看 **Protocol Buffers 和 Thrift**：当 JSON 不够快的时候，二进制格式如何登场。

---

*下一篇：[二进制觉醒：当 JSON 不够快](/posts/data-format-evolution-2-binary/)*

*本系列：*
1. *从 XML 到 JSON，复杂之死（本篇）*
2. *[二进制觉醒：当 JSON 不够快](/posts/data-format-evolution-2-binary/)*
3. *[零拷贝：当序列化本身也嫌慢](/posts/data-format-evolution-3-zero-copy/)*
4. *[列式革命：当数据以亿行计](/posts/data-format-evolution-4-columnar/)*
5. *[配置文件简史：从混沌到秩序](/posts/data-format-evolution-5-config/)*
6. *[API 范式之争：REST、GraphQL、gRPC](/posts/data-format-evolution-6-api-paradigm/)*
7. *[LLM 时代：TOON 与格式的未来](/posts/data-format-evolution-7-llm-toon/)*
