---
title: "OOMKilled 了别先调参数，第一步是看谁杀了你"
description: "Go 服务被 OOMKilled 后，别急着改 GOGC 或 GOMEMLIMIT。先用 kubectl describe 确认事故事实，再用两次 pprof diff 拆清 inuse、alloc、objects 和 RSS 之间的账。"
date: 2026-05-28T10:37:00+08:00
draft: false
author: "任博"
tags: ["Go", "GC", "pprof", "Kubernetes", "OOM", "性能排查"]
categories: ["技术实战"]
cover: "/images/go-gc-oom-first-step/cover.png"
toc: true
---

服务又被 OOMKilled 了。

Pod 刚重启，群里已经开始报菜名：把 `GOGC` 调低一点，给 `GOMEMLIMIT` 设个值，容器 limit 再加 1Gi，顺手翻一下 GC 日志。

这些动作不一定错，但顺序错了。

你现在看到的是“尸体”，不是“凶手”。OOMKilled 只说明容器被内核杀掉了，不说明是谁把内存账花爆了。可能是 Go heap 真的在涨，可能是 goroutine 泄漏把对象挂住了，可能是 RSS 里有 cgo、mmap、tmpfs，也可能只是容器 limit 给得太紧。

很多 Go 内存事故，第一步不是调 GC，而是先把账本拿出来。

**OOMKilled 是结果，不是根因。先查谁杀了你，再谈怎么救。**

## 第一步：先确认谁杀了你

线上内存告警一来，最危险的动作不是慢，而是快。

你一快，就会把“容器被杀”理解成“Go heap 有问题”；把“GC CPU 变高”理解成“GC 参数不合适”；把“RSS 比 heap 大”理解成“Go 没把内存还给系统”。

这些判断都太早。

先看 Kubernetes 给你的第一现场：

```bash
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
kubectl top pod <pod> -n <ns> --containers
kubectl get pod <pod> -n <ns> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

你要先回答几件事：

```text
Reason 是不是 OOMKilled？
Exit Code 是不是 137？
哪个 container 被杀了？
容器 memory limit 是多少？
working set 有没有持续贴近 limit？
重启前 p99、QPS、GC CPU、goroutine 数有没有一起变化？
```

这一步很土，但它能防止你一上来就跑偏。

应用日志经常来不及写完，Pod 已经没了。`kubectl describe pod` 里的 Last State、Reason、Exit Code、Events，反而是最可靠的事故入口。它不能告诉你代码哪一行有问题，但能告诉你：这是不是一个真实的 OOMKilled，发生在哪个容器，撞上的边界是什么。

如果这个问题没确认，后面所有 pprof、gctrace、调参，都可能只是对着空气用力。

我见过不少排查，一开始就把方向定成“GC 有问题”。后来翻 `describe` 才发现，被杀的是 sidecar；或者业务容器确实被杀了，但 limit 只有稳态均值上面一点点；再或者重启前 heap 没怎么涨，container working set 却贴着上限走。

这几种事故，处理方式完全不同。

如果是 sidecar 撑爆了共享 Pod 内存，你改 `GOGC` 没用；如果是 limit 给得太薄，你只盯 pprof 也会漏掉配置问题；如果 heap 不高但 RSS 高，你更不能只拿一张 heap profile 当结论。

第一步要做的事很朴素：别急着给事故起名字，先确认事故到底发生在谁身上。

## 第二步：抓两次 profile，不要迷信一次 top

如果服务还活着，先留证据。

生产环境不要把 pprof 裸露在公网。更稳的方式是只监听 localhost，或者走内网、鉴权 sidecar、临时端口转发。

服务里最小开启方式大概是这样：

```go
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("127.0.0.1:6060", nil))
    }()
}
```

现场先抓第一组：

```bash
kubectl port-forward -n <ns> pod/<pod> 6060:6060

curl -s -o heap_t0.pb.gz \
  'http://127.0.0.1:6060/debug/pprof/heap?gc=1'

curl -s 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=2' \
  > goroutine_t0.txt
```

等一个业务周期，或者等内存继续往上走，再抓第二组：

```bash
sleep 300

curl -s -o heap_t1.pb.gz \
  'http://127.0.0.1:6060/debug/pprof/heap?gc=1'

