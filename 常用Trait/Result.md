## `Result` 是什么？

`Result` 是 Rust 用于处理**可恢复错误**的标准工具。它是一个**枚举类型**，定义如下：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

* `T`: 表示**成功值类型**
* `E`: 表示**错误类型**


## 最基本的使用示例

```rust
fn divide(x: i32, y: i32) -> Result<i32, String> {
    if y == 0 {
        Err("除数不能为0".to_string())
    } else {
        Ok(x / y)
    }
}

fn main() {
    match divide(10, 0) {
        Ok(v) => println!("结果是 {}", v),
        Err(e) => println!("发生错误: {}", e),
    }
}
```


## 常见匹配方式

`match` 结构：

```rust
let result = divide(10, 2);
match result {
    Ok(val) => println!("结果: {}", val),
    Err(e) => println!("错误: {}", e),
}
```

`if let`：

```rust
if let Ok(v) = divide(20, 4) {
    println!("成功: {}", v);
}
```

`unwrap()` 和 `expect()`

```rust
let val = divide(6, 2).unwrap(); // 如果是 Err 会 panic
let val2 = divide(6, 2).expect("除法失败"); // 自定义 panic 消息
```


## `Result` 的常用方法大全

| 方法名                 | 作用                    | 示例                      |   |            |
| ------------------- | --------------------- | ----------------------- | - | ---------- |
| `is_ok()`           | 是否为 `Ok`              | `res.is_ok()`           |   |            |
| `is_err()`          | 是否为 `Err`             | `res.is_err()`          |   |            |
| `ok()`              | 转换为 `Option<T>`       | `res.ok()`              |   |            |
| `err()`             | 转换为 `Option<E>`       | `res.err()`             |   |            |
| `unwrap()`          | 获取 `Ok` 的值，失败 panic   | `res.unwrap()`          |   |            |
| `unwrap_or(x)`      | 获取 `Ok`，否则返回 `x`      | `res.unwrap_or(0)`      |   |            |
| `unwrap_or_else(f)` | 错误时调用函数生成默认值          | `res.unwrap_or_else(\| e \| 0)`       |
| `map(f)`            | 对 `Ok` 的值进行转换         | `res.map(\| x \| x \* 2)`  |
| `map_err(f)`        | 对 `Err` 的值进行转换        | `res.map_err(\| e \| e.len())` |
| `and_then(f)`       | 链式调用返回另一个 `Result`    | `res.and_then(\| x \| f(x))`    |
| `or_else(f)`        | `Err` 时返回另一个 `Result` | `res.or_else(\| e \| g(e))`    |

---

## 错误传播运算符 `?`（最常用）

```rust
fn read_number(s: &str) -> Result<i32, std::num::ParseIntError> {
    let n: i32 = s.parse()?; // 自动传播错误
    Ok(n)
}
```

> `?` 会自动将 `Err(e)` 返回出去，简化代码。

相当于：

```rust
let n = match s.parse() {
    Ok(val) => val,
    Err(e) => return Err(e),
};
```

> 要求当前函数也返回 `Result` 类型。


## 自定义错误类型

你可以自定义一个错误类型：

```rust
#[derive(Debug)]
enum MyError {
    ParseError,
    DivideByZero,
}
```

用于统一封装多种错误：

```rust
fn compute(x: &str, y: &str) -> Result<i32, MyError> {
    let a: i32 = x.parse().map_err(|_| MyError::ParseError)?;
    let b: i32 = y.parse().map_err(|_| MyError::ParseError)?;
    if b == 0 {
        Err(MyError::DivideByZero)
    } else {
        Ok(a / b)
    }
}
```


## 与 `Option` 类型的互转

```rust
let r: Result<i32, &str> = Ok(10);

let o: Option<i32> = r.ok();
let r2: Result<i32, &str> = o.ok_or("出错了");
```

---

## 链式操作：`map`、`and_then`

使用 `map`

```rust
let r = Ok(2);
let result = r.map(|x| x * 3); // Ok(6)
```

使用 `and_then`（扁平化嵌套）

```rust
fn plus_one(x: i32) -> Result<i32, String> {
    Ok(x + 1)
}

let r = Ok(2).and_then(plus_one).and_then(plus_one); // Ok(4)
```

---

## 与 `?` 配合的标准库函数（常见陷阱）

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_file() -> Result<String, io::Error> {
    let mut f = File::open("foo.txt")?; // 自动传播错误
    let mut content = String::new();
    f.read_to_string(&mut content)?;
    Ok(content)
}
```


## 在函数签名中使用 `Result`

```rust
fn try_do_something() -> Result<(), String> {
    let cond = true;
    if cond {
        Ok(())
    } else {
        Err("失败了".to_string())
    }
}
```

返回 `Result<(), E>` 表示不关心返回值，只关注是否成功。


## 总结：使用 `Result` 的场景与规范

| 场景            | 推荐写法                    |
| ------------- | ----------------------- |
| 可恢复错误         | 用 `Result<T, E>`        |
| 简化错误传播        | 使用 `?` 运算符              |
| 链式调用多个 Result | 用 `and_then` / `map`    |
| 错误类型统一        | 自定义错误枚举 + `map_err`     |
| 不关心返回值只关心成功与否 | 返回 `Result<(), E>`      |
| 多种错误类型组合处理    | 使用 `Box<dyn Error>` 或枚举 |
