---
title: "模式不是套路（九）：树形世界——递归结构的驯服术"
date: 2025-12-10T11:20:00+08:00
draft: false
tags: ["Rust", "设计模式", "树形结构", "组合模式", "访问者模式", "迭代器模式"]
categories: ["技术"]
description: "文件系统、组织架构、商品分类、权限树……树形结构无处不在，但处理起来总是一团乱麻。组合、访问者、迭代器、享元四种模式，让你优雅地驯服递归之兽。"
series: ["模式不是套路"]
---

文件系统、组织架构、商品分类、权限树……树形结构无处不在，但处理起来总是一团乱麻。组合、访问者、迭代器、享元四种模式，让你优雅地驯服递归之兽。

<!--more-->

## 问题：到处都是树

后台管理系统，需要展示菜单：

```text
├── 系统管理
│   ├── 用户管理
│   │   ├── 用户列表
│   │   └── 用户组
│   ├── 角色管理
│   └── 权限管理
├── 商品管理
│   ├── 商品分类
│   └── 商品列表
└── 订单管理
    ├── 订单列表
    └── 退款管理
```

最直接的实现：

```rust
struct Menu {
    id: String,
    name: String,
    children: Vec<Menu>,  // 递归定义
}
```

然后问题来了：

```text
需求 1：计算菜单总数
需求 2：查找某个菜单
需求 3：获取某菜单的所有父级
需求 4：渲染成 HTML
需求 5：导出成 JSON
需求 6：按权限过滤
需求 7：支持拖拽排序
```

每个需求都要写一堆递归代码，而且逻辑散落各处。

## 破局：组合模式

**组合模式**的核心是：**将对象组合成树形结构，使客户端对单个对象和组合对象的使用具有一致性**。

### 统一接口

```rust
// 树节点的统一接口
pub trait TreeNode {
    fn id(&self) -> &str;
    fn name(&self) -> &str;
    fn children(&self) -> &[Box<dyn TreeNode>];
    fn is_leaf(&self) -> bool {
        self.children().is_empty()
    }

    // 统一的操作方法
    fn count(&self) -> usize {
        1 + self.children().iter().map(|c| c.count()).sum::<usize>()
    }

    fn find(&self, id: &str) -> Option<&dyn TreeNode> {
        if self.id() == id {
            return Some(self);
        }
        for child in self.children() {
            if let Some(found) = child.find(id) {
                return Some(found);
            }
        }
        None
    }

    fn depth(&self) -> usize {
        if self.is_leaf() {
            0
        } else {
            1 + self.children().iter().map(|c| c.depth()).max().unwrap_or(0)
        }
    }
}
```

但这种方式有问题：**所有操作都定义在 TreeNode 上**，随着需求增加，trait 会越来越臃肿。

### Rust 的礼物：enum 递归

Rust 更惯用的方式是用 enum：

```rust
#[derive(Debug, Clone)]
pub enum MenuItem {
    Leaf {
        id: String,
        name: String,
        path: String,
    },
    Group {
        id: String,
        name: String,
        children: Vec<MenuItem>,
    },
}

impl MenuItem {
    pub fn id(&self) -> &str {
        match self {
            MenuItem::Leaf { id, .. } => id,
            MenuItem::Group { id, .. } => id,
        }
    }

    pub fn name(&self) -> &str {
        match self {
            MenuItem::Leaf { name, .. } => name,
            MenuItem::Group { name, .. } => name,
        }
    }

    pub fn children(&self) -> &[MenuItem] {
        match self {
            MenuItem::Leaf { .. } => &[],
            MenuItem::Group { children, .. } => children,
        }
    }

    pub fn is_leaf(&self) -> bool {
        matches!(self, MenuItem::Leaf { .. })
    }
}
```

## 演进：访问者模式

需求越来越多，每次都在 MenuItem 上加方法不现实。

**访问者模式**的核心是：**将数据结构与对它的操作分离**。

