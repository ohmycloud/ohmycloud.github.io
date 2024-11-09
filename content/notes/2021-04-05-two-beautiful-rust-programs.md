+++
title = "Two Beautiful Rust Programs"
date = 2021-04-05
[taxonomies]
  tags = ["rust", "mutex"]
+++

# Two Beautiful Rust Programs

这是一则 Rust 编程语言的短广告，目标是有经验的 `C++` 开发者。作为一则广告，它只能吊起你的胃口，具体内容请参考其他资源。

第一个程序:

```rust
fn main() {
  let mut xs = vec![1, 2, 3];
  let x: &i32 = &xs[0];
  xs.push(92);
  println!("{}", *x);
}
```

这个程序创建了一个 32 位整数的向量(`std::vector<int32_t>`)，接收第一个元素 `x` 的引用，再向向量推送一个数字，然后使用 `x`。这个程序是错误的：扩展向量可能会使对元素的引用无效，而且 `*x` 可能会取消引用一个 danging 指针。

这个程序的好处是它不会被编译。

```
error[E0502]: cannot borrow xs as mutable
    because it is also borrowed as immutable
 --> src/main.rs:4:5

     let x: &i32 = &xs[0];
                    -- immutable borrow occurs here
     xs.push(92);
     ^^^^^^^^^^^ mutable borrow occurs here
     println!(x);
              - immutable borrow later used here
```

Rust 编译器跟踪每块数据的别名状态，并禁止潜在的别名数据的突变。在这个例子中，`x` 和 `xs` 别名了向量在堆中存储的第一个整数。

Rust 不允许做傻事。

第二个程序:

```rust
use crossbeam::scope;
use parking_lot::{Mutex, MutexGuard};

fn main() {
  let mut counter = Mutex::new(0);

  scope(|s| {
    for _ in 0..10 {
      s.spawn(|_| {
        for _ in 0..10 {
          let mut guard: MutexGuard<i32> = counter.lock();
          *guard += 1;
        }
      });
    }
  }).unwrap();

  let total: &mut i32 = counter.get_mut();
  println!("total = {}", *total)
}
```

这个程序创建一个由 mutex 保护的整数计数器，生成10个线程，从每个线程开始将计数器递增10次，并打印出总数。

计数器变量位于堆栈中，这些堆栈数据的指针与其他线程共享。线程必须锁定 mutex 才能进行增量。打印总数时，绕过 mutex 读取计数器，没有任何同步。

这个程序的妙处在于，它的正确性依赖于几位精妙的推理，每一个推理都会被编译器检查。

子线程不会逃离主函数 所以可以从它的堆栈中读取计数器

子线程只通过 mutex 访问 counter。

子线程将在我们从计数器中读出总数而不使用 mutex 时终止。

如果这些约束中的任何一个被破坏，编译器就会拒绝该代码。没有必要使用 `std::shared_ptr` 只是为了防御性地确保内存不会在你的脚下被释放。

Rust 允许做危险的、聪明的、快速的事情，而不用担心引入未定义的行为。

如果你喜欢你所看到的，这里有两本我推荐的书，可以让你更深入地了解 Rust。

原文链接: https://matklad.github.io/2020/07/15/two-beautiful-programs.html
