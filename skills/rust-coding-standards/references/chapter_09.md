# 第 9 章 - 理解指针

许多高级语言隐藏了内存管理，通常以**按值传递**（拷贝数据）或**按引用传递**（对共享数据的引用）的方式，而不必担心分配、堆、栈、所有权和生命周期，一切都交给垃圾回收器或虚拟机处理。以下是几种语言在这个主题上的对比：

### 📌 语言对比 

| 语言 | 值类型 | 引用/指针类型 | 异步模型与类型 | 手动内存管理 |
|------------ |------------------------------------- |----------------------------------------------------------- |---------------------------------------------------------------------------- |------------------------------ |
| Python | 无 | 一切都是引用 | async def, await, Task, coroutines 和 asyncio.Future | ❌ 不允许 |
| Javascript | 原始类型 | 对象 | `async/await`、`Promise`、`setTimeout`。单线程事件循环 | ❌ 不允许 |
| Java | 原始类型 | 对象 | `Future<T>`、线程、Loom（绿色线程） | ❌ 几乎没有且不推荐 |
| Go | 值会被拷贝，除非使用 `&T` | 指针（`*T`、`&T`）、逃逸分析 | goroutines、`channels`、`sync.Mutex`、`context.Context` | ⚠️ 有限 |
| C | 支持原始类型和结构体 | 裸指针 `T*` 和 `*void` | 线程、事件循环（`libuv`、`libevent`） | ✅ 完全 |
| C++ | 原始类型和引用 | 裸指针 `T*` 和智能指针 `shared_ptr` 和 `unique_ptr` | 线程、`std::future`、`std::async`、（自 c++ 20 起 `co_await/coroutines`） | ✅ 大部分 |
| Rust | 原始类型、数组、`impl Copy` | `&T`、`&mut T`、`Box<T>`、`Arc<T>` | `async/await`、`tokio`、`Future`、`JoinHandle`、`Send + Sync` | ✅🔒 安全且明确 |

## 9.1 线程安全

Rust 使用 `Send` 和 `Sync` trait 来跟踪指针：
- `Send` 意味着数据可以跨线程移动。
- `Sync` 意味着数据可以被多个线程引用。

> 只有当指针背后的数据是线程安全的，该指针才是线程安全的。

| 指针类型 | 简要说明 | Send + Sync？ | 主要用途 |
|---------------- |--------------------------------------------------------------------------- |-------------------------------------- |------------ |
| `&T` | 共享引用 | 是 | 共享访问 |
| `&mut T` | 独占可变引用 | 否，不是 Send | 独占修改 |
| `Box<T>` | 堆分配的拥有型指针 | 是，如果 T: Send + Sync | 堆分配 |
| `Rc<T>` | 单线程引用计数指针 | 否，两者都不是 | 多所有者（单线程） |
| `Arc<T>` | 原子引用计数指针 | 是 | 多所有者（多线程） |
| `Cell<T>` | 用于 copy 类型的内部可变性 | 否，不是 Sync | 共享可变，非线程 |
| `RefCell<T>` | 内部可变性（动态借用检查器） | 否，不是 Sync | 共享可变，非线程 |
| `Mutex<T>` | 带独占访问的线程安全内部可变性 | 是 | 共享可变，多线程 |
| `RwLock<T>` | 线程安全的共享只读访问或独占可变访问 | 是 | 共享可变，多线程 |
| `OnceCell<T>` | 单线程一次性初始化容器（内部可变性仅一次） | 否，不是 Sync | 简单的惰性值初始化 |
| `LazyCell<T>` | `OnceCell<T>` 的惰性版本，调用函数闭包来初始化 | 否，不是 Sync | 复杂的惰性值初始化 |
| `OnceLock<T>` | `OnceCell<T>` 的线程安全版本 | 是 | 多线程单次初始化 |
| `LazyLock<T>` | `LazyCell<T>` 的线程安全版本 | 是 | 多线程复杂初始化 |
| `*const T/*mut T` | 裸指针 | 否，用户必须手动确保安全 | 裸内存 / FFI |

## 9.2 何时使用指针：

### `&T` - 共享借用：

这是 Rust 代码库中最常见的类型，它**安全、无修改**，并允许**多个读者**。

```rust
let data: String = String::from_str("this a string").unwrap();

print_len(&data);
print_capacity(&data);
print_bytes(&data);

fn print_len(s: &str) {
    println!("{}", s.len())
}

fn print_capacity(s: &String) {
    println!("{}", s.capacity())
}

fn print_bytes(s: &String) {
    println!("{:?}", s.as_bytes())
}
```
### `&mut T` - 独占借用：

这可能是 Rust 代码库中最常见的*可变*类型，它**安全，但一次只允许一个可变借用**。

```rust
let mut data: String = String::from_str("this a string").unwrap();
mark_update(&mut data);

fn mark_update(s: &mut String) {
    s.push_str("_update");
}
```

### [`Box<T>`](https://doc.rust-lang.org/std/boxed/struct.Box.html) - 堆分配

单一所有者的堆分配数据，非常适合递归类型和大型结构体。

```rust
pub enum MySubBoxedEnum<T> {
    Single(T),
    Double(Box<MySubBoxedEnum<T>>, Box<MySubBoxedEnum<T>>),
    Multi(Vec<T>), // Note that Vec is already a boxed value
}
```

