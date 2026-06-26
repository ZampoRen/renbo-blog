---
title: "tRPC-Agent-Go 快速上手：Go 生态终于有了自己的 Agent 框架"
description: "腾讯 tRPC 团队出品的 Go 语言 Agent 框架全面分析——5 分钟上手、五种 Agent 类型、GraphAgent 对标 LangGraph、与 Python 框架怎么选。"
date: 2026-06-26T10:00:00+08:00
draft: false
author: "任博"
tags: [Go, tRPC-Agent-Go, Agent, LLM, LangChain, GraphAgent, AI]
categories: ["技术实战"]
cover: "/images/trpc-agent-go-quickstart/cover.png"
toc: true
---

## 引子：Go 生态的 Agent 空白

如果你是一个 Go 后端开发者，过去两年一定被各种 AI Agent 框架刷过屏。LangChain、CrewAI、AutoGen、LangGraph…… 清一色 Python。而你手上的 Go 项目要做 Agent 能力，选项屈指可数：要么嵌一个 Python 子进程（运维噩梦），要么自己手搓胶水代码（重复造轮子），要么指望社区那几个半成品框架。

这个局面在 2025 年中开始改变。腾讯 tRPC 团队开源了 **tRPC-Agent-Go**，到今天刚好 13 个月，115 个 Release，1.4k Stars，已经在腾讯元宝、腾讯视频、腾讯新闻等业务线跑过了生产流量。

这篇文章的目标很简单：**用 10 分钟让你知道 tRPC-Agent-Go 是什么、能不能用、怎么上手**。第二篇再深入工程落地。

---

## tRPC-Agent-Go 是什么

一句话：**tRPC-Agent-Go 是腾讯 tRPC 团队出品的 Go 语言生产级 Agent 框架**，对标 Python 生态的 LangChain / LangGraph / CrewAI。

它的定位不是"Go 版的 LangChain"，而是自顶向下重新设计：以 Agent 接口为核心执行单元，以图工作流为高级编排手段，以事件流为通信载体。覆盖了单 Agent 对话、多 Agent 协作、图工作流、MCP/A2A/AG-UI 协议、RAG、可观测性、评测等全栈能力。

关键背景数据：

- 腾讯 tRPC 是覆盖近 200w+ 节点、5w+ 服务的内部 RPC 框架，tRPC-Agent-Go 是其 AI 方向的延伸
- 已生产验证于腾讯元宝、腾讯视频、腾讯新闻、IMA、QQ 音乐等业务
- 要求 Go 1.21+，Apache-2.0 协议
- **不强制依赖 tRPC 框架**，可以独立使用

---

## 核心概念速览（点到为止）

tRPC-Agent-Go 的核心抽象只有五个 Agent 类型 + 一个 Runner + Tool + Memory，一看就懂。

### 五种 Agent 类型

| Agent 类型 | 一句话 | 类比 |
|-----------|--------|------|
| **LLMAgent** | 标准 LLM 对话 + 工具调用循环 | 类似 OpenAI Assistants API |
| **ChainAgent** | 子 Agent 顺序执行，前输出 = 后输入 | 类似 LangChain Sequential Chain |
| **ParallelAgent** | 子 Agent 并发执行，结果合并 | 类似 LangGraph Parallel |
| **CycleAgent** | Planner + Executor 循环迭代 | 类似 AutoGen 的反思模式 |
| **GraphAgent** | 图工作流编排，有状态、有路由 | 对标 LangGraph |

所有 Agent 实现同一个接口：

```go
type Agent interface {
    Info() Info
    Run(ctx context.Context, req *Request, opts ...Option) (<-chan *Event, error)
}
```

### Runner

Runner 是 Agent 的执行环境，负责生命周期管理、会话状态维护、Memory 注入、事件流收发。调用方得到的是一个 `<-chan *Event` Go channel —— 天然支持背压，天然流式。

### Tool

Tool 是 Agent 与外部世界的桥梁。框架内置了 Function Tool、MCP Tool（兼容 Anthropic MCP 规范）、DuckDuckGo 搜索等。自定义工具只需要实现一个接口：

```go
type Tool interface {
    Name() string
    Description() *FuncSchema
    Call(ctx context.Context, input map[string]any) (any, error)
}
```

### Memory

