# **`static` 关键字**

在 Rust 中，`static` 用来:

* 定义 **全局变量**
* 声明生命周期 `'static`，整个程序运行期间都存在

## 全局变量

语法格式：

```rust
static NAME: 类型 = 值;
```

* 必须指定类型（不能靠类型推导）
* 必须赋初始值（常量表达式，编译期就能确定）

例子：

```rust
static PI: f64 = 3.14159;

fn main() {
    println!("PI = {}", PI);
}
```

运行结果：

```
PI = 3.14159
```


## `static` 与 `const` 的区别

Rust 中有两个类似的关键字：`const` 和 `static`，经常会混淆。

| 特性   | `const`         | `static`              |
| ---- | --------------- | --------------------- |
| 生命周期 | 无，编译时直接内联到代码    | `'static`，在全局只存一份     |
| 存储位置 | 不一定固定，可能被优化进指令里 | 存放在静态存储区（类似全局变量）      |
| 是否可变 | 永远不可变           | 可用 `static mut` 声明为可变 |
| 使用场景 | 常量表达式（编译期已知的值）  | 全局共享变量（可能运行期可变）       |

例子：

```rust
const MAX_POINTS: u32 = 100_000;  // 编译时常量
static MAX_USERS: u32 = 10;       // 程序中唯一的一份
```



## 可变 `static` 变量

默认 `static` 是 **不可变的**。如果需要全局可变变量，必须写成 `static mut`：

```rust
static mut COUNTER: i32 = 0;

fn main() {
    unsafe {
        COUNTER += 1;
        println!("COUNTER = {}", COUNTER);
    }
}
```

**注意：**

* 修改 `static mut` 必须用 `unsafe`，因为它不是线程安全的。
* 在多线程环境下可能会数据竞争（data race）。


## 线程安全的全局变量

如果你想在多线程下安全地使用 `static`，通常要结合并发原语（比如 `Mutex`、`RwLock`、`Atomic`）：

```rust
use std::sync::Mutex;

static COUNTER: Mutex<i32> = Mutex::new(0);

fn main() {
    {
        let mut num = COUNTER.lock().unwrap();
        *num += 1;
    }
    println!("COUNTER = {:?}", COUNTER.lock().unwrap());
}
```

这样就保证了线程安全。


## `static` 的生命周期 `'static`

* 所有 `static` 变量都拥有 **`'static` 生命周期**，也就是说它们活得跟整个程序一样久。
* 这意味着你可以把 `&'static` 引用传到任何地方。
* 任意位置字符串字面量总是 `&'static str` 类型。

例子：

```rust
static GREETING: &str = "Hello, world!";

fn get_greeting() -> &'static str {
    GREETING
}

fn main() {
    println!("{}", get_greeting());

    // s的类型也是：&'static str
    let s = "hello";
    println!("{}", s);
}
```

`get_greeting` 返回的 `&'static str` 安全无比，因为 `GREETING` 一直存在。


## 内部可变性 + `static`

由于 `static mut` 不安全，我们一般用 **内部可变性**（`Mutex` / `RwLock` / `Atomic`）来定义全局变量。

例子（`AtomicUsize`）：

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);

fn main() {
    COUNTER.fetch_add(1, Ordering::SeqCst);
    println!("COUNTER = {}", COUNTER.load(Ordering::SeqCst));
}
```

这样就不需要 `unsafe` 了。


## `static` vs 局部变量

对比一下：

```rust
fn main() {
    let x = 5;              // 栈上，函数退出后释放
    static Y: i32 = 10;     // 全局静态区，程序结束才释放
}
```


## 高级：`lazy_static!` / `once_cell`

很多时候我们需要 **运行时才能初始化** 的全局变量。
Rust 的 `static` 只能接受编译期常量，所以我们借助宏或库：

### `once_cell::sync::Lazy`

```rust
use once_cell::sync::Lazy;
use std::collections::HashMap;

static CONFIG: Lazy<HashMap<&'static str, &'static str>> = Lazy::new(|| {
    let mut m = HashMap::new();
    m.insert("host", "127.0.0.1");
    m.insert("port", "8080");
    m
});

fn main() {
    println!("Server running on {}:{}", CONFIG["host"], CONFIG["port"]);
}
```

这样就能安全地延迟初始化全局变量了。


## 总结

1. `static` 定义全局变量，存放在静态存储区，生命周期 `'static`。
2. 和 `const` 的区别：`const` 是编译期常量，`static` 是全局存储。
3. `static mut` 允许可变，但不安全；推荐用 `Mutex` / `Atomic`。
4. `static` 变量必须是编译期可求值的常量，若需运行时初始化，用 `Lazy`/`once_cell`。
