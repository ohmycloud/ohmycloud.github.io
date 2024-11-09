+++
title = "Parser API - 解析 INI"
date = 2021-01-19
[taxonomies]
  tags = ["rust", "grammar", "parser", "ini"]
+++

## 例子: INI

INI(initialization 的简称)文件是简单的配置文件。由于没有标准的格式，我们将编写一个能够解析这个例子文件的程序。

```ini
username = noha
password = plain_text
salt = NaCl

[server_1]
interface=eth0
ip=127.0.0.1
document_root=/var/www/example.org

[empty_section]

[second_server]
document_root=/var/www/example.com
ip=
interface=eth1
```

每一行都包含一个键和值，中间用等号隔开；或者包含一个用方括号括起来的章节名；或者是空白，没有任何意义。

每当出现一个节名，下面的键和值就属于该节，直到下一个节名。文件开头的键值对属于一个隐式的 "空"节。

## 编写 grammar

首先使用 Cargo [初始化一个新项目](https://pest.rs/book/examples/csv.html#setup)，添加依赖关系 `pest = "2.0"` 和  `pest_derive = "2.0"`。创建一个新文件 `src/ini.pest` 来保存 grammar。

我们文件中感兴趣的文本 - `username`、`/var/www/example.org` 等 - 只由几个字符组成。让我们制定一个规则来识别该集合中的单个字符。内置的规则 `ASCII_ALPHANUMERIC` 是表示任何大写或小写 ASCII 字母或任何数字的快捷方式。

```
char = { ASCII_ALPHANUMERIC | "." | "_" | "/" }
```

节名和属性键不能为空，但属性值可以为空（如上文中的 `ip=` 行）。也就是说，前者由一个或多个字符组成，`char+`; 后者由零或多个字符组成，`char*`。我们将其含义分为两条规则。

```
name = { char+ }
value = { char* }
```

现在很容易表达这两种输入行。

```
section = { "[" ~ name ~ "]" }
property = { name ~ "=" ~ value }
```

最后，我们需要一个规则来表示整个输入文件。表达式 `(section | property)?` 匹配 `section`、`property`，否则什么也不匹配。使用内置规则 `NEWLINE` 来匹配行尾。

```
file = {
    SOI ~
    ((section | property)? ~ NEWLINE)* ~
    EOI
}
```

要将解析器编译成 Rust，我们需要在 `src/main.rs` 中添加以下内容。

```rust
extern crate pest;
#[macro_use]
extern crate pest_derive;

use pest::Parser;

#[derive(Parser)]
#[grammar = "ini.pest"]
pub struct INIParser;
```

## 程序初始化

现在我们可以读取文件，并用 `pest` 进行解析。

```rust
use std::collections::HashMap;
use std::fs;

fn main() {
    let unparsed_file = fs::read_to_string("config.ini").expect("cannot read file");

    let file = INIParser::parse(Rule::file, &unparsed_file)
        .expect("unsuccessful parse") // unwrap the parse result
        .next().unwrap(); // get and unwrap the `file` rule; never fails

    // ...
}
```

我们将使用嵌套的 `HashMap` 来表达属性列表。外层哈希 map 将以章节名称作为键，以章节内容（内部哈希 map）作为值。每个内部哈希 map 将有属性键和属性值。例如，要访问 `server_1` 的 `document_root`，我们可以写 `properties["server_1"]["document_root"]`。隐含的 "空"节将由常规部分表示，名称为空字符串 `""`，这样 `properties[""]["salt"]` 就是有效的。

```rust
fn main() {
    // ...

    let mut properties: HashMap<&str, HashMap<&str, &str>> = HashMap::new();

    // ...
}
```

请注意，哈希 map 的键和值都是 `&str`，即借用的字符串。`pest` 解析器不会复制他们解析的输入，而是借用。所有用于检查解析结果的方法都会返回从原始解析字符串中借用字符串。

## 主循环

现在我们解释解析结果。我们循环浏览文件的每一行，这一行要么是节名，要么是键值属性对。如果遇到一个节名，我们更新一个变量。如果遇到一个属性对，我们就获取一个对当前章节的哈希 map 的引用，然后把这个属性对插入到这个哈希 map 中。

```rust
    // ...

    let mut current_section_name = "";

    for line in file.into_inner() {
        match line.as_rule() {
            Rule::section => {
                let mut inner_rules = line.into_inner(); // { name }
                current_section_name = inner_rules.next().unwrap().as_str();
            }
            Rule::property => {
                let mut inner_rules = line.into_inner(); // { name ~ "=" ~ value }

                let name: &str = inner_rules.next().unwrap().as_str();
                let value: &str = inner_rules.next().unwrap().as_str();

                // Insert an empty inner hash map if the outer hash map hasn't
                // seen this section name before.
                let section = properties.entry(current_section_name).or_default();
                section.insert(name, value);
            }
            Rule::EOI => (),
            _ => unreachable!(),
        }
    }

    // ...
```

在输出方面，我们用[漂亮的打印](https://doc.rust-lang.org/std/fmt/index.html#sign0) `Debug` 格式简单地转储哈希 map。

```rust
fn main() {
    // ...

    println!("{:#?}", properties);
}
```

## 空白

如果你把本章顶部的例子 INI 文件复制到 `config.ini` 文件中并运行程序，它将无法解析。我们已经忘记了等号周围的可选空格!

对于大型 grammar 来说，处理空白会很不方便。显示地编写 `whitespace` 规则并手动插入空白会让 grammar 变得难以阅读和修改。`pest` 提供了一个[特殊规则 `WHITESPACE`](https://pest.rs/book/grammars/syntax.html#implicit-whitespace) 的解决方案。如果定义了 `WHITESPACE`，它将被隐式地运行，尽可能多次地在每个波浪号 `~` 和每个重复之间运行（例如，`*` 和 `+`）。对于我们的 INI 解析器，只有空格才是合法的 whitespace。

```
WHITESPACE = _{ " " }
```

我们用一个前导的下划线 `_{ ... }` 来标记 `WHITESPACE` 规则的[静默](https://pest.rs/book/grammars/syntax.html#silent-and-atomic-rules)。}. 这样，即使它匹配，也不会出现在其他规则中。如果它不是静默的，解析就会复杂得多，因为对  `Pairs::next(...)` 的每次调用都有可能返回 `Rule::WHITESPACE` 而不是想要的下一条规则。

但是等等! 节名、键或值中不应该有空格！目前，空格是自动插入的。目前，在 `name = { char+ }` 中，空格会自动插入字符之间。对空格敏感的规则需要用前导符号 `@{ ... }` 来标记[原子](https://pest.rs/book/grammars/syntax.html#atomic)。}. 在原子规则中，自动的空白处理是被禁用的，而内部规则是静默的。

```
name = @{ char+ }
value = @{ char* }
```

## 完工

试试吧！确保文件 `config.ini` 存在，然后运行程序! 你应该看到这样的东西。

```
$ cargo run
  [ ... ]
{
    "": {
        "password": "plain_text",
        "username": "noha",
        "salt": "NaCl"
    },
    "second_server": {
        "ip": "",
        "document_root": "/var/www/example.com",
        "interface": "eth1"
    },
    "server_1": {
        "interface": "eth0",
        "document_root": "/var/www/example.org",
        "ip": "127.0.0.1"
    }
}
```
