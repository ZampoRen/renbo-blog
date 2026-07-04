---
title: "读多写少就用 sync.Map？先问这 5 个问题"
description: "sync.Map 不是普通 map 的并发升级版。它只在 key 稳定、读命中稳定、写入模式合适时才像优化；一旦牵涉业务不变量、Range 快照、hot key 覆盖和类型安全，map+锁往往更清楚。"
date: 2026-07-04T13:41:44+08:00
draft: false
author: "Zampo"
tags: ["Go", "并发", "sync.Map", "性能优化", "源码"]
categories: ["技术实战"]
cover: "/images/go-sync-map-decision/cover.svg"
toc: true
---

线上有一张设备状态表，用 `sync.Map` 存着。

读很多，写很少。平时接口很稳，一到定时刷新那几秒，读 p99 就会抖一下。你看代码，第一反应可能不是怀疑 `sync.Map`，而是怀疑刷新逻辑：是不是批量太大？是不是网络抖了？是不是 GC 刚好撞上？

这种判断很自然。

因为在不少 Go 后端的直觉里，普通 map 并发不安全，`sync.Map` 并发安全；读多写少，正好该用它。这个链条看起来太顺了，顺到没人再问第二个问题：这张 map 只是在存 key/value，还是还牵着过期队列、计数器、最近更新时间和一组业务不变量？

![sync.Map 选型封面](/images/go-sync-map-decision/cover.svg)

真正的分岔点在这里：`sync.Map` 不是普通 map 的并发升级版。它是少数访问模式下的专用优化。

读多写少只是入口，不是结论。

真正该问的是：key 稳不稳定？写入是不是一次性的？不同 goroutine 操作的是不是不相交的 key？这个 map 是否需要和其他状态一起保持一致？你是否需要 Range 得到一个稳定快照？

这些问题不问清楚，把 `map+RWMutex` 换成 `sync.Map`，经常不是优化，只是把一把看得见的锁，换成了一套更隐蔽的代价。

## 官方第一句话就不是“用它”

Go 官方文档对 `sync.Map` 的态度很克制。

它确实说 `Map` 像 `map[any]any`，可以被多个 goroutine 并发使用，Load、Store、Delete 是摊还常数时间。

但紧接着那段话更重要：`Map` 是 specialized。大多数代码应该使用普通 Go map，再配合单独的锁或协调机制。原因也写得很直：类型安全更好，也更容易维护 map 内容之外的其他不变量。

这句话经常被忽略。

因为我们太容易只记住“并发安全”，忘了官方真正给出的优化场景只有两类：

- 一个 key 的 entry 只写一次，之后读很多次，比如只增长的缓存；
- 多个 goroutine 读、写、覆盖的是彼此不相交的 key 集合。

注意，不是“所有读多写少”。

如果你的配置表 key 很稳定，启动后加载一次，后面高频读取，`sync.Map` 可能很合适。如果每个 goroutine 管自己那一批 key，彼此不怎么撞，`sync.Map` 也可能减少锁竞争。

但如果你有一批 hot key 被反复覆盖，或者每次更新都要同时改计数、过期索引、统计状态，那就不是同一个问题了。

你以为自己在选并发 map，其实是在选谁来表达临界区。

## sync.Map 快在读路径，不快在所有路径

`sync.Map` 容易被误解成“无锁 map”。

它不是。

以 Go 1.25.4 默认实现为例，`sync.Map` 内部有 `read`、`dirty`、`misses`，也有一把 `mu Mutex`。只是它把常见的读命中路径做得很短：先看 `read`，如果 key 在里面，直接读 entry，不需要拿 `mu`。

这就是它快的地方。

稳定 key、稳定命中，读请求很少走到锁那里。我在 Apple M1 Pro、Go 1.25.4 下跑的 benchmark 也说明了这一点：稳定 key 纯读时，`sync.Map` 大约 3.3-4.4 ns/op；`Mutex` 或 `RWMutex` 包住的 map 大约 135-145 ns/op。

这个数字不能外推成你的业务结论，但它能说明一件事：当 workload 正好落在 `sync.Map` 擅长的路径上，它确实很猛。

问题也在这里。

