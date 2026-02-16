# ch17：Async / Await 速查

只记录：async/await 语法、`Future` 模型、async 并发（join/race/join_all）、`Pin`/`Unpin`、`Stream`、任务 vs 线程。

## 0. async 在解决什么问题

- **IO-bound**：等待网络/磁盘时，让 CPU 去做别的事（非阻塞）
- Rust async 主要建模的是**并发**（concurrency），底层是否并行取决于运行时/线程池

## 1. `async` / `await` 基础

### 1.1 `async fn` 返回的是 Future（惰性）

- `async fn foo() -> T` 等价于 `fn foo() -> impl Future<Output = T>`
- **future 是惰性的**：不 `await` 不会做任何事（像迭代器一样）

### 1.2 `await` 是后缀关键字（便于链式调用）

```rust
let text = trpl::get(url).await.text().await;
```

### 1.3 为什么 `main` 不能直接 `async`

- async 需要**运行时**（runtime / executor）来轮询 futures
- 本章示例用 `trpl::run(async { ... })`（等价于很多运行时的 `block_on`）

```rust
fn main() {
    trpl::run(async {
        // 在这里才能用 await
    });
}
```

## 2. Future 的工作方式（只背结论）

### 2.1 `Future` trait：`poll` / `Poll`

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

```rust
enum Poll<T> {
    Ready(T),
    Pending,
}
```

要点：

- `await` 底层就是“反复 poll，Pending 就让出执行权，Ready 就继续”
- **运行时**负责：轮询、挂起、唤醒（waker 在 `Context` 里）
- 一般不要在 future `Ready` 后再次 poll（很多会 panic）

### 2.2 async 会被编译成状态机

- 每个 `await` 点是一次“可能挂起/恢复”的边界
- `await` 点之间的代码仍然是同步的：**没有 await 就不会让出执行权**

## 3. async 并发：join / race / join_all

### 3.1 `spawn_task` vs 直接 `join`

- `spawn_task` 类似 `thread::spawn`：把 future 作为任务交给运行时调度
- 也可以不 spawn：把两个 async 块直接 `join`（一个 future 里组织并发）

### 3.2 `join`：等两者都完成

```rust
let fut1 = async { /* ... */ };
let fut2 = async { /* ... */ };
trpl::join(fut1, fut2).await;
```

（`trpl::join` 是公平的：交替轮询两个 future；线程的公平性由 OS 决定。）

### 3.3 `race` / `select`：等“先完成”的那个

```rust
let a = async { /* ... */ };
let b = async { /* ... */ };
let winner = trpl::race(a, b).await; // -> Either::Left(_) / Either::Right(_)
```

- `race` 构建在更通用的 `select` 之上
- `race` 可能不公平：通常按参数顺序先 poll（实现细节依赖运行时/库）

### 3.4 任意数量：`join!` / `join_all`

- `join!`：参数数量固定但可很多个；可以不同 future 类型
- `join_all`：从迭代器收集；要求集合里元素是**同一类型**的 future

#### 3.4.1 为什么 `Vec` 装不下多个 `async {}`？

- 每个 `async {}` 都是编译器生成的**不同匿名类型**
- `Vec<T>` 必须是同一个 `T`

#### 3.4.2 用 trait object + pin 统一类型（动态集合）

典型形态（思路）：

- `Vec<Pin<Box<dyn Future<Output = ()>>>>`
- 用 `Box::pin(fut)` / `pin!` 把 future pin 住（因为很多 async future 不是 `Unpin`）

## 4. `Pin` / `Unpin`（看到报错能修就行）

### 4.1 为什么 future 需要 pin

- async 状态机可能是**自引用**的
- move 以后内部引用会指向旧地址 ⇒ 不安全
- `Pin` 用来保证“被 pin 的值不再移动”

### 4.2 `Unpin`

- 大多数普通类型是 `Unpin`（移动是安全的）
- async 产生的某些 future 可能是 `!Unpin`，需要 `Pin<&mut T>` 才能 poll

常见修复提示：

- 用 `Box::pin(...)`
- 或 `pin!(fut)`（避免不必要的堆分配，但仍需处理 trait object 类型对齐）

## 5. async 信道与所有权（像 ch16，但 recv 是 async）

要点：

- `send(val)` 会 **move** `val`（发送后不能再用）
- `recv().await` 不阻塞线程：Pending 时把执行权还给运行时
- `while let Some(msg) = rx.recv().await { ... }`：`None` 表示信道关闭

### 5.1 一个常见坑：没并发就会“消息延迟后一起到”

- 如果 `send + sleep` 和 `recv` 在同一个 async 块里线性执行，接收逻辑会等发送逻辑全部跑完才开始
- 解决：把 send/recv 拆成不同 future，用 `join` 并发推进

“错误示例”（线性执行：先把消息都发完+等待完，再开始接收，所以看起来会“延迟后一起到”）：

