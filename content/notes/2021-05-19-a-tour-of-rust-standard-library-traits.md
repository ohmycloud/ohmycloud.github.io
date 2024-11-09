+++
title = "Rust 的标准库 Trait 之旅"
date = "2021-05-19"
[taxonomies]
  tags = ["rust", "standard library traits"]
+++


## Intro

你有没有想过，这两者之间有什么区别?
- `Deref<Target = T>`, `AsRef<T>` 和 `Borrow<T>`?
- `Clone`, `Copy` 和 `ToOwned`?
- `From<T>` 和 `Into<T>`?
- `TryFrom<&str>` 和 `FromStr`?
- `FnOnce`, `FnMut`, `Fn` 和 `fn`?

或者曾经问过自己这样的问题:
- 在我的 trait 中, 我什么时候使用关联类型, 什么时候使用泛型类型?
- 什么是泛型的 blanket 实现?
- subtrait 和 supertrait 是如何工作的?
- 为什么这个 trait 没有任何方法?

那么这篇文章就是为你准备的! 它回答了以上所有的问题以及更多的问题。我们将一起对 Rust 标准库中所有最流行、最常用的 trait 进行快速飞越之旅!

你可以按顺序逐节阅读本文，也可以跳转到你最感兴趣的 trait，因为每个 trait 部分都有一个链接列表，链接到 “先决知识” 部分，你应该阅读这些链接，以便有足够的背景来理解当前部分的解释。


## Trait 基础

We'll cover just enough of the basics so that the rest of the article can be streamlined without having to repeat the same explanations of the same concepts over and over as they reappear in different traits.

我们将只涉及足够的基础知识，以便文章的其余部分可以精简，而不必在不同的 trait 中重新出现时重复相同的概念解释。

### Trait 项

Trait 项是指作为 trait 声明一部分的任何项。



#### Self

`Self` 总是指实现类型。

```rs
trait Trait {
    // always returns i32
    fn returns_num() -> i32;

    // returns implementing type
    fn returns_self() -> Self;
}

struct SomeType;
struct OtherType;

impl Trait for SomeType {
    fn returns_num() -> i32 {
        5
    }

    // Self == SomeType
    fn returns_self() -> Self {
        SomeType
    }
}

impl Trait for OtherType {
    fn returns_num() -> i32 {
        6
    }

    // Self == OtherType
    fn returns_self() -> Self {
        OtherType
    }
}
```



#### 函数

Trait 函数是任何第一个参数不使用 `self` 关键字的函数。


```rs
trait Default {
    // function
    fn default() -> Self;
}
```

Trait 函数可以通过 trait 或实现类型按照命名空间的方式来调用。


```rs
fn main() {
    let zero: i32 = Default::default();
    let zero = i32::default();
}
```



#### 方法

Trait 方法是指第一个参数使用 `self` 关键字并且类型为 `Self`、`&Self`、`&mut Self` 的任何函数。前面的类型也可以用 `Box`、`Rc`、`Arc` 或 `Pin` 来包装。

```rs
trait Trait {
    // methods
    fn takes_self(self);
    fn takes_immut_self(&self);
    fn takes_mut_self(&mut self);

    // above methods desugared
    fn takes_self(self: Self);
    fn takes_immut_self(self: &Self);
    fn takes_mut_self(self: &mut Self);
}

// example from standard library
trait ToString {
    fn to_string(&self) -> String;
}
```

可以使用实现类型上的点运算符来调用方法。

```rs
fn main() {
    let five = 5.to_string();
}
```

然而，与函数类似，它们也可以通过 trait 或实现类型按照命名空间的方式来调用。

```rs
fn main() {
    let five = ToString::to_string(&5);
    let five = i32::to_string(&5);
}
```



#### 关联类型

Trait 可以有关联类型。当我们需要在函数签名中使用 `Self` 以外的其他类型，但又希望类型由实现者选择，而不是在 trait 声明中硬编码时，这很有用。

```rs
trait Trait {
    type AssociatedType;
    fn func(arg: Self::AssociatedType);
}

struct SomeType;
struct OtherType;

// any type implementing Trait can
// choose the type of AssociatedType

impl Trait for SomeType {
    type AssociatedType = i8; // chooses i8
    fn func(arg: Self::AssociatedType) {}
}

impl Trait for OtherType {
    type AssociatedType = u8; // chooses u8
    fn func(arg: Self::AssociatedType) {}
}

fn main() {
    SomeType::func(-1_i8); // can only call func with i8 on SomeType
    OtherType::func(1_u8); // can only call func with u8 on OtherType
}
```



#### 泛型参数

“泛型参数” 泛指泛型类型参数、泛型 lifetime 参数和泛型常量参数。由于这些说起来都很拗口，所以人们通常把它们缩写为 _"generic types"_, _"lifetimes"_ 和 _"generic consts"_。由于 generic consts 没有在我们将要涉及的任何标准库 trait 中使用，所以它们不在本文的范围之内。

我们可以使用参数来泛型化一个 trait 声明。

```rs
// trait declaration generalized with lifetime & type parameters
trait Trait<'a, T> {
    // signature uses generic type
    fn func1(arg: T);

    // signature uses lifetime
    fn func2(arg: &'a i32);

    // signature uses generic type & lifetime
    fn func3(arg: &'a T);
}

struct SomeType;

impl<'a> Trait<'a, i8> for SomeType {
    fn func1(arg: i8) {}
    fn func2(arg: &'a i32) {}
    fn func3(arg: &'a i8) {}
}

impl<'b> Trait<'b, u8> for SomeType {
    fn func1(arg: u8) {}
    fn func2(arg: &'b i32) {}
    fn func3(arg: &'b u8) {}
}
```

可以为泛型类型提供默认值。最常用的默认值是 `Self`，但任何类型都可以。

```rs
// make T = Self by default
trait Trait<T = Self> {
    fn func(t: T) {}
}

// any type can be used as the default
trait Trait2<T = i32> {
    fn func2(t: T) {}
}

struct SomeType;

// omitting the generic type will
// cause the impl to use the default
// value, which is Self here
impl Trait for SomeType {
    fn func(t: SomeType) {}
}

// default value here is i32
impl Trait2 for SomeType {
    fn func2(t: i32) {}
}

// the default is overridable as we'd expect
impl Trait<String> for SomeType {
    fn func(t: String) {}
}

// overridable here too
impl Trait2<String> for SomeType {
    fn func2(t: String) {}
}
```

除了对 trait 进行参数化外，还可以对单个函数和方法进行参数化。

```rs
trait Trait {
    fn func<'a, T>(t: &'a T);
}
```


#### 泛型类型 vs 关联类型

泛型类型和关联类型都将决定权交给了实现者，让他们决定在 trait 的函数和方法中应该使用哪种具体类型，所以本节试图解释什么时候使用一种类型而不是另一种类型。

一般的经验法则是
- 当每个类型只能有一个 trait 的实现时，使用关联类型。
- 当每个类型可以有许多可能的 trait 的实现时，使用泛型类型。

假设我们想定义一个名为 `Add` 的 trait，它允许我们将值加在一起。下面是一个初始设计和只使用关联类型的实现。


```rs
trait Add {
    type Rhs;
    type Output;
    fn add(self, rhs: Self::Rhs) -> Self::Output;
}

struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Rhs = Point;
    type Output = Point;
    fn add(self, rhs: Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 1 };
    let p2 = Point { x: 2, y: 2 };
    let p3 = p1.add(p2);
    assert_eq!(p3.x, 3);
    assert_eq!(p3.y, 3);
}
```

比方说，我们想给 `Point` 添加和 `i32` 相加的能力，其中 `i32` 将和 `x` 和 `y` 成员相加。


```rs
trait Add {
    type Rhs;
    type Output;
    fn add(self, rhs: Self::Rhs) -> Self::Output;
}

struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Rhs = Point;
    type Output = Point;
    fn add(self, rhs: Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

impl Add for Point { // ❌
    type Rhs = i32;
    type Output = Point;
    fn add(self, rhs: i32) -> Point {
        Point {
            x: self.x + rhs,
            y: self.y + rhs,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 1 };
    let p2 = Point { x: 2, y: 2 };
    let p3 = p1.add(p2);
    assert_eq!(p3.x, 3);
    assert_eq!(p3.y, 3);

    let p1 = Point { x: 1, y: 1 };
    let int2 = 2;
    let p3 = p1.add(int2); // ❌
    assert_eq!(p3.x, 3);
    assert_eq!(p3.y, 3);
}
```

这抛出:

```
error[E0119]: conflicting implementations of trait `Add` for type `Point`:
  --> src/main.rs:23:1
   |
12 | impl Add for Point {
   | ------------------ first implementation here
...
23 | impl Add for Point {
   | ^^^^^^^^^^^^^^^^^^ conflicting implementation for `Point`
```

由于 `Add` trait 没有任何泛型类型的参数化，我们只能对每个类型进行一次实现，这意味着我们只能为 `Rhs` 和 `Output` 选择一次类型！为了允许 `Point` 和 `Point` 相加,以及 `i32` 和 `Point` 相加，我们必须将 `Rhs` 从关联类型重构为泛型类型，这将允许我们用不同的类型参数为 `Rhs` 多次实现 `Point` trait。


```rs
trait Add<Rhs> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}

struct Point {
    x: i32,
    y: i32,
}

impl Add<Point> for Point {
    type Output = Self;
    fn add(self, rhs: Point) -> Self::Output {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

impl Add<i32> for Point { // ✅
    type Output = Self;
    fn add(self, rhs: i32) -> Self::Output {
        Point {
            x: self.x + rhs,
            y: self.y + rhs,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 1 };
    let p2 = Point { x: 2, y: 2 };
    let p3 = p1.add(p2);
    assert_eq!(p3.x, 3);
    assert_eq!(p3.y, 3);

    let p1 = Point { x: 1, y: 1 };
    let int2 = 2;
    let p3 = p1.add(int2); // ✅
    assert_eq!(p3.x, 3);
    assert_eq!(p3.y, 3);
}
```


比方说，我们添加了一个名为 `Line` 的新类型，它包含两个 `Point`，现在在我们的程序中，将两个 `Point` 相加应该产生一个 `Line` 而不是 `Point`。考虑到 `Add` trait 当前的设计，这是不可能的，因为 `Output` 是一个关联类型，但是我们可以通过将 `Output` 从关联类型重构为泛型类型来满足这些新的要求。


```rs
trait Add<Rhs, Output> {
    fn add(self, rhs: Rhs) -> Output;
}

struct Point {
    x: i32,
    y: i32,
}

impl Add<Point, Point> for Point {
    fn add(self, rhs: Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

impl Add<i32, Point> for Point {
    fn add(self, rhs: i32) -> Point {
        Point {
            x: self.x + rhs,
            y: self.y + rhs,
        }
    }
}

struct Line {
    start: Point,
    end: Point,
}

impl Add<Point, Line> for Point { // ✅
    fn add(self, rhs: Point) -> Line {
        Line {
            start: self,
            end: rhs,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 1 };
    let p2 = Point { x: 2, y: 2 };
    let p3: Point = p1.add(p2);
    assert!(p3.x == 3 && p3.y == 3);

    let p1 = Point { x: 1, y: 1 };
    let int2 = 2;
    let p3 = p1.add(int2);
    assert!(p3.x == 3 && p3.y == 3);

    let p1 = Point { x: 1, y: 1 };
    let p2 = Point { x: 2, y: 2 };
    let l: Line = p1.add(p2); // ✅
    assert!(l.start.x == 1 && l.start.y == 1 && l.end.x == 2 && l.end.y == 2)
}
```


那么，上面的 `Add` trait 哪种最好呢？这真的取决于你的程序的要求! 合适的就是最好的。


### 作用域


Trait 项不能使用，除非该 trait 在作用域内。大多数 Rustaceans 在第一次尝试写一个用 I/O 做任何事情的程序时，都会艰难地学会这一点，因为 `Read` 和 `Write` trait 不在标准库的预加载中。


```rs
use std::fs::File;
use std::io;

fn main() -> Result<(), io::Error> {
    let mut file = File::open("Cargo.toml")?;
    let mut buffer = String::new();
    file.read_to_string(&mut buffer)?; // ❌ read_to_string not found in File
    Ok(())
}
```

`read_to_string(buf: &mut String)` 由 `std::io::Read` trait 声明，并由 `std::fs::File` 结构体实现，但为了调用它，`std::io::Read` 必须在作用域内。

```rs
use std::fs::File;
use std::io;
use std::io::Read; // ✅

fn main() -> Result<(), io::Error> {
    let mut file = File::open("Cargo.toml")?;
    let mut buffer = String::new();
    file.read_to_string(&mut buffer)?; // ✅
    Ok(())
}
```

标准库中的 prelude 是标准库中的一个模块，即 `std::prelude::v1`，它在每个其他模块的顶部被自动导入，即 `use std::prelude::v1::*`。因此，下面的 trait 总是在作用域内，我们永远不需要显式导入它们，因为它们是 prelude 的一部分。
- AsMut
- AsRef
- Clone
- Copy
- Default
- Drop
- Eq
- Fn
- FnMut
- FnOnce
- From
- Into
- ToOwned
- IntoIterator
- Iterator
- PartialEq
- PartialOrd
- Send
- Sized
- Sync
- ToString
- Ord



### 派生宏

标准库导出了一些派生宏，如果一个类型的所有成员都实现了某个 trait, 我们可以使用这些宏来快速方便地在这个类型上实现该 trait。这些派生宏以它们所实现的 trait 命名。
- Clone
- Copy
- Debug
- Default
- Eq
- Hash
- Ord
- PartialEq
- PartialOrd

使用示例:

```rs
// macro derives Copy & Clone impl for SomeType
#[derive(Copy, Clone)]
struct SomeType;
```

注意：派生宏只是过程宏，可以做任何事情，没有硬性规定一定要实现一个 trait，也没有规定只有在类型的所有成员都实现一个 trait 的情况下才能工作，这些只是标准库中派生宏所遵循的约定。

### 默认实现

Trait 可以为其函数和方法提供默认的实现。

```rs
trait Trait {
    fn method(&self) {
        println!("default impl");
    }
}

struct SomeType;
struct OtherType;

// use default impl for Trait::method
impl Trait for SomeType {}

impl Trait for OtherType {
    // use our own impl for Trait::method
    fn method(&self) {
        println!("OtherType impl");
    }
}

fn main() {
    SomeType.method(); // prints "default impl"
    OtherType.method(); // prints "OtherType impl"
}
```

如果一些 trait 方法可以只用其他 trait 方法来实现，这就特别方便。

```rs
trait Greet {
    fn greet(&self, name: &str) -> String;
    fn greet_loudly(&self, name: &str) -> String {
        self.greet(name) + "!"
    }
}

struct Hello;
struct Hola;

impl Greet for Hello {
    fn greet(&self, name: &str) -> String {
        format!("Hello {}", name)
    }
    // use default impl for greet_loudly
}

impl Greet for Hola {
    fn greet(&self, name: &str) -> String {
        format!("Hola {}", name)
    }
    // override default impl
    fn greet_loudly(&self, name: &str) -> String {
        let mut greeting = self.greet(name);
        greeting.insert_str(0, "¡");
        greeting + "!"
    }
}

fn main() {
    println!("{}", Hello.greet("John")); // prints "Hello John"
    println!("{}", Hello.greet_loudly("John")); // prints "Hello John!"
    println!("{}", Hola.greet("John")); // prints "Hola John"
    println!("{}", Hola.greet_loudly("John")); // prints "¡Hola John!"
}
```

标准库中的许多 trait 为它们的许多方法提供了默认的实现。

### Generic Blanket Impls

通用全面实现是在泛型类型而不是具体类型上的实现。为了解释为什么以及如何使用，让我们从为数字类型编写一个 `is_even` 方法开始。

```rs
trait Even {
    fn is_even(self) -> bool;
}

impl Even for i8 {
    fn is_even(self) -> bool {
        self % 2_i8 == 0_i8
    }
}

impl Even for u8 {
    fn is_even(self) -> bool {
        self % 2_u8 == 0_u8
    }
}

impl Even for i16 {
    fn is_even(self) -> bool {
        self % 2_i16 == 0_i16
    }
}

// etc

#[test] // ✅
fn test_is_even() {
    assert!(2_i8.is_even());
    assert!(4_u8.is_even());
    assert!(6_i16.is_even());
    // etc
}
```

显然，这是很啰嗦的。而且，我们所有的实现几乎都是一样的。此外，如果 Rust 决定在未来添加更多的数字类型，我们必须记得回到这段代码，并用新的数字类型更新它。我们可以使用一个通用的全面实现来解决所有这些问题。

```rs
use std::fmt::Debug;
use std::convert::TryInto;
use std::ops::Rem;

trait Even {
    fn is_even(self) -> bool;
}

// generic blanket impl
impl<T> Even for T
where
    T: Rem<Output = T> + PartialEq<T> + Sized,
    u8: TryInto<T>,
    <u8 as TryInto<T>>::Error: Debug,
{
    fn is_even(self) -> bool {
        // these unwraps will never panic
        self % 2.try_into().unwrap() == 0.try_into().unwrap()
    }
}

#[test] // ✅
fn test_is_even() {
    assert!(2_i8.is_even());
    assert!(4_u8.is_even());
    assert!(6_i16.is_even());
    // etc
}
```

Unlike default impls, which provide _an_ impl, generic blanket impls provide _the_ impl, so they are not overridable.
与默认实现不同，默认的实现提供了一个实现，而通用的全面实现提供了特定的实现，所以它们是不可覆盖的。


```rs
use std::fmt::Debug;
use std::convert::TryInto;
use std::ops::Rem;

trait Even {
    fn is_even(self) -> bool;
}

impl<T> Even for T
where
    T: Rem<Output = T> + PartialEq<T> + Sized,
    u8: TryInto<T>,
    <u8 as TryInto<T>>::Error: Debug,
{
    fn is_even(self) -> bool {
        self % 2.try_into().unwrap() == 0.try_into().unwrap()
    }
}

impl Even for u8 { // ❌
    fn is_even(self) -> bool {
        self % 2_u8 == 0_u8
    }
}
```

这抛出:

```
error[E0119]: conflicting implementations of trait `Even` for type `u8`:
  --> src/lib.rs:22:1
   |
10 | / impl<T> Even for T
11 | | where
12 | |     T: Rem<Output = T> + PartialEq<T> + Sized,
13 | |     u8: TryInto<T>,
...  |
19 | |     }
20 | | }
   | |_- first implementation here
21 |
22 |   impl Even for u8 {
   |   ^^^^^^^^^^^^^^^^ conflicting implementation for `u8`
```

These impls overlap, hence they conflict, hence Rust rejects the code to ensure trait coherence. Trait coherence is the property that there exists at most one impl of a trait for any given type. The rules Rust uses to enforce trait coherence, the implications of those rules, and workarounds for the implications are outside the scope of this article.

