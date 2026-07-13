---
title: "你以为 sync.Map 无锁？它的锁藏在三个地方"
description: "sync.Map 不是无锁 map。read/dirty 双 map 只是让某些读绕过锁，但 Load/Store/Delete 路径各有锁前锁后判断。看不懂这些边界，就别怪线上用了 sync.Map 反而更慢。"
date: 2026-07-13T14:30:00+08:00
draft: false
author: "Zampo"
tags: ["Go", "并发", "sync.Map", "并发设计", "源码", "xsync"]
categories: ["技术实战"]
cover: "/images/go-sync-map-mechanism/cover.png"
toc: true
---

这套对话，我在 code review 里见过很多次。

A 说：这个 map 读多写少，并发安全得处理一下。
B 说：那把 map + RWMutex 换成 sync.Map 吧，无锁的，更快。

第一句不完全错，第二句基本错了。

sync.Map 不是无锁 map。它内部有一把 Mutex，还有一套 read / dirty 双 map 的调度逻辑。它只是努力让你常见的读路径不走锁，但写、删除、miss、dirty 提升这些路径一个都绕不开锁。

你选 `sync.Map`，不是因为它在所有场景下都更快，而是因为你的 workload 正好落在它设计的那条窄路径上。

但绝大多数人跳过这个前提。

## read 和 dirty：一把锁两个桶

`sync.Map` 的内部结构（Go 1.25.4）可以浓缩成五样东西：

```go
type Map struct {
    mu     Mutex
    read   atomic.Pointer[readOnly]  // 无锁读的主流路径
    dirty  map[any]*entry            // 写和 miss 的后备路径
    misses int                       // 读 miss 计数
}
```

其中 `readOnly` 又包了一层：

```go
type readOnly struct {
    m       map[any]*entry
    amended bool  // dirty 里是否有 read 没有的 key
}
```

这里的核心思路是**读写分离**：大多数读操作直接从 `read` 里取，不需要拿锁；只有在 `read` 里找不到（miss），才会走锁路径查 `dirty`。

但这套设计不是无锁。它是把锁的粒度从一个极粗的操作拆成两段——读命中时不用锁，读 miss 或写时用锁。锁还在，只是它希望大部分场景绕过它。

![sync.Map 双 map 读路径](/images/go-sync-map-mechanism/inline-01-read-path.png)

## Load 路径：命中很爽，miss 两把锁

`Load` 的逻辑是三层兜底：

```
Load(key)
  → 查 read.m，命中 → 直接返回（无锁）
  → read.m 没命中且 amended=false → 返回不存在（无锁）
  → read.m 没命中且 amended=true → 拿锁，再查一次 read（防止脏提升），查 dirty
  → 如果 dirty 有，记一次 miss → 解锁
```

第一个细节是**重查 read**。你拿锁后不能直接查 dirty，因为从 read 被释放到锁拿到之间，可能已经有别的 goroutine 触发了 dirty 提升，原先的 miss 现在已经变成命中。重查一次可以避免浪费。

第二个是 **`missLocked`**。每次走慢路径，`misses` 加一。当 `misses >= len(dirty)` 时，触发 dirty 提升——把 dirty 整张 map 提升为新的 read，原来的 dirty 清空，misses 清零。

```go
func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    m.read.Store(&readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}
```

这个机制很聪明。如果一个 key 频繁 miss，意味着你对这个 key 的访问模式不稳定，`sync.Map` 会提升 dirty 来消化这个 cost。但代价就是——每次提升相当于整张 dirty map 的浅拷贝赋值操作，在 dirty map 很大时，这可能不是瞬时完成没有代价的。

## Store 路径：锁不一定，但经常要拿

`Store` （实际调用 `Swap`）的逻辑更复杂：

```
Store(key, value)
  → 查 read.m，命中且 entry 不是 expunged → CAS 更新（无锁）
  → 拿锁 → 再查 read → 条目情况分支：
    → read 命中→ expunged? 解 expunged，写 dirty → CAS 写 entry
    → dirty 命中 → 直接写 entry
    → 双不命中 → 首次写入新 key，初始化 dirty（dirtyLocked），设 amended
       → 写 dirty
  → 解锁
```

注意第三行的"**条目情况分支**"——三种可能对应三种逻辑：

- **read 命中但 entry 是 expunged**：这个 key 之前被删过，现在重新写入。要先把 expunged 状态解除，在 dirty 中重建 entry，才能写。
- **dirty 命中**：key 在 dirty 里，直接写 entry。
- **双不命中**：这是**性能最差**的一条路。首次对新 key 写入时，`dirtyLocked()` 会遍历 read 的整张 map，把非 expunged 的全部拷贝到 dirty，然后把 amended 置为 true。如果 read 里有大量 key，这个拷贝是 O(N) 的。

这就是 `sync.Map` 在 "先大量写新 key，再高频读" 的模式下表现不好的根本原因。

## Expunged：一个奇怪的第三种状态

`sync.Map` 的 entry 有三种状态：

