---
title: "架构演进史（三）：Context 之道，从技术分层到业务分层"
date: 2025-12-08T16:20:00+08:00
draft: false
tags: ["架构", "Elixir", "Phoenix", "Context", "模块化", "业务边界"]
categories: ["技术"]
description: "当我第一次看到 Phoenix 生成的代码结构时，有点困惑：为什么不是 models/、controllers/，而是 accounts/、catalog/？这不是按功能划分，而是按业务划分。"
series: ["架构演进史"]
---

当我第一次看到 Phoenix 生成的代码结构时，有点困惑：为什么不是 `models/`、`controllers/`，而是 `accounts/`、`catalog/`？这不是按功能划分，而是按业务划分。

<!--more-->

## 前情回顾

在前两篇中，我们经历了：

- **混沌时代**：代码没有边界，一切混在一起
- **MVC 时代**：技术分层（Model-View-Controller），但业务逻辑散落各处

MVC 解决了「怎么分层」的问题，但留下了一个更难的问题：**业务逻辑应该怎么组织？**

今天，我们来看 Elixir Phoenix 的答案：**Context**。

## 一个发人深省的目录结构

传统 MVC 项目的目录结构：

```text
app/
├── models/
│   ├── user.py
│   ├── order.py
│   └── product.py
├── views/
│   ├── user_views.py
│   ├── order_views.py
│   └── product_views.py
└── controllers/
    ├── user_controller.py
    ├── order_controller.py
    └── product_controller.py
```

Phoenix 项目的目录结构：

```text
lib/my_app/
├── accounts/           # 「账户」上下文
│   ├── user.ex
│   └── accounts.ex
├── catalog/            # 「商品目录」上下文
│   ├── product.ex
│   └── catalog.ex
└── orders/             # 「订单」上下文
    ├── order.ex
    └── orders.ex
```

**看出区别了吗？**

传统 MVC：按**技术类型**分（models、views、controllers）
Phoenix：按**业务领域**分（accounts、catalog、orders）

这不只是文件夹命名的不同，是**思维方式的转变**。

## 什么是 Context？

Phoenix 官方文档对 Context 的定义：

> **Context 是暴露和组织相关功能的模块。**

听起来很抽象。让我用一个类比解释。

### 类比：公司的部门

想象一家电商公司：

| 部门 | 职责 | 对外接口 |
|------|------|---------|
| 用户部门 | 管理用户注册、登录、权限 | 「帮我注册个用户」「这个用户是 VIP 吗？」 |
| 商品部门 | 管理商品、库存、价格 | 「给我看看商品列表」「这个商品还有货吗？」 |
| 订单部门 | 管理订单、支付、物流 | 「帮我创建个订单」「这个订单发货了吗？」 |

每个部门：
- 有自己的**内部运作方式**（你不需要知道）
- 对外提供**明确的服务接口**（你只需要调用）

**这就是 Context 的本质：按业务边界划分的模块，对外暴露清晰的 API。**

### 代码层面的 Context

看看 Phoenix 的 Accounts Context：

```elixir
defmodule MyApp.Accounts do
  @moduledoc """
  账户上下文。处理用户相关的所有业务逻辑。
  """

  alias MyApp.Accounts.User
  alias MyApp.Repo

  # ===== 公开 API =====

  def list_users do
    Repo.all(User)
  end

  def get_user!(id) do
    Repo.get!(User, id)
  end

  def create_user(attrs \\ %{}) do
    %User{}
    |> User.changeset(attrs)
    |> Repo.insert()
  end

  def authenticate_user(email, password) do
    user = Repo.get_by(User, email: email)

    cond do
      user && verify_password(password, user.password_hash) ->
        {:ok, user}
      user ->
        {:error, :invalid_password}
      true ->
        {:error, :user_not_found}
    end
  end

  # ===== 私有函数 =====

  defp verify_password(password, hash) do
    Bcrypt.verify_pass(password, hash)
  end
end
```

**关键点**：

1. **公开 API 是业务语言**：`create_user`、`authenticate_user`，不是技术语言
2. **实现细节被隐藏**：外部不知道用了 Bcrypt，不知道用了什么数据库
3. **边界清晰**：所有用户相关的操作都在这里，不散落到其他地方

## Controller 变薄了

有了 Context，Controller 变得极其简单：

```elixir
defmodule MyAppWeb.UserController do
  use MyAppWeb, :controller

  alias MyApp.Accounts

  def index(conn, _params) do
    users = Accounts.list_users()
    render(conn, "index.html", users: users)
  end

  def create(conn, %{"user" => user_params}) do
    case Accounts.create_user(user_params) do
      {:ok, user} ->
        conn
        |> put_flash(:info, "用户创建成功")
        |> redirect(to: Routes.user_path(conn, :show, user))

      {:error, changeset} ->
        render(conn, "new.html", changeset: changeset)
    end
  end
end
```

