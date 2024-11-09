+++
title = "Std Error in Rust"
date = 2021-04-05
[taxonomies]
  tags = ["rust", "error"]
+++


# 学习 std::io::Error

在这篇文章中，我们将剖析 Rust 标准库中 std::io::Error 类型的实现。相关代码在这里：`library/std/src/io/error.rs`。

你可以把这篇文章看成是其中之一。

- 一个标准库的特定位的研究
- 一个高级错误管理指南
- 一个漂亮的 API 设计案例

文章要求基本熟悉 Rust 错误处理。

在设计一个用于 `Result<T，E>` 的 Error 类型时，主要的问题是"如何使用这个错误？"。通常，以下情况之一为真。

- 错误被程序化处理。消费者检查错误，所以它的内部结构需要在合理的程度上暴露出来。
- 错误被传播并显示给用户。消费者不检查 `fmt::Display` 之外的错误；所以它的内部结构可以被封装。

请注意，暴露实现细节和封装细节之间存在紧张关系。实现第一种情况的常见反模式是定义一个厨房水槽枚举。

```rust
pub enum Error {
  Tokio(tokio::io::Error),
  ConnectionDiscovery {
    path: PathBuf,
    reason: String,
    stderr: String,
  },
  Deserialize {
    source: serde_json::Error,
    data: String,
  },
  ...,
  Generic(String),
}
```

这种方法有很多问题。

首先，从底层库中暴露错误会使它们成为你的公共 API 的一部分。在你的依赖关系中的主要 semver bump 会要求你也做一个新的主要版本。

其次，它将所有的实现细节都固定下来。例如，如果你注意到 ConnectionDiscovery 的大小是巨大的，那么将这个变体装箱将是一个突破性的变化。

第三，它通常表明了一个更大的设计问题。厨房水槽错误将不同的故障模式打包成一种类型。但是，如果故障模式差异很大，处理起来可能就不合理了! 这说明情况看起来更像案例二。

错误厨房水槽病的一个经常有效的治疗方法是将错误推送给调用者的模式。

考虑这个例子:

```rust
fn my_function() -> Result<i32, MyError> {
  let thing = dep_function()?;
  ...
  Ok(92)
}
```

`my_function` 调用 `dep_function`，所以 `MyError` 应该可以从 `DepError` 转换过来。更好的写法可能是这样的。

```rust
fn my_function(thing: DepThing) -> Result<i32, MyError> {
  ...
  Ok(92)
}
```

在这个版本中，调用者被迫调用 `dep_function` 并处理其错误。这就用更多的类型交换了更多的类型安全。`MyError` 和 `DepError` 现在是不同的类型，调用者可以分别处理它们。如果 `DepError` 是 `MyError` 的变体，那么就需要进行运行时匹配。

这个想法的一个极端版本是 `sans-io` 编程。大多数错误来自于 IO；如果你把所有的 IO 推给调用者，你就可以跳过大部分的错误处理。

无论枚举方法多么糟糕，它确实实现了第一种情况的最大可检查性。

以传播为中心的第二种情况下的错误管理，通常是通过使用盒状特质对象来处理。像 `Box<dyn std::error::Error>` 这样的类型可以从任何具体的错误中构造出来，可以通过 `Display` 打印出来，并且仍然可以选择通过动态下传来暴露底层错误。`Anyhow` crate 就是这种风格的一个很好的例子。

`std::io::Error` 的例子很有趣，因为它想同时具备上述两种风格。

- 这是 std，所以封装和面向未来是最重要的。
- 来自操作系统的 IO 错误往往可以被处理（比如 EWOULDBLOCK）。
- 对于系统编程语言来说，准确地暴露底层 OS 错误是很重要的。
- 未来潜在的操作系统错误集是没有限制的。
- `io::Error` 也是一种词汇类型，应该可以表示一些不完全的 os 错误。例如，Rust Paths 可以包含内部的0字节，打开这样的路径应该在进行 `syscall` 之前返回一个 `io::Error`。

下面是 `std::io::Error` 的样子。

```rust
pub struct Error {
  repr: Repr,
}

enum Repr {
  Os(i32),
  Simple(ErrorKind),
  Custom(Box<Custom>),
}

struct Custom {
  kind: ErrorKind,
  error: Box<dyn error::Error + Send + Sync>,
}
```

首先要注意的是，它内部是一个枚举，但这是一个隐藏得很好的实现细节。为了允许检查和处理各种错误条件，有一个单独的公共无字段种类枚举。

```rust
#[derive(Clone, Copy)]
#[non_exhaustive]
pub enum ErrorKind {
  NotFound,
  PermissionDenied,
  Interrupted,
  ...
  Other,
}

impl Error {
  pub fn kind(&self) -> ErrorKind {
    match &self.repr {
      Repr::Os(code) => sys::decode_error_kind(*code),
      Repr::Custom(c) => c.kind,
      Repr::Simple(kind) => *kind,
    }
  }
}
```

