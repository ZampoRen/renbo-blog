---
title: "Gemma 4 刚发布，我连夜把它装进了 OpenClaw"
date: 2026-04-03T10:20:00+08:00
draft: false
author: "任博"
description: "别再等别人写教程了。Gemma 4 发布后，最稳的一条本地路线就是：hf 下载 GGUF，Ollama 导入，再接进 OpenClaw。本文把这条链路一步一步走通。"
tags: ["Gemma 4", "OpenClaw", "Ollama", "Hugging Face", "本地部署", "AI"]
categories: ["技术实战"]
featured: false
toc: true
---

> 昨天 Gemma 4 刚发，今天它已经跑在我本地了，而且接进了 OpenClaw。

很多人一看到新模型发布，第一反应是去搜教程。

然后就会看到两类内容：

1. 一堆参数表，读完还是不知道怎么装
2. 直接甩一句 `ollama pull xxx`，结果你一跑就报错，因为库里根本还没有，或者版本不对

所以这篇文章我不聊空话，只讲一条真的能跑通的链路：

```text
hf download
→ ollama create
→ OpenClaw 接入
```

这条路的好处很现实：

- 你知道模型文件是从哪里来的
- 你知道自己下的是哪个量化版本
- 你不需要赌 Ollama 官方库有没有同步
- 你装完就能在 OpenClaw 里用，不是“理论上可行”

如果你只想要一句结论，那就是：

**先别冲 31B，也别一上来折腾一堆转换脚本。先从 Gemma 4 E4B 开始，用 GGUF 跑通本地链路，这是最稳的。**

<!--more-->

---

## 为什么我推荐你先装 E4B

Google 在 Hugging Face 上公布的 Gemma 4 家族，目前主要有这几档：

| 型号 | 架构 | 上下文 | 模态 |
|------|------|--------|------|
| `E2B` | Dense | 128K | 文本 / 图像 / 音频 |
| `E4B` | Dense | 128K | 文本 / 图像 / 音频 |
| `26B A4B` | MoE | 256K | 文本 / 图像 |
| `31B` | Dense | 256K | 文本 / 图像 |

看起来 31B 最强，26B A4B 也很诱人。

但如果你是想把它真正装到自己机器里，再接进 OpenClaw 做日常工作，事情没那么复杂：

- 你需要先跑起来
- 你需要响应速度别太离谱
- 你需要别因为第一次尝试就被内存和温度劝退

这时候，E4B 是最合理的起点。

原因很简单：

- 它不是最小，但比 E2B 更有实用价值
- 官方给了 128K context
- 官方模型卡明确写了文本、图像、音频能力
- 做成 GGUF 后，更适合走本地这条链路

我的建议很明确：

**第一步先装 E4B，把整条链路跑通。**

别一开始就拿 31B 证明自己。

能装上、能接入、能进入工作流，这比参数表好看重要得多。

---

## 这次为什么不用 `ollama pull`

因为这次我们要的是“稳”，不是“碰运气”。

很多新模型刚发的时候，最常见的问题就是：

- 官方模型库还没同步
- 社区已经同步了，但名字不统一
- 你看到别人写的 tag，自己本地根本拉不下来

而 Ollama 官方文档给出的更稳妥路径，其实是：

先拿到本地模型文件，再写 `Modelfile`，最后用 `ollama create` 导入。

对于 GGUF，最核心的写法就是：

```text
FROM /path/to/file.gguf
```

说白了就是：

**别赌。自己把模型文件下到本地，再让 Ollama 接管。**

这也是为什么我这篇文章会用：

```text
hf download -> ollama create
```

而不是直接 `ollama pull`。

---

## 第一步：装好 Ollama

先把运行环境装起来。

macOS / Linux：

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Windows 直接去官网下载：

`https://ollama.com/download`

装完先看版本：

```bash
ollama --version
```

如果后面你碰到 `connection refused`，通常不是模型有问题，而是服务没起来。手动启动：

