---
title: "TypeScript 押注 Go：10 倍提速背后，不是语言胜负，是工程约束"
description: "TypeScript native port 为什么选择 Go，而不是 Rust 或 C#？这不是语言粉丝战争，而是大型代码库迁移里的约束求解：兼容、移植成本、内存管理、语言服务速度，以及 AI 编程时代的低延迟语义上下文。"
date: 2026-05-19T20:30:00+08:00
draft: false
author: "任博"
tags: ["TypeScript", "Go", "编译器", "前端工程", "AI 编程"]
categories: ["技术观察"]
cover: "/images/typescript-go-native-port/cover.svg"
toc: true
---

TypeScript 的 10 倍提速，最容易被讲歪。

一歪成“Go 打赢 Rust”。

再歪成“以后 TypeScript 业务代码快 10 倍”。

这两个说法都抓人，也都危险。前者把工程决策讲成语言饭圈，后者直接把性能场景讲错了。

真正有意思的地方不是 Go 有多神，也不是 Rust 或 C# 有多不行，而是 TypeScript 团队这次面对的目标非常特殊：他们不是要从零设计一个新编译器，而是要把一个成熟、庞大、被整个前端生态依赖的工具链，迁到 native 实现里。

这件事一旦说清楚，Go 为什么胜出，就没那么玄学了。

![TypeScript 与 Go native port 封面图](/images/typescript-go-native-port/cover.svg)

## 10 倍提速，先别理解错

官方博客里的 10 倍，不是说你写出来的 JavaScript 业务代码运行快 10 倍。

TypeScript 最后还是编译成 JavaScript，线上跑得快不快，主要看 JS 引擎、业务逻辑、网络、渲染、数据结构和运行时环境。TypeScript 这次提速，发生在开发期工具链：`tsc`、类型检查、项目加载、语言服务、编辑器响应。

这区别很重要。

因为一个错的期待，会毁掉一个本来很有价值的技术升级。

官方 benchmark 里，VS Code 代码库的 `tsc` 检查从 77.8 秒降到 7.5 秒，Playwright 从 11.1 秒降到 1.1 秒，TypeORM 从 17.5 秒降到 1.3 秒。编辑器加载 VS Code 项目，也从约 9.6 秒降到约 1.2 秒。

这不是运行时性能新闻。

这是开发体验新闻。

![TypeScript native port 官方 benchmark](/images/typescript-go-native-port/benchmark.svg)

如果你做过大型 TS 项目，会知道这类速度意味着什么。

打开 monorepo，编辑器先卡一会儿；改完一个类型，等错误列表慢慢刷新；跑一次 `tsc --noEmit`，几十秒过去了；想做全项目重构，工具链先喘半天。

这些东西单看都像“小摩擦”。但每天重复几十次，它就不是小摩擦了。

**真正的提速，不是少等几秒，是让过去太贵的能力变成默认能力。**

全项目错误实时刷新，复杂 refactor 更敢做，编辑器能更快拿到完整语义信息。再往后看，AI coding tools 如果想理解类型、调用链、项目结构，也需要低延迟的 semantic information。

模型很重要。

但模型拿不到上下文，再聪明也只能猜。

## 这不是 rewrite，准确说是 native port

中文语境里，大家很容易把这件事说成“TypeScript 用 Go 重写”。传播上没问题，正文里必须校正。

官方更准确的说法是 native port。

rewrite 和 port 的差别很大。

rewrite 更像重新设计一套系统：结构可以换，抽象可以重做，历史包袱可以切掉，顺手还能把旧代码里不满意的地方重构一遍。

port 不一样。port 的重点是保持行为一致、结构相似、迁移可控。尤其 TypeScript 这种基础设施，不能拍脑袋说“我们做了一个更优雅的新东西，你们生态自己适配一下”。

它要兼容的是整个 JavaScript / TypeScript 世界里无数项目、插件、编辑器、构建工具、CI 流程和历史配置。

这也是为什么语言选择不能只看“谁更快”“谁更现代”。

如果目标是从零重写，Rust、C#、甚至别的语言都可以讨论。但如果目标是把现有 TypeScript 编译器和工具尽量平稳地 port 到 native codebase，问题就变了。

**当目标是 native port，语言选型就不是信仰投票，而是约束求解。**

## 为什么不是 Rust？为什么不是 C#？

围绕 TypeScript 选 Go，最热闹的问题就是：为什么不是 Rust？为什么不是 C#？

Rust 很强。性能、类型系统、内存安全、工程声誉都在线。如果这是一个从零开始的新编译器项目，Rust 当然会是强候选。

但 TypeScript 团队在 GitHub Discussion #411 里说得很清楚：他们最看重的是保持语义和代码结构兼容。团队未来还要维护 JS 代码库和 native codebase，相似的结构能让改动在两边更容易同步。

Rust 的问题不是“不行”，而是它会逼你重新思考很多东西：memory management、mutation、data structuring、polymorphism、laziness。对一个 ground-up rewrite 来说，这可能是好事；对一个 port 来说，这会增加迁移成本。

C# 更有反常识感。毕竟 Anders Hejlsberg 是 C# 和 TypeScript 的核心人物，微软自己也有成熟的 .NET 生态。外人第一反应很正常：微软为什么不用自家语言？

