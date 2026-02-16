
# ch18：面向对象编程特性（OOP in Rust）速查

只记录：Rust 视角下的 OOP 三要素（对象/封装/继承与多态）、trait objects（`dyn`、动态分发、对象安全）、以及状态模式的两种实现取舍。

## 1. Rust 是否“面向对象”？

### 1.1 对象 = 数据 + 行为（方法）

- Rust 的 **struct/enum** 存数据，`impl` 块提供方法（行为）
- 但 Rust 通常不把它们称为“对象”；**trait object** 在“数据 + 行为绑定在一起”的意义上更像传统 OOP 的对象

### 1.2 封装（encapsulation）

- Rust 用模块系统与 `pub` 控制可见性：默认私有，必要处开放公有 API
- 典型实践：**字段私有 + 方法公有**，保持不变量（invariant）

最小示例：缓存平均值，确保 `add/remove` 时同步更新：

```rust
pub struct AveragedCollection {
    list: Vec<i32>,
    average: f64,
}

impl AveragedCollection {
    pub fn add(&mut self, value: i32) {
        self.list.push(value);
        self.update_average();
    }

    pub fn remove(&mut self) -> Option<i32> {
        let result = self.list.pop();
        if result.is_some() {
            self.update_average();
        }
        result
    }

    pub fn average(&self) -> f64 {
        self.average
    }

    fn update_average(&mut self) {
        let total: i32 = self.list.iter().sum();
        self.average = total as f64 / self.list.len() as f64;
    }
}
```

要点：

- `list/average` 私有，外部无法“绕过”更新逻辑破坏一致性
- 内部实现可替换（`Vec`→`HashSet` 等），只要公有 API 不变

### 1.3 继承（inheritance）与多态（polymorphism）

Rust **没有类继承**（字段/方法继承）。

常见“用继承想要的东西”，Rust 的替代：

- **代码复用**：trait 默认实现（类似父类提供默认方法，子类可覆盖）
- **多态**：
  - **泛型 + trait bound**：编译期多态（bounded parametric polymorphism，通常静态分发/单态化）
  - **trait object**：运行期多态（动态分发）

## 2. trait objects（`dyn Trait`）：运行期多态

### 2.0 `dyn Trait` 的背景：为了解决什么问题

Rust 的泛型（`T: Trait`）主要提供 **编译期多态**：通过单态化实现静态分发，通常更快、可内联优化，但也带来限制——很多场景下你必须在编译期就“确定类型”。

`dyn Trait`（trait object）主要用来补上 **运行时多态 / 类型擦除** 这块能力，解决典型问题：

- **异质集合**：例如一个 `Vec` 里需要同时放 `Button`、`SelectBox`、`Image`……（类型不同，但共享 `draw()`）
- **可扩展的库/框架**：库作者无法预知用户会增加哪些类型，只要求“实现某个 trait 即可接入”
- **只暴露行为、不暴露具体类型**：把 API 耦合点从“具体类型”降到“trait 的方法集合”

代价与约束（速记）：

- **代价**：方法调用走 vtable（动态分发），有少量运行时开销，且可能影响内联优化
- **约束**：不是所有 trait 都能作为 `dyn Trait`（对象安全 / dyn compatibility）

### 2.1 什么时候用 trait object

当你需要一个集合/参数里装**不同具体类型**，但它们共享同一组行为（trait）：

- 同质集合（全是同一类型）→ 更倾向泛型（静态分发，优化更好）
- 异质集合（多种类型）→ trait object（动态分发）

### 2.2 最小结构：`Box<dyn Draw>`

trait object 的直觉：

- 你不关心“它到底是什么类型”，只关心“它能做什么”（方法集合）
- 有点像“鸭子类型”，但 Rust 在**编译期**就保证：放进来的类型一定实现了该 trait

```rust
pub trait Draw {
    fn draw(&self);
}

pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}

impl Screen {
    pub fn run(&self) {
        for c in self.components.iter() {
            c.draw();
        }
    }
}
```

使用者可以扩展类型集合（只要实现 `Draw`）：

```rust
struct Button;
impl Draw for Button {
    fn draw(&self) {}
}

struct SelectBox;
impl Draw for SelectBox {
    fn draw(&self) {}
}

fn main() {
    let s = Screen {
        components: vec![Box::new(Button), Box::new(SelectBox)],
    };
    s.run();
}
```

