# 第 1 章 - 编码风格与惯用法

## 1.1 优先借用而非克隆

Rust 的所有权系统鼓励**借用**（`&T`）而不是**克隆**（`T.clone()`）。 
> ❗ 性能建议

### ✅ 何时应该 `Clone`：

* 你需要修改对象，同时保留原始对象（不可变快照）。
* 当你持有 `Arc` 或 `Rc` 指针时。
* 当数据跨线程共享时，通常是 `Arc`。
* 避免对非性能关键代码进行大规模重构。
* 当缓存结果时（下方为示例）：
```rust
fn get_config(&self) -> Config {
    self.cached_config.clone()
}
```
* 当底层 API 期望拥有所有权的数据（Owned Data）时。

### 🚨 应避免的 `Clone` 陷阱：

* 在循环内自动克隆 `.map(|x| x.clone)`，更倾向在迭代器末尾调用 `.cloned()` 或 `.copied()`。
* 克隆大型数据结构，如 `Vec<T>` 或 `HashMap<K, V>`。
* 因糟糕的 API 设计而克隆，而不是调整生命周期。
* 优先使用 `&[T]` 而不是 `Vec<T>` 或 `&Vec<T>`。
* 优先使用 `&str` 或 `&String` 而不是 `String`。
* 优先使用 `&T` 而不是 `T`。
* 克隆引用参数；如果你需要所有权，应在参数中向调用者明确声明。示例：
```rust
fn take_a_borrow(thing: &Thing) {
    let thing_cloned = thing.clone(); // the caller should have passed ownership instead
}
```

### ✅ 优先借用：
```rust
fn process(name: &str) {
    println!("Hello {name}");
}

let user = String::from("foo");
process(&user);
```

### ❌ 避免多余的克隆：
```rust
fn process_string(name: String) {
    println!("Hello {name}");
}

let user = String::from("foo");
process(user.clone()); // Unnecessary clone
```

## 1.2 何时按值传递？（Copy trait）

并非所有类型都应按引用（`&T`）传递。如果一个类型**很小**且**拷贝成本低廉**，那么**按值传递**通常更好。Rust 通过 `Copy` trait 将此显式化。

### ✅ 何时按值传递，`Copy`：
* 该类型**实现了** `Copy`（`u32`、`bool`、`f32`、小型结构体）。
* 移动该值的成本可以忽略不计。

```rust
fn increment(x: u32) -> u32 {
    x + 1
}

let num = 1;
let new_num = increment(num); // `num` still usable after this point
```

### ❓ 哪些结构体应该实现 `Copy`？
* 何时考虑在自己的类型上声明 `Copy`：
* 所有字段本身都是 `Copy`。
* 结构体**很小**，最多 2（也许 3）个字（word）的内存，即 24 字节（每个字为 64 位/8 字节）。
* 结构体**表示“纯数据对象”**，不涉及所有权（没有堆分配。例如：`Vec` 和 `String`）。

