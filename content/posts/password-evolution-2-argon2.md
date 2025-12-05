---
title: "密码安全进化史（二）：Argon2，故意让自己变慢的算法"
date: 2025-12-05T11:00:00+08:00
draft: false
tags: ["密码安全", "Argon2", "哈希", "安全"]
categories: ["技术"]
description: "MD5 太快了，快到黑客可以每秒尝试 100 亿次。Argon2 反其道而行之——故意让自己变慢。"
series: ["密码安全进化史"]
cover:
  image: "/images/posts/password-evolution-2-argon2.svg"
  alt: "Argon2 - Fortress"
  relative: false
---

## 一个反直觉的想法

在大多数场景下，我们追求的是"快"：
- 网页加载要快
- 数据库查询要快
- 文件传输要快

但密码哈希是个例外。这里我们需要"慢"。

### 为什么慢反而是好事？

想象两种锁：

**快锁（MD5）**：
> 尝试一把钥匙只需 0.0000001 秒
> 小偷每秒能试 1000 万把钥匙
> 6 位密码锁：0.1 秒破解

**慢锁（Argon2）**：
> 尝试一把钥匙需要 0.5 秒
> 小偷每秒只能试 2 把钥匙
> 6 位密码锁：6 天破解

对于**正常用户**：
- 登录时等 0.5 秒？完全可以接受
- 反正只输一次密码

对于**黑客**：
- 每次尝试都要 0.5 秒？灾难！
- 原本 1 秒破解的密码，现在要几个月

**这就是"慢哈希"的精髓：让暴力破解变得不经济。**

## Argon2：专为密码设计

Argon2 是 2015 年"密码哈希竞赛"（Password Hashing Competition）的冠军。这场比赛的目的就是找出最适合存储密码的算法。

### 为什么不用 bcrypt？

在 Argon2 之前，bcrypt（1999 年）是最流行的密码哈希算法。它已经够慢了，为什么还需要 Argon2？

答案是：**GPU 和专用硬件（ASIC）**。

```text
bcrypt 的问题：

bcrypt 设计于 1999 年，那时的攻击者用 CPU 暴力破解。
bcrypt 对 CPU 来说很慢 ✓

但 2010 年代，攻击者开始用 GPU：
- 一块 GPU = 数千个并行计算核心
- GPU 擅长并行执行简单任务
- bcrypt 虽慢，但内存需求低，GPU 照样能跑

更狠的是 ASIC（专用芯片）：
- 专门为破解密码设计的硬件
- 比 GPU 还快 100 倍
```

### Argon2 的秘密武器：吃内存

Argon2 的核心创新是：**不仅消耗时间，还消耗大量内存**。

```text
bcrypt：需要 4KB 内存
Argon2：可配置为需要 64MB、256MB、甚至 1GB 内存
```

为什么这很重要？

```text
GPU 的特点：
✅ 计算核心多（几千个）
❌ 每个核心的内存很小

如果算法需要 64MB 内存：
- 一块有 4000 个核心的 GPU
- 只有 8GB 显存
- 实际只能同时跑 8GB ÷ 64MB = 125 个并行任务
- 白白浪费了 3875 个核心！

ASIC 更惨：
- 高速内存极其昂贵
- 给 ASIC 配 1GB 高速内存？成本爆炸
```

**Argon2 通过"吃内存"让 GPU 和 ASIC 失去优势，把战场拉回 CPU。**

## Argon2 的三种口味

Argon2 有三个变体，针对不同场景：

| 变体 | 特点 | 适用场景 |
|-----|------|---------|
| **Argon2d** | 抗 GPU/ASIC，但可能受侧信道攻击 | 后端密码哈希（推荐） |
| **Argon2i** | 抗侧信道攻击，但抗 GPU 稍弱 | 前端/加密密钥派生 |
| **Argon2id** | 前两者的混合，平衡安全性 | **通用推荐**（OWASP 推荐） |

