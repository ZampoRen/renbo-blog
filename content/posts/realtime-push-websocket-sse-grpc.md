---
title: "WebSocket vs SSE vs gRPC Stream：实时推送三选一的工程判断"
description: "实时推送选型没有银弹。这篇文章不讲协议细节，从工程体感、翻车经验到决策框架，帮你判断什么时候选 WebSocket，什么时候用 SSE，什么时候该上 gRPC Stream。"
date: 2026-06-27
draft: false
author: "任博"
tags:
  - WebSocket
  - SSE
  - gRPC
  - 实时推送
  - 技术选型
  - Go
categories: ["技术实战"]
cover: "/images/realtime-push-websocket-sse-grpc/cover.png"
toc: true
---

## 引子：实时推送是个伪命题

WebSocket、SSE、gRPC Stream 都能做到"实时"，但这三个词在工程上的体感天差地别。你的老板可能觉得"不就是把数据推给用户吗，哪个有现成的就上哪个"，做过的人才知道选错了是多大的坑。

我见过太多团队选型的理由是这样的："实时推送嘛，那肯定是 WebSocket"、"gRPC 最先进，用新的准没错"、"SSE 太弱了，不支持双向"。这些判断单独拎出来都不算错，但放在具体场景里，可能让你多花几倍的运维成本。更扎心的是——很多人刚上线的前两个月觉得选什么都一样，等业务量上来、用户变多、问题开始冒，才意识到当初的选择有多贵。

选错了不是不能用。WebSocket 做单向推送也能跑，gRPC 做浏览器推送也能搭 gRPC-Web 代理。但运维成本、调试体验、扩展潜力的差距会在业务增长时集中爆发——nginx 超时断开你没设置心跳、WebSocket 连接泄漏导致 goroutine 堆积、浏览器 SSE 连接数打满用户看到空白页面，每一个都是线上事故级别的翻车点。

今天不聊协议细节（这些协议的 RFC 和官方文档写得比我清楚），我只讲选型时应**真实感受到**的几个维度。看完你至少知道：为什么我建议你"默认选 SSE，不够了再上 WebSocket，微服务间用 gRPC Stream"。

---

## 第一块：三张牌各自的长相

### WebSocket：全双工是你的超能力，也是你的包袱

WebSocket 的三板斧：**客户端和服务端随时能发**、**二进制和文本都支持**、**帧开销极低**（数据帧头部只有 2-6 字节）。这些特性让它成为聊天、协同编辑、游戏、金融行情等场景的不二之选。

但它最大的问题是——**WebSocket 只负责传帧，不关心帧里面是什么**。这既是力量也是负担。你需要自己定义消息格式（JSON？Protobuf？MessagePack？）、自己实现心跳（否则 nginx 默认 60s 断开）、自己写重连逻辑、自己管连接池和 hub 模式。而当你花了一周把这些基础设施搭好，回头看需求，发现其实只需要服务端定时推送一下——SSE 两小时就搞定了。

Go 生态里两个主流库的取舍：

- **gorilla/websocket**——存量最大、文档最多，社区有大量现成示例和教程。但原仓库 2022 年底归档了，虽然有社区 fork 在维护，但总归不是官方推荐。最大的坑：**不支持并发写**——两个 goroutine 同时调用 `WriteMessage` 直接 panic。你需要用 hub goroutine 串行化所有写操作，这会增加不少样板代码。

- **coder/websocket**（原 nhooyr/websocket）——新项目首选。API 带 `context.Context`，**支持并发写**，零外部依赖，还支持 WASM 编译和 permessage-deflate 压缩。Go 官方推荐了这个库。如果你现在开新项目，没有历史包袱，别纠结，直接用它。

```go
// WebSocket 服务端 - coder/websocket
func wsHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := websocket.Accept(w, r, nil)
    if err != nil { return }
    defer conn.Close(websocket.StatusNormalClosure, "bye")

    ctx, cancel := context.WithTimeout(r.Context(), time.Hour)
    defer cancel()

    go func() {
        for {
            _, msg, err := conn.Read(ctx)
            if err != nil { break }
            log.Printf("recv: %s", msg)
        }
    }()
    // 写循环 + 心跳
}
```

