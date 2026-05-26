---
title: "context.WithValue 不是 map，是一条链"
description: "很多 Go 代码把 context.WithValue 当成万能参数袋，函数签名看起来干净，依赖却被藏进调用链。从 valueCtx、propagateCancel、Background 和 TODO 的源码边界出发，给出一份可以直接拿去做 code review 的 context 检查清单。"
date: 2026-05-26T17:20:00+08:00
draft: false
author: "任博"
tags: ["Go", "context", "WithValue", "源码", "后端工程"]
categories: ["技术实战"]
cover: "/images/go-context-value-is-chain/cover.png"
toc: true
---

函数签名很干净，代码却越来越难查。

你打开一个 Go 函数，只看到一个 `ctx context.Context`。再往里翻，发现 user id 从 `ctx.Value` 里取，tenant 从 `ctx.Value` 里取，trace id 从 `ctx.Value` 里取，logger、配置开关、灰度标记，也都从 `ctx.Value` 里取。

签名短了，依赖没少。

它们只是被藏起来了。

![context.WithValue 链式查找](/images/go-context-value-is-chain/cover.png)

上篇我们拆了 `WithTimeout`：`cancel` 不是超时按钮，而是资源释放按钮。再往下看，就会撞上 `context` 里另一个最容易被滥用的口子：`WithValue`。

它看起来太顺手了。

```go
ctx = context.WithValue(ctx, userIDKey{}, userID)
```

一行代码，不改函数签名，不加参数，不动调用层级，值就能一路传下去。

这正是危险的地方。

`WithValue` 不是让你逃开函数签名的后门。它在源码里也不是一个 map。每调用一次，它只是给原来的 context 外面再包一层。

这篇把 context 系列收住：先看 `propagateCancel` 为什么不是每层都起 goroutine，再看 `valueCtx` 为什么是一条链，最后把 `Background`、`TODO` 和一份检查清单讲清楚。

## 不是每个 WithCancel 都起 goroutine

先补一个容易被误传的细节。

有人一听 `context` 会向下传播取消，就下意识以为：每调用一次 `WithCancel`，标准库是不是都要起一个 goroutine，专门监听父 context？

不是。

Go 1.25.4 的 `propagateCancel` 里，路径很克制。

第一种情况，parent 的 `Done()` 是 nil。

比如 `context.Background()` 这种根 context，本身永远不会取消。既然父级没有取消信号，子级也不需要挂监听逻辑，直接返回。

第二种情况，parent 已经取消。

这时也不用起 goroutine，child 直接跟着取消。

第三种情况，也是标准 context 链最常见的情况：标准库能找到父级内部的 `*cancelCtx`。

此时 child 会被加入 parent 的 `children` map。等父级取消时，父级遍历 children，把取消继续往下传。

这里没有额外 goroutine。

只有在 fallback 场景里，比如 parent 是你自己实现的 `Context`，标准库拿不到内部 `cancelCtx`，也不能走 `AfterFunc` 路径，才会启动一个 goroutine，在 `parent.Done()` 和 `child.Done()` 之间 `select`。

所以“创建很多 context 就一定创建很多 goroutine”这句话不准。

真正该关心的不是这个想象出来的 goroutine 数，而是三件事：你创建了多少生命周期节点，它们什么时候从父节点摘掉，业务 goroutine 有没有真的响应取消。

这也是 context 源码给人的第一层感受：它并不爱搞魔法。

能挂 map 就挂 map，能直接返回就直接返回，实在没法观察父级，才用 goroutine 做桥。

context 的复杂度，很多时候不是标准库制造的，是我们把调用链写浑以后制造的。

## WithValue 不是 map

再看 `WithValue`。

很多人脑子里会把它想成这样：context 里面有一个隐藏 map，`WithValue` 就是往 map 里塞一个键值对。

源码不是这样。

Go 1.25.4 里，`WithValue` 的核心就是返回一个新的 `valueCtx`：

```go
type valueCtx struct {
    context.Context
    key, val any
}
```

每调用一次 `WithValue(parent, key, val)`，标准库就创建一层新的 `valueCtx`。这一层只保存一个 key、一个 val，以及它的 parent。

不是改原来的 context。

不是把值写到同一个 map。

是外面再套一层。

取值时也很直接：先看当前这一层的 key 是否匹配。匹配，返回当前层的 val；不匹配，就沿着 parent 往上找。

![valueCtx 逐层查找路径](/images/go-context-value-is-chain/inline-01.png)

这就是为什么我说它是一条链。

链这个理解，比 map 更接近真实行为，也更能解释三个后果。

第一，查找成本和链深度有关。

普通请求里放一个 request id、trace id、user id，这点成本通常不是问题。不要把它夸大成性能恐吓。

但你得知道，它不是 O(1) map 查找。它是沿着 context 链一层层看。你把 `WithValue` 当参数袋用，链深了，查找路径自然也会变长。

