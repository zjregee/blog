---
title: Golang 源码阅读笔记
slug: go
date: 2023-11-27 00:00:00+0000
categories:
    - 学习笔记系列
tags:
    - Go
---

> 该笔记基于 Golang 1.21.6。
> 
> 源码地址：https://github.com/golang/go
## 一、context

context 包提供了在 Goroutines 之间传递取消信号、超时和截止时间的一种方式，以及存储和检索请求处理相关数据的机制，是 Golang 中用于处理请求范围内数据和控制请求生命周期的重要工具。Context 接口定义了基本的上下文操作：Deadline 方法返回一个上下文被取消的时间，即完成工作的截止时间；Done 方法返回一个 Channel。这个 Channel 会在当前工作完成或上下文被取消后关闭，多次调用 Done 方法会返回同一个 Channel；Err 方法返回上下文结束的原因，它只会在 Done 方法对应的 Channel 关闭时返回非空值，如果上下文被取消返回 Canceled 错误，如果上下文超时返回 DeadlineExceeded 错误；Value 方法从上下文中获取键对应的值，对于同一个上下文来说，多次调用 Value 方法并传入相同的键会返回相同的结果，该方法可以用来传递特定的数据。

```Go
package context

type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

context 包的最大用处在于在 Goroutines 构成的树形结构中同步信号以减少计算资源的浪费。上下文可以从最顶层的 Goroutines 逐层传递到最底层，以便在上层的 Goroutines 执行出现错误时将信号即时同步给下层。一个常见的使用场景是多个 Goroutines 同时订阅上下文 Done 方法 Channel 中的消息，一旦接受到取消信号就立刻停止当前正在执行的工作。
### 1.1 默认上下文与私有结构体

context 包中有两个常用的 Background 和 TODO 函数，它们分别用于创建根上下文和未来可能会用到但目前还没有具体值的上下文。这两者返回的上下文均基于内部的私有结构体 emptyCtx，它通过空方法实现了 Context 接口的所有方法，没有任何实际功能。具体来说，backgroundCtx 是上下文的默认值，其他所有上下文都应该从它衍生出来，todoCtx 应该在不确定使用哪种上下文时使用，或仅作为一种占位符存在。

```Go
package context

type emptyCtx struct{}

func (emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return
}

func (emptyCtx) Done() <-chan struct{} {
    return nil
}

func (emptyCtx) Err() error {
    return nil
}

func (emptyCtx) Value(key any) any {
    return nil
}

type backgroundCtx struct{ emptyCtx }

func (backgroundCtx) String() string {
    return "context.Background"
}

type todoCtx struct{ emptyCtx }

func (todoCtx) String() string {
    return "context.TODO"
}

func Background() Context {
    return backgroundCtx{}
}

func TODO() Context {
    return todoCtx{}
}
```

除了这两个简单的默认上下文，context 包还提供了诸多派生上下文，派生上下文用于在已有的上下文中创建一个新的上下文，新上下文继承了原始上下文的一些信息，同时可以添加或修改一些新的属性。派生上下文通常通过 WithXXX 函数实现，例如 WithCancel、WithTimeout、WithDeadline 等，每种派生上下文的使用场景和实现原理会在后面的内容中展开。
### 1.2 派生上下文之传值

```Go
package context

func WithValue(parent Context, key, val any) Context { ... }

type valueCtx struct {
    Context
    key, val any
}

func (c *valueCtx) Value(key any) any { ... }

