---
title: "30 分钟用 Hugo 搭建个人博客，我为什么放弃了 WordPress"
date: 2026-03-27T18:20:00+08:00
draft: false
author: "任博"
description: "从 WordPress 迁移到 Hugo 后，网站加载速度从 3 秒降到 0.3 秒。这份完整指南带你从零搭建一个快速、优雅的个人博客。"
tags: ["Hugo", "博客", "静态网站", "GitHub Pages", "技术教程"]
categories: ["开发工具"]
featured: true
toc: true
weight: 2
---

> 写博客最痛苦的不是写，而是等页面加载。

三年前我用 WordPress 搭建博客，装了 5 个插件、3 个缓存优化，页面加载还是要 3 秒多。直到有一天，我在 GitHub 上看到同事的博客——**0.3 秒打开**，用的是 Hugo。

那天我花了 30 分钟迁移到 Hugo，再也没回去过。

这篇文章带你从零搭建一个功能完整、加载飞快的个人博客。**不需要服务器，不需要数据库，GitHub Pages 免费托管**。

<!--more-->

---

## 一、为什么是 Hugo？

先说结论：**如果你写的是技术博客、个人笔记、文档站点，Hugo 几乎是最佳选择**。

### 速度对比

```
┌─────────────────────────────────────────────────────────┐
│              静态网站生成器构建速度对比                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Hugo     ████████████████████  0.3 秒 (1000 篇文章)     │
│  Jekyll   ████████████████████████████████████  15 秒     │
│  Hexo     ██████████████████████████████  8 秒           │
│  WordPress ████████████████████████████████████████ N/A  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 核心优势

| 特性 | Hugo | WordPress |
|------|------|-----------|
| 构建速度 | 0.3 秒 | 需要数据库查询 |
| 部署成本 | 免费 (GitHub Pages) | 需要服务器 |
| 安全性 | 静态文件，无攻击面 | 需防 SQL 注入、XSS |
| 维护成本 | 几乎为零 | 需定期更新插件 |
| 学习曲线 | Markdown 即可 | 需熟悉后台 |

**Hugo 的代价**：没有后台管理界面，评论系统需要第三方服务。但对我来说，用写作换速度，值。

---

## 二、30 分钟快速开始

### 第一步：安装 Hugo（2 分钟）

**macOS**（推荐 Homebrew）：
```bash
brew install hugo
hugo version  # 验证安装
```

**Windows**：
```powershell
winget install Hugo.Hugo.Extended
```

**Linux**：
```bash
sudo apt-get install hugo  # Ubuntu/Debian
```

### 第二步：创建站点（1 分钟）

```bash
hugo new site my-blog
cd my-blog
```

此时目录结构：
```
my-blog/
├── archetypes/     # 文章模板
├── content/        # 文章内容 ← 你主要在这里工作
├── data/          # 数据文件
├── layouts/       # 页面布局（用主题时不用管）
├── static/        # 图片、CSS 等静态资源
├── themes/        # 主题目录
└── hugo.toml      # 配置文件 ← 待会要改
```

### 第三步：安装主题（3 分钟）

主题决定博客长什么样。推荐 **PaperMod**——简洁、快速、支持中文。

```bash
git init
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

修改 `hugo.toml`：
```toml
baseURL = 'https://yourname.github.io/'
languageCode = 'zh-cn'
title = '我的个人博客'
theme = 'PaperMod'

[params]
  author = "你的名字"
  description = "写点什么吧"
  defaultTheme = "auto"  # 自动深色/浅色模式
  ShowCodeCopyButtons = true  # 代码块复制按钮
  ShowReadingTime = true  # 显示阅读时间
  
[menu]
  [[menu.main]]
    name = "首页"
    url = "/"
    weight = 1
  [[menu.main]]
    name = "归档"
    url = "/posts/"
    weight = 2
```

### 第四步：写第一篇文章（5 分钟）

```bash
hugo new posts/hello-world.md
```

打开生成的文件，写入内容：
```markdown
---
title: "你好，世界"
date: 2026-03-27T18:20:00+08:00
draft: true  # ← 发布前改成 false
tags: ["随笔"]
---

这是我的第一篇博客文章。

## 为什么开始写博客

写博客是为了...
```

### 第五步：本地预览（1 分钟）

```bash
hugo server -D
```

打开浏览器访问 `http://localhost:1313`，看到博客了！

