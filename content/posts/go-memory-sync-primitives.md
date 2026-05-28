---
title: "channel 不只是队列，Mutex 不只是锁"
description: "Go 内存模型下篇：从 channel、Mutex、Once、WaitGroup、atomic 到 race detector，讲清同步原语真正给你的 happens-before 保证。并发代码不是看起来交接了就安全，规范承认可见性，读取才站得住。"
date: 2026-05-28T14:52:00+08:00
draft: false
author: "任博"
tags: ["Go", "并发编程", "内存模型", "channel", "mutex", "atomic"]
categories: ["技术实战"]
cover: "/images/go-memory-sync-primitives/cover.png"
toc: true
---

代码评审里经常出现这种争论：一个 goroutine 写完结果，`close(done)`；另一个 goroutine 等 `<-done` 再读结果。有人看着觉得多此一举：这不就是通知一下吗？换成一个 `bool`，或者 sleep 一小会儿，不也一样？

不一样。

上篇我们说过，`ready=true` 不是发布。普通变量上的“先写后读”，不会自动变成 goroutine 之间的可见性保证。

这一篇往下走一步：Go 里那些我们天天用的同步原语，到底分别保证了什么？

答案比“channel 是队列，Mutex 是锁”要深一点。

同步原语传递的，不只是控制权，还有可见性。

如果你只把 channel 当队列，只把 Mutex 当门闩，只把 atomic 当“无锁版 Mutex”，并发代码迟早会写得很玄学：能跑，但说不清为什么安全；出事，也说不清哪里断了链。

![同步原语建立可见性证明链](/images/go-memory-sync-primitives/inline-01.svg)

## channel：它不是队列，是一次交接

很多人喜欢把 channel 理解成队列。这个理解不算错，但太窄。

在 Go 内存模型里，channel 真正重要的是几条同步规则：

```text
send happens-before 对应 receive 完成
close happens-before 因关闭而返回零值的 receive
无缓冲 channel 上，receive happens-before 对应 send 完成
容量为 C 的 buffered channel 上，第 k 次 receive happens-before 第 k+C 次 send 完成
```

先看最常见的一种写法：

```go
type Result struct {
    Value string
    Err   error
}

ch := make(chan Result, 1)

go func() {
    r := Result{Value: "ok"}
    ch <- r
}()

r := <-ch
fmt.Println(r.Value)
```

这段代码安全，不是因为 channel “刚好把值传过去了”。真正的保证是：发送动作 happens-before 对应的接收完成。

也就是说，发送方在 `ch <- r` 之前准备好的内容，接收方在 `<-ch` 之后可以按这条同步链去理解。

这就是 channel 比普通共享 slice + bool 强的地方。

你把任务塞进共享 slice，再写一个 `hasJob = true`，消费者轮询这个 bool，看起来也像一个小队列。但 Go 规范不承认这个交接。slice 的写入、bool 的写入、消费者的读取，中间没有同步动作。

代码形状像队列，不等于它有队列该有的保证。

`close(done)` 也是同一个道理：

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

这里的 `close(done)` 不只是“告诉别人我结束了”。它 happens-before 那个因为 channel 已关闭而返回的 receive。于是 `data = loadConfig()` 通过同一个 goroutine 内的程序顺序接到 `close(done)`，再接到 `<-done` 返回，最后接到 `use(data)`。

链闭合了，读取才站得住。

所以 channel 最适合表达两类事情：所有权转移，以及完成信号。前者是“这个对象交给你了”，后者是“我之前做的事现在对你可见”。

它不是高级队列。

它是带同步语义的通信原语。

## Mutex：Unlock 不只是开门

Mutex 更容易被低估。

很多人理解锁，只停在“同一时间只能一个 goroutine 进入临界区”。这当然对，但还不够。对同一个 `sync.Mutex` 或 `sync.RWMutex` 来说，一次 `Unlock` synchronizes-before 后续某次 `Lock` 返回。

换句话说，`Unlock` 不只是把门打开。它还把门内发生过的写入发布出去。

```go
type Store struct {
    mu sync.RWMutex
    v  string
}

func (s *Store) Set(v string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.v = v
}

func (s *Store) Get() string {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.v
}
```

这段代码里，锁做了两件事。

第一，互斥。写的时候不会有人同时乱读乱写。

第二，可见性。写入方释放锁之前对 `s.v` 做的修改，对后续拿到同一把锁的读者可见。

如果只看到第一层，你很容易产生一种错觉：只要我保证“差不多不会同时写”，就行了。比如用一个普通 bool 标记“正在更新”，或者靠调用顺序约定“先更新再读取”。

这些都不是 Mutex。

