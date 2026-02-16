# ch19：模式（Patterns）速查

模式（pattern）用于**匹配值的结构/形状**并可在匹配时**解构**与**绑定变量**。常见于 `match`，也广泛出现在 `let`、`for`、函数参数等位置。

模式可以由这些元素组合：

- 字面值（literals）
- 解构的数组/枚举/结构体/元组
- 变量绑定
- 通配符 `_` / 占位符（如 `..`）

## 1. 模式出现的所有位置（All the places for patterns）

### 1.1 `match` 分支（必须穷尽）

```rust
match x {
    None => {}
    Some(i) => println!("{i}"),
}
```

- `match` **必须穷尽**（exhaustive）
- 常用最后一支 `_ => ...` 兜底；`_` 匹配任意值但**不绑定**

### 1.2 `if let` / `else if let`

适合只关心某一种（或少数）形状：

```rust
if let Some(v) = maybe {
    println!("{v}");
} else {
    println!("none");
}
```

对比 `match`：

- **不检查穷尽性**：漏分支编译器不提醒
- 模式里绑定的变量会**遮蔽**外层同名变量

### 1.3 `while let`

只要匹配就持续循环（常用于不断“取到 Ok/Some”为止）：

```rust
while let Ok(v) = rx.recv() {
    println!("{v}");
}
```

### 1.4 `for` 循环

`for PATTERN in ITER`，`PATTERN` 本身就是模式：

```rust
for (i, ch) in "abc".chars().enumerate() {
    println!("{i}: {ch}");
}
```

### 1.5 `let` 绑定

`let PATTERN = EXPR;`，包括解构：

```rust
let (x, y, z) = (1, 2, 3);
```

如果结构不匹配（比如元素个数对不上）会直接编译错误；可用 `_` / `..` 忽略：

```rust
let (x, _, z) = (1, 2, 3);
```

### 1.6 函数/闭包参数

参数位置也支持模式（包括解构）：

```rust
fn print_point(&(x, y): &(i32, i32)) {
    println!("({x}, {y})");
}
```

## 2. Refutability（可反驳性）：模式是否可能匹配失败

### 2.1 两类模式

- **irrefutable（不可反驳）**：对该类型的任意值都能匹配成功（例如 `x`、`(a, b)`）
- **refutable（可反驳）**：对某些值会匹配失败（例如 `Some(x)`、`Ok(v)`）

### 2.2 哪些位置要求哪一类

- **只能用不可反驳模式**：`let`、函数参数、`for`
  - 因为“不匹配”时没法做有意义的控制流分支
- **可用可反驳/不可反驳**：`if let`、`while let`
  - 但在 `if let`/`while let` 里用“不可反驳模式”会收到警告（因为它永远成功，失去条件判断意义）
- `match` 的分支模式通常是可反驳的；最后常用 `_` 兜底以满足穷尽性

### 2.3 典型报错与修复思路

在 `let` 用可反驳模式会报错：

```rust
// let Some(x) = some_option; // ❌ 可能是 None，匹配会失败
```

修复：换成 `if let` / `match`，给“不匹配”留后路：

```rust
if let Some(x) = some_option {
    println!("{x}");
}
```

## 3. 模式语法速查（Pattern syntax）

### 3.1 匹配字面值

```rust
match x {
    1 => println!("one"),
    _ => println!("other"),
}
```

### 3.2 变量绑定与遮蔽（named variables）

在 `match/if let/while let` 的模式里绑定的名字会进入新作用域并可能遮蔽外层：

```rust
let x = Some(5);
let y = 10;

match x {
    Some(y) => println!("shadow y = {y}"), // 这里 y 绑定的是 5
    None => println!("none"),
}

println!("outer y = {y}"); // 仍是 10
```

### 3.3 多个模式（`|`）

```rust
match x {
    1 | 2 => println!("one or two"),
    _ => {}
}
```

### 3.4 范围匹配（`..=`，闭区间）

仅适用于能在编译期判断范围是否为空的类型（数字与 `char`）：

```rust
match x {
    1..=5 => println!("1 to 5"),
    _ => {}
}
```

### 3.5 解构（destructuring）

结构体字段简写：

```rust
struct Point { x: i32, y: i32 }
let p = Point { x: 3, y: 5 };
let Point { x, y } = p;
```

带字面值的结构体匹配：

```rust
match p {
    Point { x, y: 0 } => println!("on x-axis at {x}"),
    Point { x: 0, y } => println!("on y-axis at {y}"),
    Point { x, y } => println!("else ({x}, {y})"),
}
```

枚举/嵌套解构（形状要对应变体的数据形态）：

```rust
enum Color { Rgb(i32, i32, i32), Hsv(i32, i32, i32) }
enum Message { ChangeColor(Color) }

let msg = Message::ChangeColor(Color::Rgb(0, 160, 255));
match msg {
    Message::ChangeColor(Color::Rgb(r, g, b)) => println!("{r} {g} {b}"),
    Message::ChangeColor(Color::Hsv(h, s, v)) => println!("{h} {s} {v}"),
}
```

### 3.6 忽略值（ignore）

- **忽略整个值**：`_`
- **忽略部分值**：在结构里嵌 `_`
- **忽略剩余部分**：`..`（必须无歧义）

```rust
let (x, _, z) = (1, 2, 3);

struct P3 { x: i32, y: i32, z: i32 }
match P3 { x: 1, y: 2, z: 3 } {
    P3 { x, .. } => println!("x = {x}"),
}

let (first, .., last) = (1, 2, 3, 4);
```

下划线开头变量名 vs `_` 的区别（是否绑定/是否 move）：

```rust
let s = String::from("hi");
let _t = s; // ✅ 绑定并 move 了 s（s 之后不可用）
// println!("{s}"); // ❌ use of moved value
```

```rust
let s = String::from("hi");
let _ = s; // ✅ 不绑定名字；这里也不会产生“未使用变量”警告
println!("{s}"); // ✅ `s` 仍可用：`_` 不会绑定值，因此不会像 `_t` 那样把所有权 move 走
```

### 3.7 匹配守卫（match guard）

在 `match` 分支上加额外 `if` 条件（只在 `match` 可用）：

```rust
match num {
    Some(x) if x % 2 == 0 => println!("even {x}"),
    Some(x) => println!("odd {x}"),
    None => {}
}
```

也常用于避免“模式绑定遮蔽外层变量”，改用新绑定 + guard 比较外层变量。

### 3.8 `@` 绑定：边测试边绑定

```rust
enum Message { Hello { id: i32 } }

match msg {
    Message::Hello { id: id_var @ 3..=7 } => println!("in range: {id_var}"),
    Message::Hello { id: 10..=12 } => println!("10..=12"),
    Message::Hello { id } => println!("other: {id}"),
}
```

`id_var @ 3..=7` 的含义：值必须落在范围内，同时把该值绑定到 `id_var`。
