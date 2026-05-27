---
title: "select 不是 O(1) 抽奖，case 越多，runtime 要干的活越多"
description: "沿着 Go 1.25.4 runtime/select.go 拆开 selectgo：poll order、lock order、nil channel 剔除、阻塞注册和唤醒清理。select 的随机性不是免费午餐。"
date: 2026-05-27T10:35:00+08:00
draft: false
author: "任博"
tags: ["Go", "channel", "runtime", "源码分析", "并发编程"]
categories: ["技术实战"]
cover: "/images/go-channel-selectgo/cover.png"
toc: true
---

很多人写 `select`，脑子里都有一个抽奖箱。

几个 case 往里一扔，runtime 随机摸一个。哪个 channel 先 ready，哪个就中；都没 ready，就看 default；没有 default，就等。

这个理解不能说全错，但太轻了。

`select` 不是把 case 丢给调度器随便挑一个。它要先整理 case，跳过 nil channel，生成随机轮询顺序，再按 channel 地址排出加锁顺序。没有任何 case 立即可走时，它还要把当前 goroutine 同时注册到多个 channel 的等待队列里。醒来之后，还得把没选中的那些等待记录清掉。

![selectgo 不是 O(1) 抽奖](/images/go-channel-selectgo/cover.png)

所以别把 `select` 想成一次 O(1) 抽奖。

它更像一次小型调度：先洗牌，再排锁，再扫描，必要时多点挂号，最后清场。

这篇接着前两篇，把 `selectgo` 单独拆开。不是为了劝你少用 `select`。恰恰相反，`select` 很好用，但你最好知道它到底帮你付了哪些成本。

## nil channel 不是“暂时不可用”，而是直接出局

先从一个最常见的写法说起。

很多代码会用 nil channel 动态关闭某个 `select` 分支：

```go
var ch <-chan int

if enabled {
    ch = realCh
}

select {
case v := <-ch:
    handle(v)
case <-ctx.Done():
    return
}
```

如果 `enabled == false`，`ch` 是 nil。这个接收 case 会怎么样？

很多人的直觉是：它还在 select 里，只是永远不会 ready。

runtime 的处理更干脆：它不会把这个 case 放进本轮候选集合。

Go 1.25.4 的 `runtime/select.go` 里，`selectgo` 先遍历所有 case：

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

这里的注释已经很直白：没有 channel 的 case，会被从 `pollorder` 和 `lockorder` 里省略。

也就是说，nil channel case 不是“参与抽奖但永远不中”。

它是连抽奖券都没拿到。

这就是 nil channel 在 `select` 里好用的原因。你可以把某个 channel 设成 nil，动态禁用一个分支，而不用额外拆两套 select。

但它也解释了另一个坑：如果某个本该工作的 channel 因为初始化漏掉、配置分支没走到、生命周期处理错了而保持 nil，你的程序不会收到任何提醒。这个 case 就像从来没存在过。

nil channel 最危险的地方，不是慢。

是它没有运行时对象。

## poll order：随机化不是魔法，是先洗一遍 case

`select` 的随机性从哪里来？

不是调度器最后闭眼挑一个，而是在生成 `pollorder` 时做了一次随机插入。

简化来看，每遇到一个非 nil case，runtime 会在已有顺序里随机挑一个位置，把当前 case 插进去：

```go
j := cheaprandn(uint32(norder + 1))
pollorder[norder] = pollorder[j]
pollorder[j] = uint16(i)
norder++
```

这一步的目的，是让后面扫描 ready case 时，不总是按源码顺序来。

否则就会出现一个很烦的偏向：如果多个 case 同时 ready，写在前面的 case 总是更容易赢。你以为自己写的是并发选择，实际写成了优先级队列。

`pollorder` 就是为了解掉这种固定顺序偏向。

但这里要压住一个误解：随机化不是免费午餐。runtime 仍然要遍历 case，跳过 nil，生成 `pollorder`。case 越多，这一步要处理的东西就越多。

这不是让你把 5 个 case 改成 2 个 case。

真正该警惕的是另一种写法：把 `select` 当成无限扩展的事件总线，几十上百个 channel 全塞进去，然后以为 runtime 会用一个常数时间的神秘机制帮你解决。

不会。

`select` 的公平感，是 runtime 先做了一轮洗牌换来的。

## lock order：select 真正麻烦的地方，是它可能碰到多个 channel

如果 `select` 只负责随机扫 case，事情还没那么复杂。

真正让它变重的是：一个 select 可能同时涉及多个 channel，而每个 channel 背后都有自己的 `hchan.lock`。

前两篇讲过，channel 不是无锁结构。`chansend`、`chanrecv`、`closechan` 都会围绕 `hchan.lock` 改状态。

那 select 呢？

它要同时观察多个 channel：这个能不能收，那个能不能发，哪个已经 close，哪个 buffer 有空间。要看这些状态，就不能完全不加锁。

