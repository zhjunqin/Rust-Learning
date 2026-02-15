# Rust 结构体（TRPL 第 5 章速查：5.0~5.3）

只记录 **主要流程** 与 **关键知识点**（struct 定义/实例化/调试输出/方法语法与 `impl`）。

## 0. 使用方式（跑示例）

```bash
cargo new structs_demo
cd structs_demo
# edit src/main.rs
cargo run
```

（可选）更快检查：

```bash
cargo check
```

## 1. 结构体是什么（ch05-00）

- **struct**：自定义数据类型，把多个相关值包装成一个“有意义的整体”（类似对象的“数据字段”部分）
- 对比 tuple：tuple 也能组合值，但 struct **有字段名**，表达意图更清晰、使用更灵活
- struct 往往配合 `impl`：把“数据（fields）+ 行为（methods）”组织在一起

## 2. 定义与实例化（ch05-01）

### 2.1 定义结构体 + 创建实例 + 访问字段

```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}

fn main() {
    let mut user = User {
        active: true,
        username: String::from("alice"),
        email: String::from("a@example.com"),
        sign_in_count: 1,
    };

    user.email = String::from("new@example.com");
    println!("{}", user.email);
}
```

- **可变性规则**：要修改字段，必须让**整个实例**是 `mut`（不能只让某个字段 `mut`）。

### 2.2 字段初始化简写（field init shorthand）

字段名与变量名相同可简写：

```rust
struct User {
    username: String,
    email: String,
}

fn build_user(username: String, email: String) -> User {
    User { username, email }
}
```

### 2.3 结构体更新语法（struct update syntax）与所有权

```rust
#[derive(Debug)]
struct User {
    active: bool,
    username: String,
    email: String,
}

fn main() {
    let user1 = User {
        active: true,
        username: String::from("alice"),
        email: String::from("a@example.com"),
    };

    let user2 = User {
        email: String::from("b@example.com"),
        ..user1
    };

    println!("{user2:?}");
    // println!("{user1:?}"); // user1.username 被 move，user1 可能不再可用
}
```

- `..user1` 会“复用”剩余字段的值；如果字段类型是 `String` 这类非 `Copy`，会触发 **move**，导致旧实例（或部分字段）不能再用。

补充解释：`user1` 是“整个都不能用”，还是“部分字段还能用”？

- `..user1` 会对**未被你显式赋值**的字段做“逐字段移动/拷贝”：
  - 非 `Copy` 字段（如 `String`）会 **move** 到新实例
  - `Copy` 字段（如 `bool`、整数）会 **copy**
  - 你在新实例里显式赋值的字段（示例中的 `email`）不会从 `user1` 移走
- 结果是：`user1` 可能变成 **部分 moved** 的状态——
  - **不能再把 `user1` 当作整体使用/借用**（比如 `println!("{user1:?}")` 会报错）
  - 但**没被 move 的字段仍然可以单独使用**（例如这里的 `active`、`email`）

```rust
#[derive(Debug)]
struct User {
    active: bool,
    username: String,
    email: String,
}

fn main() {
    let user1 = User {
        active: true,
        username: String::from("alice"),
        email: String::from("a@example.com"),
    };

    let user2 = User {
        email: String::from("b@example.com"), // email 没从 user1 移走
        ..user1 // username: move；active: Copy
    };

    // println!("{user1:?}"); // ❌ 报错：user1 已经部分 moved（username 被移走）
    println!("{}", user1.active); // ✅ ok：Copy 字段仍可用
    println!("{}", user1.email);  // ✅ ok：email 字段仍可用（注意：这里会把 email move 出来）

    println!("{user2:?}");
}
```

### 2.4 元组结构体 / 类单元结构体

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

struct AlwaysEqual;

