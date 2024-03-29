---
title: 数据并行计算框架 mini-rayon 实现
slug: mini-rayon
image: cover.jpg
date: 2023-09-28 00:00:00+0000
categories:
    - mini 系列
tags:
    - Rust
---

> 本文是 mini 系列的第二篇文章，mini 系列是一些知名框架或技术的最小化实现，本系列文章仅作为学习积累的目的，代码或架构在很大程度上参考了现有的一些源码或博客。本文参考 Rust 语言的知名数据并行计算框架 Rayon 实现了一个小型的数据并行计算框架 mini-rayon，并基于 mini-rayon 进行了一系列的并行计算测试，取得了一定程度上的性能提升。
> 
> Rayon 项目地址：https://github.com/rayon-rs/rayon
> 
> mini-rayon 代码实现地址：https://github.com/zjregee/mini/mini-rayon
## 一、数据并行计算框架
### 1.1 并行计算框架介绍

并行计算是一种通过执行多个计算任务来提高计算效率的计算范式。与传统的串行计算方式不同，其中每个任务按顺序执行，而并行计算允许多个任务同时进行，从而在相同的时间内处理更多的数据或完成更复杂的计算任务。并行计算的核心思想是任务分解与并发执行。在并行计算中，大的计算任务被分解成许多小的子任务，这些子任务可以独立地并行执行。这种分解使得计算机系统能够更好地利用多个处理单元或多台计算机的资源，加速整体计算过程。分布在多个处理单元上的任务之间通过合理的协同工作方式实现协同完成整体任务。

并行计算的应用范围广泛，例如在科学研究中，通过并行计算可以更快地模拟复杂的物理现象、分析遗传数据或着优化化学反应。在工程领域，有限元分析、流体力学模拟等复杂问题也常常需要并行计算来提高效率。此外，在人工智能、大数据分析等领域，通过并行计算可以更快速地训练深度学习模型、处理海量数据。尽管并行计算提供了显著的性能提升，但其开发和管理复杂性也是不可忽视的挑战。为了有效利用并行计算的优势，需要仔细设计并行算法、选择合适的并行计算架构，并进行良好的任务调度和数据管理，处理任务分解的复杂性、通信开销、数据一致性等问题。

为了充分发挥并行计算的优势，开发人员通常需要使用专门的并行计算库和工具。这些库不仅提供了高效的并行算法实现，还简化了任务调度、数据共享和通信等方面的复杂工作。

一些广泛使用的并行计算库包括 OpenMP 和 MPI。OpenMP 适用于共享内存并行，通过简单的指令注释即可将串行代码转换为并行代码，使得开发人员能够轻松地利用多核处理器。而 MPI 则更适用于分布式内存并行，提供了丰富的消息传递机制，使不同计算节点上的任务能够协同工作。除了这些基础性的库之外，还有一些针对特定领域的高级并行计算框架，如 TensorFlow 和 Pytorch 用于深度学习，Hadoop 和 Spark 用于大数据处理。这些框架在底层实现了复杂的并行计算模型，使开发人员能够专注于问题领域而不是底层的并行细节。

并行计算在不同编程语言中有着不同的实现方式，同时针对各种计算场景有着各种成熟的并行计算框架实现。通过使用这些并行计算库和框架，开发人员能够在降低开发难度的同时，更有效地利用计算资源，实现复杂问题的快速解决。

Python 的 Ray 和 Rust 的 Rayon 是两个典型的例子，Ray 是一个用于构建分布式应用和并行计算的高性能框架，它特别适用于处理大规模数据和执行复杂任务。Ray 提供了任务并行性和数据并行性的支持，允许用户轻松地并行化 Python 代码。它还具有分布式计算的能力，可以在多台计算机上执行任务。Rayon 是 Rust 的一个并行计算库，它通过简化并行性的表达方式，使得在不引入显式锁的情况下能够轻松地实现并行计算。Rayon 使用工作窃取算法，自动管理线程池，使得开发者能够更轻松地编写高性能的并行 Rust 代码。除了 Ray 和 Rayon，Java 中的 Fork/Join 框架，Go 中的 goroutines 和 channels 也是一些成熟的并行计算框架。
### 1.2 数据并行与任务并行

数据并行和任务并行是两种广泛应用于并行计算领域的并行计算模型，它们分别关注并行处理数据和并行执行任务。在数据并行中，计算任务被划分成多个相似的子任务，并且这些子任务同时在不同的处理单元上执行。数据中心的主要思想是并行地处理相同的计算操作，但是在不同的数据集上。这样的模型特别适用于处理大规模数据集，例如矩阵运算、图像处理等。在任务并行中，计算任务被分解成多个相对独立的子任务，并且这些子任务在不同的处理单元上并行执行。任务并行通常关注于同时执行不同的计算任务，每个任务可以有不同的计算流程，但它们协同完成整个计算任务。任务并行中的各个任务之间可能需要协同工作。

