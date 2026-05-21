---
title: "Go channel 不是高级锁，它是一种组织并发的语言"
description: "channel 的首要价值不是性能魔法，而是结构：用通信把 goroutine 组织起来。本文从 CSP、share memory by communicating、channel vs mutex 判断框架讲清 Go 并发代码该怎么选工具。"
date: 2026-05-21T16:02:00+08:00
draft: false
author: "任博"
tags: ["Go", "channel", "CSP", "并发编程", "mutex"]
categories: ["技术实战"]
cover: "/images/go-channel-csp-design-philosophy/cover.png"
toc: true
---

有人为了写得“更 Go”，把一个简单 cache 包成了 channel 协议。

每次 `Get`，先构造 request，发给 owner goroutine，再等 response channel 返回。还要处理 context、close、退出顺序、goroutine 泄漏。

最后代码确实没有显式 `sync.RWMutex` 了。

但问题也来了：本来一把锁能讲清楚的事，被写成了一套小型 RPC。

这不是优雅。

这是绕远。

![Go channel 设计哲学封面](/images/go-channel-csp-design-philosophy/cover.png)

Go 社区有一句被反复引用的话：

> Do not communicate by sharing memory; instead, share memory by communicating.

很多人把它理解成：Go 不喜欢锁，channel 更高级，能用 channel 就别用 mutex。

这个理解太粗了。

channel 的价值不是把锁藏起来，而是把通信显式写出来。mutex 也不是“不 Go”，它就是保护共享状态的直接工具。

真正要问的不是：channel 和 mutex 谁更高级？

真正要问的是：你现在到底是在传东西，还是在护东西？

## channel 首先讲的是结构，不是性能

Rob Pike 在那场很有名的演讲《Concurrency is not Parallelism》里，一直在压一个区分：concurrency 是 structure，parallelism 是 execution。

并发不是“同时跑得更快”的同义词。

并发先是一种组织程序的方式：把一件事拆成多个可以独立执行的单元，再让它们通过通信协调。

![Rob Pike](/images/go-channel-csp-design-philosophy/rob-pike.jpg)

这也是为什么 Go 把 goroutine、channel、select 做成语言级能力。

- goroutine 表达独立执行的工作单元。
- channel 表达工作单元之间的数据、结果、信号和同步。
- select 表达多个通信机会之间的选择。

这套东西受 Hoare 1978 年提出的 CSP（Communicating Sequential Processes）影响。Effective Go 也明确说，Go 的并发方法源自 Hoare 的 CSP，同时也可以看作 Unix pipes 的类型安全泛化。

但这里要压住边界：Go 不是把 Hoare CSP 原封不动搬进语言。

Rob Pike 在《Go at Google》里说得更准确：Go 体现的是一种带 first-class channels 的 CSP 变体。

变体两个字很重要。

Go 仍然有指针，有共享内存，有 `sync.Mutex`，有 `sync.RWMutex`，有 `sync.WaitGroup`，也有 runtime 里的线程和调度器。它不是一个纯理论模型，而是一门工程语言。

所以 channel 的主线不是“替代所有锁”。

它的主线是：当并发程序里有任务流、结果流、取消信号、所有权交接时，channel 能把这些关系写到代码表面。

![CSP 在 Go 中的工程化表达](/images/go-channel-csp-design-philosophy/inline-01.png)

这也是第一层误区的根：很多人把 channel 看成“线程安全队列”。

有缓冲 channel 确实有队列语义。你 `make(chan T, n)`，它内部会有缓冲区。发送方可以在缓冲区没满时放进去，接收方可以在缓冲区有值时取出来。

但如果只把 channel 理解成队列，就少看了它最重要的部分：阻塞、唤醒、close、select、取消、所有权流动。

channel 不只是存放数据。

它还在表达：谁等谁，谁把东西交给谁，谁能继续往下走，什么时候整个流程结束。

## “通过通信共享内存”不是禁止共享内存

“Do not communicate by sharing memory; instead, share memory by communicating.” 这句话很容易被念成口号。

一旦变成口号，它就会被滥用。

Go Blog 对这句话的解释更具体：不要用显式锁去协调共享数据访问，而是鼓励通过 channel 在 goroutine 之间传递数据引用，让同一时刻只有一个 goroutine 访问那份数据。

重点不是“不传指针”。

重点是“谁现在有权访问”。

Rob Pike 在《Go at Google》里也说过，共享是合法的，通过 channel 传递指针是惯用且高效的。

所以这句话真正像一种纪律：你可以共享，但不要让所有 goroutine 都理直气壮地同时碰同一份状态。你最好能在结构上说清楚，某一刻是谁拥有它，谁处理它，处理完交给谁。

