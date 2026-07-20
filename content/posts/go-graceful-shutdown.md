---
title: "发一次版就冒一串 502，别急着怪 K8s 探针，先看你的关闭顺序"
description: "graceful shutdown 不是加个 signal.Notify 加个 server.Shutdown() 就完事。它是一整套关闭顺序：先停接新请求，再等存量请求做完，最后才关 DB、MQ、连接池。顺序反了，收到信号就关依赖，等于让还在飞的请求集体撞墙。"
date: 2026-07-21T14:20:00+08:00
draft: false
author: "Zampo"
tags: ["Go", "graceful shutdown", "net/http", "Kubernetes", "后端"]
categories: ["技术实战"]
cover: "/images/go-graceful-shutdown/cover.svg"
toc: true
---

发一次版，监控上就冒出一串 502。

不多，十几个，几秒钟就没了。下一次发版又来一串。你先怀疑的大概是 K8s：是不是 readiness 探针没配好？是不是 terminationGracePeriod 太短？是不是 LB 摘流慢了半拍？

这些方向都不算错，但很多时候真正的问题根本不在编排层。它在你 `main` 函数最后那几行——收到信号之后，你按什么顺序关掉了这个进程。

![Go 服务优雅关闭顺序封面](/images/go-graceful-shutdown/cover.svg)

## graceful shutdown 不是一个函数，是一个顺序

大多数人对优雅关闭的理解，停在这一步：

```go
c := make(chan os.Signal, 1)
signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
<-c
srv.Shutdown(context.Background())
```

加了 `signal.Notify`，加了 `srv.Shutdown()`，看起来该做的都做了。

但优雅关闭从来不是一个函数调用。它是一整套关闭的先后次序：先停止接收新请求，再等已经进来的请求做完，最后才去关它们依赖的东西——数据库、消息队列、连接池、下游客户端。

这三步的顺序不能乱。

`srv.Shutdown()` 只负责前两步：它先关掉监听端口，不再 accept 新连接，然后等已经在处理的请求跑完。它管不着你的 DB 连接、你的 Kafka producer、你后台那几个还在写数据的 goroutine。那些是你自己拉起来的，也得你自己按顺序收。

顺序错了，`Shutdown` 做得再对也白搭。

## 收到信号就 os.Exit、就关 DB，为什么它平时看着能跑

线上翻车的写法，几乎都长一个样：收到信号，立刻动手关东西。

```go
<-c
db.Close()        // 先把 DB 关了
srv.Shutdown(ctx) // 再来收 HTTP
os.Exit(0)
```

或者更干脆，`<-c` 之后直接 `os.Exit(0)`，连 `Shutdown` 都没有。

这种代码你本地测一百遍都不会出事。因为本地没有并发存量请求，你 `Ctrl+C` 的那一刻，手里根本没有正在写事务的请求。信号来了，进程走了，干干净净。

线上不一样。发版那一刻，这台实例上可能正有几十个请求在飞：有的在查库，有的事务写了一半，有的在等下游返回。你一个 `db.Close()` 下去，这些请求手里的连接瞬间被抽走。

查库的那个请求，得到一个 `sql: database is closed`。写事务的那个，事务没提交也没回滚，就断在半路。消费到一半的消息，offset 还没确认，进程就没了。

`os.Exit(0)` 更狠。它不给任何东西收尾的机会——defer 不执行，Shutdown 不等待，正在写的请求直接跟着进程一起消失。

平时能跑，只是因为平时没人在最要命的那一秒发版。它不是对的，它只是还没轮到你赔。

## 正确的顺序：入口先关，出口最后关

把关闭顺序想成一条水管。请求从入口进来，流过你的业务逻辑，最后从出口（DB、MQ）流出去。

关的时候要反着来：先关入口，最后关出口。

第一步，关入口。停止 accept 新连接、从负载均衡摘掉自己。这一步 `srv.Shutdown()` 帮你做了大半——它先关监听端口，新请求进不来，老连接继续。

第二步，等存量请求做完。这也是 `srv.Shutdown()` 的活：它会一直等到手里的连接都处理完、回到空闲，才返回。你要给它一个带超时的 context，别让它无限等下去：

```go
ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
    log.Printf("graceful shutdown 超时，进入强杀: %v", err)
}
```

第三步，请求都做完了，这时候才轮到关依赖：

```go
// srv.Shutdown 返回之后，才关下游
db.Close()
producer.Close()
redisPool.Close()
```