但这里要小心，不能把它写成“官方认为 C# 不适合”或者“C# 因为 OOP 被否定”。这些不是官方表述。

更稳妥的说法是：官方解释集中在结构相似、迁移可控、内存布局与分配、图遍历，以及和现有 TypeScript codebase 的 coding patterns 贴合度。Go 在这些约束下更顺手。

![Anders Hejlsberg（图片来源：Wikimedia Commons, DBegley, CC BY 2.0）](/images/typescript-go-native-port/anders-hejlsberg.jpg)

## Go 赢在“少制造新问题”

Go 在这件事里的优势，不是“性能最极致”。

它赢在几个非常工程化的点上。

第一，Go 的惯用写法和现有 TypeScript 代码结构更像。Ryan Cavanaugh 的说法是，Idiomatic Go strongly resembles existing coding patterns of the TypeScript codebase。对 port 来说，这句话比“跑得快”更关键。

第二，Go 有 GC，但 TypeScript compiler 的场景里，GC 的负面代价没那么突出。批处理编译结束后进程就退出；语言服务里，AST 等大量前期分配对象本来就会长时间存在；团队也知道哪些逻辑时间点适合跑 GC。

第三，Go 允许控制 memory layout and allocation，但不要求整个代码库持续处理 memory management。这让团队可以拿到 native 实现的性能收益，又不至于把大量精力消耗在手动内存管理上。

第四，TypeScript compiler 有大量 graph processing，会在树结构里做向上、向下遍历，还会遇到 polymorphic nodes。官方认为 Go 在保持 JS 版本结构相似的前提下，写起来更 ergonomic。

当然，Go 不是没有弱点。

官方也承认，Go 的 in-proc JS interop 不如一些替代方案。只是团队认为这个弱点可以缓解，并且会设计 performant and ergonomic JS API。

这才是成熟技术选型的样子：不是把缺点藏起来，而是判断哪个缺点可控，哪个缺点会拖垮主目标。

![TypeScript native port 语言选型约束示意图](/images/typescript-go-native-port/choice-map.svg)

## 普通 TS 开发者什么时候能用？

如果你想现在动手试，方向已经有了，但别把它当稳定替代品。

当前可以看 `@typescript/native-preview`，也可以用 `tsgo` 体验 preview：

```bash
npm install -D @typescript/native-preview
npx tsgo --noEmit
```

VS Code 也有 native preview extension，README 里给过类似配置：

```json
{
  "js/ts.experimental.useTsgo": true
}
```

但重点是：preview 不是生产默认值。

typescript-go 仍然是 work in progress，尚未完全 feature parity。官方路线也不是“TypeScript 7 已经替代 TypeScript 6”，而是 JS 代码库继续走 TypeScript 6.x，native codebase 达到足够 parity 后作为 TypeScript 7.0 发布。部分项目未来也可能因为 API、legacy configuration 或其他约束继续留在 TypeScript 6。

所以普通开发者现在最合理的姿势不是立刻迁移，而是三件事：

- 在非关键项目里试 `@typescript/native-preview`，感受 `tsgo` 的类型检查速度；
- 关注 VS Code native preview extension 和 language service 的成熟度；
- 如果你的项目重度依赖 TypeScript compiler API，提前关注新 API 的兼容边界。

别急着押注，也别假装没发生。

## AI 编程时代，TypeScript 速度会变成基础设施

这次升级还有一个容易被低估的点：它不只是让人等得少一点，也会改变工具能做什么。

官方博客提到，新一代 AI tools 受益于更大的 semantic information window，而且这些信息需要更低延迟地可用。

这句话翻译到开发现场就是：AI 助手想写对代码，不能只看你当前打开的文件。它需要知道类型定义、调用关系、项目结构、错误上下文、重构影响面。TypeScript 本来就掌握这些信息，但如果取这些信息太慢，很多能力就只能降级成“猜”。

速度慢的时候，工具会保守。

速度快到足够低延迟，工具才敢实时化。

这也是为什么我不太愿意把 TypeScript 7 native 只看成一次“编译器加速”。更准确地说，它可能会把 TypeScript 语言服务推到下一层基础设施里：编辑器、重构工具、CI、AI coding tools 都会从中受益。

当然，这不是说 TS 7 一出来，AI 编程立刻质变。

工具能不能用好这些上下文，还取决于编辑器、插件、模型、API 设计和生态适配。但底座变快，本身就是条件变化。

## 最后别把这件事看成语言战争

TypeScript 选择 Go，最值得学的不是“以后大家都该用 Go 写编译器”。

这结论太粗糙。

真正值得学的是：大型工程的技术决策，很多时候不是选理论上最漂亮的东西，而是选在当前约束下最能落地、最少制造新问题的东西。

TypeScript 团队要的不是一门语言来证明审美。

他们要的是：保持行为一致，降低迁移风险，维持双代码库同步，拿到数量级性能提升，同时给未来语言服务和 AI 工具留下空间。

放在这个目标下，Go 不是“打赢了”Rust 或 C#。

Go 只是更符合这次迁移的约束。

**大型工程里，最优解常常不是最酷的语言，而是最少制造新问题的语言。**

下次再看到“某某项目为什么不用 Rust / 为什么不用 Go / 为什么不用自家语言”这种争论，别急着站队。

先问一句：它到底是在 rewrite，还是在 port？

这个问题一问，很多争论会立刻安静下来。