第二，内层同 key 会遮蔽外层。

如果外层已经放了一个 `userIDKey{}`，内层又用同一个 key 放了另一个值，取值时先命中内层。外层还在，但看不见了。

有时这是你想要的覆盖。

更多时候，它会让排查变烦：日志里看到 user id 变了，你得沿着调用链一层层找，到底是哪一层重新包了 context。

第三，value 会被 context 链引用。

你把一个大对象塞进 `Value`，只要这条 context 链还活着，它就可能跟着被保留。短请求里也许看不出问题，但长生命周期 context、异步任务、缓存式保存 context 时，就会把问题放大。

所以 `WithValue` 的问题，通常不只是“慢一点”。

更大的问题是它让依赖变暗。

函数签名变短，不等于依赖变少。

依赖藏进 `ctx.Value` 之后，调用方看不到，review 看不清，排查时也只能靠搜索和猜。

## 能放什么，不能放什么

Go 官方文档对 `WithValue` 的边界说得很窄：只用于跨 API 边界、跨进程边界的 request-scoped data。

这句话要拆开看。

第一，它应该是请求级的。

request id、trace id、认证信息里的 user id、tenant id，这类东西跟着一条请求链路走，合理。

第二，它应该是轻量元数据。

一个小字符串、一个小结构、一个不可变的身份标记，问题不大。大 slice、大对象、数据库连接、复杂配置，不该塞进去。

第三，它应该服务于“跨边界”。

比如日志中间件、追踪系统、鉴权层，需要在多层 API 之间拿到同一个 trace id 或 request id。你不希望每个业务函数都显式传一遍这些横切元数据，这时 `WithValue` 有它的位置。

但业务参数不是这个场景。

下面这种代码，短期看起来省事，长期一定会让人难受：

```go
func CreateOrder(ctx context.Context) error {
    user := ctx.Value(userKey{}).(*User)
    coupon := ctx.Value(couponKey{}).(string)
    db := ctx.Value(dbKey{}).(*sql.DB)
    // ...
}
```

调用方只看到 `CreateOrder(ctx)`，却看不到它真正依赖 user、coupon 和 db。

这不是封装。

这是藏依赖。

更稳的写法，是把真正的业务输入摆到签名里，把 context 留给生命周期和少量请求元数据：

```go
func CreateOrder(ctx context.Context, user User, coupon string) error {
    // db 从结构体依赖或显式参数来，不从 ctx 里偷
    return nil
}
```

`WithValue` 不是参数袋，是请求元数据的边境通行证。

它适合带着 request id 过关，不适合替你搬整个仓库。

## key 也有边界

`WithValue` 还有一个很实际的坑：key 冲突。

源码里要求 key 必须 comparable。更重要的是，注释明确不建议用 string 或其他内置类型，避免不同包之间撞 key。

不要这样写：

```go
ctx = context.WithValue(ctx, "user_id", id)
```

这段代码看起来方便，但任何包都可能也用 `"user_id"`。一旦不同包碰巧用了同一个字符串 key，值就可能互相覆盖，问题很难查。

更稳的做法是定义自己的 key 类型：

```go
type userIDKey struct{}

func WithUserID(ctx context.Context, id int64) context.Context {
    return context.WithValue(ctx, userIDKey{}, id)
}

func UserIDFrom(ctx context.Context) (int64, bool) {
    v, ok := ctx.Value(userIDKey{}).(int64)
    return v, ok
}
```

这里有两个好处。

第一，key 类型只属于当前包，不容易和别的包撞。

第二，取值逻辑被收口到一个函数里。业务代码不用到处写类型断言，也不用散落一堆 `ctx.Value(...)`。

你会发现，越是要谨慎使用的工具，越应该把入口收窄。

如果一个项目里到处都是裸的 `ctx.Value`，通常说明边界已经松了。不是每一处都错，但它们值得被 review。

## Background 和 TODO 行为像，责任不一样

再看两个经常被混着用的函数：`context.Background()` 和 `context.TODO()`。

从行为上看，它们确实很像。

在源码里，二者都基于 `emptyCtx`。没有 deadline，`Done()` 返回 nil，`Err()` 返回 nil，`Value()` 返回 nil。

所以你把它们放进一段代码里，很多时候运行结果没差。

但语义差很多。

`Background()` 是明确的根。

它通常用在 `main`、初始化、测试、入站请求的顶层。意思是：这条调用链从这里开始，我现在没有上游 context。

`TODO()` 是占位。

它表达的是：这里应该有一个合适的 context，但现在还没想清楚，或者周围函数签名还没改完。

这两个东西混用，短期不会炸，长期会污染代码意图。

如果你在业务代码里长期看到 `context.TODO()`，不要只把它当“也能跑”。它其实在告诉后来的人：这里的生命周期还没设计完。

`TODO` 留久了，就不是占位，是债。

这个债不一定马上出事故，但它会让后来的改造变慢。因为没人知道这里到底应该接请求的 parent context，还是应该有独立的后台生命周期。

