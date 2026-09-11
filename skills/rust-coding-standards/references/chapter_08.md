# 第 8 章 - 注释与文档

> 清晰的代码胜过清晰的注释。然而，当“为什么”不明显时，就直白地注释——或者链接到可以阅读更多上下文的地方。

## 8.1 注释与文档：了解它们的区别

| 用途 | 使用 `// comment` | 使用 `/// doc` 或 `//! crate doc` |
|-------------- |------------------------------------------- |---------------------------------------------------------------- |
| 描述为什么 | ✅ 是 - 解释棘手的推理 | ❌ 不用于文档 |
| 描述 API | ❌ 没用 | ✅ 是 - 公共接口、用法、细节、错误、panic |
| 可维护性 | 🚨 常常变得过时且难以推理 | ✅ 与代码绑定，出现在生成的文档中，并可运行测试用例 |
| 可见性 | 仅限本地开发 | 导出给用户和 `cargo doc` 等工具 |

## 8.2 何时使用注释

当某些事情无法在代码中清晰表达时，使用 `//` 注释（双斜杠），例如：
* **安全保证**，其中一些可以用代码条件更好地表达。
* 变通方法或**优化**。
* 遗留或**平台特定**行为。其中一些可以用 `#[cfg(..)]` 表达。
* 指向**设计文档**或 **ADR** 的链接。
* 不明显的假设或**陷阱**。

> 给你的注释命名！例如，关于安全保证的注释应以 `// SAFETY: ...` 开头。

### ✅ 好的注释：
```rust
// SAFETY: `ptr` is guaranteed to be non-null and aligned by caller
unsafe { std::ptr::copy_nonoverlapping(src, dst, len); }
```

### ✅ 设计上下文注释：
```rust
// CONTEXT: Reuse root cert store across subgraphs to avoid duplicate OS calls:
// [ADR-12](link/to/adr-12): TLS Performance on MacOS
```

## 8.3 注释何时会妨碍阅读

避免这些注释：
* 重述显而易见的事情（`// increment i by 1 for the next loop`）。
* 会随时间而过时的。
* 没有行动的 `TODO`（链接到某个带版本号的 issue）。
* 可以用更好的命名或更小的函数取代的。

### ❌ 糟糕的注释：
```rust
fn compute(counter: &mut usize) {
    // increment by 1
    *counter += 1;
}
```

### ❌ 太长或过时
```rust
// Originally written in 2028 for some now-defunct platform
```

## 8.4 不要编写活文档（活的注释）

把注释当作“活文档”是一个**危险的迷思**，因为注释**不是免费的**：
* 它们会**腐烂**——没人编译注释。
* 它们会**误导**——读者通常不加批判地假定它们是真的，例如“另一个开发者比我更了解这段代码”。
* 它们会**过时**——除非随代码维护，否则它们会变得无关紧要。
* 它们是**噪音**——注释可能会用多行不必要的内容把你的代码弄得杂乱。

如果某事值得在 PR 之后继续存在，就把它放进：
* 一份 **ADR**（架构设计记录）。
* 一份设计文档。
* 通过使用类型、文档注释、示例、将代码块重命名为更清晰的函数来**在代码中**记录它。
* 添加测试来覆盖并解释该变更。

> ### 🚨 如果你发现一条注释，**结合上下文阅读它**。它是否仍然说得通？如果不是，就删除或更新它，或寻求帮助。注释应当让你感到不安。

## 8.5 用代码取代注释

与其使用冗长的注释块，不如把逻辑拆分成命名的辅助函数：

#### ❌ 带注释的代码块：
```rust
fn save_user(&self) -> Result<(), MyError> {
    // check if the user is authenticated
    if self.is_authenticated() {
        // serialize user data
        let data = serde_json::to_string(self)?;
        // write to file
        std::fs::write(self.path(), data)?;
    }
}
```
**✅ 提取出来以获得清晰性**：

```rust
fn save_auth_user(&self) -> Result<PathBuf, MyError> {
    if self.is_authenticated() {
        let path = self.path();
        let serialized_user = serde_json::to_string(self)?;
        std::fs::write(path, serialized_user)?;
        Ok(path)
    } else {
        Err(MyError::UserNotAuthenticated)
    }
}
```

## 8.6 `TODO` 应该变成 issue

不要让 `// TODO:` 散落在代码库中而无人负责。取而代之：
1. 提交 Github Issue 或 Jira Ticket。（公共仓库优先使用 github issues）。
2. 在代码中引用该 issue：

```rust
// TODO(issue #42): Remove workaround after bugfix
```

这使 `TODO` 可跟踪、可执行，并且对所有人都可见。

## 8.7 何时使用文档注释

使用 `///` 文档注释来记录：
* 所有**公共函数、结构体、trait、枚举**。
* 它们的用途、用法和行为。
* 开发者正确使用它们所需了解的一切。
* 添加与 `Errors` 和 `Panics` 相关的上下文。
* 大量示例。

### ✅ 好的文档注释：

```rust
/// Loads [`User`] profile from disk
/// 
/// # Error
/// - Returns [`MyError`] if the file is missing [`MyError::FileNotFound`].
/// - Returns [`MyError`] if the content is an invalid Json, [`MyError::InvalidJson`].
fn load_user(path: &Path) -> Result<User, MyError> {...}
```

**文档注释还可以包含示例、链接甚至测试：**

