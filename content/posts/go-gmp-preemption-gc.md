---
title: "Go 为什么不怕 goroutine 死循环：抢占不是玄学，是一套停机协议"
description: "一个没有函数调用的 for 死循环，为什么没有把 Go 程序和 GC 一起拖死？从同步抢占、asyncPreempt、safe point、sysmon retake 到 GC worker，这篇把 Go 调度器的第三个关键问题讲清楚：正在跑的 goroutine，runtime 到底怎么把它停下来。"
date: 2026-06-02T17:36:48+08:00
draft: false
author: "任博"
tags: ["Go", "GMP", "调度器", "runtime", "GC", "抢占"]
categories: ["技术实战"]
cover: "/images/go-gmp-preemption-gc/cover.png"
toc: true
---

把 `GOMAXPROCS` 设成 1，再启动一个没有函数调用、没有 I/O、没有 sleep 的死循环。

按很多人对 goroutine 的理解，程序应该完了：唯一的 P 被这个 goroutine 一直占着，其他 goroutine 没机会运行，GC 想停世界也停不下来。

但 Go 1.14 之后，这个直觉已经不完整了。

你可以跑一个很小的实验：一个 goroutine 进入 tight loop，另一个 goroutine 每 200ms 打印一次。默认运行时，ticker 还能继续打印；如果关掉异步抢占，程序可能在预期时间内完全没有输出。

```bash
cd assets/go-gmp-series/03-preemption-gc
GOMAXPROCS=1 go run examples/preemption-demo.go
GOMAXPROCS=1 GODEBUG=asyncpreemptoff=1 go run examples/preemption-demo.go
```

本机 Go 1.25.4 验证结果是：默认情况下 ticker 正常输出并退出；关闭 async preempt 后，2 秒内没有任何输出，命令超时。

这件事不是“goroutine 很轻”能解释的。

它真正说明的是：现代 Go 调度器不再完全依赖 goroutine 自觉让出 CPU。runtime 有能力发现某个 G 占 P 太久，然后用同步抢占或异步抢占，把它带回调度体系里。

但这里最容易讲歪。

Go 不是每 10ms 硬切一次 goroutine，也不是收到一个信号就能在任意机器指令上停住。抢占从来不是“想停就停”。它更像一份停机协议：runtime 要能停下这个 G，还要保证 GC 看得懂它的栈、寄存器和对象引用。

抢占不是“想停就停”，而是“停了以后 GC 还能看懂”。

![Go 抢占从巡检到调度的主路径](/images/go-gmp-preemption-gc/inline-preemption-flow.png)

## Go 1.14 之前，抢占更像“请你下次路过时让一下”

先把老路径讲清楚，否则 asyncPreempt 会显得像魔法。

Go 早期的抢占主要依赖同步 safe point。最典型的位置，是函数调用序言里的栈检查。

每个 goroutine 有自己的栈边界。函数调用时，编译器生成的序言会检查当前栈是否够用。如果不够，就进入 `newstack` 路径处理扩栈。

runtime 复用了这件事。

当它想抢占某个正在运行的 G，会把这个 G 的 `gp.preempt` 设成 `true`，并把 `gp.stackguard0` 改成一个特殊值 `stackPreempt`。下一次这个 G 进入函数调用序言时，栈检查会“失败”，进入 runtime。runtime 再识别出来：这不是普通栈扩容，而是一次抢占请求。

可以把它简化成这条路径：

```text
runtime 想抢占某个 G
  -> gp.preempt = true
  -> gp.stackguard0 = stackPreempt
  -> G 下一次进入函数调用序言
  -> 栈检查失败
  -> 进入 runtime 抢占处理
```

这套机制很聪明，因为它不需要给每条用户指令都插检查点。函数调用本来就要做栈检查，顺手把抢占请求塞进去，成本低，也容易保证状态可理解。

问题也在这里。

