+++
title = "Sizedness in Rust"
date = "2021-04-11"
[taxonomies]
  tags = ["rust", "sizedness"]
+++

## Intro

Sizedness 是 Rust 中最重要的概念之一。它与其他语言特性有很多微妙的交集，只是以_"x在编译时不知道大小"_错误信息的形式出现，而这些错误信息是每个Rustacean都非常熟悉的。在这篇文章中，我们将探讨从大小类型，到无大小类型，再到零大小类型的各种风味，同时研究它们的用例、好处、痛点和变通方法。

我使用的短语表，以及它们的含义。

| Phrase | Shorthand for |
|-|-|
| sizedness | property of being sized or unsized |
| sized type | type with a known size at compile time |
| 1) unsized type _or_<br>2) DST | dynamically-sized type, i.e. size not known at compile time |
| ?sized type | type that may or may not be sized |
| unsized coercion | coercing a sized type into an unsized type |
| ZST | zero-sized type, i.e. instances of the type are 0 bytes in size |
| width | single unit of measurement of pointer width |
| 1) thin pointer _or_<br>2) single-width pointer | pointer that is _1 width_ |
| 1) fat pointer _or_<br>2) double-width pointer | pointer that is _2 widths_ |
| 1) pointer _or_<br>2) reference | some pointer of some width, width will be clarified by context |
| slice | double-width pointer to a dynamically sized view into some array |



## Sizedness

在 Rust 中，如果在编译时可以确定类型的字节大小，那么就可以确定类型的大小。确定一个类型的大小对于能够在栈上为该类型的实例分配足够的空间是很重要的。固定大小的类型可以通过值或引用来传递。如果一个类型的大小不能在编译时确定，那么它被称为不确定大小类型或 DST，动态大小类型。由于不确定大小类型不能被放置在栈上，它们只能通过引用来传递。下面是一些固定大小类型和不确定大小类型的例子。

```rust
use std::mem::size_of;

fn main() {
    // primitives
    assert_eq!(4, size_of::<i32>());
    assert_eq!(8, size_of::<f64>());

    // tuples
    assert_eq!(8, size_of::<(i32, i32)>());

    // arrays
    assert_eq!(0, size_of::<[i32; 0]>());
    assert_eq!(12, size_of::<[i32; 3]>());

    struct Point {
        x: i32,
        y: i32,
    }

    // structs
    assert_eq!(8, size_of::<Point>());

    // enums
    assert_eq!(8, size_of::<Option<i32>>());

    // get pointer width, will be
    // 4 bytes wide on 32-bit targets or
    // 8 bytes wide on 64-bit targets
    const WIDTH: usize = size_of::<&()>();

    // pointers to sized types are 1 width
    assert_eq!(WIDTH, size_of::<&i32>());
    assert_eq!(WIDTH, size_of::<&mut i32>());
    assert_eq!(WIDTH, size_of::<Box<i32>>());
    assert_eq!(WIDTH, size_of::<fn(i32) -> i32>());

    const DOUBLE_WIDTH: usize = 2 * WIDTH;

    // unsized struct
    struct Unsized {
        unsized_field: [i32],
    }

    // pointers to unsized types are 2 widths
    assert_eq!(DOUBLE_WIDTH, size_of::<&str>()); // slice
    assert_eq!(DOUBLE_WIDTH, size_of::<&[i32]>()); // slice
    assert_eq!(DOUBLE_WIDTH, size_of::<&dyn ToString>()); // trait object
    assert_eq!(DOUBLE_WIDTH, size_of::<Box<dyn ToString>>()); // trait object
    assert_eq!(DOUBLE_WIDTH, size_of::<&Unsized>()); // user-defined unsized type

    // unsized types
    size_of::<str>(); // compile error
    size_of::<[i32]>(); // compile error
    size_of::<dyn ToString>(); // compile error
    size_of::<Unsized>(); // compile error
}
```

我们如何确定固定大小类型的大小是直截了当的：所有原生类型和指针都有已知的大小，所有的结构体、元组、枚举和数组只是由原生类型和指针或其他嵌套的结构体、元组、枚举和数组组成，因此我们只需考虑到填充和对齐所需的额外字节，递归地计数字节即可。我们无法确定不确定大小类型的大小，原因同样简单明了：切片可以有任意数量的元素在其中，因此在运行时可以是任意大小的，trait 对象可以由任意数量的结构或枚举实现，因此在运行时也可以是任意大小的。

**专业提示**
- 在Rust中，视图到数组中的动态大小的指针被称为切片。例如 `&str` 是一个"字符串切片", `&[i32]` 一个 _"i32 切片"_。
- 切片是双倍宽度的，因为它们存储了一个指向数组的指针和数组中元素的数量。
- trait 对象指针是双宽度的，因为它们存储了一个指向数据的指针和一个指向 vtable 的指针。
- 不确定大小的结构体指针是双倍宽度的，因为它们存储了一个指向结构体数据的指针和结构体的大小。
- 不确定大小的结构体只能有1个不确定大小的字段，而且必须是结构体中的最后一个字段。

为了让大家真正明白关于不确定大小类型的双宽度指针的点，这里有一个比较数组和切片的注释代码示例。

```rust
use std::mem::size_of;

const WIDTH: usize = size_of::<&()>();
const DOUBLE_WIDTH: usize = 2 * WIDTH;

fn main() {
    // data length stored in type
    // an [i32; 3] is an array of three i32s
    let nums: &[i32; 3] = &[1, 2, 3];

    // single-width pointer
    assert_eq!(WIDTH, size_of::<&[i32; 3]>());

    let mut sum = 0;

    // can iterate over nums safely
    // Rust knows it's exactly 3 elements
    for num in nums {
        sum += num;
    }

    assert_eq!(6, sum);

    // unsized coercion from [i32; 3] to [i32]
    // data length now stored in pointer
    let nums: &[i32] = &[1, 2, 3];

    // double-width pointer required to also store data length
    assert_eq!(DOUBLE_WIDTH, size_of::<&[i32]>());

    let mut sum = 0;

    // can iterate over nums safely
    // Rust knows it's exactly 3 elements
    for num in nums {
        sum += num;
    }

    assert_eq!(6, sum);
}
```

