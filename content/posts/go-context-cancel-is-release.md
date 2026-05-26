---
title: "WithTimeout 的 cancel，不是你想象的那个“超时按钮”"
description: "很多 Go 代码写了 context.WithTimeout，却仍然出现 goroutine 上涨和资源保留。问题不在 timeout 没生效，而在 cancel 的职责、Done 的响应和排查路径被混在了一起。"
date: 2026-05-26T15:30:00+08:00
draft: false
author: "任博"
tags: ["Go", "context", "WithTimeout", "goroutine", "pprof"]
categories: ["技术实战"]
cover: "/images/go-context-cancel-is-release/cover.png"
toc: true
---

接口已经超时了，goroutine 曲线还在往上爬。

上篇讲到这里时，我们先压住了一个误判：这通常不是 `context` 没生效，而是你的 goroutine 没有响应取消信号。

但还有一个更隐蔽的问题，很多 Go 代码每天都在写：

```go
ctx, cancel := context.WithTimeout(parent, 200*time.Millisecond)
defer cancel()
```

这句 `defer cancel()` 到底是干什么的？

很多人心里其实把它理解成“超时按钮”。好像 cancel 是用来触发 timeout 的；既然 timeout 到点会自动发生，那函数里忘了调也没什么大不了。

这个理解只差一步，但差的正是事故里最贵的一步。

![WithTimeout cancel 不是超时按钮](/images/go-context-cancel-is-release/cover.png)

`WithTimeout` 的 cancel 不是超时按钮，是资源释放按钮。

时间到了，标准库会取消这个 context；但你的函数提前返回时，标准库不知道你这条调用链已经没用了。你不主动 cancel，它就只能等 deadline 到，或者等父 context 被取消。

这不是语法洁癖，是生命周期设计。

## timeout 会自动发生，不等于你可以不收尾

先把 `WithTimeout` 看薄一点。

Go 标准库里，`WithTimeout(parent, d)` 本质上就是：

```go
context.WithDeadline(parent, time.Now().Add(d))
```

它创建的是一个带 timer 的子 context。deadline 到了，timer 触发，context 的 `Done` 会关闭，`Err()` 会变成 `context deadline exceeded`。

所以“超时会自动发生”这句话没错。

错的是从这句话推导出：“那我不调用 cancel 也没事。”

看一个更贴近线上代码的场景：

```go
func query(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()

    return callDownstream(ctx) // 可能 50ms 就返回
}
```

你设了 10 秒超时，但下游 50 毫秒就返回了。

如果没有 `defer cancel()`，这个子 context 的 timer、父子引用，以及它下面可能挂着的 children，会继续留到 10 秒后，或者父 context 先被取消。

单看一次请求，这点资源可能不吓人。

但高 QPS 下，每一次“反正会自动超时”都会把资源多挂一会儿。挂一会儿不是泄漏到天荒地老，却足够让排查变得脏，让内存和 timer 压力变得没必要。

循环里这个问题更容易被放大。

很多人会在一个批处理或 fan-out 场景里这样写：每处理一条数据，就派生一个带 timeout 的子 context，然后把 `cancel` 丢到外层函数的 `defer` 里。代码看起来很优雅，退出时也确实会统一释放。

问题是，外层函数可能要跑很久。

如果这一轮任务 20 毫秒就结束，而它的 timeout 是 5 秒，那这 5 秒里的 timer 和父子关系都没有必要继续挂着。更糟的是，如果循环一口气跑几千轮，你会制造一批“已经完成业务，但还没完成生命周期收尾”的 context。

所以在循环里，`defer cancel()` 不一定是最好的写法。更稳的方式是让每一轮自己收尾：

```go
for _, item := range items {
    ctx, cancel := context.WithTimeout(parent, 5*time.Second)
    err := handleOne(ctx, item)
    cancel() // 这一轮结束，立刻释放
    if err != nil {
        return err
    }
}
```

这不是反对 `defer`。函数级短生命周期，`defer cancel()` 通常最稳；循环级生命周期，就要警惕把一堆本该立即释放的东西拖到函数最后。

规则不是“永远 defer”，而是“谁创建，谁明确收尾”。

官方文档说得很直：调用 CancelFunc 会取消子 context 和它的 children，移除父节点对子节点的引用，并停止关联的 timer；不调用 CancelFunc，会让 child 和 children 一直活到 parent 被取消。

这里要讲准。

忘记 cancel 不等于一定泄漏 goroutine。它更准确的代价是：你没有及时释放 context 这条生命周期链上的资源。

goroutine 会不会泄漏，是另一个问题：它有没有听见 `Done`，听见之后有没有退出来。

## cancel 不等业务真的停了

`CancelFunc` 的名字很容易误导人。

它叫 cancel，但它不会等工作停止。Go 文档里说得很直：`CancelFunc` 会告诉一个操作放弃工作，但不会等待这份工作真正停下来。

换到工程现场里，就是：cancel 只是告诉对方“别干了”，不是站在旁边等它真下班。