```rust
/// Returns the square of the integer part of any number.
/// Square is limited to `u128`.
/// 
/// # Examples
/// 
/// ```rust
/// assert_eq!(square(4.3), 16)
/// ```
fn square(x: impl ToInt) -> u128 { ... }
```

## 8.8 Rust 中的文档：如何、何时以及为什么

Rust 通过 rustdoc 提供**一流的文档工具**，这使记录代码成为编写地道且可维护的 rust 的关键部分。有一些文档专用的 lint 可以帮助处理文档，比如：

| Lint | 描述 |
|-------------- |------------------------------------------- |
| [missing_docs](https://doc.rust-lang.org/rustdoc/lints.html#missing_docs) | 警告公共函数、结构体、常量、枚举缺少文档 |
| [broken_intra_doc_links](https://doc.rust-lang.org/rustdoc/lints.html#broken_intra_doc_links) | 检测内部文档链接是否损坏。在重命名时特别有用。 |
| [empty_docs](https://rust-lang.github.io/rust-clippy/master/#empty_docs) | 禁止空文档 - 防止绕过 `missing_docs` |
| [missing_panics_doc](https://rust-lang.github.io/rust-clippy/master/#missing_panics_doc) | 警告如果函数可能 panic，文档应包含 `# Panics` 节 |
| [missing_errors_doc](https://rust-lang.github.io/rust-clippy/master/#missing_errors_doc) | 警告如果函数返回 `Result`，文档应包含 `# Errors` 节来解释 `Err` 条件 |
| [missing_safety_doc](https://rust-lang.github.io/rust-clippy/master/#missing_safety_doc) | 警告如果面向公众的函数有可见的 unsafe 块，文档应包含 `# Safety` 节 |


### `///` 与 `//!` 的区别

| 风格 | 用于 | 范围 | 示例 |
|---------- |------------------------------ |------------------------------------------- |---------------------------------------------------------------- |
| `///` | 行文档注释 | 结构体、fn、枚举、常量等公共条目 | 为 `fn`、`struct`、`enum` 等记录、提供上下文和用法 |
| `//!` | 模块级文档注释 | 模块或整个 crate | 解释 crate/模块的用途，附常见用例和快速入门 |

### `///` 条目级文档

对函数、结构体、trait、枚举、常量等使用 `///`：

```rust
/// Adds two numbers together.
///
/// # Examples
///
/// ```
/// let result = my_crate::add(2, 3);
/// assert_eq!(result, 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```
* ✅ 写清楚且描述性的**它做什么**和**如何使用它**。
* ✅ 使用 `# Examples` 节来更好地解释**如何使用它**。
* ✅ 优先编写可以通过 `cargo test` 测试的示例，即使你不得不以 `#` 开头来隐藏它们的输出：
```rust
/// ```
/// let result = my_crate::add(2, 3);
/// # assert_eq!(result, 5);
/// ```
```
* ✅ 在相关时使用 `# Panics`、`# Errors` 和 `# Safety` 节。
* 为类型添加相关上下文。

### `//!` 模块/Crate 级文档

当你想记录**模块或 crate 的用途**时使用 `//!`。它位于 `lib.rs` 或 `mod.rs` 文件的顶部，例如 `engine/mod.rs`：
```rust
//! This module implements a custom chess engine.
//! 
//! It handles board state, move generation and check detection.
//! 
//! # Example
//! ```
//! let board = chess::engine::Board::default();
//! assert!(board.is_valid());
//! ```
```

## 8.9 文档覆盖清单

📦 Crate 级（lib.rs）
- [ ] 顶部的 `//!` 文档解释**crate 做什么**，以及**它解决什么问题**。
- [ ] 包含 crate 级的 `# Examples` 或指向模块的链接。

📁 模块（mod.rs 或内联）
- [ ] `//!` 文档解释**这个模块是做什么的**、它的**导出**和**不变式**。
- [ ] 除非需要澄清，否则避免在重导出的条目上重复文档注释。

🧱 结构体、枚举、Trait
- `///` 文档解释：
    - [ ] 该类型所扮演的角色。
    - [ ] 不变式或期望。
    - [ ] 构造或用法示例。
- [ ] 如果外部用户可能对它做匹配，考虑使用 [`#[non_exhaustive]`](https://doc.rust-lang.org/reference/attributes/type_system.html#the-non_exhaustive-attribute)。

🔧 函数与方法
- `///` 文档涵盖：
    - [ ] 它做什么。
    - [ ] 参数及其含义。
    - [ ] 返回值行为。
    - [ ] 边界情况（`# Panics`、`# Errors`）。
    - [ ] 用法示例，`# Examples`。

📑 Trait
- [ ] 解释该 trait 的**用途**（标记？动态分发？）。
- [ ] 为每个方法写文档——包括**何时/为何**实现它。
- [ ] 清楚地记录默认实现的方法以及何时应覆盖。

📦 公共常量
- [ ] 记录它们配置什么，以及何时你会想使用它们。

### 📌 最佳实践
* ✅ 大方地使用示例——它们同时也是测试用例。
* ✅ 优先清晰而非形式——它是写给人而非机器的。
* ✅ 优先用文档注释解释用法，必要时把实现细节留给代码注释。
* ✅ 经常用 `cargo doc --open` 检查你的输出。
* ✅ 如果你想强制完整的文档覆盖，在顶层模块添加 `#![deny(missing_docs)]` 和其他相关的文档 lint。
