# ch12：I/O 项目（minigrep）速查

目标：实现一个极简 `grep`：`minigrep <query> <file>`，读取文件后打印包含 `query` 的行；支持环境变量切换大小写；错误输出到 `stderr`。

## 0. 一句话主流程

**args/env → Config → run(读文件→search→输出) → main 统一处理错误并设置退出码**

## 1. 创建与运行方式

```bash
cargo new minigrep
cd minigrep

# 把参数传给程序（`--` 之后是你的程序参数）
cargo run -- to poem.txt
```

## 2. 接受命令行参数

### 2.1 读取参数：`std::env::args()`

- `env::args()` 返回迭代器，常见写法：`collect::<Vec<String>>()`
- 第 0 个参数通常是程序路径；我们通常只取后面的业务参数

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    // args[1] query, args[2] file
}
```

补充：

- `env::args()` 遇到**无效 Unicode** 参数会 panic；需要兼容时用 `env::args_os()`（得到 `OsString`）。

## 3. 读取文件

```rust
use std::fs;

let contents = fs::read_to_string(file_path)?; // -> io::Result<String>
```

核心点：

- 标准库推荐的“一次性读完整文件”API：`fs::read_to_string`
- I/O 失败不要 `expect`：让错误一路返回给调用者（见第 4 节）

## 4. 模块化与错误处理（main.rs / lib.rs 分工）

二进制项目常见结构：

- `src/main.rs`：参数/配置 + 调用 `run` + 统一打印错误 + 退出码
- `src/lib.rs`：`Config` / `run` / `search` 等可测试逻辑

### 4.1 `Config::build`：用 `Result` 表达“可能失败”

典型签名：

- **成功**：`Ok(Config { ... })`
- **失败**：`Err("not enough arguments")`（这里用静态字符串即可）

`main` 里把 `Err` 转成用户可读信息，并用非 0 退出码结束：

```rust
use std::process;
use std::env;
use minigrep::Config;

fn main() {
    let config = Config::build(env::args()).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {err}");
        process::exit(1);
    });

    if let Err(e) = minigrep::run(config) {
        eprintln!("Application error: {e}");
        process::exit(1);
    }
}
```

### 4.2 `run`：返回 `Result<(), Box<dyn Error>>` + `?`

典型写法（核心思想）：

- `run(...) -> Result<(), Box<dyn Error>>`
- `fs::read_to_string(...) ?`：失败就把错误“往上抛”
- 成功返回 `Ok(())`（只关心副作用：输出）

## 5. TDD：先写测试再写 `search`

TDD 循环（本章用来驱动 `search` 的实现）：

- 写失败测试 → 写最少代码让它通过 → 重构 → 重复

### 5.1 `search` 的最小实现思路

需求：输入 `query` + `contents`，返回所有匹配行。

步骤：

1) `contents.lines()` 逐行迭代  
2) `line.contains(query)` 判断  
3) 收集匹配行

### 5.2 为什么 `search` 需要显式生命周期

常见签名：

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();
    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }
    results
}
```

要点：

- 返回的 `&str` **来自 `contents` 的切片**，所以返回值必须和 `contents` 绑定同一生命周期 `'a`
- 如果不标注，编译器无法判断返回引用到底借用自哪个参数（`query` 还是 `contents`）

## 6. 环境变量：大小写不敏感搜索

约定：设置 `IGNORE_CASE` 即开启大小写不敏感。

### 6.1 读取环境变量：`env::var(...).is_ok()`

```rust
use std::env;

let ignore_case = env::var("IGNORE_CASE").is_ok(); // 只关心“是否设置”，不关心值
```

### 6.2 `search_case_insensitive` 的核心写法

- `query.to_lowercase()` 得到新 `String`
- `line.to_lowercase().contains(&query)`（`contains` 需要 `&str`）

### 6.3 运行示例

```bash
# Linux/macOS（只对这次命令生效）
IGNORE_CASE=1 cargo run -- to poem.txt
```

PowerShell：

```powershell
$Env:IGNORE_CASE=1; cargo run -- to poem.txt
# 取消
Remove-Item Env:IGNORE_CASE
```

## 7. 错误输出到 stderr：`eprintln!`

为什么：允许用户把“正常输出”重定向到文件，同时仍在屏幕上看到错误。

```bash
# 只重定向 stdout（stderr 仍在屏幕）
cargo run > output.txt
```

做法：把错误信息的 `println!` 改成 `eprintln!`：

- 正常结果用 `println!`（stdout）
- 错误信息用 `eprintln!`（stderr）

## 8. 完整代码

下面给出一个可工作的最终版本（`minigrep` 包含 `src/main.rs` + `src/lib.rs`，并带最小测试）。

### 8.1 `src/main.rs`

```rust
use std::env;
use std::process;

use minigrep::Config;

fn main() {
    let config = Config::build(env::args()).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {err}");
        process::exit(1);
    });

    if let Err(e) = minigrep::run(config) {
        eprintln!("Application error: {e}");
        process::exit(1);
    }
}
```

### 8.2 `src/lib.rs`

```rust
use std::env;
use std::error::Error;
use std::fs;

pub struct Config {
    pub query: String,
    pub file_path: String,
    pub ignore_case: bool,
}

impl Config {
    pub fn build(mut args: impl Iterator<Item = String>) -> Result<Config, &'static str> {
        // args[0] 通常是程序名/路径
        let _program = args.next();

        let query = match args.next() {
            Some(arg) => arg,
            None => return Err("not enough arguments"),
        };

        let file_path = match args.next() {
            Some(arg) => arg,
            None => return Err("not enough arguments"),
        };

        // 只关心是否设置，不关心值
        let ignore_case = env::var("IGNORE_CASE").is_ok();

        Ok(Config {
            query,
            file_path,
            ignore_case,
        })
    }
}

pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.file_path)?;

    let results = if config.ignore_case {
        search_case_insensitive(&config.query, &contents)
    } else {
        search(&config.query, &contents)
    };

    for line in results {
        println!("{line}");
    }

    Ok(())
}

pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    contents
        .lines()
        .filter(|line| line.contains(query))
        .collect()
}

pub fn search_case_insensitive<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let query = query.to_lowercase();

    contents
        .lines()
        .filter(|line| line.to_lowercase().contains(&query))
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn case_sensitive() {
        let query = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Duct tape.";

        assert_eq!(vec!["safe, fast, productive."], search(query, contents));
    }

    #[test]
    fn case_insensitive() {
        let query = "rUsT";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Trust me.";

        assert_eq!(
            vec!["Rust:", "Trust me."],
            search_case_insensitive(query, contents)
        );
    }
}
```
