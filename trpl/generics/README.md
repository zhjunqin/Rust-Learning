# Rust 泛型 / Trait / 生命周期（TRPL 第 10 章速查：10.0~10.3）

只记录 **主要流程** 与 **最常用语法模板**（generics / trait bounds / `impl Trait` / lifetimes）。

## 0. 使用方式（跑示例）

```bash
cargo new generics_demo
cd generics_demo
# edit src/main.rs or src/lib.rs
cargo run
```

## 1. 泛型（Generics）（ch10-00, ch10-01）

### 1.1 目标：减少重复

典型路径：

- 先提取函数（把重复逻辑抽成一个函数）
- 再把“只差类型”的重复抽象成泛型

### 1.2 泛型函数 + trait bound

泛型参数写在函数名后 `fn f<T>(...)`。如果函数体需要比较/打印等能力，用 **trait bound** 限制 `T`。

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let n = vec![34, 50, 25, 100, 65];
    let c = vec!['y', 'm', 'a', 'q'];
    println!("{}", largest(&n));
    println!("{}", largest(&c));
}
```

### 1.3 泛型结构体 / 枚举

```rust
struct Point<T> {
    x: T,
    y: T,
}

struct Point2<T, U> {
    x: T,
    y: U,
}

enum Option<T> {
    Some(T),
    None,
}
```

### 1.4 `impl` 中的泛型 + 特化到具体类型

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x * self.x + self.y * self.y).sqrt()
    }
}
```

### 1.5 性能：单态化（monomorphization）

- Rust 会在**编译期**把泛型按具体类型展开成多份代码
- 所以泛型通常 **没有运行时开销**（代价是编译产物可能变大一些）

## 2. Trait 与 trait bounds（ch10-02）

### 2.1 定义 trait + 为类型实现 trait

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}

pub struct NewsArticle {
    pub headline: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("(Read more...) {}", self.headline)
    }
}
```

### 2.2 默认实现（default impl）

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```

### 2.3 作为参数：`impl Trait` vs trait bound

解释要点：

- **“trait 作为参数”**的本质：函数不关心具体类型，只要求参数实现某个 trait（具备一组方法/行为）。
- 两种写法几乎等价：
  - `impl Trait`：更简洁，适合参数较少/签名短
  - `T: Trait`（trait bound）：可在多处复用同一个类型参数 `T`，例如强制两个参数是同一类型

```rust
fn notify(item: &impl Summary) {
    println!("{}", item.summarize());
}

fn notify_same_type<T: Summary>(a: &T, b: &T) {
    let _ = (a.summarize(), b.summarize());
}
```

更直观的对比例子：

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Post(String);
struct Article(String);

impl Summary for Post {
    fn summarize(&self) -> String { format!("post: {}", self.0) }
}
impl Summary for Article {
    fn summarize(&self) -> String { format!("article: {}", self.0) }
}

// 允许不同具体类型：只要都实现 Summary
fn notify2(a: &impl Summary, b: &impl Summary) {
    println!("{} | {}", a.summarize(), b.summarize());
}

// 强制两个参数同一具体类型：用泛型参数 T
fn notify2_same<T: Summary>(a: &T, b: &T) {
    println!("{} | {}", a.summarize(), b.summarize());
}

fn main() {
    let p = Post("hello".into());
    let a = Article("world".into());
    notify2(&p, &a); // ✅ ok：不同类型也行
    // notify2_same(&p, &a); // ❌ 报错：要求同一类型
}
```

### 2.4 多个 bound 与 `where`

```rust
use std::fmt::{Debug, Display};

