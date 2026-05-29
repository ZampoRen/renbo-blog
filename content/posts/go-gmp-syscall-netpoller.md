---
title: "Go 调度器真正厉害的地方：线程卡住了，P 不能跟着陪葬"
description: "阻塞 syscall 为什么不会拖死整个 Go 程序？net.Conn.Read 为什么不是让一条线程傻等？看懂 syscall handoff、netpoller 和 work stealing，关键是把等待、线程和 P 这张执行许可证拆开。"
date: 2026-05-29T20:53:18+08:00
draft: false
author: "任博"
tags: ["Go", "GMP", "调度器", "runtime", "syscall", "netpoller"]
categories: ["技术实战"]
cover: "/images/go-gmp-syscall-netpoller/cover.png"
toc: true
---

把 `GOMAXPROCS` 设成 1，再让一个 goroutine 卡在 `syscall.Read` 里。

按直觉，整个 Go 程序应该只剩一个执行名额。这个名额被卡住了，其他 goroutine 也该一起停。

但你真跑一下，会看到另一件事：reader 卡在系统调用里，主 goroutine 的 ticker 还在继续打印。

```bash
cd assets/go-gmp-series/02-syscall-netpoller/examples/syscall-handoff
GOMAXPROCS=1 GODEBUG=schedtrace=500,scheddetail=1 go run main.go
```

示例里一个 goroutine 进入阻塞的 `syscall.Read`，3 秒后才有人往 pipe 写数据。与此同时，主 goroutine 每 200ms 仍然输出：

```text
main: still running Go code while another M may be blocked in syscall
```

这件事比“goroutine 很轻”更关键。

它说明 Go 调度器真正厉害的地方，不是能创建很多 G，而是能在一条 OS 线程卡住时，把 Go 世界继续往前推。

第一篇我们讲过：G 是任务，M 是 OS 线程，P 是执行 Go 代码所必须持有的 runtime 资源。你可以把 P 类比成执行许可证。

第二篇要回答的问题是：如果拿着许可证的 M 卡进了 syscall，许可证怎么办？

答案很直接：P 不能陪 M 一起堵车。

![一次阻塞在 Go runtime 里的分叉](/images/go-gmp-syscall-netpoller/inline-runtime-flow.png)

## syscall 卡住的是 M，不该锁死 P

一个 goroutine 调用 syscall 时，真正被内核卡住的是运行它的 M，也就是那条 OS 线程。

这个 G 不能突然被别的 M 接过去继续执行。它正在一次系统调用中，现场在那条线程上，返回前不能随便搬家。

但 P 不一样。

P 不是这条线程的私人物品。P 是 Go runtime 的执行资格，持有它的 M 才能继续跑普通 Go 代码。如果 syscall 时间很长，P 还被压在那条 M 身上，就会出现很荒唐的浪费：明明还有别的 runnable G，却没有执行资格去跑。

所以进入 syscall 时，runtime 会做一组关键动作。

在 Go 1.25.4 的 `runtime/proc.go` 里，`reentersyscall` 会保存当前 G 的 syscall 现场，把 G 状态从 `_Grunning` 改成 `_Gsyscall`。随后把当前 M 和 P 拆开：

```go
gp.m.syscalltick = gp.m.p.ptr().syscalltick
pp := gp.m.p.ptr()
pp.m = 0
gp.m.oldp.set(pp)
gp.m.p = 0
atomic.Store(&pp.status, _Psyscall)
```

这几行就是 syscall handoff 的地基。

M 记录下 `oldp`，自己不再持有 P；P 进入 `_Psyscall`，处在一个“可能很快回来，也可能被别人拿走”的中间状态。

这里不能写得太粗暴。

普通 syscall 不是一进去就立刻把 P 交给别人。短 syscall 很快返回时，`exitsyscallfast` 会优先尝试拿回原来的 `oldp`。如果 `oldp` 还在 `_Psyscall`，M 就把它重新绑定回来，G 继续跑。

