---
title: "Go 连接池报错，别先把 max_open 调大"
description: "Go 数据库连接池不是越大越稳。max_open 管并发上限，max_idle 管建连频率，idle_time 管空闲回收，lifetime 管连接寿命。真正要做的是先看 DB.Stats 和数据库侧指标，再决定该调哪一个阀门。"
date: 2026-07-04T14:53:48+08:00
draft: false
author: "Zampo"
tags: ["Go", "database/sql", "GORM", "数据库", "性能优化"]
categories: ["技术实战"]
cover: "/images/db-connection-pool/cover.svg"
toc: true
---

线上报 `connection pool exhausted`，我见过不少人的第一反应是把 `max_open` 调大。

从 20 调到 100，报警没了，接口也稳了。看起来问题解决了。

直到另一个服务把它调到 200，还是超时；DBA 又在群里说数据库连接数被打满了。你才发现，自己根本不知道刚才那个数字是在救服务，还是在把数据库往坑里推。

![Go 数据库连接池封面](/images/db-connection-pool/cover.svg)

Go 的数据库连接池最容易被误解的地方，不是参数多，而是每个参数保护的对象不一样。

`max_open` 不是油门，是刹车。它限制的是你的 Go 进程同时向数据库伸出去的手。

连接池报错时，真正要问的不是“要不要加大池子”，而是：现在到底卡在等连接、忘了还连接、频繁建连接、数据库扛不住，还是连接寿命和中间层超时打架？

## 连接池不是给你兜底的，它是挡在数据库前面的阀门

`database/sql` 里的 `*sql.DB` 很容易让人误会。

名字叫 DB，但它不是一条数据库连接。它是一个可以被多个 goroutine 并发使用的句柄，内部会按需要管理一组到底层数据库的连接。

你执行 `Query`、`Exec`，它会去拿一条可用连接。没有空闲连接时，如果还没到上限，就新建；如果已经到了 `SetMaxOpenConns` 的上限，请求就等。

这句话很关键。

一旦你设置了 `max_open`，数据库访问就有点像抢信号量。抢不到，就排队。Go 官方文档也提醒过，设置上限后，应用可能等待连接，甚至在某些调用链里把自己等死。

所以 `max_open` 调大，不等于吞吐一定变大。

它只是允许更多请求同时挤到数据库面前。数据库还有自己的 CPU、IO、锁、连接数上限、慢查询和事务。应用侧看起来“池子不满了”，数据库侧可能已经开始喘不过气。

## 先把四个旋钮分清楚

这几个参数最好不要放在一起背。

它们管的是四件事。

### `SetMaxOpenConns`：同时打开多少连接

它决定一个 Go 进程最多能打开多少条数据库连接。

设成 0，表示不限制。听起来自由，线上通常不自由。因为数据库连接不是免费的，应用实例一多，很容易把数据库总连接预算吃光。

更稳的做法是反过来算：数据库最多能给业务多少连接？线上有几个应用实例？还要给管理连接、迁移任务、其他服务留多少余量？先分预算，再压测。

不要从“我的服务需要多少”开始，要从“数据库最多能承受多少”开始。

### `SetMaxIdleConns`：空闲时留多少连接

这个参数控制 idle 连接上限。

它不是浪费。很多线上抖动，就是因为 idle 设得太小，流量一上来就集体建连接。建连本身要握手、认证、初始化会话，有些驱动和数据库还会做额外准备工作。

默认只保留 2 条 idle 连接。对并发稍微高一点的服务，这个默认值经常太保守。

但 idle 也不是越多越好。留太多，数据库侧看到的连接数会长期偏高；多实例同时存在时，空闲连接也会堆起来。

`max_idle` 的直觉可以简单一点：如果服务流量平稳，连接建立成本高，它可以接近 `max_open`；如果流量脉冲明显，要配合 idle time，把峰值之后留下的连接回收掉。

### `SetConnMaxIdleTime`：空闲多久后回收

这个参数经常被忽略，但它很适合处理“突发流量之后连接不下来”的问题。

比如一次活动、一个定时任务、一波批处理，把连接池打到高水位。高峰过了，如果只靠 `max_idle`，这些 idle 连接可能继续挂在数据库上。

`SetConnMaxIdleTime` 管的是：一条连接空闲太久，就关掉。

它解决的不是高峰，而是高峰结束后的尾巴。

### `SetConnMaxLifetime`：一条连接最多活多久

这个参数不是为了让 Go 少建连接。

它更多是为了不让一条连接活得比中间层更久。

线上数据库前面可能有 RDS Proxy、负载均衡、连接代理、防火墙、NAT。它们都可能有自己的连接超时策略。如果你的连接在 Go 里一直复用，结果中间层先把它断了，应用侧就会看到一些很烦的偶发 EOF、reset、timeout。

`max_lifetime` 的思路是：我主动在更早的时间换连接，不等中间层半夜把我踢下线。

5 分钟、10 分钟都只能算起点，不是标准答案。真正的标准是：比你链路里最短的连接超时更短，再用错误率和重连频率验证。

还有一个工程细节：如果所有实例同一时间启动，又设置了相同 lifetime，连接可能同一批过期。Go 标准库没有 `SetConnMaxLifetimeJitter` 这种 API。要做抖动，只能在配置层自己给 lifetime 加一点随机偏移，避免大家同时换连接。

![连接池参数排查流程](/images/db-connection-pool/connection-pool-flow.svg)

## pool exhausted，不一定是连接太少

看到 `pool exhausted`，最危险的动作就是直接加 `max_open`。

它可能确实太小。

但也可能是连接拿出去以后没回来。

