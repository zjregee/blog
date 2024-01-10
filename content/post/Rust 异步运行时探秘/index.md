---
title: Rust 异步运行时探秘
slug: tokio
image: cover.jpg
date: 2023-11-10 00:00:00+0000
categories:
    - 学习笔记系列
tags:
    - Rust
---

## 一、异步编程抽象

异步编程是一种编程模型，用于处理非阻塞的、并发的操作，旨在提高程序性能和资源利用率。在传统的同步编程中，每个操作都会等待上一个操作完成后再执行，而在异步编程中，程序可以继续执行其他操作，而不必等待当前操作完成，这使得程序可以在处理 I/O 操作和网络请求等耗时操作时更加高效。

> 异步编程是一个相对于同步编程的概念，与多线程编程存在一定的关联与区别。这一点，会在后续对 Rust 异步运行时的实现探秘中有更加深刻的理解。简单且不完全的来说，异步编程可以是单线程，也可以是多线程。多线程编程模型聚焦于线程同步、共享资源访问等问题。两者有相似的概念和实现，也有各自不同的聚焦问题，不同的解决方案。虽然不是等同的概念，但可以结合使用以发挥各自的优势。

在不同的编程语言中，异步编程的实现方式各有不同。常见的异步编程抽象包括 Callbacks、Promises/Futures、Async/Await 等。
### 1.1 Callbacks

基于 Callbacks 的异步编程是一种处理异步操作的编程范式，它通过在异步任务完成时调用预定义的回调函数来处理结果。这种模式通常用于事件驱动的环境，例如用户界面开发等。在基于 Callbacks 的异步编程中，启动异步操作的同时会在异步操作中注册一个用于处理操作完成后结果的回调函数。启动异步操作的线程不等待异步任务完成，而继续执行其他操作。当异步任务完成时，系统调用事先注册的回调函数，并将结果传递给该函数。这种异步编程模式存在的主要问题在于实际开发中很容易因为各种需求而实现多层复杂嵌套的回调，这种被称为回调地狱的代码通常难以维护，可读性差。下面是一个在 C++ 中实现基于 Callbacks 的异步编程的示例。

```C++
#include <thread>
#include <functional>

void async_task(const std::function<void(int, const std::string&)>& callback) {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    int error_code = 0;
    std::string result = "";
    callback(error, result);
}

void handle_data(int err_code, const std::string& result) { ... }

int main() {
    std::thread(async_task, handle_data).detach();
    ...
    return 0;
}
```
### 1.2 Futures/Promises

Futures/Promises 是异步编程中的一个经典解决方案，在诸多编程语言和框架中得到了广泛的应用。

Future 是一种表示异步操作结果的对象，通常表示一个尚未完成的操作，它代表了一个可能在将来完成的值。Future 提供了一种非阻塞的方式，允许程序继续执行其他任务而不用等待完成。Future 可以有三种状态：未完成、已完成成功、已完成失败。每一个异步操作返回一个 Future 对象，可以在需要的时候获取结果。Future 是只读的，可以注册回调函数，以便在 Future 完成时执行特定的操作。

Promise 是一种表示异步操作的进行中状态的对象，用于控制 Future 的状态。Promise 对象可以用来设置 Future 的结果，一旦 Promise 设置了结果，与之相关的 Future 就会被通知，并触发相应的回调函数。Future 是可写的，可以通过接口改变状态。

Future 和 Promise 的设计理念整体上非常相似，在不同编程语言和框架中所实现的异步编程中的概念处理上存在一定区别。一般来说，Future 可以视为异步任务的返回值，表示一个未来值的占位符，Promise 表示异步任务的执行过程，表示一个值的生产过程。将异步操作分成 Future 和 Promise 两个部分的主要原因是为了实现读写分离，对外部调用者只读，对内部实现可写。下面是在 C++ 中基于 Futures/Promises 实现异步编程的一个示例。在这个示例中，promise 用于设置异步操作的结果，future 用于获取异步操作的结果，异步线程通过 std::thread 启动，主线程通过 future.get() 等待异步操作完成，并获取结果。

```C++
#include <thread>
#include <future>

int aync_task() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return 42;
}

int main() {
    std::promise<int> promise;
    std::future<int> future = promise.get_future();
    std::thread async_thread([&promise]() {
        int result = async_task();
        promise.set_value(result);
    } catch (...) {
        promise.set_exception(std::current_exception());
    })
    int result = future.get();
    async_thread.join();
    return 0;
}
```
### 1.3 Async/Await

Async/Await 是一种现代编程语言中常见的异步编程模式，在许多编程语言中，也已经成为处理异步操作的主流方式。Async/Await 的设计出发点旨在简化异步编程，提供一种更直观、易读的方式来处理异步操作。Async/Await 可以让异步代码看起来更像同步代码，使得开发者能够使用类似于同步编程的方式编写异步代码，代码流程更加线性，使得理解代码的执行顺序变得更加容易，减少了异步操作带来的复杂性和混乱。