### SSE：看起来最弱，但恰恰因为弱所以简单

SSE 可能是工程界被低估最严重的协议之一。

它的核心原理简单到令人发指：一个持久的 HTTP 响应，服务端设好 `Content-Type: text/event-stream`，然后不断 write + flush 就行。**不需要引入任何第三方库**，Go 标准库 20 行写完。浏览器端 `new EventSource(url)` 就连上了。

最被低估的亮点是**自动重连**。EventSource 断了它会自己重连，带指数退避，还支持 `Last-Event-ID` 断点续传。你自己写过 WebSocket 重连就知道，这个功能看着简单，实现起来要考虑的事情太多：退避策略、重连次数限制、消息丢失恢复、避免重复订阅导致消息风暴。而 SSE 这些全是浏览器给你的，一行额外代码都不用写。

调试便利性更是降维打击。SSE 是纯文本 HTTP 流，一个 `curl -N http://localhost:8080/events` 就能看到所有实时推送的数据。WebSocket 和 gRPC 都需要专用工具或插件才能调试。在排查线上问题时，这种差异可能决定你花 5 分钟还是 5 小时。

正因为 SSE 够简单，它在 LLM 应用领域意外成为了事实标准。OpenAI、Anthropic、Google 的 LLM API 全部用 SSE 做 token 流式输出——不是因为 SSE 最先进，而是因为它最简单的方案往往就是正确的方案。

```go
// SSE 服务端 - 标准库，零依赖
func sseHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    flusher, ok := w.(http.Flusher)
    if !ok { return }

    for {
        select {
        case <-r.Context().Done():
            return
        case evt := <-eventCh:
            fmt.Fprintf(w, "event: %s\ndata: %s\n\n", evt.Type, evt.Data)
            flusher.Flush()
        }
    }
}
```

### gRPC Stream：强类型正规军，代价是一套新生态

gRPC Stream 的卖点很清晰：HTTP/2 多路复用 + Protobuf 强类型打包在一起。你在 `.proto` 文件里定义好接口，代码自动生成，双向流、服务端流、客户端流随便选，天然带流控和背压。

但代价也不小：你要接受 protoc 工具链、改构建流程、学会 gRPC 拦截器、健康检查、负载均衡这套生态。调试不能用 curl，得装 `grpcurl` 或者 `grpcui`。生产出现问题时，二进制协议比文本协议难排查得多。

最核心的限制：**浏览器不是 gRPC 的一等公民**。截至 2026 年，gRPC-Web 仍然只支持 unary 和 server-streaming，client-streaming 和 bidirectional streaming 全不支持。如果你要用 gRPC 双向流做浏览器聊天，写代码之前就死了这条心——它做不到，你需要 gRPC-Web 代理层（如 Envoy），而且代理了也只是单向。gRPC 团队已经明确表示将转向 WebTransport 来解决浏览器端流式通信的问题，短期内没有计划支持 gRPC over WebSocket。

所以我的判断很直接：gRPC Stream 是微服务层面的技术，不是浏览器端的技术。如果你的客户端是服务端或 native 客户端，gRPC Stream 很强；如果你的客户端是浏览器，除非你已经有一套 gRPC-Web 代理基础设施了，否则别硬上。

