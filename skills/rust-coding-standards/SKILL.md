---
name: rust-coding-standards
description: >
  Rust 最佳实践指南，在以下情况使用本技能：
  1. 编写新的 Rust 代码或函数
  2. 审查或重构现有 Rust 代码
  3. 在借用与克隆或所有权模式之间做决策
  4. 使用 Result 类型实现错误处理
  5. 优化 Rust 代码性能
  6. 为 Rust 项目编写测试或文档
compatibility: Rust 1.88+, Cargo
metadata:
  author: zeal
allowed-tools: Bash(cargo:*) Bash(rustc:*) Bash(rustfmt:*) Bash(clippy:*) Read Write Edit Glob Grep
---

# Rust 最佳实践

> **黄金法则：不要猜，要测量** 没有证据就不要「优化」——先定位瓶颈，再动手

## 速览

| #   | 主题             | 一句话要点                                                               |
| --- | ---------------- | ------------------------------------------------------------------------ |
| 1   | 编码风格与惯用法 | 默认借用；`String`/`Vec` 参数换成 `&str`/`&[T]`；`let ... else` 提前返回 |
| 2   | Clippy 与 Lint   | 把 Clippy 当编译期错误来对待；用 `expect` 而非 `allow`                   |
| 3   | 性能优化         | `--release` + 火焰图 + 基准测试；克隆留到最后一刻                        |
| 4   | 错误处理         | 库用 `thiserror`，二进制用 `anyhow`，一律用 `?`                          |
| 5   | 自动化测试       | 测试即活文档：命名像句子、一个行为、一个断言                             |
| 6   | 泛型与分发       | 能静态则静态，必须动态才用 `dyn Trait`                                   |
| 7   | 类型状态模式     | 把状态编码进类型，让非法状态无法编译                                     |
| 8   | 注释与文档       | `//` 解释为什么，`///` 解释怎么用；注释会腐烂                            |
| 9   | 理解指针         | 按「所有权 × 线程安全」选指针类型                                        |

---

## 1. 编码风格与惯用法

### 1.1 借用优先于克隆

**默认用 `&T`，把 `.clone()` 当作需要理由的操作。**

参数类型选择：

| 不要                 | 而要   |
| -------------------- | ------ |
| `String`             | `&str` |
| `&Vec<T>` / `Vec<T>` | `&[T]` |
| `&String`            | `&str` |
| `T`（大类型）        | `&T`   |

```rust
fn process(name: &str) {          // ✅ 调用者可以传 &String，也可以传字面量
    println!("Hello {name}");
}

fn process_string(name: String) { /* ... */ }
process(user.clone());            // ❌ 无意义克隆：函数并不需要所有权
```

**允许 `Clone` 的正当场景：**

- 需要边改边保留原件（快照、差分比较）
- 持有 `Rc` / `Arc` 这类引用计数指针
- 数据要跨线程共享（通常是 `Arc`）
- 缓存结果，或底层 API 明确要求拥有所有权的数据
- 避免为非性能关键路径做大规模重构

**应避免：** 循环内逐元素 `clone`（改用 `.cloned()` / `.copied()`）、克隆 `Vec`/`HashMap` 等大结构、因为 API 设计糟糕而克隆（应调整生命周期）、在函数里克隆引用参数却本可以让调用者传所有权

### 1.2 Copy 与按值传递

类型满足以下条件时适合派生 `Copy`：所有字段均为 `Copy`、结构体**很小**（≤ 2~3 个字，即 24 字节）、是「纯数据对象」不含堆分配

```rust
#[derive(Debug, Copy, Clone)]
struct Point { x: f32, y: f32, z: f32 }     // ✅ 12 字节

#[derive(Debug, Clone)]
struct BadIdea { age: i32, name: String }   // ❌ 含 String，不能 Copy
```

要点：

- 枚举的大小取决于**最大变体**；载荷全是 `Copy` 的「标签型」枚举适合 `Copy`
- 数组是**栈分配**的，即使元素是 `Copy`，按值传递大数组也容易栈溢出
- 小 `Copy` 类型按值返回/传参通常比引用更快（省一次间接寻址）

