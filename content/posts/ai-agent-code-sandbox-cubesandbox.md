---
title: "当 Agent 开始替你跑代码，为什么只套一层 Docker 不够"
slug: "ai-agent-code-sandbox-cubesandbox"
date: 2026-07-15
draft: false
description: "Agent 的风险不是会不会执行一段 Python，而是把不受信任的代码、浏览器、网络和密钥放进哪个边界。Cube Sandbox 的价值不在于又造了一个虚拟机，而在于把 MicroVM、模板快照和出网治理做成了 Agent 运行时。"
tags: ["AI Agent", "Sandbox", "Docker", "KVM", "MicroVM", "安全", "后端架构"]
categories: ["AI 工程", "后端架构"]
cover: "/images/ai-agent-code-sandbox-cubesandbox/cover.png"
toc: true
---

# 当 Agent 开始替你跑代码，为什么只套一层 Docker 不够

你把 Agent 接进内部系统后，最危险的那一步通常不是模型回答错了。

而是它开始替你执行代码：装一个包，跑一段 Python，拉一个仓库，开 Chromium 点几个页面，再顺手访问外网。演示里这是一条 `run_code()`；线上，它意味着一段由模型拼出来、由用户输入影响、依赖链也未必干净的程序，正在拿你的 CPU、网络和文件系统做事。

很多团队的第一反应是：丢进 Docker，不就隔离了吗？

Docker 当然不是不能用。它有 namespace、cgroup、capability、seccomp，也能配 AppArmor、SELinux 和 rootless。问题在于，Agent 这类负载把它推到了一个更难的位置：你不是在跑一组自己写、自己发布的服务，而是在高频创建执行环境，里面的命令、依赖、网页和输入都更不受控。容器仍与宿主共享内核；一旦把 Docker daemon、宿主目录或过宽的 capability 暴露进去，隔离边界就会被你自己打穿。[1]

这时该换的问题了。

> **Agent 沙箱不是“能不能把代码跑起来”的组件，而是“代码跑错了，最多能碰到哪里”的运行时。**

Cube Sandbox 值得看，正是因为它没有把重点放在再包一层执行 API，而是把 MicroVM、模板快照、网络出口和凭据边界一起做成了面向 Agent 的运行时。它还很年轻，远没有到“装上就高枕无忧”的程度；但它提醒了后端团队一件容易被忽略的事：Agent 执行层，本身就是一套基础设施。

![封面：同一个 Agent 执行请求，从宿主机直连、普通容器，到受控 MicroVM；突出“代码、网络、密钥”三道边界。](/images/ai-agent-code-sandbox-cubesandbox/cover.png)

## 真正要防的，不是那一段 Python

假设你的 Agent 被允许“分析一个 CSV 并生成图表”。过几周，需求会自然长出下一层：

- 能不能 `pip install` 一个缺失库？
- 能不能读取用户上传的压缩包？
- 能不能抓一个网页再做摘要？
- 能不能调用企业内部 API 补充数据？
- 能不能保留上轮生成的文件，下一轮继续修改？

每加一个“能不能”，都不是单纯的工具功能，而是在扩大执行边界。

最直观的风险是任意代码。更难处理的是旁路：代码可以借网络带走数据，可以从环境变量或文件里摸到密钥，可以耗尽 CPU、内存和磁盘，也可能借浏览器自动化碰到本不该访问的后台页面。即使没有逃逸漏洞，只要权限、挂载和出口策略给得太宽，沙箱也会变成“专门给不可信代码准备的跳板”。

所以后端开发者不该只盯着 prompt injection。Prompt injection 是诱因；执行环境、网络策略、密钥注入和审计是否收口，才决定诱因最后能造成多大损失。

一条很实用的判断是：**只要 Agent 能执行用户可影响的命令，就先按多租户、不可信工作负载设计；即使今天只有一个内部用户。** 因为能力一旦接进产品，使用边界通常只会往外扩，不会自己缩回去。

## Docker、gVisor、Firecracker：卡住的不是同一个地方

这里很容易写成“Docker 太重、gVisor 不兼容、Firecracker 启动慢”。这种说法不准确，也会把选型带歪。

它们并不是从低到高排成一条鄙视链，而是在不同地方付成本。

### Docker：快，但边界要靠你补齐

