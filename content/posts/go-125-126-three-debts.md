---
title: "Go 1.25/1.26 正在替团队还三类债：性能、写法、测试"
description: "这两版 Go 最值得看的地方，不是又多了几个语法点，而是运行时、go fix、synctest 和 gopls MCP 正在把团队长期积下来的性能债、写法债、测试债慢慢搬进工具链里。"
date: 2026-06-25T10:30:00+08:00
draft: false
author: "任博"
tags: ["Go", "Go 1.25", "Go 1.26", "GC", "go fix", "synctest", "MCP", "工程实践"]
categories: ["技术实战"]
cover: "/images/go-125-126-three-debts/cover.svg"
toc: true
---

很多团队升级 Go 版本，只做一件事：把 CI 跑绿。

跑绿以后就结束了。代码没动，测试没动，工具链没动。版本号从 1.22 变成 1.26，心里安慰自己：至少我们没落后太多。

但 Go 1.25 / 1.26 这两版，我不太建议这样看。

它们最值得关注的地方，不是又多了几个语法点，而是 Go 开始把一些团队长期积着的工程债，往运行时、工具链和标准库里搬。

性能债，交给 Green Tea GC 先还一部分。

写法债，交给重写后的 `go fix` 先还一部分。

测试债，交给 `testing/synctest` 先还一部分。

这不是“新版 Go 很厉害”的发布稿口吻。真正值得你关心的是：有些债，以前只能靠团队纪律、代码评审和人肉迁移，现在开始有工具接手了。

![Go 1.25/1.26 三类工程债](/images/go-125-126-three-debts/cover.svg)

## 性能债：别把 GC overhead 当成业务性能

Go 1.26 默认启用了 Green Tea GC。

如果只看传播口径，这个点很容易被写歪：最高 50%、零代码修改、重新编译就提速。听起来很爽，但这种说法最容易误导人。

Green Tea 真正改善的是 GC 这部分成本，不是你的业务整体性能直接涨 50%。

Go 官方更稳的说法是：对重度使用 GC 的真实程序，预期 GC overhead 降低 10% 到 40%。在较新的 amd64 CPU 上，因为可以用向量指令扫描小对象，还可能有额外的 GC overhead 改进。

这句话里最重要的词不是“40%”，而是“GC overhead”。

如果你的服务总 CPU 里，GC 只占 5%，那 GC 成本降一截，整体 CPU 也不会像广告图一样起飞。反过来，如果你的服务分配很猛、对象很多、GC 长期占着一块明显成本，那 Green Tea 就更可能被你看见。

所以升级 Go 1.26 后，不要只盯 QPS。

你更应该看这些东西：

```bash
# 看 CPU profile 里 GC 相关成本
go tool pprof http://localhost:6060/debug/pprof/profile

# 看堆分配和对象形态
go tool pprof http://localhost:6060/debug/pprof/heap

# 或者接入 runtime/metrics，看 GC CPU、alloc rate、heap 变化
```

如果你还在 Go 1.25 上，也可以用实验开关试 Green Tea：

```bash
GOEXPERIMENT=greenteagc go test ./...
GOEXPERIMENT=greenteagc go build ./...
```

到了 Go 1.26，它默认启用。如果线上遇到异常，官方也留了临时退路：

```bash
GOEXPERIMENT=nogreenteagc go build ./...
```

但这个退路不应该被当成长期配置。官方已经说明，关闭开关预计会在 Go 1.27 移除。真遇到问题，应该回到 profile、trace 和 issue，而不是默默把新 GC 永久关掉。

这件事给团队的提醒很简单：升级 Go 版本不是跑一次 benchmark 然后截图。你得先知道自己现在的 GC 债长什么样。

没有基线，所有“提升”都是情绪价值。

## 写法债：`go fix` 不再只是老古董

很多人对 `go fix` 的印象还停在很早以前：Go 1 初期用来做兼容迁移的老工具，平时基本想不起来。

Go 1.26 之后，这个印象该改了。

新的 `go fix` 被完全重写，变成 modernizers 的入口。它底层基于和 `go vet` 同一套 Go analysis framework：`vet` 更偏发现问题，`go fix` 更偏生成修改。