### 1.3 处理 `Option<T>` 与 `Result<T, E>`

| 场景                                                               | 写法                                   |
| ------------------------------------------------------------------ | -------------------------------------- |
| 需要访问内部值、做多路模式匹配、做类型转换                         | `match`                                |
| 失败即发散（`return` / `break` / `continue`），且不需要 `Err` 的值 | `let Ok(x) = expr else { ... };`       |
| `else` 分支需要额外计算，或需要额外的 `Some` 处理                  | `if let ... else { ... }`              |
| 只想把错误往上抛                                                   | `?`                                    |
| `Result` ↔ `Option` 转换                                           | `.ok()` / `.ok_or()` / `.ok_or_else()` |

```rust
// ✅ 扁平化的提前返回
let Ok(json) = serde_json::from_str(&input) else {
    return Err(MyError::InvalidJson);
};

// ✅ 循环内跳过
for x in self {
    let Some(x) = x else { continue };
}
```

**反模式：** 用 `match` 手工做 `Ok→Some` / `Err→None` 转换；`else` 分支只返回默认值却写成 `if let`（应当 `let ... else`）；测试之外使用 `unwrap` / `expect`

### 1.4 避免提前分配

`ok_or` / `map_or` / `unwrap_or` 会**无条件**求值实参，其中构造字符串或集合就会白白分配。需要分配时改用惰性变体：

| 立即求值 ❌             | 惰性求值 ✅                                        |
| ----------------------- | -------------------------------------------------- |
| `ok_or(err)`            | `ok_or_else(\|\| err)`                             |
| `map_or(default, f)`    | `map_or_else(\|e\| ..., f)`                        |
| `unwrap_or(Vec::new())` | `unwrap_or_default()` / `unwrap_or_else(Vec::new)` |

记录日志并转换错误时，用链式的 `inspect_err` + `map_err`：

```rust
x.inspect_err(|err| tracing::error!("function_name: {err}"))
 .map_err(|err| GeneralError::from(("function_name", err)))?;
```

### 1.5 迭代器 vs `for`

| 优先 `for`                                    | 优先迭代器链                               |
| --------------------------------------------- | ------------------------------------------ |
| 需要 `break` / `continue` / `return` 提前退出 | 要对集合或 `Option`/`Result` 做转换        |
| 简单迭代 + 副作用（IO 等）                    | 需要组合多个步骤、无需提前退出             |
| 可读性优先于简洁                              | 需要 `.enumerate` / `.windows` / `.chunks` |
| —                                             | 需要合并多个数据源而避免中间集合           |

```rust
let sum: i32 = (0..=10).filter(|x| x % 2 == 0).map(|x| x + 1).sum();
```

**记住迭代器是惰性的：** 没有消费函数（`.collect()`、`.sum()`、`.for_each()`）之前什么都不会发生；链式调用会在编译期融合成单个循环

**避免：** 不换行、不缩进的长链式调用；为了马上丢弃而 `collect`；对 `Copy` 元素无谓用 `into_iter`（用 `iter` + `.copied()`）；用 `.fold` 求和（用 `.sum`，编译器能专门优化）

### 1.6 `use` 声明顺序

`std`/`core`/`alloc` → 外部 crate → 工作区 crate → `super::` → `crate::`

```toml
# rustfmt.toml（reorder_imports 等截至 Rust 1.88 仍需 nightly：cargo +nightly fmt）
reorder_imports = true
imports_granularity = "Crate"
group_imports = "StdExternalCrate"
```

---

## 2. Clippy 与 Lint 纪律

编译器抓语法与类型错误，Clippy 抓性能陷阱、风格问题、冗余代码、潜在 bug 和非地道写法

```shell
# 日常 / CI 必跑
cargo clippy --all-targets --all-features --locked -- -D warnings

# 更严格（可选）
cargo clippy -- -W clippy::pedantic -W clippy::nursery
```