数据并行和任务并行分别强调了在处理大规模计算问题时的不同方面，数据并行关注如何同时处理相似的计算任务在不同数据上，而任务并行关注如何同时执行不同的计算任务。在实际应用中，这两种并行计算模型可以结合使用，以最大程度地发挥计算资源的潜力，提高计算效率。
## 二、任务调度算法实现

基于工作窃取队列实现的任务调度算法在并行计算中是一种常见的调度算法，其旨在充分利用多核处理器的性能，减少线程间的竞争和等待时间。这种算法特别适用于任务并行模型，其中任务被划分成多个子任务，并且这些子任务在不同的处理单元上并行执行。

工作窃取队列的核心是通过任务的动态调度来实现负载均衡。每个线程都维护一个本地任务队列，其中存放了该线程需要执行的任务。当一个线程完成自己的任务时，它会首先从自己的本地队列中获取新的任务执行。如果本地队列为空时，使用工作窃取算法使得空闲线程可以从其他线程的队列中窃取任务。每个线程都有一个双端队列，当一个线程需要任务执行时，它会从队列的前端取出任务执行。这种方式有效地避免了线程之间的锁竞争，提高了整体的并行性。任务在被执行的过程中可能会被分割成更小的子任务。这样，每个线程都有机会在本地队列中保持一定数量的任务，提高并行性。

基于工作窃取队列实现的任务调度算法主要有三个优点：（1）负载均衡。工作窃取队列实现算法能够在运行时自适应地实现负载均衡，保证各个线程的工作量相对均匀，提高整体性能。（2）避免锁竞争。通过将任务放置在双端队列中，线程之间无需共享锁来保护队列的操作，减少了锁竞争的可能性，提高了并行性。（3）适应性。由于工作窃取队列算法能够根据任务的执行情况动态调整任务的分配和窃取策略，因此对于各种负载情况都能表现出较好的适应性。
### 2.1 工作窃取队列

无锁工作窃取队列有多种算法实现，包括 Chase-Lev、Treiber、M&S 算法等。接下来主要介绍介常见的 Chase-Lev 算法和 Treiber 算法实现逻辑：

* Chase-Lev 算法使用双端队列来组织任务，这个队列具有两个端，一个用于入队，另一个用于出队和工作窃取。队列中的每个任务节点包含了指向前一个和后一个节点的指针。
	* 入队操作：当一个线程需要入队时，它会创建一个新的节点，将任务存放在节点中，然后通过 CAS 操作将这个新节点插入到队列的入队端。
	* 出队操作：出队操作是指从队列的出队端移除节点。由于是双端队列，线程可以直接从自己的队列的出队端执行出队操作，无需使用 CAS 操作。
	* 工作窃取操作：当一个线程需要执行任务时，它首先从自己的队列的出队端取得任务。如果自己的队列为空，那么它会随机选择其他线程的队列，尝试从队列的入队端窃取任务。这个过程被称为工作窃取。
* Treiber 算法使用单向链表来实现队列结构。队列中的每个任务节点包含指向下一个节点的指针。
	* 入队操作：当一个线程需要入队时，它会首先创建一个新的节点，将任务存放在节点中。然后，它将新节点的指针设置为当前队列的头部，并将队列的头部指针指向新节点。这个过程是一个原子操作，通常使用 CAS 来实现。
	* 出队操作：当一个线程需要出队时，它会读取当前队列的头部指针，获取头部节点。然后，它将队列的头部指针更新为头部节点的下一个节点。出队操作也是一个原子操作。
	* 工作窃取操作：当一个线程需要执行任务时，它首先尝试从自己队列的头部出队一个任务。如果自己的队列为空，那么它会随机选择其他线程的队列，尝试从队列的头部出队任务。这个过程同样可以使用 CAS 来确保线程之间的竞争安全。

Tiber 算法使用于相对简单的并发场景，任务粒度较大且相对均匀的情况。由于其实现逻辑更为简单，相比于 Chase-Lev 算法更容易实现和维护。

> 在 mini-rayon 中，采用了 Rayon 早期版本中所使用的基于 Chase-Lev 算法的 dequeue crate。目前 Rayon 所使用的是 crossbeam crate 中实现的无锁工作窃取队列。
### 2.2、工作窃取机制

在任务调度中，除了工作窃取队列，还需要通过工作窃取机制来实现完整的任务调度流程。简单来说，工作窃取机制是决定了何时以及从哪个队列中窃取任务。常见的工作窃取策略包括 Child Stealing 和 Continuation Stealing 机制。

在 Child Stealing 机制中，每个线程都维护着自己的私有任务队列，当一个线程完成了自己队列中任务时，它会尝试从其他线程的队列中窃取任务来执行。具体来说，当一个线程空闲而其他线程仍有待执行的任务时，它会随机或按照一定策略来选择一个繁忙线程的任务队列，并尝试从队列头部或尾部窃取任务。Child Stealing 机制的优势在于促使任务在各个线程之间更均匀地分布，避免特定线程成为瓶颈。由于任务是按需发生的，这种策略具有较低的竞争程度，减少了线程之间的争夺，提高了整体并行计算的效率。

