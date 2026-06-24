---
title: "Go nil 的 4 种分裂人格：一个 nil 引发的 2 小时线上排查"
description: "nil 不是一个统一的空值。nil 指针、nil interface、nil slice、nil map、nil channel 在 Go 里走的是几套完全不同的规则。真正难的不是记住 nil 会不会 panic，而是看懂它现在挂在哪个类型下面。"
date: 2026-06-24T11:30:00+08:00
draft: false
author: "任博"
tags: ["Go", "nil", "interface", "slice", "map", "channel", "底层原理"]
categories: ["技术实战"]
cover: "/images/go-nil-four-personalities/cover.svg"
toc: true
---

有一次线上接口返回了一个很诡异的 `null`。

调用方说：你们这个错误字段为什么一直有值？

我看代码，第一反应是不可能。分支里明明写着：

```go
return nil
```

这事最后拖了将近两个小时。

日志没有 panic，链路没有超时，数据库也没有慢查询。真正浪费时间的地方在于，我们一开始把注意力放错了：以为 nil 就是 nil，以为只要看到 `return nil`，调用方拿到的就一定是空。

最后翻到接口层才发现，问题不是“有没有返回 nil”，而是返回的那个 nil 被塞进了 `error` 这个 interface 里。

```go
type MyError struct{}

func (*MyError) Error() string { return "my error" }

func maybeErr() error {
    var e *MyError = nil
    return e
}

fmt.Println(maybeErr() == nil) // false
```

这段代码最恶心的地方是：`e` 这个指针确实是 nil，但 `maybeErr()` 返回的 `error` 不再是 nil。

你盯着 `return e` 看半天，会怀疑人生。

这就是 Go 里的 nil 最容易坑人的地方。它看起来像一个统一的“空值”，实际是一组挂在不同类型下面的零值。指针的 nil、interface 的 nil、slice 的 nil、map 的 nil、channel 的 nil，不是一个东西。

你真正要记住的不是“nil 会不会炸”。

你要先问：这个 nil 现在披着哪件类型的外衣？

![Go nil 的 4 种分裂人格](/images/go-nil-four-personalities/cover.svg)

## nil interface 最隐蔽：盒子空，不代表盒子不存在

线上排查里最容易漏的，就是 typed nil。

还是刚才那个例子：

```go
type MyError struct{}

func (*MyError) Error() string { return "my error" }

var e error = (*MyError)(nil)

fmt.Println(e == nil) // false
fmt.Printf("%#v\n", e) // (*main.MyError)(nil)
```

很多人第一眼会觉得不合理：里面不是 nil 吗？为什么 `e != nil`？

因为 interface 不是单纯存一个值。

你可以把它想成一个盒子，盒子里至少要记两件事：这里面装的具体类型是什么，以及这个具体值在哪里。不同实现细节会区分空接口和非空接口，但对日常排查来说，抓住这个心智模型就够了：interface 只有在“类型信息”和“值”都没有时，才等于 nil。

`var e error = (*MyError)(nil)` 这行代码做了什么？

它给 `e` 塞进了一个具体类型：`*MyError`。值确实是 nil，但类型已经在了。盒子不是空盒子，而是一个贴着 `*MyError` 标签、里面放着 nil 指针的盒子。

所以它和真正的 nil interface 不一样。

```go
var e1 error
var e2 error = (*MyError)(nil)

fmt.Println(e1 == nil) // true
fmt.Println(e2 == nil) // false
```

这类 bug 经常出现在“函数返回接口类型，但内部拿的是具体指针”的地方。尤其是 `error`。

```go
func loadUser(id int64) error {
    var err *MyError

    if id <= 0 {
        err = &MyError{}
    }

    return err
}
```

这段代码在 `id > 0` 时看起来像返回 nil，实际返回的是一个带着 `*MyError` 类型信息的 interface。调用方写 `if err != nil`，会进去。

正确写法要么直接返回 `nil`，要么别让 nil 指针穿过 interface 边界。

```go
func loadUser(id int64) error {
    if id <= 0 {
        return &MyError{}
    }
    return nil
}
```

这里的口诀很简单：

interface 的 nil，看 type，不只看 data。

以后你看到 `err != nil` 明明进了分支，但打印出来又像 `(*XxxError)(nil)`，别急着骂日志。先查返回路径里有没有 typed nil。