在基于 Async/Await 的异步编程中，编程语言通常会定义 async 和 await 关键字。async 关键字用于标识一个异步函数，异步函数返回一个 Future，代表一个尚未完成的异步计算。await 关键字用于等待一个 Future 完成，并在其结果可用时继续执行。异步函数可以包含 await 关键字，await 关键字会挂起当前的异步函数，允许其他任务在等待时执行。await 关键字只能在异步函数中使用，因为它会挂起当前的异步函数，而在同步函数中无法实现这种挂起和恢复的机制。下面是在 Python 中基于 Async/Await 实现异步编程的一个示例。

```Python
import asyncio

async def async_operation():
    await asyncio.sleep(2)

async def main():
    await async_operation()

asyncio.run(main())
```
## 二、Rust 中的异步运行时

Rust 中的异步编程模型基于 Async/Await 异步编程抽象，且拥有相比于其他主流语言不同的异步运行时实现。在 JavaScript 中，JavaScript 是一种事件驱动、单线程的语言，JavaScript 基于 Async/Await 的异步编程模型通过事件循环实现非阻塞的异步执行。在 Python 中，基于 Async/Await 的异步编程模型通过有栈协程实现，Python 的 asyncio 模块提供了异步运行时，支持异步任务的调度和执行。

Rust 中的 Async/Await 编程模型通过无栈协程实现，并通过第三方异步运行时库来支持，而不是内置于语言本身。在编译时，Rust 会将异步函数自动转换为状态机，状态机的每个状态代表函数的执行状态，await 关键字对应状态机中的一个状态转换。这个过程是在编译时完成的，因此在运行时不会引入额外的性能开销。状态机需要在第三方异步运行时库中得到调度，由异步运行时来推进异步函数的各个部分。相比于其他主流语言，Rust 中的异步运行时通过无锁设计、无栈协程、轻量级运行时和零成本抽象等往往可以得到更好的性能优势。这种设计不仅提高了异步执行的效率，还为开发者提供了更灵活的异步编程选项，以适应不同的应用场景和性能要求。

> **有栈协程 vs 无栈协程**
> 
> 相较于由操作系统内核管理且上下文切换开销大的线程，协程是由程序本身进行调度的轻量级执行单元，协程的切换更加轻量，不涉及操作系统的内核调度。有栈协程和无栈协程是两种协程实现的不同方式，有栈协程由类似于线程栈独立的执行栈，无栈协程不使用独立的执行栈，使用上更轻量。

Rust 开源社区提供了多种成熟的异步运行时实现，其中最流行的框架包括 Tokio、async-std 和 smol。Tokio 是 Rust 中最受欢迎的异步运行时，Tokio 提供了一个强大的异步生态系统，涵盖了网络编程、文件系统操作、定时器、以及其他与异步 I/O 相关的功能，并且得益于 Tokio 所提供的丰富的组件和强大的灵活性，可以使得 Tokio 适用于各种不同类型的异步编程场景。async-std 设计时注重简洁与标准库的一致性，使得开发者可以轻松地将现有代码转换为异步代码。smol 是一个轻量级的异步运行时，他的设计目标是简化异步编程并减少依赖关系，相较于 Tokio 和 async-std，smol 的功能可能更加简化，这也使得它成为一种更加轻便、适用于特定场景的异步运行时选择。
## 三、走进 Tokio

上一节中已经提到 Tokio 是 Rust 中最常用的异步运行时，它提供了编写网络应用程序所需的构建块，同时提供了针对各种系统的灵活性。从高层次上，Tokio 主要组成部分包括用于执行异步代码的多线程运行时、标准库的异步版本和一个庞大的相关库生态系统。当使用异步方式编写应用程序时，通过减少同时执行许多操作的成本，可以更好的扩展应用程序。然而，异步 Rust 代码不能自己运行，所以必须选择一个运行时来执行它。此外，在编写异步代码时，不能直接使用 Rust 标准库提供的普通阻塞 API，而必须使用由 Tokio 提供的异步版本。Tokio 提供了运行时的多种变体,从多线程、窃取工作的运行时到轻量级的单线程运行时。每个运行时都允许用户根据自己的需要进行调整。并且 Tokio 的诸多功能，例如 TCP、UDP、Unix 套接字、计时器、同步使用程序等，可以根据应用程序所需要的特性选择，从而优化编译时间或最终程序占用的空间。

尽管 Tokio 对于需要同时做很多事情的项目很有用，但也有如下的诸多情况不适合使用 Tokio：

* 通过在多个线程上并行运行，加速 CPU 密集型计算。Tokio 是为 IO 绑定应用程序设计的，其中每个单独的任务花费大部分时间等待 IO。如果应用程序所做的唯一事情是并行运行计算，那么可以使用 Rayon。
* 读取大量文件。虽然看起来 Tokio 对于只需要读取大量文件的项目很有用，但是与普通线程池相比，Tokio 在这种情况下没有提供任何优势，这是因为操作系统通常不提供异步文件 API。
* 发送单个 web 请求。如果需要使用一个异步 Rust 库，比如 reqwest，但是不需要一次做很多事情，使用这个库的阻塞版本会让项目更简单。Tokio 仍然可以工作，但是相比阻塞 API 没有真正的优势。
### 3.1 Hello Tokio