在 Continuation Stealing 机制中，将任务划分地更细粒度，讲任务拆分成更小的执行单元，称为 continuations。在这种机制中，每个线程都维护着自己的 continuation 队列，当一个线程完成了一个 continuation 时，它会尝试从其他线程的 continuation 队列中继续窃取任务。这一机制的核心思想是将任务的执行拆解为更小的、相互独立的子任务，使得在窃取任务的过程中更加细粒度和灵活。相比 Child Stealing 机制，Continuation Stealing 机制的优势在于适应了任务的不均衡性和执行时间的不确定性。通过将任务划分为更小的 continuations，系统能够更灵活地在各个线程之间调度的任务，使得执行时间差异较大的任务不至于影响整个系统的性能。这种策略对于需要处理异步时间、任务执行时间变化较大的应用场景特别有效。

> Continuation Stealing 机制的实现通常设计复杂的调度器和任务划分逻辑，因为 continuation 的粒度相对较小，需要有效地协调线程之间的任务执行顺序。在使用 Continuation Stealing 机制时，也需要程序员精心设计任务的划分和执行逻辑，以最大程度地发挥这种机制的优势。mini-rayon 通过 Child Stealing 机制实现工作窃取，以实现高效、简单的框架 API，避免为编程引入过多的开发成本。
## 三、mini-rayon 框架介绍

Rayon 是 Rust 中流行的数据并行计算框架，旨在简化并行编程并提高代码性能。它通过简单的 API 和 Rust 的所有权系统来实现并行计算，允许开发者轻松地将现有的迭代代码转换为并行执行的形式。 Rayon 的设计理念是提供一种易于使用的方式来利用多核处理器的性能优势，而无需处理底层的线程管理和同步问题。Rayon 的核心特性之一是并行迭代器，它们允许对集合进行并行操作，并提供了 map、filter 和 reduce 等集合操作的支持。Rayon 通过自动划分工作负载和动态调整线程池的大小来最大程度地利用可用的处理器核心，同时避免了一些常见的并发编程陷阱。

```Rust
let total_price = stores.iter()
                        .map(|store| store.compute_price(&list))
                        .sum();

let total_price = stores.par_iter()
                        .map(|store| store.compute_price(&list))
                        .sum();
```

mini-rayon 参考 Rayon 实现了一个轻量级的数据并行计算框架，可以轻松地将顺序计算转换为并行计算。例如上面这个常见的顺序迭代计算可以通过使用 mini-rayon 实现的并行迭代器即可将其转换为安全的并行运算。保证并行计算的安全性是使得并行计算框架变得简单易用的很重要的一部分原因。mini-rayon 所实现的并行迭代器将会负责决定如何将数据划分为任务，并通过动态调整来获得最佳性能。
## 四、框架根基 —— join 原语
### 4.1 潜在并行性

mini-rayon 基于基本原语 join 之上构建整个框架。join 的用法非常简单，可以使用如下的两个闭包调用它。join 会当这两个闭包都完成后返回，这个过程可能是并行的。可能并行的过程被称为潜在并行性，这其实是 min-rayon，同样也就是 Rayon 的核心设计原理。潜在并行性根据空闲 CPU 核心是否可用，动态决定是否使用并行线程。通过调用 join 来注释程序指示潜在并行性，并让运行时决定何时利用它。

```Rust
join(|| do_something(), || do_someting_else())
```

这种潜在并行性也是 mini-rayon 相较于其他并行计算框架的关键区别点之一。例如在使用一些线程池并行框架时，如果将两个任务放在不同的线程之上，它们将始终保持并发执行。mini-rayon 中的潜在并行性不仅可以使得 API 变得更简单，还可以提高执行效率。这是因为知道并行何时有利可图是很难提起预测的，并且总是需要有一定量的全局上下文，诸如计算机当前是否有空闲核心，目前正在进行哪些并行操作此类。潜在并行性相较于当前大部分并行计算框架的保证并行性存在鲜明的区别。
### 4.2 快排实现用例

```Rust
fn partition<T:PartialOrd+Send>(v: &mut [T]) -> usize { ... }

fn quick_sort<T:PartialOrd + Send>(v: &mut [T]) {
    if v.len() > 1 {
        let mid = partition(v);
        let (lo, hi) = v.split_at_mut(mid);
        quick_sort(lo);
        quick_sort(hi));
    }
}
```

基于 join 原语，我们可以将上面顺序运行的快排伪代码，方便地修改成如下并行化运行的快排伪代码。

``` Rust
fn quick_sort<T:PartialOrd + Send>(v: &mut [T]) {
    if v.len() > 1 {
        let mid = partition(v);
        let (lo, hi) = v.split_at_mut(mid);
        quick_sort(lo);
        quick_sort(hi));
    }
}
```
### 4.3 join 原语中的任务调度