容器的优势一直很清楚：镜像生态成熟，启动快，调试和运维成本低。对可信的内部任务、固定脚本、受控 CI job，它依然常常是最划算的选择。

但 Docker 的基本隔离建立在 Linux namespace 和 cgroup 上，容器与宿主共享同一个内核。Docker 官方也明确提醒：daemon 默认需要高权限，能控制 daemon 的用户本质上拥有很强的宿主机能力；把宿主目录挂进容器更会直接改变风险边界。[1]

对 Agent 来说，难点不是“容器必然不安全”，而是你要持续证明每一个容器都没有越权的挂载、特权、socket、capability 和网络出口。任务量从几十个变成几万次，策略配置、镜像供应链、文件清理和审计的遗漏都会累计。

### gVisor：多一道隔离，但别把兼容性当成不存在

gVisor 用用户态内核拦住大量系统调用，是容器与宿主内核之间很有价值的一层缓冲。它并不只适合 demo，官方列出了 Cloud Run、DigitalOcean App Platform 等实际使用场景，也说明 Python、Java、Node.js、PHP、Go 等运行时有回归测试覆盖。[2]

但它的定位决定了兼容性需要实测。gVisor 官方自己写得很直白：它实现的是 Linux ABI 的一个子集，仍有未实现功能和 bug；块设备文件系统、部分 iptables、受限的 `io_uring`、自定义硬件设备等都有边界。[2]

如果你的 Agent 只跑常规 Python 任务，gVisor 很可能够用。可一旦执行环境里混进浏览器、奇怪的二进制依赖、底层系统工具，问题就不再是“有没有隔离”，而是“这份工作负载在这个内核语义里能不能稳定跑”。

### Firecracker：不是慢，而是你得把快启动做成系统能力

Firecracker 证明了 MicroVM 不等于传统虚拟机的秒级冷启动。它同样支持快照恢复；官方文档也详细说明了快照如何保存和恢复运行中的 guest 状态。[3]

真正麻烦的是，把这项能力变成一个可运营的 Agent 平台：模板怎么构建、快照如何管理、克隆后的网络身份怎么处理、什么时候暂停、如何恢复、密钥从哪里进、每次外网访问如何审计。Firecracker 的快照文档甚至专门提示：跨进程恢复时网络和 vsock 可能丢包，已有连接也不保证保留；快照还要求与生成时匹配的软件和硬件配置。[3]

这不是 Firecracker 的缺点，而是运行时要承担的工作。许多团队缺的恰恰是这一层。

![正文插图 1：三种方案不是性能排名。Docker 付出“共享内核与策略治理”的成本；gVisor 付出“ABI 兼容性验证”的成本；MicroVM/Firecracker 付出“模板、快照与生命周期运营”的成本。](/images/ai-agent-code-sandbox-cubesandbox/tradeoff-map.png)

## Cube Sandbox 补的，是 Agent 运行时这一层

Cube Sandbox 的架构并不神秘：底下是 KVM MicroVM，上面是面向集群的生命周期、存储、网络和 API。

它的 `CubeHypervisor` 基于 RustVMM 和 KVM，为每个 sandbox 运行独立的 Linux kernel；`CubeShim` 接入 containerd Shim v2；控制面通过 CubeAPI、CubeMaster 和 Redis 管理创建、暂停、恢复与调度。[4]

只看这句“RustVMM + KVM”，它仍然只是 MicroVM。让它更像 Agent 基础设施的，是下面三件事。

### 模板不是镜像，快照才是它的启动路径

Cube Sandbox 会把 OCI 镜像构建成模板：准备 rootfs、冷启动一次、保存内存快照。后续创建 sandbox 时，rootfs 与内存卷从模板克隆，再从内存快照恢复，而不是每次从零启动完整 guest。[4]

存储层使用 XFS reflink 的 `FICLONE` 做 Copy-on-Write；官方文档把 rootfs 和内存卷的快照、克隆描述为 O(1) 元数据操作，不需要复制整份数据。[4]

这就是它能把“启动一台隔离 VM”压到适合 Agent 调用链的原因。项目官方在特定裸机环境的测试中宣称：单并发平均冷启动低于 60ms，50 并发创建时平均 67ms、P95 90ms、P99 137ms；每实例基础内存额外开销低于 5MB。它们是项目方基准，不能直接拿来替代你的机器、模板和并发模型上的压测，但至少说明快照路径是这个系统的核心设计，而不是一个附加功能。[5]

