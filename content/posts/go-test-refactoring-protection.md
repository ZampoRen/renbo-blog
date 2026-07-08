---
title: "重构工单里改了一半测试？说明你的测试在测实现"
description: "测试覆盖率说的是'哪些代码被跑了'，不是'哪些行为被保护了'。四个实战模式 + 一个判断框架，帮你写出重构不用改的测试。"
date: 2026-07-08T10:00:00+08:00
draft: false
author: "Zampo"
tags: ["Go", "测试", "重构", "golden file", "subtests", "fuzzing", "测试设计"]
categories: ["技术实战"]
cover: "/images/go-test-refactoring-protection/cover.png"
toc: true
---

上周我重构一个核心模块。

if-else 堆了六层，逻辑在三个分支里来回跳。改成策略模式——每个分支一个策略，干净多了。

改完跑测试。20 个用例，红了 15 个。

不是因为逻辑错了。

是因为测试里 mock 了 ServiceA 调 ServiceB 的顺序，mock 了某个内部方法的调用参数，mock 了某次数据库查询的中间结果。我把内部实现重新组织了一遍，这些 mock 全部失效。

花了两个小时改测试。改完之后，领导问了一句话："重构花了一小时，改测试花了两小时。这次重构到底有没有用？"

这个回答，我用了两年才想清楚。

## 覆盖率 80%，不代表你的测试安全

先问自己一个问题：如果你的测试覆盖率是 80%，你敢不敢在一个核心模块做大规模重构？

如果你的答案犹豫了，说明你心里清楚——现在的测试不是在保护重构，是在保护原来的实现。

**测试覆盖率说的是"哪些代码被跑了"，不是"哪些行为被保护了"。**

这是两回事。

覆盖率告诉你，这一行被跑过了。但它不告诉你，如果这一行变了，该不该报错。如果这一行被重构到另一个文件里，测试会不会无辜地飘红。

很多团队的测试做得"对"，但设计得不对——测试跟实现绑得太死，重构变成了先改测试再改代码。这不叫安全，这叫双重锁定。

## 测实现 vs 测行为

两个测试的对比，一眼就能看清区别：

```go
// ❌ 测实现：知道函数内部调了 A 再调 B
func TestProcess(t *testing.T) {
    mockA := new(MockA)
    mockB := new(MockB)
    svc := NewService(mockA, mockB)
    svc.Process(ctx, req)
    mockA.AssertExpectations(t)
    mockB.AssertExpectations(t)
    // 重构后：排序变了、参数变了、合并了——全炸
}

// ✅ 测行为：知道输入 X 得到输出 Y
func TestProcess(t *testing.T) {
    svc := NewService(realA, realB)
    got := svc.Process(ctx, req)
    assert.Equal(t, expected, got)
}
```

左边的测试知道 Service 内部怎么做事——它 mock 了 A 和 B，验证了调用顺序。重构后哪怕输出完全正确，测试也会红。

右边的测试只关心一件事：输入是什么，输出是什么。内部怎么做到的，我不问。

**一个判断框架：如果完全重写这个函数的内部逻辑，这个测试会怎么变？**

- 测试不动 → 在测行为 ✅
- 要改 mock 期望 → 在测实现 ❌
- 要改断言值 → 在测行为，但输入输出不稳，先审视接口抽象

有经验的 Go 开发者会问："那外部依赖怎么办？API、数据库、文件系统总不能全用 real 吧。"

对。mock 不是原罪，问题是把 mock 放在了哪个位置。尽可能把 mock 推到系统边界层。边界以内，用 real 对象构造输入、验证输出。边界以外，再考虑 mock 外部依赖。这样重构内部逻辑时，只影响边界以内的代码，测试不需要动。

## 把测试当作模块的第一个调用者

每次写测试前，先做一个思维切换：别把自己当成"代码验证者"，把自己当成"模块的第一个调用者"。

调用者关心的是承诺，不关心实现。

这个函数承诺"输入用户名返回用户信息"，我不在乎它先查缓存还是先查数据库。只要你返回正确的信息，对我的调用来说就是对的。

但在实际写测试时，很多人不自觉就站在了实现者的位置上。选测试框架的时候，下意识选那些能精细控制内部状态的框架。写断言的时候，不自觉地验证了不该验证的内部细节。

**站回调用者的位置，你的测试会自然变得更稳。**

原因很简单：调用者的视角天然抽象。你只关心输入输出契约，不关心内部实现。这个契约不变，你的测试就不该变。

## 四个实战模式

下面四个模式，每个解决一类问题。不需要全部用上，但至少应该知道它们存在，在合适的场景选对的那个。

### 模式一：golden file / snapshot test

适合测复杂输出——HTML 渲染、SQL 生成、协议序列化、配置导出。

你想要的不是"某个字段等于某个值"，而是"这一段输出整体符合预期"。

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

`go test -update` 更新 golden，`go test` 对比。第一次跑生成 baseline，之后每次改输出都有 diff。

重构前先更新 golden（如果重构改了输出），验证 diff 是否合理。合理就提交。不合理就修代码。测试一次通过。

这个模式最直观的价值在于：它把"判断输出是否正确"从写测试的阶段推迟到了 code review 的阶段。人审核 diff，自动化跑对比。代价只有一个：golden file 需要定期 review，否则会变成没人看的死文件。

### 模式二：调用者视角的接口设计

这是 Go 的一个经典设计原则：**接口定义在调用方，不在实现方。**

放在测试场景里，这句话的意思是：不要在 repository 层一口气定义好所有接口，让调用者去实现。而要在每个调用方需要的地方，定义它刚好够用的那个接口。

