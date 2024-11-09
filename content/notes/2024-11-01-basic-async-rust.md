+++
title = "Basic Async Rust"
date = 2024-11-01
[taxonomies]
  tags = ["rust", "futures"]
+++

## Futures

One of the key features of async programming is the concept of a future. We mentioned above that a future is a
placeholder object that represents the result of an anynchronous operation that has not yet completed. Futures allow
you to start a task and continue with other operations while the task is being executed in the background.

To truly understand how a future works, we should cover the lifecycle of a future.
When a future is created, the future is idle. It has yet to be executed. Once the future is executed, it can either
yield a value, resolve, or go to sleep because the future is pending(awaiting on a result). When the future is polled
again the poll can either return a pending or reday result. The future will continue to be polled until it is either
resolved or cancelled.

To concrete how futures work, let us build a basic counter future. The counter future will merely up to 5 and then ready
once the counter has reached 5. First we need to import the following:

```rs
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

use tokio::task::JoinHandle;
```

We will cover context and Pin after building our basic future. Seeing as our future is a counter, the
struct takes the following from:

```rs
struct CounterFuture {
    count: u32,
}
```

We then implement the Future trait with the following code:

```rs
impl Future for CounterFuture {
    type Output = u32;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        self.count += 1;
        println!("polling with result: {}", self.count);
        std::thread::sleep(Duration::from_secs(1));
        if self.count < 5 {
            cx.waker().wake_by_ref();
            Poll::Pending
        } else {
            Poll::Ready(self.count)
        }
    }
}
```

Everytime the future is polled, the count is increased by one. If the count is at five we then
state that the future is ready. We introduce the `std::thread::sleep` function to merely exaggrate
the time taken so it is easier to following this example when running this code. To run the code, we simply
need the code below:

```rs
#[tokio::main]
async fn main() {
    let counter_one = CounterFuture { count: 0 };
    let counter_two = CounterFuture { count: 0 };
    let handle_one: JoinHandle<u32> = tokio::task::spawn(async move {
        counter_one.await;
    });

    let handle_two: JoinHandle<u32> = tokio::task::spawn(async move {
        counter_two.await;
    });
    tokio::join!(handle_one, handle_two);
}
```

Running two of our futures in different tasks gives us the following printout:

```
polling with result: 1
polling with result: 1
polling with result: 2
polling with result: 2
polling with result: 3
polling with result: 3
polling with result: 4
polling with result: 4
polling with result: 5
polling with result: 5
```

We can see that one of the futures was taken off the queue, polled, and then set to idle whist another
future was taken off the task queue to be polled. These futures were polled in alternate fashion, You may have
noticed that our `poll` function is not async. This is because an async poll function would return a circular dependency
as you would be sending a future to be polled in order to resolve a future being polled. With this, we can see that the
future is the bedrock of the async computation.

We see that `poll` function takes a mutable reference of itself, however, this mutable reference is wrapped in a Pin which
we need to disscuss.

## Pin in Futures

In Rust, the compiler often moves values around in memory. For instance, if we move a variable into a function, the
memory may be moved.

It's not just moving values that may rusult in moving of memory address. Collections can also change memory address.
For instance, if a vector gets to capacity, the vector will have to be reallocated in memory changing the memory address.

Most normal primitives such as number, string, bools, structs, enum etc implement the `Unpin` trait enabling them to be
moved around. If you are unsure if your data implements the `Unpin` trait, run a doc command and check the traits your data
type implements. For example, below is the auto-trait implementations on `i32` in the standard docs:

```rs
impl RefUnwindSafe for i32
impl Send for i32
impl Sync for i32
impl Unpin for i32
impl UnwindSafe for i32
```

