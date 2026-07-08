---
title: "重构工单一半在改测试？你的测试绑太死了"
description: "覆盖率80%不敢重构，问题出在哪。四个模式 + 一个判断框架。"
date: 2026-07-08T10:00:00+08:00
draft: false
author: "Zampo"
tags: ["Go", "测试", "重构", "golden file", "subtests", "fuzzing"]
categories: ["技术实战"]
cover: "/images/go-test-refactoring-protection/cover.png"
toc: true
---

上周重构一个核心模块。if-else 堆了六层，改成策略模式，干净多了。

改完跑测试。20 个用例红了 15 个。

不是因为逻辑错了。

是因为测试里 mock 了 ServiceA 调 ServiceB 的顺序，mock 了某个内部方法的调用参数，mock 了某次数据库查询的中间结果。我把内部实现重新组织了一遍，这些 mock 全废了。

花了两个小时改测试。领导问了一句：重构花了一小时，改测试花了两小时。这次重构到底有没有用？

我答不上来。

![测实现 vs 测行为](/images/go-test-refactoring-protection/inline-01.png)

## 覆盖率 80%，到底保不保险

先问自己一个问题：如果覆盖率 80%，你敢不敢在核心模块做大规模重构？

犹豫了。说明你心里清楚——现在的测试不是保护重构，是在保护原来的实现。

**覆盖率说的是哪些代码被跑了，不是哪些行为被保护了。** 这是两回事。

覆盖率告诉你，这一行被执行过。但它不告诉你，如果这一行被重构到另一个文件里，测试会不会无辜飘红。

很多团队的测试写得"对"，但设计得不对。测试跟实现绑太死，重构变成了先改测试再改代码。这不叫安全，叫双重锁定。

有一个判断框架，每次写测试前问自己：

> 如果完全重写这个函数的内部逻辑，这个测试会怎么变？

- 测试不动 → 你在测行为 ✅
- 要改 mock 期望 → 你在测实现 ❌
- 要改断言值 → 你在测行为，但输入输出不稳，先想想接口合不合理

就这么简单。

![判断框架](/images/go-test-refactoring-protection/inline-02.png)

## 测实现 vs 测行为

两个测试一对比，一眼就看清楚了：

```go
// 测实现：知道内部调了 A 再调 B
func TestProcess(t *testing.T) {
    mockA := new(MockA)
    mockB := new(MockB)
    svc := NewService(mockA, mockB)
    svc.Process(ctx, req)
    mockA.AssertExpectations(t)
    mockB.AssertExpectations(t)
    // 重构后：排序变了、参数变了——全炸
}

// 测行为：知道输入 X 得到输出 Y
func TestProcess(t *testing.T) {
    svc := NewService(realA, realB)
    got := svc.Process(ctx, req)
    assert.Equal(t, expected, got)
}
```

左边那个测试知道 Service 内部怎么干活——它 mock 了 A 和 B，验证了调用顺序。重构后输出完全正确，测试也照红不误。

右边只关心一件事：输入什么，输出什么。内部怎么做到的，不在乎。

有经验的会问：那外部依赖怎么办？API、数据库、文件系统总不能全用 real 吧。

对。mock 不是原罪，问题是把 mock 放在哪。尽量把 mock 推到边界层。边界以内用 real 对象构造输入、验证输出。边界以外再 mock。这样重构内部逻辑，只影响边界以内的代码，测试不需要动。

## 四个实战模式

下面四个模式。不用全用，知道它们存在就行，在合适的场景选对的。

### 模式一：golden file

适合测复杂输出——HTML 渲染、SQL 生成、协议序列化。

你想要的不是"某个字段等于某个值"，而是"这段输出整体符合预期"。

```go
func TestRender(t *testing.T) {
    got := Render(input)
    golden := filepath.Join("testdata", t.Name()+".golden")
    if *update {
        os.WriteFile(golden, got, 0644)
    }
    expected, _ := os.ReadFile(golden)
    if string(expected) != string(got) {
        t.Fatalf("got != golden\n%s", diff.Text(expected, got))
    }
}
```

`go test -update` 更新 baseline，`go test` 对比。重构前先更新 golden 看 diff 是否合理，合理就提交，不合理修代码。测试一次通过。

代价：golden file 需要定期 review，不然会变成没人看的死文件。

### 模式二：调用者接口

Go 有个老原则：接口定义在调用方，不在实现方。

放在测试里，这句话的意思是：不要在 repository 层一口气定义好所有接口，在调用方需要的地方定义刚好够用的接口。

```go
// 在 repository 层定义完整接口
type UserRepo interface {
    Get(ctx, id)
    Save(ctx, u)
    Delete(ctx, id)
    List(ctx, filter)
}

// 在调用方定义刚好够用的接口
type UserGetter interface {
    Get(ctx, id) (*User, error)
}
```

