# Rust 错误处理（TRPL 第 9 章速查：9.0~9.3）

只记录 **主要流程** 与 **关键准则**（`panic!` / `Result<T,E>` / `?` / 何时 panic）。

## 0. 使用方式（跑示例）

```bash
cargo new error_handling_demo
cd error_handling_demo
# edit src/main.rs
cargo run
```

## 1. 两类错误（ch09-00）

- **不可恢复**（unrecoverable）：通常是 bug（越界、违反不变量）→ `panic!`
- **可恢复**（recoverable）：可报告/重试/降级 → `Result<T, E>`
- Rust 没有异常：用类型系统强制你在编译期承认“可能失败”

## 2. `panic!`：不可恢复错误（ch09-01）

### 2.1 触发方式

- 显式 `panic!("msg")`
- 执行会 panic 的操作（例如 `v[99]` 越界）

### 2.2 backtrace

- 开启：`RUST_BACKTRACE=1 cargo run`（或 `full` 更详细）
- 看 backtrace：找到第一行指向你自己代码文件的位置，通常是问题起点

### 2.3 unwind vs abort（影响二进制体积/清理行为）

- 默认：panic 时 **unwind**（展开栈，逐层清理）
- 可配置：panic 时 **abort**（直接终止，不展开；二进制更小）

```toml
[profile.release]
panic = 'abort'
```

## 3. `Result<T, E>`：可恢复错误（ch09-02）

### 3.1 基本用法：`match` 处理 Ok/Err

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");
    let _file = match f {
        Ok(file) => file,
        Err(e) => panic!("Problem opening the file: {e:?}"),
    };
}
```

### 3.2 区分错误原因：`ErrorKind`

常见模式：NotFound → 创建文件；其他错误 → 继续向上/或 panic（按场景）。

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let greeting_file = match File::open("hello.txt") {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {e:?}"),
            },
            other_error => panic!("Problem opening the file: {other_error:?}"),
        },
    };

    let _ = greeting_file;
}
```

### 3.3 快捷失败：`unwrap` / `expect`

- `unwrap()`：`Ok(v)` 返回 v；`Err(e)` 直接 panic
- `expect("...")`：同 unwrap，但你能提供更有上下文的 panic 信息（更推荐）

### 3.4 传播错误：返回 `Result` 给调用者

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();
    let mut f = File::open("hello.txt")?;
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

### 3.5 `?` 运算符要点

- `Ok(v)`：解包得到 v，继续执行
- `Err(e)`：**提前 return Err(e)**（会通过 `From` 做错误类型转换）
- 只能用在返回兼容类型的函数里（常见是 `Result`/`Option`）

#### `main` 也可以返回 `Result`

```rust
use std::error::Error;
use std::fs::File;

fn main() -> Result<(), Box<dyn Error>> {
    let _f = File::open("hello.txt")?;
    Ok(())
}
```

## 4. 要不要 panic（ch09-03）

默认建议：**库/可复用代码倾向返回 `Result`**，把决策权交给调用者。

### 4.1 什么时候 `unwrap/expect` 也合理？

- 示例/原型/测试：用它们当占位，后续再替换成健壮处理
- 你“比编译器知道更多”：经过人工/逻辑保证不会失败（用 `expect` 写清理由）

### 4.2 什么时候更应该 panic？

- 进入 **有害状态（bad state）**：违反假设/契约/不变量；继续运行会不安全或产生更大损害
- 输入违反函数契约：更可能是调用方 bug（应在文档说明会 panic 的条件）
- 类型系统已经保证某些情况不可能发生时，你可以减少运行时检查

### 4.3 用类型编码有效性（减少到处检查）

典型思路：用自定义类型在构造时验证（例如 `Guess::new` 保证 1..=100），让后续代码在类型层面“相信它一定合法”。

## 5. 常用写法速查：`unwrap` / `unwrap_or_else` / `expect` / `?`（以及一些同类方法）

下面按“**适用场景 → 行为**”总结，并给最小示例。

### 5.1 `unwrap()`

- **适用场景**：原型/测试/你确信不会失败（但失败了就直接崩）
- **行为**：`Ok(v)` 返回 `v`；`Err(e)` 触发 `panic!`

```rust
use std::fs::File;
fn main() {
    let _f = File::open("hello.txt").unwrap();
}
```

### 5.2 `expect("...")`

- **适用场景**：同 `unwrap`，但你想留更清晰的上下文
- **行为**：失败时 panic，并带上你写的提示文本（更好定位）

```rust
use std::fs::File;
fn main() {
    let _f = File::open("hello.txt").expect("hello.txt should exist");
}
```

### 5.3 `unwrap_or(default)` / `unwrap_or_else(|e| ...)`

- **适用场景**：失败时需要“兜底值”或“兜底逻辑”（不想 panic）
- **行为**：
  - `unwrap_or(default)`：直接返回 `default`
  - `unwrap_or_else(f)`：调用闭包 `f(err)` 生成兜底值（**惰性**，只在失败时执行）

```rust
use std::fs;
fn main() {
    let content = fs::read_to_string("hello.txt").unwrap_or("<empty>".to_string());
    let content2 = fs::read_to_string("hello.txt").unwrap_or_else(|_e| "<empty>".to_string());
    let _ = (content, content2);
}
```

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let greeting_file = File::open("hello.txt").unwrap_or_else(|error| {
        if error.kind() == ErrorKind::NotFound {
            File::create("hello.txt").unwrap_or_else(|error| {
                panic!("Problem creating the file: {error:?}");
            })
        } else {
            panic!("Problem opening the file: {error:?}");
        }
    });
}
```

### 5.4 `?`：传播错误（推荐的“默认姿势”）

- **适用场景**：当前函数本身就返回 `Result`/`Option`，你希望把失败交给调用者决定
- **行为**：
  - `Ok(v)?` 得到 `v`
  - `Err(e)?` 立刻 `return Err(e.into())`（可能通过 `From` 转换错误类型）

`Result` 上的 `?`（返回 `Result<T, E>` 的函数里使用）：

```rust
use std::fs;
use std::io;

fn read_username() -> Result<String, io::Error> {
    let s = fs::read_to_string("hello.txt")?; // Err => 提前返回
    Ok(s)
}
```

`Option` 上的 `?`（返回 `Option<T>` 的函数里使用）：

```rust
fn last_char_of_first_line(text: &str) -> Option<char> {
    let line = text.lines().next()?; // None => 提前返回 None
    line.chars().last()              // Option<char>
}

fn main() {
    assert_eq!(last_char_of_first_line("hi\nthere"), Some('i'));
    assert_eq!(last_char_of_first_line(""), None);
}
```

### 5.5 其他同类方法（常用但不必全背）

- **`map` / `map_err`**：只变换 `Ok` 或只变换 `Err`

```rust
use std::fs;
fn main() {
    let n: Result<usize, String> = fs::read_to_string("n.txt")
        .map_err(|e| format!("read failed: {e}"))
        .and_then(|s| s.trim().parse::<usize>().map_err(|e| e.to_string()));
    let _ = n;
}
```

- **`ok_or` / `ok_or_else`**：在 `Option<T>` 与 `Result<T, E>` 之间显式转换（`None` → `Err(...)`）

```rust
fn must_have(x: Option<i32>) -> Result<i32, &'static str> {
    x.ok_or("missing value")
}
```

- **`unwrap_or_default()`**：失败时返回 `Default::default()`（有默认值的类型很方便）
