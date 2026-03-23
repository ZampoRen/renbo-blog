+++
date = '2026-03-23T18:38:00+08:00'
draft = false
title = '使用 flow-hugo-publisher 技能搭建个人博客'
tags = ['OpenClaw', 'Hugo', '技能教程', '自动化']
description = '5 分钟上手，用 OpenClaw 技能自动化你的 Hugo 博客发布流程。'
+++

> 本文介绍如何使用我开发的 OpenClaw 技能 `flow-hugo-publisher` 快速搭建个人博客，从安装到部署全流程自动化。

**技能地址：** https://clawhub.ai/ZampoRen/flow-hugo-publisher

---

## 一、技能介绍

`flow-hugo-publisher` 是一个 OpenClaw 技能，用于自动化 Hugo 博客的发布流程。

### 核心功能

- ✅ 环境检测（Hugo/Git）
- ✅ 初始化博客项目
- ✅ 管理主题（PaperMod、Akanke 等）
- ✅ 本地预览
- ✅ Git 提交推送
- ✅ GitHub Actions 自动部署
- ✅ 状态持久化

### 适用人群

- 想用 Hugo 写博客但不想折腾部署
- 想要自动化发布流程
- OpenClaw 用户

---

## 二、前置条件

### 必需

| 项目 | 要求 |
|------|------|
| OpenClaw | 已安装并运行 |
| Node.js | v18+ |
| Git | 已安装 |
| GitHub 账号 | 已注册 |

### 可选

| 项目 | 说明 |
|------|------|
| Hugo | 技能会自动安装 |
| 域名 | 用于自定义域名 |

---

## 三、安装技能

提供两种安装方式，任选其一即可。

---

### 方式 1：自然语言安装（推荐新手）

在 OpenClaw 对话中直接说：

```
帮我安装 flow-hugo-publisher 技能
```

OpenClaw 会自动调用 `clawhub` 完成安装。

**优点：** 简单，无需记忆命令

---

### 方式 2：命令行安装

```bash
# 使用 clawhub 命令安装
clawhub install flow-hugo-publisher
```

安装完成后，技能会自动加载到 OpenClaw。

**优点：** 快速，适合熟悉命令行的用户

---

### 验证安装

**方式 1：在 OpenClaw 中询问**

```
查看已安装的技能
```

**方式 2：使用命令**

```bash
# 查看已安装的技能列表
clawhub list

# 或者检查技能目录
ls ~/.openclaw/skills/
```

如果看到 `flow-hugo-publisher`，说明安装成功。

---

## 四、快速开始（5 分钟上线）

### 步骤 1：触发技能

在 OpenClaw 对话中说：

```
帮我搭建一个 Hugo 博客
```

### 步骤 2：环境检测

技能会自动检测：

```
[阶段 A/步骤 1] 进行中：检测 hugo/git 环境
✅ Hugo 已安装 (v0.158.0)
✅ Git 已安装 (v2.51.0)
```

### 步骤 3：选择主题

内置 5 个常用主题：

| 主题 | 说明 | 适合 |
|------|------|------|
| `PaperMod` | 轻量快速 | 技术博客 ⭐ |
| `Akanke` | 官方示例 | 通用博客 |
| `Stack` | 侧边栏结构 | 内容型博客 |
| `Docsy` | 文档站点 | 知识库 |
| `blowfish` | 现代简洁 | 个人创作 |

### 步骤 4-10：完整流程

详细步骤请参考完整教程：[flow-hugo-publisher-tutorial.md](https://github.com/ZampoRen/flow-hugo-publisher)

---

## 五、常用命令

### 创建文章

```
帮我创建一篇文章，标题是"我的第一篇文章"
```

### 本地预览

```
启动本地预览
```

### 提交到 Git

```
提交当前变更到 GitHub
```

### 查看状态

```
查看当前博客状态
```

---

## 六、自定义域名

### DNS 配置

| 类型 | 主机记录 | 记录值 |
|------|----------|--------|
| `A` | `@` | `185.199.108.153` |
| `A` | `@` | `185.199.109.153` |
| `A` | `@` | `185.199.110.153` |
| `A` | `@` | `185.199.111.153` |
| `CNAME` | `www` | `你的用户名.github.io` |

### 配置步骤

1. 修改 `hugo.toml`：`baseURL = 'https://你的域名.com/'`
2. 创建 `static/CNAME` 文件
3. GitHub Settings → Pages → Custom domain
4. 提交并推送

---

## 七、故障排查

### Pages 部署失败

**错误：** `Get Pages site failed`

**解决：** Settings → Pages → Source 选择 GitHub Actions

### 主题加载失败

**解决：** `git submodule update --init --recursive`

### DNS 不生效

**解决：** 等待 5-30 分钟，使用 `dig 你的域名` 检查

---

## 八、总结

`flow-hugo-publisher` 技能让你可以：

- ✅ 5 分钟搭建博客
- ✅ 自动化部署流程
- ✅ 专注于内容创作
- ✅ 无需关心技术细节

**开始写作吧！** 🎉

---

**参考链接：**

- [技能仓库](https://clawhub.ai/ZampoRen/flow-hugo-publisher)
- [Hugo 官方文档](https://gohugo.io/)
- [GitHub Pages 文档](https://pages.github.com/)

---

*发布于 2026 年 3 月 23 日 · 深圳*