## nil slice 能 append，但别把它和空 slice 混成一团

第二种 nil 没那么阴，但也很常见：nil slice。

```go
var s []int

fmt.Println(s == nil) // true
fmt.Println(len(s))   // 0

s = append(s, 1)
fmt.Println(s)        // [1]
```

nil slice 可以 `len`，可以 `range`，可以 `append`。这让很多人形成一个印象：nil slice 和 empty slice 差不多。

多数业务逻辑里，它们确实可以差不多。

但一旦走到序列化、接口契约、前端展示，差别就出来了。

```go
var a []int = nil
b := []int{}

fmt.Println(a == nil) // true
fmt.Println(b == nil) // false
```

`a` 没有底层数组，slice header 里的指针是 nil；`b` 是长度为 0 的 slice，但它是一个已经初始化过的空集合。两者 `len` 都是 0，语义却不完全一样。

这件事在线上常见的表现是 JSON：你本来想告诉前端“这里是一个空列表”，结果返回了 `null`。前端如果按数组处理，就会多一层兼容逻辑。

```go
type Resp struct {
    Items []int `json:"items"`
}

json.Marshal(Resp{Items: nil})    // {"items":null}
json.Marshal(Resp{Items: []int{}}) // {"items":[]}
```

所以 nil slice 的坑不在于它不能用，而在于它太好用了，好用到你忘了它还表达“没有初始化”。

写内部逻辑时，nil slice 很舒服：零值可用，append 自然工作，不需要到处 `make([]T, 0)`。

写对外接口时，就要谨慎一点。你要的是“没有这个集合”，还是“这个集合为空”？如果 API 契约要求数组，就别把 nil slice 直接丢出去。

口诀是：

nil slice 能加人，但对外可能长得像 null。

这不是 Go 的 bug，是你没有把内部零值和外部契约分开。

## nil map 能看不能写：读起来安全，写进去就炸

nil map 和 nil slice 经常被放在一起讲，但它们的脾气完全不同。

```go
var m map[string]int

fmt.Println(m == nil)     // true
fmt.Println(len(m))       // 0
fmt.Println(m["missing"]) // 0
```

读 nil map 没问题。查一个不存在的 key，得到 value 类型的零值。`len` 也是 0，`range` 也不会出事。

但你只要写：

```go
m["key"] = 1
```

它会直接 panic：

```text
panic: assignment to entry in nil map
```

这和 slice 的差别很关键。

`append` 可以接住 nil slice，因为 append 本来就可能分配新的底层数组，返回一个新的 slice header。你写 `s = append(s, 1)`，左边重新接住结果就行。

map 的赋值不是这种模型。

`m["key"] = 1` 没有返回一个新 map 给你接。它要求当前 map 已经有一套可写的哈希表结构。nil map 没有这套结构，所以写不进去。

所以 map 的零值只适合“读”，不适合“改”。

```go
func count(words []string) map[string]int {
    var m map[string]int
    for _, w := range words {
        m[w]++ // panic
    }
    return m
}
```

这段代码看起来很顺手，尤其是写惯了 slice 的人，会下意识以为零值能撑住第一下写入。

不行。

map 要写，就先 make：

```go
func count(words []string) map[string]int {
    m := make(map[string]int)
    for _, w := range words {
        m[w]++
    }
    return m
}
```

口诀是：

nil map 能看不能动。

这句话比“map 是引用类型”有用。后者太粗，解释不了为什么读没事、写会炸。

## nil channel 不是慢，是永远等不到

channel 的 nil 更像一个开关。

```go
var ch chan int

select {
case v := <-ch:
    fmt.Println(v)
default:
    fmt.Println("default")
}
```

这段代码会走 default。因为 nil channel 永远不会 ready。

如果没有 default：

```go
var ch chan int
<-ch
```

这个 goroutine 会一直阻塞。

nil channel 最容易让人误判的地方，是它看起来像“暂时没有数据”。但它不是一个空队列，也不是一个还没收到消息的 channel。它根本没有底层 channel 结构。

所以它不会变得可读，也不会变得可写。

Go 规范里说得很干脆：nil channel 永远不会为通信准备好。

这条规则反过来也很好用。很多动态 select 都靠它关掉某个分支。

