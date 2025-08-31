# `AsRef` 是什么？

`AsRef` 是一个 **标准库 trait**，作用是：**把一个值安全、零开销地借用成另一个类型的引用 `&T`。**

它的定义很简单：

```rust
pub trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}
```

* **输入**：`self`（不会消耗所有权，只读）
* **输出**：一个 `&T`（共享借用）
* **?Sized**：允许目标类型是 `str`、`[T]` 这类动态大小类型。

# 为什么要用 `AsRef`？

* 当你写函数时，只想“读一个视图”，不在乎调用者传进来的具体类型。
* 用 `AsRef`，你就能写出**更通用、更方便调用**的函数。

例子：
你想写一个函数，输入任何能当作路径的东西（`&str`、`String`、`PathBuf`…），都能用。

```rust
use std::path::Path;

fn print_path<P: AsRef<Path>>(p: P) {
    let path: &Path = p.as_ref();
    println!("path: {:?}", path);
}

fn main() {
    print_path("a.txt");                  // &str
    print_path(String::from("b.txt"));    // String
    print_path(Path::new("c.txt"));       // &Path
}
```


# 标准库里常见的实现

Rust 已经帮你做好了很多：

* `String` → `str`
* `Vec<T>` → `[T]`
* `PathBuf` / `&PathBuf` → `Path`
* `Box<T>` / `Rc<T>` / `Arc<T>` → `T`
* **自反实现**：任何 `T` 都能 `AsRef<T>`（借用自己）

所以大部分时候你直接写 `AsRef` 约束，就能同时支持很多类型。


# 常见用法示例

## （1）字符串

```rust
fn is_empty<S: AsRef<str>>(s: S) -> bool {
    s.as_ref().is_empty()
}

assert!(is_empty(""));
assert!(is_empty(String::new()));
```

## （2）切片

```rust
fn sum<S: AsRef<[i32]>>(s: S) -> i32 {
    s.as_ref().iter().sum()
}

assert_eq!(sum(vec![1, 2, 3]), 6);   // Vec
assert_eq!(sum([1, 2, 3]), 6);       // 数组
```

## （3）自定义类型

```rust
struct Username(String);

impl AsRef<str> for Username {
    fn as_ref(&self) -> &str {
        &self.0
    }
}

fn greet<S: AsRef<str>>(s: S) {
    println!("Hello, {}!", s.as_ref());
}

greet(Username("Alice".into()));
```


# 和其它 trait 的区别

* **`Deref`**：用于 `*` 解引用、方法调用自动解引用。
  → 偏“语法糖”，而 `AsRef` 是显式调用。

* **`From` / `Into`**：转换时会拿走所有权，可能分配或拷贝。
  → `AsRef` 只返回借用，不分配。

* **`Borrow`**：保证借用后的值在**相等性/哈希**上一致，主要给集合用。
  → `AsRef` 更宽泛，只要求能借用。

* **`AsMut`**：`AsRef` 的可变版，返回 `&mut T`。


# 6什么时候用 / 什么时候实现

✅ 适合用 `AsRef`：

* 你写的函数只需要 **读引用**，不在乎输入是 `String`、`&str`、`PathBuf`…
* 想让 API 更通用。

✅ 适合实现 `AsRef`：

* 你的类型内部本质就是另一个类型，能直接返回引用（如新类型包装）。
* 实现必须 **零开销**、不做分配。

❌ 不要实现 `AsRef`：

* 如果需要额外计算/转换/分配才能得到目标引用。


# 总结

* **定义**：把值借用成 `&T`。
* **特点**：只读、零开销、返回引用。
* **用途**：写泛型函数时，统一接受多种输入类型。
* **区别**：比 `From/Into` 轻，比 `Deref` 明确，比 `Borrow` 更宽松。