这些实现重叠了，因此它们冲突，因此 Rust 拒绝了确保 trait 一致性的代码。Trait 一致性是指任何给定类型的 trait 最多存在一个实现的属性。Rust 用来强制执行 trait 一致性的规则，这些规则的含义，以及含义的变通方法都不在本文的范围内。

### Subtraits & Supertraits

"subtrait" 中的 "sub" 指的是子集，"supertrait" 中的 "super" 指的是超集。如果我们有一个这样的 trait 声明。

```rs
trait Subtrait: Supertrait {}
```

All of the types which impl `Subtrait` are a subset of all the types which impl `Supertrait`, or to put it in opposite but equivalent terms: all the types which impl `Supertrait` are a superset of all the types which impl `Subtrait`.

Also, the above is just syntax sugar for:
所有实现 `Subtrait` 的类型都是所有实现 `Supertrait` 的类型的子集，或者用相反但等价的词语来表达：所有实现 `Supertrait` 的类型都是所有实现 `Subtrait` 的类型的超集。

另外，上面的代码只是下面这段代码的语法糖:

```rs
trait Subtrait where Self: Supertrait {}
```

这是一个微妙而又重要的区别，要理解的是，约束是在 `Self` 上的，即实现 `Subtrait` 的类型，而不是在 `Subtrait` 本身。后者是没有任何意义的，因为 trait 约束只能应用于具体的类型，这些类型可以实现 trait。Trait 不能实现其他 trait。

```rs
trait Supertrait {
    fn method(&self) {
        println!("in supertrait");
    }
}

trait Subtrait: Supertrait {
    // this looks like it might impl or
    // override Supertrait::method but it
    // does not
    fn method(&self) {
        println!("in subtrait")
    }
}

struct SomeType;

// adds Supertrait::method to SomeType
impl Supertrait for SomeType {}

// adds Subtrait::method to SomeType
impl Subtrait for SomeType {}

// both methods exist on SomeType simultaneously
// neither overriding or shadowing the other

fn main() {
    SomeType.method(); // ❌ ambiguous method call
    // must disambiguate using fully-qualified syntax
    <SomeType as Supertrait>::method(&st); // ✅ prints "in supertrait"
    <SomeType as Subtrait>::method(&st); // ✅ prints "in subtrait"
}
```

Furthermore, there are no rules for how a type must impl both a subtrait and a supertrait. It can use the methods from either in the impl of the other.
此外，没有规定一个类型必须同时实现一个 subtrait 和一个 supertrait。它可以在另一个类型的实现中使用其中一个类型的方法。


```rs
trait Supertrait {
    fn super_method(&mut self);
}

trait Subtrait: Supertrait {
    fn sub_method(&mut self);
}

struct CallSuperFromSub;

impl Supertrait for CallSuperFromSub {
    fn super_method(&mut self) {
        println!("in super");
    }
}

impl Subtrait for CallSuperFromSub {
    fn sub_method(&mut self) {
        println!("in sub");
        self.super_method();
    }
}

struct CallSubFromSuper;

impl Supertrait for CallSubFromSuper {
    fn super_method(&mut self) {
        println!("in super");
        self.sub_method();
    }
}

impl Subtrait for CallSubFromSuper {
    fn sub_method(&mut self) {
        println!("in sub");
    }
}

struct CallEachOther(bool);

impl Supertrait for CallEachOther {
    fn super_method(&mut self) {
        println!("in super");
        if self.0 {
            self.0 = false;
            self.sub_method();
        }
    }
}

impl Subtrait for CallEachOther {
    fn sub_method(&mut self) {
        println!("in sub");
        if self.0 {
            self.0 = false;
            self.super_method();
        }
    }
}

fn main() {
    CallSuperFromSub.super_method(); // prints "in super"
    CallSuperFromSub.sub_method(); // prints "in sub", "in super"

    CallSubFromSuper.super_method(); // prints "in super", "in sub"
    CallSubFromSuper.sub_method(); // prints "in sub"

    CallEachOther(true).super_method(); // prints "in super", "in sub"
    CallEachOther(true).sub_method(); // prints "in sub", "in super"
}
```

Hopefully the examples above show that the relationship between subtraits and supertraits can be complex. Before introducing a mental model that neatly encapsulates all of that complexity let's quickly review and establish the mental model we use for understanding trait bounds on generic types:
希望上面的例子能表明，subtrait 和 supertrait 之间的关系可能很复杂。在介绍一个能整齐地概括所有这些复杂性的心理模型之前，让我们快速回顾并建立我们用于理解泛型上的 trait 约束的心理模型。


```rs
fn function<T: Clone>(t: T) {
    // impl
}
```

Without knowing anything about the impl of this function we could reasonably guess that `t.clone()` gets called at some point because when a generic type is bounded by a trait that strongly implies it has a dependency on the trait. The mental model for understanding the relationship between generic types and their trait bounds is a simple and intuitive one: generic types _depend on_ their trait bounds.

Now let's look the trait declaration for `Copy`:
在不了解这个函数的实现的情况下，我们可以合理地猜测 `t.clone()` 在某些时候会被调用，因为当一个泛型被一个 trait 约束时，强烈地意味着它对 trait 有依赖性。理解泛型与其 trait 约束之间关系的心理模型是一个简单而直观的模型：泛型 “依赖” 其 trait 约束。

现在让我们看看 `Copy` 的 trait 声明。

```rs
trait Copy: Clone {}
```

上面的语法看起来非常类似于在泛型类型上应用 trait 约束的语法，然而 `Copy` 根本不依赖于 `Clone`。我们前面开发的心理模型在这里并不能帮助我们。在我看来，理解 subtrait 和 supertrait 之间关系的最简单、最优雅的心理模型是：subtrait “精炼” 其 supertrait。

“精炼” 这个词故意保持有些模糊，因为它在不同的语境中可以有不同的含义。
- subtrait 可能会使它的 supertrait 的方法更加特化，速度更快，使用更少的内存，例如：`Copy: Clone`
- subtrait 可以对 supertrait 的方法的实现做出额外的保证，例如 `Eq: PartialEq`, `Ord: PartialOrd`, `ExactSizeIterator: Iterator`
- subtrait 可能使 supertrait 的方法更灵活或更容易调用，例如 `FnMut: FnOnce`, `Fn: FnMut
- subtrait 可以扩展一个 supertrait，并添加新的方法，例如 `DoubleEndedIterator: Iterator`, `ExactSizeIterator: Iterator`


### Trait 对象

泛型给了我们编译时的多态性，而 trait 对象给了我们运行时的多态性。我们可以使用 trait 对象来允许函数在运行时动态地返回不同的类型。

```rs
fn example(condition: bool, vec: Vec<i32>) -> Box<dyn Iterator<Item = i32>> {
    let iter = vec.into_iter();
    if condition {
        // Has type:
        // Box<Map<IntoIter<i32>, Fn(i32) -> i32>>
        // But is cast to:
        // Box<dyn Iterator<Item = i32>>
        Box::new(iter.map(|n| n * 2))
    } else {
        // Has type:
        // Box<Filter<IntoIter<i32>, Fn(&i32) -> bool>>
        // But is cast to:
        // Box<dyn Iterator<Item = i32>>
        Box::new(iter.filter(|&n| n >= 2))
    }
}
```

Trait 对象还允许我们在集合中存储异构类型。

```rs
use std::f64::consts::PI;

struct Circle {
    radius: f64,
}

struct Square {
    side: f64
}

trait Shape {
    fn area(&self) -> f64;
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        PI * self.radius * self.radius
    }
}

impl Shape for Square {
    fn area(&self) -> f64 {
        self.side * self.side
    }
}

fn get_total_area(shapes: Vec<Box<dyn Shape>>) -> f64 {
    shapes.into_iter().map(|s| s.area()).sum()
}

fn example() {
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { radius: 1.0 }), // Box<Circle> cast to Box<dyn Shape>
        Box::new(Square { side: 1.0 }), // Box<Square> cast to Box<dyn Shape>
    ];
    assert_eq!(PI + 1.0, get_total_area(shapes)); // ✅
}
```

Trait 对象是不确定大小的，所以它们必须总是在指针后面。我们可以根据类型中是否存在 `dyn` 关键字来区分具体类型和 trait 对象。

```rs
struct Struct;
trait Trait {}

// regular struct
&Struct
Box<Struct>
Rc<Struct>
Arc<Struct>

// trait objects
&dyn Trait
Box<dyn Trait>
Rc<dyn Trait>
Arc<dyn Trait>
```

并非所有的 trait 都可以转换为 trait 对象。如果一个 trait 满足这些要求，它就是对象安全的。
- trait 不需要 `Self: Sized`。
- 所有 trait 的方法都是对象安全的。

如果 trait 方法满足这些要求，它就是对象安全的。
- 方法需要 `Self: Sized` 或
- 该方法只在接收器位置使用 `Self` 类型。

理解为什么要求是这样的，与本文其他部分无关，但如果你仍然好奇，在 [Sizedness in Rust](https://ohmyweekly.github.io/notes/2021-04-11-sizedness-in-rust/) 中会有介绍。

### Marker Traits

标记 trait 是没有 trait 项的 trait。它们的工作是将实现类型 “标记” 为具有某些属性，否则不可能用类型系统来表示。

```rs
// Impling PartialEq for a type promises
// that equality for the type has these properties:
// - symmetry: a == b implies b == a, and
// - transitivity: a == b && b == c implies a == c
// But DOES NOT promise this property:
// - reflexivity: a == a
trait PartialEq {
    fn eq(&self, other: &Self) -> bool;
}

// Eq has no trait items! The eq method is already
// declared by PartialEq, but "impling" Eq
// for a type promises this additional equality property:
// - reflexivity: a == a
trait Eq: PartialEq {}

// f64 impls PartialEq but not Eq because NaN != NaN
// i32 impls PartialEq & Eq because there's no NaNs :)
```

### Auto Traits

自动 trait 是指如果一个类型的所有成员都实现了这个 trait，那么这个 trait 就会被自动实现。“成员” 的含义取决于类型，例如：结构体的字段、枚举的变体、数组的元素、元组的项等等。

所有的自动 trait 都是标记 trait，但不是所有的标记 trait 都是自动 trait。自动 trait 必须是标记 trait，这样编译器就可以为它们提供一个自动的缺省实现，如果它们有任何 trait 项，那就不可能了。

自动 trait 的例子。

```rs
// implemented for types which are safe to send between threads
unsafe auto trait Send {}

// implemented for types whose references are safe to send between threads
unsafe auto trait Sync {}
```

### Unsafe Traits

Trait 可以被标记为不安全，以表明实现该 trait 可能需要不安全的代码。`Send` 和 `Sync` 都被标记为 `unsafe`，因为如果它们没有被自动实现，就意味着它一定包含一些非 `Send` 或非 `Sync` 成员，如果我们想手动标记类型为 `Send` 和 `Sync`，我们作为实现者必须格外小心，以确保没有数据竞争。

```rs
// SomeType is not Send or Sync
struct SomeType {
    not_send_or_sync: *const (),
}

// but if we're confident that our impl doesn't have any data
// races we can explicitly mark it as Send and Sync using unsafe
unsafe impl Send for SomeType {}
unsafe impl Sync for SomeType {}
```

## Auto Traits



### Send & Sync

预备知识
- Marker Traits
- Auto Traits
- Unsafe Traits

```rs
unsafe auto trait Send {}
unsafe auto trait Sync {}
```

如果一个类型是 `Send`，意味着在线程之间发送是安全的。如果一个类型是 `Sync`，这意味着在线程之间共享它的引用是安全的。更准确地说，如果且仅当 `&T` 是 `Send` 时，一些类型 `T` 是 `Sync`。

几乎所有类型都是 `Send` 和 `Sync` 的。唯一值得注意的 `Send` 异常是 `Rc`，唯一值得注意的 `Sync` 异常是 `Rc`、`Cell` 和 `RefCell`。如果我们需要一个 `Rc` 的 `Send` 版本，我们可以使用 `Arc`。如果我们需要 `Cell` 或 `RefCell` 的 `Sync` 版本，我们可以 `Mutex` 或 `RwLock`。虽然如果我们使用 `Mutex` 或 `RwLock` 只是包裹一个原语类型，通常最好使用标准库提供的原子原语类型，如 `AtomicBool`、`AtomicI32`、`AtomicUsize` 等。

几乎所有的类型都是 `Sync`，这可能会让一些人感到惊讶，但是是的，即使对于没有任何内部同步的类型也是如此。这要归功于 Rust 严格的借用规则。

我们可以将同一数据的许多不可变的引用传递给许多线程，而且我们保证不会出现数据竞争，因为只要有任何不可变的引用存在， Rust 就会静态地保证底层数据不能被修改。

```rs
use crossbeam::thread;

fn main() {
    let mut greeting = String::from("Hello");
    let greeting_ref = &greeting;

    thread::scope(|scoped_thread| {
        // spawn 3 threads
        for n in 1..=3 {
            // greeting_ref copied into every thread
            scoped_thread.spawn(move |_| {
                println!("{} {}", greeting_ref, n); // prints "Hello {n}"
            });
        }

        // line below could cause UB or data races but compiler rejects it
        greeting += " world"; // ❌ cannot mutate greeting while immutable refs exist
    });

    // can mutate greeting after every thread has joined
    greeting += " world"; // ✅
    println!("{}", greeting); // prints "Hello world"
}
```

同样，我们可以将单个可变引用传递给一些数据到一个线程，我们可以保证不会出现数据竞争，因为 Rust 静态地保证了别名的可变引用不能存在，底层数据不能通过现有的单个可变引用以外的任何东西进行修改。

```rs
use crossbeam::thread;

fn main() {
    let mut greeting = String::from("Hello");
    let greeting_ref = &mut greeting;

    thread::scope(|scoped_thread| {
        // greeting_ref moved into thread
        scoped_thread.spawn(move |_| {
            *greeting_ref += " world";
            println!("{}", greeting_ref); // prints "Hello world"
        });

        // line below could cause UB or data races but compiler rejects it
        greeting += "!!!"; // ❌ cannot mutate greeting while mutable refs exist
    });

    // can mutate greeting after the thread has joined
    greeting += "!!!"; // ✅
    println!("{}", greeting); // prints "Hello world!!!"
}
```

这就是为什么大多数类型都是 `Sync` 而不需要任何显式同步。如果我们需要在多个线程中同时修改一些数据 `T`，编译器不会让我们这样做，直到我们将数据包裹在 `Arc<Mutex<T>>` 或 `Arc<RwLock<T>>` 中，所以编译器强制要求在需要时使用显式同步。

### Sized

预备知识
- Marker Traits
- Auto Traits

如果一个类型是 `Sized` 的，这意味着它的字节大小在编译时是已知的，并且可以将该类型的实例放在栈上。

类型的大小和它的含义是一个微妙而又巨大的话题，它影响到语言的很多不同方面。它是如此重要，以至于我写了整整一篇文章，叫做 [Sizedness in Rust](https://ohmyweekly.github.io/notes/2021-04-11-sizedness-in-rust/)，我强烈推荐任何想深入了解类型大小的人阅读。我总结一下与本文相关的几个关键内容。

1. 所有的泛型类型都会得到一个隐式的 `Sized` 约束。

```rs
fn func<T>(t: &T) {}

// example above desugared
fn func<T: Sized>(t: &T) {}
```

2. 由于所有泛型类型都有一个隐式的 `Sized` 约束，如果我们想退出这个隐式约束，我们需要使用特殊的 "放宽约束" 语法 `?Sized`，它目前只存在于 `Sized` trait。

```rs
// now T can be unsized
fn func<T: ?Sized>(t: &T) {}
```

3. 所有的 trait 都有一个隐式的 `?Sized` 约束。

```rs
trait Trait {}

// example above desugared
trait Trait: ?Sized {}
```

这是为了让 trait 对象可以实现 trait。同样，所有的琐碎细节都在 [Sizedness in Rust](https://ohmyweekly.github.io/notes/2021-04-11-sizedness-in-rust/) 中。

## General traits

### Default

预备知识
- Self
- Functions
- Derive Macros

```rs
trait Default {
    fn default() -> Self;
}
```

可以构建 `Default` 类型的默认值。

```rs
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

impl Default for Color {
    // default color is black
    fn default() -> Self {
        Color {
            r: 0,
            g: 0,
            b: 0,
        }
    }
}
```

这对快速建立原型很有用，但在任何情况下，我们只需要一个类型的实例，而且对它是什么并不挑剔。

```rs
fn main() {
    // just give me some color!
    let color = Color::default();
}
```

这也是一个我们可能想明确地暴露给我们的函数用户的选项。

```rs
struct Canvas;
enum Shape {
    Circle,
    Rectangle,
}

impl Canvas {
    // let user optionally pass a color
    fn paint(&mut self, shape: Shape, color: Option<Color>) {
        // if no color is passed use the default color
        let color = color.unwrap_or_default();
        // etc
    }
}
```

`Default` 在我们需要构造泛型类型的泛型语境中也很有用。

```rs
fn guarantee_length<T: Default>(mut vec: Vec<T>, min_len: usize) -> Vec<T> {
    for _ in 0..min_len.saturating_sub(vec.len()) {
        vec.push(T::default());
    }
    vec
}
```

我们可以利用 `Default` 类型的另一种方式是使用 Rust 的结构体更新语法对结构体进行部分初始化。我们可以为 `Color` 设置一个 `new` 构造函数，将每个成员作为一个参数。

```rs
impl Color {
    fn new(r: u8, g: u8, b: u8) -> Self {
        Color {
            r,
            g,
            b,
        }
    }
}
```

然而，我们也可以使用方便的构造函数，每个构造函数只接受一个特定的结构体成员，其他结构体成员则使用默认值。

```rs
impl Color {
    fn red(r: u8) -> Self {
        Color {
            r,
            ..Color::default()
        }
    }
    fn green(g: u8) -> Self {
        Color {
            g,
            ..Color::default()
        }
    }
    fn blue(b: u8) -> Self {
        Color {
            b,
            ..Color::default()
        }
    }
}
```

还有一个 `Default` 的派生宏，所以我们可以像这样编写 `Color`。

```rs
// default color is still black
// because u8::default() == 0
#[derive(Default)]
struct Color {
    r: u8,
    g: u8,
    b: u8
}
```



### Clone

预备知识
- Self
- Methods
- Default Impls
- Derive Macros

```rs
trait Clone {
    fn clone(&self) -> Self;

    // provided default impls
    fn clone_from(&mut self, source: &Self);
}
```

我们可以将 `Clone` 类型的不可变引用转换为自有值(owned values)，即 `&T` -> `T`。`Clone` 没有对这种转换的效率做出承诺，所以它可能是缓慢和昂贵的。为了快速地在一个类型上实现 `Clone`，我们可以使用派生宏。

```rs
#[derive(Clone)]
struct SomeType {
    cloneable_member1: CloneableType1,
    cloneable_member2: CloneableType2,
    // etc
}

