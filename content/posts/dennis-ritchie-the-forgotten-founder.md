---
title: "Dennis Ritchie：被乔布斯光芒盖住的奠基人"
description: "2011年10月，世界失去了两个人。一个改变了你用什么设备，一个改变了你怎么写代码。后者的讣告，几乎无人知晓。"
date: 2026-07-07T00:00:00+08:00
draft: false
author: "Zampo"
tags: ["Dennis Ritchie", "C", "Unix", "编程语言", "程序员故事"]
categories: ["科技叙事"]
cover: "/images/dennis-ritchie-the-forgotten-founder/cover.png"
toc: false
---

2011 年 10 月 5 日，Steve Jobs 去世。

全世界的新闻头条、电视直播、社交媒体，都在报道他。果粉在 Apple Store 门口摆满鲜花，人们在 Twitter 上刷屏悼念。那是一个时代巨人的谢幕，每个人都觉得自己见证了一段历史的终点。

七天后的 10 月 12 日，Dennis Ritchie 在新泽西 Berkeley Heights 的家中去世。他一个人住，被发现时已经走了好几天。

没有鲜花，没有刷屏，没有电视直播。大多数程序员是好几个星期后才知道这个消息。主流媒体只给了几段简讯。

但 Ritchie 创造的东西——C 语言和 Unix 操作系统——你今天还在用。

你写完代码，**go build**，**./server**。你用的 Go 工具链，最终是用 C 写的编译器编译出来的。你部署到的 Linux 服务器，内核大约 98% 是 C 代码。你调试用的 Python，解释器是 C 写的。你跑的 nginx，是 C 写的。你打开 macOS 终端，里面跑的是 Unix 的直系后代。

你站在一个巨大的层次结构上。每一层，都回溯到 1972 年贝尔实验室里那个男人的工作台。

## 他没有博士学位

Dennis MacAlistair Ritchie，1941 年出生于纽约 Bronxville。父亲 Alistair 是贝尔实验室的科学家，合著过开关电路理论的教科书。Dennis 在 Summit 长大，高中毕业后进了哈佛。

他读的是物理学和应用数学。1967 年，他跟着父亲的脚步进了贝尔实验室。1968 年，他在 Patrick Fischer 指导下完成了一篇博士论文——"Computational Complexity and Program Structure"。论文写完了，答辩也过了，但他从未正式拿到博士学位。

不是因为论文不行。是因为他根本没去走最后那道程序。计算机历史博物馆直到 2020 年才找回他那篇"失落的博士论文"。

他没有博士学位，但这不妨碍他后来拿到图灵奖。

图灵奖是计算机科学界的诺贝尔奖。1983 年，Ritchie 和 Ken Thompson 共同获奖，表彰他们"发展了通用操作系统理论并具体实现了 Unix 操作系统"。

## C 语言是怎么来的

这段故事在程序员圈子里流传很广，但因为太重要，值得再说一遍。

1960 年代末，贝尔实验室的 Ken Thompson 和 Dennis Ritchie 正在开发一套叫 Multics 的分时操作系统。项目流产了，但 Thompson 不甘心。他在一台被公司淘汰的 PDP-7 上写了一个简化版系统，这就是 Unix 的雏形。

最早的 Unix 内核是用汇编写的。汇编的问题是：换个 CPU 架构，整套代码就要重写。Thompson 和 Ritchie 想要一种"高一点"的语言——既能像汇编那样直接操纵内存和硬件，又能在不同机器之间移植。

他们试过 Fortran，不行。Thompson 写了一个叫 B 的语言，继承自剑桥的 BCPL。但 B 是解释型语言，性能不行，表达力也不够。

于是 Ritchie 在 B 的基础上做了一件事：加类型系统，改成编译型。

1972 年，C 语言诞生了。

C 不是从课堂里设计出来的语言。它是为了解决一个具体问题而长出来的——给 Unix 写一套可移植的内核。Ritchie 和 Thompson 用 C 重写了 Unix，从此操作系统不再只能和特定硬件绑定。

"他们写 C，是为了写一个程序，"Ritchie 的同事 Rob Pike 后来回忆说，"那个程序就是 Unix 内核。"

这个决定有多关键？

1970 年代之前，操作系统基本上是一种"硬件附属品"。换一台机器，操作系统就要重写。Unix + C 的组合打破了这一点：只要有一台机器的 C 编译器，Unix 就能移植过去。

这个模式统治了后续 50 年的系统软件。

## 为什么你今天绕不开他

衡量一个人的影响，最简单的办法是看：你离得开他创造的东西吗？

你做不到。