---

## 三、部署上线（GitHub Pages 免费托管）

### 方案对比

```
┌─────────────────────────────────────────────────────────┐
│                    部署方案对比                          │
├──────────────┬────────────┬────────────┬────────────────┤
│    方案      │    成本    │   难度     │    推荐度      │
├──────────────┼────────────┼────────────┼────────────────┤
│ GitHub Pages │   免费     │   ⭐⭐      │   ⭐⭐⭐⭐⭐     │
│ Vercel       │   免费     │   ⭐⭐      │   ⭐⭐⭐⭐       │
│ 云服务器     │  ¥50+/月   │   ⭐⭐⭐⭐    │   ⭐⭐         │
│ 虚拟主机     │  ¥20+/月   │   ⭐⭐⭐     │   ⭐⭐         │
└──────────────┴────────────┴────────────┴────────────────┘
```

### GitHub Pages 部署步骤

**1. 创建 GitHub 仓库**
```bash
# 仓库名必须是：yourusername.github.io
git remote add origin git@github.com:yourusername/yourusername.github.io.git
```

**2. 创建 GitHub Actions 工作流**

在 `.github/workflows/deploy.yml` 写入：
```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
      
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true
      
      - name: Build
        run: hugo --minify
        
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

**3. 推送并启用 Pages**

```bash
git add .
git commit -m "chore: 初始化博客"
git push -u origin main
```

然后在 GitHub 仓库页面：
1. 进入 **Settings → Pages**
2. Source 选择 **GitHub Actions**
3. 等待 1-2 分钟，访问 `https://yourusername.github.io`

---

## 四、进阶配置（按需选择）

### 1. 自定义域名

在 `static/CNAME` 文件中写入域名（不带协议）：
```
blog.example.com
```

修改 `hugo.toml`：
```toml
baseURL = 'https://blog.example.com/'
```

然后在域名服务商添加 CNAME 记录。

### 2. 评论系统

推荐 **Utterances**（基于 GitHub Issues，免费无广告）：

在 `layouts/partials/comments.html` 写入：
```html
<script src="https://utteranc.es/client.js"
        repo="yourusername/yourusername.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
```

在 `hugo.toml` 启用：
```toml
[params]
  comments = true
```

### 3. SEO 优化

```toml
[params]
  keywords = ["技术博客", "编程", "学习笔记"]
  images = ["/images/og-image.jpg"]  # 社交分享图
  
[sitemap]
  changefreq = "weekly"
  priority = 0.5
```

### 4. 文章统计

Hugo 内置字数统计和阅读时间：
```toml
[params]
  ShowWordCount = true
  ShowReadingTime = true
```

在文章中用 `.WordCount` 和 `.ReadingTime` 访问。

---

## 五、常见问题

### Q: 图片怎么上传？

放在 `static/images/` 目录，Markdown 中这样引用：
```markdown
![描述](/images/photo.jpg)
```

### Q: 怎么更换主题？

1. 删除 `themes/` 目录下的旧主题
2. 用 `git submodule add` 添加新主题
3. 修改 `hugo.toml` 中的 `theme` 字段

推荐主题：
- **PaperMod** - 简洁快速（本文使用）
- **Stack** - 卡片式布局
- **Blowfish** - 现代设计
- **Docsy** - 文档站点

### Q: 文章太多会不会变慢？

不会。Hugo 构建 1000 篇文章也只要几秒。我现在的博客有 50+ 文章，构建时间 0.5 秒。

### Q: 能从 WordPress 迁移吗？

可以。用 [hugo-import](https://github.com/palaniraja/hugo-import) 工具一键转换。

---

## 六、下一步

博客搭好了，接下来：

1. **写第一篇文章** - 不用追求完美，先写起来
2. **配置自定义域名** - 看起来更专业
3. **添加到导航** - 在社交媒体分享你的博客
4. **保持更新** - 每周至少一篇，形成习惯

---

## 资源

- [Hugo 官方文档](https://gohugo.io/documentation/)
- [PaperMod 主题文档](https://adityatelange.github.io/hugo-PaperMod/)
- [GitHub Pages 文档](https://pages.github.com/)
- [我的博客源码](https://github.com/ZampoRen/renbo-blog)

---

**最后说一句**：工具只是手段，内容才是核心。Hugo 能帮你解决速度问题，但写不出好文章——那是你的事。

现在，去写吧。
