# `Sized` 和 不定长类型（DST）


## 定长类型

在 Rust 中，每个类型的大小（size）必须**在编译时确定**，这样编译器才能为其分配内存。

这类“大小已知”的类型，被称为 `Sized` 类型。Rust 中所有类型默认都实现了 `Sized`

```rust
let x: i32 = 5;
let y: (u8, bool) = (1, true);
```

这些都是静态大小类型，大小已知 → 默认满足 `Sized` trait。

`Sized` 是一个特殊 trait，它定义在标准库中，长这样：

```rust
#[lang = "sized"]
pub trait Sized {}
```

* 大部分类型默认就是 `Sized`
* 编译器会为每个类型自动判断是否是 `Sized`


## 泛型中的 `Sized` 限制（默认约束）

```rust
// 默认等价于：fn print<T: Sized>(value: T)
fn print<T>(value: T) {
    
}
```

Rust 会自动给泛型函数加上 `T: Sized` 限制 —— 也就是说，默认**只接受大小已知的类型**。


## 不定长类型

有些类型的大小不能在编译时确定，比如：

* `[T]`：一个切片（如 `[1, 2, 3]`）
* `str`：一个 UTF-8 编码的字符串切片
* `dyn Trait`：Trait 对象（例如 `dyn Display`）

这些类型在编译时**不知道大小是多少**，无法被直接分配到栈上。它们被称为：**DST：Dynamically Sized Types（动态大小类型）**


## 想要支持 DST？使用 `T: ?Sized`

`?Sized` 表示：这个类型 **可能不是 `Sized`**，也可能是！

```rust
fn print_value<T: ?Sized>(val: &T) {
    // val 必须是引用，才能传入 DST
    println!("Address: {:p}", val);
}
```

### 正确调用：

```rust
print_value(&"hello");         // str → DST → OK
print_value(&[1, 2, 3]);       // [i32] → DST → OK
print_value(&123);            // i32 → Sized → OK
```

## 常见 DST 类型有哪些？

| 类型          | 是否 DST | 是否能直接用？ | 正确用法                           |
| ----------- | ------ | ------- | ------------------------------ |
| `i32`、`f64` | ❌      | ✅       | `let x: i32 = 1;`              |
| `[T]`（切片）   | ✅      | ❌不能直接用  | `&[T]`, `Box<[T]>`             |
| `str`       | ✅      | ❌不能直接用  | `&str`, `Box<str>`             |
| `dyn Trait` | ✅      | ❌不能直接用  | `&dyn Trait`, `Box<dyn Trait>` |

### ❌ 错误示例：

```rust
let s: str = "hello".to_string();  // ❌ 编译失败：str 是 DST，不能直接用
```

## 胖指针（Fat Pointer）

DST **不能直接使用**，但你可以用引用或 `Box` 来包裹它们。

这时 Rust 用的是：**胖指针（Fat Pointer）**

胖指针 = 普通指针 + 元数据

| DST 类型       | 胖指针内容           |
| ------------ | --------------- |
| `&[T]`       | 指针 + 长度         |
| `&str`       | 指针 + 字节数        |
| `&dyn Trait` | 指针 + vtable（虚表） |

```rust
fn show_info(slice: &[i32]) {
    println!("Address: {:p}, Length: {}", slice.as_ptr(), slice.len());
}
```

胖指针其实是一种多字段结构，内部隐藏管理元信息。


## 如何使用 DST？

### 使用引用

```rust
fn print_str(s: &str) {
    println!("{}", s);
}
```

你不能拥有 `str`，但可以拥有 `&str`


### 使用 `Box<T>` 存储 DST

`Box<T>` 支持堆上分配 DST。

```rust
fn main() {
    let boxed_str: Box<str> = "hello".to_owned().into_boxed_str();
    let boxed_slice: Box<[i32]> = vec![1, 2, 3].into_boxed_slice();
}
```



### 使用 Trait 对象（`dyn Trait`）

```rust
use std::fmt::Display;

fn print_display(x: &dyn Display) {
    println!("{}", x);
}
```

* `dyn Display` 是 DST
* `&dyn Display` 是胖指针



## 你不能创建 DST 的“值”，只能通过引用/Box

```rust
fn main() {
    let x: str = *"hello"; // ❌ 错误：str 是 DST，不能这样用
}
```

解决方案：

```rust
let x: &str = "hello"; // ✅ 正确：用胖指针存储 DST
```



## 泛型中用 `?Sized` 的完整示例

```rust
use std::fmt::Debug;

// 支持 DST 的通用打印函数
fn debug_print<T: ?Sized + Debug>(val: &T) {
    println!("{:?}", val);
}

fn main() {
    let s = "hello";
    let arr = [1, 2, 3];
    debug_print(&s);     // &str
    debug_print(&arr);   // &[i32]
    debug_print(&123);   // &i32
}
```