❗**Rust 数组是栈分配的。** 这意味着如果其底层类型是 `Copy`，它们可以被拷贝，但这会在程序栈上分配，很容易导致栈溢出。更多内容见 [第 3 章 - 栈与堆](./chapter_03.md#33-stack-vs-heap-be-size-smart)

作为参考，每种原生类型的大小（以字节为单位）：

#### 整数：

| 类型 | 大小 |
|------------- |---------- |
| i8 u8 | 1 字节 |
| i16 u16 | 2 字节 |
| i32 u32 | 4 字节 |
| i64 u64 | 8 字节 |
| isize usize | 架构相关 |
| i128 u128 | 16 字节 |

#### 浮点数：

| 类型 | 大小 |
|---------- |---------- |
| f32 | 4 字节 |
| f64 | 8 字节 |


#### 其他：

| 类型 | 大小 |
|---------- |---------- |
| bool | 1 字节 |
| char | 4 字节 |


### ✅ 适合派生 `Copy` 的结构体：
```rust
#[derive(Debug, Copy, Clone)]
struct Point {
    x: f32,
    y: f32,
    z: f32
}
```

### ❌ 不适合派生 `Copy` 的结构体：
```rust
#[derive(Debug, Clone)]
struct BadIdea {
    age: i32,
    name: String, // String is not `Copy`
}
```

### ❓哪些枚举应该是 `Copy`？
* 如果你的枚举像标签和原子值一样。
* 枚举的所有载荷（payload）都是 `Copy`。
* **❗枚举的大小取决于其最大的元素。**

### ✅ 适合派生的枚举
```rust
#[derive(Debug, Copy, Clone)]
enum Direction {
    North,
    South,
    East,
    West,
}
```

## 1.3 处理 `Option<T>` 和 `Result<T, E>`
Rust 1.65 引入了一种更好的方式来安全地解包 Option 和 Result 类型：当你有一个默认的 `return` 值、`continue` 或 `break` 作为 else 分支时，可以使用 `let Some(x) = … else { … }` 或 `let Ok(x) = … else { … }`。当缺失的情况是**预期且正常的**（而非异常）时，它允许提前返回。

### ✅ Option 和 Result 各模式匹配的适用场景
* 当你想对内部类型 `T` 和 `E` 进行模式匹配时，使用 `match`
```rust
match self {
    Ok(Direction::South) => { … },
    Ok(Direction::North) => { … },
    Ok(Direction::East) => { … },
    Ok(Direction::West) => { … },
    Err(E::One) => { … },
    Err(E::Two) => { … },
}

match self {
    Some(3|5) => { … }
    Some(x) if x > 10 => { … }
    Some(x) => { … }
    None => { … }
}
```

* 当你的类型被转换为更复杂的东西时使用 `match`，例如 `Result<T, E>` 变成 `Result<Option<T>, E>`。
```rust
match self {
    Ok(t) => Ok(Some(t)),
    Err(E::Empty) => Ok(None),
    Err(err) => Err(err),
}
```

* 当发散代码（diverging code）不需要知道模式匹配失败的情况，或不需要额外计算时，使用 `let PATTERN = EXPRESSION else { DIVERGING_CODE; }`：
```rust
let Some(&Direction::North) = self.direction.as_ref() else {
    return Err(DirectionNotAvailable(self.direction));
}
```

* 当你想在模式匹配中 `break` 或 `continue` 时，使用 `let PATTERN = EXPRESSION else { DIVERGING_CODE; }`
```rust
for x in self {
    let Some(x) = x else {
        continue;
    }
}
```

* 当 `DIVERGING_CODE` 需要额外计算时，使用 `if let PATTERN = EXPRESSION else { DIVERGING_CODE; }`：
```rust
if let Some(x) = self.next() {
    // computation
} else {
    // computation when `None/Err` or not matched
}
```

❗**如果你不关心 `Err` 情况的值，请使用 `?` 将 `Err` 传播给调用者。**

### ❌ 糟糕的 Option/Return 模式匹配：

* Result 与 Option 之间的转换（优先使用 `.ok()`、`.ok_or()` 和 `ok_or_else()`）
```rust
match self {
    Ok(t) => Some(t),
    Err(_) => None
}
```

* 当发散代码是默认值或预先计算的值时使用 `if let PATTERN = EXPRESSION else { DIVERGING_CODE; }`（优先使用 `let PATTERN = EXPRESSION else { DIVERGING_CODE; }`）：
```rust
if let Some(values) = self.next() {
    // computation
    (Some(..), values)
} else {
    (None, Vec::new())
}
```

* 在测试之外使用 `unwrap` 或 `expect`：
```rust
let port = config.port.unwrap();
```

## 1.4 防止提前分配

在处理 `or`、`map_or`、`unwrap_or`、`ok_or` 这类函数时，要考虑它们在需要内存分配时有特殊情形，比如创建新字符串、创建集合，甚至调用管理某些状态的函数，因此可以把它们替换为带 `_else` 的对应版本：

### ✅ 好的用法

```rust
let x = None;
assert_eq!(x.ok_or(ParseError::ValueAbsent), Err(ParseError::ValueAbsent));

let x = None;
assert_eq!(x.ok_or_else(|| ParseError::ValueAbsent(format!("this is a value {x}"))), Err(ParseError::ValueAbsent));


let x: Result<_, &str> = Ok("foo");
assert_eq!(x.map_or(42, |v| v.len()), 3);


let x : Result<_, String> = Ok("foo");
assert_eq!(x.map_or_else(|e|format!("Error: {e}"), |v| v.len()), 3);

let x = "1,2,3,4";
assert_eq!(x.parse_to_option_vec.unwrap_or_else(Vec::new), Ok(vec![1, 2, 3, 4]));
```

### ❌ 不好的用法

```rust
let x : Result<_, String> = Ok("foo");
assert_eq!(x.map_or(format!("Error with uninformed content"), |v| v.len()), 3);

let x = "1,2,3,4";
assert_eq!(x.parse_to_option_vec.unwrap_or(Vec::new()), Ok(vec![1, 2, 3, 4])); // could be replaced with `.unwrap_or_default`

let x = None;
assert_eq!(x.ok_or(ParseError::ValueAbsent(format!("this is a value {x}"))), Err(ParseError::ValueAbsent));
```

### 映射 Err

在处理 `Result::Err` 时，有时需要记录日志并将 Err 转换为更抽象或更详细的错误，这可以通过 `inspect_err` 和 `map_err` 完成：

```rust
let x = Err(ParseError::InvalidContent(...));

x
    .inspect_err(|err| tracing::error!("function_name: {err}"))
    .map_err(|err| GeneralError::from(("function_name", err)))?;
```

## 1.5 迭代器，`.iter` 与 `for`

首先我们需要理解使用它们各自实现基本循环的方式。考虑以下问题：我们需要对 0 到 10 之间所有偶数加 1 后求和：

* `for`：
```rust
let mut sum = 0;
for x in 0..=10 {
    if x % 2 == 0 {
        sum += x + 1;
    }
}
```

* `iter`：
```rust
let sum: i32 = (0..=10)
    .filter(|x| x % 2 == 0)
    .map(|x| x + 1)
    .sum();
```

> 两个版本做同样的事，都正确且符合惯用法，但它们各自在不同场景下更出彩。

### 何时优先使用 `for` 循环
* 当你需要**提前退出**（`break`、`continue`、`return`）时。
* 带有副作用的**简单迭代**（例如日志、IO）
    * 日志可以通过 `inspect` 和 `inspect_err` 函数在 `Iterators` 中正确地完成。
* 当可读性比简洁或链式调用更重要时。

#### 示例：
```rust
for value in &mut value {
    if *value == 0 {
        break;
    }
    *value += fancy_equation();
}
```

### 何时优先使用 `iterators` 循环（`.iter()` 和 `.into_iter()`）
* 当你需要转换集合（`transforming collections`）或 `Option/Results` 时。
* 你可以优雅地**组合多个步骤**。
* 不需要提前退出。
* 你需要通过 `.enumerate` 支持带索引的值。
```rust
let values: Vec<_> = vec.into_iter()
    .enumerate()
    .filter(|(_index, value)| value % 2 == 0)
    .map(|(index, value)| value % index)
    .collect()
```
* 你需要使用像 `.windows` 或 `chunks` 这样的集合函数。
* 你需要合并来自多个数据源的数据，且不想分配多个集合。
* 迭代器可以与 `for` 循环结合：
```rust
for value in vec.iter().enumerate()
    .filter(|(index, value)| value % index == 0) {
    // ...
}
```

> #### ❗记住：迭代器是惰性的
>
> * `.iter`、`.map`、`.filter` 在你调用其消费者（如 `.collect`、`.sum`、`.for_each`）之前不会做任何事。
> * **惰性求值**意味着迭代器链会在编译期融合成一个循环。

### 🚨 应避免的反模式

* 不要不加格式地链式调用。优先让每个链式函数独占一行并使用正确的缩进（`rustfmt` 应会处理这一点）。
* 如果链式调用让代码难以阅读，就不要这样做。
* 避免不必要地 collect/分配一个集合（例如 vector），只是为了稍后通过某个更大的操作或另一次迭代将其丢弃。
* 优先使用 `iter` 而非 `into_iter`，除非你不需要集合的所有权。
* 对于内部类型实现了 `Copy` 的集合（例如 `Vec<i32>`），优先使用 `iter` 而非 `into_iter`。
* 求和时优先使用 `.sum` 而非 `.fold`。`.sum` 专门用于求和值，因此编译器知道可以在这一方面进行优化，而 fold 有一个需要在每一步应用的黑盒闭包。如果你需要从初始值开始求和，只需在表达式中加上即可：`let my_sum = [1, 2, 3].sum() + 3`。

## 1.6 注释：提供上下文，而非杂讯

> “上下文是为了说明为什么，而不是什么或怎么做”

编写良好、类型富有表现力且命名恰当的 Rust 代码往往不言自明。许多高质量代码库依靠**很少或没有注释**也能蓬勃发展。这是件好事。

尽管如此，仍有**仅靠代码不够的时刻**——当存在性能怪癖、外部约束或不那么显而易见的权衡，需要给读者一点提示时。在这些情况下，一条简洁的注释可以避免数小时的困惑或翻阅 git 历史。

### ✅ 好的注释 

* 安全性问题：
```rust
// SAFETY: We have checked that the pointer is valid and non-null. @Function xyz.
unsafe { std::ptr::copy_nonoverlapping(src, dst, len); }
```

* 性能怪癖：
```rust
// This algorithm is a fast square root approximation
const THREE_HALVES: f32 = 1.5;
fn q_rsqrt(number: f32 ) -> f32 {
    let mut i: i32 = number.to_bits() as i32;
    i = 0x5F375A86_i32.wrapping_sub(i >> 1);
    let y = f32::from_bits(i as u32);
    y * (THREE_HALVES - (number * 0.5 * y * y))
}
```

* 清晰的代码胜过注释。然而，当“为什么”不明显时，就直白地说出来——或者链接到相关位置：
```rust
// PERF: Generating the root store per subgraph caused high TLS startup latency on MacOS
// This works as a caching alternative. See: [ADR-123](link/to/adr-123)
let subgraph_tls_root_store: RootCertStore = configuration
    .tls
    .subgraph
    .all
    .create_certificate_store()
    .transpose()?
    .unwrap_or_else(crate::services::http::HttpClientService::native_roots_store);
```

### ❌ 糟糕的注释

* 长篇大论的解释：长注释和多行注释
```rust
// Lorem Ipsum is simply dummy text of the printing and typesetting industry. 
// Lorem Ipsum has been the industry's standard dummy text ever since the 1500s, 
// when an unknown printer took a galley
fn do_something_odd() {
    …
}
```
> 如果是在描述函数，优先使用 `/// doc` 注释。

* 本可以更好地表示为函数，或者纯粹显而易见的注释
```rust
fn computation() {
    // increment i by 1
    i += 1;
}
```

### ✅ 拆分长函数而非为其写注释

如果你发现自己在一个函数中写长注释来解释“是什么”、“怎么做”或“每一步”，那可能是时候将其拆分了。因此建议进行重构。这不仅有利于可读性，也有利于可测试性：

#### ❌ 不要这样：
```rust
fn process_request(request: T) {
    // We first need to validate request, because of corner case x, y, z
    // As the payload can only be decoded when they are valid
    // Then we can perform authorization on the payload
    // lastly with the authorized payload we can dispatch to handler
}
```

#### ✅ 优先这样：
```rust
fn process_request(request: T) -> Result<(), Error> {
    validate_request_headers(&request)?;
    let payload = decode_payload(&request);
    authorize(&payload)?;
    dispatch_to_handler(payload)
}

#[cfg(test)]
mod tests {
    #[test]
    fn validate_request_happy_path() { ... }

    #[test]
    fn validate_request_fails_on_x() { ... }

    #[test]
    fn validate_request_fails_on_y() { ... }

    #[test]
    fn decode_validated_request() { ... }

    #[test]
    fn authorize_payload_xyz() { ... }
}
```

让**结构**和**命名**取代注释，并用**测试作为活文档**来增强其文档。

### 📝 TODO 不是注释 - 要妥善跟踪它们

避免在代码中留下悬而未决的 `// TODO: Lorem Ipsum` 注释。取而代之：
* 把它们转换为 Jira 或 Github Issue。
* 如有必要，为避免将来混淆，在代码中引用 issue，并在 issue 中引用代码。

```rust
// See issue #123: support hyper 2.0
```

这有助于保持代码整洁，并确保任务不会被遗忘。

### 注释作为活文档

把注释称为“活文档”时有一些陷阱：
* 代码会演进。
* 上下文会变化。
* 注释会过时。
* 大量注释会让人不愿阅读。
* 团队会害怕删除无关的注释。

如果你发现一条注释，**不要盲目信任它**。结合上下文来阅读。如果它是错误的或过时的，就修正或删除它。一条误导性的注释比完全没有注释更糟。 

> 注释应当让你感到不安——它们要求重新验证，就像过时的测试一样。

当需要更深入的理由说明时，优先：
* **链接到设计文档或 ADR**，业务逻辑适合放在设计文档中，而性能权衡适合放在 ADR 中。
* 将运行时示例和用法文档移入 Rust Docs，即 `/// doc comment`，在那里它们可以被测试，并由 `cargo doc` 等工具保持最新。

> 文档注释和文档测试，`///` 和 `//!` 见 [第 8 章 - 注释与文档](./chapter_08.md)

## 1.7 Use 声明 - “imports”（导入）

不同语言有不同的导入排序方式，在 Rust 生态中，[标准方式](https://github.com/rust-lang/rustfmt/issues/4107)是：

- `std`（`core`、`alloc` 也可归在此处）。
- 外部 crates（Cargo.toml `[dependencies]` 中的内容）。
- 工作区 crates（workspace 成员 crates）。
- 本模块 `super::`。
- 本模块 `crate::`。

```rust
// std
use std::sync::Arc;

// external crates
use chrono::Utc;
use juniper::{FieldError, FieldResult};
use uuid::Uuid;

// crate code lives in workspace
use broker::database::PooledConnection;

// super:: / crate::
use super::schema::{Context, Payload};
use super::update::convert_publish_payload;
use crate::models::Event;
```

一些企业级解决方案选择将其核心包放在 `std` 之后，这样所有以企业名开头的外部包都会位于其他包之前：

```rust
// std
use std::sync::Arc;

// enterprise external crates
use enterprise_crate_name::some_module::SomeThing;

// external crates
use chrono::Utc;
use juniper::{FieldError, FieldResult};
use uuid::Uuid;

// crate code lives in workspace
use broker::database::PooledConnection;

// super:: / crate::
use super::schema::{Context, Payload};
use super::update::convert_publish_payload;
use crate::models::Event;
```

一种无需手动控制这一点的方法是使用 `rustfmt.toml` 中的以下参数：

```toml
reorder_imports = true
imports_granularity = "Crate"
group_imports = "StdExternalCrate"
```

> 截至 Rust 1.88 版本，需要在 nightly 下执行 rustfmt 才能正确重排代码：`cargo +nightly fmt`。