顺序就这么定下来了：入口先关，请求做完，出口最后关。

`db.Close()` 那行代码没变，位置变了。从信号来了立刻执行，挪到了 `srv.Shutdown()` 返回之后。就这一下，存量请求手里的连接就一直是活的，直到它自己跑完。

## 一个还在写事务的请求，是怎么被你亲手打断的

把镜头拉到一个具体的请求上。

用户点了下单，请求进来，你的 handler 开了个事务：扣库存、写订单、记流水，三条 SQL 要一起成功。写到第二条的时候，发版了，进程收到 SIGTERM。

如果你的代码是「收到信号立刻 `db.Close()`」，那么就在这个请求写第三条 SQL 之前，它脚下的连接被抽走了。第三条报错，事务既没 commit 也没 rollback——连接都没了，谁来 rollback？

对 MySQL 来说，这条连接断开时未提交的事务会被回滚，库存和订单不至于对不上。但你的 handler 拿到的是一个中途的错误，用户看到的是下单失败，而这次失败纯粹是发版制造的，跟业务没有半点关系。

换成消息消费更难看。一条消息消费到一半，业务处理完了但 offset 还没提交，进程没了。这条消息下次会被重新投递，如果你的处理不是幂等的，就是一次重复扣款、重复发货。

而只要顺序对了，这些都不会发生。`srv.Shutdown()` 会等这个下单请求把三条 SQL 写完、事务提交、响应返回，然后它才返回，然后你才 `db.Close()`。那条连接在整个过程里一直活着。

存量请求要的不多，就是让它把手里的活干完。别在它干到一半的时候抽它的桌子。

## 超时怎么设，强杀怎么兜底，K8s 那边要对上

光有顺序还不够，还有几个边界得钉死。

**关闭超时要有，但别和编排层打架。** `srv.Shutdown()` 的 context 超时，决定了你最多愿意等存量请求多久。设 15 秒是常见选择，但它必须小于 K8s 的 `terminationGracePeriodSeconds`（默认 30 秒）。如果你 Shutdown 等 30 秒，而 K8s 25 秒就 `SIGKILL`，那你精心写的等待逻辑会被强杀打断，等于没写。让 Go 这边的超时先到，把主动权攥在自己手里。

**强杀要兜底。** 万一有个请求卡死了，`Shutdown` 超时返回错误，你不能就这么挂着。超时之后该记日志记日志，该退出退出，让编排层拉新实例接管，别让一个卡死的请求拖着整个进程不走。

**K8s 那边有个隐蔽的时间差。** Pod 被删除时，「从 endpoints 摘掉」和「给容器发 SIGTERM」这两件事是并发发生的，不保证先后。也就是说，你的进程可能已经收到 SIGTERM 开始关闭了，LB 那边还在往你这儿转发新请求。这时候新请求打到一个正在关的端口上，就是 502。

解法是给 Pod 加一个 `preStop` 睡一会儿：

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
```

这 5 秒不做别的，就是等 LB 把你从转发列表里摘干净，之后再让 SIGTERM 真正触发你的关闭流程。发版时那一小串 502，很多就是这几秒的时间差造成的。

这块点到为止，再往下就是运维的活了。对写代码的人来说，记住一件事就够：你的关闭超时要短于 K8s 的宽限期，且要留出摘流的时间。

## 下次发版前，先问关闭顺序对不对

发版冒 502、请求被截断、事务写一半，这些线上才炸、本地复现不了的毛病，根子常常是同一个：关闭顺序反了。

收到信号就 `os.Exit`，或者信号一来就先关 DB，看着能跑，是因为没赶上那要命的一秒。

真正该有的顺序很简单：入口先关（停 accept、摘流），再等存量请求做完（`srv.Shutdown` 带超时），最后才关依赖（DB、MQ、连接池）。按依赖的反序关，后拉起来的先关掉。

所以下次发版之前，别急着再加一个 signal、再调一次探针参数。先把 `main` 函数最后那段翻出来，问自己一句：

我这个关闭顺序，是入口先关、出口最后关吗？还是信号一来，就把还在飞的请求的桌子给掀了？

---

*技术参考：Go 官方文档 `net/http.Server.Shutdown` 关闭语义、`context` 取消语义；Kubernetes Pod 终止流程（endpoints 摘除与 SIGTERM 并发、terminationGracePeriodSeconds、preStop）。*
