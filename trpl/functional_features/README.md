# ch13：函数式特性（闭包 & 迭代器）速查

只记录：闭包捕获/`Fn*`、迭代器模型（惰性、适配器/消费者）、用迭代器重构 `minigrep`、性能结论（零成本抽象）。

## 1. 闭包（closure）

### 1.1 语法与用途

- 闭包：**匿名函数**，可存入变量、作为参数/返回值；并且**可捕获环境**。
- 常见语法（从“更像函数”到“更简洁”）：

```rust
fn add_one_v1(x: u32) -> u32 { x + 1 }
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x| { x + 1 };
let add_one_v4 = |x| x + 1;
```

### 1.2 类型推断：一次确定后就“锁死”

- 闭包参数/返回值通常由编译器从**第一次使用**推断出具体类型。
- 同一个闭包值不能用两种不同类型去调用（会类型不匹配）。

### 1.3 捕获环境：借用 / 可变借用 / 移动所有权

闭包捕获方式与函数参数三种方式对应：

- **不可变借用**：只读用到外部变量
- **可变借用**：闭包里修改外部变量（闭包活跃期间会占用可变借用）
- **获取所有权**：需要把值移进闭包（常见：跨线程）

```rust
fn main() {
    let mut list = vec![1, 2, 3];

    let borrows = || println!("{list:?}"); // &list
    borrows();

    let mut borrows_mut = || list.push(4); // &mut list
    borrows_mut();
    println!("{list:?}");
}
```

强制 move：

```rust
use std::thread;

fn main() {
    let list = vec![1, 2, 3];
    thread::spawn(move || {
        // list 的所有权被移动进新线程的闭包里
        println!("{list:?}");
    }).join().unwrap();
}
```

### 1.4 `FnOnce` / `FnMut` / `Fn`

闭包体对捕获值的“处理方式”决定它实现哪些 trait（从弱到强）：

- **`FnOnce`**：只能调用一次（闭包体把捕获值**移出**了闭包）
- **`FnMut`**：可多次调用，但会**修改**捕获值（捕获 `&mut` 或内部可变）
- **`Fn`**：可多次调用且不修改/不移出捕获值（也包含“不捕获环境”的闭包/函数）

最小示例（分别对应 3 种）：

**`FnOnce`**（把捕获的值 move 出闭包，因此只能调用一次）：

```rust
fn main() {
    let s = String::from("hello");

    // move 进闭包；闭包体里把 s 消费掉（move 出闭包环境）
    let consume_once = move || drop(s);

    consume_once();        // ✅ 第一次调用 OK
    // consume_once();     // ❌ 不能再调用：闭包只能调用一次（FnOnce）
}
```

**`FnMut`**（会修改捕获的值；调用闭包通常需要 `mut` 绑定）：

```rust
fn main() {
    let mut count = 0;

    // 捕获 &mut count，并修改它
    let mut inc = || {
        count += 1;
    };

    inc();
    inc();
    assert_eq!(count, 2);
}
```

**`Fn`**（只读捕获/不捕获；可多次调用且不改变环境）：

```rust
fn main() {
    let x = 10;

    // 只读捕获 x（不可变借用），闭包体不修改也不 move 出捕获值
    let plus_one = || x + 1;

    assert_eq!(plus_one(), 11);
    assert_eq!(plus_one(), 11); // ✅ 可重复调用（Fn）
}
```

标准库签名例子（理解为什么需要 `FnOnce`）：

```rust
// Option::unwrap_or_else 只需要“最多调用一次”闭包
// pub fn unwrap_or_else<F>(self, f: F) -> T where F: FnOnce() -> T
```

再对比：`slice::sort_by_key` 会对每个元素调用一次闭包，因此需要至少 `FnMut`。

补充：

- 若不需要捕获环境，很多场景可以直接传函数指针/关联函数：例如 `unwrap_or_else(Vec::new)`。

## 2. 迭代器（iterator）

### 2.1 惰性（lazy）

- 迭代器适配器（如 `map` / `filter`）本身**不执行**；要等到“消费者”消费时才执行。
- 常见消费者：`for`、`collect`、`sum`（以及很多会反复调用 `next()` 的方法）。

### 2.2 `Iterator` trait：只要求实现 `next`

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

要点：

- `next(&mut self)` 会推进迭代器内部状态，所以迭代器变量通常要 `mut`。

### 2.3 `iter` / `iter_mut` / `into_iter`