- `--all-targets`：覆盖 lib、测试、基准、示例
- `--all-features`：覆盖所有特性组合
- `--locked`：要求 `Cargo.lock` 最新（用 `cargo update` 修复）
- `-D warnings`：把警告升级为错误
- 把这条命令固化进 Makefile / Justfile / xtask / CI

### 需要重视的 lint

| Lint                                   | 含义                                |
| -------------------------------------- | ----------------------------------- |
| `redundant_clone`                      | 多余的 `clone`，有性能影响          |
| `needless_borrow`                      | 多余的 `&`                          |
| `map_unwrap_or` / `unnecessary_map_or` | 可简化的 `Option`/`Result` 嵌套处理 |
| `manual_ok_or`                         | 应改用 `.ok_or_else`                |
| `large_enum_variant`                   | 变体过大，考虑 `Box`                |
| `unnecessary_wraps`                    | 函数总是 `Some`/`Ok`，不需要包装    |
| `clone_on_copy`                        | 对 `Copy` 类型误用 `.clone()`       |
| `needless_collect`                     | 不需要分配时却 `collect`            |

### 修复大于屏蔽

```rust
// Faster matching is preferred over size efficiency
#[expect(clippy::large_enum_variant)]
enum Message { Code(u8), Content([u8; 1024]) }
```

- **绝不**随手 `#[allow(...)]`。确需豁免时，必须真正理解原因并写清理由
- 使用 `#[expect(...)]` 而非 `#[allow(...)]`：当 lint 不再触发时它会提醒你删掉过期豁免
- 对误报：先尝试重构；其次**局部** `#[expect]` + 理由；避免全局覆盖（除非是核心 crate 的既定策略，如 Bevy）

### 在 `Cargo.toml` 中统一配置

```toml
[lints.clippy]                                   # 工作区用 [workspace.lints.clippy]
all = { level = "deny", priority = 10 }
redundant_clone = { level = "deny", priority = 9 }
manual_while_let_some = { level = "deny", priority = 4 }
pedantic = { level = "warn", priority = 3 }
```

冲突时按 `priority` 高者生效

---

## 3. 性能优化

> 不要猜，要测量。Rust 通常已经足够快，没有证据不要「优化」

### 3.1 常用工具

| 工具                                 | 用途                                   |
| ------------------------------------ | -------------------------------------- |
| `cargo build --release`              | 99% 的「Rust 比 X 慢」都是忘了加它     |
| `cargo clippy -- -D clippy::perf`    | 性能相关 lint                          |
| `cargo bench`                        | 微基准，对比方案；提升 > 5% 才算有意义 |
| `cargo flamegraph` / `samply`(macOS) | 采样火焰图                             |

```shell
cargo install flamegraph
cargo flamegraph                     # 默认 profile `cargo run --release`
cargo flamegraph --bin=stress2       # 指定二进制
cargo flamegraph --unit-test -- test::in::pkg -- ...  # 单元测试
cargo flamegraph --bench some_bench --features f -- --bench
```

**读火焰图：** y 轴是栈深度（`main` 在底部），方块**宽度**是该函数占用的 CPU/在栈中的总时间；颜色无意义、随机分配。粗栈 = 重负载，细栈 = 廉价

### 3.2 避免多余的克隆

**应当传递所有权的情况：**

- crate 的 API 明确需要 owned 数据
- 实现了 `std::ops` 的运算符，需要就地取走 `self`
- 需要快照/差分（如 `a.magical_update()` 前克隆一份）
- 持有 `Arc` / `Rc`
- 小型但无法 `Copy` 的重量级共享句柄（如 HTTP client，克隆即共享连接池）
- 需要 owned 语义的 builder 收尾方法（大多数 builder 仍可用 `&mut self -> &mut Self`）
- 建模业务状态：`Validate::try_from(raw)?` 取走所有权是为了表达意图

**不应传递所有权时：** API 接受 `&[T]` 而非 `Vec<T>`；只读遍历用 `&vec`；跨线程修改用 `&mut T`

**`Cow` 表达「可能拥有」：**