```protobuf
// .proto 定义
service EventService {
  rpc Subscribe(SubscribeRequest) returns (stream Event);
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

```go
// gRPC Server-Streaming 服务端
func (s *EventServer) Subscribe(
    req *pb.SubscribeRequest,
    stream pb.EventService_SubscribeServer,
) error {
    for {
        select {
        case <-stream.Context().Done():
            return nil
        case evt := <-s.broadcast:
            if err := stream.Send(evt); err != nil {
                return err
            }
        }
    }
}
```

---

## 第二块：对比表 + 分界线判断

先把三者的关键维度放一张表里：

| 维度 | WebSocket | SSE | gRPC Stream |
|------|-----------|-----|-------------|
| 通信方向 | 双向 | 单向（服务端→客户端） | 四种模式（含双向） |
| 传输协议 | WebSocket 帧（基于 TCP） | HTTP 长连接 | HTTP/2 流 |
| 消息格式 | 自定义（文本/二进制） | UTF-8 纯文本 | Protobuf 强类型二进制 |
| 浏览器支持 | 原生 99%+ | 原生 97%（IE 不支持） | 需 gRPC-Web + Envoy 代理 |
| 自动重连 | 自己实现 | EventSource 内置 + 断点续传 | 自己实现 |
| 心跳/保活 | 自己实现 ping/pong | 应用层心跳 | HTTP/2 keepalive 内置 |
| 连接管理 | 每条连接一个逻辑流 | 每条连接一个事件流 | 多路复用（同连接多流） |
| 背压/流控 | 无内置 | 浏览器隐式节流 | HTTP/2 流控内置 |
| Go 生态 | 成熟但主库已归档 | 标准库即可，零依赖 | 官方 gRPC 非常成熟 |
| 调试便利性 | 一般（Wireshark/插件） | **高**（curl 直接看） | 一般（需 grpcurl） |
| 代理兼容性 | 需显式配置 Upgrade | 走标准 HTTP 很友好 | 需 HTTP/2，老代理不兼容 |
| 帧开销 | 2-6 字节 | 每次 HTTP flush 有开销 | Protobuf 编码小 |

### 分界线判断

对比表是信息，分界线才是判断。以下是我的五个核心分界线：

**第一条分界线：需要客户端主动发数据吗？**
这是最根本的分水岭。需要双向 → WebSocket 或 gRPC；不需要 → SSE。注意：我见过太多人为了"未来可能需要双向"而在一开始就选 WebSocket——结果两三年过去了，双向功能从来没被提上需求。更务实的做法是先用 SSE 快速交付，当真正的双向需求到来时再引入 WebSocket。

**第二条分界线：客户端是浏览器吗？**
如果是 → 排除 gRPC（除非你只做 server-streaming 且愿意搭 gRPC-Web 代理）；如果不是 → gRPC Stream 是最优解。这条分界线源自 gRPC-Web 的双向流限制——这个限制在 2026 年仍然存在，短期内看不到改变的可能。

**第三条分界线：团队已经在用 gRPC 了吗？**
如果是 → gRPC Stream 是合理扩展，一致性优于多样性；如果不是 → 没有理由为了"推送"这个单一需求引入一整套 Protobuf 工具链。

**第四条分界线：消息频率和二进制需求。**
没到毫秒级也不需要传二进制 → SSE 够用；高频小消息（金融行情、实时竞价）或者需要传二进制 → WebSocket 或 gRPC Stream。

**第五条分界线：运维力量。**
小团队或者一个人既要写业务又要管运维 → SSE 是最好的选择。SSE 几乎所有运维方面都是标准 HTTP，你现有的限流、认证、CORS、TLS 终结、CDN 全部可以直接用。WebSocket 和 gRPC 都有各自的运维门槛。

---

## 第三块：每个协议什么时候选、什么时候不该选

### WebSocket

**✅ 适合你的时候：**
- 需要客户端主动发数据——聊天消息、协同编辑操作、游戏操作、游标位置同步。这是 WebSocket 的核心护城河，SSE 无法替代。
- 消息频率极高（毫秒级）——WebSocket 帧头只有 2-6 字节，适合高频小消息。金融行情、实时竞价这类场景非它不可。
- 需要传输二进制数据——图片流、音频片段、自定义二进制协议。SSE 只能传 UTF-8 文本，WebSocket 原生支持二进制帧。
- 客户端无法保证 HTTP/2 环境——老旧反向代理、IoT 设备、特殊企业网络。WebSocket 基于 TCP Upgrade，不依赖 HTTP/2。
- 团队已经有 WebSocket 基础设施——对心跳、hub 模式、连接池了如指掌。

**❌ 不该选的时候：**
- 只需要服务端推送——用 SSE 能省 80% 的代码量和维护量。我亲眼见过一个团队用了 WebSocket 做日志推送，6 个月后因为 goroutine 泄漏上了三次事故复盘，切到 SSE 后问题全部消失。
- 维护力量有限——心跳、重连、消息协议格式、连接池、session 管理，全都得自己写。
- 微服务间通信——gRPC Stream 自带强类型契约和流控，比你用 WebSocket 自己包装的版本可靠得多。

### SSE

**✅ 适合你的时候：**
- 只需要服务端推送——通知、状态更新、日志流、监控面板、实时指标。其实 80% 的"实时推送"需求就是单向推送，SSE 是默认答案。
- 想要自动重连——EventSource 内置 + Last-Event-ID 断点续传。你自己实现重连至少要考虑指数退避、最大重试次数、重连时的消息丢失恢复。这些 SSE 全给你包了。
- 想用最少的代码实现——标准库 20 行，零第三方依赖。Go 后端的 SSE handler 加一个 broadcast channel 就是完整实现。
- LLM Token 流式输出——2026 年这个场景的绝对主流。OpenAI、Anthropic、Google 都在用 SSE。
- 调试优先——"curl -N 就能看实时推送"这种体验，只有 SSE 能给。

**❌ 不该选的时候：**
- 需要双向通信——别挣扎。SSE 就是单向的，虽然可以用 HTTP POST 模拟发送，但那本质上是两条独立的连接，不是真正的全双工。
- HTTP/1.1 环境下的浏览器连接数——每个域名最多 6 个并发连接，用户多开标签页就打满了。虽然 HTTP/2 不存在这个限制，但你的用户网络环境不归你控制。
- 需要传输二进制数据——Base64 编码不算解决方案，那只是把问题转嫁了。

### gRPC Stream

**✅ 适合你的时候：**
- 微服务间通信——服务间高性能、低延迟、强类型的流式数据交换。这是 gRPC Stream 最擅长的领域。
- 需要强类型契约——.proto 文件作为接口合同，编译期检查，跨语言自动生成。Go + Python + Rust 的多语言团队尤其实用。
- 已经用了 gRPC——已经在用 unary RPC，自然扩展到 streaming，比引入另一套方案成本更低。
- 需要内置流控和背压——HTTP/2 流控内建，慢消费者自动反压生产者，不丢消息不爆内存。
- 高吞吐场景——Protobuf 编码快、体积小，单连接多路复用节省资源。

**❌ 不该选的时候：**
- 客户端是浏览器——gRPC-Web 限制了双向流，代理层增加了运维复杂度。你的选择要么是 WebSocket 要么是 SSE。
- 团队没接触过 Protobuf——引入 protoc 工具链、修改构建流程、学习拦截器体系，这些成本高于直接使用 SSE 的代价。
- 简单的推送通知——打个日志都要编译 .proto 文件，太重了。

---

## 第四块：每个人的翻车经验

这些坑是我自己踩过或亲眼看到别人踩过的。写出来不是为了凑字数，是帮你省几个通宵的排查时间。

### WebSocket 翻车

**🔴 nginx 反向代理 60s 超时（最常见，没有之一）**
nginx `proxy_read_timeout` 默认值只有 **60 秒**。WebSocket 连接空闲超过 60 秒 → nginx 主动关闭连接 → 客户端收到 1006（abnormal closure）。**一个大坑是：TCP keepalive 不会重置这个超时**，因为 nginx 的超时是从读到最后一个字节开始算的，TCP 层面的保活数据不经过应用层。你必须传应用层数据才能重置这个超时。

解决方案：应用层每 25-30s 发一次 ping/pong 心跳 + nginx `proxy_read_timeout` 设为 3600s 或更大。

```nginx
location /ws/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;
    proxy_buffering off;
}
```

**🔴 goroutine 泄漏**
每个 WebSocket 连接启动一个读 goroutine 和一个写 goroutine。客户端断了但服务端没检测到（比如网络分区），goroutine 永远挂在那里。踩过的兄弟回来说："查了一圈发现是服务端 ReadMessage 卡住了，goroutine 从 1000 涨到 20000，GC 压力翻了几倍。"解决方案：读写超时 + 心跳超时断开 + `defer conn.Close()`。

**🔴 并发写 panic（gorilla/websocket 专属）**
gorilla/websocket 写入不是线程安全的。两个 goroutine 同时调用 `WriteMessage` → **panic**。很多人第一次上线就被这个问题炸了——为什么客户端多了就 crash？因为你每个请求过来的处理 goroutine 都在尝试写同一个 conn。要么用 coder/websocket（内置并发写安全），要么自己用 hub goroutine 串行化。

**🔴 负载均衡会话保持**
WebSocket 有状态。客户端连上节点 A → 几秒后 LB 健康检查把请求转发到节点 B → 节点 B 不认识这个连接 → 断开。要么 `ip_hash`（但 NAT 下所有用户 IP 相同，分布不均匀），要么 cookie sticky（Nginx Plus 功能），要么把 session 状态外移到 Redis。

### SSE 翻车

**🔴 nginx 默认缓冲**
这是 SSE 最常见的坑——没有之一。你写好了 SSE handler，本地 `curl -N` 跑得好好的，上线后发现事件几分钟才推送一次。罪魁祸首是 nginx 的 `proxy_buffering` 默认是 on，实时消息被 nginx 积压在缓冲区里，等缓冲区满了才一次性发给客户端。你的"实时"变成了"每 4KB 推送一次"。

```nginx
location /sse/ {
    proxy_pass http://backend;
    proxy_buffering off;
    proxy_cache off;
    chunked_transfer_encoding on;
    proxy_read_timeout 3600s;
}
```

还有一个很多人不知道的：响应头里加上 `X-Accel-Buffering: no` 也能关掉 nginx 的缓冲。

**🔴 HTTP/1.1 的 6 连接限制**
每个浏览器域名最多 **6 个并发 SSE 连接**。用户打开了 3 个标签页，每个标签页开了 SSE 流，第 4 个标签页就连不上了——新的 EventSource 连接一直处于 pending 状态。这是 Chrome/Firefox 固化的限制，标记为 Won't Fix。解决方案：切 HTTP/2（默认 100 并发流），或者用 SharedWorker 让多个标签页共享一个 SSE 连接，或者用 Service Worker 做事件分发。

**🔴 IE 不支持 + 企业防火墙拦截**
Internet Explorer 完全不支持 EventSource。部分企业防火墙会过滤 `text/event-stream` 的 Content-Type。如果你在有这些环境约束的系统中用 SSE，要么提供 polyfill 回退到 long-polling，要么同时在服务端支持 SSE 和 WebSocket，让客户端根据能力探测。

### gRPC Stream 翻车

**🔴 gRPC-Web 双向流限制（最致命）**
如果你设计了一个浏览器端的 gRPC 双向流方案，写完服务端发现浏览器端不能用——这个坑我见好几个人踩过。gRPC-Web 在 2026 年仍然不支持 client-streaming 和 bidirectional streaming。浏览器端 `duplex: 'full'` 在 Fetch API 的提案中，但没有任何稳定浏览器发布。解决方案就一个：浏览器端别用 gRPC 做双向通信，该用 WebSocket 时就用 WebSocket。

**🔴 Stream 没关闭导致内存泄漏**
Bidirectional streaming 中客户端半关闭后，服务端不做响应式关闭会怎样？stream 一直挂着，所有挂着的 stream 占用 HTTP/2 stream slot，达到 `MAX_CONCURRENT_STREAMS`（默认 100）后，新连接能建立但创建不了新流。服务看起来还活着，但实际已经不能处理新业务了。

**🔴 流控死锁——真实生产事故**
这是一个物流平台的血泪教训：他们用单条 gRPC bidirectional stream 接收 50,000 个 GPS 设备的实时位置。某天下游数据库写入降级 → 聚合器处理速度从 10,000 msg/s 降到 800 msg/s → HTTP/2 流控窗口几秒内被填满 → 50,000 个设备全部 stall，等 `WINDOW_UPDATE` 帧释放窗口。从监控上看聚合器服务 CPU 极低、无任何报错——看起来完全健康，实际已经死锁了两小时才被发现。gRPC 的背压是个好东西，但你没设计好消费者处理能力，它就会反过来咬你。

---

## 第五块：一个决策框架

我把上面所有的判断浓缩成一个决策树，拿走直接用：

```
需要客户端主动发数据吗？
  ├─ 是 → 客户端是浏览器吗？
  │        ├─ 是 → WebSocket（gRPC-Web 不支持双向流）
  │        └─ 否 → gRPC Stream（强类型 + 流控 + 多路复用）
  └─ 否 → 已经在用 gRPC 生态吗？
           ├─ 是 → gRPC Stream（一致性优于多样性）
           └─ 否 → SSE（默认选项，简单可靠）