这里还有一个注释的代码例子，比较结构体和 trait 对象。

```rust
use std::mem::size_of;

const WIDTH: usize = size_of::<&()>();
const DOUBLE_WIDTH: usize = 2 * WIDTH;

trait Trait {
    fn print(&self);
}

struct Struct;
struct Struct2;

impl Trait for Struct {
    fn print(&self) {
        println!("struct");
    }
}

impl Trait for Struct2 {
    fn print(&self) {
        println!("struct2");
    }
}

fn print_struct(s: &Struct) {
    // always prints "struct"
    // this is known at compile-time
    s.print();
    // single-width pointer
    assert_eq!(WIDTH, size_of::<&Struct>());
}

fn print_struct2(s2: &Struct2) {
    // always prints "struct2"
    // this is known at compile-time
    s2.print();
    // single-width pointer
    assert_eq!(WIDTH, size_of::<&Struct2>());
}

fn print_trait(t: &dyn Trait) {
    // print "struct" or "struct2" ?
    // this is unknown at compile-time
    t.print();
    // Rust has to check the pointer at run-time
    // to figure out whether to use Struct's
    // or Struct2's implementation of "print"
    // so the pointer has to be double-width
    assert_eq!(DOUBLE_WIDTH, size_of::<&dyn Trait>());
}

fn main() {
    // single-width pointer to data
    let s = &Struct;
    print_struct(s); // prints "struct"

    // single-width pointer to data
    let s2 = &Struct2;
    print_struct2(s2); // prints "struct2"

    // unsized coercion from Struct to dyn Trait
    // double-width pointer to point to data AND Struct's vtable
    let t: &dyn Trait = &Struct;
    print_trait(t); // prints "struct"

    // unsized coercion from Struct2 to dyn Trait
    // double-width pointer to point to data AND Struct2's vtable
    let t: &dyn Trait = &Struct2;
    print_trait(t); // prints "struct2"
}
```

**关键要点**
- 只有固定大小类型的实例才能被放置在栈上，也就是说，可以通过值来传递
- 不确定大小类型的实例不能放在栈上，必须通过引用来传递
- 指向不确定大小类型的指针是双宽度的，因为除了指向数据外，它们还需要做额外的记账工作，以跟踪数据的长度或指向一个 vtable



## `Sized` Trait

Rust中的 "Sized" trait 是一个自动 trait 和一个标记 trait。

自动 trait 是指当一个类型通过某些条件时，自动实现的 trait。标记 trait 是标记一个类型具有特定属性的 trait。标记 trait 没有任何 trait 项，如方法、关联函数、关联常量或关联类型。所有的自动 trait 都是标记 trait，但不是所有的标记 trait 都是自动 trait。自动 trait 必须是标记 trait，所以编译器可以为它们提供一个自动的缺省实现，如果 trait 有任何 trait 项，这是不可能的。

如果一个类型的所有成员也是 "确定大小的"，那么它就会得到一个自动的 `Sized` 实现。"成员"的含义取决于所包含的类型，例如：结构体的字段、枚举的变体、数组的元素、元组的项等等。一旦一个类型被 "标记" 了一个 `Sized` 的实现，这意味着在编译时就知道它的字节大小。

其他自动标记 trait 的例子是 `Send` 和 `Sync` trait。如果跨线程发送一个类型是安全的，那么这个类型就是可 `Send` 的。如果在线程之间共享该类型的引用是安全的，那么该类型就是可 `Sync` 的。如果一个类型的所有成员都是可 `Send` 和 `Sync` 的, 那么这个类型就会得到自动的 `Send` 和 `Sync` 实现。`Sized` 的特殊之处在于它不可能选择退出，不像其他自动标记 trait 可以选择退出。

```rust
#![feature(negative_impls)]

// this type is Sized, Send, and Sync
struct Struct;

// opt-out of Send trait
impl !Send for Struct {}

// opt-out of Sync trait
impl !Sync for Struct {}

impl !Sized for Struct {} // compile error
```

这似乎是合理的，因为我们可能有理由不希望我们的类型被跨线程发送或共享，但是很难想象我们会希望编译器 "忘记" 我们类型的大小，并将其视为一个不确定大小的类型，因为这不会带来任何好处，只会让类型更难处理。

另外，说得迂腐一点，`Sized` 在技术上并不是一个自动 trait，因为它没有使用 `auto` 关键字来定义，但是编译器对它的特殊处理使它的行为与自动 trait 非常相似，所以在实践中，把它看作是一个自动 trait 是可以的。

**关键要点**
- `Sized` 是一个自动标记 trait



## 泛型中的 `Sized`

每当我们编写任何泛型代码时，每一个泛型类型参数都会被默认的 `Sized` trait 自动绑定，这一点并不明显。

```rust
// this generic function...
fn func<T>(t: T) {}

// ...desugars to...
fn func<T: Sized>(t: T) {}

// ...which we can opt-out of by explicitly setting ?Sized...
fn func<T: ?Sized>(t: T) {} // compile error

// ...which doesn't compile since t doesn't have
// a known size so we must put it behind a pointer...
fn func<T: ?Sized>(t: &T) {} // compiles
fn func<T: ?Sized>(t: Box<T>) {} // compiles
```