这里的工程判断比数字重要：**当 Agent 每一步都要新建环境时，启动时间不是优化项，它会直接改变你敢不敢把隔离放在默认路径上。**

### 代码隔离之外，还把网络和密钥收进边界

只隔离文件系统还不够。Agent 执行代码时，出口才是最常被低估的通道。

Cube Sandbox 的 CubeEgress 是宿主机上的透明 L7 出口代理。HTTP/HTTPS 请求会经过规则匹配：可以按 SNI、Host、方法、路径做 allow/deny；没有命中规则时默认拒绝。它也支持在代理侧注入 `Authorization` 等 header，让 secret 不进入 sandbox 的环境变量、文件系统或进程空间，并把 allow、deny、注入等决策记进 JSONL 审计日志。[6]

这比“在 Agent prompt 里要求不要泄漏 Key”靠谱得多。前者是在系统边界上拒绝，后者只是希望模型听话。

当然，别把这句话读成绝对安全。官方文档同样列出了边界：代理主要拦截 80/443；其他 TCP/UDP 流量要靠更底层的网络策略控制；没有把根证书烘进模板的 sandbox，HTTPS 流量也不会按预期经过透明检查。[6]

安全组件真正有价值的地方，不是宣称“什么都防住了”，而是把绕不过去的通道和绕得过去的通道都写清楚。

### 它把“闲置”当成 Agent 生命周期的一部分

Agent 不是一直在跑。用户发起任务，模型思考，sandbox 执行，接着可能空闲几分钟，等下一轮工具调用。

Cube Sandbox 支持把闲置实例暂停为快照，后续请求自动恢复。官方的生命周期模型里，暂停后不再占用实际 CPU/内存，文件系统和内核状态被保留；恢复的典型延迟是亚秒到数秒，具体取决于模板和环境。[7]

但这不是免费午餐。暂停快照仍占磁盘，实例对象也还在；如果为了密度释放了暂停实例的资源配额，恢复时可能因为节点容量不足返回冲突，需要重试。[7]

这恰好是它比“给容器加个 TTL”更像运行时的地方：状态、成本和可用性被摆在同一个生命周期模型里，而不是留给业务代码各自拼。

![正文插图 2：Agent 的一轮工作：用户请求 → 受控 MicroVM 执行代码/浏览器 → 出网规则与凭据注入 → 结果返回 → 空闲后快照暂停 → 下一轮恢复。标注“快照仍占磁盘，恢复有容量约束”。](/images/ai-agent-code-sandbox-cubesandbox/agent-lifecycle.png)

## 哪些场景会真的用到它

别因为它能跑 MicroVM，就把所有后台任务都往里面塞。

Cube Sandbox 的价值会在下面几种工作负载里变得具体：

**一是 Agent 执行 Python、Shell 和用户影响的依赖。**
比如数据分析、代码修复、文件处理、报表生成。你需要的不是一个能跑 `python` 的机器，而是一块默认没有宿主机凭据、默认没有任意外网、失败后能销毁的执行面。

**二是浏览器自动化。**
官方示例提供了在 MicroVM 内运行 headless Chromium、再通过 Playwright CDP 远程控制的路径。[8] 浏览器这类任务很适合放进独立环境：它的缓存、下载文件、登录态和网页脚本都不该和业务服务混在同一台机器上。

**三是并发的代码 Agent 或评测环境。**
项目示例包括 SWE-bench / mini-swe-agent 的隔离 sandbox 路径，以及 OpenAI Agents SDK 的 Shell Agent、Django 调试 Agent。[8] 这类任务经常需要大量相似、短生命周期、状态又彼此隔离的环境。模板克隆和快照回滚，比每次重新拉镜像、重新装环境更贴近问题本身。

**四是有状态的长任务。**
如果 Agent 需要在多轮之间保留工作目录、浏览器或本地服务状态，暂停/恢复和克隆的意义就出来了。它不是只为“跑完一条命令就死”的 code interpreter 准备的。

这些场景的共同点不是 AI，而是：执行内容不完全可信，运行环境需要频繁生灭，网络和密钥需要集中治理，且你不希望每个业务团队自己发明一套隔离方案。

## 更重要的问题：你愿不愿意接住它的运维成本