```rust
use std::borrow::Cow;

fn hello_greet(name: Cow<'_, str>) { println!("Hello {name}"); }

hello_greet(Cow::Borrowed("Julia"));
hello_greet(Cow::Owned("Naomi".to_string()));
```

### 3.3 栈与堆

- 小类型（`Copy`、`usize`、`bool`）留在栈上；**大于 ~512 字节**的类型按引用传递
- 递归结构必须堆分配：`Children(Box<[Node<T>; 8]>)`
- 不要做超大栈分配：`Box::new([u8; 65536])` 会先在栈上构造再装箱；非 const 场景用 `vec![0u8; 65536].into_boxed_slice()`，或使用 `smallvec`
- `#[inline]` 只在基准证明有益时才加——Rust 本身已很擅长内联

### 3.4 零成本抽象

`.filter().map().rev().skip().take()` 之类的链在编译后与手写循环等价，且通常更易被优化

```rust
// ❌ 无用的中间集合
let doubled: Vec<_> = items.iter().map(|x| x * 2).collect();
process(doubled);

// ✅ 传递迭代器：fn process(arg: impl Iterator<Item = T>)
process(items.iter().map(|x| x * 2));
```

`.iter()` 只创建指向原集合的引用，因此可以同时存在多个迭代器

---

## 4. 错误处理

### 4.1 优先 `Result`，避免 panic

```rust
fn divide(x: f64, y: f64) -> Result<f64, DivisionError> {
    if y == 0.0 { Err(DivisionError::DividedByZero) } else { Ok(x / y) }
}
```

`panic!` 只用于不可恢复的场景：测试、断言、逻辑 bug。三个语义化替代宏：

- `todo!` —— 明确标注「此处待实现」
- `unreachable!` —— 已推理出该分支不可能发生
- `unimplemented!` —— 已知未实现，附上原因

### 4.2 生产中避免 `unwrap` / `expect`

`expect` 优于 `unwrap`（可带上下文），但两者只应用于：测试与测试辅助、可证明不可能失败处、确实没有更好选择时

替代方案：

| 需求                   | 写法                                                 |
| ---------------------- | ---------------------------------------------------- |
| 失败即返回             | `let Ok(v) = expr else { return Err(..) };`          |
| 失败需恢复             | `if let Ok(v) = expr { .. } else { .. }`             |
| 可能返回 `None` 的函数 | 改成返回 `Result<T, E>`                              |
| 需要兜底值             | `unwrap_or` / `unwrap_or_else` / `unwrap_or_default` |

### 4.3 `thiserror`：crate 级错误

```rust
#[derive(Debug, thiserror::Error)]
pub enum MyError {
    #[error("Network Timeout")]
    Timeout,
    #[error("Invalid data: {0}")]
    InvalidData(String),
    #[error(transparent)]
    Serialization(#[from] serde_json::Error),
    #[error("Invalid request. Header: {headers}, Metadata: {metadata}")]
    InvalidRequest { headers: Headers, metadata: Metadata },
}
```

**分层错误用 `#[from]` 包装**，不要把所有变体摊平：

```rust
#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("Database handler error: {0}")]
    Db(#[from] DbError),
    #[error("External services error: {0}")]
    ExternalServices(#[from] ExternalHttpError),
}
```

### 4.4 `anyhow` 只留给二进制程序

```rust
use anyhow::{Context, Result, anyhow};

fn main() -> Result<()> {
    let content = std::fs::read_to_string("config.json")
        .context("Failed to read config file")?;
    Config::from_str(&content).map_err(|err| anyhow!("Config parsing error: {err}"))
}
```

- 库中**不要**用 `anyhow::Result`：它会抹掉调用者需要的错误类型
- 二进制程序的 `context`/`anyhow!` 字符串比 `thiserror` 消息更难维护，保持简短
- 测试辅助函数里用 `anyhow` 基本没有副作用

### 4.5 用 `?` 冒泡

```rust
fn handle_request(req: &Request) -> Result<ValidatedRequest, MyError> {
    validate_headers(req)?;
    validate_body_format(req)?;
    validate_credentials(req)?;
    let body = Body::try_from(req)?;
    Ok(ValidatedRequest::try_from((req, body))?)
}
```

