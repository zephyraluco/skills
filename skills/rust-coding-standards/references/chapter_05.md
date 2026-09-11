# 第 5 章 - 自动化测试

> 测试不仅仅是为了正确性。它们是人们了解你代码如何工作的第一站。

* Rust 中的测试用属性宏 `#[test]` 声明。大多数代码编辑器可以单独或成块地编译并运行该宏下声明的函数。
* 测试可以通过 `#[cfg(test)]` 拥有特殊的编译标志。如果其中包含 `#[test]`，也可在代码编辑器中执行，这是 mock 复杂函数或覆盖 trait 的好方法。

## 5.1 测试作为活文档

在 Rust 中，如同许多其他语言一样，测试往往展示了函数应如何使用。如果一个测试清晰且针对性强，它通常比阅读函数体更有帮助；与其他测试结合时，它们就充当活文档。

### 使用描述性的名称

> 在单元测试名称中我们应该看到：
> * `unit_of_work`：我们正在调用哪个*函数*。将要执行的**动作**。这通常是正在被测试的函数的 `mod` 名称。
```rust
#[cfg(test)] 
mod test { 
    mod function_name { 
        #[test] 
        fn returns_y_when_x() { ... } 
    } 
}
```
> * `expected_behavior`：我们需要验证测试有效的一组**断言**。
> * `state_that_the_test_will_check`：具体测试用例的总体**安排**或设置。

#### ❌ 不要为测试使用通用名称
```rust
#[test]
fn test_add_happy_path() {
    assert_eq!(add(2, 2), 4);
}
```
#### ✅ 使用读起来像句子、描述期望行为的名称
> 或者，如果你的函数测试太多，可以把它们归类到一个 `mod` 中，这样更易于阅读和导航。

```rust
// OPTION 1
#[test]
fn process_should_return_blob_when_larger_than_b() {
    let a = setup_a_to_be_xyz();
    let b = Some(2);
    let expected = MyExpectedStruct { ... };

    let result = process(a, b).unwrap();

    assert_eq!(result, expected);
}

// OPTION 2
mod process {
    #[test]
    fn should_return_blob_when_larger_than_b() {
        let a = setup_a_to_be_xyz();
        let b = Some(2);
        let expected = MyExpectedStruct { ... };

        let result = process(a, b).unwrap();

        assert_eq!(result, expected);
    }
}
```

> 执行 `cargo test` 时，每种选项的测试输出会像这样：
> 选项 1：`process_should_return_blob_when_larger_than_b`。
> 选项 2：`process::should_return_blob_when_larger_than_b`。

### 使用模块来组织

大多数 IDE 可以一次性运行整个模块的测试。
输出中的测试名称也会包含模块名。
结合起来，你就可以用模块名把相关测试分组：

```rust
#[cfg(test)]
mod test { // IDEs will provide a ▶️ button here

    mod process {
        #[test] // IDEs will provide a ▶️ button here
        fn returns_error_xyz_when_b_is_negative() {
            let a = setup_a_to_be_xyz();
            let b = Some(-5);
            let expected = MyError::Xyz;
            
            let result = process(a, b).unwrap_err();
            
            assert_eq!(result, expected);
        }

        #[test] // IDEs will provide a ▶️ button here
        fn returns_invalid_input_error_when_a_and_b_not_present() {
            let a = None;
            let b = None;
            let expected = MyError::InvalidInput;

            let result = process(a, b).unwrap_err();

            assert_eq!(result, expected);
        }
    }
}
```

### 每个函数只测试一个行为

为了让测试保持清晰，它们应描述该单元所做的*一件*事。
这使人们更容易理解测试为何失败。

#### ❌ 不要在同一个测试中测试多件事
```rust
fn test_thing_parser(...) {
    assert!(Thing::parse("abcd").is_ok());
    assert!(Thing::parse("ABCD").is_err());
}
```

#### ✅ 每个测试只测一件事
```rust
#[cfg(test)]
mod test_thing_parser {
    #[test]
    fn lowercase_letters_are_valid() {
        assert!(
            Thing::parse("abcd").is_ok(),
            // Works like `eprintln`, `format` and `println` macros
            "Thing parse error: {:?}", 
            Thing::parse("abcd").unwrap_err()
        );
    }

    #[test]
    fn capital_letters_are_invalid() {
        assert!(Thing::parse("ABCD").is_err());
    }
}
```

> `Ok` 场景应先 `eprintln` 出 `Err` 情况。

### 每个测试使用尽量少、理想情况下只有一个断言

当一个测试有多个断言时，既更难理解预期行为，也往往需要多次迭代才能修复失败的测试，因为你得逐个处理断言。

❌ 不要在一个测试中包含许多断言：