比如一个 pipeline：

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}
```

`gen` 不是在展示“我不用锁”。

它是在告诉你：我负责生产这些值；我生产完会关闭 `out`；下游只需要 `range` 这个 channel，直到它结束。

这就是通信关系。

再看 worker pool：

```go
jobs := make(chan Job)
results := make(chan Result)

for i := 0; i < workerCount; i++ {
    go func() {
        for job := range jobs {
            results <- process(job)
        }
    }()
}
```

这段代码里，`jobs` 表达任务分发，`results` 表达结果回流。worker 从哪里拿活、处理完往哪里交，关系是明的。

如果你把这段用一堆共享 slice、锁、条件变量来写，也能写。

但你得自己维护更多规则：哪个 worker 能拿哪个任务，什么时候等待，什么时候唤醒，什么时候退出。

channel 在这里的优势不是“底层一定更快”。

它的优势是：并发关系更像代码本身，而不是藏在注释和脑子里。

## mutex 保护一块状态，channel 描述一段关系

Go Wiki 里有一个很朴素的建议：

> Use whichever is most expressive and/or most simple.

这句话比很多“Go 风格”争论都可靠。

Go Wiki 还提醒，新手常见错误是因为 channel 和 goroutine 好玩，就过度使用它们。不要害怕在问题适合时使用 `sync.Mutex`。

这才是 Go 的务实。

如果问题本质是保护 cache、map、计数器、结构体不变量，mutex 往往更直接。

```go
type Cache struct {
    mu sync.RWMutex
    m  map[string]int
}

func (c *Cache) Get(key string) (int, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.m[key]
    return v, ok
}

func (c *Cache) Set(key string, value int) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.m[key] = value
}
```

这段代码没有什么不 Go。

它的意思很清楚：`m` 是共享状态，读写它要进临界区。读的时候拿读锁，写的时候拿写锁。

你当然可以用 owner goroutine 加 channel 来管理这个 cache：所有读写都发消息给它，它串行处理请求，再把结果回给调用者。

有些场景这么做是合理的，比如状态变化本身就是一条事件流，或者你想把状态机收敛到一个 goroutine 里。

但如果只是普通 cache，这套写法通常会引入更多东西：请求结构、响应 channel、owner goroutine 生命周期、context 取消、close 规则。

如果一把锁能说清楚，别急着写一套协议。

![channel vs mutex 判断图](/images/go-channel-csp-design-philosophy/inline-02.png)

这里可以用一个很实用的判断：

- 任务、数据、事件、结果在 goroutine 之间流动：优先想 channel。
- cache、map、计数器、长期存在的共享状态：优先想 mutex / RWMutex。
- 锁规则复杂到很难解释：考虑能不能改成 owner goroutine + channel。
- channel 协议复杂到满屏 request / response / close / context：回头看看 mutex 是不是更诚实。

mutex 保护一块状态。

channel 描述一段关系。

这句话基本能挡掉一半的误用。

## select default 不是补丁，而是通信语义的一部分

Go 没有给 channel 做 `TrySend` / `TryRecv` 方法。

不是因为这个能力不重要，而是因为 Go 把“等哪个通信、等不等、同时等几个”统一放到了 `select` 里。

非阻塞 receive 是这样：

```go
select {
case v := <-ch:
    use(v)
case <-ctx.Done():
    return ctx.Err()
default:
    // nothing ready; do something else
}
```

非阻塞 send 也一样：

```go
select {
case ch <- v:
    // sent
default:
    // would block
}
```

`default` 的存在提醒你：channel 不是一个普通容器。它有“现在能不能通信”的状态。

如果没有 `default`，当前 goroutine 可以等待。

如果有 `default`，它可以在通信还没准备好时走开。

如果同时监听 `ctx.Done()`，它还可以把取消纳入同一个等待点。

这就是 select 的价值：不是让你少写几个 try 方法，而是把并发等待写在一个地方。

不过 select 也有边界。

Go spec 规定，当多个 case 都 ready 时，会通过 uniform pseudo-random selection 选择一个可进行的通信。这意味着 select 没有源码顺序优先级。

但这不等于强公平。

不要把 select 当成公平调度器，也不要假设长期比例一定精确。语言保证的是选择语义，不是你的业务公平性。

## close 是发送端协议，不是接收端按钮

channel 另一个常见误用，是把 `close` 当成“我不想收了”的按钮。

Go spec 对 close 的定义很明确：关闭 channel 表示不会再发送值。

这句话决定了 close 的归属。

通常只有发送方、生产者、或者拥有发送权的一方，才知道“不会再发送”这个事实。

接收方如果擅自 close，一个还在发送的 goroutine 可能立刻 panic：send on closed channel。

所以更稳的表达是：

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}
```

生产者负责关闭 `out`。

接收者通过 `for n := range gen(...)` 消费，直到 channel 关闭。