```bash
ollama serve
```

这一步很基础，但别跳。

很多人明明模型文件都下好了，最后其实是卡在 Ollama 服务没启动。

---

## 第二步：装 Hugging Face CLI

如果你平时就用 `pipx` 管理命令行工具，最顺手的方式就是：

```bash
pipx install huggingface_hub
```

装完确认一下：

```bash
hf --help
```

如果你之前没登录过 Hugging Face，先登录：

```bash
hf auth login
```

看一下当前身份：

```bash
hf auth whoami
```

还有两个命令后面很有用，先记住：

```bash
hf cache ls
hf download <model-id> <文件名>
```

我现在越来越喜欢 `hf` 这套新 CLI。

原因很简单：短，清楚，不像以前那种全名命令又长又重。

---

## 第三步：把 Gemma 4 的 GGUF 文件下到本地

Gemma 4 官方模型在 Hugging Face 上是：

```text
google/gemma-4-E4B-it
```

但如果你想最省事地接入 Ollama，不要直接拿 Safetensors。  
更稳的办法是直接下已经量化好的 GGUF。

现在 Hugging Face 上已经有不少 Gemma 4 E4B 的 GGUF 仓库。  
这次我直接换成你更容易复测的一份：

```text
unsloth/gemma-4-E4B-it-GGUF
```

为什么选它？

不是因为它“最权威”，而是因为它现在更适合拿来实测：

- 是你当前要验证的实际仓库
- 文件命名比前面那份更直接
- 适合把“下载是否正常、导入是否正常”先跑通
- 对排查“模型坏了还是接入层有问题”更有帮助

先建目录：

```bash
mkdir -p ~/models/gemma4-e4b
cd ~/models/gemma4-e4b
```

然后下载：

```bash
hf download unsloth/gemma-4-E4B-it-GGUF \
  gemma-4-E4B-it-UD-Q4_K_XL.gguf \
  --local-dir .
```

下载完确认一下：

```bash
ls -lh
```

你应该会看到类似这样的文件：

```text
gemma-4-E4B-it-UD-Q4_K_XL.gguf
```

为什么我这里先推荐 `UD-Q4_K_XL`？

因为这是一个很典型的“别一上来就想要最强”的选择。

- 太小的量化，效果容易掉得明显
- 太大的量化，体积和内存压力又上去了
- `UD` 这类动态量化，本来就是 Unsloth 这条线的重点

如果你只是想先验证这份仓库能不能正常下载、能不能正常进 Ollama，先从这一档开始最省事。

如果你的机器更强，后面可以再试同仓库里的更高量化版本。

但第一次，先别折腾。先把这一档跑起来。

---

## 第四步：导入到 Ollama

模型文件有了，接下来就是最关键的一步：把它交给 Ollama。

在同目录里创建一个 `Modelfile`：

```bash
cat > Modelfile <<'EOF'
FROM ./gemma-4-E4B-it-UD-Q4_K_XL.gguf

PARAMETER num_ctx 65536
PARAMETER temperature 1.0
PARAMETER top_p 0.95
PARAMETER top_k 64
EOF
```

这里我解释两个容易被忽略的点。

### 1. 为什么 `num_ctx` 我先写 65536

官方模型卡写的是 128K。

但“官方支持 128K”和“你本地就该直接开 128K”，完全不是一回事。

如果你是本地跑模型，尤其还是准备接进 OpenClaw 这种真实工作流里，我更建议你先从 `64K` 起步。

原因很现实：

- 64K 已经足够覆盖大部分长文档和 Agent 场景
- 内存压力会比 128K 好很多
- 响应速度也更容易接受

**别为了追一个好看的数字，把自己的机器搞得半死不活。**

### 2. 为什么把采样参数直接写进 Modelfile

Gemma 4 模型卡里对采样参数给了建议：