**专业提示**
- `?Sized` can be pronounced _"optionally sized"_ or _"maybe sized"_ and adding it to a type parameter's bounds allows the type to be sized or unsized
- `?Sized` in general is referred to as a _"widening bound"_ or a _"relaxed bound"_ as it relaxes rather than constrains the type parameter
- `?Sized` is the only relaxed bound in Rust

So why does this matter? Well, any time we're working with a generic type and that type is behind a pointer we almost always want to opt-out of the default `Sized` bound to make our function more flexible in what argument types it will accept. Also, if we don't opt-out of the default `Sized` bound we'll eventually get some surprising and confusing compile error messages.

Let me take you on the journey of the first generic function I ever wrote in Rust. I started learning Rust before the `dbg!` macro landed in stable so the only way to print debug values was to type out `println!("{:?}", some_value);` every time which is pretty tedious so I decided to write a `debug` helper function like this:
- `?Sized` 可以读作 _"optionally sized"_ 或 _"maybe sized"_，将它添加到类型参数的绑定中，可以让类型被确定大小或不确定大小。
- `?Sized` 一般被称为 "拓宽绑定" 或 "宽松绑定"，因为它放松而不是约束类型参数。
- `?Sized` 是 Rust 中唯一的宽松绑定。

那么为什么这很重要呢？任何时候，当我们在处理泛型类型，并且该类型在一个指针后面时，我们几乎总是希望选择退出默认的 `Sized` 绑定，以使我们的函数在接受什么参数类型时更加灵活。另外，如果我们不选择退出默认的 `Sized` 绑定，我们最终会得到一些令人惊讶和困惑的编译错误信息。

让我带你了解一下我在 Rust 中写的第一个泛型函数的历程。在 `dbg!` 宏登陆稳定版之前，我就开始学习 Rust 了，所以打印调试值的唯一方法就是每次都要打出 `println!("{:?}", some_value);`，这是很乏味的，所以我决定写一个像这样的调试帮助函数。

```rust
use std::fmt::Debug;

fn debug<T: Debug>(t: T) { // T: Debug + Sized
    println!("{:?}", t);
}

fn main() {
    debug("my str"); // T = &str, &str: Debug + Sized ✔️
}
```

到目前为止还不错，但函数会对传递给它的任何值拥有所有权，这有点烦人，所以我把函数改为只接受引用。

```rust
use std::fmt::Debug;

fn dbg<T: Debug>(t: &T) { // T: Debug + Sized
    println!("{:?}", t);
}

fn main() {
    dbg("my str"); // &T = &str, T = str, str: Debug + !Sized ❌
}
```

现在出现了这个错误。

```rust
error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/main.rs:8:9
  |
3 | fn dbg<T: Debug>(t: &T) {
  |        - required by this bound in `dbg`
...
8 |     dbg("my str");
  |         ^^^^^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `str`
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
help: consider relaxing the implicit `Sized` restriction
  |
3 | fn dbg<T: Debug + ?Sized>(t: &T) {
  |
```

当我第一次看到这个问题时，我发现它令人难以置信的混乱。尽管我的函数对参数的限制比以前更严格，但现在它却莫名其妙地抛出了一个编译错误！这是怎么回事？到底发生了什么？

我已经在上面的代码注释中破坏了答案，但基本上。Rust 在编译过程中把 `T` 解析为具体类型时，会执行模式匹配。这里有几个表格可以帮助澄清。

| Type | `T` | `&T` |
|------------|---|----|
| `&str` | `T` = `&str` | `T` = `str` |

| Type | `Sized` |
|------|---------|
| `str`   | ❌ |
| `&str`  | ✔️ |
| `&&str` | ✔️ |

这也是为什么我不得不在改成取用引用后，加了一个 `?Sized` 的绑定，使函数能正常工作。下面是可以工作的函数。

```rust
use std::fmt::Debug;

fn debug<T: Debug + ?Sized>(t: &T) { // T: Debug + ?Sized
    println!("{:?}", t);
}

fn main() {
    debug("my str"); // &T = &str, T = str, str: Debug + !Sized ✔️
}
```

**关键要点**
- 所有的泛型类型参数默认都是自动绑定 `Sized`。
- 如果我们有一个泛型函数，它的参数是指针后面的一些 `T`，例如 `&T`、`Box<T>`、`Rc<T>` 等，那么我们几乎总是希望用`T: ?Sized` 来退出默认的 `Sized` 约束。



## Unsized 类型



### 切片

最常见的切片是字符串切片 `&str` 和数组切片 `&[T]`。切片的好处是许多其他类型也会对其进行 coerce，所以利用切片和 Rust 的自动类型 coerce，我们可以编写灵活的 API。

类型 coerce 可以发生在几个地方，但最明显的是在函数参数和方法调用时。我们感兴趣的类型 coerce 是 deref coerce 和 unsized coerce。deref coerce 是指当 `T` 在 deref 操作之后被 coerce 成一个 `U`，即 `T: Deref<Target = U>`，例如 `String.deref() -> str`。不确定大小 coerce 是指 `T` 被 coerce 成 `U`，其中 `T` 是一个确定大小的类型，`U` 是一个不确定大小的类型，即 `T: Unsize<U>`，例如 `[i32; 3] -> [i32]`。

