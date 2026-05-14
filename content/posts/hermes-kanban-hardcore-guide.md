---
title: "Hermes Kanban 实战：别再用聊天记录管理多 Agent 团队"
description: "一篇从命令到流水线的 Hermes Kanban 硬核使用指南：任务创建、依赖链、dispatcher、review-required、Dashboard、worker_context 和技能整合。"
date: 2026-05-13T15:36:00+08:00
draft: false
author: "任博"
tags: ["Hermes", "Kanban", "AI Agent", "多 Agent", "工作流"]
categories: ["技术实战"]
cover: "/images/hermes-kanban-hardcore-guide/cover.svg"
toc: true
---

你把任务卡创建好了，researcher、writer、coder 也都分配了。

然后什么都没发生。

这不是 bug。恰恰相反，这是 Hermes Kanban 最容易被误解、也最值得讲清楚的地方：**创建 card，不等于启动 worker。**

很多人把多 Agent 系统想得太像“开几个聊天窗口”。一个负责调研，一个负责写稿，一个负责代码，大家各干各的。听起来很美，真跑起来就会发现：谁先干？谁等谁？谁审核？交付物放哪？失败了谁接？

靠聊天记录和几个 Markdown 文件也能凑合跑。但只要任务链稍微长一点，协作就开始变脆。

Hermes Kanban 解决的不是“让 Agent 更聪明”。它解决的是另一个更工程的问题：

**把 Agent 的工作，放进一条可追踪、可审核、可恢复的流水线里。**

这篇不讲概念安利，只讲怎么用。

![Hermes Kanban 看板与命令主视觉](/images/hermes-kanban-hardcore-guide/cover.svg)

## 先记住一句话：Kanban 是任务系统，不是魔法按钮

Hermes Kanban 是一块持久化的任务板。你可以用它创建任务、分配 assignee、声明 parent/child 依赖、绑定 workspace、记录 comment、阻塞等待审核、完成后给下游传 handoff。

它适合管这种任务：

- 一个任务需要多个 profile 接力，比如 researcher → writer → default；
- 下游必须等上游交付后才能开始；
- 交付物必须有路径、有验证、有审核记录；
- 任务失败后不能靠人翻聊天记录猜状态；
- 你希望 worker 不只是“答一句”，而是按岗位技能执行。

它不适合管这种任务：

- 你只是问一句命令怎么写；
- 任务几分钟就能完成，没有上下游；
- 没有明确交付物，也不需要审核；
- 你还没把需求想清楚，只是随手丢一个念头。

这也是为什么 Hermes Kanban 的六列看板很重要。它不是装饰，它是任务生命周期。

![Hermes Kanban 六列看板示意](/images/hermes-kanban-hardcore-guide/inline-01.svg)

六列可以这样理解：

| 列 | 含义 | 你该做什么 |
|---|---|---|
| Triage | 原始想法，还不是可执行任务 | 点 Specify，或用 `hermes kanban specify` 补成任务简报 |
| Todo | 任务已创建，但依赖未满足或还没分配 | 等 parent 完成，或补 assignee |
| Ready | 已经可执行，等 dispatcher 或人工 claim | 检查 gateway/dispatcher 是否在跑 |
| In Progress | worker 已 claim，正在执行 | 看 heartbeat、runs、log |
| Blocked | 等审核、等补料、或断路器触发 | 看 block reason，决定 unblock 还是打回 |
| Done | 任务完成 | 看 summary 和 metadata，给下游消费 |

不要小看这六列。多 Agent 协作最大的问题，不是 Agent 不会做事，而是状态没人管。

## 第一步：创建一张真的可执行的任务卡

最小命令长这样：

```bash
hermes kanban create "AI Agent 任务面板调研" \
  --assignee researcher \
  --body "研究题目和要求..." \
  --workspace dir:/Users/zampo/.hermes/management/assets/research/task-panel-article \
  --idempotency-key task-panel-research-20260513
```

这里有几个字段必须认真填。

`--assignee researcher` 决定这张卡由哪个 profile 执行。不是写个名字好看，dispatcher 后面就是按 assignee 去拉起对应 profile。