最常见的几个地方：`Rows` 没有 `Close`，事务没有 `Commit` 或 `Rollback`，手动拿了 `Conn` 却没有 `Close`。这些问题在低流量时不明显，一到并发上来，就像池子突然被人抽干。

排查时先看 `DB.Stats()`。

```go
stats := db.Stats()
log.Printf("open=%d in_use=%d idle=%d wait=%d wait_duration=%s",
    stats.OpenConnections,
    stats.InUse,
    stats.Idle,
    stats.WaitCount,
    stats.WaitDuration,
)
```

如果 `InUse` 长时间贴着 `MaxOpenConnections`，`WaitCount` 和 `WaitDuration` 持续上涨，说明请求真的在等连接。

下一步不要急着改配置。

先看慢查询、事务耗时、请求超时、`Rows.Close`、`Tx.Rollback` 这些地方。连接池只是把症状暴露出来，真正占着连接不还的，往往在业务代码里。

## too many connections，说明你越过了数据库的边界

`too many connections` 的方向反过来。

这时不是 Go 在池子里排队，而是数据库侧已经不想再接了。

如果一个服务有 8 个实例，每个实例 `max_open=100`，理论上光这个服务就可能打开 800 条连接。再加上其他服务、后台任务、迁移脚本、管理工具，数据库连接数很快就没余量。

所以 `max_open` 一定要按实例数算。

GORM 也一样。它底层还是 `database/sql`，要通过 `db.DB()` 拿到底层 `*sql.DB` 再配置：

```go
sqlDB, err := db.DB()
if err != nil {
    return err
}

sqlDB.SetMaxOpenConns(50)
sqlDB.SetMaxIdleConns(30)
sqlDB.SetConnMaxIdleTime(5 * time.Minute)
sqlDB.SetConnMaxLifetime(30 * time.Minute)
```

这些数字只是示例，不是推荐值。

真正应该上线的，是一套观测：应用侧看 `OpenConnections / InUse / Idle / WaitCount / WaitDuration`，数据库侧看当前连接数、活跃查询、锁等待、CPU、IO 和慢查询。两边一起看，才知道瓶颈在哪里。

只看应用，你会误以为池子小。

只看数据库，你又可能误以为应用不该并发。

## 间歇性 EOF 和 timeout，经常不是 max_open 的锅

还有一种问题更烦：平时正常，偶尔报 EOF、connection reset、timeout。

这种问题很容易被误判成连接池不够大。可你加了 `max_open`，它还是偶发。因为问题不在“有多少连接”，而在“这些连接活了多久”。

尤其是服务后面有数据库代理、负载均衡、NAT、防火墙时，连接可能在某个时间点被中间层回收。应用侧拿到的还是池子里的旧连接，真正用的时候才发现对面已经没了。

这时该看 `SetConnMaxLifetime`，不是 `SetMaxOpenConns`。

让连接在中间层超时之前主动退休，通常比等它随机死掉更可控。代价也要看见：lifetime 太短，会增加重连；太长，又可能撞上中间层超时。

这就是连接池调参的本质：不是找一个神奇数字，而是在等待、建连、数据库压力和连接寿命之间做取舍。

## 一个能落地的起始顺序

如果你现在手里只有一套乱配的连接池，我会按这个顺序重看一遍。

第一，看数据库总连接预算。

不要先问单个 Go 服务想开多少。先问数据库愿意给整个业务多少，再按实例数和服务优先级分下去。`max_open` 是预算分配，不是拍脑袋。

第二，看等待。

`WaitCount`、`WaitDuration` 持续涨，说明请求在等连接。再结合 `InUse`、慢查询、事务耗时，判断是池子太小，还是连接被长期占用。

第三，看建连频率。

如果流量一上来就频繁建连接，接口抖动，先别骂数据库。检查 `max_idle` 是不是太小，`idle_time` 是不是太短。

第四，看连接寿命。

如果错误像幽灵一样偶发，尤其是长连接、低频请求、代理链路里出现 EOF/reset，去查 `max_lifetime` 和中间层超时。

然后再改数字。

改完别只看报警消失。看等待时间有没有下降，数据库连接数有没有越界，建连频率有没有变高，错误类型有没有变化。

## 需要抖动时，不要假装标准库有这个参数

很多连接池文章会顺手写一个 `max_conn_lifetime_jitter`。

这个想法本身没错。多实例同一时间启动、同一套 lifetime，同一批连接差不多也会在同一时间退休。流量高的时候，集中回收会制造一小段重连尖峰。

但在 Go 的 `database/sql` 里，它不是一个公开 API。

所以正文里不要写成“设置这个参数”。读者真去找，会发现 `SetConnMaxLifetimeJitter` 根本不存在。

更朴素的做法，是在配置层给 lifetime 加一点随机偏移。比如基准值 30 分钟，每个实例启动时在 0 到 3 分钟之间取一个随机值，加到最终 lifetime 上。这样每个实例的回收节奏不完全一致。

```go
base := 30 * time.Minute
jitter := time.Duration(rand.Int63n(int64(3 * time.Minute)))

sqlDB.SetConnMaxLifetime(base + jitter)
```

这段代码也不是让你无脑复制。

它只是提醒一件事：工程上有些问题要靠配置策略解决，不要把所有能力都想象成库参数。

如果你的服务实例很少，启动时间本来就错开，或者连接寿命很长、回收压力不明显，这个抖动未必有价值。只有当你真的观察到集中重连、连接创建尖峰、错误率同步抖动时，它才值得加。

连接池真正该带走的一句话是：

报 exhausted 先查谁在等、谁没还；报 too many connections 先查预算；报偶发断连先查寿命。`max_open` 只是其中一个阀门，不是万能止痛药。