- `temperature=1.0`
- `top_p=0.95`
- `top_k=64`

我这里直接写进去，是因为你后面从 OpenClaw 调它的时候，就不用每次再单独配一遍。

现在导入：

```bash
ollama create gemma4-e4b -f Modelfile
```

导入完成后，确认一下：

```bash
ollama list
```

你应该能看到：

```text
gemma4-e4b
```

然后先手动跑一轮：

```bash
ollama run gemma4-e4b "用三句话介绍一下你自己。"
```

只要它能正常输出，说明你前面最难的部分其实已经过了。

---

## 第五步：接进 OpenClaw

如果你走到这里，说明 Gemma 4 已经在你机器上了。

接下来不是“继续研究”，而是把它真正接进工作流。

这一步，别再照着旧教程去写 YAML 了。  
OpenClaw 现在对 Ollama 的接法已经更直接。

### 方案 A：第一次接，直接走 onboarding

先把 Ollama provider 打开：

```bash
export OLLAMA_API_KEY="ollama-local"
```

然后执行：

```bash
openclaw onboard
```

在向导里：

1. 选择 `Ollama`
2. Base URL 填 `http://127.0.0.1:11434`
3. 选择本地模式
4. 选中 `gemma4-e4b`

就这么简单。

### 方案 B：你已经在用 OpenClaw

如果你本来就有 OpenClaw 环境，直接看模型列表：

```bash
openclaw models list
```

如果你能看到：

```text
ollama/gemma4-e4b
```

那就直接切换：

```bash
openclaw models set ollama/gemma4-e4b
```

### 方案 C：你本来就是通过 Ollama 启动 OpenClaw

那就直接：

```bash
ollama launch openclaw --config
```

让它去更新配置。

---

## 这里有一个坑，很多人会踩

**不要把 Ollama 的地址写成 `/v1`。**

正确的是：

```text
http://127.0.0.1:11434
```

不是：

```text
http://127.0.0.1:11434/v1
```

OpenClaw 的 Ollama provider 走的是原生 API，不是 OpenAI 兼容那套地址。

这个坑很烦，因为一旦配错，你会出现一种特别恶心的情况：

- 模型看起来像是连上了
- 但工具调用不稳定
- 有时还会把 tool JSON 当普通文本吐出来

所以这一步别手滑。

---

## 这条链路跑通之后，Gemma 4 最适合干什么

我不打算在这里吹“全能模型”。

一旦进入真实工作流，本地模型到底值不值钱，不看 benchmark，看你能不能天天用。

就我看，这条链路下，Gemma 4 最适合先做三类事。

### 1. 写代码和改代码

比如：

- 写 shell 脚本
- 改 Python 小工具
- 解释一段代码
- 帮你做一轮基础 review

Gemma 4 官方模型卡里明确把 coding 和 function calling 列成核心能力，这和 OpenClaw 的使用场景是对得上的。

### 2. 吃长文档

比如：

- 读 PRD
- 总结长会议纪要
- 从规格文档里找细节
- 把分散材料整理成一版草稿

这类任务云模型当然也能干，但本地跑有个决定性的优势：**你不用把东西传出去。**

### 3. 处理你不想出云的数据

这是我最看重的一点。

很多人装本地模型，是为了省钱。  
我不是。

我更在意的是：

- 内部文档
- 私有代码
- 还没发出去的方案
- 不适合上传的笔记

这些东西，一旦能在本地让 OpenClaw 帮你处理，价值就出来了。

---

## 但有些话我得提前说清楚

### 1. 官方支持多模态，不等于你现在这条链路就已经全开

Gemma 4 官方模型卡明确写了：

- E4B 支持文本、图像、音频
- Transformers 示例里也覆盖了图像、视频、音频

但你现在走的是：

```text
GGUF -> Ollama -> OpenClaw
```

这篇文章优先保证的是：

**文本 + Agent 工作流先跑通。**

