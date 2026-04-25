---
title: "别为了 Agent 而 Agent：2026 年多 Agent 工作流实战指南"
description: "测试了四个主流框架后，我的判断是：能单 Agent 解决的，别拆成多 Agent。但如果你真的需要，这篇能帮你选对框架、跑通示例、避开常见坑。"
date: 2026-04-25T10:00:00+08:00
draft: false
tags: ["AI Agent", "LangGraph", "CrewAI", "AutoGen", "OpenHands", "工作流"]
categories: ["技术实战"]
toc: true
---

一个团队花了两周，搭了三 Agent 流水线——研究员搜集数据、写作者整理报告、审核者把关质量。跑起来之后发现：每次执行花 40 秒，API 账单涨了 3 倍，调试时根本不知道哪个 Agent 说了什么。

最后他们回到单 Agent，5 行代码解决了同样的问题。

这不是个例。2026 年，多 Agent 工作流成了 AI 圈最热的方向，但**很多团队的问题不是不会搭多 Agent，而是根本不需要**。

这篇不劝你上，也不劝你不下。我实测了四个主流框架——LangGraph、CrewAI、AutoGen、OpenHands——给你三样东西：

1. **判断标准**：什么场景该用多 Agent，什么场景不该用
2. **实操指南**：每个框架的最小示例，照着能跑
3. **避坑清单**：部署前、运行中、出问题时的检查项

先说结论。

## 先判断：你真的需要多 Agent 吗？

多 Agent 不是升级，是另一种工具。Agent 越多，调试越难，成本越高。

**该用的场景：**

- 复杂多步骤工作流，有分支、循环、暂停恢复——单 Agent 写出来像意大利面
- 需要不同"角色"各司其职——研究员、写作者、审核者，每个角色的 system prompt 差异很大
- 需要人类在环审批——跑到一半暂停，等人确认后再继续
- 编码任务需要沙箱隔离——Agent 生成的代码不能直接在宿主机跑

**不该用的场景：**

| 场景 | 为什么不该用 | 替代方案 |
|------|------------|---------|
| 简单问答/单步任务 | 引入多 Agent 是过度工程 | 直接用单 Agent 或 OpenAI API |
| 高确定性流水线 | 对话式控制流不可预测 | 传统状态机/工作流引擎 |
| 实时性要求高 | LLM 调用延迟 + 多轮累积延迟 | 缓存 + 异步处理 |
| 无 LLM 的纯规则引擎 | 不需要 LLM 能力 | transitions、state_machine 等库 |
| 严格合规/审计 | 对话路径难以完全预测和审计 | 确定性工作流引擎 |

**一句话判断：** 如果你的任务可以写成"输入 → 处理 → 输出"三步，别用多 Agent。如果需要分支、循环、多角色协作、人类审批，再考虑。

## 四个框架，怎么选？

| 框架 | 一句话定位 | 核心优势 | 主要局限 | 学习曲线 |
|------|-----------|---------|---------|---------|
| **LangGraph** | 专业编排器 | 精细控制流、状态持久化、人类在环 | 概念较多，TypedDict + 图结构需要适应 | 中等 |
| **CrewAI** | 快速启动器 | API 简单、角色分工自然、上手快 | 复杂场景控制流不够灵活 | 低 |
| **AutoGen/AG2** | 自由讨论区 | 对话式交互灵活、适合探索性任务 | 可预测性低、调试困难 | 中高 |
| **OpenHands** | 安全沙箱 | 专注编码任务、Docker 隔离安全 | 仅限编码场景、必需 Docker | 中等 |

**决策树：**

```
你的核心需求是什么？
├── 自动化编码/代码修改
│   └── OpenHands（沙箱执行环境）
├── 复杂多步骤工作流（有分支/循环/暂停）
│   └── LangGraph（图编排，最灵活）
├── 快速搭建多 Agent 原型
│   └── CrewAI（API 最简单）
├── 多角色对话协作/讨论
│   └── AutoGen/AG2（对话式最自然）
└── 不确定
    └── 从 CrewAI 开始（学习成本最低）
```

下面逐个拆解。

## 1. LangGraph：有向图 + 状态机

**适合：** 需要精细控制工作流走向的场景——分支、循环、条件路由、断点恢复。

这是四个框架里控制力最强的，也是概念最多的。

### 架构

```
START → Router → [条件路由] → Agent A / Agent B → END
                  ↑                              |
                  └────────── 循环/重试 ←─────────┘
```

