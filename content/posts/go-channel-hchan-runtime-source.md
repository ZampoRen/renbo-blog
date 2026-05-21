---
title: "Go channel 源码不是在讲队列，而是在讲 goroutine 怎么排队"
description: "从 Go 1.25.4 runtime/chan.go 出发，拆解 hchan 结构、chansend/chanrecv 路径、无缓冲 channel 栈间拷贝、close 批量唤醒，以及 selectgo 如何注册和唤醒 case。"
date: 2026-05-21T15:17:00+08:00
draft: false
author: "任博"
tags: ["Go", "channel", "runtime", "源码分析", "并发编程"]
categories: ["技术实战"]
cover: "/images/go-channel-hchan-runtime-source/cover.png"
toc: true
---

goroutine dump 里满屏 `chan send`，你知道它卡住了。

但它到底卡在哪里？

是缓冲区满了？没有 receiver？被 `select` 挂进了等待队列？还是 channel 被 close 之后才醒过来，然后 panic？

如果只停留在“channel 是 goroutine 之间通信的管道”，这些问题永远说不清。

`ch <- v` 这一行代码，在 Go runtime 里不是一句“把数据塞进队列”。它会走过 `hchan`、环形缓冲区、`sendq` / `recvq`、`sudog`、`gopark`、`goready`，最后决定一个 goroutine 是继续跑，还是挂起来等别人。

![Go channel hchan 运行时封面](/images/go-channel-hchan-runtime-source/cover.png)

Go 的 channel 设计长期受 CSP 思想影响。Rob Pike 在 Go 早期设计和并发模型传播里反复强调：不要通过共享内存来通信，而要通过通信来共享内存。

![Rob Pike](/images/go-channel-hchan-runtime-source/rob-pike.jpg)

这句话听起来像哲学，但到了 runtime 里，它会变成很具体的结构：谁在等谁，数据放在哪里，哪个 goroutine 应该被唤醒。

这篇只做一件事：沿着 Go 1.25.4 的 `runtime/chan.go` 和 `runtime/select.go`，把 channel 的几条关键路径走一遍。

不是为了背源码。

是为了让你下次看到 `chan send`、`chan receive`、`close`、`select` 的时候，脑子里有一张清楚的运行时地图。

channel 不是无锁队列，它是带调度语义的协作原语。

## hchan：channel 在运行时到底长什么样

Go 代码里你写的是：

```go
ch := make(chan int, 4)
```

runtime 里对应的是一个 `hchan`。

Go 1.25.4 的结构大致是这样：

```go
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular queue
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32
    timer    *timer
    elemtype *_type
    sendx    uint   // send index
    recvx    uint   // receive index
    recvq    waitq  // list of recv waiters
    sendq    waitq  // list of send waiters
    bubble   *synctestBubble

    lock mutex
}
```

这段结构里，最值得盯住的是六个字段：

- `buf`：有缓冲 channel 的环形缓冲区。
- `qcount`：当前缓冲区里有多少元素。
- `dataqsiz`：缓冲区容量。
- `sendx` / `recvx`：环形缓冲区的写入位置和读取位置。
- `sendq` / `recvq`：阻塞的 sender / receiver 等待队列。
- `lock`：保护 `hchan` 以及阻塞在这个 channel 上的部分 `sudog` 字段。

![hchan 结构拆解](/images/go-channel-hchan-runtime-source/inline-01.png)

这已经能压住一个常见误解：channel 不是无锁结构。

源码注释写得很直接：`lock protects all fields in hchan`。`chansend`、`chanrecv`、`closechan` 这些关键路径都会拿 `c.lock`。

所以 channel 的重点不是“无锁”。

它真正帮你做的是：在一把 runtime 管理的锁下面，把数据移动、等待队列、goroutine 挂起和唤醒这些动作组织成一套稳定协议。

## makechan：有些 channel 只分配 hchan，有些还要分配 buffer

`make(chan T, n)` 最终会走到 `makechan`。

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

`mem == 0` 通常对应无缓冲 channel，或者元素大小为 0 的 channel。它不需要真正的缓冲数组。