join 在具体实现上使用了基于工作窃取机制的任务调度，其基本思想是在每次调用 join(a, b) 时，可以确定两个可以安全并行的任务 a 和 b。由于无法确认是否有空闲线程，当前调用 join 的线程将 b 添加到本地待处理工作队列中，然后立即开始执行 a。同时，还有一组其他活动的线程（通常每个处理器一个线程），每当它空闲时，就会去搜索其他线程的待处理工作队列，如果存在可窃取的任务就会窃取它并自己执行。在这种情况下，当第一个线程忙于执行 a 时，另一个线程可能会出现并开始执行 b。一旦第一个线程完成 a，它就会检查是否有其他线程已经开始执行 b 了。如果没有，第一个线程就继续自己执行 b，如果 b 被其他线程窃取，就需要等待 b 完成，在等待的过程中，第一个线程可以继续从其他处理器中窃取数据，从而尝试帮助推动整个过程完成。这部分的伪代码如下。

```Rust
fn join<A, B>(oper_a: A, oper_b: B)
where A: FnOnce() + Send,
      B: FnOnce() + Send,
{
    let job = push_onto_local_queue(oper_b);
    oper_a();
    if pop_from_local_queue(oper_b) {
        oper_b();
    } else {
        while not_yet_complete(job) {
            steal_from_others();
        }
        result_b = job.result();
    }
}
```

基于工作窃取机制的任务调度能够自然地适应处理器的负载。当每个工作线程都非常忙碌时，join(a, b) 的逻辑上会退化成顺序执行每个闭包。这使得 join 原语的抽象并不会使其比顺序执行代码差，在存在可用线程的情况下，join 原语就可以获得并行性带来的性能提升。
### 4.4 数据结构抽象
#### 4.4.1 任务数据结构

```Rust
pub trait Executable {
    fn execute(&mut self);
}

pub struct Code<F, R> {
    func: Option<F>,
    dest: *mut Option<R>,
}

pub struct Job {
    code: Box<dyn Executable>,
    latch: Arc<Latch>,
}

unsafe impl Send for Job { }
unsafe impl Sync for Job { }
```

在这里，我们实现了三个数据结构来代表 mini-rayon 中的任务执行单元。Job 代表具体执行的任务，code 字段通过 Box 和 dyn 关键字保存了实现 Executable trait 的类型，latch 字段提供了一种锁机制，用于探测任务的完成情况。Code 中的 func 字段保存了需要执行的函数，dest 字段用于保存函数的返回值，即执行结果。此外，我们通过 unsafe impl 为 Job 实现了 Send 和 Sync trait，这是因为 Job 需要在线程之间传递，每个 Job 都有可能会由其他线程来执行。

```Rust
impl<F, R> Executable for Code<F, R>
where
    F: FnOnce() -> R,
{
    fn execute(&mut self) {
        if let Some(func) = self.func.take() {
            unsafe {
                *self.dest = Some(func());
            }
        }
    }
}

impl Job {
    pub fn execute(&mut self) {
        self.code.execute();
        self.latch.set();
    }
}

```

上面是我们为 Code 和 Job 实现的同步执行函数，会通过工作线程调用来完成任务的执行。值得一提的是，我们在任务中保存的函数需要满足 FnOnce() -> R trait bound 。这有两个原因，首先我们在有需要的情况提供了保存任务返回结果的能力，其次 FnOnce、FnMut 和 Fn 是三种不同的表示闭包或函数的 trait，FnOnce trait 表示可以调用一次的闭包，它会获取捕获变量的所有权，FnMut trait 表示可以多次调用的闭包，它可以修改捕获的变量的值，但不会获取其所有权，Fn trait 表示不可变闭包引用，它不能修改捕获的变量的值，也不会获取其所有权，这三个 trait 是一种包含关系，如果一个闭包实现了 FnOnce trait，那么它也能实现 FnMut trait 和 Fn trait，如果一个闭包实现了 FnMut trait，那么它也能实现 Fn trait，在我们的场景下，每个任务只会执行一次，FnOnce trait 通过引入获取变量所有权的限制，不仅更适用于 mini-rayon 的框架语义，也提供了更丰富场景的支持。
#### 4.4.2 任务注册管理

```Rust
struct RegistryState {
    threads_at_work: usize,
    injected_jobs: Vec<Job>,
}

pub struct Registry {
    thread_infos: Vec<ThreadInfo>,
    state: Mutex<RegistryState>,
    work_available: Condvar,
    terminated: AtomicBool,
}

unsafe impl Send for Registry { }
unsafe impl Sync for Registry { }
```

