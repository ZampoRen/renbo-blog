---
title: "Go 为什么让 close(ch) 批量唤醒等待者，而不是只给 channel 关个门"
description: "close(ch) 不只是关门。源码里它会遍历 recvq 和 sendq，按队列逐个唤醒所有阻塞的 goroutine。区别是：receiver 得到零值，sender 等到的是 panic。"
date: 2026-05-27T10:21:00+08:00
draft: false
author: "任博"
tags: ["Go", "channel", "runtime", "源码分析", "并发编程"]
categories: ["技术实战"]
cover: "/images/go-channel-recv-close/cover.png"
toc: true
---

线上最烦的 channel 问题，往往不是“发不出去”。

而是你以为自己在等一个值，结果等来的是零值；你以为 `close(ch)` 只是关门，结果它把一批阻塞的 goroutine 全部叫醒；你以为 nil channel 和 closed channel 都是“不能用了”，结果一个永远没人理你，一个立刻给你结果。

上一篇讲 `ch <- v`，核心是五条发送路径。这一篇换到接收和关闭。

`<-ch` 看起来只是从队列里拿一个值。但在 runtime 里，它要判断：channel 是不是 nil？是不是已经 close？buffer 里还有没有旧值？有没有 sender 正卡在 `sendq`？这次接收该不该阻塞？

真正容易讲错的地方在这里：接收不是发送的镜像。

尤其是有缓冲 channel 满的时候，receiver 碰到等待中的 sender，不是简单“从 sender 手里拿值”。它先拿 buffer 头部的旧值，再把 sender 的新值补进 buffer 尾部。

这一个细节没想明白，后面 close、零值、`ok`、goroutine 唤醒，全都会变成一团雾。

![channel recv close runtime map](/images/go-channel-recv-close/cover.png)

## chanrecv：接收不是 chansend 的镜像

接收入口是 `chanrecv(c, ep, block)`。

你在业务代码里看到的是：

```go
v := <-ch
v, ok := <-ch
```

runtime 看到的不是“取一个值”，而是一组分叉：

1. `ch == nil`：阻塞接收会永久挂起。
2. `ch` 已关闭且 buffer 为空：写入零值，返回 `received == false`。
3. 有等待中的 sender：配对接收。
4. buffer 有数据：从 `buf[recvx]` 取。
5. 没有值且不能阻塞：返回失败。
6. 没有值且需要阻塞：把当前 goroutine 挂进 `recvq`。

先看 nil channel。

```go
if c == nil {
    if !block {
        return
    }
    gopark(nil, nil, waitReasonChanReceiveNilChan, traceBlockForever, 2)
    throw("unreachable")
}
```

nil channel 背后没有 `hchan`，没有 buffer，没有 `sendq`，也没有 `recvq`。

所以从 nil channel 接收，不是在等“以后可能有值”。它是在等一个不存在的运行时对象。

这也是为什么 nil channel 可以用来动态禁用 `select` 分支。不是它暂时没有值，而是这个 case 没有可操作的 channel。

接下来是 closed channel。

```go
if c.closed != 0 {
    if c.qcount == 0 {
        unlock(&c.lock)
        if ep != nil {
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }
    // The channel has been closed, but the channel's buffer have data.
}
```

这段代码压住一个常见误判：channel 关了，不代表 buffer 里的值立刻消失。

如果 channel 已经 close，但 buffer 里还有数据，receiver 仍然先把旧数据读完。只有 buffer 空了以后，再接收才会得到零值，并且 `ok == false`。

所以这段代码：

```go
ch := make(chan int, 1)
ch <- 1
close(ch)

v, ok := <-ch // 1, true
v, ok = <-ch  // 0, false
```

不是语法小技巧，而是 runtime 明确写出来的顺序。

closed channel 的重点不是“不能读”。

closed channel 的重点是：还能读完旧值，读完以后用零值和 `ok=false` 告诉你结束了。

如果你的元素类型本身可能是零值，比如 `0`、`""`、`nil`，却不看 `ok`，你就会把“结束信号”误判成“正常数据”。

这类 bug 不吵，但很脏。

它不会 panic，它会让你的业务悄悄走错分支。

## 有缓冲 channel 满时，recv 会做一次“换货”

接收路径里最容易讲错的是这一段：

```go
if sg := c.sendq.dequeue(); sg != nil {
    recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true, true
}
```

看到 `sendq` 里有 sender，很多人会下意识以为：receiver 直接从 sender 那里拿值。

这只对无缓冲 channel 成立。

真正的逻辑在 `recv` 里。源码注释写得很直：

