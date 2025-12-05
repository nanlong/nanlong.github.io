---
title: "密码安全进化史（四）：OPAQUE，防离线破解的终极方案"
date: 2025-12-05T13:00:00+08:00
draft: false
tags: ["密码安全", "OPAQUE", "OPRF", "安全"]
categories: ["技术"]
description: "服务器帮你验证密码，但它连你密码的哈希值都不知道。OPAQUE 实现了密码安全的圣杯。"
series: ["密码安全进化史"]
cover:
  image: "/images/posts/password-evolution-4-opaque.svg"
  alt: "OPAQUE - Next Generation"
  relative: false
---

## 前情回顾：SRP 的遗憾

在上一篇中，我们介绍了 SRP 协议。它已经很强大了：

- ✅ 密码不在网络上传输
- ✅ 服务器不知道真实密码
- ✅ 监听者无法窃取密码

但 SRP 有一个隐藏的弱点：**服务器存储的"验证器"仍然与密码直接相关**。

```text
SRP 服务器存储的：
验证器 v = g^(hash(盐, 密码)) mod N
```

这意味着：如果黑客偷到了数据库，他可以进行**离线暴力破解**：

```text
黑客的攻击方式：
1. 偷到 v 和盐
2. 猜测密码："123456"？"password"？"张三的生日"？
3. 计算 g^(hash(盐, 猜测的密码)) mod N
4. 和 v 对比，如果相等，密码猜对了！

这个过程可以离线进行，每秒尝试数十亿次。
```

## OPAQUE 的野心

OPAQUE（2018年提出，2023年成为IETF标准草案）的目标更加激进：

> **即使服务器数据库完全泄露，攻击者也无法离线破解密码。**

怎么做到的？答案是：**服务器自己都不知道验证器是怎么算出来的！**

## 一个脑洞大开的类比

想象一个特殊的"蒙眼锁匠"场景：

### 背景设定

你有一个保险箱，钥匙的制作方法很特殊：
- 钥匙 = 你的**秘密配方** + 锁匠的**独门手法**
- 两者缺一不可

### SRP 方案的问题

> 你用配方自己做了钥匙，把钥匙的"形状参数"告诉锁匠存档。
>
> 问题：如果锁匠的档案室被盗，小偷拿到"形状参数"后，可以在自己家里疯狂尝试各种配方，看哪个能做出相同形状的钥匙。每秒试十亿次，总能试出来。

### OPAQUE 方案

```text
Registration:

You                                    Blind Locksmith
 |                                              |
 |  1. Write recipe on paper, put in envelope   |
 |  ------------------------------------------> |
 |                                              | Locksmith processes it
 |                                              | blindfolded
 |  2. Envelope returned unchanged              |
 |  <------------------------------------------ |
 |                                              |
 |  3. You open the envelope                    |
 |     No longer the original recipe            |
 |     But "recipe + secret technique" fused    |
 |     This is the real key!                    |
 |                                              |
 |  4. Lock safe with key, store at locksmith   |
```

**关键点**：
- 锁匠是**蒙眼操作**的，他不知道你的配方是什么
- 但没有锁匠的"独门手法"，你做不出正确的钥匙
- 锁匠的"独门手法"是他的肌肉记忆，**不写在任何档案里**

### 如果小偷偷了锁匠的档案室？

```text
Thief gets:
+-- Your safe [yes]
+-- Some encrypted data [yes]
+-- Locksmith's "secret technique"? [NO! It's in his head!]

Thief wants to crack:
+-- Guess a recipe
+-- Try to make a key?
+-- But key = recipe + secret technique
+-- Only locksmith knows the technique!
+-- Must ask locksmith for help

Locksmith's rules:
+-- Only one visit per minute per person
+-- 5 wrong guesses = blacklisted
+-- Thief's "10 billion tries/sec"? Not happening!
```

**这就是 OPAQUE 的革命性突破：把暴力破解从"离线"逼到"在线"。**

## OPAQUE 的两个关键技术

### 1. OPRF：双盲计算

OPRF（Oblivious Pseudo-Random Function，不经意伪随机函数）是 OPAQUE 的核心黑科技。

普通的密码哈希：
```text
结果 = hash(盐, 密码)
↓
任何知道密码和盐的人都能计算这个结果
```

OPRF 的哈希：
```text
结果 = OPRF(服务器密钥, 密码)
↓
必须服务器参与才能计算
但服务器不知道密码是什么！
```

这听起来像悖论？让我解释：