需要恢复逻辑时用 `or_else` / `map_err` / `if let Ok(..) else`；只需检查或记录错误时用 `inspect_err`

### 4.6 其他要点

- **测试要覆盖错误分支。** 错误类型常未实现 `PartialEq`，可用 `err.to_string()` 断言消息
- **单一错误形态用 struct**：模块只可能有一种错误时，`#[derive(thiserror::Error)] struct HttpError { .. }` 比 enum 更直接
- **异步错误**需在 `.await` 边界与任务中满足 `Send + Sync + 'static`：

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync>> { /* ... */ }
```

- 除非必要，库中避免 `Box<dyn std::error::Error>`

---

## 5. 自动化测试

> 测试不只是正确性检查，它是别人了解你代码的第一站

### 5.1 命名与组织

测试名应读起来像句子，包含三要素：**被测单元** + **期望行为** + **触发状态**

```rust
#[cfg(test)]
mod process {
    #[test]
    fn should_return_blob_when_larger_than_b() { /* ... */ }
}
// cargo test 输出：process::should_return_blob_when_larger_than_b
```

- ❌ `test_add_happy_path` ✅ `returns_error_xyz_when_b_is_negative`
- 用 `mod` 分组：IDE 可一键跑整个模块，输出里也带模块前缀
- **一个测试只测一个行为**；多个断言拆成多个测试，样板代码用共享 setup 或 `rstest`：

```rust
#[rstest]
#[case::single("a")]
#[case::first_letter("ab")]
fn the_function_accepts_all_strings_with_a(#[case] input: &str) {
    assert!(the_function(input).is_ok());
}
```

- 每个测试尽量只有**一个断言**——否则失败时要多轮迭代才能定位
- 断言消息要带上实际状态与差异：`assert_eq!(result, expected, "differs: {}", result.diff(expected))`；只关心「是否匹配变体」时用 `matches!`

### 5.2 三类测试

| 类型     | 位置                          | 能访问             | 侧重                     |
| -------- | ----------------------------- | ------------------ | ------------------------ |
| 单元测试 | 被测模块内 `#[cfg(test)] mod` | 私有、`pub(crate)` | 实现细节、边界与错误     |
| 集成测试 | `tests/` 目录                 | 仅公共 API         | 组件协作、正常路径       |
| 文档测试 | `///` 代码块                  | 公共 API           | 用法示例，兼作正确性检查 |

- 尚未完成的测试用 `#[ignore = "原因"]`；预期 panic 用 `#[should_panic]`
- 二进制项目把可执行部分与逻辑拆到 `src/main.rs` / `src/lib.rs`，逻辑才可测
- 集成测试允许外部副作用，可借助 `testcontainers`

### 5.3 文档测试注意事项

- 随 `cargo test` 运行，但**不**随 `cargo nextest run` 运行；用 nextest 时需单独跑 `cargo test --doc`
- 用行首 `#` 隐藏样板代码（`use`、`assert_eq!` 的期望值）
- 常用属性：`no_run`（编译不执行，适合有副作用的示例）、`should_panic`、`compile_fail`（演示错误用法）、`text`（仅作代码块展示）

### 5.4 快照测试（`cargo insta`）

当正确性是**视觉或结构化**的时候，快照比断言更能表达意图（生成代码、序列化数据、渲染 HTML、CLI 输出）

```toml
insta = { version = "1.42.2", features = ["yaml"] }
```

```rust
#[test]
fn test_split_words() {
    insta::assert_yaml_snapshot!(split_words("hello from the other side"));
}
```

流程：`cargo insta test` → `cargo insta review`

**最佳实践：**

