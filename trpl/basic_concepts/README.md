# Rust 基础概念（TRPL 第 3 章速查）

只记录 **主要流程** 与 **关键知识点**；用最小代码片段帮助回忆与复习（对应 TRPL：ch03-00~ch03-05）。

## 0. 使用方式（跑示例的固定流程）

```bash
cargo new basic_concepts
cd basic_concepts
# edit src/main.rs
cargo run
```

（可选）只做语法/类型检查更快：

```bash
cargo check
```

## 1. 概览与关键字（ch03-00）

- **本章覆盖**：变量、数据类型、函数、注释、控制流
- **关键字（keywords）**：Rust 保留字不能用作变量/函数名

## 2. 变量与可变性（ch03-01）

- **默认不可变**：`let x = 5;` 之后不能再给 `x` 赋新值
- **可变变量**：用 `mut` 显式声明“这个变量会被修改”

```rust
fn main() {
    let mut x = 5;
    x = 6;
    println!("{x}");
}
```

- **常量**：用 `const` 声明；**必须**标注类型；不能用 `mut`；可在任何作用域（含全局）

```rust
const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;
```

- **遮蔽（shadowing）**：重复 `let` 创建“新变量”，可复用名字；并且**可以改变类型**（与 `mut` 区分）

```rust
fn main() {
    let spaces = "   ";
    let spaces = spaces.len(); // shadowing: &str -> usize
    println!("{spaces}");
}
```

## 3. 数据类型（ch03-02）

- **静态类型**：编译期需要确定类型；多数情况下可推断，不行时要加类型注解

```rust
fn main() {
    let guess: u32 = "42".parse().expect("Not a number!");
    println!("{guess}");
}
```

- **标量（scalar）**：整型 / 浮点 / bool / char（`char` 用单引号，代表 Unicode 标量值）
- **复合（compound）**：tuple / array
  - tuple：可解构或用 `.0/.1/...` 访问
  - array：元素类型相同、长度固定；索引越界会 panic（运行时）

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);
    let (x, y, z) = tup;
    println!("{x} {y} {z}");

    let a = [1, 2, 3, 4, 5];
    println!("{}", a[0]);
}
```

## 4. 函数如何工作（ch03-03）

- **命名风格**：snake_case
- **参数必须标注类型**（函数签名的一部分）
- **语句 vs 表达式**：
  - 语句不返回值（例如 `let y = 6;`）
  - 表达式产生值；函数体最后一个**无分号**表达式常用作返回值

```rust
fn plus_one(x: i32) -> i32 {
    x + 1 // no semicolon => expression return value
}

fn main() {
    let x = plus_one(5);
    println!("{x}");
}
```

## 5. 注释（ch03-04）

- **行注释**：`// ...`（到行尾）
- **多行注释**：每行都写 `// ...`
- 文档注释（如 `///`）后续在第 14 章展开，这里只记“Rust 也支持文档注释”

## 6. 控制流（ch03-05）

- **`if` 是表达式**：可用于 `let` 右侧；并且 `if/else` 分支返回值类型必须一致
- **条件必须是 `bool`**：Rust 不会把整数当成 truthy/falsy

```rust
fn main() {
    let number = 3;
    let kind = if number % 2 == 0 { "even" } else { "odd" };
    println!("{kind}");
}
```

- **循环**：`loop` / `while` / `for`
  - `loop`：配合 `break`/`continue`；`break expr` 可从循环“返回值”（`break expr` 与 `break expr;` 都可以）
  - `while`：条件循环
  - `for`：遍历集合或 range（更安全/更简洁，避免手写索引越界问题）

```rust
fn main() {
    // loop + break returning value
    let mut counter = 0;
    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2;
        }
    };
    println!("{result}");

    // for over a range
    for n in (1..4).rev() {
        println!("{n}");
    }

    // while (条件循环)
    let mut m = 3;
    while m != 0 {
        m -= 1;
    }
}
```

提示：无限循环运行中可用 `Ctrl-C` 终止。
