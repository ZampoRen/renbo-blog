---
title: "一个 goroutine 崩了，为什么你的 recover 根本接不住？"
date: 2026-07-13
description: "defer 不是简单的后进先出：它和 goroutine 绑定，panic 只在当前 goroutine 的栈上展开，recover 还要求直接写在被 defer 的函数里。"
slug: go-defer-panic-recover
categories:
  - golang
  - runtime
tags:
  - Go
  - Runtime
  - 并发
  - defer
  - panic
  - recover
cover: "/images/go-defer-panic-recover/cover.png"
toc: true
---

你在 HTTP 服务最外层加了 recovery middleware，以为兜底已经做完。后来某个 handler 里随手起了一个 goroutine；它空指针了，整个进程还是退出。

这不是 middleware 没挂上，也不是 recover 偶发失灵。是那个 goroutine 从来不在你的 recover 覆盖范围里。

很多人把 `defer` 记成“后进先出”，这句话没错，却少了决定线上行为的半截：**defer、panic 和 recover 都是在当前 goroutine 上完成的。** 你放进一个函数的清理动作，没法替另一个 goroutine 收尾；你放在外层的 recover，也接不住另一个 goroutine 的 panic。

这篇不复述语法。只解决几个在代码评审和事故现场会真正出问题的判断：`defer` 到底放在哪里，循环里为什么危险，recover 为什么看起来写了却没用，以及 panic 到底该不该出现在业务代码里。

## `defer` 不是一摞万能清理卡，它属于当前 goroutine

从语言语义看，`defer` 把一次函数调用登记下来，外层函数返回时再按后进先出的顺序执行。参数在登记那一刻就已经求值；它也能读写命名返回值。

这套规则足够写业务，但解释不了两个现象：为什么循环里的 `defer rows.Close()` 会拖住资源，为什么父函数的 recover 接不住子 goroutine。

把视角往 runtime 推一步，老路径里每次执行 defer，会形成一个 `_defer` 记录。Go 1.25 的 runtime 源码里，`_defer` 有 `sp`、`pc`、`fn` 和 `link` 字段；`link` 的注释写得很直接：它是 **next defer on G**。这里的 G 就是 goroutine。

也就是说，逻辑上可以把它想成这样：

```text
G#42
  _defer: close(body) -> unlock(mu) -> flush(log) -> nil

G#87
  _defer: recover(worker) -> nil
```

`G#42` 返回或 panic 时，只会消费自己的那条链。`G#87` 的 defer 不会被顺手执行，更不可能被 `G#42` 的 recover 接住。

<!-- 插图 1：defer 与 goroutine 绑定示意图（正文图，1 张）。画面：两个并排的 goroutine 栈；G#42 的 _defer 链包含 close / unlock，G#87 的链包含 recover。中间用醒目的断线和“不能跨 G 访问”标识。技术扁平风，深蓝底、青绿强调色。 -->

不过，别把 `_defer` 当成稳定 API，也别因此认定每个 defer 都会分配一个链表节点。现代 Go 有两条实现路径：没有把 defer 放进循环的很多函数，编译器会使用 open-coded defer，把函数与参数存到栈槽，用 bitmask 记录哪些 defer 已生效；在返回点直接生成调用代码。runtime 的 `panic.go` 就是这么描述它的。

因此，下面两句话要分开说：

- **语义上**，defer 一定在当前函数退出时、按 LIFO 执行；
- **实现上**，它可能走 `_defer` 链，也可能走编译器展开的栈槽和位图。具体编译器版本、函数形状都会影响路径。

源码图能帮你理解边界，不能替代语言语义。业务代码依赖前者，才不容易被版本细节带偏。

## 循环里最常见的问题，不是“慢”，是资源根本没还

下面这段代码在 review 里很常见：

```go
for _, id := range ids {
    rows, err := db.QueryContext(ctx, query, id)
    if err != nil {
        return err
    }
    defer rows.Close()

    if err := consume(rows); err != nil {
        return err
    }
}
```

问题不在于 `defer` 有一点调用开销。真正的坑是：循环不结束，所有 `rows.Close` 都不会执行。连接、文件描述符、锁，都会一直挂到外层函数返回。

短循环偶尔侥幸没出事，只是因为外层函数很快退出。批处理、导入任务、长连接消费这种路径，资源会持续堆着，最后表现为连接池耗尽、`too many open files`，或者后面的请求莫名开始排队。

**循环体里的资源，必须在本轮结束时释放；不要赌外层函数“应该很快返回”。**

有两个通常更稳的写法。资源本来就该在当轮消费完成后关闭时，显式关闭最清楚：

```go
for _, id := range ids {
    rows, err := db.QueryContext(ctx, query, id)
    if err != nil {
        return err
    }

    err = consume(rows)
    closeErr := rows.Close()
    if err != nil {
        return err
    }
    if closeErr != nil {
        return closeErr
    }
}
```