如果一个 goroutine 一直在没有函数调用的 tight loop 里，它就很久不会经过这个检查点。runtime 可以把 `stackguard0` 改掉，但对方不路过检查口，抢占就很难及时发生。

这就是 Go 1.14 之前容易被死循环刺穿的地方。

不是调度器不会抢，而是它主要靠“你下次经过门口时停一下”。如果你一直不经过门口，它就只能等。

Go 1.14 解决的不是公平的全部问题，而是“不调用函数就不让路”的尖刺问题。

## asyncPreempt：信号只是敲门，不是直接接管

Go 1.14 引入非协作式的异步抢占以后，runtime 多了一条更主动的路径。

在 Unix-like 平台上，runtime 会向目标 M 发送一个抢占信号。当前源码里，这个信号是 `SIGURG`。它不是随便挑的：Go 源码注释里写得很清楚，选择它是因为调试器通常会 pass-through，和 libc 内部信号冲突概率低，可以偶发出现，macOS 这类平台也没有 real-time signals 可依赖，而且 SIGURG 原本的 socket urgent condition 在现代程序里很少被真正使用。

但这一步只是敲门。

signal handler 不会在信号处理函数里直接运行调度器。调度器不能随便在 signal handler 里干复杂事情，它可能分配内存、启动线程、改调度状态，这些都不是 signal handler 该做的。

更准确的路径是：

1. sysmon 或其他路径发现某个 P 上的 G 跑太久，调用 `preemptone`。
2. `preemptone` 设置 `gp.preempt = true` 和 `gp.stackguard0 = stackPreempt`。
3. 如果平台支持 async preempt，且没有关闭，runtime 再对目标 M 发起 `preemptM`。
4. 信号处理器进入后，检查这个 G 是否确实想被抢占，以及当前 PC/SP 是否处在 async safe point。
5. 如果满足条件，handler 修改信号上下文，让线程恢复时看起来像刚刚调用了 `asyncPreempt`。
6. `asyncPreempt` 保存寄存器，进入 `asyncPreempt2`。
7. 最后根据场景走 `gopreempt_m` 回到 runnable，或者走 `preemptPark` 停到 `_Gpreempted`。

这里最关键的是第 4 步。

信号来了，不代表抢占一定发生。runtime 还要问：这个位置能不能停？停下来以后，GC 能不能正确扫描？当前是不是 Go 代码？栈空间够不够注入 `asyncPreempt`？是不是 runtime、reflect、assembly、atomic sequence、write barrier 相关不安全区域？

只要这些条件不满足，这次信号就只能确认一下，等下一次机会。

所以不要把异步抢占写成“Go 可以在任意机器指令上强行切走 goroutine”。这句话听起来很强，其实是错的。

异步抢占没有取消 safe point，它只是把 safe point 从“函数入口附近”扩展到了“更多经过证明可停的位置”。

## safe point：真正的安全，不是暂停本身

很多人第一次看到 safe point，会把它理解成“安全暂停的位置”。

这还不够。

Go runtime 在 `preempt.go` 的文件头注释里，把 safe point 分成三类：

1. blocked safe point：goroutine 已经被调度走、阻塞在同步原语上，或者在 system call 中。
2. synchronous safe point：运行中的 goroutine 主动检查抢占请求。
3. asynchronous safe point：用户代码中的某些指令位置，runtime 可以用信号停止 goroutine，并用保守方式扫描 stack/register roots。

这三类的共同点不是“停住了”。

共同点是：停住以后，runtime 和 GC 还能理解这个 goroutine 的状态。

在 blocked 和 synchronous safe point 下，goroutine 的 CPU 状态比较小，GC 有完整的 stack 信息，可以精确扫描。async safe point 难一点，因为 G 是在用户代码中间被信号打断的，寄存器和栈里可能有临时值，所以需要保守扫描寄存器和栈。

这也是为什么 runtime 要拒绝一些位置。

