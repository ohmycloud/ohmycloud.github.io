+++
title = "究竟什么是 Future？"
date = 2025-04-27
[taxonomies]
  tags = ["rust", "Future"]
+++

原文链接：https://natkr.com/2025-04-10-async-from-scratch-1

有很多关于如何从"用户角度"使用异步 Rust 的指南，但我认为理解它如何工作，那些 `async` 代码块实际上意味着什么也很重要。为什么你会遇到那些奇怪的固定(pinning)错误。

这是系列文章的第一篇，我们将逐步构建，重新发明现代异步 Rust 环境，试图解释其中的原因和方法。它最终不会成为 Tokio 的竞争对手，但希望它能让你之后理解这些概念变得不那么令人生畏。

我写这个系列是针对那些已经编写过 `trait` 和一两个 `async fn` 的人，但如果"轮询(polling)"、"固定(pinning)"或"唤醒器(wakers)"对你来说毫无意义，也不用担心。这正是我们将要一步一步解开的内容！

现在...如果你写过任何异步 Rust 代码，它可能看起来像这样：

```rs
async fn trick_or_treat() {
    for house in &STREET {
        match deman_treat(house).await {
            Ok(candy) => eat(candy).await,
            Err(_) => play_trick(house).await,
        }
    }
}
```

但是，呃，这究竟做了什么？为什么我需要 `await` 东西，`async fn` 与其他 `fn` 有何不同，而且所有这些实际上...到底做了什么？

# 在 Future 中...

要理解这些，我们需要回溯一下。我们将会遇到一个你可能之前没有真正见过的特性（trait）。我们将不得不处理...Future。就像 `Add` 定义了 `a + b` 是否有效一样，`Future` 定义了"可以被 `.await` 的东西"。¹ 它看起来像这样：

```rs
use std::{task::{Context, Poll}, pin::Pin};

trait Future {
    type Output;
    fn poll(
        self: Pin<&mut Self>,
        context: &mut Context<'_>,
    ) -> Poll<Self::Output> {
        Poll::Pending
    }
}
```

嗯...对于一个只有一个函数的 trait 来说，这个函数签名相当复杂。甚至可以说有点让人不知所措，尤其是如果你是 Rust 新手。
但其中大部分内容并不是真的那么重要，所以我们可以暂时做一些简化。别担心，我们稍后会回来讨论所有这些内容。现在，我们可以去掉大部分内容，假装它看起来像这样：

```rs
use std::task::Poll;

trait SimpleFuture<Output> {
    fn poll(&mut self) -> Poll<Output>;
}
```

那么这个 `(Simple)Future::poll` 到底做什么呢？

# 让我们看看 poll 是怎么回事

本质上，`Future` 是一个可以在需要等待某事时暂停自身的函数调用。²

`poll`  要求 `Future` 尝试继续执行，如果它能够完成则返回 `Poll::Ready`，如果必须再次暂停自身则返回 `Poll::Pending`。³

这可以从非常简单的开始。我们可以有一个总是准备好产生一些极其随机数字的 `Future`：

```rs
struct FairDice;
impl SimpleFuture<u8> for FairDice {
    fn poll(&mut self) -> Poll<u8> {
        Poll::Ready(4) // 通过公平骰子掷出
    }
}
```

我们也可以永远等待，争取一些喘息空间：


```rs
struct LookBusy;
impl SimpleFuture<()> for LookBusy {
    fn poll(&mut self) -> Poll<()> {
        Poll::Pending
    }
}
```

这些都是相当简单的问题，但要能够在中途暂停事物，我们需要保存所有应该保留的上下文。
这就是我们的 `Future` 作为一个类型变得相关的地方，而不仅仅是标记调用哪个 `poll` 函数。我们可以有一个需要被轮询10次才能完成的 `Future`：

```rs
struct Stubborn {
    counter: u8,
}
impl SimpleFuture<()> for Stubborn {
    fn poll(&mut self) -> Poll<()> {
        self.counter += 1;
        if self.counter == 10 {
            Poll::Ready(())
        } else {
            Poll::Pending
        }
    }
}
```

