---
title: "channel 最容易翻车的不是发送接收，而是 close"
description: "Go channel 写到真实工程里，最容易出问题的往往不是 send 和 receive，而是 close、select、buffer 和生命周期。close 是发送端协议，buffer 不是取消机制，channel 一旦出现，就要设计谁创建、谁发送、谁关闭、谁退出。"
date: 2026-05-25T10:28:00+08:00
draft: false
author: "任博"
tags: ["Go", "channel", "select", "并发编程", "goroutine"]
categories: ["技术实战"]
cover: "/images/go-channel-close-lifecycle/cover.png"
toc: true
---

上一篇讲到最后，其实只把问题说完了一半。

如果你面对的是任务、结果、信号和访问权交接，channel 确实比 mutex 更顺手。它能把 goroutine 之间的关系写出来，让代码不只是在“保护状态”，而是在表达一段通信。

但很多 channel 代码真正翻车的地方，不是发送，也不是接收。

是 close。

线上日志里突然蹦出一行：

```text
panic: send on closed channel
```

通常很难看。因为出问题的那一行，往往只是一个普通到不能再普通的发送：

```go
results <- r
```

真正的问题藏在更早的地方：有人提前把 `results` 关了。

一个 goroutine 觉得“我不想收了”，顺手 `close(results)`；另一个 worker 正好处理完任务，准备把结果发回来。于是代码不是优雅退出，而是当场炸掉。

这就是 channel 最容易被低估的地方。

你会写 `ch <- v`，也会写 `<-ch`，不代表你已经设计好了这条通信关系。

channel 最难的不是传值，是收尾。

![channel close 生命周期封面](/images/go-channel-close-lifecycle/cover.png)

## select 解决的是等待，不是公平

先看 `select`。

Go 没有给 channel 单独设计 `TrySend`、`TryRecv` 方法。它把“等哪个 channel、等不等、同时等几个”这件事，统一放进了 `select`。

非阻塞接收通常长这样：

```go
select {
case v := <-ch:
    use(v)
case <-ctx.Done():
    return ctx.Err()
default:
    // 现在没有值，先做别的
}
```

非阻塞发送也一样：

```go
select {
case ch <- v:
    // 发出去了
case <-ctx.Done():
    return ctx.Err()
default:
    // 现在发不了，别卡住
}
```

`default` 不是随手加的补丁。它改变的是这次通信的态度：没有 ready 的 case 时，要不要等。

没有 `default`，当前 goroutine 可以停下来，等某个通信条件成熟。

有 `default`，它就不等。它会立刻走默认分支，把“现在发不了 / 现在收不到”当成一个正常结果处理。

这在很多场景里很有用。比如采样、尝试投递、避免日志通道把主流程拖死。

但也很容易被滥用。

有人给 `select` 加了 `default`，以为自己“避免阻塞”了。结果真实效果是：消息偶尔发不出去，就被悄悄丢掉；取消信号还没来得及被观察，循环已经飞快空转；CPU 被一个看似安全的非阻塞分支烧掉。

`select` 还有一个常被忽略的点：多个 case 同时 ready 时，Go spec 说会从可执行通信里做一次均匀伪随机选择。

这意味着它没有源码顺序优先级。

写在上面的 case，不会因为“看起来更重要”就一定先执行。`select` 解决的是“我现在等哪些通信机会”，不是替你的业务做公平调度。

如果你需要优先级，要自己设计。

不要把 `select` 当成一个聪明的调度器。它只是一个等待点。

## close 不是刹车，是完工声明

再看 `close`。

Go spec 对 `close(ch)` 的定义很直：它记录的是“不会再有值发送到这个 channel”。

这句话决定了 close 的工程归属。

谁最知道“不会再发送”？通常是发送方，是生产者，是拥有发送权的一方。

接收方知道的是“我不想收了”。

这两件事不是一回事。

接收方不想收，应该表达取消、返回、超时、done 信号，或者让上游停止生产。它不应该随手把数据 channel 关掉，因为它并不知道还有没有别人正在发。

尤其是多发送者场景。