问题来了：多个 goroutine 同时 select 多个 channel，如果大家加锁顺序不同，就很容易互相卡住。

所以 `selectgo` 会根据 `hchan` 地址生成 `lockorder`，再按这个顺序加锁：

```go
// sort the cases by Hchan address to get the locking order.
// simple heap sort, to guarantee n log n time and constant stack footprint.
for i := range lockorder {
    j := i
    c := scases[pollorder[i]].c
    for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
        k := (j - 1) / 2
        lockorder[j] = lockorder[k]
        j = k
    }
    lockorder[j] = pollorder[i]
}
```

这段注释也很关键：按 `hchan` 地址排序，使用 heap sort，保证 `n log n` 时间和常量栈空间。

你看，源码已经把成本写出来了。

`select` 不是只做“随机挑一个”。它还要为多个 channel 排出一个稳定的加锁顺序。这个顺序不是为了好看，是为了避免死锁。

所以 case 数量多的时候，成本不只是多扫几个 if。

你还要为这些 channel 的锁关系付账。

这也是为什么“select 是 O(1) 抽奖”这个说法很误导。它把最重要的组织工作藏掉了：poll order 要生成，lock order 要排序，锁要按顺序拿。

随机只是表面，排锁才是底层代价。

## 第一轮扫描：先看有没有可以立刻走的 case

生成 `pollorder` 和 `lockorder` 之后，runtime 会先把相关 channel 锁起来，然后做第一轮扫描。

这轮扫描只问一个问题：有没有 case 现在就能走？

源码大致是这样：

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

接收 case 会按顺序看三件事：

有没有 sender 已经在 `sendq` 等？buffer 里有没有数据？channel 是否已经 closed？

发送 case 也类似：

channel 是否已经 closed？有没有 receiver 正在 `recvq` 等？buffer 有没有空间？

如果某个 case 现在就能执行，runtime 就直接跳到对应路径：`recv`、`bufrecv`、`send`、`bufsend`、`rclose` 或 `sclose`。

这一步解释了一个线上现象：`select` 并不是先把自己挂起来，再等别人唤醒。它会先尽力找一条立即可走的路。

如果你写了 default，且没有任何 case ready，它会返回 default。

如果没有 default，才进入下一步：阻塞。

这就是为什么 `select { case <-ch: ... default: ... }` 常被拿来做非阻塞接收。它不是特殊语法糖，而是 `selectgo` 在第一轮扫描找不到 ready case 后，因为 `block == false` 直接返回。

## 没有 ready case：一个 goroutine 会被注册到多个队列里

真正值得盯住的是阻塞路径。

没有 default，也没有任何 case 立即可走时，`select` 不能只挂在一个 channel 上。因为任何一个 case 未来都可能先 ready。

所以 runtime 会为每个 case 准备一个 `sudog`，把同一个 goroutine 分别挂到多个 channel 的等待队列里。

源码里的第二轮是这样：

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

这段代码很容易被低估。

一个阻塞中的 `select`，不是“这个 goroutine 等在 select 上”这么抽象。它在 runtime 里会变成多个等待记录：这个 case 对应一个 `sudog`，挂到这个 channel；那个 case 对应另一个 `sudog`，挂到另一个 channel。

同一个 goroutine，会在多个门口排队。

谁先开门，谁把它叫醒。

这也是 select 成本和 case 数量相关的另一个原因。case 越多，阻塞时要创建和入队的 `sudog` 越多，醒来后要清理的东西也越多。

所以如果你在热路径里写一个巨大 select，不要只看语法有多清爽。语法是一行，runtime 不是一行。

## 醒来之后：选中的留下，没选中的要清掉

`select` 被某个 channel 唤醒之后，事情还没结束。

因为当前 goroutine 之前注册到了多个 channel 队列里，现在只有一个 case 赢了，其他 case 对应的 `sudog` 都要从各自队列里移除。

源码第三轮写得很清楚：

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

这一步如果不做，会发生什么？

安静的 channel 队列里会堆满已经没用的等待记录。以后别的 goroutine 来 send 或 recv，可能会撞到这些脏记录。runtime 当然不会允许这种事发生，所以它必须清理。

这就是 select 的完整画面：

先把 nil case 剔掉，给剩下的 case 生成随机 poll order；再按 channel 地址排 lock order；第一轮扫描找 ready case；找不到就给每个 case 建 `sudog` 并入队；醒来以后，保留选中的，清掉没选中的。

你看，它不是抽一次奖这么简单。

它是一场有入场、排队、叫号和清场的流程。

![selectgo 的一轮选择](/images/go-channel-selectgo/inline-01.png)

## 这对写代码有什么用

理解这些细节，不是为了写代码时处处手算 runtime 成本。

大多数业务里的 select case 很少，2 个、3 个、4 个，很正常。你完全应该继续用。

这篇文章也不是要把 `select` 讲成性能雷区。那同样是误读。