这件事的意义不在于“官方又多了一个命令”。

它真正解决的是大代码库里最难看的那种债：写法没错，但越来越旧；API 没坏，但迁移没人有空做；新版本已经有更好的标准库写法，但老项目里还堆着旧习惯。

以前这类债靠什么还？

靠文档、靠 code review、靠某个认真同事顺手改一点。

问题是，人会累。更现实的是，AI coding assistant 也会复制旧语料里的旧写法。你让它生成 Go 代码，它很可能把训练集中高频出现的老习惯也带回来。

所以我反而觉得 `go fix` 在 AI 时代更重要了。

AI 写代码越多，自动现代化工具越重要。

官方建议的用法很朴素：升级到新 Go toolchain 后，在干净的 git 状态下跑它。

```bash
# 先预览，不直接改
go fix -diff ./...

# 确认后再应用
go fix ./...
```

如果项目有多平台构建标签，不要只在当前机器跑一次：

```bash
GOOS=linux   GOARCH=amd64 go fix ./...
GOOS=darwin  GOARCH=arm64 go fix ./...
GOOS=windows GOARCH=amd64 go fix ./...
```

这和 `go test ./...` 很像：一次运行只覆盖一个 build configuration。你的代码如果靠 build tags 分平台，工具也需要分平台看。

还有一个更有意思的方向：`//go:fix inline`。

它让库作者可以把某些 API 迁移规则写进源码。调用方升级后运行 `go fix`，工具就能把旧调用改成新的写法。典型例子是把旧函数调用 inline 成新函数调用，而不是让每个使用者读迁移文档、自己开 PR。

这不是黑盒重构机器人。

官方也没有说跑完就万事大吉。`go fix` 的修改目标是语义上不改变行为，但真实项目里你仍然应该 review、跑测试、分批合并。

它更像一把可审查、可回滚的清洁工具。

真正的变化是：以后代码老化不再完全靠人肉补救。团队可以把“升级 Go 后跑 modernizers”纳入固定流程，而不是等哪天重构周再说。

## 测试债：并发测试最怕的不是 goroutine

Go 项目里有一种测试，看起来很正常，其实很虚。

```go
go func() {
    doSomethingAsync()
}()

time.Sleep(100 * time.Millisecond)

if !done {
    t.Fatal("not done")
}
```

这段代码最大的问题不是丑。

它的问题是你根本不知道自己在等什么。你只是赌 100ms 足够。机器空闲时它过，CI 抖一下它挂；今天过，明天换个负载又不过。

更麻烦的是反向断言：你想证明某件事“还没有发生”。靠 `Sleep` 很难证明这个东西。你只能说，睡了这么久它还没发生。但它是不是下一毫秒就发生？你不知道。

这就是 `testing/synctest` 要解决的痛点。

它在 Go 1.24 作为实验包引入，Go 1.25 进入标准库 GA。它的核心不是把生产代码变同步，而是在测试里创建一个 isolated bubble。bubble 里的 goroutine、timer、channel、fake clock 都受它管理。

时间也不再按真实世界慢慢走。

在 bubble 里，时间只有在所有 goroutine 都 durably blocked 的时候才推进。`synctest.Wait()` 则会等到 bubble 内其他 goroutine 都进入可判断的阻塞状态。

这听起来抽象，看一个官方文档风格的例子就明白：

```go
func TestContextAfterFunc(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        ctx, cancel := context.WithCancel(t.Context())
        called := false
        context.AfterFunc(ctx, func() { called = true })

        synctest.Wait()
        if called {
            t.Fatal("called before cancel")
        }

        cancel()
        synctest.Wait()
        if !called {
            t.Fatal("not called after cancel")
        }
    })
}
```

这段测试里，真正被替换掉的不是某个 API，而是一种测试心态。

以前你在猜调度。现在你在等系统进入一个可以判断的状态。

并发测试最怕的不是 goroutine，而是你以为睡够了。

当然，synctest 也有边界。它适合测试 timer、context deadline、callback、channel、WaitGroup 这类异步行为，但不应该被想象成“所有并发测试确定化”。官方文档也说得很清楚：网络 I/O、系统调用、外部进程不属于 durably blocking。测试应该尽量自包含，不要和 bubble 外的 goroutine 或真实网络搅在一起。