## 总结：为什么 `Sized` 和 `?Sized` 重要？

| 你想做的事       | 是否需要注意 `Sized`？ | 说明                            |
| ----------- | --------------- | ----------------------------- |
| 定义泛型函数      | 默认有 `T: Sized`  | 若要支持 DST，必须写 `T: ?Sized`      |
| 接收字符串或切片参数  | ✅               | 只能用 `&str` 或 `&[T]`（胖指针）      |
| 存储 DST      | ✅               | 用 `Box<T>`、`&T`、`Rc<T>` 等包裹方式 |
| 定义 trait 对象 | ✅               | 使用 `dyn Trait`，并通过胖指针访问       |
| 构造复杂数据结构    | ✅               | 某些字段可能是 DST，需要特殊处理            |



## DST 不能直接使用？


### “引用”的本质

在 Rust 中，**引用**（`&T`）是一个指针类型，指向某个值在内存中的位置。

```rust
let x = 10;
let r = &x;
```

此时：

* `x` 是值（i32，大小已知）
* `r` 是引用，指向 `x`，占用 8 字节（64 位系统）

本质：引用是个**指针**，但加了 Rust 的所有权和借用规则

| 类型       | 实际含义       |
| -------- | ---------- |
| `&T`     | 指针，指向类型 T  |
| `&mut T` | 可变指针       |
| `Box<T>` | 堆上指针（智能指针） |


### 为什么 DST 不能直接使用？

动态大小类型（DST）是：**在编译时无法确定大小的类型**。

例如：

```rust
str      // 字符串切片（长度不固定）
[T]      // 数组切片（如 [i32]）
dyn Trait // Trait 对象（实际类型未知）
```

编译器拒绝这样写：

```rust
let s: str = *"hello";   // ❌ 错误：`str` 大小未知
let arr: [i32] = [1, 2, 3]; // ❌ 编译不通过
```

原因：**栈内存需要知道大小**

Rust 中所有变量都默认存在**栈**上，而栈是严格按大小对齐的线性内存块。

> 你要放个变量到栈上，编译器就必须知道：“这个变量到底占几个字节？”

如果你说：我要放一个 `str` 类型的值，但没有告诉我它多长（几字节），那我编译器就没法安排空间，也没法生成代码。

### 为什么通过引用就可以使用 DST？

因为：**引用类型本身是固定大小的指针**，它存储在栈上，而 DST 的“实际大小”信息可以通过**胖指针**存储。


### 胖指针（fat pointer）：让 DST “带参数”使用

对于 DST 类型，Rust 不是简单给一个地址，而是给**多个信息**组成的“胖指针”，胖指针的结构：

| DST 类型       | 胖指针内容               |
| ------------ | ------------------- |
| `&[T]`       | 指针 + 长度（多少个 T）      |
| `&str`       | 指针 + 字节数            |
| `&dyn Trait` | 指针 + 虚函数表指针（vtable） |

举例：`&str` 实际结构

```rust
let s: &str = "hello";
```

* `s` 是胖指针，占两个 `usize`（64 位下是 16 字节）
* 第一个 `usize`：指向 `h`
* 第二个 `usize`：长度 = 5（"hello" 有 5 个字节）

举例：`&dyn Display`

```rust
use std::fmt::Display;

let num: i32 = 42;
let d: &dyn Display = &num;
```

* `d` 是胖指针

  * 第一个部分：指向 `num`
  * 第二部分：指向 `vtable`，即虚函数表，包含了：

    * `Display::fmt`
    * `Drop`
    * `Any` 等方法的函数地址


## 代码实证

### 错误示例：DST 不能直接用

```rust
fn main() {
    let s: str = *"hello"; // ❌ error: the size for values of type `str` cannot be known at compilation time
}
```

### 正确示例：使用引用（胖指针）

```rust
fn main() {
    let s: &str = "hello";       // &str 是胖指针
    let arr: &[i32] = &[1, 2, 3]; // &[i32] 是胖指针
    println!("{}", s);
    println!("{:?}", arr);
}
```


## DST 的使用场景总结

| 你想用的 DST 类型 | 正确方式                            | 说明          |
| ----------- | ------------------------------- | ----------- |
| `str`       | `&str` / `Box<str>`             | 通过引用或堆分配    |
| `[T]`       | `&[T]` / `Box<[T]>`             | 不能直接写 `[T]` |
| `dyn Trait` | `&dyn Trait` / `Box<dyn Trait>` | Trait 对象    |


## 总结

1. Rust 是静态内存安全语言
2. 编译时必须知道变量大小，才能安排内存（栈）
3. DST 没有固定大小，不能放栈
4. 但引用是固定大小的指针（或胖指针），可以在编译时使用
5. 所以 DST **只能通过引用或 Box 等指针类型使用**