```rust
// 访问者 trait
pub trait MenuVisitor {
    fn visit_leaf(&mut self, id: &str, name: &str, path: &str);
    fn visit_group_start(&mut self, id: &str, name: &str);
    fn visit_group_end(&mut self, id: &str, name: &str);
}

impl MenuItem {
    pub fn accept<V: MenuVisitor>(&self, visitor: &mut V) {
        match self {
            MenuItem::Leaf { id, name, path } => {
                visitor.visit_leaf(id, name, path);
            }
            MenuItem::Group { id, name, children } => {
                visitor.visit_group_start(id, name);
                for child in children {
                    child.accept(visitor);
                }
                visitor.visit_group_end(id, name);
            }
        }
    }
}
```

不同的操作用不同的访问者实现：

```rust
// 计数访问者
struct CountVisitor {
    count: usize,
}

impl MenuVisitor for CountVisitor {
    fn visit_leaf(&mut self, _id: &str, _name: &str, _path: &str) {
        self.count += 1;
    }

    fn visit_group_start(&mut self, _id: &str, _name: &str) {
        self.count += 1;
    }

    fn visit_group_end(&mut self, _id: &str, _name: &str) {}
}

// HTML 渲染访问者
struct HtmlVisitor {
    html: String,
    indent: usize,
}

impl HtmlVisitor {
    fn new() -> Self {
        Self { html: String::new(), indent: 0 }
    }

    fn indent_str(&self) -> String {
        "  ".repeat(self.indent)
    }
}

impl MenuVisitor for HtmlVisitor {
    fn visit_leaf(&mut self, _id: &str, name: &str, path: &str) {
        self.html.push_str(&format!(
            "{}<li><a href=\"{}\">{}</a></li>\n",
            self.indent_str(), path, name
        ));
    }

    fn visit_group_start(&mut self, _id: &str, name: &str) {
        self.html.push_str(&format!(
            "{}<li>{}\n{}<ul>\n",
            self.indent_str(), name, self.indent_str()
        ));
        self.indent += 1;
    }

    fn visit_group_end(&mut self, _id: &str, _name: &str) {
        self.indent -= 1;
        self.html.push_str(&format!(
            "{}</ul>\n{}</li>\n",
            self.indent_str(), self.indent_str()
        ));
    }
}

// JSON 导出访问者
struct JsonVisitor {
    stack: Vec<serde_json::Value>,
}

impl JsonVisitor {
    fn new() -> Self {
        Self { stack: vec![serde_json::json!([])] }
    }

    fn result(mut self) -> serde_json::Value {
        self.stack.pop().unwrap()
    }
}

impl MenuVisitor for JsonVisitor {
    fn visit_leaf(&mut self, id: &str, name: &str, path: &str) {
        let leaf = serde_json::json!({
            "id": id,
            "name": name,
            "path": path,
            "type": "leaf"
        });

        if let Some(arr) = self.stack.last_mut().and_then(|v| v.as_array_mut()) {
            arr.push(leaf);
        }
    }

    fn visit_group_start(&mut self, id: &str, name: &str) {
        // 推入新的 children 数组
        self.stack.push(serde_json::json!([]));
    }

    fn visit_group_end(&mut self, id: &str, name: &str) {
        let children = self.stack.pop().unwrap();
        let group = serde_json::json!({
            "id": id,
            "name": name,
            "type": "group",
            "children": children
        });

        if let Some(arr) = self.stack.last_mut().and_then(|v| v.as_array_mut()) {
            arr.push(group);
        }
    }
}
```

使用：

```rust
// 计数
let mut counter = CountVisitor { count: 0 };
menu.accept(&mut counter);
println!("Total items: {}", counter.count);

// 渲染 HTML
let mut html_visitor = HtmlVisitor::new();
menu.accept(&mut html_visitor);
println!("{}", html_visitor.html);

// 导出 JSON
let mut json_visitor = JsonVisitor::new();
menu.accept(&mut json_visitor);
println!("{}", json_visitor.result());
```

