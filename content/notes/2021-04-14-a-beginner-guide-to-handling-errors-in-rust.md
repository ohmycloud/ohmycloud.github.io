+++
title = "Rust 中处理错误的初级指南"
date = 2021-04-14
[taxonomies]
  tags = ["rust", "error"]
+++


《Rust 编程语言》中的示例项目对于向新的潜在 Rustaceans 介绍 Rust 的不同方面和特性是非常好的。在这篇文章中，我们将通过扩展《Rust 编程语言》中的 `minigrep` 项目，看看实现更强大的错误处理基础架构的一些不同方法。

`minigrep` 项目在[第12章](https://doc.rust-lang.org/book/ch12-00-an-io-project.html)中介绍，它引导读者构建一个简单版本的 `grep` 命令行工具，这是一个用于搜索文本的工具。例如，你会传入一个查询，你要搜索的文本，以及文本所在的文件名，然后得到包含查询文本的所有行。

这篇文章的目标是用更强大的错误处理模式来扩展本书的 `minigrep` 实现，这样你就能更好地了解 Rust 项目中处理错误的不同方法。

作为参考，你可以在[这里](https://github.com/seanchen1991/error-handling-examples/tree/minigrep-control/examples/minigrep)找到本书的 `minigrep` 版本的最终代码。

## 错误处理用例

当涉及到 Rust 项目的结构时，一个常见的模式是有一个 "库" 的部分和一个 "应用" 的部分，前者是主要的数据结构、函数和逻辑，后者是将库函数联系在一起。

你可以在原始 `minigrep` 代码的文件结构中看到这一点：应用逻辑存在于 `src/bin/main.rs` 文件中，它只是一个薄薄的包裹，包裹着在 `src/lib.rs` 文件中定义的数据结构和函数；主函数所做的就是调用 `minigrep::run`。

这一点很重要，因为取决于我们是在构建一个应用程序还是一个库，会改变我们处理错误的方式。

当涉及到一个应用程序时，最终用户很可能不想知道是什么原因导致了一个错误的琐碎细节。事实上，应用程序的最终用户可能只应该在错误无法恢复的情况下被通知错误。在这种情况下，提供关于为什么发生不可恢复的错误的细节也是有用的，特别是当它与用户输入有关时。如果某种可恢复的错误发生在后台，应用程序的消费者可能不需要知道它。

相反，当涉及到一个库时，最终用户是其他开发人员，他们正在使用该库并在其之上构建一些东西。在这种情况下，我们希望尽可能多地提供关于我们的库中发生的任何错误的相关细节。然后，库的消费者将决定他们想要如何处理这些错误。

那么，当我们的项目中既有库部分又有应用部分时，这两种方法是如何一起发挥作用的呢？`main` 函数执行 `minigrep::run` 函数，并输出结果中出现的任何错误。所以我们大部分的错误处理工作将集中在库部分。

## 浮现库错误

在 `src/lib.rs` 中，我们有两个函数，`Config::new` 和 `run`，它们可能会返回错误。

```rust
impl Config {
    pub fn new(mut args: env::Args) -> Result<Config, &'static str> {
        args.next();

    let query = match args.next() {
        Some(arg) => arg,
        None => return Err("Didn't get a query string"),
    };

    let filename = match args.next() {
        Some(arg) => arg,
        None => return Err("Didn't get a file name"),
    };

    let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

    Ok(Config {
        query,
        filename,
        case_sensitive,
    })
}

pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;

    let results = if config.case_sensitive {
        search(&config.query, &contents)
    } else {
        search_case_insensitive(&config.query, &contents)
    };

    for line in results {
        println!("{}", line);
    }

    Ok(())
}
```

确切有三个地方在返回错误：两个错误发生在 `Config::new` 函数中，该函数返回一个 `Result<Config，&'static str>`。在这种情况下，`Result` 的错误变体是一个静态字符串切片。

在这里，当用户没有提供查询时，我们会返回一个错误。

```rust
let query = match args.next() {
    Some(arg) => arg,
    None => return Err("Didn't get a query string"),
};
```

这里，当用户没有提供文件名时，我们会返回一个错误。

```rust
let filename = match args.next() {
    Some(arg) => arg,
    None => return Err("Didn't get a file name"),
};
```

以这种方式将错误结构化为静态字符串的主要问题是，错误信息并没有被放置在一个中心位置，如果需要的话，我们可以轻松地重构它们。这也使得我们更难在相同类型的错误之间保持错误信息的一致性。

第三种错误发生在 `run` 函数的顶部，它返回一个 `Result<(), Box<dyn Error>>`。在这种情况下，错误变体是一个实现  `Error` [trait](https://doc.rust-lang.org/std/error/trait.Error.html) 的 trait 对象。换句话说，这个函数的错误变体是实现 `Error` trait 的类型的任何实例。

在这里，我们将调用 `fs::read_to_string` 时可能发生的任何错误冒出来。

```rust
let contents = fs::read_to_string(config.filename)?;
```

这适用于调用 `fs::read_to_string` 时可能出现的错误，因为这个函数能够返回多种类型的错误。因此，我们需要一种方法来表示这些不同的可能的错误类型；它们之间的共同点是它们都实现了 `Error` trait！最终，我们要做的是定义所有这些错误类型。

最终，我们要做的是在一个中心位置定义所有这些不同类型的错误，并让它们都成为单一类型的变体。

## 在一个中心类型中定义错误变种

我们将创建一个新的 `src/error.rs` 文件，并定义一个枚举 `AppError`，并在此过程中派生出 `Debug` trait，以便我们在需要时可以得到一个调试表示。我们将为这个枚举的每一个变体命名，使它们恰当地代表三种类型的错误。

```rust
#[derive(Debug)]
pub enum AppError {
    MissingQuery,
    MissingFilename,
    ConfigLoad,
}
```

第三个变体，`ConfigLoad`，映射到 `Config::run` 函数中调用 `fs::read_to_string` 时可能出现的错误。乍一看，这似乎有点不妥，因为如果该函数出现错误，那就是在读取提供的配置文件时出现了某种I/O问题。那么我们为什么不把它命名为 `IOError` 或者类似的东西呢？

在这种情况下，由于我们是将一个标准库函数的错误浮出水面，所以描述浮出水面的错误是如何影响它的，而不是简单地重申它，这与我们的应用更相关。当 `fs::read_to_string` 发生错误时，会阻止我们的 `Config` 加载，所以这就是为什么我们把它命名为 `ConfigLoad`。

现在我们有了这个类型，我们需要更新代码中所有返回错误的地方以利用这个 `AppError` 枚举。

## 返回 `AppError` 的变体

在我们的 `src/lib.rs` 文件的顶部，我们需要声明我们的错误模块，并将 `error::AppError` 带入作用域。

```rust
mod error;

use error::AppError;
```

在我们的 `Config::new` 函数中，我们需要更新我们作为错误返回静态字符串切片的地方，以及函数本身的返回类型。

```rust
- pub fn new(mut args: env::Args) -> Result<Config, &'static str>
+ pub fn new(mut args: env::Args) -> Result<Config, AppError>
    // --snip--

    let query = match args.next() {
        Some(arg) => arg,
-       None => return Err("Didn't get a query string"),
+       None => return Err(AppError::MissingQuery),
    };

    let filename = match args.next() {
        Some(arg) => arg,
-       None => return Err("Didn't get a file name"),
+       None => return Err(AppError::MissingFilename),
    };

    // --snip--
```

运行函数中的第三个错误，只需要我们更新它的返回类型，因为 `?` 操作符已经负责将错误冒出来，并在发生时返回。

```rust
- pub fn run(config: Config) -> Result<(), Box<dyn Error>>
+ pub fn run(config: Config) -> Result<(), AppError>
```

好了，现在我们正在使用我们的错误变体，一旦发生，这些错误变体将被浮现到我们的 `main` 函数中并打印出来。但是我们不再有之前定义的实际错误信息了！我们可以用 `thiserror` 注释错误变体。

## 用 `thiserror` 注释错误变体

`thiserror` [crate](https://docs.rs/thiserror/1.0.24/thiserror/) 是一个常用的工具，它提供了一种符合人体工程学的方式来格式化 Rust 库中的错误信息。

它允许我们在 `AppError` 枚举中用我们希望显示给最终用户的实际错误信息来注解每个变体。

让我们在 Cargo.toml 中添加它作为依赖。

```toml
[dependencies]
thiserror = "1"
```

在 `src/error.rs` 中，我们将把 `thiserror::Error` trait 带入作用域，并让我们的 `AppError` 类型派生它。我们需要派生这个 trait，以便用 `#[error]` 块来注解每个枚举变量。现在我们指定我们希望为每个特定变量显示的错误信息。

```rust
+ use std::io;

+ use thiserror::Error;

- #[derive(Debug)]
+ #[derive(Debug, Error)]
pub enum AppError {
+   #[error("Didn't get a query string")]
    MissingQuery,
+   #[error("Didn't get a file name")]
    MissingFilename,
+   #[error("Could not load config")]
    ConfigLoad {
+       #[from]
+       source: io::Error,
+   }
}
```

`ConfigLoad` 变体中增加了什么额外的东西？由于 `ConfigLoad` 错误只有在调用 `fs::read_to_string` 出现底层错误时才会发生，所以 `ConfigLoad` 变体实际上做的是围绕底层I/O错误提供额外的上下文。

`thiserror` 允许我们通过用 `#[from]` 来注解一个低级错误，以将源码转换为我们自制的错误类型，从而将其包裹在额外的上下文中。这样一来，当一个I/O错误发生时（比如我们指定了一个要搜索的文件，但实际上并不存在），我们就会得到这样一个错误。

```
Could not load config: Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

如果没有它，产生的错误信息看起来像这样。

```
Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

对于我们库的消费者来说，要想找出这个错误的来源是比较困难的，额外的上下文帮助很大。

你可以在[这里](https://github.com/seanchen1991/error-handling-examples/tree/minigrep-thiserror/examples/minigrep)找到使用这个错误的 `minigrep` 版本。

## 更加手动的方法

现在，我们将换个角度，看看如何在不将其作为依赖的情况下，实现与 `thiserror` 相同的结果。

在引擎盖下，`thiserror` 用程序宏执行了一些魔法，这对编译速度有明显的影响。在 `minigrep` 的情况下，我们的错误变体很少，而且项目也很小，所以依赖 `thiserror` 并不会增加多少编译时间，但是在一个更大更复杂的项目中，这可能是一个考虑因素。

所以在这一点上，我们将把这篇文章撕掉，换成我们自己的手动实现来结束这篇文章。走这条路的好处是，我们只需要修改 `src/error.rs` 文件就可以实现所有必要的改变（当然，除了从我们的 Cargo.toml 中删除 thiserror 之外）。

```toml
[dependencies]
- thiserror = "1"
```

让我们删除所有 `thiserror` 提供给我们的注释。我们还将用 `std::error::Error` trait 替换 `thiserror::Error` trait。

```rust
- use thiserror::Error;
+ use std::error::Error;

- #[derive(Debug, Error)]
+ #[derive(Error)]
pub enum AppError {
-   #[error("Didn't get a query string")]
    MissingQuery,
-   #[error("Didn't get a file name")]
    MissingFilename,
-   #[error("Could not load config")]
    ConfigLoad {
-      #[from]
       source: io::Error,
    }
}
```

为了恢复我们刚刚擦除的所有功能，我们需要做三件事。

1. 为 `AppError` 实现 `Display` trait，这样我们的错误变体就可以显示给用户了。
2. 为 `AppError` 实现 `Error` trait。这个 trait 代表了对错误类型的基本期望，即它们实现了 `Display` 和 `Debug`，再加上获取错误底层源或原因的能力。
3. 为 `AppError` 实现 `From<io::Error>`。这是必要的，这样我们就可以将从 `fs::read_to_string` 返回的I/O错误转换为 `AppError` 的实例。

这里是我们对 `AppError` 的 `Display` trait 的实现。它将每个错误变量映射为一个字符串，并将其写入到 `Display`  formatter 中。

```rust
use std::fmt;

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::MissingQuery => f.write_str("Didn't get a query string"),
            Self::MissingFilename => f.write_str("Didn't get a file name"),
            Self::ConfigLoad { source } => write!(f, "Could not load config: {}", source),
        }
    }
}
```

而这就是我们对 `Error` trait 的实现。要实现的主要方法是 `Error::source` 方法，它的目的是提供错误源的信息。对于我们的 `AppError` 类型，只有 `ConfigLoad` 会暴露任何底层源信息，即调用 `fs::read_to_string` 可能发生的I/O错误。在其他错误变体的情况下，没有底层的源信息需要暴露。

```rust
use std::error;

impl error::Error for AppError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            Self::ConfigLoad { source } => Some(source),
            _ => None,
        }
    }
}
```

返回类型的 `&(dyn Error + 'static')` 部分类似于我们之前看到的 `Box<dyn Error>` trait 对象。这里的主要区别是，trait 对象是在一个不可变的引用后面，而不是 `Box` 指针。这里的 `'static` lifetime 意味着 trait 对象本身只包含拥有的值，也就是说，它内部不存储任何引用。这是必要的，以便让编译器确信这里没有悬空指针的机会。

最后，我们需要一种将 `io::Error` 转换为 `AppError` 的方法。我们将通过为 `AppError for From<io::error>` 来实现。

```rust
impl From<io::Error> for AppError {
    fn from(source: io::Error) -> Self {
        Self::ConfigLoad { source }
    }
}
```

这个没什么好说的。如果我们得到一个 `io::Error`，我们要做的就是将其转换为 `AppError`，并将其封装在 `ConfigLoad` 变体中。

这就是全部了，伙计们 你可以在[这里](https://github.com/seanchen1991/error-handling-examples/tree/main/examples/minigrep)找到这个版本的 `minigrep` 实现。

## 总结

最后，我们讨论了《Rust编程语言》一书中介绍的原始 `minigrep` 实现在错误处理方面是如何有点欠缺的，以及如何考虑不同的错误处理用例。

从那里，我们展示了如何使用 `thiserror` crate 将所有可能的错误变体集中到一个类型中。

最后，我们剥开了 `thiserror` 提供的外衣，展示了如何手动复制同样的功能。

希望大家能从这篇文章中学到一些东西!

原文链接: https://dev.to/seanchen1991/a-beginner-s-guide-to-handling-errors-in-rust-40k2
