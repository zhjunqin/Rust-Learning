# ch15：智能指针（Smart Pointers）速查

只记录：`Box<T>` / `Deref`（含 deref coercion）/ `Drop` / `Rc<T>` / `RefCell<T>`（内部可变性）/ 引用环与 `Weak<T>`。

## 0. 智能指针 vs 引用（心智模型）

- **引用 `&T` / `&mut T`**：只“借用”数据，几乎无额外功能
- **智能指针**：通常是 struct，像指针一样用，但带**额外元数据/能力**；很多时候还**拥有**所指向的数据
- 常见关键 trait：
  - **`Deref` / `DerefMut`**：让 `*p`/方法调用像引用一样工作（含 deref coercion）
  - **`Drop`**：离开作用域时自动清理资源

## 1. `Box<T>`：把值放到堆上（单一所有权）

### 1.1 适用场景（速记）

- **类型大小在编译期不确定**，但需要放到要求“已知大小”的地方（典型：递归类型）
- **转移大量数据所有权**时避免复制大量栈数据（移动指针更便宜）
- **trait 对象**（只关心实现了某 trait，不关心具体类型；ch18 展开）

### 1.2 最小用法

```rust
fn main() {
    let b = Box::new(5);
    println!("b = {b}");
}
```

### 1.3 用 `Box` 打破递归：cons list 示例

递归 enum 若直接包含自身，编译期无法计算大小；改成“间接”指向：

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

核心点：`Box<List>` 的大小是固定的（指针大小），因此 `List` 的大小可计算。

## 2. `Deref`：让智能指针像引用一样用

### 2.1 `*` 背后的展开

对实现了 `Deref` 的类型 `T`：

- `*value` 会被编译器理解为：`*(value.deref())`
- `deref()` 返回的是**引用**，避免把内部值 move 出来

### 2.2 自己实现一个“像 Box 的指针”（不做堆分配）

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> Self { MyBox(x) }
}

impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target { &self.0 }
}

fn main() {
    let x = 5;
    let y = MyBox::new(x);
    assert_eq!(5, *y);
}
```

### 2.3 Deref coercion（解引用强制转换）

在函数/方法传参时，Rust 会在编译期自动插入若干次 `deref()`：

- 典型：`&String -> &str`（`String: Deref<Target=str>`）
- 你自己的 `&MyBox<String>` 也能一路被强转到 `&str`

可变性交互规则（结论）：

1) `&T -> &U` 需要 `T: Deref<Target=U>`
2) `&mut T -> &mut U` 需要 `T: DerefMut<Target=U>`
3) `&mut T -> &U` 也可以（可变借用降级成不可变借用）

## 3. `Drop`：离开作用域时自动清理（析构）

### 3.1 基本用法

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}
```

要点：

- Rust 会在值离开作用域时自动调用 `drop`（按创建的逆序 drop）
- **不能**手动调用 `value.drop()`（否则可能 double free）
- 需要“提前释放”：用 `std::mem::drop(value)`

## 4. `Rc<T>`：引用计数（共享所有权，单线程）

### 4.1 什么时候用

- 需要**多个所有者**共享同一份堆数据
- 编译期无法确定“最后一个使用者是谁”
- **只适用于单线程**（多线程用 `Arc<T>`，ch16）

### 4.2 `Rc::clone` 不是深拷贝

- `Rc::clone(&rc)` / `rc.clone()`：只增加引用计数（更像“复制指针”）
- `Rc::strong_count(&rc)`：查看强引用计数

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("hi"));
    let b = Rc::clone(&a);
    assert_eq!(Rc::strong_count(&a), 2);
    drop(b);
    assert_eq!(Rc::strong_count(&a), 1);
}
```

## 5. `RefCell<T>`：运行期借用检查（内部可变性，单线程）

### 5.1 三者对比（背结论）

- `Box<T>`：单一所有权；**编译期**借用检查；可变/不可变都可（看你怎么借）
- `Rc<T>`：多所有权；**编译期**仅允许不可变借用（共享读取）
- `RefCell<T>`：单一所有权；**运行期**借用检查；允许在不可变外壳下修改内部（内部可变性）

### 5.2 `borrow` / `borrow_mut`

- `borrow() -> Ref<T>`：不可变借用（计数 +1）
- `borrow_mut() -> RefMut<T>`：可变借用（独占）
- 违反规则：**运行时 panic**（不是编译错误）

### 5.3 典型用例：mock 对象记录调用（trait 只给 `&self`）

当 trait 方法签名是 `fn send(&self, msg: &str)`，但测试需要在 `send` 内部记录消息：

```rust
use std::cell::RefCell;