或者一个委托给另一个Future的包装器：

```rs
struct LoadedDice {
    inner: FairDice,
}
impl SimpleFuture<u8> for LoadedDice {
    fn poll(&mut self) -> Poll<u8> {
        match self.inner.poll() {
            Poll::Ready(x) => Poll::Ready(x + 1),
            Poll::Pending => Poll::Pending,
        }
    }
}
```

现在...编写所有那些"匹配poll，如果是pending则返回，如果是ready则继续"的代码块也会变得相当繁琐。幸运的是，Rust提供了 `ready!` 宏来为我们做这件事。⁴

上面的例子也可以这样写：

```rs
use std::task::ready;

struct LoadedDice {
    inner: FairDice,
}
impl SimpleFuture<u8> for LoadedDice {
    fn poll(&mut self) -> Poll<u8> {
        let x = ready!(self.inner.poll());
        Poll::Ready(x + 1)
    }
}
```

但最终我们会希望能够多次 `await`，并且在它们之间保存一些东西。例如，我们可能想要对我们的骰子结果进行配对求和：

```rs
async fn fair_dice() -> u8 {
    4 // 仍然保证完全公平
}
async fn fair_dice_pair() -> u8 {
    let first_dice = fair_dice().await;
    let second_dice = fair_dice().await;
    first_dice + second_dice
}
```

我们可以通过在枚举中保存共享状态来做到这一点，为每个 `await` 点创建一个变体。这种重新排列被称为"状态机"，这也是 `async fn` 在幕后为我们做的事情。最终看起来像这样：

```rs
enum FairDicePair {
    Init,
    RollingFirstDice {
        first_dice: FairDice,
    },
    RollingSecondDice {
        first_dice: u8,
        second_dice: FairDice,
    }
}

impl SimpleFuture<u8> for FairDicePair {
    fn poll(&mut self) -> Poll<u8> {
        // 循环让我们继续运行状态机
        // 直到其中一个ready!子句暂停我们。
        loop {
            match self {
                Self::Init => {
                    *self = Self::RollingFirstDice {
                        first_dice: FairDice,
                    };
                },
                Self::RollingFirstDice { first_dice } => {
                    // 每次我们被poll()时，我们会再次做_所有_直到
                    // 下一个ready!的事情(毕竟poll()只是另一个方法)，
                    // 所以它应该是每次调用时我们做的第一件（非平凡的）事情。
                    let first_dice = ready!(first_dice.poll());
                    *self = Self::RollingSecondDice {
                        first_dice,
                        second_dice: FairDice,
                    }
                }
                Self::RollingSecondDice { first_dice, second_dice } => {
                    let second_dice = ready!(second_dice.poll());
                    return Poll::Ready(*first_dice + second_dice)
                }
            }
        }
    }
}
```

这...只是有点...更冗长。  
但另一方面，原始的 `poll` 让我们可以做 `async fn` 无法真正表达的事情。例如，我们可以构建一个超时机制，只允许我们对某个任意包装的 `Future` 进行有限次数的轮询：⁵

```rs
struct Timeout {
    inner: Stubborn,
    polls_left: u8,
}
#[derive(Debug)]
struct TimedOut;
impl SimpleFuture<Result<(), TimedOut>> for Timeout {
    fn poll(&mut self) -> Poll<Result<(), TimedOut>> {
        match self.polls_left.checked_sub(1) {
            Some(x) => self.polls_left = x,
            None => return Poll::Ready(Err(TimedOut)),
        }
        let inner = ready!(self.inner.poll());
        Poll::Ready(Ok(inner))
    }
}
```

# 让我们来运行一下

所以...我们已经定义了我们的 `(Simple)Future`。事实上，定义了好几个。但除非我们能够真正运行它们，否则它们并没有多大价值。我们该如何做到这一点呢？

很简单。我们只需不断调用 `poll` 直到它返回 `Ready`。⁶