```rust
trait Trait {
    fn method(&self) {}
}

impl Trait for str {
    // can now call "method" on
    // 1) str or
    // 2) String since String: Deref<Target = str>
}
impl<T> Trait for [T] {
    // can now call "method" on
    // 1) any &[T]
    // 2) any U where U: Deref<Target = [T]>, e.g. Vec<T>
    // 3) [T; N] for any N, since [T; N]: Unsize<[T]>
}

fn str_fun(s: &str) {}
fn slice_fun<T>(s: &[T]) {}

fn main() {
    let str_slice: &str = "str slice";
    let string: String = "string".to_owned();

    // function args
    str_fun(str_slice);
    str_fun(&string); // deref coercion

    // method calls
    str_slice.method();
    string.method(); // deref coercion

    let slice: &[i32] = &[1];
    let three_array: [i32; 3] = [1, 2, 3];
    let five_array: [i32; 5] = [1, 2, 3, 4, 5];
    let vec: Vec<i32> = vec![1];

    // function args
    slice_fun(slice);
    slice_fun(&vec); // deref coercion
    slice_fun(&three_array); // unsized coercion
    slice_fun(&five_array); // unsized coercion

    // method calls
    slice.method();
    vec.method(); // deref coercion
    three_array.method(); // unsized coercion
    five_array.method(); // unsized coercion
}
```

**关键要点**
- 利用切片和 Rust 的自动类型强制，我们可以编写灵活的 API。



### Trait 对象

Traits 默认是 `?Sized` 的。这个程序:

```rust
trait Trait: ?Sized {}
```

抛出这个错误:

```rust
error: `?Trait` is not permitted in supertraits
 --> src/main.rs:1:14
  |
1 | trait Trait: ?Sized {}
  |              ^^^^^^
  |
  = note: traits are `?Sized` by default
```

我们很快就会讨论为什么 trait 默认为 `?Sized`，但首先让我们问问自己，一个 trait 被 `?Sized` 的含义是什么？让我们把上面的例子去掉。

```rust
trait Trait where Self: ?Sized {}
```

好的，默认情况下，trait 允许 `self` 是一个不确定大小的类型。正如我们前面所学，我们不能通过值来传递不确定大小的类型，所以这限制了我们在 trait 中定义方法的种类。应该是不可能写出一个通过取值来获取或返回 `self` 的方法，然而这令人惊讶的是，它的编译:

```rust
trait Trait {
    fn method(self); // compiles
}
```

然而，当我们试图实现该方法时，无论是通过提供一个默认的实现，还是通过实现一个不确定大小类型的 trait，我们都会得到编译错误。

```rust
trait Trait {
    fn method(self) {} // compile error
}

impl Trait for str {
    fn method(self) {} // compile error
}
```

抛出:

```rust
error[E0277]: the size for values of type `Self` cannot be known at compilation time
 --> src/lib.rs:2:15
  |
2 |     fn method(self) {}
  |               ^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `Self`
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
  = note: all local variables must have a statically known size
  = help: unsized locals are gated as an unstable feature
help: consider further restricting `Self`
  |
2 |     fn method(self) where Self: std::marker::Sized {}
  |                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/lib.rs:6:15
  |
6 |     fn method(self) {}
  |               ^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `str`
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
  = note: all local variables must have a statically known size
  = help: unsized locals are gated as an unstable feature
```

如果我们决心通过值来传递 `self`，我们可以通过显式绑定 trait 与 `Sized` 来解决第一个错误。

```rust
trait Trait: Sized {
    fn method(self) {} // compiles
}

impl Trait for str { // compile error
    fn method(self) {}
}
```

现在抛出:

```rust
error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/lib.rs:7:6
  |
1 | trait Trait: Sized {
  |              ----- required by this bound in `Trait`
...
7 | impl Trait for str {
  |      ^^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `str`
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
```

这并没有问题，因为我们知道，当我们将 trait 与 `Sized` 绑定后，我们就不能再为诸如 `str` 这样的不确定大小类型实现它了。另一方面，如果我们真的想为 `str` 实现 trait，另一种解决方案是保留 `?Sized` trait，并通过引用传递 `self`。

```rust
trait Trait {
    fn method(&self) {} // compiles
}

impl Trait for str {
    fn method(&self) {} // compiles
}
```

与其将整个 trait 标记为 `?Sized` 或 `Sized`，我们有更细化和精确的选择，将单个方法标记为 `Sized`，像这样。

```rust
trait Trait {
    fn method(self) where Self: Sized {}
}

impl Trait for str {} // compiles!?

fn main() {
    "str".method(); // compile error
}
```

令人惊讶的是，Rust编译 `impl Trait for str {}` 时没有任何抱怨，但当我们试图在一个不确定大小的类型上调用 `method` 时，它最终还是抓到了错误，所以一切正常。这有点怪异，但为我们提供了一些灵活性，只要我们从不调用 `Sized` 方法，我们就可以用一些 `Sized` 方法为不确定大小的类型实现 trait。

```rust
trait Trait {
    fn method(self) where Self: Sized {}
    fn method2(&self) {}
}

impl Trait for str {} // compiles

fn main() {
    // we never call "method" so no errors
    "str".method2(); // compiles
}
```

现在回到最初的问题，为什么 trait 默认是 `?Sized`？答案是 trait 对象。trait 对象本质上是不确定大小的，因为任何大小的类型都可以实现 trait，因此我们只有在 `Trait: ?Sized` 的情况下，才能为 `dyn Trait` 实现 `Trait`。用代码来说：

```rust
trait Trait: ?Sized {}

// the above is REQUIRED for

impl Trait for dyn Trait {
    // compiler magic here
}

// since `dyn Trait` is unsized

// and now we can use `dyn Trait` in our program

fn function(t: &dyn Trait) {} // compiles
```

如果我们尝试实际编译上述程序，我们会得到：

