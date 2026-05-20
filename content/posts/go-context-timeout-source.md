---
title: "Go context 最容易被误用的地方：以为 cancel 会替你收拾现场"
description: "从一次接口超时和 goroutine 上涨的排查开始，拆解 Go context 的 cancelCtx、timerCtx、valueCtx：cancel 到底做了什么，WithTimeout 为什么要 defer cancel，pprof 和 go vet 怎么定位问题。"
date: 2026-05-20T17:50:00+08:00
draft: false
author: "任博"
tags: ["Go", "context", "并发", "源码", "后端工程"]
categories: ["技术实战"]
cover: "/images/go-context-timeout-source/cover.png"
toc: true
---

线上接口开始超时，goroutine 数一路往上涨。

你第一反应可能是：不是已经传了 `context.WithTimeout` 吗？超时到了，goroutine 不就该停了吗？

这就是 Go `context` 最容易被误用的地方。

`context` 不负责替你停掉 goroutine。它只负责把“该停了”这句话传到门口。门里的人听不听、什么时候退场，要看你自己的代码。

![Go context 调用链生命周期封面](/images/go-context-timeout-source/cover.png)

这篇是 Go `context` 系列的第一篇。先不急着讲 HTTP、gRPC、database/sql 这些框架怎么接入，也不急着把所有 API 列一遍。

我们先把最底下这层讲清楚：`context` 到底是什么，`cancel` 到底做了什么，为什么 `WithTimeout` 后面那句 `defer cancel()` 不是仪式感，而是资源释放动作。

## 事故通常不是从源码开始的

真实排查里，很少有人一上来就打开 `context.go`。

更常见的是这样的场景：

一个接口偶发超时。压测一上来，P99 开始抖，错误日志里出现 `context deadline exceeded`。你看监控，CPU 没打满，内存也没爆，但 goroutine 数不太对，一路往上爬。

然后你打开 pprof：

```bash
curl -s 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=1'
```

采样里有一批 goroutine 卡在业务 worker 上，等某个 channel，或者等 `<-ctx.Done()`。

这时候最危险的判断是：

“是不是 Go 的 context 泄漏了？”

大多数时候，不是。

更可能是你的调用链里少了三件事之一：创建了 timeout context 但没及时 `cancel`；启动了 goroutine 但没监听 `ctx.Done()`；或者监听了 `Done`，却没有确保取消路径真的跑到。

`context` 本身不是清洁工。它更像一套调用链生命周期协议：谁是父，谁是子，什么时候该停，什么时候超时，少量请求级元数据怎么沿链传下去。

协议只负责传达，不负责替每个 goroutine 做决定。

## 先把 Context 接口看薄一点

`context.Context` 看起来很大，其实接口只有四个方法：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

这四个方法对应四件事：

- `Deadline`：这条调用链最晚什么时候结束；
- `Done`：取消信号从哪里来；
- `Err`：为什么结束，是主动取消还是超时；
- `Value`：少量 request-scoped 数据怎么取。

所以 `context` 的本质，不是“参数包”。

如果你把它当成万能 map，用它传 logger、DB handle、大对象、可选参数，代码短期会变方便，长期会变浑。因为调用方看函数签名时，已经看不出这个函数真正依赖什么。

Go 官方文档反复强调，Context values 只应该用于跨 API 边界和进程边界的 request-scoped data。比如 request id、trace id、认证相关的小型元数据。

Go 的很多 API 设计都有这种克制感。Rob Pike 这些早期核心人物留下来的影响，不是“把所有东西做进框架”，而是尽量让控制权和边界留在调用方手里。

![Rob Pike 资料照片](/images/go-context-timeout-source/rob-pike-oscon.jpg)

真正的业务参数，应该老老实实写在函数参数里。

这条边界很重要。因为一旦你理解 `context` 是生命周期协议，而不是传参工具，后面的源码就顺了。

## cancelCtx：取消树的核心节点