| 状态 | 含义 | 在 read 中？ | 在 dirty 中？ |
|------|------|-------------|-------------|
| `nil` | 已删除（软删） | 是 | 是（如果 dirty 存在） |
| `expunged` | 已从 dirty 中移除 | 是 | **否** |
| `非 nil 指针` | 有效值 | 是 | 是（如果 dirty 存在） |

为什么要多一个 `expunged`？

从 `read` 中删掉一个 key，不是直接删 map 条目，而是把 entry 的指针 CAS 为 `nil`。当 dirty 已经存在时，这个 `nil` 状态的 entry 仍在 dirty 中跟着。

真正的删除发生在下一次 dirty 提升之后。提升前，`dirtyLocked()` 会遍历 read：

```go
func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }
    read := m.loadReadOnly()
    m.dirty = make(map[any]*entry, len(read.m))
    for k, e := range read.m {
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}
```

`tryExpungeLocked` 会把 `nil` 的 entry 变成 `expunged`：

```go
func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := e.p.Load()
    for p == nil {
        if e.p.CompareAndSwap(nil, expunged) {
            return true
        }
        p = e.p.Load()
    }
    return p == expunged
}
```

复制 read 到 dirty 时，nil 状态（已删除）的 entry 不会被复制过去——而是变为 expunged。这样 dirty 就不包含这些已删除的 key，节省了空间。

但代价是：`expunged` 状态的 key 要重新写入时，必须先拿锁，解 expunged，在 dirty 里重建 entry，然后才能 CAS 写值。这条路径比普通 map 重得多。

## Range 副作用：顺带一次 dirty 提升

`Range` 的文档说它不一定对应任意一致时刻的快照。但还有一个容易被忽视的副作用：

```go
func (m *Map) Range(f func(key, value any) bool) {
    read := m.loadReadOnly()
    if read.amended {
        m.mu.Lock()
        read = m.loadReadOnly()
        if read.amended {
            read = readOnly{m: m.dirty}
            copyRead := read
            m.read.Store(&copyRead)
            m.dirty = nil
            m.misses = 0
        }
        m.mu.Unlock()
    }
    for k, e := range read.m {
        v, ok := e.load()
        if !ok { continue }
        if !f(k, v) { break }
    }
}
```

如果 `read.amended == true`（即 dirty 中有 read 没有的 key），Range 会触发一次 dirty 提升。注释写得坦白：Range 本身就是 O(N)，所以一次性把 dirty 提升到 read，算是摊销成本。

但这对调用者是透明的。你调一个 `Range`，它可能额外拿走一把锁，做一次全量 dirty 提升。如果你的 `Range` 刚好卡在高频写入的时机，提升成本就可能落在调用方头上。

## 什么时候该用，什么时候不该用

从上面这套机制，可以推导出 `sync.Map` 真正适合的两种模式，和官方文档说的几乎一致。

**适合：**

1. **写一次，读很多次**。entry 创建后只有 CAS 更新（不需要 dirtyLocked），读路径几乎稳定走 read 无锁。配置表、只增长缓存这样的场景最理想。
2. **key 集合不重叠**。不同 goroutine 操作不同的 key，dirty miss 少，竞争小。如果你每个 worker 管自己的 key 集合，`sync.Map` 可能在多核下有效。

**不适合：**

1. **高频写入新 key**。每次写新 key 都要先拷贝 read 到 dirty（`dirtyLocked`），O(N) 成本。如果写入频率超过 dirty 提升频率，整个系统就在频繁地做拷贝。
2. **大量写入同一 hot key**。如果 key 稳定停留在 dirty（经过删除再写入、expunged 再恢复），每次 Store 都要拿锁做 CAS，不比普通 map+RWMutex 便宜。
3. **需要和其他状态保持一致**。`sync.Map` 的 API 不提供事务性多 key 操作。Range 也不是一致快照。如果你的业务不变量跨多个 key 或跨 map + 其他字段，显式锁更好。
4. **在意类型安全**。`any -> any` 的 API 意味着每次 Load 出来都要断言，长期维护成本高。

## 并发 map 生态：三款第三方实现

`sync.Map` 不是唯一的并发 map 方案。生态中有几个活跃的第三方库，各自做了不同的取舍。

### xsync.Map

