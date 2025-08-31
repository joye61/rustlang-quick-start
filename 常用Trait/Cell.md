# `Cell` 是什么？

* `Cell<T>` 是标准库里提供的一种 **内部可变性（interior mutability）** 类型。
* 它允许你在**只持有不可变引用 `&self`** 的情况下，修改其中存放的值。

换句话说：平常 Rust 要求“可变就得有 `&mut`”，但 `Cell` 打破了这个限制，让你用 `&self` 也能改。


# 为什么要有 `Cell`？

在 Rust 正常规则下：

* `&T`（共享引用）只能读，不能改；
* `&mut T`（独占引用）才能改。

但有些场景下你希望**逻辑上是可变的，但语义上不想暴露 `&mut`**，例如：

* 缓存一个计算结果；
* 计数器（例如引用计数 Rc 里）；
* 封装库内部的状态，但对外提供不可变接口。

这时候，就用 `Cell`。


# `Cell` 的特点

1. **只适合存放 `Copy` 类型（小而轻的值）**，因为取出时只能拷贝。
    * 如果是非 `Copy` 类型（比如 `String`），也能放，但操作方式有限（不能直接拿走里面的值，只能替换）。
2. **不是线程安全的**（`Cell` 只能在单线程里用）。
    * 跨线程的版本是 `Atomic` 或 `Mutex`。
3. **操作基于“复制/替换”**，而不是直接借出引用。
    * 这是 `Cell` 和 `RefCell` 的最大区别。


# 常用 API

* `Cell::new(value)` —— 创建一个 `Cell`。
* `set(val)` —— 替换里面的值。
* `get()` —— 返回里面的值（要求 `T: Copy`）。
* `replace(val)` —— 替换并返回旧值。
* `take()` —— 把值取出来，里面放一个 `Default::default()`。
* `into_inner()` —— 消费掉 `Cell`，取出里面的值。


# 示例讲解

## （1）最简单的计数器

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<i32>,  // 内部可变
}

impl Counter {
    fn new() -> Self {
        Counter { count: Cell::new(0) }
    }

    fn incr(&self) {
        let n = self.count.get();
        self.count.set(n + 1);
    }

    fn get(&self) -> i32 {
        self.count.get()
    }
}

fn main() {
    let c = Counter::new();
    c.incr();
    c.incr();
    println!("{}", c.get()); // 输出 2
}
```

即使 `incr` 和 `get` 只接收 `&self`，也能修改内部状态。


## （2）替换非 Copy 类型

```rust
use std::cell::Cell;

fn main() {
    let c = Cell::new(String::from("Hello"));
    
    // 不能直接 get（因为 String 不是 Copy）
    // let s = c.get(); // ❌ 编译错误

    // 正确做法：replace
    let old = c.replace(String::from("World"));
    println!("old = {}, new = {}", old, c.into_inner());
}
```

这里的思路是“拿新值替换，旧值再返回”。


## （3）take 的用法

```rust
use std::cell::Cell;

fn main() {
    let c = Cell::new(Some(123));
    
    let val = c.take(); // 取出 Some(123)，里面变成 None
    println!("val = {:?}, inner = {:?}", val, c.get());
}
```


# `Cell` vs `RefCell`

这是新手最常混淆的地方：

| 特点     | `Cell<T>`                                | `RefCell<T>`                                |
| ------ | ---------------------------------------- | ------------------------------------------- |
| 存储方式   | 值（按拷贝/替换操作）                              | 值（按借用规则管理）                                  |
| 适合的类型  | 小的、`Copy` 类型最合适                          | 任意类型（`T` 不必 `Copy`）                         |
| 取出值的方式 | `get()`（仅 Copy） / `replace()` / `take()` | `borrow()`（`&T`） / `borrow_mut()`（`&mut T`） |
| 运行时开销  | 零开销（编译期保证）                               | 有运行时检查（可能 panic）                            |
| 使用场景   | 简单字段、小数值、计数器                             | 复杂结构，需要临时借用引用                               |

总结一句话：

* **值按 Copy 取用：用 `Cell`**
* **需要借引用：用 `RefCell`**


# 使用场景

* 封装库内部的小状态，不想暴露 `&mut self`。
* 计数器、标志位、缓存。
* 单线程环境下对内部可变性的简单支持。


# 总结

* `Cell<T>` 提供 **内部可变性**，允许在 `&self` 下修改值。
* 核心 API：`get`（Copy 类型取值）、`set`、`replace`、`take`、`into_inner`。
* **适合 Copy 类型**，操作走“复制/替换”，不会借出引用。
* 单线程可用，不保证线程安全。
* 和 `RefCell` 对比：`Cell` 更轻量，但功能有限。
