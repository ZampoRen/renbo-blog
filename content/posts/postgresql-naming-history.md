---
title: "一个数据库叫过三个名字，但 Postgres 比 PostgreSQL 大 10 岁"
date: 2026-07-08T10:00:00+08:00
slug: postgresql-naming-history
categories: ["技术"]
tags: ["PostgreSQL", "Postgres", "数据库历史", "Ingres", "开源故事"]
author: Zampo
description: "一个数据库的三个名字，不是起名无能，是一段 50 年历史的编码。从 Ingres 到 Postgres 到 PostgreSQL——半世纪命名漂流背后，藏着整个数据库业的兴衰。"
draft: false
cover: "/images/postgresql-naming-history/cover.png"
---

你每天在用 PostgreSQL。公司的生产库、本地的 dev 环境、CI 管道——某个地方一定有一行 `psql -U postgres`。但你大概不知道，这个数据库叫过三个不同的名字。

2006 年，Tom Lane——PG 最核心的维护者，长期位列全球开源贡献者前十的人——在邮件列表里写了这么一句：

> "The 1996 decision to call it PostgreSQL instead of reverting to plain Postgres was the single worst mistake this project ever made."

把项目改名叫 PostgreSQL，是它历史上最大的错误。说出这句话的人不是社区喷子，是 PG 主分支上 commit 最多的开发者之一。

一个数据库因为自己的正式名字被核心成员称为 "最大的错误"？那它本来该叫什么？为什么 pg 的默认数据库叫 `postgres` 不叫 `postgresql`？为什么 `psql` 是 `psql` 不是 `pgsql`？

这条命名线横跨 50 年，串起了一个学术项目的意外成功、一次价值 4 亿美金的商业收购、两个研究生的自发拯救、一个因为太成功而不得不结束的大学课题、一个再也改不回来的名字。

Ingres → POSTGRES → Postgres95 → PostgreSQL。每一个名字的切换，都对应着一个时代的选择和妥协。而且这些痕迹至今还在你每天使用的命令里。

## 1973，Ingres：一个意外开始的数据库

这个故事不从 PostgreSQL 开始，甚至不从 Postgres 开始。

1973 年，Michael Stonebraker 和 Eugene Wong 在 UC Berkeley 读到 IBM System R 的关系数据库论文。他们觉得能做得更好，于是申请 NSF 经费，做起了自己的关系数据库。项目名叫 Ingres，全称 "Interactive Graphics Retrieval System"，经费来自 Berkeley 经济学组的一个地理数据库研究——PostgreSQL 的祖先，最初是靠给经济学家画地图拿到的钱。

Ingres 做得很成功。到 1980 年，约 1000 份副本分发到了各大学。同年 Stonebraker 和 Wong 成立商业公司 RTI，把 Ingres 推向商用。

但 Ingres 做了一个致命的技术选择：它的查询语言是 QUEL，不是 SQL。

QUEL 比 SQL 更干净——语法更接近自然语言，没有那么多嵌套和冗余。Oracle 选了 SQL，赢了。QUEL 更干净，但干净不是生态游戏的通行证。Ingres 在商业上输给 Oracle，本质上是站在了 SQL 席卷一切的洪流对面。

Ingres 后来的转卖史是数据库界最复杂的家谱：RTI → Ingres Corp → ASK Corp → Computer Associates → Ingres Corp（again）→ Actian → HCL。三十年换了七八个东家，每次换手都离技术初心更远一点。

更有意思的是 Ingres 孵化出来的一批后继者：Sybase 的创始人 Epstein 来自 Ingres 团队，Microsoft SQL Server 的代码源头是 Sybase，而 Sybase 的基因来自 Ingres。当今主流关系型数据库的半壁江山，画起族谱来都能追溯到 1973 年 Berkeley 那间办公室。

## 1986，POSTGRES：根据 Ingres 重写，不是从 Ingres 改名

1985 年，Berkeley 的 Ingres 研究项目正式结束。Stonebraker 回到 Berkeley，启动了一个新项目：POSTGRES——Post-Ingres 的缩写。

这里有一个很容易搞混的点：POSTGRES 根据 Ingres 的经验完全重写了系统，没有沿用它的一行代码。它继承了 Ingres 的设计理念，但代码库是从零开始的。POSTGRES 的目标是解决 1980 年代数据库系统的两个根本弱点：缺乏用户自定义类型系统、关系表达力不足。它的查询语言叫 POSTQUEL，仍然不是 SQL。

POSTGRES 在 1989 年发布了第一个外部版本。到 1993 年，用户社区几乎翻倍，维护负担开始压倒开发资源。

这大概是每个大学研究者的理想结局：一个因为太成功、用户太多而不得不结束的学术项目。1994 年 6 月，POSTGRES 4.2 发布，Berkeley 项目正式收尾。项目以 MIT 变体许可证发布，允许任何人使用。

同期，Stonebraker 参与的商业分支 Illustra 在 1992 年成立。1996 年，Informix 以 4 亿美金收购了这家只有 150 人的公司——40 倍收入，Stonebraker 个人分到 650 万美金。Illustra 的 DataBlade 架构后来整合进 Informix Universal Server，再过近二十年这条线被 IBM 收购，2021 年以 3.3 亿美金转到 HCL 手里。同一套技术基础，三十年间换了四次名字，最后在三体企业手里继续跑着生产负载。

从 1986 到 1994，POSTGRES 走完了一轮完整的学术项目生命周期。一个标准的结局是：论文发完，项目归档，代码逐渐没人维护。但有两个研究生不这么想。