Registry 负责管理所有工作线程以及任务，因为 Registry 可能需要跨线程共享和传递，所以我们同样需要通过 unsafe impl 为其实现 Send 和 Sync trait。thread_infos 字段记录了每个工作线程的信息，state 字段记录目前活跃工作线程数以及还未分配工作线程的任务，当活跃工作线程数目大于 0 时，由于任务在执行过程中可能会产生新的待执行的任务，所以存在窃取任务的可能，work_available 是信号量，这一信号量的作用是当可能有新的任务可以执行或窃取时，会唤醒沉睡的工作线程数，注意这个地方是可能，而并不能保证一定有新的任务，terminated 字段用于停止整个线程池的运行，在初始化 Registry 时，会创建一定数量的工作线程，默认情况下工作线程数等于主机的 CPU 核心数。

```Rust
impl Registry {
    pub fn inject(&self, injected_jobs: Vec<Job>) {
        let mut state = self.state.lock().unwrap();
        state.injected_jobs.extend(injected_jobs);
        self.work_available.notify_all();
    }

    fn start_working(&self) {
        let mut state = self.state.lock().unwrap();
        state.threads_at_work += 1;
        self.work_available.notify_all();
    }

    fn wait_for_work(&self, was_active: bool) -> Option<Job> {
        let mut state = self.state.lock().unwrap();
        if was_active {
            state.threads_at_work -= 1;
        }
        loop {
            if let Some(job) = state.injected_jobs.pop() {
                return Some(job);
            }
            if state.threads_at_work > 0 {
                return None;
            }
            state = self.work_available.wait(state).unwrap();
        }
    }
}
```

Registry 的主要函数包括 inject、start_working 和 wait_for_work。inject 函数用于提交新的任务，因为使用 mini-rayon 进行并行计算加速的线程并不是工作线程，所以提交的新任务会首先放在 Registry 中，当 inject 函数被调用时，会触发 work_available 信号量，任务后续会被工作线程给取走。start_working 和 wait_for_work 函数是内部函数，由工作线程调用。当工作线程开始执行一个新任务时，会通过 start_working 函数告知 Registry，原因在之前已经提到，本质上是告诉 Resgitry 当前有窃取任务的可能。当工作线程没有任务执行时，会调用 wait_for_work 函数，was_active 参数代表工作线程在等待工作之前是否有在执行任务，如果有的话，需要修改目前活跃工作线程数。在工作线程调用 wait_for_work 函数后，会有四种结果，第一种情况没有获取 Registry 的锁，此时会首先等待获取锁，没有锁意味着有其他工作线程在等待任务，那么即使该工作线程获取了锁，也没有任务需要执行，需要继续等待任务，所以这里对于锁的等待是合理的，与任务的等待等价，第二种情况 Registry 存在还未分发的任务，工作线程直接取走该任务并开始执行，第三种情况，当前活跃工作线程数大于 0，放弃从 Registry 中获取任务，尝试去其他工作线程的工作队列中窃取任务，第四种情况，不存在可执行以及可窃取的任务，所以需要在 Registry 中继续等待任务。
#### 4.4.3 工作线程数据结构

```Rust
struct ThreadInfo {
    primed: Latch,
    worker: Worker<Job>,
    stealer: Stealer<Job>,
}

pub struct WorkerThread {
    registry: Arc<Registry>,
    index: usize,
}

thread_local! {
    static WORKER_THREAD_STATE: RefCell<Option<Arc<WorkerThread>>> = RefCell::new(None);
}
```

WorkerThread 对应工作线程的数据结构，在 WorkerThread 中保存了指向 Registry 的指针以及 WorkerThread 在线程池中的 index。ThreadInfo 通过 deque crate 中实现的无锁双端队列提供了相应的接口。thread_local! 宏是 Rust 中用于创建线程本地存储的宏，通过 thread_local! 宏可以使得每个工作线程方便地拥有独立的 WorkerThread 实例。

```Rust
impl WorkerThread {
    pub fn push(&self, job: Job) {
        self.registry.thread_infos[self.index].worker.push(job);
    }

    pub fn pop(&self) -> Option<Job> {
        self.registry.thread_infos[self.index].worker.pop()
    }

    pub fn steal_until(&self, latch: Arc<Latch>) {
        while !latch.probe() {
            if let Some(mut job) = steal_work(self.registry.clone(), self.index) {
                job.execute();
            } else {
                thread::yield_now();
            }
        }
    }
}
```

WorkerThread 提供的主要函数包括 push、pop 和 steal_until。push 用于工作线程在执行任务的过程中，如果任务产生了新的子任务，可以通过 push 函数将任务插入到工作队列中，pop 函数用在工作队列中弹出一个任务，由于 join 原语往往会一次提交两个任务，并且需要同时返回两个任务的结果，所以 steal_until 函数用于在第二个任务被其他工作线程窃取时，等待该任务被其他工作线程执行完成的期间去其他工作线程窃取任务执行。
### 4.5 接口设计与实现

#### 4.5.1 join

在完成了上一小节中，我们对于整个 mini-rayon 框架的核心数据结构抽象后，我们就可以通过这些数据结构完成 join 语言的接口设计与实现。

