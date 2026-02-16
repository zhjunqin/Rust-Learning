# ch16：无畏并发（Concurrency）速查

只记录：线程、消息传递（channel）、共享状态（Mutex/Arc）、`Send`/`Sync` 规则与常见坑点。

## 0. 并发 vs 并行

- **并发**：多段任务独立推进（交错执行）
- **并行**：多段任务同一时刻同时执行（多核）
- 本章很多地方用“并发”泛指“并发和/或并行”。

## 1. 线程（threads）

### 1.1 创建线程：`thread::spawn`

```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..=5 {
            println!("spawned {i}");
            thread::sleep(Duration::from_millis(10));
        }
    });

    for i in 1..=3 {
        println!("main {i}");
        thread::sleep(Duration::from_millis(10));
    }
}
```

要点：

- 主线程结束时，**其他线程也会被强制结束**（可能没跑完）。

### 1.2 等待线程结束：`JoinHandle::join`

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| 42);
    let v = handle.join().unwrap();
    assert_eq!(v, 42);
}
```

- `join()` 会阻塞当前线程直到目标线程结束
- `join()` 放置位置会影响“是否真正并发交错执行”

### 1.3 `move` 闭包：把所有权转移进线程

当新线程要使用主线程的变量时，通常需要 `move`：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];
    let handle = thread::spawn(move || {
        // v 的所有权被移动到新线程中，避免悬垂引用
        println!("{v:?}");
    });
    handle.join().unwrap();
}
```

## 2. 消息传递（message passing）：`mpsc::channel`

`mpsc` = **multiple producer, single consumer**（多生产者、单消费者）。

### 2.1 最小例子：发送/接收一个值

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send(String::from("hi")).unwrap();
    });

    let msg = rx.recv().unwrap(); // 阻塞直到收到
    println!("Got: {msg}");
}
```

要点：

- `send(val)` **会 move** `val` 给接收方（发送后不能再用 `val`）
- `recv()`：阻塞等待；`try_recv()`：不阻塞，立刻返回（适合轮询/做别的事）

### 2.2 发送多个值：把 `rx` 当迭代器

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        for s in ["hi", "from", "the", "thread"] {
            tx.send(s.to_string()).unwrap();
            thread::sleep(Duration::from_millis(10));
        }
    });

    for msg in rx {
        // 当所有发送端 drop 后，rx 迭代结束
        println!("Got: {msg}");
    }
}
```

### 2.3 多生产者：克隆发送端 `tx.clone()`

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    let tx2 = tx.clone();

    thread::spawn(move || tx.send("a").unwrap());
    thread::spawn(move || tx2.send("b").unwrap());

    for msg in rx {
        println!("{msg}");
    }
}
```

## 3. 共享状态（shared state）：`Mutex<T>` + `Arc<T>`

### 3.1 `Mutex<T>`：通过锁实现互斥访问

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);
    {
        let mut guard = m.lock().unwrap(); // -> MutexGuard<T>
        *guard += 1;
    } // guard drop => 自动解锁
    assert_eq!(*m.lock().unwrap(), 6);
}
```

要点：

- `lock()` 返回 `MutexGuard`：实现 `Deref`/`DerefMut`，并在 `Drop` 时自动解锁
- 如果持锁线程 panic，锁可能“中毒”（poison），`lock()` 会返回 `Err`（示例里直接 `unwrap`）

### 3.2 多线程共享：`Arc<Mutex<T>>`

为什么不是 `Rc<Mutex<T>>`？

- `Rc<T>` **不是 `Send`**（引用计数非线程安全）
- 多线程要用 **`Arc<T>`**（原子引用计数）