`WithCancel`、`WithTimeout`、`WithDeadline` 创建出来的 context，背后都绕不开 `cancelCtx`。

可以把它理解成一棵树里的节点。

父节点取消，会向下通知所有子节点；子节点取消，通常不会反过来取消父节点。一个请求进来，最外层 context 是根，下面每派生一个子操作，就长出一个子节点。

![context cancellation tree 示意图](/images/go-context-timeout-source/inline-01.svg)

Go 1.25.4 的 `cancelCtx` 里，核心字段大致是这样：

```go
type cancelCtx struct {
    Context

    mu       sync.Mutex
    done     atomic.Value          // chan struct{}, lazy create
    children map[canceler]struct{}
    err      atomic.Value
    cause    error
}
```

这里面最值得看的是三样东西。

第一，`done` channel 是懒加载的。只有调用 `Done()` 时才创建。一个 context 如果从来没人监听它的 `Done`，就不必急着分配 channel。

第二，`children` 保存子 context。父 context cancel 时，会遍历 children，把取消信号继续传下去。

第三，`err` 和 `cause` 记录取消原因。普通取消是 `context.Canceled`，超时是 `context.DeadlineExceeded`。

真正调用 cancel 时，标准库做的事并不玄：设置错误原因，关闭 `done` channel，递归取消 children，必要时把自己从父节点的 children map 里删掉。

它没有魔法。

它不会暂停 goroutine，不会抢占你的业务逻辑，不会帮你关闭数据库连接，也不会替你把某个阻塞的 channel 读写解开。

**cancel 不是杀 goroutine，它只是关闭 Done channel 并记录错误。**

业务 goroutine 想退出，必须自己写成能退出的样子。

## 正确代码里，Done 一定要被听见

一个比较稳的下游调用，通常会长这样：

```go
func fetch(ctx context.Context, name string) (string, error) {
    select {
    case <-time.After(50 * time.Millisecond):
        return "hello " + name, nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
    defer cancel()

    v, err := fetch(ctx, "context")
    if err != nil {
        return
    }
    fmt.Println(v)
}
```

这里有两个动作，一个都不能少。

第一，创建 `WithTimeout` 之后，立刻 `defer cancel()`。

第二，下游阻塞等待时，把 `<-ctx.Done()` 放进 `select`。

很多事故恰好就是少了第二步。代码创建了 context，也传下去了，但 worker 完全不听：

```go
func leakyWorker(ctx context.Context, id int) {
    _ = ctx // ignored: this is the bug
    ch := make(chan struct{})
    <-ch // blocks forever
}
```

这时候 timeout 到了也没用。`context` 在门外喊“该停了”，worker 在屋里戴着耳机，听不见。

你会看到 goroutine 数上涨，然后误以为 timeout 没生效。其实 timeout 生效了，只是你的 goroutine 没有协作退出。

## WithTimeout 的 cancel，不是超时按钮

很多人写 `WithTimeout` 时，会有一个误解：既然时间到了会自动超时，那我不调用 `cancel` 也没关系。

这句话只对了一半，也正因为只对一半，才危险。

`WithTimeout(parent, d)` 本质上是 `WithDeadline(parent, time.Now().Add(d))`。底层的 `timerCtx` 内嵌了 `cancelCtx`，额外保存一个 timer 和 deadline。deadline 到了，runtime timer 会触发回调，调用 cancel，把错误设置为 `DeadlineExceeded`。

但如果你的函数在 deadline 之前就返回了呢？

比如 timeout 设置 10 秒，实际 50 毫秒就拿到了结果。如果你不调用 `cancel`，这个 timer 和父子引用就会继续留到 deadline 或父 context 被取消。

这就是 `defer cancel()` 的真实意义。

**WithTimeout 的 cancel 不是超时按钮，而是资源释放按钮。**

Go 官方文档里说得很明确：不调用 `CancelFunc` 会泄漏 child 和它的 children，直到 parent 被 cancel；go vet 也会检查 CancelFunc 是否在所有控制流路径上被使用。