// macro generates impl below
impl Clone for SomeType {
    fn clone(&self) -> Self {
        SomeType {
            cloneable_member1: self.cloneable_member1.clone(),
            cloneable_member2: self.cloneable_member2.clone(),
            // etc
        }
    }
}
```

`Clone` 也可以在泛型上下文中构建一个类型的实例。下面是上一节中的一个例子，除了使用 `Clone` 而不是 `Default`。

```rs
fn guarantee_length<T: Clone>(mut vec: Vec<T>, min_len: usize, fill_with: &T) -> Vec<T> {
    for _ in 0..min_len.saturating_sub(vec.len()) {
        vec.push(fill_with.clone());
    }
    vec
}
```

人们也经常使用克隆作为逃避的方法，以避免与借用检查器打交道。管理带有引用的结构体可能很有挑战性，但我们可以通过克隆将引用变成自有值(owned values)。

```rs
// oof, we gotta worry about lifetimes 😟
struct SomeStruct<'a> {
    data: &'a Vec<u8>,
}

// now we're on easy street 😎
struct SomeStruct {
    data: Vec<u8>,
}
```

如果我们正在开发的程序的性能不是最重要的，那么我们就不需要为克隆数据而烦恼。Rust 是一种低级别的语言，暴露了很多低级别的细节，所以很容易被过早的优化所吸引，而不是真正解决手头的问题。对于许多程序来说，最好的优先顺序通常是首先建立正确性，其次是优雅性，第三是性能，只有在对程序进行剖析并确定了性能瓶颈之后才关注性能。这是很好的一般性建议，如果它不适用于你的特定程序，你就会知道。



### Copy

预备知识
- Marker Traits
- Subtraits & Supertraits
- Derive Macros

```rs
trait Copy: Clone {}
```

我们复制 `Copy` 类型，例如：`T` -> `T`。`Copy` 承诺复制操作将是一个简单的按位(bitwise)拷贝，所以它将是非常快速和高效的。我们不能自己实现 `Copy`，只有编译器可以提供一个实现，但是我们可以通过使用 `Copy` 派生宏，以及 `Clone` 派生宏来告诉编译器这样做，因为 `Copy` 是 `Clone` 的一个子 trait。

```rs
#[derive(Copy, Clone)]
struct SomeType;
```

`Copy` 完善了(refine) `Clone`。`Clone` 可能是缓慢和昂贵的，但 `Copy` 保证是快速和便宜的，所以 `Copy` 只是一个快速 `Clone`。如果一个类型实现了 `Copy`，这就使得 `Clone` 的实现变得微不足道了。

```rs
// this is what the derive macro generates
impl<T: Copy> Clone for T {
    // the clone method becomes just a copy
    fn clone(&self) -> Self {
        *self
    }
}
```

当一个类型被移动时，实现该类型的 `Copy` 会改变其行为。默认情况下，所有类型都有“移动语义”，但是一旦一个类型实现了 `Copy'，它就会得到“复制语义”。为了解释这两者之间的区别，我们来看看这些简单的场景。

```rs
// a "move", src: !Copy
let dest = src;

// a "copy", src: Copy
let dest = src;
```

在这两种情况下，`dest = src` 对 `src` 的内容进行简单的按位复制，并将结果移动到 `dest` 中，唯一的区别是，在“移动”的情况下，借用检查器使 `src` 变量无效，并确保它以后不会被用于其他地方，而在“复制”的情况下，`src` 仍然有效并可使用。

一言以蔽之。拷贝就“是”移动。移动就“是”拷贝。唯一的区别是借用检查器对它们的处理方式。

关于移动的一个更具体的例子，假设 `src` 是一个 `Vec<i32>`，其内容是这样的。

```rs
{ data: *mut [i32], length: usize, capacity: usize }
```

当我们写下 `dest = src` 时，我们的结果是：

```rs
src = { data: *mut [i32], length: usize, capacity: usize }
dest = { data: *mut [i32], length: usize, capacity: usize }
```

这个时候，`src` 和 `dest` 都有对相同数据的别名可变引用，这是一个大忌，所以借用检查器使 `src` 变量无效，这样它就不能再被使用而不会产生编译错误。

对于一个更具体的拷贝例子，假设 `src` 是一个 `Option<i32>`，它的内容是这样的：

```rs
{ is_valid: bool, data: i32 }
```

现在，当我们写下 `dest = src` 时，我们的结果是：

```rs
src = { is_valid: bool, data: i32 }
dest = { is_valid: bool, data: i32 }
```

这些都是可以同时使用的! 因此 `Option<i32>` 是可以 `Copy` 的。

虽然 `Copy` 可以是一个自动 trait，但 Rust 语言的设计者决定让类型显式地选择复制语义，而不是在类型符合条件时默默地继承复制语义，因为后者会导致令人惊讶的混乱行为，经常导致错误。


### Any

预备知识
- Self
- Generic Blanket Impls
- Subtraits & Supertraits
- Trait Objects

```rs
trait Any: 'static {
    fn type_id(&self) -> TypeId;
}
```

Rust 的多态性风格是参数化的，但如果我们想使用类似于动态类型语言的多态性风格，那么我们可以使用 `Any` trait 来模仿。我们不需要为我们的类型手动实现这个 trait，因为下面这个泛型全面实现已经覆盖了。

```rs
impl<T: 'static + ?Sized> Any for T {
    fn type_id(&self) -> TypeId {
        TypeId::of::<T>()
    }
}
```

我们从 `dyn Any` 中得到 `T` 的方法是通过使用 `downcast_ref::<T>()` 和 `downcast_mut::<T>()` 方法。

```rs
use std::any::Any;

#[derive(Default)]
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn inc(&mut self) {
        self.x += 1;
        self.y += 1;
    }
}

fn map_any(mut any: Box<dyn Any>) -> Box<dyn Any> {
    if let Some(num) = any.downcast_mut::<i32>() {
        *num += 1;
    } else if let Some(string) = any.downcast_mut::<String>() {
        *string += "!";
    } else if let Some(point) = any.downcast_mut::<Point>() {
        point.inc();
    }
    any
}

fn main() {
    let mut vec: Vec<Box<dyn Any>> = vec![
        Box::new(0),
        Box::new(String::from("a")),
        Box::new(Point::default()),
    ];
    // vec = [0, "a", Point { x: 0, y: 0 }]
    vec = vec.into_iter().map(map_any).collect();
    // vec = [1, "a!", Point { x: 1, y: 1 }]
}
```

这个 trait 很少需要使用，因为在大多数情况下，参数化多态性要优于临时多态性，后者也可以用枚举来模拟，因为枚举的类型更安全，需要的迂回更少。例如，我们可以把上面的例子写成这样。

```rs
#[derive(Default)]
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn inc(&mut self) {
        self.x += 1;
        self.y += 1;
    }
}

enum Stuff {
    Integer(i32),
    String(String),
    Point(Point),
}

fn map_stuff(mut stuff: Stuff) -> Stuff {
    match &mut stuff {
        Stuff::Integer(num) => *num += 1,
        Stuff::String(string) => *string += "!",
        Stuff::Point(point) => point.inc(),
    }
    stuff
}

fn main() {
    let mut vec = vec![
        Stuff::Integer(0),
        Stuff::String(String::from("a")),
        Stuff::Point(Point::default()),
    ];
    // vec = [0, "a", Point { x: 0, y: 0 }]
    vec = vec.into_iter().map(map_stuff).collect();
    // vec = [1, "a!", Point { x: 1, y: 1 }]
}
```

尽管 `Any` 很少被需要，但有时使用起来还是很方便的，我们将在后面的“错误处理”部分看到。


## Formatting Traits

我们可以使用 `std::fmt` 中的格式化宏将类型序列化为字符串，其中最著名的是 `println!`。我们可以将格式化参数传递给格式 `str` 中使用的 `{}` 占位符，然后用来选择使用哪个 trait 实现来序列化占位符的参数。

| Trait | Placeholder | Description |
|-------|-------------|-------------|
| `Display` | `{}` | display representation |
| `Debug` | `{:?}` | debug representation |
| `Octal` | `{:o}` | octal representation |
| `LowerHex` | `{:x}` | lowercase hex representation |
| `UpperHex` | `{:X}` | uppercase hex representation |
| `Pointer` | `{:p}` | memory address |
| `Binary` | `{:b}` | binary representation |
| `LowerExp` | `{:e}` | lowercase exponential representation |
| `UpperExp` | `{:E}` | uppercase exponential representation |



### Display & ToString

预备知识
- Self
- Methods
- Generic Blanket Impls

```rs
trait Display {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result;
}
```

`Display` 类型可以被序列化为 `String`，这对程序的终端用户很友好。例如，给 `Point` 实现 `Display`:

```rs
use std::fmt;

#[derive(Default)]
struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    println!("origin: {}", Point::default());
    // prints "origin: (0, 0)"

    // get Point's Display representation as a String
    let stringified_point = format!("{}", Point::default());
    assert_eq!("(0, 0)", stringified_point); // ✅
}
```

除了使用 `format!` 宏来获得一个类型的显示表示为 `String` 之外，我们还可以使用 `ToString` trait。

```rs
trait ToString {
    fn to_string(&self) -> String;
}
```

我们没有必要自己去实现这个 trait。事实上，我们不能这样做，因为下面这个泛型全面实现，对于任何实现 `Display` 的类型，都自动实现 `ToString`。

```rs
impl<T: Display + ?Sized> ToString for T;
```

将 `ToString` 与 `Point` 一起使用。

```rs
#[test] // ✅
fn display_point() {
    let origin = Point::default();
    assert_eq!(format!("{}", origin), "(0, 0)");
}

#[test] // ✅
fn point_to_string() {
    let origin = Point::default();
    assert_eq!(origin.to_string(), "(0, 0)");
}

#[test] // ✅
fn display_equals_to_string() {
    let origin = Point::default();
    assert_eq!(format!("{}", origin), origin.to_string());
}
```



### Debug

预备知识
- Self
- Methods
- Derive Macros
- Display & ToString

```rs
trait Debug {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result;
}
```

`Debug` 与 `Display` 有相同的签名。唯一的区别是，当我们使用 `{:?}` 格式符时，`Debug` 实现被调用。`Debug' 可以被派生。

```rs
use std::fmt;

#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

// derive macro generates impl below
impl fmt::Debug for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
```

为一个类型实现 `Debug` 也允许它在 `dbg!` 宏中使用，这比 `println!` 更有利于临时应急的打印日志。它的一些优点如下:

1. `dbg!` 打印到 stderr 而不是 stdout，所以调试日志很容易与我们程序的实际 stdout 输出分开。
2. `dbg!` 打印传递给它的表达式，以及表达式所评估的值。
3. `dbg!` 拥有其参数的所有权，并返回这些参数，所以你可以在表达式中使用它。

```rs
fn some_condition() -> bool {
    true
}

// no logging
fn example() {
    if some_condition() {
        // some code
    }
}

// println! logging
fn example_println() {
    // 🤦
    let result = some_condition();
    println!("{}", result); // just prints "true"
    if result {
        // some code
    }
}

// dbg! logging
fn example_dbg() {
    // 😍
    if dbg!(some_condition()) { // prints "[src/main.rs:22] some_condition() = true"
        // some code
    }
}
```

唯一的缺点是，`dbg!` 在发布版本中不会被自动剥离，所以如果我们不想在最终的可执行文件中使用它，就必须从我们的代码中手动删除它。



## Operator Traits

Rust 中的所有运算符都与 trait 相关。如果我们想为我们的类型实现运算符，就必须实现相关的 trait。

| Trait(s) | Category | Operator(s) | Description |
|----------|----------|-------------|-------------|
| `Eq`, `PartialEq` | comparison | `==` | equality |
| `Ord`, `PartialOrd` | comparison | `<`, `>`, `<=`, `>=` | comparison |
| `Add` | arithmetic | `+` | addition |
| `AddAssign` | arithmetic | `+=` | addition assignment |
| `BitAnd` | arithmetic | `&` | bitwise AND |
| `BitAndAssign` | arithmetic | `&=` | bitwise assignment |
| `BitXor` | arithmetic | `^` | bitwise XOR |
| `BitXorAssign` | arithmetic | `^=` | bitwise XOR assignment |
| `Div` | arithmetic | `/` | division |
| `DivAssign` | arithmetic | `/=` | division assignment |
| `Mul` | arithmetic | `*` | multiplication |
| `MulAssign` | arithmetic | `*=` | multiplication assignment |
| `Neg` | arithmetic | `-` | unary negation |
| `Not` | arithmetic | `!` | unary logical negation |
| `Rem` | arithmetic | `%` | remainder |
| `RemAssign` | arithmetic | `%=` | remainder assignment |
| `Shl` | arithmetic | `<<` | left shift |
| `ShlAssign` | arithmetic | `<<=` | left shift assignment |
| `Shr` | arithmetic | `>>` | right shift |
| `ShrAssign` | arithmetic | `>>=` | right shift assignment |
| `Sub` | arithmetic | `-` | subtraction |
| `SubAssign` | arithmetic | `-=` | subtraction assignment |
| `Fn` | closure | `(...args)` | immutable closure invocation |
| `FnMut` | closure | `(...args)` | mutable closure invocation |
| `FnOnce` | closure | `(...args)` | one-time closure invocation |
| `Deref` | other | `*` | immutable dereference |
| `DerefMut` | other | `*` | mutable derenence |
| `Drop` | other | - | type destructor |
| `Index` | other | `[]` | immutable index |
| `IndexMut` | other | `[]` | mutable index |
| `RangeBounds` | other | `..` | range |



### Comparison Traits

| Trait(s) | Category | Operator(s) | Description |
|----------|----------|-------------|-------------|
| `Eq`, `PartialEq` | comparison | `==` | equality |
| `Ord`, `PartialOrd` | comparison | `<`, `>`, `<=`, `>=` | comparison |


#### PartialEq & Eq

预备知识
- Self
- Methods
- Generic Parameters
- Default Impls
- Generic Blanket Impls
- Marker Traits
- Subtraits & Supertraits
- Sized

```rs
trait PartialEq<Rhs = Self>
where
    Rhs: ?Sized,
{
    fn eq(&self, other: &Rhs) -> bool;

    // provided default impls
    fn ne(&self, other: &Rhs) -> bool;
}
```

`PartialEq<Rhs>` 类型可以使用 `==` 运算符检查是否与 `Rhs` 类型相等。

所有的 `PartialEq<Rhs>` 实现必须确保相等是对称的和传递的。这意味着对于所有的 `a`, `b`, 和 `c`:

- `a == b` 意味着 `b == a` (对称性)
- `a == b && b == c` 意味着 `a == c` (传递性)

默认情况下 `Rhs = Self`，因为我们几乎总是想把一个类型的实例相互比较，而不是与不同类型的实例比较。这也自动保证了我们的实现是对称的和传递的。

```rs
struct Point {
    x: i32,
    y: i32
}

// Rhs == Self == Point
impl PartialEq for Point {
    // impl automatically symmetric & transitive
    fn eq(&self, other: &Point) -> bool {
        self.x == other.x && self.y == other.y
    }
}
```

如果一个类型的所有成员都实现了 `PartialEq'，那么它可以被派生。

```rs
#[derive(PartialEq)]
struct Point {
    x: i32,
    y: i32
}

#[derive(PartialEq)]
enum Suit {
    Spade,
    Heart,
    Club,
    Diamond,
}
```

一旦为我们的类型实现了 `PartialEq`，我们也可以免费得到我们类型的引用之间的相等性比较，这要感谢这些泛型全面实现。

```rs
// this impl only gives us: Point == Point
#[derive(PartialEq)]
struct Point {
    x: i32,
    y: i32
}

// all of the generic blanket impls below
// are provided by the standard library

// this impl gives us: &Point == &Point
impl<A, B> PartialEq<&'_ B> for &'_ A
where A: PartialEq<B> + ?Sized, B: ?Sized;

// this impl gives us: &mut Point == &Point
impl<A, B> PartialEq<&'_ B> for &'_ mut A
where A: PartialEq<B> + ?Sized, B: ?Sized;

// this impl gives us: &Point == &mut Point
impl<A, B> PartialEq<&'_ mut B> for &'_ A
where A: PartialEq<B> + ?Sized, B: ?Sized;

// this impl gives us: &mut Point == &mut Point
impl<A, B> PartialEq<&'_ mut B> for &'_ mut A
where A: PartialEq<B> + ?Sized, B: ?Sized;
```

由于这个 trait 是泛型化的，我们可以定义不同类型之间的相等性。标准库利用这一点，允许检查许多类似字符串的类型，如`String`、`&str`、`PathBuf`、`&Path`、`OsString`、`&OsStr` 等之间的相等性。

一般来说，我们只应该在不同类型之间实现相等性关系，如果它们实现同一种数据，并且类型之间的唯一区别是它们如何表示数据或如何允许与数据进行交互。

这里有一个可爱但糟糕的例子，说明有人可能会被诱惑实现 `PartialEq` 来检查不符合上述标准的不同类型之间的相等。

```rs
#[derive(PartialEq)]
enum Suit {
    Spade,
    Club,
    Heart,
    Diamond,
}

#[derive(PartialEq)]
enum Rank {
    Ace,
    Two,
    Three,
    Four,
    Five,
    Six,
    Seven,
    Eight,
    Nine,
    Ten,
    Jack,
    Queen,
    King,
}

#[derive(PartialEq)]
struct Card {
    suit: Suit,
    rank: Rank,
}

// check equality of Card's suit
impl PartialEq<Suit> for Card {
    fn eq(&self, other: &Suit) -> bool {
        self.suit == *other
    }
}

// check equality of Card's rank
impl PartialEq<Rank> for Card {
    fn eq(&self, other: &Rank) -> bool {
        self.rank == *other
    }
}

fn main() {
    let AceOfSpades = Card {
        suit: Suit::Spade,
        rank: Rank::Ace,
    };
    assert!(AceOfSpades == Suit::Spade); // ✅
    assert!(AceOfSpades == Rank::Ace); // ✅
}
```

这很有效，而且有点道理。一张黑桃A的牌既是A又是黑桃，如果我们要写一个处理扑克牌的库，那么我们想让它简单方便地单独检查一张牌的花色和等级是合理的。然而，我们还缺少一些东西：对称性。 我们可以 `Card == Suit` 和 `Card == Rank`，但我们不能 `Suit == Card` 或 `Rank == Card`，所以让我们解决这个问题。

```rs
// check equality of Card's suit
impl PartialEq<Suit> for Card {
    fn eq(&self, other: &Suit) -> bool {
        self.suit == *other
    }
}

// added for symmetry
impl PartialEq<Card> for Suit {
    fn eq(&self, other: &Card) -> bool {
        *self == other.suit
    }
}

// check equality of Card's rank
impl PartialEq<Rank> for Card {
    fn eq(&self, other: &Rank) -> bool {
        self.rank == *other
    }
}

// added for symmetry
impl PartialEq<Card> for Rank {
    fn eq(&self, other: &Card) -> bool {
        *self == other.rank
    }
}
```

我们有对称性! 太好了。增加对称性只是打破了传递性！这是不可能的。哎呀。现在可以这样了：

