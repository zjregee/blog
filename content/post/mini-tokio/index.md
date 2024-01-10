---
title: Rust 异步运行时实现示例
slug: mini-tokio
date: 2023-11-03 00:00:00+0000
categories:
    - mini 系列
tags:
    - Rust
---

> 本文是 mini 系列的第五篇文章，mini 系列是一些知名框架或技术的最小化实现，本系列文章仅作为学习积累的目的，代码或架构在很大程度上参考了现有的一些源码或博客。本文根据 Tokio 官方教程实现了一个 Rust 异步运行时示例以及一个实现 Future trait 可供运行时调度的值类型。
> 
> Tokio 项目地址：https://github.com/tokio-rs/tokio
> 
> mini-tokio 代码实现地址：https://github.com/zjregee/mini/mini-tokio
## 一、Futures

在 Rust 中直接调用异步函数返回的值是 future，即实现标准库所提供的 std::future::Future trait 的值，该 trait 定义如下：

```Rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context)
        -> Poll<Self::Output>;
}
```

关联类型 Output 是 future 完成后产生的类型，Pin 类型是 Rust 在异步函数中支持借用的方式。与其他语言的 future 实现方式不同，Rust 中的 future 并不表示在后台发生的计算，而是 future 本身就是计算。future 的所有者需要负责轮询 future 来推动计算，推进计算的过程是通过调用 Future::poll 来实现的。
### 1.1 implementing future

```Rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };
    let out = future.await;
    assert_eq!(out, "done");
}
```
### 1.2 async fn as a Future

```Rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

enum MainFuture {
    State0,
    State1(Delay),
    Terminated,
}

impl Future for MainFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<()>
    {
        use MainFuture::*;
        loop {
            match *self {
                State0 => {
                    let when = Instant::now() +
                        Duration::from_millis(10);
                    let future = Delay { when };
                    *self = State1(future);
                }
                State1(ref mut my_future) => {
                    match Pin::new(my_future).poll(cx) {
                        Poll::Ready(out) => {
                            assert_eq!(out, "done");
                            *self = Terminated;
                            return Poll::Ready(());
                        }
                        Poll::Pending => {
                            return Poll::Pending;
                        }
                    }
                }
                Terminated => {
                    panic!("future polled after completion")
                }
            }
        }
    }
}
```

Rust 中的 futures 是状态机。在这里，MainFuture 被表示为 future 可能状态的枚举。future 从 State 0 状态开始，当调用 poll 时，future 试图尽可能地推进其内部状态。如果 future 可以被完成，则返回 Poll::Ready，其中包含异步计算的输出。如果 future 不能完成，通常是由于它正在等待的资源没有准备好，那么返回 Poll::Pending，接收到 Poll::Pending 向调用方表明 future 将在稍后的时间完成，调用方应该稍后再次调用 poll。futures 是由其他 futures 组成的，在外部的 futures 上调用 poll 会导致调用内部 futures 的poll 函数。
## 二、Executors

异步函数返回 futures，futures 一定需要被调用 poll 来推进它们的状态。futures 是由 futures 来组成的。那么又是由谁来调用最外层 future 的 poll 呢？要运行异步函数，它们要么传递给 tokio::spawn，要么是带有 #[tokio::main] 注释的 main 函数。这导致将生成的外部 future 提交给 Tokio 执行器。执行器负责对外部的 future 调用 Future::poll，推动异步计算完成。
### 2.1 mini-tokio

```Rust
use std::collections::VecDeque;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use futures::task;

fn main() {
    let mut mini_tokio = MiniTokio::new();
    mini_tokio.spawn(async {
        let when = Instant::now() + Duration::from_millis(10);
        let future = Delay { when };
        let out = future.await;
        assert_eq!(out, "done");
    });
    mini_tokio.run();
}

struct MiniTokio {
    tasks: VecDeque<Task>,
}

type Task = Pin<Box<dyn Future<Output = ()> + Send>>;

impl MiniTokio {
    fn new() -> MiniTokio {
        MiniTokio {
            tasks: VecDeque::new(),
        }
    }

    fn spawn<F>(&mut self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        self.tasks.push_back(Box::pin(future));
    }
    
    fn run(&mut self) {
        let waker = task::noop_waker();
        let mut cx = Context::from_waker(&waker);
        
        while let Some(mut task) = self.tasks.pop_front() {
            if task.as_mut().poll(&mut cx).is_pending() {
                self.tasks.push_back(task);
            }
        }
    }
}
```

这将导致运行异步块。这个执行器有个显著的缺陷，这个执行器不断地循环所有的 futures 并调用 poll。大多数时候，futures 不会准备好执行更多的工作，并且会再次返回 Poll::Pending。这个过程会消耗 CPU 周期，通常效率不高。理想情况下，我们希望 mini-tokio 仅在 futures 能够取得进展的时候调用 poll。当任务被阻塞的资源准备好执行所请求的操作时，就会发生这种情况。如果任务想要从 TCP 套接字读取数据，那么只希望在 TCP 套接字接收到数据时轮询任务。
## 三、Wakers

