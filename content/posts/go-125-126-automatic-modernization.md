---
title: "你的旧 Go 代码，Go 1.26 能一键救回来"
date: 2026-07-13
description: "Go 1.25/1.26 不是小版本升级，而是工具链历史的一次转折——你的代码不再需要你手动维护了。gofix modernizers、Green Tea GC、testing/synctest，三个变化让你少写一半重构工单。"
slug: go-125-126-automatic-modernization
categories:
  - golang
  - toolchain
tags:
  - go-1.25
  - go-1.26
  - gofix
  - green-tea-gc
  - synctest
  - modernization
cover: "/images/go-125-126-automatic-modernization/cover.png"
---

你有没有这种代码——

```go
for i := 0; i < n; i++ {
    ...
}
```

Go 1.22 之后可以写成 `for range n`。你知道，但三年了，你还没改。

或者这种——

```go
eq := strings.IndexByte(pair, '=')
result[pair[:eq]] = pair[1+eq:]
```

Go 1.18 就有 `strings.Cut` 了。你知道怎么用，但一直没动手。

我猜你的项目里不止一处这种「知道有更好的写法，但优先级排不上号」的代码。你甚至列过 Refactoring 工单，然后排到了下周，再然后就没有然后了。

这不是你的问题。这是大多数 Go 项目的真实状态：

**知道该改，但没人专门花时间去改。**

但从 Go 1.25 开始，情况变了。不是因为你变勤快了，而是工具链开始替你动手了。

---

## 真正的主角不是新特性，是 `go fix`

每到一个新版本，我们习惯先把 release notes 扫一遍，看有没有好用的新语法、新标准库。

但 Go 1.25 和 1.26 真正重要的变化，不在新特性本身，而在一个你可能忽略的子命令：

```bash
$ go fix ./...
```

`go fix` 不是什么新东西。它从 Go 1 就有了，以前只是偶尔用来修一下 build tag 格式之类的小补丁。但在 Go 1.26 里，它被彻底重写了——底层不再是独立代码，而是和 `go vet` 共用了同一个分析框架。

**`go fix` 现在有一套标准化的、可扩展的分析器体系**。每个分析器专门处理一类代码现代化转换，然后在 `go fix ./...` 下批量、安全地执行。

截至 Go 1.26，官方已经内置了 **超过 20 个现代化分析器**（modernizers）。你可以用这个命令看到全部：

```bash
$ go tool fix help
```

输出里剪几个例子：

| 分析器 | 功能 |
|--------|------|
| `any` | 把 `interface{}` 替换成 `any` |
| `minmax` | 把 `if x < 0 { x = 0 }` 替换成 `min(max(...))` |
| `rangeint` | 把 `for i := 0; i < n; i++` 替换成 `for range n` |
| `stringscut` | 把 `strings.Index(s, ":")` 替换成 `strings.Cut` |
| `mapsloop` | 把手写 map 循环替换成 `maps.Keys` 等泛型函数 |
| `forvar` | 删除 Go 1.22 之后多余的 `x := x` 重声明 |
| `fmtappendf` | 把 `[]byte(fmt.Sprintf)` 替换成 `fmt.Appendf` |
| `inline` | 基于 `//go:fix inline` 注解的源级内联 |

你跑一遍 `go fix ./...`，这些分析器依次生效，相当于一次批量完成两年来你「知道该改但一直没改」的所有代码现代化。

![go fix 自动化流水线](/images/go-125-126-automatic-modernization/inline-01-gofix-pipeline.png)

且因为底层是同一个分析框架，分析器之间的 **联动效果** 尤其值得一说。

### 联动修复：跑一遍不够就跑两遍

这是 Go 开发组自己都没想到会这么好用的效果。

有一段代码：

```go
x := f()
if x < 0 {
    x = 0
}
if x > 100 {
    x = 100
}
```

第一次跑 `go fix`，`minmax` 分析器发现第一个 `if x < 0` 可以用 `max` 替换。改完之后，再看第二个 `if x > 100`，这时候 `minmax` 又发现可以进一步用 `min` 缩到一行。

最终结果：

```go
x := min(max(f(), 0), 100)
```

同一个分析器按先后顺序发现两次机会。如果按传统人肉方式，你可能改了一半就停了，不会意识到还有第二层可优化。