```text
+----------+                              +----------+
|   You    |                              |  Server  |
| know pwd |                              | know key |
+----------+                              +----------+
     |                                         |
     |  1. "Blindfold" password (math blind)   |
     |     Send blinded value                  |
     |  ----------------------------------------> |
     |                                          |
     |  2. Server processes with key           |
     |     (Server doesn't know password)      |
     |  <---------------------------------------- |
     |                                          |
     |  3. You "remove blindfold" (unblind)    |
     |     Get OPRF(key, password) result      |
     |                                          |
     |  Result:                                 |
     |  - You got a value that needs key       |
     |  - Server doesn't know your password    |
     |  - Eavesdropper sees nothing useful     |
```

**这就像一个神奇的翻译机器**：
- 你把密码用密码本 A 加密后发过去
- 服务器用它的密码本 B 再加密一次后发回来
- 你用密码本 A 解密
- 最终结果是只用密码本 B 加密的密码
- 但服务器从没见过解密后的密码！

### 2. AKE：认证密钥交换

在 OPRF 的基础上，OPAQUE 使用 AKE（Authenticated Key Exchange）完成：

- 双方相互认证身份
- 建立加密的会话密钥
- 防止中间人攻击

## OPAQUE 完整流程

### 注册阶段

```text
+--------+                                +--------+
|  User  |                                | Server |
+--------+                                +--------+
    |                                         |
    |  1. Generate random keypair             |
    |     (private_key, public_key)           |
    |                                         |
    |  2. Get "vault key" via OPRF            |
    |     (needs password + server)           |
    |  <-------------------------------------> |
    |                                         |
    |  3. Encrypt private key with vault key  |
    |     envelope = encrypt(priv, OPRF_out)  |
    |                                         |
    |  4. Send: public_key + envelope         |
    |  ----------------------------------------> |
    |                                         |
    |     Server stores:                      |
    |     - public_key                        |
    |     - envelope (encrypted private key)  |
    |     - OPRF key material                 |
    |                                         |
    |     Server does NOT store:              |
    |     X password                          |
    |     X password hash                     |
    |     X anything for offline cracking     |
```

### 登录阶段

```text
+--------+                                +--------+
|  User  |                                | Server |
+--------+                                +--------+
    |                                         |
    |  1. Initiate login request              |
    |  ----------------------------------------> |
    |                                         |
    |  2. Server returns envelope             |
    |  <---------------------------------------- |
    |                                         |
    |  3. Recompute "vault key" via OPRF      |
    |     (needs password + server)           |
    |  <-------------------------------------> |
    |                                         |
    |  4. Decrypt envelope to get private key |
    |     Wrong password -> decrypt fails     |
    |     Correct password -> get private key |
    |                                         |
    |  5. Complete auth with private key      |
    |  <-------------------------------------> |
    |                                         |
    |  Login successful!                      |
```

## 为什么 OPAQUE 能防止离线破解？

关键在于：**暴力破解必须在线进行**。

```text
攻击者偷到数据库后得到：
- envelope（加密的私钥）
- 公钥
- OPRF 相关数据

攻击者想破解密码：
1. 猜测密码："123456"
2. 想计算 OPRF(密钥, "123456") 来尝试解密 envelope
3. 但是... OPRF 计算需要服务器参与！
4. 攻击者必须向服务器发送请求
5. 服务器可以：限速、检测异常、锁定账户

结果：
- 不能离线每秒猜十亿次
- 只能在线每秒猜几次（受服务器限制）
- 被发现的概率极大
```

**这就是 OPAQUE 的革命性突破：把暴力破解从"离线"逼到"在线"。**

## SRP vs OPAQUE 终极对比

| 特性 | SRP | OPAQUE |
|-----|-----|--------|
| 密码不传输 | ✅ | ✅ |
| 服务器不知道密码 | ✅ | ✅ |
| 防网络监听 | ✅ | ✅ |
| 防服务器数据库泄露 | ⚠️ 可离线破解 | ✅ 只能在线破解 |
| 实现复杂度 | 中等 | 较高 |
| 标准化程度 | RFC 5054（较老） | IETF 草案（最新） |
| 量子计算抵抗 | ❌ | 可升级 |

## OPAQUE 的代价

天下没有免费的午餐，OPAQUE 的强大安全性也有代价：

1. **实现复杂**：需要实现 OPRF，比 SRP 更复杂
2. **计算开销**：每次登录需要额外的 OPRF 交互
3. **生态不成熟**：相比传统方案，库和工具较少
4. **服务器成本**：每次登录都需要服务器参与计算

## 谁在使用 OPAQUE？

