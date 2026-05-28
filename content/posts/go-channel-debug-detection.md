---
title: "goroutine 涨了为什么别先加 buffer：Go channel 阻塞要先分清 chan send 和 chan receive"
description: "goroutine 涨了就加 buffer，是把问题从第 101 次发送推迟到第 1001 次。真正的根因是 nil channel、closed panic、sender/receiver 不匹配——这些加 buffer 都解决不了。"
date: 2026-05-28T15:09:00+08:00
draft: false
author: "任博"
tags: ["Go", "channel", "pprof", "并发编程", "goroutine泄漏"]
categories: ["技术实战"]
cover: "/images/go-channel-debug-detection/cover.png"
toc: true
---

goroutine 从 200 慢慢涨到 2 万，线上服务还没挂，但你知道它不对劲。

监控图上那条线不陡，甚至有点温和。它不像事故，更像一个迟早会来的事故。

这时候很多人的第一反应是：把 channel buffer 调大一点。

`make(chan int, 100)` 改成 `make(chan int, 1000)`。重启。观察。曲线好像平了。

三天后，又涨回来了。

![goroutine 涨了别先加 buffer](/images/go-channel-debug-detection/cover.svg)

这不是 buffer 还不够大，而是你修错了地方。

channel 阻塞最麻烦的地方在于，它经常不是立刻炸。它会把 goroutine 悄悄挂在那里，让你以为系统只是“有点忙”。等到 goroutine 越积越多、内存跟着涨、延迟开始抖，你才发现：问题早就发生了。

加 buffer 最多只能买时间，不能改变通信关系。

真正要问的是三件事：谁在发？谁在收？谁负责退出？

这篇只讲上半段：怎么发现 channel 卡住了，以及三个最容易被误判的问题：nil channel、closed channel panic、buffer 当创可贴。

## 先别看代码，先看 goroutine 有没有在涨

排查 channel 阻塞，不要一上来就钻进业务代码。

业务代码太会骗人了。每一行看起来都有理由，每个分支看起来都能跑通。尤其是线上问题，代码里往往没有一个醒目的“这里会死锁”。真正先露馅的，通常是数字。

最简单的基线是 `runtime.NumGoroutine()`。

```go
import "runtime"

func init() {
    go func() {
        ticker := time.NewTicker(30 * time.Second)
        defer ticker.Stop()
        for range ticker.C {
            log.Printf("goroutines: %d", runtime.NumGoroutine())
        }
    }()
}
```

这段代码很粗糙，但有用。

它不能告诉你原因，只能告诉你趋势。如果 goroutine 数量在业务高峰后能回落，通常只是正常波动；如果它只涨不降，尤其是低峰期也不回落，那就要怀疑有 goroutine 被挂住了。

线上服务更建议直接看 Prometheus 里的 `go_goroutines`。单个时间点没意义，曲线才有意义。一个服务从 300 涨到 800，不一定是问题；从 300 涨到 3000，再也不下来，就不是“业务变好了”。

别把 goroutine 数量当 KPI。

goroutine 涨了不是系统更努力了，通常是有人回不了家。

## pprof 里最该盯的两个词：chan send 和 chan receive

发现 goroutine 异常上涨之后，下一步看栈。

服务里引入 pprof：

```go
import _ "net/http/pprof"
```

然后抓 goroutine 栈：

```bash
curl -s http://localhost:6060/debug/pprof/goroutine?debug=1
```

如果输出里大量出现这种东西：

```text
goroutine 1234 [chan send]:
main.processTask(...)
    /app/worker.go:42 +0x120
```

意思很直接：这个 goroutine 卡在发送上。它想把值发进 channel，但另一端没人接，或者 buffer 已经满了。

另一种是：

```text
goroutine 5678 [chan receive]:
main.consumer(...)
    /app/consumer.go:18 +0x80
```

它卡在接收上。它在等值，但发送方已经不发了，或者压根没人负责关闭。