这条 fast path 很重要。因为很多 syscall 并不是真的长时间阻塞，如果每次都大动干戈地转移 P，调度成本反而会被放大。

真正的问题是慢 syscall。

如果 syscall 久不回来，或者路径本来就明确知道会阻塞，runtime 就不能让 P 长期停在 `_Psyscall` 里。

显式阻塞路径 `entersyscallblock` 会更直接：它把 G 置为 `_Gsyscall` 后，在系统栈上调用 `entersyscallblock_handoff`，里面就是一句：

```go
handoffp(releasep())
```

意思很清楚：P 放出来，交给调度器决定谁来接。

`handoffp` 也不是把 P 随手扔进空闲池。它会先看这个 P 或全局有没有 runnable G，有没有 trace reader，有没有 GC mark work，有没有必要启动或唤醒一个 M，有没有 netpoll/timer 相关工作。

也就是说，P 被交出来以后，runtime 第一反应不是“休息”，而是“有没有别人马上能用”。

这就是 Go 程序不容易被一个阻塞 syscall 拖死的核心。

线程可以卡住，P 不该陪葬。

这里顺手压住两个常见误解。

第一个误解是：P 被交出去以后，那个卡在 syscall 里的 G 也跟着换到别的 M 上跑。

不是。

G 还在原来的 syscall 里。它不是被迁移了，而是暂时变成 `_Gsyscall`。被挪走的是 P，是其他 Go 代码继续执行所需的 runtime 资源。等 syscall 返回，这个 G 才有机会重新进入 Go 调度世界。

第二个误解是：既然 P 会被释放，那 syscall 就没有代价。

也不是。

M 仍然是一条真实 OS 线程。大量长时间 syscall 仍然会让线程数量增加，仍然会带来内核调度、栈、上下文切换这些成本。Go 只是尽量不让“线程卡住”扩大成“执行资格也被卡住”。

所以正确的理解不是“Go 不怕阻塞”，而是“Go 尽量把阻塞的影响限制在该阻塞的那一层”。

这句话很关键。

G 可以等，M 可以卡，但 P 要尽量继续流动。

## sysmon 像巡场经理，专门盯着谁占着 P 不放

那普通 syscall 如果慢慢卡住，谁来发现？

答案是 sysmon。

很多人只把 sysmon 记成“抢占线程”，这太窄了。它更像 runtime 的巡场经理，不拿 P，也能周期性醒来看看场子里有没有异常。

在 `sysmon` 的循环里，它会做几件事：

- 如果超过一段时间没人 poll network，就做一次非阻塞 `netpoll(0)`；
- 调用 `retake(now)`，尝试回收卡在 syscall 里的 P，也尝试抢占运行太久的 G；
- 检查 force GC；
- 处理 schedtrace 等后台工作。

`retake` 里对 `_Psyscall` 的注释写得很直白：如果 P 在 syscall 里超过一个 sysmon tick，就可以考虑 retake。

但这里也要小心，不能把源码注释压成“超过 20us 一定抢”。源码还有条件：如果这个 P 本地没活、系统里还有 spinning 或 idle M，而且没超过 10ms，runtime 可能选择不回收，避免做无意义的搬运。

调度器不是越激进越好。

太保守，P 被 syscall 压住，别的 G 没法跑；太激进，每次短 syscall 都折腾 P，又白白增加调度成本。

好的调度器不是“看到阻塞就抢”，而是知道什么时候该等，什么时候该把许可证拿回来。

这也是理解 GMP 时最容易漏掉的层次：P 的流动不是一个开关，而是一组权衡。

## netpoller 更进一步：连 M 都不该傻等

syscall handoff 解决的是：M 卡住时，P 不能长期被它扣住。

但网络 I/O 还有更进一步的优化空间。

如果每个 `net.Conn.Read` 都让一条 M 长时间卡在内核 read 上，高并发服务很快会把线程堆起来。Go 不想这么干。