虽然 OPAQUE 还比较新，但已经开始被采用：

- **WhatsApp**：端到端加密密钥的备份恢复
- **Cloudflare**：探索用于身份验证服务
- **Signal**：考虑用于账户恢复
- **企业安全系统**：高安全需求场景

## 未来展望

OPAQUE 代表了密码认证的未来方向：

```text
过去：服务器知道你的密码（明文存储）
     ↓
现在：服务器知道密码的哈希（可离线破解）
     ↓
未来：服务器什么都不知道，但仍能验证你（OPAQUE）
```

更远的未来可能是：

- **无密码认证**：生物识别 + 硬件密钥
- **去中心化身份**：区块链 + 零知识证明
- **后量子安全**：抵抗量子计算机的算法

## 技术细节（给好奇的读者）

### OPRF 的数学原理

OPRF 基于椭圆曲线密码学：

```text
用户有：密码 pwd
服务器有：密钥 k

OPRF 计算过程：
1. 用户：生成随机数 r，计算 blinded = hash_to_curve(pwd) * r
2. 用户 → 服务器：发送 blinded
3. 服务器：计算 evaluated = blinded * k（服务器不知道 pwd）
4. 服务器 → 用户：发送 evaluated
5. 用户：计算 result = evaluated * (1/r) = hash_to_curve(pwd) * k

结果：
- 用户得到 OPRF(k, pwd)
- 服务器不知道 pwd
- 监听者只看到 blinded 和 evaluated，推不出 pwd
```

### 为什么服务器数据泄露不能离线破解？

```text
泄露的数据包含：envelope = 加密(私钥, OPRF(k, pwd))

攻击者想法：
- 猜测 pwd' = "123456"
- 计算 OPRF(k, pwd')
- 尝试解密 envelope

问题：
- 计算 OPRF(k, pwd') 需要密钥 k
- k 存在服务器内存中，不在数据库里
- 即使泄露了数据库，k 仍然安全
- 攻击者必须请求服务器帮他计算 OPRF
- 服务器可以限制、监控、拒绝
```

## 总结

OPAQUE 实现了密码安全的"圣杯"：

```text
✅ 密码永远不离开你的设备
✅ 服务器不知道你的密码
✅ 服务器甚至不知道密码的哈希
✅ 数据库泄露也无法离线暴力破解
✅ 双向认证，防止钓鱼网站
```

它的核心魔法是 **OPRF**——一种让服务器参与计算、但又不知道计算内容的神奇技术。

虽然 OPAQUE 目前还不像传统密码方案那样普及，但它代表了密码认证的未来。随着网络安全威胁越来越严峻，我们可能会看到越来越多的服务采用 OPAQUE 或类似的先进协议。

**毕竟，最好的密码保护方式，就是让任何人（包括你信任的服务器）都接触不到你的密码。**

---

## 系列总结：密码安全的进化之路

```text
+---------------------------------------------------------------------+
|              Password Security Evolution Map                        |
+---------------------------------------------------------------------+
|                                                                     |
|  MD5          Argon2         SRP           OPAQUE                   |
|   |             |             |              |                      |
|   v             v             v              v                      |
|  Fast hash -> Slow hash -> No pwd sent -> No offline crack          |
|                                                                     |
|  Problem:     Improve:       Improve:       Improve:                |
|  Instant      Cost up        Server blind   DB leak safe            |
|  crack                                                              |
|                                                                     |
|  <------------------- Security increases -------------------->      |
|  <------------------- Complexity increases ------------------>      |
+---------------------------------------------------------------------+
```

**选型建议**：

| 场景 | 推荐方案 |
|-----|---------|
| 一般 Web 应用 | Argon2id（够用，生态成熟） |
| 高安全需求（金融、医疗） | SRP 或 OPAQUE |
| 终极安全需求 | OPAQUE + 硬件密钥 |

密码安全没有银弹，但选择正确的方案，能让攻击者的成本从"几秒钟"变成"几千年"。

---

*上一篇：[SRP：证明你知道密码却不说出密码](/posts/password-evolution-3-srp/)*

*本系列：*
1. *[MD5：一部血泪史](/posts/password-evolution-1-md5/)*
2. *[Argon2：慢哈希的艺术](/posts/password-evolution-2-argon2/)*
3. *[SRP：证明你知道密码却不说出密码](/posts/password-evolution-3-srp/)*
4. *OPAQUE：防离线破解的终极方案（本篇）*

*延伸阅读：零知识证明、后量子密码学、WebAuthn 无密码认证*