只要 key 不在 `read` 里，而 `dirty` 里可能有新 key，Load 就会进入慢路径，拿 `mu`，查 `dirty`，记录 miss。miss 多了，`dirty` 会被提升成新的 `read`。Store 新 key、删除后重新写入、处理 expunged entry，也都可能走到内部锁和状态维护。

所以 `sync.Map` 的快，不是因为没有锁，而是因为它努力让某些读走不到锁那里。

如果你的写入模式一直把请求推到慢路径，它就不是“更高级的 map”，而是一套带着 atomic、interface、entry、dirty/read 维护成本的结构。

这也是为什么我跑的 benchmark 里另一个场景很有意思：单 hot key，50% 读 50% 覆盖时，`sync.Map` 大约 116-122 ns/op，还有 36 B/op、1 alloc/op；`RWMutex` 包住普通 map 大约 63-65 ns/op，0 alloc。

不要把这组数字读成“hot key 一定不能用 sync.Map”。

应该读成：只看读写比例不够。写的是不是同一个 key，value 有没有装箱分配，读是否稳定命中，都会改变结果。

## 你以为锁保护的是 map，其实锁保护的是不变量

这是 `sync.Map` 最容易把代码带偏的地方。

普通 map 外面包一把锁，看起来笨，但它有一个好处：临界区很显眼。后来的人一眼就知道，这几行代码必须一起发生。

比如：

```go
type Users struct {
    mu       sync.Mutex
    users    map[string]*User
    count    int
    lastSeen map[string]time.Time
}

func (s *Users) Add(id string, u *User, now time.Time) {
    s.mu.Lock()
    defer s.mu.Unlock()

    if _, ok := s.users[id]; !ok {
        s.count++
    }
    s.users[id] = u
    s.lastSeen[id] = now
}
```

这把锁保护的不是 `users` 这个 map 本身。

它保护的是三个状态之间的关系：`users` 里有没有这个人，`count` 要不要增加，`lastSeen` 要不要一起更新。

如果你为了“并发安全”把它改成几张 `sync.Map`，代码可能变成这样：

```go
users.Store(id, u)
lastSeen.Store(id, now)
atomic.AddInt64(&count, 1)
```

每一个单独操作都并发安全。

整件事不一定正确。

因为“只有新用户才增加 count”这个不变量已经不见了。你当然可以继续用 CAS、LoadOrStore、额外状态把它补回来，但那时你已经在重新发明一套临界区，只是它不再像一把 Mutex 那样清楚。

很多时候，`map+锁` 看起来土，恰恰因为它把边界写在了代码表面。

这也是我在 code review 里看到 `sync.Map` 会先问的问题：原来那把锁到底保护了什么？如果答案是“只保护这个 key/value”，可以继续讨论性能；如果答案是“保护一组状态一起变化”，那就别急着删锁。

## Range 不是快照，any 也不是类型安全

`sync.Map.Range` 还有一个很容易踩的坑：它不是一致快照。

官方文档写得很明确：Range 不一定对应 Map 内容的某个一致时刻。并发 Store/Delete 发生时，Range 可能看到任意时点的映射。它还可能是 O(N)，即使回调很早就返回。

这对一些场景很致命。

比如你想定时导出在线用户列表，或者根据当前设备状态做一轮批量决策。你以为 Range 拿到的是“此刻完整状态”，实际上它只是遍历过程中看到的一组结果。对日志采样、缓存清理，这可能够用；对结算、路由切换、批量一致更新，就不够。

还有类型安全。

`sync.Map` 的 API 是 `any -> any`。Load 出来以后，你要自己做类型断言。key/value 的约束靠约定，不靠编译器。

如果项目很小，封装得好，这不是大问题。可很多误用不是这样来的。它们是从“少写一把锁”开始，慢慢散落在业务代码各处：这里 Store 一个 `*User`，那里 Load 后断言一下，另一个地方又把 key 拼成字符串。

维护成本就是这么长出来的。

普通 map + 锁可以封装成一个小类型，把并发边界和类型约束都收进去：

```go
type DeviceTable struct {
    mu sync.RWMutex
    m  map[string]DeviceState
}

func (t *DeviceTable) Get(id string) (DeviceState, bool) {
    t.mu.RLock()
    defer t.mu.RUnlock()
    v, ok := t.m[id]
    return v, ok
}
```

