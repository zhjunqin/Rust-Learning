# ch20：高级特性（Advanced Features）速查

本章内容“平时不常用，但遇到就必须懂”：`unsafe`、高级 trait/类型、函数指针与返回闭包、宏（声明宏/过程宏）。

## 0. 一句话定位

- **`unsafe`**：并不“关闭所有检查”，只开放 5 个“超能力”，并把内存安全责任转交给你
- **高级 trait/类型**：帮助你表达更精确的约束/意图（关联类型、默认类型参数、完全限定语法、`!`、DST…）
- **高级函数/闭包**：函数指针 `fn`、返回闭包（`impl Fn` / `Box<dyn Fn>`）
- **宏**：编译期展开/生成代码（`macro_rules!` 与过程宏）

## 1. 不安全 Rust（Unsafe Rust）

### 1.1 `unsafe` 开放的 5 个超能力

只有在 `unsafe` 中才能做的事（unsafe superpowers）：

- 解引用裸指针（raw pointer）
- 调用不安全函数/方法（`unsafe fn`）
- 访问或修改可变静态变量（`static mut`）
- 实现不安全 trait（`unsafe trait` / `unsafe impl`）
- 访问 `union` 字段

重要澄清：

- `unsafe` **不会**关闭借用检查器；在 `unsafe` 里用引用依然要过借用检查
- `unsafe` 的意义是：这些操作的**安全性契约**（invariant / preconditions）编译器不再替你证明，你要自己保证
- 经验法则：**把 `unsafe` 块缩到最小**，并尽量封装成安全 API

### 1.2 裸指针 `*const T` / `*mut T`

裸指针与引用的区别（能力更强，但更危险）：

- 可绕过借用规则（可同时拥有多 `*mut` / 混用 `*const` 与 `*mut`）
- 不保证有效、可为空、无自动清理

创建裸指针可以在安全代码里；**解引用必须 `unsafe`**：

```rust
fn main() {
    let mut num = 5;

    let r1 = &raw const num; // *const i32
    let r2 = &raw mut num;   // *mut i32

    unsafe {
        println!("{}", *r1);
        *r2 += 1;
    }
}
```

### 1.3 `unsafe fn`：调用者负责满足契约

```rust
unsafe fn dangerous() {}

fn main() {
    unsafe { dangerous() }
}
```

实践建议：

- 为 `unsafe fn` 写清楚调用者要保证什么（常用 `// SAFETY: ...` 注释）
- 在 `unsafe fn` 里做不安全操作也尽量用更小的 `unsafe {}` 包起来（2024 edition 默认会对 “unsafe op in unsafe fn” 给 warning）

### 1.4 用 `unsafe` 构建“安全抽象”：`split_at_mut` 思路

安全地把一个 `&mut [T]` 拆成两个不重叠的 `&mut [T]`，借用检查器无法直接证明“不同部分不重叠”，因此标准库内部用裸指针 + `from_raw_parts_mut` 做安全封装。

关键点：

- 在安全边界做足断言（例如 `mid <= len`）
- `unsafe` 内只做“编译器不懂但你能证明正确”的那一小段

### 1.5 FFI：`extern "C"` 与 `#[no_mangle]`

调用外部函数（例如 C）通常被视为 `unsafe`，因为 Rust 无法验证对方遵守 Rust 的规则：

```rust
unsafe extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe { println!("{}", abs(-3)); }
}
```

从其它语言调用 Rust：

```rust
#[unsafe(no_mangle)]
pub extern "C" fn call_from_c() {
    println!("Just called a Rust function from C!");
}
```

### 1.6 `static` / `static mut`

- 不可变 `static`：安全读取（值有固定内存地址，通常是 `'static` 引用）
- `static mut`：读写都必须 `unsafe`，多线程下易数据竞争（UB 风险极高）

### 1.7 `unsafe trait` / `union`

- `unsafe trait`：trait 自身携带编译器无法验证的不变式，实现者用 `unsafe impl` 承诺满足它
- `union` 字段访问不安全：Rust 无法知道当前存的到底是哪种字段类型

### 1.8 用 Miri 动态检测未定义行为（UB）

（需要 nightly）

```bash
rustup +nightly component add miri
cargo +nightly miri run
cargo +nightly miri test
```

Miri 是“运行时”检查器：能抓到很多 UB 模式，但不是万能；要结合测试一起用。

## 2. 高级 trait（Advanced Traits）

### 2.1 关联类型（associated types）

用关联类型把“占位类型”绑定到 trait 上（`Iterator::Item` 是典型）：

```rust
trait MyIter {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

对比“trait 泛型参数”：

- **关联类型**：同一类型对该 trait **只能实现一次**（`Item` 只能选一个）
- **泛型参数**：同一类型可以对不同泛型参数多次实现（会带来调用时需要额外类型信息/歧义）

### 2.2 默认泛型类型参数：运算符重载的关键

标准库 `Add`（简化）：

```rust
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

默认参数用途：

- 扩展 trait 而不破坏旧代码
- “大多数情况默认就够用”，少数情况可自定义（如 `Millimeters + Meters`）

### 2.3 同名方法/关联函数的消歧义

方法（带 `self`）可用 `Trait::method(&value)` 指定：

```rust
Pilot::fly(&person);
Wizard::fly(&person);
person.fly(); // 默认调用类型自身 impl 的方法（若存在）
```

关联函数（不带 `self`）可能需要**完全限定语法**：

```rust
<Dog as Animal>::baby_name()
```

通用形式：

```rust
<Type as Trait>::function(receiver_if_method, args...)
```

### 2.4 超（父）trait（supertraits）