```Rust
use mini_redis::{client, Result};

#[tokio::main]
async fn main() -> Result<()> {
    let mut client = client::connect("127.0.0.1:6379").await?;
    client.set("hello", "world".into()).await?;
    let result = client.get("hello").await?;
    println!("got value from the server; result={:?}", result);
    Ok(())
}
```

使用异步编程，不能立即完成的操作被挂起到后台。线程没有被阻塞，并且可以继续运行其他事情。一旦操作完成，任务将被解除挂起，并从它停止的地方继续处理。如上的示例只有一个任务，因此在挂起期间什么也不会发生，但是异步程序通常有许多这样的任务。

在 async 函数中对 .await 的任何调用都会将控制返回给线程。当操作在后台进行时，线程可以做其他工作。Rust 的异步操作是惰性的，这导致了与其他语言不同的运行时语义。异步函数像其他 Rust 函数一样被调用。然而，调用这些函数并不会导致函数体的执行。相反，调用 async 函数返回一个表示操作的值。这在概念上类似于零参数闭包。要实际运行该操作，应对返回值使用 .await 操作符。async 函数的返回值是实现 Future trait 的匿名类型。

```Rust
#[tokio::main]
async fn main() {
    println!("hello");
}

fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("hello");
    })
}
```

用于启动应用程序的主函数不同于大多数 Rust 代码中常见的函数，它是一个 async 函数，并且带有 #[tokio::main] 标注。当我们想要进入异步上下文时，使用 async fn。然而，异步函数必须由运行时执行。运行时包含异步任务调度器，提供事件 I/O，计时器等。运行时不会自动启动，所以 main 函数需要启动它。#[tokio::main] 函数是一个宏，它将 async fn main() 转换为初始化运行时实例并执行 async main 函数的同步 fn main()。
### 3.2 Spawning

并发性和并行性不是一回事。如果在两个任务之间交替进行，那么只是在同时处理两个任务，但不是并行的。使用 Tokio 的优点之一是异步代码允许并发地处理许多任务，而不必使用普通线程并行地处理它们。事实上，Tokio 可以在一个线程上并发地运行多个任务。

一个 Tokio 任务是一个异步的绿色线程，它们是通过传递一个异步块给 tokio::spawn 来创建的。spawn 函数返回一个 JoinHandle，调用者可以使用它与生成的任务交互。异步块可以有一个返回值，调用者可以使用 JoinHandle 上的 .await 获取返回值。等待 JoinHandle 返回一个结果。当任务在执行过程中遇到错误时，JoinHandle 将返回 Err。当任务出现 panic，或者运行时强制取消任务时，就会发生这种情况。

> 绿色线程是一种用户空间线程，由应用程序或运行时系统而不是操作系统内核来管理，相较于传统的操作系统线程更加轻量。绿色线程通过用户空间的协作式调度实现，而非操作系统的线程调度器进行抢占式调度。在协作式调度中，一个绿色线程主动让出执行权给其他线程，而不是强制性地倍系统调度切换。相较于协程，绿色线程是一种更宽泛的概念，而协程通常是一种具体实现，调度方式可以是协作式的，也可以是抢占式的，可以是绿色线程的一种形式。

任务是调度程序管理的执行单元。生成任务将其提交给 Tokio 调度器，然后确保任务在有工作要做时执行。生成的任务可以在与其生成的线程相同的线程上执行，也可以在不同的运行时线程上执行。任务可以在生成之后在线程之间移动。Tokio 的任务很清凉，在底层，它们只需要单一的分配和 64 字节的内存，应用程序可以自由地生成数千甚至数百万个任务。
#### 3.2.1 'static bound

当在 Tokio 运行时上生成任务时，其类型的生成器必须是静态的。这意味着生成的任务必须不包含对任务外部拥有的数据的任何引用。下面的代码存在编译错误，这是因为默认情况下变量不会移动到异步块中，v 数组仍然属于 main 函数。通过使用 task::spawn(async move {}) 可以指示编译器将 v 数组移动到生成的任务中。任务拥有它的所有数据，使它成为 'static。如果必须同时从多个任务访问单个数据块，可以使用例如 Arc 之类的同步原语。

```Rust
use tokio::task;

#[tokio::main]
async fn main() {
    let v = vec![1, 2, 3];
    task::spawn(async {
        println!("Here's a vec: {:?}", v);
    });
}
```

> 一个常见的误解是 'static 意味着永远存在，但事实上，一个标记为 'static 的值仅仅意味着不存在内存泄漏，意味着永远保持这个值是正确的。这一点很重要，因为编译器无法推断新生成的任务存在了多长时间。通过 'static，开发者提供了这项任务能够永远运行下去的保障（当然实际上不会永远运行下去，提供这个保障意味着在开发者看来一直持有这个值是合理的），这样 Tokio 就可以让这项任务能运行多久就运行多久。这部分可以参考 [https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program)。
#### 3.2.2 Send bound