```rs
fn run_future<Output, F: SimpleFuture<Output>>(mut fut: F) -> Output {
    loop {
        if let Poll::Ready(out) = fut.poll() {
            return out;
        }
    }
}
```

例如：

```rs
println!("=> {}", run_future(FairDice));
// => 4
```

现在，这里有一个问题。只是一个小问题。一个微小的问题。一个微小的玩具级问题。  
在等待我们的 `Future` 完成时，我们浪费了大量的 CPU 周期，只是一遍又一遍地调用 `poll`。⁷ 这并不理想，但现在，让我们暂时搁置这个问题。我们很快就会回来解决它。

# 进入组合器世界

正如我们所见，尝试将所有逻辑都写成 `poll` 很快就会变得难以控制，但有时我们确实需要表达一些常规函数调用序列无法表达的东西。⁸
有没有办法让我们组合它们，这样我们就可以使用最适合工作的方式？  
嗯，是的。我们可以编写组合器，将我们的特殊逻辑泛化为新的构建块，然后我们的 `async fn` 可以重用这些构建块。  
例如，我们的 `Timeout` 示例可以修改为接受任意 `Future`，而不仅仅是 `Stubborn`：

```rs
struct Timeout<F> {
    inner: F,
    polls_left: u8,
}
struct TimedOut;
impl<F, Output> SimpleFuture<Result<Output, TimedOut>> for Timeout<F>
where
    F: SimpleFuture<Output>,
{
    fn poll(&mut self) -> Poll<Result<Output, TimedOut>> {
        match self.polls_left.checked_sub(1) {
            Some(x) => self.polls_left = x,
            None => return Poll::Ready(Err(TimedOut)),
        }
        let inner = ready!(self.inner.poll());
        Poll::Ready(Ok(inner))
    }
}
fn with_timeout<F, Output>(
    inner: F,
    max_polls: u8,
) -> impl SimpleFuture<Result<Output, TimedOut>>
where
    F: SimpleFuture<Output>,
{
    Timeout {
        inner,
        polls_left: max_polls,
    }
}
```

然后我们可以在我们的 `async fn` 中使用它，通过在 `await` 之前包装子 `Future`：⁹

```rs
async fn send_email(target: &str, msg: &str) {}
struct TimedOut;
async fn with_timeout<F: Future>(inner: F, max_polls: u8) -> Result<F::Output, TimedOut> { Ok(inner.await) }

async fn send_email_with_retry() {
    for _ in 0..5 {
        if with_timeout(send_email("nat@nullable.se", "message"), 10).await.is_ok() {
            return;
        }
    }
    panic!("repeatedly timed out trying to send email, giving up...");
}
```

然后我们可以在我们的async fn中使用它，通过在await之前包装子Future：⁹

```rs
async fn send_email(target: &str, msg: &str) {}
struct TimedOut;
async fn with_timeout<F: Future>(inner: F, max_polls: u8) -> Result<F::Output, TimedOut> { Ok(inner.await) }
async fn send_email_with_retry() {
    for _ in 0..5 {
        if with_timeout(send_email("nat@nullable.se", "message"), 10).await.is_ok() {
            return;
        }
    }
    panic!("repeatedly timed out trying to send email, giving up...");
}
```

# 输入，输出

我们花了一些时间研究如何组合我们的 Futures...但它们实际上还没有...做任何事情。如果一个 Future 在电脑中运行，但没有人在周围运行它...我们实际上做的也不过是消耗一些电力而已。

为了有用，我们需要能够与外部系统交互。网络调用等等。

例如，让我们尝试从TCP套接字读取一些内容。我们将提供一个服务器，每当我们连接时，它都会提供我们的行李代码。当然，这是为了安全保管。

```rs
let listener = std::net::TcpListener::bind("127.0.0.1:9191").unwrap();
std::thread::spawn(move || {
    use std::{io::Write, time::Duration};
    let (mut conn, _) = listener.accept().unwrap();
    // Ensure that the client needs to wait for
    std::thread::sleep(Duration::from_millis(200));
    conn.write_all(&[1, 2, 3, 4, 5]).unwrap();
});
```

