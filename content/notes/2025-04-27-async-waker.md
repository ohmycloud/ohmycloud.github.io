+++
title = "Waker"
date = 2025-04-27
[taxonomies]
  tags = ["rust", "waker"]
+++

原文链接：https://natkr.com/2025-04-15-async-from-scratch-2

# 翻译

那么，你已经读过了我上一篇文章。你受到了启发。甚至感到兴奋。将 `SimpleFuture` 部署到生产环境。启动了几个工作线程来分担负载。然后就可以安心地度过周五了。毕竟这是Rust，能出什么问题呢？

...然后有人查看了 CPU 使用率。

哎呀。幸好我们不用为那些 CPU 时间付费，对吧？

也许我们应该研究一下那些我们遗留未解决的星号注释。今天我们不会全部解决¹，但我们必须从某处开始。

那么，`poll` 到底是什么时候发生的呢？

现在我要开始揭开帷幕，揭露第一个谎言：`poll` 不仅仅负责一项工作（尝试取得进展）。

它实际上还有一个秘密的第二项工作：确保无论什么在运行 `Future`，在下一次适合再次 `poll` 它的时候得到通知。这就是 `wakers`（以及我之前一笔带过的 `Context`）的用武之地。²大致看起来是这样的：

```rust
use std::sync::Arc;

trait Wake: Send + Sync {
    // If you haven't seen `self: Foo<Self>` before, 
    // it lets you define methods that apply to certain wrapper types instead.
    // If it helps, `&self` is the same as `self: &Self`.
    fn wake(self: Arc<Self>);
}

struct Context {
    waker: Arc<dyn Wake>,
    // and some other stuff we don't really care about right now
}
```

`Future` 负责确保一旦有新的任务要做，就调用 `wake`，而运行时可以自由地不去 poll Future，直到那个时候。

为了管理这个，我们需要改变我们的 `(Simple)Future` trait 来传递 context：

```rust
use std::task::Poll;

trait SleepyFuture<Output> {
    fn poll(
        &mut self,
        // Our new and shiny
        context: &mut Context,
    ) -> Poll<Output>;
}
```

我们得先学会睡眠，然后才能学会奔跑。

现在，我们旧的运行器基本上仍然是合法的。⁴我们可以继续不断地轮询并提供一个无操作的 `Wake` 来让编译器闭嘴。不被唤醒而 `poll` 我们的 `Future` 总是可以的...只是 `Future` 不能依赖它。

```rust
struct InsomniacWaker;
impl Wake for InsomniacWaker {
    fn wake(self: Arc<Self>) {
        // Who needs to wake up if you never managed to fall asleep?
    }
}

fn insomniac_runner<Output, F: SleepyFuture<Output>>(mut fut: F) -> Output {
    let mut context = Context {
        waker: Arc::new(InsomniacWaker),
    };
    loop {
        if let Poll::Ready(out) = fut.poll(&mut context) {
            return out;
        }
    }
}
```

但是...这并不特别有用。我们现在传递了 context，但是...我们仍然在消耗所有的 CPU 时间。

相反，我们应该提供一个 `Waker`，当没有事情要做时暂停线程：

```rust
use std::sync::{Condvar, Mutex};

#[derive(Default)]
struct SleepWaker {
    awoken: Mutex<bool>,
    wakeup_cond: Condvar,
}

impl SleepWaker {
    fn sleep_until_awoken(&self) {
        let mut awoken = self.wakeup_cond
            .wait_while(
                self.awoken.lock().unwrap(),
                |awoken| !*awoken,
            )
            .unwrap();
        *awoken = false;
    }
}

impl Wake for SleepWaker {
    fn wake(self: Arc<Self>) {
        *self.awoken.lock().unwrap() = true;
        self.wakeup_cond.notify_one();
    }
}
```

条件变量(Condvars)本身就是一个很深的兔子洞，但这里的想法基本上是 `Condvar::wait_while` 每次在 notify_one 被调用时（以及在初始 wait_while 调用时）会对一个被 Mutex 锁定的值进行测试，但在之间会解锁 Mutex⁵。`sleep_until_awoken` 等待 `wake` 被调用，然后重置自己，为下一次调用做好准备。⁶

