---
title: "加一个字段要改 8 个文件？你写的可能不是 Go 架构"
description: "Go 项目的可维护性，不看目录有多像架构图，而看新人能不能在 30 秒内找到修改点、一个 grep 定位业务逻辑。少分层不是偷懒，不承担变化的分层才是架构税。"
date: 2026-07-01T15:30:00+08:00
draft: false
author: "Zampo"
tags: ["Go", "架构", "项目结构", "DDD", "Clean Architecture", "工程实践"]
categories: ["技术实战"]
cover: "/images/go-architecture-minimalism/cover.svg"
toc: true
---

产品说，订单列表加一个 `coupon_code`。

你以为就是 SQL 多 select 一个字段，response 多返回一个字段。结果打开项目以后，手开始慢慢凉下来：

`OrderDTO` 要加，`OrderEntity` 要加，`OrderModel` 要加。mapper 要补一行。`OrderRepository` interface 要动。`OrderUsecase` 要接。handler 要再转一遍。测试里的 mock 还要跟着改。

业务逻辑没有变。

你只是多传了一个字段，却改了 8 个文件。

这不是 Go 项目“有架构”。这更像是 Java 包袱搬进 Go 以后，外面包了一层整洁架构的壳。

![Go 架构极简主义](/images/go-architecture-minimalism/cover.svg)

我不反对分层，也不反对 DDD、Clean Architecture、六边形架构。它们在复杂业务、复杂团队、复杂边界里当然有价值。

但很多 Go 项目的问题不在“没有架构”。恰恰相反，问题是一开始就太有架构了。

目录看起来很专业，修改点却到处散。interface 很多，真正替换实现的地方几乎没有。DTO、domain model、db model 分得很干净，字段却 1:1 复制。新人进来先学半天目录宗教，再开始找业务逻辑。

这篇想压住一个判断：

Go 项目的架构，不看目录层数，看你能不能在 30 秒内找到修改点，用一个业务词 grep 出主要路径。

## 分层不是错，空转的分层才是成本

一个常见的后端惯性是：handler 不能碰业务，service 承接业务，usecase 编排流程，domain 保持纯净，repository 负责存储，adapter 负责外部系统。

听起来都对。

问题是，真实项目里很多层并没有承担真实边界，只是在转发参数。

```go
func (h *OrderHandler) List(ctx context.Context, req ListReq) (ListResp, error) {
    return h.uc.List(ctx, req.ToDTO())
}

func (u *OrderUsecase) List(ctx context.Context, dto ListOrderDTO) (ListOrderDTO, error) {
    return u.repo.List(ctx, dto)
}
```

这种代码不是不能写。问题是，如果每一层都只是换个名字继续往下传，它没有降低复杂度，只是在制造跳转。

你加字段时改 8 个文件，代价不在那 8 行代码本身，而在你每次都要重新确认：

这个字段在哪一层变形？哪一层做权限？哪一层做过滤？哪一层只是复制？

如果答案是“都没有，只是每层都要加一遍”，那这一层就不是抽象，是架构税。

真正有价值的层，应该承担变化。

例如 HTTP request 和业务命令格式不一样，DTO 有价值。数据库字段和业务概念不一样，model 拆开有价值。一个仓储背后确实有两种实现，interface 有价值。业务流程足够复杂，usecase 作为编排入口有价值。

反过来，如果它只是为了让目录符合某张架构图，就别急着写。

Go 不是没有抽象。Go 只是很不鼓励你提前许愿。

## 目录模板不是免死金牌

Go 社区里最容易被误解的一个仓库，是 `golang-standards/project-layout`。

它 star 很高，名字里还有 standards。很多人第一次建 Go 项目，顺手就把 `cmd/`、`internal/`、`pkg/`、`api/`、`configs/`、`scripts/` 全搬过来。

但这个仓库 README 自己写得很清楚：它不是 Go core team 定义的官方标准。简单项目、PoC、个人项目用这套 layout 是 overkill。更直接一点，它还提醒你：clone 以后保留需要的，删掉不需要的。

最荒诞的地方就在这里。

很多团队复制了目录，却跳过了适用条件。

Go 官方文档对项目布局的口径其实更朴素：简单 package 可以就放在根目录；简单 command 可以就是 `main.go` 加几个文件；项目大了，再把 supporting packages 放进 `internal`；多个 binary，再用 `cmd/<name>/main.go`。服务项目通常不是给别人 import 的库，所以 Go 逻辑放在 `internal`、命令放在 `cmd`，这也合理。

