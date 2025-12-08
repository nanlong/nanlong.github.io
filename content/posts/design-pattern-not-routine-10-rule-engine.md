---
title: "模式不是套路（十）：规则引擎——让业务人员写代码"
date: 2025-12-10T11:30:00+08:00
draft: false
tags: ["Rust", "设计模式", "规则引擎", "解释器模式", "DSL"]
categories: ["技术"]
description: "促销规则天天变，每次改都要发版？解释器模式帮你构建 DSL，让业务人员自己配置规则。从简单条件到复杂表达式，看规则引擎如何炼成。"
series: ["模式不是套路"]
---

促销规则天天变，每次改都要发版？解释器模式帮你构建 DSL，让业务人员自己配置规则。从简单条件到复杂表达式，看规则引擎如何炼成。

<!--more-->

## 问题：改不完的业务规则

电商促销系统：

```rust
fn calculate_discount(order: &Order, user: &User) -> Decimal {
    // 新用户首单 8 折
    if user.is_new && order.is_first_order {
        return order.total * dec!(0.8);
    }

    // VIP 用户满 200 减 50
    if user.is_vip && order.total >= dec!(200) {
        return order.total - dec!(50);
    }

    // 双十一全场 9 折
    if is_double_eleven() {
        return order.total * dec!(0.9);
    }

    // 生日当天 85 折
    if user.is_birthday_today() {
        return order.total * dec!(0.85);
    }

    order.total
}
```

然后产品经理每周都来：

```text
第 1 周："新用户首单改成 7.5 折"
第 2 周："VIP 满减改成满 300 减 80"
第 3 周："加个限时秒杀，指定商品 5 折"
第 4 周："多个优惠可以叠加"
第 5 周："不同品类折扣不同"
第 6 周："加个阶梯满减：满 100 减 10，满 200 减 30，满 500 减 100"
```

每次改动都要：

```text
1. 开发修改代码
2. 测试验证
3. 提交审核
4. 发布上线

耗时：1-3 天
业务：等不及，竞品都上了
```

## 破局：规则可配置化

核心思路：**把业务规则从代码中抽离，变成可配置的数据**。

### 第一步：条件表达式

```rust
// 规则条件
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Condition {
    // 简单条件
    IsNewUser,
    IsVip,
    IsFirstOrder,
    IsBirthday,

    // 比较条件
    OrderTotalGte(Decimal),  // 订单金额 >= X
    OrderTotalLte(Decimal),  // 订单金额 <= X
    UserLevelEq(i32),        // 用户等级 = X
    ProductCategoryIn(Vec<String>),  // 商品分类在列表中

    // 时间条件
    DateRange { start: NaiveDate, end: NaiveDate },
    TimeRange { start: NaiveTime, end: NaiveTime },
    DayOfWeek(Vec<Weekday>),

    // 组合条件
    And(Vec<Condition>),
    Or(Vec<Condition>),
    Not(Box<Condition>),
}

impl Condition {
    pub fn evaluate(&self, ctx: &RuleContext) -> bool {
        match self {
            Condition::IsNewUser => ctx.user.is_new,
            Condition::IsVip => ctx.user.is_vip,
            Condition::IsFirstOrder => ctx.order.is_first_order,
            Condition::IsBirthday => ctx.user.is_birthday_today(),

            Condition::OrderTotalGte(amount) => ctx.order.total >= *amount,
            Condition::OrderTotalLte(amount) => ctx.order.total <= *amount,
            Condition::UserLevelEq(level) => ctx.user.level == *level,

            Condition::ProductCategoryIn(categories) => {
                ctx.order.items.iter().any(|item| categories.contains(&item.category))
            }

            Condition::DateRange { start, end } => {
                let today = Local::now().date_naive();
                today >= *start && today <= *end
            }

            Condition::TimeRange { start, end } => {
                let now = Local::now().time();
                now >= *start && now <= *end
            }

            Condition::DayOfWeek(days) => {
                days.contains(&Local::now().weekday())
            }

            Condition::And(conditions) => {
                conditions.iter().all(|c| c.evaluate(ctx))
            }

            Condition::Or(conditions) => {
                conditions.iter().any(|c| c.evaluate(ctx))
            }

            Condition::Not(condition) => !condition.evaluate(ctx),
        }
    }
}
```