```go
var in <-chan int

if enabled {
    in = realInput
}

select {
case v := <-in:
    handle(v)
case <-ctx.Done():
    return
}
```

当 `enabled == false`，`in` 是 nil，这个 case 就不会被选中。它还写在代码里，但本轮 select 里等于被关掉了。

这比额外写一层 if 更干净，尤其是多个输入源动态启停时。

但也正因为它太干净，bug 也会很隐蔽。你以为某个 goroutine 在等数据，实际它等的是一个永远不会 ready 的 nil channel。表现出来就是：没有 panic，没有错误日志，就是卡住。

排查这类问题时，不要只看“有没有人发送”。先确认 channel 变量本身是不是 nil。

口诀是：

nil channel 是透明墙：select 看得见代码，看不见路。

它不是慢，它是永远不到。

## nil 也有大小：空值不等于没有成本

还有一个容易被忽略的点：不同 nil 值的“外壳”大小不一样。

在 64 位机器上跑下面这段代码：

```go
var p *int
var i any
var s []int
var m map[string]int
var ch chan int

fmt.Println(unsafe.Sizeof(p))  // 8
fmt.Println(unsafe.Sizeof(i))  // 16
fmt.Println(unsafe.Sizeof(s))  // 24
fmt.Println(unsafe.Sizeof(m))  // 8
fmt.Println(unsafe.Sizeof(ch)) // 8
```

我本机 Go 1.26.4 跑出来是：pointer 8 字节，interface 16 字节，slice 24 字节，map 和 channel 各 8 字节。

这不是让你为了几个字节去手工优化。大多数业务代码没必要盯着这个。

它真正有用的地方，是帮你拆掉一个错觉：nil 不是一团虚无。

`var s []int` 的底层数组是 nil，但 slice header 仍然有三段信息：指针、长度、容量。`var i any` 没装具体值，但 interface 自身也有类型和值的槽位。`map` 和 `channel` 变量更像一个指向 runtime 结构的句柄，nil 只是这个句柄还没指向真实结构。

所以不要把所有 nil 都理解成“什么都没有”。

很多时候，nil 的差异不在数据，而在描述数据的那层结构。

口诀是：

不是所有 nil 都一样空。

理解这句话，再看前面几个坑就顺了：interface 坑在类型槽位，slice 坑在 header 和外部表现，map 坑在还没有可写结构，channel 坑在没有可通信对象。

## 遇到 nil 问题，按这张表查

把几种常见 nil 放在一起看，会清楚很多。

| 类型 | nil 时能做什么 | nil 时不能做什么 | 最容易踩的坑 | 排查口诀 |
| --- | --- | --- | --- | --- |
| pointer | 比较、作为接收者传递（方法内部要自保） | 解引用 | 空指针 panic | 指针 nil，看 dereference |
| interface | 只有类型和值都空才等于 nil | 不能只看里面的具体值 | typed nil 导致 `err != nil` | interface 的 nil，看 type |
| slice | `len`、`range`、`append` | 直接表达空数组契约时要小心 | JSON 变 `null` | nil slice 能加人 |
| map | `len`、`range`、读 key | 写 key | `assignment to entry in nil map` | nil map 能看不能动 |
| channel | 在 select 中可用来关闭分支 | 直接收发会永久阻塞 | goroutine 卡死无日志 | nil channel 是透明墙 |

下次接口又返回了奇怪的 `null`，别急着从数据库查到网关。

按这个顺序看：

- 返回值是不是 interface，尤其是 `error`？有没有 typed nil？
- 对外返回的是 nil slice，还是 empty slice？契约要求 `null` 还是 `[]`？
- map 写入前有没有 `make`？是不是只初始化了外层结构，里面的 map 还是 nil？
- goroutine 卡住时，channel 是真的没人发，还是 channel 本身就是 nil？
- 你以为“空”的东西，外面那层 header 或 interface 槽位还在不在？

nil 真正麻烦的地方，从来不是它太复杂。

恰恰相反，是它长得太简单。

一个词，盖住了几套完全不同的运行时行为。你把它当成统一的空值，它就会在 interface、slice、map、channel 这些地方换着方式坑你。

下次你的接口返回 `null`，从 interface 查起。

如果是 goroutine 无声卡住，从 nil channel 查起。

如果是写 map 突然 panic，别怀疑 runtime，先看看你有没有 make。

nil 不会说话。类型会。