`--body` 是任务简报。别只写一句“调研一下”。你应该把目标、输入、输出要求、验收标准、风险边界写进去。body 写得烂，worker 执行再认真也会跑偏。

`--workspace` 决定任务工作目录。Hermes Kanban 支持三种 workspace：

```bash
--workspace scratch
--workspace dir:/Users/zampo/path/to/workspace
--workspace worktree
```

`scratch` 适合临时任务。`dir:<path>` 适合你已经有固定资料目录或博客项目。`worktree` 更适合代码任务，尤其是多个 worker 并行改同一个仓库时，避免互相踩文件。

`--idempotency-key` 很实用。重复执行同一条创建命令时，Kanban 会返回已有任务 id，而不是新建一堆重复卡。自动化脚本里强烈建议加。

如果你想拿 JSON 输出，可以加：

```bash
hermes kanban create "调研任务" \
  --assignee researcher \
  --json
```

这在后面串 parent/child 时很好用。

## 第二步：用 parent/child 把任务链串起来

多 Agent 最怕“下游抢跑”。researcher 还没交材料，writer 已经开始脑补；writer 还没预览，default 已经准备发布。最后不是效率高，是事故快。

Hermes Kanban 的做法是用 parent/child 依赖链卡住顺序。

```bash
# 先创建上游
RESEARCH=$(hermes kanban create "调研任务" \
  --assignee researcher \
  --json | jq -r .id)

# 下游自动依赖上游
hermes kanban create "写作任务" \
  --assignee writer \
  --parent $RESEARCH
```

这个 child 一开始不会直接进入 ready。它会待在 todo，直到 parent complete。上游完成后，Kanban 会把下游从 todo promote 到 ready。

这点很关键。

**依赖不是提醒，是闸门。**

没有 parent complete，下游 worker 不应该开工。这样你就不会靠“我记得上游差不多做完了”来启动任务。

如果任务已经建好了，也可以后补依赖：

```bash
hermes kanban link <parent_task_id> <child_task_id>
```

发现依赖关系写错了，再解除：

```bash
hermes kanban unlink <parent_task_id> <child_task_id>
```

真实任务链通常会长这样：

```text
researcher card
  → 研究完成，comment 交付物路径
  → block: review-required
  → default 审核通过，complete researcher card
  → writer child card 自动 promote 到 ready
  → writer 写稿、配图、Hugo 预览
  → writer block: review-required
  → default 审核
  → default complete writer card
  → default 执行 content-publishing-gate
```

![researcher 到 writer 再到 default 的任务链](/images/hermes-kanban-hardcore-guide/inline-02.svg)

这套流程慢吗？如果你只看单步，它肯定比直接丢给一个 Agent 慢。但如果你算上返工、漏交付、误发布、上下游扯皮，它反而更快。

## 第三步：会看任务，才谈得上调度任务

创建完任务，不要马上问“为什么没跑”。先看状态。

常用命令就这几个：

```bash
hermes kanban list
hermes kanban list --assignee writer
hermes kanban list --status ready
hermes kanban show <task_id>
hermes kanban assignees
hermes kanban runs <task_id>
hermes kanban stats
```

`list` 看任务列表，适合扫全局。

`show` 看单张卡详情，里面有 body、comments、events、runs。worker 接单前应该先读这个，而不是只看标题。

`assignees` 看有哪些 profile，以及每个 profile 名下有多少任务。

`runs` 看一次任务尝试的历史。比如哪次 claim 了、哪次 blocked、哪次因为 API error 挂了。

`stats` 看整体状态分布。ready 堆太多，通常说明 dispatcher 没跑、worker 起不来，或者 assignee 配错。

任务执行中还可以写 heartbeat：

```bash
hermes kanban heartbeat <task_id> \
  --note "已读取任务 body 和上游交付物，正在做写前检查。"
```

heartbeat 的价值不是“汇报一下我还活着”。它是给 default 和后续接手者看的执行痕迹。任务被打断后，别人能知道它停在哪里。

## 第四步：搞清楚 dispatcher，不然你会一直等