### 第二步：动作表达式

```rust
// 规则动作
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Action {
    // 折扣
    PercentDiscount(Decimal),      // 打 X 折 (0.8 = 8折)
    FixedDiscount(Decimal),        // 固定减 X 元
    FixedPrice(Decimal),           // 固定价 X 元

    // 满减
    ThresholdDiscount {
        threshold: Decimal,
        discount: Decimal,
    },

    // 阶梯满减
    TieredDiscount(Vec<(Decimal, Decimal)>),  // [(threshold, discount), ...]

    // 赠品
    GiveGift { product_id: String, quantity: u32 },

    // 积分
    GivePoints(u64),

    // 组合动作
    All(Vec<Action>),        // 所有动作都执行
    Best(Vec<Action>),       // 选择最优惠的一个
}

impl Action {
    pub fn apply(&self, ctx: &RuleContext) -> ActionResult {
        match self {
            Action::PercentDiscount(rate) => {
                let discount = ctx.order.total * (Decimal::ONE - rate);
                ActionResult::Discount(discount)
            }

            Action::FixedDiscount(amount) => {
                ActionResult::Discount(*amount)
            }

            Action::ThresholdDiscount { threshold, discount } => {
                if ctx.order.total >= *threshold {
                    ActionResult::Discount(*discount)
                } else {
                    ActionResult::None
                }
            }

            Action::TieredDiscount(tiers) => {
                // 找到满足条件的最高档
                let discount = tiers.iter()
                    .filter(|(threshold, _)| ctx.order.total >= *threshold)
                    .map(|(_, discount)| *discount)
                    .max()
                    .unwrap_or(Decimal::ZERO);

                ActionResult::Discount(discount)
            }

            Action::GiveGift { product_id, quantity } => {
                ActionResult::Gift {
                    product_id: product_id.clone(),
                    quantity: *quantity,
                }
            }

            Action::GivePoints(points) => {
                ActionResult::Points(*points)
            }

            Action::All(actions) => {
                let results: Vec<_> = actions.iter()
                    .map(|a| a.apply(ctx))
                    .collect();
                ActionResult::Multiple(results)
            }

            Action::Best(actions) => {
                actions.iter()
                    .map(|a| a.apply(ctx))
                    .max_by(|a, b| a.discount_value().cmp(&b.discount_value()))
                    .unwrap_or(ActionResult::None)
            }
        }
    }
}

#[derive(Debug, Clone)]
pub enum ActionResult {
    None,
    Discount(Decimal),
    Gift { product_id: String, quantity: u32 },
    Points(u64),
    Multiple(Vec<ActionResult>),
}

impl ActionResult {
    pub fn discount_value(&self) -> Decimal {
        match self {
            ActionResult::Discount(d) => *d,
            ActionResult::Multiple(results) => {
                results.iter().map(|r| r.discount_value()).sum()
            }
            _ => Decimal::ZERO,
        }
    }
}
```

### 第三步：完整规则

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Rule {
    pub id: String,
    pub name: String,
    pub description: String,
    pub priority: i32,          // 优先级，数字越大优先级越高
    pub enabled: bool,
    pub condition_str: String,  // DSL 条件表达式字符串
    pub action: Action,
    pub exclusive: bool,        // 是否排他（不与其他规则叠加）
    pub valid_from: Option<DateTime<Utc>>,
    pub valid_to: Option<DateTime<Utc>>,
}