如果元素里没有指针，runtime 可以把 `hchan` 和 `buf` 放在同一块连续内存里。这样 GC 不需要扫描 buffer 里的元素。

如果元素里有指针，`hchan` 和 `buf` 分开分配，buffer 要接受 GC 追踪。

这不是文章主线，但它解释了一个事实：channel 从创建开始，就不是一个抽象“管道”。runtime 需要根据元素类型、容量、GC 成本，决定它在内存里怎么落地。

## chansend：`ch <- v` 会先找 receiver，再考虑 buffer

发送入口是 `chansend(c, ep, block, callerpc)`。

你可以把它理解成五条路径。

### 第一条：nil channel，永久阻塞

```go
if c == nil {
    if !block {
        return false
    }
    gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
    throw("unreachable")
}
```

向 nil channel 发送，如果是阻塞操作，就直接 `gopark`，等待原因是 `waitReasonChanSendNilChan`。

它不会等到某一天突然变好。

因为 nil channel 背后没有 `hchan`，没有 `sendq`，也没有任何 goroutine 能来唤醒它。

这也是为什么 nil channel 常被用来在 `select` 里动态关闭某个 case：不是“这个 case 慢一点”，而是 runtime 根本不会把它纳入可执行路径。

### 第二条：已经有 receiver 在等，直接交给它

拿到锁之后，发送方先看 `recvq`：

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

这个顺序很重要。

如果已经有 receiver 阻塞在 `recvq` 里，发送方不会先把数据塞进 buffer，再让 receiver 去取。它会调用 `send`，把数据直接交给那个等待的 receiver。

源码注释说得很清楚：

```go
// Found a waiting receiver. We pass the value we want to send
// directly to the receiver, bypassing the channel buffer (if any).
```

也就是说，即使这是一个有缓冲 channel，只要此刻有 receiver 在等，runtime 也会优先做直接交付。

这背后的判断很朴素：人都已经在窗口前等着了，就没必要先把东西放进仓库，再让他从仓库里拿。

### 第三条：没有 receiver，但缓冲区有空间

如果没有等待的 receiver，runtime 才看 buffer：

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

这才是你脑子里那个“队列”画面。

`sendx` 指向下一次写入位置。写完后 `sendx++`，如果到头了就回到 0。`qcount` 增加。

所以有缓冲 channel 的 buffer 是环形队列。

但要注意：这只是 channel 的一条路径，不是 channel 的全部。

### 第四条：非阻塞发送，直接失败

`select` 的 `default` 分支会用到非阻塞语义。

如果没有 receiver，buffer 也满了，且 `block == false`：

```go
if !block {
    unlock(&c.lock)
    return false
}
```

这时 runtime 不会把当前 goroutine 挂起来。

它只是告诉上层：这次发送没选中。

### 第五条：真阻塞，把当前 goroutine 挂进 sendq

如果是普通 `ch <- v`，没有 receiver，buffer 也没空间，就只能阻塞。

runtime 会拿一个 `sudog`，把当前 goroutine、要发送的数据地址、channel 关联起来，然后挂到 `sendq`：

```go
gp := getg()
mysg := acquireSudog()
mysg.elem = ep
mysg.g = gp
mysg.c = c
gp.waiting = mysg
c.sendq.enqueue(mysg)
gp.parkingOnChan.Store(true)
gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceBlockChanSend, 2)
```

到这里，`ch <- v` 不再是“执行慢一点”。

当前 goroutine 已经被 runtime 挂起了。

它不是睡一会儿再试，而是等某个 receiver 或 close 操作来改变局面。

![chansend 和 chanrecv 的关键路径](/images/go-channel-hchan-runtime-source/inline-02.png)

## 无缓冲 channel：sender 把数据写到 receiver 的栈上

无缓冲 channel 最容易被说成“没有队列”。

这句话没错，但太轻了。

源码里更关键的地方在 `sendDirect`：

```go
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
    // src is on our stack, dst is a slot on another stack.
    dst := sg.elem
    typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.Size_)
    memmove(dst, src, t.Size_)
}
```

