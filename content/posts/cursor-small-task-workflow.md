---
title: "给 Cursor 一个小需求，怎样让它先计划、再修改、再验收"
description: "用一个最小登录页 demo，演示 Cursor Agent 的正确使用顺序：先分析、再计划、再执行、最后验收。附可复制提示词和自检模板。"
date: 2026-05-20T10:39:00+08:00
draft: false
author: "任博"
tags: ["Cursor", "AI 编程", "Agent", "开发工作流", "技术实战"]
categories: ["技术实战"]
cover: "/images/cursor-small-task-workflow/cover.png"
toc: true
---

这篇不讲安装，也不讲一堆按钮。

我们只做一件小事：给 Cursor 一个登录页，让它把页面视觉调得更像一个正式产品，但不允许它改业务逻辑。

任务很小，正好适合练手。小到你能看完所有 diff；也真实到足够暴露问题。很多人用 Agent 翻车，不是让它做了多复杂的架构，而是从一句“帮我优化一下页面”开始，最后发现字段、校验、接口路径都被顺手动了。

这一次不追求“让 Cursor 改得多快”。

我们只看一件事：它能不能按你的节奏走，分析、计划、执行、验收。

![Cursor 小需求实操封面](/images/cursor-small-task-workflow/cover.png)

## 准备一个小到能验收的项目

我用的是一个最小登录页 demo，只有四个核心文件：

```text
cursor-agent-demo/
├── index.html
├── style.css
├── login.js
└── tests/login.test.js
```

页面里有两个字段：邮箱和密码。

表单的业务契约也很明确：

```html
<form id="loginForm" action="/api/login" method="post">
  <input id="email" name="email" type="email" required />
  <input id="password" name="password" type="password" required minlength="8" />
</form>
```

这次需求只允许改视觉层：背景、卡片、阴影、圆角、按钮质感。

不能改这些东西：

- 表单字段名
- `form action`
- 请求路径
- `required`
- `minlength`
- 提交流程

如果你第一次练 Cursor Agent，我建议也从这种项目开始。不要一上来就让它改业务模块、重构页面、接新接口。项目越大，你越容易被一堆看起来很努力的 diff 淹没。

Agent 不是不能做大任务。

但你得先练会怎么管住小任务。

## 第一步：只分析，不许改

很多人给 Cursor 的第一句话是：

```text
帮我优化登录页。
```

这句话太危险。

“优化”是什么？只改样式，还是改交互？能不能抽组件？能不能动校验？能不能顺手把 HTML 结构也重排？你没说清楚，Agent 就只能猜。

我的第一句话会这样写：

```text
先不要改代码。

请只阅读 index.html、style.css 和 login.js，告诉我：
1. 登录页的视觉层主要在哪个文件；
2. 表单行为由哪些字段和属性决定；
3. 如果只改视觉，不应该碰哪些东西。
```

注意开头那句：先不要改代码。

这不是礼貌用语，是刹车。

Cursor 官方文档里，Agent 可以运行终端命令、搜索代码、编辑文件。Plan Mode 也强调在写代码前先研究代码库、提出问题、生成可审查计划。换句话说，你不应该把 Cursor 的第一步设置成“动手”，而应该设置成“理解”。

![Cursor 分析阶段示意](/images/cursor-small-task-workflow/step-01-analysis.png)

这一步你要看的不是答案多漂亮，而是它有没有识别出边界。

一个合格的分析结果，至少应该说清楚：

- 视觉主要在 `style.css`
- 表单结构在 `index.html`
- 行为逻辑在 `login.js`
- 这次不应该改 `action="/api/login"`
- 不应该改 `name="email"` / `name="password"`
- 不应该改 `required` 和 `minlength="8"`

如果 Cursor 连这些都没看出来，别急着让它继续。

继续之前先纠偏，比改完再擦屁股便宜得多。

## 第二步：让它写计划，而且计划要能审

分析通过后，再让它写计划。

我会这样追问：

```text
请给出一个实现计划。

要求：
1. 按文件列出准备修改什么；
2. 明确哪些文件不改；
3. 只做视觉层调整；
4. 每一步都要能用 diff 和测试验证；
5. 先不要写代码，等我确认。
```

这一步很容易被跳过，但它决定后面会不会失控。

坏计划通常长这样：

```text
我会优化登录页样式，提升用户体验，并确保功能正常。
```

这种话没用。听起来正确，实际没有边界。

好计划应该更像这样：

```text
计划：
1. 只修改 style.css。
2. 调整 body 背景，从纯渐变改成渐变 + 柔和光斑。
3. 调整 .login-card 的 padding、圆角、阴影和边框。
4. 保持 index.html、login.js 不变。
5. 修改后运行 npm test，确认登录契约未变。
```

这里最关键的不是“改什么”，而是“不改什么”。

很多 Agent 事故，不是少做了，而是多做了。

![Cursor 计划阶段示意](/images/cursor-small-task-workflow/step-02-plan.png)

看到计划后，不要客气。该删就删，该收窄就收窄。

比如它如果提出“顺便把登录表单封装成组件”，你就应该直接拒绝。这个需求只改视觉，不做结构重构。你越早把范围压住，后面的 diff 越干净。

## 第三步：只让它执行计划里的第一步

计划确认后，也不要说“开始吧”。

我更建议这样写：

```text
可以执行。

只实现计划中的视觉修改。
只允许修改 style.css。
不要修改 index.html 和 login.js。
改完后停下来，列出实际改动文件和验证建议。
```