impl Rule {
    pub fn is_valid(&self) -> bool {
        if !self.enabled {
            return false;
        }

        let now = Utc::now();

        if let Some(from) = self.valid_from {
            if now < from {
                return false;
            }
        }

        if let Some(to) = self.valid_to {
            if now > to {
                return false;
            }
        }

        true
    }

    // 注意：condition_str 需要先编译为 Expr 才能求值
    // 实际求值通过 RuleEngine 进行（见后文）
}
```

## 演进：解释器模式

业务人员希望用更自然的方式写规则：

```text
# 新用户首单优惠
condition: user.is_new AND order.is_first_order
action: discount 20%

# VIP 满减
condition: user.is_vip AND order.total >= 200
action: subtract 50

# 双十一活动
condition: date BETWEEN "2024-11-11" AND "2024-11-11" AND time BETWEEN "00:00" AND "23:59"
action: discount 10%
```

**解释器模式**的核心是：**给定一个语言，定义它的文法表示，并定义一个解释器**。

### 词法分析

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum Token {
    // 关键字
    And,
    Or,
    Not,
    True,
    False,

    // 比较符
    Eq,       // ==
    Ne,       // !=
    Gt,       // >
    Ge,       // >=
    Lt,       // <
    Le,       // <=

    // 字面量
    Ident(String),
    Number(Decimal),
    String(String),

    // 标点
    LParen,
    RParen,
    Dot,
    Comma,

    // 结束
    Eof,
}

pub struct Lexer {
    input: Vec<char>,
    pos: usize,
}

impl Lexer {
    pub fn new(input: &str) -> Self {
        Self {
            input: input.chars().collect(),
            pos: 0,
        }
    }

    pub fn tokenize(&mut self) -> Result<Vec<Token>, LexError> {
        let mut tokens = Vec::new();

        while self.pos < self.input.len() {
            self.skip_whitespace();

            if self.pos >= self.input.len() {
                break;
            }

            let token = self.next_token()?;
            tokens.push(token);
        }

        tokens.push(Token::Eof);
        Ok(tokens)
    }

    fn next_token(&mut self) -> Result<Token, LexError> {
        let ch = self.input[self.pos];

        match ch {
            '(' => { self.pos += 1; Ok(Token::LParen) }
            ')' => { self.pos += 1; Ok(Token::RParen) }
            '.' => { self.pos += 1; Ok(Token::Dot) }
            ',' => { self.pos += 1; Ok(Token::Comma) }
            '=' => {
                self.pos += 1;
                if self.peek() == Some('=') {
                    self.pos += 1;
                }
                Ok(Token::Eq)
            }
            '!' => {
                self.pos += 1;
                if self.peek() == Some('=') {
                    self.pos += 1;
                    Ok(Token::Ne)
                } else {
                    Ok(Token::Not)
                }
            }
            '>' => {
                self.pos += 1;
                if self.peek() == Some('=') {
                    self.pos += 1;
                    Ok(Token::Ge)
                } else {
                    Ok(Token::Gt)
                }
            }
            '<' => {
                self.pos += 1;
                if self.peek() == Some('=') {
                    self.pos += 1;
                    Ok(Token::Le)
                } else {
                    Ok(Token::Lt)
                }
            }
            '"' => self.read_string(),
            _ if ch.is_ascii_digit() => self.read_number(),
            _ if ch.is_ascii_alphabetic() || ch == '_' => self.read_ident(),
            _ => Err(LexError::UnexpectedChar(ch)),
        }
    }

    fn read_ident(&mut self) -> Result<Token, LexError> {
        let start = self.pos;
        while self.pos < self.input.len() {
            let ch = self.input[self.pos];
            if ch.is_ascii_alphanumeric() || ch == '_' {
                self.pos += 1;
            } else {
                break;
            }
        }

        let ident: String = self.input[start..self.pos].iter().collect();

        let token = match ident.to_lowercase().as_str() {
            "and" => Token::And,
            "or" => Token::Or,
            "not" => Token::Not,
            "true" => Token::True,
            "false" => Token::False,
            _ => Token::Ident(ident),
        };

        Ok(token)
    }

    fn read_number(&mut self) -> Result<Token, LexError> {
        let start = self.pos;
        while self.pos < self.input.len() {
            let ch = self.input[self.pos];
            if ch.is_ascii_digit() || ch == '.' {
                self.pos += 1;
            } else {
                break;
            }
        }

        let num_str: String = self.input[start..self.pos].iter().collect();
        let num = Decimal::from_str(&num_str)
            .map_err(|_| LexError::InvalidNumber(num_str))?;

        Ok(Token::Number(num))
    }

    fn read_string(&mut self) -> Result<Token, LexError> {
        self.pos += 1;  // skip opening quote
        let start = self.pos;

        while self.pos < self.input.len() && self.input[self.pos] != '"' {
            self.pos += 1;
        }

        let s: String = self.input[start..self.pos].iter().collect();
        self.pos += 1;  // skip closing quote

        Ok(Token::String(s))
    }

    fn skip_whitespace(&mut self) {
        while self.pos < self.input.len() && self.input[self.pos].is_whitespace() {
            self.pos += 1;
        }
    }

    fn peek(&self) -> Option<char> {
        self.input.get(self.pos).copied()
    }
}
```

