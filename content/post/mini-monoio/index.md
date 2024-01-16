---
title: 基于 thread-per-core 模型的 Rust 异步运行时 mini-monoio 实现示例
slug: mini-monoio
date: 2023-11-14 00:00:00+0000
categories:
    - mini 系列
tags:
    - Rust
---

> 本文是 mini 系列的第六篇文章，mini 系列是一些知名框架或技术的最小化实现，本系列文章仅作为学习积累的目的，代码或架构在很大程度上参考了现有的一些源码或博客。Monoio 是字节跳动开源的不同于 Tokio 设计的 Rust 异步运行时实现，本文根据其官方教程介绍基于 thread-per-core 模型的异步运行时示例。
> 
> Monoio 项目地址：https://github.com/bytedance/monoio
> 
> mini-monoio 代码实现地址：https://github.com/zjregee/mini/mini-monoio
## 一、Rust 异步机制

Rust 中的异步机制通过 Async + Await 语法糖，在 HIR 阶段被展开为 Generator 语法，然后 Generator 又会在 MIR 阶段展开成状态机，生成结构最终实现 Future trait，从而可供异步运行时调度。Rust 通过 Future trait 描述状态机对外暴露的接口，异步任务的本质就是实现 Future trait 的状态机，程序可以利用 poll 方法推动状态机执行，poll 方法会告诉程序现在遇到阻塞，或是任务执行完毕返回结果。

> 程序由计算逻辑和 IO 组成，异步运行时的本质在于如何高效地和操作系统交互，协调表达计算逻辑和 IO 之间的关系。

```Rust
async fn do_http() -> i32 {
    1
}

fn do_http() -> DOHTTPFuture { DoHTTPFuture }

struct DoHTTPFuture;

impl Future for DoHTTPFuture {
    type Output = i32;

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        Poll::Ready(1)
    }
}
```

上面的代码示例是一种等价的写法，我们可以手动实现一些实现 Future trait 的值，异步函数也会由编译器自动生成返回一个实现 Future trait 的匿名结构。上面的例子较为简单，一旦涉及到 await，本质上异步任务就变成了一个状态机，每次 await 代表一次状态转换。在 await 调用时，线程不会在此等待，而是保存当前的状态以供下次恢复并执行其他任务。
### 1.1 异步 IO 支撑

为了实现异步运行时所期望的异步并发能力，即在等待 IO 完成的过程中能够处理其他任务，我们需要操作系统内核提供相应的能力。提供异步 IO 能力的常见方法包括两种：epoll 和 io_uring。epoll 是 Linux 中优秀的 IO 事件通知机制，可以同时监听多个 fd 的状态，实现非阻塞的 IO 操作。io_uring 是更为先进的异步 syscall 机制，其主要特点包括：零拷贝，减少数据在用户空间和内核空间之间的复制次数，提高性能；高性能事件通知，相比 epoll 提供了更高效的事件通知机制及多种事件类型支持；批处理操作，io_uring 支持批处理操作，允许应用程序将多个 IO 操作一次性提交给内核，从而减少系统调用的开销。

> 异步 IO 对于异步运行时的性能是非常重要的，在 Tokio 官网中也提到过在读取大量文件这一看起来适合异步运行时这一场景，由于操作系统通常不提供异步文件 API，Tokio 相比于普通线程池没有任何优势。此外，值得一体的是 epoll 并不能真正的提供异步 IO 能力。
### 1.2 Reactor 模式 vs Proactor 模式

在基于系统提供的异步 IO 能力之上，Reactor 模式和 Proactor 模式是两种常用的用于处理并发 IO 操作的设计模式，它们分别采用了不同的方式来处理事件和异步操作。Reactor 模式基于事件驱动，由一个中心化的事件分发器负责监听并分发事件。在 Reactor 模式中，事件分发器通常是同步的，在一个主循环中等待事件的发生，然后分发事件给相应的处理器。处理器负责处理 IO 事件，包括接受连接、读取数据、写入数据等，每个事件类型都有对应的处理器。Proactor 模式也是事件驱动的，但它采用异步的方式。在 Proactor 模式中，发起 IO 操作后，系统会异步地通知操作完成。Proactor 模式中有一个中心化的 IO 完成处理器负责等待异步 IO 操作完成的通知，并在操作完成后调用相应的回调函数。两者的根本区别在于 Reactor 模式强调同步事件分发，处理器直接负责底层 IO。Proactor 模式强调异步 IO 操作，IO 完成处理器负责等待通知并调用回调函数，处理器无需直接关注底层 IO。
### 1.3 异步运行时组件梳理

