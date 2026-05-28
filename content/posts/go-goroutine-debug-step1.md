---
title: "goroutine 为什么不能靠猜：Go 排查第一步要先用 NumGoroutine 和 pprof 采样"
description: "goroutine 涨了，别直接怀疑 context 泄漏。先用 NumGoroutine 确认趋势，再用 pprof 把数量变成栈，最后用 go vet 抓住静态路径上的低级错误——这个顺序不能反。"
date: 2026-05-27T18:16:00+08:00
draft: false
author: "任博"
tags: ["Go", "goroutine", "pprof", "context", "后端工程"]
categories: ["技术实战"]
cover: "/images/go-goroutine-debug-step1/cover.png"
toc: true
---

线上 goroutine 数开始往上爬，很多人的第一反应是猜。

是不是 `context` 泄漏？是不是超时没生效？是不是某个 channel 没人读？是不是锁没释放？

这些猜法都可能对。

但问题在于，你现在还没有证据。你只有一个数字：goroutine 变多了。

`runtime.NumGoroutine()` 只能告诉你当前有多少 goroutine。它不会告诉你这些 goroutine 停在哪里，不会告诉你谁创建了它们，也不会告诉你为什么没有退出。

所以排查这类问题，第一步不是解释 `context` 原理，也不是翻代码找可疑点。

第一步是采样。

![goroutine 排查最小链路](/images/go-goroutine-debug-step1/inline-01.png)

这篇只讲上半段：从 goroutine 数上涨开始，怎么用一条最小链路把问题从“感觉不对”推进到“有栈可看”。

这条链路很短：

```text
goroutine 涨了
  ↓
先采样，确认趋势
  ↓
接入 pprof，抓 goroutine profile
  ↓
用 go tool pprof 看占比
  ↓
跑 go vet，抓静态错误
```

别急着给事故起名字。

**没有采样，所有判断都只是下注。**

## 第一步：先确认它是不是真的在涨

先做一个最小采样。

```go
package main

import (
    "log"
    "runtime"
    "time"
)

func startGoroutineSampler() {
    ticker := time.NewTicker(30 * time.Second)

    go func() {
        defer ticker.Stop()
        for range ticker.C {
            log.Printf("goroutines=%d", runtime.NumGoroutine())
        }
    }()
}
```

这段代码不是用来定位根因的。

它只回答一个问题：goroutine 数到底是不是持续上涨？

如果请求高峰时上去，低峰时能回来，这未必是泄漏。它可能只是并发量变大，服务还在正常消化请求。

真正值得警惕的是另一种曲线：流量停了，压测停了，业务也没继续增加，但 goroutine 数还挂在那里，甚至继续慢慢爬。

这才说明你需要往下查。

采样时别只盯一个孤立数字。最好把 goroutine 数和请求量、错误率、延迟放在同一个时间窗口里看。

如果请求量涨，goroutine 跟着涨，请求量回落后 goroutine 也回落，这更像容量和并发问题。如果请求量回落了，goroutine 没回落，才更像有东西没退出。如果错误率开始上升，goroutine 也开始堆，优先看外部依赖、连接池、网络读写和下游超时。

还有一个小细节：采样间隔不要太密。每秒打一次日志，看起来很积极，实际很快就把日志淹没了。30 秒或 1 分钟一条，配合监控曲线，通常已经足够看趋势。

你要的不是“更密的数字”，而是“能说明问题的趋势”。

这里要先压住一个误判：

**goroutine 数一涨，不等于 context 泄漏。**

它也可能是 channel 没人读，锁竞争，网络读写卡住，后台任务堆积，连接池配置不合理。`context` 相关问题只是其中一类。

很多排查跑偏，就是因为一开始把“现象”当成了“结论”。

`NumGoroutine()` 只能把你送到门口。真正进门，要靠 pprof。

## 第二步：接入 pprof，把数量变成栈

最小接入方式，是引入 `net/http/pprof`，单独起一个调试端口。

```go
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof"
)

func startPprof() {
    go func() {
        log.Println(http.ListenAndServe("127.0.0.1:6060", nil))
    }()
}
```