注释已经把场面说穿了：`src` 在当前 goroutine 的栈上，`dst` 是另一个 goroutine 栈上的位置。

无缓冲 channel 的 send 不是在缓冲区里排队，是 sender 把手伸到 receiver 的栈上。

当然，“伸手”只是帮助理解。真正发生的是 runtime 在持有 channel 锁并处理栈移动安全的前提下，用 `memmove` 完成跨 goroutine 栈的数据拷贝。

这就是无缓冲 channel 为什么天然带同步语义：数据交付和 goroutine 唤醒被绑在一起。

不是先放东西再通知。

是双方在同一个 rendezvous 点完成交接。

## chanrecv：接收路径和发送对称，但有一个细节很容易讲错

接收入口是 `chanrecv(c, ep, block)`。

它的大方向和发送对称：

1. nil channel：永久阻塞。
2. closed 且 buffer 空：返回零值，`received == false`。
3. 有等待的 sender：直接配对。
4. buffer 有数据：从 buffer 取。
5. 没有数据且需要阻塞：把当前 goroutine 挂进 `recvq`。

但“有等待 sender”这条路径要分两种情况。

源码里是这样：

```go
if sg := c.sendq.dequeue(); sg != nil {
    recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true, true
}
```

真正的逻辑藏在 `recv` 里：

```go
if c.dataqsiz == 0 {
    if ep != nil {
        recvDirect(c.elemtype, sg, ep)
    }
} else {
    qp := chanbuf(c, c.recvx)
    if ep != nil {
        typedmemmove(c.elemtype, ep, qp)
    }
    typedmemmove(c.elemtype, qp, sg.elem)
    c.recvx++
    if c.recvx == c.dataqsiz {
        c.recvx = 0
    }
    c.sendx = c.recvx
}
```

无缓冲 channel 很直观：receiver 直接从 sender 的栈上拿数据。

有缓冲 channel 呢？

如果有 sender 已经阻塞，说明 buffer 是满的。receiver 会从 buffer 头部取出一个元素，同时把这个 sender 要发送的值放进 buffer 尾部。

源码注释说这两个位置“map to the same buffer slot”，因为队列满时，取头和写尾会落在同一个槽位。

这个细节很重要。

不能简单说“有等待 sender 时，receiver 总是直接从 sender 拷贝”。那只对无缓冲 channel 成立。

对有缓冲且满的 channel，receiver 拿的是 buffer 头部旧值，sender 的新值被补进 buffer。

这才符合 FIFO。

## closechan：不是关门，是批量开门

很多人把 `close(ch)` 理解成“把 channel 关掉”。

这当然没错，但 runtime 里更准确的画面是：设置 `closed`，然后批量唤醒阻塞在这个 channel 上的 goroutine。

关键代码是：

```go
func closechan(c *hchan) {
    if c == nil {
        panic(plainError("close of nil channel"))
    }

    lock(&c.lock)
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel"))
    }

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
}
```

这里有三个动作：

第一，`c.closed = 1`。

第二，把 `recvq` 和 `sendq` 里的 goroutine 都取出来，先放进一个临时 `gList`。

第三，释放 channel 锁之后，再逐个 `goready`。

为什么不在持锁时直接唤醒？

`hchan` 结构里有注释：不要在持有这个锁时改变另一个 G 的状态，否则可能和栈收缩产生死锁。

所以 close 不是简单改一个标志位。

close(ch) 不是在关门，是打开所有阻塞 goroutine 的门。

只不过门打开之后，receiver 和 sender 的命运不同：

- 被唤醒的 receiver 得到零值，`ok == false`。
- 被唤醒的 sender 会继续走到 panic：`send on closed channel`。

如果一个 channel 上挂了大量阻塞 goroutine，close 的唤醒就是按队列逐个处理，复杂度和阻塞 goroutine 数量相关。

![close 唤醒阻塞 goroutine](/images/go-channel-hchan-runtime-source/inline-03.png)

## nil channel 和 closed channel：它们不是一类边界