经典计数器：

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            let mut n = counter.lock().unwrap();
            *n += 1;
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    assert_eq!(*counter.lock().unwrap(), 10);
}
```

坑点速记：

- **死锁**：多把锁/多资源时，锁顺序不一致会互相等待（Rust 不能在编译期完全避免逻辑死锁）
- **锁粒度**：尽量缩小 guard 的作用域（用 `{}` 包住）

### 3.3 为什么常见写法是 `Arc<Mutex<T>>`？能只用其中一个吗？

`Arc<Mutex<T>>` 解决的是**两个不同问题**：

- **`Arc<T>`**：让同一份数据在**多个线程中有多个所有者**（原子引用计数、可在线程间共享/移动）。
- **`Mutex<T>`**：让多个线程对同一份数据的**可变访问互斥**（同一时刻只允许一个线程拿到“可变访问”）。

分别单独用，通常不能完成“多线程共享并修改”：

- **只用 `Arc<T>`**：只解决“共享所有权”，不自动允许你在多线程里随便修改 `T`。
  - 适合：只读共享；或 `T` 自身就是线程安全可变（例如原子类型 `Atomic*`）。
- **只用 `Mutex<T>`**：`Mutex<T>` 本身仍是**单一所有权**的一个值。
  - 你把它 `move` 进一个线程后，其他线程/主线程就无法再持有同一个 `Mutex<T>`；
  - 要让多个线程都能持有它，需要 `Arc` 这种“多所有权容器”。

常见替代：

- 计数器等简单数值：`Arc<AtomicUsize>`（不需要锁）
- 读多写少：`Arc<RwLock<T>>`

## 4. `Send` / `Sync`：可扩展并发保证（标记 trait）

### 4.1 `Send`

- `T: Send`：`T` 的所有权可以在线程间移动
- 例外：`Rc<T>` 不是 `Send`（引用计数更新非线程安全）
- 规律：如果一个类型完全由 `Send` 的字段组成，通常会**自动**是 `Send`

### 4.2 `Sync`

- `T: Sync`：`&T` 可以在线程间共享（多线程持有引用是安全的）
- 等价理解：若 `&T: Send`，则 `T: Sync`
- 常见非 `Sync`：`RefCell<T>` / `Cell<T>`（运行时借用检查/内部可变性非线程安全）
- `Mutex<T>` 是 `Sync`（锁保护了内部可变性）

### 4.3 手动实现 `Send`/`Sync` 是 `unsafe`

一般不需要手动实现；如果要实现，意味着你要自己保证并发不变量（属于不安全 Rust 的范畴）。

### 4.4 `Send` / `Sync` 背后的设计背景（结合代码理解）

核心目标：把“某个类型能不能安全地跨线程**移动/共享**”做成**可组合、可推导、可被库复用**的类型级约束，让很多并发错误在**编译期**暴露。

#### 4.4.1 `Send`：能不能把**所有权**搬到另一个线程？

`thread::spawn(move || { ... })` 往往要把捕获的值移动到新线程，因此要求这些值能跨线程转移所有权（通常可理解为需要 `Send`，并且捕获内容不借用短生命周期数据）。

✅ `Vec<i32>` 是 `Send`：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];
    thread::spawn(move || {
        println!("{v:?}");
    })
    .join()
    .unwrap();
}
```

❌ `Rc<T>` 不是 `Send`（引用计数非线程安全，编译器直接拒绝）：

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let x = Rc::new(1);
    thread::spawn(move || {
        println!("{x}");
    });
}
```

✅ 多线程共享引用计数用 `Arc<T>`：

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let x = Arc::new(1);
    let x2 = Arc::clone(&x);

    let h = thread::spawn(move || {
        println!("{x2}");
    });

    h.join().unwrap();
    println!("{x}");
}
```

#### 4.4.2 `Sync`：能不能把**共享引用 `&T`**给多个线程用？

直觉：如果类型 `T` 是 `Sync`，意味着把 `&T` 分发到多个线程并发使用是安全的（常用等价理解：`T: Sync` ⇔ `&T: Send`）。

❌ `RefCell<T>` 不是 `Sync`（运行时借用计数非线程安全）：

```rust
use std::cell::RefCell;
use std::sync::Arc;
use std::thread;

fn main() {
    let x = Arc::new(RefCell::new(0));
    let x2 = Arc::clone(&x);

    thread::spawn(move || {
        *x2.borrow_mut() += 1;
    })
    .join()
    .unwrap();
}
```

✅ `Mutex<T>` 是 `Sync`（锁保证互斥）：

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let x = Arc::new(Mutex::new(0));
    let x2 = Arc::clone(&x);

    thread::spawn(move || {
        *x2.lock().unwrap() += 1;
    })
    .join()
    .unwrap();

    assert_eq!(*x.lock().unwrap(), 1);
}
```

#### 4.4.3 用 trait bound 显式表达“并发能力边界”

把约束写进函数签名，调用点就会得到清晰的编译期反馈：

```rust
use std::thread;

fn run_in_thread<T: Send + 'static>(x: T) -> thread::JoinHandle<T> {
    thread::spawn(move || x)
}