如果分支多、每一轮都需要多处提前返回，就把循环体收进一个小函数，让 defer 的作用域缩回单次迭代：

```go
for _, id := range ids {
    if err := func() error {
        rows, err := db.QueryContext(ctx, query, id)
        if err != nil {
            return err
        }
        defer rows.Close()
        return consume(rows)
    }(); err != nil {
        return err
    }
}
```

这里不是反对 defer。`mu.Lock()` 后紧挨着 `defer mu.Unlock()`，拿到文件后紧挨着 `defer f.Close()`，仍然是 Go 里最不容易漏清理的写法。你应该警惕的是：资源生命周期比当前函数短，或者 defer 的数量随请求量、循环次数增长。

至于高频路径，也别靠印象下结论“defer 慢”。Go 1.14 起，官方发布说明已说明大多数 defer 的性能接近直接调用；当前 runtime 仍会对不含循环 defer 的函数使用 open-coded defer。先用 benchmark、pprof 或汇编确认它是不是热路径、是否真的没走优化；只有证据表明它在烧钱，才用显式调用换取那点成本。为了省几个纳秒写出会漏 Unlock 的代码，通常是把账算反了。

还有一个容易被混淆的账：defer 的优化不等于“循环里的 defer 已经没有问题”。open-coded defer 的一个关键前提正是函数中没有把 defer 放进循环。即便某个版本把调用成本压得很低，资源仍要等外层函数返回才释放；性能与生命周期是两件事，后者通常更早把系统拖垮。

## panic 发生后，程序不是立刻死，而是沿当前栈一层层“退场”

`panic` 会中断普通控制流。runtime 为这次异常维护 `_panic` 状态，其中有 panic 值、指向更早 panic 的 `link`，以及是否已经恢复的标记。出现 panic 后，runtime 从当前 goroutine 的栈上找可执行的 deferred call：先执行离现场最近的一层，再继续向调用方展开。

可以把它看成一次反向撤场：

```text
handler -> service -> repository -> panic
              ↑             ↑
        defer metrics  defer tx.Rollback

执行顺序：tx.Rollback -> metrics -> handler 的 recover -> 恢复或进程退出
```

<!-- 插图 2：panic 传播与栈展开图（正文图，1 张）。画面：纵向调用栈 handler → service → repository → panic，红色箭头向上；每层旁边标记 deferred call 的执行顺序，handler 顶部放 recover 闸门。标明“仅当前 goroutine”。技术扁平风，沿用深蓝、青绿，panic 用橙红。 -->

如果一路没有合格的 recover，当前 goroutine 的 defer 会被执行完，随后运行时打印 panic 与栈信息并终止程序。这里的“当前 goroutine”不能省略：子 goroutine 的 panic 不会穿越调度器，跳进创建它的 handler 栈里。

所以 HTTP middleware 只能保护执行 `ServeHTTP` 的那条 goroutine。你自己 `go func()` 启动后台任务时，应该在它的入口就决定异常策略：要么让它返回 error 并上报，要么在入口单独 recover、记录完整栈和任务上下文，然后按业务决定重试、熔断或退出。

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            logger.Error("worker panic", "panic", r, "stack", debug.Stack())
        }
    }()

    runWorker(ctx)
}()
```

这段保护的目标是隔离一个后台任务，不是证明系统已经安全。recover 之后如果队列状态、内存数据或外部事务已经不可信，悄悄继续跑可能比退出更危险。日志里至少要带 panic 值、堆栈、任务标识；对关键任务还要有存活检测和重启策略。

## recover 有三个容易写错的边界

recover 的规则比“放在 defer 里”更窄。它要在**正在 panic 的 goroutine**中，被**直接由 defer 调用的函数**调用，才能拿到 panic 值并停止这次展开。

第一，跨 goroutine 不行，前面已经看到原因。父 goroutine 的 defer 链和子 goroutine 的 defer 链是两份状态。

第二，套一层 helper 也不行：

```go
func recoverPanic() any { return recover() }

