---
title: "ready=true 不是发布：Go 内存模型真正要你保证什么"
description: "从一个看似无害的 ready 标志位开始，讲清 Go happens-before、DRF-SC、race detector、sync/atomic 和内存屏障的边界：并发安全不是看起来有顺序，而是规范承认它有顺序。"
date: 2026-05-22T16:40:00+08:00
draft: false
author: "任博"
tags: ["Go", "并发编程", "内存模型", "happens-before", "data race", "atomic"]
categories: ["技术实战"]
cover: "/images/go-memory-model/happens-before.png"
toc: true
---

线上偶现一个很烦的 bug：配置已经加载了，日志也打印了“ready”，但另一个 goroutine 读到的还是旧值。

代码看起来没什么毛病：先写数据，再把 `ready` 置成 `true`。读的一侧先等 `ready`，再读数据。人脑看这段逻辑，很容易得出一个结论：既然 `ready` 已经是 `true`，前面的数据当然也该准备好了。

问题就在这里。

在 Go 里，`ready=true` 不等于发布。普通变量上的“先写后读”，也不等于 goroutine 之间有可见性保证。

并发安全不是让代码看起来有先后，而是让规范承认它有先后。

这篇是 Go 内存模型系列的第一篇。它不打算把你拖进 CPU 指令、缓存一致性协议和汇编细节里。我们先解决一个更常见、更工程化的问题：goroutine A 写了变量，goroutine B 什么时候保证能看到？

答案只有一个：两边要存在 happens-before 关系。

![happens-before 关系示意图](/images/go-memory-model/happens-before.svg)

## 最危险的代码，通常看起来最顺

先看这段很多人都写过的代码：

```go
var data string
var ready bool

go func() {
    data = "payload"
    ready = true
}()

for !ready {
}
fmt.Println(data)
```

直觉上，写入顺序很清楚：`data` 先被赋值，`ready` 后变成 `true`。读的一侧又是看到 `ready` 才打印 `data`，似乎不会出错。

但这段代码在 Go 内存模型里是错的。

错不在“CPU 一定会重排”，也不在“编译器一定会优化坏”。这些说法太具体，反而容易把问题讲窄。真正的问题更简单：`data` 和 `ready` 都是普通变量，不同 goroutine 对它们并发读写，中间没有任何同步动作。

没有同步，就没有 happens-before。

没有 happens-before，读到 `ready=true` 也不保证读到 `data="payload"`。

Go 官方内存模型开头有一句非常硬的话：如果你必须读完整篇内存模型文档才能理解程序行为，那你写得太聪明了。不要这么聪明。

这句话不是说工程师不用懂内存模型。恰好相反，懂它是为了知道哪些“聪明写法”应该直接删掉。

## happens-before 不是时间线，是证明链

很多人第一次看到 happens-before，会把它理解成“真实时间上先发生”。这会误导判断。

happens-before 更像一条可见性证明链。链上的每一段，要么来自同一个 goroutine 里的顺序执行，要么来自一个明确的同步动作。

Go 官方把这条链拆成两类来源：

```text
同一个 goroutine 内的顺序：sequenced-before
跨 goroutine 的同步顺序：synchronized-before

happens-before = 二者并集的传递闭包
```

这句话听起来抽象，放回工程里就很好懂。

同一个 goroutine 里：

```go
a = 1
b = 2
```

你可以按程序顺序理解。但两个 goroutine 之间，光靠代码顺序不够。你需要 channel、mutex、Once、WaitGroup、atomic 这类同步动作，把两个 goroutine 接起来。

例如用 channel close 做发布：

```go
var data string
done := make(chan struct{})

go func() {
    data = "payload"
    close(done)
}()

<-done
fmt.Println(data)
```

这里不只是“通知了一下”。`close(done)` happens-before 那个因为 channel 已关闭而返回的 receive。于是 `data = "payload"` 通过同 goroutine 内顺序接到 `close(done)`，再接到 `<-done`，最后接到 `fmt.Println(data)`。

链闭合了，读才有保证。

这也是为什么 channel 在 Go 里不只是传值。它还在传递“之前写入已经可见”的承诺。

## data race 的定义，比“偶现 bug”更严格

Go 对 data race 的定义可以压成四个条件：

```text
同一个内存位置
两个操作并发发生
至少一个是写
中间没有同步
```

满足这四个条件，程序就是错的。它不需要先在生产环境出事故，也不需要 race detector 报警，才算错。

一个常见误解是：只要机器上跑了很多天没出问题，就说明这段并发读写没事。

这其实是在拿平台偶然表现当语言保证。

