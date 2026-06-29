---
title: "你装的 Ollama，在多用户场景下会慢 20 倍——但这不是它的错"
date: 2026-06-29T20:00:00+08:00
draft: false
tags: ["LLM", "Ollama", "vLLM", "MLX", "AI", "本地部署", "技术选型", "后端开发"]
description: "一个人用很流畅，三个人同时用就卡炸了——你用 Ollama 的方式可能错了。不是因为 Ollama 不行，而是你的场景变了。"
keywords: ["Ollama", "vLLM", "本地LLM", "推理引擎选型", "MLX", "llama.cpp", "PagedAttention", "Continuous Batching"]
---
你装了 Ollama，跑个 Qwen3 8B。一个人用，很流畅。

然后给团队搭了个内部 chatbot。三个人同时用——卡炸了。

你以为是模型太大，换了更小的模型。还是卡。

这时候你可能开始怀疑：是不是 Ollama 性能不行？

不是。问题根本不在于 Ollama 快不快。而在于你把它用在了它不被设计服务的场景里。

---

## 一个人 v.s. 三个人，不是「慢一点」，是架构决定的

先说一组数据。同一块 GPU，跑同一个模型：

| 并发数 | Ollama（总 tok/s） | vLLM（总 tok/s） | 差距 |
|--------|-------------------|-----------------|------|
| 1 | ~62 | ~71 | ~1.1x |
| 8 | ~82 | ~187 | ~2.3x |
| 50 | ~155 | ~920 | ~5.9x |
| 100+ | 封顶 | 继续涨 | ~15-20x |

一个人用，Ollama 和 vLLM 几乎一样快。

两个人开始，vLLM 慢慢拉开距离。

五十个人？Ollama 的吞吐量基本封顶了——请求在排队，后来的等着前面的完成才能开始。vLLM 还在涨。

这不是 Ollama「性能不行」。「性能」这个词让你觉得是优化做得不够，换个版本、换个模型、换个量化格式就能追上。但这不是优化问题——是**架构不为此设计**。

Ollama 做推理的方式很简单：来一个请求，生成完，再来下一个。它不做 continuous batching。

什么叫 continuous batching？想象一个收银台。传统做法：排一队，一个一个结。轮到你才开始，结完才叫下一个。

Continuous batching 的做法：柜台一有空位，就让新人进来。不等前面的人全结完。柜台利用率直接拉满。

vLLM 就是这样调度请求的。每一轮推理迭代，它重新看一遍「现在谁还需要计算」，有空位就把新请求塞进去。GPU 利用率从 ~50% 拉到 85-92%。

Ollama 的设计目标从来不是「服务多用户」。它的价值是另一件事——让开发者用一条命令跑起一个模型。

```bash
ollama pull qwen3:8b
ollama run qwen3:8b
```

两条命令，从选模型到本地 API，全程不超过两分钟。这个体验就是 Ollama 的核心竞争力。

vLLM 的启动就没这么友好了：

```bash
vllm serve Qwen/Qwen3-8B
```

装依赖、配 GPU、调参数。第一次跑大概率会遇到 CUDA 版本问题和 OOM。但一旦跑起来，它就是一台真正的多用户推理服务器。

**Ollama 和 vLLM 不是「谁更快」的关系，是易用性和并发能力之间的取舍。Ollama 给了你极简的开发体验，代价是多用户场景下吞吐量差一个量级。vLLM 给了你 PagedAttention + continuous batching，代价是你要花时间把它跑起来。**

---

## 但等等——单用户场景，Ollama 确实和 vLLM 差不多

我上面说一个人用时它们差不多，不是客气话。就是字面意思。

看数据：RTX 4090 上跑 Llama 3.1 8B 4-bit，单用户场景：

| 引擎 | tok/s（约） |
|------|-----------|
| ExLlamaV2 | ~150 |
| TensorRT-LLM | ~140 |
| vLLM | ~120 |
| llama.cpp / Ollama（GGUF Q4_K_M） | ~110 |

Ollama 确实垫底，但差距是 110 对 120——不是一个量级的差距。这点差异主要来自量化格式（GGUF Q4_K_M vs AWQ/EXL2），不是引擎本身。

但这个前提很重要：**单用户**。

一旦变成多用户——团队共享一块 GPU、内部 chatbot、批量文档处理——Ollama 和 vLLM 的差距就从 1.1x 跳到了 15-20x。不是线性增长，是指数级拉开。

所以别只看单用户 benchmark 选工具。选型的变量不是「这个引擎每秒能吐多少 token」，而是「你的场景是几个人在用」。

---

## 等一下——Ollama 和 LM Studio，它们根本不是引擎

这可能是最容易搞混的事情。

Ollama 的底层跑的是 llama.cpp。LM Studio 的底层跑的还是 llama.cpp。

它们是 llama.cpp 的封装，不是竞争对手。就像你用 Docker 的时候不会觉得 Docker 和 containerd 是竞争对手——一个是你直接打交道的工具，一个是真正干活的东西。