struct MockMessenger {
    sent: RefCell<Vec<String>>,
}

impl MockMessenger {
    fn new() -> Self {
        Self { sent: RefCell::new(vec![]) }
    }
}

fn main() {
    let m = MockMessenger::new();
    m.sent.borrow_mut().push("hi".into());
    assert_eq!(m.sent.borrow().len(), 1);
}
```

### 5.4 `Rc<RefCell<T>>`：多个所有者 + 可变

- `Rc` 负责“多所有权”
- `RefCell` 负责“内部可变性（运行期借用）”

常见组合：`Rc<RefCell<T>>`。

## 6. 引用环（reference cycles）与 `Weak<T>`

### 6.1 引用环为何会泄漏

- `Rc<T>` 只有当 `strong_count == 0` 才释放数据
- 如果 A 强引用 B，B 强引用 A ⇒ 两边计数都不会归零 ⇒ **内存泄漏**

### 6.2 用 `Weak<T>` 破环（不表达所有权）

- `Rc::downgrade(&rc) -> Weak<T>`：增加 `weak_count`，不增加 `strong_count`
- `weak.upgrade() -> Option<Rc<T>>`：对象还活着 ⇒ `Some(Rc<T>)`；否则 `None`

典型结构：树

- 父拥有子：`children: RefCell<Vec<Rc<Node>>>`（强引用）
- 子指向父但不拥有：`parent: RefCell<Weak<Node>>`（弱引用）

这样父丢弃 ⇒ 子也丢弃；子丢弃不会影响父；同时避免环。

## 7. 单向链表（Singly Linked List）示例

### 7.1 最常见：`Box` 独占 next（每个节点只有一个所有者）

```rust
#[derive(Debug)]
struct Node {
    val: i32,
    next: Option<Box<Node>>,
}

impl Node {
    fn new(val: i32) -> Self {
        Self { val, next: None }
    }

    fn push_front(val: i32, next: Option<Box<Node>>) -> Box<Node> {
        Box::new(Node { val, next })
    }

    fn iter(&self) -> impl Iterator<Item = i32> + '_ {
        // 用闭包迭代沿 next 往后走（不分配新节点）
        std::iter::successors(Some(self), |n| n.next.as_deref()).map(|n| n.val)
    }
}

fn main() {
    // 构建：3 -> 2 -> 1
    let list = Node::push_front(3, Some(Node::push_front(2, Some(Node::push_front(1, None)))));
    let v: Vec<i32> = list.iter().collect();
    assert_eq!(v, vec![3, 2, 1]);
}
```

要点：

- `next: Option<Box<Node>>` 表示 **next 节点被当前节点独占拥有**
- drop 时沿链释放（不需要 `Rc`）

### 7.2 共享尾部（持久化链表风格）：`Rc` 共享 next（可选）

当你想让多个链表“共享同一段尾部”时，把 `next` 改成 `Rc<Node>`：

```rust
use std::rc::Rc;

#[derive(Debug)]
struct RcNode {
    val: i32,
    next: Option<Rc<RcNode>>,
}

fn main() {
    let tail = Rc::new(RcNode { val: 1, next: None });
    let a = Rc::new(RcNode { val: 2, next: Some(Rc::clone(&tail)) }); // 2 -> 1
    let b = Rc::new(RcNode { val: 3, next: Some(Rc::clone(&tail)) }); // 3 -> 1

    assert_eq!(Rc::strong_count(&tail), 3); // tail 被 a/b/tail 三处共享
    let _ = (a, b);
}
```

要点：

- `Rc` 只适用于单线程；它让“共享结构”变得可行，但带引用计数开销