异步运行时大体上可以分为两个组件，Executor 模块和 Reactor 模块。Executor 模块包含执行器和任务队列，它的工作就是不断的推动任务运行，当所有的任务执行完毕必须等待时，把执行权交给 Reactor 模块。Reactor 模块负责与操作系统交互，管理注册的 IO，并等待 IO 就绪，还需要把 IO 所关联的任务唤醒重新加入 Executor 模块的任务队列中。由于 IO 要通过异步运行时管理，因此，在异步编程中不能直接使用 Rust 标准中的 IO 函数，而是需要通过使用异步运行时提供的 IO 函数来实现异步 IO。异步运行时提供的 IO 函数不仅会将 IO 通过 Reactor 模块注册，还会通过异步任务上下文所携带的 waker 来唤醒任务将异步任务重新执行。
## 二、Monoio 设计要点

Monoio 是字节跳动根据自身业务需求实现并开源的 Rust 异步运行时，旨在兼顾平台兼容性的情况下，实现高效的 thread-per-core Rust 异步运行时。本章内容结合 Monoio 提供的官方博客以及文档介绍 Monoio 的设计要点。
### 2.1 基于 GAT 的生命周期租借

不同于常见的基于 epoll 异步 IO 的 Rust 异步运行时，Monoio 基于 io_uring 实现 Rust 异步运行时。epoll 和 io_uring 的一个主要区别在于 epoll 是一种基于就绪状态的通知，而 io_uring 是一种基于完成的通知。基于就绪状态的通知模式，任务会通过 epoll 等待并感知 IO 就绪，并在就绪时再执行 syscall。使用 io_uring 可以减少 syscall 的使用，从而减少上下文切换次数。除此之外，io_uring 允许用户和内核共享两个无锁队列，其中一个无锁队列实现用户态程序写，内核态消费，另一个无锁队列实现内核态写，用户态消费，从而实现零拷贝减少内核中数据拷贝。

这两种模式的差异会很大程度上影响异步运行的设计和 IO 接口。在第一种模式下，等待时不需要持有 buffer，只有执行 syscall 的时候才需要 buffer，所以这种模式下可以允许用户在真正执行 syscall 的时候传入 &mut Buffer；而在第二种模式下，在将 IO 请求提交给内核后，内核可以在任何时候访问 buffer，必须确保该 IO 请求返回前 buffer 的有效性，这种情况下无法通过 Rust 编译器实现生命周期检查。为了解决这个问题，异步运行时需要捕获 buffer 的所有权，从而保证 buffer 的有效性。Monoio 通过 GAT 实现生命周期租借来保证 buffer 的有效性。

> GAT 是 Rust 编程语言中一个新特性，它允许在 trait 中定义关联类型时使用泛型参数。在引入 GAT 之前，trait 中的关联类型不能直接依赖于 trait 的泛型参数，这导致了一些限制，GAT 可以帮助编写更通用、灵活和类型安全的代码。
### 2.2 thread-per-core 模型

基于任务窃取策略的异步运行时可以较为充分地利用 CPU，应对通用场景可以做到较好的性能。同时跨线程任务调度会带来额外开销，且对 Task 本身有 Send 和 Sync 约束，导致无法很好地使用 thread local storage。实际应用中很多场景并不需要跨线程调度，如 nginx 负载均衡代理可以通过 thread-per-core 模型实现，这样不仅可以减少跨线程通信的开销，提高性能，也可以尽可能地利用 thread local 来做极低成本的任务间通信。基于 thread-per-core 模型的运行时特点包括：所有任务在固定线程运行，没有任务窃取；任务队列为 thread local 数据结构操作无锁无竞争；对于特定场景，如网关代理，更容易充分利用硬件性能，做到比较好的水平扩展性。
### 2.3 跨线程能力支持