由 tokio::spawn 生成的任务必须实现 Send。这允许 Tokio 运行时在任务在 .await 处挂起时在线程之间移动任务。当所有 .await 调用处保存的数据是 Send 的，那么任务就是 Send 的。当任务调用 .await 时，任务将执行权返回给调度程序，下次执行该任务时，任务将从上次返回的点恢复。为了实现这个功能。任务需要保存 .await 处使用的所有状态。如果这些状态是 Send 的，那么任务本身就可以跨线程移动，相反，如果这些状态不是 Send 的，任务就无法满足 Send。

例如如下代码是正确的：

```Rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        {
            let rc = Rc::new("hello");
            println!("{}", rc);
        }
        yield_now().await;
    });
}
```

而如下代码是错误的：

```Rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        let rc = Rc::new("hello");
        yield_now().await;
        println!("{}", rc);
    });
}
```
### 3.3 Shared State

在 Tokio 中有几种不同的方式来共享状态：用互斥锁来保护共享状态；使用一个任务用于管理状态，通过消息传递对其进行保护。通常第一种方法用于简单数据，第二种方法用于需要异步工作的情况（例如 I/O 原语）。在本小节中，示例代码的共享状态是一个 HashMap，操作都不是异步的，可以使用互斥锁。

```Rust
use tokio::net::TcpListener;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();
    println!("Listening");
    let db = Arc::new(Mutex::new(HashMap::new()));
    loop {
        let (socket, _) = listener.accept().await.unwrap();
        let db = db.clone();
        println!("Accepted");
        tokio::spawn(async move {
            process(socket, db).await;
        });
    }
}
```

需要注意的是，上述的代码使用 std::sync::Mutex 而不是 tokio::sync::Mutex 来保护 HashMap。一个常见的错误是在异步代码中无条件地使用 tokio::sync::Mutex，这是一个通过 .await 调用锁定的异步互斥锁。同步互斥锁在等待获取锁时会阻塞当前线程，反过来，这将阻止其他任务的处理，在这种情况下使用异步互斥锁并没有任何帮助，因为异步互斥锁内部同样使用的是同步互斥锁。根据经验，在异步代码中使用同步互斥锁是可行的，只要争用保持在较低的水平，且锁不会在调用 .await 时仍然持有。
#### 3.3.1 tasks，threads and contention

当争用最少时，使用阻塞互斥锁来保护短临界区是一种可接受的策略。当锁被争夺时，执行任务的线程必须阻塞并等待互斥锁。这不仅会阻塞当前任务，还会阻塞当前线程上调度的所有其他任务。默认情况下，Tokio 运行时使用多线程调度器。任务安排在运行时管理的任意数量的线程上。如果计划执行大量任务，并且它们都需要访问互斥锁，那么就会出现争用。如果使用 current_thread 运行时风格的 Tokio 运行时，那么互斥锁将永远不会被争夺。

> current_thread 运行时风格的 Tokio 运行时是轻量级的单线程运行时。当只生成少量任务并打开少量套接字时，这是一个不错的选择。例如，在异步客户端库之上提供同步 API 桥接时，它可以很好地工作。

如果同步互斥锁的争抢成为一个问题，最好的解决办法很少是切换至 Tokio 互斥锁，通常需要考虑这些优化：切换到专用任务来管理状态和使用消息传递；对互斥对象分片；重构代码以避免互斥。
#### 3.3.2 holding a MutexGuard across an .await

```Rust
use std::sync::{Mutex, MutexGuard};

async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;
    do_something_async().await;
}
```

上述代码存在错误，因为 std::sync::MutexGuard 类型不是 Send。为了避免这个错误，需要让互斥锁的析构函数在调用 .await 之前运行。修改后的下面的代码是正确的：

```Rust
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    {
        let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
        *lock += 1;
    }
    do_something_async().await;
}
```

下面的代码同样存在错误：

```Rust
use std::sync::{Mutex, MutexGuard};
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;
    drop(lock);
    do_something_async().await;
}
```

这是因为编译器当前仅根据作用域信息计算 future 是否为 Send。这一点有望在未来进行更新，以支持显式地删除它，但现在必须显式地使用作用域。

除此之外，事实上问题的另一个根本原因不在于互斥锁有没有实现 Send，不应该试图通过使用某种方式 tokio::spawn 没有实现 Send 的任务。因为如果 Tokio 如果在任务持有锁的情况下挂其任务，其他试图争抢锁的任务可能被安排在同一个线程上执行，这将导致死锁，等待互斥锁的任务将阻止持有互斥锁的任务释放互斥锁。

**重构代码，不要在调用 .await 时持有锁**

之前已经提供了这样的一个代码示例在调用 .await 之前释放锁，还可以通过一些更健壮的方式来实现这一点。例如在如下的代码中，将互斥锁封装在一个结构体中，并且只将互斥锁在该结构体的非异步方法中上锁。这样的设计模式可以保证不会出现 Send 问题，因为互斥锁不会出现在异步函数中的任何地方。

