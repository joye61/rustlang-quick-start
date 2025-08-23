# 为什么需要 `Rc`？

在 Rust 里，一个值有且只能有一个所有者，默认情况不允许多个所有者共享一块堆内存。

但现实中我们经常需要多个地方同时使用同一份数据，比如：

* **树结构**：多个父节点共享子节点；
* **图结构**：多个路径共享某个节点；
* **配置数据**：多个对象都依赖相同配置。

为了解决 **单线程下共享所有权** 的需求，Rust 提供了 **`Rc<T>` (Reference Counted)**。


# `Rc` 的原理

* `Rc<T>` 内部维护了一个 **引用计数器**（`strong_count`）。
* 每次 `clone`，计数器 +1；每次 `Rc` 被 drop，计数器 -1。
* 当计数器归零时，数据会被自动释放。

> 注意：`Rc::clone` 只增加计数，不会真正复制数据（很轻量）。



# `Rc` 的用法

### 基础示例：多个所有者共享数据

```rust
use std::rc::Rc;

fn main() {
    let s1 = Rc::new(String::from("hello Rc"));

    let s2 = Rc::clone(&s1); // 增加引用计数
    let s3 = Rc::clone(&s1);

    println!("s1: {}", s1);
    println!("引用计数: {}", Rc::strong_count(&s1));
    println!("s2: {}", s2);
    println!("s3: {}", s3);
}
```

输出：

```
s1: hello Rc
引用计数: 3
s2: hello Rc
s3: hello Rc
```

`s1`、`s2`、`s3` 同时拥有数据 `"hello Rc"`。


### 用在树结构中

`Rc` 常见用途就是构造 **不可变的树/链表**。

```rust
use std::rc::Rc;

struct Node {
    value: i32,
    next: Option<Rc<Node>>,
}

fn main() {
    let a = Rc::new(Node { value: 10, next: None });
    let b = Rc::new(Node { value: 20, next: Some(Rc::clone(&a)) });
    let c = Rc::new(Node { value: 30, next: Some(Rc::clone(&a)) });

    println!("a 引用计数: {}", Rc::strong_count(&a)); // 3
}
```

节点 `a` 被 `b` 和 `c` 共享，不需要复制。


# `Rc` 的限制

**重要：`Rc` 不是万能的**

1. **不是线程安全的**
   `Rc<T>` 只能在单线程里使用。如果跨线程使用会报错。
   多线程下要用 **`Arc<T>`**。

2. **不可变共享，不支持可变引用**
   `Rc` 只能提供 **不可变访问**（`&T`），因为允许多个所有者，如果同时可变就会有数据竞争。

   如果你想要在单线程下 **可变共享**，需要配合 **`RefCell<T>`**。
   典型组合：`Rc<RefCell<T>>`。

# 进阶：`Rc<RefCell<T>>`

如果我们想让多个所有者都能修改数据，可以用 `RefCell<T>`（运行时检查借用）

```rust
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let v = Rc::new(RefCell::new(5));

    let v1 = Rc::clone(&v);
    let v2 = Rc::clone(&v);

    *v1.borrow_mut() += 10;
    *v2.borrow_mut() += 20;

    println!("v = {}", v.borrow()); // 35
}
```

`RefCell` 提供 **可变借用**（即使外面是 `Rc` 共享的）。


# Rc` 的弱引用：`Weak<T>`

有时候，我们需要共享但**不增加引用计数**，避免循环引用（内存泄漏）。

👉 比如树的子节点有父节点指针，如果都用 `Rc`，会出现 **循环引用**，数据永远不会释放。
解决办法：子节点指向父节点时用 **`Weak<T>`**。

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,   // 弱引用
    children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
    let parent = Rc::new(Node {
        value: 1,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    let child = Rc::new(Node {
        value: 2,
        parent: RefCell::new(Rc::downgrade(&parent)), // 弱引用
        children: RefCell::new(vec![]),
    });

    parent.children.borrow_mut().push(Rc::clone(&child));

    println!("parent 引用计数 = {}", Rc::strong_count(&parent)); // 1
    println!("child 引用计数 = {}", Rc::strong_count(&child));   // 1
}
```

这样就避免了循环引用（父引用子是强引用，子引用父是弱引用）。


# 总结

* **`Rc<T>`**：单线程下的 **引用计数智能指针**，允许多个所有者共享数据。
* **不能跨线程**，也不能直接可变。
* 组合方式：

  * `Rc<T>` → 多个所有者共享不可变数据。
  * `Rc<RefCell<T>>` → 多个所有者共享可变数据（单线程）。
  * `Rc` + `Weak` → 避免循环引用。