为什么要这么啰嗦？

因为 Agent 很擅长“顺手”。顺手重排结构，顺手抽函数，顺手改命名，顺手补一个它觉得更好的交互。

这些动作单看都像在帮忙，但它们会让验收变复杂。

这次实际改动只落在 `style.css`：

```diff
diff --git a/style.css b/style.css
-  background: linear-gradient(135deg, #111827 0%, #1f2937 50%, #0f172a 100%);
+  background:
+    radial-gradient(circle at 20% 20%, rgba(59,130,246,.35), transparent 32%),
+    linear-gradient(135deg, #111827 0%, #1f2937 52%, #020617 100%);

-  padding: 34px;
-  border-radius: 24px;
-  background: rgba(255,255,255,.94);
-  box-shadow: 0 28px 80px rgba(0,0,0,.35);
+  padding: 38px;
+  border-radius: 28px;
+  background: rgba(255,255,255,.96);
+  box-shadow: 0 32px 90px rgba(0,0,0,.38);
+  border: 1px solid rgba(255,255,255,.45);
```

这个 diff 有两个好信号：

第一，只改了 `style.css`。

第二，改动内容和计划一致，都是视觉层。

![Cursor diff 检查示意](/images/cursor-small-task-workflow/step-03-diff.png)

这时候不要马上让它“再优化一下”。

先看 diff。

只要 diff 里出现了计划之外的文件，比如 `index.html`、`login.js`、路由文件、接口文件，就先停。不是说一定错，但你要先问清楚：为什么动它？是不是必要？能不能撤掉？

AI 编程最容易失控的地方，不是第一刀。

是你连续点了五次继续以后，已经不知道哪一刀改坏了东西。

## 第四步：验收，不要只看页面好不好看

视觉改完后，当然要打开页面看。

但只看页面是不够的。

这次需求真正要验收的是：视觉变了，业务契约没变。

我在 demo 里准备了一个最小测试：

```bash
npm test
```

实际输出是：

```text
> test
> node tests/login.test.js

PASS: login contract unchanged — route, fields, validation and submit path are intact.
```

这句通过，说明几件事没被动：

- `action` 还是 `/api/login`
- 字段还是 `email` 和 `password`
- 密码长度规则还在
- 前端提交逻辑没有被改坏

再看工作区状态：

```bash
git status --short
```

输出是：

```text
 M style.css
?? evidence.diff
```

这就很清楚：真正的代码改动只在 `style.css`，`evidence.diff` 是本次写文章保留的证据文件，不属于业务实现。

![Cursor 验收命令输出](/images/cursor-small-task-workflow/step-04-terminal.png)

最后再做一次手动检查：打开页面，确认视觉确实变化。

![登录页修改后效果](/images/cursor-small-task-workflow/step-05-result.png)

这才算完成。

不是 Cursor 说“我已经完成”，也不是页面看起来顺眼，而是你能拿出证据：改了哪些文件，没改哪些契约，跑了什么验证，结果是什么。

## 可复制的完整提示词

如果你想直接照着练，可以用这套四段式提示词。

第一段：分析。

```text
先不要改代码。

请只阅读与这个需求相关的文件，告诉我：
1. 这个功能涉及哪些文件；
2. 哪些文件负责视觉，哪些文件负责业务逻辑；
3. 如果只做这次需求，不应该碰哪些文件、字段、接口或行为。
```

第二段：计划。

```text
请给出实现计划。

要求：
1. 按文件列出准备修改什么；
2. 明确哪些文件不改；
3. 标注可能影响业务逻辑的风险点；
4. 给出修改后的验证方式；
5. 先不要写代码，等我确认。
```

第三段：执行。

```text
可以执行。

只实现计划中已确认的部分。
不要做额外重构。
不要顺手修改计划之外的文件。
改完后停下来，列出实际改动文件、关键 diff 和建议验证命令。
```

第四段：验收。

```text
请基于当前改动帮我做一次验收说明：
1. 实际改了哪些文件；
2. 是否出现计划之外的改动；
3. 哪些业务契约保持不变；
4. 我应该运行哪些测试或命令；
5. 如果要回滚，应该回滚哪些改动。
```

这套提示词不炫技，也不追求所谓“神级 prompt”。它的价值只有一个：把 Agent 的行动拆到你能审查的粒度。

## 最后给一份自检模板

以后你给 Cursor 一个小需求，可以按这个模板过一遍。

```text
【任务目标】
我要改什么：
这次只允许改什么：
这次明确不允许改什么：

【分析阶段】
Cursor 是否先分析而不是直接改代码：是 / 否
它是否识别出相关文件：是 / 否
它是否识别出禁区：是 / 否

【计划阶段】
计划是否按文件列出：是 / 否
计划是否写清不改哪些文件：是 / 否
计划是否包含验证方式：是 / 否

【执行阶段】
实际改动文件：
是否出现计划之外的文件：是 / 否
如果有，原因是什么：

【验收阶段】
已运行命令：
命令结果：
手动检查路径：
是否保留可回滚状态：是 / 否

【结论】
这次改动能否合并：能 / 不能
不能合并的原因：
```

很多人用 Cursor 的问题，不是不会提问，而是太早进入“让它干活”的状态。

真正稳的用法是反过来：先让它说清楚，再让它动手；先让它证明没越界，再考虑继续。

Cursor 可以很快。

但快不是第一目标。可控才是。
