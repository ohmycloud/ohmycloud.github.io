+++
title = "Stupid Rust Question"
date = 2022-03-22
[taxonomies]
  tags = ["rust", "stupid question"]
+++

# panic! 宏的返回值类型是什么?

```rust
use std::fs::File;

fn main() {
  let f = File::open("hello.txt");
  let f = match f {
    Ok(file) => file,
    Err(error) => {
      panic!("There was a problem opening the file: {:?}", error)
    },    
  };
}
```

既然 match 模式匹配中, 每个分支(arm)的返回值类型必须一样, 那么上面的代码中, `panic!` 返回
的是什么类型呢?

答案是 [never](https://doc.rust-lang.org/core/primitive.never.html) 类型。

# Err(e) => return Err(e) 和 Err(e) => Err(e) 的区别

```rust
/// https://www.reddit.com/r/rust/comments/tibmk3/erre_return_erre_vs_erre_erre/
use std::num::ParseIntError;

fn main() -> Result<(), ParseIntError> {
    let number_str = "10";
    let number = match number_str.parse::<i32>() {
        Ok(number)  => number,
        Err(e) => return Err(e),
    };
    println!("{}", number);
    Ok(())
}
```

既然 match 模式匹配中, 每个分支(arm)的返回值类型必须一样, 那么 `return Err(e)` 返回的是
什么类型呢?

当出现解析错误的时候, `return Err(e)` 直接返回(后面的代码不会被执行), return 表达式的返回
类型也是 never。

# Option.map 和 Option.and_than

Option 类型是容器类型, 里面要么有值(Some(T)), 要么没值(None), 对 Option 进行 map 操作,
返回值要么是 Some(U), 要么是 None

```
pub fn and_then<U, F>(self, f: F) -> Option<U>
where
    F: FnOnce(T) -> Option<U>,
```

Returns None if the option is None, otherwise calls f with the wrapped value and returns the result.

Some languages call this operation flatmap.

Examples

```rust
fn sq(x: u32) -> Option<u32> { Some(x * x) }
fn nope(_: u32) -> Option<u32> { None }

assert_eq!(Some(2).and_then(sq).and_then(sq), Some(16));
assert_eq!(Some(2).and_then(sq).and_then(nope), None);
assert_eq!(Some(2).and_then(nope).and_then(sq), None);
assert_eq!(None.and_then(sq).and_then(sq), None);
```