### 权限过滤访问者

```rust
struct FilterVisitor {
    permissions: HashSet<String>,
    result: Vec<MenuItem>,
    stack: Vec<Vec<MenuItem>>,  // 用于构建过滤后的树
}

impl FilterVisitor {
    fn new(permissions: HashSet<String>) -> Self {
        Self {
            permissions,
            result: vec![],
            stack: vec![vec![]],
        }
    }

    fn result(mut self) -> Vec<MenuItem> {
        self.stack.pop().unwrap()
    }
}

impl MenuVisitor for FilterVisitor {
    fn visit_leaf(&mut self, id: &str, name: &str, path: &str) {
        if self.permissions.contains(id) {
            let item = MenuItem::Leaf {
                id: id.to_string(),
                name: name.to_string(),
                path: path.to_string(),
            };
            self.stack.last_mut().unwrap().push(item);
        }
    }

    fn visit_group_start(&mut self, _id: &str, _name: &str) {
        self.stack.push(vec![]);
    }

    fn visit_group_end(&mut self, id: &str, name: &str) {
        let children = self.stack.pop().unwrap();

        // 只有当有可见的子节点时，才保留组
        if !children.is_empty() {
            let group = MenuItem::Group {
                id: id.to_string(),
                name: name.to_string(),
                children,
            };
            self.stack.last_mut().unwrap().push(group);
        }
    }
}

// 使用
let permissions: HashSet<String> = ["user_list", "order_list"].iter().map(|s| s.to_string()).collect();
let mut filter = FilterVisitor::new(permissions);
menu.accept(&mut filter);
let filtered_menu = filter.result();
```

## 进阶：迭代器模式

有时候你只想遍历树，不需要访问者那么复杂。

**迭代器模式**的核心是：**提供一种方法顺序访问聚合对象中的各个元素**。

### 深度优先迭代器

```rust
pub struct DfsIterator<'a> {
    stack: Vec<&'a MenuItem>,
}

impl<'a> DfsIterator<'a> {
    pub fn new(root: &'a MenuItem) -> Self {
        Self { stack: vec![root] }
    }
}

impl<'a> Iterator for DfsIterator<'a> {
    type Item = &'a MenuItem;

    fn next(&mut self) -> Option<Self::Item> {
        let node = self.stack.pop()?;

        // 将子节点逆序压栈（这样弹出时是正序）
        for child in node.children().iter().rev() {
            self.stack.push(child);
        }

        Some(node)
    }
}

impl MenuItem {
    pub fn dfs_iter(&self) -> DfsIterator {
        DfsIterator::new(self)
    }
}

// 使用
for item in menu.dfs_iter() {
    println!("{}: {}", item.id(), item.name());
}

// 配合标准库方法
let leaf_count = menu.dfs_iter().filter(|m| m.is_leaf()).count();
let all_names: Vec<_> = menu.dfs_iter().map(|m| m.name()).collect();
```

### 广度优先迭代器

```rust
use std::collections::VecDeque;

pub struct BfsIterator<'a> {
    queue: VecDeque<&'a MenuItem>,
}

impl<'a> BfsIterator<'a> {
    pub fn new(root: &'a MenuItem) -> Self {
        let mut queue = VecDeque::new();
        queue.push_back(root);
        Self { queue }
    }
}

impl<'a> Iterator for BfsIterator<'a> {
    type Item = &'a MenuItem;

    fn next(&mut self) -> Option<Self::Item> {
        let node = self.queue.pop_front()?;

        for child in node.children() {
            self.queue.push_back(child);
        }

        Some(node)
    }
}

impl MenuItem {
    pub fn bfs_iter(&self) -> BfsIterator {
        BfsIterator::new(self)
    }
}
```

### 带路径的迭代器

有时候需要知道当前节点的路径：

