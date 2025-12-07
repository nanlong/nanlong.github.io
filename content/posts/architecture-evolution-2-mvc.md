---
title: "架构演进史（二）：MVC 启蒙，框架带来的秩序"
date: 2025-12-08T16:10:00+08:00
draft: false
tags: ["架构", "MVC", "Django", "Rails", "Flask", "框架"]
categories: ["技术"]
description: "2005 年，Ruby on Rails 横空出世，一句「约定优于配置」震动了整个 Web 开发界。它告诉程序员：不是你决定代码怎么组织，是框架告诉你怎么组织。"
series: ["架构演进史"]
---

2005 年，Ruby on Rails 横空出世，一句「约定优于配置」震动了整个 Web 开发界。它告诉程序员：不是你决定代码怎么组织，是框架告诉你怎么组织。

<!--more-->

## 前情回顾

在上一篇中，我们看到了混沌时代的代码：
- SQL、HTML、业务逻辑混在一起
- 全局变量满天飞
- 复制粘贴到处都是
- 改一个 bug 引入两个新 bug

这不是代码，是一碗意大利面。

今天，我们来看 MVC 如何拯救了这个乱局。

## 一段改变命运的视频

2005 年，DHH（David Heinemeier Hansson）发布了一个视频：**「15 分钟用 Rails 写一个博客」**。

视频里，他敲了几行代码，一个完整的博客系统就跑起来了。有文章列表、有增删改查、有表单验证。

**这个视频在开发者社区引起了广泛关注。**

那时候，用 Java 写同样的功能，需要：
- 配置 Tomcat
- 写 web.xml
- 写 Struts 配置
- 写 Hibernate 映射
- 写 DAO、Service、Action...

**对于中等复杂度的项目，可能需要一周甚至更长时间。**

Rails 用 15 分钟完成了同样的事。它是怎么做到的？

## MVC：一个古老而强大的模式

MVC（Model-View-Controller）其实不是 Rails 发明的。它诞生于 1979 年的 Smalltalk，比 Web 还老。

但 Rails 把它带入了 Web 开发的主流。

### MVC 是什么？

想象一家餐厅：

| 角色 | 职责 | 对应 MVC |
|------|------|---------|
| 服务员 | 接待客人、传递菜单、上菜 | Controller |
| 厨师 | 做菜、管理食材 | Model |
| 菜品摆盘 | 让菜好看 | View |

**服务员不做菜**（Controller 不处理业务逻辑）
**厨师不接待客人**（Model 不处理 HTTP 请求）
**摆盘不影响味道**（View 不改变数据）

这就是**分离关注点**。每个角色做好自己的事。

### 代码层面的 MVC

看看 Django（Python）的 MVC 实现：

**Model（models.py）—— 数据和业务规则**：

```python
from django.db import models

class User(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def is_active(self):
        return self.last_login > timezone.now() - timedelta(days=30)
```

Model 负责：定义数据结构、数据验证、业务规则。它**不知道** HTTP 是什么，也不知道页面长什么样。

**View（views.py）—— 处理请求，协调响应**：

```python
from django.shortcuts import render, get_object_or_404
from .models import User

def user_list(request):
    users = User.objects.filter(is_active=True)
    return render(request, 'users/list.html', {'users': users})

def user_detail(request, user_id):
    user = get_object_or_404(User, id=user_id)
    return render(request, 'users/detail.html', {'user': user})
```

View（在 Django 里叫 View，在 Rails 里叫 Controller）负责：接收请求、调用 Model、选择模板、返回响应。

**Template（list.html）—— 展示数据**：

```html
{% for user in users %}
<div class="user-card">
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>
</div>
{% endfor %}
```

Template 负责：把数据渲染成 HTML。它**不关心**数据从哪来，只管展示。

## 对比：混沌 vs MVC

让我们对比同一个功能的两种实现：

### 混沌版

```php
<?php
$conn = mysql_connect("localhost", "root", "123456");
mysql_select_db("mydb");
$result = mysql_query("SELECT * FROM users WHERE status = 1");
echo "<html><body><table>";
while ($row = mysql_fetch_array($result)) {
    echo "<tr><td>" . $row['name'] . "</td></tr>";
}
echo "</table></body></html>";
?>
```

**问题**：数据库连接、SQL 查询、HTML 全部混在一起。改任何一样都要动这个文件。

### MVC 版

**Model**：

```python
class User(models.Model):
    name = models.CharField(max_length=100)
    status = models.BooleanField(default=True)

    @classmethod
    def active_users(cls):
        return cls.objects.filter(status=True)
```

**View**：

```python
def user_list(request):
    users = User.active_users()
    return render(request, 'users/list.html', {'users': users})
```