![channel 阻塞排查路径](/images/go-channel-debug-detection/inline-01.svg)

看到 `chan send` / `chan receive`，不要只定位到那一行代码就结束。

那一行只是尸体倒下的地方，不一定是凶手出现的地方。

你要顺着调用链往上问：这个 channel 是谁创建的？谁把它传进来的？谁应该消费？消费方什么时候会退出？发送方知不知道对面已经不收了？

channel 问题不是“某一行写错”，更多是“这条通信关系没人负责收尾”。

## 陷阱一：nil channel 不会 panic，它会让你一直等

很多人第一次踩 nil channel，会觉得 Go 很不够意思。

对一个 `nil` channel 发送或接收，不会 panic。

它会永久阻塞。

```go
func buggy() {
    var ch chan int  // nil，没有 make
    ch <- 42         // 卡住，goroutine 再也醒不过来
}
```

这段代码能编译。运行时也不会给你一个漂亮的错误。它只是停在那里，像什么都没发生。

这就是 nil channel 比 panic 更危险的地方：panic 至少会让你看见爆点；永久阻塞经常只是让 goroutine 数量慢慢变多。

nil channel 最常见的来源，不是有人故意写 `nil`，而是初始化路径漏了。

比如某个配置开关没开，某个分支没有 `make`，某个结构体字段只声明了类型但没有初始化。代码评审时看过去都很正常，线上某个条件一触发，它就变成一个永远不会就绪的 channel。

但 nil channel 不是完全没用。

在 `select` 里，它有一个很实用的语义：禁用某个 case。

```go
func withOptionalTimeout(recvCh <-chan int, enforce bool) (int, error) {
    var timeout <-chan time.Time
    if enforce {
        timeout = time.After(10 * time.Second)
    }

    select {
    case v := <-recvCh:
        return v, nil
    case <-timeout: // timeout 为 nil 时，这个 case 永远不会被选中
        return 0, errors.New("timed out")
    }
}
```

`timeout` 为 nil 时，这个分支就被动态关掉了。这是 Go 里很常见的写法。

所以问题不在于 nil channel 本身，而在于你有没有主动设计它。

如果你没有明确想用 nil channel 禁用分支，那 `var ch chan int` 后面就必须能看到 `make` 的路径。否则它不是“还没准备好”，而是“永远不会准备好”。

## 陷阱二：closed channel panic，本质是收尾权混乱

nil channel 是静悄悄地卡住，closed channel 更像突然炸雷。

最常见的三种 panic：

```go
// 1. 关闭 nil channel
close(ch) // ch 为 nil，panic

// 2. 重复关闭
close(ch)
close(ch) // panic: close of closed channel

// 3. 向已关闭 channel 发送
close(ch)
ch <- 1   // panic: send on closed channel
```

第三种最常见，也最难看。

一个 goroutine 觉得自己处理完了，顺手 `close(ch)`。另一个 goroutine 还在跑，刚准备把结果发回来。于是线上直接出现：

```text
panic: send on closed channel
```

很多团队会把这个问题修成“发送前判断一下 channel 关没关”。这条路基本走不通。

Go 没有提供一个可靠的、并发安全的“检查 channel 是否已关闭，然后再发送”的通用动作。因为你检查完的一瞬间，另一个 goroutine 仍然可能把它关掉。

真正要修的不是“发送前多看一眼”，而是“谁有资格 close”。

channel 的 close 不是垃圾清理动作，而是通信协议的一部分。

最稳的原则是：由唯一的 sender 关闭 channel。receiver 不要关。多个 sender 还在跑的时候，不要随便关。

唯一 sender 的情况很简单：

```go
go func() {
    defer close(ch)
    for i := 0; i < 10; i++ {
        ch <- i
    }
}()
```

多个 sender 呢？等所有 sender 都结束，再由一个协调者关闭。

```go
var wg sync.WaitGroup

for i := 0; i < 3; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        for j := 0; j < 5; j++ {
            ch <- id*10 + j
        }
    }(i)
}

go func() {
    wg.Wait()
    close(ch)
}()
```