注意这个地址：`127.0.0.1:6060`。

生产环境不要把 pprof 裸奔暴露到公网。更稳的做法是只绑定本机或内网地址，再通过跳板机、端口转发、临时安全组规则访问。

接上 pprof 后，先抓 goroutine profile。

```bash
curl -s 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=1'
curl -s 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=2'
```

这两个命令看起来只差一个数字，排查时差别很大。

`debug=1` 更适合看聚合后的栈摘要。你可以先判断哪一类 goroutine 数量最多。

`debug=2` 更适合看完整 goroutine 栈。它会把等待状态、阻塞位置、创建位置都打出来。

换句话说：

`debug=1` 帮你看“哪一堆最多”。

`debug=2` 帮你看“这一堆具体死在哪”。

看一个最小例子。

```go
func worker(ch <-chan int) {
    // bug：上游已经取消了，但这个 goroutine 完全不听。
    <-ch
}

func main() {
    ch := make(chan int)

    for i := 0; i < 20; i++ {
        ctx, cancel := context.WithCancel(context.Background())
        go worker(ch)
        cancel()
        _ = ctx
    }

    select {}
}
```

这段代码的问题，不是 `cancel()` 没调用。

它调用了。

真正的问题是 `worker` 根本没有接收 `ctx`，也没有监听 `ctx.Done()`。上游把“该停了”这句话送出去了，但 worker 在另一个房间里，门都没开。

抓 `debug=1`，你可能看到这样的摘要：

```text
goroutine profile: total 24
20 @ 0x10234ffa0 0x1022e8868 0x1022e8434 0x1024f37dc 0x102357e04
#   0x1024f37db    main.worker+0x2b    scenarios/b_ignore_done/main.go:16
```

这已经给出第一条线索：24 个 goroutine 里，有 20 个都停在 `main.worker`。

再看 `debug=2`：

```text
goroutine 9 [chan receive]:
main.worker(...)
    scenarios/b_ignore_done/main.go:16
created by main.main in goroutine 1
    scenarios/b_ignore_done/main.go:27 +0x48
```

这里有三个关键信息：

1. 等待状态是 `[chan receive]`；
2. 阻塞位置是 `main.worker` 第 16 行；
3. 创建位置是 `main.main` 第 27 行。

这比“goroutine 涨了”有用太多。

你现在不是在猜 `context`。你已经拿到了等待点和创建点。

读 goroutine 栈时，先看方括号里的等待状态。它经常比函数名更早告诉你方向。

看到 `[chan receive]`，先问谁该发送、为什么没人发送、这个 channel 会不会永远没人关闭。

看到 `[IO wait]`，先看网络读写、下游响应、超时是否真的传到了调用链里。

看到 `[select]`，不要只说“卡在 select”。要继续看这个 select 里有哪些 case，哪个 case 本该退出，`ctx.Done()` 有没有在里面。

看到 `[semacquire]`，再去想锁、WaitGroup、runtime semaphore 相关问题。

这些都还不是最终结论，但它们会把你的搜索范围砍掉一大半。

排查最怕的是一上来全局搜索 `context`。代码库里到处都是 `ctx`，你搜出来的是噪音，不是线索。

## 第三步：用 go tool pprof 看占比

goroutine 不多时，`debug=2` 直接看就够了。

但线上 profile 往往很长。几百个、几千个 goroutine 混在一起，肉眼翻文本很容易漏掉主线。

这时可以让 `go tool pprof` 帮你聚合。

```bash
go tool pprof http://127.0.0.1:6060/debug/pprof/goroutine
```

进入交互模式后，常用这几个命令：

```text
(pprof) top
(pprof) traces
(pprof) list worker
```

也可以直接看 top：

```bash
go tool pprof -top http://127.0.0.1:6060/debug/pprof/goroutine
```

示例输出里，可能会有这样几行：

```text
Showing nodes accounting for 24, 100% of 24 total
      flat  flat%   sum%        cum   cum%
        23 95.83% 95.83%         23 95.83%  runtime.gopark
         0     0%   100%         20 83.33%  main.worker
         0     0%   100%         20 83.33%  runtime.chanrecv
```