现在我们只需要改变我们的运行器，在每次 `poll` 之间调用 `sleep_until_awoken`：

```rust
fn run_sleepy_future<Output, F: SleepyFuture<Output>>(mut fut: F) -> Output {
    let waker = Arc::<SleepWaker>::default();
    let mut context = Context { waker: waker.clone() };
    loop {
        match fut.poll(&mut context) {
            Poll::Ready(out) => return out,
            Poll::Pending => waker.sleep_until_awoken(),
        }
    }
}
```

为了确保...在继续之前让我们试一试。确保我们的唤醒机制工作，并且我们在可能的情况下确实在睡眠：

```rust
struct ImmediatelyAwoken(bool);
impl SleepyFuture<()> for ImmediatelyAwoken {
    fn poll(&mut self, context: &mut Context) -> Poll<()> {
        if self.0 {
            Poll::Ready(())
        } else {
            self.0 = true;
            context.waker.clone().wake();
            Poll::Pending
        }
    }
}

struct BackgroundAwoken(bool);
impl SleepyFuture<()> for BackgroundAwoken {
    fn poll(&mut self, context: &mut Context) -> Poll<()> {
        if self.0 {
            Poll::Ready(())
        } else {
            self.0 = true;
            let waker = context.waker.clone();
            std::thread::spawn(|| {
                std::thread::sleep(std::time::Duration::from_millis(200));
                waker.wake();
            });
            Poll::Pending
        }
    }
}

let before_immediate = std::time::Instant::now();
run_sleepy_future(ImmediatelyAwoken(false));
println!("=> immediate: {:?}", before_immediate.elapsed());

let before_background = std::time::Instant::now();
run_sleepy_future(BackgroundAwoken(false));
println!("=> background: {:?}", before_background.elapsed());
```

=> immediate: 7µs
=> background: 200.148889ms

呼！这看起来对我来说是合理的。在索伦之眼（失眠）看到我们之前，让我们继续前进...