```Rust
use std::sync::Mutex;

struct CanIncrement {
    mutex: Mutex<i32>,
}

impl CanIncrement {
    fn increment(&self) {
        let mut lock = self.mutex.lock().unwrap();
        *lock += 1;
    }
}

async fn increment_and_do_stuff(can_incr: &CanIncrement) {
    can_incr.increment();
    do_something_async().await;
}
```

**使用一个任务来管理状态，并使用消息传递对其进行操作**

这种方法通常在共享资源是 I/O 资源时使用。

**使用 Tokio 的异步锁**

异步互斥锁的主要特点是它可以在 .await 中被持有而不会出现任何问题。也就是说，异步互斥锁的开销要比普通互斥锁高，通常最好使用另外两种方法中的一种。

```Rust
use tokio::sync::Mutex;

async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock = mutex.lock().await;
    *lock += 1;
    do_something_async().await;
}
```
### 3.4 Channels

```Rust
use mini_redis::client;

#[tokio::main]
async fn main() {
    let mut client = client::connect("127.0.0.1:6379").await.unwrap();
    let t1 = tokio::spawn(async {
        let res = client.get("foo").await;
    });
    let t2 = tokio::spawn(async {
        client.set("foo", "bar".into()).await;
    });
    t1.await.unwrap();
    t2.await.unwrap();
}
```

上面的代码无法编译成功，因为两个任务都需要以某种方式访问 client，由于 client 没有实现 Copy，如果没有一些代码来促进这种共享，它将无法编译。虽然可以为每个任务打开一个连接，但这显然也不是一个理想的方法。在这种情况下也不能使用 std::sync::Mutex，因为需要在持有锁的情况下调用 .await。这里可以使用 Tokio 提供的异步锁，但这只允许一个正在运行的请求。对于诸如 redis 此类实现流水线客户端的场景来收，异步互斥会导致连接利用率不足。

上述的问题可以通过消息传递解决。使用此策略，管理 client 的任务能够获得独占访问权限，以便调用 get 和 set。此外，channel 还可以用作缓冲区，操作可以在 client 任务繁忙时发送给 client 任务。一旦 client 任务可以处理新请求，它就会从 channel 中提取下一个请求。这可以产生更好的吞吐量，并扩展到支持连接池。
#### 3.4.1 Tokio‘s channel primitives

Tokio 提供了许多种 channel，每种 channel 都有不同的用途：mpsc，多生产者，单消费者 channel，可以发送多个值；oneshot，单生产者，单消费者 channel，可以发送单个值；broadcast，多生产者，多消费者 channel，可以发送多个值，每个接收者都能看到每个值；watch，单生产者，多消费者 channel，可以发送多个值，但不保留历史记录，接收方只能看到最近的值。

如果需要一个多生产者多消费者 channel，其中每条消息只有一个消费者看到，可以使用 async-channel 这个 crate。还有一些 channel 可以在异步 Rust 代码之外使用，比如 std::sync::mpsc 和 crossbeam::channel。这些 channel 通过阻塞线程来等待消息，这在异步代码中是不允许的。

当每个 Sender 都超出范围或被丢弃，就不可能再向 channel 中发送更多消息。此时，Receiver 上的 recv 调用将返回 None，这意味着所有发送方都已离开并且 channel 已经关闭。oneshot 调用发送不需要 .await。这是因为 oneshot channel 上发送会立即失败或成功，而不需要任何形式的等待。
#### 3.4.2 一次错误理解

```Rust
use bytes::Bytes;
use mini_redis::client;
use tokio::sync::{mpsc, oneshot};

#[derive(Debug)]
enum Command {
    Get {
        key: String,
        resp: Responder<Option<Bytes>>,
    },
    Set {
        key: String,
        val: Bytes,
        resp: Responder<()>,
    },
}

type Responder<T> = oneshot::Sender<mini_redis::Result<T>>;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(32);
    let tx2 = tx.clone();
    let manager = tokio::spawn(async move {
        let mut client = client::connect("127.0.0.1:6379").await.unwrap();
        while let Some(cmd) = rx.recv().await {
            match cmd {
                Command::Get { key, resp } => {
                    let res = client.get(&key).await;
                    let _ = resp.send(res);
                }
                Command::Set { key, val, resp } => {
                    let res = client.set(&key, val).await;
                    // Ignore errors
                    let _ = resp.send(res);
                }
            }
        }
    });
    let t1 = tokio::spawn(async move {
        let (resp_tx, resp_rx) = oneshot::channel();
        let cmd = Command::Get {
            key: "foo".to_string(),
            resp: resp_tx,
        };
        if tx.send(cmd).await.is_err() {
            eprintln!("connection task shutdown");
            return;
        }
        let res = resp_rx.await;
        println!("GOT (Get) = {:?}", res);
    });
    let t2 = tokio::spawn(async move {
        let (resp_tx, resp_rx) = oneshot::channel();
        let cmd = Command::Set {
            key: "foo".to_string(),
            val: "bar".into(),
            resp: resp_tx,
        };
        if tx2.send(cmd).await.is_err() {
            eprintln!("connection task shutdown");
            return;
        }
        let res = resp_rx.await;
        println!("GOT (Set) = {:?}", res);
    });
    t1.await.unwrap();
    t2.await.unwrap();
    manager.await.unwrap();
}
```