### [`Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html) - 引用计数（单线程）

你需要在单个线程中对数据有多个引用。最常见的例子是链表实现。

### [`Arc<T>`](https://doc.rust-lang.org/std/sync/struct.Arc.html) - 原子引用计数（多线程）

你需要在多个线程中对数据有多个引用。最常见的用例是通过 `Arc<[T]>` 跨线程共享只读 Vec，以及包装 `Mutex` 以便轻松跨线程共享，`Arc<Mutex<T>>`。

### [`RefCell<T>`](https://doc.rust-lang.org/std/cell/struct.RefCell.html) - 运行时检查的内部可变性

当你需要共享访问和修改数据的能力时使用，借用规则在运行时强制执行。**它可能 panic！**。

```rust
use std::cell::RefCell;
let x = RefCell::new(42);
*x.borrow_mut() += 1;

assert_eq!(&*x.borrow(), 42, "Not meaning of life");
```

Panic 示例：
```rust
use std::cell::RefCell;
let x = RefCell::new(42);

let borrow = x.borrow();

let mutable = x.borrow_mut();
```

### [`Cell<T>`](https://doc.rust-lang.org/std/cell/struct.Cell.html) - 仅限 Copy 的内部可变性

某种程度上是 `RefCell` 快速而安全的版本，但它仅限于实现了 `Copy` trait 的类型：

```rust
use std::cell::Cell;

struct SomeStruct {
    regular_field: u8,
    special_field: Cell<u8>,
}

let my_struct = SomeStruct {
    regular_field: 0,
    special_field: Cell::new(1),
};

let new_value = 100;

// ERROR: `my_struct` is immutable
// my_struct.regular_field = new_value;

// WORKS: although `my_struct` is immutable, `special_field` is a `Cell`,
// which can always be mutated with copy values
my_struct.special_field.set(new_value);
assert_eq!(my_struct.special_field.get(), new_value);
```

### [`Mutex<T>`](https://doc.rust-lang.org/std/sync/struct.Mutex.html) - 线程安全可变性

一种独占访问指针，允许一个线程读写其中包含的数据。它通常被包裹在 `Arc` 中，以允许对 Mutex 的共享访问。

### [`RwLock<T>`](https://doc.rust-lang.org/std/sync/struct.RwLock.html) - 线程安全可变性

类似于 `Mutex`，但它允许多个线程读或单个线程写。它通常被包裹在 `Arc` 中，以允许对 RwLock 的共享访问。


### [`*const T/*mut T`](https://doc.rust-lang.org/std/primitive.pointer.html) - 裸指针

本质上是**不安全的**，并且是 FFI 所必需的。Rust 使其用法显式化，以避免意外误用和不情愿的手动内存管理。

```rust
let x = 5;
let ptr = &x as *const i32;
unsafe {
    println!("PTR is {}", *ptr)
}
```

### [`OnceCell`](https://doc.rust-lang.org/std/cell/struct.OnceCell.html) - 单线程单次初始化容器

当你需要在多个数据结构之间共享配置时最有用。

```rust
use std::{cell::OnceCell, rc::Rc};

#[derive(Debug, Default)]
struct MyStruct {
    distance: usize,
    root: Option<Rc<OnceCell<MyStruct>>>, 
}

fn main() {
    let root = MyStruct::default();
    let root_cell = Rc::new(OnceCell::new());
    if let Err(previous) = root_cell.set(root) {
        eprintln!("Previous Root {previous:?}");
    }
    let child_1 = MyStruct{
        distance: 1,
        root: Some(root_cell.clone())
    };

    let child_2 = MyStruct{
        distance: 2,
        root: Some(root_cell.clone())
    };


    println!("Child 1: {child_1:?}");
    println!("Child 2: {child_2:?}");
}
```

### [`LazyCell`](https://doc.rust-lang.org/std/cell/struct.LazyCell.html) - `OnceCell` 的惰性初始化

当初始化数据可以延迟到实际被调用时才进行时有用。

### [`OnceLock`](https://doc.rust-lang.org/std/sync/struct.OnceLock.html) - 线程安全的 `OnceCell`

当你需要一个 `static` 值时有用。

```rust
use std::sync::OnceLock;

static CELL: OnceLock<u32> = OnceLock::new();

// `OnceLock` has not been written to yet.
assert!(CELL.get().is_none());

// Spawn a thread and write to `OnceLock`.
std::thread::spawn(|| {
    let value = CELL.get_or_init(|| 12345);
    assert_eq!(value, &12345);
})
.join()
.unwrap();

// `OnceLock` now contains the value.
assert_eq!(
    CELL.get(),
    Some(&12345),
);
```

### [`LazyLock`](https://doc.rust-lang.org/std/sync/struct.LazyLock.html) - 线程安全的 `LazyCell`

类似于 `OnceLock`，但静态值的初始化稍微复杂一些。

```rust
use std::sync::LazyLock;

static CONFIG: LazyLock<HashMap<&str, T>> = LazyLock::new(|| {
    let data = read_config();
    let mut config: HashMap<&str, T> = data.into();
    config.insert("special_case", T::default());
    config
});

let _ = &*CONFIG;
```

## 参考资料
- [Mara Bos - Rust Atomics and Locks](https://marabos.nl/atomics/)
- [Semicolon video on pointers](https://www.youtube.com/watch?v=Ag_6Q44PBNs)