Controller 现在只做三件事：
1. 接收 HTTP 请求
2. 调用 Context 的业务方法
3. 返回 HTTP 响应

**不再有业务逻辑。** 业务逻辑全在 Context 里。

## Context 的核心价值

### 价值 1：业务边界清晰

当产品经理说「用户下单」，开发者知道：
- 去 `Orders` Context 找 `create_order/1`
- 不用翻遍整个项目

当新人入职，他可以问：「这个项目有哪些业务领域？」
答案就是：看 `lib/my_app/` 下有哪些文件夹。

### 价值 2：依赖方向可控

```text
Controller → Context → Schema/Repo
   ↓
  View
```

依赖是**单向的**。Context 不知道 Controller 的存在。Schema 不知道 Context 的存在。

这意味着：
- 换一个 Web 框架？只需要重写 Controller 和 View
- 换一个数据库？只需要改 Context 内部实现
- 业务逻辑不受影响

### 价值 3：可测试性大增

测试 Context，不需要启动 HTTP 服务器：

```elixir
defmodule MyApp.AccountsTest do
  use MyApp.DataCase

  alias MyApp.Accounts

  describe "create_user/1" do
    test "有效参数创建用户成功" do
      attrs = %{name: "Alice", email: "alice@example.com", password: "secret123"}
      assert {:ok, user} = Accounts.create_user(attrs)
      assert user.name == "Alice"
    end

    test "无效邮箱返回错误" do
      attrs = %{name: "Bob", email: "invalid", password: "secret123"}
      assert {:error, changeset} = Accounts.create_user(attrs)
      assert "邮箱格式不正确" in errors_on(changeset).email
    end
  end
end
```

**测试的是业务逻辑，不是 HTTP 路由。**

## 跨 Context 的调用

有时候，一个业务操作涉及多个 Context。比如「创建订单」需要：
- 检查用户是否存在（Accounts）
- 检查商品库存（Catalog）
- 创建订单（Orders）

Phoenix 的建议是：**让 Controller 协调，或者创建一个 Facade**。

### 方案 1：Controller 协调

```elixir
def create(conn, %{"order" => order_params}) do
  user = Accounts.get_user!(order_params["user_id"])
  product = Catalog.get_product!(order_params["product_id"])

  if Catalog.in_stock?(product, order_params["quantity"]) do
    {:ok, order} = Orders.create_order(user, product, order_params)
    # ...
  else
    # ...
  end
end
```

简单场景可以这样做。

### 方案 2：创建一个业务流程模块

复杂场景，创建一个专门的模块：

```elixir
defmodule MyApp.Checkout do
  @moduledoc """
  结账流程。协调多个 Context 完成下单。
  """

  alias MyApp.{Accounts, Catalog, Orders}

  def place_order(user_id, product_id, quantity) do
    with {:ok, user} <- get_active_user(user_id),
         {:ok, product} <- get_available_product(product_id, quantity),
         {:ok, order} <- Orders.create_order(user, product, quantity),
         :ok <- Catalog.decrease_stock(product, quantity) do
      {:ok, order}
    end
  end

  defp get_active_user(user_id) do
    case Accounts.get_user(user_id) do
      nil -> {:error, :user_not_found}
      user -> if user.active, do: {:ok, user}, else: {:error, :user_inactive}
    end
  end

  defp get_available_product(product_id, quantity) do
    case Catalog.get_product(product_id) do
      nil -> {:error, :product_not_found}
      product -> if Catalog.in_stock?(product, quantity), do: {:ok, product}, else: {:error, :out_of_stock}
    end
  end
end
```

这个模块：
- 不属于任何一个 Context
- 协调多个 Context 完成业务流程
- 是一个「用例」或「应用服务」

## 从 Context 到 DDD

用了一段时间 Phoenix Context 后，我突然意识到：**这就是 DDD（领域驱动设计）的雏形！**

| Phoenix 概念 | DDD 概念 |
|-------------|---------|
| Context | Bounded Context（限界上下文） |
| Schema | Entity（实体） |
| Context 模块 | Application Service（应用服务） |
| 跨 Context 协调 | Domain Service（领域服务） |

Phoenix 没有刻意宣传 DDD，但它的设计**自然地引导你走向 DDD 的思维方式**。

## Context 设计的原则

### 原则 1：按业务能力划分，不按数据划分

**错误**：

```text
lib/my_app/
├── users/      # User 表相关
├── orders/     # Order 表相关
└── products/   # Product 表相关
```

这只是把 `models/` 换了个名字。

**正确**：

```text
lib/my_app/
├── accounts/       # 用户注册、认证、授权
├── catalog/        # 商品浏览、搜索、分类
├── orders/         # 下单、支付、履约
└── inventory/      # 库存管理、预警
```

按「业务能力」划分。一个 Context 可能涉及多个表，一个表也可能被多个 Context 使用。

### 原则 2：Context 之间松耦合

Context 之间通过**公开 API** 通信，不直接访问对方的内部：