func value(c Context, key any) any { ... }
```

WithValue 函数接受一个父上下文，以及一个 KV 对，返回一个新的上下文。这个新的上下文包含了父上下文的所有信息，并在其中添加了 KV 对数据。valueCtx 的实现原理较为简单，在父上下文的基础上额外增加了 key、val 字段用于保存 KV 对数据。此处值得一提的是 value 函数，value 函数会通过循环从该上下文从上搜索直至到上下文树顶部，在过程中不断根据上下文的类型进行匹配，处理不同类型的上下文。当前上下文是 valueCtx 时，如果键匹配则返回对应的值，否则继续往上层上下文查找；当前上下文是 cancelCtx 时，如果键是 &cancelCtxKey，则返回当前上下文，否则继续往上层上下文查找；当前上下文是 withoutCancelCtx 时，如果键是 &cancelCtxKey，则返回当前上下文，否则继续往上层上下文查找；当前上下文是 timerCtx 时，如果键是 &cancelCtxKey，则返回该上下文关联的 cancelCtx，否则继续往上层上下文查找；当前上下文是 backgroundCtx 或 todoCtx 时，直接返回 nil；对于其他上下文，直接调用上下文的 Value 方法。这里只陈述搜索逻辑，因为尚未提到其他上下文，不具体解释其中的一些行为。
### 1.3 派生上下文之取消

```Go
package context

var cancelCtxKey int

type CancelFunc func()
type CancelCauseFunc func(cause error)

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) { ... }
func WithCancelCause(parent Context) (ctx Context, cancel CancelCauseFunc) {...}

func Cause(c Context) error { ... }

type canceler interface {
    cancel(removeFromParent bool, err, cause error)
    Done() <-chan struct{}
}

type cancelCtx struct {
    Context
    mu       sync.Mutex
    done     atomic.Value
    children map[canceler]struct{}
    err      error
    cause    error
}
```

WithCancel 函数返回一个父上下文的拷贝上下文，新上下文使用新的 Done channel，新的 Done channel 当 WithCancel 函数返回的 cancel 函数被调用或者父上下文的 Done channel 被关闭时关闭。WithCancelCause 函数类似，但会额外设置取消原因，这个取消原因可以在被取消的上下文和其派生上下文中通过 Cause 函数来获取，WithCancelCause 函数返回的 cancel 函数只有在上下文还没有取消时可以取消以及设置取消原因。

WithCancel 和 WithCancelCause 函数返回的上下文均基于内部的私有结构体 cancelCtx，cancelCtx 各字段的作用包括：Context 用于记录父上下文；mu 字段用于保护其他字段的并发访问；done 字段是一个支持原子操作的值容器，用于记录 cancelCtx 的 Done Channel；children 字段是一个存储实现 canceler 接口的子上下文的集合，用于在取消该上下文时，取消其他子上下文，一个实现了 canceler 接口的上下文支持通过 cancel 方法立刻取消该上下文，cancelCtx 和在之后会涉及到的 timerCtx 均实现了该接口；err 字段用于记录该上下文的取消原因或错误状态；cause 字段用于记录该上下文取消的根本原因，即该原因可能是上层上下文传递下来的或是一些底层的错误信息，Cause 函数的实现原理即通过该上下文的 Value 函数循环向上层上下文寻找 cancelCtx，并返回该 cancelCtx 的 cause 字段信息。此外，cancelCtx 的诸多字段会通过惰性创建，在使用的时候进行初始化。

WithCancel 和 WithCancelCause 函数的实现类似，通过创建一个 cancelCtx，并调用 cancelCtx 的 propagateCancel 方法设置该上下文的父上下文，以便在父上下文被取消时取消该上下文，并返回封装了 cancelCtx 的 cancel 方法的 cancel 函数。propagateCancel 和 cancel 方法是 cancelCtx 的核心方法，在展开 propagateCancel 方法的完整实现逻辑之前，需要首先介绍 context 包中另外两个私有结构体 afterFuncCtx 和 stopCtx 以及工具函数 parentCancelCtx 和 removeChild。

```Go
package context

type afterFuncCtx struct {
    cancelCtx
    once sync.Once
    f    func()
}

type stopCtx struct {
    Context
    stop func() bool
}
```

afterFuncCtx 的目的是实现一个带有延迟执行函数的上下文，它包含了一个函数 f，该函数会在该上下文取消时执行，且通过 once 字段确保只执行一次，事实上这个 once 字段还可以保证最多执行一次。stopCtx 被用于当一个带有延迟执行函数的上下文需要注册 cancelCtx 子上下文时，作为该 cancelCtx 上下文的父上下文。stopCtx 相当于一个中间上下文，原本没有 stopCtx 的情况下，在一个带有延迟执行函数的上下文中注册 cancelCtx 子上下文时，cancelCtx 的父上下文则为这个带有延迟执行函数的上下文。stopCtx 提供了取消上下文并注销注册的能力，通过 stop 字段记录的函数可以使得当这个带有延迟执行函数的上下文被取消需要执行延迟执行函数的时候不再执行该 cancelCtx 的 cancel 方法。

```Go
package context

