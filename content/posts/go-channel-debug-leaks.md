---
title: "goroutine 泄漏为什么不一定是忘了 cancel：Go channel 还要看 sender 和 receiver 谁没回家"
description: "channel 导致的 goroutine 泄漏通常分两种方向：sender 发了没人接，或者 receiver 等了没人发。排查时要走出三条线才能锁定代码里的真实责任方。"
date: 2026-05-28T15:20:00+08:00
draft: false
author: "任博"
tags: ["Go", "channel", "并发编程", "goroutine泄漏", "select", "pprof"]
categories: ["技术实战"]
cover: "/images/go-channel-debug-leaks/cover.png"
toc: true
---

goroutine 泄漏最讨厌的地方，不是它会炸，而是它不炸。

它先是悄悄多几个，再多几十个，最后把内存、延迟、CPU 一起拖下水。你回头看日志，发现代码没报错；你再看监控，发现曲线只是在往上爬。

很多人看到这种情况，第一反应是：是不是忘了 cancel。

有时候是。但不总是。

更常见的真实原因是：channel 这头还在发，那头已经不收了；或者 range 还在等 close，但 close 根本不会来；或者 select 里放了一个 default，把等待变成了空转。

这篇只做一件事：把“goroutine 为什么没退出”拆成四个场景讲清楚，再给你一套能落地的退出规则。

![goroutine 泄漏排查主图](/images/go-channel-debug-leaks/cover.svg)

## 一、先把问题说准：泄漏不是“线程多了”，是“没人回家了”

goroutine 数量上升，不等于一定有 bug。

真正值得警惕的是两种信号：

- 高峰过去了，goroutine 数量却不回落
- pprof 里堆着一批 `chan send` 或 `chan receive`

前者说明有生命周期没收住，后者说明你已经碰到阻塞点了。

排查时别先盯业务逻辑。先看两样东西：

```go
runtime.NumGoroutine()
```

和：

```bash
curl -s http://localhost:6060/debug/pprof/goroutine?debug=1
```

一个看趋势，一个看栈。趋势告诉你“有事发生了”，栈告诉你“卡在哪”。

## 二、Goroutine 泄漏，最常见就两种方向

### 1）sender 还在发，receiver 已经走了

这是最常见的泄漏。

```go
func leakySender() {
    ch := make(chan int)
    go func() {
        for i := 0; ; i++ {
            ch <- i
        }
    }()

    for i := 0; i < 5; i++ {
        fmt.Println(<-ch)
    }
    // 这里返回后，没人再收 ch
}
```

表面上看，循环只发了几个值；实际上，后面的 sender goroutine 还在原地等。

这类问题最容易被 buffer 误导。buffer 大一点，现象晚一点出现，于是你以为修好了。其实只是把卡住的时刻往后拖。

### 2）receiver 还在等，sender 已经不发了

```go
func leakyReceiver() {
    ch := make(chan int)
    go func() {
        ch <- 1
        ch <- 2
        // 忘了 close(ch)
    }()

    for v := range ch {
        fmt.Println(v)
    }
}
```

`range ch` 的语义很直接：一直读，直到 channel 被关闭。

所以 `range` 不退出，通常不是它坏了，而是 close 没来。

![channel 退出路径示意图](/images/go-channel-debug-leaks/inline-01.svg)

## 三、Dead select case：看起来在等，其实根本不会等到

`select` 里最危险的不是没有 case，而是你以为某个 case 会触发，实际上它永远触发不了。

### 场景一：default 把等待变成忙等

```go
for {
    select {
    case v := <-workCh:
        process(v)
    default:
        // 空转，CPU 会一直跑
    }
}
```

这段代码的问题不在于“快”，而在于“太急”。

channel 没数据的时候，它不等，直接走 default，然后马上回来再试。空闲时这不是非阻塞，这是自我消耗。

如果你是想“隔一会儿再检查一次”，就该用 `ticker`，不是 default：