### 语法分析

```rust
// AST 节点
#[derive(Debug, Clone)]
pub enum Expr {
    Bool(bool),
    Number(Decimal),
    String(String),
    Variable(Vec<String>),  // user.is_vip -> ["user", "is_vip"]

    // 比较
    Eq(Box<Expr>, Box<Expr>),
    Ne(Box<Expr>, Box<Expr>),
    Gt(Box<Expr>, Box<Expr>),
    Ge(Box<Expr>, Box<Expr>),
    Lt(Box<Expr>, Box<Expr>),
    Le(Box<Expr>, Box<Expr>),

    // 逻辑
    And(Box<Expr>, Box<Expr>),
    Or(Box<Expr>, Box<Expr>),
    Not(Box<Expr>),

    // 函数调用
    Call(String, Vec<Expr>),
}

pub struct Parser {
    tokens: Vec<Token>,
    pos: usize,
}

impl Parser {
    pub fn new(tokens: Vec<Token>) -> Self {
        Self { tokens, pos: 0 }
    }

    pub fn parse(&mut self) -> Result<Expr, ParseError> {
        self.parse_or()
    }

    fn parse_or(&mut self) -> Result<Expr, ParseError> {
        let mut left = self.parse_and()?;

        while self.current() == &Token::Or {
            self.advance();
            let right = self.parse_and()?;
            left = Expr::Or(Box::new(left), Box::new(right));
        }

        Ok(left)
    }

    fn parse_and(&mut self) -> Result<Expr, ParseError> {
        let mut left = self.parse_not()?;

        while self.current() == &Token::And {
            self.advance();
            let right = self.parse_not()?;
            left = Expr::And(Box::new(left), Box::new(right));
        }

        Ok(left)
    }

    fn parse_not(&mut self) -> Result<Expr, ParseError> {
        if self.current() == &Token::Not {
            self.advance();
            let expr = self.parse_not()?;
            return Ok(Expr::Not(Box::new(expr)));
        }

        self.parse_comparison()
    }

    fn parse_comparison(&mut self) -> Result<Expr, ParseError> {
        let left = self.parse_primary()?;

        let op = match self.current() {
            Token::Eq => Some(|l, r| Expr::Eq(Box::new(l), Box::new(r))),
            Token::Ne => Some(|l, r| Expr::Ne(Box::new(l), Box::new(r))),
            Token::Gt => Some(|l, r| Expr::Gt(Box::new(l), Box::new(r))),
            Token::Ge => Some(|l, r| Expr::Ge(Box::new(l), Box::new(r))),
            Token::Lt => Some(|l, r| Expr::Lt(Box::new(l), Box::new(r))),
            Token::Le => Some(|l, r| Expr::Le(Box::new(l), Box::new(r))),
            _ => None,
        };

        if let Some(op) = op {
            self.advance();
            let right = self.parse_primary()?;
            return Ok(op(left, right));
        }

        Ok(left)
    }

    fn parse_primary(&mut self) -> Result<Expr, ParseError> {
        match self.current().clone() {
            Token::True => {
                self.advance();
                Ok(Expr::Bool(true))
            }
            Token::False => {
                self.advance();
                Ok(Expr::Bool(false))
            }
            Token::Number(n) => {
                self.advance();
                Ok(Expr::Number(n))
            }
            Token::String(s) => {
                self.advance();
                Ok(Expr::String(s))
            }
            Token::Ident(name) => {
                self.advance();
                let mut path = vec![name];

                // 处理属性访问: user.is_vip
                while self.current() == &Token::Dot {
                    self.advance();
                    if let Token::Ident(prop) = self.current().clone() {
                        self.advance();
                        path.push(prop);
                    } else {
                        return Err(ParseError::ExpectedIdent);
                    }
                }

                // 处理函数调用: contains(...)
                if self.current() == &Token::LParen {
                    self.advance();
                    let mut args = Vec::new();

                    if self.current() != &Token::RParen {
                        args.push(self.parse()?);
                        while self.current() == &Token::Comma {
                            self.advance();
                            args.push(self.parse()?);
                        }
                    }

                    self.expect(Token::RParen)?;
                    return Ok(Expr::Call(path.join("."), args));
                }

                Ok(Expr::Variable(path))
            }
            Token::LParen => {
                self.advance();
                let expr = self.parse()?;
                self.expect(Token::RParen)?;
                Ok(expr)
            }
            _ => Err(ParseError::UnexpectedToken),
        }
    }

    fn current(&self) -> &Token {
        &self.tokens[self.pos]
    }

    fn advance(&mut self) {
        if self.pos < self.tokens.len() - 1 {
            self.pos += 1;
        }
    }

    fn expect(&mut self, token: Token) -> Result<(), ParseError> {
        if self.current() == &token {
            self.advance();
            Ok(())
        } else {
            Err(ParseError::ExpectedToken(token))
        }
    }
}
```