fn f<T, U>(t: &T, u: &U)
where
    T: Display + Clone,
    U: Clone + Debug,
{
    let _ = (t.to_string(), u.clone());
}
```

### 2.5 返回 `impl Trait`（注意：只能返回单一具体类型）

```rust
fn returns_summarizable() -> impl Summary {
    NewsArticle { headline: "hi".into() }
}
```

### 2.6 孤儿规则（orphan rule）

只能在以下两种情况之一为类型实现 trait：

- trait 是你这个 crate 定义的
- 类型是你这个 crate 定义的

（因此不能给标准库类型实现标准库 trait：例如给 `Vec<T>` 实现 `Display`。）

### 2.7 Blanket impl（标准库常见）

```rust
use std::fmt::Display;

fn main() {
    let s = 3.to_string(); // 因为标准库对所有 Display 实现了 ToString
    let _ = s;
}
```

## 3. 生命周期（Lifetimes）（ch10-03）

### 3.1 目标：避免悬垂引用

生命周期注解**不改变**引用活多久，只是描述“这些引用之间的关系”，让借用检查器能证明安全。

### 3.1.1 生命周期、悬垂引用与 borrow checker 的关系（工作原理速记）

- **悬垂引用（dangling reference）**：引用指向的数据已经被释放/离开作用域，但引用还在被使用。
- Rust 的 **借用检查器（borrow checker）** 会在**编译期**拒绝可能产生悬垂引用（以及违反借用规则）的代码；运行时不需要 GC/异常来兜底。
- 可以把“生命周期”理解为编译器在做的一件事：给每个引用推导一个“有效范围”（region），然后检查：
  - 引用的有效范围 **不能** 超过被引用数据的有效范围（不能 outlive）
  - 同一时间的借用必须满足规则（例如 `&T` 与 `&mut T` 不能冲突）

借用检查器大致做什么（高层步骤）：

1. **为每个引用推导/分配生命周期区域**（很多时候可以省略并自动推导；遇到歧义才需要你写 `'a`）。
2. **从代码产生约束**：比如赋值、函数参数/返回值、解引用使用点，会产生“哪个引用必须活到哪里”的约束。
3. **求解并验证**：如果存在某个引用可能在其数据被 drop 后仍被使用（或存在非法并发借用），编译失败。

一个最小的“借用检查器如何阻止悬垂引用”的例子：

```rust
fn main() {
    let r: &i32;
    {
        let x = 5;
        r = &x; // r 借用了 x
    } // x 在这里 drop
    // println!("{r}"); // ❌ 不允许：r 可能指向已释放的 x
}
```

直觉解释：`r` 的作用域比 `x` 长，所以 `r` 不能指向 `x`。编译器用生命周期区域比较（外层 > 内层）就能拒绝它。

补充：现代 Rust 使用 **NLL（Non-Lexical Lifetimes，非词法生命周期）**，引用的“活跃期”通常到**最后一次使用**为止，而不一定到代码块末尾，这也是很多“先用不可变引用、后再创建可变引用”的代码能通过的原因。

### 3.2 函数返回引用：通常必须把返回值生命周期和参数关联

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest(string1.as_str(), string2);
    println!("The longest string is {result}");
}
```

含义速记：返回值活得**不超过** `x` 和 `y` 中较短者（`'a` 是它们重叠的那段）。

结合 borrow checker 的原理，为什么这里需要显式生命周期？

- `longest` 的返回值是一个引用，但它**可能来自 `x` 也可能来自 `y`**。
- 借用检查器需要在**函数签名层面**知道“返回引用和哪个输入引用相关”，才能验证“返回引用不会比其数据活得更久”（no outlive / no dangling）。
- 当有**多个输入引用**时，如果你写成 `fn longest(x: &str, y: &str) -> &str`，省略规则只能推到 `x` 有 `'a`、`y` 有 `'b`，但**返回值该绑定 `'a` 还是 `'b` 不明确**，编译器不会猜测，于是要求你标注。
- 写成 `fn longest<'a>(x: &'a str, y: &'a str) -> &'a str` 的含义是：返回引用的生命周期与 `x`/`y` 的生命周期**共同约束**（实际可用期是两者重叠部分，即“较短者”）。这样 borrow checker 就能在调用点验证 `result` 的使用是否超出安全范围。