func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    done := parent.Done()
    if done == closedchan || done == nil {
        return nil, false
    }
    p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
    if !ok {
        return nil, false
    }
    pdone, _ := p.done.Load().(chan struct{})
    if pdone != done {
        return nil, false
    }
    return p, true
}

func removeChild(parent Context, child canceler) {
    if s, ok := parent.(stopCtx); ok {
        s.stop()
        return
    }
    p, ok := parentCancelCtx(parent)
    if !ok {
        return
    }
    p.mu.Lock()
    if p.children != nil {
        delete(p.children, child)
    }
    p.mu.Unlock()
}
```

parentCancelCtx 函数用于返回父上下文内置的 cancelCtx，removeChild 函数用于从父上下文中移除某个实现 canceler 接口的子上下文，当父上下文是 stopCtx 时，调用 stopCtx 记录的 stop 函数注销子上下文在父上下延迟执行函数中的注册，如果父上下文内置的是 cancelCtx，则将子上下文从父上下文中的 children 字段中删除。值得一提的是，当父上下文是 stopCtx 时，不需要将子上下文从实际父上下文的 children 字段删除，因为子上下文的 cancel 方法通过延迟函数的函数调用行为注册，而没有通过 children 字段注册。

```Go
package context

func AfterFunc(ctx Context, f func()) (stop func() bool) {
    a := &afterFuncCtx{
        f: f,
    }
    a.cancelCtx.propagateCancel(ctx, a)
    return func() bool {
        stopped := false
        a.once.Do(func() {
            stopped = true
        })
        if stopped {
            a.cancel(true, Canceled, nil)
        }
        return stopped
    }
}

type afterFuncer interface {
    AfterFunc(func()) func() bool
}

func (a *afterFuncCtx) cancel(removeFromParent bool, err, cause error) {
    a.cancelCtx.cancel(false, err, cause)
    if removeFromParent {
        removeChild(a.Context, a)
    }
    a.once.Do(func() {
        go a.f()
    })
}
```

这里需要进一步解释带有延迟执行函数的上下文实现原理。AfterFunc 函数用于在上下文中注册一个函数，这个函数会在该上下文被取消时执行这个函数，同时会返回一个 stop 函数，该函数可以在上下文被取消之前放弃在上下文被取消时执行这个函数。AfterFunc 函数的实现依赖于已经提到的 afterFuncCtx，它创建一个中间 cancelCtx，并通过中间 cancelCtx 的 propagateCancel 方法该上下文中注册 afterFuncCtx 子上下文，当该上下文被取消时，会调用 afterFuncCtx 子上下文的 cancel 方法，从而实现在原上下文中增加一个延迟执行函数。当在原上下文返回之前调用 AfterFunc 函数返回的 stop 函数时，会提前取消 afterFuncCtx 上下文，并将其从原上下文中注销注册。

```Go
package context

var closedchan = make(chan struct{})

func init() {
    close(closedchan)
}

