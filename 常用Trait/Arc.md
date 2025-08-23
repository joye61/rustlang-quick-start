# 为什么需要 `Arc`？

在 **单线程** 下，如果想让多个所有者共享一块数据，我们用 `Rc<T>`。
但如果数据需要被 **多个线程共享**，`Rc<T>` 就不行了，因为它 **不是线程安全的**。

这时就要用 `Arc<T>` —— **Atomic Reference Counted**，即 **原子引用计数智能指针**。
它和 `Rc<T>` 功能几乎相同，只不过内部用 **原子操作** 来修改引用计数，从而保证线程安全。


# `Arc` 的特点

* **线程安全**：可以跨线程共享数据。
* **引用计数**：和 `Rc<T>` 一样，`clone` 只增加计数，不会复制数据。
* **性能稍慢**：因为有原子操作开销。
* **适合多线程场景**：例如线程池、全局共享配置、缓存数据。


# Arc 的基本用法

### 多线程共享不可变数据

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);

    let mut handles = vec![];

    for i in 0..3 {
        let data_cloned = Arc::clone(&data); // 增加引用计数
        let handle = thread::spawn(move || {
            println!("线程 {} sees {:?}", i, data_cloned);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

可能输出：

```
线程 0 sees [1, 2, 3]
线程 2 sees [1, 2, 3]
线程 1 sees [1, 2, 3]
```

这里的 `vec![1, 2, 3]` 被所有线程共享。



# Arc 的限制

**`Arc<T>` 只能保证多线程安全的不可变共享**。

* 它保证 **指针本身的引用计数是线程安全的**；
* 但 **内部的数据不一定线程安全**！

举例：

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);

    let d1 = Arc::clone(&data);
    let d2 = Arc::clone(&data);

    let t1 = thread::spawn(move || {
        // ❌ 错误：不能同时对 Vec 进行可变修改
        // d1.push(4);
        println!("{:?}", d1);
    });

    let t2 = thread::spawn(move || {
        println!("{:?}", d2);
    });

    t1.join().unwrap();
    t2.join().unwrap();
}
```

会报错，因为 `Vec` 不是线程安全的。


# Arc 的常见组合

为了让多个线程共享并 **修改数据**，通常需要搭配以下容器：

### `Arc<Mutex<T>>` —— 多线程可变共享（互斥锁）

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let mut handles = vec![];

    for _ in 0..10 {
        let c = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = c.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("结果: {}", *counter.lock().unwrap());
}
```

输出：

```
结果: 10
```

`Arc` 负责跨线程共享，`Mutex` 负责加锁保证修改安全。



### `Arc<RwLock<T>>` —— 多读单写共享

如果大部分情况是 **读多写少**，用 `RwLock<T>` 更高效。

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let data = Arc::new(RwLock::new(5));

    let d1 = Arc::clone(&data);
    let t1 = thread::spawn(move || {
        let mut num = d1.write().unwrap();
        *num += 10;
    });

    let d2 = Arc::clone(&data);
    let t2 = thread::spawn(move || {
        let num = d2.read().unwrap();
        println!("读取到: {}", *num);
    });

    t1.join().unwrap();
    t2.join().unwrap();
}
```


# Arc vs Rc 对比

| 特性    | `Rc<T>`（单线程）  | `Arc<T>`（多线程）        |
| ----- | ------------- | -------------------- |
| 线程安全  | ❌ 否           | ✅ 是                  |
| 性能    | 更快（无原子操作）     | 稍慢（有原子操作）            |
| 使用场景  | 单线程数据共享       | 多线程数据共享              |
| 可变性支持 | 需配合 `RefCell` | 需配合 `Mutex`/`RwLock` |


# 总结

* **`Arc<T>`**：多线程安全的引用计数智能指针。
* 常用场景：线程间共享不可变数据。
* 如果要修改数据，常搭配：

  * `Arc<Mutex<T>>`（保证互斥修改）；
  * `Arc<RwLock<T>>`（多读单写，性能更好）。