amd64 的内存模型通常比 arm64 更强，有些错误在你的开发机上不容易复现；某个 Go 版本的编译器这次没有做某种优化，也不代表以后不会；测试数据量小、时序刚好，也可能把问题藏起来。Go 程序的正确性不能靠“我这里没复现”。

正确的问题不是：它现在会不会刚好跑通？

正确的问题是：这段读写之间有没有 happens-before？

## Go 给你的大承诺：DRF-SC

Go 内存模型里有一个很重要的承诺：DRF-SC。

全称是 data-race-free programs execute in a sequentially consistent manner。翻成工程语言就是：只要你的程序没有 data race，你基本可以按“单处理器上交错执行”的方式理解并发结果。

这其实是 Go 在帮你把世界变简单。

底层当然复杂。CPU 有缓存，编译器会优化，运行时要调度 goroutine，不同架构还有不同的内存顺序。但只要你把同步关系写清楚，Go 就尽量让你回到一个可推理的模型里。

Go 的内存模型不是鼓励你写更聪明的无锁代码，而是帮你少写这种代码。

Russ Cox 在 2021 年更新 Go 内存模型时也强调过这条路线：Go 不想把多数开发者推进 C/C++ 那套细颗粒 memory ordering 游戏里。Go 的 `sync/atomic` 没有暴露 relaxed、acquire、release 那一堆选项，所有 atomic 操作按顺序一致模型来理解。

这不是因为底层不存在差异，而是因为 API 做了取舍。

少给一点旋钮，多给一点可推理性。

## 几种常见同步原语，到底保证了什么

很多并发 bug 的根源，是把“互斥”“等待”“通知”“可见性”混成了一件事。它们有关，但不是一回事。

### channel：send、receive、close 都可能建立同步

Go 内存模型对 channel 有几条关键规则：

```text
send happens-before 对应 receive 完成
close happens-before 因关闭而返回零值的 receive
无缓冲 channel 上，receive happens-before 对应 send 完成
容量为 C 的 buffered channel 上，第 k 次 receive happens-before 第 k+C 次 send 完成
```

日常写法里，最常用的是“发送结果”和“关闭 done channel”：

```go
type Result struct {
    Value string
    Err   error
}

ch := make(chan Result, 1)

go func() {
    ch <- Result{Value: "ok"}
}()

r := <-ch
fmt.Println(r.Value)
```

`ch <- Result{...}` 到 `<-ch` 之间有同步关系，所以接收方能看到发送方在发送前准备好的数据。

这也是为什么 channel 适合表达所有权转移和完成信号。它不是“高级队列”，它是带同步语义的通信原语。

### Mutex：Unlock 不只是开门，也是在发布写入

很多人对 mutex 的理解停在“同一时间只有一个 goroutine 能进去”。这当然对，但不够。

对同一个 `sync.Mutex` 或 `sync.RWMutex`，一次 `Unlock` happens-before 后续某次 `Lock` 返回。

所以这段代码里的锁，不只是保护临界区，还保证可见性：

```go
var mu sync.Mutex
var n int

mu.Lock()
n = 42
mu.Unlock()

mu.Lock()
fmt.Println(n)
mu.Unlock()
```

如果没有锁，`n = 42` 和 `fmt.Println(n)` 之间没有跨 goroutine 的证明链。加上锁后，释放锁之前的写入，对之后成功拿到锁的 goroutine 可见。

### Once：别手写 double-checked locking

`sync.Once` 是 lazy initialization 的标准答案，不是因为它看起来优雅，而是因为它有明确的内存模型保证：`once.Do(f)` 中真正执行的 `f` 返回，happens-before 任何一次 `once.Do(f)` 调用返回。

错误写法通常长这样：

```go
var inited bool
var obj *Client

func Get() *Client {
    if inited {
        return obj
    }
    obj = newClient()
    inited = true
    return obj
}
```

这段的危险点不是“可能初始化两次”这么简单。更麻烦的是：另一个 goroutine 看到 `inited=true`，不代表它一定看到 `obj` 已经完整初始化。

`ready=true` 不是发布，`inited=true` 也不是。

用 `sync.Once`：

```go
var once sync.Once
var obj *Client

func Get() *Client {
    once.Do(func() {
        obj = newClient()
    })
    return obj
}
```

这不是保守，是省脑子。并发代码里，省脑子通常就是省事故。

### WaitGroup：它等待结束，不保护过程

Go 1.25 加了 `WaitGroup.Go`，可以启动 goroutine 并自动管理计数。文档里的同步关系也很明确：`Done`，或者 `WaitGroup.Go` 里函数 `f` 的返回，会 happens-before 被它解除阻塞的 `Wait` 返回。

这意味着下面这种“所有 worker 写完，再统一读结果”的模式是成立的：

