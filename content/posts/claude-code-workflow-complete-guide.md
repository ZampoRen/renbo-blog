---
title: "Claude Code 工作流完全指南：从聊天工具到自动化工程系统"
date: 2026-04-07T14:30:00+08:00
draft: false
author: "任博"
description: "为什么同样的 AI 工具，有人用来聊天摸鱼，有人却能搭建自动化工程系统？差别不在于模型能力，而在于你有没有一套可靠的工作流。本文基于 2025 年最新架构，详解 CLAUDE.md、Skills、Subagents、Hooks、MCP 五大核心机制的完整工程实践。"
tags: ["AI 工程化", "Claude Code", "自动化", "工作流", "开发者工具"]
categories: ["技术实战"]
featured: true
toc: true
---

很多人用 Claude Code 的方式是这样的：

打开终端，输入 `claude`，然后开始聊天。

让它写个脚本，改个 bug，查个文档。能用，但总觉得差点意思——每次都要重新交代背景，同样的任务说一遍又一遍，输出质量时好时坏，稍微复杂点的任务就容易跑偏。

**问题不在模型，而在工作流。**

2025 年的 Claude Code 早已不是单纯的聊天工具。通过五层扩展机制——CLAUDE.md、Skills、Subagents、Hooks、MCP——你可以把它改造成一个可靠的自动化工程系统。

这篇文章不聊概念，只给能落地的工程实践框架。看完之后，你可以直接照着搭建自己的工作流。

<!--more-->

## 核心架构：五层扩展机制

先上一张架构图：

```
用户自然语言输入
        │
        ▼
┌─────────────────────────────────────────────┐
│              Claude Code 主 Agent            │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ CLAUDE.md │  │  Skills  │  │  /commands│ │
│  │ (项目宪法) │  │ (自动技能) │  │ (手动触发) │ │
│  └──────────┘  └──────────┘  └───────────┘ │
└──────────────────────┬──────────────────────┘
                       │ 任务拆解与调度
          ┌────────────┴────────────┐
          ▼                         ▼
┌──────────────────┐    ┌──────────────────────┐
│   Subagents 团队  │    │    MCP Tools 工具箱   │
│  @researcher     │    │  GitHub / Slack      │
│  @writer         │    │  PostgreSQL / Figma  │
│  @code-reviewer  │    │  自定义 MCP Server    │
└──────────────────┘    └──────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────┐
│              Hooks 事件系统                   │
│  PreToolUse → PostToolUse → Stop → SubagentStop│
│  状态写入 workflow_state.json                  │
└──────────────────────────────────────────────┘
```

用一句话概括这五层：

> **CLAUDE.md 是记忆，Skills 是知识，Commands 是流程，Subagents 是团队，Hooks 是纪律。**

五者协同，Claude Code 从聊天工具进化为可靠的自动化工程系统。

---

## 一、CLAUDE.md：项目宪法

这是 Claude 每次启动必读的"项目宪法"。

**位置：** `.claude/CLAUDE.md` 或根目录 `CLAUDE.md`

**核心作用：** 定义项目背景、规范、协作规则、禁止行为。

**一个示例：**

```markdown
# 项目：智能内容生产工作流

## 项目背景
这是一个自动化内容生产系统，用于生成技术博客、产品文档和营销文案。

## 核心规范
- 所有内容输出前必须经过事实核查
- 代码示例必须是可运行的，不写伪代码
- 文章字数：技术博客 1500-2500 字，产品文档 800-1200 字

## 工作流状态文件
每次执行任务，必须更新 `workflow_state.json`，格式：
{
  "task_id": "唯一 ID",
  "status": "planning|executing|reviewing|done|failed",
  "current_agent": "agent 名称",
  "progress": 0-100,
  "outputs": []
}

## Agent 协作规则
- researcher 先行，writer 后续，reviewer 最终把关
- 任务依赖：writer 必须等 researcher 完成后才能开始
- 所有 Agent 的结果写入 memory/ 目录供后续使用

## 禁止行为
- 不得在未完成 research 步骤时直接写内容
- 不得跳过 code-reviewer 直接提交代码
```

**关键点：**