![img](https://natkr.com/2025-04-15-async-from-scratch-2/helldivers2-detector-tower.png)


我们已经有足够多的问题了。

# 组合器的回归

这对大多数组合器来说也"直接可用"，只要它们确保将 Context 向下传递到树中。这里是上次的 `with_timeout` 例子；我们需要改变的只是添加 `context` 参数并搜索/替换⁷ `SimpleFuture -> SleepyFuture`：

```rust
use std::task::ready;

struct Timeout<F> {
    inner: F,
    polls_left: u8,
}

#[derive(Debug)]
struct TimedOut;

impl<F, Output> SleepyFuture<Result<Output, TimedOut>> for Timeout<F>
where
    F: SleepyFuture<Output>,
{
    fn poll(
        &mut self,
        context: &mut Context,
    ) -> Poll<Result<Output, TimedOut>> {
        match self.polls_left.checked_sub(1) {
            Some(x) => self.polls_left = x,
            None => return Poll::Ready(Err(TimedOut)),
        }
        let inner = ready!(self.inner.poll(context));
        Poll::Ready(Ok(inner))
    }
}

fn with_timeout<F, Output>(
    inner: F,
    max_polls: u8,
) -> impl SleepyFuture<Result<Output, TimedOut>>
where
    F: SleepyFuture<Output>,
{
    Timeout {
        inner,
        polls_left: max_polls,
    }
}
```

# 睡眠式I/O（或：表达我们的兴趣）

但这一切都是（相对）简单模式。如果我们的I/O例程没有实际被唤醒，这一切都是无用的。可惜...操作系统并不原生支持我们（或 Rust 的）`Wake` trait。

基于我们上次的 `TcpRead` 例子，`Future` 本身仍然相当简单：

```rust
use std::{io::Read, net::TcpStream};

fn wake_when_readable(
    socket: &mut std::net::TcpStream,
    context: &mut Context,
) { todo!() }

struct TcpRead<'a> {
    socket: &'a mut TcpStream,
    buffer: &'a mut [u8],
}

impl SleepyFuture<usize> for TcpRead<'_> {
    fn poll(&mut self, context: &mut Context) -> Poll<usize> {
        match self.socket.read(self.buffer) {
            Err(err) if err.kind() == std::io::ErrorKind::WouldBlock => {
                wake_when_readable(self.socket, context);
                Poll::Pending
            }
            size => Poll::Ready(size.unwrap()),
        }
    }
}
```

但是...呃...我们到底该如何定义 `wake_when_readable`？这...将不得不依赖于你的操作系统，并且超出了 Rust 标准库为我们提供的范围。

在Linux世界⁸中，这种API⁹是 [epoll](https://man7.org/linux/man-pages/man7/epoll.7.html)。它仍然会阻塞，但它让我们可以要求操作系统在任何一组"文件"¹⁰准备好时唤醒我们。在 Rust 世界中，我们可以使用 [nix](https://docs.rs/nix/0.29.0/nix/sys/epoll/index.html) crate来访问它，它提供了一个安全但与系统 API 一对一的映射。¹¹

epoll API 使用起来相当简单：我们需要创建一个 Epoll，注册我们关心的事件¹²，然后等待一些事件发生。wait 在任何注册的事件发生时返回。当我们完成时，我们注销事件。

理论上，我们可以从主循环中等待。反正它在等待的时候也没有什么更好的事情可做。但唤醒可能来自任何地方，不仅仅是直接的I/O事件。¹³而且我们需要处理所有这些。所以那行不通。

因此，我们将把这个任务推给一个次要的 `I/O` 驱动线程，它将我们的 `epoll` 事件转换为唤醒。我们已经知道如何处理这个了！¹⁴

```rust
use nix::sys::epoll::{Epoll, EpollCreateFlags, EpollEvent};
use std::{collections::BTreeMap, sync::LazyLock};

static EPOLL: LazyLock<Epoll> =
    LazyLock::new(|| Epoll::new(EpollCreateFlags::EPOLL_CLOEXEC).unwrap());
static REGISTERED_WAKERS: Mutex<BTreeMap<u64, Arc<dyn Wake>>> = Mutex::new(BTreeMap::new());

fn io_driver() {
    let mut events = [EpollEvent::empty(); 16];
    loop {
        let event_count = EPOLL.wait(&mut events, 1000u16).unwrap();
        let wakers = REGISTERED_WAKERS.lock().unwrap();
        for event in &events[..event_count] {
            let waker_id = event.data();
            if let Some(waker) = wakers.get(&waker_id) {
                waker.clone().wake();
            } else {
                // This could also be an "innocent" race condition,
                // if the event is delivered just as we're deregistering a waker.
                println!("=> (Waker {waker_id} not found, bug?)")
            }
        }
    }
}
```

然后，我们需要某种方式来注册对"文件"的兴趣（并在不再需要时注销它）。这只是确保我们的 `io_driver` 能看到它：

```rust
use nix::sys::epoll::EpollFlags;
use std::{ops::RangeFrom, os::fd::AsFd};

// We need some unique ID for each reason to be awoken..
// In reality you'd probably want some way to reuse these.
static NEXT_WAKER_ID: Mutex<RangeFrom<u64>> = Mutex::new(0..);

struct Interest<T: AsFd> {
    // Interest needs to own the file (or borrow it),
    // to make sure that the file stays alive for as long as our interest does.
    fd: T,
    registered_waker_id: Option<u64>,
}

impl<T: AsFd> Interest<T> {
    fn new(fd: T) -> Self {
        Interest {
            fd,
            registered_waker_id: None,
        }
    }

    fn register(&mut self, mut flags: EpollFlags, context: &mut Context) {
        let is_new = self.registered_waker_id.is_none();
        let id = *self
            .registered_waker_id
            .get_or_insert_with(|| NEXT_WAKER_ID.lock().unwrap().next().unwrap());
        REGISTERED_WAKERS
            .lock()
            .unwrap()
            .insert(id, context.waker.clone());
        // It's enough to get awoken once - if the Future is still interested then it should call `register`
        // to renew its interest.
        flags |= EpollFlags::EPOLLONESHOT;
        let mut event = EpollEvent::new(flags, id);
        if is_new {
            EPOLL.add(&self.fd, event).unwrap()
        } else {
            EPOLL.modify(&self.fd, &mut event).unwrap()
        }
    }
}

impl<T: AsFd> Drop for Interest<T> {
    fn drop(&mut self) {
        if let Some(id) = self.registered_waker_id {
            // what if we have multiple interests open on the same fd? (read+write? multiple reads?)
            EPOLL.delete(&self.fd).unwrap();
            REGISTERED_WAKERS.lock().unwrap().remove(&id).unwrap();
        }
    }
}
```

最后，我们可以将这一切组合到我们的 TcpRead 中...我们需要稍微改变它以保持 Interest 的状态，但是..它应该仍然足够容易识别：

```rust
use std::{io::Read, net::TcpStream};

struct TcpRead<'a> {
    socket: Interest<&'a mut TcpStream>,
    buffer: &'a mut [u8],
}
impl SleepyFuture<usize> for TcpRead<'_> {
    fn poll(&mut self, context: &mut Context) -> Poll<usize> {
        match self.socket.fd.read(self.buffer) {
            Err(err) if err.kind() == std::io::ErrorKind::WouldBlock => {
                // EPOLLIN is the event for when we're allowed to read
                // from the "file".
                self.socket.register(EpollFlags::EPOLLIN, context);
                Poll::Pending
            }
            size => Poll::Ready(size.unwrap()),
        }
    }
}
```

最后，我们可以把所有部分重新组合起来，并对我们的老朋友，行李代码服务器进行测试：

```rust
std::thread::spawn(io_driver);

let luggage_code_server_address = "127.0.0.1:9191";
let mut socket = TcpStream::connect(luggage_code_server_address).unwrap();
socket.set_nonblocking(true).unwrap();
let mut buffer = [0; 16];
let received = run_sleepy_future(
    // Let's limit the number of poll()s to make sure we're not cheating!
    // This is just for demonstration; remember that the runtime is /allowed/
    // to poll as often as it wants to.
    with_timeout(
        TcpRead {
            socket: Interest::new(&mut socket),
            buffer: &mut buffer,
        },
        // One initial poll to establish interest, then one poll once the data is ready for us.
        2,
    ),
).unwrap();
println!("=> The luggage code is {:?}", &buffer[..received]);
```

=> The luggage code is [1, 2, 3, 4, 5]

成功！我们回到了起点...但至少我们的计算机¹⁵可以稍微轻松一点。

# 另一个总结...暂时如此

希望现在你对那个奇怪的 Context东西有了更多的了解。

但我们仍然没有完全回到真正的 Future trait¹⁶。所以...下一篇将会是关于这个：解释我们需要理解的剩余概念，以便能够理解Future。¹⁷

1. 毕竟，这是一个系列的原因。  
2. 不要与quakers或shakers混淆。  
3. 实际上，Context包含一个Waker。它与我们的Arc<dyn Wake>非常接近，但如果你能维护自己的vtable，则不需要使用Arc。如果"vtable"这个词对你来说没有意义，只需实现Wake并完成它。如果"vtable"这个词对你有意义...你可能仍然应该实现Wake并完成它。  
4. 如果我们引用它，那就不是自我抄袭！哦，我想它也满足类型契约...  
5. 否则，我们永远无法修改Mutex保护的值！  
6. 如果这看起来像我们只是在重新实现Condvar...我们确实在这样做。不同的是Condvar::wait没有规定在notify_one在wait之前被调用时立即返回。这对我们来说是个问题；它会悄悄地阻止Future在poll期间唤醒自己。  
7. 我听说将动词名词化现在很流行。我们能把一个名词化的动词再动词化吗？  
8. 跨平台支持留给读者作为练习。享受吧！  
9. 不是唯一的API，还有其他的。但它是那个达到"标准"权衡的API，介于不太慢和不太实验性之间。也许几年后，当你写"natkr所有的错误"这篇文章时，io_uring将无处不在。当你这样做时，请把它发给我，我很期待阅读它！  
10. 在Unix意义上，"一切"都是"文件"，包括网络套接字。  
11. 对于在家跟随的人，我将使用nix v0.29.0，因为这是我写这篇文章时的最新版本。  
12. 如果我们能具体链接到事件标志列表，那就太好了，但可惜...  
13. 定时器、后台线程等等...  
14. 有时这被称为 reactor。  
15. 以及电费单。  
16. 如果它能随时站起来，那就太有帮助了。  
17. 剧透：这意味着 [Pin](https://doc.rust-lang.org/stable/std/pin/struct.Pin.html) 和关联类型。  