如果你已经需要用 `sync.Once` 来防止重复 close，它可以当兜底，但不该是第一反应。

```go
var closeOnce sync.Once
safeClose := func() {
    closeOnce.Do(func() { close(ch) })
}
```

`sync.Once` 能避免重复关灯，但不能告诉你谁该关灯。

真正健康的 channel 代码，应该一眼能看出关闭权在哪里。看不出来，就说明生命周期设计还没完成。

## 陷阱三：buffer 只能吸收尖峰，不能修生命周期

现在回到最常见的补丁：加 buffer。

```go
ch := make(chan int, 100)

go func() {
    for i := 0; i < 10000; i++ {
        ch <- i // 发到第 101 个还是会阻塞
    }
}()

// 只消费了 50 个就停了
for i := 0; i < 50; i++ {
    <-ch
}
```

buffer 是 100，看起来很安全。但消费方只读了 50 个就走了。发送方继续发，第 101 次发送照样卡住。

你把 buffer 从 100 调到 1000，只是把问题从第 101 次推迟到第 1001 次。

它不是修复，是延期。

buffer 适合什么？适合吸收短时间的速率差。

比如生产者偶尔突发一小波，消费者稳定处理；比如日志、指标、任务队列有可控的峰值；比如你明确知道上游最多会瞬间多发多少，下游能在多久内消化掉。

这时 buffer 是削峰工具。

但如果问题是 receiver 已经退出了，sender 还不知道；或者 sender 退出了，receiver 还在 `range`；或者请求取消了，后台 goroutine 还在往结果 channel 发。那 buffer 再大也只是拖延。

该修的是退出路径。

```go
func properLifecycle(ctx context.Context) <-chan int {
    ch := make(chan int)

    go func() {
        defer close(ch)
        for i := 0; ; i++ {
            select {
            case ch <- i:
            case <-ctx.Done():
                return
            }
        }
    }()

    return ch
}
```

这里的重点不是“无缓冲一定更好”。重点是发送方知道什么时候该退出，并且退出时负责关闭 channel。

很多线上泄漏，缺的不是 buffer，缺的是取消信号。

## 一条排查路径：先分清卡在发送还是接收

下次遇到 goroutine 涨，不要先改 buffer。

按这个顺序走：

第一，看 `go_goroutines` 或 `runtime.NumGoroutine()`，确认是不是只涨不降。

第二，用 pprof 抓 goroutine 栈，看大量堆积的是 `chan send` 还是 `chan receive`。

第三，如果是 `chan send`，优先问：receiver 是否退出了？buffer 是否满了？发送方有没有监听 `ctx.Done()`？

第四，如果是 `chan receive`，优先问：sender 是否退出了？channel 是否应该 close？有没有 nil channel 或永远不会触发的分支？

第五，再去看 buffer。它是不是为了吸收可预期尖峰？如果不是，它多半只是在掩盖生命周期问题。

这套路径不复杂，但能挡掉很多“凭感觉改参数”的冲动。

Go 的 channel 很容易让人产生错觉：只要会写 `<-` 和 `ch <-`，就算会用了。

不是。

channel 真正要设计的是一段关系：谁交付，谁接收，谁宣布结束，谁在取消时退出。

buffer 不是创可贴，close 不是扫尾动作，nil channel 也不是一个普通空值。

把这三件事分清，你已经能排掉一半 channel 阻塞问题。

下篇继续拆更容易在线上拖死服务的几类问题：sender 没人接、receiver 没人发、`select default` 忙等，以及 `for range` 等不到 close。它们看起来都像“小疏忽”，但最后都会变成 goroutine 泄漏。

如果你正在排查 Go 服务里的 goroutine 暴涨，可以先收藏这篇。下一次看到 `chan send` 或 `chan receive`，别急着加 buffer，先把这条通信关系画出来。
