## Mutex 是什么

* **Mutex**（Mutual Exclusion）中文意思是“互斥”，就是保证 **同一时间只能有一个线程访问某块数据**。
* 在多线程程序中，如果多个线程同时修改同一个数据，可能会出现 **数据竞争（Data Race）**，导致程序不稳定。
* `Mutex` 可以避免数据竞争，是 **多线程安全的共享数据容器**。

> 可以想象成：一把钥匙和一个房间，只有拿到钥匙的线程可以进入房间操作数据，其他线程必须等待。


## 基本用法

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5); // 创建一个 Mutex，保护内部数据 5

    {
        let mut num = m.lock().unwrap(); // 获取锁
        *num += 1; // 修改数据
    } // MutexGuard 离开作用域，锁自动释放

    println!("m = {:?}", m); // m = Mutex { data: 6 }
}
```

**说明：**

1. `Mutex::new(data)` 创建互斥锁，保护 `data`。
2. `lock()` 获取锁，返回 `MutexGuard`。
3. `MutexGuard` 是智能指针，解引用后可访问内部数据。
4. 离开作用域时，锁自动释放。


## 多线程共享数据

多线程使用 Mutex 时，通常配合 `Arc`（原子引用计数）共享数据：

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap()); // Result: 10
}
```

**说明：**

* `Arc` 让多个线程可以共享同一个 Mutex。
* `Mutex` 保证同一时间只有一个线程访问数据。
* 最终结果是 10，说明操作是线程安全的。


## 常用方法和概念

| 方法 / 类型            | 作用                      |
| ------------------ | ----------------------- |
| `Mutex::new(data)` | 创建互斥锁                   |
| `lock()`           | 获取锁，返回 `MutexGuard`     |
| `MutexGuard`       | 智能指针，解引用访问数据，离开作用域自动释放锁 |
| `Arc<Mutex<T>>`    | 多线程共享可变数据的组合            |


## 注意事项

1. **避免死锁**

   * 不要在持有锁时请求其他锁。
   * 不要持有锁做耗时操作。

2. **PoisonError**

   * 如果线程在持锁时 panic，该锁会被标记为“poisoned”。
   * 再次 `lock()` 会返回错误，需要处理 `PoisonError`。

3. **性能考虑**

   * Mutex 是阻塞的，频繁锁/解锁可能影响性能。
   * 小型数据修改可考虑使用 `Atomic` 类型代替。


## 总结

* **Mutex = 线程安全的可变数据容器**
* **锁机制 = 同一时间只有一个线程访问数据**
* **MutexGuard = 智能指针，作用域结束自动释放锁**
* 多线程共享数据时常用 **Arc\<Mutex\<T\>\>**

