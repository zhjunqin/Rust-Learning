# ch11：测试（Testing）速查

只记录：**怎么写测试**、**怎么运行/过滤**、**怎么组织单元/集成测试**。

## 1. 基本心智模型

- 测试用来验证：代码是否按预期工作（类型系统/借用检查无法证明“逻辑正确”）。
- `cargo test` 会以 **test profile** 编译并运行“测试二进制”：
  - 默认 **并行** 跑测试
  - 默认 **捕获 stdout**（通过的测试里 `println!` 不会显示）

## 2. 写单元测试（unit tests）

### 2.1 最小模板

```rust
pub fn add_two(x: i32) -> i32 { x + 2 }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_adds_two() {
        assert_eq!(add_two(3), 5);
    }
}
```

- `#[test]`：标记测试函数；`cargo test` 会收集并执行。
- `#[cfg(test)]`：只在 `cargo test` 时编译该模块；`cargo build`/`cargo run` 不编译其中代码。
- `use super::*;`：在 `tests` 模块中引入被测模块的项（常见做法）。

### 2.2 常用断言宏

```rust
#[test]
fn asserts() {
    let ok = 2 + 2 == 4;
    assert!(ok);

    assert_eq!(1 + 1, 2);
    assert_ne!(1 + 1, 3);

    // 自定义失败信息（额外参数会交给 format!）
    let got = 3;
    assert_eq!(got, 4, "expected 4, got {got}");
}
```

补充：

- `assert_eq!` / `assert_ne!` 失败时会打印 `left`/`right` 的 Debug 值 ⇒ 被比较类型通常需要实现 `Debug`（自定义类型常用 `#[derive(Debug, PartialEq)]`）。

### 2.3 断言会 panic：`#[should_panic]`

```rust
pub struct Guess { value: u32 }
impl Guess {
    pub fn new(value: u32) -> Self {
        if !(1..=100).contains(&value) {
            panic!("Guess value must be between 1 and 100, got {value}");
        }
        Self { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn rejects_too_large() {
        Guess::new(200);
    }

    #[test]
    #[should_panic(expected = "between 1 and 100")]
    fn rejects_with_message() {
        Guess::new(200);
    }
}
```

- `#[should_panic]`：测试 **发生 panic 才算通过**。
- `expected = "..."`：要求 panic 信息包含该子串（让测试更精确）。

### 2.4 用 `Result` 写测试（便于 `?`）

```rust
#[test]
fn result_test() -> Result<(), String> {
    let v = 2 + 2;
    if v == 4 { Ok(()) } else { Err(format!("got {v}")) }
}
```

- 返回 `Result` 的测试里可以用 `?` 串起来（例如文件 IO 等）。
- 这类测试 **不能** 与 `#[should_panic]` 同用。
- 想断言某操作返回 `Err`：用 `assert!(value.is_err())`，不要写 `value?`（否则会提前返回）。

## 3. 运行测试（`cargo test`）

### 3.1 基本命令

```bash
cargo test
```

### 3.2 Cargo 参数 vs 测试二进制参数：用 `--` 分隔

```bash
# cargo 自己的帮助
cargo test --help

# 测试二进制的帮助（-- 后）
cargo test -- --help
```

### 3.3 并行/顺序：`--test-threads`

```bash
# 串行跑（避免共享状态互相干扰，如读写同名文件）
cargo test -- --test-threads=1
```

### 3.4 输出捕获与显示：`--show-output`

```bash
# 默认：通过的测试 stdout 会被捕获不显示；失败的会显示输出
cargo test

# 显示通过测试的 stdout
cargo test -- --show-output
```

### 3.5 过滤：按名称跑部分测试

```bash
# 名称/模块路径包含该字符串的测试会运行
cargo test add

# 只跑某个集成测试文件（tests/foo.rs）
cargo test --test foo
```

### 3.6 忽略耗时测试：`#[ignore]` / `--ignored`

```rust
#[test]
#[ignore]
fn expensive_test() {
    // ...
}
```

```bash
# 默认不跑 ignore 的测试
cargo test

# 只跑 ignore 的测试
cargo test -- --ignored

# 包含 ignore 在内，跑全部
cargo test -- --include-ignored
```

## 4. 组织结构：单元测试 vs 集成测试

### 4.1 单元测试（unit）

- **位置**：与源码同文件（`src/...`）里，惯例 `#[cfg(test)] mod tests { ... }`
- **特点**：可测试 **私有** 函数/模块（同 crate 内）

### 4.2 集成测试（integration）

- **位置**：项目根目录的 `tests/`（与 `src/` 同级）
- **特点**：
  - `tests/` 下 **每个 `.rs` 文件** 都会被当成一个独立 crate 编译
  - 只能通过 **公有 API** 使用你的库（像真正用户一样）

目录与最小示例：

```text
my_crate
├── Cargo.toml
├── src
│   └── lib.rs
└── tests
    └── integration_test.rs
```

```rust
// tests/integration_test.rs
use my_crate::add_two;

#[test]
fn it_adds_two() {
    assert_eq!(add_two(2), 4);
}
```

### 4.3 集成测试共享代码：`tests/common/mod.rs`

如果写成 `tests/common.rs`，它会被当作一个“集成测试 crate”并出现在输出中（`running 0 tests`）。
常见做法是改成模块目录：

```text
tests
├── common
│   └── mod.rs
└── integration_test.rs
```

```rust
// tests/common/mod.rs
pub fn setup() {
    // ...
}
```

```rust
// tests/integration_test.rs
mod common;

#[test]
fn uses_setup() {
    common::setup();
    assert!(true);
}
```

### 4.4 二进制 crate 的集成测试

- 只有 `src/main.rs`（没有 `src/lib.rs`）的**纯二进制 crate**：集成测试没法像库那样 `use` 其内部函数。
- 常见结构：把主要逻辑放到 `src/lib.rs`，`src/main.rs` 只做少量调用；这样集成测试就能测试库部分。