```rs
fn main() {
    // Ace of Spades
    let a = Card {
        suit: Suit::Spade,
        rank: Rank::Ace,
    };
    let b = Suit::Spade;
    // King of Spades
    let c = Card {
        suit: Suit::Spade,
        rank: Rank::King,
    };
    assert!(a == b && b == c); // ✅
    assert!(a == c); // ❌
}
```

实现 `PartialEq` 以检查不同类型之间的相等关系的一个好例子是一个处理距离的程序，它使用不同的类型来代表不同的测量单位。

```rs
#[derive(PartialEq)]
struct Foot(u32);

#[derive(PartialEq)]
struct Yard(u32);

#[derive(PartialEq)]
struct Mile(u32);

impl PartialEq<Mile> for Foot {
    fn eq(&self, other: &Mile) -> bool {
        self.0 == other.0 * 5280
    }
}

impl PartialEq<Foot> for Mile {
    fn eq(&self, other: &Foot) -> bool {
        self.0 * 5280 == other.0
    }
}

impl PartialEq<Mile> for Yard {
    fn eq(&self, other: &Mile) -> bool {
        self.0 == other.0 * 1760
    }
}

impl PartialEq<Yard> for Mile {
    fn eq(&self, other: &Yard) -> bool {
        self.0 * 1760 == other.0
    }
}

impl PartialEq<Foot> for Yard {
    fn eq(&self, other: &Foot) -> bool {
        self.0 * 3 == other.0
    }
}

impl PartialEq<Yard> for Foot {
    fn eq(&self, other: &Yard) -> bool {
        self.0 == other.0 * 3
    }
}

fn main() {
    let a = Foot(5280);
    let b = Yard(1760);
    let c = Mile(1);

    // symmetry
    assert!(a == b && b == a); // ✅
    assert!(b == c && c == b); // ✅
    assert!(a == c && c == a); // ✅

    // transitivity
    assert!(a == b && b == c && a == c); // ✅
    assert!(c == b && b == a && c == a); // ✅
}
```

`Eq` 是一个标记 trait，是 `PartialEq<Self>` 的子 trait。

```rs
trait Eq: PartialEq<Self> {}
```

如果我们为一个类型实现 `Eq`，在 `PartialEq` 所要求的对称性和传递性的基础上，我们还保证了自反性，即 对所有 `a`, `a == a`。在这个意义上，`Eq` 完善了 `PartialEq`，因为它代表了一个更严格的相等性版本。如果一个类型的所有成员都是`Eq` 的，那么 `Eq` 实现就可以为该类型派生。

浮点类型是 `PartialEq` 的，但不是 `Eq` 的，因为 `NaN != NaN`。几乎所有其他的 `PartialEq` 类型都是 `Eq`，当然，除非它们包含浮点。

一旦一个类型实现了 `PartialEq` 和 `Debug`，我们就可以在 `assert_eq!` 宏中使用它。我们也可以比较 `PartialEq` 类型的集合。

```rs
#[derive(PartialEq, Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn example_assert(p1: Point, p2: Point) {
    assert_eq!(p1, p2);
}

fn example_compare_collections<T: PartialEq>(vec1: Vec<T>, vec2: Vec<T>) {
    // if T: PartialEq this now works!
    if vec1 == vec2 {
        // some code
    } else {
        // other code
    }
}
```



#### Hash

预备知识
- Self
- Methods
- Generic Parameters
- Default Impls
- Derive Macros
- PartialEq & Eq

```rs
trait Hash {
    fn hash<H: Hasher>(&self, state: &mut H);

    // provided default impls
    fn hash_slice<H: Hasher>(data: &[Self], state: &mut H);
}
```

这个 trait 与任何运算符无关，但谈论它的最好时机是在 `PartialEq` & `Eq` 之后，所以它在这里。`Hash` 类型可以使用 `Hasher` 进行散列。

```rs
use std::hash::Hasher;
use std::hash::Hash;

struct Point {
    x: i32,
    y: i32,
}

impl Hash for Point {
    fn hash<H: Hasher>(&self, hasher: &mut H) {
        hasher.write_i32(self.x);
        hasher.write_i32(self.y);
    }
}
```

有一个派生宏，它生成的实现与上述相同。

```rs
#[derive(Hash)]
struct Point {
    x: i32,
    y: i32,
}
```

如果一个类型同时实现了 `Hash` 和 `Eq`，这些实现必须相互一致，即对于所有的 `a` 和 `b`，如果 `a == b`，那么 `a.hash() == b.hash()`。所以我们应该总是使用派生宏来实现两者，或者手动实现两者，但不能混合使用，否则就有可能破坏上述不变性。

为一个类型实现 `Eq` 和 `Hash` 的主要好处是，它允许我们将该类型作为键存储在 `HashMap` 和 `HashSet` 中。

```rs
use std::collections::HashSet;

// now our type can be stored
// in HashSets and HashMaps!
#[derive(PartialEq, Eq, Hash)]
struct Point {
    x: i32,
    y: i32,
}

fn example_hashset() {
    let mut points = HashSet::new();
    points.insert(Point { x: 0, y: 0 }); // ✅
}
```



#### PartialOrd & Ord

预备知识
- Self
- Methods
- Generic Parameters
- Default Impls
- Subtraits & Supertraits
- Derive Macros
- Sized
- PartialEq & Eq

```rs
enum Ordering {
    Less,
    Equal,
    Greater,
}

trait PartialOrd<Rhs = Self>: PartialEq<Rhs>
where
    Rhs: ?Sized,
{
    fn partial_cmp(&self, other: &Rhs) -> Option<Ordering>;

    // provided default impls
    fn lt(&self, other: &Rhs) -> bool;
    fn le(&self, other: &Rhs) -> bool;
    fn gt(&self, other: &Rhs) -> bool;
    fn ge(&self, other: &Rhs) -> bool;
}
```

`PartialOrd<Rhs>` 类型可以使用 `<`, `<=`, `>`, 和 `>=` 运算符与 `Rhs` 类型进行比较。

所有的 `PartialOrd` 实现必须确保比较是不对称的和传递的。这意味着对于所有的 `a`, `b`, 和 `c`:

- `a < b` 意味着 `!(a > b)` (不对称性)
- `a < b && b < c` 意味着 `a < c` (传递性)

`PartialOrd` 是 `PartialEq` 的一个子 trait，它们的实现必须总是相互一致。

```rs
fn must_always_agree<T: PartialOrd + PartialEq>(t1: T, t2: T) {
    assert_eq!(t1.partial_cmp(&t2) == Some(Ordering::Equal), t1 == t2);
}
```

`PartialOrd` 是对 `PartialEq` 的细化，当比较 `PartialEq` 类型时，我们可以检查它们是否相等，但当比较 `PartialOrd` 类型时，我们可以检查它们是否相等，如果它们不相等，我们可以检查它们是否不相等，因为第一项小于或大于第二项。

默认情况下 `Rhs = Self`，因为我们几乎总是想把一个类型的实例相互比较，而不是和不同类型的实例比较。这也自动保证了我们的实现是对称的和传递的。

```rs
use std::cmp::Ordering;

#[derive(PartialEq, PartialOrd)]
struct Point {
    x: i32,
    y: i32
}

// Rhs == Self == Point
impl PartialOrd for Point {
    // impl automatically symmetric & transitive
    fn partial_cmp(&self, other: &Point) -> Option<Ordering> {
        Some(match self.x.cmp(&other.x) {
            Ordering::Equal => self.y.cmp(&other.y),
            ordering => ordering,
        })
    }
}
```

如果一个类型的所有成员都实现了 `PartialOrd`，那么它可以被派生。

```rs
#[derive(PartialEq, PartialOrd)]
struct Point {
    x: i32,
    y: i32,
}

#[derive(PartialEq, PartialOrd)]
enum Stoplight {
    Red,
    Yellow,
    Green,
}
```

`PartialOrd` 派生宏基于其成员的字母顺序对类型进行排序。

```rs
// generates PartialOrd impl which orders
// Points based on x member first and
// y member second because that's the order
// they appear in the source code
#[derive(PartialOrd, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

// generates DIFFERENT PartialOrd impl
// which orders Points based on y member
// first and x member second
#[derive(PartialOrd, PartialEq)]
struct Point {
    y: i32,
    x: i32,
}
```

`Ord` is a subtrait of `Eq` and `PartialOrd<Self>`:
`Ord` 是 `Eq` 和 `PartialOrd<Self>` 的子 trait。


```rs
trait Ord: Eq + PartialOrd<Self> {
    fn cmp(&self, other: &Self) -> Ordering;

    // provided default impls
    fn max(self, other: Self) -> Self;
    fn min(self, other: Self) -> Self;
    fn clamp(self, min: Self, max: Self) -> Self;
}
```

如果我们为一个类型实现 `Ord`，在 `PartialOrd` 所要求的不对称性和传递性的基础上，我们还保证不对称性是完全的，即对于任何给定的 `a` 和 `b`，`a == b` 或 `a > b` 中只有一个是真的。在这个意义上，`Ord` 完善了 `Eq` 和 `PartialOrd`，因为它代表了一个更严格的比较版本。如果一个类型实现了 `Ord`，我们就可以用这个实现来实现 `PartialOrd`、`PartialEq` 和 `Eq`。

```rs
use std::cmp::Ordering;

// of course we can use the derive macros here
#[derive(Ord, PartialOrd, Eq, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

// note: as with PartialOrd, the Ord derive macro
// orders a type based on the lexicographical order
// of its members

// but here's the impls if we wrote them out by hand
impl Ord for Point {
    fn cmp(&self, other: &Self) -> Ordering {
        match self.x.cmp(&self.y) {
            Ordering::Equal => self.y.cmp(&self.y),
            ordering => ordering,
        }
    }
}
impl PartialOrd for Point {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}
impl PartialEq for Point {
    fn eq(&self, other: &Self) -> bool {
        self.cmp(other) == Ordering::Equal
    }
}
impl Eq for Point {}
```

浮点数实现了 `PartialOrd`，但不是 `Ord`，因为 `NaN < 0 == false` 和 `NaN >= 0 == false` 同时为真。几乎所有其他的 `PartialOrd` 类型都是 `Ord`，当然，除非它们包含浮点数。

一旦一个类型被认为是 `Ord` 的，我们就可以将其存储在 `BTreeMap` 和 `BTreeSet` 中，并且可以使用 `sort()` 方法对其进行排序，以及对数组、`Vec` 和 `VecDeque` 等任何类型的切片进行解引用。

```rs
use std::collections::BTreeSet;

// now our type can be stored
// in BTreeSets and BTreeMaps!
#[derive(Ord, PartialOrd, PartialEq, Eq)]
struct Point {
    x: i32,
    y: i32,
}

fn example_btreeset() {
    let mut points = BTreeSet::new();
    points.insert(Point { x: 0, y: 0 }); // ✅
}

// we can also .sort() Ord types in collections!
fn example_sort<T: Ord>(mut sortable: Vec<T>) -> Vec<T> {
    sortable.sort();
    sortable
}
```



### Arithmetic Traits

| Trait(s) | Category | Operator(s) | Description |
|----------|----------|-------------|-------------|
| `Add` | arithmetic | `+` | addition |
| `AddAssign` | arithmetic | `+=` | addition assignment |
| `BitAnd` | arithmetic | `&` | bitwise AND |
| `BitAndAssign` | arithmetic | `&=` | bitwise assignment |
| `BitXor` | arithmetic | `^` | bitwise XOR |
| `BitXorAssign` | arithmetic | `^=` | bitwise XOR assignment |
| `Div` | arithmetic | `/` | division |
| `DivAssign` | arithmetic | `/=` | division assignment |
| `Mul` | arithmetic | `*` | multiplication |
| `MulAssign` | arithmetic | `*=` | multiplication assignment |
| `Neg` | arithmetic | `-` | unary negation |
| `Not` | arithmetic | `!` | unary logical negation |
| `Rem` | arithmetic | `%` | remainder |
| `RemAssign` | arithmetic | `%=` | remainder assignment |
| `Shl` | arithmetic | `<<` | left shift |
| `ShlAssign` | arithmetic | `<<=` | left shift assignment |
| `Shr` | arithmetic | `>>` | right shift |
| `ShrAssign` | arithmetic | `>>=` | right shift assignment |
| `Sub` | arithmetic | `-` | subtraction |
| `SubAssign` | arithmetic | `-=` | subtraction assignment |


仔细研究所有这些将是非常多余的。反正大多数只适用于数字类型。我们只讨论 `Add` 和 `AddAssign`，因为 `+` 操作符通常被重载来做其他事情，如向集合添加项目或将事物串联起来，这样我们就能覆盖最有趣的地方，而不会重复。


#### Add & AddAssign

预备知识
- Self
- Methods
- Associated Types
- Generic Parameters
- Generic Types vs Associated Types
- Derive Macros

```rs
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

`Add<Rhs, Output = T>` 类型可以和 `Rhs` 类型相加，并将产生 `T` 作为输出。

例子 `Add<Point, Output = Point>` 是针对 `Point` 实现的。

```rs
#[derive(Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;
    fn add(self, rhs: Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let p3 = p1 + p2;
    assert_eq!(p3.x, p1.x + p2.x); // ✅
    assert_eq!(p3.y, p1.y + p2.y); // ✅
}
```

但是如果我们只有对 `Point` 的引用呢？那我们还能让它们相加吗？让我们试试。

```rs
fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let p3 = &p1 + &p2; // ❌
}
```

不幸的是没有。编译器会抛出异常：

```
error[E0369]: cannot add `&Point` to `&Point`
  --> src/main.rs:50:25
   |
50 |     let p3: Point = &p1 + &p2;
   |                     --- ^ --- &Point
   |                     |
   |                     &Point
   |
   = note: an implementation of `std::ops::Add` might be missing for `&Point`
```

在 Rust 的类型系统中，对于某些类型 `T` 来说，`T`、`&T` 和 `&mut T` 都被视为唯一的不同类型，这意味着我们必须为它们分别提供 trait 实现。让我们为 `&Point` 定义一个 `Add` 实现：

```rs
impl Add for &Point {
    type Output = Point;
    fn add(self, rhs: &Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let p3 = &p1 + &p2; // ✅
    assert_eq!(p3.x, p1.x + p2.x); // ✅
    assert_eq!(p3.y, p1.y + p2.y); // ✅
}
```

然而，有些事情还是感觉不大对劲。我们有两个独立的 `Add` 实现，分别用于 `Point` 和 `&Point`，它们目前做的是同样的事情，但不能保证将来也会这样做。例如，我们决定当我们把两个 `Point` 相加时，我们想创建一个包含这两个 `Point` 的 `Line`，而不是创建一个新的 `Point`，我们会像这样更新我们的 `Add` 程序：

```rs
use std::ops::Add;

#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

#[derive(Copy, Clone)]
struct Line {
    start: Point,
    end: Point,
}

// we updated this impl
impl Add for Point {
    type Output = Line;
    fn add(self, rhs: Point) -> Line {
        Line {
            start: self,
            end: rhs,
        }
    }
}

// but forgot to update this impl, uh oh!
impl Add for &Point {
    type Output = Point;
    fn add(self, rhs: &Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let line: Line = p1 + p2; // ✅

    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let line: Line = &p1 + &p2; // ❌ expected Line, found Point
}
```

我们目前对 `&Point` 的 `Add` 实现造成了不必要的维护负担，我们希望这个实现与 `Point` 的实现相匹配，而不必在每次改变 `Point` 的实现时都要手动更新。我们希望尽可能地保持我们的代码是 DRY（Don't Repeat Yourself）。幸运的是这是可以实现的。

```rs
// updated, DRY impl
impl Add for &Point {
    type Output = <Point as Add>::Output;
    fn add(self, rhs: &Point) -> Self::Output {
        Point::add(*self, *rhs)
    }
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let line: Line = p1 + p2; // ✅

    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    let line: Line = &p1 + &p2; // ✅
}
```

`AddAssign<Rhs>` 类型允许我们相加并分配 `Rhs` 类型给它们。Trait 声明如下：

```rs
trait AddAssign<Rhs = Self> {
    fn add_assign(&mut self, rhs: Rhs);
}
```

为 `Point` 和 `&Point` 类型的实现的例子如下：

```rs
use std::ops::AddAssign;

#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32
}

impl AddAssign for Point {
    fn add_assign(&mut self, rhs: Point) {
        self.x += rhs.x;
        self.y += rhs.y;
    }
}

impl AddAssign<&Point> for Point {
    fn add_assign(&mut self, rhs: &Point) {
        Point::add_assign(self, *rhs);
    }
}

fn main() {
    let mut p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 3, y: 4 };
    p1 += &p2;
    p1 += p2;
    assert!(p1.x == 7 && p1.y == 10);
}
```



### 闭包 Traits

| Trait(s) | Category | Operator(s) | Description |
|----------|----------|-------------|-------------|
| `Fn` | closure | `(...args)` | immutable closure invocation |
| `FnMut` | closure | `(...args)` | mutable closure invocation |
| `FnOnce` | closure | `(...args)` | one-time closure invocation |



#### FnOnce, FnMut, & Fn

预备知识
- Self
- Methods
- Associated Types
- Generic Parameters
- Generic Types vs Associated Types
- Subtraits & Supertraits

```rs
trait FnOnce<Args> {
    type Output;
    fn call_once(self, args: Args) -> Self::Output;
}

trait FnMut<Args>: FnOnce<Args> {
    fn call_mut(&mut self, args: Args) -> Self::Output;
}

trait Fn<Args>: FnMut<Args> {
    fn call(&self, args: Args) -> Self::Output;
}
```

虽然这些 trait 存在，但在稳定的 Rust 中，我们不可能为自己的类型实现这些特性。我们唯一能创建的实现这些 trait 的类型是闭包。根据闭包从其环境中捕获的内容，决定了它是实现了 `FnOnce`、`FnMut` 还是 `Fn`。

`FnOnce` 闭包只能被调用一次，因为它在执行中会消耗一些值。

```rs
fn main() {
    let range = 0..10;
    let get_range_count = || range.count();
    assert_eq!(get_range_count(), 10); // ✅
    get_range_count(); // ❌
}
```

迭代器上的 `.count()` 方法会消耗迭代器，所以它只能被调用一次。因此，我们的闭包只能被调用一次。这就是为什么当我们试图第二次调用它时，会出现这个错误。

```
error[E0382]: use of moved value: `get_range_count`
 --> src/main.rs:5:5
  |
4 |     assert_eq!(get_range_count(), 10);
  |                ----------------- `get_range_count` moved due to this call
5 |     get_range_count();
  |     ^^^^^^^^^^^^^^^ value used here after move
  |
note: closure cannot be invoked more than once because it moves the variable `range` out of its environment
 --> src/main.rs:3:30
  |
3 |     let get_range_count = || range.count();
  |                              ^^^^^
note: this value implements `FnOnce`, which causes it to be moved when called
 --> src/main.rs:4:16
  |
4 |     assert_eq!(get_range_count(), 10);
  |                ^^^^^^^^^^^^^^^