Cube Sandbox 不是一个轻量 SDK，而是一套运行时。它的架构里已经出现了 Redis、containerd、Buildkit、KVM、XFS reflink、eBPF 网络面、出口代理和控制面组件。[4]

这代表它解决的问题更多，也代表你要接住更多故障域：模板版本、内核与 KVM 能力、磁盘快照容量、网络策略、控制面状态、节点调度，以及升级时的兼容性。官方路线图里还把 Kubernetes 原生部署、跨节点暂停/恢复、故障恢复、调度增强列为“coming soon”。这不是扣分项，但说明有些生产运维能力仍在补齐。[9]

它也不是没有宿主前提。标准部署需要能提供 KVM 的 Linux 环境；当云厂商没有暴露 `/dev/kvm` 时，官方提供的是替换宿主内核并启用 PVM 的路径，而不是“随便一台普通实例就无条件可用”。[10]

所以，下面几种情况我不会急着推荐它：

- 你的任务都是可信、固定、低频的内部脚本，且容器策略已经足够严格；
- 你只需要一次性的异步作业，根本不需要快照、恢复和多租户隔离；
- 团队还没有能力维护 KVM、模板和网络策略，却先想借 MicroVM 获得“绝对安全”；
- 你没有明确的执行面需求，只是因为 Agent 热才想部署一套复杂运行时。

安全不是把组件堆得越多越好。安全是让攻击面、权限和运行成本，刚好匹配你真正开放出去的能力。

## 结尾：别让 Agent 直接继承你的机器

Cube Sandbox 想明白一件事：Agent 沙箱的关键不在“能不能跑代码”，而在“代码跑错后最多能碰到哪里”。

它把一个常被忽视的事实摆到了台面上：Agent 的执行器不是普通业务服务。它既接触不可信代码，也接触网络、密钥、文件和浏览器状态。把它直接放进业务宿主机，等于默认让模型的每次工具调用继承你的机器边界。

如果你正在给 Agent 加 `run_code`、Shell、浏览器或代码修复能力，先别急着比较哪个框架更聪明。先把下面三件事问清楚：

1. 这段代码最多能读到什么、写到什么、耗尽什么？
2. 它能访问哪些域名、哪些内部服务、以谁的身份访问？
3. 环境销毁、暂停、恢复和审计，谁负责，失败时怎么证明它没有留下脏状态？

Docker、gVisor、Firecracker 和 Cube Sandbox 都可能是答案的一部分。

但先问边界，再选运行时。别让 Agent 直接继承你的机器。

---

## 数据与参考

1. Docker 官方安全文档：namespace/cgroup 隔离、daemon 权限、宿主目录挂载与 user namespace。https://docs.docker.com/engine/security/
2. gVisor 官方兼容性文档：已覆盖的语言运行时与已知 ABI / 文件系统 / `io_uring` 等边界。https://gvisor.dev/docs/user_guide/compatibility/
3. Firecracker 官方快照文档：快照恢复语义、网络/vsock 与硬件软件匹配限制。https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md
4. Cube Sandbox 架构文档：KVM MicroVM、RustVMM、CubeShim、CubeCoW、控制/数据面与安全层。https://cubesandbox.com/architecture/overview.html
5. Cube Sandbox README：项目方在裸机环境的冷启动、并发创建和基础内存开销基准；实际数据应以自己的模板和环境压测为准。https://github.com/tencentcloud/CubeSandbox
6. Cube Sandbox Security Proxy：出网规则、凭据注入、审计与协议边界。https://cubesandbox.com/guide/security-proxy.html
7. Cube Sandbox Sandbox Lifecycle：暂停/恢复、空闲策略、资源配额和恢复拒绝边界。https://cubesandbox.com/guide/lifecycle.html
8. Cube Sandbox 示例列表：代码执行、Playwright 浏览器自动化、SWE-bench / mini-swe-agent、OpenAI Agents SDK 集成。https://cubesandbox.com/guide/tutorials/examples.html
9. Cube Sandbox Roadmap：Kubernetes 原生部署、跨节点暂停/恢复、故障恢复、调度增强仍列为后续工作。https://github.com/tencentcloud/CubeSandbox#roadmap
10. Cube Sandbox PVM 部署文档：无 `/dev/kvm` 的云环境需要 PVM 内核与额外部署步骤。https://cubesandbox.com/guide/pvm-deploy.html