**Template**：

```html
<table>
{% for user in users %}
    <tr><td>{{ user.name }}</td></tr>
{% endfor %}
</table>
```

**改进**：
- 换数据库？只改 Model 的配置
- 改页面样式？只改 Template
- 改查询逻辑？只改 Model 的方法
- 每个文件都很短，职责单一

## 约定优于配置

Rails 的另一个革命性理念是「约定优于配置」（Convention over Configuration）。

### 以前：配置地狱

Java Web 项目需要大量 XML 配置：

```xml
<servlet>
    <servlet-name>userServlet</servlet-name>
    <servlet-class>com.example.UserServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>userServlet</servlet-name>
    <url-pattern>/users/*</url-pattern>
</servlet-mapping>
<!-- 每个 Servlet 都要配置... -->
```

一个中型项目，配置文件可能有几千行。

### 现在：约定即配置

Rails/Django 说：**按我的规则放文件，不需要配置。**

```text
app/
├── models/
│   └── user.rb        # User 模型
├── controllers/
│   └── users_controller.rb  # /users/* 的控制器
└── views/
    └── users/
        ├── index.html.erb   # GET /users
        └── show.html.erb    # GET /users/:id
```

文件名是 `users_controller.rb`，框架就知道它处理 `/users` 路由。
方法名是 `index`，框架就知道它对应 `GET /users`。

**零配置，全靠约定。**

## 我的 Django/Flask 时代

从 PHP 混沌中逃离后，我拥抱了 Python 生态。

### Django：全功能框架

Django 的口号是「The web framework for perfectionists with deadlines」（有 deadline 的完美主义者的框架）。

它提供了一切：
- ORM（对象关系映射）
- Admin 后台（自动生成）
- 表单处理
- 用户认证
- 国际化
- ...

**开箱即用。** 对于 CRUD 应用，Django 的生产力惊人。

### Flask：微框架

但有时候 Django 太「重」了。Flask 走了另一条路——只提供核心功能，其他全靠插件。

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.route('/users')
def user_list():
    users = get_users_from_db()
    return render_template('users.html', users=users)
```

**简单、灵活、可控。**

### Tornado：异步先驱

后来我还用过 Tornado，它的特点是异步非阻塞：

```python
class UserHandler(tornado.web.RequestHandler):
    async def get(self):
        users = await self.db.query("SELECT * FROM users")
        self.render("users.html", users=users)
```

在 Python 还没有 `async/await` 的年代，Tornado 就实现了高并发。

## MVC 的局限

用了几年 MVC 框架后，我开始遇到新的问题。

### 问题 1：胖 Controller

理论上，Controller 应该很薄——只负责协调。但实际中：

```python
def create_order(request):
    # 验证用户
    user = get_user(request)
    if not user.is_active:
        return error("用户未激活")

    # 验证商品
    product = Product.objects.get(id=request.POST['product_id'])
    if product.stock < request.POST['quantity']:
        return error("库存不足")

    # 计算价格
    price = product.price * quantity
    if user.is_vip:
        price *= 0.9
    if has_coupon(user, request.POST.get('coupon')):
        price -= get_coupon_value(request.POST['coupon'])

    # 创建订单
    order = Order.objects.create(
        user=user,
        product=product,
        quantity=quantity,
        price=price
    )

    # 扣减库存
    product.stock -= quantity
    product.save()

    # 发送通知
    send_email(user.email, "订单创建成功")
    send_sms(user.phone, "您的订单已创建")

    # 记录日志
    log_order_created(order)

    return success(order)
```

**这个 Controller 有 50 行业务逻辑。** 它知道太多了：价格计算、库存管理、通知发送...

测试它？需要 mock 数据库、mock 邮件、mock 短信...

### 问题 2：贫血 Model

为了避免胖 Controller，有人把逻辑放到 Model 里。但 Model 变成了什么？

```python
class Order(models.Model):
    # 字段定义...

    def calculate_price(self):
        pass

    def validate_stock(self):
        pass

    def apply_discount(self):
        pass

    def send_notification(self):
        pass

    def update_inventory(self):
        pass