- CLAUDE.md 支持嵌套加载，比如 `tests/CLAUDE.md`、`src/db/CLAUDE.md` 会在对应目录下自动激活
- 这是"长期记忆"，每次会话都会自动读取
- 写清楚"什么不能做"比"要做什么"更重要

---

## 二、Skills：自动激活的技能包

Skills 是"专业知识手册"，检测到相关需求时自动激活，无需手动触发。

**位置：** `.claude/skills/[技能名]/SKILL.md`

**一个内容生产技能示例：**

```markdown
---
name: content-production
description: 当用户需要创作技术文章、博客、文档或任何长篇内容时自动激活。
  关键词：写文章、生成内容、起草文档、创作博客
---

# 内容生产标准流程

## 激活条件
本技能在检测到内容创作需求时自动激活。

## 执行流程

### Step 1：选题与大纲
1. 分析目标读者（开发者/产品经理/业务人员）
2. 确定内容深度（入门/进阶/深度技术）
3. 生成 3-5 个候选标题，让用户选择
4. 产出结构化大纲，包含每节预期字数

### Step 2：委托研究
调用 @researcher 子代理执行：
- 搜集最新资料（使用 web_search）
- 核查关键事实
- 整理竞品内容分析
- 输出到：memory/research_[task_id].md

### Step 3：内容撰写
基于研究结果撰写正文：
- 每节完成后更新 workflow_state.json 的 progress 字段
- 代码示例必须实际测试过
- 引用来源必须注明

### Step 4：质量审核
完成后自动触发 @content-reviewer 执行：
- 事实准确性检查
- 语言流畅度评分
- SEO 关键词密度分析
- 输出评分报告

## 产物规范
- 正文文件：outputs/[task_id]_article.md
- 研究记录：memory/research_[task_id].md
- 审核报告：outputs/[task_id]_review.md
```

**触发方式：**

- **自动：** 语义匹配自动激活（比如用户说"写篇文章"）
- **手动：** `/content-production [参数]`

**注意：** 2025 年架构中，Custom Commands 已合并入 Skills 体系，但 `.claude/commands/` 目录仍兼容旧格式。

---

## 三、Subagents：专业代理团队

Subagents 是隔离上下文的专属 AI 角色，相当于你的"AI 员工团队"。

**核心优势：**

- 保持主上下文干净
- 可使用不同模型（比如研究员用 Sonnet，审查用 Opus）
- 支持并行执行

**位置：** `.claude/agents/[角色名].md`

**一个研究员 Agent 示例：**

```yaml
---
name: researcher
description: 信息搜集研究专家。当需要查找最新资料、核查事实、竞品分析时调用。
  关键词：研究、搜集信息、核查事实、背景调研、市场分析
tools: WebSearch, Read, Write, Bash
model: claude-sonnet-4-20250514
---

你是一名专业研究员，专注于快速、准确地搜集和整理信息。

## 你的工作流程

1. 理解研究目标和受众
2. 执行多角度搜索（至少 3 个不同查询）
3. 交叉验证关键事实
4. 整理成结构化报告

## 记忆维护
每次完成研究后，将关键发现追加到：
- `memory/project_context.md`（长期有用的背景知识）
- `memory/research_[task_id].md`（本次研究产物）

## 输出格式
```markdown
# 研究报告：[主题]
任务 ID：[task_id]
完成时间：[timestamp]

## 关键发现
...

## 数据来源
...

## 待核实项
...
```

完成后在 workflow_state.json 中将对应子任务状态更新为 "done"。
```

**常见角色：**

- `@researcher`：信息搜集专家
- `@writer`：内容创作专家
- `@code-reviewer`：代码质量审查专家

**调用方式：** 主 Agent 调度或 Skills 中自动调用

---

## 四、MCP：外部工具接入

MCP（Model Context Protocol）是给 Claude Code 安装"外部工具"的标准协议。

**添加官方 MCP 服务：**

```bash
# 添加 GitHub MCP
claude mcp add github -- npx @modelcontextprotocol/server-github

# 添加文件系统 MCP
claude mcp add filesystem -- npx @modelcontextprotocol/server-filesystem /path/to/allowed

# 添加数据库 MCP
claude mcp add postgres -- npx @modelcontextprotocol/server-postgres postgresql://localhost/mydb
```