```go
// For synchronous channels, both values are the same.
// For asynchronous channels, the receiver gets its data from
// the channel buffer and the sender's data is put in the
// channel buffer.
```

无缓冲 channel，也就是 `dataqsiz == 0`，receiver 会从 sender 的栈上直接拷贝：

```go
if c.dataqsiz == 0 {
    if ep != nil {
        recvDirect(c.elemtype, sg, ep)
    }
}
```

有缓冲 channel 呢？情况完全不同。

如果 `sendq` 里已经有 sender，说明这个 sender 之前为什么会阻塞？因为 buffer 满了。

这时候 receiver 不能直接拿 sender 的新值。否则就破坏了 FIFO：buffer 里排在前面的旧值还没被读，新来的值却插队交付了。

所以 runtime 做的是这件事：

```go
qp := chanbuf(c, c.recvx)

// copy data from queue to receiver
typedmemmove(c.elemtype, ep, qp)

// copy data from sender to queue
typedmemmove(c.elemtype, qp, sg.elem)

c.recvx++
if c.recvx == c.dataqsiz {
    c.recvx = 0
}
c.sendx = c.recvx
```

翻成人话：receiver 拿走 buffer 头部的旧值；sender 的新值补到队尾。

源码里说“head”和“tail”会落在同一个槽位，因为队列满了。你可以把它想成一个满格的环形队列：receiver 刚把一个格子腾出来，sender 马上把自己的值填进去。

这不是多余动作。

这是 channel 维持 FIFO 的代价。

有缓冲 channel 最容易骗人。你看到的是“有 sender 在等”，但 receiver 真正拿到的，可能不是那个 sender 手里的新值，而是 buffer 里更早排队的旧值。

## 无缓冲 channel：数据不是进队列，是跨栈交接

无缓冲 channel 经常被一句话带过：没有 buffer，所以发送和接收必须同时出现。

这句话没错，但它太轻。

源码里的关键是 `sendDirect` 和 `recvDirect`。

发送给等待中的 receiver 时：

```go
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
    // src is on our stack, dst is a slot on another stack.
    dst := sg.elem
    typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.Size_)
    memmove(dst, src, t.Size_)
}
```

接收等待中的 sender 时：

```go
func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
    // dst is on our stack or the heap, src is on another stack.
    src := sg.elem
    typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.Size_)
    memmove(dst, src, t.Size_)
}
```

注释已经把画面说穿了：一个 goroutine 栈上的值，被 runtime 拷贝到另一个 goroutine 的栈上。

当然，不要把它想成业务代码真的“伸手去摸别人的栈”。这是 runtime 在持有 channel 锁、处理写屏障和栈移动安全之后完成的拷贝。

但这个画面很重要。

无缓冲 channel 不是“容量为 0 的队列”。

它更像一个 rendezvous 点：双方都到场，数据才完成交接；交接完成，等待的一方才被唤醒。

所以无缓冲 channel 天然带同步语义。不是因为它神秘，而是因为数据移动和 goroutine 唤醒被绑在了同一个运行时协议里。

如果你只是想“存一下以后再取”，无缓冲 channel 不适合。

如果你想表达“对方接住之前，我不继续往下走”，它才对味。

## closechan：不是关门，是批量开门

现在看 `close(ch)`。

很多人把 close 理解成“把门关上”。这个比喻只能解释一半。

在 runtime 里，`closechan` 先设置 `closed = 1`，然后做一件更重要的事：把 `recvq` 和 `sendq` 里阻塞的 goroutine 全部取出来，放进一个临时列表，最后统一 `goready`。

核心代码是：

```go
c.closed = 1

var glist gList

// release all readers
for {
    sg := c.recvq.dequeue()
    if sg == nil {
        break
    }
    if sg.elem != nil {
        typedmemclr(c.elemtype, sg.elem)
        sg.elem = nil
    }
    gp := sg.g
    gp.param = unsafe.Pointer(sg)
    sg.success = false
    glist.push(gp)
}

// release all writers (they will panic)
for {
    sg := c.sendq.dequeue()
    if sg == nil {
        break
    }
    sg.elem = nil
    gp := sg.g
    gp.param = unsafe.Pointer(sg)
    sg.success = false
    glist.push(gp)
}

unlock(&c.lock)

for !glist.empty() {
    gp := glist.pop()
    gp.schedlink = 0
    goready(gp, 3)
}
```

这里最关键的是两个 “release all”。

close 不是只改一个标志位。它会释放所有正在等接收的 goroutine，也会释放所有正在等发送的 goroutine。