这段代码看起来规范：

```go
func handle(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
    defer cancel()

    go worker(ctx)
    return doSomething(ctx)
}
```

但如果 `worker` 是这样写的：

```go
func worker(ctx context.Context) {
    jobs := make(chan Job)
    for {
        job := <-jobs
        handleJob(job)
    }
}
```

那 `cancel()` 调得再漂亮也没用。

这个 goroutine 卡在 `<-jobs` 上，根本没有把 `ctx.Done()` 放进等待点。外面 deadline 到了，`Done` 关了，`Err()` 也有了，但它还在等下一份永远不会来的 job。

更稳的写法应该把业务等待和取消信号放到同一个 `select` 里：

```go
func worker(ctx context.Context, jobs <-chan Job) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case job := <-jobs:
            handleJob(job)
        }
    }
}
```

这才是协作式取消。

不是你把 `ctx` 传进去就算设计了生命周期，而是阻塞点真的有一条退出路。

Done 没被听见，timeout 就只是日志里的一句话。

很多代码骗过 review，靠的就是“表面上一路传了 ctx”。函数签名有，参数也有，调用链也有；但真正卡住的地方，是一个没 select 的 channel receive，一个自写的重试循环，一个没有 deadline 的第三方调用。

ctx 只是路过了现场，没有进入事故发生的房间。

还有一种更隐蔽：你确实监听了 `Done`，但监听的位置太外层。

比如外层循环有 `select`，可一旦进入 `handleJob(job)`，里面又做了一个没有 timeout 的网络调用，或者进入一个长时间计算的内部循环。此时取消信号已经到了，外层也有退出分支，但 goroutine 正卡在内层不可中断的工作里，还是退不出来。

所以检查 `Done` 不能只做文本搜索。

不要只问“有没有 `<-ctx.Done()`”。

要问：它离真正会阻塞的地方有多近？

如果等待点在 channel 上，`Done` 就要和 channel receive 放在同一个 `select`。如果等待点在外部调用上，就要确认这个调用自己支持 context 或 deadline。否则你只是把逃生门画在了走廊里，事故发生的房间还是没有出口。

## go vet 能抓低级错，但抓不住业务耳朵

`go vet` 很有用，但不要神化它。

它能抓住一类静态问题：你从 `WithCancel`、`WithTimeout`、`WithDeadline` 拿到了 `CancelFunc`，却没有在所有控制流路径上使用。

比如这段代码：

```go
func bad() {
    ctx, _ := context.WithTimeout(context.Background(), time.Second)
    _ = ctx
}
```

跑：

```bash
go vet ./...
```

会看到类似提示：

```text
the cancel function returned by context.WithTimeout should be called,
not discarded, to avoid a context leak
```

这类错误不该靠人眼硬看。尤其是分支提前 return、循环里创建子 context、错误路径漏释放，review 很容易扫过去。

但 `go vet` 的边界也要说清。

它能提醒你 cancel 没有被使用，不能证明你的 goroutine 会响应取消。下面这种代码，静态上可能很规矩：

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()

worker(ctx) // 里面可能完全不听 Done
```

`cancel` 调了，不代表现场真的会退。

所以 `go vet` 是第一道栅栏，不是最终判决。它负责把“不该犯的低级错”拦下来；业务代码有没有把 `Done` 放进真正的阻塞点，还得你回到调用栈里看。

我建议把 `go vet` 放在两个地方。

第一，放进本地习惯。写完涉及 `WithTimeout`、`WithCancel` 的代码，顺手跑一次。不要等 CI 才发现一个 cancel 被丢了。

第二，放进排查现场。线上 goroutine 涨了以后，你可能已经在 pprof 里看到某批 worker 卡住。这时回到对应包跑 `go vet`，能快速确认有没有明显的 lostcancel 问题。

但不要用它替代人工判断。

`go vet` 报了 lostcancel，说明你至少有一处收尾路径不完整；它没报，不说明你的生命周期设计就完整。静态工具能抓“cancel 有没有被调用”，抓不住“业务有没有把取消当回事”。

## pprof 看的是等待点，不是判决书

如果线上 goroutine 已经涨起来，别从源码想象根因，先抓 profile。

最小接入方式是：

```go
import _ "net/http/pprof"