```Rust
pub fn join<A, RA, B, RB>(oper_a: A, oper_b: B) -> (RA, RB)
where
    A: FnOnce() -> RA + Send + 'static,
    B: FnOnce() -> RB + Send + 'static,
    RA: Send + 'static,
    RB: Send + 'static,
{
    let worker_thread = WorkerThread::current();
    if worker_thread.is_none() {
        return join_inject(oper_a, oper_b);
    }
    let worker_thread = worker_thread.unwrap();
    let mut result_b = None;
    let code_b = Code::new(oper_b, &mut result_b as *mut Option<RB>);
    let latch_b = Arc::new(Latch::new());
    let job_b = Job::new(Box::new(code_b), latch_b.clone());
    worker_thread.push(job_b);
    let result_a = oper_a();
    if let Some(mut job_b) = worker_thread.pop() {
        job_b.execute();
    } else {
        worker_thread.steal_until(latch_b);
    }
    (result_a, result_b.unwrap())
}

fn join_inject<A, RA, B, RB>(oper_a: A, oper_b: B) -> (RA, RB)
where
    A: FnOnce() -> RA + Send + 'static,
    B: FnOnce() -> RB + Send + 'static,
    RA: Send + 'static,
    RB: Send + 'static,
{
    let mut result_a = None;
    let code_a = Code::new(oper_a, &mut result_a as *mut Option<RA>);
    let latch_a = Arc::new(Latch::new());
    let job_a = Job::new(Box::new(code_a), latch_a.clone());
    let mut result_b = None;
    let code_b = Code::new(oper_b, &mut result_b as *mut Option<RB>);
    let latch_b = Arc::new(Latch::new());
    let job_b = Job::new(Box::new(code_b), latch_b.clone());
    get_registry().inject(vec![job_a, job_b]);
    latch_a.wait();
    latch_b.wait();
    (result_a.unwrap(), result_b.unwrap())
}
```

join 函数接受两个满足 FnOnce() -> R + Send + 'static trait bound 的闭包作为参数，需要满足 Send trait 的原因在于闭包与返回值都需要在工作线程之间传递，并由某一个工作线程执行其逻辑，'static 约束了闭包与返回值的生命周期，因为在编译的过程中，无法明确地知道闭包和返回值和会何时使用，所以出于简化的目的，避免引入声明生命周期过多的复杂性，直接声明闭包与返回值需要满足 'static 生命周期。

> 事实上，'static 生命周期并不意味着一定需要程序运行过程中全局可用，'static 实际上是希望告诉编译器，开发者可以保证这个生命周期是能够满足的。在 mini-rayon 的场景中，编译器并不能自己判断出这一点，其本质原因在于 mini-rayon 是一个运行时框架，任务的执行是动态的，其生命周期是隐含的保证。

在 join 函数中，我们会首先判断执行 join 函数的线程是否是工作线程，如果调用 join 函数是 mini-rayon 的使用线程，而非工作线程，则会通过 join_inject 函数将两个任务包装后插入到 Registry 中，get_registry 函数会获取 mini-rayon 的全局 Registry。执行 join 函数的工作线程，会首先执行第一个任务，并将第二个任务插入到工作队列中，在执行第一个任务的过程中，如果第二个任务被其他工作线程窃取，则会触发并行加速，如果其他工作线程都处于忙碌状态，无人窃取任务，那么 join 函数就会退化至顺序执行两个任务，从而实现了潜在并行性的语义。
#### 4.5.2 thread pool

在上述 join 原语接口的设计中， 由于全局 Registry 存在冷启动的情况，所以 mini-rayon 通过 ThreadPool 数据结构封装了单独的 Registry，这一设计可以实现对 Registry 细粒度的控制，通过 install 接口提交任务。

```Rust
pub struct ThreadPool {
    registry: Arc<Registry>,
}

impl ThreadPool {
    pub fn new() -> ThreadPool {
        let registry = Registry::new();
        registry.wait_until_primed();
        ThreadPool {
            registry,
        }
    }

    pub fn install<OP, R>(&self, op: OP) -> R
    where
        OP: FnOnce() -> R + Send + 'static,
        R: Send + 'static,
    {
        let mut result = None;
        let code = Code::new(op, &mut result as *mut Option<R>);
        let latch = Arc::new(Latch::new());
        let job = Job::new(Box::new(code), latch.clone());
        self.registry.inject(vec![job]);
        latch.wait();
        result.unwrap()
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        self.registry.terminate();
    }
}
```
### 4.6 Rust feature 带来的优势

Rust 作为一种系统级编程语言，提供了强大的抽象能力，在编译时将这些抽象转换为高效的机器码，通过零成本抽象避免运行时开销的引入。Rust feature 为 mini-rayon 带来了一些天然的优势，下面会展开介绍如何通过 Rust feature 实现最小化任务推送开销以及避免并行编程中常见的数据竞争错误。
#### 4.6.1 最小化任务推送开销