**简单原则：不确定用哪个？选 Argon2id。**

## Argon2 的三个旋钮

Argon2 让你自己调节"有多慢"：

```text
Argon2(密码, 盐, 时间成本, 内存成本, 并行度) → 哈希值
```

### 1. 时间成本（迭代次数）

```text
时间成本 = 1：快，但不够安全
时间成本 = 3：中等（推荐起点）
时间成本 = 10：慢，非常安全
```

### 2. 内存成本（KB）

```text
内存成本 = 65536（64MB）：基本防护（推荐最低值）
内存成本 = 262144（256MB）：良好防护
内存成本 = 1048576（1GB）：极端防护
```

### 3. 并行度

```text
并行度 = 1：单线程
并行度 = 4：利用 4 个 CPU 核心（推荐）
并行度 = 8：利用 8 个 CPU 核心
```

### OWASP 推荐配置（2023）

```text
最低配置：
- Argon2id
- 内存：64 MB
- 迭代：3 次
- 并行度：4
- 输出：32 字节

高安全配置：
- Argon2id
- 内存：256 MB
- 迭代：4 次
- 并行度：4
- 输出：32 字节
```

## 实际代码长什么样？

### Python 示例

```python
from argon2 import PasswordHasher

# 创建哈希器（使用默认的安全配置）
ph = PasswordHasher(
    time_cost=3,        # 迭代次数
    memory_cost=65536,  # 64MB 内存
    parallelism=4       # 4 线程
)

# 注册时：哈希密码
password = "用户输入的密码"
hash = ph.hash(password)
# 存储 hash 到数据库

# 登录时：验证密码
try:
    ph.verify(hash, password)
    print("密码正确！")
except:
    print("密码错误！")
```

### Rust 示例

```rust
use argon2::{
    password_hash::{PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2
};

// 注册时：哈希密码
let password = b"user_password";
let salt = SaltString::generate(&mut rand::thread_rng());
let argon2 = Argon2::default();
let hash = argon2.hash_password(password, &salt)?.to_string();
// 存储 hash 到数据库

// 登录时：验证密码
let parsed_hash = PasswordHash::new(&hash)?;
argon2.verify_password(password, &parsed_hash)?;
```

## 一张图看懂 Argon2 的内部

```text
Input: Password + Salt + Parameters

+---------------------------------------------+
|              Memory Matrix                  |
|  +-----+-----+-----+-----+-----+-----+      |
|  | B0  | B1  | B2  | B3  | ... | Bn  |      |  <- Round 1: Fill
|  +-----+-----+-----+-----+-----+-----+      |
|    |     |     |     |           |          |
|    v     v     v     v           v          |
|  +-----+-----+-----+-----+-----+-----+      |
|  | B0' | B1' | B2' | B3' | ... | Bn' |      |  <- Round 2: Mix
|  +-----+-----+-----+-----+-----+-----+      |
|    |     |     |     |           |          |
|    v     v     v     v           v          |
|         ... (repeat N rounds) ...           |
|                                             |
|  Each block depends on previous blocks      |
|  Must keep entire matrix in memory          |
+---------------------------------------------+
                    |
                    v
              Final Hash
```

**关键点**：每个块都依赖其他块，你不能只算一小部分。必须把整个内存矩阵都填满、都保留。这就是为什么它"吃内存"。

## Argon2 vs 其他算法

| 算法 | 发明年份 | 内存需求 | 抗 GPU | 抗 ASIC | 推荐程度 |
|------|---------|---------|--------|---------|---------|
| MD5 | 1991 | 极低 | ❌ | ❌ | ⛔ 禁止 |
| SHA-256 | 2001 | 极低 | ❌ | ❌ | ⛔ 禁止 |
| bcrypt | 1999 | 4KB | ⚠️ 一般 | ⚠️ 一般 | ✅ 可用 |
| scrypt | 2009 | 可配置 | ✅ 好 | ✅ 好 | ✅ 推荐 |
| **Argon2** | 2015 | **可配置** | ✅ **极好** | ✅ **极好** | ✅ **首选** |