所有 Agent 共享一个 `TypedDict` 状态，图结构编译时确定，运行时按边执行。通过检查点实现断点恢复、人类在环、时间旅行调试。

### 最小可运行示例

```bash
# 环境配置
python -m venv langgraph-env
source langgraph-env/bin/activate
pip install langgraph langchain-openai langchain
export OPENAI_API_KEY="your-key"
```

```python
"""LangGraph 最小示例：Researcher → Writer 协作"""
import os
from typing import TypedDict, Annotated
import operator
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_core.messages import HumanMessage, AIMessage

os.environ["OPENAI_API_KEY"] = "your-key"
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# 定义共享状态
class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    research_result: Annotated[str, operator.add]
    next: str

# 定义节点
def researcher_node(state: AgentState) -> dict:
    response = llm.invoke([
        HumanMessage(content=f"请针对以下问题提供详细研究结果:\n{state['messages'][-1].content}")
    ])
    return {
        "messages": [AIMessage(content=f"[Researcher] {response.content}")],
        "research_result": response.content,
        "next": "writer"
    }

def writer_node(state: AgentState) -> dict:
    response = llm.invoke([
        HumanMessage(content=f"基于以下研究结果生成最终报告:\n{state['research_result']}")
    ])
    return {
        "messages": [AIMessage(content=f"[Writer] {response.content}")],
        "next": "END"
    }

# 构建图
workflow = StateGraph(AgentState)
workflow.add_node("researcher", researcher_node)
workflow.add_node("writer", writer_node)
workflow.add_edge(START, "researcher")
workflow.add_edge("researcher", "writer")
workflow.add_edge("writer", END)

# 编译并执行
memory = MemorySaver()
app = workflow.compile(checkpointer=memory)

result = app.invoke(
    {"messages": [HumanMessage(content="分析 Python 在 AI 领域的主导地位")],
     "research_result": "", "next": ""},
    config={"configurable": {"thread_id": "demo-1"}}
)
```

核心概念就两个：**节点**（每个 Agent 做什么）和**边**（谁传给谁）。条件路由让 Router 根据问题类型分发给不同 Agent：

```python
def router_node(state: AgentState) -> dict:
    last_msg = state["messages"][-1].content.lower()
    if any(kw in last_msg for kw in ["code", "编程", "代码"]):
        return {"next": "coder"}
    else:
        return {"next": "generalist"}

router_graph.add_conditional_edges(
    "router",
    lambda state: state["next"],
    {"coder": "coder", "generalist": "generalist"}
)
```

**最容易踩的坑：** `KeyError: 'messages'`——节点返回的状态少了 TypedDict 里定义的某个键。每个节点必须返回完整的状态字段，少一个就报错。

## 2. CrewAI：角色分工 + 任务流水线

**适合：** 快速搭建多 Agent 原型。角色分工自然的场景——研究员、写作者、审核者。

CrewAI 的学习成本是四个里最低的。它的隐喻很直觉：定义角色、定义任务、组成团队、执行。

```bash
# 环境配置
python -m venv crewai-env
source crewai-env/bin/activate
pip install crewai crewai-tools
export OPENAI_API_KEY="your-key"
```

```python
"""CrewAI 最小示例：Researcher + Writer 协作"""
import os
os.environ["OPENAI_API_KEY"] = "your-key"

from crewai import Agent, Task, Crew, Process

researcher = Agent(
    role="高级研究员",
    goal="收集和分析相关信息",
    backstory="你是一个经验丰富的研究员，擅长信息搜集和分析。",
    verbose=True
)

writer = Agent(
    role="技术写作者",
    goal="将研究结果转化为清晰的报告",
    backstory="你是一个技术写作者，擅长将复杂信息简化。",
    verbose=True
)

research_task = Task(
    description="研究 Python 在 AI 领域的主导地位及其原因",
    agent=researcher,
    expected_output="结构化的研究分析"
)

write_task = Task(
    description="基于研究结果撰写一份简洁报告",
    agent=writer,
    expected_output="最终报告"
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential  # 顺序执行
)

result = crew.kickoff()
print(result)
```

需要项目经理来协调？加一个 `manager_agent` 切换成层级模式就行：

```python
manager = Agent(
    role="项目经理",
    goal="协调团队完成项目",
    backstory="你是一个项目经理，擅长任务分配和协调。",
    verbose=True
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.hierarchical,
    manager_agent=manager
)
```

**CrewAI 的边界：** 简单任务很顺手，但复杂场景下控制流不够灵活。如果你发现自己需要条件分支、循环重试、断点恢复——换 LangGraph。