在之前提到的任务调度过程中，为了充分发挥 join 原语带来的性能提升，避免并行抽象带来的额外成本，减少任务推送开销无疑是非常重要的。这里有两个主要原因，首先每次调用 join 都需要将其中的一个任务推送到本地队列，其次事实上处理器的数量很可能远远少于任务的数量，在这种情况下，大多数任务将不会被窃取，任务推送开销在此时变成了主要的并行抽象成本。

为了减少任务推送开销，mini-rayon 利用了诸多 Rust feature：

* join 原语时根据其参数的闭包类型进行通用定义的，这意味着 Rust 的单态化会在每个 join 原语的调用处生成不同的 join 原语实现副本。当 join 调用 oper_a() 和 oper_b() 时（在不被窃取的情况下），这些调用是静态分配的，它们可以进行内联，而无需分配空间创建闭包。
* 因为 join 原语会阻塞直至两个闭包完成，在这个过程中可以充分利用栈中的分配，完全避免堆的分配（例如，我们放入本地工作队列的闭包对象是在栈上分配的）。

通过上述方式，Rust 实现的推送任务开销实际上已经足够低，此外，还有一些方法可以进一步降低：

* 许多工作窃取实现使用启发式方法来尝试决定何时跳过推送并行任务的工作。例如，一些策略尝试完全避免推送任务，除非存在可能窃取任务的空闲工作线程。
* 通过一些经典的方式进一步优化 join 原语所生成的汇编代码。
#### 4.6.2 避免数据竞争

mini-rayon 通过 Rust feature 可以使得在向顺序代码添加并行性时，轻松地避免数据竞争，从而防止引入一些难以解释的并发错误。

```Rust
fn quick_sort<T:PartialOrd+Send>(v: &mut [T]) {
    if v.len() > 1 {
        let mid = partition(v);
        let (lo, hi) = v.split_at_mut(mid);
        mini_rayon::join(|| quick_sort(lo),
                         || quick_sort(lo));
    }
}
```

首先，join 原语的两个闭包可能共享一些可变状态，因此其中一个闭包所做的更改可能会影响另一个闭包。例如在之前的快排实现实例中错误地调用 lo 上的 quick_sort 时，这将无法通过 Rust 的编译。Rust 使得这类错误成为编译错误，而不是难以解决的崩溃错误。

```Rust
fn share_rc<T:PartialOrd+Send>(rc: Rc<i32> {
    mini_rayon::join(|| something(rc.clone()),
                     || something(rc.clone()));
}
```

其次，另一个常见的错误是在闭包中使用非线程安全的数据类型。例如在上面的例子中使用了在线程之间共享不安全的 Rc 数据类型。这同样无法通过 Rust 的编译，这是由于 join 原语将闭包声明为 Send。Send 指示数据是否可以安全地跨线程传输。join 原语将两个闭包声明为 Send，实际上是在指示编译器在这些闭包中访问的数据必须能够安全地在线程中相互传输。

换言之，Rust 自身的所有权、借用、多线程机制避免了数据竞争，而 mini-rayon 框架得益于这些 Rust feature，同样使得在修改顺序代码时避免引入过多的心智负担。
## 五、构建在 join 之上的并行迭代器

上一章节详细介绍了 mini-rayon 中的 join 原语实现细节，mini-rayon 所提供的并行迭代器实际上就是构建在 join 原语之上的包装。因为 join 原语封装了所有的不安全性，并行迭代器不需要处理任何与并行性相关的 unsafe 代码。

```Rust
pub trait ParallelIterator {
    type Item;
    type Shared: Sync;
    type State: ParallelIteratorState<Shared=Self::Shared, Item=Self::Item> + Send;
	
    fn state(self) -> (Self::Shared, Self::State);
}
```

上述代码是并行迭代器 trait ParallelIterator 的核心部分。state 方法将迭代器划分为一些共享状态和一些单一线程的状态。共享状态可能会被所有工作线程访问，因此它必须是 Sync 的，需要跨线程共享。单一线程的状态只需要是 Send 的，支持传输到其他线程。

```Rust
pub trait ParallelIteratorState: Sized {
    type Item;
    type Shared: Sync;

    fn len(&mut self) -> ParallelLen;

    fn split_at(self, index: usize) -> (Self, Self);

    fn for_each<OP>(self, shared: &Self::Shared, op: OP)
    where OP: FnMut(Self::Item);
}
```

ParallelteratorState trait 表示剩余工作的一部分，例如要处理的子数据切片。他包含了三个方法：len 方法给出了剩余工作量的概念；split_at 方法将这个状态分为分为另外两个部分；for_each 方法生成该迭代器中的所有值。因此对于一个 slice &[T] 所对应的并行迭代器，len 方法返回 slice 的长度，split_at 方法将 slice 分为两个子 slice，for_each 方法迭代数组并对每个元素调用 op。

```Rust
fn process(shared, state) {
    if state.len() is too big {
        let midpoint = state.len() / 2;
        let (state1, state2) = state.split_at(midpoint);
        rayon::join(|| process(shared, state1), || process(shared, state2));
    } else {
        state.for_each(|item| {
            // process item
        })
    }
}
```