curl -s 'http://127.0.0.1:6060/debug/pprof/goroutine?debug=2' \
  > goroutine_t1.txt
```

然后看差异：

```bash
go tool pprof -top -inuse_space \
  -diff_base heap_t0.pb.gz ./app heap_t1.pb.gz

go tool pprof -top -inuse_objects \
  -diff_base heap_t0.pb.gz ./app heap_t1.pb.gz

go tool pprof -http=:0 \
  -diff_base heap_t0.pb.gz ./app heap_t1.pb.gz
```

`-diff_base` 的价值，不是告诉你“谁最大”，而是告诉你“谁在变大”。

这两个问题差别很大。

一次 `top` 里排第一的函数，可能只是它负责分配大块临时 buffer。它大，不等于它泄漏。比如批量 JSON、压缩、图片处理、日志拼接，都会在某些窗口里看起来很显眼，但请求结束以后对象就该走了。

泄漏更像一条趋势：同一类对象在两个时间点之间持续正增长，业务回落以后也不回落。

**泄漏排查靠趋势，不靠截图。**

只看一张 profile 截图，很容易把“正常的大户”当成“真正的凶手”。两次采样加 diff，才是在问正确的问题：过去这几分钟，到底是谁多占了内存。

这也是为什么事故现场要尽量保存原始 profile 文件，而不是只在群里发一张截图。截图只能证明你当时看到了什么，profile 文件还能让后面的人换一个视角重新分析：看 `list`，看调用链，看 objects，看 diff。

我建议现场至少留这几类文件：

```text
heap_t0.pb.gz / heap_t1.pb.gz
goroutine_t0.txt / goroutine_t1.txt
kubectl describe pod 输出
事故窗口里的关键监控截图或查询条件
本次判断结论和排除项
```

这不是流程洁癖。

内存问题经常复发，而且复发时未必是同一个根因。上一次是全局 map，只增不删；下一次可能是 RSS 高但 heap 不高；再下一次可能是 goroutine 卡住，把一堆本该释放的对象挂在栈上。

如果只留下截图，下一次大家只能重新猜。如果留下 profile 和判断过程，下一次至少能快速回答：这次是不是同一个问题。

内存事故最怕事后复盘只剩一句话：“当时好像是某个函数很大。”

## 第三步：别把 inuse、alloc、objects 混成一锅粥

pprof 里最常见的误判，是把几个视角混着用。

```bash
# 当前还占着堆内存：看泄漏、常驻缓存、全局引用
go tool pprof -top -inuse_space ./app heap.pb.gz

# 从启动以来累计分配最多：看分配速率、GC 为什么忙
go tool pprof -top -alloc_space ./app heap.pb.gz

