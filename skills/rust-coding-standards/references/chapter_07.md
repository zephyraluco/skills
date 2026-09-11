# 第 7 章 - 类型状态模式

在编译期对状态建模，通过使非法状态不可表示来防止 bug。它利用 Rust 的泛型和类型系统来创建只有满足特定条件才能到达的子类型，使某些操作在编译期就是非法的。

> 最近它已成为 Rust 编程的标准设计模式。然而，它并非 Rust 独有，因为它是可实现的，并启发了其他语言效仿 [swift](https://swiftology.io/articles/typestate/) 和 [typescript](https://catchts.com/type-state)。

## 7.1 什么是类型状态模式？

**类型状态模式**是一种设计模式，你将系统的不同**状态**编码为**类型**，而不是运行时的标志或枚举。这允许编译器强制执行状态转换，并在编译期阻止非法操作。它还改善了开发者体验，因为开发者只能根据类型的状态访问特定函数。

> 非法状态变成编译错误，而不是运行时 bug。

## 7.2 为什么要使用它？

* 避免对状态有效性的运行时检查。如果你到达了某些状态，你就可以对你拥有的数据做出某些假设。
* 将状态转换建模为类型转换。这类似于状态机，但在编译期。
* 防止数据误用，例如使用未初始化的对象。
* 提高 API 安全性和正确性。
* phantom data 字段在编译后被移除，因此不会分配额外内存。

## 7.3 简单示例：文件状态

[Github 示例](https://github.com/apollographql/rust-best-practices/tree/main/examples/simple-type-state)
```rust
use std::{io, path::{Path, PathBuf}};

struct FileNotOpened;
struct FileOpened;

#[derive(Debug)]
struct File<State> {
    /// Path to the opened file
    path: PathBuf,
    /// Open `File` handler
    handle: Option<std::fs::File>,
    /// Type state manager
    _state: std::marker::PhantomData<State>
}

impl File<FileNotOpened> {
    /// `open` is the only entry point for this struct.
    /// * When called with a valid path, it will return a `File<FileOpened>` with a valid `handler` and `path`
    /// * `open` serves as an alternative to `new` and `defaults` methods (usable when your struct needs valid data to exist).
    fn open(path: &Path) -> io::Result<File<FileOpened>> {
        // If file is invalid, it will return `std::io::Error`
        let file = std::fs::File::open(path)?;
        Ok(
            File {
                path: path.to_path_buf(),
                // Always valid
                handle: Some(file),
                _state: std::marker::PhantomData::<FileOpened>
            }
        )
    }
}

impl File<FileOpened> {
    /// Reads the content of the `File` as a `String`.
    /// `read` can only be called by state `File<FileOpened>`
    fn read(&mut self) -> io::Result<String> {
        use io::Read;

        let mut content = String::new();
        let Some(handle)= self.handle.as_mut() else {
            unreachable!("Safe to unwrap as state can only be reached when file is open");
        };
        handle.read_to_string(&mut content)?;
        Ok(content)
    }

    /// Returns the valid path buffer.
    fn path(&self) -> &PathBuf {
        &self.path
    }
}
```

## 7.4 真实世界示例

### 带编译期保证的 Builder 模式

> 强制用户在调用 `.build()` 之前**设置必填字段**。

[Github 示例](https://github.com/apollographql/rust-best-practices/tree/main/examples/type-state-builder)

类型状态模式可以有多个关联状态：

```rust
use std::marker::PhantomData;

struct MissingName;
struct NameSet;
struct MissingAge;
struct AgeSet;

#[derive(Debug)]
struct Person {
    name: String,
    age: u8,
    email: Option<String>,
}

struct Builder<NameState, AgeState> {
    name: Option<String>,
    age: u8,
    email: Option<String>,
    _name_marker: PhantomData<NameState>,
    _age_marker: PhantomData<AgeState>,
}

impl Builder<MissingName, MissingAge> {
    fn new() -> Self {
        Builder { name: None, age: 0, _name_marker: PhantomData, _age_marker: PhantomData, email: None }
    }

    fn name(self, name: String) -> Builder<NameSet, MissingAge> {
        Builder { name: Some(name), _name_marker: PhantomData::<NameSet>, age: self.age, _age_marker: PhantomData, email: None }
    }

    fn age(self, age: u8) -> Builder<MissingName, AgeSet> {
        Builder { age, _age_marker: PhantomData::<AgeSet>, name: None, _name_marker: PhantomData, email: None }
    }
}

impl Builder<NameSet, MissingAge> {
    fn age(self, age: u8) -> Builder<NameSet, AgeSet> {
        Builder { age, _age_marker: PhantomData::<AgeSet>, name: self.name, _name_marker: PhantomData::<NameSet>, email: None }
    }
}

impl Builder<MissingName, AgeSet> {
    fn email(self, email: String) -> Self {
        Self { name: self.name , age: self.age , email: Some(email) , _name_marker: self._name_marker , _age_marker: self._age_marker }
    }

    fn name(self, name: String) -> Builder<NameSet, AgeSet> {
        Builder { name: Some(name), _name_marker: PhantomData::<NameSet>, age: self.age, _age_marker: PhantomData::<AgeSet>, email: self.email }
    }
}

impl Builder<NameSet, AgeSet> {
    fn build(self) -> Person {
        Person { 
            name: self.name.unwrap_or_else(|| unreachable!("Name is guarantee to be set")), 
            age: self.age,
            email: self.email,
        }
    }
}
```

虽然比通常的 builder 冗长一些，但这保证了所有必要的字段都存在（注意 email 是仅存在于最终 builder 中的可选字段）。

#### 用法：
```rust
// ✅ Valid cases
let person: Person = Builder::new().name("name".to_string()).age(30).build();
let person: Person = Builder::new().age(30).name("name".to_string()).build();
let person: Person = Builder::new().age(30).name("name".to_string()).email("myself@email.com".to_string()).build();

// ❌ Invalid cases
let person: Person = Builder::new().name("name".to_string()).build(); // ❌ Compile error: Age required to `build`
let person: Person = Builder::new().age(30).build(); // ❌ Compile error: Name required to `build`
let person: Person = Builder::new().age(30).email("myself@email.com".to_string()).build(); // ❌ Compile error: Name required to `build`
let person: Person = Builder::new().build();// ❌ Compile error: Name and Age required to `build`
```

### 网络协议状态机

在连接之前发送消息这类非法转换**根本无法编译**：

```rust
// Mock example
struct Disconnected;
struct Connected;

struct Client<State> {
    stream: Option<std::net::TcpStream>,
    _state: std::marker::PhantomData<State>
}

impl Client<Disconnected> {
    fn connect(addr: &str) -> std::io::Result<Client<Connected>> {
        let stream = std::net::TcpStream::connect(addr)?;
        Ok(Client {
            stream: Some(stream),
            _state: std::marker::PhantomData::<Connected>
        })
    }
}

impl Client<Connected> {
    fn send(&mut self, msg: &str) {
        use std::io::Write;
        let Some(stream) = self.stream.as_mut() else {
            unreachable!("Stream is guarantee to be set");
        };
        stream.write_all(msg.as_bytes())
    }
}
```

## 7.5 优缺点

### ✅ 在以下情况使用类型状态模式：
* 你想要**编译期状态安全**。
* 你需要强制**API 约束**。
* 你在编写严重依赖变体的库/crate。
* 你想用**类型安全的代码路径**取代运行时布尔值或枚举。
* 你需要编译期正确性。

### ❌ 在以下情况避免使用：
* 编写像枚举这样的琐碎状态。
* 不需要类型安全。
* 当它导致过于复杂的泛型时。
* 当需要运行时灵活性时。

### 🚨 缺点与注意事项
* 可能导致更**冗长的解决方案**。
* 可能导致**复杂的类型签名**。
* 可能需要 **unsafe** 来根据不同状态返回**变体输出**。
* 可能需要大量重复（例如重复使用相同的结构体字段）。
* PhantomData 对初学者不直观，可能感觉有点取巧。

> 当该模式能**减少 bug、提高安全性或简化逻辑**时才使用它，而不要为了炫技。
