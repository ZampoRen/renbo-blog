---
title: "加 buffer 不是修 bug：Go channel 阻塞排查实战"
description: "goroutine 涨了，channel 卡了，第一反应加 buffer？这篇从 pprof 采样到三条线分析法，把 Go channel 六大常见陷阱、排查路径和修复模式一次讲透。"
date: 2026-05-21T18:30:00+08:00
draft: false
author: "任博"
tags: ["Go", "channel", "并发编程", "pprof", "调试", "goroutine泄漏"]
categories: ["技术实战"]
cover: "/images/go-channel-practice-debugging/cover.svg"
toc: true
---

goroutine 数量从 200 慢慢涨到 2 万。

监控图上那条线，不陡，但一直在爬。就像水龙头没拧紧——不喷，但也不会停。

你的第一反应是什么？

加大 buffer。

`make(chan int, 100)` 改成 `make(chan int, 1000)`。重启。观察。好像好了。

三天后，又涨回来了。

![Go channel 阻塞排查封面](/images/go-channel-practice-debugging/cover.svg)

这不是运气差。这是 Go 服务最常见的线上问题之一：channel 阻塞导致 goroutine 泄漏。而"加 buffer"是社区里最流行、也最没用的解法。

buffer 只是把崩溃推迟了。问题的根因——谁在等谁、谁该退出、谁该 close——一个都没解决。

这篇文章做一件事：从"goroutine 涨了"这个信号出发，走一遍完整的排查路径，把 channel 的六个最常见陷阱逐个拆开。

不是背 API。

是下次线上出问题的时候，你知道先看什么。

## 发现：怎么知道 channel 卡住了

排查 channel 阻塞的第一步不是看代码，是看数字。

### runtime.NumGoroutine()：最简单的基线

```go
import "runtime"

func init() {
    go func() {
        ticker := time.NewTicker(30 * time.Second)
        defer ticker.Stop()
        for range ticker.C {
            log.Printf("goroutines: %d", runtime.NumGoroutine())
        }
    }()
}
```

这段代码放到任何 Go 服务的 `init()` 里，每 30 秒打一行日志。goroutine 数量只升不降，大概率有泄漏。

线上服务建议接入 Prometheus 的 `go_goroutines` 指标，设告警阈值。数字本身不能告诉你原因，但能告诉你"有事发生了"。

### pprof：看到底谁卡在 channel 上

```bash
# 确保服务里引入了 pprof
import _ "net/http/pprof"

# 采样
curl -s http://localhost:6060/debug/pprof/goroutine?debug=1
```

输出里如果看到大量这样的栈：

```
goroutine 1234 [chan send]:
main.processTask(...)
    /app/worker.go:42 +0x120
```

或者：

```
goroutine 5678 [chan receive]:
main.consumer(...)
    /app/consumer.go:18 +0x80
```

`chan send` 和 `chan receive` 就是阻塞点。goroutine 卡在发送或接收上，等另一端没人来。

![排查流程](/images/go-channel-practice-debugging/inline-01.svg)

### go vet：静态检查

```bash
go vet ./...
```

`go vet` 能发现部分 channel 相关的静态问题，比如变量遮蔽（shadow）导致的意外 nil channel。它不是万能的，但零成本，CI 里加上没坏处。

发现了 `chan send` / `chan receive` 阻塞之后，下一步是判断：它到底卡在哪种情况？

这就需要认识 channel 的六个陷阱。

## 六大陷阱

每个陷阱都给你看：bug 长什么样，为什么会这样，怎么修。

### 陷阱一：Nil Channel —— 永久阻塞

对一个 `nil` channel 发送或接收，不会 panic。

它会**永久阻塞**。

```go
func buggy() {
    var ch chan int  // nil —— 没有 make
    ch <- 42        // 卡死，goroutine 泄漏
}
```

这段代码编译能过，运行不报错，只是那个 goroutine 再也醒不过来了。

**为什么会 nil？** 最常见的原因：变量声明了但没初始化，或者在某个分支里被赋值为 nil。

**怎么用它？** nil channel 在 `select` 里有正经用途——动态禁用某个 case：