```rust
#[test]
fn test_valid_inputs() {
    assert!(the_function("a").is_ok());
    assert!(the_function("ab").is_ok());
    assert!(the_function("ba").is_ok());
    assert!(the_function("bab").is_ok());
}
```

如果你在测试不同的行为，就写多个测试，每个都有描述性的名称。
为避免样板代码，可以使用共享的 setup 函数，或使用 [rstest](https://crates.io/crates/rstest) 用例*并配上描述性的测试名*：
```rust
#[rstest]
#[case::single("a")]
#[case::first_letter("ab")]
#[case::last_letter("ba")]
#[case::in_the_middle("bab")]
fn the_function_accepts_all_strings_with_a(#[case] input: &str) {
    assert!(the_function(input).is_ok());
}
```

> 使用 `rstest` 时的注意事项
>
> * 无论是 IDE 还是人都更难运行/定位特定测试。
> * 期望值与条件的命名在视觉上颠倒了（期望值在前）。

## 5.2 为你的文档添加测试示例

我们会在后面深入讨论文档，因此本节只简要介绍如何为文档添加测试。Rustdoc 可以把示例转换成可执行的测试，使用 `///` 有一些优势：

* 这些测试会随 `cargo test` 运行，但**不**随 `cargo nextest run` 运行。如果使用 `nextest`，请确保单独运行 `cargo t --doc`。
* 它们既作为文档又作为正确性检查，并且由于编译器会检查它们，因此会随变更保持最新。
* 无需额外的测试样板代码。你可以通过在行前加 `#` 轻松隐藏测试部分。
* ❗ 如果文档测试与其他非公开面向的测试之间存在重复，也没有问题。

```rust
/// Helper function that adds any two numeric values together.
/// This function reasons about which would be the correct type to parse based on the type
/// and the size of the numeric value.
/// 
/// # Examples
/// 
/// ```rust
/// # use crate_name::generic_add;
/// use num::numeric;
/// 
/// # assert_eq!(
/// generic_add(5.2, 4) // => 9.2
/// # , 9.2)
/// 
/// # assert_eq!(
/// generic_add(2, 2.0) // => 4
/// # , 4)
/// ```
```

这段文档代码展示出来会像：
```rust
use num::numeric;

generic_add(5.2, 4) // => 9.2
generic_add(2, 2.0) // => 4
```

## 5.3 单元测试 vs 集成测试 vs 文档测试

一般来说，不去深究*测试金字塔命名*，Rust 有 3 类测试：

### 单元测试

放在与被测单元声明所在的**同一模块**中的测试，这使测试运行器能够访问私有函数和父级的 `use` 声明。它们也可以在需要时使用其他模块的 `pub(crate)` 函数。单元测试可以更专注于**实现和边界情况检查**。

* 它们应尽可能简单，测试单元的一个状态和一个行为。KISS。
* 它们应测试错误和边界情况。
* 同一单元的不同测试可以合并在单个 `#[cfg(test)] mod test_unit_of_work {...}` 下，允许为不同的 `units_of_work` 设置多个子模块。
* 尽量将 API 的外部状态/副作用降到最低，并将这些测试集中在 `mod.rs` 文件中。
* 尚未完全实现的测试可以用 `#[ignore = "optional message"]` 属性忽略。
* 故意 panic 的测试应标注 `#[should_panic]` 属性。

```rust
#[cfg(test)]
mod unit_of_work_tests {
    use super::*;

    #[test]
    fn unit_state_behavior() {
        let expected = ...;
        let result = ...;
        assert_eq!(result, expected, "Failed because {}", result - expected);
    }
}
```

### 集成测试

放在 `tests/` 目录下的测试，它们完全处于你的库之外，使用与其他任何代码相同的代码，无法访问私有和 crate 级别的函数，这意味着它们**只能测试**你**公共 API** 上的函数。

> 它们的目的是测试代码的许多部分能否正确地协同工作，那些单独工作正确的代码单元在集成时可能出问题。

* 测试正常路径和常见用例。
* 允许外部状态和副作用，[testcontainers](https://rust.testcontainers.org/) 可能会有帮助。
* 如果要测试二进制程序，尽量把**可执行程序**和**函数**分别拆分到 `src/main.rs` 和 `src/lib.rs`。

```
├── Cargo.lock 
├── Cargo.toml 
├── src 
│   └── lib.rs 
└── tests 
    ├── mod.rs 
    ├── common 
    │   └── mod.rs 
    └── integration_test.rs
```

### 文档测试

如 [5.2](#52-add-test-examples-to-your-docs) 节所述，文档测试应包含正常路径、一般公共 API 用法，以及能改善文档的更强大的属性，比如为代码块自定义 CSS。

### 属性：

* `ignore`：告诉 rust 忽略该代码，通常不推荐；如果你只想要一段代码格式的文本，请使用 `text`。
* `should_panic`：告诉 rust 编译器该示例块会 panic。
* `no_run`：编译但不执行代码，类似于 `cargo check`。在处理文档的副作用时非常有用。
* `compile_fail`：测试 rustdoc 该块应导致编译失败，当你想演示错误的用法时很重要。

## 5.4 如何使用 `assert!`

Rust 自带 2 个用于断言的宏：
* `assert!` 用于断言布尔值，如 `assert!(value.is_ok(), "'value' is not Ok: {value:?}")`
* `assert_eq!` 用于检查两个不同值是否相等，`assert_eq!(result, expected, "'result' differs from 'expected': {}", result.diff(expected))`。

### 🚨 `assert!` 提醒
* Rust 断言支持格式化字符串，如前面的例子，这些字符串会在失败时打印出来，因此添加实际状态及其与期望的差异是一个好习惯。
* 如果你不关心精确的模式匹配值，使用 `matches!` 结合 `assert!` 可能是一个不错的替代方案。
```rust
assert!(matches!(error, MyError::BadInput(_), "Expected `BadInput`, found {error}"));
```
* 明智地使用 `#[should_panic]`。它只应在 panic 是期望行为时使用，优先返回结果而不是 panic。
* 还有一些其他工具可以提升你的测试体验，比如：
    * [`rstest`](https://crates.io/crates/rstest)：基于 fixture 的测试框架，带过程宏。
    * [`pretty_assertions`](https://crates.io/crates/pretty_assertions)：覆盖 `assert_eq` 和 `assert_ne`，并在它们之间创建彩色的差异对比。

## 5.5 使用 `cargo insta` 进行快照测试

> 当正确性是视觉或结构化的时候，快照比断言更能说明问题。

1. 添加到你的依赖：
```toml
insta = { version = "1.42.2", features = ["yaml"] }
```
> 对于大多数真实世界应用，建议使用可序列化值的 YAML 快照。这是因为它们在版本控制和差异查看器中看起来最好，并支持 redaction。要使用它，请启用 insta 的 yaml feature。

2. 为了更好的审阅体验，安装 CLI：`cargo install cargo-insta`。

3. 编写一个简单的测试：
```rust
fn split_words(s: &str) -> Vec<&str> {
    s.split_whitespace().collect()
}

#[test]
fn test_split_words() {
    let words = split_words("hello from the other side");
    insta::assert_yaml_snapshot!(words);
}
```

4. 运行 `cargo insta test` 执行，运行 `cargo insta review` 审阅冲突。

要了解更多关于 `cargo insta` 的信息，请查看其[文档](https://insta.rs/docs/quickstart/)，因为这是一个非常完整且文档完善的工具。

### 什么是快照测试？

快照测试将你的输出（文本、Json、HTML、YAML 等）与保存的“黄金”版本进行比较。在后续运行中，除非人工批准，否则如果输出发生变化测试就会失败。它非常适合：
* 生成代码。
* 序列化复杂数据。
* 渲染的 HTML。
* CLI 输出。

#### ❌ 什么不适合用快照测试
* 非常稳定、纯数字或小型结构化数据相关的逻辑（优先 `assert_eq!`）。
* 关键路径逻辑（优先精确的单元测试）。
* 不稳定的测试、随机生成的输出，除非经过 redaction。
* 外部资源的快照，使用 mock 和 stub。

## 5.6 ✅ 快照最佳实践

* 命名快照，这会给快照文件有意义的名字，例如 `snapshots/this_is_a_named_snapshot.snap`
```rust
assert_snapshot!("this_is_a_named_snapshot", output);
```

* 保持快照小而清晰。 
```rust
// ✅ Best case:
assert_snapshot!("app_config/http", whole_app_config.http);

// ❌ Worst case:
assert_snapshot!("app_config", whole_app_config); // Huge object
```

> #### 🚨 避免对巨大对象做快照 
> 巨大对象难以审阅和推理。

* 避免对简单类型（原生类型、扁平枚举、小型结构体）做快照：
```rust
// ✅ Better:
assert_eq!(meaning_of_life, 42);

// ❌ OVERKILL:
assert_snapshot!("the_meaning_of_life", meaning_of_life); // meaning_of_life == 42
```

* 对不稳定的字段（随机生成、时间戳、uuid 等）使用 [redactions](https://insta.rs/docs/redactions/)：
```rust
use insta::assert_json_snapshot;

#[test]
fn endpoint_get_user_data() {
    let data = http::client.get_user_data();
    assert_json_snapshot!(
        "endpoints/subroute/get_user_data",
        data,
        ".created_at" => "[timestamp]",
        ".id" => "[uuid]"
    );
}
```
* 将快照提交到 git。它们会存储在测试旁边的 `snapshots/` 目录中。
* 在接受之前仔细审阅变更。