```

`FnMut` 闭包可以被多次调用，也可以改变它从环境中捕获的变量。我们可以说 `FnMut` 闭包是执行副作用的，或者说是有状态的。下面是一个闭包的例子，它通过跟踪到目前为止看到的最小值，从迭代器中过滤出所有非升序的值。

```rs
fn main() {
    let nums = vec![0, 4, 2, 8, 10, 7, 15, 18, 13];
    let mut min = i32::MIN;
    let ascending = nums.into_iter().filter(|&n| {
        if n <= min {
            false
        } else {
            min = n;
            true
        }
    }).collect::<Vec<_>>();
    assert_eq!(vec![0, 4, 8, 10, 15, 18], ascending); // ✅
}
```

`FnMut` 完善了 `FnOnce`，即 `FnOnce` 需要取得其参数的所有权，只能调用一次，但 `FnMut` 只需要取得可变的引用，可以多次调用。`FnMut` 可以在任何可以使用 `FnOnce` 的地方使用。

`Fn` 闭包可以被多次调用，并且不改变它从环境中捕获的任何变量。我们可以说 `Fn` 闭包没有副作用或无状态。下面是一个闭包的例子，它过滤掉了所有小于它从环境中捕获的迭代器中的某个栈变量的值。

```rs
fn main() {
    let nums = vec![0, 4, 2, 8, 10, 7, 15, 18, 13];
    let min = 9;
    let greater_than_9 = nums.into_iter().filter(|&n| n > min).collect::<Vec<_>>();
    assert_eq!(vec![10, 15, 18, 13], greater_than_9); // ✅
}
```

`Fn` 细化了 `FnMut`，即 `FnMut` 需要可变的引用并可多次调用，但 `Fn` 只需要不可变的引用并可多次调用。`Fn` 可以用在任何可以使用 `FnMut` 的地方，包括可以使用 `FnOnce` 的地方。

如果一个闭包没有从它的环境中捕获任何东西，那么从技术上讲，它不是一个闭包，而只是一个匿名声明的内联函数，并且可以作为一个普通的函数指针被转换、使用和传递，也就是 `fn`。函数指针可以在任何可以使用 `Fn` 的地方使用，这包括可以使用 `FnMut` 和 `FnOnce `的地方。

```rs
fn add_one(x: i32) -> i32 {
    x + 1
}

fn main() {
    let mut fn_ptr: fn(i32) -> i32 = add_one;
    assert_eq!(fn_ptr(1), 2); // ✅

    // capture-less closure cast to fn pointer
    fn_ptr = |x| x + 1; // same as add_one
    assert_eq!(fn_ptr(1), 2); // ✅
}
```

传递普通函数指针以代替闭包的例子:

```rs
fn main() {
    let nums = vec![-1, 1, -2, 2, -3, 3];
    let absolutes: Vec<i32> = nums.into_iter().map(i32::abs).collect();
    assert_eq!(vec![1, 1, 2, 2, 3, 3], absolutes); // ✅
}
```



### 其他 Trait

| Trait(s) | Category | Operator(s) | Description |
|----------|----------|-------------|-------------|
| `Deref` | other | `*` | immutable dereference |
| `DerefMut` | other | `*` | mutable derenence |
| `Drop` | other | - | type destructor |
| `Index` | other | `[]` | immutable index |
| `IndexMut` | other | `[]` | mutable index |
| `RangeBounds` | other | `..` | range |



#### Deref & DerefMut

预备知识
- Self
- Methods
- Associated Types
- Subtraits & Supertraits
- Sized

```rs
trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}

trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

可以使用解引用操作符 `*` 把实现了 `Deref<Target = T>` trait 的类型解引用为  `T` 类型。这对于智能指针类型（如 `Box` 和 `Rc`）有明显的用途。然而，我们很少看到在 Rust 代码中显式地使用解引用操作符，这是因为 Rust 的一个叫做 “强制解引用”(_deref coercion_)的特性。

当类型作为函数参数传递、从函数返回或作为方法调用的一部分使用时，Rust 会自动地对类型进行解引用。这就是为什么我们可以将 `&String` 和 `&Vec<T>` 传递给期望 `&str` 和 `&[T]` 的函数的原因，因为 `String` 实现了 `Deref<Target = str>`, `Vec<T>` 实现了 `Deref<Target = [T]>`。

`Deref` 和 `DerefMut` 只应为智能指针类型而实现。人们试图误用和滥用这些 trait 的最常见方式是试图将某种 OOP 式的数据继承塞进 Rust 中。这是行不通的。Rust 不是面向对象的。让我们来看看几种不同的情况，在哪些情况下，如何以及为什么它不起作用。让我们从这个例子开始:

```rs
use std::ops::Deref;

struct Human {
    health_points: u32,
}

enum Weapon {
    Spear,
    Axe,
    Sword,
}

// a Soldier is just a Human with a Weapon
struct Soldier {
    human: Human,
    weapon: Weapon,
}

impl Deref for Soldier {
    type Target = Human;
    fn deref(&self) -> &Human {
        &self.human
    }
}

enum Mount {
    Horse,
    Donkey,
    Cow,
}

// a Knight is just a Soldier with a Mount
struct Knight {
    soldier: Soldier,
    mount: Mount,
}

impl Deref for Knight {
    type Target = Soldier;
    fn deref(&self) -> &Soldier {
        &self.soldier
    }
}

enum Spell {
    MagicMissile,
    FireBolt,
    ThornWhip,
}

// a Mage is just a Human who can cast Spells
struct Mage {
    human: Human,
    spells: Vec<Spell>,
}

impl Deref for Mage {
    type Target = Human;
    fn deref(&self) -> &Human {
        &self.human
    }
}

enum Staff {
    Wooden,
    Metallic,
    Plastic,
}

// a Wizard is just a Mage with a Staff
struct Wizard {
    mage: Mage,
    staff: Staff,
}

impl Deref for Wizard {
    type Target = Mage;
    fn deref(&self) -> &Mage {
        &self.mage
    }
}

fn borrows_human(human: &Human) {}
fn borrows_soldier(soldier: &Soldier) {}
fn borrows_knight(knight: &Knight) {}
fn borrows_mage(mage: &Mage) {}
fn borrows_wizard(wizard: &Wizard) {}

fn example(human: Human, soldier: Soldier, knight: Knight, mage: Mage, wizard: Wizard) {
    // all types can be used as Humans
    borrows_human(&human);
    borrows_human(&soldier);
    borrows_human(&knight);
    borrows_human(&mage);
    borrows_human(&wizard);
    // Knights can be used as Soldiers
    borrows_soldier(&soldier);
    borrows_soldier(&knight);
    // Wizards can be used as Mages
    borrows_mage(&mage);
    borrows_mage(&wizard);
    // Knights & Wizards passed as themselves
    borrows_knight(&knight);
    borrows_wizard(&wizard);
}
```

因此，乍一看，上面的内容看起来很不错！然而，在仔细研究后，很快就发现了问题。首先，强制解引用只对引用起作用，所以当我们真正想要传递所有权时，它就不起作用了。

```rs
fn takes_human(human: Human) {}

fn example(human: Human, soldier: Soldier, knight: Knight, mage: Mage, wizard: Wizard) {
    // all types CANNOT be used as Humans
    takes_human(human);
    takes_human(soldier); // ❌
    takes_human(knight); // ❌
    takes_human(mage); // ❌
    takes_human(wizard); // ❌
}
```

此外，强制解引用在泛型上下文中不起作用。比方说，我们只在人类身上实现一些 trait：

```rs
trait Rest {
    fn rest(&self);
}

impl Rest for Human {
    fn rest(&self) {}
}

fn take_rest<T: Rest>(rester: &T) {
    rester.rest()
}

fn example(human: Human, soldier: Soldier, knight: Knight, mage: Mage, wizard: Wizard) {
    // all types CANNOT be used as Rest types, only Human
    take_rest(&human);
    take_rest(&soldier); // ❌
    take_rest(&knight); // ❌
    take_rest(&mage); // ❌
    take_rest(&wizard); // ❌
}
```

另外，尽管强制解引用在很多地方都能工作，但并不是所有地方都能工作。它对操作数不起作用，尽管操作符只是方法调用的语法糖。比方说，我们想让 `Mage`(法师) 用 `+=` 运算符来学习 `Spell`(咒语)，这很可爱。

```rs
impl DerefMut for Wizard {
    fn deref_mut(&mut self) -> &mut Mage {
        &mut self.mage
    }
}

impl AddAssign<Spell> for Mage {
    fn add_assign(&mut self, spell: Spell) {
        self.spells.push(spell);
    }
}

fn example(mut mage: Mage, mut wizard: Wizard, spell: Spell) {
    mage += spell;
    wizard += spell; // ❌ wizard not coerced to mage here
    wizard.add_assign(spell); // oof, we have to call it like this 🤦
}
```

在 OOP 式数据继承的语言中，方法中 `self` 的值总是等于调用该方法的类型，但在 Rust 中，`self` 的值总是等于实现该方法的类型。

```rs
struct Human {
    profession: &'static str,
    health_points: u32,
}

impl Human {
    // self will always be a Human here, even if we call it on a Soldier
    fn state_profession(&self) {
        println!("I'm a {}!", self.profession);
    }
}

struct Soldier {
    profession: &'static str,
    human: Human,
    weapon: Weapon,
}

fn example(soldier: &Soldier) {
    assert_eq!("servant", soldier.human.profession);
    assert_eq!("spearman", soldier.profession);
    soldier.human.state_profession(); // prints "I'm a servant!"
    soldier.state_profession(); // still prints "I'm a servant!" 🤦
}
```

当在一个新类型上实现 `Deref` 或 `DerefMut` 时，上述的问题尤其严重。假设我们想创建一个 `SortedVec` 类型，它只是一个 `Vec`，但它总是按排序顺序排列。下面是我们如何做的。

```rs
struct SortedVec<T: Ord>(Vec<T>);

impl<T: Ord> SortedVec<T> {
    fn new(mut vec: Vec<T>) -> Self {
        vec.sort();
        SortedVec(vec)
    }
    fn push(&mut self, t: T) {
        self.0.push(t);
        self.0.sort();
    }
}
```

很明显，我们不能在这里实现 `DerefMut<Target = Vec<T>>`，否则任何使用 `SortedVec` 的人都可以轻易地破坏排序的顺序。然而，实现 `Deref<Target = Vec<T>>` 肯定是安全的，对吗？试着在下面的程序中发现这个错误。

```rs
use std::ops::Deref;

struct SortedVec<T: Ord>(Vec<T>);

impl<T: Ord> SortedVec<T> {
    fn new(mut vec: Vec<T>) -> Self {
        vec.sort();
        SortedVec(vec)
    }
    fn push(&mut self, t: T) {
        self.0.push(t);
        self.0.sort();
    }
}

impl<T: Ord> Deref for SortedVec<T> {
    type Target = Vec<T>;
    fn deref(&self) -> &Vec<T> {
        &self.0
    }
}

fn main() {
    let sorted = SortedVec::new(vec![2, 8, 6, 3]);
    sorted.push(1);
    let sortedClone = sorted.clone();
    sortedClone.push(4);
}
```

我们从未为 `SortedVec` 实现 `Clone`，所以当我们调用 `.clone()` 方法时，编译器使用强制解引用来解决对 `Vec` 的方法调用，所以它返回一个 `Vec` 而不是 `SortedVec`!

```rs
fn main() {
    let sorted: SortedVec<i32> = SortedVec::new(vec![2, 8, 6, 3]);
    sorted.push(1); // still sorted

    // calling clone on SortedVec actually returns a Vec 🤦
    let sortedClone: Vec<i32> = sorted.clone();
    sortedClone.push(4); // sortedClone no longer sorted 💀
}
```

总之，以上这些限制、约束或麻烦都不是 Rust 的缺点，因为 Rust 一开始就没有被设计成一种 OO 语言或支持任何 OOP 模式。

本节的主要启示是，不要试图用 `Deref` 和 `DerefMut` 实现来表现可爱或聪明。它们实际上只适合于智能指针类型，目前只能在标准库中实现，因为智能指针类型目前需要不稳定的特性和编译器的魔法才能工作。如果我们想要类似于 `Deref` 和 `DerefMut` 的功能和行为，那么我们实际上可能要找的是 `AsRef` 和 `AsMut`，我们将在后面讨论。



#### Index & IndexMut

预备知识
- Self
- Methods
- Associated Types
- Generic Parameters
- Generic Types vs Associated Types
- Subtraits & Supertraits
- Sized

```rs
trait Index<Idx: ?Sized> {
    type Output: ?Sized;
    fn index(&self, index: Idx) -> &Self::Output;
}

trait IndexMut<Idx>: Index<Idx> where Idx: ?Sized {
    fn index_mut(&mut self, index: Idx) -> &mut Self::Output;
}
```

我们可以将 `[]` 索引到有 `T` 值的 `Index<T, Output = U>` 类型，索引操作将返回 `&U` 值。对于语法糖，编译器会在任何从索引操作返回的值前面自动插入一个解引用运算符 `*`。

```rs
fn main() {
    // Vec<i32> impls Index<usize, Output = i32> so
    // indexing Vec<i32> should produce &i32s and yet...
    let vec = vec![1, 2, 3, 4, 5];
    let num_ref: &i32 = vec[0]; // ❌ expected &i32 found i32

    // above line actually desugars to
    let num_ref: &i32 = *vec[0]; // ❌ expected &i32 found i32

    // both of these alternatives work
    let num: i32 = vec[0]; // ✅
    let num_ref = &vec[0]; // ✅
}
```

一开始有点让人困惑，因为看起来 `Index` trait 并不遵循它自己的方法签名，但实际上这只是有问题的语法糖。

因为 `Idx` 是一个泛型类型，`Index` trait 可以为一个给定的类型实现很多次，在 `Vec<T>` 的情况下，我们不仅可以使用 `usize` 对其进行索引，我们还可以使用 `Range<usize>` 对其进行索引，以获得切片。

```rs
fn main() {
    let vec = vec![1, 2, 3, 4, 5];
    assert_eq!(&vec[..], &[1, 2, 3, 4, 5]); // ✅
    assert_eq!(&vec[1..], &[2, 3, 4, 5]); // ✅
    assert_eq!(&vec[..4], &[1, 2, 3, 4]); // ✅
    assert_eq!(&vec[1..4], &[2, 3, 4]); // ✅
}
```

为了展示我们如何实现 `Index`，这里有一个有趣的例子，展示了我们如何使用一个新类型和 `Index` trait 来实现 `Vec` 的包装索引和负索引。

```rs
use std::ops::Index;

struct WrappingIndex<T>(Vec<T>);

impl<T> Index<usize> for WrappingIndex<T> {
    type Output = T;
    fn index(&self, index: usize) -> &T {
        &self.0[index % self.0.len()]
    }
}

impl<T> Index<i128> for WrappingIndex<T> {
    type Output = T;
    fn index(&self, index: i128) -> &T {
        let self_len = self.0.len() as i128;
        let idx = (((index % self_len) + self_len) % self_len) as usize;
        &self.0[idx]
    }
}

#[test] // ✅
fn indexes() {
    let wrapping_vec = WrappingIndex(vec![1, 2, 3]);
    assert_eq!(1, wrapping_vec[0_usize]);
    assert_eq!(2, wrapping_vec[1_usize]);
    assert_eq!(3, wrapping_vec[2_usize]);
}

#[test] // ✅
fn wrapping_indexes() {
    let wrapping_vec = WrappingIndex(vec![1, 2, 3]);
    assert_eq!(1, wrapping_vec[3_usize]);
    assert_eq!(2, wrapping_vec[4_usize]);
    assert_eq!(3, wrapping_vec[5_usize]);
}

#[test] // ✅
fn neg_indexes() {
    let wrapping_vec = WrappingIndex(vec![1, 2, 3]);
    assert_eq!(1, wrapping_vec[-3_i128]);
    assert_eq!(2, wrapping_vec[-2_i128]);
    assert_eq!(3, wrapping_vec[-1_i128]);
}

#[test] // ✅
fn wrapping_neg_indexes() {
    let wrapping_vec = WrappingIndex(vec![1, 2, 3]);
    assert_eq!(1, wrapping_vec[-6_i128]);
    assert_eq!(2, wrapping_vec[-5_i128]);
    assert_eq!(3, wrapping_vec[-4_i128]);
}
```

没有要求 `Idx` 类型必须是数字类型或 `Range`，它可以是一个枚举! 下面是一个例子，使用篮球位置索引到一个篮球队，以检索该队的球员。

```rs
use std::ops::Index;

enum BasketballPosition {
    PointGuard,
    ShootingGuard,
    Center,
    PowerForward,
    SmallForward,
}

struct BasketballPlayer {
    name: &'static str,
    position: BasketballPosition,
}

struct BasketballTeam {
    point_guard: BasketballPlayer,
    shooting_guard: BasketballPlayer,
    center: BasketballPlayer,
    power_forward: BasketballPlayer,
    small_forward: BasketballPlayer,
}

impl Index<BasketballPosition> for BasketballTeam {
    type Output = BasketballPlayer;
    fn index(&self, position: BasketballPosition) -> &BasketballPlayer {
        match position {
            BasketballPosition::PointGuard => &self.point_guard,
            BasketballPosition::ShootingGuard => &self.shooting_guard,
            BasketballPosition::Center => &self.center,
            BasketballPosition::PowerForward => &self.power_forward,
            BasketballPosition::SmallForward => &self.small_forward,
        }
    }
}
```



#### Drop

预备知识
- Self
- Methods

```rs
trait Drop {
    fn drop(&mut self);
}
```

如果一个类型实现了 `Drop`，那么当它超出作用域但在被销毁之前，`drop` 将在该类型上调用。我们很少需要为我们的类型实现这个功能，但是一个很好的例子是，如果一个类型持有一些外部资源，当类型被销毁时，这些资源需要被清理掉。

在标准库中有一个 `BufWriter` 类型，允许我们对 `Write` 类型进行缓冲写入。然而，如果 `BufWriter` 在其缓冲区的内容被刷新到底层的 `Write` 类型之前就被销毁了呢？幸好这是不可能的! `BufWriter` 实现了 `Drop` trait，所以每当它离开作用域时，`flush` 总是被调用。

```rs
impl<W: Write> Drop for BufWriter<W> {
    fn drop(&mut self) {
        self.flush_buf();
    }
}
```

另外，Rust 中的 `Mutex` 没有 `unlock()` 方法，因为它们不需要这些方法。在一个 `Mutex` 上调用 `lock()` 会返回一个 `MutexGuard`，当 `Mutex` 超出作用域时，由于它的 `Drop` 实现，它会自动解锁。

```rs
impl<T: ?Sized> Drop for MutexGuard<'_, T> {
    fn drop(&mut self) {
        unsafe {
            self.lock.inner.raw_unlock();
        }
    }
}
```

一般来说，如果你在某些资源上实现一个抽象，在使用后需要清理，那么这就是使用 `Drop` trait 的一个很好的理由。