func (c *cancelCtx) cancel(removeFromParent bool, err, cause error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    if cause == nil {
        cause = err
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return
    }
    c.err = err
    c.cause = cause
    d, _ := c.done.Load().(chan struct{})
    if d == nil {
        c.done.Store(closedchan)
    } else {
        close(d)
    }
    for child := range c.children {
        child.cancel(false, err, cause)
    }
    c.children = nil
    c.mu.Unlock()
    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```

cancelCtx 的 cancel 方法用于关闭 Done Channel 取消该上下文，并且会通过 child 字段调用每个子上下文的 cancel 方法取消子上下文，并通过 removeChild 函数将自己从父上下文中删去，过程中会通过 closedchan 实现复用。

```Go
package context

var goroutines atomic.Int32

func (c *cancelCtx) propagateCancel(parent Context, child canceler) {
    c.Context = parent
    done := parent.Done()
    if done == nil {
        return
    }
    select {
    case <-done:
        child.cancel(false, parent.Err(), Cause(parent))
        return
    default:
    }
    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if p.err != nil {
            child.cancel(false, p.err, p.cause)
        } else {
            if p.children == nil {
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
        return
    }
    if a, ok := parent.(afterFuncer); ok {
        c.mu.Lock()
        stop := a.AfterFunc(func() {
            child.cancel(false, parent.Err(), Cause(parent))
        })
        c.Context = stopCtx{
            Context: parent,
            stop: stop,
        }
        c.mu.Unlock()
        return
    }
    goroutines.Add(1)
    go func() {
        select {
            case <-parent.Done():
                child.cancel(false, parent.Err(), Cause(parent))
            case <-child.Done():
        }
    }()
}
```

在梳理了上述内容之后，最核心的 propagteCancel 方法的实现变得十分清晰。在注册子上下文至父上下文时，如果父上下文内置了 cancelCtx，则直接将自己的 cancel 方法通过父上下文的 children 字段注册，如果父上下文实现了 afterFuncer 接口，则通过父上下文的 AfterFunc 方法以父上下文延迟执行函数的形式注册，并通过中间上下文 stopCtx，保存注销注册的函数，如果父上下文不属于这两类，则创建一个 Goroutine 通过 select 来监听父上下文和子上下文的取消情况。

```Go
package context

func WithoutCancel(parent Context) Context { ... }

type withoutCancelCtx struct {
    c Context
}

func (withoutCancelCtx) Deadline() (deadline time.Time, ok bool) {
    return
}

func (withoutCancelCtx) Done() <-chan struct{} {
    return nil
}

func (withoutCancelCtx) Err() error {
    return nil
}

func (c withoutCancelCtx) Value(key any) any {
    return value(c, key)
}

func (c withoutCancelCtx) String() string {
    return contextName(c.c) + ".WithoutCancel"
}
```

WithoutCancel 函数提供了一个父上下文的拷贝，并没有继续维持上下文中之间的取消能力，当父上下文取消时，子上下文不会取消。
### 1.4 派生上下文之超时

```Go
package context

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) { ... }
func WithDeadlineCause(parent Context, d time.Time, cause error) (Context, CancelFunc) { ... }
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) { ... }
func WithTimeoutCause(parent Context, timeout time.Duration, cause error) (Context, CancelFunc) { ... }
```

WithDeadline、WithDeadlineCause、WithTimeout 和 WithTimeoutCause 函数提供了一个带有截止时间的上下文，该上下文会集成父上下文的特性，并在截止时间到达时，取消该上下文。

```Go
package context

func WithDeadlineCause(parent Context, d time.Time, cause error) (Context, CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        return WithCancel(parent)
    }
    c := &timerCtx{
        deadline: d,
    }
    c.cancelCtx.propagateCancel(parent, c)
    dur := time.Until(d)
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded, cause)
        return c, func() { c.cancel(false, Canceled, nil) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded, cause)
        })
    }
    return c, func() { c.cancel(true, Canceled, nil) }
}

type timerCtx struct {
    cancelCtx
    timer *time.Timer
    deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
    return c.deadline, true
}

