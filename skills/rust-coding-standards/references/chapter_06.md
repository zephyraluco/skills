# 第 6 章 - 泛型、动态分发与静态分发

> 能静态则静态，必须动态才动态

Rust 允许你用两种方式处理多态代码：
* **泛型 / 静态分发**：编译期，按每次使用单态化。
* **Trait 对象 / 动态分发**：运行期 vtable，单一实现。

理解这些权衡能让你写出更快、更小、更灵活的代码。

## 6.1 [泛型](https://doc.rust-lang.org/book/ch10-00-generics.html)

每种编程语言都有有效地处理概念重复的工具。在 Rust 中，泛型就是这样的工具：具体类型或其他属性的抽象占位符。我们可以在编译和运行代码时并不知道将填入什么的情况下，表达泛型的行为或它们与其他泛型的关系。

我们使用泛型来为函数签名或结构体等条目创建定义，然后可以将它们用于许多不同的具体数据类型。让我们先看看如何使用泛型定义函数、结构体、枚举和方法。泛型也可用于实现类型状态模式，并将结构体的功能约束到某些期望的类型，更多关于类型状态的内容见[第 7 章](./chapter_07.md)。

[Generics by Examples](https://doc.rust-lang.org/rust-by-example/generics.html)。

### 泛型性能

你可能想知道使用泛型类型参数是否存在运行时开销。好消息是，使用泛型类型不会让你的程序比使用具体类型运行得更慢。Rust 通过在编译期对使用泛型的代码执行单态化来实现这一点。单态化是通过填入编译时使用的具体类型，将泛型代码转换为特定代码的过程。编译器会检查泛型参数的所有出现，并为泛型代码被调用的具体类型生成代码。

## 6.2 静态分发：`impl Trait` 或 `<T: Trait>`

静态分发基本上是泛型的一个受限版本，即带 trait 约束的泛型，在编译期它能够检查你的泛型是否满足所声明的 trait。

### ✅ 最适合：
* 你想通过付出编译期成本来获得**零运行时开销**。
* 你需要**紧凑循环或性能**。
* 你的类型在**编译期已知**。
* 你处理的是**一次性实现**（单态化）。

### 🏎️ 示例：带泛型的高性能函数
```rust
fn specialized_sum<T: MyTrait, U: Iterator<Item = T>>(iter: U) -> T {
    iter.map(|x| x.random_mapping()).sum()
}

// or, equivalent, more modern
fn specialized_sum<T: MyTrait>(iter: impl Iterator<Item = T>) -> T {
    iter.map(|x| x.random_mapping()).sum()
}
```

这会为每次使用编译成**专用机器码**，快速且可内联。

## 6.3 动态分发：`dyn Trait`

通常动态分发与某种指针或引用一起使用，如 `Box<dyn Trait>`、`Arc<dyn Trait>` 或 `&dyn trait`。

### ✅ 最适合：
* 你绝对需要运行时多态。
* 你需要在一个集合中**存储不同的实现**。
* 你想**在稳定接口背后抽象内部细节**。
* 你在编写**插件式架构**。

> ❗ 更接近你在面向对象语言中会得到的东西，并且可能伴随一些昂贵的开销。可以完全避免泛型，让你混合实现相同 trait 的类型。

### 🚚 示例：异构集合

```rust
trait Animal {
    fn greet(&self) -> String;
}

struct Dog;
impl Animal for Dog {
    fn greet(&self) -> String {
        "woof".to_string()
    }
}

struct Cat;
impl Animal for Cat {
    fn greet(&self) -> String {
        "meow".to_string()
    }
}

fn all_animals_greeting(animals: Vec<Box<dyn Animal>>) {
    for animal in animals {
        println!("{}", animal.greet())
    }
}
```

## 6.4 权衡总结

| | 静态分发（impl Trait） | 动态分发（dyn Trait） |
|------------------- |------------------------------ |---------------------------------- |
| 性能 | ✅ 更快，可内联 | ❌ 更慢：vtable 间接寻址 |
| 编译时间 | ❌ 更慢：单态化 | ✅ 更快：共享代码 |
| 二进制体积 | ❌ 更大：按类型生成代码 | ✅ 更小 |
| 灵活性 | ❌ 僵化，一次只能一种类型 | ✅ 可以在集合中混合类型 |
| 在 trait fn() 中使用 | ❌ trait 必须是对象安全的 | ✅ 可用于 trait 对象 |
| 错误 | ✅ 更清晰 | ❌ 类型被抹除，可能使错误令人困惑 |

* 当你控制调用点并想要性能时，优先使用泛型/静态分发。
* 当你需要抽象、插件或混合类型时使用动态分发。🚨 有运行时开销。
* 如果不确定，从泛型开始，为它们加 trait 约束——当灵活性胜过速度时再使用 `Box<dyn Trait>`。

> 在 trait 需要存在于指针之后前，优先静态分发。

## 6.5 动态分发的最佳实践

动态分发 `Ptr<dyn Trait>` 是一个强大的工具，但也有显著的性能权衡。只有当**类型擦除或运行时多态**必不可少时才应该动用它。了解何时需要 Trait 对象很重要：

### ✅ 在以下情况使用动态分发：

* 你需要在集合中使用异构类型：
```rust
fn all_animals_greeting(animals: Vec<Box<dyn Animal>>) {
    for animal in animals {
        println!("{}", animal.greet())
    }
}
```

* 你想要运行时插件或热插拔组件。
* 你想向调用者抽象内部细节（库设计）。


### ❌ 在以下情况避免动态分发：

* 你控制具体类型。
* 你在性能关键路径中编写代码。
* 你可以在保持简洁的同时用其他方式表达相同逻辑，例如泛型。

## 6.6 🚨 Trait 对象的人体工学

* 当你不需要所有权时，优先使用 `&dyn Trait` 而非 `Box<dyn Trait>`。
* 跨线程共享访问使用 `Arc<dyn Trait>`。
* 如果 trait 有返回 `Self` 的方法，不要使用 `dyn Trait`。
* **避免过早装箱**。除非你确定有益或是必需的（递归），否则不要在结构体内部装箱。
```rust
// ✅ Use generics when possible
struct Renderer<B: Backend> {
    backend: B
}

// ❌ Premature Boxing
struct Renderer {
    backend: Box<dyn Backend> // Boxing too early
}
```
* 如果你必须在公共 API 中暴露 `dyn trait`，请在边界处 `Box`，而不是在内部。
* **对象安全（Object Safety）**：你只能从对象安全的 trait 创建 `dyn Traits`：
    * 它**没有泛型方法**。
    * 它不要求 `Self: Sized`。
    * 所有方法签名使用 `&self`、`&mut self` 或 `self`。
    ```rust
    // ✅ Object Safe
    trait Runnable {
        fn run(&self);
    }

    // ❌ Not Object Safe
    trait Factory {
        fn create<T>() -> T; // generic methods are not allowed
    }
    ```
