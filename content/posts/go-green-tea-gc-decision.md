---
title: "同事甩 10–40% 截图让你升 Go 1.26？先量自己的 GC 占比"
description: "官方 10–40% 说的是 GC 自己的开销，不是业务整体提速。先量自己的 GC CPU 占比，再决定跟版本、A/B 对比，还是临时关掉 Green Tea。"
date: 2026-07-17T02:40:00+08:00
draft: false
author: "Zampo"
tags: ["Go", "Go 1.26", "Green Tea GC", "GC", "性能", "工程决策"]
categories: ["技术实战"]
cover: "/images/go-green-tea-gc-decision/cover.svg"
toc: true
---

值班群里有人甩了张截图。Go 官方说 Green Tea 能把 GC 开销砍 10–40%。

有人已经在催：下周升 1.26，业务应该快一截。

你盯着监控里 GC CPU 那条曲线——几个点的占比，不高不低——犹豫要不要为它改构建参数。

我最想纠正的误判就这一句：**版本说明写了 10–40%，所以升上就稳赚。**

## 10–40% 是 GC 自己的 CPU，不是你服务的

官方博客写得很清楚：对 GC 压力很重的真实程序，GC 开销大约降 10–40%。很多程序落在 10% 附近，少数能接近 40%。

注意限定词：说的是 GC 自己的开销，不是 QPS，不是 p99，不是你服务的整体 CPU。

官方博客自己做过换算。一个程序如果大约 10% 的时间花在 GC 上，GC 成本再降 10–40%，整体 CPU 大概只动 1–4 个百分点。

所以同事转发那张截图的时候，你在心里要先乘一个数：你服务里 GC 占总 CPU 的比例。**广告百分比不直接等于你的收益，要先剪掉你 GC 那部分的蛋糕，再看 Green Tea 能切多少。**

你的服务 GC 只占 5%，就算 GC 侧砍掉 40%，总 CPU 也只是几个百分点的事。反过来，分配很猛、小对象很多、profile 里 GC 栈长期显眼——那 Green Tea 才更可能让你看见。

**这是广告百分比的工程算术，不是「所有人立刻开」的口号。**

## 升完无感，先别怀疑

Go 1.26 起，Green Tea **默认启用**。构建时 `GOEXPERIMENT=nogreenteagc` 才能关掉。1.25 则相反：默认经典 GC，要显式 `GOEXPERIMENT=greenteagc` 才打开。

两边都是编译期开关，运行时切不了。想对比，得打两份二进制。

官方也写得很直：有的负载几乎不受益，甚至完全不受益。单页上经常只扫到一个对象时，新扫描路径不一定划算。

社区两边的对照结果，正好在同一张图里。

DoltHub 在 1.25 实验开关下跑 sysbench：吞吐和 p50 延迟跟经典 GC 基本一样。HydrAIDE 在 100 万对象创建压测里报告 GC CPU 降了大概 22%、整体运行时间降了大约 5%。issue #73581 里，tile38 这类指针密集的堆能看到约 35% 的 GC 开销下降；稀疏、打散的树形结构效果不明显，甚至小幅回退。

同一套 GC，两种体感，不矛盾。

业务侧无感时，别为实验位硬立项。GC 占比高时那一栏动了，才是它该发光的地方。

还有一种更常见的「无感」：瓶颈在磁盘、网络、下游 RPC。GC CPU 真降了，用户延迟仍然纹丝不动。别因为业务线没动就关掉 Green Tea。瓶颈在线下的时候，GC CPU 降再多也不会体现在用户延迟上。GC 优化别写进业务 SLA。

升完 1.26 如果看不出来业务变化，先确认你的二进制是默认编译的，然后查一下你的 GC 占比。无感是算术结果，不是故障信号。

## 想证伪，两份二进制就够了

不用先搭一套性能平台。

同负载、预热后稳态，两样东西就够了：

- **GC CPU 占比**（`runtime/metrics`：`/cpu/classes/gc/total:cpu-seconds` ÷ `/cpu/classes/total:cpu-seconds`）
- **业务分位**（QPS + p50/p95/p99，跟你平常看的那一组走）

Go 1.26，默认 vs 关闭：

```bash
go build -o app-green ./cmd/yourapp
GOEXPERIMENT=nogreenteagc go build -o app-classic ./cmd/yourapp

# 确认实验位
go version -m ./app-green | grep -i experiment || true
go version -m ./app-classic | grep -i experiment || true
```

Go 1.25 反过来：默认经典 GC，`GOEXPERIMENT=greenteagc` 才会开 Green Tea。

辅助可以再看 `GODEBUG=gctrace=1` 的 mark 时间和 `go tool pprof` 的 GC 栈。但主判断不要离开两项：GC 占比和业务延迟。

GC 占比下降，业务持平或更好——保持默认。
GC 占比下降，业务几乎不动——仍然可以开，但别写进 OKR 说「提速 20%」。
GC 占比和业务都在噪声范围内——跟版本就行，不用做开关平台。
明确变差——临时 `nogreenteagc` 止血，同时去 <https://go.dev/issue/new> 提 issue。

官方已经写明：1.26 的退出开关预计在 Go 1.27 移除。关掉是排查通道，不是团队长期标准。

## 我怎么决定

先排除另一类题：OOM、泄漏、拧 `GOGC`。那是旧文的线，本篇不重做。

不知道 GC 占比？先 metrics / pprof，再谈开关。没有基线，所有「提升」都是感觉。

GC 占比很低，业务也不吃 CPU？1.26 直接跟版本，别改 `GOEXPERIMENT`。1.25 默认别开实验，除非你想采一份对照。

GC 占比清晰可见？同负载 A/B。看 GC 占比，也看业务分位。只看一个，容易误判。

A/B 明确回退？临时 `nogreenteagc` + 提 issue。不要默默把关闭开关写进 Dockerfile 当永久配置。

你下次再看到那张 10–40% 的截图，先问自己一个问题：**我服务里，GC 到底占多少？**
