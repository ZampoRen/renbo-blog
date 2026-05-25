---
title: "为了写得“更 Go”，他把一个 cache 写成了小型 RPC"
description: "Go channel 的价值不是替代 mutex，也不是把锁藏起来，而是把 goroutine 之间的通信关系写清楚。一个简单 cache 被 channel 化，往往不是更 Go，而是把一把锁绕成了一套协议。"
date: 2026-05-25T09:53:00+08:00
draft: false
author: "任博"
tags: ["Go", "channel", "CSP", "并发编程", "mutex"]
categories: ["技术实战"]
cover: "/images/go-channel-csp-overuse/cover.png"
toc: true
---

有些 Go 代码，一看就知道作者很努力。

为了让一个普通 cache 看起来“更 Go”，他把 `sync.RWMutex` 拿掉了，换成一个专门的 goroutine 来“拥有”这份状态。每次 `Get`，调用方先构造 request，发进 channel，再等另一个 channel 把结果送回来。`Set` 也是一样。再往后，还要处理取消、退出顺序、后台 goroutine 什么时候停。

最后代码里确实看不见那把锁了。

但你多了一套协议。

本来 `map + RWMutex` 三十行能讲清楚的事，被写成了一个本地小型 RPC：请求结构、响应通道、owner goroutine、生命周期管理，一个都不少。

这不是优雅。

这是绕远。

Go 社区有一句被反复引用的话：

> Do not communicate by sharing memory; instead, share memory by communicating.

很多人把它念成了另一句话：Go 不喜欢锁，channel 才高级，能用 channel 就别用 mutex。

这个理解太粗，也很容易害人。

channel 的价值不是藏锁，而是写关系。mutex 也不是“不 Go”，它就是保护共享状态的直接工具。真正要问的不是“哪个更有 Go 味”，而是：你现在到底是在传东西，还是在护东西？

## channel 首先讲的是结构，不是性能

Rob Pike 在《Concurrency is not Parallelism》里一直在压一个区分：concurrency 是 structure，parallelism 是 execution。

并发不是“同时跑得更快”的同义词。

并发先是一种组织程序的方式：把一件事拆成几个可以独立推进的单元，再让它们通过通信协调。

这也是 Go 把 goroutine 和 channel 做成语言级能力的原因。goroutine 表达一个独立执行的工作单元，channel 表达这些工作单元之间的数据、结果、信号和同步。

所以 channel 最重要的地方，不是它内部有没有队列，也不是它能不能替代某个锁。它真正让代码变清楚的地方，是把“谁等谁、谁交给谁、谁处理完往哪里走”写到了代码表面。

比如一个很普通的任务分发：

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

这段代码里，`jobs` 不是为了显得高级。

它在表达一件具体的事：任务从生产者流向 worker。worker 拿到任务，处理完，把结果交回 `results`。

关系很清楚。

如果你用共享 slice、锁和条件变量也能写出类似效果，但你得自己维护更多规则：哪个 worker 能拿任务，什么时候等待，什么时候继续，什么时候不该再抢。

channel 在这种场景里的优势，不是“底层一定更快”。它的优势是：并发关系更像代码本身，而不是藏在注释和脑子里。

这也是很多 channel 滥用的根源。

一旦你只把 channel 看成“更 Go 的同步工具”，你就会忍不住把所有并发问题都往 channel 上套。cache 要套，计数器要套，状态表也要套。结果代码表面上没有锁，实际却长出了一堆隐形规则。

你不是少写了一把锁。

你是多造了一套协议。

## “通过通信共享内存”不是禁止共享

“Do not communicate by sharing memory; instead, share memory by communicating.” 这句话很漂亮，也很容易被误读。

它不是在说：Go 禁止共享内存。

Go 没有这么做。Go 有指针，有共享变量，有 `sync.Mutex`，有 `sync.RWMutex`，标准库里也明明白白提供这些工具。Rob Pike 在《Go at Google》里也说过，共享是合法的，通过 channel 传递指针是惯用且高效的。

重点不是“不共享”。

重点是“同一刻谁有权处理它”。

Go Blog 对这句话的解释更具体：不要用显式锁来协调一堆 goroutine 同时访问同一份数据，而是鼓励通过 channel 在 goroutine 之间传递数据引用，让同一时刻只有一个 goroutine 访问那份数据。

这更像一种访问权纪律。

你可以传指针，可以共享对象，也可以让数据从一个阶段交到另一个阶段。关键是结构上要说得清楚：现在是谁处理它，处理完交给谁，谁不应该再碰它。

这和“所有共享状态都必须被 channel 包起来”不是一回事。

如果数据本来就在不同 goroutine 之间流动，channel 很自然。比如任务、事件、结果、错误、取消信号。这些东西本来就有方向，本来就需要交接。

但如果问题本质只是保护一份长期存在的状态，mutex 往往更直接。

看一个普通 cache：

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

它的意思非常直：`m` 是共享状态，读写它要进临界区。读的时候拿读锁，写的时候拿写锁。规则短，边界清楚，读的人不需要猜。

你当然可以把它改成 owner goroutine：所有读写都发消息给它，它串行处理请求，再把结果回给调用者。

大概会长成这样：