虽然 ErrorKind 和 Repr 都是枚举，但公开暴露 ErrorKind 就没那么可怕了。一个 `#[non_exhaustive]Copy` 无字段枚举的设计空间是一个点 - 没有合理的替代方案或兼容性隐患。

有些 `io::Errors` 只是原始的操作系统错误代码。

```rust
impl Error {
  pub fn from_raw_os_error(code: i32) -> Error {
    Error { repr: Repr::Os(code) }
  }
  pub fn raw_os_error(&self) -> Option<i32> {
    match self.repr {
      Repr::Os(i) => Some(i),
      Repr::Custom(..) => None,
      Repr::Simple(..) => None,
    }
  }
}
```

特定平台的 `sys::decode_error_kind` 函数负责将错误代码映射到 `ErrorKind` 枚举。所有这些都意味着代码可以通过检查 `.kind()` 来跨平台处理错误类别。然而，如果需要以一种依赖于操作系统的方式处理一个非常特殊的错误代码，这也是可能的。API 小心翼翼地提供了一个方便的抽象，而没有抽象掉重要的低级细节。

一个 `std::io::Error` 也可以从一个 `ErrorKind` 中构造出来。

```rust
impl From<ErrorKind> for Error {
  fn from(kind: ErrorKind) -> Error {
    Error { repr: Repr::Simple(kind) }
  }
}
```

这提供了跨平台访问错误代码风格的错误处理。如果你需要尽可能快的错误，这很方便。

最后，还有第三种完全自定义的变体表示。

```rust
impl Error {
  pub fn new<E>(kind: ErrorKind, error: E) -> Error
  where
    E: Into<Box<dyn error::Error + Send + Sync>>,
  {
    Self::_new(kind, error.into())
  }

  fn _new(
    kind: ErrorKind,
    error: Box<dyn error::Error + Send + Sync>,
  ) -> Error {
    Error {
      repr: Repr::Custom(Box::new(Custom { kind, error })),
    }
  }

  pub fn get_ref(
    &self,
  ) -> Option<&(dyn error::Error + Send + Sync + 'static)> {
    match &self.repr {
      Repr::Os(..) => None,
      Repr::Simple(..) => None,
      Repr::Custom(c) => Some(&*c.error),
    }
  }

  pub fn into_inner(
    self,
  ) -> Option<Box<dyn error::Error + Send + Sync>> {
    match self.repr {
      Repr::Os(..) => None,
      Repr::Simple(..) => None,
      Repr::Custom(c) => Some(c.error),
    }
  }
}
```

需要注意的地方。

- 通用的 `new` 函数委托给单态的 `_new` 函数。这改善了编译时间，因为在单态化过程中需要重复的代码更少。我认为这也改善了一些运行时：`_new` 函数没有被标记为内联，所以会在调用处产生一个函数调用。这是好的，因为错误构造是冷路径，节省指令缓存是受欢迎的。

- 自定义变体被框住了 - 这是为了让整体 `size_of` 更小。错误的 `on-the-stack` 大小是很重要的：即使没有错误，你也要为此付出代价!

这两种类型都是指"静态错误"。

```rust
type A =   &(dyn error::Error + Send + Sync + 'static);
type B = Box<dyn error::Error + Send + Sync>
```

在 `dyn Trait + '_` 中，`'_` 被省略为 `'static`，除非 trait 对象是在引用后面，在这种情况下，它被省略为 `&'a dyn Trait + 'a`。

`get_ref`, `get_mut` 和 `into_inner` 提供了对底层错误的完全访问。类似于 `os_error` 的情况，抽象模糊了细节，但也提供了钩子来获取底层数据的原样。

同样，Display 的实现揭示了内部表示的最重要细节。

```rust
impl fmt::Display for Error {
  fn fmt(&self, fmt: &mut fmt::Formatter<'_>) -> fmt::Result {
    match &self.repr {
      Repr::Os(code) => {
        let detail = sys::os::error_string(*code);
        write!(fmt, "{} (os error {})", detail, code)
      }
      Repr::Simple(kind) => write!(fmt, "{}", kind.as_str()),
      Repr::Custom(c) => c.error.fmt(fmt),
    }
  }
}
```

综上所述，std::io::Error:

- 封装了它的内部表现形式，并通过框定大的枚举变体来优化它。
- 通过 `ErrorKind` 模式提供了一种方便的方法来处理基于类别的错误。
- 完全暴露底层操作系统的错误（如果有的话）。

可以透明地包裹任何其他错误类型。

