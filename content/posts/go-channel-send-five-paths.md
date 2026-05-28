---
title: "Go 为什么宁可让 chansend 卡住，也不让你误以为 ch <- v 发成功了"
description: "ch <- v 在 Go runtime 里不是发往一个队列就完事了。chansend 有五条路径：先看有没有 receiver，再看 buffer 有没有空位，最后才决定是挂起还是直接失败。每一条路径背后都藏着一个取舍。"
date: 2026-05-26T17:40:00+08:00
draft: false
author: "任博"
tags: ["Go", "channel", "runtime", "源码分析", "并发编程"]
categories: ["技术实战"]
cover: "/images/go-channel-send-five-paths/cover.png"
toc: true
---

goroutine dump 里刷出一排 `[chan send]`，你知道它卡住了。

但更麻烦的问题是：它到底卡在哪？

是 channel 本身是 nil？是已经有人在等接收，只差一次交付？是 buffer 满了？还是它已经被 runtime 挂进 `sendq`，等下一次接收或关闭来唤醒？

如果你脑子里只有一句“channel 是队列”，排查到这里基本就停了。你会本能地看 buffer 多大，然后想：要不要从 100 改成 1000？

这往往不是修 bug，只是把 bug 往后推。

![chansend 五条路径封面](/images/go-channel-send-five-paths/cover.png)

`ch <- v` 在 Go runtime 里不是一句“把数据塞进队列”。它会先经过 `hchan`，再根据当前状态走不同路径：nil channel、等待中的 receiver、可写 buffer、非阻塞失败、真正挂起。

同样是 `[chan send]`，背后的命运可能完全不同。

很多线上排查就是死在这里。

日志里只剩一行栈：

```text
goroutine 18423 [chan send]:
main.(*Worker).emit(...)
    /app/worker.go:87
```

你顺着代码点进去，只看到一行很普通的发送：

```go
w.results <- result
```

这行代码不会告诉你 receiver 是不是已经退出了，不会告诉你 buffer 里是不是堆满了旧结果，也不会告诉你这个 channel 是不是在某个分支里被置成了 nil。

业务代码给你的信息太少，runtime 路径才是地图。

排查 channel 阻塞，先别问 buffer 多大。先问：它走到了哪条路。

## hchan：你以为是管道，runtime 看见的是一张状态表

Go 代码里你写：

```go
ch := make(chan int, 4)
```

runtime 里拿到的是一个 `hchan`。

Go 1.25.4 的 `runtime/chan.go` 里，核心字段大致是这些：

```go
type hchan struct {
    qcount   uint
    dataqsiz uint
    buf      unsafe.Pointer
    elemsize uint16
    closed   uint32
    elemtype *_type
    sendx    uint
    recvx    uint
    recvq    waitq
    sendq    waitq
    lock     mutex
}
```

这段结构不用背。真正要盯住的是六类东西：

- `buf`：有缓冲 channel 的底层缓冲区。
- `qcount`：当前 buffer 里有多少元素。
- `dataqsiz`：buffer 容量。
- `sendx` / `recvx`：环形队列的写入和读取位置。
- `sendq` / `recvq`：阻塞的 sender / receiver 队列。
- `lock`：保护这张状态表的锁。

把这六类字段连起来，channel 的轮廓就出来了。

`buf/qcount/dataqsiz/sendx/recvx` 负责“东西放哪里”。`sendq/recvq` 负责“谁在等谁”。`lock` 负责把这些变化串成一个一致状态。

所以 runtime 看 channel，不是只看“有没有格子能放数据”。它还要看有没有 goroutine 已经排队，当前操作愿不愿意等，channel 有没有关闭，以及这次状态改变之后该唤醒谁。

这里先压住一个误解：channel 不是无锁结构。

源码注释说得很直：`lock protects all fields in hchan`。`chansend`、`chanrecv`、`closechan` 都会围绕这把锁组织状态变化。

所以 channel 的厉害之处，不是“没有锁”。

它真正帮你做的是：在 runtime 管理的一把锁下面，把数据拷贝、等待队列、goroutine 挂起和唤醒，编排成一套稳定协议。

channel 不是无锁队列，是带调度语义的等待协议。

## makechan：创建 channel 时，内存已经分了三种命

`make(chan T, n)` 最后会走到 `makechan`。

这里有一个容易被忽略的细节：runtime 不总是用同一种方式分配 channel。

简化后是三条路：

```go
switch {
case mem == 0:
    c = (*hchan)(mallocgc(hchanSize, nil, true))
    c.buf = c.raceaddr()
case !elem.Pointers():
    c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
    c.buf = add(unsafe.Pointer(c), hchanSize)
default:
    c = new(hchan)
    c.buf = mallocgc(mem, elem, true)
}
```

