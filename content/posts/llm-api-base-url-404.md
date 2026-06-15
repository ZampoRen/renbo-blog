---
cover: /images/llm-api-base-url-404/cover.png
title: "为什么你只改了 base_url，LLM API 还是报 404？"
description: "OpenAI-compatible 不是换个 base_url 就完事。真正决定请求、响应和流式解析的，是 endpoint path 背后的协议契约。"
date: 2026-06-15T17:00:00+08:00
draft: false
tags: ["LLM API", "OpenAI", "Anthropic", "Go", "后端工程"]
categories: ["技术实战"]
toc: true
---

你在 Go 项目里接一个新的模型供应商。

文档写着“OpenAI compatible”。你心里一松：那不就是改三个值吗？

```go
BaseURL: "https://api.xxx.com/v1/chat/completions"
APIKey:  "sk-..."
Model:   "xxx-model"
```

结果一跑，404。

你把 key 换了，把 model 换了，把代理关了，把日志打开，最后才发现真正发出去的 URL 长这样：

```text
https://api.xxx.com/v1/chat/completions/chat/completions
```

SDK 已经会在 base URL 后面拼 `/chat/completions`，你又把 endpoint path 塞进了 base URL。

这类问题看起来很低级。

但它背后有一个更容易被忽略的误判：很多人把“平台入口”和“协议动作”混成了一个东西。

base URL 只是告诉客户端：请求要发到哪台服务、哪个 API 根路径。

真正决定请求体怎么写、响应体怎么读、流式事件怎么解析的，是后面的 endpoint path。

`/v1/chat/completions`、`/v1/messages`、`/v1/responses` 不是三个 URL 后缀。

它们是三套协议契约。

## 404 只是第一层症状

很多 LLM API 的接入问题，第一眼都像配置问题。

404，像是 URL 配错。

400，像是参数传错。

流式读不到数据，像是网络断了。

响应字段找不到，像是 SDK 版本不对。

这些判断不一定错，但经常只停在表层。

你真正要问的是：我现在调用的到底是哪一种协议？

同样叫“发一段 prompt，让模型回一句话”，落到 API 上，至少有三种常见形态。

OpenAI Chat Completions 是 `messages → choices`。

Anthropic Messages 是 `messages → content blocks`。

OpenAI Responses 是 `input → output items`。

名字很像，代码差得很远。

看 Go DTO 会更直观。

Chat Completions 的输入核心，是 role + content 的 messages 数组：

```go
type chatCompletionsRequest struct {
    Model       string        `json:"model"`
    Messages    []chatMessage `json:"messages"`
    Temperature float64       `json:"temperature,omitempty"`
}

type chatMessage struct {
    Role    string `json:"role"`
    Content string `json:"content"`
}
```

响应也很直白。文本通常从这里拿：

```go
return out.Choices[0].Message.Content, nil
```

所以很多早期封装都会把内部统一结构做成这样：给一组 messages，拿第一个 choice 的 message content。

这在 Chat Completions 里很自然。

但你不能把这个心智直接搬到 `/v1/messages`。

## Messages API 不是换了一个 path

Anthropic Messages 也有 messages，但它不是同一套对象模型。

它的 content 更像一个 typed block 数组：

```go
type anthropicMessage struct {
    Role    string                  `json:"role"`
    Content []anthropicContentBlock `json:"content"`
}

type anthropicContentBlock struct {
    Type string `json:"type"`
    Text string `json:"text,omitempty"`
}
```

它还有一个很容易在迁移时漏掉的字段：`max_tokens`。

在 Go DTO 里，这个字段不是装饰品：

```go
type anthropicMessagesRequest struct {
    Model     string             `json:"model"`
    MaxTokens int                `json:"max_tokens"`
    System    string             `json:"system,omitempty"`
    Messages  []anthropicMessage `json:"messages"`
}
```

注意这里的 `System`。

在 Chat Completions 里，你习惯把 system 当成 messages 里的一个 role。

在 Messages API 里，system 通常是顶层字段。

返回值也不是 `choices[0].message.content`。

你要遍历 content blocks，筛选 `type == "text"`：

