---
title: "God Prompt → MCP → Agent Skills：AI Agent 工程的三次架构演进"
description: "从单体式 system prompt 到标准化工具连接，再到行为模块化——用一个真实使用者的视角，梳理 AI Agent 工程过去两年经历的三次架构演进，并给出清晰的选型判断。"
date: "2026-06-28"
draft: false
author: "任博"
tags:
  - AI Agent
  - God Prompt
  - MCP
  - Agent Skills
  - 架构演进
  - Prompt Engineering
  - Hermes
  - tRPC-Agent-Go
categories:
  - AI 工程
cover: "/images/agent-skills-evolution/cover.png"
toc: true
---

## 引子

前几天看到一个视频，标题很猛："Anthropic just made AI agents 10X better with new standard"——讲的是 agentskills.io。评论区一片叫好，但很少有人问一个更根本的问题：为什么我们需要一个新标准？

Anthropic 发过的标准不少了。MCP 是其中一个，去年底发布的 agentskills.io 是另一个。但这两个标准解决的根本不是同一类问题——MCP 解决的是连接问题，Skills 解决的是行为问题。而且，这中间还有一整个被大多数人忽略的时代：God Prompt 时代。

如果你只用 1 个 Agent，写个 2000 字的 system prompt 完全够用。demo 能跑，生产也能跑，顶多偶尔出点小毛病。但当你开始维护 5 个、10 个 Agent 时，God Prompt 的维护成本会指数级上升——这不是模型能力问题，是工程架构问题。

这篇文章不讲 agentskills.io 的规范细节（感兴趣自己去看官网），而是用我这两年实际使用和构建 Agent 系统的体感，讲三个阶段各自对应的架构思维，以及你现在到底该用什么。

---

## 第一块：God Prompt 时代——Monolith

### 为什么大家都这么写

2024 年初我刚接触 Agent 开发时，第一个 reflex 就是往 system prompt 里堆东西。这不是我懒，是整个生态都在这么做。LangChain 的 AgentExecutor、早期的 Claude function calling demo、各种开源 Agent 模板——没有一个不是在示范"把所有逻辑写进 prompt"。

理由很朴素：简单、直接、见效快。

你不需要写框架，不需要设计接口，不需要考虑复用。三个工具、两个工作流、五条边界规则——一个 markdown 文件全搞定。改行为就是改 prompt，不用重新编译、不用部署、不用走 CI。对于原型验证阶段来说，这几乎是完美的方案。

我自己就写过那种 3000+ 字的 system prompt。开头是人格设定和通用原则，中间是工具定义和调用规则，结尾是输出格式和边缘 case。写完之后跑 demo，效果惊艳——模型好像真的"理解了"整个业务逻辑。

### Lost in the Middle——看不见的陷阱

但问题很快出现了。随着 prompt 越来越长，模型开始在一些中间位置漏掉指令。

这不是玄学。Transformer 的注意力机制呈 U 形曲线——prompt 开头和结尾的内容被很好关注，中间部分被系统性忽略。Liu et al.（2024, Stanford）的研究清楚表明：当关键信息位于长 prompt 的中间位置时，模型的命中率显著下降。

我遇到过一个实际场景：一个 5000 tokens 的 system prompt，中间位置有一段关于"当用户请求数据导出时，先检查权限再执行"的指令。模型在对话早期还记得，但几轮对话之后就开始直接执行导出而跳过权限检查。调试了两个小时才发现不是模型变笨了，是指令被"埋"在 attention 的盲区了。

### 最隐蔽的问题——团队协作灾难

如果 God Prompt 只有技术问题，其实还好——加钱换更大的 context window 能缓解。真正致命的问题是工程管理层面的。

让我用一个简单的思想实验来说明：假设你在维护一个 4000 字的 God Prompt，里面包含了 15 个工具的使用规则、8 个工作流步骤、20 条边缘 case 处理。现在产品经理说"在 A 场景下加一个确认步骤"。你改了一行，然后呢？

你根本不敢保证这一行不会导致 B 场景的 regression。因为你无法单独测试某一个子行为——整个 Agent 的行为由一段连续的 prompt 文本定义，没有模块边界，没有接口契约，没有单元测试。

结果是：没有人敢改 God Prompt。它变成了团队里的"圣物"——只有当初写它的人才敢碰，其他人能绕就绕。这跟后端单体应用的困境如出一辙：一个巨大的文件，所有人害怕修改，因为不知道改了哪里会引发什么连锁反应。

社区把这种状态叫做"token archaeology"——在 100K 的 context window 里翻找，期望找到问题出在哪一行。这其实不是调试，是考古。

**一个分界线：** 如果你的团队只有 1-2 个 Agent，并且不打算扩展到更多，God Prompt 完全够用。但一旦你开始维护 3 个以上的 Agent，或者同一个 Agent 需要处理超过 5 个不同类型的任务，你就需要考虑下一步了。

---

## 第二块：MCP 时代——微服务化

### 标准化的不是行为，是连接

2024 年 11 月，Anthropic 发布了 MCP（Model Context Protocol），被形象地称为"AI 界的 USB-C"。