**常见 MCP 服务：**

- GitHub：代码仓库操作
- FileSystem：扩展目录访问权限
- PostgreSQL：数据库查询
- Figma：设计文件读取
- Slack：消息通知

**开发自定义 MCP Server（Python 示例）：**

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
import json

app = Server("workflow-tools")

@app.list_tools()
async def list_tools():
    return [
        Tool(
            name="update_workflow_state",
            description="更新工作流状态",
            inputSchema={"type": "object", "properties": {"status": {"type": "string"}}}
        ),
        Tool(
            name="get_workflow_state",
            description="获取当前工作流状态",
            inputSchema={"type": "object"}
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "update_workflow_state":
        with open("workflow_state.json", "r+") as f:
            state = json.load(f)
            state.update(arguments)
            f.seek(0)
            json.dump(state, f, indent=2, ensure_ascii=False)
        return [TextContent(type="text", text="状态更新成功")]
    
    elif name == "get_workflow_state":
        with open("workflow_state.json") as f:
            state = json.load(f)
        return [TextContent(type="text", text=json.dumps(state, ensure_ascii=False))]

async def main():
    async with stdio_server() as streams:
        await app.run(streams[0], streams[1], app.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

在 `.claude/settings.json` 中注册：

```json
{
  "mcpServers": {
    "workflow-tools": {
      "command": "python",
      "args": ["./custom_mcp_server.py"]
    }
  }
}
```

---

## 五、Hooks：事件驱动的自动化守卫

这是最强大也最容易被忽视的特性，实现真正的**确定性执行保证**。

**Hooks 是什么：** 在特定生命周期点自动执行的 Shell 命令或脚本。

**核心事件：**

| 事件 | 触发时机 | 典型用途 |
|------|---------|---------|
| `SessionStart` | 会话开始 | 加载上下文、初始化状态 |
| `UserPromptSubmit` | 用户提交问题后 | 记录问题、预处理 |
| `PreToolUse` | 工具调用前 | 权限校验、参数验证 |
| `PostToolUse` | 工具调用后 | 状态记录、结果审计 |
| `Stop` | Agent 完成时 | 质量守卫、产物检查 |
| `SubagentStop` | 子代理完成时 | 验收子任务结果 |

**配置位置：** `.claude/settings.json` 中的 `hooks` 字段

**一个完整配置示例：**

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "python .claude/hooks/pre_tool_use.py"
      }]
    }],
    "PostToolUse": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "python .claude/hooks/post_tool_use.py"
      }]
    }],
    "Stop": [{
      "hooks": [{
        "type": "command",
        "command": "python .claude/hooks/stop.py"
      }]
    }]
  }
}
```

**PostToolUse Hook 示例（状态记录）：**

```python
#!/usr/bin/env python3
"""
PostToolUse Hook：记录每次工具调用到工作流状态
"""
import json
import sys
import os
from datetime import datetime

def main():
    # 从 stdin 读取 Claude Code 传入的事件数据
    event = json.load(sys.stdin)
    
    tool_name = event.get("tool_name", "unknown")
    tool_input = event.get("tool_input", {})
    tool_result = event.get("tool_result", {})
    
    # 更新工作流状态
    state_file = "workflow_state.json"
    if os.path.exists(state_file):
        with open(state_file, "r") as f:
            state = json.load(f)
    else:
        state = {"logs": [], "tool_calls": []}
    
    # 记录工具调用日志
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "tool": tool_name,
        "input_summary": str(tool_input)[:200],
        "success": "error" not in str(tool_result).lower()
    }
    
    if "logs" not in state:
        state["logs"] = []
    state["logs"].append(log_entry)
    
    # 保持日志不超过 100 条
    state["logs"] = state["logs"][-100:]
    state["last_activity"] = datetime.now().isoformat()
    
    with open(state_file, "w") as f:
        json.dump(state, f, indent=2, ensure_ascii=False)

if __name__ == "__main__":
    main()
```

**Stop Hook 示例（质量守卫）：**

```python
#!/usr/bin/env python3
"""
Stop Hook：Agent 完成时验证必要条件
阻止 Agent 在未满足质量条件时宣告完成
"""
import json
import sys
import os