```go
type getReq struct {
    key   string
    reply chan getResp
}

type getResp struct {
    value int
    ok    bool
}

func (c *Cache) Get(key string) (int, bool) {
    reply := make(chan getResp)
    c.gets <- getReq{key: key, reply: reply}
    resp := <-reply
    return resp.value, resp.ok
}
```

这段代码不是错。

如果这个 cache 背后其实是一台小状态机，所有变化都必须按事件顺序发生，或者它还要和别的异步流程协作，这种 owner 模型可能很合适。

问题是，很多时候它只是一个普通 cache。

这时你为了绕开一把锁，引入了新的概念：`getReq`、`getResp`、`reply`、后台处理循环、调用方等待、owner goroutine 的存活边界。读者看代码时，不再是理解“这份 map 要加锁”，而是要先理解“这套本地协议怎么工作”。

这就是我说的小型 RPC。

它没有网络，但有请求；没有服务发现，但有固定的处理者；没有 HTTP 状态码，但有 response；没有跨进程调用，却已经有了一次消息往返。

有些场景这么做是合理的。比如状态变化本身就是一条事件流，或者你想把状态机收敛到一个地方，让所有变化按顺序发生。

但如果只是普通 cache，这套写法通常会引入更多东西：请求结构、响应通道、后台 goroutine、取消处理、收尾规则。

如果一把锁能说清楚，别急着写一套协议。

Go 的味道不是不用锁。

Go 的味道是问题边界清楚。

## 你是在传东西，还是在护东西？

channel 和 mutex 不是谁替代谁。

它们表达的是两类不同问题。

mutex 保护一块状态。

channel 描述一段关系。

这个判断很朴素，但能挡掉很多过度设计。

当你写的是任务分发、结果回流、阶段流水、异步通知，这些东西本身就在 goroutine 之间移动，channel 通常更自然。它让代码直接呈现流向：谁生产，谁消费，谁把结果送回来。

当你写的是 cache、map、计数器、连接表、配置快照、结构体内部不变量，这些东西通常是长期存在的共享状态。多个 goroutine 来读写它，最直接的问题就是“怎么保护这份状态不被同时改坏”。这时候 mutex 很可能更诚实。

Go Wiki 里有一句比很多风格争论都可靠的话：

> Use whichever is most expressive and/or most simple.

用最能表达问题、也最简单的那个。

这句话听起来不酷，但它很工程。

很多人写 channel 误用，不是因为不知道 mutex，而是觉得 mutex 太朴素了，好像不够体现语言特性。于是一个简单状态访问，被包装成消息系统。代码看上去“并发模型”很完整，实际可读性却下降了。

你要多问自己两句：

第一，数据是在流动，还是只是被保护？

第二，channel 版本是不是引入了更多生命周期规则？

如果一个 channel 方案需要额外 goroutine、请求/响应结构、复杂的退出管理，还要让每个调用方理解这套协议，那它就不再是简化。它只是把复杂度从锁转移到了通信协议上。

反过来也一样。

如果你的 mutex 方案已经出现多个锁交错、持锁顺序难解释、状态变化散落在各处，那也该停下来想想：是不是应该把状态收拢到一个更明确的所有者，让外部通过消息和事件来驱动它？

工具没有信仰，只有边界。

channel 适合表达流动，mutex 适合守住边界。

## 别把 Go 写成姿势表演

我见过不少代码，问题不是不会用 channel，而是太想证明自己会用 channel。

一旦写代码变成姿势表演，就很容易把简单问题复杂化。

cache 本来只需要保护 map，你却给它设计了 request type。读一个值，本来是一次函数调用，你却让它变成一次消息往返。代码里少了 `Lock()`，但多了调用路径、多了阻塞点，也多了出错面。

这类代码最麻烦的地方，不是慢一点。

而是它会让后来的人不知道该从哪里理解它。读者必须先搞懂你自己发明的协议，才能明白你只是想读写一个 map。

channel 当然重要。它是 Go 并发模型里很有辨识度的一部分。没有 channel，Go 写 pipeline、worker、异步结果和取消传播都会少掉很多表达力。

但 channel 不是“高级锁”。

它不是为了让你把所有共享状态都藏到 goroutine 后面。它真正有价值的地方，是当程序里确实存在通信、协作、交接和流动时，它能把这些关系写出来。

所以回到开头那个 cache。

如果你真正需要的是一个状态机，一个事件处理器，一个顺序处理的 owner，那用 channel 没问题。那不是为了“更 Go”，而是因为你的问题本来就是消息和状态转移。

如果你只是要保护一个 map，那就把锁写出来。

朴素一点没关系。

代码不是写给语言社区投票的，是写给下一个维护它的人看的。

这篇只讲到 channel 的第一层：它首先是一种结构表达，不是性能魔法，也不是 mutex 的替代品。

那到底什么时候该用 channel，什么时候该用 mutex？下一篇给你一套可以直接拿走的五步判断流程，不用再纠结"哪个更 Go"。

如果你觉得有收获，点个关注，别错过下篇的判断框架和实战案例。

下一篇再往下走一点：当 channel 真正进入工程代码，`select`、`close` 和缓冲容量这些细节，才是最容易把人绊倒的地方。很多 goroutine 泄漏、假死、偶发卡住，不是因为你不会写 channel，而是因为你没把通信协议收尾。