```go
func withOptionalTimeout(recvCh <-chan int, enforce bool) (int, error) {
    var timeout <-chan time.Time
    if enforce {
        timeout = time.After(10 * time.Second)
    }
    select {
    case v := <-recvCh:
        return v, nil
    case <-timeout:    // timeout 为 nil 时，这个 case 被禁用
        return 0, errors.New("timed out")
    }
}
```

`timeout` 是 nil，`select` 就永远不会选它。这是 Go 的惯用技巧，不是 bug。

**修复原则：** 如果你没有主动使用 nil channel 的意图，`var ch chan int` 后面必须跟 `make`。

### 陷阱二：Closed Channel Panic —— 谁该关灯

三种 panic 场景：

```go
// 1. 关闭 nil channel
close(ch) // ch 为 nil → panic

// 2. 重复关闭
close(ch)
close(ch) // panic: close of closed channel

// 3. 向已关闭 channel 发送
close(ch)
ch <- 1   // panic: send on closed channel
```

第三种最隐蔽。通常发生在：一个 goroutine 关了 channel，另一个 goroutine 还在发。

**关闭原则：** 只在唯一的 sender goroutine 里关闭 channel。不要从 receiver 端关闭。不要在有多个 sender 的时候从 sender 端关闭。

唯一 sender 直接关：

```go
go func() {
    defer close(ch)  // sender 负责关
    for i := 0; i < 10; i++ {
        ch <- i
    }
}()
```

多个 sender，用 WaitGroup：

```go
var wg sync.WaitGroup
for i := 0; i < 3; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        for j := 0; j < 5; j++ {
            ch <- id*10 + j
        }
    }(i)
}
go func() {
    wg.Wait()
    close(ch)  // 所有 sender 完成后再关
}()
```

**兜底方案：** 如果实在不确定谁该关，用 `sync.Once`：

```go
var closeOnce sync.Once
safeClose := func() {
    closeOnce.Do(func() { close(ch) })
}
```

但这是兜底，不是首选。需要 `sync.Once` 来防重复关闭，说明 close 的职责分配本身有问题。

### 陷阱三：Buffer 当创可贴 —— 推迟崩溃

这是文章标题说的那个。

```go
ch := make(chan int, 100)  // "大一点就不会死了吧"
go func() {
    for i := 0; i < 10000; i++ {
        ch <- i  // 发到第 101 个还是会阻塞
    }
}()
// 只消费了 50 个就停了
for i := 0; i < 50; i++ {
    <-ch
}
// 剩下的 9950 个值卡在 buffer 里，第 101 次发送永久阻塞
```

buffer 从 100 调到 1000，只是把问题从"第 101 次卡住"推迟到"第 1001 次卡住"。

**什么时候该用 buffer？** 生产消费速率不匹配，但总量可控。比如：生产者偶尔突发，消费者稳定处理，buffer 吸收尖峰。

**什么时候不该？** 生命周期没管好。sender 不知道 receiver 已经退出了，或者 receiver 不知道 sender 已经退出了。这时候该修的是退出逻辑，不是 buffer 大小。

```go
// 正确：用 context 管生命周期，不靠 buffer
func properLifecycle(ctx context.Context) {
    ch := make(chan int)  // 无缓冲就够了
    go func() {
        defer close(ch)
        for i := 0; ; i++ {
            select {
            case ch <- i:
            case <-ctx.Done():
                return
            }
        }
    }()
}
```

### 陷阱四：Goroutine 泄漏 —— sender 没人接，receiver 没人发

这是最常见的一类。两种方向：

**方向 A：sender 没有 receiver**

```go
func leakySender() {
    ch := make(chan int)
    go func() {
        for i := 0; ; i++ {
            ch <- i  // 第 6 次发送永久阻塞
        }
    }()
    for i := 0; i < 5; i++ {
        fmt.Println(<-ch)
    }
    // 函数返回，没人读 ch 了，sender goroutine 泄漏
}
```

**方向 B：receiver 没有 sender**

```go
func leakyReceiver() {
    ch := make(chan int)
    go func() {
        ch <- 1
        ch <- 2
        // 退出了但没有 close(ch)
    }()
    for v := range ch {  // 永久阻塞，等第 3 个值或 close
        fmt.Println(v)
    }
}
```

**检测手段：**