nil channel 和 closed channel 经常被放在一起讲，但 runtime 行为完全不同。

nil channel 没有 `hchan`，所以：

```go
var ch chan int
ch <- 1   // 永久阻塞
<-ch      // 永久阻塞
close(ch) // panic: close of nil channel
```

closed channel 有 `hchan`，只是 `closed` 标志已经置位，所以：

```go
ch := make(chan int, 1)
ch <- 1
close(ch)

v, ok := <-ch // 先拿到缓冲区里的 1, ok == true
v, ok = <-ch  // 缓冲区空后拿零值, ok == false
ch <- 2       // panic: send on closed channel
close(ch)     // panic: close of closed channel
```

这也是很多线上 bug 的来源：

nil channel 的问题是“永远没人唤醒”。

closed channel 的问题是“所有人都醒了，但醒来后要按 closed 语义继续走”。

## selectgo：select 不是 O(1) 抽奖，它要组织 case、加锁、入队、清理

`select` 的入口在 `runtime/select.go`：`selectgo`。

它不是把所有 case 丢给一个神秘调度器随机挑一个。

它做的事情更具体。

### 第一步：生成随机 poll 顺序，跳过 nil channel

源码里会遍历所有 case：

```go
for i := range scases {
    cas := &scases[i]

    // Omit cases without channels from the poll and lock orders.
    if cas.c == nil {
        cas.elem = nil // allow GC
        continue
    }

    j := cheaprandn(uint32(norder + 1))
    pollorder[norder] = pollorder[j]
    pollorder[j] = uint16(i)
    norder++
}
```

nil channel 的 case 会被剔除出 poll order 和 lock order。

这解释了一个常见写法：把 channel 设为 nil，可以动态禁用某个 select 分支。

不是因为 nil channel “暂时不可读”。

而是它根本不会进入这轮可选集合。

### 第二步：按 channel 地址排序，决定加锁顺序

`select` 可能同时涉及多个 channel。runtime 必须给这些 channel 加锁，但不能乱加，否则容易死锁。

所以它会根据 `hchan` 地址生成 `lockorder`，再按顺序加锁。

这里也能看出，select 的成本和 case 数量相关。

不要把 select 说成一次常数时间的抽奖。

它至少要组织 case、生成 poll order、生成 lock order，还要扫描 case 看有没有可立即执行的路径。

### 第三步：先找能立即执行的 case

第一轮扫描会看每个 case 是否已经 ready：

```go
for _, casei := range pollorder {
    casi = int(casei)
    cas = &scases[casi]
    c = cas.c

    if casi >= nsends {
        sg = c.sendq.dequeue()
        if sg != nil {
            goto recv
        }
        if c.qcount > 0 {
            goto bufrecv
        }
        if c.closed != 0 {
            goto rclose
        }
    } else {
        if c.closed != 0 {
            goto sclose
        }
        sg = c.recvq.dequeue()
        if sg != nil {
            goto send
        }
        if c.qcount < c.dataqsiz {
            goto bufsend
        }
    }
}
```

接收 case 会检查：有没有等待 sender、buffer 有没有数据、channel 是否关闭。

发送 case 会检查：channel 是否关闭、有没有等待 receiver、buffer 有没有空间。

如果找到能走的分支，直接执行。

### 第四步：没有 ready case，就把当前 goroutine 注册到所有相关队列

如果没有 default，也没有任何 ready case，select 就要阻塞。

runtime 会给每个 case 分配一个 `sudog`，挂到对应 channel 的 `sendq` 或 `recvq`：

```go
for _, casei := range lockorder {
    casi = int(casei)
    cas = &scases[casi]
    c = cas.c
    sg := acquireSudog()
    sg.g = gp
    sg.isSelect = true
    sg.elem = cas.elem
    sg.c = c

    if casi < nsends {
        c.sendq.enqueue(sg)
    } else {
        c.recvq.enqueue(sg)
    }
}

gopark(selparkcommit, nil, waitReason, traceBlockSelect, 1)
```

这一步很关键：一个 select 阻塞，不是只挂在一个 channel 上。