thread-per-core 模型不代表没有跨线程能力，用户依然可以使用跨线程共享的数据结构，并且 Monoio 提供了跨线程等待的能力。跨线程等待的本质是在别的线程唤醒本线程的任务，Monoio 在 waker 中标记任务的所有权，如果当前线程并不是任务所属线程，那么 Monoio 通过无锁队列将任务发送到其所属线程上并唤醒。除了提供跨线程等待能力外，Monoio 也提供了 spawn_blocking 能力，供用户执行较重的计算逻辑，以免影响到同线程的其他任务。
### 2.4 异步运行时兼容

通过二次封装增加内存拷贝开销来兼容其他异步运行时，从而兼容其他运行异步运行时的生态。
## 三、mini-monoio 实现
### 3.1 Reactor 模块

mini-monoio 通过 polling crate 完成 epoll 操作实现 Reactor 模式。Reactor 数据结构和核心方法如下：

```Rust
pub struct Reactor {
    poller: Poller,
    waker_mapping: rustc_hash::FxHashMap<u64, Waker>,
    buffer: Vec<Event>,
}

impl Reactor {
    pub fn add(&mut self, fd: RawFd) { ... }
    pub fn modify_readable(&mut self, fd: RawFd, cx: &mut Context) { ... }
    pub fn modify_writable(&mut self, fd: RawFd, cx: &mut Context) { ... }
    pub fn delete(&mut self, fd: RawFd) { ... }
    pub fn wait(&mut self) { ... }
}
```

Reactor 数据结构使用 HashMap 管理事件对应的 waker，Poller 是对 epoll 操作的封装，buffer 用于存储已经完成待处理的事件。在 mini-monoio 的实现中，由于只关心读和写，并且 TCP 连接是全双工的，同一个 fd 上的读写无关，可以对应不同的 waker，将对 fd 的读事件表示为 fd * 2，对 fd 的写事件表示为 fd * 2 + 1。

> 这里可以通过 slab 减少内存频繁分配和释放带来的开销，slab 是一个用于分配和管理连续内存块的分配器，用于管理大小相等的内存块，这些内存块通常用于存储相同类型的数据结构。

add、modify_readable、modify_writeable、delete 用于设置所感兴趣的事件与 waker 之间的对应关系。当没有任务需要执行需要等待 IO 的情况下，wait 函数用于收割通过 Poller 管理并已经完成的事件，并通过事件对应的 waker 唤醒相应任务重新执行。
### 3.2 Executor 模块

**任务定义**

异步任务对应一个 Future，因为不知道 Future 的具体类型，所以这里通过 LocalBoxFuture 存储。

```Rust
pub struct Task {
    future: RefCell<LocalBoxFuture<'static, ()>>,
}
```

**工作队列定义**

通过 VecDeque 实现工作队列，因为 mini-monoio 所采用的 per-thread-core 模型，所以这里只需要通过 Rc 管理任务，而不需要考虑跨线程。

```Rust
pub struct TaskQueue {
    queue: RefCell<VecDeque<Rc<Task>>>,
}
```

**waker 定义**

> waker 需要实现函数的动态分发，并手动维护任务的引用计数，在此略过，具体细节可参考本文参考资料。

```Rust
impl Task {
    fn wake_(self: Rc<Self>) {
        Self::wake_by_ref_(&self)
    }

    fn wake_by_ref_(self: &Rc<Self>) {
        EX.with(|ex| ex.local_queue.push(self.clone()));
    }
}
```

在实现 waker 的动态分发的基础上，任务需要实现相应的 wake 函数，以供需要重新调度任务时将任务推送入工作队列中。

**Executor 实现**

