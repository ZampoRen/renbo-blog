---
title: "你每天都在用 Git，但你可能不懂 rebase"
description: "你在 feature 分支上写了三个提交，准备提 PR。同事说：先 rebase 一下 main，把历史整理干净。你照做了，但心里没底：rebase 到底改了什么？会不会出问题？"
date: 2026-04-14T16:10:00+08:00
draft: false
slug: "git-rebase-explained"
tags: ["Git", "版本控制", "rebase", "merge", "协作", "工程实践"]
categories: ["技术"]
cover:
  image: "/images/git-rebase-explained/cover.svg"
  alt: "Git rebase vs merge 对比"
---

![Git rebase vs merge 对比](/images/git-rebase-explained/cover.svg)

你在 feature 分支上写了三个提交，准备提 PR。

同事说：先 rebase 一下 main，把历史整理干净。

你照做了，但心里没底：rebase 到底改了什么？会不会出问题？

很多开发者对 rebase 的理解，停留在"会敲命令，不知道它到底做了什么"。

<!--more-->

## rebase 到底做了什么

`git rebase` 最常见的误解是：它只是把提交"搬到另一个位置"。

不对。

**rebase 不是移动提交，是在新的基底上重放提交。**

这意味着什么？

你的提交内容看起来一样，但它们的父提交已经变了。

父提交变了，commit hash 就全变了。

所以 rebase 本质上是在**改写历史**。

Git 不是把旧提交平移过去，而是重新构造出一批"新的提交版本"。

正因为如此，rebase 才会带来干净的线性历史，也才会在共享协作里产生风险。

![rebase vs merge 差异](/images/git-rebase-explained/inline-01.svg)

## 为什么很多团队喜欢 rebase

作者给出的主要理由，是 rebase 可以让历史更线性、更易读。

如果不用 rebase，团队开发里很容易出现一堆交叉的 merge 轨迹，历史图会越来越乱。

对小项目来说问题不大，但在多人协作、开源仓库、长期维护项目里，这种噪音会明显增加 review 和审计成本。

rebase 的价值主要体现在三点：

- 让分支历史更干净
- 在不额外制造 merge commit 的前提下，把功能分支同步到最新的主线
- 在发起 PR 前，把自己的提交整理成更容易被审阅的形式

如果从内容写作角度表达，可以概括为：rebase 追求的是"呈现出一条清晰的工作轨迹"，而不是保留所有混乱的开发过程。

## 一个具体例子

视频里举了一个典型例子：

- 主分支 main 原本走到 B
- 你从 B 拉出 feature 分支，做了 D、E、F 三个提交
- 这时 main 上又新增了 G
- 现在你想把 feature 分支更新到最新 main 之上

如果在 `feature` 分支执行 `git rebase main`，Git 大致会这样处理：

1. 找到 feature 和 main 的共同祖先 B
2. 暂存你在 feature 上做的 D、E、F
3. 把分支基底移动到 main 的最新提交 G
4. 再把 D、E、F 重新应用到 G 后面

最终结果看起来像是：你的开发工作从一开始就是基于 G 展开的。

历史会更直、更干净，但 D、E、F 实际上已经变成了新的 D'、E'、F'，不再是原来的提交。

## merge 和 rebase 的真正区别

作者反复强调，不要把 rebase 神化，也不要妖魔化。

它和 merge 解决的是相关但不完全相同的问题。

**merge 是 additive，rebase 是 reconstructive。**

更通俗一点：

- merge 是"把两条路接上"
- rebase 是"把你的脚印重新铺到另一条路上"

merge 会额外产生一个 merge commit，保留真实的分叉与汇合过程。

rebase 会获得更线性的历史，但会改写提交历史。

**rebase 换来整洁，代价是改写历史。**

## 什么时候该用，什么时候别碰

写到这儿，得给 rebase 一个公平的位置。

rebase 不是万能，也不是绝对危险。

### 适合 rebase 的场景

**本地分支整理：** 你在自己的 feature 分支上开发，还没推送到远端。

**PR 前整理历史：** 准备提 PR 前，把提交整理成更清晰的形式。

**同步最新主线：** 把功能分支更新到最新 main 之上，但不想产生 merge commit。

### 不适合 rebase 的场景

**共享分支：** 如果这个分支已经有其他人在用，别乱 rebase。

**公共主线：** main/master 这种公共分支，永远不要 rebase。

**已推送的提交：** 如果提交已经推送且别人可能拉取了，rebase 会强制他们同步。

**本地分支适合 rebase，共享分支更适合 merge。**

![什么时候用 rebase](/images/git-rebase-explained/inline-02.svg)

## 正确使用 rebase 的姿势

如果你决定用 rebase，记住这几条：

- 只在本地分支、未推送的提交上使用
- rebase 前确认这个分支没有其他人依赖
- 如果 rebase 后需要强推，先确认团队是否接受这种做法
- 不要为了"线性历史"而牺牲协作安全

---

**rebase 不是黑魔法，是工程取舍。**

它不是"一定比 merge 高级"，也不是"绝对危险、最好别碰"。

它是一种很有用的工程整理手段，但它通过改写历史换来整洁。

所以关键在于：你知道自己在改什么，也知道什么时候不该改。