## 1994，Postgres95：两个研究生的深夜加码

1994 年，Berkeley 研究生 Andrew Yu 和 Jolly Chen 做了一个决定：给 POSTGRES 加上 SQL 支持。

这在今天是理所当然的。1994 年不是。1994 年的 POSTGRES 还在用 POSTQUEL，而 Oracle 已经靠 SQL 站稳了脚跟，SQL 正在成为关系型数据库的事实标准。POSTGRES 一个学术界的东西，连 SQL 都不支持，几乎注定被遗忘。

Yu 和 Chen 不是 Stonebraker 的学生，不是教授指派的课题。他们就是看着 POSTGRES 的代码，觉得不能让它在学术界慢慢死去。于是用 ANSI C 重写了代码，体积减少 25%，性能在 Wisconsin Benchmark 上快了 30-50%。交互工具从古老的 monitor 换成了 psql，基于 GNU Readline，一条命令用到了今天。

他们也给项目起了一个新名字：Postgres95。

加个年份后缀，简单直接，和当时的 Postgres 区别开。没有人想过 "95 年过去了，Postgres96、Postgres97 怎么叫"。版本号从 0.01 开始——这是明确的信号：项目从这里重启了，不再沿袭 POSTGRES 的版本序列。

1995 年 9 月，Postgres95 1.0 公开发布。1996 年 7 月，Marc Fournier 提供了第一个非大学开发服务器。PG 真正脱离 Berkeley，从一个大学项目变成了社区项目。

这里有个值得讲的细节：psql 这个名字里的 "ps" 是两字母缩写，不是 "pgsql" 也不是 "pgs95sql"。当时的开发者潜意识里已经在用短的 pg- 前缀。这个习惯延续到今天——pg_dump、pg_restore、pgAdmin、pgvector，没有人想过要改成 "postgresql-dump"。

PG 的版权声明至今还写着 "Portions Copyright © 1994, The Regents of the University of California"。二十多年了，Postgres95 的代码遗产仍然留在每个 PG 发行版的许可证里。

## 1996，PostgreSQL：那个再也改不回去的名字

Postgres95 的版本号危机比任何人预想来得都快。

1996 年，社区意识到 "Postgres95" 在 1997 年就会显得尴尬。官方文档的原话："Postgres95 would not stand the test of time。"

决定改名。新名字要兼顾两件事：承认 SQL 能力，同时标记与原始 POSTGRES 的血脉关系。最后落到 PostgreSQL 上——Postgres 加 SQL。

版本号做了一个极富象征意义的操作：从 Postgres95 1.0 直接跳到 6.0，跳过 2.0、3.0、4.0、5.0 所有中间版本。这个跳跃的本质是一种尊严声明——PostgreSQL 是 POSTGRES 的合法继承人，要把版本序列接回 Berkeley 的 4.2。

1996 年 10 月，postgresql.org 域名上线。1997 年 1 月，PostgreSQL 6.0 发布。

然后就是 Tom Lane 在 2006 年那封邮件。十年之后回头看，他觉得把名字改长是最大错误。理由很直接：这个名字太拗口了，Postgres 作为缩写反而成了日常主流，PostgreSQL 只在正式文档里才会被完整写出来。每次跟人介绍都要解释半天，社区里永远有新人问 "为什么叫 PostgreSQL 又简称 Postgres"——问题本身就是改名造成的。

2007 年，PG 核心团队重新讨论了改名方案：趁 11 年还不算太晚，能不能把正式名改回 Postgres？最终决定保留 PostgreSQL 作为正式名，同时承认 Postgres 为官方别名，两条线并行，再也不改了。法语和日语社区的强烈反对是改不回的重要原因——两个语言社区已经围绕 PostgreSQL 这个全名建立了大量文档和社区习惯，改名的迁移成本已经高到无法承受。

到今天，PG 的许可协议页面第一行写着：

> "PostgreSQL Database Management System (also known as Postgres, formerly as Postgres95)"

一个数据库，三个名字，被刻在同一句话里。

`psql` 不带参数连接时，默认连到的数据库就是 `postgres`。官方说 "also known as Postgres"，默认的数据库就叫 `postgres`——这已经不只是文档层面的事儿了。

## 官方承认的别名，不是偷懒的缩写

所以，Postgres 到底是不是 PostgreSQL 的缩写？

很多人这么以为。但 Postgres 是 1986 年的项目原名，比 PostgreSQL 这个名字大 10 岁。官方文档写得很直接："Postgres is still considered an official project name, both because of tradition and because people find it easier to pronounce Postgres than PostgreSQL。"

两个名字都是 PostgreSQL Community Association of Canada 的注册商标，地位相同。Postgres 在商标框架里是官方认可的别名，和 PostgreSQL 平行，不是它的缩略版。

下一次有人在评审会上纠正你 "应该叫 PostgreSQL 不是 Postgres"，你可以把这篇转发给他。不是抬杠——Postgres 作为名字是 1986 年就定下的，比 1996 年的 PostgreSQL 早整整十年。

一条命名线索了 50 年的兴衰。从 Berkeley 办公室里给经济学家画地图的经费起步，到两个研究生自发敲出 1.0 版，到 Tom Lane 说的 "最大错误"，再到许可协议页面上三个名字排在一起的总结——数据库的三个名字本质不是起名的问题，是一段历史的编码，打在了你每天都会敲的那行命令里。