Memory 管理用户的长期记忆。分为工具驱动模式（Agent 自主决定存取记忆）和自动提取模式（系统后台静默提取对话要点）。**推荐自动提取模式**，用户无感知。

> Skills 系统也值得一提——它完全兼容 Anthropic Agent Skills 规范，你可以直接把 Claude 技能仓库指给 tRPC-Agent-Go 用。但这是第二篇的重点，这里先按下不表。

---

## 5 分钟上手：跑通第一个 LLMAgent + Tool 调用

下面这段代码你直接 copy 到一个 Go 文件里，配置好 API Key 就能跑。

### 前置条件

- Go 1.21+
- 一个 OpenAI 兼容的 API Key（DeepSeek / 通义千问 / OpenAI 均可）
- 确保 `GOPROXY` 能访问 `trpc.group`（建议 `export GOPROXY=https://goproxy.io,direct`）

### 完整代码

```go
package main

import (
	"context"
	"fmt"
	"log"
	"math"

	"trpc.group/trpc-go/trpc-agent-go/agent/llmagent"
	"trpc.group/trpc-go/trpc-agent-go/event"
	"trpc.group/trpc-go/trpc-agent-go/model"
	"trpc.group/trpc-go/trpc-agent-go/model/openai"
	"trpc.group/trpc-go/trpc-agent-go/runner"
	"trpc.group/trpc-go/trpc-agent-go/tool"
	"trpc.group/trpc-go/trpc-agent-go/tool/function"
)

// 计算器函数，后面注册为 Tool
func calculate(_ context.Context, params map[string]any) (any, error) {
	a := params["a"].(float64)
	b := params["b"].(float64)
	op := params["op"].(string)
	switch op {
	case "add":
		return a + b, nil
	case "sub":
		return a - b, nil
	case "mul":
		return a * b, nil
	case "div":
		if b == 0 {
			return nil, fmt.Errorf("division by zero")
		}
		return a / b, nil
	case "pow":
		return math.Pow(a, b), nil
	default:
		return nil, fmt.Errorf("unknown operator: %s", op)
	}
}

func main() {
	ctx := context.Background()

	// 1. 创建模型（OpenAI 兼容接口）
	// API Key 通过环境变量 OPENAI_API_KEY 传入
	// 用 DeepSeek 则设 DEEPSEEK_API_KEY，传 openai.WithVariant("deepseek")
	model := openai.New("gpt-4o-mini")

	// 2. 用标准 Go 函数创建一个计算器 Tool
	calcTool := function.NewFunctionTool(
		calculate,
		function.WithName("calculator"),
		function.WithDescription("四则运算。参数：a,b 为数值，op 取值 add/sub/mul/div/pow"),
	)

	// 3. 创建 LLMAgent
	agent := llmagent.New("math-assistant",
		llmagent.WithModel(model),
		llmagent.WithTools([]tool.Tool{calcTool}),
		llmagent.WithInstruction("你是一个数学助手，使用 calculator 工具处理所有计算问题。"),
		llmagent.WithMaxToolRound(5), // 防止无限工具调用循环
	)

	// 4. 创建 Runner
	r := runner.NewRunner("demo-app", agent)

	// 5. 运行，传入结构化消息
	events, err := r.Run(ctx, "user-1", "session-1",
		model.NewUserMessage("计算 1234 × 5678 等于多少？"),
	)
	if err != nil {
		log.Fatal(err)
	}

	// 6. 消费事件流
	for evt := range events {
		if evt.Error != nil {
			fmt.Printf("❌ 错误: %s\n", evt.Error.Message)
			continue
		}
		if evt.Response == nil || len(evt.Response.Choices) == 0 {
			continue
		}
		choice := evt.Response.Choices[0]

		switch evt.Response.Object {
		case "chat.completion.chunk":
			// 流式文本块
			fmt.Print(choice.Delta.Content)
		case "chat.completion":
			fmt.Println("\n--- 最终响应 ---")
			fmt.Println(choice.Message.Content)
		default:
			// tool.response 等事件静默处理
		}

		// 检测工具调用
		if len(choice.Message.ToolCalls) > 0 {
			for _, tc := range choice.Message.ToolCalls {
				fmt.Printf("🔧 调用工具: %s\n  参数: %s\n",
					tc.Function.Name, string(tc.Function.Arguments))
			}
		}
	}
}
```

