---
title: "Hermes 今天挂了？别重装，一行补丁先救 Codex"
description: "2026-05-27 前后，openai-codex / gpt-5.5 用户集中遇到 `TypeError: 'NoneType' object is not iterable`、fallback 和卡死。问题大概率不在你的配置，而在 Codex backend 与 openai-python Responses stream 解析之间的兼容坑。"
date: 2026-05-27T11:10:00+08:00
draft: false
author: "任博"
tags: ["Hermes Agent", "OpenAI", "Codex", "故障修复", "AI 工具"]
categories: ["技术实战"]
cover: "/images/hermes-codex-fix-tutorial/cover.png"
toc: true
---

你只是让 Hermes 回一句 `ping`，它却开始连续 fallback。

日志里反复出现这几行：

```text
Provider: openai-codex  Model: gpt-5.5
TypeError: 'NoneType' object is not iterable
trying fallback...
```

这时候最容易做错一件事：开始重装 Hermes、重配 token、怀疑自己的 profile 坏了。

先别急。

今天这类报错，大概率不是你本地配置突然失忆，而是 Codex 后端返回流和 `openai-python` 解析逻辑撞上了一个兼容坑。

![Hermes Codex/OpenAI 快速止血教程](/images/hermes-codex-fix-tutorial/cover.png)

今天先别重装 Hermes，先绕开 Codex 这颗雷。

## 这次到底坏在哪

把故障链路压短一点说：

`openai-codex` 通过 Codex Responses API 流式返回结果时，前面的 `response.output_item.done` 事件里已经带了输出内容，但最后的 `response.completed` 事件里，`response.output` 可能是 `null`。

问题来了。

`openai-python` 的 Responses stream accumulator 在处理 completed 事件时，会把最终 response 交给 `parse_response()`。而解析函数里有一段逻辑等价于：

```python
for output in response.output:
    ...
```

如果 `response.output` 是 `None`，Python 就会直接炸：

```text
TypeError: 'NoneType' object is not iterable
```

Hermes 用 `openai-codex` / `gpt-5.5` 做 primary provider 时，这个异常会把 primary 调用打断。于是你看到 fallback，或者在某些链路里直接卡住。

这不是“模型不聪明”，也不是“prompt 写坏了”。它更像是最后一帧数据格式变了，而客户端解析器没给空值兜底。

![Codex backend 到 openai-python 的故障链路](/images/hermes-codex-fix-tutorial/inline-01.png)

公开 issue 里已经能看到同类现象：Hermes 侧有人报 `Error: 'NoneType' object is not iterable`；openai-python 侧也有 issue 指向 `response.completed` 里 `output=None` 后，`parse_response()` 无空值保护的问题。

故障排查第一步，不是动手修，是先别修错地方。

## 先判断你是不是同一个问题

不要凭感觉改代码，先看日志。

如果你符合下面几个特征，基本可以按本文处理：

```text
Provider: openai-codex
Model: gpt-5.5
TypeError: 'NoneType' object is not iterable
Non-retryable error
trying fallback
```

再跑一个最小验证：

```bash
hermes -m gpt-5.5 --provider openai-codex -z "ping" --no-stream
```

如果这条都失败，而你切到其他 provider 又能正常返回，那就别继续怀疑文章、skill、memory、gateway 了。

问题大概率就在 Codex 这条链路上。

## 方法 A：本地热补丁，先让 Codex 活过来

推荐先走这个方法，尤其是你还想继续用 `openai-codex` / `gpt-5.5`，不想临时迁移到别的 provider。

先进入 Hermes 源码目录。常见安装位置是：

```bash
cd ~/.hermes/hermes-agent
```

如果你的源码不在这里，先用自己的安装路径替换。不要在不确定路径的情况下复制命令。

第一步，备份原文件：

```bash
cp agent/codex_runtime.py agent/codex_runtime.py.bak
```

第二步，打开：

```bash
agent/codex_runtime.py
```

找到 `run_codex_stream` 里处理 stream 的异常逻辑，在 `except RuntimeError` 块后面，插入这个兜底：

```python
except TypeError as exc:
    err_text = str(exc)
    if "'NoneType' object is not iterable" not in err_text:
        raise
    if collected_output_items:
        return SimpleNamespace(
            output=list(collected_output_items), status="completed",
            model=api_kwargs.get("model"), usage=None,
        )
```

这段代码做的事情很克制：

只有遇到 `NoneType object is not iterable` 这个特定错误时，才接管；如果之前已经从 stream 里收集到了 output item，就直接用这些已收集内容组一个 completed response 返回。

它不是“吞掉所有错误”。

如果是别的 `TypeError`，继续抛出去；如果根本没有收集到输出，也不会假装成功。

这就是止血补丁该有的边界。

如果你不确定补丁该插在哪里，就不要靠肉眼硬猜。最稳的办法是先在文件里搜索 `except RuntimeError` 和 `collected_output_items`。这两个词同时出现在 `run_codex_stream` 附近，才说明你找到了同一段逻辑。

你也可以用下面这条命令辅助定位：

```bash
grep -n "except RuntimeError\|collected_output_items\|run_codex_stream" agent/codex_runtime.py
```

但定位只是定位，不等于可以闭眼改。真正落补丁前，至少确认三件事：

- 你改的是当前 Hermes 实际运行的源码目录，不是另一个旧 clone。
- 文件已经备份，改坏了可以立刻恢复。
- 你插入的是 `TypeError` 的特定兜底，不是把所有异常都吞掉。