```rust
pub struct PathIterator<'a> {
    stack: Vec<(&'a MenuItem, Vec<String>)>,
}

impl<'a> PathIterator<'a> {
    pub fn new(root: &'a MenuItem) -> Self {
        Self {
            stack: vec![(root, vec![root.name().to_string()])],
        }
    }
}

impl<'a> Iterator for PathIterator<'a> {
    type Item = (&'a MenuItem, Vec<String>);

    fn next(&mut self) -> Option<Self::Item> {
        let (node, path) = self.stack.pop()?;

        for child in node.children().iter().rev() {
            let mut child_path = path.clone();
            child_path.push(child.name().to_string());
            self.stack.push((child, child_path));
        }

        Some((node, path))
    }
}

impl MenuItem {
    pub fn path_iter(&self) -> PathIterator {
        PathIterator::new(self)
    }
}

// 使用
for (item, path) in menu.path_iter() {
    println!("{}: {}", path.join(" > "), item.name());
}
// 输出：
// 系统管理: 系统管理
// 系统管理 > 用户管理: 用户管理
// 系统管理 > 用户管理 > 用户列表: 用户列表
// ...
```

## 进阶：享元模式

场景：权限系统，每个用户有一棵权限树。

```rust
// 朴素实现：每个用户都有完整的权限树副本
struct User {
    id: String,
    permissions: MenuItem,  // 每个用户都拷贝一份
}

// 问题：1000 个用户 = 1000 棵树，内存爆炸
```

**享元模式**的核心是：**运用共享技术有效地支持大量细粒度对象**。

```rust
use std::sync::Arc;

// 共享的权限节点
#[derive(Debug, Clone)]
pub struct PermissionNode {
    pub id: String,
    pub name: String,
    pub children: Vec<Arc<PermissionNode>>,
}

// 权限模板（全局共享）
pub struct PermissionTemplate {
    root: Arc<PermissionNode>,
    index: HashMap<String, Arc<PermissionNode>>,
}

impl PermissionTemplate {
    pub fn new(root: PermissionNode) -> Self {
        let root = Arc::new(root);
        let mut index = HashMap::new();
        Self::build_index(&root, &mut index);
        Self { root, index }
    }

    fn build_index(node: &Arc<PermissionNode>, index: &mut HashMap<String, Arc<PermissionNode>>) {
        index.insert(node.id.clone(), Arc::clone(node));
        for child in &node.children {
            Self::build_index(child, index);
        }
    }

    pub fn get_node(&self, id: &str) -> Option<Arc<PermissionNode>> {
        self.index.get(id).cloned()
    }
}

// 用户权限：只存储有权限的节点 ID
struct UserPermissions {
    template: Arc<PermissionTemplate>,
    granted: HashSet<String>,  // 只存 ID，不存树
}

impl UserPermissions {
    pub fn new(template: Arc<PermissionTemplate>, granted: HashSet<String>) -> Self {
        Self { template, granted }
    }

    pub fn has_permission(&self, id: &str) -> bool {
        self.granted.contains(id)
    }

    // 按需生成权限树视图
    pub fn build_tree(&self) -> Option<FilteredPermissionTree> {
        self.build_tree_from(&self.template.root)
    }

    fn build_tree_from(&self, node: &Arc<PermissionNode>) -> Option<FilteredPermissionTree> {
        // 先检查子节点
        let children: Vec<_> = node.children
            .iter()
            .filter_map(|child| self.build_tree_from(child))
            .collect();

        // 如果有权限或有可见的子节点
        if self.granted.contains(&node.id) || !children.is_empty() {
            Some(FilteredPermissionTree {
                node: Arc::clone(node),
                children,
            })
        } else {
            None
        }
    }
}

struct FilteredPermissionTree {
    node: Arc<PermissionNode>,
    children: Vec<FilteredPermissionTree>,
}
```

现在：

```text
1000 个用户：
  - 1 个 PermissionTemplate（共享）
  - 1000 个 HashSet<String>（每个用户几十个 ID）

内存占用大幅降低
```