func main() {
    go http.ListenAndServe("127.0.0.1:6060", nil)
    // ...
}
```

然后看 goroutine profile：

```bash
curl -s 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=1'
curl -s 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=2'
```

`debug=1` 更适合先看聚合摘要，哪些栈数量最多；`debug=2` 更适合看完整 goroutine 栈，确认它停在 channel、IO、select、锁，还是你自己的某个 worker 里。

但 pprof 也不会替你判案。

它告诉你的是：一批 goroutine 停在这里。

它不会直接告诉你：这是忘了 cancel，还是监听 Done 的位置写错了，还是某个第三方调用没有响应 ctx。

pprof 看到的是等待点，不是判决书。

真正的排查要回到三条线。

![context timeout 三条线排查法](/images/go-context-cancel-is-release/inline-01.png)

第一条线，创建路径。

谁调用了 `WithTimeout`？它的 parent 是谁？这个子 context 的生命周期应该跟函数一样短，还是跟一个异步任务一样长？如果在循环里创建，每一轮是不是都及时释放？

第二条线，取消路径。

返回的 `cancel` 在哪里调用？成功路径、错误路径、提前返回路径都能走到吗？如果不能简单 `defer`，是否有明确的 owner 负责收尾？

第三条线，响应路径。

所有可能长时间等待的 goroutine，有没有在真正阻塞的地方监听 `ctx.Done()`？不是函数签名上有 ctx，而是等待点上能听见它。

少看任何一条，结论都容易偏。

只看创建路径，你会把所有问题都归到“忘 cancel”。只看响应路径，你又可能忽略 timer 和父子引用长期保留。只看 pprof，你只能看到堆在哪里，看不到为什么没人收。

排查 context 相关问题，最怕一个词：应该。

“应该超时了。”

“应该 cancel 了。”

“应该退出了。”

线上问题不认应该，只认路径。

## 一份可以直接拿去用的排查顺序

如果你现在手上就有一个 Go 服务，日志里已经出现 `context deadline exceeded`，goroutine 数也不太对，不要一上来改代码。

按这个顺序走。

第一步，确认趋势。

看 goroutine 数是不是只在流量高峰上去、低峰能下来。如果能下来，它未必是泄漏；如果压测停了还不回落，再进入下一步。

第二步，抓 pprof。

先用 `debug=1` 看哪类栈最多，再用 `debug=2` 看完整等待点。你要找的不是“context”这个词，而是 goroutine 停在了哪里：channel receive、select、锁、网络 IO，还是某个业务 worker。

第三步，回代码查创建路径。

找到这批 goroutine 对应的上游，查谁创建了 context。特别看循环、fan-out、重试、异步任务这几类代码。它们最容易在“每一轮都创建子 context”之后，没有及时释放。

第四步，查取消路径。

每个 `CancelFunc` 都应该有 owner。成功返回谁调？错误返回谁调？提前退出谁调？异步任务完成谁调？如果回答是“理论上会调”，那就继续找，直到路径落到具体代码。

第五步，查响应路径。

把所有可能长时间等的点列出来：队列、channel、外部请求、数据库查询、锁、内部循环。逐个确认它们能不能因为 `ctx.Done()` 退出。不能退出的地方，才是真正会留下 goroutine 的地方。

这套顺序的价值，不是显得严谨。

它能防止你把三类问题混成一锅：

- 忘 cancel：资源释放路径问题；
- 没听 Done：业务响应路径问题；
- pprof 看到堆积：等待点定位问题。

三件事有关联，但不是一件事。

把它们拆开，事故才会变小。

## 几个最容易写错的边界

第一，`cancel` 可以多次调用。

标准库保证第一次之后的调用不再产生效果。所以你不必为了“会不会重复 cancel”把代码写得很绕。真正该担心的不是多调一次，而是某条错误路径一次都没调。

第二，`ctx.Err()` 不是根因。

它只告诉你这条 context 为什么结束：主动取消，还是 deadline 到了。它不会告诉你哪个 worker 没退、哪个 channel 没人关、哪个调用没有响应取消。把 `context deadline exceeded` 当成根因，排查会停得太早。

第三，不是所有工作都能被 context 立刻打断。

如果代码已经进入一段不检查 ctx 的 CPU 循环，或者调用了一个不接收 context 的库函数，外面的 cancel 只能改变 context 状态，不能凭空插进那段代码。你要么把检查点放进去，要么给那层调用换成支持 context/deadline 的 API。

这几个边界看起来细，其实都是同一个判断：context 管信号，不管强制执行。

## 最后，记住这三个问题

上篇我们说，`context` 不是清洁工，它只是通知协议。

这一篇再把 `WithTimeout` 的边界往里压一层：timeout 到点会触发取消，但提前收尾这件事，仍然是你的责任。

下次你看到这句代码：

```go
ctx, cancel := context.WithTimeout(parent, d)
defer cancel()
```

不要把它当模板。

把它拆成三个问题：

`cancel` 负责释放什么？

`Done` 有没有被真正听见？

pprof 里的等待点，能不能沿着创建、取消、响应三条线解释清楚？

如果这三个问题能对上，`context` 就不是一团玄学。

它只是很克制地要求你把生命周期写明白。

下一篇会继续拆 `Context` 里另一个容易被滥用的口子：`WithValue`。它看起来像一个顺手的参数袋，但真正在源码里，它不是 map，而是一条链。很多调用链变浑，就是从“顺手塞一下”开始的。

如果你还没看上篇，可以先回到那篇：为什么超时到了，goroutine 还在涨。这个系列后面会继续沿着 `context` 的源码和线上排查现场往下拆，建议关注起来，不然下次看到 `context deadline exceeded`，你很可能还是只盯着错误字符串看。
