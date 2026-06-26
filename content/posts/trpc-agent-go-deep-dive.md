---
title: "tRPC-Agent-Go 深度实践：生产级 Agent 的工程落地之道"
description: "GraphAgent 检查点机制、Human-in-the-Loop、Skills 可复用工作流、A2A/MCP 协议集成、可观测性与评测——tRPC-Agent-Go 的生产级特性全面解析。"
date: 2026-06-27T10:00:00+08:00
draft: false
author: "任博"
tags: [tRPC-Agent-Go, Go, Agent, GraphAgent, Skills, A2A, MCP, 生产环境]
categories: ["技术实战"]
toc: true
---

## 从 Demo 到生产

跑通 tRPC-Agent-Go 的 Demo 只需 5 分钟，但上线生产，真正的工程问题才开始浮现：

Agent 跑了 5 分钟、调了十几个工具，网络抖了一下——整个流程要重来。审批类场景下，Agent 分析完需要人点头，但很多框架把 Agent 设计成全自动黑盒，压根没有中断点。团队有 30 个 Agent，每个挂不同的 prompt 和脚本，靠什么管理？其他团队的 Agent 服务（也许是 Python 的 ADK）怎么调？外部的 MCP 工具怎么接？

tRPC-Agent-Go 的三个核心深度特性正好回答这些问题：**GraphAgent 检查点**（状态持久化与恢复）、**Skills 系统**（可复用工作流）、**A2A + MCP 协议集成**（Agent 与外界通信）。

---

## GraphAgent 检查点与 Human-in-the-Loop

GraphAgent 是 tRPC-Agent-Go 最大的差异化能力。但没有检查点的图工作流是「玩具」——跑一半断了就是真断了。

检查点支持两种持久化模式：**谱系持久化**（保存节点执行轨迹，适合审计合规）和**原子持久化**（保存完整状态快照，适合长耗时的数据处理流水线）。启用方式：

```go
sg := graph.NewStateGraph(graph.MessagesStateSchema())
sg.AddNode("analyze", analyzeNode)
sg.AddNode("approve", approveNode)
sg.AddNode("execute", executeNode)
sg.AddEdge("analyze", "approve")
sg.AddConditionalEdges("approve", approvalRouter, map[string]string{
    "approved": "execute",
    "rejected": "__end__",
})

cp := checkpointer.NewInMemoryCheckpointer()
ga := graphagent.New("audit-workflow",
    graphagent.WithStateGraph(sg),
    graphagent.WithCheckpointer(cp),
)
```

每个节点执行后框架自动持久化。恢复执行时传入检查点 ID：

```go
events, err := runner.Run(ctx, "user-1", "session-1",
    model.NewUserMessage("继续审批流程"),
    agent.WithResumeFromCheckpoint("cp-abc123"),
)
```

这就是所谓的「时间旅行」——从任意检查点恢复，跳过已完成节点。

### Human-in-the-Loop

审批流的工程实现：节点返回等待状态后 Runner 阻塞，通过新的 Run 调用传递人工决策：

```go
approveNode := func(ctx context.Context, s graph.State) (any, error) {
    return graph.State{"status": "awaiting_approval"}, nil
}

// 用户提交审批结果
runner.Run(ctx, "user-1", "session-1",
    model.NewUserMessage("已批准，按方案A执行"),
)
```

与 LangGraph 的 `interrupt` 类似，但 tRPC-Agent-Go 不需要显式声明中断点——节点返回特定状态后框架自然等待。如果你是 Go 技术栈且需要审批、内容审核、人机协同类的 Agent 系统，**这是目前 Go 生态里唯一成熟的选择**。

---

## Skills 系统——可复用的 Agent 工作流

Skills 的核心价值不是「又一个技能框架」，而是**让 Agent 能力像配置一样管理**。

### 三层信息模型

为什么需要三层？全量注入太贵，零注入又不够用：

概览层（Overlay）→ 每轮自动注入，成本几乎为零；正文层（Body）→ 模型通过 `skill_load` 按需加载；文档脚本层（Docs + Scripts）→ 按需注入，隔离环境执行。

每个技能是一个目录，核心是 `SKILL.md`：

```yaml
---
name: report-summary
description: Summarize long reports into short bullet points.
---
Steps
1) Read input from $WORK_DIR/inputs/report.txt
2) Run: python3 scripts/summarize.py > $OUTPUT_DIR/summary.txt
3) Return summary and mention summary.txt
```

### 挂载技能仓库

```go
repo, _ := skill.NewFSRepository("./shared-skills", "./team-skills")
exec := local.New()

agent := llmagent.New("skills-assistant",
    llmagent.WithModel(openai.New("gpt-4o-mini")),
    llmagent.WithSkills(repo),
    llmagent.WithCodeExecutor(exec),
    // ⚠️ 关键：否则 Agent 输出的代码块会被自动执行
    llmagent.WithEnableCodeExecutionResponseProcessor(false),
)
```

