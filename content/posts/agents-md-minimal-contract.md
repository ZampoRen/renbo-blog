---
title: "AGENTS.md 别再写成流程大全：它该替你的 AI Agent 把三件事说清楚"
slug: "agents-md-minimal-contract"
description: "Agent 进错目录、照着过期命令执行，只留一句“已完成”，根子经常不在提示词不够长，而在项目没有把边界、唯一事实源和验收条件交代清楚。"
date: 2026-07-22T17:45:50+08:00
draft: false
author: "Zampo"
cover: "/images/agents-md-minimal-contract/cover.png"
tags: ["AI Agent", "AGENTS.md", "Codex", "Claude Code", "Hermes", "工程实践"]
categories: ["工程实践"]
toc: true
---

Agent 回答“已完成”时，改动已经落到文件里了。人一复核，才发现它从仓库根启动，却把 `services/payments/` 的规则当成全项目规则；又或者它从几份互相打架的说明里，挑了一条已经废弃的测试命令跑了一遍。

这类返工很烦，因为 agent 看上去并没有偷懒。它读了文档，也执行了命令。只是它从一开始就在猜：该在哪个目录干活，哪份说明算数，怎样才算真的完成。

我现在看项目里的 `AGENTS.md`，先不看它写了多少流程。我先看它有没有把这三件事写死：边界、唯一事实源、完成标准。

**AGENTS.md 最有价值的工作，是让 agent 少猜。**

## 同名文件，进到不同 agent 里可能已经不是同一种东西

不少团队看到 `AGENTS.md` 这几个字，就默认它是一套可移植的项目约定。这个直觉很容易害人。

Codex 的官方说明里，项目指令会从项目根往当前工作目录逐层拼接，越靠近当前目录的文件越后出现。它还会在每一层优先选择 `AGENTS.override.md`，再选择 `AGENTS.md`。你在仓库根运行，和在 `services/orders/` 里运行，拿到的指令链可能不同。[1]

Claude Code 的主文件叫 `CLAUDE.md`。它会从当前工作目录向上找 `CLAUDE.md` 或 `CLAUDE.local.md`；子目录的内容也不会在启动时一股脑塞进来，Claude 读到对应子目录文件时才会按需带入。需要按目录或文件类型定规则时，官方提供的是 `.claude/rules/`。[2]

Hermes 也兼容 `AGENTS.md`，但发现语义又换了一套：当前文档说明，`.hermes.md` / `HERMES.md`、`AGENTS.md` / `agents.md`、`CLAUDE.md` / `claude.md`、Cursor rules 按优先顺序命中一种来源。`AGENTS.md` 与 `CLAUDE.md` 按当前工作目录查找，父目录和子目录副本不会自动合并；它自己的 `.hermes.md` 才会向上查到 Git 根。[3]

这不是要你给三套工具各写一份百科全书。它提醒的是：写任何上下文文件前，先把作用域问明白。

“从仓库根启动，适用于 `services/orders/**`。”

这句话看着土，却比一页构建历史更管用。它让人和 agent 都知道，眼前这份规则到底落在哪一块代码上。monorepo 里尤其要把这件事写出来：哪个子包能改，迁移目录能不能碰，当前任务从哪里启动。否则根目录有一份文件，只是维护者的想象；具体 agent 是否会读到、怎样叠加，取决于它自己的发现规则。

## 文件越像运行手册，过期后越容易把人带偏

我见过很多越写越长的 `AGENTS.md`：环境变量、端口、接口字段、部署顺序、线上故障记录、临时迁移状态，全堆在一起。每次 agent 按旧命令做错，大家就再补一句“注意”。

然后这份文件慢慢长成了一个谁都不敢删的缓存。

问题不在于命令不能写。稳定、每次都会用到的入口，当然该留。问题在于，同一个事实被复制到 `AGENTS.md`、README、Runbook 和 CI 里，迟早会分叉。agent 没有能力替你判断哪一份历史更新，它只会在自己读到的内容里挑一条看起来合理的路。

我更愿意让 `AGENTS.md` 只负责指路：接口定义去哪看，构建和测试以哪个目标为准，部署规则由哪份 runbook 维护。文件里给链接、给入口、给冲突时的优先级，不复制会变的细节。

**项目合同里最值钱的一句话，往往是“这个问题以哪一处为准”。**

这也解释了为什么“写得更全”常常没有换来更稳定的 agent。更多内容只会带来更多版本；版本一旦冲突，agent 仍然要猜。把事实收回到一个可维护的源头，才有可能把猜测收掉。

## “已完成”后面，得有一条人和机器都能复核的线

还有一种 `AGENTS.md` 看上去很认真，收尾那句却写着：改完后充分测试，确保没有问题。

这句话几乎等于没写。

它没有目标目录，没有具体检查，也没有通过判据。agent 可以跑一条和目标服务无关的测试，然后很诚实地告诉你“测试通过”。你也很难在 review 里追问：它到底测了什么，没测什么。

把完成条件写成能看到的东西，沟通会立刻变简单。比如下面这种格式：

```md
# Scope
- 本次只改 `services/orders/**`，从仓库根执行。
- 不触碰 `db/migrations/**`；迁移走专门流程。

# Source of truth
- API 字段以 `docs/orders-api.md` 为准。
- 构建和测试入口以 `Makefile` 的 `test-orders` 目标为准。

# Done means
- 改动位于 `services/orders/**`，除非任务单明确说明。
- `make test-orders` 退出码为 0。
- `golangci-lint run ./services/orders/...` 退出码为 0。
- 回报执行过的命令、未覆盖项和阻塞项。
```

目录和命令只是示意，换到你的项目时得换成真实入口。这个写法的重点也不在于让 agent 绝对听话。自然语言给的是行动解释，做不了强制门。

Claude Code 的官方文档把上下文文件和 hook 区分得很清楚：如果某个动作无论模型怎么判断都不能发生，该用 `PreToolUse` hook。项目里那些不能碰的发布动作、密钥路径、权限边界，同样该交给 CI、hook 或权限系统。Markdown 能减少歧义，挡不住违规操作。[2]

可观察的验收还有一个实际好处：它逼着 agent 的“完成”带上证据。它跑了哪条命令，退出码是什么，哪一块没覆盖，卡在哪个前提上。人接手时不必从一句“已完成”里猜现场。

## 真正要维护的是三处边界

你不用把现有文件推倒重写。下次 agent 又跑偏，直接拿这三件事对照：

- 它到底从哪里启动，规则覆盖哪个目录，哪些区域禁止碰？
- 这个问题的权威来源在哪，文件里有没有复制一份迟早过期的细节？
- “完成”靠什么观察，命令、通过条件和未覆盖项有没有说出来？

前两项解决它往哪里走、听谁的；第三项让它别把自我陈述当交付。

项目里的上下文文件不需要显得无所不知。让它像一份短合同：边界讲清楚，事实指到正确的地方，完成时交出能复核的证据。下次再看到 agent 说“已完成”，你就能立刻知道该问哪一句：它在哪儿改的，依据哪份事实，拿什么证明这次真的做完了？

---

## 参考资料

1. OpenAI, [Custom instructions with AGENTS.md](https://developers.openai.com/codex/guides/agents-md/)
2. Anthropic, [How Claude remembers your project](https://docs.anthropic.com/en/docs/claude-code/memory)
3. Hermes Agent, [Project Context Files](https://hermes-agent.nousresearch.com/docs/)
