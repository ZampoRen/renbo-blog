---
title: "ready=true 不是发布：Go 内存模型真正要你保证什么"
description: "从一个看似顺手的 ready 标志位开始，讲清 Go happens-before、data race 和 DRF-SC：并发安全不是看起来有顺序，而是规范承认它有顺序。"
date: 2026-05-28T14:40:00+08:00
draft: false
author: "任博"
tags: ["Go", "并发编程", "内存模型", "happens-before", "data race"]
categories: ["技术实战"]
cover: "/images/go-memory-hb-intro/cover.png"
toc: true
---

线上最烦的并发 bug，往往不是崩得轰轰烈烈，而是偶尔读到一个不该出现的旧值。

配置已经加载了，日志也打了“ready”，另一个 goroutine 却像没看见一样，读到的还是旧数据。你盯着代码看半天，越看越觉得冤：明明是先写数据，再把 `ready` 置成 `true`。读的一侧也是先等 `ready`，再读数据。

人脑看这段逻辑，会自然补上一句：既然 `ready` 已经是 `true`，前面的数据当然也准备好了。

问题就出在这个“当然”。

在 Go 里，`ready=true` 不等于发布。普通变量上的“先写后读”，也不等于 goroutine 之间有可见性保证。

并发安全不是看起来有顺序，而是规范承认它有顺序。

这篇只讲 Go 内存模型里最该先吃透的一层：goroutine A 写了变量，goroutine B 什么时候保证能看到？答案不是“代码写在前面”，也不是“机器上跑过没问题”，而是两边有没有一条 happens-before 关系。

![happens-before 证明链](/images/go-memory-hb-intro/happens-before-chain.svg)

## 最危险的代码，通常看起来最顺

先看这段很多人都写过、也很容易在 review 里漏掉的代码：

```go
var data string
var ready bool

go func() {
    data = "payload"
    ready = true
}()

for !ready {
}
fmt.Println(data)
```

直觉上，它顺得不能再顺。

写的一侧：先写 `data`，再写 `ready`。

读的一侧：先看到 `ready`，再打印 `data`。

如果这是一段单线程代码，这种理解没问题。但它跨了 goroutine，规则就变了。你不能把同一个 goroutine 里的源码顺序，直接搬到另一个 goroutine 的可见性里。

这段代码在 Go 内存模型里是错的。

错不在“CPU 一定会重排”，也不在“编译器一定会优化坏”。这些说法太具体，反而会把问题讲窄。真正的问题更直接：`data` 和 `ready` 都是普通变量，不同 goroutine 对它们并发读写，中间没有任何同步动作。

没有同步，就没有 happens-before。

没有 happens-before，读到 `ready=true` 也不保证读到 `data="payload"`。

这里最容易误导人的，是代码长得太像“发布”。你把数据准备好，再立一个旗子，看起来像把数据交给了别人。但在 Go 规范眼里，这个旗子只是一个普通 bool。

没有同步，ready 只是普通 bool，不是一张通行证。

## happens-before 不是时间线，是证明链

很多人第一次看到 happens-before，会把它理解成“真实时间上先发生”。这会让后面的判断全歪掉。

happens-before 更像一条可见性证明链。链上的每一段，要么来自同一个 goroutine 内的顺序执行，要么来自一个明确的同步动作。最后这些小段连起来，才形成跨 goroutine 的保证。

可以先把它压成三句话：

```text
同一个 goroutine 内的程序顺序：sequenced-before
跨 goroutine 的同步顺序：synchronized-before

happens-before = 这些关系连起来之后形成的证明链
```

同一个 goroutine 里：

```go
data = "payload"
ready = true
```

你可以说 `data = "payload"` 在 `ready = true` 之前。这个“之前”只在当前 goroutine 内成立。

但另一个 goroutine 能不能据此看到 `data`？不能。中间还缺一座桥。

桥是什么？是 channel、mutex、Once、WaitGroup、atomic 这类同步动作。它们的价值，不只是“通知一下”或者“挡一下并发”，而是把两个 goroutine 的可见性关系接起来。

比如用 channel close 做发布：

```go
var data string
done := make(chan struct{})

go func() {
    data = "payload"
    close(done)
}()

<-done
fmt.Println(data)
```

这段代码和前面的 `ready` 版本，表面上都像“等一个信号”。但规范地位完全不同。

`close(done)` happens-before 那个因为 channel 已关闭而返回的 receive。于是链条变成了：

```text
data = "payload"
    happens-before close(done)
close(done)
    happens-before <-done 返回
<-done 返回
    happens-before fmt.Println(data)
```

链闭合了，读取才有保证。

这就是 happens-before 最重要的用法：它不是帮你描述“我感觉代码先后顺序是这样”，而是让你证明“规范承认这次读取应该看到之前的写入”。

## data race 的定义，比偶现 bug 更严格

很多人判断并发代码有没有问题，习惯看现象：有没有 panic？有没有线上报错？压测有没有跑出来？

这套判断在 data race 面前不够用。