如果是多发送者场景，事情就更不能随手 close。你需要一个协调者确认所有 sender 都结束，再关闭结果 channel；或者用 `context.Context` / done channel 传播取消。

这也是 channel 写法的代价：它让通信关系清楚，但你也必须把生命周期协议写清楚。

buffer 也救不了生命周期问题。

buffer 是缓冲，不是取消。容量是背压的一部分，不是收尾协议。

Go Blog 的 pipeline 文章里提醒过：如果某个阶段没有消费完 inbound values，尝试发送的 goroutine 可能会一直阻塞。goroutine 不是垃圾，没人用就会自己消失；它必须自己退出。

这句话在真实项目里很值钱。

很多 channel 泄漏不是因为 channel 难，而是因为只设计了“正常传完”，没设计“下游提前走了”。

## 三种常见 channel 滥用

第一种，是用 channel 包简单状态。

比如一个计数器，本来只是 `n++` 需要保护。用 `sync.Mutex`，读者一眼就知道临界区在哪里。你非要启动一个 goroutine，让所有增减操作都发消息进去，代码看起来更“并发”，但问题并没有变清楚。

Effective Go 自己也提醒过，这套方法可能走得太远。引用计数这种东西，可能最好就是在整数变量外面放一把 mutex。

第二种，是用 channel 逃避所有权设计。

有些代码把指针通过 channel 到处传，但没有规定“传过去以后，原来的 goroutine 还碰不碰”。这时 channel 只是换了一种共享方式，data race 仍然会发生。

真正的“share memory by communicating”不是把指针塞进 channel 就算完。它要求你在协议上说清楚：谁拿到，谁处理；处理期间，别人不要同时改；处理完，是否交回去。

第三种，是把 buffer 当成取消机制。

很多人发现 goroutine 卡住，就把 `make(chan T)` 改成 `make(chan T, 100)`。短期看，阻塞消失了。长期看，只是把问题往后推了一百格。

buffer 可以吸收突发流量，可以表达容量和背压，但它不回答这些问题：下游退出了吗？上游还会不会发？谁来 close？错误怎么传？

如果这些问题没答案，buffer 再大也只是临时仓库。

所以 channel 不是不能用。

恰恰相反，channel 很适合那些本来就有流动关系的问题。只是它一旦出现，你就要认真对待协议：发送、接收、关闭、取消、退出，这些都不是附属品。

## 一个可以直接拿走的判断流程

下次再纠结 channel 还是 mutex，不要先争风格。

按这五个问题问：

第一，你是在传东西，还是在护东西？

传任务、数据、事件、结果、错误、取消信号，优先想 channel。保护 map、cache、计数器、连接表、共享配置，优先想 mutex。

第二，数据有没有明确的拥有者？

如果数据沿流程移动，希望同一时间只有一个 goroutine 处理，channel 很合适。如果数据长期存在，多处都要读写，mutex 更自然。

第三，用 mutex 后规则会不会变复杂？

如果代码里开始出现多个锁、锁顺序、跨函数持锁、容易死锁的规则，说明问题可能不只是“保护状态”。这时可以考虑把状态交给一个 owner goroutine，用消息表达访问。

第四，用 channel 后协议会不会变复杂？

如果为了读一个 cache，要设计 request、response、close、context、owner goroutine 退出，那可能已经过度设计了。

第五，下游会不会提前退出？

只要用了 channel，尤其是 pipeline / worker pool，就要问：下游提前退出时，上游 send 会不会永久阻塞？谁负责 close？是否需要 context？

这些问题比“哪个更 Go”有用得多。

Go 风格不是把所有并发都写成 channel。

Go 风格是让代码里的并发关系尽量可读、可解释、可收尾。

有时答案是 channel。

有时答案就是一把锁。

## 最后，别把工具当信仰

channel 是 Go 很漂亮的一部分。

它把 CSP 的思想落进了日常工程：让 goroutine 独立执行，让通信成为结构，让数据和信号沿着清楚的路径流动。

但它不是高级锁。

也不是免死金牌。

用 channel 仍然可能 data race，只要你通过 channel 传了指针，然后多个 goroutine 又同时改它。用 channel 仍然可能泄漏，只要你没设计取消和退出。用 channel 仍然可能变复杂，只要你把简单状态保护写成复杂协议。

反过来，mutex 也没什么丢人。

它保护短临界区、共享 map、cache、计数器，直接、清楚、成本低。

判断工具前，先判断问题。

你是在传东西，还是在护东西？

如果是在传任务、传结果、传信号、交接访问权，让 channel 把这段关系写出来。

如果只是在保护一块共享状态，老老实实用 mutex。

好的 Go 并发代码，不是看起来用了多少 channel。

是别人接手时，一眼能看出：谁在工作，谁在等待，谁拥有数据，谁负责结束。