通过 wakers，资源能够通知等待任务该资源已准备好继续某些操作。

```Rust
fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
```

poll 的 Context 参数有一个 waker() 方法，此方法返回绑定到当前任务的 waker。waker 有一个 wake() 方法，调用此方法向执行器发出信号，表明应该安排执行关联的任务。资源在转换到就绪状态时调用 wake()，以通知执行器轮询任务将能够取得进展。
### 3.1 updating delay

```Rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use std::thread;

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            let waker = cx.waker().clone();
            let when = self.when;
            thread::spawn(move || {
                let now = Instant::now();
                if now < when {
                    thread::sleep(when - now);
                }
                waker.wake();
            });
            Poll::Pending
        }
    }
}
```

当 future 返回 Poll::Pending 时，它必须确保在某个时刻向 waker 发出信号，忘记这样做会导致任务无限期搁置。当返回 Poll::Pending 后忘记唤醒任务是 bug 的常见来源。

```Rust
impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}
```

在之前的 Delay 实现中，在返回 Poll::Pending 之前，我们调用了 cx.wake().wake_by_ref()。这是因为我们还没有实现计数器线程，所以我们内联地给 waker 发送信号，这样将导致 future 立即重新调度，再次执行，并且可能无法完成。这样的实现会导致一个忙循环，除了浪费一些 CPU 周期，也并没有什么问题。
### 3.2 update mini-tokio

更新后的 mini-tokio 使用一个 channel 来存储被调度的任务。channel 允许任务排队并从任何线程执行。wakers 必须是 Send 和 Sync 的，所以在这里使用 crossbeam crate，因为标准库中的 channel 不是 Sync 的。可以发送到其他线程的类型是 Send，大多数类型都是 Send，但像 Rc 这样的类型不是。可以通过不可变引用并发访问的类型是 Sync。类型可以是 Send，但不是 Sync的。例如 Cell 它可以通过不可变引用进行修改，因此并发访问是不安全的。

```Rust
use crossbeam::channel;
use std::sync::{Arc, Mutex};

struct MiniTokio {
    scheduled: channel::Receiver<Arc<Task>>,
    sender: channel::Sender<Arc<Task>>,
}

struct Task {
    future: Mutex<Pin<Box<dyn Future<Output = ()> + Send>>>,
    executor: channel::Sender<Arc<Task>>,
}

impl Task {
    fn schedule(self: &Arc<Self>) {
        self.executor.send(self.clone());
    }
}
```

wakers 是 Sync 的，并且可以被 clone。当 wake 被调用时，任务必须被调度执行。在这里通过 channel 实现这一点。当 wake() 在 waker 上被调用时，任务被推送到 channel 的发送端口。Task 结构将实现 wake 逻辑。现在需要将 schedule 函数与 std::task::Waker 挂钩。标准库提供了一个 low-level 的 API，通过手动构造的虚函数表可以实现这一点。这个实现方式为实现者提供了最大的灵活性，但需要大量 unsafe 的样板代码。在这里不直接使用 RawWakerVTable，而是使用 futures crate 提供的 ArcWake 实用程序。这个 crate 允许通过实现一个简单的 trait，将 Task 结构体公开为一个 waker。

```Rust
use futures::task::{self, ArcWake};
use std::sync::Arc;
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        arc_self.schedule();
    }
}
```

当之前的计时器线程调用 waker.wake() 时，任务被推送到 channel 中。

```Rust
impl MiniTokio {
    fn run(&self) {
        while let Ok(task) = self.scheduled.recv() {
            task.poll();
        }
    }

    fn new() -> MiniTokio {
        let (sender, scheduled) = channel::unbounded();
        MiniTokio { scheduled, sender }
    }

    fn spawn<F>(&self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        Task::spawn(future, &self.sender);
    }
}

impl Task {
    fn poll(self: Arc<Self>) {
        let waker = task::waker(self.clone());
        let mut cx = Context::from_waker(&waker);
        let mut future = self.future.try_lock().unwrap();
        let _ = future.as_mut().poll(&mut cx);
    }

    fn spawn<F>(future: F, sender: &channel::Sender<Arc<Task>>)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let task = Arc::new(Task {
            future: Mutex::new(Box::pin(future)),
            executor: sender.clone(),
        });
        let _ = sender.send(task);
    }

}
```

Task:poll() 函数使用 futures crate 中的 ArcWake 实用程序创建 waker，这个 waker 用于创建 task::Context，该 task::Context 被传递给 poll。
## 四、总结