这里有个实现细节也值得记住：runtime 不是一边拿着 `c.lock` 一边直接唤醒。它先把等待者收进 `glist`，释放锁之后再 `goready`。

这不是代码洁癖。`hchan` 附近的注释明确提醒，不要在持有 channel 锁时改变另一个 G 的状态，否则可能和栈收缩打架。close 看起来是一行代码，实际是在小心处理锁、等待队列和调度器之间的边界。

但醒来以后，两类人的命运完全不同。

receiver 醒来后，`success=false`，接收结果就是零值和 `ok=false`。

sender 醒来后，也看到 `success=false`，但它继续走的是 panic 路径：`send on closed channel`。

这就是 close 最容易误伤人的地方。

同样是被 close 唤醒，receiver 得到的是结束通知，sender 得到的是一次崩溃。

close(ch) 不是在关门，是打开所有阻塞 goroutine 的门。只是门后面，有人走向正常结束，有人走向 panic。

这也是为什么 close 权通常应该在发送方。因为 close 表达的不是“我不想收了”，而是“我不会再发了”。

接收方不想收，应该走取消：`context`、`done`、超时、返回错误。它随手 close 数据 channel，本质上是在替所有 sender 宣布“你们都别发了”。

如果还有 sender 没停，panic 就只是时间问题。

真实项目里最常见的翻车，是多 worker 往同一个 `results` channel 发结果，接收方拿够了结果，或者某个错误分支提前返回，于是顺手 `close(results)`。

```go
results := make(chan Result)

for i := 0; i < workerCount; i++ {
    go func() {
        for job := range jobs {
            results <- process(job)
        }
    }()
}

// 接收方不想继续收了，直接 close：危险
close(results)
```

这段代码的问题不在 `results <- process(job)` 那一行。

那一行只是最后炸出来的地方。真正的问题是：接收方没有资格替所有 worker 宣布“不会再发送”。它最多只能宣布“我不想要了”。

这两个语义必须拆开。

更稳的写法，是让发送端或协调者负责 close，让取消走另一条路径：

```go
select {
case results <- r:
case <-ctx.Done():
    return
}
```

如果是多 sender，就让 `WaitGroup` 等所有发送者结束，再由协调 goroutine 关闭结果 channel。

这样 close 的含义才干净：不是赶人走，而是所有生产者真的完工了。

close 一旦变成“我不想收了”的快捷按钮，channel 的所有权就乱了。

所有权一乱，panic 只是 runtime 帮你把问题喊出来。

## nil 和 closed：一个没有对象，一个状态已结束

nil channel 和 closed channel 经常被放在一起讲，像是两种“不能用”的状态。

这个说法会害人。

nil channel 没有 `hchan`。

```go
var ch chan int
ch <- 1    // 永久阻塞
<-ch       // 永久阻塞
close(ch) // panic: close of nil channel
```

它的问题是：没有运行时对象，所以没有队列、没有锁、没有等待关系，也没有未来某个 close 能把你叫醒。

closed channel 有 `hchan`。

只是这个 `hchan` 的 `closed` 标志已经置位。

```go
ch := make(chan int, 1)
ch <- 1
close(ch)

v, ok := <-ch // 1, true
v, ok = <-ch  // 0, false
ch <- 2       // panic: send on closed channel
close(ch)     // panic: close of closed channel
```

它的问题不是“没人理你”。

恰恰相反，它会立刻理你：接收会继续按 closed 语义返回；发送和重复 close 会直接 panic。

所以 nil 和 closed 不是一类边界。

nil 是没有对象，closed 是对象进入终态。

这句话在排查里很有用。

看到 goroutine 卡在 `[chan receive (nil chan)]`，你要查初始化、赋值路径、select 禁用逻辑。

看到接收到了零值和 `ok=false`，你要查谁关闭了 channel、buffer 是否已经被 drain、关闭时还有没有 receiver 在等。

看到 `send on closed channel`，你要查关闭权是不是错放到了接收方，或者多 sender 场景有没有协调者。

不要把这些问题混成一句“channel 不可用了”。混在一起，就没法排。

如果你想让团队里的人马上感受到区别，可以让他跑一段最小代码：

```go
package main

import "fmt"

func main() {
    var nilCh chan int

    closedCh := make(chan int, 1)
    closedCh <- 7
    close(closedCh)

    v, ok := <-closedCh
    fmt.Println(v, ok) // 7 true

    v, ok = <-closedCh
    fmt.Println(v, ok) // 0 false

    select {
    case <-nilCh:
        fmt.Println("nil ready")
    default:
        fmt.Println("nil not ready")
    }
}
```