## 转换 Traits



### From & Into

预备知识
- Self
- Functions
- Methods
- Generic Parameters
- Generic Blanket Impls

```rs
trait From<T> {
    fn from(T) -> Self;
}
```

`From<T>` 类型允许我们将 `T` 转换为 `Self`。

```rs
trait Into<T> {
    fn into(self) -> T;
}
```

`Into<T>` 类型允许我们将 `Self` 转换为 `T`。

这些 trait 是一个硬币的两个不同侧面。我们只能为我们的类型实现 `From<T>`，因为 `Into<T>` 的实现是由下面这个泛型全面实现自动提供的。

```rs
impl<T, U> Into<U> for T
where
    U: From<T>,
{
    fn into(self) -> U {
        U::from(self)
    }
}
```

这两个  trait 存在的原因是，它允许我们以稍微不同的方式在泛型类型上编写 trait 约束(trait bound)。

```rs
fn function<T>(t: T)
where
    // these bounds are equivalent
    T: From<i32>,
    i32: Into<T>
{
    // these examples are equivalent
    let example: T = T::from(0);
    let example: T = 0.into();
}
```

关于何时使用这两种方法并没有硬性规定，所以在每种情况下都要选择最合理的方法。现在让我们看看一些关于 `Point` 的例子。

```rs
struct Point {
    x: i32,
    y: i32,
}

impl From<(i32, i32)> for Point {
    fn from((x, y): (i32, i32)) -> Self {
        Point { x, y }
    }
}

impl From<[i32; 2]> for Point {
    fn from([x, y]: [i32; 2]) -> Self {
        Point { x, y }
    }
}

fn example() {
    // using From
    let origin = Point::from((0, 0));
    let origin = Point::from([0, 0]);

    // using Into
    let origin: Point = (0, 0).into();
    let origin: Point = [0, 0].into();
}
```

实现不是对称的，所以如果我们想把 `Point` 转换成元组和数组，我们也必须显示地地添加这些。

```rs
struct Point {
    x: i32,
    y: i32,
}

impl From<(i32, i32)> for Point {
    fn from((x, y): (i32, i32)) -> Self {
        Point { x, y }
    }
}

impl From<Point> for (i32, i32) {
    fn from(Point { x, y }: Point) -> Self {
        (x, y)
    }
}

impl From<[i32; 2]> for Point {
    fn from([x, y]: [i32; 2]) -> Self {
        Point { x, y }
    }
}

impl From<Point> for [i32; 2] {
    fn from(Point { x, y }: Point) -> Self {
        [x, y]
    }
}

fn example() {
    // from (i32, i32) into Point
    let point = Point::from((0, 0));
    let point: Point = (0, 0).into();

    // from Point into (i32, i32)
    let tuple = <(i32, i32)>::from(point);
    let tuple: (i32, i32) = point.into();

    // from [i32; 2] into Point
    let point = Point::from([0, 0]);
    let point: Point = [0, 0].into();

    // from Point into [i32; 2]
    let array = <[i32; 2]>::from(point);
    let array: [i32; 2] = point.into();
}
```

`From<T>` 的一个普遍用途是缩减模板代码。假设我们在程序中加入一个包含三个 `Point` 的 `Triangle` 类型，这里有许多方法可以构建它。

```rs
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn new(x: i32, y: i32) -> Point {
        Point { x, y }
    }
}

impl From<(i32, i32)> for Point {
    fn from((x, y): (i32, i32)) -> Point {
        Point { x, y }
    }
}

struct Triangle {
    p1: Point,
    p2: Point,
    p3: Point,
}

impl Triangle {
    fn new(p1: Point, p2: Point, p3: Point) -> Triangle {
        Triangle { p1, p2, p3 }
    }
}

impl<P> From<[P; 3]> for Triangle
where
    P: Into<Point>
{
    fn from([p1, p2, p3]: [P; 3]) -> Triangle {
        Triangle {
            p1: p1.into(),
            p2: p2.into(),
            p3: p3.into(),
        }
    }
}

fn example() {
    // manual construction
    let triangle = Triangle {
        p1: Point {
            x: 0,
            y: 0,
        },
        p2: Point {
            x: 1,
            y: 1,
        },
        p3: Point {
            x: 2,
            y: 2,
        },
    };

    // using Point::new
    let triangle = Triangle {
        p1: Point::new(0, 0),
        p2: Point::new(1, 1),
        p3: Point::new(2, 2),
    };

    // using From<(i32, i32)> for Point
    let triangle = Triangle {
        p1: (0, 0).into(),
        p2: (1, 1).into(),
        p3: (2, 2).into(),
    };

    // using Triangle::new + From<(i32, i32)> for Point
    let triangle = Triangle::new(
        (0, 0).into(),
        (1, 1).into(),
        (2, 2).into(),
    );

    // using From<[Into<Point>; 3]> for Triangle
    let triangle: Triangle = [
        (0, 0),
        (1, 1),
        (2, 2),
    ].into();
}
```

对于何时、如何或为什么我们应该为我们的类型实现 `From<T>`，没有任何规则，所以这取决于我们对每种情况的最佳判断。

`Into<T>` 的一个流行用法是使需要自有值(owned values)的函数在接受自有值或借用值时具有通用性。

```rs
struct Person {
    name: String,
}

impl Person {
    // accepts:
    // - String
    fn new1(name: String) -> Person {
        Person { name }
    }

    // accepts:
    // - String
    // - &String
    // - &str
    // - Box<str>
    // - Cow<'_, str>
    // - char
    // since all of the above types can be converted into String
    fn new2<N: Into<String>>(name: N) -> Person {
        Person { name: name.into() }
    }
}
```



## 错误处理

谈论错误处理和 `Error` trait 的最佳时机是在讨论完 `Display`、`Debug`、`Any` 和 `From` 之后，在讨论 `TryFrom` 之前，因此错误处理部分与转换 trait 部分尴尬地一分为二。



### Error

预备知识
- Self
- Methods
- Default Impls
- Generic Blanket Impls
- Subtraits & Supertraits
- Trait Objects
- Display & ToString
- Debug
- Any
- From & Into

```rs
trait Error: Debug + Display {
    // provided default impls
    fn source(&self) -> Option<&(dyn Error + 'static)>;
    fn backtrace(&self) -> Option<&Backtrace>;
    fn description(&self) -> &str;
    fn cause(&self) -> Option<&dyn Error>;
}
```

在 Rust 中，错误被返回，而不是被抛出。我们来看看一些例子。

由于整数类型除以0会引起 panic，如果我们想让我们的程序更安全、更明确，我们可以实现一个 `safe_div` 函数，返回一个 `Result`，就像这样。

```rs
use std::fmt;
use std::error;

#[derive(Debug, PartialEq)]
struct DivByZero;

impl fmt::Display for DivByZero {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "division by zero error")
    }
}

impl error::Error for DivByZero {}

fn safe_div(numerator: i32, denominator: i32) -> Result<i32, DivByZero> {
    if denominator == 0 {
        return Err(DivByZero);
    }
    Ok(numerator / denominator)
}

#[test] // ✅
fn test_safe_div() {
    assert_eq!(safe_div(8, 2), Ok(4));
    assert_eq!(safe_div(5, 0), Err(DivByZero));
}
```

由于错误是返回的，而不是抛出的，所以必须显式处理，如果当前函数不能处理一个错误，它应该将其传播给调用者。传播错误的最习惯的方法是使用 `?` 操作符，它只是现在被废弃的 `try!` 宏的语法糖，它只是做这个。

```rs
macro_rules! try {
    ($expr:expr) => {
        match $expr {
            // if Ok just unwrap the value
            Ok(val) => val,
            // if Err map the err value using From and return
            Err(err) => {
                return Err(From::from(err));
            }
        }
    };
}
```

如果我们想写一个将文件读成 `String` 的函数，我们可以这样写，用 `?` 将 `io::Error` 传播到它们可能出现的任何地方。

```rs
use std::io::Read;
use std::path::Path;
use std::io;
use std::fs::File;

fn read_file_to_string(path: &Path) -> Result<String, io::Error> {
    let mut file = File::open(path)?; // ⬆️ io::Error
    let mut contents = String::new();
    file.read_to_string(&mut contents)?; // ⬆️ io::Error
    Ok(contents)
}
```

但是，假设我们正在读取的文件实际上是一个数字列表，我们想把它们加在一起，我们会像这样更新我们的函数。

```rs
use std::io::Read;
use std::path::Path;
use std::io;
use std::fs::File;

fn sum_file(path: &Path) -> Result<i32, /* What to put here? */> {
    let mut file = File::open(path)?; // ⬆️ io::Error
    let mut contents = String::new();
    file.read_to_string(&mut contents)?; // ⬆️ io::Error
    let mut sum = 0;
    for line in contents.lines() {
        sum += line.parse::<i32>()?; // ⬆️ ParseIntError
    }
    Ok(sum)
}
```

但是现在我们的 `Result` 的错误类型是什么？它可以返回一个 `io::Error` 或者一个 `ParseIntError`。我们将看一下解决这个问题的三种方法，从临时应急的方法开始，最后是最稳健的方法。

第一种方法是认识到所有实现了 `Error` 的类型也实现了 `Display`，所以我们可以将所有的错误映射到 `String`，并使用 `String` 作为我们的错误类型。

```rs
use std::fs::File;
use std::io;
use std::io::Read;
use std::path::Path;

fn sum_file(path: &Path) -> Result<i32, String> {
    let mut file = File::open(path)
        .map_err(|e| e.to_string())?; // ⬆️ io::Error -> String
    let mut contents = String::new();
    file.read_to_string(&mut contents)
        .map_err(|e| e.to_string())?; // ⬆️ io::Error -> String
    let mut sum = 0;
    for line in contents.lines() {
        sum += line.parse::<i32>()
            .map_err(|e| e.to_string())?; // ⬆️ ParseIntError -> String
    }
    Ok(sum)
}
```

对每个错误进行字符串化处理的明显缺点是，我们丢弃了类型信息，这使得调用者更难处理错误。

上述方法的一个非显而易见的好处是我们可以定制字符串以提供更多的特定环境信息。例如，`ParseIntError` 通常字符串化为 "invalid digit found in string"，这是非常模糊的，没有提到无效的字符串是什么或者它试图解析成什么整数类型。如果我们要调试这个问题，这个错误信息几乎是无用的。然而，我们可以通过自己提供所有与上下文相关的信息来使其明显改善。

```rs
sum += line.parse::<i32>()
    .map_err(|_| format!("failed to parse {} into i32", line))?;
```

第二种方法是利用标准库中的这种泛型全面实现。

```rs
impl<E: error::Error> From<E> for Box<dyn error::Error>;
```

这意味着任何 `Error` 类型都可以通过 `?` 运算符隐式地转换为 `Box<dyn error::Error>`，所以我们可以在任何产生错误的函数的 `Result` 返回类型中把错误类型设置为 `Box<dyn error::Error>`，`?` 运算符将为我们完成其余的工作。

```rs
use std::fs::File;
use std::io::Read;
use std::path::Path;
use std::error;

fn sum_file(path: &Path) -> Result<i32, Box<dyn error::Error>> {
    let mut file = File::open(path)?; // ⬆️ io::Error -> Box<dyn error::Error>
    let mut contents = String::new();
    file.read_to_string(&mut contents)?; // ⬆️ io::Error -> Box<dyn error::Error>
    let mut sum = 0;
    for line in contents.lines() {
        sum += line.parse::<i32>()?; // ⬆️ ParseIntError -> Box<dyn error::Error>
    }
    Ok(sum)
}
```

虽然更加简洁，但这似乎与之前的方法有相同的缺点，即丢弃了类型信息。这大部分是真的，但是如果调用者知道我们函数的实现细节，他们仍然可以使用 `error::Error` 上的 `downcast_ref()` 方法来处理不同的错误类型，这和它在 `dyn Any` 类型上的作用是一样的。

```rs
fn handle_sum_file_errors(path: &Path) {
    match sum_file(path) {
        Ok(sum) => println!("the sum is {}", sum),
        Err(err) => {
            if let Some(e) = err.downcast_ref::<io::Error>() {
                // handle io::Error
            } else if let Some(e) = err.downcast_ref::<ParseIntError>() {
                // handle ParseIntError
            } else {
                // we know sum_file can only return one of the
                // above errors so this branch is unreachable
                unreachable!();
            }
        }
    }
}
```

第三种方法是最稳健和类型安全的方法，可以聚合这些不同的错误，是使用一个枚举建立我们自己的自定义错误类型。

```rs
use std::num::ParseIntError;
use std::fs::File;
use std::io;
use std::io::Read;
use std::path::Path;
use std::error;
use std::fmt;

#[derive(Debug)]
enum SumFileError {
    Io(io::Error),
    Parse(ParseIntError),
}

impl From<io::Error> for SumFileError {
    fn from(err: io::Error) -> Self {
        SumFileError::Io(err)
    }
}

impl From<ParseIntError> for SumFileError {
    fn from(err: ParseIntError) -> Self {
        SumFileError::Parse(err)
    }
}

impl fmt::Display for SumFileError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            SumFileError::Io(err) => write!(f, "sum file error: {}", err),
            SumFileError::Parse(err) => write!(f, "sum file error: {}", err),
        }
    }
}

impl error::Error for SumFileError {
    // the default impl for this method always returns None
    // but we can now override it to make it way more useful!
    fn source(&self) -> Option<&(dyn error::Error + 'static)> {
        Some(match self {
            SumFileError::Io(err) => err,
            SumFileError::Parse(err) => err,
        })
    }
}

fn sum_file(path: &Path) -> Result<i32, SumFileError> {
    let mut file = File::open(path)?; // ⬆️ io::Error -> SumFileError
    let mut contents = String::new();
    file.read_to_string(&mut contents)?; // ⬆️ io::Error -> SumFileError
    let mut sum = 0;
    for line in contents.lines() {
        sum += line.parse::<i32>()?; // ⬆️ ParseIntError -> SumFileError
    }
    Ok(sum)
}

fn handle_sum_file_errors(path: &Path) {
    match sum_file(path) {
        Ok(sum) => println!("the sum is {}", sum),
        Err(SumFileError::Io(err)) => {
            // handle io::Error
        },
        Err(SumFileError::Parse(err)) => {
            // handle ParseIntError
        },
    }
}
```



## Conversion Traits Continued



### TryFrom & TryInto

预备知识
- Self
- Functions
- Methods
- Associated Types
- Generic Parameters
- Generic Types vs Associated Types
- Generic Blanket Impls
- From & Into
- Error

`TryFrom` and `TryInto` are the  versions of `From` and `Into`.
`TryFrom` 和 `TryInto` 是 `From` 和 `Into` 的不可靠版本。

```rs
trait TryFrom<T> {
    type Error;
    fn try_from(value: T) -> Result<Self, Self::Error>;
}

trait TryInto<T> {
    type Error;
    fn try_into(self) -> Result<T, Self::Error>;
}
```

Similarly to `Into` we cannot impl `TryInto` because its impl is provided by this generic blanket impl:
与 `Into` 类似，我们不能实现 `TryInto`，因为它的实现是由下面这个泛型全面实现提供的。

```rs
impl<T, U> TryInto<U> for T
where
    U: TryFrom<T>,
{
    type Error = U::Error;

    fn try_into(self) -> Result<U, U::Error> {
        U::try_from(self)
    }
}
```

Let's say that in the context of our program it doesn't make sense for `Point`s to have `x` and `y` values that are less than `-1000` or greater than `1000`. This is how we'd rewrite our earlier `From` impls using `TryFrom` to signal to the users of our type that this conversion can now fail:
假设在我们的程序中，`Point` 的 `x` 和 `y` 的值小于 `-1000` 或大于 `1000` 是不合理的。这就是我们如何使用 `TryFrom` 重写我们先前的 `From` 实现，向我们类型的用户发出信号，这个转换现在可以失败。

```rs
use std::convert::TryFrom;
use std::error;
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

#[derive(Debug)]
struct OutOfBounds;

impl fmt::Display for OutOfBounds {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "out of bounds")
    }
}

impl error::Error for OutOfBounds {}

// now fallible
impl TryFrom<(i32, i32)> for Point {
    type Error = OutOfBounds;
    fn try_from((x, y): (i32, i32)) -> Result<Point, OutOfBounds> {
        if x.abs() > 1000 || y.abs() > 1000 {
            return Err(OutOfBounds);
        }
        Ok(Point { x, y })
    }
}

// still infallible
impl From<Point> for (i32, i32) {
    fn from(Point { x, y }: Point) -> Self {
        (x, y)
    }
}
```

And here's the refactored `TryFrom<[TryInto<Point>; 3]>` impl for `Triangle`:
这里是重构后的 `TryFrom<[TryInto<Point>; 3]>` 实现，用于 `Triangle`。

```rs
use std::convert::{TryFrom, TryInto};
use std::error;
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

#[derive(Debug)]
struct OutOfBounds;

impl fmt::Display for OutOfBounds {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "out of bounds")
    }
}

impl error::Error for OutOfBounds {}

impl TryFrom<(i32, i32)> for Point {
    type Error = OutOfBounds;
    fn try_from((x, y): (i32, i32)) -> Result<Self, Self::Error> {
        if x.abs() > 1000 || y.abs() > 1000 {
            return Err(OutOfBounds);
        }
        Ok(Point { x, y })
    }
}

struct Triangle {
    p1: Point,
    p2: Point,
    p3: Point,
}

impl<P> TryFrom<[P; 3]> for Triangle
where
    P: TryInto<Point>,
{
    type Error = P::Error;
    fn try_from([p1, p2, p3]: [P; 3]) -> Result<Self, Self::Error> {
        Ok(Triangle {
            p1: p1.try_into()?,
            p2: p2.try_into()?,
            p3: p3.try_into()?,
        })
    }
}

fn example() -> Result<Triangle, OutOfBounds> {
    let t: Triangle = [(0, 0), (1, 1), (2, 2)].try_into()?;
    Ok(t)
}
```



### FromStr

预备知识
- Self
- Functions
- Associated Types
- Error
- TryFrom & TryInto

```rs
trait FromStr {
    type Err;
    fn from_str(s: &str) -> Result<Self, Self::Err>;
}
```

`FromStr` types allow performing a fallible conversion from `&str` into `Self`. The idiomatic way to use `FromStr` is to call the `.parse()` method on `&str`s:
`FromStr` 类型允许执行从 `&str` 到 `Self` 的错误转换。使用 `FromStr` 的习惯方法是对 `&str` 调用 `.parse()` 方法。

```rs
use std::str::FromStr;

fn example<T: FromStr>(s: &'static str) {
    // these are all equivalent
    let t: Result<T, _> = FromStr::from_str(s);
    let t = T::from_str(s);
    let t: Result<T, _> = s.parse();
    let t = s.parse::<T>(); // most idiomatic
}
```

`Point` 实现的例子:

```rs
use std::error;
use std::fmt;
use std::iter::Enumerate;
use std::num::ParseIntError;
use std::str::{Chars, FromStr};

#[derive(Debug, Eq, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn new(x: i32, y: i32) -> Self {
        Point { x, y }
    }
}

#[derive(Debug, PartialEq)]
struct ParsePointError;

impl fmt::Display for ParsePointError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "failed to parse point")
    }
}

impl From<ParseIntError> for ParsePointError {
    fn from(_e: ParseIntError) -> Self {
        ParsePointError
    }
}

impl error::Error for ParsePointError {}

impl FromStr for Point {
    type Err = ParsePointError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let is_num = |(_, c): &(usize, char)| matches!(c, '0'..='9' | '-');
        let isnt_num = |t: &(_, _)| !is_num(t);

        let get_num =
            |char_idxs: &mut Enumerate<Chars<'_>>| -> Result<(usize, usize), ParsePointError> {
                let (start, _) = char_idxs
                    .skip_while(isnt_num)
                    .next()
                    .ok_or(ParsePointError)?;
                let (end, _) = char_idxs
                    .skip_while(is_num)
                    .next()
                    .ok_or(ParsePointError)?;
                Ok((start, end))
            };

        let mut char_idxs = s.chars().enumerate();
        let (x_start, x_end) = get_num(&mut char_idxs)?;
        let (y_start, y_end) = get_num(&mut char_idxs)?;

        let x = s[x_start..x_end].parse::<i32>()?;
        let y = s[y_start..y_end].parse::<i32>()?;

        Ok(Point { x, y })
    }
}

#[test] // ✅
fn pos_x_y() {
    let p = "(4, 5)".parse::<Point>();
    assert_eq!(p, Ok(Point::new(4, 5)));
}

#[test] // ✅
fn neg_x_y() {
    let p = "(-6, -2)".parse::<Point>();
    assert_eq!(p, Ok(Point::new(-6, -2)));
}

#[test] // ✅
fn not_a_point() {
    let p = "not a point".parse::<Point>();
    assert_eq!(p, Err(ParsePointError));
}
```

`FromStr` has the same signature as `TryFrom<&str>`. It doesn't matter which one we impl for a type first as long as we forward the impl to the other one. Here's a `TryFrom<&str>` impl for `Point` assuming it already has a `FromStr` impl:
`FromStr` 与 `TryFrom<&str>` 的签名相同。只要我们把实现转发给另一个类型，哪一个实现并不重要。下面是一个针对 `Point` 的 `TryFrom<&str>` 实现，假设它已经有一个 `FromStr` 实现。

```rs
impl TryFrom<&str> for Point {
    type Error = <Point as FromStr>::Err;
    fn try_from(s: &str) -> Result<Point, Self::Error> {
        <Point as FromStr>::from_str(s)
    }
}
```


### AsRef & AsMut

预备知识
- Self
- Methods
- Sized
- Generic Parameters
- Sized
- Deref & DerefMut

```rs
trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}

trait AsMut<T: ?Sized> {
    fn as_mut(&mut self) -> &mut T;
}
```

`?Sized` 说明 T 类型是大小不确定的。`As` 作为介词, 表明发生了类型转换。

`AsRef` is for cheap reference to reference conversions. However, one of the most common ways it's used is to make functions generic over whether they take ownership or not:

`AsRef` 是用于廉价的引用到引用的转换。然而，它最常见的使用方式之一是使函数在是否拥有所有权上通用。

```rs
// accepts:
//  - &str
//  - &String
fn takes_str(s: &str) {
    // use &str
}

// accepts:
//  - &str
//  - &String
//  - String
fn takes_asref_str<S: AsRef<str>>(s: S) {
    let s: &str = s.as_ref();
    // use &str
}

fn example(slice: &str, borrow: &String, owned: String) {
    takes_str(slice);
    takes_str(borrow);
    takes_str(owned); // ❌
    takes_asref_str(slice);
    takes_asref_str(borrow);
    takes_asref_str(owned); // ✅
}
```

The other most common use-case is returning a reference to inner private data wrapped by a type which protects some invariant. A good example from the standard library is `String` which is just a wrapper around `Vec<u8>`:

另一个最常见的用例是返回一个对内部私有数据的引用，该数据由一个保护某些不变性的类型包裹。标准库中的一个很好的例子是 `String`，它只是 `Vec<u8>` 的一个包装器。

```rs
struct String {
    vec: Vec<u8>,
}
```

This inner `Vec` cannot be made public because if it was people could mutate any byte and break the `String`'s valid UTF-8 encoding. However, it's safe to expose an immutable read-only reference to the inner byte array, hence this impl:
这个内部的 `Vec` 不能被公开，因为如果它被公开，人们可以改变任何字节并破坏 `String` 的有效 UTF-8 编码。然而，公开内部字节数组的不可变的只读引用是安全的，因此有了这个实现:

```rs
impl AsRef<[u8]> for String;
```

Generally, it often only makes sense to impl `AsRef` for a type if it wraps some other type to either provide additional functionality around the inner type or protect some invariant on the inner type.

Let's examine a example of bad `AsRef` impls:
一般来说，只有当一个类型包装了其他类型，为内部类型提供了额外的功能，或者保护了内部类型的某些不变性时，为其实现 `AsRef` 才有意义。

让我们来看看一个不好的 `AsRef` 实现的例子。

```rs
struct User {
    name: String,
    age: u32,
}

impl AsRef<String> for User {
    fn as_ref(&self) -> &String {
        &self.name
    }
}

impl AsRef<u32> for User {
    fn as_ref(&self) -> &u32 {
        &self.age
    }
}
```

This works and kinda makes sense at first, but quickly falls apart if we add more members to `User`:
这在一开始是可行的，而且有点道理，但如果我们给 `User` 增加更多的成员，很快就会崩溃。

```rs
struct User {
    name: String,
    email: String,
    age: u32,
    height: u32,
}

impl AsRef<String> for User {
    fn as_ref(&self) -> &String {
        // uh, do we return name or email here?
    }
}

impl AsRef<u32> for User {
    fn as_ref(&self) -> &u32 {
        // uh, do we return age or height here?
    }
}
```

A `User` is composed of `String`s and `u32`s but it's not really the same thing as a `String` or a `u32`. Even if we had much more specific types:
`User` 是由 `String` 和 `u32` 组成的，但它和 `String` 或 `u32` 并不是真正的一回事。即使我们有更具体的类型。

```rs
struct User {
    name: Name,
    email: Email,
    age: Age,
    height: Height,
}
```

It wouldn't make much sense to impl `AsRef` for any of those because `AsRef` is for cheap reference to reference conversions between semantically equivalent things, and `Name`, `Email`, `Age`, and `Height` by themselves are not the same thing as a `User`.

A good example where we would impl `AsRef` would be if we introduced a new type `Moderator` that just wrapped a `User` and added some moderation specific privileges:
实现 `AsRef` 对这些都没有意义，因为 `AsRef` 是用来在语义上等同的事物之间进行廉价的引用转换，而 `Name`、`Email`、`Age` 和 `Height` 本身就和 `User` 不是一回事。

一个很好的例子是，如果我们引入一个新的类型 `Moderator`，它只是包裹了一个 `User`，并增加了一些特定的管理权限，我们就会使用 `AsRef`。

```rs
struct User {
    name: String,
    age: u32,
}

// unfortunately the standard library cannot provide
// a generic blanket impl to save us from this boilerplate
impl AsRef<User> for User {
    fn as_ref(&self) -> &User {
        self
    }
}

enum Privilege {
    BanUsers,
    EditPosts,
    DeletePosts,
}

// although Moderators have some special
// privileges they are still regular Users
// and should be able to do all the same stuff
struct Moderator {
    user: User,
    privileges: Vec<Privilege>
}

impl AsRef<Moderator> for Moderator {
    fn as_ref(&self) -> &Moderator {
        self
    }
}

impl AsRef<User> for Moderator {
    fn as_ref(&self) -> &User {
        &self.user
    }
}

// this should be callable with Users
// and Moderators (who are also Users)
fn create_post<U: AsRef<User>>(u: U) {
    let user = u.as_ref();
    // etc
}

fn example(user: User, moderator: Moderator) {
    create_post(&user);
    create_post(&moderator); // ✅
}
```

This works because `Moderator`s are just `User`s. Here's the example from the `Deref` section except using `AsRef` instead:
这样做是因为 `Moderator` 就是 `User`。下面是 `Deref` 部分的例子，只是用 `AsRef` 代替。


```rs
use std::convert::AsRef;

struct Human {
    health_points: u32,
}

impl AsRef<Human> for Human {
    fn as_ref(&self) -> &Human {
        self
    }
}

enum Weapon {
    Spear,
    Axe,
    Sword,
}

// a Soldier is just a Human with a Weapon
struct Soldier {
    human: Human,
    weapon: Weapon,
}

impl AsRef<Soldier> for Soldier {
    fn as_ref(&self) -> &Soldier {
        self
    }
}

impl AsRef<Human> for Soldier {
    fn as_ref(&self) -> &Human {
        &self.human
    }
}

enum Mount {
    Horse,
    Donkey,
    Cow,
}

// a Knight is just a Soldier with a Mount
struct Knight {
    soldier: Soldier,
    mount: Mount,
}

impl AsRef<Knight> for Knight {
    fn as_ref(&self) -> &Knight {
        self
    }
}

impl AsRef<Soldier> for Knight {
    fn as_ref(&self) -> &Soldier {
        &self.soldier
    }
}

impl AsRef<Human> for Knight {
    fn as_ref(&self) -> &Human {
        &self.soldier.human
    }
}

enum Spell {
    MagicMissile,
    FireBolt,
    ThornWhip,
}

// a Mage is just a Human who can cast Spells
struct Mage {
    human: Human,
    spells: Vec<Spell>,
}

impl AsRef<Mage> for Mage {
    fn as_ref(&self) -> &Mage {
        self
    }
}

impl AsRef<Human> for Mage {
    fn as_ref(&self) -> &Human {
        &self.human
    }
}

enum Staff {
    Wooden,
    Metallic,
    Plastic,
}

// a Wizard is just a Mage with a Staff
struct Wizard {
    mage: Mage,
    staff: Staff,
}

impl AsRef<Wizard> for Wizard {
    fn as_ref(&self) -> &Wizard {
        self
    }
}

impl AsRef<Mage> for Wizard {
    fn as_ref(&self) -> &Mage {
        &self.mage
    }
}

impl AsRef<Human> for Wizard {
    fn as_ref(&self) -> &Human {
        &self.mage.human
    }
}

fn borrows_human<H: AsRef<Human>>(human: H) {}
fn borrows_soldier<S: AsRef<Soldier>>(soldier: S) {}
fn borrows_knight<K: AsRef<Knight>>(knight: K) {}
fn borrows_mage<M: AsRef<Mage>>(mage: M) {}
fn borrows_wizard<W: AsRef<Wizard>>(wizard: W) {}

fn example(human: Human, soldier: Soldier, knight: Knight, mage: Mage, wizard: Wizard) {
    // all types can be used as Humans
    borrows_human(&human);
    borrows_human(&soldier);
    borrows_human(&knight);
    borrows_human(&mage);
    borrows_human(&wizard);
    // Knights can be used as Soldiers
    borrows_soldier(&soldier);
    borrows_soldier(&knight);
    // Wizards can be used as Mages
    borrows_mage(&mage);
    borrows_mage(&wizard);
    // Knights & Wizards passed as themselves
    borrows_knight(&knight);
    borrows_wizard(&wizard);
}
```

`Deref` didn't work in the prior version of the example above because deref coercion is an implicit conversion between types which leaves room for people to mistakenly formulate the wrong ideas and expectations for how it will behave. `AsRef` works above because it makes the conversion between types explicit and there's no room leftover to develop any wrong ideas or expectations.
`Deref` 在上述例子的先前版本中不起作用，因为 deref coercion 是一种隐式类型转换，为人们错误地制定错误的想法和期望留下了空间。`AsRef` 在上面起作用，因为它使类型间的转换变得明确，没有余地来发展任何错误的想法或期望。


### Borrow & BorrowMut

预备知识
- Self
- Methods
- Generic Parameters
- Subtraits & Supertraits
- Sized
- AsRef & AsMut
- PartialEq & Eq
- Hash
- PartialOrd & Ord

```rs
trait Borrow<Borrowed>
where
    Borrowed: ?Sized,
{
    fn borrow(&self) -> &Borrowed;
}

trait BorrowMut<Borrowed>: Borrow<Borrowed>
where
    Borrowed: ?Sized,
{
    fn borrow_mut(&mut self) -> &mut Borrowed;
}
```

These traits were invented to solve the very specific problem of looking up `String` keys in `HashSet`s, `HashMap`s, `BTreeSet`s, and `BTreeMap`s using `&str` values.

We can view `Borrow<T>` and `BorrowMut<T>` as stricter versions of `AsRef<T>` and `AsMut<T>`, where the returned reference `&T` has equivalent `Eq`, `Hash`, and `Ord` impls to `Self`. This is more easily explained with a commented example:
这些 trait 的发明是为了解决在 `HashSet`, `HashMap`, `BTreeSet`, 和 `BTreeMap` 中使用 `&str` 值查找 `String` 键的特殊问题。

我们可以把 `Borrow<T>` 和 `BorrowMut<T>` 看作是 `AsRef<T>` 和 `AsMut<T>` 的更严格的版本，其中返回的引用 `&T` 与 `Self` 的 `Eq`、`Hash` 和 `Ord` 等值。用一个注释的例子可以更容易地解释这个问题。

```rs
use std::borrow::Borrow;
use std::hash::Hasher;
use std::collections::hash_map::DefaultHasher;
use std::hash::Hash;

fn get_hash<T: Hash>(t: T) -> u64 {
    let mut hasher = DefaultHasher::new();
    t.hash(&mut hasher);
    hasher.finish()
}

fn asref_example<Owned, Ref>(owned1: Owned, owned2: Owned)
where
    Owned: Eq + Ord + Hash + AsRef<Ref>,
    Ref: Eq + Ord + Hash
{
    let ref1: &Ref = owned1.as_ref();
    let ref2: &Ref = owned2.as_ref();

    // refs aren't required to be equal if owned types are equal
    assert_eq!(owned1 == owned2, ref1 == ref2); // ❌

    let owned1_hash = get_hash(&owned1);
    let owned2_hash = get_hash(&owned2);
    let ref1_hash = get_hash(&ref1);
    let ref2_hash = get_hash(&ref2);

    // ref hashes aren't required to be equal if owned type hashes are equal
    assert_eq!(owned1_hash == owned2_hash, ref1_hash == ref2_hash); // ❌

    // ref comparisons aren't required to match owned type comparisons
    assert_eq!(owned1.cmp(&owned2), ref1.cmp(&ref2)); // ❌
}

fn borrow_example<Owned, Borrowed>(owned1: Owned, owned2: Owned)
where
    Owned: Eq + Ord + Hash + Borrow<Borrowed>,
    Borrowed: Eq + Ord + Hash
{
    let borrow1: &Borrowed = owned1.borrow();
    let borrow2: &Borrowed = owned2.borrow();

    // borrows are required to be equal if owned types are equal
    assert_eq!(owned1 == owned2, borrow1 == borrow2); // ✅

    let owned1_hash = get_hash(&owned1);
    let owned2_hash = get_hash(&owned2);
    let borrow1_hash = get_hash(&borrow1);
    let borrow2_hash = get_hash(&borrow2);

    // borrow hashes are required to be equal if owned type hashes are equal
    assert_eq!(owned1_hash == owned2_hash, borrow1_hash == borrow2_hash); // ✅

    // borrow comparisons are required to match owned type comparisons
    assert_eq!(owned1.cmp(&owned2), borrow1.cmp(&borrow2)); // ✅
}
```

It's good to be aware of these traits and understand why they exist since it helps demystify some of the methods on `HashSet`, `HashMap`, `BTreeSet`, and `BTreeMap` but it's very rare that we would ever need to impl these traits for any of our types because it's very rare that we would ever need create a pair of types where one is the "borrowed" version of the other in the first place. If we have some `T` then `&T` will get the job done 99.99% of the time, and `T: Borrow<T>` is already implemented for all `T` because of a generic blanket impl, so we don't need to manually impl it and we don't need to create some `U` such that `T: Borrow<U>`.

知道这些 trait 并理解它们存在的原因是很好的，因为这有助于解开 `HashSet`、`HashMap`、`BTreeSet` 和 `BTreeMap` 上的一些方法，但是我们很少需要为我们的任何类型实现这些 trait，因为我们很少需要创建一对类型，其中一个是另一个的 "借用" 版本。如果我们有一些 `T`，那么 `T` 在 99.99% 的情况下都能完成工作，而且 `T: Borrow<T>` 已经为所有的 `T` 实现了，因为有一个通用的一揽子实现，所以我们不需要手动实现它，我们也不需要创建一些 `U`，使 `T: Borrow<U>`。


### ToOwned

预备知识
- Self
- Methods
- Default Impls
- Clone
- Borrow & BorrowMut

```rs
trait ToOwned {
    type Owned: Borrow<Self>;
    fn to_owned(&self) -> Self::Owned;

    // provided default impls
    fn clone_into(&self, target: &mut Self::Owned);
}
```

`ToOwned` is a more generic version of `Clone`. `Clone` allows us to take a `&T` and turn it into an `T` but `ToOwned` allows us to take a `&Borrowed` and turn it into a `Owned` where `Owned: Borrow<Borrowed>`.

In other words, we can't "clone" a `&str` into a `String`, or a `&Path` into a `PathBuf`, or an `&OsStr` into an `OsString`, since the `clone` method signature doesn't support this kind of cross-type cloning, and that's what `ToOwned` was made for.

For similar reasons as `Borrow` and `BorrowMut`, it's good to be aware of this trait and understand why it exists but it's very rare we'll ever need to impl it for any of our types.

`ToOwned` 是 `Clone` 的一个更通用的版本。`Clone` 允许我们把一个 `&T` 变成一个 `T`，但 `ToOwned` 允许我们把一个 `&Borrowed` 变成一个 `Owned`，其中 `Owned: Borrow<Borrowed>`。

换句话说，我们不能把一个 `&str` 克隆成一个 `String`，或者把一个 `&Path` 克隆成一个 `PathBuf`，或者把一个 `&OsStr` 克隆成一个 `OsString`，因为 `clone` 方法签名不支持这种跨类型克隆，而这正是 `ToOwned` 的用途。

出于与 `Borrow` 和 `BorrowMut` 类似的原因，知道这个 trait 并理解它存在的原因是很好的，但我们很少需要为我们的任何类型实现这个 trait。

## Iteration Traits



### Iterator

预备知识
- Self
- Methods
- Associated Types
- Default Impls