更常见的路径是：G 等，M 不傻等。

当 goroutine 调用 `net.Conn.Read`，它会经过 `internal/poll` 进入 runtime 的 poller 路径。fd 当前没 ready 时，runtime 不是让业务 M 一直堵在 read 上，而是把当前 G park 到 `pollDesc` 上。

`runtime/netpoll.go` 里的 `netpollblock` 逻辑可以粗略看成三步：

1. 找到 `pollDesc` 上的读槽 `rg` 或写槽 `wg`；
2. 把它从 `pdNil` 改成 `pdWait`；
3. 通过 `gopark(... waitReasonIOWait ...)` 让当前 G 进入等待。

这时 G 睡下去了，但 M 可以回到调度器，继续找别的 G 跑。

等底层平台 poller 返回事件，Linux 上是 epoll，macOS/BSD 上是 kqueue，Windows 上是 IOCP，runtime 会调用 `netpollready` 和 `netpollunblock`，把等在 `pollDesc.rg/wg` 里的 G 拿出来，放进一个可运行列表。

注意，这里返回的不是“网络数据”。

netpoller 返回的不是网络数据，而是一批该醒来的 G。

fd ready 只是说明等待条件满足了。这个 G 被重新标记为 runnable，之后还要等某个 M 持有 P，把它调度起来，再继续执行后面的 read/write 路径。

这个差别很重要。

如果你把 netpoller 理解成“一个 epoll 线程帮业务读数据”，就会把 Go 的网络模型讲歪。更准确的说法是：Go runtime 把平台 poller 集成进调度器，netpoll 是可运行 G 的来源之一。

它不绕开调度器。它最后仍然回到那件事：谁拿到 P，谁推进 Go 代码。

你也可以跑研究素材里的第二个示例：

```bash
cd assets/go-gmp-series/02-syscall-netpoller/examples/netpoller
GOMAXPROCS=1 GODEBUG=schedtrace=500,scheddetail=1 go run main.go
```

示例里 3 个 reader goroutine 先阻塞在 `conn.Read`，client 延迟写入后，reader 依次醒来。`schedtrace` 中也能看到多个 G 处于 `IO wait`。

这个实验不需要你记住每个 schedtrace 字段。先抓住主线就够了：网络等待被挂到了 poller 上，M 和 P 没必要一直陪着等。

这也是为什么 Go 服务里可以同时挂住大量网络连接。

如果每个连接的等待都对应一条长期阻塞的业务线程，高并发很快会变成线程数量问题。线程多了，不只是内存占用变大，调度器还要在更多内核线程之间切换，CPU 时间会被花在“谁该醒、谁该睡”上。

Go 的做法不是消灭等待。等待一定存在，网络另一端没发数据，你怎么都得等。

它做的是把等待放到更合适的位置：让 G 等 fd ready，让平台 poller 批量等事件，让 M 回来继续服务其他 G。

这也是 `IO wait` 容易被误读的原因。

看到一堆 goroutine 处在 `IO wait`，不要立刻得出“线程都堵住了”的结论。它更可能说明这些 G 被 park 在 netpoll 路径上，正在等外部 I/O。真正要继续看的，是 runnable G 是否堆积、P 是否有空闲、M 是否异常增长，以及下游服务是不是迟迟不给响应。

这层边界看清以后，排查网络阻塞才不会一上来就误判方向。

## P 没活干时，不会马上躺平

到这里，syscall 和 netpoller 解决的是“等待”问题。

但调度器还有另一个问题：一个 P 手里没活了怎么办？

很多简化版文章会说：先本地队列，再全局队列，再偷别人的。

这个说法能入门，但不够。

Go 1.25.4 里的 `findRunnable` 路径更像一张找活清单。一个 M 持有 P，发现当前没有直接可跑的 G，它不会马上睡觉，而是会沿着一串来源找工作：

