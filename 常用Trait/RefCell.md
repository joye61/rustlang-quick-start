## `RefCell` 是什么？

在 Rust 里，**编译器通常会在编译阶段就检查借用规则**（谁能读，谁能写，能不能同时借用等）。

但是有时候，你写代码时 **很难在编译期保证**这些规则（比如在某些复杂的数据结构里）。

**`RefCell<T>` 就是一个“逃生通道”**：

* 它把 **借用规则的检查** 从 **编译期** 推迟到 **运行时**。
* 编译时你可以放心写，运行时如果借用规则被破坏，就会 **panic**（程序崩溃）。

一句话：**`RefCell` 允许在运行时动态检查借用规则**。


## 核心特性

* **`RefCell<T>` 是不可变的容器**，但是你可以在里面进行可变借用。

* 它提供两种方法：

  1. `borrow()` —— 获取一个不可变引用（返回 `Ref<T>`）
  2. `borrow_mut()` —— 获取一个可变引用（返回 `RefMut<T>`）

* 规则（和普通引用一样）：

  * 可以有 **多个不可变借用** (`Ref<T>`)。
  * 只能有 **一个可变借用** (`RefMut<T>`)，并且不可和不可变借用同时存在。

区别是：

* 普通引用：**编译时检查**。
* `RefCell`：**运行时检查**，违规时 `panic!`。


## 示例讲解

### 多个不可变借用

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(10);

    let r1 = data.borrow(); // 第一次不可变借用
    let r2 = data.borrow(); // 第二次不可变借用

    println!("r1 = {}, r2 = {}", *r1, *r2);
}
```

多个 `borrow()` 同时存在，没问题。输出：

```
r1 = 10, r2 = 10
```


### 单个可变借用

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(10);

    {
        let mut r = data.borrow_mut(); 
        *r += 5;  // 修改里面的值
    } // r 在这里被释放

    println!("new value = {}", data.borrow()); 
}
```

输出：

```
new value = 15
```

### ❌ 错误用法：同时借用可变和不可变

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(10);

    let r1 = data.borrow();       // 不可变借用
    let r2 = data.borrow_mut();   // ❌ 尝试可变借用

    println!("{} {}", *r1, *r2);
}
```

运行时会 `panic`：

```
thread 'main' panicked at 'already borrowed: BorrowMutError'
```


## 使用场景

`RefCell` 最常见的场景就是：**在单线程里突破编译器的限制**，尤其是和 **智能指针 `Rc`** 一起用。

因为 `Rc` 允许多个所有者，但 `Rc` 自己只提供不可变访问。
要让多个所有者能 **修改共享的数据**，就需要 `Rc<RefCell<T>>`。

示例：多个对象共享同一个值，并能修改它：

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let shared = Rc::new(RefCell::new(0));

    let a = Rc::clone(&shared);
    let b = Rc::clone(&shared);

    *a.borrow_mut() += 10;  // 修改
    *b.borrow_mut() += 20;  // 修改

    println!("final value = {}", shared.borrow()); // 输出 30
}
```


## 总结

* `RefCell<T>` 允许 **在运行时进行借用检查**。
* 提供 `borrow()` 和 `borrow_mut()`：

  * `borrow()` → `Ref<T>`，可多个同时存在。
  * `borrow_mut()` → `RefMut<T>`，只能有一个，且不能和 `Ref<T>` 共存。
* 如果违反规则 → **运行时 panic**。
* 常与 `Rc` 搭配，用来在单线程环境里实现“多个所有者共享可变数据”。