不要被 `runtime.gopark` 吓到。

它只是 Go runtime 把 goroutine 停起来的底层位置。你真正要盯的，是业务函数和等待类型。

这里的主线是：`main.worker` 占了 20 个 goroutine，底层落到 `runtime.chanrecv`。

换成人能听懂的话就是：一批 worker 卡在 channel receive 上。

这里还要注意一个容易误读的点：goroutine profile 里的 `top` 不是 CPU profile 的 `top`。

CPU profile 里，你看到的是时间花在哪里。goroutine profile 里，你更多是在看“有多少 goroutine 停在某类栈上”。所以不要看到 `flat%`、`cum%` 就按性能热点去解释。

这时最有价值的不是“runtime.gopark 占比很高”，而是“哪一个业务函数反复出现在同一批栈里”。

如果 `main.worker`、`readLoop`、`consumeMessage`、`waitForResult` 这种函数反复出现，你就有方向了。接下来回代码时，不是从整个项目开始翻，而是从这些函数的阻塞点和创建点开始翻。

如果输出里全是 runtime 函数，看不见业务函数，也别急着下结论。先用 `traces` 看完整调用链，或者用 `debug=2` 回到原始栈。很多时候业务函数不在第一屏，但在创建链路和调用链路里。

这一步的价值，不是让你背 pprof 输出。

它让排查从“可能是 context”变成了“这批 goroutine 卡在某个业务函数的某个等待点上”。

**排查不是猜得更准，而是让证据越来越窄。**

## 第四步：go vet 是静态闸门，不是侦探

pprof 是现场采样。

`go vet` 是静态闸门。

它们不是一类工具，不要混着期待。

`go vet` 对 `context` 最有用的一点，是能检查 `CancelFunc` 是否被丢弃。尤其是 `WithCancel`、`WithTimeout`、`WithDeadline` 返回的 cancel，有没有在所有控制流路径上使用。

比如这段：

```go
func main() {
    fmt.Println("goroutines before:", runtime.NumGoroutine())

    ctx, _ := context.WithTimeout(context.Background(), time.Minute)
    _ = ctx

    fmt.Println("goroutines after:", runtime.NumGoroutine())
}
```

跑：

```bash
go vet ./...
```

它会直接指出问题：

```text
scenarios/a_lost_cancel/main.go:14:7: the cancel function returned by context.WithTimeout should be called, not discarded, to avoid a context leak
```

Go 官方 `context` 文档说得很硬：调用 `CancelFunc` 会取消 child 和它的 children，移除 parent 对 child 的引用，并停止关联 timer；不调用 `CancelFunc`，child 和 children 会一直保留到 parent 被取消。`go vet` 会检查 CancelFunc 是否在所有控制流路径上使用。

所以这个习惯要变成肌肉记忆：

```go
ctx, cancel := context.WithTimeout(parent, 2*time.Second)
defer cancel()
```

尤其是函数提前 return、循环里创建 timeout context、错误路径分支很多时，`go vet` 往往比肉眼 review 靠谱。

比如这种代码，肉眼很容易漏掉错误分支：

```go
func query(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)

    if err := check(); err != nil {
        return err
    }

    defer cancel()
    return call(ctx)
}
```

`defer cancel()` 写了，但写晚了。`check()` 提前返回时，cancel 根本不会执行。

更稳的写法是创建之后立刻安排释放：

```go
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
defer cancel()
```

如果是在循环里创建 context，也不要机械套 `defer`。循环次数很多时，`defer` 会拖到外层函数退出才执行，资源释放被拉长。可以把每次循环包进一个小函数，或者在本轮处理结束后显式 `cancel()`。

`go vet` 的价值就在这里：它帮你守住“所有控制流路径都要用到 cancel”这条底线。

但边界也要说清楚。

**go vet 能抓丢掉的 cancel，抓不住不听话的 goroutine。**

它能抓“你把 cancel 扔了”。

它抓不住“你的 goroutine 收到了 ctx，却没有在阻塞点监听 `ctx.Done()`”。它也不会替你判断业务 goroutine 为什么卡在 channel 上。