```go
ticker := time.NewTicker(100 * time.Millisecond)
defer ticker.Stop()

for {
    select {
    case v := <-workCh:
        process(v)
    case <-ticker.C:
        doHealthCheck()
    }
}
```

### 场景二：某个 channel 变成 nil

nil channel 在 select 里不会 panic，它只是永远不会就绪。

```go
var ch2 <-chan int
for {
    select {
    case v := <-ch1:
        handle(v)
    case v := <-ch2:
        handle(v)
    }
}
```

如果 `ch2` 没有被正确初始化，这个分支就是死的。代码看起来正常，执行路径却永远走不到。

### 场景三：发送方压根不会再发

```go
go func() {
    if false {
        ch <- 42
    }
}()

v := <-ch // 永久等待
```

这种问题最麻烦，因为代码没错，逻辑却断了。你看到的是一个接收动作，真正缺的是上游的触发条件。

## 四、Range 不退出，通常不是“循环问题”，是“收尾缺失”

很多人盯着 `for v := range ch` 以为 range 是元凶。不是。

range 只是忠实执行协议：你不 close，我就一直等。

真正麻烦的是 fan-in 场景。

多个 source 汇到一个 merged channel，如果所有 worker 都停了，但 merged 没关，消费方就会一直等下去。

```go
func fixedFanIn(sources []<-chan int) <-chan int {
    merged := make(chan int)
    var wg sync.WaitGroup

    for _, src := range sources {
        wg.Add(1)
        go func(ch <-chan int) {
            defer wg.Done()
            for v := range ch {
                merged <- v
            }
        }(src)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}
```

这里的关键不是 WaitGroup，而是“谁来宣布结束”。

如果没人宣布结束，range 就只能一直等。

## 五、真正正确的模式：每个 goroutine 都要有退出路径

channel 不是队列，goroutine 也不是会自动消失的任务。

你要给它三个东西：

- 创建路径：它从哪来
- 响应路径：它在等谁
- 取消路径：它什么时候走

最稳的写法，还是 `context.Context`。

```go
func worker(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return
            case v, ok := <-in:
                if !ok {
                    return
                }
                out <- v * 2
            }
        }
    }()
    return out
}
```

这段代码里有三个收口：

- `ctx.Done()` 负责主动退出
- `ok == false` 负责识别输入关闭
- `defer close(out)` 负责把后面的消费者放走

只要这三件事齐了，泄漏概率会低很多。

## 六、buffer 不是创可贴，只是给你多一点时间

buffer 有用，但用途很窄。

它适合吸收短时间的峰值，不适合修复错误的生命周期设计。

如果问题是“上游比下游快一点”，buffer 能缓一缓。

如果问题是“下游已经没了，上游还不知道”，buffer 只会让你晚一点看到炸点。

所以判断很简单：

- 峰值问题 → 可以考虑 buffer
- 生命周期问题 → 优先改退出逻辑

别把“没立刻卡死”误判成“系统没问题”。

## 七、排查时，先问这三句话

下次你看到 goroutine 涨了，别急着重启，先问：

1. 谁在发？
2. 谁在收？
3. 谁负责宣布结束？

这三句问清楚，很多 channel 问题就已经少了一半。

再往下，就按这条路径走：

- 看 goroutine 是否只涨不降
- 看 pprof 里是 `chan send` 还是 `chan receive`
- 看 select 里有没有 default 忙等
- 看 range 对应的 close 有没有到
- 看每个 goroutine 有没有 ctx.Done() 或等价的退出路径

## 八、最后记住两句

第一句：buffer 只能买时间，不能替你收尾。

第二句：channel 出问题，通常不是某一行写错了，而是一段关系没人负责结束。

你真正要修的，不是“这次为什么卡住”，而是“下次它怎么自己走”。

goroutine 泄漏不是忘了 cancel 这么简单，但也没复杂到要靠运气。

把退出路径补齐，问题就会从“玄学”变成“流程”。