```rs
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;

    // provided default impls
    fn size_hint(&self) -> (usize, Option<usize>);
    fn count(self) -> usize;
    fn last(self) -> Option<Self::Item>;
    fn advance_by(&mut self, n: usize) -> Result<(), usize>;
    fn nth(&mut self, n: usize) -> Option<Self::Item>;
    fn step_by(self, step: usize) -> StepBy<Self>;
    fn chain<U>(
        self,
        other: U
    ) -> Chain<Self, <U as IntoIterator>::IntoIter>
    where
        U: IntoIterator<Item = Self::Item>;
    fn zip<U>(self, other: U) -> Zip<Self, <U as IntoIterator>::IntoIter>
    where
        U: IntoIterator;
    fn map<B, F>(self, f: F) -> Map<Self, F>
    where
        F: FnMut(Self::Item) -> B;
    fn for_each<F>(self, f: F)
    where
        F: FnMut(Self::Item);
    fn filter<P>(self, predicate: P) -> Filter<Self, P>
    where
        P: FnMut(&Self::Item) -> bool;
    fn filter_map<B, F>(self, f: F) -> FilterMap<Self, F>
    where
        F: FnMut(Self::Item) -> Option<B>;
    fn enumerate(self) -> Enumerate<Self>;
    fn peekable(self) -> Peekable<Self>;
    fn skip_while<P>(self, predicate: P) -> SkipWhile<Self, P>
    where
        P: FnMut(&Self::Item) -> bool;
    fn take_while<P>(self, predicate: P) -> TakeWhile<Self, P>
    where
        P: FnMut(&Self::Item) -> bool;
    fn map_while<B, P>(self, predicate: P) -> MapWhile<Self, P>
    where
        P: FnMut(Self::Item) -> Option<B>;
    fn skip(self, n: usize) -> Skip<Self>;
    fn take(self, n: usize) -> Take<Self>;
    fn scan<St, B, F>(self, initial_state: St, f: F) -> Scan<Self, St, F>
    where
        F: FnMut(&mut St, Self::Item) -> Option<B>;
    fn flat_map<U, F>(self, f: F) -> FlatMap<Self, U, F>
    where
        F: FnMut(Self::Item) -> U,
        U: IntoIterator;
    fn flatten(self) -> Flatten<Self>
    where
        Self::Item: IntoIterator;
    fn fuse(self) -> Fuse<Self>;
    fn inspect<F>(self, f: F) -> Inspect<Self, F>
    where
        F: FnMut(&Self::Item);
    fn by_ref(&mut self) -> &mut Self;
    fn collect<B>(self) -> B
    where
        B: FromIterator<Self::Item>;
    fn partition<B, F>(self, f: F) -> (B, B)
    where
        F: FnMut(&Self::Item) -> bool,
        B: Default + Extend<Self::Item>;
    fn partition_in_place<'a, T, P>(self, predicate: P) -> usize
    where
        Self: DoubleEndedIterator<Item = &'a mut T>,
        T: 'a,
        P: FnMut(&T) -> bool;
    fn is_partitioned<P>(self, predicate: P) -> bool
    where
        P: FnMut(Self::Item) -> bool;
    fn try_fold<B, F, R>(&mut self, init: B, f: F) -> R
    where
        F: FnMut(B, Self::Item) -> R,
        R: Try<Ok = B>;
    fn try_for_each<F, R>(&mut self, f: F) -> R
    where
        F: FnMut(Self::Item) -> R,
        R: Try<Ok = ()>;
    fn fold<B, F>(self, init: B, f: F) -> B
    where
        F: FnMut(B, Self::Item) -> B;
    fn fold_first<F>(self, f: F) -> Option<Self::Item>
    where
        F: FnMut(Self::Item, Self::Item) -> Self::Item;
    fn all<F>(&mut self, f: F) -> bool
    where
        F: FnMut(Self::Item) -> bool;
    fn any<F>(&mut self, f: F) -> bool
    where
        F: FnMut(Self::Item) -> bool;
    fn find<P>(&mut self, predicate: P) -> Option<Self::Item>
    where
        P: FnMut(&Self::Item) -> bool;
    fn find_map<B, F>(&mut self, f: F) -> Option<B>
    where
        F: FnMut(Self::Item) -> Option<B>;
    fn try_find<F, R>(
        &mut self,
        f: F
    ) -> Result<Option<Self::Item>, <R as Try>::Error>
    where
        F: FnMut(&Self::Item) -> R,
        R: Try<Ok = bool>;
    fn position<P>(&mut self, predicate: P) -> Option<usize>
    where
        P: FnMut(Self::Item) -> bool;
    fn rposition<P>(&mut self, predicate: P) -> Option<usize>
    where
        Self: ExactSizeIterator + DoubleEndedIterator,
        P: FnMut(Self::Item) -> bool;
    fn max(self) -> Option<Self::Item>
    where
        Self::Item: Ord;
    fn min(self) -> Option<Self::Item>
    where
        Self::Item: Ord;
    fn max_by_key<B, F>(self, f: F) -> Option<Self::Item>
    where
        F: FnMut(&Self::Item) -> B,
        B: Ord;
    fn max_by<F>(self, compare: F) -> Option<Self::Item>
    where
        F: FnMut(&Self::Item, &Self::Item) -> Ordering;
    fn min_by_key<B, F>(self, f: F) -> Option<Self::Item>
    where
        F: FnMut(&Self::Item) -> B,
        B: Ord;
    fn min_by<F>(self, compare: F) -> Option<Self::Item>
    where
        F: FnMut(&Self::Item, &Self::Item) -> Ordering;
    fn rev(self) -> Rev<Self>
    where
        Self: DoubleEndedIterator;
    fn unzip<A, B, FromA, FromB>(self) -> (FromA, FromB)
    where
        Self: Iterator<Item = (A, B)>,
        FromA: Default + Extend<A>,
        FromB: Default + Extend<B>;
    fn copied<'a, T>(self) -> Copied<Self>
    where
        Self: Iterator<Item = &'a T>,
        T: 'a + Copy;
    fn cloned<'a, T>(self) -> Cloned<Self>
    where
        Self: Iterator<Item = &'a T>,
        T: 'a + Clone;
    fn cycle(self) -> Cycle<Self>
    where
        Self: Clone;
    fn sum<S>(self) -> S
    where
        S: Sum<Self::Item>;
    fn product<P>(self) -> P
    where
        P: Product<Self::Item>;
    fn cmp<I>(self, other: I) -> Ordering
    where
        I: IntoIterator<Item = Self::Item>,
        Self::Item: Ord;
    fn cmp_by<I, F>(self, other: I, cmp: F) -> Ordering
    where
        F: FnMut(Self::Item, <I as IntoIterator>::Item) -> Ordering,
        I: IntoIterator;
    fn partial_cmp<I>(self, other: I) -> Option<Ordering>
    where
        I: IntoIterator,
        Self::Item: PartialOrd<<I as IntoIterator>::Item>;
    fn partial_cmp_by<I, F>(
        self,
        other: I,
        partial_cmp: F
    ) -> Option<Ordering>
    where
        F: FnMut(Self::Item, <I as IntoIterator>::Item) -> Option<Ordering>,
        I: IntoIterator;
    fn eq<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialEq<<I as IntoIterator>::Item>;
    fn eq_by<I, F>(self, other: I, eq: F) -> bool
    where
        F: FnMut(Self::Item, <I as IntoIterator>::Item) -> bool,
        I: IntoIterator;
    fn ne<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialEq<<I as IntoIterator>::Item>;
    fn lt<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialOrd<<I as IntoIterator>::Item>;
    fn le<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialOrd<<I as IntoIterator>::Item>;
    fn gt<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialOrd<<I as IntoIterator>::Item>;
    fn ge<I>(self, other: I) -> bool
    where
        I: IntoIterator,
        Self::Item: PartialOrd<<I as IntoIterator>::Item>;
    fn is_sorted(self) -> bool
    where
        Self::Item: PartialOrd<Self::Item>;
    fn is_sorted_by<F>(self, compare: F) -> bool
    where
        F: FnMut(&Self::Item, &Self::Item) -> Option<Ordering>;
    fn is_sorted_by_key<F, K>(self, f: F) -> bool
    where
        F: FnMut(Self::Item) -> K,
        K: PartialOrd<K>;
}
```

`Iterator<Item = T>` 类型可以被迭代，并会产生 `T` 类型。没有 `IteratorMut` trait。每个 `Iterator` 实现可以通过 `Item` 关联类型指定它是返回不可变引用、可变引用还是拥有其值。

| `Vec<T>` 方法   | 返回                       |
|-----------------|---------------------------|
| `.iter()`       | `Iterator<Item = &T>`     |
| `.iter_mut()`   | `Iterator<Item = &mut T>` |
| `.into_iter()`  | `Iterator<Item = T>`      |

对于初学者来说，有些东西不是很明显，但中级 Rustaceans 认为是理所当然的，那就是大多数类型都不是他们本身的迭代器。如果一个类型是可迭代的，我们几乎总是实现一些自定义的迭代器类型来迭代它，而不是试图让它自己迭代。

```rs
struct MyType {
    items: Vec<String>
}

impl MyType {
    fn iter(&self) -> impl Iterator<Item = &String> {
        MyTypeIterator {
            index: 0,
            items: &self.items
        }
    }
}

struct MyTypeIterator<'a> {
    index: usize,
    items: &'a Vec<String>
}

impl<'a> Iterator for MyTypeIterator<'a> {
    type Item = &'a String;
    fn next(&mut self) -> Option<Self::Item> {
        if self.index >= self.items.len() {
            None
        } else {
            let item = &self.items[self.index];
            self.index += 1;
            Some(item)
        }
    }
}
```

为了便于教学，上面的例子展示了如何从头开始实现一个 `Iterator`，但在这种情况下，习惯性的解决方案将只是遵从 `Vec` 的 `iter` 方法。

```rs
struct MyType {
    items: Vec<String>
}

impl MyType {
    fn iter(&self) -> impl Iterator<Item = &String> {
        self.items.iter()
    }
}
```

另外，这也是一个很好的通用全面实现，要注意。

```rs
impl<I: Iterator + ?Sized> Iterator for &mut I;
```

它说任何对迭代器的可变引用也是迭代器，这一点很有用，因为它允许我们使用 `self` 接收器，就像使用 `&mut self` 接收器一样。了解这一点很有用，因为它允许我们使用迭代器方法与 `self` 接收器，就像它们有 `&mut self` 接收器一样。

举个例子，想象一下，我们有一个函数，它可以处理一个超过三个项的迭代器，但是函数的第一步是取出迭代器的前三项，并在迭代剩下的项之前分别处理它们，下面是一个初学者可能尝试写这个函数的方法。

```rs
fn example<I: Iterator<Item = i32>>(mut iter: I) {
    let first3: Vec<i32> = iter.take(3).collect();
    for item in iter { // ❌ iter consumed in line above
        // process remaining items
    }
}
```

嗯，这很烦人。`take` 方法有一个 `self` 接收器，所以我们似乎不能在不消耗整个迭代器的情况下调用它。下面是上面代码的一个天真的重构。

```rs
fn example<I: Iterator<Item = i32>>(mut iter: I) {
    let first3: Vec<i32> = vec![
        iter.next().unwrap(),
        iter.next().unwrap(),
        iter.next().unwrap(),
    ];
    for item in iter { // ✅
        // process remaining items
    }
}
```

这还算好的。然而，惯用的重构其实是这样的。

```rs
fn example<I: Iterator<Item = i32>>(mut iter: I) {
    let first3: Vec<i32> = iter.by_ref().take(3).collect();
    for item in iter { // ✅
        // process remaining items
    }
}
```

不太容易发现。但无论如何，现在我们知道了。

另外，对于什么可以是迭代器，什么不能是迭代器，并没有什么规则或约定。如果类型是 `Iterator`，那么它就是一个迭代器。标准库中的一些创造性的例子如下。

```rs
use std::sync::mpsc::channel;
use std::thread;

fn paths_can_be_iterated(path: &Path) {
    for part in path {
        // iterate over parts of a path
    }
}

fn receivers_can_be_iterated() {
    let (send, recv) = channel();

    thread::spawn(move || {
        send.send(1).unwrap();
        send.send(2).unwrap();
        send.send(3).unwrap();
    });

    for received in recv {
        // iterate over received values
    }
}
```

### IntoIterator

预备知识
- Self
- Methods
- Associated Types
- Iterator

```rs
trait IntoIterator
where
    <Self::IntoIter as Iterator>::Item == Self::Item,
{
    type Item;
    type IntoIter: Iterator;
    fn into_iter(self) -> Self::IntoIter;
}
```

`IntoIterator` 类型可以被转换为迭代器，因此得名。当一个类型在 `for-in` 循环中使用时，会调用 `into_iter` 方法。

```rs
// vec = Vec<T>
for v in vec {} // v = T

// above line desugared
for v in vec.into_iter() {}
```

不仅 `Vec` 实现了 `IntoIterator`，如果我们想分别迭代不可变引用或可变引用而不是拥有其值，`&Vec` 和 `&mut Vec` 也分别实现了 `IntoIterator`。

```rs
// vec = Vec<T>
for v in &vec {} // v = &T

// above example desugared
for v in (&vec).into_iter() {}

// vec = Vec<T>
for v in &mut vec {} // v = &mut T

// above example desugared
for v in (&mut vec).into_iter() {}
```

### FromIterator

预备知识
- Self
- Functions
- Generic Parameters
- Iterator
- IntoIterator

```rs
trait FromIterator<A> {
    fn from_iter<T>(iter: T) -> Self
    where
        T: IntoIterator<Item = A>;
}
```

`FromIterator` 类型可以从迭代器中创建，因此也被称为 `FromIterator`。`FromIterator` 最常见和最习惯的用法是调用 `Iterator` 上的 `collect` 方法。

```rs
fn collect<B>(self) -> B
where
    B: FromIterator<Self::Item>;
```

将 `Iterator<Item = char>` 收集成 `String` 的例子如下:

```rs
fn filter_letters(string: &str) -> String {
    string.chars().filter(|c| c.is_alphabetic()).collect()
}
```

标准库中的所有集合都实现了 `IntoIterator` 和 `FromIterator`，这样可以更容易在它们之间进行转换。

```rs
use std::collections::{BTreeSet, HashMap, HashSet, LinkedList};

// String -> HashSet<char>
fn unique_chars(string: &str) -> HashSet<char> {
    string.chars().collect()
}

// Vec<T> -> BTreeSet<T>
fn ordered_unique_items<T: Ord>(vec: Vec<T>) -> BTreeSet<T> {
    vec.into_iter().collect()
}

// HashMap<K, V> -> LinkedList<(K, V)>
fn entry_list<K, V>(map: HashMap<K, V>) -> LinkedList<(K, V)> {
    map.into_iter().collect()
}

// and countless more possible examples
```

## I/O Traits

### Read & Write

预备知识
- Self
- Methods
- Scope
- Generic Blanket Impls

```rs
trait Read {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;

    // provided default impls
    fn read_vectored(&mut self, bufs: &mut [IoSliceMut<'_>]) -> Result<usize>;
    fn is_read_vectored(&self) -> bool;
    unsafe fn initializer(&self) -> Initializer;
    fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize>;
    fn read_to_string(&mut self, buf: &mut String) -> Result<usize>;
    fn read_exact(&mut self, buf: &mut [u8]) -> Result<()>;
    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized;
    fn bytes(self) -> Bytes<Self>
    where
        Self: Sized;
    fn chain<R: Read>(self, next: R) -> Chain<Self, R>
    where
        Self: Sized;
    fn take(self, limit: u64) -> Take<Self>
    where
        Self: Sized;
}

trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;

    // provided default impls
    fn write_vectored(&mut self, bufs: &[IoSlice<'_>]) -> Result<usize>;
    fn is_write_vectored(&self) -> bool;
    fn write_all(&mut self, buf: &[u8]) -> Result<()>;
    fn write_all_vectored(&mut self, bufs: &mut [IoSlice<'_>]) -> Result<()>;
    fn write_fmt(&mut self, fmt: Arguments<'_>) -> Result<()>;
    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized;
}
```

通用的全面实现值得了解。

```rs
impl<R: Read + ?Sized> Read for &mut R;
impl<W: Write + ?Sized> Write for &mut W;
```

这说明任何对 `Read` 类型的可变引用也是 `Read`，对 `Write` 也是如此。了解这一点很有用，因为它允许我们使用任何带有 `self` 接收器的方法，就像它有一个 `&mut self` 接收器一样。我们已经在 `Iterator` trait 部分介绍了如何做到这一点以及为什么它很有用，所以我不打算在这里再次重复。

我想指出，`&[u8]` 实现了 `Read`，`Vec<u8>` 实现了 `Write`，所以我们可以很容易地使用 `String` 对我们的文件处理函数进行单元测试，这些函数很容易转换为 `&[u8]` 和 `Vec<u8>`。

```rs
use std::path::Path;
use std::fs::File;
use std::io::Read;
use std::io::Write;
use std::io;

// function we want to test
fn uppercase<R: Read, W: Write>(mut read: R, mut write: W) -> Result<(), io::Error> {
    let mut buffer = String::new();
    read.read_to_string(&mut buffer)?;
    let uppercase = buffer.to_uppercase();
    write.write_all(uppercase.as_bytes())?;
    write.flush()?;
    Ok(())
}

// in actual program we'd pass Files
fn example(in_path: &Path, out_path: &Path) -> Result<(), io::Error> {
    let in_file = File::open(in_path)?;
    let out_file = File::open(out_path)?;
    uppercase(in_file, out_file)
}


// however in unit tests we can use Strings!
#[test] // ✅
fn example_test() {
    let in_file: String = "i am screaming".into();
    let mut out_file: Vec<u8> = Vec::new();
    uppercase(in_file.as_bytes(), &mut out_file).unwrap();
    let out_result = String::from_utf8(out_file).unwrap();
    assert_eq!(out_result, "I AM SCREAMING");
}
```

## 结论

我们一起学到了很多东西! 事实上，太多了。它现在是我们的了:

![rust standard library traits](../assets/jason-jarvis-stdlib-traits.png)

_Artist credit: [The Jenkins Comic](https://thejenkinscomic.wordpress.com/2020/05/06/memory/)_


## 讨论

在这里讨论这篇文章
- [Github](https://github.com/pretzelhammer/rust-blog/discussions)
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/ml9shl/tour_of_rusts_standard_library_traits/)
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-tour-of-rusts-standard-library-traits/57974)
- [Twitter](https://twitter.com/pretzelhammer/status/1379561720176336902)
- [lobste.rs](https://lobste.rs/s/g27ezp/tour_rust_s_standard_library_traits)
- [rust subreddit](https://www.reddit.com/r/rust/comments/mmrao0/tour_of_rusts_standard_library_traits/)


## 通知

当下一篇博文发布时，会收到通知
- [Following pretzelhammer on Twitter](https://twitter.com/pretzelhammer) or
- Watching this repo's releases (click `Watch` -> click `Custom` -> select `Releases` -> click `Apply`)



## 更多阅读

- [Sizedness in Rust](https://ohmyweekly.github.io/notes/2021-04-11-sizedness-in-rust/)
- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Learning Rust in 2020](./learning-rust-in-2020.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](./too-many-brainfuck-compilers.md)