### 求值

```rust
pub struct Evaluator<'a> {
    ctx: &'a RuleContext,
}

impl<'a> Evaluator<'a> {
    pub fn new(ctx: &'a RuleContext) -> Self {
        Self { ctx }
    }

    pub fn evaluate(&self, expr: &Expr) -> Result<Value, EvalError> {
        match expr {
            Expr::Bool(b) => Ok(Value::Bool(*b)),
            Expr::Number(n) => Ok(Value::Number(*n)),
            Expr::String(s) => Ok(Value::String(s.clone())),

            Expr::Variable(path) => self.get_variable(path),

            Expr::Eq(l, r) => {
                let lv = self.evaluate(l)?;
                let rv = self.evaluate(r)?;
                Ok(Value::Bool(lv == rv))
            }
            Expr::Ne(l, r) => {
                let lv = self.evaluate(l)?;
                let rv = self.evaluate(r)?;
                Ok(Value::Bool(lv != rv))
            }
            Expr::Gt(l, r) => {
                let lv = self.evaluate(l)?.as_number()?;
                let rv = self.evaluate(r)?.as_number()?;
                Ok(Value::Bool(lv > rv))
            }
            Expr::Ge(l, r) => {
                let lv = self.evaluate(l)?.as_number()?;
                let rv = self.evaluate(r)?.as_number()?;
                Ok(Value::Bool(lv >= rv))
            }
            Expr::Lt(l, r) => {
                let lv = self.evaluate(l)?.as_number()?;
                let rv = self.evaluate(r)?.as_number()?;
                Ok(Value::Bool(lv < rv))
            }
            Expr::Le(l, r) => {
                let lv = self.evaluate(l)?.as_number()?;
                let rv = self.evaluate(r)?.as_number()?;
                Ok(Value::Bool(lv <= rv))
            }

            Expr::And(l, r) => {
                let lv = self.evaluate(l)?.as_bool()?;
                if !lv {
                    return Ok(Value::Bool(false));  // 短路求值
                }
                let rv = self.evaluate(r)?.as_bool()?;
                Ok(Value::Bool(rv))
            }
            Expr::Or(l, r) => {
                let lv = self.evaluate(l)?.as_bool()?;
                if lv {
                    return Ok(Value::Bool(true));  // 短路求值
                }
                let rv = self.evaluate(r)?.as_bool()?;
                Ok(Value::Bool(rv))
            }
            Expr::Not(e) => {
                let v = self.evaluate(e)?.as_bool()?;
                Ok(Value::Bool(!v))
            }

            Expr::Call(name, args) => self.call_function(name, args),
        }
    }

    fn get_variable(&self, path: &[String]) -> Result<Value, EvalError> {
        match path.as_slice() {
            ["user", "is_new"] => Ok(Value::Bool(self.ctx.user.is_new)),
            ["user", "is_vip"] => Ok(Value::Bool(self.ctx.user.is_vip)),
            ["user", "level"] => Ok(Value::Number(self.ctx.user.level.into())),
            ["order", "total"] => Ok(Value::Number(self.ctx.order.total)),
            ["order", "is_first_order"] => Ok(Value::Bool(self.ctx.order.is_first_order)),
            ["order", "item_count"] => Ok(Value::Number(self.ctx.order.items.len().into())),
            _ => Err(EvalError::UnknownVariable(path.join("."))),
        }
    }

    fn call_function(&self, name: &str, args: &[Expr]) -> Result<Value, EvalError> {
        match name {
            "contains" => {
                if args.len() != 2 {
                    return Err(EvalError::WrongArgCount);
                }
                let list = self.evaluate(&args[0])?.as_list()?;
                let item = self.evaluate(&args[1])?;
                Ok(Value::Bool(list.contains(&item)))
            }
            "length" => {
                if args.len() != 1 {
                    return Err(EvalError::WrongArgCount);
                }
                let list = self.evaluate(&args[0])?.as_list()?;
                Ok(Value::Number(list.len().into()))
            }
            _ => Err(EvalError::UnknownFunction(name.to_string())),
        }
    }
}

#[derive(Debug, Clone, PartialEq)]
pub enum Value {
    Bool(bool),
    Number(Decimal),
    String(String),
    List(Vec<Value>),
}

impl Value {
    pub fn as_bool(&self) -> Result<bool, EvalError> {
        match self {
            Value::Bool(b) => Ok(*b),
            _ => Err(EvalError::TypeMismatch),
        }
    }

    pub fn as_number(&self) -> Result<Decimal, EvalError> {
        match self {
            Value::Number(n) => Ok(*n),
            _ => Err(EvalError::TypeMismatch),
        }
    }

    pub fn as_list(&self) -> Result<&Vec<Value>, EvalError> {
        match self {
            Value::List(l) => Ok(l),
            _ => Err(EvalError::TypeMismatch),
        }
    }
}
```