比如 PC 不在 Go 代码里，拒绝。栈空间不够注入 `asyncPreempt` 调用，拒绝。处在 compiler 标记的 unsafe point，拒绝。很多 assembly 函数没有 locals pointer maps，也拒绝。runtime、internal runtime、reflect 相关函数，当前实现里也会更保守。

你可以把 safe point 理解成 Go runtime 和 GC 之间的一份停机协议：

- 调度器说：我可以让这个 G 暂时停一下。
- GC 说：可以，但你得把它停在我能看懂的位置。
- 编译器说：哪些位置能看懂，我会把元信息标出来。
- signal handler 说：如果当前位置不合适，我不硬来。

safe point 的本质，是 runtime 和 GC 之间的一份停机协议。

这句话比“Go 有抢占式调度”更重要。

因为线上排查时，真正害人的往往不是不知道 Go 能抢占，而是把“能抢占”误读成“任何时候都能立刻抢占”。

## sysmon retake：后台巡检员怎么发现谁占 P 太久

那谁来发现一个 G 跑太久？

还是 sysmon。

上一篇讲 syscall 和 netpoller 时，sysmon 已经出现过。它不是 G/M/P 里的任何一个，也不需要持有 P 才能运行。它更像 Go runtime 的后台巡检员，周期性醒来看看有没有事情需要处理。

在 `sysmon` 循环里，`retake(now)` 会做两类事。

第一类，是对 `_Prunning` 或 `_Psyscall` 的 P 检查同一个 `schedtick` 是否持续太久。当前 Go 1.25.4 源码里，`forcePreemptNS` 是 `10 * 1000 * 1000`，也就是 10ms。如果同一 schedtick 超过这个时间，runtime 会调用 `preemptone(pp)`。

第二类，是处理 `_Psyscall`。如果某个 M 卡在 syscall 里，P 也处在 `_Psyscall`，而系统里又有其他工作需要推进，runtime 可以把这个 P 从 syscall 状态里拿回来，交给别的 M。

这两类动作看起来不一样，底层判断其实一致：P 不能被长期浪费。

一个 G 跑太久，P 被它长期占住，别的 G 没机会。一个 M 卡在 syscall 里还扣着 P，别的 Go 代码也没机会。sysmon 的 retake 就是在后台问：这张执行许可证是不是该被拿回来？

但这里也要压住一个误解：10ms 不是硬实时调度周期。

源码里的 `preemptone` 注释明确说，这是 best-effort。它可能通知失败，可能通知到错误时机，即使通知到了，真正的抢占也要等到未来某个合适位置发生。

所以更准确的说法是：10ms 是 sysmon 发起抢占请求的阈值，不是 Go 对 goroutine 做硬切片的承诺。

这层区别很关键。

如果把它讲成“Go 每 10ms 切换一次 goroutine”，读者会形成一个过于整齐的模型。真实 runtime 没这么整齐。它一直在做工程折中：能及时打断，但不能乱停；能回收 P，但不能每次短 syscall 都过度搬运；能让 GC 推进，但不能把业务 goroutine 当成随便可切的黑盒。

## GC 不是单独来一脚，它要调度器配合

抢占式调度和 GC 的关系，很多文章只写到“死循环不会拖死 GC”。这句话方向对，但太粗。

Go GC 不是“把全世界暂停住，然后自己慢慢扫完整个堆”。现代 Go GC 是并发 mark-sweep。当前 `mgc.go` 文件头注释给出的高层流程里，确实有 STW：sweep termination 要 STW，mark termination 也要 STW。但中间的 mark phase 是并发执行的，write barrier、mutator assists、background mark workers 都会参与。

换句话说，GC 和调度器不是两个互不相干的模块。

GC 要停世界时，需要所有 P 和相关 G 到达可控状态。正在运行的 goroutine 如果一直不让路，STW 就会被拖慢。safe point 和抢占，就是把“正在跑的人”带到可检查状态的机制。