- `iter()`：产生 `&T`
- `iter_mut()`：产生 `&mut T`
- `into_iter()`：**拿走所有权**，产生 `T`

分别示例：

**`iter()`**（迭代不可变引用 `&T`）：

```rust
fn main() {
    let v = vec![1, 2, 3];
    for x in v.iter() {
        // x: &i32
        println!("{x}");
    }
    println!("{v:?}"); // ✅ v 仍可用（只是借用）
}
```

**`iter_mut()`**（迭代可变引用 `&mut T`）：

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    for x in v.iter_mut() {
        // x: &mut i32
        *x += 1;
    }
    assert_eq!(v, vec![2, 3, 4]);
}
```

**`into_iter()`**（拿走集合所有权，迭代 `T`）：

```rust
fn main() {
    let v = vec![1, 2, 3];
    // into_iter() 会把 v move 进迭代器（v 的所有权被拿走）
    // 循环过程中元素一个个被 move 出来绑定到 x
    for x in v.into_iter() {
        // x: i32
        println!("{x}");
    }
    // 循环结束后，迭代器被 drop；此时 v 这个变量早已被 move 走
    // 注意：不是 “v 变成空 Vec”，而是 “v 完全不能再访问”
    // println!("{v:?}"); // ❌ 编译错误：use of moved value
}
```

`into_iter()` 的实际应用场景（你**不再需要原集合**，而是想把元素 **move 出来** 做事）：

- **消费并转换为新集合（零拷贝移动元素）**：

```rust
fn main() {
    let v: Vec<String> = vec!["a".into(), "bb".into()];
    let lens: Vec<usize> = v.into_iter().map(|s| s.len()).collect();
    assert_eq!(lens, vec![1, 2]);
}
```

- **拆出所有权（例如 `HashMap<K, V>` 的 `(K, V)`）**：

```rust
use std::collections::HashMap;

fn main() {
    let m: HashMap<String, i32> = [("a".into(), 1), ("b".into(), 2)].into();
    let pairs: Vec<(String, i32)> = m.into_iter().collect(); // move 出 (K, V)
    assert_eq!(pairs.len(), 2);
}
```

- **把数据 move 进线程/任务里消费**（配合 `move || { ... }` 很常见）：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];
    let handle = thread::spawn(move || v.into_iter().sum::<i32>());
    assert_eq!(handle.join().unwrap(), 6);
}
```

### 2.4 适配器 vs 消费者（记住这条就够了）

- **适配器**：`map` / `filter` / `zip` ... → 返回新迭代器（惰性）
- **消费者**：`collect` / `sum` / `for` ... → 消耗迭代器得到结果

最小链式例子：

```rust
fn main() {
    let v = vec![1, 2, 3];
    let v2: Vec<_> = v.iter().map(|x| x + 1).collect();
    assert_eq!(v2, vec![2, 3, 4]);
}
```

捕获环境的闭包 + `filter`：

```rust
#[derive(Debug, PartialEq)]
struct Shoe { size: u32, style: &'static str }

fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter()
        .filter(|s| s.size == shoe_size) // 闭包捕获 shoe_size
        .collect()
}
```

## 3. 用迭代器改进 `minigrep`（ch12 项目）

### 3.1 `Config::build` 接收迭代器：消除 `clone`

思路：

- `main` 直接把 `env::args()`（迭代器）交给 `Config::build`
- `Config::build` 获取迭代器所有权，用 `next()` 取参数并**move** 进 `Config`

签名要点：

```rust
pub fn build(mut args: impl Iterator<Item = String>) -> Result<Config, &'static str> {
    // args.next();  // 跳过程序名
    // query/file_path 用 match args.next()
    // ...
}
```

### 3.2 `search` 用 `filter + collect`：去掉可变中间 `Vec`

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    contents
        .lines()
        .filter(|line| line.contains(query))
        .collect()
}
```

（同理可改写 `search_case_insensitive`：对 `query`/`line` 做 `to_lowercase()` 再 `contains`。）

## 4. 性能结论：迭代器是零成本抽象

- 基准测试表明：显式 `for` 循环与迭代器链性能**非常接近**（甚至迭代器略快）。
- 关键原因：迭代器链会被编译器优化成和手写循环非常接近的底层代码（包括循环展开、消除边界检查等）。
- 结论：放心使用闭包/迭代器来提升表达力，通常不会为此付出额外运行时开销。