## 陷阱：别掉进这些坑

### 陷阱一：递归栈溢出

```rust
// ❌ 深度递归可能栈溢出
fn count_recursive(node: &MenuItem) -> usize {
    1 + node.children().iter().map(|c| count_recursive(c)).sum::<usize>()
}

// 如果树深度超过几千层，会 stack overflow
```

解决方案：**用迭代替代递归**

```rust
// ✅ 迭代实现，不会栈溢出
fn count_iterative(root: &MenuItem) -> usize {
    let mut count = 0;
    let mut stack = vec![root];

    while let Some(node) = stack.pop() {
        count += 1;
        for child in node.children() {
            stack.push(child);
        }
    }

    count
}
```

### 陷阱二：访问者状态泄漏

```rust
// ❌ 复用访问者会有问题
let mut counter = CountVisitor { count: 0 };
menu1.accept(&mut counter);
println!("Menu1: {}", counter.count);  // 正确

menu2.accept(&mut counter);
println!("Menu2: {}", counter.count);  // 错误！累加了 menu1 的结果
```

解决方案：

```rust
// ✅ 方案 1：每次创建新访问者
let count1 = {
    let mut v = CountVisitor { count: 0 };
    menu1.accept(&mut v);
    v.count
};

// ✅ 方案 2：访问者提供重置方法
impl CountVisitor {
    fn reset(&mut self) {
        self.count = 0;
    }
}

// ✅ 方案 3：访问者消费自己
impl CountVisitor {
    fn count(mut self, menu: &MenuItem) -> usize {
        menu.accept(&mut self);
        self.count
    }
}
```

### 陷阱三：循环引用

```rust
// ❌ 如果树变成图，会无限循环
struct Node {
    children: Vec<Rc<RefCell<Node>>>,
    parent: Option<Rc<RefCell<Node>>>,  // 父引用导致循环
}

// 遍历时会无限循环或内存泄漏
```

解决方案：

```rust
// ✅ 用 Weak 打破循环
use std::rc::{Rc, Weak};

struct Node {
    children: Vec<Rc<RefCell<Node>>>,
    parent: Option<Weak<RefCell<Node>>>,  // Weak 不增加引用计数
}

// 或者用 ID 引用
struct Node {
    id: String,
    children: Vec<String>,  // 存子节点 ID
    parent_id: Option<String>,
}
```

### 陷阱四：可变遍历

```rust
// ❌ 遍历时修改树结构
for item in menu.dfs_iter() {
    if item.name() == "delete_me" {
        menu.remove_child(item.id());  // 危险！
    }
}
```

解决方案：

```rust
// ✅ 先收集要删除的，再删除
let to_delete: Vec<_> = menu.dfs_iter()
    .filter(|item| item.name() == "delete_me")
    .map(|item| item.id().to_string())
    .collect();

for id in to_delete {
    menu.remove_child(&id);
}
```

## 完整示例