在第一次阅读上面的代码时，我产生了误解，我认为对 t1 调用 .await，由于只能等待 t1 返回，才能运行接下来的代码，但是 t1 又在等待接受消息，所以这个地方会产生死锁。但其实不是这样，tokio::spawn 中的异步代码块不是在对 tokio::spawn 返回的 JoinHandle 调用 .await 时才开始运行的，异步任务已经提交到异步运行时开始运行了，调用 .await 是在等待执行结果。在这里混淆了对 async fn 调用 .await 的惰性执行与异步阻塞。
#### 3.4.2 backpressure and bounded channels

无论何时引入并发性或队列，都必须确保队列是有界的，并且系统能够优雅地处理负载。无界队列最终将填满所有可用内存，并导致系统以不可预测的方式失败。Tokio 注意避免隐形排队，其中很大一部分原因在于异步操作是惰性的。考虑如下的代码：

```Rust
loop {
    async_op();
}
```

如果异步操作急切地运行，循环将重复地将一个新的 async_op 排在队列中运行，而不确保之前的操作已经完成。这导致隐性无界排队。基于回调的系统和基于渴望 future 的系统特别容易受到这种影响。然而，对于 Tokio 和异步 Rust 代码，上面的代码片段根本不会导致 async_op 运行。这是因为 .await 没有被调用。如果将代码片段更新为使用 .await，则循环在重新开始之前等待操作完成。

```Rust
loop {
    async_op().await;
}
```

必须明确引入并发和队列，可以实现这一点的方法包括 tokio::spawn、select!、join! 和 mpsc::channel 等。这样做时，需要确保并发的总量是有限的。例如，在编写 TCP accept 循环时，确保打开的套接字总数是有限的。当使用 mpsc::channel 时，选择一个可管理的容量。注意并选择好的边界是编写可靠的 Tokio 应用程序的重要部分。
### 3.5 I/O

Tokio 中的 I/O 操作方式与 std 中的基本相同，但是是异步的，有一个用于读取的 trait AsyncRead 和一个用于写入的 trait AsyncWrite。特定类型需要适当地实现这些特征（TcpStream、File 和 Stdout）。AsyncRead 和 AsyncWrite 也已经由许多数据结构实现。
#### 3.5.1 AsyncRead and AsyncWrite

这两个 trait 提供了异步读写字节流的功能，这些 trait 上的方法不会被直接调用，就像不会从 Future trait 中手动调用 poll 方法一样。相反，可以通过 AsyncReadExt 和 AsyncWriteExt 提供的实用程序方法来使用它们。

**async fn read()**

```Rust
use tokio::fs::File;
use tokio::io::{self, AsyncReadExt};

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];
    let n = f.read(&mut buffer[..]).await?;
    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```

AsyncReadExt::read 提供了一个将数据读入缓冲区的异步方法，返回所读的字节数。当 read() 返回 Ok(0) 时，这表示流已经关闭。任何 read() 的进一步调用都将立即以 Ok(0) 完成。对于 TcpStream 实例，这表示套接字的读部分已经关闭。

**async fn read_to_end()**

AsyncReadExt::read_to_end 从流中读取所有字节，直到 EOF。

**async fn write()**

AsyncWriteExt::write 将缓冲区写入，返回写入的字节数。

**async fn write_all()**

AsyncWriteExt::write_all 将整个缓冲区写入。
#### 3.5.2 helper functions

```Rust
use tokio::fs::File;
use tokio::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut reader: &[u8] = b"hello";
    let mut file = File::create("foo.txt").await?;
    io::copy(&mut reader, &mut file).await?;
    Ok(())
}
```

此外，就像 std 一样，tokio::io 模块包含许多有用的实用函数以及用于处理标准输入、标准输出和标准错误的 API。例如，tokio::io::copy 异步地将 reader 的全部内容复制到 writer 中。值得注意的是，这里利用了字节数组也实现了 AsyncRead 的事实。
#### 3.5.3 echo server example

在这里将使用两种略微不同的方式实现 echo 服务器。

**using io::copy()**

```Rust
use tokio::io;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6142").await?;
    loop {
        let (mut socket, _) = listener.accept().await?;
        tokio::spawn(async move {
            // copy data here
        });
    }
}
```

在这里只有一个 TcpStream，这个值同时实现了 AsyncRead 和 AsyncWrite。因为 io::copy 对 read 和 write 都要求 &mut，所以套接字不能同时用于两个参数。下面的代码是无法通过编译的：

```Rust
io::copy(&mut socket, &mut socket).await;
```

**splitting a reader + writer**

为了解决这个问题，必须将套接字拆分成一个读句柄和一个写句柄。拆分 reader/writer 的最佳方法取决于特定的类型。任何 reader + write 类型都可以用 io::split 实用程序进行分割。这两个句柄可以独立使用，包括用于单独的任务。

```Rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

#[tokio::main]
async fn main() -> io::Result<()> {
    let socket = TcpStream::connect("127.0.0.1:6142").await?;
    let (mut rd, mut wr) = io::split(socket);
    tokio::spawn(async move {
        wr.write_all(b"hello\r\n").await?;
        wr.write_all(b"world\r\n").await?;
        Ok::<_, io::Error>(())
    });
    let mut buf = vec![0; 128];
    loop {
        let n = rd.read(&mut buf).await?;
        if n == 0 {
            break;
        }
        println!("GOT {:?}", &buf[..n]);
    }
    Ok(())
}
```

因为 io::split 支持任何实现 AsyncRead + AsyncWrite 并返回独立句柄的值，所以内部使用 Arc 和 Mutex。这个开销可以通过 TcpStream 来避免。TcpStream 提供了两个专门的拆分函数。TcpStream::split 接受对流的引用，并返回读句柄和写句柄。因为使用了引用，所以两个句柄必须保持在调用 split() 的同一个任务上。这种专门的分割是零成本的，不需要 Arc 或 Mutex。TcpStream 还提供了 into_split，它支持可以跨任务移动的句柄，只需要一个 Arc。

```Rust
tokio::spawn(async move {
    let (mut rd, mut wr) = socket.split();
    if io::copy(&mut rd, &mut wr).await.is_err() {
        eprintln!("failed to copy");
    }
});
```

因为 io::copy() 是在拥有 TcpStream 的同一个任务上调用的，所以在这里可以使用 TcpStream::split。

**manual copying**

```Rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6142").await?;
    loop {
        let (mut socket, _) = listener.accept().await?;
        tokio::spawn(async move {
            let mut buf = vec![0; 1024];
            loop {
                match socket.read(&mut buf).await {
                    Ok(0) => return,
                    Ok(n) => {
                        if socket.write_all(&buf[..n]).await.is_err() {
                            return;
                        }
                    }
                    Err(_) => {
                        return;
                    }
                }
            }
        });
    }
}
```

由于使用了 AsyncRead 和 AsyncWrite 实用程序，所以必须将拓展 trait 纳入范围。

**allocating a buffer**

避免使用 stack 上分配的缓冲区，.await 调用期间存在的所有任务数据都必须由任务存储。在这个例子中，buf 在 .await 调用中使用。所有任务数据都存储在单个分配中，可以将其视为一个枚举，其中每个变量都是需要为特定的 .await 调用存储的数据。每个接受的套接字生成的任务的内部结构可能如下所示：

```Rust
struct Task {
    task: enum {
        AwaitingRead {
            socket: TcpStream,
            buf: [BufferType],
        },
        AwaitingWriteAll {
            socket: TcpStream,
            buf: [BufferType],
        }
    }
}
```

这会让任务结构变得非常庞大，并且由于缓冲区大小通常是 page 大小，这将使得任务的大小略大于 page 大小，变得非常尴尬。编译器相比基本的枚举类型进一步优化了异步块的布局。实际上，变量不会像基本的枚举类型那样在不同的种类之间移动，但是任务大小仍然至少与最大的变量一样大。基于此，通常更有效的做法是为缓冲区分配专门的内存，而不是使用通用的分配方式。
### 3.6 Bridging with Sync Code

在之前的使用 Tokio 的诸多示例中，通过 #[tokio::main] 标注了 main 函数，使得整个代码异步。在某些情况下，可能需要运行一小部分同步代码，这部分可以参考 spawn_blocking。在其他情况下，可能更容易将应用程序结构分为大部分同步，使用较小的或逻辑上不同的异步部分。例如，GUI 应用程序可能希望在主线程上运行 GUI 代码，并在旁边的另一个线程上运行 Tokio 运行时。这一小节介绍如何将 async/await 隔离到项目的一小部分。
#### 3.6.1 what #[tokio::main] expands to

```Rust
#[tokio::main]
async fn main() {
    println!("Hello world");
}

fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            println!("Hello world");
        })
}
```

#[tokio::main] 宏是一个宏，它将 main 函数替换为一个非异步 main 函数，该函数启动一个运行时，然后调用代码。从宏观上看，要在我们的项目中使用 async/await，可以做类似的事情，利用 block_on 方法在适当的地方进入异步上下文。
#### 3.6.2 a synchronous interface to mini-redis

```Rust
use tokio::net::ToSocketAddrs;
use tokio::runtime::Runtime;

pub use crate::clients::client::Message;

pub struct BlockingClient {
    inner: crate::clients::Client,
    rt: Runtime,
}

impl BlockingClient {
    pub fn connect<T: ToSocketAddrs>(addr: T) -> crate::Result<BlockingClient> {
        let rt = tokio::runtime::Builder::new_current_thread()
            .enable_all()
            .build()?;
        let inner = rt.block_on(crate::clients::Client::connect(addr))?;
        Ok(BlockingClient { inner, rt })
    }
}
```