Go 对 data race 的工程化判断，可以压成四个条件：

```text
同一个内存位置
两个操作并发发生
至少一个是写
中间没有同步
```

满足这四个条件，程序就是错的。它不需要先在生产环境闹出事故，也不需要 race detector 先报警，才算错。

回到 `ready` 那段代码：

```go
// goroutine A
ready = true

// goroutine B
for !ready {
}
```

这是同一个内存位置 `ready`。一个写，一个读。两个 goroutine 并发发生。中间没有同步。

`data` 也一样：一个 goroutine 写，另一个 goroutine 读，中间没有 happens-before 证明链。

很多团队最容易踩的坑，是把“没复现”当成“没问题”。这其实是在拿平台偶然表现当语言保证。

你的开发机可能是 amd64，内存顺序比某些架构更强；这次 Go 编译器可能没有做让问题暴露的优化；测试数据量小，时序刚好，也可能把问题藏起来。可这些都不是 Go 规范给你的承诺。

正确的问题不是：它现在会不会刚好跑通？

正确的问题是：这段读写之间有没有 happens-before？

如果没有，它不是“有点冒险”，而是已经越过了并发代码的安全边界。

还有一种更隐蔽的误判：把 `time.Sleep` 当同步。

```go
go func() {
    data = "payload"
    ready = true
}()

time.Sleep(time.Millisecond)
fmt.Println(data)
```

这类代码在本地演示里经常“看起来能跑”。但 sleep 只是在赌调度时间，不是在建立可见性关系。它可能让写入更早发生，却没有告诉 Go 内存模型：这次读取必须看到那次写入。

并发代码里，时间不是同步原语。

只要你靠“多等一会儿”“机器够快”“一般先执行到这里”来证明正确性，本质上还是没有证明。真正能让代码站住的，不是你等了多久，而是你用了什么同步动作。

## DRF-SC：Go 给你的大承诺

Go 内存模型里有一个很重要的承诺：DRF-SC。

全称是 data-race-free programs execute in a sequentially consistent manner。翻成工程语言就是：只要你的程序没有 data race，你基本可以按“单处理器上交错执行”的方式理解并发结果。

这句话非常关键，因为它解释了 Go 为什么不希望普通业务代码天天和内存序搏斗。

底层当然复杂。CPU 有缓存，编译器会优化，运行时要调度 goroutine，不同架构还有不同的内存顺序。但只要你把同步关系写清楚，Go 就尽量把你拉回一个可推理的世界里。

DRF-SC 的意思很朴素：你不乱来，Go 就让世界重新变简单。

反过来也成立：一旦你用普通变量做跨 goroutine 通知，一旦你让读写之间没有同步，Go 就不会再替你维持那个“看起来应该如此”的世界。

Go 的内存模型不是鼓励你写更聪明的无锁代码，而是帮你少写这种代码。

这也是为什么 Go 的 `sync/atomic` 没有像 C/C++ 那样把 relaxed、acquire、release 一堆旋钮全摊给你。Go 更愿意给多数工程师一套可推理、少选项的并发模型。

少给一点旋钮，多给一点可推理性。

## 下次审代码，先问这三个问题

内存模型不应该变成一套背诵题。你真正要带走的，是一个审代码时能立刻用上的判断框架。

第一，是否有普通变量跨 goroutine 读写？

看到全局变量、共享 struct 字段、bool 状态位、map、slice 指针，都要先停一下。尤其是 `ready`、`done`、`inited`、`closed` 这种名字，它们看起来像同步，其实经常只是普通变量。

第二，至少一个操作是不是写？

多个 goroutine 只读同一份不可变数据，通常不是问题。真正危险的是一边写，一边读，或者多边写。

第三，中间有没有同步动作把它们接起来？

不是有没有 `if ready`，不是有没有 `sleep`，不是有没有“我觉得先发生”。你要找的是 channel send/receive/close、mutex lock/unlock、Once、WaitGroup、atomic 这类规范承认的同步关系。

如果三问下来发现：普通共享变量、并发读写、没有同步，那就不要再纠结“线上会不会刚好没事”。这段代码已经站不住了。

把它改掉。

用 channel 传递结果，用 `close(done)` 发布完成，用 mutex 保护共享状态，用 `sync.Once` 做初始化，用 atomic 处理足够简单的状态发布。选哪个不重要，重要的是你必须让规范能看见那条链。

`ready=true` 不是发布。

真正的发布，是写入和读取之间有一条说得清、查得到、被 Go 内存模型承认的 happens-before 证明链。

下篇我们继续往下走：channel、Mutex、Once、WaitGroup、atomic 这些同步原语，到底分别给了你什么保证；以及为什么 `atomic` 不是 mutex 的高级替代品。

如果你正在写 Go 并发代码，尤其是服务初始化、配置热更新、缓存刷新、后台 worker 协调这一类场景，建议关注后续这一组文章。内存模型不需要每天挂在嘴边，但每次共享状态跨 goroutine 时，你都要知道自己在向规范借哪一条保证。