## 进阶：规则引擎

把所有组件组装成完整的规则引擎：

```rust
pub struct RuleEngine {
    rules: Vec<CompiledRule>,
}

pub struct CompiledRule {
    pub rule: Rule,
    pub condition_expr: Expr,
}

impl RuleEngine {
    pub fn new() -> Self {
        Self { rules: Vec::new() }
    }

    pub fn add_rule(&mut self, rule: Rule) -> Result<(), CompileError> {
        // 编译条件表达式
        let mut lexer = Lexer::new(&rule.condition_str);
        let tokens = lexer.tokenize()?;
        let mut parser = Parser::new(tokens);
        let condition_expr = parser.parse()?;

        self.rules.push(CompiledRule { rule, condition_expr });

        // 按优先级排序
        self.rules.sort_by(|a, b| b.rule.priority.cmp(&a.rule.priority));

        Ok(())
    }

    pub fn evaluate(&self, ctx: &RuleContext) -> Vec<ActionResult> {
        let evaluator = Evaluator::new(ctx);
        let mut results = Vec::new();
        let mut exclusive_matched = false;

        for compiled in &self.rules {
            if !compiled.rule.is_valid() {
                continue;
            }

            if exclusive_matched && compiled.rule.exclusive {
                continue;
            }

            let condition_result = evaluator.evaluate(&compiled.condition_expr);

            if let Ok(Value::Bool(true)) = condition_result {
                let action_result = compiled.rule.action.apply(ctx);
                results.push(action_result);

                if compiled.rule.exclusive {
                    exclusive_matched = true;
                }
            }
        }

        results
    }

    pub fn find_best_discount(&self, ctx: &RuleContext) -> Decimal {
        self.evaluate(ctx)
            .iter()
            .map(|r| r.discount_value())
            .max()
            .unwrap_or(Decimal::ZERO)
    }
}
```