真正该避免的是：明明在写调度，却假装自己只是在写几个分支；明明需要表达优先级、背压、取消和退出，却全塞进一个越来越大的 select，让后来的人只能靠猜。

`select` 最适合表达清楚的小规模等待关系。一旦它开始承担“路由中心”的角色，你就应该停下来问一句：这里到底是控制流，还是我偷懒写出来的隐形调度器？

真正有用的是，它能帮你压住三类误判。

第一，别把 nil channel 当成“暂时没数据”。

在普通 send/recv 上，nil channel 会永久阻塞；在 select 里，nil case 会被剔出 `pollorder` 和 `lockorder`。如果你是故意用 nil 禁用分支，这是好设计。如果不是，排查方向就应该回到初始化和赋值路径。

第二，别把 default 当成随手兜底。

有 default 的 select，本质上是在说：没有 case 立即 ready，我不等。这个语义很强。你要么记录丢弃，要么返回错误，要么降级。不要把 default 写成“先避免阻塞再说”，最后把背压、取消和丢失都藏起来。

第三，别把大 select 当成免费路由器。

如果你把几十个 channel 都塞进一个 select，让它承担事件分发、优先级、取消、超时、状态切换，那就要意识到：每轮 select 都要组织这些 case。阻塞时还要把同一个 goroutine 注册到多个队列里，醒来后再清理。

这不是说不能写。

而是你要知道自己买的是什么。

如果 case 数量是固定的、小规模的，select 很清楚。如果 case 数量动态膨胀，或者你想表达的是“从 N 个来源里统一消费”，有时用一个 fan-in goroutine、一个聚合 channel，或者显式的调度结构，会比巨大 select 更容易解释，也更容易观测。

## 排查 select 问题，先问这五个问题

下次你看到 goroutine dump 里卡在 `[select]`，不要只说“它在等 channel”。

这个信息太粗。

你可以顺着 `selectgo` 问五个更具体的问题。

第一，哪些 case 里的 channel 可能是 nil？

如果某个分支被动态置 nil，那是设计；如果不是设计，就查初始化顺序、配置开关、关闭流程。nil case 不会参与这轮选择。

第二，是否有 default？

有 default，说明这段代码选择“不等”。如果你本来想表达等待取消、等待超时或等待结果，却因为 default 直接绕走，那 bug 可能不是阻塞，而是过早放弃。

第三，多个 case 同时 ready 时，你是否依赖源码顺序？

不要依赖。`pollorder` 会随机化扫描顺序。select 不是“写在前面的优先”。如果你真要优先级，就要显式写两层 select 或额外状态机，不要把优先级寄托在 case 排列上。

第四，case 数量是不是被当成无限扩展点？

小规模 select 是清晰控制流。巨大 select 往往是隐形调度器。后者的问题不是“慢一点”这么简单，而是调试时你很难回答：这一轮到底有哪些 case 有效、哪些被 nil 禁用了、哪个 channel 先唤醒了它。

第五，醒来后有没有清楚处理 send/recv/close 的语义差异？

接收 closed channel 会返回零值和 `ok=false`；向 closed channel 发送会 panic；nil channel 在 select 里会被跳过。这些差异如果在业务层被混成一句“channel 不可用”，排查很快会乱。

把这五个问题问完，`select` 就不再是一个黑箱。

它会变成一组可以拆开的运行时动作。

## 最后：select 的优雅，是 runtime 替你做了脏活

三篇 channel 写到这里，整条线可以收住了。

第一篇讲 `ch <- v`：发送不是一个动作，而是 nil、直接交付、buffer、非阻塞失败、挂进 `sendq` 五条路。

第二篇讲 `<-ch` 和 `close(ch)`：接收不是发送的镜像，close 也不是关门，而是批量叫醒等待中的 goroutine。

这一篇讲 `select`：它不是 O(1) 抽奖，而是一次围绕多个 channel 的组织工作。

channel 最容易被低估的地方就在这里。它的 API 很短，短到让人以为背后也很简单。

但 runtime 里真正发生的是：队列、锁、跨栈拷贝、等待记录、随机轮询、加锁排序、挂起、唤醒和清理。

这些脏活平时不需要你写，是 Go 替你包起来了。

但你不能因此假装它不存在。

`select` 的优雅，不是因为它没有成本。

是因为它把成本藏进了一套可靠的运行时协议里。

下次再写一个很大的 select，或者在 dump 里看到 `[select]`，别只问“哪个 case 中了”。

先问：这轮 select 里，哪些 case 真的进场了？runtime 又替你排了多少队、拿了多少锁、清了多少尾巴？

能问到这一层，Go 并发就不再只是会写语法了。

如果你想继续把 Go 并发从“能跑”推进到“能解释线上卡在哪里”，可以继续关注这个系列。channel 表面简单，真正的代价都藏在等待、唤醒和清理的边界里。
