# Rust 枚举与模式匹配（TRPL 第 6 章速查：6.0~6.3）

只记录 **主要流程** 与 **关键知识点**（enum / `Option<T>` / `match` / `if let` / `let...else`）。

## 0. 使用方式（跑示例）

```bash
cargo new enums_demo
cd enums_demo
# edit src/main.rs
cargo run
```

（可选）更快检查：

```bash
cargo check
```

## 1. 枚举是什么（ch06-00）

- **enum**：用一组可枚举的 **变体（variants）** 定义一个类型；一个值在任意时刻只会是其中一个变体。
- 适合表达“某个值只能是几种情况之一”，并且可以让每个变体携带不同数据。

## 2. 定义枚举与携带数据（ch06-01）

### 2.1 基本定义 + 命名空间

```rust
enum IpAddrKind {
    V4,
    V6,
}

fn route(ip_kind: IpAddrKind) {
    match ip_kind {
        IpAddrKind::V4 => println!("v4"),
        IpAddrKind::V6 => println!("v6"),
    }
}
```

- 变体在类型命名空间里：`IpAddrKind::V4`

### 2.2 变体携带数据（比“enum + struct”更直接）

```rust
enum IpAddr {
    V4(String),
    V6(String),
}

fn main() {
    let _home = IpAddr::V4(String::from("127.0.0.1"));
    let _loopback = IpAddr::V6(String::from("::1"));
}
```

- 变体构造器像函数：`IpAddr::V4(...)`
- 每个变体可以携带不同类型/不同数量的数据（例如 `V4(u8,u8,u8,u8)` 与 `V6(String)`）

### 2.3 `Option<T>`：用类型系统替代 Null

- `Option<T>` 表示“有值/无值”：
  - `Some(T)`：有值
  - `None`：无值
- `Option<T>` 与 `T` 是不同类型：你必须显式处理 `None`，不能把 `Option<i32>` 当 `i32` 用。

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}
```

## 3. `match`：模式匹配与穷尽性（ch06-02）

### 3.1 `match` 是表达式

- 分支写法：`pattern => expr,`
- `match` 的结果值是“被匹配分支”的结果值

### 3.2 绑定值（从变体里取出数据）

```rust
enum Coin {
    Penny,
    Quarter(String), // 简化：用 String 代表州名
}

fn value(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Quarter(state) => {
            println!("state: {state}");
            25
        }
    }
}
```

### 3.3 穷尽性 + 通配 `_`

- **必须覆盖所有情况**（exhaustive），否则编译报错
- 不关心某些值时用 `_`：

```rust
fn main() {
    let n = 4u8;
    match n {
        3 => println!("hat"),
        7 => println!("lose hat"),
        _ => (), // 忽略其他值
    }
}
```

## 4. `if let` / `let...else`：处理单一模式的简写（ch06-03）

### 4.1 `if let`：少写样板，但少了穷尽性强制检查

```rust
fn main() {
    let config_max: Option<u8> = Some(3);
    if let Some(max) = config_max {
        println!("max = {max}");
    }
}
```

- 可配 `else`：相当于 `match` 的 `_` 分支

### 4.2 `let...else`：保持“愉快路径”（Happy Path）

- 模式匹配成功：把绑定值带到外层作用域继续用
- 不匹配：进入 `else`，并且 **必须提前返回/中断流程**（常见用 `return`）

```rust
fn get_port(env: Option<u16>) -> u16 {
    let Some(p) = env else {
        return 8080;
    };
    p
}
```