```

Model 变成了「上帝对象」，什么都做。

更常见的情况是：**Model 只有字段定义，没有行为。** 所有逻辑都在 Controller 或「Service 层」里。

这叫「贫血模型」（Anemic Domain Model）——对象只有数据，没有行为。虽然技术上仍是面向对象（有类和对象），但失去了 OOP 的核心优势：数据和行为的封装。业务逻辑散落在 Service 层，Model 退化为数据容器。

### 问题 3：业务逻辑散落

MVC 解决了「技术关注点」的分离：数据库、HTTP、HTML 分开了。

但**业务逻辑**呢？

「用户下单」的完整流程在哪里？
- `OrderController.create()` 里一部分
- `Order.save()` 里一部分
- `OrderService.process()` 里一部分
- 信号/钩子里一部分

MVC 没有告诉你：**业务逻辑应该放哪里。**

### 问题 4：Service 层的困惑

为了解决胖 Controller，很多项目引入了 Service 层：

```text
Controller → Service → Model
```

但 Service 应该怎么组织？

```text
services/
├── user_service.py      # 用户相关
├── order_service.py     # 订单相关
├── payment_service.py   # 支付相关
└── notification_service.py  # 通知相关
```

这只是按「技术功能」分类。当业务变复杂，你会发现：

- `create_order()` 应该在 `order_service` 还是 `payment_service`？
- 如果涉及多个 Service，应该在哪里协调？
- `OrderService` 和 `ProductService` 互相调用，循环依赖了...

**MVC + Service 层，解决了技术分层，但没有解决业务分层。**

## 一个真实的演进

让我展示一个真实项目的演进过程：

**第 1 个月：干净的 MVC**

```text
app/
├── models.py      # 所有模型
├── views.py       # 所有视图
└── templates/
```

**第 6 个月：拆分模块**

```text
app/
├── users/
│   ├── models.py
│   └── views.py
├── orders/
│   ├── models.py
│   └── views.py
└── products/
```

**第 12 个月：加入 Service**

```text
app/
├── users/
│   ├── models.py
│   ├── views.py
│   └── services.py
├── orders/
│   ├── models.py
│   ├── views.py
│   └── services.py
```

**第 18 个月：混乱开始**

```text
app/
├── users/
│   ├── models.py
│   ├── views.py
│   ├── services.py
│   ├── utils.py
│   └── helpers.py
├── orders/
│   ├── models.py
│   ├── views.py
│   ├── services.py
│   ├── calculators.py
│   └── validators.py
├── common/
│   ├── base_service.py
│   └── mixins.py
```

文件是分开了，但**依赖关系变成了蜘蛛网**。

`OrderService` 依赖 `UserService`、`ProductService`、`PaymentService`...
`UserService` 又依赖 `OrderService`（获取用户的订单）...

**MVC 告诉你怎么分层，但没告诉你怎么分模块。**

## MVC 的价值与边界

尽管有这些局限，MVC 仍然是一个巨大的进步。

### MVC 解决了什么

| 问题 | 解决方案 |
|------|---------|
| 代码混乱 | 强制分层（M-V-C） |
| 没有规范 | 约定优于配置 |
| 重复造轮子 | 框架提供通用功能 |
| 入门门槛高 | 降低了 Web 开发的门槛 |

### MVC 没解决什么

| 问题 | 现状 |
|------|------|
| 业务逻辑放哪 | Controller 或 Service，没有明确指导 |
| 复杂业务建模 | 倾向于贫血模型 |
| 模块边界 | 按技术分层，不按业务分层 |
| 大型项目组织 | 容易变成蜘蛛网依赖 |

## 总结

**MVC 的贡献**：
- 分离了技术关注点（数据库、HTTP、HTML）
- 「约定优于配置」降低了门槛
- 框架化让团队协作更容易

**MVC 的局限**：
- 只解决了「技术分层」，没解决「业务分层」
- 容易产生胖 Controller 或贫血 Model
- 大型项目容易陷入依赖混乱

**核心教训**：

> **MVC 解决了技术分层，但业务分层需要额外的设计。**

当业务足够简单时，MVC 足够了。但当业务变复杂，我们需要新的思路。

下一篇，我们来看 Elixir Phoenix 的 Context 设计——如何从「技术分层」走向「业务分层」。

---

*上一篇：[混沌时代：当代码没有架构](/posts/architecture-evolution-1-chaos/)*

*下一篇：[Context 之道：从技术分层到业务分层](/posts/architecture-evolution-3-context/)*

*本系列：*
1. *[混沌时代：当代码没有架构](/posts/architecture-evolution-1-chaos/)*
2. *MVC 启蒙：框架带来的秩序（本篇）*
3. *[Context 之道：从技术分层到业务分层](/posts/architecture-evolution-3-context/)*
4. *[DDD 觉醒：让代码说业务的语言](/posts/architecture-evolution-4-ddd/)*
5. *[边界的艺术：六边形与洋葱](/posts/architecture-evolution-5-hexagonal/)*
6. *[单体的边界：何时该拆？](/posts/architecture-evolution-6-modular-monolith/)*
7. *[微服务与云原生：分布式的代价](/posts/architecture-evolution-7-microservices/)*
8. *[没有银弹：架构决策的本质](/posts/architecture-evolution-8-no-silver-bullet/)*
