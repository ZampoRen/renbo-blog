---
title: "线上锁竞争别急着换 RWMutex：Go 同步原语背后的三笔账"
description: "Mutex、RWMutex、sync.Pool 不是性能优化按钮。它们背后真正做的是吞吐、尾延迟、公平性和 GC 压力之间的取舍。看懂这三笔账，线上 p99 抖动时才不会把优化写成新问题。"
date: 2026-07-02T17:08:53+08:00
draft: false
author: "Zampo"
tags: ["Go", "并发", "Mutex", "RWMutex", "sync.Pool", "性能优化", "源码"]
categories: ["技术实战"]
cover: "/images/go-sync-primitives-locks-pool/cover.svg"
toc: true
---

接口 p99 突然抖起来，pprof 里 mutex wait 很显眼。

会议上最容易出现的三个动作是：锁太粗，换成 `RWMutex`；分配太多，把 buffer 放进 `sync.Pool`；再不行，把临界区附近的代码拆一拆。

听起来都像优化。

但这三个动作也都可能把问题带偏。Mutex 不只是“排队拿锁”；RWMutex 不是“读多写少就快”；sync.Pool 更不是一个稳定保存对象的池子。它们背后真正做的，是吞吐、尾延迟、公平性和 GC 压力之间的交易。

![Go 同步原语三笔账](/images/go-sync-primitives-locks-pool/cover.svg)

如果你不知道这笔交易是什么，线上排查时看到的每个指标都会变成直觉题。

这篇只压一个判断：Go 的同步原语不是性能按钮，而是调度和内存可见性的契约。你不是在选“哪个更快”，你是在选“愿意把哪种代价暴露给系统”。

这个判断听起来有点绕，但落到代码里很具体：锁竞争要看临界区，读写锁要看读路径是否真能并行，Pool 要看分配和 GC 压力是否真的下降。

## Mutex 不是绝对公平，它默认先保吞吐

很多人对 Mutex 的想象，是大家排队，一个一个拿锁。

Go 的 Mutex 没这么简单。

Go 1.25.4 的源码注释把 Mutex 分成两种模式：正常模式和饥饿模式。正常模式下，等待者确实按 FIFO 排队，但被唤醒的 goroutine 并不直接拥有锁。它还要和刚来的 goroutine 一起抢。

刚来的 goroutine 已经在 CPU 上跑着，天然占便宜。被唤醒的等待者可能输，输多了就回到队头继续等。

这不是 bug，这是取舍。

正常模式吞吐更好。源码注释也写得很直：normal mode has considerably better performance。一个 goroutine 可以连续多次抢到锁，减少交接成本。

代价是尾延迟。

如果某个等待者超过约 1ms 还没拿到锁，Mutex 会尝试切到 starvation mode。饥饿模式下，解锁的 goroutine 直接把所有权交给队首等待者；新来的 goroutine 不抢锁，也不自旋，只能排队。

![Mutex 正常模式与饥饿模式](/images/go-sync-primitives-locks-pool/mutex-modes.svg)

这时公平性上来了，吞吐会下降。

所以 mutex profile 里看到 wait，第一件事不是喊“锁慢”。你要先问：临界区是不是太大？锁里面有没有 I/O、日志、JSON 编码、跨服务调用？是不是一把锁保护了多个互不相关的状态？

Mutex 本身已经在替你平衡吞吐和尾延迟。你真正要做的，是别把一个臃肿临界区丢给它，然后怪它不够快。

这里有一个很实用的排查顺序。

先看锁里面到底做了什么，而不是先看锁外面能换成什么。把 `Lock` 到 `Unlock` 之间的代码圈出来，逐行问：这行是不是必须在锁里？读数据库、写日志、格式化大对象、调用 callback、发 channel，这些动作只要混进临界区，Mutex 再怎么优化都救不了你。

然后再看锁的粒度。一个 map 一把锁很常见，但一个结构体里十几个互不相关的字段都被同一把锁罩住，就会把本来不会互相影响的路径绑在一起。线上看到的 mutex wait，可能不是某个热点函数太慢，而是无关业务被同一把锁误伤。

最后才考虑换原语。因为换成 RWMutex 只能改变排队规则，不能替你缩短临界区。

## RWMutex 买的是读并行，不是免费性能

`RWMutex` 最容易被一句话害惨：读多写少就用它。

这句话不算错，但太粗了。粗到会误导工程判断。

RWMutex 真正买到的是多个 reader 可以同时进临界区。如果你的读路径真的只读内存状态、临界区足够长、写比例很低，它可能很合适。

可如果读路径里面还有全局缓存、日志、I/O、单点资源，或者临界区短到只读一个字段，RWMutex 的协调成本就未必划算。

更关键的是，它不是让 writer 默默排到天荒地老。