第一种写法，测试一个用 UserRepo 的服务，必须 mock 整个 Repo——Get、Save、Delete、List 全部 mock 一遍。重构时只改了 Delete 的实现，所有用 UserRepo 的测试也得动。

第二种写法，测试只关心 UserGetter，只 mock 一个 Get 方法。重构改了 Delete？不关我事，我没用到它。

不是什么高级技巧，就是最小接口原则的测试版本。

![接口设计对比](/images/go-test-refactoring-protection/inline-03.png)

### 模式三：性质测试

不是测具体值，是测代码的性质。

举个最直接的例子：对一组用户按年龄排序。

```go
func TestSortedUsersByAge(t *testing.T) {
    users := []User{{"A", 30}, {"B", 25}, {"C", 30}}
    result := SortByAge(users)

    // 性质：年龄非递减
    for i := 1; i < len(result); i++ {
        if result[i].Age < result[i-1].Age {
            t.Fatal("not sorted by age")
        }
    }

    // 性质：元素数量不变
    if len(result) != len(users) {
        t.Fatal("lost users")
    }
}
```

不用断言"第一个是 B"，而是断言"所有元素按年龄非递减排列"。

不管你怎么重构 SortByAge——用 sort.Slice、用手写冒泡、用并发归并——只要排序逻辑正确，它就通过。

适合排序、校验、过滤、去重这类行为。复杂业务逻辑——比如"按某种权重计算订单总价"——性质不好抽象，靠性质测试不够。

### 模式四：subtests

不适合 golden file 的场景（输出不是大块文本），用 subtests。

```go
func TestParseConfig(t *testing.T) {
    tests := []struct {
        name  string
        input string
        want  Config
    }{
        {"基本字段", `{"host": "localhost"}`, Config{Host: "localhost"}},
        {"默认值填充", `{}`, Config{Host: "default", Port: 8080}},
        {"未知字段忽略", `{"host":"x","unknown":"y"}`, Config{Host: "x", Port: 8080}},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := ParseConfig(tt.input)
            if !reflect.DeepEqual(got, tt.want) {
                t.Fatalf("got %+v, want %+v", got, tt.want)
            }
        })
    }
}
```

table-driven 不是重点。重点是每个 case 用 t.Run 独立运行，每个 case 的名字说清楚它在测什么规则。

重构解析逻辑——从标准库 json 换成手写 parser——每个 subtest 独立验证自己那条规则是否被破坏。一条失败不影响其他规则。

## 还有两个容易被忽略的

### fuzzing：找你没想过的输入

不是高级技巧。就是找"你没想到的输入"用的。

```go
func FuzzParseConfig(f *testing.F) {
    f.Fuzz(func(t *testing.T, input string) {
        cfg := ParseConfig(input)
        if cfg.Port != 0 {
            if cfg.Port < 1 || cfg.Port > 65535 {
                t.Errorf("port out of range: %d", cfg.Port)
            }
        }
    })
}
```

这里不是测 ParseConfig("xxx") 返回什么，是测"任何输入它都不能 panic，端口必须在范围内"。

fuzzing 找的不是常规 bug。是"这个输入团队没人想到"的边界情况。重构后跑一轮 fuzz，比再写 10 个用例管用。

代价是跑得慢。不推荐每次 CI 都跑全量 fuzz，作为 pre-release 或定期任务就行。

### benchmark：保护性能不退化

重构改了内部实现，功能测试全过了。性能呢？

![Benchmark 保护清单](/images/go-test-refactoring-protection/inline-04.png)

```go
func BenchmarkProcess(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Process(bigInput)
    }
}
```

benchmark 配上 `-benchmem`，能在重构后发现内存分配多了 30%、延迟翻了一倍。这些行为退化，功能测试看不出来。

## 综合一下

重构前有个函数同时做三件事：解析 JSON → 校验字段 → 格式化输出。

重构后拆成三段独立函数：ParseConfig、ValidateConfig、FormatOutput。

测试怎么对应？

| 模块 | 测试方法 | 重构后测试动了吗？ |
|------|----------|-------------------|
| ParseConfig | golden file 测解析结果 | 不动 |
| ValidateConfig | 性质测试（合法→通过，非法→报错） | 不动 |
| FormatOutput | table-driven + subtests 测每种格式 | 不动 |
| 整体链路 | 1-2 个 happy path | 不动 |

重构前后，测试一个不改。

跑测试，20 个用例全绿。不是因为运气好，是每个测试都站在调用者的位置，只验证行为，不关心实现。

![重构保护清单](/images/go-test-refactoring-protection/inline-05.png)

---

**下次给核心模块写测试前，问自己三个问题：**

1. 如果完全重写这个函数的内部逻辑，这个测试会红吗？
2. 我是在断言输出，还是在扒实现细节？
3. 这个测试能不能独立说明一条行为规则被保护了？

三个都能说清楚，你的测试就能保护重构。

如果有收获，可以关注。后面接着聊怎么架 fixture、怎么选 mock 边界、怎么用 testing/synctest 跑并发测试。