另一个跨分析器的例子：`stringsbuilder` 检测到循环里用 `+=` 拼接字符串（典型的二次复杂度 bug），先把它改成 `strings.Builder`。改完之后，同文件里又发现了 `s.WriteString(fmt.Sprintf(...))`，`fmtappendf` 分析器接着把它优化成 `fmt.Fprintf(&s, ...)`。

Go 团队的建议是：**跑两次 `go fix`**，就能达到一个不错的固定点。

### `//go:fix inline`：让包作者来定义迁移规则

上面那些 modernizers 是 Go 官方维护的，针对的是语言和标准库的特性。但 Go 1.26 还做了一个更骚的操作：

**你可以在自己的函数上加一条 `//go:fix inline` 注解，让 `go fix` 自动化你的 API 迁移。**

比如你有一个弃用的旧函数，它的实现已经变成对新函数的一行调用了：

```go
// Deprecated: 用 os.ReadFile 代替
//go:fix inline
func ReadFile(filename string) ([]byte, error) {
    return os.ReadFile(filename)
}
```

用户跑 `go fix` 时，所有对 `ioutil.ReadFile` 的调用会被自动替换成 `os.ReadFile`，连 import 都会自动切换。

这能处理多复杂的情况？给你看一个反直觉的例子：

```go
// oldmath.Sub(y, x) 的参数顺序是反的
//go:fix inline
func Sub(y, x int) int {
    return newmath.Sub(x, y)
}
```

调用方写的是 `oldmath.Sub(1, 10)`，内联后的结果会变成 `newmath.Sub(10, 1)`——参数自动正过来了。

这个能力的想象空间很大。你在公司内部维护了一套公共库，某天要重构 API 签名。以前可能要写一个迁移脚本、群发邮件、挨个项目提 MR。现在？一边加 `//go:fix inline` 注解，一边在 release notes 里加一行「跑 go fix ./...」。剩下的交给机器。

Google 内部的数据是：**通过这个机制已经自动完成超过 18,000 个代码变更**。而且这只是开始。

---

## testing/synctest：并发测试不再 flaky

如果你写过 Go 并发测试，你一定遇到过这个选择：

**要慢，还是要 flaky？**

比如测试 `context.WithDeadline`：

```go
func TestWithDeadline(t *testing.T) {
    deadline := time.Now().Add(1 * time.Second)
    ctx, _ := context.WithDeadline(t.Context(), deadline)
    time.Sleep(time.Until(deadline) + 100*time.Millisecond)
    if err := ctx.Err(); err != context.DeadlineExceeded {
        t.Fatalf("not canceled after deadline")
    }
}
```

等 1 秒是慢，等 100ms 是 flaky。你永远没法又可靠又快到毫秒级。

`testing/synctest` 从 Go 1.24 开始作为实验特性引入，Go 1.25 正式 GA，彻底解决了这个问题。它的设计叫 **bubble**（气泡）：

气泡内的 goroutine 使用合成时间。你在气泡里把时间往前拨 1 秒，这个操作会 **同步触发** 所有等待时间的 goroutine，不会有真实的睡眠。

效果是什么？上面的测试可以压缩到几毫秒执行，并且完全确定：

```go
func TestWithDeadline(t *testing.T) {
    ctx, _ := context.WithDeadline(t.Context(), ...)
    // 在 bubble 内，时间可以任意快进，不需要 sleep
    // 所有并发行为在毫秒级完成
}
```

这篇文章不展开 synctest 的全部细节，但最核心的判断就一句话：

**如果你手上有 flaky 的并发测试，Go 1.25 的 synctest 比你任何一个 mock 方案都干净。**

---

## Green Tea GC：不需要改一行代码的 10-50% GC 下降

升到 Go 1.25 设置 `GOEXPERIMENT=greenteagc`，或者直接升到 Go 1.26（默认启用），你的 GC 开销就降了。

Green Tea GC 的核心创意是反直觉的：**按页扫描，而不是按对象扫描。**

Go 的标记-清扫 GC 本质上是一个图遍历算法。从 root 出发，沿着指针把可达对象全部标记出来。这个问题有一个很糟糕的内存访问模式：对象之间的指针跳转是随机的，CPU 几乎每次都要去主存里拿数据。