Kanban 卡在 ready，并不代表 Hermes 坏了。

ready 的意思是：任务已经具备执行条件，正在等待 worker claim。

谁来 claim？通常是 dispatcher。

Hermes 的 dispatcher 默认跑在 gateway 里。你可以启动 gateway：

```bash
hermes gateway start
```

如果你只想手动触发一次调度，可以跑：

```bash
hermes kanban dispatch
```

想先看看会发生什么，不真的 spawn worker：

```bash
hermes kanban dispatch --dry-run
```

也可以完全绕过 dispatcher，手动拉起对应 profile 去执行：

```bash
hermes -p writer chat -q "请执行 Kanban 卡片 t_xxx，按任务 body 要求交付。" \
  -s copywriting-work-basics \
  -s wechat-official-account-writer
```

这里一定要分清：

**create 是建卡，dispatch 是派工，profile chat 是执行。**

创建 card 只是把任务放进系统。gateway/dispatcher 没运行，任务就会老老实实待在 ready。它不会自己凭空变成一段后台魔法。

这反而是好事。一个可靠的任务系统，应该让你知道“为什么没执行”，而不是偷偷尝试、偷偷失败、偷偷丢日志。

## 第五步：把 review-required 当成发布门，不要当成麻烦

多 Agent 工作流里，最容易出问题的是“完成”的定义。

worker 说完成了，到底是写完了，还是验证过了？是文件真的存在，还是只在回复里说“已生成”？是可以发布，还是需要人审？

Hermes Kanban 里最稳的模式是：worker 不直接 complete 发布级任务，而是 comment 交付说明，然后 block 到 review-required。

员工交付：

```bash
hermes kanban comment <task_id> "交付说明：
- 稿件路径：/Users/zampo/Documents/renbo-blog/content/posts/xxx.md
- 图片路径：/Users/zampo/Documents/renbo-blog/static/images/xxx/
- 本地构建：hugo 通过
- 预览链接：http://127.0.0.1:1313/posts/xxx/
- 风险：等待 default 审核内容价值与发布门"
```

然后阻塞：

```bash
hermes kanban block <task_id> \
  "review-required: Hugo 成稿和配图已完成，等待 default 审核。"
```

default 审核通过后再完成：

```bash
hermes kanban comment <task_id> "审核结论：通过，进入发布门。"

hermes kanban complete <task_id> \
  --summary "审核通过，Hugo 成稿可进入 content-publishing-gate。"
```

如果审核不通过，不要新开一张乱七八糟的修订卡。直接 unblock 原卡，让 worker 按 comment 修订：

```bash
hermes kanban unblock <task_id>
```

这套流程看起来“重”，但它压住了一个老问题：Agent 很容易把“我生成了内容”说成“任务完成”。

**交付物存在，不等于可以发布。**

review-required 就是把这条边界写进流程里。

## 第六步：用 complete 的 summary 和 metadata 给下游递刀

很多任务链断在 handoff 上。

上游完成时只写一句“已完成调研”，下游打开卡片一脸茫然：文件在哪？哪些事实确认过？哪些是推断？哪些风险不能写死？

Hermes Kanban 的 `complete` 支持 summary 和 metadata。你应该把它当成结构化 handoff，而不是结束语。

例如：

```bash
hermes kanban complete <task_id> \
  --summary "调研完成：已交付 researcher-to-writer handoff，覆盖 Kanban 命令、Dashboard、dispatcher、worker_context。" \
  --metadata '{
    "changed_files": ["researcher-to-writer-handoff.md"],
    "decisions": ["文章应写成硬核教程，不写成产品介绍"],
    "verification": ["命令来自 hermes kanban --help 与本次实战链路"]
  }'
```

下游 worker 用 `hermes kanban show <task_id>` 就能看到 parent 的 closing run。更进一步，也可以看完整上下文：

```bash
hermes kanban context <task_id>
```

这就是 worker_context 的价值：下游不需要靠记忆接单，也不需要翻十几段聊天记录。任务 body、parent 结果、comments、events 都在同一个地方。