再答一个问题：团队有时间维护一套 WebSocket 基础设施吗？
  ├─ 有 → WebSocket 足够灵活，可以涵盖更多未来需求
  └─ 没有 → SSE 或更简单的方案更务实
```

### checklist 速查

| 问题 | 是 → | 否 → |
|------|------|------|
| 需要客户端发数据给服务端？ | WebSocket / gRPC | SSE |
| 客户端是浏览器？ | WebSocket / SSE | gRPC Stream |
| 已在用 gRPC？ | gRPC Stream | WebSocket / SSE |
| 需要强类型契约？ | gRPC Stream | WebSocket / SSE |
| 需要自动重连？ | SSE | 自己实现（WebSocket / gRPC）|
| 需传输二进制数据？ | WebSocket / gRPC | SSE |
| 队伍运维能力有限？ | SSE | WebSocket / gRPC |
| LLM Token 流式输出？ | SSE | WebSocket（需双向时）|
| 微服务间实时管道？ | gRPC Stream | WebSocket |

### 核心原则（九字真言）

**默认选 SSE，不够了再上 WebSocket，微服务间用 gRPC Stream。**

这九个字比翻三小时文档都管用。具体来说：

1. **默认选 SSE**：如果你只做服务端推送，SSE 是最简单、最便宜、最易调试的方案。不要为了"未来可能需要双向"而一开始就用 WebSocket。
2. **SSE 不够了才考虑 WebSocket**：需要双向通信、二进制、高频消息或者浏览器必须用 WebSocket 时再升级。不是先上 WebSocket 再用它的子集。
3. **gRPC Stream 是微服务层的事，不是浏览器的事**：服务间通信选 gRPC Stream；浏览器端受 gRPC-Web 限制，别强求。
4. **混合架构最实际**：边缘用 WebSocket/SSE 对浏览器，内部用 gRPC Stream 做服务间管道。中间通过 API Gateway 做协议转换——这在大型系统中是最常见的架构。

---

## 结尾：选你需要的，不是选最先进的

没有完美的实时推送协议。每个协议都是取舍的结果，不是技术水平的证明。

- **WebSocket 火力全开**——能双向、能传二进制、帧开销低。但它的灵活需要你用代码和运维经验来买单。
- **SSE 经济适用**——简单、零依赖、自动重连、curl 就能调试。但它有单向的天花板，你不能踮着脚够双向需求。
- **gRPC 正规军**——强类型、流控、多路复用、微服务标配。但你得接受整套生态的沉重感，浏览器端还不是它的一等公民。

这三者的关系不是谁替代谁，是**你需要什么你就选什么**。做技术选型别跟风，把场景量化出来，一条条比对，答案自然就有了。

最后送三句话：

**实时推送的精度，取决于你对场景的理解深度。选对你需要的，不是选最先进的。SSE 够用就不要上 WebSocket，WebSocket 够用就不要上 gRPC——少即是多，在工程选型里永远成立。**