所以我的建议很简单：

根部用 `Background()`，迁移中才用 `TODO()`；一旦代码进入稳定业务路径，`TODO()` 就应该被消掉。

## context 管的是三件事，不是所有事

到这里，三篇文章其实讲的是同一件事。

第一篇讲：context 不是清洁工。

超时到了，goroutine 还在涨，通常不是 context 没生效，而是业务 goroutine 没有响应取消信号。`Done` 只是通知，退出要靠代码协作。

第二篇讲：`WithTimeout` 的 cancel 不是超时按钮。

timeout 到点会自动发生，但提前收尾不会自动发生。`cancel` 的职责，是释放 context 生命周期链上的资源，停止 timer，解除父子引用。忘 cancel 不等于一定泄漏 goroutine，但一定会让收尾路径不干净。

这一篇讲：`WithValue` 不是 map。

它是一条链。它适合携带少量请求元数据，不适合帮你隐藏业务依赖。

这三件事合起来，才是 context 的设计边界。

它管生命周期。

它管取消信号。

它允许少量请求元数据沿链路传递。

除此之外，它不替你杀 goroutine，不替你关闭资源，不替你设计函数依赖，也不替你把一团业务参数藏得合理。

context 的克制，不是功能少，而是不替你隐藏责任。

## 一份可以直接拿去用的检查清单

下次你 review 一段 Go 代码，看到 `context`，不要只看“有没有传 ctx”。

按下面这份清单扫一遍。

### 1. 看创建路径

谁调用了 `WithCancel`、`WithTimeout`、`WithDeadline`？

返回的 `cancel` 有没有明确 owner？

函数级短生命周期，创建后 `defer cancel()` 通常最稳。循环、fan-out、批处理里，不要把每一轮的 cancel 都拖到外层函数最后；一轮结束，就应该一轮收尾。

### 2. 看响应路径

所有可能长时间等待的地方，有没有把 `ctx.Done()` 放进真正的等待点？

不是函数签名上有 `ctx`，也不是上层传过了 `ctx`，而是 channel、网络、队列、锁、内部循环这些实际阻塞点，能不能因为取消信号退出。

如果 `Done` 只出现在外层，真正耗时的内部调用完全不接 context，那还是没有出口。

### 3. 看 Value 边界

每个 `WithValue` 都问三句：

- 这个值是不是 request-scoped？
- 它是不是轻量元数据？
- 不放 context，函数签名会不会更清楚？

三句里有一句答不上来，就不要急着塞。

request id、trace id、user id、tenant id，通常可以。业务参数、DB handle、logger 配置、大对象、可选参数，通常不该放。

### 4. 看 key 类型

有没有直接用 string 当 key？

如果有，优先改成包内自定义 key 类型，并用辅助函数收口读写。

不要让业务代码散落一堆 `ctx.Value("xxx")`。这会让冲突、类型断言和默认值处理都变成隐形风险。

### 5. 看根 context

`Background()` 是否只出现在真正的根部？

`TODO()` 是否只是临时迁移？

如果稳定业务代码里还留着 `TODO()`，就要问清楚：这里的上游 context 到底是谁？为什么现在还没传进来？

### 6. 看 pprof 和 vet 的边界

`go vet` 能抓一部分 lostcancel，但抓不住业务 goroutine 有没有听见 `Done`。

pprof 能告诉你 goroutine 停在哪里，但不能直接替你判定根因。

工具给线索，路径给答案。

## 把 context 用窄一点

很多 Go 代码变难维护，不是因为没有用 context，而是因为把 context 用得太宽。

取消也交给它。

收尾也指望它。

业务参数也塞给它。

依赖关系也藏进它。

最后函数签名看起来干净，调用链却越来越黑。

这不是 context 的问题。

这是我们把一个克制的工具，当成了万能容器。

如果这个系列只带走一句话，我希望是这句：

context 负责传递边界，不负责替你承担责任。

上篇的责任，是取消之后谁退出。

中篇的责任，是创建之后谁收尾。

这一篇的责任，是依赖应该放在签名里，还是只能作为请求元数据沿链路传递。

把这三条分清，`context` 就不玄了。

它只是在逼你把调用链写明白。

如果你正在维护 Go 服务，可以从今天开始做一个很小的动作：打开项目，搜索 `WithTimeout`、`WithCancel`、`WithValue`、`context.TODO()`。

不用一口气重构。

先给每一处标上责任：谁创建，谁取消，谁响应，谁取值，谁能解释这个值为什么应该在 context 里。

能解释清楚的，留下。

解释不清的，改回显式代码。

这个系列三篇，到这里就收住。后面我会继续写 Go 并发和后端工程里那些“看起来是语法问题，实际是设计边界问题”的坑。如果你也在写 Go 服务，建议关注起来。很多线上事故，最后都不是输在某个 API 不会用，而是输在责任边界没写清楚。