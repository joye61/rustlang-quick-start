## 什么是 Option

`Option<T>` 是 Rust **标准库提供的枚举类型**，用来表示 **可能存在或可能不存在的值**。它定义如下：

```rust
enum Option<T> {
    None,       // 没有值
    Some(T),    // 包含一个 T 类型的值
}
```

* `T` 是泛型类型参数。
* 与其他语言的 `null`/`nullable` 不同，Rust **强制使用 Option**，避免空指针错误。
* 所有可能缺失的值都推荐使用 `Option`。


## 创建 Option

```rust
let some_number: Option<i32> = Some(5);
let no_number: Option<i32> = None;
```

* `Some(value)` 表示存在值。
* `None` 表示没有值。
* 类型可以自动推导。

```rust
let x = Some(10); // x: Option<i32>
let y: Option<f64> = None;
```


## 解包 Option

使用 `match`：

```rust
let x = Some(7);

match x {
    Some(v) => println!("Value is {}", v),
    None => println!("No value"),
}
```

使用 `if let`：

```rust
if let Some(v) = x {
    println!("Value is {}", v);
} else {
    println!("No value");
}
```

`if let` 是 `match` 的简化形式，适合只关心某个分支。


## Option 常用方法

| 方法                         | 描述                        |
| -------------------------- | ------------------------- |
| `is_some()`                | 是否是 Some                  |
| `is_none()`                | 是否是 None                  |
| `unwrap()`                 | 获取值，如果是 None 则 panic      |
| `unwrap_or(default)`       | 获取值，如果是 None 返回默认值        |
| `unwrap_or_else(f)`        | 获取值，如果是 None 调用闭包生成默认值    |
| `map(f)`                   | 对 Some 的值应用函数，返回新的 Option |
| `and_then(f)` / `flat_map` | 链式调用返回 Option 的函数         |
| `ok_or(err)`               | 将 Option 转成 Result        |
| `expect(msg)`              | 类似 unwrap，但 panic 时自定义消息  |


示例：

```rust
let a = Some(5);
let b = None;

println!("{}", a.is_some()); // true
println!("{}", b.is_none()); // true

let value = a.unwrap(); // 5
let default_value = b.unwrap_or(10); // 10

let c = a.map(|x| x * 2); // Some(10)
```

## 链式操作 Option

Option 支持很多函数式风格操作，非常方便：

```rust
fn increment(x: Option<i32>) -> Option<i32> {
    x.map(|v| v + 1)
}

let n = Some(5);
let m = increment(n); // Some(6)
let o = increment(None); // None
```

* `map` 对 Some 的值应用函数，不改变 None。
* `and_then` 用于返回 Option 的函数链式调用：

```rust
fn half(x: i32) -> Option<i32> {
    if x % 2 == 0 { Some(x / 2) } else { None }
}

let result = Some(8).and_then(half).and_then(half); // Some(2)
```


## Option 与模式匹配

Option 是一个枚举，所以完全可以用 `match`：

```rust
let x = Some(42);

match x {
    Some(v) if v > 40 => println!("Big number: {}", v),
    Some(v) => println!("Small number: {}", v),
    None => println!("No number"),
}
```

* 可以用 **条件匹配 `if`** 过滤值。
* 这种模式匹配是 Rust 安全处理空值的核心。


## Option 与 Result 转换

* `ok_or(err)`：Option -> Result
* `ok_or_else(f)`：Option -> Result，用闭包生成错误

```rust
let x = Some(5);
let r: Result<i32, &str> = x.ok_or("Value is missing");
```

* `Some(5)` -> `Ok(5)`
* `None` -> `Err("Value is missing")`


## 总结

`Option<T>` 的核心思想：

1. 安全地表示值可能缺失。
2. 使用枚举变体 `Some(T)` 和 `None`。
3. 强制显式处理缺失值，避免 null 引发的错误。
4. 支持模式匹配、链式操作、方法调用。
5. 与 `Result` 配合可以方便地处理错误与缺失值。