使用示例：

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut engine = RuleEngine::new();

    // 从配置加载规则
    engine.add_rule(Rule {
        id: "new_user_discount".to_string(),
        name: "新用户首单优惠".to_string(),
        description: "新用户首单享受 8 折优惠".to_string(),
        priority: 100,
        enabled: true,
        condition_str: "user.is_new AND order.is_first_order".to_string(),
        action: Action::PercentDiscount(dec!(0.8)),
        exclusive: false,
        valid_from: None,
        valid_to: None,
    })?;

    engine.add_rule(Rule {
        id: "vip_threshold".to_string(),
        name: "VIP 满减".to_string(),
        description: "VIP 用户满 200 减 50".to_string(),
        priority: 90,
        enabled: true,
        condition_str: "user.is_vip AND order.total >= 200".to_string(),
        action: Action::ThresholdDiscount {
            threshold: dec!(200),
            discount: dec!(50),
        },
        exclusive: false,
        valid_from: None,
        valid_to: None,
    })?;

    // 评估
    let ctx = RuleContext {
        user: User {
            is_new: true,
            is_vip: false,
            level: 1,
            // ...
        },
        order: Order {
            total: dec!(300),
            is_first_order: true,
            items: vec![],
            // ...
        },
    };

    let results = engine.evaluate(&ctx);
    println!("Applied rules: {:?}", results);

    let best_discount = engine.find_best_discount(&ctx);
    println!("Best discount: {}", best_discount);

    Ok(())
}
```

## 陷阱：别掉进这些坑

### 陷阱一：规则冲突

```yaml
# 规则 1：新用户 8 折
# 规则 2：VIP 85 折
# 问题：新用户同时是 VIP 怎么办？
```

解决方案：

```rust
// ✅ 方案 1：定义清晰的优先级
engine.add_rule(Rule {
    priority: 100,  // 新用户优先
    exclusive: true,  // 排他，不与其他叠加
    // ...
});

// ✅ 方案 2：取最优
Action::Best(vec![
    Action::PercentDiscount(dec!(0.8)),
    Action::PercentDiscount(dec!(0.85)),
])

// ✅ 方案 3：业务人员指定冲突处理
Rule {
    conflict_resolution: ConflictResolution::TakeFirst,
    // ConflictResolution::TakeAll,
    // ConflictResolution::TakeBest,
}
```

### 陷阱二：规则循环

```yaml
# 规则 A：如果订单总额 > 100，给 10% 折扣
# 规则 B：如果有折扣，订单总额 = 原价 - 折扣
# 问题：折扣改变总额，总额影响折扣...
```

解决方案：

```rust
// ✅ 规则只读取原始数据，不读取中间结果
pub struct RuleContext {
    pub original_order: Order,  // 原始订单数据
    // 不包含计算中的中间结果
}