GitHub: [puzpuzpuz/xsync](https://github.com/puzpuzpuz/xsync)（1.7k stars，v4.5.0）
作者：puzpuzpuz，Go 并发数据结构库

**设计思路**：基于 CLHT（Cache-Line Hash Table）的改进版 + C++ absl::flat_hash_map 的 meta memory 和 SWAR 查找。每个 bucket 固定 5 个 entry（正好占满 64 字节 cache line），无锁读（读完全不写内存），写时用自旋 + CAS + 扩容辅助。

**关键特点：**

- 泛型 API：`Map[K, V]`，编译期类型安全，不需要 any 断言
- Get 操作**完全不涉及写操作**，没有内存竞争
- 提供 `Compute`（原子化读-改-写）、`Size`（O(1) 大小查询）等扩展方法
- 读取线程参与扩容（cooperative resize），多个读 goroutine 帮它搬数据

**适合场景**：需要频繁读且读 path 要极致快；需要 Size/Compute 等高级操作的场景。

**主要代价**：写操作使用自旋 CAS，写 goroutine 数量多时竞争不小；无锁读 + 自旋写的设计在写极端频繁时比 sync.Map 更费力。另外依赖泛型，需要 Go 1.24+。

### cornelk/hashmap

GitHub: [cornelk/hashmap](https://github.com/cornelk/hashmap)（1.9k stars，v1.0.8）
作者：cornelk

**设计思路**：基于哈希表索引 + 排序链表的无锁 map。所有修改（insert/update/delete）通过原子 CAS 在链表节点上完成。

**关键特点：**

- 纯 lock-free，不涉及任何互斥锁
- 数值 key 类型使用专门的 xxhash，速度更快
- 读路径非常快（自称 benchmark 里纯读比 sync.Map 快 2-3 倍）

**适合场景**：纯读或读极多的场景；数值 key 的读密集型 cache。

**主要代价**：写和删除涉及链表节点 CAS，写竞争多时可能频繁重试；项目更新较少（最后 release 在 2022 年）；遇到哈希冲突时需要遍历链表，极端情况下 O(N)。

### alphadose/haxmap

GitHub: [alphadose/haxmap](https://github.com/alphadose/haxmap)（1k stars，v1.4.1）

**设计思路**：xxhash + Harris lock-free list。号称速度和内存效率兼顾。

**关键特点：**

- 泛型：`haxmap.New[int, string]()`，类型安全
- 自测 benchmark 纯读比 sync.Map 快约 3x、比 cornelk 略快
- 支持 `Len()`（O(N) 遍历）、`ForEach`、`Del` 多个 key 批量删除

**适合场景**：对读延迟要求极高的场景，读为主、写适量。

**主要代价**：依赖 Harris lock-free list，删除和热点 key 写入时竞争不小。更新不算活跃，最后 release 在 2024 年 10 月。

### 横向对比

| | sync.Map | xsync.Map | cornelk/hashmap | haxmap |
|---|---|---|---|---|
| **类型安全** | ❌ `any`→`any` | ✅ 泛型 | ✅ 泛型 | ✅ 泛型 |
| **读锁** | 无（命中 read） | 无（纯原子读） | 无（lock-free） | 无（lock-free） |
| **写锁** | Mutex | 自旋 CAS | CAS 链表操作 | CAS 链表操作 |
| **Size** | ❌ 需要 Range 遍历 | ✅ O(1) | ❌ | ❌ Len() O(N) |
| **Compute** | ❌ | ✅ Atomic RMW | ❌ | ❌ |
| **改进时间 (GHA)** | Go 1.9 (2017) | 活跃 (2026-07) | 不活跃 (2022) | 不活跃 (2024) |
| **Go 版本要求** | 任何 | 1.24+ | 1.22+ | 1.18+ |
| **外部依赖** | 无 | 无 | 无 | 无 |

## 选哪个？

回到最开头的问题：什么时候该用哪个？

一个粗糙的判断框架：

1. **你的并发 map 是否只保护单个 key/value 本身？**
   如果还要保护其他状态一起变化 → 用普通 `map + Mutex/RWMutex`。`sync.Map` 的事务边界不够。

2. **主要访问模式是稳定 key 的频繁读 + 低频写？**
   → `sync.Map` 最合适，读路径基本无锁，写路径可控。

3. **读极多写极少，且需要类型安全？**
   → `xsync.Map` 值得测。它的读完全不写内存，在高度读密集的场景下可能更好。

4. **纯读 cache，key 主要是数值类型？**
   → `cornelk/hashmap` 在数值 key 场景下有优势。

5. **需要 Size、Compute 等扩展功能？**
   → `xsync.Map` 提供，标准库不提供。

6. **极端性能追求，愿意接受额外依赖和复杂性？**
   → benchmark 你自己跑，用 `go test -bench=. -benchmem` 在你的真实 workload 下压测。别人的数字不替你做决定。

## 回到那把锁

`sync.Map` 是一个很聪明的设计。read/dirty 双 map + 三级 entry 状态，让读命中路径上的工作变得极轻。它在对的 workload 上表现非常好。

但它在错的 workload 上比普通 `map + RWMutex` 更重——不仅有自己的分摊成本（dirty 提升、expunged 恢复、amended 判断），还失去了一部分类型安全和临界区表达。

下次 code review 里有人提议"读多写少所以换 sync.Map"，不妨让他先回答这三个问题：

1. 你的 key 稳定吗，还是每天都有新 key 进来？
2. 你说的"写"是新 key 写入，还是现有 key 的 CAS 更新？
3. 这把锁保护的是 map 本身，还是一组业务不变量？

回答完这三个问题，选什么已经出来了。