```go
var b strings.Builder
for _, block := range out.Content {
    if block.Type == "text" {
        b.WriteString(block.Text)
    }
}
```

这不是“字段名不一样”这么简单。

字段名背后是数据结构。

数据结构背后，是供应商对“一次模型交互”的理解。

Chat Completions 的世界里，模型像是在聊天记录后面补一条候选回复。

Messages API 的世界里，一条消息可以由多个 block 组成：文本、图片、工具调用、thinking，都可以成为不同类型的内容块。

如果你的业务代码只认识一个 string content，它迟早会在多模态、工具调用或流式解析上撞墙。

## Responses 更不是 Chat Completions 的新名字

Responses API 更容易让人误判。

因为它也是 OpenAI 的接口，很多人下意识以为：这只是 Chat Completions 的新版本。

更准确的理解是，它把“一次返回”建模成一个 response 状态机。

请求不再只是 messages：

```go
type responsesRequest struct {
    Model        string               `json:"model"`
    Instructions string               `json:"instructions,omitempty"`
    Input        []responsesInputItem `json:"input"`
}
```

输出也不是 choices：

```go
type responsesResponse struct {
    ID     string                `json:"id"`
    Object string                `json:"object"`
    Output []responsesOutputItem `json:"output"`
}
```

`output` 里面可能有 message，也可能有 reasoning、function_call 或其他 item。

所以提取文本时，你不能假设 `output[0].content[0].text` 一定存在。

更稳的写法是先筛 item 类型，再筛 content 类型：

```go
for _, item := range out.Output {
    if item.Type != "message" {
        continue
    }
    for _, content := range item.Content {
        if content.Type == "output_text" {
            b.WriteString(content.Text)
        }
    }
}
```

这段代码看起来啰嗦，但它表达了一个重要判断：Responses 的 output 不是“那句回答”，而是“一次交互里发生过的事情”。

有些事情是推理。

有些事情是消息。

有些事情是工具调用。

你只想拿最终文本，就必须明确筛选，而不是赌第一个元素刚好是你要的东西。

很多线上 bug 就是从这种赌开始的。

测试环境里模型只回一段文本，`output[0]` 永远可用。上线之后一开 reasoning，一接 tool call，一切都变了。

## “OpenAI-compatible”不是免检标签

现在很多平台都会写自己兼容 OpenAI。

这句话有用，但不能当成免检标签。

兼容不是布尔值。

更合理的拆法是五层：

| 层级 | 你要验证什么 | 最小验证方式 |
|---|---|---|
| L1 路径兼容 | `/chat/completions` 能不能返回文本 | hello world + 401/404 错误体 |
| L2 参数兼容 | `temperature`、`max_tokens` 等是否生效 | golden request / response |
| L3 流式兼容 | SSE chunk 是否符合你的 parser | stream parser 单测 |
| L4 工具兼容 | tools、tool_choice、tool_calls 能否往返 | tool call loop 集成测试 |
| L5 结构化/多模态 | JSON schema、vision、reasoning 是否保真 | capability matrix |

很多“兼容”只到 L1 或 L2。

你拿它跑普通文本没问题，一上工具调用就开始露馅。

这不是平台一定不靠谱，而是你把“能返回一句话”理解成了“全协议兼容”。

Groq 官方文档就写得很直接：它 mostly compatible，但有些 OpenAI 字段不支持，比如 `logprobs`、`logit_bias`、`top_logprobs`、`messages[].name`，`n` 如果传入也必须等于 1。

DashScope 的 OpenAI 兼容也有边界。比如 `response_format` 支持范围、`tool_choice`、`parallel_tool_calls` 这类字段，并不能默认按 OpenAI 原样理解。

OpenRouter 又是另一种情况。它会在多模型、多 provider 之间做 schema normalization，请求可能被转换、补齐或降级。它解决的是“统一入口”的问题，不是帮你抹平所有模型能力差异。

所以接第三方 provider 时，最危险的不是它报错。

最危险的是它不报错。

参数被静默忽略，工具调用被降级，结构化输出变成普通 JSON 文本。日志里一片正常，业务上已经开始漂。

## 流式解析最容易暴露协议混用

很多人以为流式就是 SSE。

