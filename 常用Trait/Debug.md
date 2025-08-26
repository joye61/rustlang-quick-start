## `Debug` 是什么？

`Debug` 是 Rust 标准库里的一个 trait（`std::fmt::Debug`），它的作用是：**让你的类型可以用 `{:?}` 或 `{:#?}` 打印出来，方便调试。**

区别于：

* `Display`：面向用户的友好字符串（用 `{}`）。
* `Debug`：给开发者看的调试输出（用 `{:?}`）。


## 快速示例

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);   // 输出：Point { x: 1, y: 2 }
    println!("{:#?}", p);  // 美化输出：多行缩进
}
```

输出：

```
Point { x: 1, y: 2 }
Point {
    x: 1,
    y: 2,
}
```


## 如何让类型支持 `Debug`

有三种方式：

### (1) 自动派生（最常用）

```rust
#[derive(Debug)]
struct Foo { a: i32, b: bool }
```

编译器自动帮你生成 `Debug` 实现，直接能 `{:?}` 打印。

### (2) 手动实现

适合需要**自定义打印格式**的场景：

```rust
use std::fmt;

struct Creds {
    user: String,
    password: String,
}

impl fmt::Debug for Creds {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Creds")
            .field("user", &self.user)
            .field("password", &"***HIDDEN***")
            .finish()
    }
}

fn main() {
    let c = Creds { user: "alice".into(), password: "123456".into() };
    println!("{:?}", c);
    // 输出: Creds { user: "alice", password: "***HIDDEN***" }
}
```

### (3) 容器类型自动转发

比如 `Vec<T>`, `Option<T>`, `Result<T,E>`：只要里面的 `T` / `E` 实现了 `Debug`，容器本身也能 `Debug`。


## `Debug` 的核心原理

`Debug` trait 定义如下：

```rust
pub trait Debug {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result;
}
```

重点：

* `f` 是格式化器，可以控制输出样式。
* `f.alternate()`：如果用户用了 `{:#?}` 就会返回 `true`，可以据此选择输出风格。
* Rust 提供了 `debug_struct`, `debug_tuple`, `debug_list`, `debug_map` 等构建器，帮你快速打印结构化类型。


## 调试专用工具 `dbg!`

Rust 提供了一个超级方便的宏 `dbg!`：

```rust
fn main() {
    let x = 2;
    let y = dbg!(x * 3) + 1;
    println!("y = {}", y);
}
```

输出到 **stderr**：

```
[src/main.rs:3] x * 3 = 6
```

注意：

* `dbg!` 会显示 文件名 + 行号 + 表达式 + 结果。
* 它会返回被打印的值（所以还能继续参与计算）。
* 常用于临时调试。


## 与 `Display` 的区别

| Trait     | 格式符    | 面向谁？    | 用途举例                    |
| --------- | ------ | ------- | ----------------------- |
| `Debug`   | `{:?}` | 开发者调试用  | `println!("{:?}", foo)` |
| `Display` | `{}`   | 用户可读的输出 | `println!("{}", foo)`   |

如果你只实现了 `Debug`，用 `{}` 会报错。


## 泛型与 `Debug`

当你 `derive(Debug)` 一个带泛型的类型时，编译器会自动加约束：

```rust
#[derive(Debug)]
struct Wrapper<T>(T);
// 等价于：impl<T: Debug> Debug for Wrapper<T> { ... }
```

如果 `T` 没有实现 `Debug`，就不能 `{:?}` 打印 `Wrapper<T>`。


## 常见陷阱与注意点

1. **敏感数据**
   不要直接 `derive(Debug)` 打印密码、密钥，应该手动实现 `Debug` 隐藏它们。

2. **性能**
   `Debug` 打印可能非常大（例如大数组、深层嵌套），调试时用，生产环境少用。

3. **递归数据结构**
   如果存在循环引用（比如 `Rc<RefCell<...>>`），可能导致无限递归打印，需谨慎。

4. **测试依赖 Debug**
   `assert_eq!` / `assert_ne!` 依赖 `Debug` 打印出断言失败时的值，所以写测试时类型最好都实现 `Debug`。

## 进阶：美化输出

通过 `f.alternate()` 实现不同模式：

```rust
use std::fmt;

struct Pair(i32, i32);

impl fmt::Debug for Pair {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        if f.alternate() {
            write!(f, "Pair(\n  {},\n  {}\n)", self.0, self.1)
        } else {
            write!(f, "Pair({}, {})", self.0, self.1)
        }
    }
}

fn main() {
    let p = Pair(1, 2);
    println!("{:?}", p);   // Pair(1, 2)
    println!("{:#?}", p);  // 多行美化输出
}
```


# 总结

* `Debug` 用来调试打印，格式符 `{:?}`。
* 最常用方式：`#[derive(Debug)]`。
* 手动实现可以定制输出，尤其适合隐藏敏感信息。
* `{:#?}` 是美化版输出。
* `dbg!` 宏是调试利器。
* 测试断言需要 `Debug`。
* `Debug` 是“给开发者看”的，不保证输出稳定或适合用户。