```go
// 1. 测试里用 goleak
import "go.uber.org/goleak"

func TestNoLeak(t *testing.T) {
    defer goleak.VerifyNone(t)
    doSomething()
}

// 2. 运行时基线对比
before := runtime.NumGoroutine()
doSomething()
time.Sleep(100 * time.Millisecond)
after := runtime.NumGoroutine()
if after > before {
    log.Printf("WARNING: leaked %d goroutines", after-before)
}
```

**修复原则：** 每个 goroutine 都必须有退出路径。要么靠 `close(ch)`，要么靠 `ctx.Done()`，要么靠 `select` 里的 `default` + 条件判断。没有退出路径的 goroutine 就是泄漏。

### 陷阱五：Dead Select Case —— 永远不就绪的分支

`select` 里某个 case 因为逻辑错误永远无法触发。

**场景一：default 导致忙等**

```go
for {
    select {
    case v := <-workCh:
        process(v)
    default:
        // workCh 为空时，CPU 100% 空转
    }
}
```

`default` 让 `select` 变成非阻塞。channel 没数据时它不等，直接走 `default`，然后循环回来再试。如果 channel 长时间没数据，这就是一个 CPU 100% 的死循环。

**修复：** 用 `time.Ticker` 替代 `default`：

```go
ticker := time.NewTicker(100 * time.Millisecond)
defer ticker.Stop()
for {
    select {
    case v := <-workCh:
        process(v)
    case <-ticker.C:
        doHealthCheck()
    }
}
```

**场景二：channel 变量被意外覆盖为 nil**

```go
ch2 = nil  // 意外！
for {
    select {
    case v := <-ch1:
        handle(v)
    case v := <-ch2:  // 永远不触发
        handle(v)
    }
}
```

**场景三：发送方永远不会发**

```go
go func() {
    if false {  // 条件永远为 false
        ch <- 42
    }
    // 没有 close(ch)
}()
v := <-ch  // 永久阻塞
```

**排查方法：** 在 `select` 前后加日志，打印每个 channel 变量的值。如果某个分支从不执行，检查对应 channel 是否为 nil、是否有人在写、是否已被关闭。

### 陷阱六：Range 不退出 —— 忘了 close 的连锁反应

`for v := range ch` 会一直阻塞，直到 channel 被关闭。如果发送方忘了 `close(ch)`，接收方永远不退出。

```go
func producer() <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; i < 5; i++ {
            ch <- i
        }
        // 忘了 close(ch)
    }()
    return ch
}

func consumer() {
    for v := range producer() {  // 打印 0-4 后永久阻塞
        fmt.Println(v)
    }
    fmt.Println("done")  // 永远到不了这里
}
```

**fan-in 场景更隐蔽：**

```go
func fanIn(sources []<-chan int) <-chan int {
    merged := make(chan int)
    for _, src := range sources {
        go func(ch <-chan int) {
            for v := range ch {
                merged <- v
            }
            // src 关闭后这里会退出，但 merged 从未关闭
        }(src)
    }
    return merged  // 消费者 range merged 时永久阻塞
}
```

**修复：** 用 WaitGroup 关闭 merged：

```go
func fixedFanIn(sources []<-chan int) <-chan int {
    merged := make(chan int)
    var wg sync.WaitGroup
    for _, src := range sources {
        wg.Add(1)
        go func(ch <-chan int) {
            defer wg.Done()
            for v := range ch {
                merged <- v
            }
        }(src)
    }
    go func() {
        wg.Wait()
        close(merged)  // 所有源关闭后关 merged
    }()
    return merged
}
```

## 设计模式：正确使用 channel 的姿势

认识了陷阱，再看正确模式。

### Fan-out / Fan-in

一个 channel 分发给多个 worker（fan-out），多个 worker 的结果汇聚到一个 channel（fan-in）。

```go
func fanOutFanIn(input <-chan int, workers int) <-chan int {
    // fan-out: 每个 worker 从同一个 input 读
    results := make(chan int)
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for v := range input {
                results <- v * 2  // 处理后写入 results
            }
        }()
    }
    // fan-in: 所有 worker 完成后关闭 results
    go func() {
        wg.Wait()
        close(results)
    }()
    return results
}
```

### Worker Pool

jobs channel 发任务，results channel 收结果：