Mutex 的价值不只是挡住并发，而是给共享状态划出一个规范承认的边界。进这个边界读写，出来的时候发布；下一次再进去的人，看到的是这条链之后的世界。

这也是为什么一组字段共同构成业务不变量时，Mutex 往往比一堆 atomic 更容易把话说清楚。

比如一个熔断器状态：

```go
type Breaker struct {
    mu        sync.Mutex
    state     State
    failures  int
    lastError time.Time
}
```

`state`、`failures`、`lastError` 不是三个孤立数字。它们合起来才说明熔断器现在处在什么阶段。

如果你把它们拆成三个 atomic 字段，单次读写也许都安全，但你未必读到一张一致快照。新 `state` 配旧 `failures`，旧 `lastError` 配新状态，这种组合在业务上可能根本不存在。

atomic 能保证单个操作的原子性和顺序一致语义，但它不会自动替你维护复合不变量。

## Once：别手写“我觉得只会初始化一次”

懒加载代码里，最常见的危险写法长这样：

```go
var inited bool
var client *Client

func GetClient() *Client {
    if inited {
        return client
    }
    client = newClient()
    inited = true
    return client
}
```

这段代码的危险不只是“可能初始化两次”。更麻烦的是：另一个 goroutine 看到 `inited=true`，不代表它一定看到 `client` 已经完整初始化。

`ready=true` 不是发布，`inited=true` 也不是。

标准答案是 `sync.Once`：

```go
var once sync.Once
var client *Client

func GetClient() *Client {
    once.Do(func() {
        client = newClient()
    })
    return client
}
```

`Once` 的文档保证很明确：真正执行的那个函数返回，synchronizes-before 任何一次 `Do` 调用返回。

这句话翻成工程语言就是：只要 `GetClient` 返回了，你就不用再猜 `client` 的初始化对当前 goroutine 是否可见。

很多人手写 double-checked locking，是觉得自己能省一点锁开销，或者觉得这段逻辑太简单，不值得上 `Once`。

问题是，并发代码里最贵的往往不是锁，而是你后来解释不清它为什么安全。

`sync.Once` 不是保守，是省脑子。

省脑子，在并发代码里通常就是省事故。

## WaitGroup：等的是结束，不是保护过程

`WaitGroup` 很容易被误用，因为它的名字里有一个 “Wait”。

它确实能等一组 goroutine 结束。并且文档也说明，`Done` 会 synchronizes-before 它解除阻塞的 `Wait` 返回；Go 1.25 里的 `WaitGroup.Go` 也把函数返回和 `Wait` 返回之间的同步关系说清楚了。

所以这种模式是成立的：

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

所有 worker 都写完，`Wait` 返回后再统一读取结果。这条链说得通。

但 WaitGroup 不是锁。

WaitGroup 等的是结束，不是保护过程。

如果 worker 还在写 `results`，另一个 goroutine 在 `Wait` 返回前就去读它，依然可能形成 data race。`WaitGroup` 不会因为名字里有 Wait，就替你保护执行期间的共享状态。

这类误用在线上很常见：主流程起了几个后台 worker，然后想“反正最后会 Wait”，中间就顺手读取一下共享 map、共享 slice、共享计数器。

这不是等待结束。

这是在过程里并发读写。

如果你需要执行期间读写共享状态，用 Mutex、channel 所有权转移，或者设计成不可变快照。不要让 WaitGroup 承担它没有承诺过的事。

## atomic：更低层，不是更高级

atomic 最容易被神化。

它名字里有“原子”，看起来又不像锁那样阻塞，于是很多人会自然得出一个结论：能用 atomic，就比 mutex 高级。

这个判断很危险。

Go 的 atomic 文档给了一个很强、也很克制的保证：如果 atomic 操作 A 的效果被 atomic 操作 B 观察到，那么 A synchronizes-before B；并且所有 atomic 操作表现得像按某个顺序一致的总顺序执行。

这让 atomic flag 可以做简单发布：

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

当 `Load` 观察到 `Store(true)` 的结果后，同步链就建立起来了。

但注意前提：这里的状态非常简单。一个 flag 表达一件事，读者看到 flag 后读取前面发布的数据。

一旦你开始维护多个字段，事情就变了：

```go
var count atomic.Int64
var sum atomic.Int64

c := count.Load()
s := sum.Load()
fmt.Println(c, s)
```

每次 `Load` 都是 atomic 的，不代表 `count` 和 `sum` 组成了一张一致快照。你可能读到新的 `count`，配上旧的 `sum`。

这不是 atomic 坏。

是你让它干了 mutex 或不可变 snapshot 更适合干的事。

如果多个字段共同构成业务含义，通常有三种更稳的做法：

