---
title: "装了 Claude Code，别上来就写 prompt"
description: "从安装到第一个项目，Plan Mode + claude.md 才是 Claude Code 的灵魂。不是功能越多越好用，而是用对方法才不翻车。"
date: 2026-05-08T10:00:00+08:00
draft: false
tags: ["Claude Code", "AI 编程", "终端工具", "开发效率"]
categories: ["技术实战"]
cover: "/images/claude-code-beginner-guide/cover.svg"
toc: true
---

装好 Claude Code，打开终端，输入第一句话：

> 帮我做一个办公室排班应用。

五分钟后，Claude 生了一堆文件。`npm install` 跑不起来，页面打不开，你盯着终端不知道该从哪开始修。

问题不在 Claude Code，在你给 prompt 的方式。

很多人装完 Claude Code 的第一反应是把它当聊天机器人用——扔一句话，等它干活。结果就是：代码能生成，但跑不起来；功能有，但结构乱；改一处，崩三处。

这篇文章不罗列功能，不背参数。只讲一件事：**怎么从装好 Claude Code 到做出第一个能跑的项目，中间不翻车。**

---

## Claude Code 是什么

一句话：Anthropic 出的终端 AI 编程工具。

它不是 IDE，不是编辑器，是一个跑在终端里的 AI 代理。你能跟它对话，它能读你的代码、改你的文件、跑你的命令、提交你的 Git。

Anthropic 团队里有个说法很准确：

> Claude Code 就像那个只用终端的同事——从不碰 GUI，但什么都能搞定。

它和 Cursor 的区别也很简单：Claude Code 是终端里的 AI，Cursor 2.0 之后更偏向对话式开发环境。两者不冲突，但分工不能再按老版本 Cursor 来理解——不要再把 Cursor 简单当成“看代码和调试”的辅助位。

---

## 安装：三分钟搞定

### 前置条件

- 装了 Node.js（`node -v` 能出版本就行）
- 有 Claude 付费订阅（Pro / Max / Teams / Enterprise），或者有 API Key

> 2026 年 4 月 21 日 Anthropic 曾短暂从 Pro 计划中移除 Claude Code 访问权限，后恢复。定价政策可能波动，写这篇文章时的价格是：Pro $20/月、Max 5x $100/月、Max 20x $200/月。

### 安装命令

```bash
# 全局安装
npm install -g @anthropic-ai/claude-code

# 启动
claude
```

第一次启动会让你选主题（推荐暗色），然后浏览器登录授权。完事就能用了。

Windows 用户用 PowerShell 跑同样的命令。Mac/Linux 也可以从官网复制 bash 脚本安装，效果一样。

---

## 别跳过这一步：Git 初始化

Claude Code 用 Git 来追踪变更和自动提交。新项目第一件事：

```bash
cd my-project
git init
```

不初始化 Git，Claude 就没法自动提交，你也看不到它改了什么。这不是可选项，是必选项。

---

## 正确打开方式：在 IDE 里开终端

Claude Code 是终端工具，但不意味着你要离开 IDE。

推荐做法：

1. 在 VS Code 里打开项目文件夹；如果你已经在用 Cursor，也可以直接用它的终端
2. 打开内置终端（`Ctrl+\`` 或 `Cmd+\``）
3. 输入 `claude` 启动

这样你既有文件树和代码高亮，又有 AI 终端。两边配合，比纯终端效率高得多。

---

## Plan Mode：这才是核心功能

装好 Claude Code 后，大多数人做的第一件事是直接输入需求。

**这是翻车的起点。**

Claude Code 有两种模式：

| 模式 | 行为 | 适合场景 |
|------|------|----------|
| Plan Mode | Claude 先分析、提问、出方案，不写代码 | 复杂任务、新项目、需求不明确 |
| YOLO Mode | Claude 自主执行直到完成 | 简单任务、你已经确认方案 |

切换方式：`Shift+Tab`

Plan Mode 的核心价值是：**Claude 永远不会在 Plan Mode 里自动写代码。** 它会先理解你的需求，问清楚边界，给出方案，等你确认了才动手。

### Plan Mode 实战

假设你要做一个办公室排班应用。

**错误做法（直接写 prompt）：**

```
帮我做一个排班应用，要有日历视图，能添加删除任务，能分配给团队成员。
```

结果：Claude 会直接开始写代码，但技术栈、框架、数据结构全由它猜。猜错了，后面全得改。

**正确做法（Plan Mode）：**

1. `Shift+Tab` 切到 Plan Mode
2. 用 bullet points 写需求：

```
我要做一个办公室排班应用，要求：
- 日历视图，类似 Outlook
- 能添加/删除排班任务
- 支持周期性排班（每周/每月重复）
- 能分配给团队成员
- 能添加/删除团队成员
- 技术栈：Next.js + TypeScript + Tailwind

请先确认几个问题：
1. 需要后端还是纯前端？
2. 数据存储用 localStorage 还是数据库？
3. 用户认证需要吗？
```

3. Claude 会先回答你的问题，给出方案，列出文件结构
4. 你确认方案后，它才开始写代码

**这就是 Plan Mode 的价值：先对齐，再动手。**

---

## claude.md：Claude Code 的灵魂

Claude 的会话没有无限记忆。你退出终端再进来，上下文就丢了。

`claude.md` 就是解决这个问题的——它是 Claude Code 的持久记忆文件，每次启动新会话时自动加载。

### 生成 claude.md

在 Claude Code 里输入：

```
/init
```

Claude 会自动分析你的代码库，生成一个 `claude.md` 文件。你也可以自己写。

### claude.md 应该写什么

三个维度：

