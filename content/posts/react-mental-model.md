---
title: "每个前端都在用 React，但很多人把它学反了"
description: "你学了 React，但可能学反了。不是你不会写 useState、useEffect，而是你一直没想明白：React 到底在解决什么问题。React 最重要的不是那堆 API，而是它让 UI 变成了一件可以被推理的事。"
date: 2026-04-14T15:00:00+08:00
draft: false
slug: "react-mental-model"
tags: ["前端", "React", "JavaScript", "UI", "状态管理"]
categories: ["技术"]
cover:
  image: "/images/react-mental-model/cover.svg"
  alt: "React 心智模型"
---

![React 心智模型](/images/react-mental-model/cover.svg)

你学了 React，但可能学反了。

不是你不会写 `useState`、`useEffect`，而是你一直没想明白：React 到底在解决什么问题。

很多前端工程师都是这样：能用 React 写页面，但说不清它和 jQuery 的根本差异在哪。

<!--more-->

## React 为什么会出现

React 是 Facebook 在 2013 年推出的。

但 React 真正要解决的，不是"让页面更炫"，而是大型、有状态的 Web 应用越来越复杂的问题。

在 React 流行之前，前端大量依赖命令式 DOM 操作。

你要手动找节点、改文本、绑事件、同步状态。代码一多，UI 的更新路径会越来越乱：哪个地方改了数据，哪个节点该更新，谁先谁后，都容易失控。

**React 提出的核心想法很简单：**

界面应该是应用状态的函数。

也就是说，你不应该一条条命令式地告诉浏览器"去改哪个节点"，而是应该先描述：在当前状态下，界面应该长什么样。

React 再负责把真实 DOM 调整成这个结果。

**React 最厉害的地方，不是造出了一堆 hook，而是把前端界面重新变成了函数问题。**

![组件是函数](/images/react-mental-model/inline-01.svg)

## 组件到底是什么

很多初学者以为 React component 是某种模板、类或者控件。

**组件本质上是一个函数。**

这个函数接收输入（props），返回 UI 描述。注意，返回的不是浏览器真实 DOM，而是 React element——一个轻量级 JavaScript 对象，用来描述"UI 应该是什么"。

React 在 render 时会调用这个函数，得到一棵元素树；之后再由 React DOM 把它高效地映射到浏览器里的真实 DOM。

这个视角带来几个直接好处：

- 组件更容易理解：它像普通函数一样工作
- UI 更容易组合：复杂界面可以由多个小组件拼起来
- 输入输出关系更清楚：相同输入应该得到相同描述结果

## JSX 是什么，不是什么

很多初学者第一次看到 JSX，会误以为 React 发明了一种奇怪的新模板语言。

**JSX 只是语法糖，本质上还是 JavaScript。**

JSX 的存在，只是为了让嵌套的 UI 结构更好写、更好读。它会被编译工具转成普通 JavaScript 调用。

更准确的说法：

- JSX 不是 DOM
- JSX 也不会直接创建 DOM 节点
- JSX 只是用更直观的写法生成 React element

一旦理解 JSX 只是表达层语法糖，读者对 React 整体心智会轻很多。

## 虚拟 DOM 的正确位置

React 最容易被神化的概念之一就是虚拟 DOM。

很多文章会把它写成 React 成功的唯一秘诀，但它只是实现细节，不是学习 React 的入口。

每次 React render，都会生成一棵元素树。它不是浏览器 DOM 的完整镜像，而是一种用 JavaScript 对象表示 UI 的模型。

当状态变化时，React 会：

1. 重新执行组件函数
2. 得到一棵新的元素树
3. 对比新旧差异（reconciliation）
4. 只把真正变更的部分提交到真实 DOM

**React 的优化重点，不是阻止你重新计算 UI，而是把最终真正要动浏览器的部分控制住。**

## State：React 怎么记住东西

React 的 state，本质上是组件在多次 render 之间保留下来的数据。

它和普通局部变量最大的区别是：普通变量不会跨 render 保留，而 state 会。

当你调用 `setState` 时，React 并不是就地修改某个旧界面，而是安排一次新的 render。

组件函数重新运行，React 把更新后的 state 交给它，再生成一份新的 UI 描述。

**React 更像是在不断生成"新快照"，而不是原地改旧界面。**

## Render 和 Commit 不是一回事

很多人会把"React 组件执行了"直接等同于"页面已经改了"。

其实不是。

**Render 是算账，Commit 才是付款。**

- **Render：** 计算当前 UI 应该长什么样
- **Commit：** 把计算结果真正应用到浏览器 DOM，并执行 effect

这个区分特别重要，因为很多人会误以为 render 阶段页面就已经更新了。

其实 render 阶段只是准备，commit 阶段才是落地。

## Hooks 该怎么理解

视频对 hooks 的解释比很多教程更干净：

**hooks 不是魔法 API，而是 React 用来把持久状态和副作用，绑定到组件执行位置上的机制。**

最需要理解的两个 hook：

- `useState`：让组件拥有跨 render 记忆
- `useEffect`：在 React 提交 DOM 之后，与 React 控制之外的系统同步

### effect 不是默认工具

很多人把 effect 当成默认工具：

```jsx
useEffect(() => {
  // 数据获取
  // 事件监听
  // 定时器
  // 一切逻辑
}, []);
```

**effect 不是 React 里干任何事的地方，而是用来处理副作用的出口。**

什么算副作用？

- 网络请求
- 浏览器 API
- 订阅/取消订阅
- 与外部系统同步

很多本来应该通过状态、props、派生计算解决的问题，被新手错误地塞进了 effect。

结果：组件越来越难维护，依赖数组永远调不对。

## 列表 Key：不是让 React 不报 warning 的门票

很多人只把 key 当成"React 不报 warning 的门票"，用 index 随便填一个。

**key 的意义不只是性能，更是 identity。**

它告诉 React：这次 render 里的这个元素，和上次 render 里的哪个元素是"同一个东西"。

如果 key 不稳定，React 就可能搞错对应关系，导致：

- 组件状态被错误复用
- 列表局部更新异常
- 表单项错位

## Context：不是万能状态管理

React Context 的定位也被讲得很克制。

它确实可以跨组件层级传值，减少 props 一层层往下传的麻烦。

**但它不应该被理解成"React 自带的通用状态管理器"。**

更适合 context 的场景，是低频、稳定、具有全局环境意味的数据：

- 主题
- 语言/本地化
- 登录状态
- feature flags
- 环境配置

如果把高频变化的数据也一股脑塞进 context，很容易造成不必要的 rerender，组件也会变得更耦合。

**props 管显式数据流，context 管环境式依赖。**

![React 心智模型总结](/images/react-mental-model/inline-02.svg)

## 真正难的是什么

真正难的不是学会 JSX，而是接受"UI 只是状态的结果"这件事。

不是记住多少 hook API，而是理解组件、状态、render/commit、effect 的边界。

不是把虚拟 DOM 神化成唯一亮点，而是理解它只是实现细节。

**React 最重要的不是那堆 API，而是它让 UI 变成了一件可以被推理的事。**