### 预期输出

你会看到事件流依次输出：LLM 流式文本 → 工具调用请求 → 最终 LLM 响应。大致像这样：

```
1234 × 5678 的计算结果为：
🔧 调用工具: calculator
  参数: {"a":1234,"b":5678,"op":"mul"}

--- 最终响应 ---
计算 1234 × 5678 = 7,006,652。
```

**从 `go get` 到跑通，按我的实测只要 5 分钟**——前提是网络没问题。如果你在中国大陆，注意设置 `GOPROXY=https://goproxy.cn,https://goproxy.io,direct`。

---

## GraphAgent 亮点：tRPC-Agent-Go 的最大差异化能力

如果说 LLMAgent 对标的是 OpenAI Assistants API，那 **GraphAgent 就是 tRPC-Agent-Go 用来正面硬刚 LangGraph 的武器**。

它的核心设计是 **"Graph as Agent"**——既是图工作流引擎，也实现了统一的 Agent 接口，可以被 Runner 调度、被其他 Agent 嵌套。这不是一个"工作流库加了个 Agent 壳"，而是从底层就把图执行和 Agent 生命周期融合到一起。

### 相比 LangGraph 的核心差异

| 维度 | GraphAgent | LangGraph |
|------|-----------|-----------|
| 执行模型 | BSP（Bulk Synchronous Parallel）超级步 | Pregel 风格 |
| 多条件扇出 | ✅ 原生支持 | 需手动处理 |
| 工具路由 | ✅ 内置 `AddToolsConditionalEdges` | 需自行判断 |
| A2A 集成 | ✅ 图中可直接调用远程 Agent | 无原生支持 |
| Agent 嵌套 | ✅ 图中嵌入 LLMAgent/CycleAgent 等 | ✅ |

### 一段真实可用的条件路由代码

下面用 GraphAgent 实现一个**情感分析路由**：分析用户输入的情感，正面走"积极处理"流程，负面走"升级处理"流程。

```go
package main

import (
	"context"
	"fmt"
	"log"

	"trpc.group/trpc-go/trpc-agent-go/agent/graphagent"
	"trpc.group/trpc-go/trpc-agent-go/event"
	"trpc.group/trpc-go/trpc-agent-go/graph"
	"trpc.group/trpc-go/trpc-agent-go/model"
	"trpc.group/trpc-go/trpc-agent-go/runner"
)

func main() {
	ctx := context.Background()

	// 1. 创建状态图（用默认消息结构）
	sg := graph.NewStateGraph(graph.MessagesStateSchema())

	// 2. 添加节点（每个节点是一个处理函数）
	sg.AddNode("analyze", func(ctx context.Context, s graph.State) (any, error) {
		// 简化的情感判定
		msg := fmt.Sprintf("%v", s["messages"])
		if len(msg) > 10 {
			return graph.State{"sentiment": "positive"}, nil
		}
		return graph.State{"sentiment": "negative"}, nil
	})

	sg.AddNode("handle-positive", func(ctx context.Context, s graph.State) (any, error) {
		fmt.Println("✅ 积极反馈，正常回复")
		return graph.State{"response": "收到您的正面反馈，感谢支持！"}, nil
	})

	sg.AddNode("handle-negative", func(ctx context.Context, s graph.State) (any, error) {
		fmt.Println("⚠️ 负面反馈，升级处理")
		return graph.State{
			"response": "已升级到人工客服，将在24小时内联系您。",
			"priority": "high",
		}, nil
	})

	// 3. 设置条件路由 — 根据 sentiment 走不同分支
	sg.AddConditionalEdges("analyze",
		func(ctx context.Context, s graph.State) (string, error) {
			sentiment := s["sentiment"].(string)
			if sentiment == "positive" {
				return "handle-positive", nil
			}
			return "handle-negative", nil
		},
		map[string]string{
			"handle-positive": "positive-processor",
			"handle-negative": "negative-processor",
		},
	)

	// 4. 设置入口和出口
	sg.SetEntryPoint("analyze")
	sg.SetFinishPoint("handle-positive")
	sg.SetFinishPoint("handle-negative")

	// 5. 创建 GraphAgent
	agent := graphagent.New("sentiment-router",
		graphagent.WithStateGraph(sg),
	)

	r := runner.NewRunner("demo", agent)
	events, err := r.Run(ctx, "user-1", "session-1",
		model.NewUserMessage("你们的产品太棒了！"),
	)
	if err != nil {
		log.Fatal(err)
	}

	for evt := range events {
		if evt.Error != nil {
			fmt.Printf("❌ 错误: %s\n", evt.Error.Message)
			continue
		}
		if evt.Response == nil || len(evt.Response.Choices) == 0 {
			continue
		}
		if evt.Response.Object == "chat.completion" {
			fmt.Println("最终响应:", evt.Response.Choices[0].Message.Content)
		}
	}
}
```