```go
// ❌ 在 repository 层定义完整接口
type UserRepo interface {
    Get(ctx, id)
    Save(ctx, u)
    Delete(ctx, id)
    List(ctx, filter)
}

// ✅ 在调用方定义刚好够用的接口
type UserGetter interface {
    Get(ctx, id) (*User, error)
}
```

第一种写法，测试一个使用 UserRepo 的服务时，必须 mock 整个 UserRepo——Get、Save、Delete、List 全部 mock 一遍。重构时哪怕只改了 Delete 的实现，所有用 UserRepo 的测试也得动。

第二种写法，测试只关心 UserGetter 这个接口，只 mock 一个 Get 方法。重构时改了 Delete 的实现？不关我的事，我没用到它。

这不是什么高级技巧。它就是"最小接口原则"的测试版本。但很多团队写得顺手之后，会把接口定义成庞大的"服务接口"，然后让所有调用者都依赖它——测试也跟着遭殃。

### 模式三：approval test / 性质测试

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

这个测试，不管你怎么重构 SortByAge——用 sort.Slice、用 sort.Sort、用手写冒泡甚至用半合并——只要排序逻辑正确，它就一定通过。

性质测试的有效边界很明显：它只适合那些性质清晰的操作。排序、校验、过滤、去重这类行为天然适合。而复杂业务逻辑——比如"按某种权重计算订单总价"——性质不好抽象，靠性质测试就不够。

Go 标准库提供了 `testing/quick` 包来做随机性质测试，但功能有限。社区有 `leanovate/gopter` 更强一些。不过即使不依赖任何框架，手写性质断言也是值得养成的习惯。

### 模式四：subtests + 明确的行为断言

不适合 golden file 的场景——也就是输出不是大块文本，而是可分解的行为规则——用 subtests 处理。

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

table-driven test 在这里不是核心。核心是：**每个 case 用 t.Run 独立运行，每个 case 的名字说清楚它在测什么行为规则。**

重构解析逻辑时——比如从标准库 json 换成手写 parser，或者加入了错误容忍机制——每个 subtest 独立验证自己那条规则是否被破坏。一条规则失败不影响其他规则。

## 还有两个容易被忽略的

### fuzzing：找你没定义过的行为

fuzzing 不是高级技巧。它是找"你没想到的输入"的工具。

```go
func FuzzParseConfig(f *testing.F) {
    f.Fuzz(func(t *testing.T, input string) {
        cfg := ParseConfig(input)
        // 契约：不 panic
        // 契约：端口在合法范围
        if cfg.Port != 0 {
            if cfg.Port < 1 || cfg.Port > 65535 {
                t.Errorf("port out of range: %d", cfg.Port)
            }
        }
    })
}
```

这里不是测 ParseConfig("xxx") 返回什么，而是测"任何输入，它都不能 panic，端口必须在范围内"。

fuzzing 找到的不是常规 bug，是"这个输入团队没人想到"的边界情况。重构后跑一轮 fuzz，比再写 10 个用例还能防住意外。

代价是跑得慢。所以不推荐在每次 CI 都跑全量 fuzz，而是作为 pre-release 或定期任务。

### benchmark：保护性能不退化

```go
func BenchmarkProcess(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Process(bigInput)
    }
}
```

重构改了内部实现，功能测试通过了。但性能呢？

benchmark 配上 `-benchmem`，能在重构后发现内存分配多了 30%、延迟翻了一倍。这些行为退化，功能测试是看不出来的。

把 benchmark 加入 CI，设一个性能阈值，超出就打回。这样重构的速度优化不会以另一种方式丢失。

## 综合案例：一个完整的重构保护

重构前有个函数同时做三件事：解析 JSON → 校验字段 → 格式化输出。

重构后拆成三段独立函数：ParseConfig、ValidateConfig、FormatOutput。

测试怎么对应？

| 模块 | 测试方法 | 重构后测试动了吗？ |
|------|----------|-------------------|
| ParseConfig | golden file 测解析结果 | ✅ 不动 |
| ValidateConfig | 性质测试（合法→通过，非法→报错） | ✅ 不动 |
| FormatOutput | table-driven + subtests 测每种格式 | ✅ 不动 |
| 整体集成 | 1-2 个 happy path 测全链路 | ✅ 不动 |

重构前后，测试一个不改。

重构完成，运行测试。20 个用例全部飘绿。不是因为运气好，是因为每个测试都站在了调用者的位置，只验证行为契约，不关心内部实现怎么组合。

**能保护重构的测试，不是技术问题，是设计问题。**

把测试当作模块的第一个调用者来设计，你自然就会思考这个函数的输入输出契约是什么。契约不变，测试就不动。内部实现随便改，只要承诺不破，测试就信任你。

回到开头的场景。

那次花了两个小时改测试。如果当时每个测试都是站在调用者位置写的——只测输入输出、不 mock 内部调用顺序、性质清晰的部分用性质测试——那 20 个用例可能只会红 2-3 个，而不是 15 个。

花两小时改测试，暴露的不是你测试写得不够好。

**是你把测试当成了"代码的验证者"，而不是"模块的保护者"。**

---

**行动框架：下次给核心模块写测试前，先问自己三个问题：**

1. 如果我完全重写这个函数的内部逻辑，这个测试会红吗？
2. 我的测试是在断言输出，还是在内核实现细节？
3. 这个测试能否独立说明一条行为规则被保护了？

三个答案都能说清楚，你的测试就能保护重构。

如果你觉得有收获，可以关注。后面还会继续聊 Go 测试设计的更多实战——怎么架 fixture、怎么选 mock 边界、怎么用 testing/synctest 跑并发测试。