这句话只说对了一半。

SSE 只是外壳，真正决定你怎么读数据的，是 event 类型和 payload 结构。

Chat Completions 里，你习惯读：

```text
choices[].delta.content
```

Anthropic Messages 的流式事件不是这个形状。它有 `message_start`、`content_block_start`、`content_block_delta`、`content_block_stop`、`message_delta`、`message_stop` 这样的事件流。

文本增量在 `content_block_delta` 里。

Responses API 又是另一套事件，比如 `response.output_text.delta`。

如果你的 stream parser 只写成“遇到 data 就按 choices delta 解”，它只能处理一种协议。

换 base URL 没用。

换 SDK 也不一定有用。

因为错的不是连接方式，是你把三套事件模型塞进了一个 parser。

这里最应该做的，不是写一堆 if else 去猜。

而是承认它们本来就是不同协议：每个协议有自己的 DTO、自己的 extractor、自己的 stream parser。外层再聚合到你的业务结构。

业务层可以统一。

协议层不要假装统一。

## 后端项目里该怎么落地

如果你正在写一个 Go 后端，要接多个 LLM provider，我建议把“兼容层”拆得笨一点。

笨，反而稳。

第一，base URL 只保存根地址，不保存 endpoint path。

```go
BaseURL: "https://api.xxx.com/v1"
```

endpoint path 由具体 client 方法决定：

```go
postJSON(ctx, strings.TrimRight(baseURL, "/")+"/chat/completions", reqBody, &out)
```

不要让配置同时承担“平台入口”和“协议动作”两个职责。

第二，每个 endpoint 独立建 DTO。

不要为了省事写一个巨大的 `UniversalMessage`，里面塞一堆可选字段，然后到处判断空值。

那种统一只是看起来优雅。

真正调试时，你会发现所有问题都变成：“这个字段到底是这个协议没有，还是这次响应没给，还是被兼容层吞了？”

第三，每个协议独立写 extractor。

Chat Completions 就读 choices。

Messages 就遍历 content blocks。

Responses 就遍历 output items。

业务层最后只收一个你自己的结构，比如：

```go
type ModelText struct {
    Text   string
    Usage  Usage
    RawID  string
    Source string
}
```

这一步很关键。

很多团队说要“统一多模型接口”，最后统一到一半，统一成了一个巨大的请求结构：这里有 `messages`，那里有 `input`，再塞一个 `system`，再塞一个 `instructions`，工具调用字段也往里面挂。

代码表面上只有一个入口。

维护成本全藏到了空字段和隐式约定里。

新人接手时最痛苦的不是字段多，而是不知道哪个字段在哪个 provider 下真的有效。一个参数传进去，没有报错，也没有生效。你查日志，只能看到请求成功；你查业务结果，只觉得模型“不听话”。

这时候所谓统一接口，反而变成了遮羞布。

更稳的做法是把统一放到业务语义上，而不是协议细节上。

比如你的业务只需要“给定用户输入，返回一段可展示文本”，那业务层拿 `ModelText` 就够了。至于底下是 Chat Completions 的 choice，Messages 的 text block，还是 Responses 的 output_text，交给各自 extractor。

如果你的业务需要工具调用，也不要急着把所有 provider 的工具结构揉成一个万能 JSON。先定义自己的业务事件：模型请求调用哪个工具、参数是什么、调用结果如何回填。再分别适配每个协议的工具调用格式。

统一的目标不是让所有协议看起来一样。

统一的目标是让业务代码不用知道协议差异，同时让协议差异在适配层里清清楚楚地存在。

第四，给兼容层写能力矩阵。

不要只写：

```yaml
provider: openrouter
compatible: true
```

应该写得更像这样：

```yaml
provider: groq
chat_completions: true
stream: true
tools: partial
json_schema: unknown
unsupported:
  - logprobs
  - logit_bias
  - messages.name
```

第五，测试不要只测 hello world。

至少测四类：普通文本、错误体、流式文本、工具调用。

如果你的产品依赖结构化输出，再单独测 JSON schema。

这套测试不一定复杂，但一定要独立于真实业务。

你要知道是 provider 兼容层的问题，还是你自己的业务代码问题。

