---
title: "我把 Claude Code 改造成了自动化工程系统，不再陪它聊天了"
date: 2026-04-07T15:00:00+08:00
draft: false
author: "Zampo"
description: "同样的 AI 工具，有人用来聊天摸鱼，有人却能搭建自动化工程系统。差别不在于模型能力，而在于工作流。"
tags: ["AI 工程化", "Claude Code", "自动化", "工作流", "开发者工具"]
categories: ["技术实战"]
featured: true
toc: true
---

你有没有这种感觉：

用 Claude Code 的时候，每次都要重新交代背景。

让它写个脚本，要说明项目结构；让它查个 bug，要解释代码逻辑；让它写篇文章，要重复一遍要求。

**同样的任务，说一遍又一遍。**

输出质量还时好时坏——心情好时写得不错，心情不好时直接跑偏。

问题不在模型。

**问题在于，你一直在陪它聊天，而不是让它工作。**

2025 年的 Claude Code，早已不是单纯的聊天工具。

通过五层扩展机制，你可以把它改造成一个可靠的自动化工程系统：

> **CLAUDE.md 是记忆，Skills 是知识，Commands 是流程，Subagents 是团队，Hooks 是纪律。**

今天这篇文章，不聊概念，只给能落地的框架。

看完之后，你可以直接照着搭建自己的工作流。

<!--more-->

---

## 01 先上一张架构图

```
用户输入
   │
   ▼
┌────────────────────────────────────┐
│         Claude Code 主 Agent        │
│   CLAUDE.md  │  Skills  │  Commands │
└─────────────────┬──────────────────┘
                  │
      ┌───────────┴───────────┐
      ▼                       ▼
┌──────────────┐    ┌─────────────────┐
│ Subagents    │    │   MCP Tools     │
│ @researcher  │    │  GitHub/Slack   │
│ @writer      │    │  PostgreSQL     │
│ @reviewer    │    │  自定义工具      │
└──────────────┘    └─────────────────┘
      │
      ▼
┌────────────────────────────────────┐
│         Hooks 事件系统              │
│  状态记录 → 质量守卫 → 自动通知     │
└────────────────────────────────────┘
```

看不懂没关系，记住这句话就行：

**五者协同，Claude Code 从聊天工具进化为可靠的自动化工程系统。**

下面逐个拆解。

---

## 02 CLAUDE.md：项目宪法

这是 Claude 每次启动必读的"项目宪法"。

**位置：** `.claude/CLAUDE.md` 或根目录 `CLAUDE.md`

**作用：** 定义项目背景、规范、协作规则、禁止行为。

**一个示例：**

```markdown
# 项目：智能内容生产工作流

## 核心规范
- 所有内容输出前必须经过事实核查
- 代码示例必须是可运行的，不写伪代码
- 文章字数：技术博客 1500-2500 字

## Agent 协作规则
- researcher 先行，writer 后续，reviewer 最终把关
- writer 必须等 researcher 完成后才能开始

## 禁止行为
- 不得在未完成 research 时直接写内容
- 不得跳过 reviewer 直接提交代码
```

**关键点：**

- CLAUDE.md 支持嵌套加载，比如 `tests/CLAUDE.md` 会在对应目录下自动激活
- 这是"长期记忆"，每次会话都会自动读取
- **写清楚"什么不能做"比"要做什么"更重要**

---

## 03 Skills：自动激活的技能包

Skills 是"专业知识手册"，检测到相关需求时自动激活，无需手动触发。

**位置：** `.claude/skills/[技能名]/SKILL.md`

**一个内容生产技能示例：**

```markdown
---
name: content-production
description: 当用户需要创作技术文章时自动激活
---

# 内容生产标准流程

## 执行流程

### Step 1：选题与大纲
1. 分析目标读者
2. 确定内容深度
3. 生成 3-5 个候选标题，让用户选择

### Step 2：委托研究
调用 @researcher 子代理执行：
- 搜集最新资料
- 核查关键事实
- 输出到：memory/research_[task_id].md

### Step 3：内容撰写
- 代码示例必须实际测试过
- 引用来源必须注明

### Step 4：质量审核
完成后自动触发 @content-reviewer 执行
```

**触发方式：**