通过 ParallelIterator trait 和 ParallelteratorState trait，我们可以遵循相同的模版来实现一些并行操作。例如上面的这个 process 伪代码函数，首先检查有多少工作，如果太多，就分为两部分，否则，就按顺序处理。
## 六、框架性能测试

我们在之前实现的基础上对 mini-rayon 进行性能测试，测试对象包括顺序计算的快速排序、通过 spawn 新线程实现的并行快速排序、通过 threadpool crate 使用线程池的并行快速排序和通过 mini-rayon 实现的并行快速排序，根据不同方式实现的快速排序如下面的代码所示。性能测试主机配置为 20 CPU 核心 32G 内存。

```Rust
fn partition<T>(v: &mut [T]) -> usize
where
    T: PartialOrd + Send + 'static
{
    let pivot = v.len() - 1;
    let mut i = 0;
    for j in 0..pivot {
        if v[j] <= v[pivot] {
            v.swap(i, j);
            i += 1;
        }
    }
    v.swap(i, pivot);
    i
}

fn quick_sort<T>(v: &mut [T])
where
    T: PartialOrd + Send + 'static
{
    if v.len() <= 1 {
        return;
    }
    let mid = partition(v);
    let (lo, hi) = v.split_at_mut(mid);
    quick_sort(lo);
    quick_sort(hi);
}

fn quick_sort_parallel_by_spawn<T>(v: &'static mut [T])
where
    T: PartialOrd + Send + 'static
{
    if v.len() <= 200 {
        quick_sort(v);
        return;
    }
    let mid = partition(v);
    let (lo, hi) = v.split_at_mut(mid);
    let lo_handle = thread::spawn(|| quick_sort_parallel_by_spawn(lo));
    let hi_handle = thread::spawn(|| quick_sort_parallel_by_spawn(hi));
    lo_handle.join().unwrap();
    hi_handle.join().unwrap();
}

fn quick_sort_parallel_by_threadpool<T>(pool: &ThreadPool, v: &'static mut [T])
where
    T: PartialOrd + Send + 'static
{
    if v.len() <= 200 {
        quick_sort(v);
        return;
    }
    let mid = partition(v);
    let (lo, hi) = v.split_at_mut(mid);
    pool.execute(|| quick_sort_parallel_by_spawn(lo));
    pool.execute(|| quick_sort_parallel_by_spawn(hi));
    pool.join();
}

fn quick_sort_parallel_by_mini_rayon<T>(v: &'static mut [T])
where
    T: PartialOrd + Send + 'static
{
    if v.len() <= 200 {
        quick_sort(v);
        return;
    }
    let mid = partition(v);
    let (lo, hi) = v.split_at_mut(mid);
    join(|| quick_sort_parallel_by_mini_rayon(lo), || quick_sort_parallel_by_mini_rayon(hi));
}
```

测试结果如下图所示，测试中我们分别使用了 10^6 和 10^8 两种数量级大小的随机数组进行排序测试。在两种数量级情况下，mini-rayon 都取得了最好的结果，且相比于顺序计算可以获得 5-7 倍的性能提升，这一性能提升无疑是非常显著的。此外，无论是在直接创建新线程或使用线程池的情况下并行化快速排序均性能表现不佳，在这里猜测主要原因在于直接创建新线程的情况下，由于任务粒度不大，频繁创建和销毁新线程严重影响了性能表现，即使是使用线程池，由于缺乏任务窃取机制，线程在使用的过程中同步成本过大，线程的利用率不高，即使完成了任务也需要在等待其他线程完成任务的过程中处于无法复用的状态，从而导致了相比于直接创建新线程仅能获得一点性能提升。

![](1.png)

> 在通过 threadpool crate 使用线程池的并行快速排序实现中，我们使用了 pool.join 函数，该函数实际上需要完成整个线程池的同步，而快速排序只需完成函数内两个闭包的同步即可返回，这一点是性能表现不佳的另一个原因。事实上，为了解决这个问题，我们无法通过 threadpool crate 提供的接口同步两个闭包，即使使用 channel 进行同步的情况下，由于引入了新的 channel 开销，根据实际测试表明在这个场景下性能表现也不会有明显的性能提升。另外，另一个常见的 crossbeam crate 提供了 scope 函数，支持创建作用域，在作用域内启动并行任务，并等待作用域内任务完成即可返回，但此函数仍会创建新的线程，难以有显著的性能提升。因此，即使是一个快速排序简单的测试场景，实际上已经可以证明 mini-rayon 和 Rayon 在数据并行计算加速中存在的价值。此外，测试中均保证了排序结果的有效性，测试代码位于 https://github.com/zjregee/mini/blob/main/mini-rayon/examples/quick_sort.rs
## 参考资料

* https://smallcultfollowing.com/babysteps/blog/2015/12/18/rayon-data-parallelism-in-rust/
* https://github.com/nikomatsakis/rayon/
* https://github.com/rayon-rs/rayon