fn main() {
    let _black = Color(0, 0, 0);
    let _origin = Point(0, 0, 0);
    let _x = AlwaysEqual;
}
```

- 元组结构体：有类型名但无字段名（用 `.0/.1/...` 或解构）
- 类单元结构体：无字段，常用于“只为了实现 trait/标记类型”

类单元结构体的常见用途（为什么要定义一个“空 struct”）：

- **作为标记类型（marker type）**：用“类型”区分不同语义/不同实现，而不是靠运行时字段；这类类型通常是 **零大小类型（ZST）**，几乎不占内存。
- **给类型挂行为（实现 trait）**：即便不存任何数据，也可以通过 `impl`/trait 让它具备一组方法；常见于“策略/后端/解析器”等可替换实现。

```rust
trait Greeter {
    fn greet(&self) -> &'static str;
}

struct Friendly;
struct Formal;

impl Greeter for Friendly {
    fn greet(&self) -> &'static str { "hi" }
}
impl Greeter for Formal {
    fn greet(&self) -> &'static str { "hello" }
}

fn main() {
    let a = Friendly;
    let b = Formal;
    println!("{} {}", a.greet(), b.greet());
}
```

### 2.5 结构体字段的所有权（只记结论）

- 常用做法是让 struct **拥有**数据（用 `String` 而不是 `&str`）
- 若要在 struct 里存引用，需要 **生命周期标注**（后面章节再展开）

## 3. 结构体示例：Rectangle（ch05-02）

核心流程：从 `width/height` 两个变量 → tuple → struct（字段命名表达意图）。

### 3.1 用 `#[derive(Debug)]` + `{:?}` / `{:#?}` / `dbg!` 调试

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let scale = 2;
    let rect = Rectangle {
        width: dbg!(30 * scale),
        height: 50,
    };
    dbg!(&rect); // dbg! 打到 stderr；这里用 &rect 避免 move
}
```

- `println!("{:?}", x)`：需要 `x: Debug`
- `println!("{:#?}", x)`：更好看的 Debug 输出
- `dbg!(expr)`：打印“文件:行号 + 值”，并**返回该表达式的值**；对非 `Copy` 值要注意所有权（通常传 `&x`）

补充说明（这些用法会影响 Cargo 的 build 方式吗？）

- **对 `println!` / `dbg!` 来说基本不影响**：不论 `cargo run` 还是 `cargo run --release`，只要代码执行到这些宏，就会打印输出（release 不会“自动关闭”这些输出）。
- **可能影响“能否编译”**：
  - `println!("{:?}")` / `{:#?}` 要求类型实现 `Debug`（通常 `#[derive(Debug)]`），否则编译报错。
  - `dbg!(expr)` 会取得 `expr` 的值并返回它；对非 `Copy` 值可能触发 move，所以常写 `dbg!(&x)`。
- **输出差异**：
  - `dbg!` 输出到 **stderr**，而 `println!` 输出到 **stdout**（重定向/日志收集时会体现）。
- **只有“调试断言/条件编译”的输出会受 profile 影响**：例如 `debug_assert!` / `debug_assert_eq!`，或 `#[cfg(debug_assertions)]` 包起来的打印，通常在 `--release` 下不会生效/不会编译进产物。

## 4. 方法语法与 `impl`（ch05-03）

### 4.1 方法（method）与 `self`

- 方法定义在 `impl Type` 块里
- 第一个参数一定是 `self`（`self`/`&self`/`&mut self`），表示接收者

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}

fn main() {
    let r1 = Rectangle { width: 30, height: 50 };
    let r2 = Rectangle { width: 10, height: 40 };
    println!("{}", r1.area());
    println!("{}", r1.can_hold(&r2));
}
```

### 4.2 字段同名方法（getter）与访问规则

- `rect.width` 是字段
- `rect.width()` 是方法（哪怕同名也能区分：有没有 `()`）

### 4.3 关联函数（associated functions）：没有 `self`

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn square(size: u32) -> Self {
        Self { width: size, height: size }
    }
}

fn main() {
    let sq = Rectangle::square(3);
    println!("{sq:?}");
}
```

- 调用方式：`Type::func(...)`（例如 `String::from`）

### 4.4 自动引用/解引用与为什么没有 `->`

- Rust 方法调用会自动为接收者补 `&` / `&mut` 或做 `*` 解引用，让调用符合方法签名
- 因此不需要 C/C++ 的 `->` 运算符

### 4.5 多个 `impl` 块

- 同一个类型可以写多个 `impl` 块（语法允许；通常按组织需要拆分）