第一种，`mem == 0`。常见于无缓冲 channel，或者元素大小为 0 的 channel。它不需要真正的缓冲数组。

第二种，元素里没有指针。runtime 可以把 `hchan` 和 `buf` 放在同一块连续内存里，GC 不需要扫描 buffer 里的元素。

第三种，元素里有指针。`hchan` 和 `buf` 分开分配，buffer 要接受 GC 追踪。

这不是抠源码细节。它提醒你：channel 从出生开始，就不是一个抽象“管道”。它在内存里怎么放，取决于容量、元素大小、元素里有没有指针，以及 GC 成本。

如果你只把 channel 想成一个队列，这一层永远看不见。

## chansend：五条路，不是一个动作

发送入口是 `chansend(c, ep, block, callerpc)`。

你在业务代码里看到的是一行：

```go
ch <- v
```

runtime 看到的是五个问题。

![chansend 五条路径拆解](/images/go-channel-send-five-paths/inline-01.png)

### 第一条：nil channel，直接进死胡同

源码一开始就判断：

```go
if c == nil {
    if !block {
        return false
    }
    gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
    throw("unreachable")
}
```

向 nil channel 发送，如果是阻塞操作，就会 `gopark`，等待原因是 `waitReasonChanSendNilChan`。

它不是“现在没人收，所以等等”。

nil channel 背后没有 `hchan`，没有 `sendq`，没有 `recvq`，也就没有任何正常路径能把它唤醒。

这也是为什么 nil channel 常被用来动态禁用 `select` 分支。把一个 channel 设成 nil，不是让它慢一点，而是让它彻底退出这轮可选集合。

### 第二条：recvq 里有人等，直接交付

通过 nil 检查后，runtime 会加锁，然后先看 `recvq`：

```go
lock(&c.lock)

if c.closed != 0 {
    unlock(&c.lock)
    panic(plainError("send on closed channel"))
}

if sg := c.recvq.dequeue(); sg != nil {
    send(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true
}
```

这个顺序很关键。

如果已经有 receiver 阻塞在 `recvq` 里，发送方不会先把数据塞进 buffer，再通知 receiver 来拿。源码注释写得很清楚：直接把值交给等待中的 receiver，绕过 channel buffer。

哪怕这是一个有缓冲 channel，只要此刻有人已经在等，runtime 也会优先做直接交付。

这个选择会让很多人意外。因为我们习惯把“有缓冲 channel”想成“永远先进 buffer”。但在 runtime 的视角里，等待中的 receiver 已经说明一件事：对方现在就能接。

既然对方已经站在门口，继续绕一圈 buffer 只会制造多余动作。

这像窗口前已经站了一个人，你手里的东西不必先放进仓库，再让他从仓库里取。

### 第三条：没人等，但 buffer 有空位

如果没有等待中的 receiver，runtime 才看 buffer：

```go
if c.qcount < c.dataqsiz {
    qp := chanbuf(c, c.sendx)
    typedmemmove(c.elemtype, qp, ep)
    c.sendx++
    if c.sendx == c.dataqsiz {
        c.sendx = 0
    }
    c.qcount++
    unlock(&c.lock)
    return true
}
```

这才是很多人脑子里的“队列”画面。

`sendx` 指向下一次写入位置。写完后递增，到头就回到 0。`qcount` 加一。

所以，有缓冲 channel 的 buffer 确实是环形队列。

但这只是五条路径里的第三条。你不能拿这一条解释所有 channel 行为。

把 channel 只理解成 buffer，是很多排查误判的起点。

### 第四条：非阻塞发送，发不了就返回

`select` 加 `default` 时，会用到非阻塞语义。

如果没有 receiver，buffer 也满了，且 `block == false`：

```go
if !block {
    unlock(&c.lock)
    return false
}
```

runtime 不会把当前 goroutine 挂起来。

它只是告诉上层：这次发送没有成功，你可以去走别的分支。

所以 `default` 不是“兜底打印一行日志”那么简单。它改变了这次通信的态度：我不等。

### 第五条：普通发送，真阻塞，挂进 sendq

最后才是你在 dump 里最常见的画面。

普通 `ch <- v`，没有 receiver，buffer 也没空间，那就只能阻塞。runtime 会拿一个 `sudog`，把当前 goroutine、要发送的数据地址、channel 绑在一起，然后挂进 `sendq`：