GC 并发标记时，也离不开调度器。background mark workers 是 per-P 的后台 goroutine。mark phase 期间，调度器会在 `findRunnable` 路径里尝试安排 GC worker。P 空闲时，可以跑 idle-time marking；GC 需要吞吐时，也可能运行 dedicated 或 fractional worker。

分配路径还有 mark assist。

如果某个 goroutine 分配太快，欠了 GC 标记债，`gcAssistAlloc` 会让它在分配路径上帮 GC 做一部分 scan work。这个机制的直觉很简单：谁分配得多，谁帮 GC 多干点活。它不是调度器把你神秘地“抓去 GC”，而是在分配路径上把一部分 GC work 反馈给 mutator，帮助并发标记追上应用分配速度。

GC 不只是暂停世界，它还要组织世界一起干活。

这也是第三篇必须和前两篇连起来看的原因。

第一篇讲 P，你会知道 P 是执行 Go 代码的许可证，也是很多 per-P runtime 资源的承载点。第二篇讲 syscall 和 netpoller，你会知道 P 不能陪阻塞的 M 一起堵车。到第三篇，P 又变成 GC 协作的关键入口：谁持有 P，谁推进业务；谁持有 P，也可能推进 GC。

Go runtime 真正维护的不是一个简单的 goroutine 队列，而是一套执行资格、等待状态、GC 进度之间的平衡。

## 一个死循环背后的完整链路

现在回到开头那个死循环。

当 `GOMAXPROCS=1` 时，程序只有一个 P。一个 goroutine 进入没有函数调用的 tight loop，另一个 ticker goroutine 也想运行。

在旧模型里，你会觉得 ticker 没机会了。因为唯一的 P 被 tight loop 占住，而 tight loop 不调用函数，也不主动让出。

现代 Go 的路径更像这样：

1. tight loop 所在的 G 在某个 M 上运行，M 持有唯一的 P。
2. sysmon 在后台巡检，发现同一个 schedtick 持续超过约 10ms。
3. sysmon 调用 `preemptone(pp)`，设置 `gp.preempt` 和 `stackguard0`。
4. 同步抢占路径先埋好：如果之后遇到函数调用序言，就能通过栈检查进 runtime。
5. 如果支持异步抢占，runtime 向运行该 G 的 M 发送抢占信号。
6. signal handler 检查当前位置是不是 async safe point。
7. 如果位置合适，handler 把恢复后的执行流改到 `asyncPreempt`。
8. `asyncPreempt` 保存寄存器，进入调度路径，把当前 G 放回 runnable 或停到 preempted 状态。
9. M 回到调度器，唯一的 P 有机会运行 ticker goroutine、GC worker 或其他 runnable G。

这条链路里，没有任何一步叫“goroutine 自己很善良，所以主动让出 CPU”。

它靠的是 sysmon 的巡检、preempt 标记、函数序言里的同步检查、Unix-like 平台的抢占信号、编译器和 runtime 共同维护的 safe point 元信息，以及最终回到调度器的状态转换。

也没有任何一步叫“runtime 想停哪就停哪”。

信号只是入口，safe point 才是门槛。抢占请求只是请求，调度完成才算真的让出。

把这两边都看清，才不会把 Go 调度器神化，也不会低估它。

## 读源码时，别把实现细节写成语言承诺

最后补一层边界。

本文讲的是 Go 1.14+ 引入异步抢占之后，结合本机 Go 1.25.4 runtime 源码观察到的主线。像 `SIGURG`、`forcePreemptNS = 10ms`、`isAsyncSafePoint` 的具体拒绝条件、GC worker 调度细节，都属于当前实现细节。

它们很适合帮助你建立 runtime 心智模型，但不应该被写成 Go 语言规范的永久承诺。

平台也有差异。本文主要按 Unix-like 平台讲抢占信号路径。Windows、wasm、plan9 等平台实现会不一样。cgo、non-Go code、runtime 内部函数、汇编路径，也会让抢占和信号处理变得更复杂。