注意这里的边界：这不等于“忘记 cancel 一定会泄漏 goroutine”。

更准确的说法是：忘记 cancel 会让 context child、children、timer 等资源留得更久；至于 goroutine 会不会泄漏，要看你的业务 goroutine 是否在等待取消信号，以及取消路径是否真的能跑到。

把这句话讲准，比吓人更重要。

## go vet 能抓住一部分低级错误

下面这段代码是故意写错的：

```go
for i := 0; i < 5; i++ {
    ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
    go leakyWorker(ctx, i)
}
```

它有两个问题：

- `WithTimeout` 返回的 cancel 被丢掉；
- `leakyWorker` 又忽略了 `ctx.Done()`。

对第一类问题，`go vet` 能直接给出提示：

```bash
go vet ./examples/...
```

输出类似：

```text
examples/wrong/main.go:17:8: the cancel function returned by context.WithTimeout should be called, not discarded, to avoid a context leak
```

这就是为什么我建议你把 `go vet` 放进日常检查，而不是只在上线前手跑一次。它抓不住所有生命周期问题，但能抓住一批“不该犯”的错误。

尤其是循环里创建 timeout context、分支里提前 return、错误路径漏掉 cancel，这些问题靠肉眼 review 很容易漏。

## pprof 看什么：看等待点，不要急着甩锅

如果 goroutine 数已经涨起来，`go vet` 只能帮你看静态路径，下一步就要采样。

最小路径是先接入 pprof：

```go
import _ "net/http/pprof"

func main() {
    go http.ListenAndServe("127.0.0.1:6060", nil)
    // ...
}
```

然后抓 goroutine profile：

```bash
curl -s 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=1'
curl -s 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=2'
```

在本次示例里，采样能看到这样的片段：

```text
goroutine profile: total 14
10 @ ...
# main.worker ... examples/pprof-demo/main.go:34
```

![pprof goroutine 输出示意](/images/go-context-timeout-source/inline-03.svg)

这段信息只能说明一件事：有一批 goroutine 停在 `main.worker`。

它还不能单独证明“context 有问题”，也不能单独证明“忘 cancel 导致 goroutine 泄漏”。你还要回到代码里看：这些 worker 是谁创建的？对应的 cancel 在哪里？成功路径、错误路径、提前返回路径都会调用吗？worker 有没有在自己的 select 里响应 `ctx.Done()`？

排查 context 相关问题时，我建议按三条线走：

1. 创建路径：谁调用了 `WithCancel` / `WithTimeout` / `WithDeadline`？
2. 取消路径：返回的 `cancel` 是否在所有路径上被调用？
3. 响应路径：所有可能阻塞的 goroutine 是否监听 `ctx.Done()`？

少看任何一条，结论都容易偏。

## propagateCancel：不是每个 WithCancel 都起 goroutine

还有一个常见误解：每调用一次 `WithCancel`，标准库是不是就起一个 goroutine 去监听父 context？

不是。

`propagateCancel` 里大致有几条路径。

如果 parent 的 `Done()` 是 nil，说明它不会取消，直接返回。

如果 parent 已经取消，child 立刻取消。

如果能找到标准库内部的 `*cancelCtx`，child 会被加入 parent 的 `children` map。这是标准 context 链最常见的情况，不需要额外 goroutine。

只有在某些 fallback 情况下，比如 parent 是自定义 Context，标准库无法拿到内部 `cancelCtx`，才会启动一个 goroutine，在 parent.Done 和 child.Done 之间 select。

所以不要把 context 性能问题简单归结为“创建太多 context 就创建太多 goroutine”。这句话不准。

真正该关心的是：你创建了多少生命周期节点，它们什么时候解除父子引用，timer 是否及时停止，业务 goroutine 是否真的能退出。

## valueCtx：WithValue 不是 map，是链