## 3. AutoGen/AG2：对话式消息传递

**适合：** 多角色讨论、探索性任务、需要 Agent 间自由对话的场景。

AutoGen 的思路和其他三个不一样——它不是编排工作流，而是让 Agent 互相聊天。状态隐含在对话历史里，角色通过 system message 动态定义。

> **注意：** AutoGen 存在 v0.2（经典 API）和 v0.4+（Core Runtime）两套架构。新手从 v0.2 开始，包名是 `autogen-agentchat`。

```bash
python -m venv autogen-env
source autogen-env/bin/activate
pip install autogen-agentchat openai
export OPENAI_API_KEY="your-key"
```

```python
"""AutoGen 最小示例：Coder + UserProxy 对话"""
import os
import autogen

os.environ["OPENAI_API_KEY"] = "your-key"

llm_config = {
    "config_list": [{"model": "gpt-4o", "api_key": os.environ.get("OPENAI_API_KEY")}],
    "temperature": 0.7,
}

coder = autogen.AssistantAgent(
    name="coder",
    llm_config=llm_config,
    system_message="你是一个 Python 编程专家。请编写可运行的代码。",
)

user_proxy = autogen.UserProxyAgent(
    name="user_proxy",
    human_input_mode="NEVER",  # 完全自动化
    code_execution_config={
        "last_n_messages": 3,
        "work_dir": "coding",
        "use_docker": False,
    },
)

user_proxy.initiate_chat(
    coder,
    message="请写一个 Python 函数，计算斐波那契数列的前 10 项。",
)
```

多 Agent 协作用 GroupChat——把几个 Agent 扔进一个群聊，设定轮次上限和发言顺序：

```python
pm = autogen.AssistantAgent(name="pm", llm_config=llm_config,
    system_message="你是一个产品经理。请描述 Web 应用的功能需求。")

developer = autogen.AssistantAgent(name="developer", llm_config=llm_config,
    system_message="你是一个全栈开发者。根据需求编写技术实现方案。")

user_proxy = autogen.UserProxyAgent(name="user_proxy",
    human_input_mode="NEVER", max_consecutive_auto_reply=1)

groupchat = autogen.GroupChat(
    agents=[pm, developer, user_proxy],
    messages=[], max_round=6,
    speaker_selection_method="round_robin",
)

manager = autogen.GroupChatManager(groupchat=groupchat, llm_config=llm_config)
user_proxy.initiate_chat(manager, message="请设计并实现一个待办事项清单 Web 应用。")
```

**AutoGen 的代价：** 灵活度高，但可预测性低。对话路径不像 LangGraph 那样是确定的图结构——你可能不知道 Agent 们会聊成什么样。调试也比其他框架难，因为状态分散在对话历史里。

## 4. OpenHands：沙箱编码环境

**适合：** 自动化编码、代码修改、需要沙箱隔离安全的场景。

OpenHands 和其他三个不是一类东西——它不是编排框架，是编码 Agent 的执行环境。Agent 通过 EventStream 与 Docker 沙箱交互，生成的代码在隔离环境里跑，安全。

```bash
# 确保 Docker 已安装并运行
docker --version

# 克隆并启动
git clone https://github.com/All-Hands-AI/OpenHands.git
cd OpenHands
echo "LLM_API_KEY=your-key" >> .env
echo "LLM_MODEL=gpt-4o" >> .env
docker compose up -d

# 访问 http://localhost:3000
```

或者 CLI 方式：

```bash
pip install openhands
export LLM_API_KEY="your-key"
export LLM_MODEL="gpt-4o"
openhands
# 输入任务描述即可
```

**多 Agent 协作：** OpenHands 本身只处理单个编码 Agent。需要编排的话，上层用 LangGraph/CrewAI/AutoGen——OpenHands 提供执行环境，上层框架负责调度。

**必需 Docker。** 没有 Docker 就别考虑这个。

## 成本分析

所有框架本身免费开源，主要成本来自 LLM API 调用：

| 使用场景 | 估算成本 (GPT-4o) |
|----------|------------------|
| 简单 2-Agent 协作（单次执行） | ~$0.03-0.15 |
| 复杂多 Agent 工作流（10+ 节点） | ~$0.30-1.00 |
| GroupChat（4 Agent, 20 轮） | ~$0.30-1.00 |
| 编码任务（OpenHands 单次） | ~$0.10-0.50 |
| 月成本估算（1000 次执行/天） | ~$900-3000/月 |