这段代码不演示 panic，只演示两个边界的差异。

closed channel 会立刻给你结果，而且会先吐出 buffer 里的旧值。nil channel 在 `select` 里不会 ready，只会被 `default` 绕过去。

排查时这个区别很值钱：closed 是一个已经发生的事件，nil 是一个没有对象的状态。

前者要查谁发出了结束通知，后者要查为什么这个通信入口根本没建起来。

## 排查接收和关闭，先问这五个问题

如果你在线上遇到接收异常、零值误判、send on closed channel，别先改 buffer。

先问五个问题。

第一，接收方有没有检查 `ok`？

只写 `v := <-ch`，在 closed channel 上也能拿到零值。如果零值本身是合法数据，不检查 `ok` 就等于闭着眼睛分辨“正常值”和“结束”。

第二，buffer 里的旧值是否会在 close 后继续被读完？

`close(ch)` 不会清空 buffer。它只是宣布不会再有新值。接收方仍然会先拿旧值，再看到 `ok=false`。

第三，有等待 sender 时，这是无缓冲还是有缓冲？

无缓冲 channel 是跨栈交接。有缓冲且已满时，receiver 拿 buffer 头部旧值，sender 的新值补进 buffer。别把这两条路径讲成一条。

第四，谁有资格 close？

close 不是取消按钮。它是发送端的完工声明。单 sender 可以自己 close；多 sender 要有协调者；receiver 不该因为“不想收了”就关数据 channel。

这里最好落到代码上。

如果你有一组 worker 往 `results` 里写结果，不要让某个 reader 看到够了就直接 `close(results)`。reader 不知道后面还有没有 worker 正在 send。它 close 的那一刻，可能正好把某个 sender 从 `sendq` 里叫醒，然后让它 panic。

更稳的结构是：worker 只负责 send；协调者等所有 worker 退出后，再 close。

```go
var wg sync.WaitGroup
results := make(chan Result, 64)

for _, job := range jobs {
    wg.Add(1)
    go func(job Job) {
        defer wg.Done()
        results <- run(job)
    }(job)
}

go func() {
    wg.Wait()
    close(results)
}()

for r := range results {
    handle(r)
}
```

这段代码的重点不是 `WaitGroup` 本身，而是职责分开：发送者结束由 `wg` 证明，关闭动作由协调者执行，接收方只消费和识别结束。

如果接收方想提前退出，也不要用 close 数据 channel 来“通知”发送方。那是反方向操作。更稳的是另设 `context` 或 `done` 信号，让发送方在 send 前后都能看见退出条件：

```go
select {
case results <- r:
    return nil
case <-ctx.Done():
    return ctx.Err()
}
```

这才是关闭权的核心：数据 channel 的 close 用来通知 receiver “不会再有新值”，不是用来通知 sender “你别发了”。

很多 channel 事故，就是把这两个方向混在一起了。谁都觉得自己是在“收尾”，最后变成有人还在发送，有人已经 close。

第五，nil 是故意禁用，还是初始化漏了？

nil channel 在 `select` 里可以是设计；在普通发送/接收里通常是事故。区别不在语法，在你有没有把这个状态写成明确协议。

把这五个问题问完，channel 问题会从“玄学并发”变成可定位的运行时状态。

## 最后：channel 的难点，是醒来以后发生什么

这一篇讲接收和关闭，其实只想压住一个判断：channel 的难点不只是“谁在等谁”，还有“醒来以后发生什么”。

receiver 从 close 中醒来，拿到的是零值和 `ok=false`。

sender 从 close 中醒来，等到的是 panic。

无缓冲接收醒来，意味着一次跨栈交接已经完成。

有缓冲满队列里的 sender 醒来，意味着自己的值被补进了 buffer，而不是直接交给刚才那个 receiver。

nil channel 最狠：它甚至没有“醒来以后”。

这就是为什么只用“管道”“队列”“关门”这些比喻解释 channel，很快会不够用。

channel 在 runtime 里的真实形状，是一个 `hchan`，一个可选 buffer，两条等待队列，一把锁，以及一套关于挂起、交接、关闭和唤醒的协议。

下一篇会继续往下走：`selectgo` 到底怎么组织 case、怎么跳过 nil channel、怎么生成 poll order 和 lock order，以及为什么 `select` 不是一次 O(1) 的随机抽奖。

如果你想把 Go 并发从“会写语法”推进到“能解释线上卡在哪里”，可以继续关注这个系列。channel 表面简单，真正的代价都藏在等待和唤醒的边界里。