当一个 trait 的默认实现需要依赖另一个 trait 的能力时：

```rust
use std::fmt::Display;

trait OutlinePrint: Display {
    fn outline_print(&self) {
        let s = self.to_string();
        // ... 打印边框 ...
    }
}
```

含义：实现 `OutlinePrint` 的类型必须先实现 `Display`。

### 2.5 newtype 模式绕过孤儿规则（并控制 API 面）

孤儿规则：只能在“trait 或 type 至少一方是本地”的情况下实现 trait。

解决：用元组结构体包一层本地类型：

```rust
use std::fmt;

struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}
```

取舍：

- 无运行时开销（编译期可消掉）
- 但默认没有内部类型的方法；需要手动代理或实现 `Deref`（是否暴露全部方法取决于你的设计目的）

## 3. 高级类型（Advanced Types）

### 3.1 newtype：类型安全 / 单位标注 / 隐藏实现

newtype 用“不同类型”强制区分（避免把 `Meters` 当 `Millimeters` 传错）；
也可封装内部实现，只暴露你想要的 API（轻量封装）。

### 3.2 类型别名（type alias）：同义词，不是新类型

```rust
type Kilometers = i32;
```

- `Kilometers` 与 `i32` **完全同一类型**（不会得到 newtype 的额外类型检查）
- 主要用途：**缩短冗长类型**、提高可读性（例如 `Thunk = Box<dyn Fn() + Send + 'static>`）
- 常见模式：为 `Result<T, E>` 固定错误类型（如 `std::io::Result<T>`）

### 3.3 never type：`!`

`!` 表示“不会产生值”（发散/diverging）：

- `continue` / `panic!()` / 永不结束的 `loop` 都可视为 `!`
- `!` 可以被强转为任意类型，因此 `match` 某分支以 `continue` 结束仍能与其他分支类型对齐

### 3.4 动态大小类型（DST）与 `Sized`

- `str`（不是 `&str`）是 DST：编译期不知道大小，所以**不能**直接创建 `str` 变量
- DST 的黄金法则：**必须放在指针后面**（如 `&str`/`Box<str>`，以及 `&dyn Trait`/`Box<dyn Trait>`）

`Sized`：

- 编译期大小已知的类型自动实现 `Sized`
- 泛型默认隐式加上 `T: Sized`
- 若要接受可能 unsized 的类型：用 `T: ?Sized`，并把参数放在指针后（如 `&T`）

## 4. 高级函数与闭包（Advanced Functions & Closures）

### 4.1 函数指针 `fn`

函数可以强转为 `fn` 类型并作为参数传递：

```rust
fn add_one(x: i32) -> i32 { x + 1 }

fn do_twice(f: fn(i32) -> i32, arg: i32) -> i32 {
    f(arg) + f(arg)
}

fn main() {
    println!("{}", do_twice(add_one, 5));
}
```

要点：

- `fn` 是类型；闭包是实现了 `Fn/FnMut/FnOnce` 的匿名类型
- 函数指针实现了 `Fn*` 三 trait，因此“需要闭包”的地方通常也能传函数
- 只收 `fn` 的典型场景：与不支持闭包的外部代码（如 C）交互

### 4.2 返回闭包：`impl Fn` 与 `Box<dyn Fn>`

返回一个闭包（单一实现）：

```rust
fn returns_closure() -> impl Fn(i32) -> i32 {
    |x| x + 1
}
```

但注意：**每个闭包都有独立的具体类型**。两个函数即便都返回 `impl Fn(i32)->i32`，其不透明类型也不同；
若要把“不同闭包实现”放进同一个集合，使用 trait object：

```rust
fn a() -> Box<dyn Fn(i32) -> i32> { Box::new(|x| x + 1) }
fn b() -> Box<dyn Fn(i32) -> i32> { Box::new(|x| x * 2) }

fn main() {
    let v = vec![a(), b()];
    println!("{}", v[0](10));
}
```

## 5. 宏（Macros）

### 5.1 宏 vs 函数

宏是“元编程”：写生成代码的代码（编译期展开）。

宏相对函数的能力：

- 接收可变数量参数（`println!`）
- 在编译期展开，可生成实现/语法结构（例如 `derive` 生成 trait impl）

代价：

- 定义更难读/难维护
- 使用前需要先定义或引入作用域（对比函数可先用后定义）

### 5.2 声明宏：`macro_rules!`（macros by example）

思路类似 `match`：用“代码模式”匹配输入 token，并展开成输出代码。

抓住这几个符号：

- `$x:expr`：捕获一个表达式
- `$( ... ),*`：重复 0 次或多次，以逗号分隔

（例如 `vec!` 的简化定义就使用了这种重复匹配与展开。）

### 5.3 过程宏（procedural macros）

三类：

- 自定义 `#[derive(...)]`
- 类属性宏（attribute-like）：`#[route(GET, "/")]`
- 类函数宏（function-like）：`sql!(SELECT ...)`

共性：

- 输入/输出都是 `TokenStream`
- 通常放在独立的 `proc-macro` crate
- 常用 `syn`（解析为 AST）+ `quote`（生成 Rust 代码）

过程宏函数签名形态（示意）：

```rust
// derive
#[proc_macro_derive(MyDerive)]
pub fn my_derive(input: proc_macro::TokenStream) -> proc_macro::TokenStream { input }

// attribute-like
#[proc_macro_attribute]
pub fn my_attr(attr: proc_macro::TokenStream, item: proc_macro::TokenStream)
    -> proc_macro::TokenStream { item }

// function-like
#[proc_macro]
pub fn my_macro(input: proc_macro::TokenStream) -> proc_macro::TokenStream { input }
```