```go
gp := getg()
mysg := acquireSudog()
mysg.elem = ep
mysg.g = gp
mysg.c = c
c.sendq.enqueue(mysg)
gopark(
    chanparkcommit,
    unsafe.Pointer(&c.lock),
    waitReasonChanSend,
    traceBlockChanSend,
    2,
)
```

走到这里，`ch <- v` 已经不是“慢一点”。

当前 goroutine 被 runtime 停下来了。它不是睡一会儿自动重试，而是在等局面改变：某个 receiver 来接，或者 channel 被 close。

这才是 `[chan send]` 真正值得警惕的地方。

## 看到 chan send，按这张清单排

下次 dump 里看到 `[chan send]`，不要只问“是不是 buffer 太小”。先按五条路倒推。

第一，确认它是不是 nil channel。尤其是 `select` 里被动态置 nil 的 channel，或者初始化路径里没有赋值的 channel。

你可以先搜这个 channel 的赋值路径：它有没有在某个分支里被设成 nil？有没有因为配置、开关、初始化顺序导致一直没有创建？nil channel 的可怕之处在于，它不是慢，它是没有运行时对象。

第二，看有没有 receiver 活着。发送卡住，常常不是发送方的问题，而是接收方提前退出、被 context 取消、或者卡在别的地方。

这时不要只盯着发送那一行，要顺着接收方往回查：谁负责启动接收 goroutine？它的退出条件是什么？它退出时有没有通知发送方？

第三，看 buffer 是不是被当成修复手段。buffer 能削峰，不能替你设计生命周期。接收方不工作，100 改 10000 也只是晚点爆。

真正该问的是：buffer 满的时候，业务希望发生什么？等待、丢弃、降级、返回错误，还是跟随 context 退出？如果这个答案没写进代码，runtime 只能帮你把 goroutine 挂起来。

第四，看这段发送到底应不应该阻塞。如果不该阻塞，就应该用 `select` 加 `default` 或 `ctx.Done()` 明确表达“不等”。

例如这种代码，至少把退出路径说清楚了：

```go
select {
case ch <- v:
    return nil
case <-ctx.Done():
    return ctx.Err()
}
```

如果业务允许丢弃，也要明说：

```go
select {
case ch <- v:
default:
    metrics.Drop("result_channel_full")
}
```

第五，如果确实应该阻塞，就要继续查谁负责让它醒来：receiver 在哪里？close 由谁发起？退出协议有没有写清楚？

channel 排查最怕一句话：先加个 buffer 试试。

因为 buffer 只改变第三条路的容量，不改变 nil、不改变接收方退出、不改变 sendq 里的等待关系，也不改变 close 的责任。

更实用的修复方式，是把五条路翻译成五类动作。

nil 路径，要修初始化和开关逻辑。不要让一个本该工作的 channel 长期保持 nil，除非你就是要在 `select` 里禁用它。

直接交付路径，要确认接收方是不是正常排队。如果 receiver 存在但处理慢，问题可能在接收后的业务逻辑，而不是 channel 本身。

buffer 路径，要确认容量代表什么。容量不是越大越好，它应该对应一个明确的业务窗口：允许积压多少任务、多少结果、多少信号。

非阻塞路径，要确认失败之后怎么处理。丢弃、重试、记录指标、返回错误，必须选一个。没有选择，就是把压力藏起来。

sendq 路径，要确认谁负责唤醒。普通发送一旦挂进去，就需要接收方或 close 改变局面；如果这两个角色都没人负责，goroutine 泄漏只是时间问题。

所以，源码不是为了让你写出更玄的解释。源码是为了逼你把问题问具体。

## 最后：channel 的难点不是传值，是等待关系

`ch <- v` 看起来像一句赋值。

但在 runtime 里，它可能是永久阻塞，可能是直接交付，可能是写入环形缓冲区，可能是非阻塞失败，也可能是把当前 goroutine 挂进等待队列。

这五条路看明白之后，再看 goroutine dump，味道就不一样了。

你不再只看到“卡住了”。

你会开始追问：它为什么没走直接交付？为什么 buffer 没空位？为什么 receiver 不在？为什么这个发送没有退出路径？

这才是源码真正有用的地方：不是让你背 `runtime/chan.go`，而是让你排查线上问题时，脑子里有一张运行时地图。

下一篇继续往下走：`chanrecv`、无缓冲 channel 的栈间拷贝、`closechan` 如何批量唤醒等待中的 goroutine，以及为什么 `close(ch)` 很多时候不是“关门”，而是把所有人同时叫醒。

如果你想把 Go 并发从“会写 API”推进到“能解释线上卡在哪里”，可以继续关注这个系列。channel 的表面很简单，真正的代价都藏在 runtime 的等待关系里。