def main():
    event = json.load(sys.stdin)
    
    # 防止无限循环
    if os.path.exists("/tmp/stop_hook_active"):
        sys.exit(0)
    
    state_file = "workflow_state.json"
    if not os.path.exists(state_file):
        sys.exit(0)
    
    with open(state_file, "r") as f:
        state = json.load(f)
    
    # 检查必要产物是否存在
    required_outputs = state.get("required_outputs", [])
    missing = []
    
    for output in required_outputs:
        if not os.path.exists(output):
            missing.append(output)
    
    if missing:
        # 向 Claude 输出错误，阻止其停止
        print(json.dumps({
            "decision": "block",
            "reason": f"以下必要产物尚未生成，请继续完成：{', '.join(missing)}"
        }))
        sys.exit(0)
    
    # 更新状态为 done
    state["status"] = "done"
    state["completed_at"] = __import__("datetime").datetime.now().isoformat()
    
    with open(state_file, "w") as f:
        json.dump(state, f, indent=2, ensure_ascii=False)

if __name__ == "__main__":
    main()
```

**Hook 的输出机制：**

- `exit 0`：正常通过
- `exit 2` + JSON 输出：可阻止操作（如 `{"decision": "block", "reason": "..."}`）

---

## 工作流状态追踪：workflow_state.json

`workflow_state.json` 是整个系统的"运行时大脑"。

**完整状态结构：**

```json
{
  "task_id": "task_20250303_001",
  "description": "写一篇关于 MCP 协议的技术博客",
  "status": "executing",
  "created_at": "2025-03-03T10:00:00",
  "updated_at": "2025-03-03T10:15:32",
  "current_agent": "writer",
  "progress": 45,
  
  "subtasks": [
    {
      "id": "st_001",
      "name": "市场调研",
      "agent": "researcher",
      "status": "done",
      "started_at": "2025-03-03T10:01:00",
      "completed_at": "2025-03-03T10:08:00",
      "output": "memory/research_task_20250303_001.md"
    },
    {
      "id": "st_002",
      "name": "内容撰写",
      "agent": "writer",
      "status": "executing",
      "depends_on": ["st_001"],
      "started_at": "2025-03-03T10:09:00",
      "output": null
    }
  ],
  
  "required_outputs": [
    "outputs/article.md",
    "outputs/review_report.md"
  ],
  
  "outputs": [
    "memory/research_task_20250303_001.md"
  ],
  
  "logs": [
    {
      "timestamp": "2025-03-03T10:08:00",
      "tool": "Write",
      "input_summary": "写入研究报告",
      "success": true
    }
  ],
  
  "metrics": {
    "total_tool_calls": 23,
    "total_duration_seconds": 930,
    "agent_calls": {
      "researcher": 1,
      "writer": 1
    }
  }
}
```

**状态管理脚本（scripts/state-manager.py）：**

```python
#!/usr/bin/env python3
"""工作流状态管理工具"""
import json
import sys
from datetime import datetime
from pathlib import Path

STATE_FILE = Path("workflow_state.json")

def load_state():
    if STATE_FILE.exists():
        return json.loads(STATE_FILE.read_text())
    return {}

def save_state(state):
    state["updated_at"] = datetime.now().isoformat()
    STATE_FILE.write_text(json.dumps(state, indent=2, ensure_ascii=False))

def advance_task(task_id, status, output=None):
    """推进子任务状态"""
    state = load_state()
    for task in state.get("subtasks", []):
        if task["id"] == task_id:
            task["status"] = status
            if status == "executing":
                task["started_at"] = datetime.now().isoformat()
            elif status == "done":
                task["completed_at"] = datetime.now().isoformat()
                if output:
                    task["output"] = output
                    state.setdefault("outputs", []).append(output)
    
    # 更新总体进度
    subtasks = state.get("subtasks", [])
    if subtasks:
        done_count = sum(1 for t in subtasks if t["status"] == "done")
        state["progress"] = int(done_count / len(subtasks) * 100)
    
    save_state(state)
    print(f"✓ 任务 {task_id} 状态更新为 {status}，总进度 {state['progress']}%")

