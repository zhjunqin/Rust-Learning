# ch02：猜数字游戏（guessing game）速查

目标：生成 \(1..=100\) 的随机数，循环读取用户输入并比较大小，直到猜中为止；对非法输入不崩溃而是继续。

## 1. 创建项目与常用命令

```bash
cargo new guessing_game
cd guessing_game

cargo run
cargo build
cargo check
cargo doc --open
```

## 2. 添加随机数依赖（`rand`）

`Cargo.toml`：

```toml
[dependencies]
rand = "0.8.5"
```

要点：

- **`Cargo.lock`**：锁定依赖版本，保证可重现构建
- **`cargo update`**：在不改变 `Cargo.toml` 版本约束的前提下更新到允许范围内的最新版本

## 3. 主流程（最小闭环）

1. 提示输入
2. `stdin().read_line(&mut String)` 读入一行（包含换行）
3. `trim()` 去掉 `\n` / `\r\n`
4. `parse::<u32>()` 转成数字
5. `guess.cmp(&secret_number)` 得到 `Ordering::{Less, Greater, Equal}`
6. `match` 打印 “Too small/Too big/You win”，猜中就 `break`
7. `Err(_)` 的输入用 `continue` 忽略并重新输入

## 4. 用到的关键点（按出现顺序）

### 4.1 `stdin` + `read_line` + `Result`

```rust
use std::io;

let mut guess = String::new();
io::stdin().read_line(&mut guess).expect("Failed to read line");
```

- `read_line` **追加**到字符串（不覆盖），所以需要 `&mut String`
- 返回 `Result`：教学中常用 `expect` 直接崩溃；更完善的版本用 `match` 处理错误

### 4.2 生成随机数：`rand::thread_rng().gen_range(1..=100)`

```rust
use rand::Rng;

let secret_number = rand::thread_rng().gen_range(1..=100);
```

- `Rng` 是 trait，需要 `use rand::Rng;` 才能调用 `gen_range`

### 4.3 `cmp` + `Ordering` + `match`

```rust
use std::cmp::Ordering;

match guess.cmp(&secret_number) {
    Ordering::Less => println!("Too small!"),
    Ordering::Greater => println!("Too big!"),
    Ordering::Equal => println!("You win!"),
}
```

### 4.4 Shadowing（遮蔽）：同名变量做“类型转换”

```rust
let mut guess = String::new();
// ...
let guess: u32 = guess.trim().parse().expect("Please type a number!");
```

这里第二个 `guess` 会遮蔽第一个 `guess`，常用于把 `String` 变成数字类型。

### 4.5 循环与错误输入：`loop` + `continue` + `break`

```rust
let guess: u32 = match guess.trim().parse() {
    Ok(num) => num,
    Err(_) => continue, // 非法输入：忽略，进入下一轮循环
};
```

## 5. 完整代码（`src/main.rs` 最终版）

```rust
use rand::Rng;
use std::cmp::Ordering;
use std::io;

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1..=100);

    loop {
        println!("Please input your guess.");

        let mut guess = String::new();
        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        println!("You guessed: {guess}");

        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}
```
