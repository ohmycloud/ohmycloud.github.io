+++
title = "Pest Grammars"
date = 2021-01-20
[taxonomies]
  tags = ["rust", "pest", "parser"]
+++

# Grammar

与许多解析工具一样，`pest` 使用与 Rust 代码不同的正式 grammar 进行操作。`pest` 使用的格式称为解析表达式 grammar，或 PEG。当构建一个项目时，`pest` 会自动将位于单独文件中的 PEG 编译成您可以调用的普通 Rust 函数。

## 如何激活 `pest`

大多数项目至少会有两个使用 `pest` 的文件：解析器 (比如 `src/parser/mod.rs`) 和 grammar (`src/parser/grammar.pest`)。假设它们在同一个目录下。

```rust
use pest::Parser;

#[derive(Parser)]
#[grammar = "parser/grammar.pest"] // relative to project `src`
struct MyParser;
```

每当你编译这个文件时，`pest` 会自动使用 grammar 文件生成这样的项。

```rust
pub enum Rules { /* ... */ }

impl Parser for MyParser {
    pub fn parse(Rules, &str) -> pest::Pairs { /* ... */ }
}
```

你永远不会看到 `enum Rules` 或 `impl Parser` 的纯文本。这些代码只存在于编译过程中。然而，您可以像使用其他枚举一样使用 `Rules`，并且您可以通过 [Parser API 章节](https://pest.rs/book/parser_api.html)中描述的 `Pairs` 接口使用 `parse(...)`。

## 关于 PEGs 的警告!

解析表达式 grammar 看起来和你可能习惯的其他解析工具很相似，比如正则表达式、BNF grammar 和其他工具（Yacc/Bison、LALR、CFG）。然而，PEGs 的行为却有微妙的不同。PEGs 是[急切的](https://pest.rs/book/grammars/peg.html#eagerness)、[非回溯的](https://pest.rs/book/grammars/peg.html#non-backtracking)、[有序的](https://pest.rs/book/grammars/peg.html#ordered-choice)、[不含糊的](https://pest.rs/book/grammars/peg.html#unambiguous)。

如果你不认识以上任何一个名字，不要害怕! 你已经比认识的人快了一步 - 当你使用 `pest` 的 PEGs 时，你不会被与其他工具的比较所绊倒。

如果你之前使用过其他解析工具，一定要仔细阅读下一节。我们会提到一些关于 PEGs 的常见错误。

## 解析表达式语法

解析表达式语法(PEG)只是严格地表示了如果你用手写一个解析器会写的简单的命令式代码。

```
number = {            // To recognize a number...
    ASCII_DIGIT+      //   take as many ASCII digits as possible (at least one).
}
expression = {        // To recognize an expression...
    number            //   first try to take a number...
    | "true"          //   or, if that fails, the string "true".
}
```

事实上，pest 产生的代码与上面注释中的伪代码十分相似。

### Eagerness

当在输入字符串上运行[重复的](https://pest.rs/book/grammars/syntax.html#repetition) PEG 表达式时。

```
ASCII_DIGIT+      // one or more characters from '0' to '9'
```

它尽可能多地运行该表达式（"急切地"或 "贪婪地"匹配）。它要么成功，消耗它所匹配的任何内容，并将剩余的输入传递到解析器的下一步。

```
"42 boxes"
 ^ Running ASCII_DIGIT+

"42 boxes"
   ^ Successfully took one or more digits!

" boxes"
 ^ Remaining unparsed input.
```

或失败，什么也不消耗。

```
"galumphing"
 ^ Running ASCII_DIGIT+
   Failed to take one or more digits!

"galumphing"
 ^ Remaining unparsed input (everything).
```

如果一个表达式未能匹配，那么这个失败就会向上传播，最终导致解析失败，除非这个失败在 grammar 中的某个地方被"抓住"。选择操作符是"捕获"这种失败的一种方法。

### 有序选择

[选择操作符](https://pest.rs/book/grammars/syntax.html#ordered-choice)，写成一条竖线 `|`，是有序的。PEG 表达式 `first | second` 的意思是 "先试 `first`，但如果失败了，再试 `second`"。

在许多情况下，顺序并不重要。例如，`"true" | "false"` 将匹配字符串 `"true"` 或字符串 `"false"`（如果两者都不出现，则失败）。

然而，有时顺序确实很重要。考虑一下 PEG 表达式 `"a" | "ab"`。你可能期望它能匹配字符串 `"a"` 或字符串 `"ab"`。但事实并非如此 - 该表达式的意思是 "尝试 `"a"`；但如果失败，则尝试 `"ab"`。如果你正在匹配字符串 "abc"，尝试 `"a"` 不会失败；相反，它将成功匹配 `"a"`，留下 `"bc"` 未被解析。

一般来说，当编写一个有选择的解析器时，把最长或最具体的选择放在前面，而把最短或最一般的选择放在最后。

### 非回溯

在解析过程中，一个 PEG 表达式要么成功，要么失败。如果成功了，下一步就照常进行。但如果它失败了，整个表达式就会失败。引擎不会后退再试。

请看下面这个 grammar，在字符串 `"frumious"` 上进行匹配。

```
word = {     // to recognize a word...
    ANY*     //   take any character, zero or more times...
    ~ ANY    //   followed by any character
}
```

你可能期望这条规则能够解析任何至少包含一个字符（相当于 `ANY+`）的输入字符串。但它不会。相反，第一个 `ANY*` 会急切地吃掉整个字符串 - 它会得偿所愿的。然后，下一个 `ANY` 将一无所有，所以它会失败。

```
"frumious"
 ^ (word)

"frumious"
         ^ (ANY*) Success! Continue to `ANY` with remaining input "".

""
 ^ (ANY) Failure! Expected one character, but found end of string.
```

在有回溯功能的系统中（比如正则表达式），你会往后退一步，"吐出"一个字符，然后再试。但 PEG 不会这样做。在规则 `first~second` 中，一旦 `first` 解析成功，就已经消耗了一些字符，永远不会再回来，`second` 只能在 `first` 没有消耗的输入上运行。

### 毫不含糊

这些规则构成了一个优雅而简单的系统。每个 PEG 规则都会在输入字符串的剩余部分上运行，消耗尽可能多的输入。一旦一个规则完成，剩下的输入就会被传递给解析器的其他部分。

例如，表达式 `ASCII_DIGIT+`，"一个或多个数字"，将始终匹配可能的最大的连续数字序列。不存在意外地让后面的规则回溯并以一种不直观和非局部的方式窃取一些数字的危险。

这与其他解析工具形成了鲜明的对比，比如正则表达式和 CFG，在这些工具中，规则的结果往往取决于一些距离的代码。事实上，LR解析器中著名的"移位/还原冲突"在 PEG 中并不存在问题。

### 不要惊慌

这一切在一开始可能有点反常。但正如你所看到的，基本的逻辑是非常简单和直接的。你可以琐碎地逐步完成任何 PEG 表达式的执行。

- 试试这个。
- 如果它成功了，就尝试下一件事。
- 否则，尝试另一件事。

```
(this ~ next_thing) | (other_thing)
```

这些规则结合在一起，使得 PEG 成为编写解析器的非常愉快的工具。

## pet 解析器的语法

`pet` grammar 是规则的列表。规则是这样定义的。

```
my_rule = { ... }

another_rule = {        // comments are preceded by two slashes
    ...                 // whitespace goes anywhere
}
```

由于规则名被翻译成 Rust enum 变体，所以不允许成为 Rust 关键字。

定义规则的左大括号 `{` 前面可以有[影响其操作](https://pest.rs/book/grammars/syntax.html#silent-and-atomic-rules)的符号。

```
silent_rule = _{ ... }
atomic_rule = @{ ... }
```

### 表达式

Grammar 规则是由表达式建立起来的（因此称为"解析表达式文法"）。这些表达式是对如何解析输入字符串的简明、正式的描述。

表达式是可以组合的：它们可以从其他表达式中构建出来，也可以互相嵌套，以产生任意复杂的规则（尽管你应该将非常复杂的表达式分解成多个规则，以使它们更容易管理）。

PEG 表达式既适用于高级意义，如"一个函数签名，后面是一个函数体"，也适用于低级意义，如"一个分号，后面是换行"。组合形式"后面是"，即[序列操作符](https://pest.rs/book/grammars/syntax.html#sequence)，在这两种情况下都是一样的。

### 终端

最基本的规则是双引号的文字字符串。`"text"`。

如果一个字符串前面有一个逗号，那么它可以不区分大小写（仅适用于 ASCII 字符）: `^"text"`。

在一个范围内的单个字符被写成两个单引号字符，用两个点分开：`'0'...'9'`。

你可以用特殊规则 `ANY` 来匹配任何单个字符。这相当于 `'\u{00}'...'\u{10FFFF}'`，任何一个 Unicode 字符。

```
"a literal string"
^"ASCII case-insensitive string"
'a'..'z'
ANY
```

最后，你可以直接写出其他规则的名称来引用它们，甚至可以递归使用规则。

```
my_rule = { "slithy " ~ other_rule }
other_rule = { "toves" }
recursive_rule = { "mimsy " ~ recursive_rule }
```

### 序列

序列运算符写成一个波浪号 `~`。

```
first ~ and_then

("abc") ~ (^"def") ~ ('g'..'z')        // matches "abcDEFr"
```

当匹配一个序列表达式时，尝试匹配 `first`。如果 `first` 匹配成功，则接下来尝试 `and_then`。但是，如果 `first` 失败，则整个表达式失败。

表达式的列表可以与序列链在一起，这表明所有的组件必须出现，按照指定的顺序。

### 有序选择

选择运算符写成一条竖线 `|`。

```
first | or_else

("abc") | (^"def") | ('g'..'z')        // matches "DEF"
```

当匹配一个选择表达式时，尝试匹配 `first`。如果 `first` 匹配成功，则整个表达式立即成功。但是，如果 `first` 失败，接下来会尝试 `or_else`。

注意，`first` 和 `or_else` 总是在同一个位置尝试，即使 `first` 在失败之前匹配了一些输入。当遇到解析失败时，引擎会尝试下一个有序的选择，就像没有匹配到输入一样。失败的解析永远不会消耗任何输入。

```
start = { "Beware " ~ creature }
creature = {
    ("the " ~ "Jabberwock")
    | ("the " ~ "Jubjub bird")
}

"Beware the Jubjub bird"
 ^ (start) Parses via the second choice of `creature`,
           even though the first choice matched "the " successfully.
```

借用术语，把这种操作看成是"交替"或简单的 "OR"，有点诱人，但这是误导。之所以特别使用 "选择" 这个词，是因为这个操作不仅仅是逻辑上的 "OR"。

### 重复

有两个重复运算符：星号 `*` 和加号 `+`。它们被放在一个表达式之后。星号 `*` 表示前面的表达式可以出现零次或多次。加号 `+` 表示前面的表达式可以出现一次或多次（必须至少出现一次）。

问号运算符 `?` 类似，但它表示表达式是可选的 - 它可以出现0次或1次。

```
("zero" ~ "or" ~ "more")*
 ("one" | "or" | "more")+
           (^"optional")?
```

请注意，`expr*` 和 `expr?` 总是会成功，因为它们被允许匹配零次。例如，`"a"* ~ "b"?` 即使在空的输入字符串上也会成功。

其他重复次数可以用大括号来表示。

```
expr{n}           // exactly n repetitions
expr{m, n}        // between m and n repetitions, inclusive

expr{, n}         // at most n repetitions
expr{m, }         // at least m repetitions
```

因此，`expr*` 等同于 `expr{0，}`；`expr+` 等同于 `expr{1，}`；`expr?` 等同于 `expr{0，1}`。

### 谓词

在表达式前面加上安括号 `&` 或感叹号 `!`，就会变成一个不消耗任何输入的谓词。你可能知道这些运算符为 "向前查看" 或 "不进位"。

写成安培符 `&` 的正式谓词试图匹配其内部表达式。如果内部表达式成功，解析就会继续，但位置与谓词相同 - `&foo ~ bar` 因此是一种 "AND" 语句。"输入字符串必须匹配 `foo` AND `bar`"。如果内部表达式失败，整个表达式也会失败。

写成感叹号的否定谓词 `!`，试图匹配其内部表达式。如果内部表达式失败，则谓词成功，并在与谓词相同的位置继续解析。如果内部表达式成功，则谓词失败 - `!foo ~ bar` 因此是一种 "NOT" 语句。"输入的字符串必须与 `bar` 匹配，但不能是 `foo`"。

这就引出了一个常见的惯用法，意思是"任何字符但是"：

```
not_space_or_tab = {
    !(                // if the following text is not
        " "           //     a space
        | "\t"        //     or a tab
    )
    ~ ANY             // then consume one character
}

triple_quoted_string = {
    "'''"
    ~ triple_quoted_character*
    ~ "'''"
}
triple_quoted_character = {
    !"'''"        // if the following text is not three apostrophes
    ~ ANY         // then consume one character
}
```

### 操作符优先级和分组 (WIP)

重复运算符星号 `*`、加号 `+` 和问号 `?` 适用于紧接前面的表达式。

```
"One " ~ "or " ~ "more. "+
"One " ~ "or " ~ ("more. "+)
    are equivalent and match
"One or more. more. more. more. "
```

较大的表达式可以通过用括号包围来重复。

```
("One " ~ "or " ~ "more. ")+
    matches
"One or more. One or more. "
```

重复运算符的优先性最高，其次是谓词运算符、序列运算符，最后是有序选择。

```
my_rule = {
    "a"* ~ "b"?
    | &"b"+ ~ "a"
}

// equivalent to

my_rule = {
      ( ("a"*) ~ ("b"?) )
    | ( (&("b"+)) ~ "a" )
}
```

### 输入的开始和结束

规则 `SOI` 和 `EOI` 分别匹配输入字符串的开始和结束。两者都不消耗任何文本。它们只表明解析器当前是否在输入的一个边缘。

例如，为了确保一条规则匹配整个输入，其中任何语法错误都会导致解析失败（而不是成功但不完整的解析）。

```
main = {
    SOI
    ~ (...)
    ~ EOI
}
```

### 隐含的空白

许多语言和文本格式允许在逻辑标记之间任意留白和注释。例如，Rust 认为 `4+5` 相当于 `4 + 5` 和 `4 /* comment */ + 5`。

可选规则 `WHITESPACE` 和 `COMMENT` 实现了这种行为。如果定义了这两个规则中的任何一个(或两个)，它们将被隐式地插入到每个[序列](https://pest.rs/book/grammars/syntax.html#sequence)和每个[重复](https://pest.rs/book/grammars/syntax.html#repetition)之间([原子规则](https://pest.rs/book/grammars/syntax.html#atomic)除外)。

```
expression = { "4" ~ "+" ~ "5" }
WHITESPACE = _{ " " }
COMMENT = _{ "/*" ~ (!"*/" ~ ANY)* ~ "*/" }
```

```
"4+5"
"4 + 5"
"4  +     5"
"4 /* comment */ + 5"
```

正如你所看到的，`WHITESPACE` 和 `COMMENT` 是重复运行的，所以它们只需要匹配一个空白字符或一个注释。上面的 grammar 相当于。

```
expression = {
    "4"   ~ (ws | com)*
    ~ "+" ~ (ws | com)*
    ~ "5"
}
ws = _{ " " }
com = _{ "/*" ~ (!"*/" ~ ANY)* ~ "*/" }
```

请注意，隐式空格不会插入规则的开头或结尾 - 例如，表达式不匹配 `" 4+5 "`。如果你想在规则的开头和结尾加入隐式空格，你需要把它夹在两个空规则之间（通常是 `SOI` 和 `EOI`，[如上所述](https://pest.rs/book/grammars/syntax.html#start-and-end-of-input)）。

```
WHITESPACE = _{ " " }
expression = { "4" ~ "+" ~ "5" }
main = { SOI ~ expression ~ EOI }
```

```
"4+5"
"  4 + 5   "
```

(请务必将 `WHITESPACE` 和 `COMMENT` 规则标记为[静默](https://pest.rs/book/grammars/syntax.html#silent-and-atomic-rules)规则，除非你想在其他规则中看到它们！)

### 静默规则和原子规则

静默规则就像普通规则一样 - 当运行时，它们的功能是一样的 - 除了它们不产生 [pairs](https://pest.rs/book/parser_api.html#pairs)或 [tokens](https://pest.rs/book/parser_api.html#tokens)。如果一条规则是静默的，那么它永远不会出现在解析结果中。

要创建一个静默规则，请在左边的大括号 `{` 前加上一个下划线 `_`。

```
silent = _{ ... }
```

### 原子

pest 有两种原子规则：原子和复合原子。要做一个，在左大括号 `{` 前写上一个符号。

```
atomic = @{ ... }
compound_atomic = ${ ... }
```

这两种原子规则都可以防止[隐式空格](https://pest.rs/book/grammars/syntax.html#implicit-whitespace)：在原子规则中，波浪号 `~` 表示 "紧接着"，[重复操作符](https://pest.rs/book/grammars/syntax.html#repetition)（星号 `*` 和加号 `+`）没有隐式分隔。此外，所有从原子规则中调用的其他规则也被视为原子规则。

两者的区别在于它们如何产生内部规则的标记。在一个原子规则中，内部匹配规则是[静默的](https://pest.rs/book/grammars/syntax.html#silent-and-atomic-rules)。相比之下，复合原子规则会像普通规则一样产生内部 token。

当您要解析的文本忽略空白时，原子规则是很有用的，除了少数情况，例如文字字符串。在这种情况下，您可以编写 `WHITESPACE` 或 `COMMENT` 规则，然后使您的字符串匹配规则成为原子规则。

### 非原子的

有时候，你会想要取消原子解析的效果。例如，你可能想在表达式内部进行字符串插值，里面的表达式仍然可以像正常的一样有空格。

```python
#!/bin/env python3
print(f"The answer is {2 + 4}.")
```

这是你使用非原子规则的地方。在定义的大括号前面写一个感叹号 `!` 无论是否从原子规则中调用，该规则都将作为非原子规则运行。

```
fstring = @{ "\"" ~ ... }
expr = !{ ... }
```

### 堆栈(WIP)

pest 维护了一个可以直接从 grammar 中操作的栈。一个表达式可以用关键字 `PUSH` 进行匹配并推到栈上，然后再用关键字 `PEEK` 和 `POP` 进行精确匹配。

使用栈可以对完全相同的文本进行多次匹配，而不是相同的模式。

例如:

```
same_text = {
    PUSH( "a" | "b" | "c" )
    ~ POP
}
same_pattern = {
    ("a" | "b" | "c")
    ~ ("a" | "b" | "c")
}
```

在这种情况下，`same_pattern` 会匹配 `"ab"`，而 `same_text` 不会。

一个实际的用途是解析 Rust 的 "[原始字符串字面值](https://doc.rust-lang.org/book/second-edition/appendix-02-operators.html#non-operator-symbols)"，它看起来像这样。

```rust
const raw_str: &str = r###"
    Some number of number signs # followed by a quotation mark ".

    Quotation marks can be used anywhere inside: """"""""",
    as long as one is not followed by a matching number of number signs,
    which ends the string: "###;
```

当解析一个原始字符串时，我们必须跟踪引号前出现了多少个数字符号 `#`。我们可以使用栈来完成这个任务。

```
raw_string = {
    "r" ~ PUSH("#"*) ~ "\""    // push the number signs onto the stack
    ~ raw_string_interior
    ~ "\"" ~ POP               // match a quotation mark and the number signs
}
raw_string_interior = {
    (
        !("\"" ~ PEEK)    // unless the next character is a quotation mark
                          // followed by the correct amount of number signs,
        ~ ANY             // consume one character
    )*
}
```

### 小抄

| 语法              |	含义                       |	语法             | 含义               |
|:-----------------|:-----------------------------|:-------------------|:-------------------|
| `foo = { ... }`  |  regular rule                |	`baz = @{ ... }`   | atomic             |
| `bar = _{ ... }` |  silent`	                  | `qux = ${ ... }`   | compound-atomic    |
|                  |                              | `plugh = !{ ... }` | non-atomic         |
| `"abc"`	       | exact string                 | `^"abc"`	       | case insensitive   |
| `'a'..'z'`       | character range              |	`ANY`              | any character      |
| `foo ~ bar`      | sequence	                  | `baz | qux`	       | ordered choice     |
| `foo*`           | zero or more                 | `bar+`             | one or more        |
| `baz?`           | optional	                  | `qux{n}`           | exactly n          |
| `qux{m, n}`      | between m and n  (inclusive) |                    |                    |
| `&foo`           | positive predicate	          | `!bar`             | negative predicate |
| `PUSH(baz)`      | match and push	              |                    |                    |
| `POP`            | match and pop	              | `PEEK`             | match without pop  |

## 内置规则

除了 `ANY`，匹配任何单一的 Unicode 字符外，`pest` 还提供了几条规则，让解析文本更加方便。

### ASCII 规则

在可打印的 ASCII 字符中，它通常对匹配字母字符和数字很有用。对于数字，pest 提供了常见的（基数）的数字。

| Built-in rule	        | Equivalent                        |
|:----------------------|:----------------------------------|
| `ASCII_DIGIT`	        | `'0'..'9'`                        |
| `ASCII_NONZERO_DIGIT`	| `'1'..'9'`                        |
| `ASCII_BIN_DIGIT`	    | `'0'..'1'`                        |
| `ASCII_OCT_DIGIT`	    | `'0'..'7'`                        |
| `ASCII_HEX_DIGIT`	    | `'0'..'9' | 'a'..'f' | 'A'..'F'`  |

对于字母字符，要区分大写和小写。

| Built-in rule	    | Equivalent
|:------------------|:------------------------|
| ASCII_ALPHA_LOWER	| `'a'..'z'`              |
| ASCII_ALPHA_UPPER	| `'A'..'Z'`              |
| ASCII_ALPHA	    | `'a'..'z' | 'A'..'Z'`   |

And for miscellaneous use:

| Built-in rule	        | Meaning	            | Equivalent
|:----------------------|:----------------------|-----------------------------|
| ASCII_ALPHANUMERIC	| any digit or letter	| `ASCII_DIGIT | ASCII_ALPHA` |
| NEWLINE	            | any line feed format	| `"\n" | "\r\n" | "\r"`      |

### 统一码规则

为了更容易正确解析任意 Unicode 文本，pest 包含了大量对应 Unicode 字符属性的规则。这些规则分为一般类别和二进制属性规则。

Unicode 字符根据其一般用途被划分为不同的类别。每一个字符都属于一个类别，就像每一个 ASCII 字符都是一个控制字符、一个数字、一个字母、一个符号或一个空格一样。

此外，每个 Unicode 字符都有一个二进制属性列表（真或假），它满足或不满足这些属性。字符可以属于任何数量的这些属性，这取决于它们的含义。

例如，字符 "A"，"拉丁文大写字母A"，属于一般的 "大写字母" 类别，因为它的一般用途是字母。它具有 "大写字母" 的二元属性，但不具有 "表情符号" 的属性。相比之下，"负数平方的拉丁文大写字母A" 这个字符，因为在文本中一般不作为字母出现，所以属于一般类别 "其他符号"。它同时具有 "大写字母" 和 "表情符号" 的二元属性。

详情请参考《Unicode 标准》第四章。

### 一般类别

从形式上看，类别是不重叠的：每个 Unicode 字符正好属于一个类别，没有一个类别包含另一个类别。然而，由于某些类别组经常一起使用，pest 在下面暴露了类别的层次结构。例如，规则 `CASED_LETTER` 在技术上不是 Unicode 通用类别，而是匹配属于 UPPERCASE_LETTER  或LOWERCASE_LETTER 的字符，这些都是通用类别。

- LETTER
- CASED_LETTER
- UPPERCASE_LETTER
- LOWERCASE_LETTER
- TITLECASE_LETTER
- MODIFIER_LETTER
- OTHER_LETTER
- MARK
- NONSPACING_MARK
- SPACING_MARK
- ENCLOSING_MARK
- NUMBER
- DECIMAL_NUMBER
- LETTER_NUMBER
- OTHER_NUMBER
- PUNCTUATION
- CONNECTOR_PUNCTUATION
- DASH_PUNCTUATION
- OPEN_PUNCTUATION
- CLOSE_PUNCTUATION
- INITIAL_PUNCTUATION
- FINAL_PUNCTUATION
- OTHER_PUNCTUATION
- SYMBOL
- MATH_SYMBOL
- CURRENCY_SYMBOL
- MODIFIER_SYMBOL
- OTHER_SYMBOL
- SEPARATOR
- SPACE_SEPARATOR
- LINE_SEPARATOR
- PARAGRAPH_SEPARATOR
- OTHER
- CONTROL
- FORMAT
- SURROGATE
- PRIVATE_USE
- UNASSIGNED

### Binary properties

这些属性中有许多是用来定义 Unicode 文本算法的，如双向算法和文本分割算法。这类属性对于大多数解析器来说可能并不有用。

但是，XID_START 和 XID_CONTINUE 这两个属性特别值得注意，因为它们被定义为 "协助标识符的标准处理"，"如编程语言变量"。详见技术报告31。

- ALPHABETIC
- BIDI_CONTROL
- CASE_IGNORABLE
- CASED
- CHANGES_WHEN_CASEFOLDED
- CHANGES_WHEN_CASEMAPPED
- CHANGES_WHEN_LOWERCASED
- CHANGES_WHEN_TITLECASED
- CHANGES_WHEN_UPPERCASED
- DASH
- DEFAULT_IGNORABLE_CODE_POINT
- DEPRECATED
- DIACRITIC
- EXTENDER
- GRAPHEME_BASE
- GRAPHEME_EXTEND
- GRAPHEME_LINK
- HEX_DIGIT
- HYPHEN
- IDS_BINARY_OPERATOR
- IDS_TRINARY_OPERATOR
- ID_CONTINUE
- ID_START
- IDEOGRAPHIC
- JOIN_CONTROL
- LOGICAL_ORDER_EXCEPTION
- LOWERCASE
- MATH
- NONCHARACTER_CODE_POINT
- OTHER_ALPHABETIC
- OTHER_DEFAULT_IGNORABLE_CODE_POINT
- OTHER_GRAPHEME_EXTEND
- OTHER_ID_CONTINUE
- OTHER_ID_START
- OTHER_LOWERCASE
- OTHER_MATH
- OTHER_UPPERCASE
- PATTERN_SYNTAX
- PATTERN_WHITE_SPACE
- PREPENDED_CONCATENATION_MARK
- QUOTATION_MARK
- RADICAL
- REGIONAL_INDICATOR
- SENTENCE_TERMINAL
- SOFT_DOTTED
- TERMINAL_PUNCTUATION
- UNIFIED_IDEOGRAPH
- UPPERCASE
- VARIATION_SELECTOR
- WHITE_SPACE
- XID_CONTINUE
- XID_START

## 例子: JSON

JSON 是一种流行的数据序列化格式，它源于 JavaScript 的语法。JSON 文档是树状的，并且可能是递归的--对象和数组这两种数据类型可以包含其他值，包括其他对象和数组。

下面是一个 JSON 文档的例子。

```json
{
    "nesting": { "inner object": {} },
    "an array": [1.5, true, null, 1e-6],
    "string with escaped double quotes" : "\"quick brown foxes\""
}
```

让我们写一个程序，将 JSON 解析成一个 Rust 对象，也就是抽象语法树，然后将 AST 序列化回 JSON。

### 设置

我们将从定义 Rust 中的 AST 开始。每个 JSON 数据类型都由一个枚举变体来表示。

```rust
enum JSONValue<'a> {
    Object(Vec<(&'a str, JSONValue<'a>)>),
    Array(Vec<JSONValue<'a>>),
    String(&'a str),
    Number(f64),
    Boolean(bool),
    Null,
}
```

为了避免反序列化字符串时的复制，JSONValue 从原始未解析的 JSON 中借用字符串。为了使其工作，我们不能解释字符串转义序列：输入字符串 "\n" 将由 JSONValue::String("\n") 表示，这是一个有两个字符的 Rust 字符串，尽管它表示的是一个只有一个字符的 JSON 字符串。

让我们继续看序列化器。为了清晰起见，它使用分配的 Strings，而不是提供 std::fmt::Display 的实现，后者会更习惯。

```rust
fn serialize_jsonvalue(val: &JSONValue) -> String {
    use JSONValue::*;

    match val {
        Object(o) => {
            let contents: Vec<_> = o
                .iter()
                .map(|(name, value)|
                     format!("\"{}\":{}", name, serialize_jsonvalue(value)))
                .collect();
            format!("{{{}}}", contents.join(","))
        }
        Array(a) => {
            let contents: Vec<_> = a.iter().map(serialize_jsonvalue).collect();
            format!("[{}]", contents.join(","))
        }
        String(s) => format!("\"{}\"", s),
        Number(n) => format!("{}", n),
        Boolean(b) => format!("{}", b),
        Null => format!("null"),
    }
}
```

请注意，在对象和数组的情况下，函数会递归地调用自己。这种模式出现在整个解析器中。AST创建函数在解析结果中递归迭代，而语法的规则也包括了自己。

### grammar 的编写

让我们从 whitespace 开始。JSON 空格可以出现在任何地方，除了字符串内部（必须单独解析）和数字中的数字之间（不允许）。这使得它很适合 pest 的隐式空白。在 `src/json.pest`:

```
WHITESPACE = _{ " " | "\t" | "\r" | "\n" }
```

JSON 规范包括解析 JSON 字符串的图。我们可以直接从该页面写出语法。让我们把 object 写成一个用逗号分隔的对的序列。

```
object = {
    "{" ~ "}" |
    "{" ~ pair ~ ("," ~ pair)* ~ "}"
}
pair = { string ~ ":" ~ value }

array = {
    "[" ~ "]" |
    "[" ~ value ~ ("," ~ value)* ~ "]"
}
```

对象和数组规则展示了如何用分隔符解析一个潜在的空列表。有两种情况：一种是空列表，另一种是至少有一个元素的列表。这是必要的，因为数组中的逗号，如 `[0，1，]`，在 JSON 中是非法的。

现在我们可以写 value，它代表任何单一的数据类型。我们将模仿我们的 AST，将 boolean 和 null 写成单独的规则。

```
value = _{ object | array | string | number | boolean | null }

boolean = { "true" | "false" }

null = { "null" }
```

让我们把字符串的逻辑分成三个部分。`char` 是一个匹配字符串中任何逻辑字符的规则，包括任何反斜杠转义序列。`inner` 代表字符串的内容，不包括周围的双引号。`string` 匹配字符串的内部内容，包括周围的双引号。

`char` 规则使用成语 `!(...) ~ ANY`，它匹配除了括号中给出的字符之外的任何字符。在这种情况下，除了双引号 `""` 和反斜杠 `\` 之外，任何字符在字符串内部都是合法的，这需要单独的解析逻辑。

```
string = ${ "\"" ~ inner ~ "\"" }
inner = @{ char* }
char = {
    !("\"" | "\\") ~ ANY
    | "\\" ~ ("\"" | "\\" | "/" | "b" | "f" | "n" | "r" | "t")
    | "\\" ~ ("u" ~ ASCII_HEX_DIGIT{4})
}
```

因为 `string` 被标记为复原子，所以 `string` token 对也会包含一个 `inner` 对。因为 `inner` 被标记为原子，所以在 `inner` 中不会出现 `char` 对。由于这些规则是原子性的，所以在不同的标记之间不允许有空格。

数字有四个逻辑部分：一个可选的符号、一个整数部分、一个可选的分数部分和一个可选的指数。我们将把数字标记为原子，这样它的部分之间就不能出现空白。

```
number = @{
    "-"?
    ~ ("0" | ASCII_NONZERO_DIGIT ~ ASCII_DIGIT*)
    ~ ("." ~ ASCII_DIGIT*)?
    ~ (^"e" ~ ("+" | "-")? ~ ASCII_DIGIT+)?
}
```

我们需要一个最终规则来表示整个 JSON 文件。JSON 文件的唯一合法内容是一个对象或数组。我们将把这个规则标记为沉默，这样一个解析后的 JSON 文件只包含两个标记对：解析后的值本身，以及 EOI 规则。

```
json = _{ SOI ~ (object | array) ~ EOI }
```

### AST 生成

让我们把 grammar 编译成 Rust。

```rust
extern crate pest;
#[macro_use]
extern crate pest_derive;

use pest::Parser;

#[derive(Parser)]
#[grammar = "json.pest"]
struct JSONParser;
```

我们将写一个同时处理解析和 AST 生成的函数。该函数的用户可以在输入字符串上调用它，然后将返回的结果作为 JSONValue 或解析错误。

```rust
use pest::error::Error;

fn parse_json_file(file: &str) -> Result<JSONValue, Error<Rule>> {
    let json = JSONParser::parse(Rule::json, file)?.next().unwrap();

    // ...
}
```

现在我们需要根据规则，递归处理 `Pair`。我们知道 json 是一个对象或者数组，但是这些值本身可能包含一个对象或者数组！这时，我们就需要写一个辅助递归函数，直接将 `Pair` 解析成 `JSONValue`。最合理的处理方式是写一个辅助递归函数，直接将 `Pair` 解析成 JSONValue。

```rust
fn parse_json_file(file: &str) -> Result<JSONValue, Error<Rule>> {
    // ...

    use pest::iterators::Pair;

    fn parse_value(pair: Pair<Rule>) -> JSONValue {
        match pair.as_rule() {
            Rule::object => JSONValue::Object(
                pair.into_inner()
                    .map(|pair| {
                        let mut inner_rules = pair.into_inner();
                        let name = inner_rules
                            .next()
                            .unwrap()
                            .into_inner()
                            .next()
                            .unwrap()
                            .as_str();
                        let value = parse_value(inner_rules.next().unwrap());
                        (name, value)
                    })
                    .collect(),
            ),
            Rule::array => JSONValue::Array(pair.into_inner().map(parse_value).collect()),
            Rule::string => JSONValue::String(pair.into_inner().next().unwrap().as_str()),
            Rule::number => JSONValue::Number(pair.as_str().parse().unwrap()),
            Rule::boolean => JSONValue::Boolean(pair.as_str().parse().unwrap()),
            Rule::null => JSONValue::Null,
            Rule::json
            | Rule::EOI
            | Rule::pair
            | Rule::value
            | Rule::inner
            | Rule::char
            | Rule::WHITESPACE => unreachable!(),
        }
    }

    // ...
}
```

对象和数组的情况值得特别注意。数组令牌对的内容只是一个值的序列。由于我们使用的是 Rust 迭代器，我们可以简单地将每个值递归地映射到它的解析 AST 节点，然后将它们收集到一个 Vec 中。对于对象，过程是类似的，除了迭代器是在对上，我们需要分别从对上提取名称和值。

数字和布尔的情况下，使用 Rust 的 str::parse 方法将解析后的字符串转换为相应的 Rust 类型。每一个合法的 JSON 数字都可以直接解析成一个 Rust 浮点数！我们在 Rust 的 str::parse 方法上运行 parse_value。

我们对解析结果运行 parse_value 来完成转换。

```rust
fn parse_json_file(file: &str) -> Result<JSONValue, Error<Rule>> {
    // ...

    Ok(parse_value(json))
}
```

### 精加工

我们的主要功能现在非常简单。首先，我们从一个名为 data.json 的文件中读取 JSON 数据。接下来，我们将文件内容解析成一个 JSON AST。最后，我们将 AST 序列化回一个字符串并打印出来。

```rust
use std::fs;

fn main() {
    let unparsed_file = fs::read_to_string("data.json").expect("cannot read file");

    let json: JSONValue = parse_json_file(&unparsed_file).expect("unsuccessful parse");

    println!("{}", serialize_jsonvalue(&json));
}
```

试试吧! 将本章顶部的示例文档复制到 data.json 中，然后运行程序! 你应该看到这样的东西。

```
$ cargo run
  [ ... ]
{"nesting":{"inner object":{}},"an array":[1.5,true,null,0.000001],"string with escaped double quotes":"\"quick brown foxes\""}
```

## 例子: J 语言

J 语言是一种受 APL 影响的数组编程语言。在 J 语言中，对单个数字(`2*3`)的操作可以很容易地应用于整个数字列表(`2*3 4 5`，返回 `6 8 10`)。

J 中的操作符被称为动词。动词要么是一元的（取一个参数，如 `*: 3`，"3 的平方"），要么是二元的（取两个参数，两边各一个，如 `5 - 4`，"5减4"）。

下面是一个 J 程序的例子。

```
'A string'

*: 1 2 3 4

matrix =: 2 3 $ 5 + 2 3 4 5 6 7
10 * matrix

1 + 10 20 30
1 2 3 + 10

residues =: 2 | 0 1 2 3 4 5 6 7
residues
```

使用 J 的[解释器](https://jsoftware.com/)运行上述程序，在标准输出上得到如下结果。

```
A string

1 4 9 16

 70  80  90
100 110 120

11 21 31
11 12 13

0 1 0 1 0 1 0 1
```

在这一节中，我们将为 J 的一个子集写一个 grammar，然后我们将通过一个解析器，通过迭代 `pest` 给我们的规则来建立一个 AST。你可以在[本书的资源库](https://github.com/pest-parser/book/tree/master/examples/jlang-parser)中找到完整的源代码。

### Grammar

我们将从程序规则开始，逐节建立 grammar。

```
program = _{ SOI ~ "\n"* ~ (stmt ~ "\n"+) * ~ stmt? ~ EOI }
```

每个 J 程序都包含由一个或多个换行符分隔的语句。请注意前面的下划线，它告诉 `pest` [屏蔽](https://pest.rs/book/grammars/syntax.html#silent-and-atomic-rules) `program` 规则 - 我们不想让 `program` 作为一个 token 出现在解析流中，我们想要的是底层语句。

语句就是一个简单的表达式，由于只有一种这样的可能性，所以我们也将这个 `stmt` 规则[屏蔽](https://pest.rs/book/grammars/syntax.html#silent-and-atomic-rules)，这样我们的解析器就会收到一个底层 `expr` 的迭代器。

```
stmt = _{ expr }
```

表达式可以是对变量标识符的赋值，也可以是单项表达式、对偶表达式、单个字符串或术语数组。

```
expr = {
      assgmtExpr
    | monadicExpr
    | dyadicExpr
    | string
    | terms
}
```

一元表达式由一个动词组成，其唯一的操作数在右边；三元表达式的操作数在动词的两边。赋值表达式将标识符与表达式相关联。

在 J 中，没有操作符的优先性 - 求值是右联的（从右到左），括号内的表达式先被求值。

```
monadicExpr = { verb ~ expr }

dyadicExpr = { (monadicExpr | terms) ~ verb ~ expr }

assgmtExpr = { ident ~ "=:" ~ expr }
```

项的列表应该至少包含一个十进制、整数、标识符或小括号表达式；我们只关心这些基础值，所以我们用前导下划线[屏蔽](https://pest.rs/book/grammars/syntax.html#silent-and-atomic-rules) `term` 规则。

```
terms = { term+ }

term = _{ decimal | integer | ident | "(" ~ expr ~ ")" }
```

J 的几个动词在这个 grammar 中是有定义的，J 的[全部词汇](https://code.jsoftware.com/wiki/NuVoc)要广泛得多。

```
verb = {
    ">:" | "*:" | "-"  | "%" | "#" | ">."
  | "+"  | "*"  | "<"  | "=" | "^" | "|"
  | ">"  | "$"
}
```

现在我们可以进入词法规则了。J 中的数字和平常一样，除了负数用前导的 `_` 下划线表示外（因为 `-` 是一个动词，它作为单项式执行否定，作为对偶式执行减法）。J 中的标识符必须以字母开头，但之后可以包含数字。字符串由单引号包围；引号本身可以通过用附加引号转义来嵌入。

请注意我们如何使用 `pest` 的 `@` 修饰符使这些规则中的每一条都是[原子的](https://pest.rs/book/grammars/syntax.html#atomic)，这意味着[隐式空白](https://pest.rs/book/grammars/syntax.html#implicit-whitespace)是被禁止的，而且内部规则（即 `ident` 中的 `ASCII_ALPHA`）变为 [silent](https://pest.rs/book/grammars/syntax.html#silent-and-atomic-rules) - 当我们的解析器接收到这些 token 时，它们将是终端的。

```
integer = @{ "_"? ~ ASCII_DIGIT+ }

decimal = @{ "_"? ~ ASCII_DIGIT+ ~ "." ~ ASCII_DIGIT* }

ident = @{ ASCII_ALPHA ~ (ASCII_ALPHANUMERIC | "_")* }

string = @{ "'" ~ ( "''" | (!"'" ~ ANY) )* ~ "'" }
```

J 中的空白只由空格和制表符组成。换行的意义在于它们是对语句的定界，因此它们不在本规则之内。

```
WHITESPACE = _{ " " | "\t" }
```

最后，我们必须处理注释。J 中的注释以 `NB.` 开始，一直到它们所在行的末尾。关键的是，我们决不能消耗注释行末的换行；这是为了将注释之前的任何语句与后续行的语句分开。

```
COMMENT = _{ "NB." ~ (!"\n" ~ ANY)* }
```

### 解析和 AST 生成

本节将介绍一个使用上述 grammar 的解析器。这里省略了库中的内容和自明的代码，你可以在[本书的资源库](https://github.com/pest-parser/book/tree/master/examples/jlang-parser)中找到解析器的全部内容。

首先我们将枚举我们 grammar 中定义的动词，区分一元动词和二元动词。这些枚举将在我们的 AST 中作为标签使用。

```rust
pub enum MonadicVerb {
    Increment,
    Square,
    Negate,
    Reciprocal,
    Tally,
    Ceiling,
    ShapeOf,
}

pub enum DyadicVerb {
    Plus,
    Times,
    LessThan,
    LargerThan,
    Equal,
    Minus,
    Divide,
    Power,
    Residue,
    Copy,
    LargerOf,
    LargerOrEqual,
    Shape,
}
```

那么我们就来列举一下 AST 的各类节点。

```rust
pub enum AstNode {
    Print(Box<AstNode>),
    Integer(i32),
    DoublePrecisionFloat(f64),
    MonadicOp {
        verb: MonadicVerb,
        expr: Box<AstNode>,
    },
    DyadicOp {
        verb: DyadicVerb,
        lhs: Box<AstNode>,
        rhs: Box<AstNode>,
    },
    Terms(Vec<AstNode>),
    IsGlobal {
        ident: String,
        expr: Box<AstNode>,
    },
    Ident(String),
    Str(CString),
}
```

为了解析 J 程序中的顶层语句，我们有下面的 `parse` 函数，它接受一个字符串形式的 J 程序，并将其传递给 `pest` 进行解析。我们得到一个 `Pair` 的序列。正如 grammar 中所规定的那样，一个语句只能由一个表达式组成，所以下面的匹配会解析这些顶层表达式中的每一个，并将它们包装在一个 `Print` AST 节点中，以符合 J 解释器的 REPL 行为。

```rust
pub fn parse(source: &str) -> Result<Vec<AstNode>, Error<Rule>> {
    let mut ast = vec![];

    let pairs = JParser::parse(Rule::program, source)?;
    for pair in pairs {
        match pair.as_rule() {
            Rule::expr => {
                ast.push(Print(Box::new(build_ast_from_expr(pair))));
            }
            _ => {}
        }
    }

    Ok(ast)
}
```

AST 节点是通过遍历 `Pair` 迭代器，按照我们 grammar 文件中设定的期望值，从表达式中构建出来的。常见的行为被抽象出单独的函数，如 `parse_monadic_verb` 和 `parse_dyadic_verb`，代表表达式本身的 `Pair` 则在递归调用 `build_ast_from_expr` 中传递。

```rust
fn build_ast_from_expr(pair: pest::iterators::Pair<Rule>) -> AstNode {
    match pair.as_rule() {
        Rule::expr => build_ast_from_expr(pair.into_inner().next().unwrap()),
        Rule::monadicExpr => {
            let mut pair = pair.into_inner();
            let verb = pair.next().unwrap();
            let expr = pair.next().unwrap();
            let expr = build_ast_from_expr(expr);
            parse_monadic_verb(verb, expr)
        }
        // ... other cases elided here ...
    }
}
```

二元动词从它们的字符串表示方式直接映射到 AST 节点。

```rust
fn parse_dyadic_verb(pair: pest::iterators::Pair<Rule>, lhs: AstNode, rhs: AstNode) -> AstNode {
    AstNode::DyadicOp {
        lhs: Box::new(lhs),
        rhs: Box::new(rhs),
        verb: match pair.as_str() {
            "+" => DyadicVerb::Plus,
            "*" => DyadicVerb::Times,
            "-" => DyadicVerb::Minus,
            "<" => DyadicVerb::LessThan,
            "=" => DyadicVerb::Equal,
            ">" => DyadicVerb::LargerThan,
            "%" => DyadicVerb::Divide,
            "^" => DyadicVerb::Power,
            "|" => DyadicVerb::Residue,
            "#" => DyadicVerb::Copy,
            ">." => DyadicVerb::LargerOf,
            ">:" => DyadicVerb::LargerOrEqual,
            "$" => DyadicVerb::Shape,
            _ => panic!("Unexpected dyadic verb: {}", pair.as_str()),
        },
    }
}
```

如同一元动词一样。

```rust
fn parse_monadic_verb(pair: pest::iterators::Pair<Rule>, expr: AstNode) -> AstNode {
    AstNode::MonadicOp {
        verb: match pair.as_str() {
            ">:" => MonadicVerb::Increment,
            "*:" => MonadicVerb::Square,
            "-" => MonadicVerb::Negate,
            "%" => MonadicVerb::Reciprocal,
            "#" => MonadicVerb::Tally,
            ">." => MonadicVerb::Ceiling,
            "$" => MonadicVerb::ShapeOf,
            _ => panic!("Unsupported monadic verb: {}", pair.as_str()),
        },
        expr: Box::new(expr),
    }
}
```

最后，我们定义了一个函数来处理数字和字符串等项。数字需要一些操作来处理 J 的前导下划线，表示否定，但除此之外，处理过程是典型的。

```rust
fn build_ast_from_term(pair: pest::iterators::Pair<Rule>) -> AstNode {
    match pair.as_rule() {
        Rule::integer => {
            let istr = pair.as_str();
            let (sign, istr) = match &istr[..1] {
                "_" => (-1, &istr[1..]),
                _ => (1, &istr[..]),
            };
            let integer: i32 = istr.parse().unwrap();
            AstNode::Integer(sign * integer)
        }
        Rule::decimal => {
            let dstr = pair.as_str();
            let (sign, dstr) = match &dstr[..1] {
                "_" => (-1.0, &dstr[1..]),
                _ => (1.0, &dstr[..]),
            };
            let mut flt: f64 = dstr.parse().unwrap();
            if flt != 0.0 {
                // Avoid negative zeroes; only multiply sign by nonzeroes.
                flt *= sign;
            }
            AstNode::DoublePrecisionFloat(flt)
        }
        Rule::expr => build_ast_from_expr(pair),
        Rule::ident => AstNode::Ident(String::from(pair.as_str())),
        unknown_term => panic!("Unexpected term: {:?}", unknown_term),
    }
}
```

### 运行解析器

现在我们可以定义一个 `main` 函数，将 J 程序传递给我们的 `pest`-enabled 解析器。

```rust
fn main() {
    let unparsed_file = std::fs::read_to_string("example.ijs")
      .expect("cannot read ijs file");
    let astnode = parse(&unparsed_file).expect("unsuccessful parse");
    println!("{:?}", &astnode);
}
```

在 example.ijs 中使用这段代码。

```
_2.5 ^ 3
*: 4.8
title =: 'Spinning at the Boundary'
*: _1 2 _3 4
1 2 3 + 10 20 30
1 + 10 20 30
1 2 3 + 10
2 | 0 1 2 3 4 5 6 7
another =: 'It''s Escaped'
3 | 0 1 2 3 4 5 6 7
(2+1)*(2+2)
3 * 2 + 1
1 + 3 % 4
x =: 100
x - 1
y =: x - 1
y
```

当我们运行解析器时，我们会在标准输出上得到以下抽象语法树。

```
$ cargo run
  [ ... ]
[Print(DyadicOp { verb: Power, lhs: DoublePrecisionFloat(-2.5),
    rhs: Integer(3) }),
Print(MonadicOp { verb: Square, expr: DoublePrecisionFloat(4.8) }),
Print(IsGlobal { ident: "title", expr: Str("Spinning at the Boundary") }),
Print(MonadicOp { verb: Square, expr: Terms([Integer(-1), Integer(2),
    Integer(-3), Integer(4)]) }),
Print(DyadicOp { verb: Plus, lhs: Terms([Integer(1), Integer(2), Integer(3)]),
    rhs: Terms([Integer(10), Integer(20), Integer(30)]) }),
Print(DyadicOp { verb: Plus, lhs: Integer(1), rhs: Terms([Integer(10),
    Integer(20), Integer(30)]) }),
Print(DyadicOp { verb: Plus, lhs: Terms([Integer(1), Integer(2), Integer(3)]),
    rhs: Integer(10) }),
Print(DyadicOp { verb: Residue, lhs: Integer(2),
    rhs: Terms([Integer(0), Integer(1), Integer(2), Integer(3), Integer(4),
    Integer(5), Integer(6), Integer(7)]) }),
Print(IsGlobal { ident: "another", expr: Str("It\'s Escaped") }),
Print(DyadicOp { verb: Residue, lhs: Integer(3), rhs: Terms([Integer(0),
    Integer(1), Integer(2), Integer(3), Integer(4), Integer(5),
    Integer(6), Integer(7)]) }),
Print(DyadicOp { verb: Times, lhs: DyadicOp { verb: Plus, lhs: Integer(2),
    rhs: Integer(1) }, rhs: DyadicOp { verb: Plus, lhs: Integer(2),
        rhs: Integer(2) } }),
Print(DyadicOp { verb: Times, lhs: Integer(3), rhs: DyadicOp { verb: Plus,
    lhs: Integer(2), rhs: Integer(1) } }),
Print(DyadicOp { verb: Plus, lhs: Integer(1), rhs: DyadicOp { verb: Divide,
    lhs: Integer(3), rhs: Integer(4) } }),
Print(IsGlobal { ident: "x", expr: Integer(100) }),
Print(DyadicOp { verb: Minus, lhs: Ident("x"), rhs: Integer(1) }),
Print(IsGlobal { ident: "y", expr: DyadicOp { verb: Minus, lhs: Ident("x"),
    rhs: Integer(1) } }),
Print(Ident("y"))]
```