func f() {
    defer func() {
        _ = recoverPanic() // 拿到 nil；panic 继续向外传
    }()
    panic("boom")
}
```

`recover` 不是普通的“查一下当前是否异常”的函数。runtime 要确认它就是那次 deferred call 的直接调用。把它藏进通用 helper、方法调用或另一层 deferred function，都会失去这个资格。

第三，在普通执行路径调用 recover 也不行。它只会返回 `nil`，不会给你的代码装上一个全局异常捕获器。

实际代码里，最不容易误读的形式就是：

```go
defer func() {
    if r := recover(); r != nil {
        // 记录、转换或在边界处隔离
    }
}()
```

把 recover 贴在 defer 的匿名函数里，看起来笨一点，却把作用域、调用时机和语义都摊在了代码审查者眼前。

## recover 接住的是控制流，不会替你把现场复原

这也是 recover 最容易被滥用的地方。它能让栈停止展开，却不会撤销 panic 之前已经发生的副作用。

假设一个 worker 已经从 channel 取走消息，写完了半条数据库记录，随后在更新内存索引时 panic。你在 goroutine 外层 recover 住了它，进程还活着；但那条消息是否该重试、数据库是否已提交、索引是否还能继续提供服务，都没有自动答案。

这类场景里，recover 后直接打印一条日志然后继续，往往只是把一次明确的崩溃换成了一段更难追的脏状态。真正要设计的是恢复策略：

- 请求入口通常可以写 500、打点并结束这次请求，下一次请求从干净边界重新开始；
- 消费者要明确消息确认发生在什么位置，能否幂等重试，失败是否应该转入死信队列；
- 进程级共享状态一旦可能被破坏，宁可让健康检查把实例摘掉或重启，也不要让它带着不确定状态继续接流量。

所以不要把 `recover` 写成“服务不会挂”的承诺。它更像断路器：把炸掉的那段调用链隔开，给你记录、降级和退出的机会。接下来怎么处理，仍要由资源一致性和业务语义决定。

## panic 适合切断“不可能继续”的路径，不适合替代错误设计

业务层把数据库超时、第三方 500、用户参数错误都改成 panic，再靠最外层 recover 翻回 error，通常是把正常失败伪装成异常控制流。调用者看不到失败类型，局部处理机会也消失，测试很快会变成“祈祷 middleware 没漏”。

panic 更适合三类边界。

启动阶段的不可恢复配置错误可以 panic：端口、密钥、依赖配置都不合法，服务本来就不该假装启动成功。

框架最外层可以 recover：HTTP handler、RPC 请求入口、worker 任务入口。它的职责是阻止单个请求或任务把整个进程拖死，同时把异常变成可观测事件。标准库 `encoding/json` 也使用过类似的内部模式：递归编码的内部层用 panic 快速退栈，最外层统一 recover，再向调用者返回 error。重点在于，panic 没有泄漏成公共 API。

最后是框架内部：当内部递归层级很深、错误要跨很多不适合层层传递的私有函数回到唯一边界时，panic/recover 可以降低内部样板代码。但这个边界必须小、封闭、可测试。公共 API 仍然返回 error，不能让使用者猜测哪里会突然炸掉。

这也是 Go 和 try-catch 的分歧。Go 没把异常当成日常分支的通行证。普通失败用显式 `error`，调用链上的每一层都有权处理、包装或继续返回；panic 留给程序不变量被打破、启动无法继续、框架内部快速退栈这些少数场景。

不是因为 panic “邪恶”，而是它会跳过中间层的正常决策。跳得越远，恢复后的状态越难判断。

## 代码评审时，拿这四个问题过一遍

遇到 defer、panic、recover，不必背 runtime 源码。问四件事就够了：

1. 这个 defer 所属的函数什么时候返回？资源是否能等到那一刻？
2. 这段代码有没有在循环里累计 defer？如果有，每轮能否缩小成一个函数作用域？
3. panic 在哪条 goroutine 上发生？那条 goroutine 的入口有没有明确的处理策略？
4. recover 是不是直接写在被 defer 的函数里？恢复后，状态还能不能信？

把这四个问题问清，defer 就不再只是“后进先出”。它是一份跟着 goroutine 和函数边界走的清理承诺；panic 是沿同一条栈展开的非常规退出；recover 则是一道必须放在正确位置的窄门。

真要追源码时，也别从“哪一个 runtime 函数更快”开始。先把函数返回边界、资源归还时机和 goroutine 入口画出来。多数 defer 事故不需要改 runtime，不需要替换语法，甚至不需要加新的中间件；把一段循环提成小函数，或在 worker 入口补上一段可观测的保护，就已经把最危险的边界摆正了。

边界摆正，排障才有起点。

先把错误放回它发生的那条 G。

下次看到 `go func()`，别只问它什么时候结束。也问一句：它崩了以后，谁来收拾现场？

---

### 参考资料

- Go 语言规范：[`Defer statements`](https://go.dev/ref/spec#Defer_statements)、[`Handling panics`](https://go.dev/ref/spec#Handling_panics)
- Go Blog：[Defer, Panic, and Recover](https://go.dev/blog/defer-panic-and-recover)
- Go 1.14 Release Notes：[defer 性能改进](https://go.dev/doc/go1.14)
- Go 1.21 Release Notes：[recover 直接调用语义](https://go.dev/doc/go1.21)
- Go 1.25.4 runtime 源码：[`runtime2.go`](https://go.dev/src/runtime/runtime2.go)、[`panic.go`](https://go.dev/src/runtime/panic.go)