```go
var wg sync.WaitGroup
results := make([]int, 4)

for i := range results {
    i := i
    wg.Go(func() {
        results[i] = i * 10
    })
}

wg.Wait()
fmt.Println(results)
```

但 WaitGroup 不是锁。它保证的是“结束之后可见”，不是“执行期间互斥”。

如果一个 goroutine 还在写 map，另一个 goroutine 在 `Wait` 返回前去读，这依然可能是 data race。WaitGroup 不能替你保护共享状态的读写过程。

### atomic：适合发布简单状态，不适合维护复合不变量

Go 的 atomic 很克制。官方文档说得很清楚：如果 atomic 操作 A 的效果被 atomic 操作 B 观察到，那么 A synchronized-before B；并且所有 atomic 操作表现得像按某个顺序一致的总顺序执行。

这让 atomic flag 可以用来做简单发布：

```go
var data int
var ready atomic.Bool

go func() {
    data = 99
    ready.Store(true)
}()

for !ready.Load() {
}
fmt.Println(data)
```

`Load` 观察到 `Store(true)` 后，这条同步链就建立起来了。

但 atomic 不是 mutex 的高级替代品。最容易出事的是多个字段：

```go
var count atomic.Int64
var sum atomic.Int64

c := count.Load()
s := sum.Load()
fmt.Println(c, s)
```

每次 Load 都是 atomic 的，不代表 `count` 和 `sum` 组成了一张一致快照。你可能读到新 `count` 配旧 `sum`。如果这两个字段共同构成一个业务不变量，用 mutex 往往更直接；或者把它们做成不可变 snapshot，再用 `atomic.Pointer[T]` 一次发布整份快照。

atomic 更低层，不是更高级。

## 内存屏障要懂，但别拿它写业务推理

聊 Go 内存模型，很容易滑到 StoreLoad、StoreStore、LoadLoad、LoadStore 这些屏障名词。它们确实重要，但要放在正确位置。

这些概念主要解释的是硬件和运行时如何约束重排：哪些读不能越过哪些写，哪些写必须先对其他核心可见。不同架构差异很大。amd64 的 TSO 相对强，arm64 更弱，需要更多显式同步指令来表达同样的约束。

但业务代码不应该写成“我觉得这里需要一个 StoreLoad 屏障”。Go 标准库也没有把这些 fence 当作普通业务 API 暴露给你。

Go 给你的抽象是：用 channel、mutex、Once、WaitGroup、atomic 建立同步关系。至于底层在 arm64 或 amd64 上怎么落到指令、编译器怎么限制优化、运行时怎么配合，那是实现负责的事。

![内存屏障与 Go 同步原语关系](/images/go-memory-model/memory-barrier.svg)

这并不代表可以完全不懂重排。你至少要知道两件事：

第一，编译器和 CPU 有权在不改变单线程可观察行为的前提下调整执行顺序。你在源码里看到的先后，不会自动变成跨 goroutine 的可见性。

第二，Go 编译器也不是随便乱动。Russ Cox 的文章里列过一些限制，例如不能凭空引入 data race，不能把可能影响并发可见性的读写移出条件、循环或函数边界。这些限制让 Go 比某些语言更可调试，但它们不是让你依赖未同步读写的许可证。

内存屏障是地基，不是你每天手工搬的砖。

## race detector 是烟雾报警器，不是建筑结构图

`go test -race` 很有用。并发改动上线前，我会把它当成必跑项。但它有一个边界必须说清楚：race detector 是运行时检测，不是静态证明。

常用命令就这几类：

```bash
go test -race ./...
go test -race -run TestName ./pkg
go run -race ./cmd/demo
go build -race ./cmd/server
GORACE="halt_on_error=1 strip_path_prefix=$(pwd)/" go test -race ./...
```

它的工作方式大致是：在 `-race` 编译时对内存访问插桩，运行时记录读写关系，发现没有同步保护的冲突访问就报告。Go 官方博客说明它基于 ThreadSanitizer。

这意味着它只能发现实际跑到的路径。

没报警不代表没火，可能只是你没把那条路径跑出来。

官方文档也给过开销量级：内存约 5-10 倍，运行时间约 2-20 倍。所以它适合放在 CI、压力测试、复现环境和关键并发改动回归里，不适合默认开在所有生产进程上。

race 报告出来后，不要只看第一行 `WARNING: DATA RACE`。真正要看的有两类栈：冲突访问发生在哪里，以及相关 goroutine 是在哪里创建的。前者告诉你哪两个读写打架，后者告诉你它们为什么会同时存在。

![数据竞争检测流程](/images/go-memory-model/race-detector-flow.svg)

还有一个任务单里容易混淆的点：`go tool trace` 不检测 data race。