```rust
use std::time::Duration;

fn main() {
    trpl::run(async {
        let (tx, mut rx) = trpl::channel();

        // 注意：这里的 sleep 都发生在接收开始之前
        for s in ["hi", "from", "async", "channel"] {
            tx.send(s.to_string()).unwrap();
            trpl::sleep(Duration::from_millis(200)).await;
        }

        // 直到上面循环结束后，才开始 recv
        while let Some(msg) = rx.recv().await {
            println!("received '{msg}'");
        }
    });
}
```

“修正示例”（并发推进 send/recv：把两段逻辑拆成两个 future，用 `join` 同时运行）：

```rust
use std::time::Duration;

fn main() {
    trpl::run(async {
        let (tx, mut rx) = trpl::channel();

        // async move：把 tx move 进去；发送结束后 tx 会立刻 drop，recv 才会返回 None 退出循环
        let send_fut = async move {
            for s in ["hi", "from", "async", "channel"] {
                tx.send(s.to_string()).unwrap();
                trpl::sleep(Duration::from_millis(200)).await;
            }
        };

        let recv_fut = async {
            while let Some(msg) = rx.recv().await {
                println!("received '{msg}'");
            }
        };

        trpl::join(send_fut, recv_fut).await;
    });
}
```

### 5.2 `async move`：让发送端在发送完后尽早 drop，正确关闭信道

- 信道只有在 `tx` 被 drop（或显式 close）后，`recv().await` 才会返回 `None`
- 发送 future 里用 `async move { ... }` 把 `tx` move 进去，future 结束就 drop

## 6. Streams：异步的“迭代器”

### 6.1 `Stream` / `StreamExt`

- `Stream` 底层是 `poll_next -> Poll<Option<Item>>`
- 你通常用 `StreamExt::next().await`
- 需要 `use trpl::StreamExt;` 才能用 `next`

```rust
use trpl::StreamExt;

while let Some(item) = stream.next().await {
    // ...
}
```

### 6.2 从迭代器创建流、组合流

- `stream_from_iter(iter)`：迭代器 → 流
- `ReceiverStream`：把异步信道接收端变成 `Stream`
- `StreamExt` 提供很多组合子：`filter` / `map` / `merge` / `timeout` / `throttle` / `take` 等

实践要点：

- 某些组合子会要求流被 pin（编译器提示时按建议 pin）
- unbounded 信道：超时并不会“丢消息”，只是在轮询时返回超时结果；下次轮询消息可能已到达

## 7. tasks vs threads（如何选）

### 7.1 线程

- OS 管理；创建/切换开销更大；线程数多会耗内存
- 适合：**非常可并行**的计算密集任务（CPU-bound）

### 7.2 任务（async tasks）

- 运行时管理；可创建大量任务；任务内可通过 await 协作切换（cooperative）
- 适合：**非常并发**的大量 IO/事件源（IO-bound、多路复用）

### 7.3 组合使用

- 线程做阻塞/计算密集型工作；通过（异步）信道把结果发回 async 侧
- 很多运行时底层本身就是多线程 + work stealing（任务在工作线程间移动）

## 8. 真实项目里如何“执行” async/await（不用 `trpl`）

在真实项目里，你需要一个 **异步运行时（runtime/executor）** 来驱动 `Future::poll`，并提供计时器/网络 IO 等能力。最常见选择是 **Tokio**。

### 8.1 Tokio：最常见

方式 A：用宏启动 runtime（最常见写法）：

```rust
#[tokio::main]
async fn main() {
    // tokio::spawn / tokio::time::sleep / tokio::net / reqwest 等
}
```

方式 B：手动创建 runtime（需要更细粒度控制时）：

```rust
fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        // async code
    });
}
```

测试（Tokio）：

```rust
#[tokio::test]
async fn it_works() {
    assert_eq!(1 + 1, 2);
}
```

### 8.2 async-std：更接近 std 风格的运行时

```rust
#[async_std::main]
async fn main() {
    // async_std::task::spawn / sleep / net ...
}
```

### 8.3 futures + executor：最小依赖（常见于库/小工具）

最简单的阻塞执行一个 future：

```rust
fn main() {
    futures::executor::block_on(async {
        // async code
    });
}
```

### 8.4 库（library）里怎么写更“通用”

- 通常 **不要**在库里固定某个 runtime；对外暴露 `async fn` / `impl Future`，由调用者决定在哪个 runtime 中执行
- 需要 spawn 时：让调用者传入 handle（例如 Tokio 的 `Handle`），或提供“同步 + 异步”两套 API

### 8.5 遇到阻塞操作怎么办（常见坑）

- 不要在 async 任务里直接做长时间阻塞（会卡住运行时线程，导致其它任务饥饿）
- 常见做法：把阻塞工作丢给专门的阻塞线程池（例如 Tokio 的 `spawn_blocking`）