注意这个顺序：先有项目复杂度，再长出目录。

不是先把目录搭满，再等业务来证明它们有用。

`cmd`、`internal`、`pkg` 都不是装饰品。

`cmd` 表示命令入口。`internal` 表示 Go 工具链会强制的 import 边界。`pkg` 如果要用，最好意味着你愿意让外部项目 import，并承担兼容承诺。

如果一个服务只给自己跑，`pkg` 下面塞满 `utils`、`logger`、`response`、`constant`，大概率不是“公共库”，只是垃圾桶换了个更体面的名字。

## Go 的 package 不是文件柜

Java/Spring 背景的开发者，很容易把 Go package 当成分层文件柜：

```text
internal/
  handler/
  service/
  repository/
  model/
  dto/
```

这个结构看着熟。但你改一个订单规则时，注意力会被迫横穿所有技术层。

订单的入口在 handler，规则在 service，结构在 model，SQL 在 repository，转换在 mapper。业务叫 order，但代码散在五个抽屉里。

Go 更自然的组织方式，通常不是按“技术角色”分包，而是先让业务能力靠近：

```text
internal/
  order/
    handler.go
    query.go
    repository.go
    model.go
  user/
    handler.go
    service.go
```

这不是说每个项目都必须这么写。重点是，包名应该帮助你理解业务，而不是证明你知道 MVC。

Go 官方关于 package name 的建议很直白：包名要短、清楚，避免 `util`、`common`、`misc` 这种没有上下文的名字；不要把所有 API、types、interfaces 都塞到一个大包里。包名应该成为调用者理解代码的前缀。

这句话落到业务代码里，就是：

`order.Create` 比 `service.CreateOrder` 更像 Go。

`user.Repository` 不一定比 `repository.UserRepository` 更高级，但它至少让你先看见业务对象，而不是先看见分层名词。

当你不知道一个函数该放哪，别先问“它属于 service 还是 usecase”。先问：它服务哪个业务概念？谁会读它？谁会改它？

## interface 不是给未来许愿的

Go 里还有一个很常见的过度设计：每个 struct 旁边都配一个 interface。

```go
type OrderRepository interface {
    Find(ctx context.Context, id int64) (*Order, error)
}

type orderRepository struct {
    db *sql.DB
}
```

很多时候，这个 interface 只有一个实现。没有测试替身，没有多存储后端，也没有跨包依赖反转。它存在的唯一理由是：大家觉得“面向接口编程”比较专业。

Peter Bourgon 那句老话很适合贴在代码评审里：interfaces are consumer contracts, not producer contracts。

接口是消费方的契约，不是提供方的自我介绍。

如果调用方只需要 `Find`，就在调用方附近定义一个小接口。这样依赖更窄，测试也更顺。反过来，provider 侧预先声明一个大而全的 repository interface，经常会让所有人跟着它的想象力走。

更糟的是，它会破坏阅读路径。

你 Cmd+Click 进去，先看到 interface，再找实现，再找构造，再找注入。明明只有一个实现，Go to Definition 却被人为绕了一圈。

接口当然有价值。

当你确实有多个实现，当测试需要替身，当一个包不应该依赖另一个具体类型，当你想把依赖收窄到几个方法，interface 很好用。

但不要为了“以后可能换数据库”，让今天每一次阅读都多跳两层。

未来的不确定性，不能无限透支今天的可读性。

## main 可以丑一点，但要丑得诚实

很多人受框架影响，会觉得 `main.go` 必须越薄越好。最好只剩几行启动代码，依赖全部藏进 DI 容器、init 函数、全局变量或框架魔法里。

Go 的口味不太一样。

`main` 负责组装依赖，并不低级。它可以是 composition root：创建 db，创建 repository，创建 service，创建 handler，把组件图明明白白接起来。

```go
func main() {
    db := openDB()
    orders := order.NewRepository(db)
    svc := order.NewService(orders)
    api := httpapi.NewOrderHandler(svc)

    log.Fatal(http.ListenAndServe(":8080", api.Routes()))
}
```

这段代码不“高级”。但后来的人一眼能看懂：订单服务依赖什么，HTTP 层接了谁，组件从哪里来。

真正危险的不是 main 长一点。

危险的是依赖藏起来。构造函数签名看不出它会碰数据库，包级变量在别处被 init 改掉，测试时还要猜全局状态什么时候初始化。

当然，main 也不能变成业务垃圾桶。订单折扣、库存规则、审批流程，不应该写在 main 里。