你可以把 summary 理解成“给人看的交接摘要”，metadata 理解成“给系统和下游自动化读的结构化字段”。

建议至少放这些字段：

```json
{
  "changed_files": ["交付物路径"],
  "decisions": ["关键判断"],
  "verification": ["已跑过的检查"],
  "risks": ["仍需人工确认的事项"]
}
```

别把 metadata 当摆设。多 Agent 流水线越长，结构化 handoff 越值钱。

## 第七步：把技能系统接进 Kanban，而不是让 worker 临场发挥

Hermes 的一个强项是 skills。Kanban 管任务，skills 管做法。两者应该绑在一起。

最直接的方式，是在创建 card 时指定技能：

```bash
hermes kanban create "公众号成稿" \
  --assignee writer \
  --skill copywriting-work-basics \
  --skill wechat-official-account-writer \
  --body "请按标准写作流程执行：读材料、构思、写稿、配图、Hugo 预览、提交 review-required。"
```

也可以在卡片 body 里直接写清楚：

```text
请使用 /content-publishing-gate 执行发布门检查。
请使用 /flow-hugo-publisher 完成 Hugo 预览与交付前检查。
请使用 /humanizer 做去 AI 味自检。
```

如果某些技能放在外部目录，profile 配置里要把目录接进来。典型做法是在 profile 的 `config.yaml` 里配置 `external_dirs`，让对应 profile 能发现团队公共技能包。

启动 worker 时也可以显式加载技能：

```bash
hermes -p writer chat \
  -s copywriting-work-basics \
  -s wechat-official-account-writer \
  -q "执行 Kanban 卡片 t_xxx"
```

这件事很重要。

没有 skills，Kanban 只能保证“谁做、什么时候做、做完怎么交”。它不能自动保证 writer 会按公众号写法、researcher 会按事实边界、default 会按发布门执行。

**Kanban 管流程，skills 管手艺。**

把两者绑起来，worker 才不会每次都像刚入职。

## 第八步：Dashboard 适合盯全局，CLI 适合做动作

命令行适合精确操作，但任务多了以后，还是需要一块可视化面板。

启动 Dashboard：

```bash
hermes dashboard
```

默认地址是：

```text
http://127.0.0.1:9119
```

Dashboard 里最常用的是几件事。

第一，看六列状态。你能一眼看出任务卡在哪个阶段，ready 是不是堆积，blocked 是不是没人处理。

第二，按 profile 分 lane。researcher、writer、coder、default 各自有哪些任务，谁在跑，谁卡住，比纯命令行扫列表更直观。

第三，打开 drawer 看详情。body、comments、events、runs 都可以在详情面板里看。

第四，nudge dispatcher。任务在 ready 但 dispatcher 没动时，可以从界面触发调度。

第五，处理 Triage。原始想法可以先进 Triage，再点 Specify，让它变成可执行任务。

我自己的习惯是：CLI 负责创建、comment、block、complete；Dashboard 负责看状态、查阻塞、盯全局。

两者不是替代关系。CLI 是手术刀，Dashboard 是监控屏。

## 一条可复用的完整流水线

如果你要搭一条内容生产链，可以直接照这个模板改。

第一步，创建 researcher 任务：

```bash
RESEARCH=$(hermes kanban create "Hermes Kanban 技术调研" \
  --assignee researcher \
  --workspace dir:/Users/zampo/.hermes/profiles/researcher/workspace/hermes-kanban \
  --body "输出 researcher-to-writer handoff，区分已确认事实、来源说法和推断。" \
  --idempotency-key hermes-kanban-research-20260513 \
  --json | jq -r .id)
```

第二步，创建 writer 子任务：

```bash
WRITING=$(hermes kanban create "Hermes Kanban 公众号成稿" \
  --assignee writer \
  --parent $RESEARCH \
  --workspace dir:/Users/zampo/Documents/renbo-blog \
  --skill copywriting-work-basics \
  --skill wechat-official-account-writer \
  --body "消费 researcher handoff，产出 Hugo 主稿、配图、本地预览结果，并提交 review-required。" \
  --json | jq -r .id)
```

第三步，researcher 完成后提交审核：

