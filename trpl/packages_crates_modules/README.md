# 包 / Crate / 模块系统（TRPL 第 7 章速查：7.0~7.5）

只记录 **主要流程** 与 **关键规则**（package/crate/module/路径/`use`/拆分文件/可见性）。

## 0. 使用方式（跑示例）

```bash
cargo new restaurant --lib
cd restaurant
# edit src/lib.rs / src/main.rs
cargo test
```

（可选）更快检查：

```bash
cargo check
```

## 1. 核心概念速记（ch07-00, ch07-01）

- **Package**：一个或多个 crate 的捆绑（有 `Cargo.toml`），用于构建/测试/发布。
- **Crate**：Rust 编译时的最小代码单位（binary crate / library crate）。
- **Crate root**：crate 的根模块入口文件：
  - binary：通常 `src/main.rs`
  - lib：通常 `src/lib.rs`
- **一个 package 的约定**：
  - `src/main.rs` 存在 ⇒ 有一个与包同名的 binary crate
  - `src/lib.rs` 存在 ⇒ 有一个与包同名的 library crate（最多 1 个）
  - `src/bin/*.rs` ⇒ 每个文件都是一个独立的 binary crate

## 2. 模块（module）与模块树（ch07-02）

- `mod name { ... }`：定义模块（内联）
- `mod name;`：声明模块（告诉编译器去对应文件加载代码）
- crate 的模块树根是隐式的 `crate` 模块（crate root 文件内容会成为根的一部分）

### 2.1 模块与私有性（privacy）

- **默认私有**：模块内的项默认对父模块私有
- 父模块不能访问子模块的私有项；子模块可以访问父模块内的项
- `pub mod` 只让“模块名”可见，模块内部的项仍需按需 `pub`

## 3. 路径（path）：如何引用模块树中的项（ch07-03）

路径两种形式：

- **绝对路径**：从 crate 根开始（当前 crate 用 `crate::...`；外部 crate 用 `crate_name::...`）
- **相对路径**：从当前模块开始（`self::...` / `super::...` / `ident::...`）

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // 绝对路径
    crate::front_of_house::hosting::add_to_waitlist();
    // 相对路径（同级里能看到 front_of_house）
    front_of_house::hosting::add_to_waitlist();
}
```

### 3.1 `super`：从父模块起步的相对路径

```rust
fn deliver_order() {}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::deliver_order();
    }
    fn cook_order() {}
}
```

### 3.2 `pub` 的几个常见坑

- `pub mod hosting` 只让父模块“看见 hosting”；要调用 `add_to_waitlist` 还需要 `pub fn add_to_waitlist`.
- **struct vs enum**：
  - `pub struct`：结构体公有，但字段仍默认私有（字段要单独 `pub`）
  - `pub enum`：枚举公有，则其变体默认也公有

## 4. `use`：把路径引入作用域（ch07-04）

### 4.1 `use` 只影响当前作用域

- `use` 相当于在当前作用域创建“快捷方式”，不影响模块树结构，也不决定哪些文件会被编译进来。

### 4.2 惯用写法（重要）

- **引入函数**：惯用做法是引入父模块，然后 `module::func()` 调用，避免丢失来源信息。
- **引入类型（struct/enum 等）**：惯用做法是引入到类型名（完整路径到类型）。

```rust
use std::collections::HashMap; // 类型：引到 HashMap

mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting; // 函数：引到 hosting 模块

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    let mut m: HashMap<String, i32> = HashMap::new();
    m.insert("k".into(), 1);
}
```

### 4.3 重命名 / 重导出 / 合并 use

- **`as`**：解决同名冲突
- **`pub use`**：重导出（让外部用更短的路径访问你的 API）
- **嵌套路径**：减少重复
- **glob `*`**：一次引入很多（测试里常见；业务代码谨慎）

```rust
use std::fmt::Result;
use std::io::Result as IoResult;

pub use crate::front_of_house::hosting; // 对外暴露 restaurant::hosting::...

use std::{cmp::Ordering, io}; // 嵌套路径
use std::collections::*; // glob（谨慎）
```

## 5. 拆分模块到不同文件（ch07-05）

核心规则：**在模块树里某处 `mod xxx;` 声明一次即可**，之后用模块路径引用；`mod` 不是“include”。

常见路径（新风格；旧的 `mod.rs` 仍支持但不推荐混用）：

- crate 根声明 `mod front_of_house;`
  - `src/front_of_house.rs`  或 `src/front_of_house/mod.rs`（旧）
- 在 `front_of_house` 里声明子模块 `mod hosting;`
  - `src/front_of_house/hosting.rs` 或 `src/front_of_house/hosting/mod.rs`（旧）

```text
src/
  lib.rs
  front_of_house.rs
  front_of_house/
    hosting.rs
```