```rust
error[E0371]: the object type `(dyn Trait + 'static)` automatically implements the trait `Trait`
 --> src/lib.rs:5:1
  |
5 | impl Trait for dyn Trait {
  | ^^^^^^^^^^^^^^^^^^^^^^^^ `(dyn Trait + 'static)` automatically implements trait `Trait`
```

这就是编译器告诉我们要冷静，因为它自动为 `dyn Trait` 提供了 `Trait` 的实现。同样，由于 `dyn Trait` 是不确定大小的，编译器只能在 `Trait: ?Sized` 的情况下提供这个实现。如果我们将 `Trait` 与 `Sized` 绑定，那么 `Trait` 就变成了 "对象不安全" 的了，这意味着我们不能将实现 `Trait` 的类型转为 `dyn Trait` 的 trait 对象。正如预期的那样，这个程序不能编译:

```rust
trait Trait: Sized {}

fn function(t: &dyn Trait) {} // compile error
```

抛出:

```rust
error[E0038]: the trait `Trait` cannot be made into an object
 --> src/lib.rs:3:18
  |
1 | trait Trait: Sized {}
  |       -----  ----- ...because it requires `Self: Sized`
  |       |
  |       this trait cannot be made into an object...
2 |
3 | fn function(t: &dyn Trait) {}
  |                ^^^^^^^^^^ the trait `Trait` cannot be made into an object
```

让我们尝试用 `Sized` 方法制作一个 `?Sized` trait，看看能否将它转一个 trait 对象。

```rust
trait Trait {
    fn method(self) where Self: Sized {}
    fn method2(&self) {}
}

fn function(arg: &dyn Trait) { // compiles
    arg.method(); // compile error
    arg.method2(); // compiles
}
```

正如我们之前看到的那样，只要我们不调用 trait 对象上的 `Sized` 方法，一切都没问题。

**关键要点**
- 所有的 traits 默认都是 `?Sized` 的。
- `Trait: ?Sized` 是 `impl Trait for dyn Trait` 所必需的。
- 我们可以在每个方法的基础上要求 `Self: Sized`。
- 由 `Sized` 绑定的 trait 不能成为 trait 对象。



### trait 对象限制

即使一个 traitt 是对象安全的，也会有一些与大小相关的边缘情况，这些情况限制了哪些类型可以转换为 trait 对象，以及一个 trait 对象可以表示多少个和什么样的 trait。



#### 不能将不确定大小的类型转换为  Trait 对象

```rust
fn generic<T: ToString>(t: T) {}
fn trait_object(t: &dyn ToString) {}

fn main() {
    generic(String::from("String")); // compiles
    generic("str"); // compiles
    trait_object(&String::from("String")); // compiles, unsized coercion
    trait_object("str"); // compile error, unsized coercion impossible
}
```

抛出:

```rust
error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/main.rs:8:18
  |
8 |     trait_object("str"); // compile error
  |                  ^^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `str`
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
  = note: required for the cast to the object type `dyn std::string::ToString`
```

为什么将一个 `&String` 传给一个期望得到 `&dyn ToString` 的函数，是因为类型胁迫。`String` 实现了 `ToString`，我们可以通过不确定大小的胁迫将 `String` 这样的确定大小的类型转换成 `dyn ToString` 这样的不确定大小的类型。`str` 也实现了 `ToString`，将 `str` 转换为 `dyn ToString` 也需要一个不确定大小的胁迫，但 `str` 已经是不确定大小的了！我们如何将一个已经是不确定大小的类型，变成另一个不确定大小的类型？

`&str` 指针是双宽的，存储一个数据指针和数据长度。`&dyn ToString` 指针也是双宽度的，存储一个指向数据的指针和一个指向 vtable 的指针。要把一个 `&str` 胁迫成一个 `&dyn toString`，就需要一个三倍宽度的指针来存储一个指向数据的指针、数据长度和一个指向 vtable 的指针。Rust 不支持三倍宽度指针，所以不可能将一个不确定大小的类型转换成一个 trait 对象。

前面2段用表格总结了一下。

| Type | Pointer to Data | Data Length | Pointer to VTable | Total Width |
|-|-|-|-|-|
| `&String` | ✔️ | ❌ | ❌ | 1 ✔️ |
| `&str` | ✔️ | ✔️ | ❌ | 2 ✔️ |
| `&String as &dyn ToString` | ✔️ | ❌ | ✔️ | 2 ✔️ |
| `&str as &dyn ToString` | ✔️ | ✔️ | ✔️ | 3 ❌ |



#### 不能创建 Multi-Trait 对象

```rust
trait Trait {}
trait Trait2 {}

fn function(t: &(dyn Trait + Trait2)) {}
```

抛出:

```rust
error[E0225]: only auto traits can be used as additional traits in a trait object
 --> src/lib.rs:4:30
  |
4 | fn function(t: &(dyn Trait + Trait2)) {}
  |                      -----   ^^^^^^
  |                      |       |
  |                      |       additional non-auto trait
  |                      |       trait alias used in trait object type (additional use)
  |                      first non-auto trait
  |                      trait alias used in trait object type (first use)
```

请记住，trait 对象指针是双宽度的：存储1个指向数据的指针和另一个指向 vtable 的指针，但这里有2个trait，所以有2个 vtable，这就需要 `&(dyn Trait + Trait2)` 指针是3个宽度。像 `Send` 和 `Sync` 这样的自动 trait 是允许的，因为它们没有方法，因此没有 vtable。

这方面的变通方法是通过使用另一个 trait 来组合 vtable，比如这样。

```rust
trait Trait {
    fn method(&self) {}
}

trait Trait2 {
    fn method2(&self) {}
}

trait Trait3: Trait + Trait2 {}

// auto blanket impl Trait3 for any type that also impls Trait & Trait2
impl<T: Trait + Trait2> Trait3 for T {}

// from `dyn Trait + Trait2` to `dyn Trait3`
fn function(t: &dyn Trait3) {
    t.method(); // compiles
    t.method2(); // compiles
}
```

这个变通方法的一个缺点是，Rust 不支持 supertrait 向上转换。这意味着，如果我们有一个 `dyn Trait3`，我们不能在需要 `dyn Trait` 或 `dyn Trait2` 的地方使用它。这个程序不能编译。

```rust
trait Trait {
    fn method(&self) {}
}

trait Trait2 {
    fn method2(&self) {}
}

trait Trait3: Trait + Trait2 {}

impl<T: Trait + Trait2> Trait3 for T {}

struct Struct;
impl Trait for Struct {}
impl Trait2 for Struct {}

fn takes_trait(t: &dyn Trait) {}
fn takes_trait2(t: &dyn Trait2) {}

fn main() {
    let t: &dyn Trait3 = &Struct;
    takes_trait(t); // compile error
    takes_trait2(t); // compile error
}
```

抛出:

```rust
error[E0308]: mismatched types
  --> src/main.rs:22:17
   |
22 |     takes_trait(t);
   |                 ^ expected trait `Trait`, found trait `Trait3`
   |
   = note: expected reference `&dyn Trait`
              found reference `&dyn Trait3`

error[E0308]: mismatched types
  --> src/main.rs:23:18
   |
23 |     takes_trait2(t);
   |                  ^ expected trait `Trait2`, found trait `Trait3`
   |
   = note: expected reference `&dyn Trait2`
              found reference `&dyn Trait3`
```

这是因为 `dyn Trait3` 是一个不同于 `dyn Trait` 和 `dyn Trait` 的类型，因为它们有不同的 vtable 布局，尽管  `dyn Trait3` 确实包含 `dyn Trait` 和 `dyn Trait2` 的所有方法。这里的变通办法是增加显式转换方法。

```rust
trait Trait {}
trait Trait2 {}

trait Trait3: Trait + Trait2 {
    fn as_trait(&self) -> &dyn Trait;
    fn as_trait2(&self) -> &dyn Trait2;
}

impl<T: Trait + Trait2> Trait3 for T {
    fn as_trait(&self) -> &dyn Trait {
        self
    }
    fn as_trait2(&self) -> &dyn Trait2 {
        self
    }
}

struct Struct;
impl Trait for Struct {}
impl Trait2 for Struct {}

fn takes_trait(t: &dyn Trait) {}
fn takes_trait2(t: &dyn Trait2) {}

fn main() {
    let t: &dyn Trait3 = &Struct;
    takes_trait(t.as_trait()); // compiles
    takes_trait2(t.as_trait2()); // compiles
}
```

这是一个简单而直接的工作方法，似乎是 Rust 编译器可以为我们自动完成的事情。Rust 并不羞于执行类型胁迫，正如我们在 deref 和 unsized 胁迫中所看到的那样，那么为什么没有 trait 向上胁迫呢？这是一个很好的问题，有一个熟悉的答案：Rust核心团队正在研究其他更高优先级和更高影响的功能。很公平。

**关键要点**
- Rust 不支持宽度超过2的指针，所以...
    - 我们不能将不确定大小的类型转换 trait 对象
    - 我们不能有多个 trait 对象，但我们可以通过将多个 trait 强转成一个 trait 来解决这个问题。



### 用户自定义的不确定大小类型

```rust
struct Unsized {
    unsized_field: [i32],
}
```

我们可以通过赋予结构体一个不确定大小的字段来定义一个不确定大小的结构体。不确定大小的结构体只能有1个不确定大小的字段，而且它必须是结构体中的最后一个字段。这是一个要求，这样编译器就可以在编译时确定结构中每个字段的起始偏移量，这对高效快速的字段访问非常重要。此外，使用双宽度指针最多只能跟踪一个不确定大小的字段，因为更多的不确定大小的字段将需要更多的宽度。

那么我们到底该如何实例化这个东西呢？和我们处理任何不确定大小类型的方式一样：先做一个可确定大小的版本，然后胁迫它变成不确定大小的版本。然而，`Unsized` 的定义总是不确定大小的，没有办法制作它的可确定大小版本！唯一的变通办法是使结构体通用化，使它可以存在于确定大小的版本中和不确定大小的版本中。

```rust
struct MaybeSized<T: ?Sized> {
    maybe_sized: T,
}

fn main() {
    // unsized coercion from MaybeSized<[i32; 3]> to MaybeSized<[i32]>
    let ms: &MaybeSized<[i32]> = &MaybeSized { maybe_sized: [1, 2, 3] };
}
```

那么这有什么用处呢？没有什么特别引人注目的，用户定义的不确定大小的类型现在是一个非常半成品的功能，它们的局限性超过了任何好处。这里提到它们纯粹是为了全面性。

**有趣的事实：** `std::fi::OsStr` 和 `std::path::Path` 是标准库中的2个不确定大小的结构，你可能已经在不知不觉中使用过了。

**关键 要点**
- 用户定义的不确定大小类型现在是一个半成品的功能，它们的局限性超过了任何好处



## Zero-Sized 类型

Zero-Sized 乍听起来很奇异，但到处都在使用。


### Unit 类型

最常见的零大小类型是 Unit 类型: `()`. 所有的空块 `{}` 都评估为 `()`，如果块是非空的，但最后一个表达式用分号 `;` 丢弃，那么它也评估为 `()`。例子如下:

```rust
fn main() {
    let a: () = {};
    let b: i32 = {
        5
    };
    let c: () = {
        5;
    };
}
```

每一个没有显式返回类型的函数都会默认返回 `()`。

```rust
// with sugar
fn function() {}

// desugared
fn function() -> () {}
```

由于 `()` 是零字节，所以 `()` 的所有实例都是一样的，这使得 `Default`、`PartialEq` 和 `Ord` 的实现非常简单。

```rust
use std::cmp::Ordering;

impl Default for () {
    fn default() {}
}

impl PartialEq for () {
    fn eq(&self, _other: &()) -> bool {
        true
    }
    fn ne(&self, _other: &()) -> bool {
        false
    }
}

impl Ord for () {
    fn cmp(&self, _other: &()) -> Ordering {
        Ordering::Equal
    }
}
```

编译器理解 `()` 是零大小的，并优化了与 `()` 实例的交互。例如，`Vec<()>` 永远不会进行任何堆分配，从 `Vec` 中推送和弹出 `()` 只是增加和减少它的 `len` 字段。

```rust
fn main() {
    // zero capacity is all the capacity we need to "store" infinitely many ()
    let mut vec: Vec<()> = Vec::with_capacity(0);
    // causes no heap allocations or vec capacity changes
    vec.push(()); // len++
    vec.push(()); // len++
    vec.push(()); // len++
    vec.pop(); // len--
    assert_eq!(2, vec.len());
}
```

上面的例子没有实际应用，但是有没有什么情况下，我们可以有意义地利用上面的想法呢？令人惊讶的是，是的，我们可以通过将 `Value` 设置为 `()`，从 `HashMap<Key，Value>` 中得到一个高效的 `HashSet<Key>` 实现，这正是 Rust 标准库中 `HashSet` 的工作原理。

```rust
// std::collections::HashSet
pub struct HashSet<T> {
    map: HashMap<T, ()>,
}
```

**关键要点**
- ZST 的所有实例都是彼此相等的。
- Rust 编译器知道优化与 ZSTs 的交互。



### 用户自定义的 Unit 结构体

Unit 结构体是指不含任何字段的结构体，如

```rust
struct Struct;
```

属性，使 Unit 结构体比 `()` 更有用。
- 我们可以在自己的 Unit 结构体上实现任何我们想要的 trait，Rust 的 trait 孤儿规则阻止我们实现标准库中定义的 `()` 的 trait。
- 在我们的程序中，Unit 结构体可以被赋予有意义的名称。
- Unit 结构体，就像所有结构体一样，默认情况下是不可复制的，这在我们的程序中可能很重要。



### Never 类型

第二种最常见的 ZST 是 never 类型: `!`。 之所以称为 never 类型，是因为它代表的是永远不会解析到任何值的计算。

`!` 的几个有趣的特性使它不同于 `()`。

- `!` 可以被胁迫成任何其他类型。
- 不可能创建 `!` 的实例。

第一个有趣的属性对人体工程学非常有用，允许我们使用像这样的方便的宏。

```rust
// nice for quick prototyping
fn example<T>(t: &[T]) -> Vec<T> {
    unimplemented!() // ! coerced to Vec<T>
}

fn example2() -> i32 {
    // we know this parse call will never fail
    match "123".parse::<i32>() {
        Some(num) => num,
        None => unreachable!(), // ! coerced to i32
    }
}

fn example3(some_condition: bool) -> &'static str {
    if !some_condition {
        panic!() // ! coerced to &str
    } else {
        "str"
    }
}
```

`break`, `continue` 和 `return` 表达式也拥有类型 `!`:

```rust
fn example() -> i32 {
    // we can set the type of x to anything here
    // since the block never evaluates to any value
    let x: String = {
        return 123 // ! coerced to String
    };
}

fn example2(nums: &[i32]) -> Vec<i32> {
    let mut filtered = Vec::new();
    for num in nums {
        filtered.push(
            if *num < 0 {
                break // ! coerced to i32
            } else if *num % 2 == 0 {
                *num
            } else {
                continue // ! coerced to i32
            }
        );
    }
    filtered
}
```

`!` 的第二个有趣的属性允许我们在类型层面上将某些状态标记为不可能。让我们以这个函数签名为例。

```rust
fn function() -> Result<Success, Error>;
```

我们知道，如果函数返回并成功，`Result` 将包含一些类型为 `Success` 的实例，如果函数出错，`Result` 将包含一些类型为 `Error` 的实例。现在我们来对比一下这个函数的签名。

```rust
fn function() -> Result<Success, !>;
```

我们知道，如果函数返回并且成功了，`Result` 将持有一些类型为 `Success` 的实例，如果出错了...但等等，它永远不会出错，因为不可能创建 `!` 的实例。鉴于上面的函数签名，我们知道这个函数永远不会出错。那这个函数签名呢:

```rust
fn function() -> Result<!, Error>;
```

前面的反义词现在是真的：如果这个函数返回，我们知道它肯定出错了，因为成功是不可能的。

前一个例子的实际应用是 `FromStr` 对 `String` 的实现，因为将 `&str` 转换为 `String` 是不可能失败的。

```rust
#![feature(never_type)]

use std::str::FromStr;

impl FromStr for String {
    type Err = !;
    fn from_str(s: &str) -> Result<String, Self::Err> {
        Ok(String::from(s))
    }
}
```

后一个例子的实际应用是一个运行无限循环的函数，这个函数永远不打算返回，就像服务器响应客户端的请求一样，除非有一些错误。

```rust
#![feature(never_type)]

fn run_server() -> Result<!, ConnectionError> {
    loop {
        let (request, response) = get_request()?;
        let result = request.process();
        response.send(result);
    }
}
```

这个 `feature` 标记是必要的，因为当 never 类型存在并在 Rust 内部工作时，在用户代码中使用它仍然被认为是实验性的。

**要点**
- `!` 可以被胁迫成任何其他类型。
- 不可能创建 `!` 的实例，我们可以用它来标记某些状态，在类型级别上是不可能的。



### 用户定义的伪 Never 类型

虽然不可能定义一个可以强制到任何其他类型的类型，但可以定义一个不可能创建实例的类型，比如一个 `enum`，没有任何变体。

```rust
enum Void {}
```

这使得我们可以从前面的2个例子中移除 `feature` 标记，并使用稳定的 Rust 实现它们。

```rust
enum Void {}

// example 1
impl FromStr for String {
    type Err = Void;
    fn from_str(s: &str) -> Result<String, Self::Err> {
        Ok(String::from(s))
    }
}

// example 2
fn run_server() -> Result<Void, ConnectionError> {
    loop {
        let (request, response) = get_request()?;
        let result = request.process();
        response.send(result);
    }
}
```

这是 Rust 标准库使用的技术，因为 `String` 的 `FromStr` 实现的 `Err` 类型是 `std::convert::Infallible`，它被定义为:

```rust
pub enum Infallible {}
```



### PhantomData

第三种最常用的 ZST 可能是 `PhantomData`。`PhantomData` 是一个零大小的标记结构，它可以用来 "标记" 一个包含的结构体具有某些属性。它和它的自动标记 trait 表亲如 `Sized`、`Send`、`Sync` 等在目的上是相似的，但作为一个标记结构体的使用方式有点不同。对 `PhantomData` 进行彻底的解释并探索它的所有用例不在本文的范围内，所以我们只简单地介绍一个简单的例子。回顾一下前面介绍的这个代码片段。

```rust
#![feature(negative_impls)]

// this type is Send and Sync
struct Struct;

// opt-out of Send trait
impl !Send for Struct {}

// opt-out of Sync trait
impl !Sync for Struct {}
```

很不幸，我们必须使用一个 `feature` 标记，我们是否可以只使用稳定的 Rust 来达到同样的结果？我们已经了解到，一个类型只有当它的所有成员也是 `Send` 和 `Sync` 时才是 `Send` 和 `Sync` 的，所以我们可以像 `Rc<()>` 一样在 `Struct` 中添加一个 `!Send` 和 `!Sync` 成员。

```rust
use std::rc::Rc;

// this type is not Send or Sync
struct Struct {
    // adds 8 bytes to every instance
    _not_send_or_sync: Rc<()>,
}
```

这不太理想，因为它增加了 `Struct` 的每个实例的大小，而且我们现在每次要创建一个 `Struct` 时，还得凭空想象出一个  `Rc<()>`。由于 `PhantomData` 是一个 ZST，它解决了这两个问题。

```rust
use std::rc::Rc;
use std::marker::PhantomData;

type NotSendOrSyncPhantom = PhantomData<Rc<()>>;

// this type is not Send or Sync
struct Struct {
    // adds no additional size to instances
    _not_send_or_sync: NotSendOrSyncPhantom,
}
```

**关键要点**
- `PhantomData` 是一个零大小的标记结构，它可以用来 "标记" 一个包含的结构体具有某些属性。



## 结论

- 只有确定大小类型的实例才能被放置在栈上，也就是说，可以通过值来传递
- 不确定大小类型的实例不能放在栈上，必须通过引用来传递。
- 指向不确定大小类型的指针是双宽度的，因为除了指向数据外，它们还需要做额外的记账工作，以跟踪数据的长度或指向一个  vtable。
- `Sized` 是一个 "自动" 标记 trait。
- 所有的泛型类型参数默认都是自动绑定 `Sized` 的。
- 如果我们有一个泛型函数，它的参数是指针后面的一些 `T`，例如 `&T`、`Box<T>`、`Rc<T>` 等，那么我们几乎总是希望用 `T: ?Sized` 来退出默认的 `Sized` 约束。
- 利用切片和 Rust 的自动类型强制，我们可以编写灵活的 API。
- 所有的 trait 默认为 `Sized`。
- `Trait: ?Sized` 是 `impl Trait for dyn Trait` 所必需的。
- 我们可以根据每个方法要求 `Self: Sized`。
- 由 `Sized ` 绑定的 trait 不能被制作成 trait 对象。
- Rust 不支持宽度超过2的指针，所以...
    - 我们不能将不确定大小的类型转换为 trait 对象
    - 我们不能有多 trait 对象，但我们可以通过将多个 trait 转化成一个 trait 来解决这个问题。
- 用户定义的不确定大小的类型现在是一个半成品的功能，它们的局限性超过了任何好处
- ZST 的所有实例都是彼此相等的。
- Rust 编译器知道优化与 ZSTs 的交互。
- `!` 可以被胁迫成任何其他类型。
- 不可能创建 `!` 的实例，我们可以用它来标记某些状态，在类型级别上是不可能的。
- `PhantomData` 是一个零大小的标记结构，它可以用来 "标记" 一个包含的结构体具有某些属性。



## 讨论

在这里讨论本文:
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-sizedness-in-rust/46293?u=pretzelhammer)
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/hx2jd0/sizedness_in_rust/)
- [Twitter](https://twitter.com/pretzelhammer/status/1286669073137491973)
- [rust subreddit](https://www.reddit.com/r/rust/comments/hxips7/sizedness_in_rust/)
- [Github](https://github.com/pretzelhammer/rust-blog/discussions)


## 通知

当发表下一篇博文时，会收到通知:
- [Following pretzelhammer on Twitter](https://twitter.com/pretzelhammer) or
- Watching this repo's releases (click `Watch` -> click `Custom` -> select `Releases` -> click `Apply`)



## 更多阅读

- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Tour of Rust's Standard Library Traits](./tour-of-rusts-standard-library-traits.md)
- [Learning Rust in 2020](./learning-rust-in-2020.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](./too-many-brainfuck-compilers.md)