- runtime 必须优先看的工作，比如 GC、trace、finalizer；
- 当前 P 的 `runnext` 和本地 runq；
- 全局 runq，避免全局队列饥饿；
- 非阻塞 `netpoll(0)`，看看有没有 I/O 醒来的 G；
- `stealWork`，从其他 P 那里找一批活；
- idle-time GC；
- 如果确实没活，释放 P，进入 blocking netpoll 或 stopm。

把 netpoll 放进这条路里，你就能看懂它为什么不是孤立模块。

fd ready 后的 G，最终还是要被调度器在“找活”时捞出来。sysmon 可以 poll，findRunnable 可以 poll，没活干时的 M 也可能 blocking poll。

调度器一直在问同一个问题：现在这张 P 许可证，给谁用最值？

如果本地没有，看看全局；全局没有，看看 I/O；I/O 没有，看看别人是不是忙得堆了一队 G。

这就是 work stealing 出场的位置。

## work stealing 偷的不是公平，偷的是浪费

work stealing 很容易被讲成一句“随机偷一半”。

这句话不是完全错，但太粗。

在 `stealWork` 里，runtime 最多尝试 4 轮。每轮从 `cheaprand()` 给出的随机起点开始，按 `stealOrder` 遍历所有 P；跳过当前 P，也跳过 idle P。只有最后一轮才会考虑 timer 和 `runnext` 这些更敏感的东西。

普通 runq 的偷取，确实可以简化成“偷一部分，接近一半”。`runqgrab` 里会从目标 P 的 runq 里抓一批，`runqsteal` 把这批放到当前 P 的 runq，并返回其中一个 G 让当前 M 立即执行。

为什么要批量偷？

因为偷一次要跨 P 操作，要考虑同步和竞争。只偷一个，太频繁；全拿走，又可能把别人掏空。抓一批，是在成本和均衡之间取一个工程上的折中。

所以 work stealing 的本质不是“追求绝对公平”。

work stealing 偷的不是公平，偷的是局部空闲造成的浪费。

有的 P 忙，有的 P 闲，这是本地队列必然带来的副作用。runtime 接受本地队列，是为了减少全局竞争；再用 stealing，把局部失衡拉回来。

这还是同一条主线：P 不能闲着。

syscall 里被压住的 P 要能拿回来；网络 I/O 等待不能让 M 傻站着；一个 P 本地没活，也要去别处找活。

Go 调度器最核心的能力，是把等待和执行资格拆开。

## 把这三条路放回一次真实请求

把 syscall handoff、netpoller、work stealing 分开讲，容易让人以为它们是三块独立拼图。

真实服务里不是这样。

想象一个很普通的 HTTP 服务：一个请求进来，handler 里查 Redis，查数据库，再写一点日志。代码看起来都是顺序调用，但 runtime 看到的是一连串状态转换。

goroutine 开始跑时，它只是某个 P 本地队列里被取出来的 G。M 持有 P，把它切到 `_Grunning`。如果它发起网络读写，fd 暂时没 ready，这个 G 会被 park 到 pollDesc 上，状态进入等待。M 不需要陪它等数据，而是回到调度器。

接下来这个 P 可能会跑另一个请求的 G，也可能处理一个 timer，也可能发现本地没活，去全局队列看一眼，再去 netpoll 看看有没有刚醒的 I/O G。

如果这一圈都没有，它还会去别的 P 那里偷活。

这时第一个请求并没有“消失”。它只是挂在 netpoller 后面，等 fd ready。等 epoll 或 kqueue 返回事件，runtime 把它重新放回 runnable 世界。之后某个 M 持有某个 P，再把它捞出来继续执行。

如果中途不是网络 I/O，而是一次真正会卡住线程的 syscall，路径又不一样：G 进 `_Gsyscall`，M 可能卡在内核里，P 进入 `_Psyscall`。短 syscall 返回，M 可能拿回 oldp；长 syscall 卡住，sysmon 或 blocking 路径会把 P 交给别人。