```go
results := make(chan Result)

for i := 0; i < workerCount; i++ {
    go func() {
        for job := range jobs {
            results <- process(job)
        }
    }()
}
```

这时候谁能 `close(results)`？

不是某个 worker。它只知道自己结束了，不知道别的 worker 是否还在发送。

也不是随便一个接收者。它只知道自己不想继续读，不知道生产端是否已经全部停下。

更稳的做法，是引入协调者：等所有发送者结束，再关闭结果 channel。

```go
var wg sync.WaitGroup
results := make(chan Result)

for i := 0; i < workerCount; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for job := range jobs {
            results <- process(job)
        }
    }()
}

go func() {
    wg.Wait()
    close(results)
}()
```

这段代码真正重要的不是 `WaitGroup` 本身，而是它把关闭权放回了正确位置：所有发送都结束之后，才宣布没有结果了。

close 不是“让别人别发”的按钮。

close 是“我已经不会再发”的声明。

这两个意思差一点，线上就可能差一个 panic。

还有一个细节也值得记住：channel 关闭之后，接收方还能把已发送但尚未接收的值读完。读完以后，再接收会立刻得到元素类型的零值。多值接收里的 `ok`，就是用来区分“正常收到零值”和“channel 已经关闭”。

```go
v, ok := <-ch
if !ok {
    return
}
```

如果你的元素类型本身就可能是零值，比如 `0`、`""`、`nil`，却不看 `ok`，那你迟早会把“已经结束”误当成“收到一个正常值”。

这里还有一个很实用的判断：数据 channel 和停止信号最好不要混在一起。

很多事故都从一句“我关掉 channel 通知大家停”开始。听起来没问题，实际要看你关的是哪一个 channel。

如果这是 `jobs`，而且只有生产者往里面发，生产者完成后关闭它，worker 用 `for job := range jobs` 退出，这很自然。

```go
go func() {
    defer close(jobs)
    for _, job := range batch {
        jobs <- job
    }
}()
```

但如果这是 `results`，而且多个 worker 都会往里面发，那接收方就不能因为自己不想收了而关闭它。它一关，仍在工作的 worker 发送结果时就会撞上 panic。

接收方真正要表达的，是“我不再需要了”。这应该走另一个通道：`context`、`done`、超时，或者业务层的取消信号。

```go
select {
case results <- r:
case <-ctx.Done():
    return
}
```

这段代码的意思很清楚：结果能送出去就送；如果下游已经取消，worker 自己退出。

不要让 close 同时承担两个角色：既表示“生产结束”，又表示“消费者不想要”。一旦一个动作有两个含义，后来的人迟早会按错。

## buffer 只能买时间，不能买协议

很多人遇到 channel 卡住，第一反应是加 buffer。

```go
ch := make(chan T)
```

改成：

```go
ch := make(chan T, 100)
```

短期看，阻塞少了。

但这通常只是把问题往后推了 100 格。

buffer 有它的价值。它可以吸收突发流量，可以让生产和消费之间有一点弹性，也可以表达容量和背压。

问题是，buffer 不回答生命周期问题。

下游提前退出了吗？

上游还会不会继续发？

错误怎么传？

谁负责关闭？

取消信号往哪里走？

这些问题，容量再大也不会自动有答案。

Go Blog 那篇 pipeline 文章里有一句很重的话：goroutine 不是垃圾，它不会因为没人再用就自己消失；它必须自己退出。

这句话在真实项目里很值钱。

很多 goroutine 泄漏，不是因为 channel 难，而是因为代码只设计了“正常传完”，没设计“下游提前走了”。

比如下游只需要第一个结果：

```go
out := merge(c1, c2)
fmt.Println(<-out)
return
```

如果上游还有 goroutine 正在往 `out` 发送，而没人继续接收，它就可能永远卡在发送那一行。

你给 `out` 加一个 buffer，也许这次能过。

但只要数量变了、路径变了、错误分支变了，问题还会回来。

这时你真正需要的，不是更大的 channel，而是明确的退出协议：`context.Context`、done channel、错误传播、继续 drain，或者让上游能被通知停下来。

buffer 只能买时间，不能买协议。

## 生命周期要在写 channel 前设计