```bash
hermes kanban comment $RESEARCH "交付说明：
- handoff: /path/researcher-to-writer-handoff.md
- 原始资料：/path/sources/
- 风险：部分数据为来源说法，写作时不能写成已确认事实。"

hermes kanban block $RESEARCH \
  "review-required: 调研交付物已完成，等待 default 审核完整性。"
```

第四步，default 审核通过并 complete researcher：

```bash
hermes kanban complete $RESEARCH \
  --summary "调研审核通过，可供 writer 使用。" \
  --metadata '{"changed_files":["/path/researcher-to-writer-handoff.md"],"verification":["资料路径已检查"]}'
```

这时 writer 卡会自动从 todo promote 到 ready。

第五步，writer 写稿、配图、预览后提交 review-required：

```bash
hermes kanban comment $WRITING "交付说明：
- 稿件路径：/Users/zampo/Documents/renbo-blog/content/posts/hermes-kanban-hardcore-guide.md
- 图片路径：/Users/zampo/Documents/renbo-blog/static/images/hermes-kanban-hardcore-guide/
- 本地构建：hugo 通过
- 预览链接：http://127.0.0.1:1313/posts/hermes-kanban-hardcore-guide/"

hermes kanban block $WRITING \
  "review-required: Hugo 成稿和配图已完成，等待 default 审核。"
```

第六步，default 审核，通过后执行发布门。这里不是 writer 的权限范围。正式 git commit、push、博客发布确认、微信公众号投递，都应该由 default 使用对应发布技能执行。

这条链路的核心不是命令多，而是边界清楚。

researcher 不写文章，writer 不发布，default 不靠感觉审核。每个环节都有输入、输出、路径、验证和状态。

## 常见坑：90% 都不是 Kanban 本身的问题

第一，body 写得太随便。

你给 worker 一张只有标题的卡，它当然只能猜。Kanban 不是需求分析师。任务简报越具体，worker 执行越稳。

第二，创建卡后忘了 dispatcher。

ready 只是“可执行”，不是“正在执行”。gateway 没开、dispatcher 没跑、profile 起不来，任务就会停在 ready。

第三，把 blocked 当失败。

blocked 很多时候是正常状态，尤其是 review-required。它说明流程在等人审，不说明任务坏了。

第四，worker 自己 complete 发布级任务。

这会绕过审核。内容、代码、发布这类任务，只要有外部影响，默认应该 block 到 review-required，让 default 过发布门。

第五，comment 不写路径。

“已完成”没有价值。交付物路径、构建结果、预览链接、风险，才有价值。下游只认能检查的东西。

第六，skills 没接进来。

没有技能约束，profile 很容易按通用助手方式执行。写作任务会写成报告，研究任务会写成摘要，发布任务会漏检查。

第七，metadata 留空。

短任务无所谓，长链路一定会痛。summary 和 metadata 是下游理解上游工作的入口，不要省。

## 最小实践清单

如果你现在就想把 Hermes Kanban 用起来，不要一上来设计复杂系统。先照这个顺序做。

1. 给每类 worker 分清 profile：researcher、writer、coder、default。
2. 给每类任务写一个固定 body 模板。
3. 任务创建时必须带 assignee、workspace、idempotency-key。
4. 有上下游就用 parent/child，不靠口头提醒。
5. 交付必须 comment，comment 必须有路径。
6. 需要人工审核的任务，一律 block: `review-required:`。
7. default 审核通过后再 complete。
8. 完成时写 summary，重要任务补 metadata。
9. gateway/dispatcher 负责自动派工；必要时用手动 profile chat 执行。
10. Dashboard 看全局，CLI 做精确动作。

做到这十条，多 Agent 协作就已经从“聊天窗口堆叠”变成“任务流水线”。

这也是 Hermes Kanban 最有价值的地方。

它没有假装 Agent 会突然变成一个成熟团队。它承认 Agent 需要任务、依赖、审核、交接和发布门。

人类团队要这些东西，Agent 团队也一样。

区别只是，过去这些流程写在会议、Slack 和人的记忆里；现在，它们应该写进 Kanban。