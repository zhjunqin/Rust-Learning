# Rust 所有权（TRPL 第 4 章速查：4.1~4.3）

只记录 **主要流程** 与 **关键规则**，配最小示例帮助记忆（所有权 / 引用与借用 / slice）。

## 0. 使用方式（跑示例）

```bash
cargo new ownership
cd ownership
# edit src/main.rs
cargo run
```

（可选）更快的检查：

```bash
cargo check
```

## 1. 所有权（ch04-01）

### 1.1 三条规则（背下来）

1. 每个值都有且只有一个 **所有者**（owner）
2. 任一时刻只能有 **一个所有者**
3. 所有者离开作用域（scope）时，值会被 **drop**

### 1.2 栈/堆与为什么需要所有权（只记结论）

- **栈**：大小固定、拷贝快
- **堆**：大小可变/运行时分配，用指针间接访问；释放时机必须正确
- Rust 用编译期规则决定何时释放堆数据（运行时不额外付出 GC 成本）

### 1.3 `String`：move vs clone vs Copy

- `String`（堆上数据）：赋值/传参默认是 **move**（移动），原变量不能再用
- 需要深拷贝堆数据时用 `clone()`
- 仅栈上数据（如整数、bool、char、仅含 Copy 成员的 tuple）通常实现 `Copy`：赋值/传参是 **拷贝**

```rust
fn main() {
    // move: String
    let s1 = String::from("hello");
    let s2 = s1;
    // println!("{s1}"); // error: use of moved value
    println!("{s2}");

    // clone: deep copy
    let a = String::from("hi");
    let b = a.clone();
    println!("{a} {b}");

    // Copy: stack-only
    let x = 5;
    let y = x;
    println!("{x} {y}");
}
```

补充解释：当 `let s2 = s1;` 时底层发生了什么？

- **`String` 本体在栈上**，通常可以理解为 3 个字段：`ptr`（指向堆数据）、`len`、`cap`。
- `let s2 = s1;` **只会复制栈上的这 3 个字段** 到 `s2`，也就是“复制指针/长度/容量”；**不会复制堆上的字符串内容**。
- 此时若两者都还有效，它们会指向同一块堆内存；两者离开作用域都会尝试 `drop`，就会造成 **double free**。
- Rust 选择的策略是 **move**：完成赋值后把 `s1` 标记为已被移动（因此 `s1` 不能再用），只剩 `s2` 拥有那块堆内存；最终由 `s2` 在离开作用域时负责释放。

（如果你确实要“复制堆上的内容”，用 `clone()`，它会分配新的堆内存并拷贝数据。）

### 1.4 函数传参/返回与所有权

- 传 `String` 进函数：通常 **move**
- 传 `Copy` 类型进函数：**copy**
- 从函数返回值：也会发生所有权转移（move）

```rust
fn takes(s: String) {
    println!("{s}");
}

fn gives() -> String {
    String::from("owned")
}

fn main() {
    takes(String::from("moved"));
    let s = gives();
    println!("{s}");
}
```

## 2. 引用与借用（ch04-02）

### 2.1 借用：用 `&T` 读取但不拿走所有权

- `&T`：不可变引用（默认不可改）
- `&mut T`：可变引用（允许修改）

```rust
fn len(s: &String) -> usize {
    s.len()
}

fn main() {
    let s = String::from("hello");
    let n = len(&s);
    println!("{s} {n}");
}
```

### 2.2 引用规则（背下来）

- 任意时刻：
  - **要么** 1 个可变引用 `&mut T`
  - **要么** 多个不可变引用 `&T`
- 引用必须始终有效（Rust 禁止悬垂引用）
- 作用域以“最后一次使用”为准：不可变引用用完后，才允许创建可变引用

```rust
fn main() {
    let mut s = String::from("hi");
    let r1 = &s;
    let r2 = &s;
    println!("{r1} {r2}"); // r1/r2 last use here

    let r3 = &mut s; // ok: r1/r2 no longer used
    r3.push('!');
    println!("{r3}");
}
```

补充解释（理解 `let r3 = &mut s;` 之后 `s` 的状态）：

- `s` **仍然拥有**这个 `String`（所有权没变），只是此时 `s` 被 **可变借用** 给了 `r3`。
- 在 `r3` 这个可变借用还“活着”的这段范围内，你**不能**再通过 `s` 去读/写/再次借用（不论 `&s` 还是 `&mut s`），只能通过 `r3` 来修改。
- 当 `r3` 的**最后一次使用**结束后（此例是 `println!("{r3}")`），借用结束，你又可以正常使用 `s`（包括修改）。

```rust
fn main() {
    let mut s = String::from("hi");
    let r1 = &s;
    let r2 = &s;
    println!("{r1} {r2}");

    let r3 = &mut s;
    r3.push('!');
    // s.push('?'); // 不允许：r3 仍在使用范围内
    println!("{r3}"); // r3 最后一次使用

    s.push('?'); // 允许：r3 已经不再使用
    println!("{s}");
}
```

### 2.3 悬垂引用（dangling reference）

- Rust 不允许返回“指向已被释放局部变量”的引用；通常做法是 **直接返回拥有所有权的值**（例如返回 `String`）

错误示例（**不能编译**：返回了局部变量 `s` 的引用）：

```rust
fn dangle() -> &String {
    let s = String::from("hello");
    &s // s 离开作用域会被 drop，这个引用会悬垂
}
```

正确写法（返回拥有所有权的值）：

```rust
fn no_dangle() -> String {
    String::from("hello")
}
```

## 3. Slice（切片，ch04-03）

### 3.1 `&str`：字符串切片（不拥有所有权）

- 字符串字面值类型就是 `&str`
- slice 引用底层数据的一段连续区间：`&s[a..b]`、`&s[..b]`、`&s[a..]`、`&s[..]`
- **注意 UTF-8 边界**：字符串切片索引必须落在合法字符边界，否则会 panic

### 3.2 用 slice 写更好的 API：`&String` → `&str`

- `fn first_word(s: &str) -> &str` 更通用：既能接收 `&String` 也能接收 `&str`

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &b) in bytes.iter().enumerate() {
        if b == b' ' {
            return &s[..i];
        }
    }
    s
}

fn main() {
    let s = String::from("hello world");
    let w = first_word(&s);       // &String -> &str
    let lit = first_word("hello"); // &str
    println!("{w} {lit}");
}
```

### 3.3 其他 slice：`&[T]`

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];
    let s: &[i32] = &a[1..3];
    assert_eq!(s, &[2, 3]);
}
```