一个简单但很有效的做法，是给每个 provider 留一组 golden case。

比如：

- 普通文本：固定 prompt，断言能拿到非空文本；
- 错误请求：故意传错 model 或 key，断言错误体能被正确解析；
- 流式文本：断言 parser 能拼出完整文本，而不是只读到第一块；
- 工具调用：断言 tool call 的 name、arguments、call id 能完整往返；
- 结构化输出：如果依赖 JSON schema，断言失败时能暴露具体原因。

这组测试不用追求覆盖所有模型。

它的价值是给你一条边界：今天这个 provider 到底兼容到哪一层。

以后再换模型、换路由、换 SDK 版本，你不是靠感觉判断“应该没问题”，而是跑一遍 case，看哪一层开始裂。

这里还有一个团队协作上的好处。

当你把兼容性写成测试和能力矩阵，代码评审就不再变成“我觉得这个 provider 应该支持”。评审的人可以直接问：工具调用测了吗？流式事件测了吗？结构化输出是平台保证，还是我们自己解析出来的？

这个问题一问，很多模糊争论会立刻停下来。

技术选型最怕的不是能力不够，而是边界不清。能力不够可以换方案，边界不清会让每个人在不同地方补丁式修一遍。

## 一个更实用的排查顺序

下次 LLM API 接入报 404、400 或流式读不到数据，不要一上来换 key、换模型、换网络。

按这个顺序查：

1. 打印最终 URL。确认 base URL 里没有重复 endpoint path。
2. 确认当前调用的是哪套协议：Chat Completions、Messages，还是 Responses。
3. 对照协议检查请求 DTO：system 放哪、max_tokens 是否必填、input/messages 形状是否正确。
4. 对照协议检查响应 extractor：choices、content blocks、output items 不要混用。
5. 如果是 stream，先看 event type，不要只看是不是 SSE。
6. 如果是第三方兼容层，按 L1 到 L5 做能力验证，不要相信一个“compatible”就收工。

这几步做完，大部分问题会变得很普通。

普通不是坏事。

普通的问题能复现，能写测试，能交给下一个同事维护。

真正折磨人的，是你一直把协议问题当配置问题查。

## 该带走的判断

LLM API 的接入，最容易误导人的地方，是它们长得太像 HTTP 接口。

一个 base URL，一个 API key，一个 model，几个 JSON 字段，看起来像任何后端都写过的普通外部服务调用。

但模型 API 的复杂度不在 HTTP 这一层。

它在“交互对象”这一层。

你调用的是聊天补全、消息块，还是一次 response 状态机？

你要的是普通文本、结构化输出，还是带工具调用的多阶段过程？

这些问题不先想清楚，base URL 改得再漂亮，也只是把请求发到了错误的契约上。

别把 `OpenAI-compatible` 当成一句承诺。

把它当成一张待验证的清单。

<!--
图片需求清单：
1. 正文解释图：base URL 与 endpoint path 拼接关系。建议展示正确 URL `https://api.xxx.com/v1` + `/chat/completions`，以及错误拼接 `/v1/chat/completions/chat/completions`。
2. 正文解释图：三种协议对象模型对比。Chat Completions：messages -> choices；Anthropic Messages：messages -> content blocks；Responses：input -> output items。
3. 正文解释图：OpenAI-compatible L1-L5 能力矩阵，突出“兼容不是布尔值”。
封面图：本任务明确不生成，交由 default 发布门处理。

事实核验说明：
- Go DTO 与 extractor 来自上游 `ai_api_protocols_go_mock.go`，已实际执行 `go run` 通过。
- Anthropic streaming event flow 与 `content_block_delta` 对照 Claude 官方文档。
- OpenAI Responses streaming event `response.output_text.delta` 对照 OpenAI 官方文档/Agents SDK 文档。
- Groq unsupported OpenAI fields 对照 Groq OpenAI Compatibility 官方文档。
- OpenRouter schema normalization 对照 OpenRouter API reference。
- DashScope 兼容边界按 researcher handoff 使用，发布前建议 default 如需强引用具体字段，再回到阿里云官方文档页面人工复核一次。
-->