![channel 生命周期四问](/images/go-channel-close-lifecycle/lifecycle-flow.png)

判断一个 channel 方案稳不稳，不要只看发送和接收是否对得上。

要问四个问题：谁创建？谁发送？谁关闭？谁退出？

这四个问题没答案，代码就算能跑，也只是暂时没炸。

一个相对清楚的 pipeline，通常像这样：

```go
func gen(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

这里有几个边界是明的。

`gen` 创建 `out`。

`gen` 里的 goroutine 是唯一发送者。

发送完成后，它负责 `close(out)`。

如果外部取消，`ctx.Done()` 让它自己退出。

接收方只负责消费：

```go
for n := range gen(ctx, 1, 2, 3) {
    use(n)
}
```

它不需要关 `out`，也不需要猜什么时候结束。它只要读到 channel 关闭，循环自然结束。

这就是舒服的 channel 代码：不是因为它用了什么高级技巧，而是因为边界清楚。

反过来，如果一段代码里每个 goroutine 都可能发，每个地方都可能 close，取消信号有时走 context、有时走 done channel、有时靠关数据 channel 表达，那就危险了。

不是一定会立刻出 bug。

但后来的人很难判断：这个 channel 到底是谁的？谁有权结束它？我现在加一个发送者，会不会踩到别人的 close？

没人负责结束的 channel，迟早会反咬你。

## 一张真正该贴在脑子里的检查表

当你决定用 channel，不要急着写代码。

先把这张表过一遍：

| 问题 | 要回答什么 |
|---|---|
| 谁创建？ | channel 的所有权在哪里，生命周期从哪开始 |
| 谁发送？ | 是单发送者，还是多发送者；多发送者由谁协调 |
| 谁关闭？ | 关闭者是否真的知道“不会再发送” |
| 谁接收？ | 接收者是否可能提前退出 |
| 怎么取消？ | 用 context、done channel，还是其他明确机制 |
| 错误怎么传？ | 错误走结果 channel、单独 err channel，还是 context cause |
| buffer 代表什么？ | 吸收突发、限制容量、背压，还是只是遮住阻塞 |

这张表比“channel 怎么写”重要。

因为 channel 一旦进入工程代码，它就不只是一个类型。

它是一份协议。

协议里最容易漏的，从来不是“正常路径怎么传”。正常路径谁都会写。

真正容易漏的是反方向：取消、错误、提前返回、重复 close、多发送者、没人接收。

也就是所有线上问题最喜欢出现的地方。

## 系列最后，回到那句老话

这一组文章讲 channel，其实一直在压同一个判断。

上篇说：别为了“更 Go”，把一个普通 cache 写成小型 RPC。channel 不是高级锁。

中篇说：channel 还是 mutex，先问五个问题。mutex 保护状态，channel 描述关系。

这一篇说：当你真的选择 channel，别只设计怎么传，还要设计怎么停。

Go 的并发模型漂亮，不是因为它让代码看起来更花。

它漂亮在：如果你用得对，goroutine 在哪里工作，channel 在哪里交接，select 在哪里等待，close 在哪里宣布结束，都能被读出来。

但它也很诚实。

你没设计生命周期，它不会替你兜底。

你让接收方乱 close，它就 panic 给你看。

你把 buffer 当取消机制，它就把泄漏藏得更晚一点。

你以为 `select` 会替你公平调度，它就提醒你：语言只负责通信选择，业务规则自己写。

所以最后这句话，比任何语法技巧都重要：

channel 最难的不是传值，是收尾。

能把这句话记住，很多 Go 并发代码会少掉一半偶发事故。

---

**Channel 设计哲学三部曲回顾：**

- **上篇**：channel 的价值不是替代 mutex，而是把通信关系写出来
- **中篇**：判断工具前先判断问题——传东西用 channel，护东西用 mutex
- **下篇**（本篇）：channel 不仅要有关系，还要有生命周期

如果你觉得这三篇有用，可以关注。后续还会拆解 Go 其他核心机制——GC、GMP 调度、内存模型——每一篇都按这个节奏：不堆概念，讲清楚怎么选、怎么用、怎么不翻车。