最后一点意味着 `io::Error` 可以用于临时错误，因为 `&str` 和 String 可以转换为 `Box<dyn std::error::Error>`。

```rust
io::Error::new(io::ErrorKind::Other, "something went wrong")
```

它也可以作为 anyhow 的简单替换。我想一些库可能会用这个来简化他们的错误处理。

```rust
io::Error::new(io::ErrorKind::InvalidData, my_specific_error)
```

例如，`serde_json` 提供了以下方法。

```rust
fn from_reader<R, T>(rdr: R) -> Result<T, serde_json::Error>
where
  R: Read,
  T: DeserializeOwned,
```

读取会因为 `io::Error` 而失败，所以 `serde_json::Error` 需要能够在内部表示 `io::Error`。我认为这是倒退的 (但我不知道整个上下文，如果被证明是错的我会很高兴！)，签名应该是这样的。

```rust
fn from_reader<R, T>(rdr: R) -> Result<T, io::Error>
where
  R: Read,
  T: DeserializeOwned,
```

那么，`serde_json::Error` 就不会有 `Io` 的变体，而会以 `InvalidData` 的形式被藏到 `io::Error` 中。
补遗, 2021-01-25

重新阅读这篇文章，我现在认为正确的返回类型应该是：

```rust
fn from_reader<R, T>(
  rdr: R,
) -> Result<Result<T, serde_json::Error>, io::Error>
where
  R: Read,
  T: DeserializeOwned,
```

这迫使 IO 和反序列化错误分开处理，这在这种情况下是有意义的。IO 错误可能是程序领域之外的硬件/环境问题，而序列化错误很可能是系统中的某个错误。

我认为 `std::io::Error` 是一个非常了不起的类型，它能够在没有太多妥协的情况下为许多不同的用例服务。但我们是否可以做得更好呢？

`std::io::Error` 的首要问题是，当一个文件系统操作失败时，你不知道它是为哪个路径失败的。这是可以理解的 - Rust 是一种系统语言，所以它不应该比 OS 原生提供的东西增加多少脂肪。OS 返回的是一个整数返回代码，而将其与一个堆分配的 `PathBuf` 耦合在一起，可能是一个不可接受的开销!

我很惊讶地得知，事实上，`std` 对每一个与路径相关的系统调用都会进行分配。

它需要以某种形式存在。`OS API` 需要在字符串的结尾有一个不幸的零字节. 但我想知道对短路径使用堆栈分配的缓冲区是否有意义。可能不会 - 路径通常不会那么短，而且现代分配器能有效地处理瞬时分配。

我不知道这里有什么明显的好办法。一个选择是在编译时（一旦我们得到 `std-aware cargo`）或运行时（`a-la RUST_BACKTRACE`）添加开关，以堆分配所有与路径相关的 IO 错误。一个类似形的问题是 `io::Error` 不携带 backtrace。

另一个问题是，`std::io::Error` 的效率不高。

它的体积是相当大的。

```rust
assert_eq!(size_of::<io::Error>(), 2 * size_of::<usize>());
```

对于自定义的情况，会产生双重的间接和分配。

```rust
    enum Repr {
      Os(i32),
      Simple(ErrorKind),
      // First Box :|
      Custom(Box<Custom>),
    }

    struct Custom {
      kind: ErrorKind,
      // Second Box :(
      error: Box<dyn error::Error + Send + Sync>,
    }
```

我想我们现在可以解决这个问题了

首先，我们可以通过使用一个瘦的特质对象来摆脱双重内向性，比如失败或 anyhow。现在 GlobalAlloc 已经存在，这是一个比较直接的实现。

其次，我们可以利用指针是对齐的这一事实，将 Os 和 Simple 变体都用最小的有效位集储藏到 usize 中。我认为我们甚至可以发挥创意，使用第二个最小有意义的位，把第一个位留作小众。这样一来，即使是像 `io::Result<i32>` 这样的东西，也可以是指针大小的!

本篇文章到此结束。下一次你要为你的库设计一个错误类型的时候，花点时间去看看 `std::io::Error` 的源头，你可能会发现一些值得偷的东西。

讨论在 /r/rust.Net 上进行。

## 额外的谜题

看看实现中的这一行。

```rust
impl fmt::Display for Error {
  fn fmt(&self, fmt: &mut fmt::Formatter<'_>) -> fmt::Result {
    match &self.repr {
      Repr::Os(code) => {
        let detail = sys::os::error_string(*code);
        write!(fmt, "{} (os error {})", detail, code)
      }
      Repr::Simple(kind) => write!(fmt, "{}", kind.as_str()),
      Repr::Custom(c) => c.error.fmt(fmt),
    }
  }
}
```

原文链接: https://matklad.github.io/2020/10/15/study-of-std-io-error.html