```rust
use std::collections::{HashMap, HashSet, VecDeque};
use std::sync::Arc;

// ==================== 树节点定义 ====================

#[derive(Debug, Clone)]
pub enum MenuItem {
    Leaf {
        id: String,
        name: String,
        path: String,
        icon: Option<String>,
    },
    Group {
        id: String,
        name: String,
        icon: Option<String>,
        children: Vec<MenuItem>,
    },
}

impl MenuItem {
    pub fn id(&self) -> &str {
        match self {
            MenuItem::Leaf { id, .. } => id,
            MenuItem::Group { id, .. } => id,
        }
    }

    pub fn name(&self) -> &str {
        match self {
            MenuItem::Leaf { name, .. } => name,
            MenuItem::Group { name, .. } => name,
        }
    }

    pub fn children(&self) -> &[MenuItem] {
        match self {
            MenuItem::Leaf { .. } => &[],
            MenuItem::Group { children, .. } => children,
        }
    }

    pub fn is_leaf(&self) -> bool {
        matches!(self, MenuItem::Leaf { .. })
    }
}

// ==================== 访问者模式 ====================

pub trait MenuVisitor {
    fn visit_leaf(&mut self, id: &str, name: &str, path: &str, icon: Option<&str>);
    fn visit_group_start(&mut self, id: &str, name: &str, icon: Option<&str>);
    fn visit_group_end(&mut self, id: &str, name: &str);
}

impl MenuItem {
    pub fn accept<V: MenuVisitor>(&self, visitor: &mut V) {
        match self {
            MenuItem::Leaf { id, name, path, icon } => {
                visitor.visit_leaf(id, name, path, icon.as_deref());
            }
            MenuItem::Group { id, name, icon, children } => {
                visitor.visit_group_start(id, name, icon.as_deref());
                for child in children {
                    child.accept(visitor);
                }
                visitor.visit_group_end(id, name);
            }
        }
    }
}

// 计数访问者
pub struct CountVisitor {
    pub total: usize,
    pub leaves: usize,
    pub groups: usize,
}

impl CountVisitor {
    pub fn new() -> Self {
        Self { total: 0, leaves: 0, groups: 0 }
    }
}

impl MenuVisitor for CountVisitor {
    fn visit_leaf(&mut self, _: &str, _: &str, _: &str, _: Option<&str>) {
        self.total += 1;
        self.leaves += 1;
    }

    fn visit_group_start(&mut self, _: &str, _: &str, _: Option<&str>) {
        self.total += 1;
        self.groups += 1;
    }

    fn visit_group_end(&mut self, _: &str, _: &str) {}
}

// ==================== 迭代器模式 ====================

pub struct DfsIterator<'a> {
    stack: Vec<&'a MenuItem>,
}

impl<'a> DfsIterator<'a> {
    pub fn new(root: &'a MenuItem) -> Self {
        Self { stack: vec![root] }
    }
}

impl<'a> Iterator for DfsIterator<'a> {
    type Item = &'a MenuItem;

    fn next(&mut self) -> Option<Self::Item> {
        let node = self.stack.pop()?;
        for child in node.children().iter().rev() {
            self.stack.push(child);
        }
        Some(node)
    }
}

pub struct BfsIterator<'a> {
    queue: VecDeque<&'a MenuItem>,
}

impl<'a> BfsIterator<'a> {
    pub fn new(root: &'a MenuItem) -> Self {
        let mut queue = VecDeque::new();
        queue.push_back(root);
        Self { queue }
    }
}

impl<'a> Iterator for BfsIterator<'a> {
    type Item = &'a MenuItem;

    fn next(&mut self) -> Option<Self::Item> {
        let node = self.queue.pop_front()?;
        for child in node.children() {
            self.queue.push_back(child);
        }
        Some(node)
    }
}

impl MenuItem {
    pub fn dfs_iter(&self) -> DfsIterator {
        DfsIterator::new(self)
    }

    pub fn bfs_iter(&self) -> BfsIterator {
        BfsIterator::new(self)
    }
}

// ==================== 便捷方法 ====================

impl MenuItem {
    pub fn count(&self) -> usize {
        self.dfs_iter().count()
    }

    pub fn find(&self, id: &str) -> Option<&MenuItem> {
        self.dfs_iter().find(|m| m.id() == id)
    }

    pub fn depth(&self) -> usize {
        fn calc_depth(node: &MenuItem) -> usize {
            match node {
                MenuItem::Leaf { .. } => 0,
                MenuItem::Group { children, .. } => {
                    1 + children.iter().map(calc_depth).max().unwrap_or(0)
                }
            }
        }
        calc_depth(self)
    }

    pub fn filter_by_permissions(&self, permissions: &HashSet<String>) -> Option<MenuItem> {
        match self {
            MenuItem::Leaf { id, name, path, icon } => {
                if permissions.contains(id) {
                    Some(MenuItem::Leaf {
                        id: id.clone(),
                        name: name.clone(),
                        path: path.clone(),
                        icon: icon.clone(),
                    })
                } else {
                    None
                }
            }
            MenuItem::Group { id, name, icon, children } => {
                let filtered: Vec<_> = children
                    .iter()
                    .filter_map(|c| c.filter_by_permissions(permissions))
                    .collect();

                if !filtered.is_empty() {
                    Some(MenuItem::Group {
                        id: id.clone(),
                        name: name.clone(),
                        icon: icon.clone(),
                        children: filtered,
                    })
                } else {
                    None
                }
            }
        }
    }
}

// ==================== 使用示例 ====================

fn main() {
    // 构建菜单树
    let menu = MenuItem::Group {
        id: "root".to_string(),
        name: "系统".to_string(),
        icon: None,
        children: vec![
            MenuItem::Group {
                id: "system".to_string(),
                name: "系统管理".to_string(),
                icon: Some("setting".to_string()),
                children: vec![
                    MenuItem::Leaf {
                        id: "user_list".to_string(),
                        name: "用户列表".to_string(),
                        path: "/system/users".to_string(),
                        icon: None,
                    },
                    MenuItem::Leaf {
                        id: "role_list".to_string(),
                        name: "角色列表".to_string(),
                        path: "/system/roles".to_string(),
                        icon: None,
                    },
                ],
            },
            MenuItem::Group {
                id: "order".to_string(),
                name: "订单管理".to_string(),
                icon: Some("order".to_string()),
                children: vec![
                    MenuItem::Leaf {
                        id: "order_list".to_string(),
                        name: "订单列表".to_string(),
                        path: "/orders".to_string(),
                        icon: None,
                    },
                ],
            },
        ],
    };

    // 使用访问者
    let mut counter = CountVisitor::new();
    menu.accept(&mut counter);
    println!("Total: {}, Leaves: {}, Groups: {}",
        counter.total, counter.leaves, counter.groups);

    // 使用迭代器
    println!("\nDFS traversal:");
    for item in menu.dfs_iter() {
        println!("  {} ({})", item.name(), item.id());
    }

    println!("\nBFS traversal:");
    for item in menu.bfs_iter() {
        println!("  {} ({})", item.name(), item.id());
    }

    // 权限过滤
    let permissions: HashSet<String> = ["user_list", "order_list"]
        .iter()
        .map(|s| s.to_string())
        .collect();

    if let Some(filtered) = menu.filter_by_permissions(&permissions) {
        println!("\nFiltered menu:");
        for item in filtered.dfs_iter() {
            println!("  {} ({})", item.name(), item.id());
        }
    }

    // 便捷方法
    println!("\nMenu stats:");
    println!("  Count: {}", menu.count());
    println!("  Depth: {}", menu.depth());

    if let Some(found) = menu.find("order_list") {
        println!("  Found: {}", found.name());
    }
}
```

## 小结

处理树形结构时，四种模式各有用途：

| 模式 | 解决的问题 | Rust 实现 |
|------|-----------|----------|
| 组合 | 统一处理叶节点和容器 | enum + 递归方法 |
| 访问者 | 分离数据和操作 | trait + accept/visit |
| 迭代器 | 统一遍历方式 | Iterator trait |
| 享元 | 共享重复节点 | Arc + ID 引用 |

关键洞见：

1. **enum 是 Rust 的组合模式**，天然支持递归结构
2. **访问者分离关注点**，但要注意状态管理
3. **迭代器比递归更安全**，不会栈溢出
4. **享元用 Arc 共享**，用 ID 引用避免循环

下一篇，我们看如何让业务人员写代码：**规则引擎——让业务人员写代码**。解释器模式和策略模式的进阶应用。

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
> - **第九篇：树形世界——递归结构的驯服术**（本文）
> - 第十篇：规则引擎——让业务人员写代码
> - 第十一篇：终篇——反模式与断舍离
