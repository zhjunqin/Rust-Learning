# ch14：进一步认识 Cargo / crates.io 速查

只记录：**release profile**、**文档与发布**、**workspaces**、**install 二进制**、**扩展 cargo 子命令**。

## 1. Release profiles（发布配置）

Cargo 常用两套 profile：

- **dev**：`cargo build`（默认：`opt-level = 0`，编译快）
- **release**：`cargo build --release`（默认：`opt-level = 3`，运行快）

在 `Cargo.toml` 里覆盖默认设置（只写你想改的项即可）：

```toml
[profile.dev]
opt-level = 1

[profile.release]
opt-level = 3
```

常用命令：

```bash
cargo build
cargo build --release
```

## 2. 文档（rustdoc）与文档测试

### 2.1 文档注释：`///` 与 `//!`

- `///`：给“后面的项”写文档（函数/结构体/trait/模块等），支持 Markdown
- `//!`：给“当前容器项”写文档（crate 根 `src/lib.rs` 或模块根文件），用于整体介绍

### 2.2 生成并打开文档

```bash
cargo doc
cargo doc --open
```

输出目录：`target/doc/`。

### 2.3 文档注释会被当成测试（doctest）

- 文档里的示例代码块会被 `cargo test` 执行（帮助保证文档与代码同步）

```bash
cargo test
```

### 2.4 文档里常见小节（速记）

- **Examples**
- **Panics**
- **Errors**
- **Safety**（涉及 `unsafe` 时）

## 3. 设计对用户友好的公有 API：`pub use` 重导出

当内部模块层级很深时，可以用 `pub use` 把“用户常用类型/函数”重导出到更浅的路径：

```rust
// 例：把 crate::kinds::PrimaryColor 以 crate::PrimaryColor 的形式暴露
pub use crate::kinds::PrimaryColor;
```

效果：内部组织不必为了 API 用户而强行重排，但用户能更容易 `use your_crate::PrimaryColor;`。

## 4. 发布到 crates.io

### 4.1 登录（保存 token）

```bash
cargo login
# 然后粘贴 crates.io 的 API token
```

- token 会保存在 `~/.cargo/credentials`，**属于秘密信息**（泄露要立即撤销并重生成）。

### 4.2 `Cargo.toml` 元数据（发布前必备）

最关键：

- `name`：必须在 crates.io 唯一（先到先得）
- `description`
- `license`（建议用 SPDX 标识符，比如 `MIT OR Apache-2.0`）或 `license-file`

示例：

```toml
[package]
name = "guessing_game"
version = "0.1.0"
edition = "2024"
description = "A fun game where you guess what number the computer has chosen."
license = "MIT OR Apache-2.0"
```

### 4.3 发布 / 版本更新（语义化版本）

```bash
cargo publish
```

要点：

- 发布是**永久性的**：同一版本不能覆盖、不能删除（可以发布新版本号）。
- 更新版本：改 `Cargo.toml` 的 `version`，按 semver 递增，然后再次 `cargo publish`。

### 4.4 撤回（yank）：阻止新项目使用某版本（但不破坏已有 lock）

```bash
cargo yank --vers 1.0.1
cargo yank --vers 1.0.1 --undo
```

- yank 不能删除已发布代码；如果误传秘密信息，仍需立刻更换秘密信息。

## 5. Cargo workspaces（工作空间）

工作空间要点：

- 多个包共享同一个 **`Cargo.lock`** 与 **顶层 `target/`**
- 避免成员 crate 之间重复编译依赖

### 5.1 创建工作空间（根 `Cargo.toml`）

```bash
mkdir add && cd add
```

根 `Cargo.toml`（workspace 根不写 `[package]`）：

```toml
[workspace]
members = ["adder", "add_one"]
resolver = "3"
```

（`cargo new` 在 workspace 里创建成员时，通常会自动把新成员加入 `members`。）

### 5.2 在 workspace 中构建/运行/测试

```bash
# 在 workspace 根目录
cargo build

# 运行指定包（常见：workspace 有多个 binary）
cargo run -p adder

# 跑整个 workspace 的测试
cargo test --workspace

# 只跑某个成员的测试
cargo test -p add_one
```

### 5.3 workspace 内依赖：路径依赖要显式声明

workspace 不会“自动假定成员互相依赖”，需要在成员的 `Cargo.toml` 中写：

```toml
[dependencies]
add_one = { path = "../add_one" }
```

### 5.4 外部依赖：锁版本共享，但每个 crate 仍需自己声明

- workspace 只有一个 `Cargo.lock` ⇒ 版本解析尽量统一
- 但 **某 crate 想用外部依赖，仍必须在自己的 `Cargo.toml` 写依赖**（不因别的成员用了就自动可用）

### 5.5 workspace 发布

- workspace 内的 crate **需要逐个发布**
- 可用 `-p` 指定发布哪个 crate（类似运行/测试）

## 6. 安装二进制：`cargo install`

```bash
cargo install ripgrep
```

要点：

- 只能安装包含 binary target 的 crate（有 `src/main.rs` 或其他 `[[bin]]`）
- 默认安装到 `~/.cargo/bin`，确保它在 `$PATH` 中（比如安装 `rg` 后可 `rg --help`）。

## 7. 扩展 Cargo：自定义子命令

规则：

- 只要 `$PATH` 里存在可执行文件 `cargo-something`，就能用 `cargo something` 运行

查看已安装子命令：

```bash
cargo --list
```

很多扩展子命令可以直接通过 `cargo install cargo-something` 安装后使用。