fn main() {
    let h = run_in_thread(String::from("ok"));
    assert_eq!(h.join().unwrap(), "ok");
}
```

要点：

- `Send`/`Sync` 让并发“可扩展”：标准库之外的并发原语也能用同一套规则表达安全性
- 大多数类型会自动获得 `Send`/`Sync`；只有包含非线程安全成分（如 `Rc`/`RefCell`）时才会被类型系统阻止

### 4.5 `Send` / `Sync` 在真实项目里的常见用法（场景速查）

把它们当作两类“能力”来记：

- **需要 `Send`**：把值/任务/闭包**交给别的线程/执行器去跑**
- **需要 `Sync`**：把 `&T` **同时给多个线程用**（共享读，或通过锁实现共享写）

#### 4.5.1 线程池 / worker：任务必须是 `Send`

很多线程池 API（概念上）都会要求你提交的任务可跨线程移动：

```rust
use std::thread;

// 一个最小“线程池接口”示意：任务需要能 Send + 'static
//
// - Send：任务（闭包）能安全地跨线程移动
// - 'static：任务里不能捕获“会提前失效的引用”（不能借用当前栈帧的局部变量）
//   因为线程/线程池可能在 submit_to_workers 返回之后才执行该任务；
//   任务通常应当通过 move 捕获拥有所有权的数据（如 String/Vec/Arc），而不是借用 &T。
//   'static 保证：job 内部不能包含这种短生命周期借用；它要么
//   捕获/持有 拥有所有权的数据（例如 String、Vec，通过 move 进去），要么
//   只借用 真正全局有效的引用（例如字符串字面值 &'static str），要么
//   借用的是 Arc<T> 这类本身拥有数据的智能指针（借用 Arc 也通常能满足，因为 Arc 会被 move 进闭包）。
fn submit_to_workers(job: impl FnOnce() + Send + 'static) {
    thread::spawn(job);
}

fn main() {
    let s = String::from("work");
    submit_to_workers(move || println!("{s}"));
}
```

直觉：worker 线程不在当前栈帧里运行，所以 job 不能借用短生命周期数据。

#### 4.5.2 跨线程 channel：消息类型要 `Send`

发送到另一个线程的消息，必须能安全跨线程移动：

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<String>();

    thread::spawn(move || {
        tx.send("hi".to_string()).unwrap(); // String: Send
    });

    println!("{}", rx.recv().unwrap());
}
```

如果消息里藏了 `Rc<T>` / `RefCell<T>` 这类非线程安全成分，通常会在编译期被拒绝。

#### 4.5.3 全局配置/缓存：只读共享用 `Arc<T>`（`T: Sync`）

```rust
use std::sync::Arc;
use std::thread;

#[derive(Debug)]
struct Config {
    base_url: String,
}

fn main() {
    let cfg = Arc::new(Config { base_url: "https://example.com".into() });

    let c1 = Arc::clone(&cfg);
    let c2 = Arc::clone(&cfg);
    let h1 = thread::spawn(move || println!("{:?}", c1.base_url));
    let h2 = thread::spawn(move || println!("{:?}", c2.base_url));

    h1.join().unwrap();
    h2.join().unwrap();
}
```

要点：共享的是 `Arc<T>` 的克隆（多所有权），而读取依赖的是 `T` 的 `Sync` 性质。

#### 4.5.4 共享可变状态：`Arc<Mutex<T>>` / `Arc<RwLock<T>>`

- **写多**：`Arc<Mutex<T>>`
- **读多写少**：`Arc<RwLock<T>>`

（`Mutex/RwLock` 通过锁把“共享写”变成安全的互斥访问，因此它们能用于 `Sync` 场景。）

#### 4.5.5 计数器/指标：用原子类型替代锁（仍依赖 `Send`/`Sync`）

```rust
use std::sync::{
    Arc,
    atomic::{AtomicUsize, Ordering},
};
use std::thread;

fn main() {
    let hits = Arc::new(AtomicUsize::new(0));
    let mut hs = vec![];

    for _ in 0..4 {
        let hits = Arc::clone(&hits);
        hs.push(thread::spawn(move || {
            hits.fetch_add(1, Ordering::Relaxed);
        }));
    }

    for h in hs { h.join().unwrap(); }
    assert_eq!(hits.load(Ordering::Relaxed), 4);
}
```

#### 4.5.6 常见“为什么编不过”的修复路线图

- 想跨线程共享但用了 `Rc<T>` ⇒ 改 `Arc<T>`
- 想跨线程修改但用了 `RefCell<T>` ⇒ 改 `Mutex<T>` / `RwLock<T>` / 原子类型