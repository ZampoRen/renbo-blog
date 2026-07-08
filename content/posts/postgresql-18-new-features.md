---
title: "PostgreSQL 18 来了，这次真能快 3 倍"
description: "异步 I/O、UUIDv7、跳过扫描、虚拟生成列——PG 18 是近年来变化最大的一个版本。值得升级吗？"
date: 2026-07-08T10:00:00+08:00
draft: false
author: "Zampo"
tags: ["PostgreSQL", "数据库", "PG18", "AIO", "UUIDv7", "性能"]
categories: ["数据库"]
cover: "/images/postgresql-18-new-features/cover.png"
toc: true
---

PostgreSQL 18 去年 9 月就发布了，但我发现身边很多人还没升。

问了一圈，理由差不多：PG 17 用得好好的，升它干嘛？

也对，如果不看 release notes，确实不知道 PG 18 改了什么。

但 PG 18 跟之前那些小步迭代不太一样——这次有几个改动，值得你认真考虑要不要升。

## 最大的那个：异步 I/O

PG 一直有个老问题：读数据的时候，CPU 得等磁盘。

顺序扫描等、bitmap 扫描等、vacuum 也要等。每个 I/O 请求发出去，进程就挂起，等数据回来了再接着跑。这在机械硬盘时代是常识，但在 NVMe SSD 和傲腾面前，CPU 等磁盘的时间越来越浪费。

PG 18 引入了异步 I/O 子系统。简单说：不再一个个发请求等返回，而是一次发一堆，谁先回来谁先处理。

![异步 I/O 原理对比](/images/postgresql-18-new-features/inline-01.png)

官方数据：在某些场景下，顺序扫描性能提升 **3 倍**。不是 30%，是 3 倍。

怎么开？加一行配置：

```
io_method = 'io_uring'
```

`io_uring` 是 Linux 5.1+ 的高性能异步接口。如果内核版本不够，也可以用 `worker`（走工作线程模拟异步），差一点但也比 `sync`（老模式）强。

**谁受益最大：** 顺序扫描多的场景（数仓、ETL、全表扫描报表）、大表 vacuum、bitmap 扫描。

## UUIDv7：终于不用纠结主键怎么选了

UUID 做分布式主键是好东西，全球唯一、不依赖自增。

但 UUIDv4 是纯随机的，写入 B-tree 索引的时候会疯狂分裂，性能惨不忍睹。好多团队因为这个原因又退回自增 ID 了。

PG 18 原生支持 UUIDv7，它是时间戳 + 随机数组合的。时间戳部分保证新生成的 UUID 大致有序，B-tree 索引写入不再分裂。

```sql
-- PG 18 直接用
SELECT uuidv7();

-- 旧写法 gen_random_uuid() 现在可以用 uuidv4() 代替
SELECT uuidv4();
```

这事的价值不在于"多了一个函数"，在于**你可以用 UUID 做主键了，同时不会牺牲写入性能**。

对分布式系统、多数据中心同步、微服务间数据交换的场景，这是个实在的利好。

## B-tree 跳过扫描：一个索引覆盖更多查询

有个经典场景：建了 `(a, b, c)` 复合索引，查询是 `WHERE b = 1`。因为没用到 a 列做筛选，优化器不走索引，只能扫全表。

以前的做法是多建一个 `(b)` 索引。但索引多了写入慢、占空间。

PG 18 的跳过扫描能自动"跳过"不参与筛选的前缀列，直接扫描匹配 b 列的部分。效果相当于一个复合索引能覆盖更多查询模式。

![B-tree 跳过扫描工作原理](/images/postgresql-18-new-features/inline-02.png)

对 OR 条件的查询也有优化——以前 OR 很难用到索引，现在优化器能把它转化成索引可用的形式。

**谁受益最大：** 复合索引多、查询模式多样、不能给每种查询都建独立索引的生产系统。

## 虚拟生成列：省空间

生成列之前只能存下来（stored），占空间。PG 18 支持虚拟生成列——值在读取时实时计算，不占存储空间。

```sql
CREATE TABLE users (
    first_name text,
    last_name text,
    full_name text GENERATED ALWAYS AS (first_name || ' ' || last_name) VIRTUAL
);
```

而且虚拟列现在是默认行为，不写 `STORED` 或 `VIRTUAL` 就是虚拟的。

## pg_upgrade 升级体验大改

以前大版本升级最大的痛：统计信息不保留。

升完级，优化器对数据分布一无所知，得重新跑 ANALYZE。在几十 TB 的库上，ANALYZE 跑完之前，查询性能是崩的。很多团队因为这个原因不敢升级。

PG 18 改了：**统计信息在升级时保留**。升完级，查询性能基本无缝过渡。

另外 pg_upgrade 加了个 `--swap` 模式，不再需要 copy/clone/link 数据文件，直接交换目录，升级时间大幅缩短。

```
pg_upgrade --swap --jobs=4
```

## 体验改善的小东西

- **EXPLAIN ANALYZE 默认显示 BUFFERS** — 以前要手动加 `(BUFFERS)`，现在默认就有
- **页面校验和默认开启** — 新 initdb 的集群自动启用，数据完整性更有保障
- **RETURNING 支持 OLD/NEW** — INSERT/UPDATE/DELETE/MERGE 都能取到变更前后的值
- **OAuth 认证** — 对接 SSO 不再需要额外插件
- **Wire Protocol 3.2** — 2003 年以来第一次协议版本升级，为后续功能铺垫
- **vacuum 主动冻结** — 减少紧急冻结触发的几率

![升级决策清单](/images/postgresql-18-new-features/inline-03.png)

## 要不要升？

说实在的，PG 18 最大的升级价值在 AIO。如果你的业务是 IO 密集型的——大数据量的顺序扫描、频繁的 vacuum、大量 bitmap 扫描——升了就立竿见影。

UUIDv7 和跳过扫描是长期改善，不会让你升完当天就看见效果，但在这个版本上做的表结构设计，两三年后回头看会觉得当时选对了。

升级本身比以前安全了：统计信息保留，pg_upgrade 带 `--swap` 速度快得多。但大版本升级不是没有风险的，建议先在从库或测试环境跑一轮。

———

这篇不是教程，是判断框架。四个最能打的功能：**AIO（性能）、UUIDv7（主键设计）、跳过扫描（索引效率）、升级体验（运维成本）**。四个都值得，看你最痛哪个。