```Rust
scoped_tls::scoped_thread_local!(pub(crate) static EX: Executor);

pub struct Executor {
    local_queue: TaskQueue,
    pub(crate) reactor: Rc<RefCell<Reactor>>,
    _marker: PhantomData<Rc<()>>,
}

impl Executor {
    pub fn spawn(fut: impl Future<Output = ()> + 'static) { ... }

    pub fn block_on<F, T, O>(&self, f: F) -> O
    where
        F: Fn() -> T,
        T: Future<Output = O> + 'static,
    {
        let _waker = waker_fn::waker_fn(|| {});
        let cx = &mut Context::from_waker(&_waker);
        EX.set(self, || {
            let fut = f();
            pin_utils::pin_mut!(fut);
            loop {
                if let std::task::Poll::Ready(t) = fut.as_mut().poll(cx) {
                    break t;
                }
                while let Some(t) = self.local_queue.pop() {
                    let future = t.future.borrow_mut();
                    let w = waker(t.clone());
                    let mut context = Context::from_waker(&w);
                    let _ = Pin::new(future).as_mut().poll(&mut context);
                }
                if let std::task::Poll::Ready(t) = fut.as_mut().poll(cx) {
                    break t;
                }
                self.reactor.borrow_mut().wait();
            }
        })
    }
}
```

由于每个工作线程都会有各自的 Executor 数据结构，所以这里通过 scoped_tls crate 实现 local thread storage。Executor 数据结构包括一个任务队列，对应的 reactor 数据结构。PhantomData 用于确保 Executor 不能实现 Send 和 Sync， 从而存在跨线程传输或访问的风险。

Executor 数据结构的核心接口包括 spawn 和 block_on 函数。spawn 函数将传入的异步任务 Future 包装后推送入工作队列。block_on 函数是异步运行时的主循环入口，它首先创建一个虚拟的 waker 对象用来创建异步运行时的执行上下文，这个虚拟的 waker 对象本身不做任何事情，然后在当前线程循环执行工作队列中的任务以及等待 IO，当传入的异步任务完成后退出。这里利用 pin_utils crate 宏创建 Pin 指针，确保异步任务不会在运行中移动。
### 3.3 IO 组件实现

这里实现了一个适配 mini-monoio 的 TcpStream IO 组件，通过为其适配 tokio::io::AsyncRead 和 tokio::io::AsyncWrite trait，并将相应的异步 IO 任务挂载至 mini-monoio 的 Reactor 模块，从而支持由 mini-monoio 推动整个异步流程的进行。

```Rust
pub struct TcpStream {
    stream: StdTcpStream,
}

impl From<StdTcpStream> for TcpStream {
    fn from(stream: StdTcpStream) -> Self {
        let reactor = get_reactor();
        reactor.borrow_mut().add(stream.as_raw_fd());
        Self { stream }
    }
}

impl Drop for TcpStream {
    fn drop(&mut self) {
        let reactor = get_reactor();
        reactor.borrow_mut().delete(self.stream.as_raw_fd());
    }
}

impl tokio::io::AsyncRead for TcpStream {
    fn poll_read(
        mut self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
        buf: &mut tokio::io::ReadBuf<'_>,
    ) -> Poll<io::Result<()>> { ... }
}

impl tokio::io::AsyncWrite for TcpStream {
    fn poll_write(
        mut self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
        buf: &[u8],
    ) -> Poll<Result<usize, io::Error>> { ... }

    fn poll_flush(
        self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
    ) -> Poll<Result<(), io::Error>> { ... }

    fn poll_shutdown(
        self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
    ) -> Poll<Result<(), io::Error>> { ... }
}
```
## 参考资料

* https://www.cloudwego.io/zh/blog/2023/04/17/%E5%AD%97%E8%8A%82%E5%BC%80%E6%BA%90-monoio-%E5%9F%BA%E4%BA%8E-io-uring-%E7%9A%84%E9%AB%98%E6%80%A7%E8%83%BD-rust-runtime/
* https://www.ihcblog.com/rust-runtime-design-1/
* https://rustmagazine.github.io/rust_magazine_2021/chapter_12/monoio.html