- **自动：** 语义匹配自动激活（比如用户说"写篇文章"）
- **手动：** `/content-production [参数]`

**注意：** 2025 年架构中，Custom Commands 已合并入 Skills 体系。

---

## 04 Subagents：专业代理团队

Subagents 是隔离上下文的专属 AI 角色，相当于你的"AI 员工团队"。

**核心优势：**

- 保持主上下文干净
- 可使用不同模型
- 支持并行执行

**位置：** `.claude/agents/[角色名].md`

**一个研究员 Agent 示例：**

```yaml
---
name: researcher
description: 信息搜集研究专家
tools: WebSearch, Read, Write
model: claude-sonnet-4-20250514
---

你是一名专业研究员，专注于快速、准确地搜集和整理信息。

## 工作流程

1. 理解研究目标和受众
2. 执行多角度搜索（至少 3 个不同查询）
3. 交叉验证关键事实
4. 整理成结构化报告
```

**常见角色：**

- `@researcher`：信息搜集专家
- `@writer`：内容创作专家
- `@code-reviewer`：代码质量审查专家

**调用方式：** 主 Agent 调度或 Skills 中自动调用

---

## 05 MCP：外部工具接入

MCP（Model Context Protocol）是给 Claude Code 安装"外部工具"的标准协议。

**添加官方 MCP 服务：**

```bash
# 添加 GitHub MCP
claude mcp add github -- npx @modelcontextprotocol/server-github

# 添加数据库 MCP
claude mcp add postgres -- npx @modelcontextprotocol/server-postgres postgresql://localhost/mydb
```

**常见 MCP 服务：**

- GitHub：代码仓库操作
- FileSystem：扩展目录访问权限
- PostgreSQL：数据库查询
- Figma：设计文件读取
- Slack：消息通知

**也可以开发自定义 MCP Server**，用 Python 写几十行代码就能给自己的工具加一层 MCP 接口。

这里不展开，有需要的可以看官方文档。

---

## 06 Hooks：事件驱动的自动化守卫

这是最强大也最容易被忽视的特性。

**Hooks 是什么：** 在特定生命周期点自动执行的脚本。

**核心事件：**

| 事件 | 触发时机 | 典型用途 |
|------|---------|---------|
| `SessionStart` | 会话开始 | 加载上下文 |
| `PreToolUse` | 工具调用前 | 权限校验 |
| `PostToolUse` | 工具调用后 | 状态记录 |
| `Stop` | Agent 完成时 | 质量守卫 |

**配置位置：** `.claude/settings.json` 中的 `hooks` 字段

**一个完整配置示例：**

```json
{
  "hooks": {
    "PostToolUse": [{
      "command": "python .claude/hooks/post_tool_use.py"
    }],
    "Stop": [{
      "command": "python .claude/hooks/stop.py"
    }]
  }
}
```

**PostToolUse Hook 示例（状态记录）：**

```python
#!/usr/bin/env python3
import json, sys, os
from datetime import datetime

def main():
    event = json.load(sys.stdin)
    tool_name = event.get("tool_name", "unknown")
    
    # 记录工具调用日志
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "tool": tool_name,
        "success": True
    }
    
    # 写入状态文件
    with open("workflow_state.json", "a") as f:
        json.dump(log_entry, f)

if __name__ == "__main__":
    main()
```

**Stop Hook 可以做什么？**

比如检查必要产物是否存在，如果缺失就阻止 Agent 宣告完成。

```python
# 检查必要产物
if not os.path.exists("outputs/article.md"):
    print("文章还没写完，不能停止")
    sys.exit(2)  # 阻止停止
```

**这就是确定性执行保证。**

---

## 07 workflow_state.json：运行时大脑

`workflow_state.json` 是整个系统的"运行时大脑"。

**完整状态结构：**

```json
{
  "task_id": "task_20250303_001",
  "description": "写一篇关于 MCP 协议的技术博客",
  "status": "executing",
  "progress": 45,
  "subtasks": [
    {
      "name": "市场调研",
      "agent": "researcher",
      "status": "done"
    },
    {
      "name": "内容撰写",
      "agent": "writer",
      "status": "executing"
    }
  ],
  "outputs": ["memory/research.md"]
}
```

**状态管理脚本：**