如果你改完之后 Hermes 直接启动失败，通常是缩进错了。Python 对缩进很敏感，这种补丁最常见的失误不是逻辑错，而是把 `except TypeError` 放到了错误的层级。

恢复也很简单：

```bash
cp agent/codex_runtime.py.bak agent/codex_runtime.py
```

然后重新检查缩进，再改一次。

热补丁是止血，不是长期方案。

等 Hermes 或上游 SDK 正式修复合并后，你应该回到官方版本，而不是长期背着本地补丁跑。本地补丁的价值，是让你今天能继续工作；它不应该变成团队半年后的隐形债务。

## 方法 B：不动源码，先切 provider

如果你不想改本地代码，或者你只是想先把今天的工作流恢复，直接切 provider 更稳。

例如切到 DeepSeek：

```bash
hermes config set model.provider deepseek
hermes config set model.default deepseek-v4-flash
```

然后重新开一个 Hermes 会话，再跑一条：

```bash
hermes -z "ping" --no-stream
```

只要能正常返回，说明 Hermes 主流程没坏。你只是暂时绕开了 `openai-codex` 这条故障链。

这个方法的代价也要说清楚：

你换了 provider，也就换了模型能力、上下文表现、工具调用稳定性和成本结构。临时救火没问题，但如果你本来依赖 `gpt-5.5` 的输出风格或 Codex OAuth 链路，后面还是要切回来验证。

## 怎么确认修好了

别只看“没有报错”。修复后至少做三步。

第一步，测 Codex 最小调用：

```bash
hermes -m gpt-5.5 --provider openai-codex -z "ping" --no-stream
```

能正常返回，说明最小链路通了。

第二步，开一个正常 CLI 会话，问一个需要工具前不必动用复杂上下文的问题。比如：

```bash
hermes -m gpt-5.5 --provider openai-codex -z "用一句话回答：现在 primary provider 正常吗？" --no-stream
```

第三步，看日志里还有没有连续 fallback。

如果 primary 还在失败，只是 fallback 回答了你，那不算修好。那只是备用 provider 替你兜底了。

真正的修好，是 `openai-codex` 这条 primary 路径自己能走通。

还有一个容易忽略的点：如果你在 gateway、Telegram、Discord 里使用 Hermes，改完代码后可能需要重启对应进程。CLI 新开一次就会加载新代码，但长期跑着的 gateway 不一定会自动吃到你刚刚改过的文件。

所以验证顺序最好是：

```text
先测 CLI 最小命令
再重启 gateway
最后测真实入口
```

不要反过来。真实入口混着消息平台、gateway、profile、memory、fallback，一旦失败，你很难判断是补丁没生效，还是入口链路里另一个环节有问题。

## 常见误判：别把另一个 Codex 故障混进来

这里特别提醒一句。

Hermes 之前还遇到过另一类 Codex 后端变更：`extra_headers` 被后端拒绝，报的是 HTTP 400 和 `Unsupported parameter: extra_headers`。那是另一个坑。

这次本文处理的是：

```text
TypeError: 'NoneType' object is not iterable
response.completed 里的 output 可能是 null
```

两个问题都发生在 Codex 链路上，但修法不是一回事。

如果你看到的是 HTTP 400，就别照着本文去改 `NoneType`。如果你看到的是 token 过期，也别改 parser。技术故障最怕“一看都是 Codex，就当成同一个病”。

排障时先把错误文本贴出来看，不要只看 provider 名字。

## 哪些情况不要硬补

有几种情况，别急着套这段补丁。

如果你的日志是 `HTTP 400 Unsupported parameter: extra_headers`，那是另一类 Codex backend 参数校验问题，不是这次 `response.output=None` 的问题。

如果你的报错是认证失败、401、403、token expired，也不要往解析器上补。

如果你用的不是 `openai-codex` provider，而是 OpenRouter、Anthropic、DeepSeek、本地模型，那本文也不是你的第一排查路径。

补丁不是护身符。补错地方，只会让后面的排障更难。

## 我建议你怎么选

如果你现在正在工作，Hermes 是生产力工具，不是研究对象：先切 provider，把活干完。

如果你依赖 `gpt-5.5`，并且愿意承担本地热补丁的维护成本：打补丁，然后用最小命令验证。

如果你维护团队环境：不要让每个人手工改。先在一台机器上验证补丁，再做统一说明，等官方修复出来后统一回滚。

团队里最好把这次处理写成三句话，而不是丢一段代码让大家自己悟：

```text
现象：openai-codex / gpt-5.5 报 NoneType object is not iterable。
原因：Codex backend completed 帧可能返回 output=null，SDK 解析缺少兜底。
处理：要么临时切 provider，要么按统一补丁止血，等官方修复后回滚。
```

这样做的好处是，后来的人不会把“今天的临时补丁”误读成“Hermes 推荐配置”。

我更建议个人用户先选方法 B，团队维护者再评估方法 A。因为切 provider 的副作用是可见的：模型变了、输出变了，你马上能感知；本地补丁的副作用不一定马上暴露，尤其是在多人环境里。

这次故障最值得记住的，不是那一行 Python 怎么写。

而是：当 Agent 工具突然集体抽风，先判断故障层级。

是你的 prompt？你的配置？Hermes runtime？provider？SDK？还是上游接口返回变了？

层级判断对了，修复只需要几分钟。

层级判断错了，你会把一个上游兼容问题，修成一场本地系统重装。

如果你也在用 Hermes 做日常开发、写作和自动化，我会继续记录这类真实故障、补丁和工作流经验。

关注我，后面不讲玄学，只讲能救命的那几步。