Go 官方文档明确说：只要有 goroutine 调用 `Lock` 等待，并且锁已经被 reader 持有，后续并发的 `RLock` 会阻塞，直到 writer 获得并释放锁。源码里对应的动作，是 writer 把 `readerCount` 减去一个很大的 `rwmutexMaxReaders`，宣告“有 writer pending”。后来 reader 一进来看到计数为负，就会停在 `readerSem`。

![RWMutex 写者等待时会挡住新 reader](/images/go-sync-primitives-locks-pool/rwmutex-writer.svg)

这就是写者优先。

它解决的是另一种病：reader 源源不断进来，writer 永远插不上队。但代价也很直接：一旦 writer pending，新来的 reader 会被挡住。你以为自己只是给 map 包了一层读写锁，结果在写入尖峰时，读请求也开始排队。

研究底稿里有一组简化 benchmark，环境是 Go 1.25.4、darwin/arm64、Apple M1 Pro。数据只能说明观察方向，不能外推成普适结论：

| Benchmark | 平均 ns/op | B/op | allocs/op | 能说明什么 |
|---|---:|---:|---:|---|
| MutexReadOnly | 103.64 | 0 | 0 | Mutex 会把读者串行化 |
| RWMutexReadOnly | 80.04 | 0 | 0 | 读锁可以并行，但 microbench 波动要谨慎看 |
| MutexWriteEvery100 | 118.40 | 0 | 0 | 读写统一串行 |
| RWMutexWriteEvery100 | 48.31 | 0 | 0 | 写少、读能并行时可能收益明显 |

这里最值得带走的不是“RWMutex 比 Mutex 快多少”。

真正该带走的是：换 RWMutex 前，先确认读路径能不能真的并行，writer pending 时读延迟能不能接受，再用自己的临界区测一次。

否则你只是把一把简单的锁，换成了一套更复杂的排队规则。

还有一种场景尤其容易误判：配置、路由表、规则表这类“读很多、偶尔整体更新”的数据。

直觉上它们适合 RWMutex。但如果更新方式是构造一份新快照，再用 `atomic.Value` 或指针替换，读路径甚至可以不拿锁。此时你要比较的就不是 Mutex 和 RWMutex，而是“共享可变状态”与“不可变快照”。

Go 里很多并发问题的更优解，不是更高级的锁，而是减少共享。锁是最后的边界，不是第一个设计动作。

## sync.Pool 不是对象池，是 GC 可以清掉的临时缓冲层

`sync.Pool` 的误用，通常来自它的名字。

很多人看到 Pool，就自然把它理解成对象池：我 Put 进去，后面 Get 出来，减少分配，性能变好。

官方文档不是这么承诺的。

Pool 里任何对象都可能在任何时候被自动移除，不通知调用方。如果 Pool 持有的是唯一引用，对象就可能被回收。`Get` 也可以忽略 Pool，把它当成空的。

实现上，Pool 是 per-P 本地缓存加 victim cache。`Get` 会先找当前 P 的 private，再找 shared，再偷其他 P，最后看上一轮 GC 留下来的 victim。GC 开始 STW 时，`poolCleanup` 会丢掉更旧的 victim，把当前 primary cache 移到 victim，然后清空 primary。

![sync.Pool 会在 GC 周期里轮转缓存](/images/go-sync-primitives-locks-pool/pool-gc.svg)

这套设计很聪明：高并发下减少共享竞争，GC 来时又不会让 Pool 无限持有对象。

但也正因为这样，Pool 不是对象生命周期管理工具。你用它买的是 GC 压力下降，不是“对象一定还在”。

研究底稿里的 Pool benchmark 也很适合说明边界：

| Benchmark | 平均 ns/op | B/op | allocs/op | 能说明什么 |
|---|---:|---:|---:|---|
| PoolSmallObject | 8.45 | 0 | 0 | 16B 小对象收益很小 |
| AllocateSmallObject | 11.05 | 16 | 1 | Go 小对象分配本身很快 |
| PoolBuffer4K | 66.25 | 0 | 0 | 高频 4K buffer 复用能明显降分配 |
| AllocateBuffer4K | 666.67 | 4096 | 1 | 每次分配 4K 会带来 GC 压力 |

不要把这张表读成“Pool 快 10 倍”。它只说明一件事：对象足够大、足够高频、生命周期短、可以安全 Reset 时，Pool 更可能有价值。

反过来，小对象、低频对象、大小差异巨大的对象，Pool 可能只是复杂度。

Go issue #23199 里有一个很典型的反例：把可增长的 `bytes.Buffer` 放进 Pool，偶发大请求可能把 256MiB 级别的大 buffer 留下来，后面 1KiB 的小请求也拿到这个大 buffer。讨论里的示例提到，某些时机下可能需要几十个 GC cycle 才释放最初的大块内存。