func (c *timerCtx) cancel(removeFromParent bool, err, cause error) {
    c.cancelCtx.cancel(false, err, cause)
    if removeFromParent {
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

WithDeadlineCause 函数通过内部的私有结构体 timerCtx 实现，timerCtx 依赖 cancelCtx 实现大部分功能，实现逻辑较为简单。当创建一个 timerCtx 时，根据截至时间和 time.AfterFunc 函数创建一个延迟执行函数，当时间截止后，调用 timerCtx 的 cancel 方法，此外，也可以通过 WIthDeadlineCause 函数返回的 cancel 函数提前调用 timerCtx 的 cancel 方法。
## 二、errors
### 2.1 基本定义

error 是 builtin 包内置的一个接口，定义了表示错误的基本类型，error 接口只有一个方法，nil 表示没有错误。

```Go
package builtin

type error interface {
    Error() string
}
```

New 方法返回一个包含 string 的 error 实现 errorString。

```Go
package errors

func New(text string) error {
	return &errorString{text}
}

type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}

var ErrUnsupported = New("unsupported operation")
```
### 2.2 错误链

为了提供关于错误的额外上下文，在错误处理过程中实现更复杂的操作，通过 errors 包提供的相关函数以及 fmt 包提供的 %w 标志，可以简单的将错误包裹在另一个错误中，形成清晰的错误处理链条。

Unwrap 函数用于获取错误链中的下一个错误，可以通过循环来迭代整个错误链。Is 函数用于比较错误链中的错误，判断某个特定类型的错误是否存在于错误莲中，这种方式比直接使用 == 更加令灵活，因为它不仅会检查最外层的错误。As 函数用于将错误断言为某个接口类型，并将该类型的值存储在目标变量中，这对于处理实现了特定接口的错误很有用。Is 和 As 函数都都可以以整个错误链为对象进行递归操作，提供对错误链的原生支持。

```Go
package errors

func Unwrap(err error) error {
    u, ok := err.(interface {
        Unwrap() error
    })
    if !ok {
        return nil
    }
    return u.Unwrap()
}

func Is(err, target error) bool {
    if target == nil {
        return err == target
    }
    isComparable := reflectlite.TypeOf(target).Comparable()
    return is(err, target, isComparable)
}

func is(err, target error, targetComparable bool) bool {
    for {
        if targetComparable && err == target {
            return true
        }
        if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
            return true
        }
        switch x := err.(type) {
        case interface{ Unwrap() error }:
            err = x.Unwrap()
            if err == nil {
                return false
            }
        case interface{ Unwrap() []error }:
            for _, err := range x.Unwrap() {
                if is(err, target, targetComparable) {
                    return true
                }
            }
            return false
        default:
            return false
        }
    }
}

func As(err error, target any) bool {
    if err == nil {
        return false
    }
    if target == nil {
        panic("errors: target cannot be nil")
    }
    val := reflectlite.ValueOf(target)
    typ := val.Type()
    if typ.Kind() != reflectlite.Ptr || val.IsNil() {
        panic("errors: target must be a non-nil pointer")
    }
    targetType := typ.Elem()
    if targetType.Kind() != reflectlite.Interface && !targetType.Implements(errorType) {
        panic("errors: *target must be interface or implement error")
    }
    return as(err, target, val, targetType)
}

func as(err error, target any, targetVal reflectlite.Value, targetType reflectlite.Type) bool {
    for {
        if reflectlite.TypeOf(err).AssignableTo(targetType) {
            targetVal.Elem().Set(reflectlite.ValueOf(err))
            return true
        }
        if x, ok := err.(interface{ As(any) bool }); ok && x.As(target) {
            return true
        }
        switch x := err.(type) {
        case interface{ Unwrap() error }:
            err = x.Unwrap()
            if err == nil {
                return false
            }
        case interface{ Unwrap() []error }:
            for _, err := range x.Unwrap() {
                if err == nil {
                    continue
                }
                if as(err, target, targetVal, targetType) {
                    return true
                }
            }
            return false
        default:
            return false
        }
    }
}

var errorType = reflectlite.TypeOf((*error)(nil)).Elem()
```
### 2.3 错误合并

Join 方法用于将数个 error 合并成一个 error，其具体实现通过返回一个包含 error 数组的 error 实现 joinError。joinError 实现的 Error 方法会将内部 error 数组的多个 error 所提供的错误信息通过换行符连接，Unwrap 方法用于将 joinError 方便地纳入错误链之中，在上一小节中错误链提供的方法会原生支持 joinError。

```Go
package errors

func Join(errs ...error) error {
    n := 0
    for _, err := range errs {
        if err != nil {
            n++
        }
    }
    if n == 0 {
        return nil
    }
    e := &joinError{
        errs: make([]error, 0, n),
    }
    for _, err := range errs {
        if err != nil {
            e.errs = append(e.errs, err)
        }
    }
    return e
}

type joinError struct {
    errs []error
}

func (e *joinError) Error() string {
    if len(e.errs) == 1 {
        return e.errs[0].Error()
    }
    b := []byte(e.errs[0].Error())
    for _, err := range e.errs[1:] {
        b = append(b, '\n')
        b = append(b, err.Error()...)
    }
    return unsafe.String(&b[0], len(b))
}

func (e *joinError) Unwrap() []error {
    return e.errs
}
```

## 三、bufio

bufio 包包装了 io.Reader 和 io.Writer 对象，并提供了带缓冲的 IO 操作，用以提高 IO 操作的性能。该包包含了一些用于读写数据的缓冲区，以减少系统调用的次数，从而提高效率。
### 3.1 Reader

```Go
package bufio

type Reader struct {
    buf          []byte
    rd           io.Reader
    r, w         int
    err          error
    lastByte     int
    lastRuneSize int
}

func NewReaderSize(rd io.Reader, size int) *Reader { ... }
func NewReader(rd io.Reader) *Reader { ... }
func (b *Reader) Size() int { ... }
func (b *Reader) Reset(r io.Reader) { ... }
func (b *Reader) Peek(n int) ([]byte, error) { ... }
func (b *Reader) Discard(n int) (int, error) { ... }
func (b *Reader) Read(p []byte) (int, error) { ... }
func (b *Reader) ReadByte() (byte, error) { ... }
func (b *Reader) UnreadByte() error { ... }
func (b *Reader) ReadRune() (rune, int, error) { ... }
func (b *Reader) UnreadRune() { ... }
func (b *Reader) Buffered() int { .. }
func (b *Reader) ReadSlice(delim byte) ([]byte, error) { ... }
func (b *Reader) ReadLine() ([]byte, bool, error) { ... }
func (b *Reader) ReadBytes(delim byte) ([]byte, error) { ... }
func (b *Reader) ReadString(delim byte) (string, error) { ... }
func (b *Reader) WriteTo(w io.Writer) (int64, error) { ... }
```

在 Reader 结构体中，buf 用于存储数据的字节切片，在读取时将数据存储在这个缓冲区中，以减少对底层 io.Reader 的实际读取次数，rd 用于存储 Reader 使用的底层数据源，它是一个实现了 io.Reader 接口的对象，r、w 用于跟踪缓冲区的读写状态，lastByte 用于存储最后一个读取的字节，lastRuneSize 用于记录最后读取的一个 Unicode 字符的大小。通过 lastByte 和 lastRuneSize 可以将最后一个字节或 Unicode 字符放回缓冲区，以便后续的读取操作能够重新读取该字节。

Peek 方法在不消费数据的情况下返回接下来 n 个字节的数据，当 n 大于缓存 Reader 缓存区大小时，则最多返回缓存区大小的数据。Discard 方法跳过接下来 n 个字节的数据，不同于 Peek 方法，在不发生错误的前提下，Discard 方法一定会跳过 n 个字节的数据。Read 方法读取 n 个字节至指定数组中，在 Reader 中存在缓存数据时，会直接将缓存的数据拷贝至指定数组，且不额外读取新的数据，当 Reader 中不存在缓存数据时，当指定数组的字节大小大于 Reader 的缓冲区大小时，直接调用内部 io.Reader 提供的方法读取一次数据至指定数组，避免内存拷贝开销，当指定数组的字节大小小于 Reader 的缓冲区大小时，调用内部 io.Reader 提供的方法读取一次数据至缓冲区，并将缓冲区的数据再次拷贝至指定数组。值得一提的是，Read 方法最多只会调用内部 io.Reader 的接口读取一次，不保障一定能读出数据，而之前提到的 Peek 方法和 Discard 方法由于内部使用了循环，至少保障能读出一次数据（若循环次数大于内部设定的值，则也无法读出数据）。

ReadByte 和 ReadRune 方法类似，在不发生错误的前提下，循环填充缓冲区，以至能够缓冲区中的数据能够拷贝返回一个字节的数据或是 Unicode 字符。UnreadByte 和 UnReadRune 方法用于将消费的最后一个字节数据或 Unicode 字符放回缓冲区，以便后续的读取操作能够重新读取该字节数据或 Unicode 字符。

ReadSlice 方法用以从输入流中读取以指定字节为分割符的一行数据，循环调用内部 io.Reader 提供的方法填充缓冲区，直至出现指定字符、填满缓冲区或出现错误时返回数据，扫描指定字符时会在每次循环更新扫描开始位置，确保不重复扫描同一个数据区域。ReadLine 方法用于从输入流中处理以换行符 '\\r' 和 '\\n' 结尾的一行数据，并自动去除换行符，当缓冲区满时，如果数据以 '\\r' 结尾，则可能存在 '\\r' 和 '\\n' 跨越缓冲区的情况，会将 '\\r' 放回缓冲区。ReadSlice 方法当缓冲区填满时即时没有出现指定字符也会返回，ReadBytes 方法循环调用 ReadSlice 直至出现指定字符，并将多次读取的数据拼接成一个完成的数据返回。ReadString 方法类似，但返回的不是字节数组而是字符串类型。

上述提到的所有方法有些是返回缓冲区的引用，有些需要将缓冲区的数据进行拷贝。具体来说，Peek、ReadSlice、ReadLine 实现了零拷贝，Read、ReadByte、ReadRune、ReadBytes、ReadString 需要额外一次内存拷贝。

WriteTo 方法用于将 Reader 的所有数据写入实现 io.Writer 接口的对象中，该方法首先将缓存中的数据写入，如果 Reader 内部的 io.Reader 实现了 WriterTo 方法，则直接调用内部 io.Reader 的 WriterTo 方法完成后续流程，如果传入的实现了 io.Writer 接口的对象 实现了 ReaderFrom 方法，则直接通过该 ReadFrom 方法完成后续流程，否则循环填充缓存，并将缓存中的数据写入，直至遇到 io.EOF 错误写入所有数据。
### 3.2 Writer

```Go
package bufio

type Writer struct {
    err error
    buf []byte
    n int
    wr io.Writer
}

func NewWriterSize(w io.Writer, size int) *Writer { ... }
func NewWriter(w io.Writer) *Writer { ... }
func (b *Writer) Size() int { ... }
func (b *Writer) Reset(w io.Writer) { ... }
func (b *Writer) Flush() error { ... }
func (b *Writer) Available() int { ... }
func (b *Writer) AvailableBuffer() []byte { ... }
func (b *Writer) Buffered() int { ... }
func (b *Writer) Write(p []byte) (int, error) { ... }
func (b *Writer) WriteByte(c byte) error { ... }
func (b *Writer) WriteRune(r rune) (int, error) { ... }
func (b *Writer) WriteString(s string) (int, error) { ... }
func (b *Writer) ReadFrom(r io.Reader) (int64, error) { ... }
```

Writer 的实现相对简单，err 记录读写过程中的错误，buf 用于写入缓存，n 用于记录缓存的字节数，wr 记录封装的 io.Writer。Writer 缓存的默认大小为 4096 Bytes，且在创建完成后大小不会发生变化。Flush 函数用于将 Writer 缓存的数据写入内部的 io.Writer。Write、WriteByte、WriteRune、WriteString 函数用于写入指定的数据，在写入过程中，会首先尝试填充缓存，并当缓存满的时候调用 Flush 并循环直至需要写入的所有数据均以写入缓存或写入内部的 io.Writer，并且在过程中发现需要写入的数据过大时会将数据直接写入内部的 io.Writer，避免拷贝带来的过大开销。ReadFrom 函数用于从一个数据源读取数据，并将获取的数据写入，在读取过程中会首先将读取的数据写入缓存，并当缓存的数据全部写入后，通过内部的 io.Writer 实现的 ReadFrom 函数继续剩余的过程。

```Go
func (b *Writer) AvailableBuffer() []byte {
    return b.buf[b.n:][:0]
}
```

AvailableBuffer 方法提供了一种零拷贝的写入方法，这个函数的目的是返回一个空的缓冲区，其容量等于 Writer 缓存未被填满的空间。通过这个方法，调用者可以在 Writer 缓存中直接追加数据，从而避免其他写入方法带来的额外一次内存拷贝。需要注意的是，函数返回的缓冲区仅在下一次写入操作之前有效，否则会引入数据错乱。
### 3.3 ReadWriter

```Go
package bufio

type ReadWriter struct {
    *Reader
    *Writer
}

func NewReadWriter(r *Reader, w *Writer) *ReadWriter { ... }
```

bufio 包还提供了一个 ReadWriter 结构体，允许将 Reader 和 Writer 作为一个整体来处理读取和写入。
### 3.4 Scanner

bufio 中的 Scanner 类型是一个用于读取文本数据的方便的工具，Scanner 提供了逐行或逐个指定分割符读取文本的能力，并可以通过自定义的 SplitFunc 函数来处理不同的分割符。

```Go
package bufio

type Scanner struct {
    r io.Reader
    split SplitFunc
    maxTokenSize int
    token []byte
    buf []byte
    start int
    end int
    err error
    empties int
    scanCalled bool
    done bool
}

type SplitFunc func(data []byte, atEOF bool) (int, []byte, error)
func ScanBytes(data []byte, atEOF bool) (int, []byte, error) { ... }
func ScanRunes(data []byte, atEOF bool) (int, []byte, error) { ... }
func ScanLines(data []byte, atEOF bool) (int, []byte, error) { ... }
func ScanWords(data []byte, atEOF bool) (int, []byte, error) { ... }

func NewScanner(r io.Reader) *Scanner { ... }
func (s *Scanner) Err() error { ... }
func (s *Scanner) Buffer(buf []byte, max int) { ... }
func (s *Scanner) Split(split SplitFunc) { ... }
func (s *Scanner) Bytes() []byte { ... }
func (s *Scanner) Text() string { ... }
func (s *Scanner) Scan() bool { ... }
```

Scanner 结构体中 r 记录 io.Reader 输入流，通过这个输入流读取数据进行扫描，split 记录定义如何分割输入文本数据的函数，maxTokenSize 记录 token 的最大大小，token 记录最后一个被 split 函数返回的 token，buf 一个用作参数传递给 split 函数的缓冲区，buf 保存了从输入流中读取的数据，start 表示 buf 中第一个未处理字节的位置，end 表示 buf 中数据的结束位置，err 记录潜在的错误，empties 记录连续空 token 的次数，scanCalled 标记是否调用过 Scan 方法，done 表示是否扫描结束。

在 bufio 包中提供了四种 SplitFunc 实现，分别是 ScanBytes、ScanRunes、ScanLines 和 ScanWords，默认情况下通过 NewScanner 创建的 Scanner 会使用 ScanLines。通过 Buffer 和 Split 方法可以修改 Scanner 所使用的缓冲区、token 的最大大小以及所使用的 SplitFunc。Bytes 和 Text 方法返回最后一个记录的 token。

Scan 方法是 Scanner 结构体的核心方法，它通过循环直到生成一个 token 或者扫描结束。在循环过程中，它首先尝试通过缓存中的已有数据生成 token，如果缓冲区数据遇到错误无法生成 token则返回错误，如果需要更多数据，首先判断当前缓冲区的使用情况，如果当前缓冲区前半部分有很多空闲空间或缓冲区已满则将数据移动到缓冲区的开始位置，如果缓冲区仍然填满，则成倍扩大缓冲区的大小，然后通过循环内部 io.Reader 的接口获取一次数据，如果连续取到空数据的次数超过阈值，则返回错误。Scan 方法的主要逻辑是不断从输入流中读取数据，并使用 split 函数生成 token，直到遇到 io.EOF 错误或其他错误为止，在这个过程中，会通过成倍扩大缓冲区大小不断调整缓冲区大小以适应输入数据。

Scanner 结构体提供的 Scan 方法与 Reader 结构体提供的 ReadBytes 方法类似，均可以实现相似的功能，Scan 方法提供了更高度定制的分割行为以及对错误的处理，在某些简单的读取场景，使用 ReadBytes 方法可能更为方便。