- 给快照命名：`assert_snapshot!("app_config/http", whole_app_config.http)`
- **保持小而聚焦**，绝不对巨大对象整体做快照
- 简单类型（原生类型、扁平枚举）用 `assert_eq!`，不要用快照
- 易变字段（时间戳、UUID、随机值）用 [redactions](https://insta.rs/docs/redactions/)
- 快照文件（`snapshots/`）提交进 git，接受变更前仔细审阅
- 不适合：稳定纯数值逻辑、关键路径、未脱敏的不稳定输出、外部资源（用 mock/stub）

---

## 6. 泛型与分发

> 能静态则静态，必须动态才动态

| 维度       | 静态（`impl Trait` / `<T: Trait>`） | 动态（`dyn Trait`）   |
| ---------- | ----------------------------------- | --------------------- |
| 性能       | ✅ 更快，可内联                     | ❌ vtable 间接寻址    |
| 编译时间   | ❌ 单态化，较慢                     | ✅ 共享代码，较快     |
| 二进制体积 | ❌ 按类型生成，更大                 | ✅ 更小               |
| 灵活性     | ❌ 一次只能一种类型                 | ✅ 集合中混合多种实现 |
| 错误信息   | ✅ 清晰                             | ❌ 类型被擦除，较难懂 |

```rust
// 静态分发：为每个具体类型生成专用代码
fn specialized_sum<T: MyTrait>(iter: impl Iterator<Item = T>) -> T {
    iter.map(|x| x.random_mapping()).sum()
}

// 动态分发：异构集合
fn all_animals_greeting(animals: Vec<Box<dyn Animal>>) { /* ... */ }
```

**用 `dyn` 的正当理由：** 异构集合、运行时插件/热插拔、在稳定接口背后擦除内部类型

**避免 `dyn`：** 具体类型在你掌控之中、处于性能关键路径、能用泛型同样简洁地表达

### Trait 对象的人体工学

- 不需要所有权时用 `&dyn Trait` 而不是 `Box<dyn Trait>`；跨线程共享用 `Arc<dyn Trait>`
- trait 若有返回 `Self` 的方法，就不能做 trait 对象
- **避免过早装箱**：结构体内部优先用泛型；公共 API 必须暴露 `dyn` 时，在边界装箱

```rust
struct Renderer<B: Backend> { backend: B }          // ✅
struct Renderer { backend: Box<dyn Backend> }       // ❌ 过早装箱
```

**对象安全**要求：没有泛型方法、不要求 `Self: Sized`、方法签名为 `&self` / `&mut self` / `self`

---

## 7. 类型状态模式

把系统的不同**状态编码成类型**（配合 `PhantomData`），由编译器强制状态转换，让非法操作变成编译错误

```rust
use std::marker::PhantomData;

struct Disconnected;
struct Connected;

struct Client<State> {
    stream: Option<std::net::TcpStream>,
    _state: PhantomData<State>,
}

impl Client<Disconnected> {
    fn connect(addr: &str) -> std::io::Result<Client<Connected>> {
        let stream = std::net::TcpStream::connect(addr)?;
        Ok(Client { stream: Some(stream), _state: PhantomData::<Connected> })
    }
}

impl Client<Connected> {
    fn send(&mut self, msg: &str) { /* 只有 Connected 才能调用 */ }
}
```

- `PhantomData` 在编译后被擦除，**零运行时开销**
- 「未打开的文件」只能通过 `open()` 进入「已打开」状态；`read()` / `send()` 只在有效状态上存在
- **Builder 场景**：用多组标记（`MissingName`/`NameSet`、`MissingAge`/`AgeSet`），只在 `Builder<NameSet, AgeSet>` 上实现 `build()`，从而在编译期强制必填字段。略冗长，但比运行时校验更可靠

**适合：** 编译期状态安全、强制 API 约束、替代运行时 bool/enum 状态、依赖变体的库

**不适合 / 代价：** 琐碎状态、需要运行时灵活性、类型签名膨胀、为返回不同状态变体可能需要 `unsafe`、字段重复、`PhantomData` 对新手不直观

> 只在能**减少 bug、提高安全性或简化逻辑**时使用，不要为了炫技

---

## 8. 注释与文档

> 清晰的代码胜过清晰的注释；当「为什么」不明显时，就直白注释或链接到能读到更多上下文的地方

### 8.1 `//` 与 `///` 的分工

| 用途                                   | `// 注释`   | `///` / `//!` 文档            |
| -------------------------------------- | ----------- | ----------------------------- |
| 解释**为什么**（陷阱、优化、平台差异） | ✅          | ❌                            |
| 描述 API（用法、参数、错误、panic）    | ❌          | ✅                            |
| 可维护性                               | 🚨 容易过时 | ✅ 与代码绑定、可编译、可测试 |
| 可见性                                 | 仅源码      | `cargo doc` 导出给用户        |

### 8.2 何时写注释

只在这些东西代码本身无法清晰表达时：

- 安全保证 —— 用 `// SAFETY: ...` 开头（或尽量用 `#[cfg]`/类型表达）
- 变通方案或性能优化 —— `// PERF: ...`
- 遗留代码、平台特定行为
- 指向设计文档 / ADR 的链接 —— `// CONTEXT: ... [ADR-12](link)`
- 不明显的假设与坑

**不要写：** 复述显而易见的事（`// i += 1`）、会随手过时的说明、没有 issue 追踪的 `TODO`、本可以用更好的命名或更小的函数取代的说明

### 8.3 注释不是「活文档」

注释会**腐烂**（没人编译它）、会**误导**（读者默认它是对的）、会**过时**、会成为**噪音**。发现注释时**结合上下文重新验证**：不正确就修正或删除。需要长期存在的知识应该进入 ADR、设计文档、类型/文档注释/示例，或一条测试

**用代码取代注释：** 把带注释的代码块拆成命名良好的小函数：

```rust
fn save_auth_user(&self) -> Result<PathBuf, MyError> {
    if self.is_authenticated() {
        let path = self.path();
        std::fs::write(path, serde_json::to_string(self)?)?;
        Ok(path)
    } else {
        Err(MyError::UserNotAuthenticated)
    }
}
```

**`TODO` 要变成 issue：** `// TODO(#42): bugfix 后移除该 workaround`

### 8.4 文档注释

- `///`：条目级（函数、结构体、枚举、trait、常量）。写清**做什么**与**怎么用**，并补 `# Examples`、`# Errors`、`# Panics`、`# Safety` 小节
- `//!`：模块或 crate 级，放在 `lib.rs` / `mod.rs` 顶部，解释**这个模块/ crate 解决什么问题**

````rust
/// 从磁盘加载 [`User`] 配置
///
/// # Errors
/// - 文件缺失返回 [`MyError::FileNotFound`]
/// - JSON 非法返回 [`MyError::InvalidJson`]
///
/// # Examples
/// ```
/// # use my_crate::*;
/// let user = load_user(Path::new("u.json"))?;
/// ```
````

**文档 lint**（建议在 crate 根开启）：

| Lint                     | 作用                                   |
| ------------------------ | -------------------------------------- |
| `missing_docs`           | 警告公共条目缺文档                     |
| `broken_intra_doc_links` | 检测失效的内部链接（重命名时尤其有用） |
| `empty_docs`             | 禁止空文档绕过 `missing_docs`          |
| `missing_panics_doc`     | 要求 `# Panics` 节                     |
| `missing_errors_doc`     | 要求 `# Errors` 节                     |
| `missing_safety_doc`     | 要求 `# Safety` 节                     |

### 8.5 文档覆盖清单

- **crate（`lib.rs`）**：`//!` 说明 crate 做什么、解决什么问题，并给出示例或模块导航
- **模块**：`//!` 说明职责、导出与不变式；重导出条目避免重复文档
- **类型 / trait**：`///` 说明角色、不变式、构造与用法；外部可能匹配的类型考虑 `#[non_exhaustive]`
- **函数 / 方法**：做什么、参数、返回值、边界情况、示例
- **常量**：配置什么、何时使用
- 用 `cargo doc --open` 检查输出；需要强制覆盖时加 `#![deny(missing_docs)]`

---

## 9. 理解指针

### 9.1 `Send` / `Sync`

- `Send`：值可以**跨线程移动**
- `Sync`：值可以**被多个线程引用**（`&T: Send`）

> 只有指针背后的数据是线程安全的，指针本身才是线程安全的

| 指针                          | 说明                       | Send + Sync               | 主要用途                         |
| ----------------------------- | -------------------------- | ------------------------- | -------------------------------- |
| `&T`                          | 共享引用                   | ✅（若 `T: Sync`）        | 共享只读                         |
| `&mut T`                      | 独占可变引用               | ❌ 非 `Send`              | 独占修改                         |
| `Box<T>`                      | 堆上所有权指针             | ✅（若 `T: Send + Sync`） | 堆分配、递归结构                 |
| `Rc<T>`                       | 单线程引用计数             | ❌                        | 单线程多所有者                   |
| `Arc<T>`                      | 原子引用计数               | ✅                        | 多线程多所有者                   |
| `Cell<T>`                     | 仅 `Copy` 类型的内部可变性 | ❌ 非 `Sync`              | 单线程共享可变                   |
| `RefCell<T>`                  | 运行时借用检查的内部可变性 | ❌ 非 `Sync`              | 单线程共享可变（**可能 panic**） |
| `Mutex<T>`                    | 线程安全独占可变性         | ✅                        | 跨线程共享可变                   |
| `RwLock<T>`                   | 多读或单写                 | ✅                        | 读多写少的共享可变               |
| `OnceCell<T>` / `LazyCell<T>` | 单线程一次性 / 惰性初始化  | ❌ 非 `Sync`              | 简单惰性值                       |
| `OnceLock<T>` / `LazyLock<T>` | 上述的线程安全版本         | ✅                        | `static` 全局初始化              |
| `*const T` / `*mut T`         | 裸指针                     | ❌ 手动保证               | FFI、裸内存                      |

### 9.2 如何选择

| 需求                         | 选择                                                                 |
| ---------------------------- | -------------------------------------------------------------------- |
| 只读共享（最常见）           | `&T`，参数用 `&str` / `&[T]`                                         |
| 独占修改                     | `&mut T`                                                             |
| 单一所有者的堆数据、递归类型 | `Box<T>`（`Vec` 本身已含 `Box`）                                     |
| 同一线程多所有者（链表等）   | `Rc<T>` / `Rc<RefCell<T>>`                                           |
| 跨线程共享只读               | `Arc<[T]>`、`Arc<T>`                                                 |
| 跨线程共享可变               | `Arc<Mutex<T>>`、`Arc<RwLock<T>>`                                    |
| 单线程内绕过借用检查         | `Cell<T>`（`Copy`）/ `RefCell<T>`（有 panic 风险）                   |
| 全局或跨结构共享的一次初始化 | `OnceLock` / `LazyLock`（多线程）、`OnceCell` / `LazyCell`（单线程） |
| FFI                          | `*const T` / `*mut T` + `unsafe` + `// SAFETY` 注释                  |

---

## 审查清单

- [ ] 没有无理由的 `.clone()`；参数用 `&str` / `&[T]` / `&T`，小 `Copy` 类型按值传递
- [ ] `Option`/`Result` 用了合适的 `match` / `let ... else` / `?`，而非 `unwrap`
- [ ] 需要分配的 `or` 系列全部用了 `_else` / `_default` 变体
- [ ] `cargo clippy --all-targets --all-features --locked -- -D warnings` 通过，豁免使用 `#[expect]` + 理由
- [ ] 性能结论有基准或火焰图支撑，且是 `--release` 构建
- [ ] 库返回自定义错误类型（`thiserror`），二进制才用 `anyhow`
- [ ] 测试命名可读、一个行为一个测试、断言带上下文，错误分支有覆盖
- [ ] 公共 API 有 `///` 文档与可运行的 `# Examples`，覆盖 `# Errors` / `# Panics` / `# Safety`
- [ ] `TODO` 都关联了 issue
- [ ] 分发方式（泛型 / `dyn`）有明确理由，没有过早装箱
- [ ] 指针类型满足跨线程需求，`Rc` / `RefCell` 没有误入多线程路径