# 当前存活对象数量：看小对象爆炸、map entry 异常
go tool pprof -top -inuse_objects ./app heap.pb.gz
```

`inuse_space` 回答的是：现在谁还占着堆。

它适合看无界缓存、全局 map、请求对象被错误持有。比如某个 `map[string]*Session` 只写不删，或者某个全局 slice 一直 append，从 diff 里通常能看出来。

`alloc_space` 回答的是：从启动以来谁分配得最多。

它适合看分配速率，不适合直接定性泄漏。批量 JSON、压缩、图片处理、日志结构化，都可能让 `alloc_space` 很高，但对象活得很短。这个时候问题可能是 GC 被迫频繁工作，不一定是对象没有释放。

`inuse_objects` 回答的是：现在活着的对象数量有多少。

有些问题不是单个对象大，而是数量爆炸。map entry、短小结构体、闭包、channel 里堆积的消息，单个看不吓人，数量一上来，GC 扫描成本也会被拖高。

更稳的判断顺序可以写成这样：

```text
inuse_space 持续涨：看泄漏、无界缓存、全局引用。
alloc_space 很高但 inuse 平稳：看分配速率和 GC CPU。
inuse_objects 持续涨：看小对象数量、map entry、队列堆积。
goroutine 数持续涨：先查 goroutine 泄漏，再看它持有什么对象。
```

不要看到 `alloc_space` 第一名，就说它泄漏；也不要看到 heap 不高，就说不是内存问题。你要先问：我现在看的这个指标，到底回答的是哪一个问题。

这个问题一问，很多争论会马上停下来。

有人说“这个函数分配最多”，你要追问：它是累计分配最多，还是现在还活着最多？有人说“heap profile 看起来不大”，你要追问：那 container working set 和 Go managed memory 呢？有人说“goroutine 多一点没事”，你要追问：这些 goroutine 的栈上有没有挂着 request、buffer、session？

排查不是找一个最大数字，然后宣布破案。

排查是把每个数字放回它自己的账本里。

## 一个很容易误判的现场

假设事故窗口里你看到的是这样一组数字：

```text
container working set：920MiB / 1GiB
heap live：430MiB
Go managed memory：680MiB
goroutine：800 -> 11000
GC CPU：5% -> 28%
```

很多人会盯着 `heap live` 那个 430MiB，然后说：heap 才这么大，应该不是 Go 的问题。也有人会盯着 GC CPU 28%，马上说：GC 太重，先调参数。

这两种判断都不够。

heap live 只有 430MiB，确实说明当前存活堆不是 920MiB。但 goroutine 从 800 涨到 11000，这个信号不能跳过。goroutine 本身有栈，更麻烦的是它可能在栈上挂着 request、response、session、buffer。你在 heap profile 里看到的对象，未必是“主动泄漏”的缓存，也可能是被一堆卡住的 goroutine 间接留住。

这时更像一个排查分叉：

```text
heap live 持续涨：继续追 inuse_space diff。
goroutine 数持续涨：先看 goroutine profile。
RSS 远高于 Go managed memory：再查 native / mmap / tmpfs。
```

如果 goroutine profile 里大量栈都卡在同一个 `chan send`，创建点来自同一个请求处理函数，方向就很清楚了：这不是 GC 不努力，而是生命周期没闭合。

比如请求超时了，调用方已经走了，下游 channel 还在等发送；或者 session 放进 map 后，超时路径没有 delete。GC 只能回收不可达对象。对象还被 goroutine 或全局 map 挂着，它就不能删。

这种情况下，你把 `GOGC` 调低，只会让 GC 更频繁地扫描这堆“还活着”的对象。你把 limit 加大，事故可能晚点来，但根因还在。你真正要修的是：让 goroutine 有退出路径，让 map 有删除策略，让缓存有 TTL 或 size limit。

**GC 不是清洁工，它不会替你删除还被引用的对象。**

## 第四步：RSS 比 heap 大，不代表 GC 坏了

容器里最容易吵起来的地方，是 RSS 和 heap 对不上。

你在 pprof 里看到 live heap 只有 430MiB，监控里 container working set 已经 920MiB。群里很容易出现一句话：Go GC 没把内存还回去。

有可能，但不能这么下结论。

Go runtime 里有几组字段值得一起看：

```go
var m runtime.MemStats
runtime.ReadMemStats(&m)