Rust 的 async/await 特性是由 traits 支持的，这允许第三方 crate 提供执行细节：异步 Rust 操作是惰性的，需要调用者去轮询它们；wakers 被传递给 futures，将 future 与调用它的任务联系起来；当资源尚未准备好完成操作时，将返回 Poll::Pending 并记录任务的 waker；当资源准备就绪时，通知任务的 waker；执行器接收通知并安排任务的执行；再次轮询任务，这一次资源准备就绪，任务取得进展。
### 4.1 一些尚未解决的问题

Rust 的异步模型允许单个 future 在执行时跨任务迁移。考虑以下代码：

```Rust
use futures::future::poll_fn;
use std::future::Future;
use std::pin::Pin;

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let mut delay = Some(Delay { when });
    poll_fn(move |cx| {
        let mut delay = delay.take().unwrap();
        let res = Pin::new(&mut delay).poll(cx);
        assert!(res.is_pending());
        tokio::spawn(async move {
            delay.await;
        });
        Poll::Ready(())
    }).await;
}
```

poll_fn 函数使用闭包创建 Future 实例。在这个例子中，Delay::poll 被不同的 waker 实例调用了不止一次。当发生这种情况时，我们必须确保在传递给最近的 poll 调用的 waker 实例上调用 wake。在实现 future 时，重要的是要假设每个对 poll 的调用都可以提供一个不同的 waker 实例，poll 函数必须用新的 waker 替代之前记录的 waker。

之前的 Delay 实现在每次轮询时都会生成一个新线程。如果轮询过于频繁（例如 select! 这个 future 和一些其他的 future，当其中一个发生事件时，都要进行轮询），效率会非常低。一种方法是记住是否已经生成了一个新线程，只有在还没有生成一个线程的情况下才生成一个新线程。但是，如果这样做，必须确保线程的 waker 在以后的轮询调用中更新，否则不会唤醒最近的 waker。

```Rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
    waker: Option<Arc<Mutex<Waker>>>,
}

impl Future for Delay {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if let Some(waker) = &self.waker {
            let mut waker = waker.lock().unwrap();
            if !waker.will_wake(cx.waker()) {
                *waker = cx.waker().clone();
            }
        } else {
            let when = self.when;
            let waker = Arc::new(Mutex::new(cx.waker().clone()));
            self.waker = Some(waker.clone());
            thread::spawn(move || {
                let now = Instant::now();
                if now < when {
                    thread::sleep(when - now);
                }
                let waker = waker.lock().unwrap();
                waker.wake_by_ref();
            });
        }
        if Instant::now() >= self.when {
            Poll::Ready(())
        } else {
            Poll::Pending
        }
    }
}
```

这里有点复杂，核心思想是每次调用 poll 时，future 检查提供的 waker 是否与先前记录的 waker 匹配。如果两个 waker 不匹配，需要更新记录的 waker。
### 4.2 notify 实用工具

wakers 是异步 Rust 工作的基础，通常没有必要降低到这个层次编写代码。在 Delay 的例子中，通过使用 tokio::sync::Notify 实用程序完全使用 async/await 来实现它。这个实用程序提供了一个基本的任务通知机制，它处理 waker 的细节，包括确保记录的 waker 与当前任务匹配。

```Rust
use tokio::sync::Notify;
use std::sync::Arc;
use std::time::{Duration, Instant};
use std::thread;

async fn delay(dur: Duration) {
    let when = Instant::now() + dur;
    let notify = Arc::new(Notify::new());
    let notify_clone = notify.clone();
    thread::spawn(move || {
        let now = Instant::now();
        if now < when {
            thread::sleep(when - now);
        }
        notify_clone.notify_one();
    });
    notify.notified().await;
}
```
### 4.3 定义 trait 中的异步方法

在 Rust 中，异步方法通过 async 关键字来定义，然而，trait 不能直接定义异步方法。异步方法的处理需要在运行时处理 Future，这种处理依赖于特定的异步运行时环境，这导致了异步方法的实现涉及到更多的运行时支持，而 trait 的设计初衷在于提供一种在编译时进行静态分发的方法。

为了在 trait 中支持异步方法，需要引入更复杂的机制，目前常见的使用方法是通过 async-trait crate 来实现定义 trait 中的异步方法。总的来说，Rust 的设计哲学是尽可能提供简单、静态的机制来支持各种编程范式，而异步编程则引入了一些动态的概念。这两者之间的差异导致了异步方法不直接支持 trait 的原因。

> 值得一提的是，最新的 Rust 版本在近期已经提供了 async trait 的支持。
> 
> https://blog.rust-lang.org/inside-rust/2023/05/03/stabilizing-async-fn-in-trait.html
## 参考资料

* https://tokio.rs/tokio/tutorial/async