| 维度 | 内容 | 示例 |
|------|------|------|
| What | 技术栈、项目结构、关键文件 | "Next.js + TypeScript，src/app 是路由，src/components 是组件" |
| Why | 设计决策、架构选择的原因 | "用 Zustand 不用 Redux，因为项目小，不需要复杂状态管理" |
| How | 运行命令、工程原则 | "每次修改后跑 npm test，新特性先建 Git 分支" |

### 渐进式披露

不要把几百行信息塞进一个 `claude.md`。更好的做法是：

```markdown
# claude.md（索引文件）

## 项目概述
技术栈：Next.js + TypeScript + Tailwind
项目结构：见 .claude/architecture.md

## 工程规范
见 .claude/engineering-principles.md

## 常用命令
- 开发：npm run dev
- 测试：npm test
- 构建：npm run build
```

然后把详细信息分到专门的文件里。Claude 会按需读取。

### 验证 claude.md 是否生效

最简单的方法：退出 Claude Code，重新启动，然后问它一个项目相关的问题。如果它能答上来，说明 `claude.md` 在起作用。

---

## 快捷键：记住这几个就够了

| 快捷键 | 功能 |
|--------|------|
| `Esc` 两次 | 清空当前输入 |
| `↑/↓` | 切换历史命令 |
| `Option+Enter`（Mac）/ `Alt+Enter`（Win） | prompt 里换行 |
| `Shift+Tab` | 切换模式（Plan Mode / YOLO Mode） |
| `Esc` | 中止正在执行的操作 |

不需要背，用几次就记住了。

---

## 其他实用命令

| 命令 | 功能 |
|------|------|
| `/compact` | 上下文满了？压缩总结，继续聊 |
| `/model` | 查看/切换当前模型（Sonnet / Opus） |
| `/tasks` | 查看后台长任务 |
| `/agents` | 创建和管理专门代理（样式代理、后端代理等） |

`/compact` 特别实用。Claude 的上下文窗口有限，聊久了会满。跑一下 `/compact`，它会总结之前的对话，然后无缝衔接。

---

## 从 0 到 1：做一个能跑的项目

把上面的东西串起来，走一遍完整流程。

### 第一步：创建项目

```bash
mkdir chore-app && cd chore-app
git init
```

### 第二步：启动 Claude Code

在 IDE 里打开 `chore-app` 文件夹，开终端，输入：

```bash
claude
```

### 第三步：Plan Mode 出方案

`Shift+Tab` 切到 Plan Mode，输入：

```
我要做一个办公室排班应用：
- 日历视图
- 添加/删除排班任务
- 周期性排班
- 分配给团队成员
- Next.js + TypeScript + Tailwind

请先确认技术细节再动手。
```

Claude 会问几个问题，给出方案。确认后它开始写代码。

### 第四步：生成 claude.md

代码生成后，输入：

```
/init
```

Claude 分析项目结构，生成 `claude.md`。以后每次启动都会自动加载项目上下文。

### 第五步：修 Bug

真实项目不可能一次跑对。假设日历有个 Bug——点击时间显示错误。

你不需要自己找。直接告诉 Claude：

```
日历上点击某个时间段，高亮显示的时间不对。帮我排查并修复。
```

Claude 会用 `claude.md` 里的上下文理解项目，找到问题，修复它。

### 第六步：加新功能

想加实时同步？切到 Plan Mode，描述需求，让 Claude 给出几个方案（纯前端 / WebSocket / 其他），选一个，然后让它干。

如果开了 YOLO Mode，Claude 会自主完成，不需要你一步步确认。

---

## 常见误区

### 误区一：prompt 写得太模糊

> ❌ "帮我做个网站"
> ✅ 用 bullet points 列出需求 + 技术栈 + 约束条件

Claude 不是算命先生。你给的信息越模糊，它猜的成分越大。

### 误区二：不用 Plan Mode 直接干

复杂任务不用 Plan Mode，等于蒙眼开车。不是不能到目的地，是路上大概率撞墙。

### 误区三：忘了 claude.md

每次新项目都跑一下 `/init`。没有 `claude.md` 的 Claude Code，每次重启都是失忆状态。

### 误区四：完全放手不管

YOLO Mode 不是让你去喝咖啡。Anthropic 团队的说法是"Smart Vibe Coding"——小步快跑，频繁测试，盯着 to-do 列表，不对劲就按 `Esc` 中止。

### 误区五：按老版本 Cursor 的方式理解工具分工

Claude Code 和 Cursor 可以一起用，但不要再把 Cursor 2.0 简单理解成“代码浏览 + 调试”的辅助 IDE。它已经明显偏向对话式开发，真正的差异不是谁负责看代码、谁负责写代码，而是交互入口不同：Claude Code 把工作流压进终端，Cursor 把工作流放进对话环境。

如果你日常主要在终端里跑命令、改项目、处理 Git，Claude Code 会更顺手；如果你更习惯在对话里组织需求和上下文，Cursor 仍然有它的位置。关键不是二选一，而是别拿旧版本印象做今天的工具判断。

---

## 写在最后

回到开头那个翻车场景。

如果你装好 Claude Code 后，先 `git init`，再切到 Plan Mode 用 bullet points 写需求，等项目跑起来后跑一下 `/init` 生成 `claude.md`——

那五分钟后你不会盯着一堆跑不起来的代码发呆。你会看到一个能用的应用，和一个知道下次该怎么用的自己。

Claude Code 不是聊天机器人。它是那个只用终端的同事。你得用对方式跟它说话，它才能帮你把活干好。

**Plan Mode + claude.md，记住这两个，你就已经掌握了 Claude Code 最核心的工作流。**