```elixir
# ❌ 错误：直接访问另一个 Context 的 Repo
def create_order(user_id, ...) do
  user = MyApp.Repo.get!(MyApp.Accounts.User, user_id)
  # ...
end

# ✅ 正确：调用 Context 的公开 API
def create_order(user_id, ...) do
  user = MyApp.Accounts.get_user!(user_id)
  # ...
end
```

### 原则 3：Context 内部高内聚

一个 Context 内部的代码应该紧密相关。如果发现一个 Context 变得很大，考虑拆分。

**信号**：
- Context 文件超过 500 行
- Context 有多个不相关的功能
- 团队中不同人负责同一个 Context 的不同部分

## 真实项目的演进

让我分享一个真实项目从 MVC 到 Context 的演进：

### 阶段 1：传统 MVC

```text
web/
├── controllers/
├── models/
└── views/
```

### 阶段 2：引入 Service 层

```text
lib/my_app/
├── services/
│   ├── user_service.ex
│   └── order_service.ex
web/
├── controllers/
└── views/
```

问题：Service 之间互相调用，依赖混乱。

### 阶段 3：引入 Context

```text
lib/my_app/
├── accounts/
│   ├── accounts.ex      # Context 入口
│   ├── user.ex          # Schema
│   └── credential.ex    # Schema
├── orders/
│   ├── orders.ex
│   └── order.ex
lib/my_app_web/
├── controllers/
└── views/
```

**改变**：
- 代码按业务领域组织
- 每个 Context 有明确的边界
- 依赖方向清晰

## 常见问题

### Q：一个 Schema 可以属于多个 Context 吗？

**A：可以，但要小心。**

有时候一个表（比如 `users`）被多个 Context 使用：
- `Accounts` Context：用户注册、登录
- `Billing` Context：用户的付费信息
- `Admin` Context：用户管理

方案：
1. **共享 Schema**：都用 `MyApp.Accounts.User`
2. **各自定义 Schema**：`Accounts.User`、`Billing.Customer`、`Admin.ManagedUser`

DDD 更推荐方案 2，每个 Context 有自己的视角。但 Phoenix 实践中，方案 1 更常见。

### Q：Context 之间可以调用吗？

**A：可以，但要控制方向。**

原则：
- **单向依赖**：A 可以调用 B，但 B 不能反过来调用 A
- **通过公开 API**：不访问对方的内部实现
- **考虑异步**：复杂场景用事件/消息代替直接调用

### Q：Context 怎么测试？

**A：单元测试 Context，集成测试跨 Context 流程。**

```elixir
# 单元测试：测试单个 Context
defmodule MyApp.AccountsTest do
  # ...
end

# 集成测试：测试跨 Context 流程
defmodule MyApp.CheckoutTest do
  test "完整下单流程" do
    user = create_user()
    product = create_product()
    assert {:ok, order} = Checkout.place_order(user.id, product.id, 1)
  end
end
```

## 总结

**Context 的核心价值**：
- 按**业务边界**而非技术层次组织代码
- 每个 Context 是一个**自治的模块**
- 对外暴露**清晰的 API**，隐藏内部实现
- 为走向 **DDD** 打下基础

**从 MVC 到 Context 的转变**：

| 维度 | MVC | Context |
|------|-----|---------|
| 组织方式 | 按技术类型（models, views） | 按业务领域（accounts, orders） |
| 关注点 | 技术实现 | 业务能力 |
| 边界 | 模糊（Service 互相调用） | 清晰（通过公开 API） |
| 可发现性 | 要找功能，需要翻多个文件夹 | 功能就在对应的 Context 里 |

**核心教训**：

> **技术分层是程序员的思维，业务分层是领域专家的思维。**

Context 教会我们用业务的视角看代码。这是走向 DDD 的第一步。

下一篇，我们来看完整的 DDD——当业务足够复杂，Context 还不够，我们需要实体、值对象、聚合根、领域事件...

---

*上一篇：[MVC 启蒙：框架带来的秩序](/posts/architecture-evolution-2-mvc/)*

*下一篇：[DDD 觉醒：让代码说业务的语言](/posts/architecture-evolution-4-ddd/)*

*本系列：*
1. *[混沌时代：当代码没有架构](/posts/architecture-evolution-1-chaos/)*
2. *[MVC 启蒙：框架带来的秩序](/posts/architecture-evolution-2-mvc/)*
3. *Context 之道：从技术分层到业务分层（本篇）*
4. *[DDD 觉醒：让代码说业务的语言](/posts/architecture-evolution-4-ddd/)*
5. *[边界的艺术：六边形与洋葱](/posts/architecture-evolution-5-hexagonal/)*
6. *[单体的边界：何时该拆？](/posts/architecture-evolution-6-modular-monolith/)*
7. *[微服务与云原生：分布式的代价](/posts/architecture-evolution-7-microservices/)*
8. *[没有银弹：架构决策的本质](/posts/architecture-evolution-8-no-silver-bullet/)*