为了做到这一点，我们需要做几件事：

1. 创建套接字（这在Rust的API中隐式发生）  
2. 连接到远程目标  
3. 将套接字[配置](https://doc.rust-lang.org/stable/std/net/struct.TcpStream.html#method.set_nonblocking)为非阻塞（因为否则接收本身就会等待消息，阻止任何其他Futures在同一线程上运行）¹⁰  
4. 尝试[读取](https://doc.rust-lang.org/stable/std/net/struct.TcpStream.html#method.read)消息  
5. 如果读取返回 [WouldBlock](https://doc.rust-lang.org/stable/std/io/enum.ErrorKind.html#variant.WouldBlock)，则返回 `Pending` 并在下一次轮询时从步骤4重试  

将它们放在一起看起来像这样：

```rs
use std::{io::Read, net::TcpStream};

struct TcpRead<'a> {
    socket: &'a mut TcpStream,
    buffer: &'a mut [u8],
}
impl<'a> SimpleFuture<usize> for TcpRead<'a> {
    fn poll(&mut self) -> Poll<usize> {
        match self.socket.read(self.buffer) {
            Err(err) if err.kind() == std::io::ErrorKind::WouldBlock => Poll::Pending,
            size => Poll::Ready(size.unwrap()),
        }
    }
}

let luggage_code_server_address = "127.0.0.1:9191";
let mut socket = TcpStream::connect(luggage_code_server_address).unwrap();
socket.set_nonblocking(true).unwrap();

let mut buffer = [0; 16];
let received = run_future(TcpRead {
    socket: &mut socket,
    buffer: &mut buffer,
});
println!("=> The luggage code is {:?}", &buffer[..received]);
// => The luggage code is [1, 2, 3, 4, 5]
```

# 下次再见...

希望你现在对 Futures 如何交互的总体思路有了一些了解。我们已经看到了如何定义、运行、组合它们，并用它们与网络服务进行通信。

但正如我提到的，我们实际上只讨论了我们简化的 `SimpleFuture` 变体。在这个系列的其余部分，我将专注于一个接一个地拉开那些帷幕，直到我们回到真正的 Future trait。

首先，我们的 `SimpleFuture` 相当浪费，因为我们需要不断地轮询，而不仅仅是在我们有有用的事情要做的时候。解决这个问题的方法叫做唤醒器(waker)。但那是下次的话题了...

更新：现在已经发布了，去看看吧！

¹ 实际上，`.await` 是由 `IntoFuture` 定义的...但那只是一个薄薄的转换包装器。  
² 比如等待计时器、通过网络接收消息，诸如此类的事情。  
³ 如果这听起来像 `Option`...它基本上就是！除了当我们的类型嵌入它们代表的含义时，代码通常会变得更清晰。Option 可能因为许多原因而是 None，但 `Pending` 总是表示正在进行中的工作。  
⁴ 如果这让你想起?运算符...是的，这是另一个相似之处。  
⁵ 实际上，你会想用时间而不是尝试计算 `poll` 调用次数...但处理时间会引入更多我现在不想处理的活动部件。  
⁶ 在这一点之后再次调用 `poll` 是未定义的行为，但通常它要么会崩溃，要么会永远返回 `Pending`。  
⁷ 有人曾经说过关于这暗示的理智的事情...  
⁸ 而且即使是那些常规序列也需要调用最终会做事情的原语。毕竟，Futures 不是凭空形成的。  
⁹ 在我们想象的世界中，Rust 支持 await-ing SimpleFuture 而不是 Future。  
¹⁰ 是的，实际上这应该在连接之前发生，因为我们不想在等待连接时让我们的线程保持忙碌。现在，让我们假装连接已经存在。由于现在超出范围的复杂原因，Rust 标准库并不真正处理异步连接操作。Tokio 为你做了这个。