看到没？**条件路由的逻辑和 LangGraph 几乎一比一对应**，但写的是 Go——静态类型编译检查、单二进制部署、goroutine 原生并发。

GraphAgent 还支持 **Human-in-the-Loop**（人工审批中断点）、**检查点 + 时间旅行**（回溯到任意步骤重放）、**多条件扇出**（一个节点并行触发多条分支）。这些是典型的 BFS/审批工作流场景，在 Go 生态中目前只有 tRPC-Agent-Go 能做到。

---

## 与 Python 框架怎么选

这个问题我经常被问到。直接给结论：**如果你团队的技术栈是 Go，选 tRPC-Agent-Go。如果你的团队全是 Python，别硬转，继续用 LangChain。** 下面是选型对比表：

| 维度 | tRPC-Agent-Go | LangChain / LangGraph | CrewAI |
|------|:------------:|:---------------------:|:------:|
| 语言 | Go | Python | Python |
| 部署形态 | 单二进制 | Python 运行时 | Python 运行时 |
| 并发模型 | goroutine（原生） | asyncio | asyncio |
| 图工作流 | ✅ GraphAgent (BSP) | ✅ LangGraph (Pregel) | ❌ |
| A2A 协议 | ✅ 原生支持 | ❌ | ❌ |
| MCP 协议 | ✅ | ✅ | ✅ |
| Skills (Anthropic 兼容) | ✅ | ❌ | ❌ |
| OpenTelemetry 可观测 | ✅ | ❌ (走 LangSmith) | ❌ |
| 评测系统 | ✅ EvalSet + Metric | ✅ LangSmith | ❌ |
| 学习曲线 | 中等 | 陡峭（LangGraph） | 低 |
| 生态规模 | 1.4k stars | 100k+ / 97k stars | 20k+ |
| 生产验证 | ✅ 腾讯内部多个业务 | ✅ 全球广泛 | ✅ 部分场景 |

### 我推荐这么选

- **你的团队主要写 Go** → tRPC-Agent-Go，没有悬念。跨语言调用 Agent 的开销远大于框架本身的任何特性差异。
- **你是微服务架构，Agent 需要作为独立服务部署** → tRPC-Agent-Go，单二进制部署优势巨大。
- **你需要低成本跑通一个 PoC** → Python LangChain / CrewAI，**快速验证**。
- **你需要审批工作流、有状态图编排、Human-in-the-Loop** → tRPC-Agent-Go 的 GraphAgent 是目前完成度最高的方案，无论语言。
- **你在腾讯云 / tRPC 生态内** → tRPC-Agent-Go，深度集成。
- **你需要 Claude / Gemini 原生 API（不走兼容层）** → 目前暂时用 Python，tRPC-Agent-Go 的 Model 层目前仅 OpenAI 兼容接口。

---

## 一句话总结

**tRPC-Agent-Go 适合：** Go 技术栈、微服务体系、需要生产级 Agent 能力（图编排 / 可观测 / 评测 / RAG）的团队。

**不适合：** 纯 Python 团队、快速原型阶段、需要非 OpenAI 模型原生支持的场景。

项目目前仍处于快速迭代期（13 个月 115 个版本），有腾讯内部业务压舱，社区的坑也有人在填。第一篇先到这里，第二篇我会深入 **生产级工程落地**——GraphAgent 的检查点机制、Skills 系统的完整用法、如何与现有微服务集成，敬请期待。

> 参考链接：[GitHub](https://github.com/trpc-group/trpc-agent-go) | [文档站](https://trpc-group.github.io/trpc-agent-go/) | [GraphAgent 详解](https://trpc-group.github.io/trpc-agent-go/zh/blog/graphagent/)
