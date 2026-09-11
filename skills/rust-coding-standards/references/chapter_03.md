# 第 3 章 - 性能思维

性能工作的**黄金法则**：

> 不要猜，要测量。

Rust 代码通常已经相当快——没有证据就不要“优化”。只在找到瓶颈后才进行优化。

### 好的起步方式
* 在构建时使用 `--release` 标志（可能听起来理所当然，但经常听到有人抱怨他们的 Rust 代码比 X 语言代码慢，99% 的情况是因为他们没有使用 `--release` 标志）。
* `$ cargo clippy -- -D clippy::perf` 会给你关于性能最佳实践的重要提示。
* [`cargo bench`](https://doc.rust-lang.org/cargo/commands/cargo-bench.html) 是一个 cargo 工具，用于创建微基准测试并测试不同的代码方案。写一个测试场景，将你的方案与原始代码做基准对比，如果你的改进大于 5%，可能就是一个不错的性能提升。
* [`cargo flamegraph`](https://github.com/flamegraph-rs/flamegraph) 一个强大的 Rust 代码性能分析器。在 MacOS 上，[samply](https://github.com/mstange/samply) 可能是更好的开发体验选择。

> #### 基准测试的延伸阅读：
> - [How to build a Custom Benchmarking Harness in Rust](https://bencher.dev/learn/benchmarking/rust/custom-harness/)


## 3.1 火焰图（Flamegraph）

火焰图帮助你可视化 CPU 在每项任务上花费了多少时间。

```shell
# Installing flamegraph
cargo install flamegraph

# cargo support provided through the cargo-flamegraph binary!
# defaults to profiling cargo run --release
cargo flamegraph

# by default, `--release` profile is used,
# but you can override this:
cargo flamegraph --dev

# if you'd like to profile a specific binary:
cargo flamegraph --bin=stress2

# Profile unit tests.
# Note that a separating `--` is necessary if `--unit-test` is the last flag.
cargo flamegraph --unit-test -- test::in::package::with::single::crate
cargo flamegraph --unit-test crate_name -- test::in::package::with::multiple:crate

# Profile integration tests.
cargo flamegraph --test test_name

# Run criterion benchmark
# Note that the last --bench is required for `criterion 0.3` to run in benchmark mode, instead of test mode.
cargo flamegraph --bench some_benchmark --features some_features -- --bench

# Run workspace example
cargo flamegraph --example some_example --features some_features
```

> ❗ 始终在启用 `--release` 的情况下运行性能分析，`--dev` 标志不切实际，因为它未启用优化。

结果看起来会是一个火焰图，其中：

* `y 轴`显示**栈深度编号**。观察火焰图时，程序的主函数会靠近底部，被调用的函数会堆叠在其上方，而被它们调用的函数再堆叠在其上。

* `每个方块的宽度`显示**该函数占用 CPU 或被包含在调用栈中的总时间**。如果某个函数的方块比其他函数更宽，说明它每次执行消耗的 CPU 比其他函数多，或者它被调用的次数比其他函数多。

> ❗ **每个方块的颜色**不重要，并且是**随机选择的**。

### 🚨 记住
* 粗栈：CPU 使用重
* 细栈：强度低（廉价）

## 3.2 避免多余的克隆

> 克隆很廉价……**直到它不廉价**

在[优先借用而非克隆](./chapter_01.md#11-borrowing-over-cloning)和[需要重视的重要 Clippy lint](./chapter_02.md#23-important-clippy-lints-to-respect)两节中，我们提到了克隆的影响以及相关的 clippy lint [`redundant_clone`](https://rust-lang.github.io/rust-clippy/master/#redundant_clone)，因此在本节中我们将稍微探讨“何时传递所有权”。

* 🚨 如果你真的需要克隆，把它留到最后一刻。

### 何时传递所有权？

* 只有当你确实需要一个全新的拥有所有权的副本时才 `.clone()`。几个例子：
    * Crate API 设计需要拥有所有权的数据。
    * 重载了 `std::ops`，但仍需要持有旧数据的所有权：
    ```rust
    use std::ops::Add;

    #[derive(Debug, Copy, Clone, PartialEq)]
    struct Point {
        x: i32,
        y: i32,
    }

    impl Add for Point {
        type Output = Self;

        fn add(self, other: Self) -> Self {
            Self {
                x: self.x + other.x,
                y: self.y + other.y,
            }
        }
    }

    assert_eq!(Point { x: 1, y: 0 } + Point { x: 2, y: 3 },
               Point { x: 3, y: 3 });
    ```
    * 需要做比较快照，或者由于 API 的原因你需要数据的多个拥有所有权的实例。
    ```rust
    fn snapshot(a: &MyValue, b:&MyValue) -> MyValueDiff {
        a - b
    }

    impl Sub for MyValue {
        type Output = MyValueDiff;

        fn sub(self, other: Self) -> MyValue {
            ...
        }
    }

    fn main() {
        let mut a = MyValue::default();
        let b = a.clone();

        a.magical_update();
        println!("{:?}", snapshot(&a, &b));
    }
    ```
* 你持有引用计数指针（`Arc, Rc`）。
* 你有小型结构体，大到无法 `Copy`，但代价与 `std::collections` 相当。一个例子是 HTTP 客户端，如 `hyper_util::client::legacy::Client`，克隆它可以让你共享连接池。
* 你有一个链式结构体修改器需要拥有所有权的修改，一些 **builder** 需要拥有所有权的修改，但大多数自定义 builder 可以用 `pub fn with_xyz(&mut self, value: Xyz) -> &mut Self` 实现。
```rust
// Inline `HashMap` insertion extension

fn insert_owned(mut self, key: K, value: V) -> Self {
    self.insert(key, value);
    self
}
```
* 所有权也可以是为业务逻辑/状态建模的好方法。例如：
```rust
let not_validated: String = ...;// some user source
let validated = Validate::try_from(not_validated)?;
// Technically that `try_from` maybe didn't need ownership, but taking it lets us model intent
```

### 何时**不**传递所有权？

* 优先选择接受引用（`fn process(values: &[T])`）的 API 设计，而不是所有权（`fn process(values: Vec<T>)`）。
* 如果你只需要对元素的读访问，优先使用 `.iter` 或切片：
```rust
for item in &some_vec {
    ...
}
```
* 你需要修改由另一个线程拥有的数据，使用 `&mut MyStruct`。

### 使用 `Cow` 处理 `Maybe Owned` 数据

有时你实际上不需要拥有所有权的数据，但从 API 角度看这一点并不明确，因此使用 [`std::borrow::Cow`](https://doc.rust-lang.org/std/borrow/enum.Cow.html) 是高效应对这种情况的一种方式：

```rust
use std::borrow::Cow;

fn hello_greet(name: Cow<'_, str>) {
    println!("Hello {name}");
}

hello_greet(Cow::Borrowed("Julia"));
hello_greet(Cow::Owned("Naomi".to_string()));
```

## 3.3 栈与堆：体积要聪明！

### ✅ 好的实践

* 将小类型（`impl Copy`、`usize`、`bool` 等）保留在**栈上**。
* 避免按值传递巨大的类型（`> 512 字节`）或转移其所有权。优先按引用传递（例如 `&T` 和 `&mut T`）。
* 为递归数据结构进行堆分配：
```rust
enum OctreeNode<T> {
    Node(T),
    Children(Box<[Node<T>; 8]>),
}
```
* 按值返回小类型，实现了 `Copy` 或克隆代价低廉的类型按值返回是高效的（例如 `struct Vector2 {x: f32, y: f32}`）。

### ❗ 需要留心

* 只在基准测试证明有益时才使用 `#[inline]`，Rust 在**没有**提示的情况下已经相当擅长内联。
* 避免大规模的栈分配，把它们装箱。例如 `let buffer: Box<[u8; 65536]> = Box::new(..)` 会首先在栈上分配 `[u8; 65536]` 然后再装箱，非 const 的替代方案是 `let buffer: Box<[u8]> = vec![0; 65536].into_boxed_slice()`。
* 对于大型 `const` 数组，考虑使用 [crate smallvec](https://docs.rs/smallvec/latest/smallvec/)，它的行为像数组，但足够智能，会将大数组分配到堆上。

## 3.4 迭代器与零成本抽象

Rust 迭代器是惰性的，但最终会被编译成非常高效的紧凑循环，只在被消费时调用。链式调用 `.filter()`、`.map()`、`.rev()`、`.skip()`、`.take()`、`.collect()` 通常不会带来额外开销，编译器能够充分推理出如何优化它们。
* 处理集合时优先使用 `iterators` 而非手写 `for` 循环，编译器比手动实现能更好地优化它们。
* 调用 `.iter()` 只会创建指向原始集合的**引用**，这允许你持有同一集合的多个迭代器。

#### ❗ 除非确实需要，否则避免创建中间集合：

* 考虑让 `process` 接受一个 `iterator`。
* ❌ 不好 - 无用的中间集合：
```rust
let doubled: Vec<_> = items.iter().map(|x| x * 2).collect();
process(doubled);
```
* ✅ 好 - 传递迭代器（`fn process(arg: impl Iterator<Item = T>)`）：
```rust
let doubled_iter = items.iter().map(|x| x * 2);
process(doubled_iter);
```