它会把同一个 goroutine 对应的多个 `sudog` 注册到多个 channel 的队列里。

哪个 channel 先就绪，哪个路径就把它唤醒。

### 第五步：醒来后清理没选中的 case

被唤醒之后，runtime 还要把那些没选中的 `sudog` 从其他 channel 队列里清理掉：

```go
for _, casei := range lockorder {
    k = &scases[casei]
    if sg == sglist {
        casi = int(casei)
        cas = k
        caseSuccess = sglist.success
    } else {
        c = k.c
        if int(casei) < nsends {
            c.sendq.dequeueSudoG(sglist)
        } else {
            c.recvq.dequeueSudoG(sglist)
        }
    }
    releaseSudog(sglist)
    sglist = sgnext
}
```

这就是 select 复杂的地方。

它不只是随机选一个 case。

它要处理 nil case、随机轮询、公平性、加锁顺序、阻塞注册、唤醒后的清理，以及 send/recv/close 的不同语义。

## 回到那行代码：`ch <- v` 到底发生了什么

现在可以把整条链收起来。

一次 send 大概会这样走：

```text
ch <- v
  ↓
chansend
  ↓
如果 ch == nil：永久阻塞
  ↓
加锁，检查 closed
  ↓
如果 recvq 有 receiver：直接交付，唤醒 receiver
  ↓
否则如果 buffer 有空间：写入 buf[sendx]
  ↓
否则如果非阻塞：返回 false
  ↓
否则：当前 goroutine 包成 sudog，挂进 sendq，gopark
```

一次 recv 则是：

```text
<-ch
  ↓
chanrecv
  ↓
如果 ch == nil：永久阻塞
  ↓
加锁，检查 closed / buffer
  ↓
如果 sendq 有 sender：配对接收
  ↓
否则如果 buffer 有数据：从 buf[recvx] 取
  ↓
否则如果非阻塞：返回 false
  ↓
否则：当前 goroutine 包成 sudog，挂进 recvq，gopark
```

一次 close 是：

```text
close(ch)
  ↓
加锁，检查 nil / already closed
  ↓
closed = 1
  ↓
清空 recvq，receiver 醒来后得到零值和 ok=false
  ↓
清空 sendq，sender 醒来后 panic
  ↓
释放锁
  ↓
goready 批量唤醒
```

一次 select 是：

```text
selectgo
  ↓
跳过 nil channel case
  ↓
生成随机 poll order
  ↓
生成 lock order 并加锁
  ↓
扫描是否有 ready case
  ↓
没有 ready 且无 default：把当前 G 注册到多个 channel 队列
  ↓
gopark
  ↓
被某个 case 唤醒
  ↓
清理其他未命中的 sudog
```

读到这里，channel 就不再是一个“管道”的比喻了。

你应该能看见它的真实形状：一个 `hchan`，一个可选的环形 buffer，两条等待队列，一把锁，以及围绕 goroutine 挂起和唤醒设计出来的协议。

## 最后：别把 channel 神化，也别把它矮化成队列

channel 最容易被两种说法误导。

一种是把它神化：好像用了 channel，并发问题就天然优雅了。

另一种是把它矮化：说它不过就是一个队列，能 send、能 receive、能 close。

源码告诉你的都不是这两件事。

channel 的价值，不在于它“比 mutex 快”，也不在于它“无锁”。这些说法要么不准确，要么根本不是重点。

它真正提供的是一套语言级协作协议：

谁有数据，谁在等待，谁该被挂起，谁该被唤醒，关闭时谁拿零值，谁应该 panic，select 里哪个 case 应该赢。

这些事情如果都让业务代码自己处理，会非常容易错。

所以理解 channel 的关键，不是记住 `hchan` 里有多少字段。

而是记住这句话：

channel 不是魔法，它只是把数据移动、等待队列和 goroutine 调度打包成了一个可信的协作原语。

下次再看到 goroutine 卡在 `chan send`，你至少可以继续往下问：

它是在等 receiver，等 buffer 空位，还是等一个 close 把它唤醒？

这个问题问出来，排查就已经不是猜了。