而现代 CPU 跑得比内存快 100 倍。GC 的大部分时间不是在"运行"，而是在**等内存**。

![GC 标记路径对比：传统 vs Green Tea](/images/go-125-126-automatic-modernization/inline-02-gc-comparison.png)

Green Tea 的解法很粗暴：**把扫描单元从"对象"提升到"页"。** 每次拿到一个页，把页内所有待扫描对象一锅端，在内存里连续访问。如果页 A 指向页 B，就把整个页 B 放到工作队列里，等轮到它时再把 B 的全部对象扫描完。

对比两种路径：

- **传统标记-清扫**：对象之间随机跳转，路径像在城市里穿街走巷
- **Green Tea**：按页批量处理，路径像在高速公路上直行

效果呢？在 Google 的实际工作负载中，GC CPU 开销降低了 **10% 到 50%** 不等。对于堆较大的服务，效果更明显。对于堆小的服务可能受益有限，但也不会更差。

而且注意：**零代码改动。** 改一行 go.mod 的版本号，你的 GC 就变快了。

---

## 那么问题来了：要不要升？

**如果你的 Go 版本在 1.21 以上（含），应该升到 1.26。**

理由三条，一条比一条硬：

**第一，Green Tea GC 没有成本。** 零代码改动，默认启用。只要你的堆不算太小，大概率能拿到 10% 以上的 GC CPU 下降。如果因为某些原因不想用，设置 `GOEXPERIMENT=nogreenteagc` 即可。

**第二，go fix 降低的不是性能成本，而是维护成本。** 这个更难量化，但你可以想一个问题：你的项目里有多少「知道该改但一直没改」的代码？它们不代表错误，但代表着每一个新成员读到这些代码时的认知摩擦。`go fix ./...` 一次性清掉这部分摩擦，而且几乎没有误改风险——所有 modernizer 在设计时都考虑了安全回退，有冲突的 fix 会被自动丢弃。

**第三，synctest 解决了一个之前根本没法优雅解决的问题。** 并发测试的 flaky 是 Go 项目里最讨厌的 bug 类型之一：本地跑不过，CI 偶尔红，排查起来用尽毕生修为。synctest 不是锦上添花，是把这个痛点变成了一行 `go test`。

唯一需要犹豫的场景是：**你的项目还在用 Go 1.18 或更早**，升级语言版本本身需要做兼容性评估。那主要的成本是升级 Go 版本，而不是升级后的功能。这种情况下你应该先评估 1.18→1.21 的成本，再从 1.21 跳到 1.26。

---

## 没有你，代码也变好了

回到开头的问题。

你那些「知道该改但一直没改」的代码，它们还在那里。三年也没怎么出过事——老风格不致命，只是不够好。

以前你需要抽出时间、在工单系统里排上优先级、手动逐行重构、提交 MR、等待 review。这个循环一次两次可以，但不会有人把两年积累的几十处改进都走一遍。

现在你只需要两件事：

1. 升级 go.mod 里的版本号
2. 跑一遍 `go fix ./...`

不是写代码的人变勤快了，是工具链设计思路变了。

Go 1.25 到 1.26 这个周期，在我看来代表一个新的趋势：**工具链不再只负责「帮你运行代码」，也开始「帮你维护代码」。** `go fix` 从冷门子命令变成了核心能力。`//go:fix inline` 让你自己也能定义迁移规则。分析器之间可以联动，跑两次就能达到手动优化的效果。

对于代码量上万行的 Go 项目来说，「每一次升级都会让你的代码自愈一点」，这比任何一个具体特性都重要。

---

### 你可以带走的三句话

1. **升级到 Go 1.26 后第一件事：跑 `go fix ./...`；如果一次不够，跑两次。**
2. **Green Tea GC 是零成本的性能收益，不值得为兼容性犹豫超过两天。**
3. **有并发测试 flaky？`testing/synctest` 在 Go 1.25 已经 GA，它在标准库里，不需要第三方 mock 框架。**

如果你对 synctest 的实现细节感兴趣——Bubble 是怎么隔离 goroutine 的、合成时钟怎么避免死锁——我后面可以再写一篇展开。也可以聊聊 `//go:fix inline` 的内部原理：源级内联器如何保证 7,000 行逻辑不出语义 bug。

关注我，后面接着写。