```python
# 查看当前状态
python scripts/state-manager.py status

# 推进任务
python scripts/state-manager.py advance st_002 done
```

**注意：** 这是社区最佳实践，非官方强制规范，但强烈建议使用。

---

## 08 实战示例：内容生产工作流

### 启动流程

```bash
# 进入项目目录
cd my-content-workflow

# 启动 Claude Code
claude

# 触发工作流
/workflow-start 写一篇 2000 字的技术博客

# 查看状态
python scripts/state-manager.py status

# 如果中断，下次恢复
/workflow-resume
```

### 执行过程

```
📋 工作流启动：写技术博客
🆔 任务 ID：task_20250303_001

[Orchestrator] 拆解为 3 个子任务
  ├─ 市场调研 → @researcher
  ├─ 内容撰写 → @writer
  └─ 质量审核 → @reviewer

[Researcher] 开始执行...
  ✓ 搜索最新资料
  ✓ 整理研究报告
  ✅ 研究完成，进度 33%

[Writer] 开始撰写...
  ✓ 撰写引言
  ✓ 撰写正文
  ✅ 文章完成，进度 67%

[Reviewer] 开始审核...
  ✓ 事实准确性检查
  ✓ 流畅度评分
  ✅ 审核完成，进度 100%
```

---

## 09 代码工程工作流：RIPER 模式

RIPER 是业界广泛使用的 Claude Code 工程流程：

```
R - Research（研究：只读代码库，不动代码）
I - Innovate（创新：头脑风暴解决方案）
P - Plan（规划：生成详细执行计划，用户确认）
E - Execute（执行：严格按计划实施）
R - Review（复盘：验证与总结）
```

**核心思想：**

- R 阶段只读不写，避免瞎改
- P 阶段必须用户确认后才能继续
- E 阶段严格按计划执行，遇到偏差立即暂停

这样能最大程度避免 AI 乱改代码。

---

## 10 自我进化机制

这是整个系统最核心的能力：

**工作流会从自身的失败中学习，不断完善。**

### 失败记录

每当 Hook 检测到异常，自动记录到 `memory/lessons_learned.md`：

```markdown
## 2025-03-03 | 未完成审核就提交
**任务**：写技术博客
**问题**：writer 完成后直接提交，跳过了 reviewer
**解决**：在 Stop Hook 中增加审核状态检查
```

### 定期自我审视

```bash
/self-improve
```

这个命令会分析最近 10 次工作流执行，识别改进点，提出具体建议。

### 迭代节奏

```
每天：自动记录所有工具调用日志
每周：分析失败模式，更新 Subagent prompt
每月：review CLAUDE.md，更新项目级规范
每季：重新评估 MCP 工具栈
```

---

## 11 快速上手清单

```bash
# Step 1: 安装 Claude Code
npm install -g @anthropic-ai/claude-code

# Step 2: 创建项目结构
mkdir my-workflow && cd my-workflow
mkdir -p .claude/{agents,skills,hooks} memory outputs scripts

# Step 3: 创建 CLAUDE.md

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
```

---

## 12 结语

回到开头的问题：

**为什么同样的 AI 工具，有人用来聊天摸鱼，有人却能搭建自动化工程系统？**

差别不在于模型能力。

**差别在于，你有没有一套可靠的工作流。**

CLAUDE.md 给你记忆，Skills 给你知识，Commands 给你流程，Subagents 给你团队，Hooks 给你纪律。

五者协同，Claude Code 从聊天工具进化为可靠的自动化工程系统。

**不要一开始就追求完美。**

先从 CLAUDE.md 开始，然后加一个 Subagent，再配一个 Hook。

慢慢迭代，慢慢完善。

**工作流不是一次性建成的，而是在一次次使用中长出来的。**

---

**参考资料：**

- [Claude Code 官方文档](https://code.claude.com/docs/en/overview)
- [Skills](https://code.claude.com/docs/en/skills)
- [Sub-agents](https://code.claude.com/docs/en/sub-agents)
- [MCP](https://code.claude.com/docs/en/mcp)
- [Hooks](https://code.claude.com/docs/en/hooks)

---

*本文基于 2025 年 Claude Code 架构编写，部分配置可能随版本更新有所变化，请以官方文档为准。*
