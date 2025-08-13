# 支持重载的 trait 列表

我们把 Rust 语言的重载理解为**通过实现某个标准库提供的 trait，让运算符或特定语法对自定义类型生效**，Rust 标准库里有一整套这样的 trait，分布在几个主要模块里。

下面整理了一份**Rust 标准库可重载 trait 列表**：

---

## 1. `std::ops` —— 运算符重载

这些 trait 让你能重载算术、位运算、索引、解引用等操作。

| Trait                     | 对应运算符 / 语法   | 方法签名                              |
| ------------------------- | ------------ | --------------------------------- |
| `Add` / `AddAssign`       | `+` / `+=`   | `fn add(self, rhs) -> Output`     |
| `Sub` / `SubAssign`       | `-` / `-=`   | `fn sub(self, rhs) -> Output`     |
| `Mul` / `MulAssign`       | `*` / `*=`   | `fn mul(self, rhs) -> Output`     |
| `Div` / `DivAssign`       | `/` / `/=`   | `fn div(self, rhs) -> Output`     |
| `Rem` / `RemAssign`       | `%` / `%=`   | `fn rem(self, rhs) -> Output`     |
| `Neg`                     | `-x`（一元负号）   | `fn neg(self) -> Output`          |
| `Not`                     | `!x`         | `fn not(self) -> Output`          |
| `BitAnd` / `BitAndAssign` | `&` / `&=`   | `fn bitand(self, rhs) -> Output`  |
| `BitOr` / `BitOrAssign`   | `\|` / `\|=` | `fn bitor(self, rhs) -> Output`   |
| `BitXor` / `BitXorAssign` | `^` / `^=`   | `fn bitxor(self, rhs) -> Output`  |
| `Shl` / `ShlAssign`       | `<<` / `<<=` | `fn shl(self, rhs) -> Output`     |
| `Shr` / `ShrAssign`       | `>>` / `>>=` | `fn shr(self, rhs) -> Output`     |
| `Index` / `IndexMut`      | `a[b]`       | `fn index(&self, idx) -> &Output` |
| `Deref` / `DerefMut`      | `*x`（解引用）    | `fn deref(&self) -> &Target`      |

---

## 2. `std::cmp` —— 比较运算符重载

这些 trait 让你能重载 `==`, `<` 等比较运算符。

| Trait        | 对应运算符 / 语法         | 方法签名                                            |
| ------------ | ----------------------- | -------------------------------------------------- |
| `PartialEq`  | `==` / `!=`             | `fn eq(&self, other) -> bool`                      |
| `Eq`         | 标记完全等价性            | 空 trait，需配合 `PartialEq`                         |
| `PartialOrd` | `<` / `<=` / `>` / `>=` | `fn partial_cmp(&self, other) -> Option<Ordering>` |
| `Ord`        | 完全有序比较              | `fn cmp(&self, other) -> Ordering`                 |

---

## 3. `std::fmt` —— 自定义格式化（`println!` 等）

严格来说不是运算符，但它重载了**格式化占位符的行为**。

| Trait      | 对应占位符  | 方法签名                                  |
| ---------- | ------ | -------------------------------------------- |
| `Debug`    | `{:?}` | `fn fmt(&self, f: &mut Formatter) -> Result` |
| `Display`  | `{}`   | 同上                                         |
| `Binary`   | `{:b}` | 同上                                         |
| `Octal`    | `{:o}` | 同上                                         |
| `LowerHex` | `{:x}` | 同上                                         |
| `UpperHex` | `{:X}` | 同上                                         |
| `LowerExp` | `{:e}` | 同上                                         |
| `UpperExp` | `{:E}` | 同上                                         |
| `Pointer`  | `{:p}` | 同上                                         |

---

## 4. `std::str` / `std::string` / `std::borrow` —— 隐式类型转换

不是运算符，但会影响 `&str` / `String` / `&T` 等的自动转换。

| Trait                  | 作用          | 方法签名                            |
| ---------------------- | ----------- | ----------------------------------- |
| `From` / `Into`        | 类型转换        | `fn from(T) -> Self`                |
| `TryFrom` / `TryInto`  | 可失败类型转换     | `fn try_from(T) -> Result<Self, E>` |
| `AsRef` / `AsMut`      | 借用转换        | `fn as_ref(&self) -> &U`            |
| `Borrow` / `BorrowMut` | 借用同一类型的不同视图 | 类似 AsRef，但用于集合键等        |

---

## 5. `std::iter` —— 迭代器行为

实现这些 trait 可以让你的类型支持 `for ... in` 循环和迭代方法链。

| Trait                 | 作用                  |
| --------------------- | ------------------- |
| `IntoIterator`        | 支持 `for x in value` |
| `Iterator`            | 提供 `.next()` 迭代能力   |
| `DoubleEndedIterator` | 双端迭代                |
| `ExactSizeIterator`   | 已知长度迭代器             |

---

## 6. 其它零散可重载 trait

* `std::hash::Hash` —— 让类型可放入 `HashMap` / `HashSet`
* `std::default::Default` —— `T::default()`
* `std::clone::Clone` / `std::marker::Copy` —— 复制语义
* `std::marker::Drop` —— 析构时运行自定义逻辑
* `std::convert::FromIterator` / `std::iter::FromIterator` —— `.collect()` 的目标类型构造规则
* `std::marker::Unpin` / `Send` / `Sync` —— 并发和 Pin 行为标记（特殊用途）

---

**总结**
Rust 没有 C++ 那种“运算符重载到处都是”的自由，但标准库通过一组精心设计的 trait，把所有常用的运算符、比较、索引、解引用、格式化、类型转换等行为统一成了**trait 重载机制**。
基本规律：

1. **算术/逻辑运算符** → `std::ops`
2. **比较运算符** → `std::cmp`
3. **输出格式化** → `std::fmt`
4. **类型转换** → `std::convert` / `std::borrow`
5. **迭代相关** → `std::iter`
6. **其它特殊功能** → 对应模块的 trait

---