`context` 里最容易被滥用的另一个 API，是 `WithValue`。

它看起来太方便了：

```go
ctx = context.WithValue(ctx, requestIDKey{}, "req-123")
```

但源码里，`WithValue` 并不是往一个 map 里塞值。每调用一次，它就包一层 `valueCtx`：

```go
type valueCtx struct {
    Context
    key, val any
}
```

取值时，先看当前层 key 是否匹配；不匹配，就继续往 parent 查。

![context value chain 查找示意图](/images/go-context-timeout-source/inline-02.svg)

这会带来三个后果。

第一，查找成本和链深度有关。源码没有把复杂度写成文档承诺，但实现就是沿链逐层找。

第二，同一个 key 在更内层设置时，会遮蔽外层的值。这种 shadowing 有时是你想要的，有时会让排查变得很烦。

第三，value 会被 context 链引用。你把大对象塞进去，如果 context 生命周期很长，就可能放大内存保留问题。

所以 `WithValue` 的边界要收紧：request id、trace id、auth token、user id 这类轻量请求元数据可以；DB handle、logger 配置、大 slice、业务可选参数，不该放进去。

一句话：`WithValue` 是跨 API 边界携带少量请求元数据的工具，不是给你绕开函数签名的后门。

## Background 和 TODO 行为像，语义不一样

`context.Background()` 和 `context.TODO()` 很多人混着用。

从运行行为看，它们确实接近：都不会取消，没有 deadline，也没有 value。

但语义不一样。

`Background()` 是明确的根。通常用在 `main`、初始化、测试、入站请求的顶层。

`TODO()` 是占位。它表达的是：这里应该传一个合适的 context，但现在还没想清楚，或者函数签名还在迁移中。

如果一段业务代码长期保留 `context.TODO()`，它不是“也能跑所以没事”，而是在告诉后来的人：这里的调用链生命周期还没设计完。

这类语义差异，源码层面看不出性能差别，但工程上很重要。

## 一份可以直接用的检查清单

如果你现在维护一个 Go 服务，可以先不读完所有源码，直接做三件事。

第一，查所有 `WithTimeout` / `WithDeadline` / `WithCancel`。

每个返回的 `cancel` 都应该有明确去处。最常见、也最安全的写法是创建后立刻：

```go
ctx, cancel := context.WithTimeout(parent, 2*time.Second)
defer cancel()
```

如果不能 defer，比如在循环或异步启动里，也要确保成功路径、错误路径、提前返回路径都有释放动作。

第二，查所有 goroutine 的阻塞点。

凡是可能长时间等待 channel、网络、锁、队列、外部任务的 goroutine，都要问一句：它怎么知道上游已经不等了？

典型写法是：

```go
select {
case <-ctx.Done():
    return ctx.Err()
case result := <-resultCh:
    return result
}
```

第三，查 `WithValue`。

看到 `context.WithValue` 时，不要只问“能不能用”，要问这三个问题：

- 这个值是不是 request-scoped？
- 它是不是轻量元数据？
- 不放 context，函数签名会不会更清楚？

如果答案不稳，就别往 context 里塞。

## 最后：context 管的是生命周期，不是你的现场

`context` 的设计很克制。

它不把取消做成强制中断，不让子操作取消父操作，不鼓励把 Context 存进 struct，也不鼓励把 Value 当参数袋。

这背后其实是一种 Go 风格：生命周期要显式，控制权要清楚，退出要协作。

你可以把 `context` 想成调用链里的一张“停工通知”。它能告诉所有下游：请求取消了，deadline 到了，这批工作该收了。

但它不会替你关掉机器。

真正可靠的 Go 服务，不是“到处都传了 ctx”就够了，而是每个创建、取消、等待、退出的路径都能对上。

下次线上再看到 goroutine 涨上去，不要先问“context 怎么没生效”。

先问三件事：

`cancel` 调了吗？

`Done` 听了吗？

退出路径走得到吗？
