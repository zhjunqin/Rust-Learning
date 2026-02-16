
# ch21：最终项目——多线程 Web Server 速查

目标：用标准库手写一个极简 HTTP server，并用**线程池**提升吞吐量，最后实现**优雅停机**。

> 这不是生产级 web server 的最佳实践；目的是理解 TCP/HTTP 原始字节、线程池与清理的通用思路。真实项目通常使用成熟 crate（并常结合 async/await）。

## 0. 项目路线图（本章做什么）

1. 监听 TCP 连接（`TcpListener`）
2. 读取并解析少量 HTTP 请求（只看请求行）
3. 构造 HTTP 响应（状态行 + headers + body）
4. 单线程版本跑通
5. 用线程池并发处理连接（限制线程数，避免 DoS）
6. 优雅停机：停止接收新任务 + `join` 所有线程

## 1. 单线程 web server（Single-threaded）

### 1.1 创建项目与运行

```bash
cargo new hello
cd hello
cargo run
```

浏览器访问：`127.0.0.1:7878`

### 1.2 监听 TCP：`TcpListener::bind` + `incoming()`

关键点：

- 监听 `127.0.0.1:7878`（7878 在九宫格上像 “rust”）
- `incoming()` 迭代的是“连接尝试”，每次得到一个 `TcpStream`

骨架：

```rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();
        // handle_connection(stream);
        let _ = stream;
    }
}
```

### 1.3 读取 HTTP 请求：`BufReader` + `lines()`

HTTP 是文本协议；一个请求以空行结束（连续 `\r\n` → 读出来会得到空字符串行）。

最小“读请求行”做法（只取第一行）：

```rust
use std::io::{BufRead, BufReader};
use std::net::TcpStream;

fn request_line(stream: &TcpStream) -> String {
    let mut reader = BufReader::new(stream);
    reader.lines().next().unwrap().unwrap()
}
```

### 1.4 写响应：状态行 + `Content-Length` + body

HTTP 响应格式（简化）：

```text
HTTP-Version Status-Code Reason-Phrase\r\n
headers...\r\n
\r\n
body...
```

返回 HTML 文件的最小套路：

- `fs::read_to_string` 读取 `hello.html` / `404.html`
- 用 `Content-Length` 告诉浏览器 body 长度
- `write_all` 写回 `TcpStream`

### 1.5 路由：只处理 `/`，其他返回 404（以及 `/sleep`）

- 只看请求行（例如 `GET / HTTP/1.1`）
- `/` → `hello.html`
- 其他 → `404.html`
- `/sleep` → `sleep(5s)` 后再返回 `hello.html`（用来模拟慢请求）

要点：把重复代码抽出来，分支只决定（状态行, 文件名）。

## 2. 多线程：线程池提升吞吐量（ThreadPool）

### 2.1 为什么要线程池

单线程一次只能处理一个连接：如果一个请求很慢（`/sleep`），后面的快请求也会被阻塞。

两种对比：

- **每请求一个线程**：`thread::spawn(move || handle_connection(stream))`（能并发，但可能无限建线程 → 资源耗尽）
- **线程池**：固定 N 个 worker 线程 + 任务队列 → 限制并发度，提升吞吐量

### 2.2 期望的 API（像 `thread::spawn` 一样用）

```rust
let pool = ThreadPool::new(4);

for stream in listener.incoming() {
    let stream = stream.unwrap();
    pool.execute(|| {
        handle_connection(stream);
    });
}
```

`execute` 的闭包 bound（对齐 `thread::spawn`）：

- `FnOnce()`：任务只执行一次
- `Send + 'static`：可跨线程移动，且不借用短生命周期引用

### 2.3 线程池内部结构（最小模型）

- `ThreadPool`：
  - `workers: Vec<Worker>`
  - `sender: mpsc::Sender<Job>`
- `Worker`：
  - `id`
  - `thread: JoinHandle<()>`
- `Job`：
  - 类型别名：`Box<dyn FnOnce() + Send + 'static>`
- 任务队列：
  - `mpsc::channel()`
  - `receiver` 需要共享给多个 worker：`Arc<Mutex<mpsc::Receiver<Job>>>`

Worker 循环（关键一行）：

- 避免 `while let` 把 `MutexGuard` 的临时值持有到整个循环体结束（会导致“拿着锁执行 job”，其他 worker 取不到任务）
- 惯用写法：把锁与 `recv()` 的临时值限制在 `let` 语句右侧

```rust
let job = receiver.lock().unwrap().recv().unwrap();
job();
```

## 3. 优雅停机与清理（Graceful shutdown）

目标：主线程退出前，让 worker **停止等待新任务**，并 `join` 完成退出。

### 3.1 为什么直接 `join` 会卡死

worker 线程在 `loop { recv(); job(); }` 里永远等任务；如果不先让它们退出循环，`join` 会一直等。

### 3.2 关闭信道：丢弃 `sender` → `recv()` 返回 Err → worker 退出

关键思路：

- 在 `ThreadPool` 的 `Drop` 里先 drop sender（让信道关闭）
- 修改 worker：当 `recv()` 返回 `Err` 时 `break` 并打印 “disconnected; shutting down”

实现细节：

- `ThreadPool` 里的 `sender` 常用 `Option<Sender<Job>>` 包一层，方便 `take()` 提前丢弃
- join workers 时可用 `workers.drain(..)` 把 worker 移出 vector，避免 “只借用拿不到所有权” 的问题

### 3.3 用 `take(2)` 演示优雅关闭

为了演示，在 `main` 里限制只处理 2 个连接：

```rust
for stream in listener.incoming().take(2) {
    // ...
}
println!("Shutting down.");
```

离开 `main` 作用域时 `ThreadPool::drop` 运行，完成：关闭 sender → worker 退出 → join。

## 4. 常用命令

```bash
cargo run
cargo check
cargo test
cargo doc --open
```