所以 `go vet` 通过，不代表没有 context 相关泄漏。

它只说明某一类静态错误暂时没被发现。

## 这个阶段，先别急着修

做到这里，很多人会忍不住马上改代码。

看到 channel receive，就想把 channel close 掉。

看到 cancel 丢了，就到处补 `defer cancel()`。

看到某个 worker 数量最多，就想先把它的并发调小。

这些动作不一定错，但很容易把现场弄乱。

排查的前半段，目标不是修复，而是把证据固定下来。至少先保留三样东西：采样曲线、goroutine profile、`go vet` 输出。

这三样东西会回答三个不同问题。

采样曲线回答：它是不是真的持续上涨？上涨和流量、错误率、延迟有没有时间关系？

goroutine profile 回答：最多的一批 goroutine 停在哪里？等待状态是什么？创建点在哪里？

`go vet` 回答：有没有能静态抓出来的 cancel 路径问题？

这三个问题没回答完，就急着改，很容易出现一种尴尬结果：你改了代码，goroutine 数暂时下去了，但你不知道是哪个改动生效，也不知道线上同类问题会不会再回来。

更糟的是，你可能只消掉了表面现象。

比如 worker 卡在 channel receive，你把 channel close 了，goroutine 退出了；但真正的问题可能是上游任务派发和退出协议没有设计清楚。下一次换一个 channel，问题还会回来。

所以这里有个判断：

**修复前先留证据，不然你只是把现场打扫干净了。**

等证据链收窄，再回代码，修复才有方向。

这也是我更建议把这套流程写进团队 runbook 的原因。

线上排查最怕靠个人经验。熟悉 Go 的人知道先抓 pprof，新同事可能只会看监控；做过几次事故的人知道先留现场，没经历过的人会急着重启服务。靠经验传话，迟早会漏。

runbook 不需要写得复杂，五行就够：先记录时间窗口，再保存 goroutine 曲线，再抓 `debug=1` 和 `debug=2`，再跑 `go tool pprof -top`，最后补一份 `go vet ./...` 输出。

这五步的意义，是让团队在最慌的时候也按同一条路往前走。

排查流程真正的价值，不是显得专业。

是少走弯路，也少背黑锅，尤其是在半夜线上报警的时候。

很多线上问题最后都不是“技术太难”，而是排查顺序乱了。一个人看日志，一个人翻代码，一个人怀疑最近上线，另一个人去调连接池。每个人都在忙，但证据没有汇合，结论只是在群里来回飘。

把顺序固定下来，团队才会围着同一组证据讨论：曲线说明什么，栈停在哪里，静态检查有没有报错，下一步该看哪段代码。这样排查会慢一点开头，但后面会快很多。

## 最后，把顺序守住

goroutine 涨了以后，最容易犯的错，是太快解释。

一看到 `context`，就怀疑 cancel。

一看到 channel，就怀疑没人 close。

一看到 `runtime.gopark`，就怀疑 runtime 出问题。

这些反应都很正常，但都太早。

下次线上 goroutine 数开始爬，先按这个顺序走：

```text
1. 用 NumGoroutine 采样，看它是不是真的持续上涨
2. 接 pprof，抓 goroutine profile
3. 用 debug=1 看聚合，用 debug=2 看完整栈
4. 用 go tool pprof 看占比，不靠肉眼翻长文本
5. 跑 go vet，把 lost cancel 这类静态错误先拦住
```

到这一步，你还没真正修 bug。

但你已经完成了最重要的一件事：把问题从“我猜可能是 context”推进到了“我知道这批 goroutine 卡在哪里”。

后面才轮到代码分析。

下一篇会继续往下走：拿到 pprof 和 vet 结果后，怎么按“创建路径、取消路径、响应路径”三条线回到代码，判断到底是 cancel 没走到，还是 goroutine 根本没听 `ctx.Done()`。

如果你也经常遇到 Go 并发、context、pprof 这类线上排查问题，可以关注我。后面这组文章会继续按真实排查路径拆，不讲玄学。 