MCP 解决了一个很实在的问题：在它出现之前，给 LLM 接入外部工具是各自为战的噩梦。GitHub 要写一套 API 包装器，Slack 要写一套，数据库要写一套——每个数据源都需要自定义集成代码，而且每家的写法都不一样。

MCP 的回答是：统一用 JSON-RPC 2.0 协议暴露工具、资源和提示词模板。不管你背后是文件系统、数据库还是云平台，对外暴露的接口风格是一样的。

### MCP 做到了什么

MCP 最直接的好处是工具可以复用了。

在我日常使用的 Hermes Agent 里，MCP server 就是一个独立的进程，通过 STDIO 或 HTTP 与 Agent 通信。想给 Agent 加一个 GitHub 工具？写一个 MCP server，注册到 Agent 配置里，它就能用了。不用改一行系统 prompt。

同样，在 tRPC-Agent-Go 里，`trpc-mcp-go` 提供了 struct-first 的 API 定义方式，Go 开发者可以很自然地定义 MCP 工具，自动生成 schema。

这带来的工程改进是实质性的：

1. 工具可以独立维护——GitHub MCP server 的 bug 不会影响数据库 MCP server
2. 工具可以复用——同一个 MCP server 可以给多个 Agent 使用
3. 工具可以被测试——每个 MCP server 可以独立验证其行为

### MCP 的边界——行为仍然靠塞 prompt

但是，MCP 有一个很明确的边界：**它标准化了连接，但没有标准化行为。**

这句话什么意思？让我用一个真实的例子来说明。

假设你给 Agent 接上了 DataDog MCP server 和 Kubernetes MCP server。现在 Agent 可以查询 DataDog 的指标、可以执行 kubectl 命令。但当你对它说"诊断一下 Kafka consumer lag 为什么突然飙升"，它仍然需要知道：

- 先查什么后查什么（工作流）
- 哪些指标真正重要（领域知识）
- 需要交叉引用哪些数据（推理路径）

这些行为定义，MCP 不管。开发者又回到了老路：**把工作流塞进 system prompt。**

于是出现了一个有趣的现象：MCP 让工具层变得干净了，但 God Prompt 在行为层死灰复燃。工具连接变成了微服务，但行为定义仍然是一个单体。

社区有人说得特别好："MCP is just raw plumbing."——它把管道铺好了，但 Agent 怎么用这些管道去完成一个具体任务，MCP 没有定义。

MCP 不是错的——它是必要的中间步骤。就像后端架构从单体演进到微服务时，你首先需要标准化服务间的通信协议（HTTP/gRPC），然后才会思考"服务边界应该怎么划分"。MCP 就是这个通信协议。

---

## 第三块：Agent Skills 时代——领域驱动

### 三层信息模型不是技术设计，是架构取舍

2025 年 10 月，Anthropic 发布了 Agent Skills（Claude 产品线），12 月开放为 agentskills.io 标准。到 2026 年中，几乎所有主流 Agent 工具——Cursor、Copilot、Gemini CLI、Claude Code——都已经支持 SKILL.md。

Skills 的核心设计是三层渐进式披露（Progressive Disclosure）：

```
Level 1: Metadata  ─── name + description（~100 tokens，始终在 context 中）
Level 2: Body      ─── SKILL.md 正文（<5000 tokens，被触发时加载）
Level 3: Resources ── scripts/ + references/ + assets/（按需加载，可以无限大）
```

这个设计看起来像个技术细节，但它背后是一个深刻的架构取舍：**不是所有信息都需要同时出现在 context 里。**

God Prompt 的做法是"一次加载，全部可见"——把所有工具定义、工作流规则、边缘 case 一股脑塞进 system prompt。它在 token 经济上是极其低效的，因为大多数指令在大多数对话中用不到。

Skills 的做法是"按需加载，渐进披露"——Agent 只看到每个 skill 的名称和一句话描述（~100 tokens），当某个 task 需要时，再通过工具调用加载完整的 SKILL.md 正文。这就像后端接口的分页查询，而不是全表扫描。

### 从两套 Skills 系统的实战体感

我在过去半年深度使用了两套 Skills 系统：Hermes Agent 和 tRPC-Agent-Go。虽然它们的实现语言和架构完全不同，但设计思路惊人地一致。

**Hermes Skills** 的核心哲学是："Skills are the agent's procedural memory"——Skill 不是一段被动的 prompt 文本，而是 Agent 的"过程性记忆"，记录了"如何完成某一类特定任务"。最有价值的一点是：Hermes 的 Agent 在完成复杂任务后，可以**自主创建 skill**，通过 `/learn` 命令从对话记录中提取可复用的工作流。这就是"self-improving"的核心机制——Agent 把自己的成功经验固化为可复用的 skill，下次遇到类似场景时自动触发。

我实际用下来，最有价值的使用场景是 write approval gate。Agent 自主创建的 skill 不会直接生效，而是先暂存等待人工审核——这种"提交 PR"的模式保证了质量，又保留了 Agent 自我进化的能力。

