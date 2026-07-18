---
title: "同事甩 10–40% 截图让你升 Go 1.26？先量自己的 GC 占比"
description: "官方 10–40% 说的是 GC 自身的 CPU 消耗。先量 GC 占总 CPU 的比例，再决定正常升级、做 A/B，还是临时关闭 Green Tea。"
date: 2026-07-17T02:40:00+08:00
draft: false
author: "Zampo"
tags: ["Go", "Go 1.26", "Green Tea GC", "GC", "性能", "工程决策"]
categories: ["技术实战"]
cover: "/images/go-green-tea-gc-decision/cover.png"
toc: true
---

值班群里有人贴了 Go 官方的截图：Green Tea 能把 GC 开销降 10–40%。

下一句往往是：那下周就升 1.26，服务应该能快不少。

我会先去看一条监控：GC CPU 在总 CPU 里占多少。没有这个数，截图再漂亮，也不能替你做性能判断。

**网上的 10–40% 说的是 GC 自己的 CPU，不是你的服务；先量自己的 GC 占比，再决定要不要为 Green Tea 做 A/B。**

## 先把 10–40% 的分母说清

官方的说法有边界：对 GC 压力很重的真实程序，GC 开销预计可下降约 10–40%。多数程序接近 10%，少数可以接近 40%。这里减少的是垃圾回收本身消耗的 CPU 时间，不是 QPS 一定上涨，也不等于 p99 一定下降。

官方给过一个很实用的换算。如果一个程序有 10% 的 CPU 时间花在 GC，上述改善折算到总 CPU，大致是 1–4%。这个数字不小，但它和“服务快了 40%”是两回事。

你的服务里 GC 若只占 5% 的总 CPU，即便 GC 自身少花 40%，总 CPU 的变化也很有限。反过来，GC 在 CPU 性能分析里长期占着明显一段，或者分配频繁、堆里有大量小的指针对象，Green Tea 才值得认真对比。

这里没有玄学，先看两个数：GC CPU 占比，以及 GC 自身能少花多少。前者很低时，升级后业务指标没有明显变化，本来就是可能出现的结果。

## 默认升级，不等于默认立项

Go 1.26 默认启用 Green Tea。要关掉，需要在构建时加 `GOEXPERIMENT=nogreenteagc`；Go 1.25 则相反，默认还是经典 GC，显式设置 `GOEXPERIMENT=greenteagc` 才会打开 Green Tea。

这是编译期开关。线上改一个环境变量不会把已经运行的二进制切到另一套 GC；想比较，就构建两份二进制。

官方也没有承诺所有程序都会受益。有些负载变化很小，甚至没有收益。单个内存页经常只扫描一个对象时，新路径未必划算。这种情况不需要把它解释成“功能没开上”。

DoltHub 在 Go 1.25 的实验开关下跑 sysbench，吞吐和 p50 延迟与经典 GC 基本持平。HydrAIDE 的一组百万对象创建压测里，作者报告 GC CPU 下降约 22%，整体运行时间下降约 5%。Go 的 issue #73581 还提到，tile38 这类指针关系很密集的堆，GC 开销可下降约 35%；稀疏、分散的树形结构，效果可能不明显，也可能小幅变差。

这些结果放在一起并不冲突。它们只是提醒我们：Green Tea 优化的是 GC，业务指标是否变化，取决于 GC 原来是不是主要消耗。

如果服务的瓶颈在磁盘、网络或下游 RPC，GC CPU 即使下降，p99 也未必会动。此时不该把“延迟没变”当成关闭 Green Tea 的理由，更不能把 GC 的局部改善写进业务提速目标。

## 要比较，先做一组小而干净的 A/B

我不会先搭一套性能平台。对大多数服务，同样的负载、相同的 `GOMAXPROCS`、预热后的稳定阶段，已经足够做第一轮判断。

主指标只有两类：

- GC CPU 占比：`runtime/metrics` 里的 `/cpu/classes/gc/total:cpu-seconds` 除以 `/cpu/classes/total:cpu-seconds`
- 业务指标：QPS，以及你平时用于判断服务质量的 p50、p95、p99

Go 1.26 下可以这样构建两份二进制：

```bash
go build -o app-green ./cmd/yourapp
GOEXPERIMENT=nogreenteagc go build -o app-classic ./cmd/yourapp

go version -m ./app-green | grep -i experiment || true
go version -m ./app-classic | grep -i experiment || true
```

Go 1.25 的方向反过来：经典 GC 是默认值，给 Green Tea 那份二进制加 `GOEXPERIMENT=greenteagc`。

`GODEBUG=gctrace=1` 里的标记时间、CPU 性能分析里的 GC 栈都可以辅助看，但它们替代不了前面两项。GC 内部有变化，不等于用户请求就更快；业务 p99 变了，也不能只归因给 GC。

我会按结果往下走：GC 占比下降、业务指标持平或更好，就保持默认；GC 占比下降但业务侧没有明显变化，也可以继续使用，只是别把它写成性能项目的战果；两边都在正常波动范围内，就跟着版本升级，不为这个开关单独维护一条链路。

只有出现明确回退，才临时用 `nogreenteagc` 关闭，并把可复现的负载和性能分析结果带去提 issue。Go 1.26 的发布说明已经写明，这个退出开关预计会在 1.27 移除，它适合排查，不适合成为团队长期配置。

## 我的选择其实很简单

不知道 GC 占比时，我先补 metrics 或 pprof。没有基线，“提升”只是感觉。

GC 占比很低，业务也不受 CPU 限制时，我会正常升到 1.26，不会为了 Green Tea 再开一个专项。还在 1.25 的服务，也不会为了追一个截图主动把实验 GC 推进生产。

GC 占比高，或者升级后确实出现值得解释的变化时，我才做同负载 A/B，同时看 GC CPU 和业务延迟分位。只看其中一个，结论很容易偏。

下次有人把 10–40% 的截图发到群里，先回到你的监控上：**GC 到底占了总 CPU 的多少？**