这就是 Go 调度器最容易被低估的地方。

它不是在维护一个“goroutine 列表”，也不是简单地把 goroutine 平均摊到线程上。它在维护一组执行资格的流动：哪些 G 该等，哪些 M 可以继续干活，哪些 P 不能闲着，哪些 P 不能被卡住的 M 长期扣住。

如果你只看用户代码，会觉得这些调用都只是“阻塞了一下”。

如果你从 GMP 看，会发现每一种阻塞的代价都不一样：

- G 等 I/O：通常可以 park，M 和 P 继续被调度器利用。
- M 卡 syscall：G 不能迁走，但 P 可以被释放或回收。
- P 本地没活：不是睡觉，而是先找全局、找 netpoll、找别的 P。
- G 跑太久：后面还要靠抢占和 safe point 让它别长期霸占 P。

这也是排查问题时很有用的边界。

看到大量 goroutine 在 `IO wait`，不等于线程都被打满；很多时候它们只是挂在 poller 上等 fd ready。看到很多 M 在 syscall，也不等于 P 一定被占死；要继续看 P 的状态、runnable G 数量、sysmon 是否能 retake。看到某个 P 本地队列空，也不等于它真的闲；它可能正在偷活、poll network，或者被 GC 相关路径接管。

“阻塞”这个词太粗了。它把 G、M、P 三层差异全部抹平。

而 Go 调度器恰恰是靠不抹平这些差异，才把高并发服务撑起来。

## 下次看到 goroutine 阻塞，先问四个问题

很多线上问题会被一句“goroutine 阻塞了”糊住。

这句话太粗。阻塞的到底是什么？

下次你看 goroutine dump、schedtrace，或者排查一个 Go 服务为什么延迟抖，先按四个问题拆：

1. G 在什么状态？是 `_Gwaiting`、`_Gsyscall`、`_Grunnable`，还是 `_Grunning`？
2. M 有没有被 OS syscall 卡住？如果卡住，它还持有 P 吗？
3. P 在什么状态？是 `_Prunning`、`_Psyscall`、`_Pidle`，还是被 GC/STW 相关路径影响？
4. 谁会把工作接回来？是 `exitsyscallfast`、`handoffp/startm`、`sysmon/retake`、`netpollready/injectglist`，还是 work stealing？

这四个问题，比背函数名更有用。

因为它们帮你把“阻塞”拆开了：G 可以等，M 可以卡，P 可以被释放或重新分配。

一旦这层拆开，你就不会再把 Go 调度器想成一个普通线程池。

线程池的直觉是：线程拿任务，任务卡住，线程就卡住。Go runtime 的直觉是：任务、线程、执行资源要分开管理。卡住一处，不该把整套执行资格一起锁死。

这也是为什么第一篇一直强调 P 不是 CPU 核心，而是执行 Go 代码的 runtime 资源。

第二篇只是把这个判断放进了更真实的场景里：syscall、netpoll、work stealing、sysmon、GC，都在争夺或归还这张许可证。

当然，本文讲的是 Go 1.25.4 runtime 当前实现下适合建立心智模型的主线。调度顺序不是语言规范承诺，平台后端也会影响 netpoll 的具体实现。

但主线不会变：

P 是 Go 世界的执行资格。谁持有它，谁才能推进普通 Go 代码；谁长期占着不用，runtime 就要想办法把它拿回来。

看懂这一点，再看第三篇的抢占和 GC，就不会把它们当成两个新话题。

抢占要解决的是：一个 G 跑太久，怎么让出 P。

GC 要解决的是：业务代码之外，runtime 自己的标记、扫描和 STW，怎么和 P 协作。

归根到底，还是那张许可证怎么流动。

如果你只记一句话，就记这句：线程可以等，P 要流动。

后面继续写 GMP 调度器第三篇：抢占、GC 和 safe point。关注我，下一篇把“一个 goroutine 死循环为什么不该拖死 GC”讲清楚。