这段代码不酷，但读它的人知道自己拿到的是什么，也知道锁在哪里。

工程里很多“更稳”的东西，长得都不酷。

## benchmark 只说明一件事：workload 决定答案

我跑的 benchmark 有几组结果很适合放在一起看。

稳定 key 纯读，`sync.Map` 很快。

稳定 key，99% 读、1% 写，`sync.Map` 仍然很快，约 7.2-8.1 ns/op；`RWMutex` 是 54-57 ns/op 左右。

goroutine 操作不相交 key 集合时，`sync.Map` 约 25-31 ns/op，也符合官方说的第二类优化场景。

但单 hot key 高频覆盖时，它就不是必胜。插入新 key + 删除旧 key 的自建场景里，`sync.Map` 速度更快，但分配更多。

这些结果放在一起，真正的结论不是“sync.Map 快”或“sync.Map 慢”。

真正的结论是：并发 map 没有脱离 workload 的答案。

你要测自己的 key 分布、读写比例、写入形态、value 大小、是否装箱、是否需要批量一致语义。尤其是线上 p99 抖动，不能只看平均 ns/op。内部慢路径、批量刷新、Range、GC 分配，都可能只在某几个窗口放大。

如果你没有压测，也没有不变量分析，把 `map+锁` 换成 `sync.Map`，只是换了一种不确定性。

## 下次选型，先问这 5 个问题

我更建议把 `sync.Map` 当成 code review 里的问题触发器，而不是默认答案。

看到它，先问五个问题。

![并发 map 选型流程图](/images/go-sync-map-decision/decision-flow.svg)

第一，这个 map 是否只维护 key/value 本身？

如果它还要和计数器、队列、过期索引、统计状态一起变化，优先用普通 map + Mutex/RWMutex。你需要的是一个明确临界区，不是一个并发容器。

第二，key 是否稳定，每个 key 是否基本写一次、读很多次？

如果是，只增长缓存、只读配置表、初始化后基本不变的映射，`sync.Map` 值得测。

第三，不同 goroutine 操作的 key 集合是否大多不重叠？

如果每个 worker 负责自己的 key 集合，`sync.Map` 可能减少全局锁竞争。如果所有人都在抢同几个 hot key，不要期待它变魔术。

第四，是否需要稳定快照、精确 len、按条件批量更新？

如果需要，不要把这些语义交给 `Range`。你可能需要普通 map + 锁、不可变快照、copy-on-write，或者把状态收敛到单 goroutine ownership。

第五，是否在意类型安全和可读性？

长期维护的业务代码，类型和边界经常比少几行锁更值钱。能封装成 `map[K]V + lock` 的小类型，就别急着把 `any` 散出去。

一个粗糙但实用的选择表是：

| 场景 | 优先选择 |
|---|---|
| 只增长缓存，key 写一次读很多次 | `sync.Map` |
| goroutine 操作互不重叠 key | `sync.Map` 可测 |
| 需要维护 map + 其他字段一致性 | `map + Mutex` |
| 读多写少但有明确读写临界区 | `map + RWMutex` 可测 |
| 高频覆盖/删除/热 key 写入 | `map + Mutex/RWMutex` 或分片锁，先压测 |
| 需要类型安全和清晰 API | `map + 锁` 封装成自定义类型 |
| 大量 key、热点分散、追求吞吐 | 分片 map 或专门并发 map，按业务压测 |

这张表也不是法律。

它只是把“读多写少”拆开，让你不要只靠一句口号做选型。

## 别把并发安全误解成业务正确

`sync.Map` 很有价值。

它在稳定读命中、写一次读多次、不相交 key 集合这些场景里，确实能把锁竞争压得很低。Go 标准库给它留位置，不是没有理由。

但它的位置很窄。

它保证的是单项操作的并发可见性，不保证你的业务状态天然一致；它能让某些读路径绕开锁，不代表所有写入、删除、Range 都没有代价；它省掉了显式锁，也拿走了一部分类型安全和临界区表达。

下次你在 code review 里看到 `map+RWMutex`，别急着说“读多写少，换 sync.Map”。

先问一句：这把锁原来保护的是 map，还是保护一组业务不变量？

如果只是前者，再谈优化。

如果是后者，那把锁可能正是代码里最清楚的部分。