要点：

- trait object 必须放在某种指针后面（如 `&dyn Trait` / `Box<dyn Trait>`），因为 `dyn Trait` 是 DST（动态大小类型）

### 2.3 动态分发 vs 静态分发（取舍）

- **泛型**：编译期知道具体类型 → **静态分发**，可内联/优化（但集合常要求同一类型）
- **trait object**：编译期不知道具体类型 → **动态分发**（通过 vtable 在运行时找方法），有少量运行时开销，可能影响内联优化

同一个 `Screen` 的“泛型版本”（同质集合）：

```rust
pub struct Screen2<T: Draw> {
    pub components: Vec<T>,
}

impl<T: Draw> Screen2<T> {
    pub fn run(&self) {
        for c in self.components.iter() {
            c.draw();
        }
    }
}
```

### 2.4 对象安全（dyn compatibility）速记

不是所有 trait 都能变成 `dyn Trait`。直觉上：

- 方法签名不能依赖“调用者需要知道 `Self` 的具体大小/类型”才能调用
- 例如很多返回 `Self`、或有泛型方法的 trait 往往不适合作为 trait object

（细节规则见 Rust Reference：dyn compatibility。）

## 3. OOP 设计模式：状态模式（state pattern）

目标：一个 `Post` 随状态变化而改变行为（draft → review → published），外部只通过 `Post` 的 API 交互。

### 3.1 “传统 OOP 风格”：trait object + 状态对象

核心结构（抓住这几个点）：

- `Post` 持有 `state: Option<Box<dyn State>>`
- 状态转移方法使用 `self: Box<Self>` 消费旧状态，返回新状态
- `Option::take()` 临时拿走 state，避免“部分移动/未初始化字段”
- `Published` 重写 `content`，其他状态默认返回空字符串

最小骨架（展示 `Option::take` 与 `self: Box<Self>` 的用法）：

```rust
pub struct Post {
    state: Option<Box<dyn State>>,
    content: String,
}

impl Post {
    pub fn new() -> Self {
        Self {
            state: Some(Box::new(Draft)),
            content: String::new(),
        }
    }

    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }

    pub fn request_review(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.request_review());
        }
    }

    pub fn approve(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.approve());
        }
    }

    pub fn content(&self) -> &str {
        self.state.as_ref().unwrap().content(self)
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<dyn State>;
    fn approve(self: Box<Self>) -> Box<dyn State>;
    fn content<'a>(&self, _post: &'a Post) -> &'a str {
        ""
    }
}

struct Draft;
struct PendingReview;
struct Published;
```

这种写法的优点：

- `Post` 的对外 API 稳定；新增状态通常只需新增状态类型并实现 `State`

缺点/成本：

- 状态之间会产生耦合（某些状态要知道下一个状态是什么）
- 会有一些样板重复（`take` + 委托模式）
- trait object 带来动态分发开销（通常很小，但存在）

### 3.2 “更 Rust 的方式”：把状态编码进类型（类型状态机）

思路：不同状态用不同类型表示，让“非法状态/非法转移”变成**编译错误**。

典型形态：

- `DraftPost`：能 `add_text`，但**没有** `content()`
- `PendingReviewPost`：也没有 `content()`
- `Post`（Published）：才有 `content()`
- 转移消耗 `self`：`DraftPost::request_review(self) -> PendingReviewPost`，`PendingReviewPost::approve(self) -> Post`

最小骨架（用类型系统禁止“未发布时读取内容”）：

```rust
pub struct DraftPost {
    content: String,
}

pub struct PendingReviewPost {
    content: String,
}

pub struct Post {
    content: String,
}

impl Post {
    pub fn new() -> DraftPost {
        DraftPost {
            content: String::new(),
        }
    }

    pub fn content(&self) -> &str {
        &self.content
    }
}

impl DraftPost {
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }

    pub fn request_review(self) -> PendingReviewPost {
        PendingReviewPost {
            content: self.content,
        }
    }
}

impl PendingReviewPost {
    pub fn approve(self) -> Post {
        Post {
            content: self.content,
        }
    }
}
```

优点：

- “未发布内容被读取”这类 bug 直接变成编译错误

代价：

- 调用方需要接收新值并反复绑定/遮蔽（`let post = post.request_review();`）
- API 形态更偏“类型驱动”，不完全像传统 OOP 把状态隐藏在对象内部