这不是说不能 Pool buffer。

而是说：Put 回去之前，你要先决定容量上限。太大的丢掉，或者按 size bucket 分桶。否则你以为自己在减少分配，实际上是在把偶发大对象的内存成本摊给后面所有小请求。

Pool 还有一个经常被忽略的要求：Reset 必须可靠。

buffer、encoder、临时结构体放回去之前，如果状态没清干净，下一次 Get 到它的请求可能读到上一个请求留下来的数据。这个问题比“性能没有变快”更危险，因为它会变成偶发的数据串味，难复现，也难解释。

所以我不太喜欢把 Pool 当成默认优化。只有当 `benchmem` 里 `B/op`、`allocs/op` 明确难看，heap profile 也说明临时对象确实在制造 GC 压力时，再引入它。引入以后还要写清楚三件事：对象怎么 Reset，容量上限是多少，什么情况下不 Put 回去。

没有这三条，Pool 就不是优化，是未来事故的伏笔。

## Once、WaitGroup、Cond 也不是“会用 API 就够了”

这篇不展开讲所有 sync 原语，但有三个边界很值得记住。

`Once` 的难点不是“只执行一次”。如果只是一次 CAS，看起来就能做。但源码注释明确说这种实现是错的：两个 goroutine 同时调用时，CAS 输的人可能在赢家的 `f` 还没执行完就返回。`Once.Do` 要保证的是：任何一次 Do 返回时，那个 `f` 已经执行完成。

所以它需要慢路径加 Mutex。

另外，如果 `f` panic，Once 认为它已经返回，后续不会再调用。需要失败重试的初始化，不要直接交给 Once。

`WaitGroup` 等的是完成，不负责错误传播、取消和并发限流。Go 1.25 起有 `WaitGroup.Go(f)`，文档也写了 `f must not panic`。如果你要拿错误、要超时、要取消，通常要配合 context、channel，或者用 errgroup 这一类工具。

`Cond` 则更像复杂共享状态下的条件变量。`Wait` 醒来以后，不代表条件一定成立，必须放在 for 循环里重新检查。简单的一次性通知，channel 往往更清楚。

这些边界背后还是同一个问题：同步原语给的是契约，不是业务语义。取消、错误、重试、限流、生命周期，都要你自己补上。

## 选同步原语前，先问这张 checklist

线上排查时，我会把问题拆成三笔账。

第一笔，锁竞争账。

- 能不能不共享可变状态？不可变快照、owner goroutine、channel，有时比换锁更干净。
- 临界区能不能缩小？先把 I/O、日志、编码、外部调用挪出去。
- 锁保护的是一个状态，还是顺手保护了一堆不相关的东西？
- mutex profile 里看到 wait 后，有没有结合 goroutine profile、trace、业务路径一起看？

第二笔，读写协调账。

- 读路径是否真的能并行？还是里面藏着同一个全局资源？
- writer pending 后，新 reader 排队是否能接受？
- 写入尖峰时，读请求的 p99 有没有被拖上去？
- 有没有用自己的临界区做 `go test -bench -benchmem`，而不是用“读多写少”四个字拍板？

第三笔，GC 压力账。

- 对象是否足够大、足够高频、生命周期短？
- Put 回 Pool 前是否 Reset 干净？有没有脏数据泄露风险？
- 对象大小是否稳定？大对象是否要丢弃或分桶？
- 观察的是 `allocs/op`、`B/op`、heap profile、GC pause，还是只盯着 `ns/op`？

验证也别太复杂，先从最朴素的两步开始：用 `go test -bench . -benchmem` 看自己的代码路径，再用 pprof 看 mutex、heap 和 goroutine。benchmark 只负责回答“这个假设在我的临界区里是否成立”，pprof 负责回答“线上真正消耗在哪里”。

少了这一步，任何同步原语选型都容易变成代码审美。

真正可靠的结论，应该长这样：在这段临界区、这个并发度、这组读写比例下，RWMutex 让 p99 下降了多少，是否引入新的 reader 排队；Pool 让 `B/op` 降了多少，heap 是否真的少涨。说不出这些，最好先别急着动。

如果你只想记一句话，就记这个：Mutex 处理的是互斥，RWMutex 处理的是读写协调，sync.Pool 处理的是临时对象复用。它们都可能提升性能，但前提是你知道自己把哪种代价交给了它。

下次 p99 抖起来，pprof 里 mutex wait 很刺眼，别急着把 Mutex 全换成 RWMutex，也别急着往代码里塞 Pool。

先把这三笔账算清楚。

很多线上问题，不是因为 Go 的同步原语不够快，而是我们把一个工程判断，偷懒写成了 API 替换。