def show_status():
    """展示当前工作流状态"""
    state = load_state()
    if not state:
        print("没有活跃的工作流")
        return
    
    print(f"\n📋 工作流：{state.get('description', 'N/A')}")
    print(f"📊 总进度：{state.get('progress', 0)}%")
    print(f"🔄 当前 Agent：{state.get('current_agent', 'N/A')}")
    print(f"📌 状态：{state.get('status', 'N/A')}")
    print("\n子任务：")
    for task in state.get("subtasks", []):
        icon = {"done": "✅", "executing": "🔄", "pending": "⏳", "failed": "❌"}.get(task["status"], "❓")
        print(f"  {icon} [{task['agent']}] {task['name']} - {task['status']}")

if __name__ == "__main__":
    cmd = sys.argv[1] if len(sys.argv) > 1 else "status"
    if cmd == "status":
        show_status()
    elif cmd == "advance":
        advance_task(sys.argv[2], sys.argv[3], sys.argv[4] if len(sys.argv) > 4 else None)
```

**注意：** `workflow_state.json` 是社区最佳实践，非官方强制规范，但强烈建议使用。

---

## 实战示例：内容生产工作流

### 启动流程

```bash
# 1. 进入项目目录
cd my-content-workflow

# 2. 启动 Claude Code
claude

# 3. 触发工作流
/workflow-start 写一篇 2000 字的技术博客，主题：如何用 Claude Code 构建 AI 工作流

# 4. Claude 自动执行：
# - 创建 workflow_state.json
# - 调度 @researcher 搜集资料
# - 等 researcher 完成后调度 @writer
# - 最后调度 @content-reviewer 审核
# - Hooks 全程记录每步状态

# 5. 中途可以查看状态
python scripts/state-manager.py status

# 6. 如果中断，下次恢复
/workflow-resume
```

### 执行过程示例输出

```
📋 工作流启动：写技术博客 - MCP 工作流
🆔 任务 ID：task_20250303_001

[Orchestrator] 拆解为 3 个子任务
  ├─ st_001: 市场调研 → @researcher
  ├─ st_002: 内容撰写 → @writer（依赖 st_001）
  └─ st_003: 质量审核 → @content-reviewer（依赖 st_002）

[Researcher Agent] 开始执行...
  ✓ 搜索"Claude Code MCP 2025 最佳实践"
  ✓ 搜索"AI 工作流编排案例"
  ✓ 整理研究报告 → memory/research_task_20250303_001.md
  ✅ 研究完成，进度 33%

[Writer Agent] 开始撰写...
  ✓ 读取研究报告
  ✓ 撰写引言（300 字）
  ✓ 撰写第一章（600 字）
  ✓ 撰写第二章（700 字）
  ✓ 撰写结论（400 字）
  ✅ 文章完成 → outputs/article.md，进度 67%

[Content Reviewer] 开始审核...
  ✓ 事实准确性：通过
  ✓ 流畅度评分：87/100
  ✓ SEO 关键词：已覆盖 8/10 目标词
  ✅ 审核报告 → outputs/review_report.md，进度 100%

✅ 工作流完成！
  📄 文章：outputs/article.md（2183 字）
  📊 审核：outputs/review_report.md
  💾 状态：workflow_state.json（已归档）
```

---

## 代码工程工作流：RIPER 模式

RIPER 是业界广泛使用的 Claude Code 工程流程：

```
R - Research（研究：只读代码库，不动代码）
I - Innovate（创新：头脑风暴解决方案）
P - Plan（规划：生成详细执行计划）
E - Execute（执行：严格按计划实施）
R - Review（复盘：验证与总结）
```

**`.claude/commands/riper.md`：**

```markdown
# /riper [功能描述]

按 RIPER 五阶段开发新功能。

## R - Research（只读阶段）
- 阅读相关源码，禁止修改任何文件
- 分析现有架构模式
- 记录到 memory/riper_research.md
- 更新 state: {phase: "research", progress: 20}

## I - Innovate（创新阶段）
- 生成 3+ 个实现方案
- 分析各方案的优劣
- 推荐最优方案并说明理由
- 更新 state: {phase: "innovate", progress: 40}

## P - Plan（规划阶段）
- 生成详细的文件变更清单
- 估算每个步骤的风险等级
- 用户确认后才能继续
- 更新 state: {phase: "plan", progress: 60}

