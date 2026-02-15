# Rust 常见集合（TRPL 第 8 章速查：8.0~8.3）

只记录 **主要流程** 与 **关键知识点**（`Vec<T>` / `String`/`&str` / `HashMap<K, V>`）。

## 0. 使用方式（跑示例）

```bash
cargo new collections_demo
cd collections_demo
# edit src/main.rs
cargo run
```

（可选）更快检查：

```bash
cargo check
```

## 1. 集合是什么（ch08-00）

- 集合类型的数据在 **堆上**：可增长/可缩小，大小不必编译期已知
- 常用三类：`Vec<T>`（列表）、`String`（UTF-8 文本）、`HashMap<K,V>`（键值映射）

## 2. `Vec<T>`：可变长列表（ch08-01）

### 2.1 创建与更新

```rust
fn main() {
    let mut v: Vec<i32> = Vec::new();
    v.push(1);
    v.push(2);

    let w = vec![1, 2, 3];
    println!("{} {}", v.len(), w[0]);
}
```

### 2.2 读取元素：索引 vs `get`

- `v[i]`：返回 `T` 的引用；**越界会 panic**
- `v.get(i)`：返回 `Option<&T>`；越界返回 `None`（更安全）

```rust
fn main() {
    let v = vec![10, 20, 30];
    let a = &v[1];
    let b = v.get(1);
    println!("{a} {:?}", b);
}
```

### 2.3 借用规则常见坑：拿着引用时不能 `push`

- `push` 可能触发扩容/搬迁，导致已有引用失效；借用检查器会阻止这种代码。

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    let first = &v[0]; // 不可变借用 v 的某个元素
    // v.push(4);      // ❌ 编译错误：first 还在使用范围内，不能再可变借用 v
    println!("{first}");
}
```

### 2.4 遍历

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    for x in &v {
        println!("{x}");
    }
    for x in &mut v {
        *x += 50;
    }
}
```

### 2.5 存多种类型：用 enum 包一层

```rust
enum Cell {
    Int(i32),
    Float(f64),
    Text(String),
}

fn main() {
    let row = vec![Cell::Int(3), Cell::Text("hi".into()), Cell::Float(1.2)];
    let _ = row.len();
}
```

## 3. `String` / `&str`：UTF-8 文本（ch08-02）

### 3.1 先分清术语

- `&str`：字符串切片（借用视图），字符串字面值就是 `&'static str`
- `String`：可增长、可变、**拥有所有权** 的 UTF-8 字符串（标准库类型）

### 3.2 创建与追加

```rust
fn main() {
    let mut s = String::new();
    s.push_str("foo");
    s.push('b');

    let t = "bar".to_string();
    println!("{s} {t}");
}
```

### 3.3 拼接：`+` vs `format!`

- `s1 + &s2`：会 **move** 掉 `s1`（`add(self, &str) -> String`）；`&String` 会通过 deref coercion 变成 `&str`
- `format!`：更清晰；使用引用；不 move 掉参与拼接的字符串

```rust
fn main() {
    let s1 = String::from("Hello, ");
    let s2 = String::from("world");
    let s3 = s1 + &s2; // s1 moved（所有权被消费）；此后不能再访问 s1
    // println!("{s1}"); // ❌ 编译错误：use of moved value
    let s4 = format!("{s3}!"); // format! 使用借用，不会 move 掉 s3
    // s4 是一个全新的 String：会分配新的堆内存，把 s3 的内容格式化拷贝到 s4（两者不共享同一段堆内存）
    // println!("{s3}"); // ✅ 仍然可以访问 s3
    println!("{s2} {s4}");
}
```

### 3.4 为什么不能 `s[i]` 索引字符串

- `String` 底层是 `Vec<u8>`（UTF-8）；字节索引不等于“字符索引”
- 同一个“字符”可能占多个字节；索引还无法保证 O(1)
- 需要子串用 `&s[a..b]`（**按字节范围**，并且必须落在 UTF-8 字符边界，否则运行时 panic）

### 3.5 遍历：按字符 or 按字节

```rust
fn main() {
    for c in "Зд".chars() {
        println!("{c}");
    }
    for b in "Зд".bytes() {
        println!("{b}");
    }
}
```

## 4. `HashMap<K, V>`：键值对（ch08-03）

### 4.1 创建/插入/读取/遍历

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);

    let blue = scores.get("Blue").copied().unwrap_or(0);
    println!("{blue}");

    for (k, v) in &scores {
        println!("{k}: {v}");
    }
}
```

### 4.2 所有权：`insert` 可能 move

- `Copy` 类型会 copy 进 map
- 像 `String` 这种拥有所有权的值会被 move 进 map（之后原变量不能再用）

```rust
use std::collections::HashMap;

fn main() {
    // Copy：插入后原变量仍可用
    let x = 1;
    let mut m1: HashMap<&str, i32> = HashMap::new();
    m1.insert("k", x);
    println!("{x}"); // ✅ ok

    // Move：插入后原变量不可再用
    let key = String::from("team");
    let val = String::from("Blue");
    let mut m2: HashMap<String, String> = HashMap::new();
    m2.insert(key, val); // key/val moved
    // println!("{key} {val}"); // ❌ 编译错误：use of moved value
}
```

### 4.3 更新策略

- **覆盖旧值**：重复 `insert` 同一个 key
- **只在不存在时插入**：`entry(key).or_insert(default)`
- **基于旧值更新**：`*map.entry(k).or_insert(0) += 1`

返回值速记（常用部分）：

- `insert(k, v)`：返回 `Option<V>`（如果该 key 之前有旧值，则返回旧值；否则返回 `None`）
- `entry(k)`：返回 `Entry<K, V>`（枚举：`Occupied` 或 `Vacant`，表示该 key 是否已存在）
- `or_insert(default)`：返回 `&mut V`（若 key 不存在则先插入 `default`，并返回 map 中该值的可变引用）

```rust
use std::collections::HashMap;

fn main() {
    let text = "hello world wonderful world";
    let mut counts: HashMap<&str, u32> = HashMap::new();

    for w in text.split_whitespace() {
        let c = counts.entry(w).or_insert(0); // or_insert 返回该 key 对应 value 的可变引用：&mut V（无则插入 default）
        *c += 1;
    }

    println!("{counts:?}");
}
```

### 4.4 哈希函数

- `HashMap` 默认 hasher 更偏安全（抗 DoS），不一定最快；性能极端敏感时可显式指定其他 hasher（进阶）