**tRPC-Agent-Go Skills** 是我最近在 Go 生态中使用的一套设计。它有三层加载机制（概览 → Body → Docs，通过工具返回值传递而不是 prompt 注入），以及交互模式切换（Legacy vs Stateless）。Stateless 模式的设计尤其巧妙——body 通过 tool result 返回而非追加到 system message 中，这样 system prompt 的 cache 前缀不会被破坏，在反复调用时显著节省 token。

它还实现了 Evolution 模块——审查已完成会话，将可复用的流程提取为 managed Skill。这与 Hermes 的自主创建 skill 是同一个理念：**最好的 skill 不是人写的，是 Agent 自己在实践中发现并固化的。**

### 核心差异：不是 prompt 更短了，是行为可组合了

Skills 真正改变的，不是 prompt 的长度，而是行为的可组合性。

在 God Prompt 时代，你有一个巨大的 system prompt，里面包含了 Agent 所有的行为定义。新增一个行为就是在 prompt 里加一段文本。修改一个行为就是编辑原有段落。这本质上是**线性叠加**——所有行为的定义纠缠在同一个文本空间里。

在 Skills 时代，你有多个独立的 SKILL.md 文件，每个 skill 定义了一个明确的行为边界。Agent 可以根据任务的语义自动选择和组合多个 skill。新增一个行为就是创建一个新的 SKILL.md。修改一个行为就是编辑对应的 skill 文件。这本质上是**模块组合**——每个行为的定义隔离在独立的文件空间里。

这个差异在团队协作中尤为明显。God Prompt 只能串行——只有一个人能编辑它。Skills 可以并行——不同团队维护不同的 skill，各自独立迭代、独立测试、独立发布。

用后端架构的类比来说：

| 后端演进 | Agent 演进 |
|---------|-----------|
| 单体应用 | God Prompt |
| 微服务 + 服务发现 | MCP（连接标准化） |
| 领域驱动设计 | Skills（行为模块化） |

"Your God Prompt Is the New Monolith"——这句话不是比喻，是事实。

---

## 第四块：现在该用什么

理论说了这么多，回到最实际的问题：你现在到底该用什么？

我自己的判断标准是一个简单的决策矩阵：

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| **1-2 个 Agent，简单任务** | God Prompt | 简单场景不需要过度设计。3-5 个工具、1-2 个固定工作流，精心设计的 system prompt 比引入 Skills 系统更经济 |
| **3-10 个 Agent，专用领域** | Skills + MCP | MCP 管"能连什么"，Skills 管"怎么做"。这是当前性价比最高的组合 |
| **10+ Agent，生产级系统** | Skills + MCP + 自建 Skill Registry | 需要集中管理、版本控制、权限控制的 skill 注册中心。考虑 gh skill install 或自建 git 仓库 |
| **需要连接外部工具** | 引入 MCP | Slack、GitHub、数据库、云平台——MCP 是现在最标准的方式 |
| **Go 技术栈** | tRPC-MCP-Go + tRPC-Agent-Go | Go 生态中唯一同时支持 MCP 和 agentskills.io 标准的框架组合 |

### 实操建议

1. **从最小集开始。** 不要一上来就搞 20 个 skill。先写 3-5 个最常用的工作流作为 skill，观察 Agent 的触发准确率。满意了再扩。

2. **Skill 要短。** SKILL.md 正文建议 < 500 行。如果更长，把细节移到 references/ 目录下。一个 skill 太长说明它做的事太多了——考虑拆分成多个 skill。

3. **让 Agent 自己写 Skill。** 最有价值的 skill 往往来自 Agent 在对话中发现的重复模式。Hermes 的 `/learn` 和 tRPC-Agent-Go 的 Evolution 模块都是值得借鉴的设计。Agent 比你更清楚什么模式是重复出现的。

4. **团队共享。** 把 skill 目录提交到 git 仓库。这和共享 .cursorrules 或 .claude/commands 是同一路径——只是规模更大、能力更强。

5. **MCP 与 Skills 不要二选一。** 两者互补，解决不同层次的问题。MCP 是"能做什么"，Skills 是"怎么做"。一个完整的 Agent 系统两者都需要。

---

## 结尾

写这篇文章的时候，我一直在想一个问题：God Prompt 是不是错的？

答案是否定的。

在只有 1-2 个 Agent 的场景下，God Prompt 仍然是最快、最经济的方案。它没有错——就像单体应用在项目初期不是错的选择一样。错的是在规模已经增长之后，仍然用单体架构去应对复杂度。

三次演进的关系不是替代，是需求推动的架构升级：

- **God Prompt** 让你快速验证 Agent 能力
- **MCP** 让工具接入变得标准和可复用
- **Skills** 让行为定义变得模块化和可组合

每一层都是在前一层基础上的自然演进。当你发现 God Prompt 开始难以维护时，你会自然想到需要 MCP 来清理工具层。当你发现 MCP 之后行为仍然靠塞 prompt 时，你会自然想到需要 Skills 来模块化行为。

这不是技术路线竞赛，而是一个开发者在大规模工程实践中会自然走过的认知路径。认清你在哪个阶段，选对方案，然后在该升级的时候果断升级——这就够了。