**成本优化建议：**

- 用小模型做路由/分类任务（GPT-3.5-turbo、Claude Haiku）
- 缓存重复调用（LangChain 缓存、AutoGen cache_seed）
- CI/CD 测试中使用便宜模型或 mock
- 设置 max_tokens 和 max_consecutive_auto_reply 控制对话长度

## 检查清单

### 部署前必查

- [ ] **Python 版本**：3.10+（推荐 3.11）
- [ ] **虚拟环境**：已创建并激活（venv/conda）
- [ ] **API Key**：已设置环境变量（`OPENAI_API_KEY` 或 `LLM_API_KEY`）
- [ ] **框架版本**：已锁定版本（生产环境推荐 `pip freeze > requirements.txt`）
- [ ] **Docker**：如使用 OpenHands 或代码执行隔离，确认 Docker 已安装并运行
- [ ] **网络**：可访问 LLM API 端点（国内需配置代理或使用国内模型）
- [ ] **依赖冲突**：运行 `pip check` 检查依赖冲突
- [ ] **测试**：已运行最小示例验证环境可用

### 运行中监控

- [ ] **Token 消耗**：监控每次执行的 token 用量，设置预算上限
- [ ] **执行延迟**：记录每个节点/Agent 的执行时间，识别瓶颈
- [ ] **错误率**：统计 API 调用失败率、超时率
- [ ] **循环检测**：检测 Agent 是否陷入无限循环（设置 max_round/max_iterations）
- [ ] **内存使用**：监控进程内存，特别是长时间运行的工作流
- [ ] **日志记录**：启用框架日志，记录关键执行路径

### 出问题时的排查步骤

```
1. 确认 API Key 有效且未过期
   └── curl -H "Authorization: Bearer $OPENAI_API_KEY" https://api.openai.com/v1/models

2. 检查框架版本是否与文档匹配
   └── pip show langgraph / crewai / autogen-agentchat / openhands

3. 运行最小示例验证基础功能
   └── 复制文档中的 Hello World 示例

4. 检查环境变量是否正确加载
   └── python -c "import os; print(os.environ.get('OPENAI_API_KEY'))"

5. 启用调试/日志模式
   └── LangGraph: stream_mode="updates"
       CrewAI: verbose=True
       AutoGen: logging.getLogger("autogen").setLevel(logging.DEBUG)
       OpenHands: 查看 Docker 日志

6. 检查 LLM 模型是否可用
   └── 确认 API Key 有对应模型的访问权限

7. 检查网络连接
   └── 国内环境需配置代理或使用国内模型
```

### 安全注意事项

- [ ] **API Key 管理**：使用环境变量或密钥管理工具，不要硬编码
- [ ] **代码执行沙箱**：AutoGen 代码执行建议使用 Docker 隔离
- [ ] **权限最小化**：Agent 只授予完成任务所需的最小权限
- [ ] **输入验证**：对用户输入进行验证，防止注入攻击
- [ ] **输出审查**：Agent 生成的代码/内容建议人工审查后再使用
- [ ] **成本限制**：设置 API 调用频率限制和预算上限
- [ ] **敏感数据**：不要在 Agent 对话中传递敏感信息（密码、密钥等）

## 学习路径建议

1. **入门（1-2 天）**：从 CrewAI 开始，理解多 Agent 基本概念
2. **进阶（1 周）**：学习 LangGraph，掌握图编排和状态管理
3. **拓展（2 周）**：尝试 AutoGen/AG2，理解对话式协作
4. **专项（按需）**：如有编码自动化需求，学习 OpenHands
5. **生产（持续）**：结合 LangSmith 等工具，建立可观测性

## 最后说一句

能单 Agent 解决的，别拆成三个。

多 Agent 不是升级，是另一种工具。先跑通最小示例，再扩展复杂度。关注成本，重视调试。

如果你看完这篇，判断自己的场景确实需要多 Agent——照着上面的示例跑，别踩我已经标出来的坑。如果判断不需要——恭喜你，省了两周围绕三个 Agent 转的日子。

---

**参考资源：**

- [LangGraph 官方文档](https://langchain-ai.github.io/langgraph/)
- [CrewAI 官方文档](https://docs.crewai.com/)
- [AutoGen (Microsoft) 官方文档](https://microsoft.github.io/autogen/)
- [AG2 (社区分支) 官方文档](https://docs.ag2.ai/)
- [OpenHands 官方文档](https://docs.all-hands.dev/)