## 常见问题

### Q：Argon2 会不会让我的服务器变慢？

**A：会，但这是值得的。**

```text
一次 Argon2 计算（推荐配置）：约 0.3-0.5 秒

影响分析：
- 用户登录时多等 0.5 秒？几乎无感
- 注册时多等 0.5 秒？无所谓
- 服务器 CPU 占用高？每次登录只算一次，影响很小
- 内存占用 64MB？计算完立即释放，可接受

对比收益：
- 黑客破解成本提高 1000 万倍
- 数据库泄露后，用户有时间改密码
```

### Q：应该用多大的内存？

**A：在你的服务器能承受的范围内，尽量大。**

```text
决策流程：
1. 预估同时登录的用户数（比如 100）
2. 每个用户占用 64MB 内存
3. 峰值内存 = 100 × 64MB = 6.4GB
4. 你的服务器能承受吗？
   - 能 → 用 64MB 或更高
   - 不能 → 降到 32MB，或增加服务器内存
```

### Q：旧密码用 MD5 存的，怎么迁移？

**A：渐进式迁移。**

```text
方案一：双重哈希（不推荐但可行）
新哈希 = Argon2(旧的 MD5 哈希)
问题：如果旧 MD5 已被破解，这个方案没用

方案二：下次登录时迁移（推荐）
1. 用户登录，提交密码
2. 用 MD5 验证（如果还是旧格式）
3. 验证通过后，用 Argon2 重新哈希，存储新格式
4. 逐步完成迁移

代码示例：
if is_old_format(stored_hash):
    if md5(password) == stored_hash:
        new_hash = argon2(password)
        update_database(user, new_hash)
        return "登录成功"
else:
    return argon2.verify(stored_hash, password)
```

## Argon2 的局限性

虽然 Argon2 是目前最好的密码哈希算法，但它仍有局限：

```text
Argon2 能防止的：
✅ 彩虹表攻击
✅ GPU/ASIC 暴力破解
✅ 数据库泄露后的快速批量破解

Argon2 不能防止的：
❌ 弱密码（"123456" 还是很快被破解）
❌ 密码在传输中被截获
❌ 服务器被完全控制
❌ 用户主动泄露密码
```

**Argon2 只是密码安全的一环，不是全部。**

## 总结

```text
密码存储进化：

MD5           →    bcrypt       →    Argon2
太快              慢，但内存小        慢 + 吃内存

对抗：
普通 CPU          专用 CPU           GPU + ASIC
```

Argon2 通过两个维度增加破解成本：
1. **时间成本**：每次计算需要更长时间
2. **内存成本**：每次计算需要大量内存，让 GPU/ASIC 失去优势

如果你正在开发一个需要存储密码的系统，**请使用 Argon2id**。这是 2024 年的行业最佳实践。

---

但是，即使用了 Argon2，密码仍然要传到服务器上，服务器仍然会"见到"你的密码（即使只是短暂地）。有没有办法让服务器**永远不知道**你的密码呢？

这就是我们下一篇的主题：**SRP 协议**——一种"零知识证明"风格的密码认证方案。

---

*上一篇：[MD5：一部血泪史](/posts/password-evolution-1-md5/)*
*下一篇：[SRP：证明你知道密码，却不说出密码](/posts/password-evolution-3-srp/)*

*本系列：*
1. *[MD5：一部血泪史](/posts/password-evolution-1-md5/)*
2. *Argon2：慢哈希的艺术（本篇）*
3. *[SRP：零知识证明风格的认证](/posts/password-evolution-3-srp/)*
4. *[OPAQUE：防离线破解的终极方案](/posts/password-evolution-4-opaque/)*