```go
// 1. 用 mutex 包住整组状态
mu.Lock()
state := current
mu.Unlock()

// 2. 用 channel 交出所有权
updates <- snapshot

// 3. 用 atomic.Pointer 一次发布不可变快照
ptr.Store(&Snapshot{Count: c, Sum: s})
```

atomic 更低层，不是更高级。

它适合表达很窄、很清楚的同步关系；不适合把复杂状态拆碎以后，假装每一片安全，整体就安全。

## race detector：烟雾报警器，不是结构图

`go test -race` 很有用。

并发改动上线前，我会把它当成必跑项。常用命令也不复杂：

```bash
go test -race ./...
go test -race -run TestName ./pkg
go run -race ./cmd/demo
go build -race ./cmd/server
```

但它的边界必须说清楚：race detector 是运行时检测，不是静态证明。

它会在 `-race` 编译时对内存访问插桩，运行时记录读写关系，发现没有同步保护的冲突访问就报告。Go 官方文档也说明，它只能发现实际执行到的 race。

没报警不代表没火，可能只是你没把那条路径跑出来。

官方文档给过一个开销量级：内存大约 5 到 10 倍，执行时间大约 2 到 20 倍。所以它适合放在 CI、压力测试、复现环境和关键并发改动回归里，不适合默认开在所有生产进程上。

race 报告出来后，不要只看第一行 `WARNING: DATA RACE`。

真正要看的有两类栈：冲突访问发生在哪里，以及相关 goroutine 是在哪里创建的。前者告诉你哪两个读写打架，后者告诉你它们为什么会同时存在。

还有一个工具边界也顺手讲清楚：`go tool trace` 不检测 data race。

它看的是 runtime execution trace：调度、syscall、网络阻塞、sync blocking、锁竞争、利用率。它能帮你分析“为什么卡”“谁在等”，但不会替代 `-race` 判断数据竞争。

工具边界可以这样记：

```text
-race：抓实际执行到的数据竞争
go tool trace：看调度和阻塞
goleak：测 goroutine 是否泄漏
go vet / staticcheck：做静态质量检查
```

把工具用错方向，会给你一种很危险的安全感。

## 真正该看的，是交接点

内存模型听起来很抽象。落到工程里，大多数并发模式都在问同一个问题：谁写，谁读，中间靠什么交接？

生产者消费者是交接。

```go
jobs := make(chan Job)

go func() {
    job := buildJob()
    jobs <- job
}()

job := <-jobs
handle(job)
```

安全点不在“生产者消费者”这个名字上，而在 `jobs <- job` 和 `<-jobs` 之间的同步关系上。

配置热更新也是交接。

如果配置构建完成后，通过 `atomic.Pointer[Config]` 一次发布不可变对象，读者每次 Load 到的是一整份配置快照。这个模型能讲清楚。

如果你一边改配置对象里的字段，一边让其他 goroutine 持有同一个指针读它，就很难讲清楚。

熔断器、限流器、缓存刷新、后台 worker 协调，本质上都一样。

不要先问“这个模式叫什么”。

先看交接点。

共享状态跨 goroutine 流动时，必须有一个同步动作负责盖章。没有这个章，模式再经典也只是长得像。

## 写并发代码时，按这五问自查

如果你不想每次都翻内存模型，可以先用这五个问题过滤大多数风险。

第一，共享变量是不是被多个 goroutine 访问，而且至少一边在写？

如果是，别急着看代码“顺不顺”。继续问下一句。

第二，这些访问之间有没有明确同步？

你要找的是 channel send/receive/close、同一把 mutex 的 Unlock/Lock、Once、WaitGroup 的 Done/Wait、atomic Store/Load 这类规范承认的同步关系。不是 `sleep`，不是普通 bool，也不是“按理说应该先执行”。

第三，你保护的是单个值，还是一组不变量？

单个状态位可以考虑 atomic。一组字段、一个 map、一个对象生命周期，优先考虑 mutex、channel 所有权转移，或者一次发布不可变快照。

第四，你是在等待结束，还是在保护过程？

WaitGroup 只解决结束之后的可见性，不解决执行期间的并发读写。

第五，你有没有把 race detector 当成证明？

`-race` 没报警，只能说明这次运行没有抓到。真正的安全证明，还是读写之间有没有 happens-before。

这五问比背一堆术语有用。

回到开头那个问题：为什么不用一个普通 bool 替代 `close(done)`？

因为普通 bool 只是一个值。

`close(done)` 是一个同步动作。

并发代码里，值只能表达状态；同步动作才能发布状态。

如果你正在写 Go 服务初始化、配置热更新、缓存刷新、后台 worker 协调这一类代码，建议把这组文章收藏起来。内存模型不需要每天挂在嘴边，但每次共享状态跨 goroutine 时，你都要知道自己在向规范借哪一条保证。
