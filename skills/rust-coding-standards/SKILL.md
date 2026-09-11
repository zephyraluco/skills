---
name: rust-coding-standards
description: >
  Rust 最佳实践指南，在以下情况使用本技能：
  (1) 编写新的 Rust 代码或函数，
  (2) 审查或重构现有 Rust 代码，
  (3) 在借用与克隆或所有权模式之间做决策，
  (4) 使用 Result 类型实现错误处理，
  (5) 优化 Rust 代码性能，
  (6) 为 Rust 项目编写测试或文档。
license: MIT
compatibility: Rust 1.88+, Cargo
metadata:
  author: zeal
  version: "0.0.1"
allowed-tools: Bash(cargo:*) Bash(rustc:*) Bash(rustfmt:*) Bash(clippy:*) Read Write Edit Glob Grep
---

# Rust 最佳实践

在编写或审查 Rust 代码时应用这些准则。内容基于 Apollo GraphQL 的 [Rust Best Practices Handbook](https://github.com/apollographql/rust-best-practices)。

## 最佳实践参考

在审查之前，请先熟悉 Apollo 的 Rust 最佳实践。在同一次回复中并行阅读所有相关章节。提供反馈时参考以下文件：

- [第 1 章 - 编码风格与惯用法](references/chapter_01.md)：借用与克隆、Copy trait、Option/Result 处理、迭代器、注释
- [第 2 章 - Clippy 与 Lint](references/chapter_02.md)：Clippy 配置、重要 lint、工作区 lint 设置
- [第 3 章 - 性能思维](references/chapter_03.md)：性能分析、避免多余克隆、栈与堆、零成本抽象
- [第 4 章 - 错误处理](references/chapter_04.md)：Result 与 panic、thiserror 与 anyhow、错误层级
- [第 5 章 - 自动化测试](references/chapter_05.md)：测试命名、每个测试一个断言、快照测试
- [第 6 章 - 泛型与分发](references/chapter_06.md)：静态与动态分发、trait 对象
- [第 7 章 - 类型状态模式](references/chapter_07.md)：编译期状态安全、何时使用
- [第 8 章 - 注释与文档](references/chapter_08.md)：何时注释、文档注释、rustdoc
- [第 9 章 - 理解指针](references/chapter_09.md)：线程安全、Send/Sync、指针类型

## 快速参考

### 借用与所有权
- 优先使用 `&T` 而非 `.clone()`，除非需要转移所有权
- 函数参数中优先使用 `&str` 而非 `String`，`&[T]` 而非 `Vec<T>`
- 小的 `Copy` 类型（≤24 字节）可以按值传递
- 当所有权不明确时使用 `Cow<'_, T>`

### 错误处理
- 对可能失败的操作返回 `Result<T, E>`；生产环境避免 `panic!`
- 测试之外永远不要使用 `unwrap()`/`expect()`
- 库错误使用 `thiserror`，二进制程序才用 `anyhow`
- 错误传播优先使用 `?` 运算符而非 match 链

### 性能
- 始终使用 `--release` 标志进行基准测试
- 运行 `cargo clippy -- -D clippy::perf` 获取性能提示
- 避免在循环中克隆；对 Copy 类型使用 `.iter()` 而非 `.into_iter()`
- 优先使用迭代器而非手写循环；避免中间 `.collect()` 调用

### Lint（静态检查）
定期运行：`cargo clippy --all-targets --all-features --locked -- -D warnings`

需要关注的关键 lint：
- `redundant_clone` - 不必要的克隆
- `large_enum_variant` - 过大的变体（考虑装箱）
- `needless_collect` - 过早的收集

使用 `#[expect(clippy::lint)]` 而非 `#[allow(...)]`，并附上理由注释。

### 测试
- 使用描述性的测试名：`process_should_return_error_when_input_empty()`
- 尽可能每个测试只有一个断言
- 公共 API 示例使用文档测试（`///`）
- 考虑使用 `cargo insta` 对生成的输出做快照测试

### 泛型与分发
- 性能关键代码优先使用泛型（静态分发）
- 仅在需要异构集合时使用 `dyn Trait`
- 在 API 边界装箱，而非内部

### 类型状态模式
将有效状态编码进类型系统，以在编译期捕获非法操作：
```rust
struct Connection<State> { /* ... */ _state: PhantomData<State> }
struct Disconnected;
struct Connected;

impl Connection<Connected> {
    fn send(&self, data: &[u8]) { /* only connected can send */ }
}
```

### 文档
- `//` 注释解释*为什么*（安全性、变通方法、设计理由）
- `///` 文档注释为公共 API 解释*是什么*和*怎么用*
- 每个 `TODO` 都需要关联一个 issue：`// TODO(#42): ...`
- 为库启用 `#![deny(missing_docs)]`
