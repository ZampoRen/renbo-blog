---
title: "我写了个 OpenClaw 技能，5 分钟自动搭建 Hugo 博客"
date: 2026-03-27T18:25:00+08:00
draft: false
author: "Zampo"
description: "不用记命令、不用配置 GitHub Actions，一句话让 AI 帮你搭建并部署 Hugo 博客。这个开源技能，推荐给你。"
tags: ["OpenClaw", "Hugo", "自动化", "技能教程", "GitHub Pages"]
categories: ["开发工具"]
featured: false
toc: true
weight: 3
---

> 搭博客最烦的不是写，是配置。

上周有个朋友问我："想用 Hugo 写博客，但光看教程就劝退了，有没有更简单的办法？"

我说有啊，用我写的那个技能，**一句话就能搭好**。

他不信。我在 OpenClaw 里说了一句"帮我搭建一个 Hugo 博客"，5 分钟后，博客上线了，地址发给了他。

他沉默了三秒，说："我之前的时间都喂狗了？"

<!--more-->

---

## 一、这个技能是啥？

`flow-hugo-publisher` 是我给 OpenClaw 写的一个技能，用来**自动化 Hugo 博客的整个发布流程**。

### 它能做什么

```
┌─────────────────────────────────────────────────────────┐
│              flow-hugo-publisher 能力地图                 │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  环境检测  →  自动安装 Hugo/Git                          │
│      ↓                                                  │
│  项目初始化 → 选主题、建仓库、配 GitHub Actions          │
│      ↓                                                  │
│  文章发布  →  本地预览 → Git 提交 → 自动部署             │
│      ↓                                                  │
│  状态管理  →  记住你的工作目录和发布历史                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 和手动搭建对比

| 步骤 | 手动搭建 | 用技能 |
|------|----------|--------|
| 安装 Hugo | 查教程 → 装 Homebrew → 安装 | 自动检测，缺失自动装 |
| 选主题 | 逛主题市场 → 纠结半天 | 5 个内置主题，一句话选 |
| 配 GitHub Actions | 复制 YAML → 改参数 → 试错 | 自动生成，无需配置 |
| 部署 | 手动 push → 等 Actions → 查错误 | 全自动，推送即发布 |
| 耗时 | 1-2 小时 | 5 分钟 |

**适合谁用**：
- 想用 Hugo 写博客但不想折腾部署
- 已经是 OpenClaw 用户
- 喜欢自动化，讨厌重复劳动

---

## 二、安装：两种方式任选

### 方式 1：自然语言安装（推荐）

在 OpenClaw 对话里直接说：

```
帮我安装 flow-hugo-publisher 技能
```

结束。OpenClaw 会自动调用 `clawhub` 完成安装。

### 方式 2：命令行安装

```bash
clawhub install flow-hugo-publisher
```

### 验证安装

```bash
# 查看已安装的技能列表
clawhub list

# 或者检查技能目录
ls ~/.openclaw/skills/
```

看到 `flow-hugo-publisher` 就是成功了。

**技能地址**：https://clawhub.ai/ZampoRen/flow-hugo-publisher

---

## 三、5 分钟上线流程

### 第 1 步：触发技能

在 OpenClaw 里说：

```
帮我搭建一个 Hugo 博客
```

### 第 2 步：环境检测

技能会自动检查：

```text
[阶段 A/步骤 1] 进行中：检测 hugo/git 环境
[阶段 A/步骤 1] 完成：Hugo v0.158.0 + Git 环境通过
[阶段 A/步骤 1] 下一步：确认工作目录
```

缺什么会自动提示安装命令。

### 第 3 步：选主题

内置 5 个常用主题：

| 主题 | 特点 | 适合场景 |
|------|------|----------|
| `PaperMod` | 轻量快速，支持深色模式 | 技术博客 ⭐ |
| `Akanke` | 官方示例，功能全面 | 通用博客 |
| `Stack` | 侧边栏布局，信息密度高 | 内容型博客 |
| `Docsy` | 文档站点风格 | 知识库/文档 |
| `blowfish` | 现代设计，视觉突出 | 个人创作 |

我推荐 **PaperMod**，也是我自己用的。

### 第 4 步：自动化流程

接下来技能会自动执行：

```text
[阶段 B/步骤 2] 创建项目目录
[阶段 B/步骤 3] 初始化 Hugo 站点 + 安装主题
[阶段 B/步骤 4] 初始化 Git 仓库
[阶段 B/步骤 5] 启动本地预览 → http://localhost:1313
[阶段 B/步骤 6] 生成 GitHub Actions 工作流
[阶段 B/步骤 7] 提交并推送到 GitHub
```

每一步都有进度输出，随时可以中断。

### 第 5 步：上线

推送后，GitHub Actions 会自动构建部署。

1-2 分钟后，访问 `https://你的用户名.github.io`，博客好了。