所以这篇真正希望你带走的，不是某个源码行号，而是一个判断框架。

下次你看到 goroutine 长时间运行，不要只问“Go 不是会调度吗？”

你应该问四个问题：

1. 这个 G 是否处在可以被同步或异步抢占的位置？
2. sysmon 是否有机会发起抢占请求？
3. 当前位置是不是 runtime 能让 GC 看懂的 safe point？
4. 如果涉及 syscall、cgo、runtime/assembly 路径，P 和 G/M 的状态是不是已经分开？

这四个问题，比背“Go 是抢占式调度”更有用。

因为“抢占式”只是标签，safe point 才是边界；“GC 会 STW”只是现象，调度器协作才是机制。

## 这套模型怎么拿去排查问题

写 runtime 原理，如果最后不能帮你看线上现象，就容易变成漂亮但没用的知识。

抢占和 GC 这一篇，最直接的用途是帮你避免三个误判。

第一个误判：看到某个 goroutine CPU 很高，就以为它一定会把整个进程拖死。

更准确的问法是：它是不是长时间占住了 P？sysmon 有没有机会发起抢占？它是不是在 Go user code 里，还是跑进了 cgo、syscall、runtime 或 assembly 路径？如果是在普通 Go 代码里的 tight loop，Go 1.14+ 的异步抢占通常能缓解“永远不让路”的问题；如果是在 runtime 无法安全抢占的位置，就不能指望一个“抢占式调度”标签解决一切。

第二个误判：看到 GC pause，就以为问题都在 GC 算法本身。

GC 要停世界，前提是世界能被带到 safe point。某些 goroutine 长时间跑在不容易停的位置，会影响 STW 到达时间。排查 GC 延迟时，除了看堆大小、分配速率、GOGC，也要看有没有异常长跑的 goroutine、cgo 路径、系统调用、runtime 内部热点。

第三个误判：看到 “10ms” 就以为 Go 有硬实时公平性。

没有。10ms 是当前源码里 sysmon 发起抢占请求的时间片阈值，不是调度完成时间，也不是业务延迟上限。runtime 做的是 best-effort：尽量发现长期占用，尽量在安全位置停下，尽量让 P 回到可用状态。它很强，但不是实时系统。

所以线上排查时，不要问“Go 为什么还没把它切走”。这个问法太粗。

你要问的是：runtime 有没有入口能发现它，有没有 safe point 能停下它，停下以后 GC 和调度器能不能理解它。

## GMP 系列回顾

到这里，GMP 调度器这条主线基本成形了。

第一篇，我们拆开 G、M、P：G 是任务，M 是 OS 线程，P 是执行 Go 代码必须持有的 runtime 资源。理解 P，才能理解 Go 为什么不只是“很多 goroutine 跑在线程池上”。

第二篇，我们看 syscall 和 netpoller：线程可以卡住，但 P 不能陪葬；网络 I/O 可以让 G 等 fd ready，而不是让 M 傻等数据。

第三篇，也就是这一篇，我们把问题推进到正在运行的 goroutine：如果一个 G 不主动让路，runtime 怎么把它带回调度体系；如果 GC 要扫描世界，调度器怎么让世界停在能被理解的位置。

这三篇合在一起，其实都在讲同一件事：Go 调度器最厉害的地方，不是创建 goroutine，而是把执行资格管住。

G 可以等，M 可以卡，P 要流动；G 可以跑，但不能永远霸占；GC 可以停世界，但必须停在世界能被理解的位置。

理解到这里，再看 Go 并发问题，就不会只剩一句“goroutine 很轻”。

如果你想继续看 Go runtime 系列，下一步可以顺着这条线往 GC、内存分配、channel 调度和网络 poller 的源码细节走。关注我，后面继续把这些底层机制拆成能拿去排查问题的心智模型。
