## `Copy` 直观理解

在 Rust 里，变量的赋值通常会发生**所有权转移（move）**。
但有些类型很轻量，比如 `i32`、`f64`、`char`，如果每次赋值都转移所有权就很麻烦。

于是 Rust 提供了 `Copy` trait：**实现了 `Copy` 的类型，在赋值/传参时不会 move，而是直接复制一份字节拷贝（bitwise copy）**。

这就像：

* `Copy` 类型：赋值是“复印件”，两份互不影响。
* 非 `Copy` 类型：赋值是“过户”，老的变量不能再用了。


## `Copy` 的作用范围

哪些类型可以 `Copy`？

* **所有基本标量类型**：`i32`, `u64`, `bool`, `char`, `f32`, `f64` 等。
* **固定大小的数组**（比如 `[i32; 3]`），只要里面的元素也是 `Copy`。
* **元组**，只要里面所有元素都 `Copy`。
* **自己定义的 struct/enum**，只要它的所有字段/成员都是 `Copy`，你也可以让它 `#[derive(Copy, Clone)]`。

哪些类型不能 `Copy`？

* 含有堆分配、需要 drop 的类型，比如 `String`, `Vec<T>`, `Box<T>`, `HashMap<K,V>`。
* 含有 `Drop` 实现的类型（手动清理资源的类型），绝对不能 `Copy`。


## `Copy` 和 `Clone` 的关系

这是很多新手容易混淆的点。

* **`Copy` 是隐式的**：你写 `let b = a;`，如果 `a` 是 `Copy`，会自动复制。
* **`Clone` 是显式的**：你要写 `let b = a.clone();`。

规则：

1. 所有 `Copy` 类型都必须实现 `Clone`。
2. 但是不是所有 `Clone` 类型都能 `Copy`（比如 `String` 可以 `clone()` 但不能 `Copy`）。

例子：

```rust
let a = 42;       // i32 是 Copy
let b = a;        // 这里 a 还可以用
println!("{}", a);

let s1 = String::from("hello");
let s2 = s1;      // move，s1 不能再用了
// println!("{}", s1); // ❌ 报错
let s3 = s2.clone(); // ✅ 显式克隆
```


## `Copy` 的底层原理

Rust 编译器在编译时会根据类型是否实现 `Copy` 来决定**赋值/传参的语义**。

* 如果是 `Copy`：
  * 赋值/传参 → 生成汇编就是一份**字节复制**（memcpy 的意思）。
  * 不会生成 `Drop` 调用。
* 如果不是 `Copy`：
  * 赋值/传参 → 所有权转移，老的变量不能用了。
  * 出作用域时，编译器会调用 `Drop` 清理。

换句话说：
`Copy` 类型完全是“栈上数据”，不会涉及资源释放。


## 自定义类型实现 `Copy`

你可以给自己的 struct 派生 `Copy`，但必须保证里面的字段也都是 `Copy`。例子：

```rust
#[derive(Copy, Clone, Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1;   // ✅ p1 还可以用，因为 Point 是 Copy
    println!("{:?}, {:?}", p1, p2);
}
```

例子 ❌（不能 Copy）：

```rust
#[derive(Copy, Clone)]
struct Person {
    name: String,  // ❌ String 不是 Copy
    age: u32,
}
```

会报错：

```
the trait `Copy` may not be implemented for this type
```

## `[derive(Copy, Clone)]`

它表示让编译器帮你同时自动实现 Copy 和 Clone trait。等价于：

```rust
impl Copy for Point {}
impl Clone for Point {
    fn clone(&self) -> Self {
        *self   // 这里的 *self 依赖 Copy 的实现，等于直接字节拷贝
    }
}
```

也就是说：

* Copy 让类型能隐式复制。
* Clone 让类型能显式调用 .clone()。
* 当你 derive 它们时，Clone 的实现就是简单调用 Copy。

为什么要同时写 `Copy` 和 `Clone`，Rust 规定：**所有实现了 Copy 的类型，也必须实现 Clone**

## `Copy` 的细节和陷阱

### (1) `Copy` 不会调用自定义逻辑

`Copy` 是**按位复制**，不会调用任何函数。
所以如果你想在复制时做点额外事情（比如日志），用 `Clone`。

### (2) `Copy` 和 `Drop` 是互斥的

Rust 明确规定：
一个类型要么是 `Copy`，要么实现 `Drop`，两者不可兼得。
原因很简单：如果能 `Copy`，那编译器不知道该调用几次 `Drop`（会重复释放）。

### (3) 引用是 `Copy`

很多人容易忽略：

* `&T` 是 `Copy`，复制引用不会 move。
* 但 `&mut T` 不是 `Copy`，因为那会破坏 Rust 的独占可变借用规则。

例子：

```rust
fn main() {
    let x = 10;
    let r1 = &x;
    let r2 = r1;  // ✅ &i32 是 Copy
    println!("{} {}", r1, r2);

    let mut y = 20;
    let m1 = &mut y;
    // let m2 = m1; // ❌ &mut T 不是 Copy
}
```

### (4) `Copy` 的语义 ≈ C 语言的浅拷贝

和 C 里 `memcpy` 一个 struct 一样，就是字节级别的拷贝，没有智能指针那种“深拷贝”。


## 总结

* **`Copy` = 隐式 bitwise copy，不会发生 move**。
* 适用于轻量、栈上存储、无 Drop 的类型。
* 所有 `Copy` 类型也必须 `Clone`，但不是所有 `Clone` 都能 `Copy`。
* 引用 `&T` 是 `Copy`，`&mut T` 不是。
* 自定义类型要 `Copy`，必须保证所有字段都是 `Copy`。

一句话总结：
**`Copy` 是 Rust 编译器为轻量类型提供的一种“自动复印”机制，它避免了频繁的所有权转移，但只能用在安全的、无需清理资源的类型上。**