更准确的边界是：main 负责 wiring，不负责业务。

它可以丑一点，但要丑得诚实。

## 什么时候分层是值得的？

写到这里，很容易被误读成“Go 项目就应该少文件、少目录、少设计”。不是。

少不是目的。可定位才是目的。

如果你的业务有清晰的领域模型，有复杂规则，有多种入口，有 HTTP 和 gRPC 两套 transport，有消息消费，有多个存储实现，有团队协作边界，那分层当然有价值。

DDD 也不是错。Clean Architecture 也不是错。

错的是把它们当成起步模板，而不是复杂度出现后的应对手段。

一个 marketplace 后端，用户、卖家、商品、订单、钱包、评论、库存、支付都在里面，domain、repository、service、transport 分开可能很正常。因为每一层都在隔离变化。

但一个三张表 CRUD 服务，上来就 `internal/app/modules/order/usecase/impl`，再配一堆 mapper 和 interface，大概率只是在把简单问题加工成复杂问题。

判断标准不是“用了哪种架构名词”。

判断标准是：下一次真实修改，会不会更安全、更快、更容易被理解。

## 别急着重构，先跑一遍修改路径

如果你已经接手了一个层很多的 Go 项目，第一反应不要是“我要把它改成极简结构”。

这种重构很容易变成另一种架构洁癖。原来是分层洁癖，现在是删目录洁癖。团队还没理解问题，只看到你把熟悉的结构拆了，后面一定会吵。

更稳的做法，是拿一次真实需求跑一遍修改路径。

比如还是 `coupon_code`。不要先画目标架构图，先记录这次变更碰了哪些文件：

```bash
grep -R -E "coupon_code|CouponCode" internal cmd
```

然后把文件分成三类。

第一类是真正承载业务变化的文件：查询条件、展示规则、权限判断、状态流转。它们值得被认真保护。

第二类是边界转换文件：HTTP request/response、DB row、外部 API payload。它们不一定多余，但要确认边界是不是真的不同。

第三类最可疑：只是复制字段、转发调用、同步 interface、更新 mock。它们不一定马上删，但要贴上标签。下次再改字段，如果它们还是只负责跟着动，就说明这里有结构性成本。

这一步的好处是，它不靠口水判断。

你不用跟团队争“DDD 到底对不对”，只要把一次变更的文件列表摆出来。哪些层在保护变化，哪些层在放大变化，大家一眼能看见。

Go 项目的结构调整，最好从这种证据开始。

不是因为某个目录“不 Go”，而是因为它连续三次在真实需求里只制造了同步成本。

## Go 项目结构 7 问

下次你想给项目加层、加 interface、加 `pkg`、加 mapper 之前，先过这 7 个问题。

1. 新人能不能在 30 秒内找到某个业务动作的入口？
2. `grep CreateOrder` 能不能看到主要业务路径，而不是只看到 interface、mock、mapper？
3. 改一个字段通常要改几个结构体？这些结构体之间是不是 1:1 复制？
4. 每个 interface 是否至少满足一个真实需求：测试替身、多个实现、跨包依赖反转？
5. `pkg` 下的代码是否真的承诺给外部项目 import？如果没有，为什么不放 `internal`，或者直接按业务包组织？
6. `internal` 是否在表达 import 边界，还是只是为了让目录看起来专业？
7. `main` 是否只是 composition root，还是混进了业务规则？

这 7 个问题，比争“到底该不该 DDD”更有用。

因为它们不问信仰，只问代价。

真正能改善项目结构的，往往不是一次宏大的重构，而是几次小的、带证据的收缩：删掉一个只转发的 mapper，把一个 provider 侧大 interface 挪到消费方，把 `pkg` 里没人外部 import 的代码收回业务包，把 `internal` 下面七八级路径压回两三级。

每次只动一个点。

动完以后再看下一次需求是不是少跳了几层。这个反馈比任何架构争论都诚实。

如果一次业务字段变更，主要成本花在同步结构体、同步接口、同步 mapper，而不是业务规则本身，这不是解耦，是架构税。

如果一个业务词 grep 下去，只能搜到抽象层、mock 层、转发层，却看不到真正规则和 SQL，这不是封装，是藏线索。

如果一个目录结构让新人先记住 20 个框架名词，再找到一个订单查询，那它看起来再整洁，也是在消耗团队。

Go 架构极简主义不是“把代码都堆到一起”。

它只是提醒你：结构要从修改成本里长出来，不要从焦虑里长出来。

先让代码能被找到。

再让它变得漂亮。
