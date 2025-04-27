+++
title = "同级目录之间的模块导入"
date = 2025-01-03
[taxonomies]
  tags = ["rust", "crate"]
+++

最近在改造一个使用 [nom](https://docs.rs/nom/latest/nom/)解析 Curl 命令行指令的项目 [nomcurl](https://github.com/YuniqueUnic/nomcurl), 遇到了一个问题, `curl` 目录下的 `parser` 引用了 `url` 目录下的 `parser` 中的 `CurlURL` 结构体, 编辑器一直提示 `rustc: failed to resolve: unresolved import`

在 `curl/parser.rs` 中, 我导入了另外一个目录中的 `CurlURL` 结构体:

```rust
use crate::url::parser::CurlURL;
```

项目的目录结构如下(精简后):

```
├── src
│   ├── curl
│   │   ├── mod.rs
│   │   ├── parser.rs
│   ├── lib.rs
│   ├── main.rs
│   └── url
│       ├── mod.rs
│       ├── parser.rs
│       └── protocol.rs
```

其中:

`url/parser.rs` 中定义的 `CurlURL` 结构是这样的:

```rs
#[derive(Debug, PartialEq)]
pub struct CurlURL<'a> {
    pub schema: Schema,
    pub authority: Option<Authority<'a>>,
    pub path: &'a str,
    pub uri: &'a str,
    pub queries: Vec<QueryString<'a>>,
    pub fragment: Option<&'a str>,
}
```

`url/mod.rs` 内容如下:

```rs
pub mod parser;
pub mod protocol;
```

`lib.rs` 内容如下:

```rs
pub mod curl;
pub mod url;
```

`curl/mod.rs` 内容如下:

```rs
pub mod parser;
```

编译器还是报 `unresolved import`。解决办法是在 `main.rs` 中添加：

```rs
pub mod url;
```

原因参考 [claude.ai](https://claude.ai/) 给出的解答: https://claude.ai/chat/0d32b24c-8903-4f39-93a4-992da354def5