```go
func workerPool(jobs []int, workers int) []int {
    jobCh := make(chan int, len(jobs))
    resultCh := make(chan int, len(jobs))

    // 启动 worker
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobCh {
                resultCh <- process(job)
            }
        }()
    }

    // 发任务
    for _, j := range jobs {
        jobCh <- j
    }
    close(jobCh)

    // 等完成
    go func() {
        wg.Wait()
        close(resultCh)
    }()

    // 收结果
    var results []int
    for r := range resultCh {
        results = append(results, r)
    }
    return results
}
```

### Pipeline

多阶段管道，每阶段从上一级读、处理、写入下一级：

```go
func pipeline(ctx context.Context) <-chan int {
    // Stage 1: 生成
    gen := func() <-chan int {
        out := make(chan int)
        go func() {
            defer close(out)
            for i := 0; ; i++ {
                select {
                case out <- i:
                case <-ctx.Done():
                    return
                }
            }
        }()
        return out
    }

    // Stage 2: 过滤偶数
    filter := func(in <-chan int) <-chan int {
        out := make(chan int)
        go func() {
            defer close(out)
            for v := range in {
                if v%2 != 0 {
                    select {
                    case out <- v:
                    case <-ctx.Done():
                        return
                    }
                }
            }
        }()
        return out
    }

    return filter(gen())
}
```

每个 stage 都 `defer close(out)`，每个 stage 都 `select` 监听 `ctx.Done()`。这样整条管道可以被 context 取消，不会泄漏。

### Select + ctx.Done() 统一取消

所有需要退出的 goroutine，统一用这个模式：

```go
go func() {
    defer close(out)
    for {
        select {
        case <-ctx.Done():
            return
        case out <- work():
        }
    }
}()
```

不要自创 done channel，不要用 `bool` flag，不要用 `atomic`。`context.Context` 就是 Go 给你的标准取消协议。

## 排查清单

从"goroutine 涨了"到"定位根因"的完整路径：

**第一步：确认**

- `runtime.NumGoroutine()` 基线对比，确认数量只升不降
- `curl localhost:6060/debug/pprof/goroutine?debug=1` 看栈

**第二步：分类**

- 栈里是 `chan send` → 找谁该读这个 channel
- 栈里是 `chan receive` → 找谁该写这个 channel
- 栈里是 `select` → 检查每个 case 的 channel 状态

**第三步：三条线分析法**

- **创建路径：** 这个 channel 是谁 `make` 的？生命周期归谁管？
- **取消路径：** 谁负责关闭？有没有 `ctx.Done()` 退出机制？
- **响应路径：** 发送方有没有 receiver？接收方有没有 sender？

三条线都走通，channel 就不会阻塞。任何一条断了，就是泄漏点。

**第四步：修复 + 验证**

- 修复代码
- `goleak.VerifyNone(t)` 验证无泄漏
- `runtime.NumGoroutine()` 基线对比验证数量稳定

## 速查表

![Go Channel 调试速查表](/images/go-channel-practice-debugging/inline-02.svg)

## 最佳实践

几条从坑里爬出来的经验：

**sender 负责 close。** 这是最基本的。谁生产数据，谁在生产结束后关 channel。receiver 不要关，因为你不知道还有没有 sender 在路上。

**多 sender 用 WaitGroup。** 等所有 sender 完成后，再由一个单独的 goroutine 关闭 channel。

**select + ctx.Done() 统一取消。** 不要自创 done channel。`context.Context` 是 Go 标准库给你的取消协议，用它。

**buffer 大小不是调优参数。** buffer 0（无缓冲）是默认值，也是最安全的值。只有在你明确知道生产消费速率差异、且总量可控时，才加 buffer。buffer 不是越大越好，越大只是把问题藏得越深。

**测试里加 goleak。** 一行代码，零成本，能捕获 90% 的 goroutine 泄漏。

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

回到开头的场景。

goroutine 涨了，加 buffer 好了一阵，又涨回来。

现在你知道了：加 buffer 只是推迟崩溃，不是修复问题。真正要查的是——哪个 channel 的哪一端没人了，哪条 goroutine 的退出路径断了。

排查 channel 阻塞不需要灵感，需要路径。

pprof 看栈，三条线分析法定位，context + defer close 修复，goleak 验证。每一步都有工具，每一步都有确定的判断标准。

下次 goroutine 再涨，别急着重启。

先看数字，再看栈，再看谁该关灯。