// 或者限制求值轮数
impl RuleEngine {
    fn evaluate_with_limit(&self, ctx: &RuleContext, max_rounds: u32) -> Vec<ActionResult> {
        for _ in 0..max_rounds {
            // 评估一轮
        }
    }
}
```

### 陷阱三：性能问题

```rust
// ❌ 每次请求都编译规则
fn handle_request(order: &Order) {
    let rule = parse_rule(rule_config);  // 每次都解析！
    rule.evaluate(ctx);
}
```

解决方案：

```rust
// ✅ 预编译规则
pub struct RuleEngine {
    rules: Vec<CompiledRule>,  // 编译后的规则
}

impl RuleEngine {
    pub fn compile_all(&mut self, configs: Vec<RuleConfig>) {
        for config in configs {
            let compiled = self.compile(&config);
            self.rules.push(compiled);
        }
    }

    // 热更新时增量编译
    pub fn update_rule(&mut self, config: RuleConfig) {
        let compiled = self.compile(&config);
        // 替换对应规则
    }
}
```

### 陷阱四：调试困难

```text
用户反馈：优惠没生效
开发：哪条规则没匹配？为什么？
```

解决方案：

```rust
// ✅ 添加调试模式
impl RuleEngine {
    pub fn evaluate_with_trace(&self, ctx: &RuleContext) -> EvaluationTrace {
        let mut trace = EvaluationTrace::new();

        for rule in &self.rules {
            let condition_result = self.evaluate_condition(rule, ctx);
            trace.add_rule_evaluation(RuleEvaluation {
                rule_id: rule.id.clone(),
                rule_name: rule.name.clone(),
                condition_matched: condition_result.is_ok_and(|v| v),
                reason: condition_result.err().map(|e| e.to_string()),
            });
        }

        trace
    }
}

#[derive(Debug)]
pub struct EvaluationTrace {
    pub evaluations: Vec<RuleEvaluation>,
}

impl EvaluationTrace {
    pub fn print(&self) {
        for eval in &self.evaluations {
            println!("[{}] {} -> {}",
                eval.rule_id,
                eval.rule_name,
                if eval.condition_matched { "✓ 匹配" } else { "✗ 不匹配" }
            );
            if let Some(reason) = &eval.reason {
                println!("    原因: {}", reason);
            }
        }
    }
}
```

## 小结

规则引擎涉及两个核心模式：

| 模式 | 解决的问题 | Rust 实现 |
|------|-----------|----------|
| 解释器 | 解析和执行 DSL | Lexer + Parser + Evaluator |
| 策略 | 可替换的动作 | enum Action + trait |

关键洞见：

1. **DSL 比 JSON 配置更灵活**，业务人员更容易理解
2. **编译时解析**，运行时只做求值，性能更好
3. **提供调试工具**，让规则可追踪
4. **处理好冲突和循环**，避免业务逻辑错误

下一篇是本系列的终篇：**反模式与断舍离**。我们会讨论什么时候不该用模式，以及设计模式的常见误用。

---

> **系列导航**
>
> - 第一篇：Rust 让设计模式返璞归真
> - 第二篇：配置系统——从硬编码到热加载
> - 第三篇：插件架构——让系统长出翅膀
> - 第四篇：数据管道——像乐高一样组装逻辑
> - 第五篇：状态流转——订单不只是 CRUD
> - 第六篇：事件驱动——模块间的优雅对话
> - 第七篇：外部集成——与第三方和平共处
> - 第八篇：框架设计——把控制权还给用户
> - 第九篇：树形世界——递归结构的驯服术
> - **第十篇：规则引擎——让业务人员写代码**（本文）
> - 第十一篇：终篇——反模式与断舍离