**一个让你头疼的坑**：同时启用 `WithSkills` 和 `WithCodeExecutor` 时，必须关闭响应阶段代码执行处理器。否则 Agent 输出中的 Markdown 代码块会被框架自动执行——在 Skills 场景下这通常是灾难。

另外 `SkillLoadMode` 默认是 `turn`（每轮重加载），跨多轮对话需设为 `session`。

**Skills 适合谁？** 5 个以上 Agent 或需要非技术人员管理 prompt 的团队。如果你已在用 Anthropic Claude Skills，可以直接把仓库指给 tRPC-Agent-Go，零迁移成本。

---

## A2A + MCP：Agent 与外部世界的通信

A2A 和 MCP 解决的是不同问题。**A2A（Agent-to-Agent）是 Agent 间跨运行时通信**，基于 Google A2A 规范；**MCP（Model Context Protocol）是 Agent 访问外部工具/数据**，基于 Anthropic MCP 规范。在 tRPC-Agent-Go 中，MCP 被集成到 Tool 层，A2A 在 Server 层——两者互补而非竞争。

### 一行代码暴露 A2A 服务

```go
// Server
server, _ := a2aserver.New(
    a2aserver.WithHost(":8080"),
    a2aserver.WithAgent(agent, true), // true = streaming
)
server.Start()

// Client
import "trpc.group/trpc-go/trpc-a2a-go/client"
a2aClient, _ := client.NewA2AClient("http://remote-agent:8080/")
response, _ := a2aClient.SendMessage(ctx, params)
```

已验证与 Google ADK Python A2A Server 互通，支持流式输出、工具调用、代码执行——Go 服务里调用 Python Agent，反之亦然。

### 选 A2A 还是 MCP？

**MCP 就够了**：你要调的外部能力是工具/API，不是完整 Agent。绝大部分场景 MCP 能覆盖。

**需要 A2A**：你的系统中有多个独立部署的 Agent 服务需要互相调用，且可能不在同一个运行时——比如 Go + Python 混部。A2A 的跨语言互通是独有优势。

---

## 可观测性与评测

### OpenTelemetry + Langfuse

```go
import "trpc.group/trpc-go/trpc-agent-go/telemetry/langfuse"

clean, _ := langfuse.Start(ctx)
defer clean(ctx)
runner := runner.NewRunner("app", agent)
events, _ := runner.Run(ctx, "user-1", "session-1",
    model.NewUserMessage("Hello"),
    runner.WithSpanAttributes(
        attribute.String("langfuse.user.id", "user-1"),
        attribute.String("langfuse.session.id", "session-1"),
    ),
)
```

覆盖 LLM 调用耗时与 token 消耗、工具调用成功率、整次执行耗时。

### Evaluation

```go
evaluator, _ := evaluation.New("app", runner,
    evaluation.WithNumRuns(3),
)
defer evaluator.Close()
result, _ := evaluator.Evaluate(ctx, "math-basic")
```

内置静态匹配和 LLM Judge 两种评估器，支持 PromptIter 自动优化提示词。

### Prompt Caching

据官方数据缓存命中可节省 **90% token 成本**。在 Skills 场景下通过将 SKILL.md 内容物化到 tool result 消息提升缓存效率：

```go
llmagent.WithSkillsLoadedContentInToolResults(true)
```

---

## 选型建议

- **LLMAgent vs GraphAgent**：纯对话、简单工具调用用 LLMAgent 就够。任何需要状态机、条件路由、检查点恢复、Human-in-the-Loop 的场景——上 GraphAgent。
- **A2A vs MCP**：绝大部分场景 MCP 覆盖了工具接入需求。只有需要与独立部署的 Agent 服务跨运行时协作时才上 A2A。
- **Skills 采纳时机**：5 个 Agent 以下用 Instruction + Tools 硬编码就行。5 个以上或需要热加载能力时，Skills 的价值开始体现。

---

## 总评

项目目前 GitHub Stars 约 **1.4k**，Forks **178**，50+ 贡献者，13 个月 115 个 Release。已在腾讯元宝、腾讯视频、腾讯新闻、IMA、QQ 音乐等内部业务线生产验证。

我会不会在生产里用？
- **Go 技术栈后端团队需要构建生产级 Agent 服务**——答案是肯定的。目前 Go 生态里没有第二个框架提供同样完整的能力栈。
- **已有 Python Agent 系统且无 Go 化需求**——迁移成本不值，LangChain 生态更成熟。
- **需要审批、人机协同类图工作流**——tRPC-Agent-Go 的 GraphAgent + Checkpointer 是 Go 生态的唯一成熟方案。

对比字节跳动 Eino 和社区版 LangChain Go，tRPC-Agent-Go 在功能完整性上遥遥领先。短板在于社区生态仍在早期——第三方集成少、中文教程少。但核心产品力已经足够扎实。

**一句话**：它是 Go Agent 框架里最能打的，但它的上限取决于社区能否尽快长起来。