如果你下一步要继续玩图像或音频，需要单独验证三件事：

- 你下载的 GGUF 是否保留了对应能力
- 你当前 Ollama 版本是否完整支持
- OpenClaw 这一层是否已经把输入正确透传

别把“官方模型支持多模态”和“你本地已经完整支持多模态”混成一件事。

不是。

### 2. 128K 不是起点，64K 才是

我知道很多人看到 `128K context` 会兴奋。

但本地部署最怕的一种思路，就是参数先拉满，结果第一天就把自己劝退。

我的建议非常简单：

- 第一天先开 `64K`
- 跑顺了再往上试
- 别一开始就拿机器硬扛

### 3. 新模型发布的前 48 小时，不要迷信任何“最全教程”

包括这篇。

我能保证的是，这篇文章里的链路和命令是按当前官方文档校过的。  
但新模型刚发的时候，社区仓库、Ollama 支持情况、周边工具兼容，都会快速变化。

所以你要养成一个习惯：

**教程可以看，但最后一定回到官方页面确认。**

---

## 常见问题

### Q1：OpenClaw 看不到我刚导入的模型

先跑这三个命令：

```bash
ollama list
openclaw models list
echo $OLLAMA_API_KEY
```

如果 `ollama list` 有，`openclaw models list` 没有，通常就是：

- `OLLAMA_API_KEY` 没设
- Ollama 服务没起来
- 你还没重新 onboard

这种时候别怀疑模型，先怀疑接入层。

### Q2：`ollama create` 很慢，是不是卡住了

不一定。

常见原因有两个：

- 文件没下完整
- 第一次导入时在做缓存写入

先看 GGUF 文件大小是不是正常，再重跑一次：

```bash
ollama create gemma4-e4b -f Modelfile
```

### Q3：为什么不直接从 `google/gemma-4-E4B-it` 下官方权重

当然可以。

但那样你后面还要处理转换或兼容链路。  
如果你的目标不是“研究权重格式”，而是“今天就把它装进 OpenClaw 里用起来”，GGUF 明显更省事。

### Q4：一定要用 Unsloth 这份量化吗

不是一定。

但这篇文章现在统一用 `unsloth/gemma-4-E4B-it-GGUF`，原因很明确：

- 你正在实际测试这份
- 我们已经知道前一份社区量化有坑
- 现在最重要的是先拿一份更容易排查的版本，把路径跑通

就这么简单。

---

## 最后

Gemma 4 这种新模型，最容易让人掉进去的坑，不是装不上。

而是你花了一晚上研究，最后它还是没有进入你的工作流。

真正有价值的不是“我知道 Gemma 4 发布了”。

真正有价值的是：

- 它已经在你机器上
- 它已经能被 OpenClaw 调到
- 它已经开始替你处理真实任务

这才叫本地部署。

如果你今天只做一件事，我建议就是把这条链路跑通：

```text
hf download
→ ollama create
→ openclaw onboard
```

先跑通。别犹豫。

你会发现，很多关于本地 AI 的讨论，一旦进入“真的能用”这个阶段，噪音会立刻少一大半。

---

## 参考资料

- [Gemma 4 官方模型卡](https://huggingface.co/google/gemma-4-E4B-it)
- [Gemma 4 E4B 的 GGUF 量化列表](https://huggingface.co/models?other=base_model%3Aquantized%3Agoogle%2Fgemma-4-E4B-it)
- [Unsloth 的 Gemma 4 E4B GGUF 仓库](https://huggingface.co/unsloth/gemma-4-E4B-it-GGUF)
- [Hugging Face CLI 文档](https://huggingface.co/docs/huggingface_hub/guides/cli)
- [Ollama 导入模型文档](https://docs.ollama.com/import)
- [OpenClaw Model Providers 文档](https://docs.openclaw.ai/concepts/model-providers)
- [OpenClaw Models CLI 文档](https://docs.openclaw.ai/cli/models)
