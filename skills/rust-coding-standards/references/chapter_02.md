# 第 2 章 - Clippy 与 Lint 纪律

请确保你的 Rust 编译器已安装 `cargo clippy`。在 rust 项目中于终端运行 `cargo clippy -V`，你应该会看到类似这样的输出 `clippy 0.1.86 (05f9846f89 2025-03-31)`。如果终端未能显示 clippy 版本，请运行以下命令 `rustup update && rustup component add clippy`。

Clippy 的文档可在[此处](https://doc.rust-lang.org/clippy/usage.html)找到。

## 2.1 为什么要在意 lint？

Rust 编译器是一个强大的工具，能捕获许多错误。然而，一些更深入的分析需要额外的工具，这正是 `cargo clippy` 发挥作用的地方。Clippy 检查：
* 性能陷阱。
* 风格问题。
* 冗余代码。
* 潜在 bug。
* 非地道的 Rust 代码。

## 2.2 始终运行 `cargo clippy`

将以下内容加入你的日常工作流：

```shell
$ cargo clippy --all-targets --all-features --locked -- -D warnings
```

* `--all-targets`：检查库、测试、基准测试和示例。
* `--all-features`：检查所有特性开启时的代码，自动解决相互冲突的特性。
* `--locked`：要求 `Cargo.lock` 是最新的，可通过 `$ cargo update` 解决。
* `-D warnings`：将警告视为错误

可以额外添加的选项：

* `-- -W clippy::pedantic`：相当严格或偶尔误报的 lint。
* `-- -W clippy::nursery`：可选添加，用于检查仍在开发中的新 lint。
* ❗ 将其添加到你的 Makefile、Justfile、xtask 或 CI 流水线中。

> ApolloGraphQL 的示例
>
> 在 `Router` 项目中有一个为 lint 配置的 `xtask`，可以通过 `cargo xtask lint` 执行。

## 2.3 需要重视的重要 Clippy Lint

| Lint 名称 | 原因 | 链接 |
| --------- | ----| -----|
| `redundant_clone` | 检测不必要的 `clone`，有性能影响 | [链接（nursery + perf）](https://rust-lang.github.io/rust-clippy/master/#redundant_clone) |
| `needless_borrow` 组 | 移除多余的 `&` 借用 | [链接（style）](https://rust-lang.github.io/rust-clippy/master/#needless_borrow) |
| `map_unwrap_or` / `map_or` | 简化嵌套的 `Option/Result` 处理 | [`map_unwrap_or`](https://rust-lang.github.io/rust-clippy/master/#map_unwrap_or) [`unnecessary_map_or`](https://rust-lang.github.io/rust-clippy/master/#unnecessary_map_or) [`unnecessary_result_map_or_else`](https://rust-lang.github.io/rust-clippy/master/#unnecessary_result_map_or_else) |
| `manual_ok_or` | 建议使用 `.ok_or_else` 而不是 `match` | [链接（style）](https://rust-lang.github.io/rust-clippy/master/#manual_ok_or) |
| `large_enum_variant` | 如果枚举有非常大的变体（对内存不利）则发出警告，建议对其 `Box` | [链接（perf）](https://rust-lang.github.io/rust-clippy/master/#large_enum_variant) |
| `unnecessary_wraps` | 如果你的函数总是返回 `Some` 或 `Ok`，你不需要 `Option`/`Result` | [链接（pedantic）](https://rust-lang.github.io/rust-clippy/master/#unnecessary_wraps) |
| `clone_on_copy` | 捕获对 `u32`、`bool` 等 `Copy` 类型的意外 `.clone()` | [链接（complexity）](https://rust-lang.github.io/rust-clippy/master/#clone_on_copy) |
| `needless_collect` | 在不需要分配时，阻止对迭代器进行收集和分配 | [链接（nursery）](https://rust-lang.github.io/rust-clippy/master/#needless_collect) |

## 2.4 修复警告，而不是让它们噤声！

**绝不**直接 `#[allow(clippy::lint_something)]`，除非：

* 你**真正理解**为什么会出现该警告，并且有理由说明为何那样更好。
* 你**记录了**它为什么被忽略。
* ❗ 不要用 `allow`，而用 `expect`，当该 lint 不再成立时它会给出警告：`#[expect(clippy::lint_something)]`。

### 示例：

```rust
// Faster matching is preferred over size efficiency
#[expect(clippy::large_enum_variant)]
enum Message {
    Code(u8),
    Content([u8; 1024]),
}
```

> 修复方式是：
> 
> ```rust
> // Faster matching is preferred over size efficiency
> #[expect(clippy::large_enum_variant)]
> enum Message {
>     Code(u8),
>     Content(Box<[u8; 1024]>),
> }
> ```

### 处理误报

有时即使你的代码是正确的，Clippy 也会抱怨，这种情况下有两种解决方案：
1. 尝试重构代码，以消除该警告。
2. **局部**使用 `#[expect(clippy::lint_name)]` 覆盖该 lint，并附上理由注释。
3. 避免全局覆盖，除非是核心 crate 的问题；一个很好的例子是 Bevy Engine，它有一组默认应被允许的 lint。

## 2.5 配置工作区/包的 lint

在 `Cargo.toml` 文件中可以确定使用哪些 lint 以及它们彼此的优先级。如果有 2 个或更多冲突的 lint，将选择优先级更高的那个。包的配置示例：

```toml
[lints.rust]
future-incompatible = "warn"
nonstandard_style = "deny"

[lints.clippy]
all = { level = "deny", priority = 10 }
redundant_clone = { level = "deny", priority = 9 }
manual_while_let_some = { level = "deny", priority = 4 }
pedantic = { level = "warn", priority = 3 }
```

而工作区则使用：

```toml
[workspace.lints.rust]
future-incompatible = "warn"
nonstandard_style = "deny"

[workspace.lints.clippy]
all = { level = "deny", priority = 10 }
redundant_clone = { level = "deny", priority = 9 }
manual_while_let_some = { level = "deny", priority = 4 }
pedantic = { level = "warn", priority = 3 }
```