fmt.Println("HeapAlloc", m.HeapAlloc)
fmt.Println("HeapInuse", m.HeapInuse)
fmt.Println("HeapIdle", m.HeapIdle)
fmt.Println("HeapReleased", m.HeapReleased)
fmt.Println("NextGC", m.NextGC)
```

几个字段先记住含义：

```text
HeapAlloc：已分配的堆对象字节数。
HeapInuse：正在使用的 span 字节数。
HeapIdle：空闲 span，可复用，也可能归还 OS。
HeapReleased：已经归还给 OS 的物理内存。
HeapIdle - HeapReleased：runtime 保留、可归还但尚未归还的部分。
NextGC：下一轮 GC 的目标 heap size。
```

如果 `HeapIdle - HeapReleased` 很大，通常说明最近有过内存尖峰，runtime 还保留了一部分页，准备复用或稍后归还。这个时候 heap profile 下降了，RSS 不一定马上跟着下降。

更关键的是，RSS 本来就不只包含 Go heap。

它还可能包含 goroutine stack、runtime metadata、二进制映射、cgo 分配、`syscall.Mmap`、压缩库或图片库的原生内存、文件映射、日志 buffer，甚至 memory-backed `emptyDir`。

所以 OOM 排查时，至少要把三条线放在一起：

```text
Go heap live
Go managed memory = Sys - HeapReleased
container working set / RSS
```

Go 的 `GOMEMLIMIT` 也不是“整个进程 RSS 的硬上限”。它约束的是 Go runtime 管理的内存，不包括二进制映射、其他语言管理的内存，以及操作系统代表 Go 程序持有的一些内存。

这就是为什么 `GOMEMLIMIT` 有用，但不是免死金牌。它管不到所有非 Go 内存，也挡不住一个已经被业务引用牢牢挂住的 live heap。

![OOMKilled 排查时先拆三本账](/images/go-gc-oom-first-step/inline-01.png)

这张图要表达的不是“哪个指标更高级”，而是：同一个内存事故，Kubernetes、Go runtime、操作系统看到的账本不一样。

Kubernetes 关心容器有没有撞边界。Go runtime 关心自己管理的内存。操作系统看到的是进程驻留内存和 native 分配。你用其中一本账去解释另外两本账，十有八九会误判。

还有一个细节容易被忽略：容器里的 working set 不是 Go runtime 指标，heap profile 也不是容器指标。它们可以相互解释，但不能相互替代。

如果 `HeapIdle - HeapReleased` 很大，你可以怀疑最近有过堆尖峰，runtime 还没把页完全还给操作系统；如果 Go managed memory 接近 working set，说明主要账可能还在 Go runtime 里；如果 working set 明显大于 Go managed memory，就要把眼睛从 GC 上移开，去看 cgo、mmap、文件映射、tmpfs、sidecar 这些账。

这一步的目标不是马上定位代码行，而是避免把错误的问题交给错误的工具。

pprof 很强，但它解释不了所有 RSS。

GC 很强，但它管不了所有内存。

## 一条更稳的排查顺序

把前面几步串起来，线上不要按“谁声音大”排查，按这条顺序走：

```text
1. kubectl describe：确认 OOMKilled、Exit Code、container、limit。
2. 对齐监控：container working set、heap live、Go managed memory、goroutine count。
3. 抓两次 heap / goroutine profile，中间隔一个业务周期。
4. 用 diff_base 看增长，而不是只看一次 top。
5. 分清 inuse_space、alloc_space、inuse_objects 分别回答什么。
6. 如果 heap 不高但 RSS 高，再查 cgo、mmap、stack、tmpfs、sidecar。
```

这条路线看起来慢，但它比“先调一下试试”快得多。

因为它每一步都有产出：第一步确认事故边界，第二步保住现场证据，第三步找增长趋势，第四步拆清指标口径。哪怕最后发现不是 Go heap 的问题，这条路线也没有白走。你至少排除了一个方向，而且知道下一步应该去看 native 内存、Pod 配置，还是业务生命周期。

真正浪费时间的，反而是没有证据的调参。

调低 `GOGC` 以后服务还会不会 OOM？不知道。加大 limit 以后问题是不是消失了？也不知道。`GOMEMLIMIT` 设上以后，是挡住了事故，还是把延迟拖坏了？如果没有前面的账本，你只能靠感觉。

感觉在事故现场很贵。

因为调参很容易制造错觉。`GOGC` 变了，GC 频率变了；`GOMEMLIMIT` 设了，GC 更努力了；limit 加大了，服务暂时不死了。但这些动作没有回答核心问题：谁在持有对象，谁在疯狂分配，谁把 RSS 撑起来了。

参数可以救火，但不能替你破案。

真正值得留下来的，是一套能复用的现场动作：谁被杀了，边界是多少，profile 怎么抓，diff 怎么看，三本账怎么对齐。下一次 OOM 再来，你不需要靠某个高手临场发挥，也不需要在群里猜半小时。

更重要的是，这套动作能让团队少吵很多无效的架。以前大家争的是“是不是 GC 的锅”，现在可以先问四个更具体的问题：heap 有没有涨，alloc 有没有高，goroutine 有没有漏，RSS 和 Go managed memory 差在哪。问题一具体，排查就会从情绪变回工程。工程排查最怕的不是问题复杂，而是连问题口径都没对齐。

你只要把这条路线重新跑一遍。它不保证每次都快，但能保证你不会从第一分钟就跑错方向，少走弯路。

下一篇我们再聊：什么时候可以动 `GOGC` 和 `GOMEMLIMIT`，以及为什么很多所谓“GC 调优”，其实是在用参数掩盖生命周期问题。

如果你也在排查 Go 服务的 OOM，把这篇先收藏。下次群里有人一上来就说“调 GOGC”，你可以先把这句话丢给他：

**别急着救火，先确认火从哪烧起来。**