## E - Execute（执行阶段）
- 严格按 P 阶段的计划执行
- 每完成一个文件更新进度
- 遇到偏差立即暂停报告
- Hooks 自动验证每次写文件后测试通过

## R - Review（复盘阶段）
- 运行完整测试套件
- 验证功能符合需求
- 将经验追加到 memory/lessons_learned.md
- 更新 state: {phase: "done", progress: 100}
```

---

## 自我进化机制

这是整个系统最核心的能力：**工作流会从自身的失败中学习，不断完善**。

### 失败记录

每当 Hook 检测到异常，自动记录到 `memory/lessons_learned.md`：

```python
def record_failure(state, error_type, context):
    lesson = {
        "date": datetime.now().isoformat(),
        "task_type": state.get("description", "")[:50],
        "error_type": error_type,
        "context": context,
        "resolution": "待分析"
    }
    
    lessons_file = Path("memory/lessons_learned.md")
    existing = lessons_file.read_text() if lessons_file.exists() else ""
    new_entry = f"\n## {lesson['date']} | {error_type}\n**任务**：{lesson['task_type']}\n**上下文**：{context}\n---\n"
    lessons_file.write_text(existing + new_entry)
```

### 定期自我审视

**`.claude/commands/self-improve.md`：**

```markdown
# /self-improve

分析最近 10 次工作流执行，识别改进点。

## 执行步骤

1. 读取 workflow_state.json 的历史记录
2. 读取 memory/lessons_learned.md
3. 分析高频失败模式
4. 提出具体改进建议：
   - 哪些 Subagent 的 prompt 需要优化
   - 哪些 Hooks 需要新增
   - CLAUDE.md 哪些规则需要强化
5. 生成改进提案 → memory/improvement_proposals.md
6. 征询用户确认后，自动更新对应的 .md 文件
```

### 迭代节奏

```
每天：  自动记录所有工具调用日志（PostToolUse Hook）
每周：  /self-improve 分析失败模式，更新 Subagent prompt
每月：  review CLAUDE.md，更新项目级规范
每季：  重新评估 MCP 工具栈，淘汰低效工具，引入新能力
```

---

## 快速上手清单

```bash
# Step 1: 安装 Claude Code
npm install -g @anthropic-ai/claude-code

# Step 2: 创建项目结构
mkdir my-workflow && cd my-workflow
mkdir -p .claude/{agents,commands,skills/content-production,hooks} memory outputs scripts

# Step 3: 创建 CLAUDE.md（参考上文示例）

# Step 4: 创建第一个 Subagent
cat > .claude/agents/researcher.md << 'EOF'
---
name: researcher
description: 信息搜集研究专家
tools: WebSearch, Read, Write
---
你是专业研究员...
EOF

# Step 5: 启动并测试
claude
# 输入：搜集一些关于 AI 工作流的最新资料
```

---

## 结语

回到开头的问题：

为什么同样的 AI 工具，有人用来聊天摸鱼，有人却能搭建自动化工程系统？

差别不在于模型能力，而在于**你有没有一套可靠的工作流**。

CLAUDE.md 给你记忆，Skills 给你知识，Commands 给你流程，Subagents 给你团队，Hooks 给你纪律。

五者协同，Claude Code 从聊天工具进化为可靠的自动化工程系统。

**不要一开始就追求完美。** 先从 CLAUDE.md 开始，然后加一个 Subagent，再配一个 Hook。慢慢迭代，慢慢完善。

**工作流不是一次性建成的，而是在一次次使用中长出来的。**

---

**官方文档参考：**

- [Claude Code 官方文档](https://code.claude.com/docs/en/overview)
- [Skills](https://code.claude.com/docs/en/skills)
- [Sub-agents](https://code.claude.com/docs/en/sub-agents)
- [MCP](https://code.claude.com/docs/en/mcp)
- [Hooks](https://code.claude.com/docs/en/hooks)

**第三方参考：**

- [Alexop 定制化指南](https://alexop.dev/posts/claude-code-customization-guide-claudemd-skills-subagents/)

---

*本文基于 2025 年 Claude Code 架构编写，部分配置可能随版本更新有所变化，请以官方文档为准。*