---

## 四、日常使用：发布文章

### 创建文章

```
帮我创建一篇文章，标题是"我的第一篇文章"
```

技能会在 `content/posts/` 下创建 Markdown 文件，并用默认编辑器打开。

### 本地预览

```
启动本地预览
```

访问输出的地址（通常是 `http://localhost:1313`）查看效果。

### 提交发布

```
提交当前变更到 GitHub
```

技能会：
1. 检查未提交的变更
2. 生成提交信息（可手动修改）
3. 提交并推送
4. GitHub Actions 自动部署

### 查看状态

```
查看当前博客状态
```

输出：
```text
当前工作目录：/Users/xxx/renbo-blog
Git 根目录：/Users/xxx/renbo-blog
当前分支：master
文章总数：5
最近提交：abc1234
部署地址：https://xxx.github.io/
部署状态：deployed
```

---

## 五、自定义域名（可选）

### DNS 配置

在域名服务商添加记录：

| 类型 | 主机记录 | 记录值 |
|------|----------|--------|
| `A` | `@` | `185.199.108.153` |
| `A` | `@` | `185.199.109.153` |
| `A` | `@` | `185.199.110.153` |
| `A` | `@` | `185.199.111.153` |
| `CNAME` | `www` | `你的用户名.github.io` |

### 配置步骤

1. 修改 `hugo.toml`：
   ```toml
   baseURL = 'https://你的域名.com/'
   ```

2. 创建 `static/CNAME` 文件，写入域名（不带协议）：
   ```
   你的域名.com
   ```

3. GitHub 仓库 → Settings → Pages → Custom domain → 填写域名

4. 提交并推送

DNS 生效需要 5-30 分钟，可以用 `dig 你的域名` 检查。

---

## 六、常见问题

### Q: Pages 部署失败，提示 "Get Pages site failed"

**原因**：Pages 的 Source 没选对。

**解决**：GitHub 仓库 → Settings → Pages → Source 选择 **GitHub Actions**。

### Q: 主题加载失败

**解决**：
```bash
git submodule update --init --recursive
```

### Q: 端口被占用，预览启动失败

技能会自动尝试 1313 → 1314 → 1315 递增端口，并记录实际使用的端口到状态文件。

### Q: 能用其他主题吗？

可以。技能支持自定义主题仓库地址：
```
帮我用这个主题：https://github.com/xxx/xxx-theme.git
```

### Q: 状态文件在哪？

`~/.openclaw/state/hugo-publisher-state.json`

记录了工作目录、Git 状态、发布历史等。

---

## 七、技能架构（给想二次开发的你）

技能采用分阶段托管执行模型：

```
阶段 A: 环境确认与安装
    ↓
阶段 B: 初始化项目（首次）
    ↓
阶段 C: 日常发布快路径（后续默认）
    ↓
阶段 D: 状态写回与收尾输出
```

每个阶段有明确的输入输出，支持人工介入确认。

**核心文件**：
- `SKILL.md` - 技能主逻辑
- `references/workflow.md` - 工作流与命令模板
- `references/state-schema.md` - 状态文件结构
- `references/github-actions.md` - GitHub Actions 部署指南

**项目地址**：https://github.com/ZampoRen/flow-hugo-publisher

---

## 八、最后说两句

写这个技能的初衷很简单：**我不想再重复配置 GitHub Actions 了**。

每次搭新博客，都要复制 YAML、改参数、试错、查文档。明明这些都可以自动化，为什么还要手动做？

所以我把整个流程封装成一个技能，现在只需要一句话。

**工具的价值，不是让你做更多事，而是让你少做那些本不必做的事。**

---

**相关链接**：

- [技能仓库 - ClawHub](https://clawhub.ai/ZampoRen/flow-hugo-publisher)
- [技能源码 - GitHub](https://github.com/ZampoRen/flow-hugo-publisher)
- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [Hugo 官方文档](https://gohugo.io/)

---

*有问题欢迎在评论区留言，或者去 GitHub 提 Issue。*
