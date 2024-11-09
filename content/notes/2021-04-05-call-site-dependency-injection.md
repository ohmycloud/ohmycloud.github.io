+++
title = "Call Site Dependency Injection"
date = 2021-04-05
[taxonomies]
  tags = ["rust", "dependency injection"]
+++

本帖文档调用站点依赖注入模式。这是一个相当低级的样本，和企业 DI 没有什么关系。这个模式有点 Rust 特有。

通常，当你实现一个需要用户提供一些功能的类型时，首先想到的是在构造函数中提供它。

```rust
struct Engine {
    config: Config,
    ...
}

impl Engine {
    fn new(config: Config) -> Engine { ... }
    fn go(&mut self) { ... }
}
```

在这个例子中，我们实现了 Engine，调用者提供了 Config。

另一种方法是将依赖关系传递给每个方法调用。

```rust
struct Engine {
    ...
}

impl Engine {
    fn new() -> Engine { ... }
    fn go(&mut self, config: &Config) { ... }
}
```

在 Rust 中，后者(call-site injection)有时用 lifetime 更好。让我们来看看这些例子吧!

## Lazy 字段

在第一个例子中，我们想根据其他字段惰性地计算一个字段的值。就像这样:

```rust
struct Widget {
    name: String,
    name_hash: Lazy<u64>,
}

impl Widget {
    fn new(name: String) -> Widget {
        Widget {
            name,
            name_hash: Lazy::new(|| {
                compute_hash(&self.name)
            }),
        }
    }
}
```

这个设计的问题是在 Rust 中无法使用。Lazy 中的闭包需要访问 **self**，而这将创建一个自引用的数据结构!

解决的办法是在使用 Lazy 的地方提供闭包。

```rust
struct Widget {
    name: String,
    name_hash: OnceCell<u64>,
}

impl Widget {
    fn new(name: String) -> Widget {
        Widget {
            name,
            name_hash: OnceCell::new(),
        }
    }
    fn name_hash(&self) -> u64 {
        *self.name_hash.get_or_init(|| {
            compute_hash(&self.name)
        })
    }
}
```

## 间接哈希表

下一个例子是关于将一个自定义的哈希函数插入到哈希表中。在 Rust 的标准库中，这只能在类型级别上实现，通过实现类型的 Hash 特性。更通用的设计是在运行时用哈希函数给表做参数。这是 `C++` 所做的。然而在 Rust 中，这就不够通用了。

考虑一个字符串互译器，它将字符串存储在一个向量中，并额外维护一个基于哈希的索引。

```rust
struct Interner {
    vec: Vec<String>,
    set: HashSet<usize>,
}

impl Interner {
    fn intern(&mut self, s: &str) -> usize { ... }
    fn lookup(&self, i: usize) -> &str { ... }
}
```

**set** 字段将字符串存储在一个哈希表中，但它是用相邻 **vec** 的索引来表示它们。

用一个闭包来构造 **set** 不会成功，原因和 Lazy 一样 - 这将创建一个自引用结构。在 `C++` 中，存在一个变通的方法 - 可以将 **vec** 装箱，并在 **Interner** 和闭包之间共享一个稳定的指针。在 Rust 中，这会产生别名，阻止使用 **&mut Vec**。

奇怪的是，在 std API 中，使用排序的 vec 而不是哈希是可行的。

```rust
struct Interner {
    vec: Vec<String>,
    // Invariant: sorted
    set: Vec<usize>,
}

impl Interner {
    fn intern(&mut self, s: &str) -> usize {
        let idx = self.set.binary_search_by(|&idx| {
            self.vec[idx].cmp(s)
        });
        match idx {
            Ok(idx) => self.set[idx],
            Err(idx) => {
                let res = self.vec.len();
                self.vec.push(s.to_string());
                self.set.insert(idx, res);
                res
            }
        }
    }
    fn lookup(&self, i: usize) -> &str { ... }
}
```

这是因为闭包是在调用站点而不是在构造站点供给的。

hashbrown crate 通过 [RawEntry](https://docs.rs/hashbrown/0.9.1/hashbrown/hash_map/struct.HashMap.html#method.raw_entry_mut) 为哈希提供了这种风格的 API。

## Per 容器分配器

第三个例子来自 Zig 编程语言。与 Rust 不同，Zig 没有一个祝福的全局分配器。相反，Zig 中的容器有两种风味。"Managed" 风味接受一个分配器作为构造参数，并将其存储为一个字段（[Source](https://github.com/ziglang/zig/blob/1590ed9d6aea95e5a21e3455e8edba4cdb374f2c/lib/std/array_list.zig#L36-L43)）。而 "Unmanaged" 风味则在每个方法中添加一个分配器参数（[Source](https://github.com/ziglang/zig/blob/1590ed9d6aea95e5a21e3455e8edba4cdb374f2c/lib/std/array_list.zig#L436-L440)）。

第二种方式更节俭 - 可以用一个分配器引用与许多容器。

## 胖指针

最后一个例子来自于 Rust 语言本身。为了实现动态调度，Rust 使用了胖指针，它有两个字宽。第一个字指向对象，第二个字指向 vtable。这些指针是在泛用具体类型的时候制造的。

这与 `C++` 不同，`C++` 的 vtable 指针是在构造过程中嵌入到对象本身中的。

看了这些例子后，我对 Scala 式的隐式参数很热衷。考虑一下这段带有 Zig 风格向量的 Rust 代码的假设。

```rust
{
    let mut a = get_allocator();
    let mut xs = Vec::new();
    let mut ys = Vec::new();
    xs.push(&mut a, 1);
    ys.push(&mut a, 2);
}
```

这里的问题是 Drop - 释放向量需要访问分配器，而如何提供一个分配器并不清楚。Zig 通过使用 defer 语句而不是 destructors 躲避了这个问题。在使用隐式参数的 Rust 中，我想下面的方法可以用。

```rust
impl<implicit a: &mut Allocator, T> Drop for Vec<T>
```

最后，我想分享最后一个例子，CSDI 思维帮助我发现了一个更好的应用级架构。

rust-analyzer 的很多行为是可以配置的。有嵌套提示的切换，完成度可以调整，一些功能根据编辑器的不同而有不同的工作方式。第一个实现是将一个全局的 Config 结构和其他分析状态一起存储。然后各个子系统读取这个 Config 的位。为了避免通过这个共享结构将不同的功能耦合在一起，配置键是动态的。

```rust
type Config = HashMap<String, String>;
```

这个系统是可行的，但感觉相当笨拙。

现在的实现要简单得多。现在每个方法都接受一个特定的 config 参数，而不是将一个单一的 Config 作为状态的一部分来存储。

```rust
fn get_completions(
    analysis: &Analysis,
    config: &CompletionConfig,
    file: FileId,
    offset: usize,
)

fn get_inlay_hints(
    analysis: &Analysis,
    config: &HintsConfig,
    file: FileId,
)
```

不仅代码更简单，而且更灵活。因为配置不再是状态的一部分，所以可以根据上下文的不同，对同一功能使用不同的配置。例如，显式调用的完成和异步的完成可能是不同的。

在 [/r/rust](https://old.reddit.com/r/rust/comments/kmd41e/blog_post_call_site_dependency_injection/) 上讨论。

原文链接: https://matklad.github.io/2020/12/28/csdi.html