### 3.3 结构体里存引用：需要生命周期参数

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}
```

### 3.4 生命周期省略（elision）三条规则

1. 每个引用参数各自获得一个输入生命周期
2. 若只有一个输入生命周期，则它赋给所有输出生命周期
3. 若是方法且有 `&self`/`&mut self`，输出生命周期默认为 `self` 的生命周期

对应示例（理解编译器如何补全）：

1) **每个引用参数各自获得一个输入生命周期**

```rust
// 省略写法
fn foo(x: &i32, y: &i32) {}
// 等价于（编译器补全）
fn foo_explicit<'a, 'b>(x: &'a i32, y: &'b i32) {}
```

2) **只有一个输入生命周期 ⇒ 赋给所有输出生命周期**

```rust
// 省略写法
fn first_word(s: &str) -> &str {
    // ...
    s
}
// 等价于
fn first_word_explicit<'a>(s: &'a str) -> &'a str {
    s
}
```

3) **方法且有 `&self`/`&mut self` ⇒ 输出生命周期默认为 self 的生命周期**

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

impl<'a> ImportantExcerpt<'a> {
    // 省略写法：返回值借用了 self.part，因此输出生命周期跟随 &self
    fn part(&self) -> &str {
        self.part
    }

    // 等价于（显式写法）
    fn part_explicit<'b>(&'b self) -> &'b str {
        self.part
    }
}
```

反例（3 条省略规则仍无法让签名“无歧义”，因此会编译失败）：

1) **多个输入引用 + 返回引用**：规则 1 会给两个参数分配不同生命周期，但规则 2/3 都不适用，输出生命周期无法决定（返回的是 `x` 还是 `y`？）

```rust
// ❌ 不能编译：缺少生命周期标注
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}
```

解释：编译器只能推到 `fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str`，
但返回值应该跟 `'a` 还是 `'b` 关联不明确，所以必须显式写：
`fn longest<'a>(x: &'a str, y: &'a str) -> &'a str`（见上面的正例）。

2) **返回局部变量的引用**：即使你想写生命周期，也无法让它安全（局部值会在函数结束时被 drop）

```rust
// ❌ 不能编译：返回了指向局部变量的引用（会悬垂）
fn bad_ref() -> &str {
    let s = String::from("hello");
    &s[..]
}
```

解释：这里的问题不是“省略规则推不出”，而是**根本不允许**返回局部变量引用。
修复方式通常是返回拥有所有权的值（例如 `String`），或让引用来自参数。

3) **方法的规则 3 可能“太强”**：当方法既有 `&self` 又有另一个引用参数时，
如果你想返回“另一个引用”，省略规则会默认把输出绑定到 `self`，从而导致编译错误。

```rust
struct Holder<'a> {
    s: &'a str,
}

impl<'a> Holder<'a> {
    // ❌ 不能编译：省略规则会把返回值生命周期绑定到 &self
    fn choose_other(&self, other: &str) -> &str {
        other
    }

    // ✅ 显式标注：返回值跟随 other
    fn choose_other_explicit<'b>(&self, other: &'b str) -> &'b str {
        other
    }
}
```

### 3.5 `'static`

- 字符串字面值是 `&'static str`
- 不要把 `'static` 当作“修错误的万能药”；多数时候应修正借用关系/返回拥有所有权的值

```rust
fn main() {
    // 字符串字面值的生命周期贯穿整个程序
    let s: &'static str = "I have a static lifetime.";
    println!("{s}");
}
```

一个常见误区（不要这样“修复”借用问题）：

```rust
// ❌ 错误示例：把局部 String 的引用强行当成 'static 是不可能的
fn bad() -> &'static str {
    let s = String::from("hello");
    &s[..]
}
```

## 4. 组合：泛型 + trait bounds + 生命周期

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    println!("Announcement: {ann}");
    if x.len() > y.len() { x } else { y }
}
```