而 llama.cpp 本身又是一个更底层的故事。它最初只是一个 GitHub 上业余项目——在 MacBook 上跑 LLaMA 的 C++ 实现。结果因为它引入了 GGUF 格式（把模型权重、分词器、元数据打包成一个文件，支持从 1.5-bit 到 8-bit 的各种量化），它成了整个本地 LLM 生态的基础。

GGUF 这个东西解决了什么？一个 FP16 精度的 7B 模型大约 14GB。Q4_K_M 量化后 4GB。8GB 显存的消费级显卡就能跑。

这层关系理清了，你选工具的时候就不会纠结「Ollama 和 llama.cpp 选哪个」这种伪问题。你没选 Ollama 而不是 llama.cpp——你选的是「llama.cpp + Ollama 的封装层」。

---

## Mac 上的事更复杂——但也更简单

如果你是 M 系列 Mac 用户，有一个好消息和一个坏消息。

好消息：Apple 有一个专门为 M 芯片做的框架叫 MLX。在 M4 Pro 上跑 Qwen3-Coder-30B，MLX 能到 ~130 tok/s，llama.cpp 只有 ~89，旧版 Ollama 只有 ~43。

MLX 快在哪？M 系列 Mac 的 CPU 和 GPU 共享一块物理内存池——Apple 叫它 Unified Memory。普通 PC 上，CPU 和 GPU 各有各的内存，数据要在 PCIe 总线上来回搬。Mac 上不用搬。这就是 MLX 能比 llama.cpp Metal 快 20-60% 的原因。

一台 192GB 的 Mac Studio，理论上能加载需要 4-8 张 A100 才能跑的大模型。当然，代价是 GPU 原始算力不如 NVIDIA——memory-bound 场景 Mac 赢，compute-bound 场景 NVIDIA 赢。

坏消息：别在 Mac 上尝试装 vLLM。它没有原生 Metal 支持，装上了也跑不起来。Mac 用户的推理引擎就是 MLX——或者直接用新版 Ollama（0.19+ 已经在 Mac 上自动切 MLX 后端了，43 tok/s 那个数字是旧版的历史数据）。

一个不直观的结论：如果你是 Mac 用户且已经在用 Ollama 0.19 以上版本，你不用做任何事。Ollama 已经自动帮你选了 MLX。

---

## 什么时候该切 vLLM

**你该用 Ollama，当：**
- 你在本地开发、做原型、调 prompt
- 只有你一个人用
- 你希望在 5 分钟内从零跑到 `ollama run`
- 你给代码写 integration test，需要一个本地 OpenAI 兼容 API

**你该切到 vLLM，当：**
- 团队开始共享一块 GPU
- 并发超过 3-5 个请求
- 你在给公司搭内部 chatbot / coding assistant
- 吞吐量瓶颈让你无法忍受

**你该用 SGLang（vLLM 的竞品），当：**
- 要做 Agent 工作流、RAG、多轮对话
- 大量请求共享同一个前缀（比如都挂在同一个 system prompt 上）
- SGLang 的 Radix Attention 会缓存这些共享前缀，tool-use 延迟更低

**你在 Mac 上就别折腾了：**
- MLX 是最优解
- 或者用 Ollama 0.19+（它已经在后台切 MLX 了）
- 别想装 vLLM，没有意义

---

## 三个最常踩的坑

**错误 1：用 Ollama 做生产多用户 API**

这是最贵的错误。一个人用完全没问题，三个人以上就该考虑 vLLM 了。不需要等到「卡炸了」再换——等那个时候用户体验已经很差了。

Ollama 的定位就是开发者原型工具。它不该被当生产系统用。

**错误 2：在消费级 24GB 卡上用 vLLM 还想跑其他 GPU 任务**

vLLM 默认启动就预占 90% VRAM。不是「需要的时候才占」，是「一启动就占了」。如果你只有一块 24GB 的卡，vLLM 吃完剩下 2.4GB，你连浏览器都开不了几个。

解决：`--gpu-memory-utilization 0.6` 把它降到 60%。

但反过来想：如果你只有一块 24GB 消费级卡，你是不是真的需要 vLLM？还是你的场景其实就是单用户开发？

**错误 3：追单用户 benchmark 数字决定用什么工具**

看了个 Reddit 帖子说「ExLlamaV2 单用户比 vLLM 快 20 tok/s」，就把整个 pipeline 换了。

单用户 tok/s 的数字，在真实工程里是最不重要的变量。量化格式（Q4 vs Q8 vs FP16）对 tok/s 的影响比你换引擎大。更重要的是——你要解决的是易用性还是并发？你是一个人还是十个人？你的瓶颈是 tok/s 还是请求排队？

这三个问题答完，工具自己就归位了。

---

本地跑 LLM 的工具生态看起来乱，但其实只有两件事要判断：

第一，你是一个人用，还是在给一群人提供 API。

第二，你在 Mac 上还是在 Linux+GPU 上。

两个问题答完，你的选项就只剩一两个。不存在「最好的本地 LLM 工具」。只存在最适合你当前场景的工具。而「当前场景」这件事，是会变的——你今天在 `ollama run` 做原型，明天团队开始用了，后天就该 `vllm serve` 了。

这不可怕。可怕的是明明十个人在用了，还在死守 Ollama，然后问为什么卡。
