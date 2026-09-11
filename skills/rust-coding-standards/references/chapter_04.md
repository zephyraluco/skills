# 第 4 章 - 错误处理

Rust 强制执行严格的错误处理方式，但你*如何*处理它们决定了代码是让人感到顺手、一致且安全，还是晦涩而痛苦。本章深入探讨跨库与二进制程序中对可失败操作进行建模和管理的最佳实践。

> 即使你决定用 `unwrap` 或 `expect` 让应用崩溃，Rust 也强制你明确表达这是有意为之。

## 4.1 优先使用 `Result`，避免 panic 🫨

Rust 有一个强大的类型用于包裹可失败数据：[`Result<T, E>`](https://doc.rust-lang.org/std/result/)，它使我们能够根据需求处理错误情况，并据此管理应用状态。

* 如果你的函数可能失败，优先返回 `Result`：
```rust
fn divide(x: f64, y: f64) -> Result<f64, DivisionError> {
    if y == 0.0 {
        Err(DivisionError::DividedByZero)
    } else {
        Ok(x / y)
    }
}
```

* 仅在不可恢复的情况下使用 `panic!` —— 通常是测试、断言、bug，或出于某些明确原因需要让应用崩溃。
* 有 3 个相关的宏可以在适当条件下替代 `panic!`：
    * `todo!`，类似于 panic，但会提醒编译器你知道有代码缺失。
    * `unreachable!`，你已经对代码块做了推理并确定 `xyz` 条件不可能发生，如果万一可能发生，你希望被提醒。
    * `unimplemented!`，特别适用于提醒某块尚未实现并给出原因。

## 4.2 生产环境中避免 `unwrap`/`expect`

虽然 `expect` 比 `unwrap` 更受青睐，因为它可以携带上下文，但在生产代码中应避免使用它们，因为有更聪明的替代方案。考虑到这一点，它们应用在以下场景：
- 测试、断言或测试辅助函数中。
- 当失败不可能发生时。
- 当更聪明的选项无法处理特定情况时。

### 🚨 处理 `unwrap`/`expect` 的替代方式：

* 如果你的 `Result`（或 `Option`）在 `Result::Err` 时有一个预定义的提前返回值，且不需要知道 `Err` 的值，使用 `let Ok(..) = else { return ... }` 模式，因为它有助于扁平化函数：
```rust
let Ok(json) = serde_json::from_str(&input) else {
    return Err(MyError::InvalidJson);
}
```
* 如果你的 `Result`（或 `Option`）在 `Result::Err` 时需要错误恢复，且不需要知道 `Err` 的值，使用 `if let Ok(..) else { ... }` 模式：
```rust
if let Ok(json) = serde_json::from_str(&input) else {
    ...
} else {
    Err(do_something_with_input(&input))
}
```
* 可能需要处理 `Option::None` 值的函数建议返回 `Result<T, E>`，其中 `E` 是 crate 或模块级别的错误，如上面的例子。
* 最后是 `unwrap_or`、`unwrap_or_else` 或 `unwrap_or_default`，这些函数帮助你为解包创建替代退出路径，以管理未初始化的值。

## 4.3 使用 `thiserror` 处理 Crate 级别的错误

手动派生 Error 既冗长又容易出错，Rust 生态有一个非常好的 crate 来帮助解决这个问题：`thiserror`。它允许你创建错误类型，轻松实现 `From` trait 以及简单的错误消息（`Display`），改善开发者体验，同时与 `?` 无缝协作并集成 `std::error::Error`：

```rust
#[derive(Debug, thiserror::Error)]
pub enum MyError {
    #[error("Network Timeout")]
    Timeout,
    #[error("Invalid data: {0}")]
    InvalidData(String),
    #[error(transparent)]
    Serialization(#[from] serde_json::Error),
    #[error("Invalid request information. Header: {headers}, Metadata: {metadata}")]
    InvalidRequest {
        headers: Headers,
        metadata: Metadata
    }
}
```

### 错误层级与包装

对于分层系统，最佳实践是使用嵌套的 `enum/struct` 错误配合 `#[from]`：

```rust
use crate::database::DbError;
use crate::external_services::ExternalHttpError;

#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("Database handler error: {0}")]
    Db(#[from] DbError),
    #[error("External services error: {0}")]
    ExternalServices(#[from] ExternalHttpError)
}
```

## 4.4 把 `anyhow` 留给二进制程序

`anyhow` 是一个很棒的 crate，对于处于起步阶段、需要加速开发的项目非常有用。然而，存在一个转折点，它会在你的代码中痛苦地传递，考虑到这一点，`anyhow` 仅推荐用于**二进制程序**，这类场景需要符合人体工学的错误处理，且不需要精确的错误类型：

```rust
use anyhow::{Context, Result, anyhow};

fn main() -> Result<()> {
    let content = std::fs::read_to_string("config.json")
        .context("Failed to read config file")?;
    Config::from_str(&content)
        .map_err(|err| anyhow!("Config parsing error: {err}"))
}
```

### 🚨 `Anyhow` 的坑

* 在整个代码库中保持 `context` 和 `anyhow` 字符串最新比维护 `thiserror` 消息更难，因为后者有单一入口点。
* `anyhow::Result` 会抹除调用者可能需要的上下文，因此避免在库中使用它。
* 测试辅助函数可以放心使用 `anyhow`，几乎不会有什么问题。

## 4.5 使用 `?` 冒泡错误

优先使用 `?` 而非 `match` 链这类冗长的替代方案：
```rust
fn handle_request(req: &Request) -> Result<ValidatedRequest, MyError> {
    validate_headers(req)?;
    validate_body_format(req)?;
    validate_credentials(req)?;
    let body = Body::try_from(req)?;

    Ok(ValidatedRequest::try_from((req, body))?)
}
```

> 如果需要错误恢复，使用 `or_else`、`map_err`、`if let Ok(..) else`。要**检查或记录错误**，使用 `inspect_err`。

## 4.6 单元测试应该覆盖错误

许多错误没有实现 PartialEq 和 Eq，因此很难在它们之间做直接断言，但可以用 `format!` 或 `to_string()` 检查错误消息，使错误有意义并通过测试验证：

```rust
#[test]
fn error_does_not_implement_partial_eq() {
    let err = divide(10., 0.0).unwrap_err();
    assert_eq!(err.to_string(), "division by zero");
}

#[test]
fn error_implements_partial_eq() {
    let err = process(my_value).unwrap_err();

    assert_eq!(
        err,
        MyError {
            ..
        }
    )
}
```

## 4.7 重要主题

### 自定义错误结构体

有时你不需要用枚举来处理错误，因为你的模块只可能有一种错误。这可以用 `struct Errors` 解决：

```rust
#[derive(Debug, thiserror::Error, PartialEq)]
#[error("Request failed with code `{code}`: {message}")]
struct HttpError {
    code: u16,
    message: String
}
```

### 异步错误

使用异步运行时（如 Tokio）时，确保你的错误在需要的地方实现 `Send + Sync + 'static`，尤其是在任务中或跨 `.await` 边界时：

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    ...
    Ok(())
}
```

> 除非确实需要，否则避免在库中使用 `Box<dyn std::error::Error>`