它看的是 runtime execution trace：调度、syscall、网络阻塞、sync blocking、锁竞争、利用率。它能帮你分析“为什么卡”“谁在等”，但不会替代 `-race` 判断数据竞争。

工具边界可以简单记：

```text
-race：抓实际执行到的数据竞争
go tool trace：看调度和阻塞
goleak：测 goroutine 是否泄漏
go vet / staticcheck：做静态质量检查
```

把工具用错方向，会给你一种很危险的安全感。

## 几个模式，背后其实都是同一条链

内存模型听起来抽象，落到工程里，常见并发模式背后其实都在解决同一件事：谁负责写，谁负责读，中间用什么动作交接。

生产者消费者是最典型的例子。生产者把任务写进 channel，消费者从 channel 里取任务。这个模型安全，不是因为 channel 名字叫“队列”，而是因为 send 和 receive 之间有同步关系。任务对象在 send 前准备好的字段，receive 后有可见性保证。

```go
jobs := make(chan Job)

go func() {
    job := buildJob()
    jobs <- job
}()

job := <-jobs
handle(job)
```

如果你把 channel 换成共享 slice，再用一个普通 bool 表示“有新任务”，问题马上回来。slice 的写入、bool 的写入、消费者的读取，中间没有同步链。偶尔能跑，不代表模型成立。

safe publication 也是同一回事。一个对象构造完以后，要么通过 channel 交给读者，要么在 mutex 保护下放进共享状态，要么用 `atomic.Pointer[T]` 一次发布不可变快照。不要一边改对象内部字段，一边让别的 goroutine 读同一个指针。

熔断器模式也一样。一个服务是否 open、half-open、closed，通常不是一个孤立 bool，而是一组状态：失败次数、最后失败时间、半开探测窗口、当前状态。如果只是用几个 atomic 分别读写，单次读写也许都安全，但组合起来未必是一致状态。这里更稳的做法通常是 mutex 包住状态机，或者把状态做成不可变 snapshot 后整体发布。

判断一个模式是否安全，不要先看名字。

看交接点。

只要共享状态跨 goroutine 流动，就必须有一个同步动作负责“盖章”。没有这个章，模式再经典也只是代码形状像而已。

## 写并发代码时，按这几个问题自查

如果你不想每次都翻内存模型，可以先用这几个问题过滤大多数风险。

第一，共享变量是不是被多个 goroutine 访问，而且至少一边在写？如果是，继续问第二个问题。

第二，这些访问之间有没有明确同步？channel send/receive/close、同一把 mutex 的 Unlock/Lock、Once、WaitGroup 的 Done/Wait、atomic Store/Load，至少要有一条链。

第三，你保护的是单个值，还是一组不变量？单个状态位可以考虑 atomic；一组字段、一个 map、一个对象生命周期，优先 mutex 或 channel 所有权转移。

第四，你是在等待结束，还是在保护过程？WaitGroup 只解决结束后的可见性，不解决执行期间的并发读写。

第五，你有没有把“本机没复现”当成安全证明？如果有，先跑 `go test -race`，再回到 happens-before 推理。

这几句看起来朴素，但比背一堆屏障名更有用。

## 一个实际修复顺序

回到开头的 `ready/data`。

如果只是完成通知，最清楚的修法是 channel：

```go
var data string
done := make(chan struct{})

go func() {
    data = loadConfig()
    close(done)
}()

<-done
use(data)
```

如果读写会持续发生，用 mutex：

```go
type Store struct {
    mu   sync.RWMutex
    data string
}

func (s *Store) Set(v string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.data = v
}

func (s *Store) Get() string {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.data
}
```

如果是一次性初始化，用 Once：

```go
var once sync.Once
var cfg Config

func ConfigValue() Config {
    once.Do(func() {
        cfg = loadConfig()
    })
    return cfg
}
```

如果是低层状态位，且你能保证状态很简单，再考虑 typed atomic：

```go
var ready atomic.Bool

ready.Store(true)
if ready.Load() {
    // 只处理这个状态位能表达清楚的事情
}
```

不要为了显得“无锁”而把一组业务状态拆成一堆 atomic。那通常不是性能优化，是把未来的排查成本提前埋进去。

## 结尾：别问它会不会跑通，问它有没有链

Go 内存模型最有价值的地方，不是给你一套炫技词汇，而是把并发正确性压回一个问题：有没有 happens-before？

有，普通读写才有可见性证明。

没有，就算代码看起来顺、日志看起来对、amd64 上跑了很久没炸，也只是碰巧。

下次你再看到这种代码：

```go
data = x
ready = true
```

别急着说“这不挺清楚吗”。

先问一句：`ready=true` 是不是被同步动作发布出去的？

如果不是，它就只是一个普通写入。

普通写入不是承诺。同步才是。