所以升级 Go 1.25/1.26 后，团队最应该先做的不是“全量改测试”。

更现实的做法是扫一遍代码库：哪些测试里有 `time.Sleep`？哪些 sleep 是为了等 goroutine、timer、context 或 background callback？先挑最 flaky、最拖 CI 时间的那几处改。

你不用一次性还清测试债。

但至少别再把 `Sleep(100ms)` 当成并发测试的信仰。

## AI 工具链：MCP 不是热闹，是入口

Go 1.25 / 1.26 这条线里，还有一个容易被低估的方向：gopls MCP server 和官方 MCP Go SDK。

我不想把这部分写成“Go 全面拥抱 AI”的热闹话。现在这么写太容易虚。

更准确一点说：Go 生态开始把语言服务器、分析框架、MCP SDK 这些能力，接到 AI assistant 可以调用的工具层里。

gopls v0.20.0 加入了实验性的 MCP server。官方文档说得很克制：它暴露的是 gopls 功能的一个 subset，不是整个 IDE 被稳定搬给 AI。

用法也很直接。

```bash
# 安装新版 gopls
go install golang.org/x/tools/gopls@latest

# detached 模式，通过 stdio 跑 MCP server
gopls mcp

# 给模型导出使用说明
gopls mcp -instructions > /path/to/contextFile.md

# Claude Code 接入
claude mcp add gopls -- gopls mcp
```

attached 模式则可以通过：

```bash
gopls serve -mcp.listen=localhost:8092
```

两种模式的差别很关键：attached 能共享当前 LSP session 的内存状态，看到未保存 buffer；detached 更像一个独立的 headless gopls，只看磁盘上保存过的文件。

这对 Go 工程师意味着什么？

未来 AI 写 Go 代码，差距不会只在 prompt。更大的差距在于它能不能接入类型信息、引用关系、诊断、重构建议这些结构化工具。

MCP 的价值，是把 gopls 的脑子接给 AI。

但安全边界也要一起说。gopls MCP 会读文件、运行 `go` 命令加载包信息、写缓存，甚至在你开启 Go telemetry 时上传遥测。它不是恶意工具，但放进 AI 工作流以后，就不能再用“反正只是问问模型”的心态对待。

我更建议先在非敏感 repo 里试 detached 模式，确认团队的 assistant、权限和审查流程能接住，再考虑更深的接入。

## 升级后别只跑 CI，按这张清单还债

如果你现在还在 Go 1.22 / 1.23，我不建议你把升级 Go 1.25 / 1.26 理解成“追新”。

更好的理解是：借这次升级，把以前没人愿意系统处理的债，拿出来还一轮。

可以按这个顺序做：

1. 先给 GC 压力建基线。看 pprof、trace、runtime metrics，确认服务是不是 GC-heavy。升级后对比 GC CPU、alloc rate、heap profile，不要只看 QPS。

2. 从干净 git 状态跑 `go fix -diff ./...`。先看 diff，再分批应用。多平台项目按 `GOOS` / `GOARCH` 多跑几次。

3. 扫测试里的 `time.Sleep`。优先改那些为了等异步行为、经常 flaky、拖慢 CI 的测试，尝试用 `testing/synctest` 替换“猜时间”。

4. 如果团队已经在用 Claude Code、Gemini CLI 这类 assistant，拿一个非敏感 Go repo 试 gopls MCP detached 模式。别一上来就接生产核心仓库。

5. 如果你维护公共库，关注 `//go:fix inline`。以后 API 迁移不一定只能靠 README 提醒用户，也可以把迁移规则交给工具。

Go 1.25 / 1.26 的重点，不是让你写得更花。

它更像是在提醒团队：语言本身稳定，不代表工程实践可以停在原地。运行时在替你省一部分性能成本，工具链在替你清一部分旧写法，标准库在替你减少一部分 flaky 测试。

剩下那部分，还是要你自己还。

但至少这一次，Go 不只是给了新版本号。它给了几把能真的用起来的铲子。