上面的代码通过存储一个 Runtime 对象并使用它的 block_on 方法构建一个 mini-redis 的同步接口。在这里使用了 current_thread 运行时，这是因为 Tokio 默认的 multi_thread 运行时会产生一堆后台线程，所以它可以有效地同时运行许多事情。对于上面的示例，一次只做一件事，因此运行多个线程不会获得任何好处。这使得 current_thread 运行时非常适合，它不产生任何线程。enable_all 调用在 Tokio 运行时启用 IO 和定时器驱动程序。如果未启用它们，则运行时无法执行 IO 或计时器。因为 current_thread 运行时不产生任何线程，所以它只在调用 block_on 时操作。一旦 block_on 返回，该运行时上的所有生成任务将冻结，直到再次调用 block_on。如果生成的任务在不调用 block_on 时必须继续运行，则使用 multi_thread。

```Rust
use bytes::Bytes;
use std::time::Duration;

impl BlockingClient {
    pub fn get(&mut self, key: &str) -> crate::Result<Option<Bytes>> {
        self.rt.block_on(self.inner.get(key))
    }

    pub fn set(&mut self, key: &str, value: Bytes) -> crate::Result<()> {
        self.rt.block_on(self.inner.set(key, value))
    }

    pub fn set_expires(
        &mut self,
        key: &str,
        value: Bytes,
        expiration: Duration,
    ) -> crate::Result<()> {
        self.rt.block_on(self.inner.set_expires(key, value, expiration))
    }

    pub fn publish(&mut self, channel: &str, message: Bytes) -> crate::Result<u64> {
        self.rt.block_on(self.inner.publish(channel, message))
    }
}
```
#### 3.6.3 other approaches

**spawning things on a runtime**

```Rust
use tokio::runtime::Builder;
use tokio::time::{sleep, Duration};

fn main() {
    let runtime = Builder::new_multi_thread()
        .worker_threads(1)
        .enable_all()
        .build()
        .unwrap();
    let mut handles = Vec::with_capacity(10);
    for i in 0..10 {
        handles.push(runtime.spawn(my_bg_task(i)));
    }
    std::thread::sleep(Duration::from_millis(750));
    println!("Finished time-consuming task.");
    for handle in handles {
        runtime.block_on(handle).unwrap();
    }
}

async fn my_bg_task(i: u64) {
    let millis = 1000 - 50 * i;
    println!("Task {} sleeping for {} ms.", i, millis);
    sleep(Duration::from_millis(millis)).await;
    println!("Task {} stopping.", i);
}
```

Runtime 对象有一个名为 spawn 的方法，当调用此方法，将创建一个在运行时上运行的新的后台任务。上面的例子在运行时生成 10 个后台任务，然后等待它们全部完成。这可能是在图形应用程序中实现后台网络请求的一种好方法，因为网络请求在 GUI 主线程上运行太费时了。可以在后台运行的 Tokio 运行时中生成请求，并在请求完成时让任务将信息发送回 GUI 代码。如果想要一个进度条，甚至可以增量地发送信息。

在这个例子中，将运行时配置为 multi_thread 运行时是非常重要的。如果更改为 current_thread 运行时，耗时的任务将在任何后台任务开始之前完成。这是因为在 current_thread 运行时上产生的后台任务只会在调用 block_on 时执行，否则运行时没有任何地方可以运行它们。

这个示例通过调用 spawn 调用返回的 JoinHandle 上的 block_on 来等待生成的任务完成，还可以通过消息传递通道，例如 tokio::sync::mpsc，或通过修改受到例如互斥锁等保护的共享值。对于 GUI 中的进度条来说，这是一种很好的方法，因为 GUI 每帧读取共享值。spawn 方法也可以用于 Handle 类型，可以克隆 Handle 类型以获得运行时的许多句柄，并且每个 Handle 可用于在运行时上生成新任务。

**sending messages**

```Rust
use tokio::runtime::Builder;
use tokio::sync::mpsc;

pub struct Task {
    name: String,
}

async fn handle_task(task: Task) {
    println!("Got task {}", task.name);
}

#[derive(Clone)]
pub struct TaskSpawner {
    spawn: mpsc::Sender<Task>,
}

impl TaskSpawner {
    pub fn new() -> TaskSpawner {
        let (send, mut recv) = mpsc::channel(16);
        let rt = Builder::new_current_thread()
            .enable_all()
            .build()
            .unwrap();
        std::thread::spawn(move || {
            rt.block_on(async move {
                while let Some(task) = recv.recv().await {
                    tokio::spawn(handle_task(task));
                }
            });
        });
        TaskSpawner {
            spawn: send,
        }
    }

    pub fn spawn_task(&self, task: Task) {
        match self.spawn.blocking_send(task) {
            Ok(()) => {},
            Err(_) => panic!("The shared runtime has shut down."),
        }
    }
}
```

第三种方法是生成一个运行时，并使用消息传递与它通信。这比其他两种方法涉及更多的样板文件，但是是最灵活的方法。可以使用多种方法配置这个示例，例如使用信号量来限制活动任务的数量，或者使用相反方向的 channel 来发送响应。当使用这种方式生成一个运行时，这实际上是一种 atcor 模式的应用。

> 本章节主要参考了 Tokio 的官方教程，原文还提到了 Select、Streams、Graceful Shutdown 等内容可供参考。
## 参考资料

* https://tokio.rs/tokio/tutorial