C 语言是今天地球上使用最广泛的系统级语言。TIOBE 编程语言排行榜上，C 在过去 40 多年里一直稳居前 3。这不是怀旧，不是"老程序员的情怀"。Linux 内核用 C，Windows 内核曾经用 C，macOS 和 iOS 的内核（XNU）用 C 和 C++，Android 的底层运行时用 C。几乎每一块网络硬件、路由器、交换机、嵌入式设备，固件都是用 C 写的。

Python 的解释器是 C（CPython）。Ruby 的解释器（CRuby）是 C。PHP 的解释器是 C。Node.js 的底层引擎 V8 大量用 C++，但 V8 能运行的系统环境仍然是 C 构筑的。Redis 是 C。nginx 是 C。SQLite 是 C。你日常开发离不开的每一层基础设施，要么直接用 C 写的，要么运行在 C 写的东西之上。

WIRED 在 Ritchie 去世后的报道里引用了 Rob Pike 的一段话，我读了很多遍：

"Pretty much everything on the web uses those two things: C and UNIX. The browsers are written in C. The UNIX kernel — that pretty much the entire Internet runs on — is written in C. Web servers are written in C, and if they're not, they're written in Java or C++, which are C derivatives, or Python or Ruby, which are implemented in C. And all of the network hardware running these programs I can almost guarantee were written in C."

另一个引述来自 MIT 的 Martin Rinard：

"Jobs was the king of the visible, and Ritchie is the king of what is largely invisible."

两句话，把整件事说完了。

## 不是比较，是排序

我必须说清楚：这不是一篇贬低乔布斯的文章。

Jobs 改变了消费电子产品的形态。没有他，iPhone 和它开启的智能手机时代可能会晚很多年才到来——或者以完全不同的面貌到来。他对世界的贡献不需要 Ritchie 的光环来衬托。

值得反思的不是"谁更伟大"，而是"当两个分量如此接近的人在同一周离开，公众的注意力分配为什么如此悬殊"。

一个人改变了你用什么设备。另一个人改变了你怎么写代码。

世界为前者的离去而集体悲痛。对后者，大多数人甚至没听说过他的名字。

这不是谁的错。这本来就是这两种创新的本质差别。消费电子产品的创新是可见的、可触摸的、可以放在口袋里每天使用的。而系统软件的创新是不可见的——你不需要知道 C 语言是谁写的也能写出好程序，就像你不必知道混凝土是谁发明的也能住进大楼。

但这不意味着发明混凝土的人不重要。

## 安静的传奇

Ritchie 生前是什么样的人？几乎所有认识他的人都用同一个词描述：quiet。

他性格温和，不争不抢，在贝尔实验室同一座大楼里工作了将近 40 年。Rob Pike 说他们在走廊里做了 20 年邻居，Ritchie 从来不是一个喜欢出风头的人。他甚至从不在公开场合为自己的贡献辩护。

2011 年初，Ritchie 还拿到了日本奖（Japan Prize）——这在日本相当于诺贝尔奖级别的荣誉。同年 10 月他就走了。

他去世后，贝尔实验室的官网发了一篇简短的悼念文章。Hacker News 上很多人自发转发，Reddit 的编程板块出现了一个帖子："Dennis Ritchie has died"。但和 Jobs 去世时的全球性哀悼相比，这份关注几乎可以忽略不计。

这不是冷漠——是因为他创造的东西已经太融入日常，以至于人们根本意识不到它在那里。就像你不会每天感谢发明了开关的人，但你每天都在用开关。

## 回到你的命令行

写这篇文章的初衷，不是要写一篇悼念文。

距离 2011 年已经过去十五年。Ritchie 已经离开我们很久了。这中间，Rust 成了系统编程的新希望，Go 在云原生领域站稳了脚跟，AI 在改写"编程"这件事本身的定义。

但 Ritchie 留下的遗产没有被取代。

Rust 的编译器需要 C 运行时。Go 的编译器最初是用 C 写的。Linux 和 Windows 依然用 C。每一个新兴技术的底层，那些内存管理、系统调用、硬件交互，仍然在用 C 或 C 的思维模式在写。

下次你打开终端，敲下一个命令，二进制文件被执行——这一刻的每一次操作，都发生在 Ritchie 参与构建的层次之上。

这不是伤感，这是一个事实。

你不需要为他悲伤。你只需要知道：有一个安静的男人，花了一生时间，在一个实验室里写了一门语言和一个系统。这门语言和这个系统，成了你今天写代码时脚下那块看不见的地面。

他的名字叫 Dennis Ritchie。

同一周离开的两个传奇，一个改变了你用什么设备，一个改变了你怎么写代码。

现在你知道了。