So why do we concern ourselves with [pin](https://rust-lang.github.io/rfcs/2349-pin.html) and [unpinning](https://blog.cloudflare.com/pin-and-unpin-in-rust/)?
We know that futures get moved as we use the async move in our code when spawning a task. However, moving can be dangerous.
To demonstrate the data, we can build a basic struct that **references itself** with the following code:

```rs
use std::ptr;

struct SelfReferential {
    data: String,
    self_pointer: *const String,
}
```

The `*const String` is a raw pointer to a string. This means that the pointer directly references the memory address where
the data is. The pointer offers no safety guarantees. This means that the reference does not update if the data being
pointed to moves. We are using a raw pointer to demonstrate why pinning is needed. For this demonstration to take place,
we need to define the constructor of the struct, and printing of the structs reference using the code below:

```rs
impl SelfReferential {
    fn new(data: String) -> SelfReferential {
        let mut sr = SelfReferential {
            data,
            self_pointer: ptr::null(),
        };
        sr.self_pointer = &sr.data as *const String;
        sr
    }
    fn print(&self) {
        unsafe {
            println!("{}", *self.self_pointer);
        }
    }
}
```

To then expose the danger of moving the struct by creating two instances of the `SelfReferential` struct,
swap these instances in memory, and then print what data the raw pointer is pointing to with the following code:

```rs
fn main() {
    let mut first = SelfReferential::new("first".to_string());
    let mut second = SelfReferential::new("second".to_string());
    unsafe {
        ptr::swap(&mut first, &mut second);
    }
    first.print();
}
```

If you try and run the code, you will get an error, which is highly probable to be a segmentation fault.
The segmentation fault is an error caused by accessing memory that does not belong to the program. We can see that moving
struct with references to itself can be dangerous. Pinning essentially ensures that the future remains at a fixed memory address.
This is import because futures can be paused or resumed which can change the memory address.

We have nearly covered all of the components in the basic future that we have defined. The only component is the context.

## Context in Futures

A `Context` only serves to provide access to a waker to wake a task. A `Waker` is a handle that notifies the executor when the task
is ready to be run.

While this is the primary role of `Context` today, it's important to note that this functionality might evolve in the future.
The design of `Context` has allowed space for expansion such as the introduction of additional responsibilities or capabilities as Rust's
asynchronous ecosystem grows.

Lets look at a stripped down version of our poll function so we can focus on the path of waking up the future:

```rs
fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
    ...
        if self.count < 5 {
            cx.waker().wake_by_ref();
            Poll::Pending
        } else {
            Poll::Ready(self.count)
        }
}
```

We can see that the waker is wrapped in the context, and is only utilised when the result of the poll is going to be pending.
The waker is essentially waking up the future so it can be executed. If the future is completed then there is no need for any more
execution to be done. If we were to remove the waker and run our program again, we would get the following printout:

```
polling with result: 1
polling with result: 1
```

We can seee that our program would not have completed and the program hangs. This is because our tasks are still
idle but there is no way to wake them up again to be polled and executed to completion. Futures need the `Waker::wake()`
function so the wake function can be called when the future should be polled again. The process takes the following steps:

1. The poll function for a future is called and the result is that the future needs to wait for some async operation to
completed before the future is able to return a value.
2. The future then regisers its interest of being notified of the completion of the operation by calling a method that references the waker.
3. the executor then taks note of the interest in the future's operation and stores the waker in a queue.
4. At some later time the operation completes and the executor is notified. The executor retrieves the wakers from the queue and
calls the `wake_by_ref` on each one waking up the futures.
5. The `wake_by_ref` signals the associated task that should be scheduled for execution. The way in which this is done
can vary depending on the runtime.
6. When the future is executed, the executor will call the poll method of the future again and the future will determine
whether the operation has completed returning a value if completion is achieved.

We can see that Futures are used with an async/await function but let's have a think about how else they can be used.
We can also use a timeout on a thread of execution. This means that the thread finishes when so much time has elapsed
meaning that we do not end up in a situation where the program hangs indefinitely. This is useful when we have a function
that can be slow to complete and we want to move or error early. Remember that threads provide the underlying functionality
for executing tasks. We import timeout from `tokio::time` and set up a slow task. In this case, we put this as a sleep for
10 seconds to exaggerate the effect.

```rs
use std::time::Duration;
use tokio::time::timeout;

async fn slow_task() -> &'static str {
    tokio::time::sleep(Duration::from_secs(10)).await;
    "Slow Task Completed"
}
```

Now we set up our timeout, in this case, setting it to 3 seconds. This means that the thread will end if the Future
is not compelted within these 3 seconds. We match the result and print "Task timed out".

```rs
#[tokio::main]
async fn main() {
    let duration = Duration::from_secs(3);
    let result = timeout(duration, slow_task()).await;

    match result {
        Ok(value) => println!("Task completed successfully: {}", value),
        Err(_) => println!("Task timed out"),
    }
}
```

Note
When we apply a timeout to a future, as shown in the example, there's a possibility that the future(in this case, `slow_task()`)
will be cancelled if it doesn't complete within the specified duration. This introduces the concept of "cancel safety".

Cancel safety refers to the fact that when a future is canceled, it's essential to ensure that any resources or state it
was using are handled correctly. This means that if a task is in the middle of an operation when it's canceled, it shouldn't
leave the system in a bad state, like holding onto locks, leaving files open, or partially modifying data.

In Rust's async ecosystem, most operations are cancel-safe by default, which means they can be safely interrupted without
causing issues. However, it's still a good practice to be aware of how your tasks interact with external resources or state
and ensure that thoese interactions are cancel-safe.

In our example, if the `slow_task()` is canceled due to the timeout, the task itself simply stopped, and the timeout returns
an error indicating the task didn't compelte in time. Since `tokio::time::sleep` is a cancel-safe operation, there is no risk
of the resource leaks or inconsistent states. However, if the task involved more complex operations, such as network communication
or file I/O, additional care might be needed to ensure that the cancellation is handled appropriately.

For CPU-intensive work, we can also off-load work to a separate threadpool and the future resolves when the work is finished.
We have now covered the context of futures.

Polling directly is not the most efficient way as our executor will be busy polling futures that are not ready.
To explainhow we can prevent busy polling, we will move onto waking futures remotely.

## Waking Futures Remotely

Imagine that we make a network call to another computer using async Rust. The routing of the network call and reveiving
of the response happens outside of our Rust program. Considering this, it does not make sense to constantly poll our
networking future until we get a signal from the OS that data has been received at the port we are listening to.
We can hold on the polling of the future by externally referencing the future's waker, and waking the future when we need to.

To see this in action, we can simulate an external call with channels. First, we need the following imports:

```rs
use std::pin::Pin;
use std::task::{Context, Poll, Waker};
use std::sync::{Arc, Mutex};
use std::future::Future;
use tokio::sync::mpsc;
use tokio::task;
```
With these imports, we can now define our future which takes the form below:

```rs
struct MyFuture {
    state: Arc<Mutex<MyFutureState>>,
}

struct MyFutureState {
    data: Option<Vec<u8>>,
    waker: Option<Waker>,
}
```

Here, we can see that the state of our `MyFuture` can be accessed from another thread. The state of our `MyFuture` has
the waker and data. To make our `main` function more concise, we define a constructor for `MyFuture` with the following code:

```rs
impl MyFuture {
    fn new() -> (Self, Arc<Mutex<MyFutureState>>) {
        let state = Arc::new(Mutex::new(MyFutureState {
            data: None,
            waker: None,
        }));
        (
            MyFuture {
                state: state.clone(),
            },
            state,
        )
    }
}
```

For our constructor, we can see that we construct the future, but we also return a reference to the state so we can access
the waker outside of the future. Finally we implement the `Future` trait for our future with the code below:

```rs
impl Future for MyFuture {
    type Output = String;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        println!("Polling the future");
        let mut state = self.state.lock().unwrap();

        if state.data.is_some() {
            let data = state.data.take().unwrap();
            Poll::Ready(String::from_utf8(data).unwrap())
        } else {
            state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

Here, we can see that we print every time we poll the future so we can keep track of how many times wo poll our future.
We then access the state and see there is any data. If there is no data, we pass the waker into the state so we can wake
the future from outside of the future. If there is data in the state, we know that we are ready, and we return a `Ready`.

Our future is now ready to test. Inside our `main` function, we create our future, channel for communication, and spawn our
future with the following code:

```rs
let (my_future, state) = MyFuture::new();
let (tx, mut rx) = mpsc::channel::<()>(1);
let task_handle = task::spawn(async move {
    my_future.await
});

tokio::time::sleep(tokio::time::Duration::from_secs(3)).await;
println!("spawning trigger task");
```

We can see that we are sleeping for three seconds. This sleep gives us time to check if we are polling multiple times.
If our approach works as intended, we should only get one poll during the time of sleeping. We then spawn our trigger
task with the code below:

```rs
let trigger_task = task::spawn(async move {
   rx.recv().await;
  let mut state = state.lock().unwrap();
  state.data = Some(b"Hello from the outside".to_vec());

  loop {
      if let Some(waker) = state.waker.take() {
          waker.wake();
          break;
      }
  }
});
tx.send(()).await.unwrap();
```

We can see that once our trigger task receives the message in the channel, it gets the state of our future, and populate the data.
We then check to see if the waker is present. Once we get hold of the waker, we wake the future.

Finally, we await on both of the async tasks with the following code:

```rs
let outcome = task_handle.await.unwrap();
println!("Task completed with outcome: {}", outcome);
trigger_task.await.unwrap();
```

If we run our code, we get the printout below:

```
Polling the future
spawning trigger task
Polling the future
Task completed with outcome: Hello from the outside
```

We can see that our polling pnly happens once on the initial setup, and then happens one more time when we wake the future
with the data. Async runtimes setup efficient ways to listen to OS events so they do not have to blindly poll futures.
For instance, Tokio has an event loop that listens to OS events, and then handles them so the event wakes up the right task.

## Sharing data between Futures

We can share data between futures. We may want to share data between futures for the following reasons:

- Aggregating Results
- Dependent Computations
- Caching Results
- Synchronization
- Shared State
- Task Coordination and supervision
- Resource Management
- Error Propagation

While sharing data between futures is useful, there are some things that we need to be mindful of when doing so.
We can highlight them as we work through a simple example.
First, we will be relying on the standard Mutex with the following import:

```rs
use std::sync::{Arc, Mutex};
use tokio::task::JoinHandle;
use core::task::Poll;
use tokio::time::Duration;
use std::task::Context;
use std::pin::Pin;
use std::future::Future;
```

For our example, we will be using a basic struct that has a counter.
One async task will be for increasing the count, and the other task will be decreasing the count.
If both tasks hit the shared data the same number of times, the end result will be zero. Therefore,
we need to build a basic enum to define what type of task is being run with the code below:

```rs
#[derive(Debug)]
enum CounterType {
    Increment,
    Decrement,
}
```

We can then define our shared data struct with the following code:

```rs
struct SharedData {
    counter: i32,
}

impl SharedData {
    fn increment(&mut self) {
        self.counter += 1;
    }
    fn decrement(&mut self) {
        self.counter -= 1;
    }
}
```

Now that our shared data struct is defined, we can define our counter future with the code below:

```rs
struct CounterFuture {
    counter_type: CounterType,
    data_reference: Arc<Mutex<SharedData>>,
    count: u32
}
```

Here, we have defined the type of operation the future will perform on the shared data.
We also have access to the shared data and a count to stop the future once the total number of executions of the
shared data has happened for the future.

The signature of our poll function takes the following form:

```rs
impl Future for CounterFuture {
    type Output = u32,

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        ...
    }
}
```

Inside our poll function we first cover getting access to the shared data with the following code:

```rs
std::thread::sleep(Duration::from_secs(1));
let mut guard = match self.data_reference.try_lock() {
    Ok(guard) => guard,
    Err(error) => {
        println!("error for {:?}: {}", self.counter_type, error);
        cx.waker().wake_by_ref();
        return Poll::Pending
    }
}
```
