# Rust 入门（TRPL 第 1 章速查）

只记录 **主流程** 与 **常用命令示例**（对应 TRPL：ch01-00~ch01-03）。

## 0. 总览（你将完成什么）

- **安装工具链**：用 `rustup` 安装/更新 Rust，并能打开本地文档
- **跑通 Hello World（rustc）**：写 `main.rs`，用 `rustc` 编译，再运行可执行文件
- **用 Cargo 管项目**：`cargo new/build/run/check`，理解项目结构与构建产物目录

## 1. 安装 Rust（rustup）

- **安装（Linux/macOS）**：

```bash
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
```

- **Windows**：按官方安装器流程安装（通常需要 MSVC 构建工具链/Visual Studio 组件）

- **验证安装**：

```bash
rustc --version
```

- **更新 / 卸载**：

```bash
rustup update
rustup self uninstall
```

- **本地文档（离线查标准库/API）**：

```bash
rustup doc
```

- **构建环境提示**：可能需要链接器/构建工具链（macOS：`xcode-select --install`；Ubuntu：安装 `build-essential` 等）

- **可选：提前缓存依赖，后续离线使用**：

```bash
cargo new get-dependencies
cd get-dependencies
cargo add rand@0.8.5 trpl@0.2.0
# later: cargo ... --offline
```

## 2. Hello, World（直接用 rustc）

- **创建目录（示例路径）**：

```bash
mkdir -p ~/projects/hello_world
cd ~/projects/hello_world
```

- **写最小程序**（`main.rs`）：

```rust
fn main() {
    println!("Hello, world!");
}
```

- **编译与运行是两步**：

```bash
rustc main.rs
./main
# Windows: .\main.exe
```

- **关键点速记**：
  - `fn main()` 是可执行程序入口
  - `println!` 带 `!`：它是宏（macro），不是普通函数
  - 多数语句行以 `;` 结尾
  - Rust 是**预编译**语言：先产出可执行文件，再运行

## 3. Hello, Cargo（用 Cargo 管项目）

- **检查 Cargo**：

```bash
cargo --version
```

- **创建项目**：

```bash
cargo new hello_cargo
cd hello_cargo
```

- **项目结构要点**：
  - `Cargo.toml`：项目配置（`[package]` / `[dependencies]` 等）
  - `src/main.rs`：源码放在 `src/`
  - 构建产物默认在 `target/` 下（debug：`target/debug/`）

- **构建 / 运行 / 快速检查**：

```bash
cargo build
./target/debug/hello_cargo

cargo run

cargo check
```

- **依赖锁文件**：首次构建会生成 `Cargo.lock`，记录依赖解析到的实际版本（通常无需手动编辑）

- **发布构建（优化，产物在 `target/release/`）**：

```bash
cargo build --release
```

- **常见上手三步（已有项目）**：

```bash
git clone example.org/someproject
cd someproject
cargo build
```