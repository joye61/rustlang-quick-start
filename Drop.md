## Drop 是什么？

`Drop` 是 Rust 的一个 **trait**，用于**在值离开作用域时自动清理资源**。
通俗理解：它类似其他语言中的析构函数（C++ 的 destructor），用来释放内存、关闭文件、断开网络连接等。

核心方法：

```rust
fn drop(&mut self)
```

重点：**当值离开作用域时，Rust 会自动调用这个方法。**

> 注意：Rust 中的资源管理靠 **所有权（Ownership）**，所以 Drop 主要是释放 **非内存资源**，比如文件句柄、Socket、数据库连接等。


## Drop 的基本使用

```rust
struct Resource {
    name: String,
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("Dropping Resource: {}", self.name);
    }
}

fn main() {
    let res = Resource { name: String::from("File1") };
    println!("Resource created");
} // 这里 res 离开作用域，drop 自动调用
```

输出：

```
Resource created
Dropping Resource: File1
```

解释：

* 当 `res` 离开作用域，Rust 自动调用 `drop` 方法。
* 不需要手动调用，**Rust 的 RAII（资源获取即初始化）机制**保证资源自动释放。


## Drop 的触发时机

1、**作用域结束**

```rust
{
    let x = Resource { name: "X".to_string() };
} // x 离开作用域 → drop 被调用
```

2、**变量被覆盖**

```rust
let x = Resource { name: "X".to_string() };
let x = Resource { name: "Y".to_string() }; // 原来的 X 会先被 drop
```

3、**Vec、Box 等集合或智能指针元素被销毁**

```rust
let v = vec![Resource { name: "A".to_string() }];
// v 离开作用域 → drop 每个元素
```

4、**显式调用 std::mem::drop**

```rust
let x = Resource { name: "X".to_string() };
std::mem::drop(x); // 提前 drop，x 后续不可再使用
```

> 注意：`std::mem::drop` 与 `Drop::drop` 不同，前者是 **显式提前销毁**，不能直接调用 `drop(x)`，Rust 不允许手动调用 `Drop::drop` 方法。


## Drop 与所有权

* Rust **所有权系统**保证每个值在任意时刻只有一个所有者。
* 当所有者离开作用域时，`Drop` 被自动调用。
* **不能实现 Copy 的类型**通常需要 Drop。

示例：

```rust
#[derive(Copy, Clone)]
struct Dummy;

impl Drop for Dummy {
    fn drop(&mut self) {
        println!("Dropping Dummy");
    }
}
// ❌ 报错：实现 Drop 的类型不能同时实现 Copy
```

原因：引用数据如果被复制，存在同一指针被二次Drop的风险


## Drop 与 Rc / Arc / Box

* `Box<T>`、`Rc<T>`、`Arc<T>` 都实现了 Drop
* 当智能指针离开作用域：

  * Box → 释放堆上的内存
  * Rc / Arc → 引用计数减 1，计数为 0 时释放

```rust
use std::rc::Rc;

struct Resource {
    name: String,
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("Dropping Resource: {}", self.name);
    }
}

fn main() {
    let a = Rc::new(Resource { name: "A".to_string() });
    let b = Rc::clone(&a);
    println!("Count: {}", Rc::strong_count(&a));
} // Rc 引用计数变为 0 → drop 调用
```


## Drop 的注意事项

1、**不要在 drop 内部使用被移走的值**

```rust
struct Foo {
    inner: String,
}

impl Drop for Foo {
    fn drop(&mut self) {
        // println!("{}", self.inner); // 可以访问，因为是 &mut self
    }
}
```

2、**不要手动调用 Drop::drop**

* 正确做法：

```rust
std::mem::drop(val); // 显式释放
```

3、**Drop 的执行顺序**

* Rust 会**先 drop 后定义的变量**：

```rust
let a = Resource { name: "A".to_string() };
let b = Resource { name: "B".to_string() };
// 离开作用域顺序：b -> a
```

4、**循环引用会导致 Drop 不被调用**

* Rc/RefCell 循环引用 → 内存泄漏
* Arc + Mutex 也可能

---

## Drop 的进阶用法

1、**管理非内存资源**

```rust
use std::fs::File;
use std::io::Write;

struct TempFile {
    file: File,
}

impl Drop for TempFile {
    fn drop(&mut self) {
        println!("Deleting temp file");
        // 自动关闭文件
    }
}
```

2、**提前释放资源**

```rust
{
    let res = Resource { name: "temp".to_string() };
    // 需要提前释放
    std::mem::drop(res);
    println!("res dropped early");
}
```


## 总结

* `Drop` 是 Rust 的析构机制，用于资源释放
* Rust 自动在作用域结束时调用 drop
* 可以显式使用 `std::mem::drop` 提前释放
* Drop 不能与 Copy 共存
* Drop 的顺序：**后声明先 drop**
* 常用场景：文件、网络连接、智能指针、堆内存管理
