---
title: "PostgreSQL 真正厉害的，不是性能，而是它把最后一道权限判断留在了数据库里"
description: "我越来越觉得，很多系统真正失守的地方，不在浏览器、不在 API 网关，而在数据库这一跳。PostgreSQL 最值得后端团队认真学的，不是某几个安全功能，而是它真的给了你把最后一道权限判断收回数据库内部的能力。"
date: 2026-04-17T09:33:20+08:00
draft: false
slug: "postgresql-last-permission-check"
author: "任博"
tags: ["PostgreSQL", "数据库安全", "后端开发", "RLS", "RBAC", "权限设计"]
categories: ["技术"]
featured: true
toc: true
cover:
  image: "/images/postgresql-last-permission-check/cover.svg"
  alt: "PostgreSQL 最后一道权限判断示意图"
video_source: "https://www.youtube.com/watch?v=S_Z8Y0vMSzo"
---

![PostgreSQL 最后一道权限判断示意图](/images/postgresql-last-permission-check/cover.svg)

我越来越觉得，很多团队的安全问题，不是前面没做，而是最后收得不够。

浏览器有隔离，接口有鉴权，服务上了 HTTPS，机器放在内网里，日志和审计也配了。可等请求真的落到 PostgreSQL，后端连过去的，还是一个几乎什么都能做的高权限角色。

前面一层一层都像样，最后这一跳却像没上锁。

这也是我最近看 PostgreSQL 安全这条视频时，最有感触的一点：PostgreSQL 真正厉害的，不是它默认很安全，也不只是它功能多，而是它把最后一道权限判断留在了数据库里。

<!--more-->

我想先把结论说在前面。

**PostgreSQL 的安全，不是它默认很安全，而是它允许我把权限真正收回数据库内部。**

这句话为什么重要？因为很多团队今天的做法，本质上还是：应用层负责判断，数据库负责执行。只要应用层说“这个请求合法”，数据库就照做。

这样做不是完全不行。问题在于，它太依赖应用层永远不犯错。

而真正的系统，不会给自己这种幻想。

## 我为什么越来越不信“一把 admin 走天下”这套做法

如果你做过后端，下面这个场景大概率不陌生。

服务启动时，用一个数据库账号连上 PostgreSQL。后面所有请求都通过这个连接池访问数据库。用户在应用层已经登录过了，也做了接口鉴权、菜单权限、RBAC，甚至连一些字段可见性都在服务层处理了。

看起来挺完整。

但数据库自己知道这些吗？

很多时候并不知道。

对 PostgreSQL 来说，真正连进来的还是同一个角色。它看到的不是“张三正在看自己的订单”，而只是“某个高权限角色正在查 orders 表”。

这意味着什么？

意味着只要应用层哪天写错一段逻辑、漏掉一次校验、放过一条本不该过的 SQL，数据库层就很可能没有第二道门。

我现在越来越不认同这种做法。不是因为它不方便——它太方便了，方便到很多团队根本懒得再往下想。账号少、代码简单、连接池好复用、性能还不错，几乎每个理由都说得通。

问题是，安全这件事最怕的就是“理由都说得通”。

真正出事的时候，没人会因为你当初少建了几个 role、少写了几条 policy，就觉得这次事故情有可原。

**数据库不是存储终点，它也应该是权限边界。**

## PostgreSQL 值得学的，不是“功能大全”，而是这套收口思路

我不太想把这篇写成 PostgreSQL 安全功能导览。那种文章通常很完整，但也很难留下判断。

我更想讲清楚的是，PostgreSQL 为什么值得后端团队认真学。

在我看来，真正值钱的不是某一个单点特性，而是它给了我一套一层层往里收的工具：

- role：先定义谁在进门
- schema 和 `search_path`：先定义他能看到哪一片区域
- table / column 级 `GRANT`：再定义他能碰哪些对象、哪些字段、哪些动作
- RLS：就算前面都放行了，到了某一行，还能再判一次
- `pg_hba.conf`、`pgaudit`：再把连接来源和审计收进来

这套东西最让我服气的地方，不是它多，而是它不偷懒。

它不默认信任应用层已经做过的判断。请求进到数据库边界，就重新按数据库自己的规则再验一遍。

我觉得这才是 PostgreSQL 安全设计真正高级的地方。

![从应用层到 PostgreSQL 行级数据的权限收口路径](/images/postgresql-last-permission-check/inline-01.svg)

## 第一件该做的事：别再把 schema 只当分类目录

很多人对 schema 的理解，停在“给表分文件夹”。我以前也差不多这么看。

后来我越来越觉得，schema 真正值钱的地方，是它能帮我先把边界划开。

如果是一套正常的业务系统，我会先把数据按安全级别拆开。比如：

- `public`：用户侧业务会直接接触的数据
- `private`：账号、密码哈希、session、第三方授权信息
- `worker`：后台任务、异步作业、内部服务专用表

像这样：

```sql
CREATE SCHEMA public;
CREATE SCHEMA private;
CREATE SCHEMA worker;

REVOKE ALL ON SCHEMA private FROM PUBLIC;
REVOKE ALL ON SCHEMA worker FROM PUBLIC;
```

然后只把该给业务角色看到的 schema 开出去：

```sql
CREATE ROLE app_visitor NOLOGIN;
CREATE ROLE app_backend NOLOGIN;
CREATE ROLE internal_worker NOLOGIN;

GRANT USAGE ON SCHEMA public TO app_visitor;
GRANT USAGE ON SCHEMA public, private TO app_backend;
GRANT USAGE ON SCHEMA worker TO internal_worker;
```

再进一步，我会把默认可见范围也收一下：

```sql
ALTER ROLE app_visitor SET search_path = public;
ALTER ROLE app_backend SET search_path = public, private;
ALTER ROLE internal_worker SET search_path = worker, public;
```

这么做的价值很实际。

如果某个角色根本不需要知道 `private` 里有什么表，那我最希望的状态不是“应用层别去查”，而是“数据库里压根就别让它看见”。

这不是形式主义。

这是在把错误操作的空间提前砍掉。

**schema 不是拿来摆整齐的，是拿来划边界的。**

## 第二件该做的事：权限别按表发，要按字段发

我见过太多系统，接口层写得很细，数据库层却是整表整列放开。

这通常不是因为团队不懂安全，而是因为按字段设计权限太麻烦。可数据库安全这件事，最怕的就是“嫌麻烦”。

比如一个 `users` 表里，很可能同时有这些字段：

- `id`
- `name`
- `membership_level`
- `openid`
- `unionid`
- `password_hash`
- `session_token`

这些字段都和用户有关。但在我看来，“和用户有关”跟“用户就应该能看”完全不是一回事。

我会怎么做？

第一步，先把真正给用户看的字段单独发出去：

```sql
GRANT SELECT (id, name, membership_level)
ON TABLE public.users
TO app_visitor;
```

第二步，把不该由用户角色修改的字段锁死：

```sql
REVOKE UPDATE (membership_level)
ON TABLE public.users
FROM app_visitor;
```

第三步，把只给后端程序使用的敏感字段放进独立表，独立 schema，再单独给后端角色权限：

```sql
CREATE TABLE private.user_identities (
  user_id bigint PRIMARY KEY,
  openid text,
  unionid text
);

GRANT SELECT (user_id, openid, unionid)
ON TABLE private.user_identities
TO app_backend;
```

如果你问我这里最重要的判断是什么，我会说就是这一句：

**字段和用户有关，不等于用户就应该能看。**

这句话听起来像常识，真正做到的团队并不多。

尤其是现在很多系统一边喊最小权限，一边又把一堆“反正以后可能会用到”的字段默认开放。平时没事，看不出问题；一旦应用层有洞，这些顺手给出去的权限，立刻就会变成事故入口。

所以如果你真要开始做 PostgreSQL 安全，我建议你不要先想着 RLS，也不要一上来就谈零信任。

先把字段权限收干净。

这是成本最低、收益也最明显的一步。

## 第三件该做的事：把“这行数据到底给不给看”收回 RLS

在 PostgreSQL 这套能力里，我最看重的是 RLS。

因为前面的 schema、`GRANT`、列权限，更多还是静态授权。你给了某个角色某些权限，只要不撤销，它们就一直在那里。

RLS 不一样。

它解决的是更细的一件事：

这个角色今天、这次、对这一行，到底该不该放行。

我拿订单场景举个最典型的例子。

假设我有一张 `orders` 表，我想要的规则是：

- 买家只能看和修改自己的订单
- 卖家可以看与自己相关的订单，但不能改
- 其他人就算连的是同一个业务角色，也不该看到无关订单

如果全写在应用层，当然也能做。但我现在更倾向于把最后一步收回数据库。

先开 RLS：

```sql
ALTER TABLE public.orders ENABLE ROW LEVEL SECURITY;
```

再定义策略：

```sql
CREATE POLICY orders_select_policy
ON public.orders
FOR SELECT
USING (
  buyer_id = current_setting('app.user_id')::bigint
  OR seller_id = current_setting('app.user_id')::bigint
);

CREATE POLICY orders_update_policy
ON public.orders
FOR UPDATE
USING (
  buyer_id = current_setting('app.user_id')::bigint
);
```

应用层接到请求时，把用户上下文注入数据库会话：

```sql
SET app.user_id = '10086';
SET ROLE app_visitor;
```

这时候，同样是 `app_visitor`，PostgreSQL 也不会傻乎乎地直接全放。它会在每次访问某一行订单时，再判断当前用户 ID 和这行数据里的 `buyer_id`、`seller_id` 到底配不配。

配，就过。

不配，就拦。

这就是我为什么觉得 RLS 特别值钱。

它把“这条数据到底给不给你”这件事，从应用代码里一堆散落的 if/else，收回到了数据库自己手里。

我会用一句更好记的话来概括：

**`GRANT` 决定你有没有钥匙，RLS 决定这扇门这次开不开。**

这句不是修辞，真的是机制差别。

如果前两层解决的是“你大体能进到哪”，那 RLS 解决的就是“这一行你这次到底能不能碰”。

![三段式角色结构与 RLS 接管后的业务访问流](/images/postgresql-last-permission-check/inline-02.svg)

## 第四件该做的事：别走极端，角色设计要兼顾性能

写到这里，很多人会自然想到一个问题：那我是不是应该给每个用户都建一个 role？

理论上最细。

工程上通常不划算。

原因也不神秘。很多数据库性能节点和 role 是绑在一起的。连接池复用、会话复用、缓存命中，很多时候都建立在“同类连接反复使用”这个前提上。你把角色切得太碎，连接池就容易被打散，缓存也更难有效复用。

这也是为什么很多项目最后会滑向另一个极端：为了图省事，干脆一直用一个 admin 角色。

这个极端我也不认。

在我看来，比较像样的做法，是把角色拆成三层：

1. 连接角色：只负责连数据库和连接池
2. 业务角色：真正参与 `search_path`、`GRANT`、RLS
3. 管理角色：只做初始化、迁移、运维，不参与日常业务流量

比如：

```sql
CREATE ROLE connector LOGIN PASSWORD '***';
GRANT CONNECT ON DATABASE appdb TO connector;

CREATE ROLE app_visitor NOLOGIN;
GRANT USAGE ON SCHEMA public TO app_visitor;
GRANT SELECT ON TABLE public.orders TO app_visitor;

CREATE ROLE app_admin LOGIN PASSWORD '***';
GRANT ALL PRIVILEGES ON DATABASE appdb TO app_admin;
```

实际运行时，我会让应用先用 `connector` 连进来，再根据请求上下文切到业务角色：

```sql
SET app.user_id = '10086';
SET ROLE app_visitor;
SELECT * FROM public.orders;
```

这套东西没有“一把 admin 走天下”那么爽，但它至少把三件原本混在一起的事拆开了：

- 连接职责
- 业务权限职责
- 管理职责

一旦拆开，边界就开始清楚。

我自己现在越来越倾向这种设计。不是因为它最优雅，而是因为它比较像一个团队能长期维护的方案。

## 第五件该做的事：把连接来源和审计补上，别只盯业务权限

如果只盯 role、schema、grant、RLS，这篇还差最后一块。

因为数据库安全不只是“查的时候怎么拦”，还包括“谁能连进来”和“出了事能不能回看”。

所以我的建议是，真正落地时至少再补两件事：

第一件，收 `pg_hba.conf`。

把来源 IP、认证方式、允许连接的网段先收住。不要把数据库暴露给一切本不该来的连接。

示意配置像这样：

```conf
# 只允许应用服务所在网段访问 appdb
host    appdb    connector    10.10.0.0/24    scram-sha-256

# 管理员只能从堡垒机或运维网段连入
host    all      app_admin    10.10.99.0/24   scram-sha-256
```

第二件，补审计。

如果你的系统对数据读写安全真的敏感，我建议不要只靠应用日志。数据库侧也应该保留足够的信息。

比如启用 `pgaudit`，把关键对象访问、DDL、角色切换这类动作记下来。思路上至少要做到：

- 谁连进来了
- 用的是什么角色
- 查了哪些敏感对象
- 改了哪些关键数据

数据库不是只负责执行，也应该负责留下痕迹。

## 如果让我给一个 PostgreSQL 安全落地顺序，我会这么排

很多文章讲安全，容易讲成一堆理念。我更想给一个后端团队回去就能动手的顺序。

如果现在让我接手一个 PostgreSQL 项目，准备把安全收回来，我会按这个顺序做：

### 第一步：盘点现状
先回答三个问题：

- 线上是不是还在用高权限角色直连数据库？
- 哪些表属于敏感对象？
- 现在的权限到底发到了表级，还是已经细到字段级？

这一步别急着改，先把真实情况摸清。

### 第二步：先拆 schema
把 `public`、`private`、`worker` 这类边界先分出来。

能先隔离的，先隔离。

### 第三步：收字段权限
把“用户侧需要看什么”“后端程序需要看什么”“哪些字段谁都不该直接碰”分开。

这一步通常比你想象中更有效。

### 第四步：挑最敏感的一张表上 RLS
不要一上来全库铺开。

先挑一张最典型、最敏感、权限边界也最清楚的表，比如订单、工单、客户资料、租户数据，先把策略跑通。

### 第五步：重构角色模型
把连接角色、业务角色、管理角色拆开。别再让 admin 参与日常业务流量。

### 第六步：补 `pg_hba.conf` 和审计
把连接入口和事后追溯补齐。

这六步走完，你的 PostgreSQL 安全才算真正开始成体系，而不是停留在“我们也做了鉴权”。

## 我最想压住的一个误判

如果整篇文章最后只能留一句话，我最想压住的误判是这个：

“数据库在内网里，应用层也已经做了鉴权，所以 PostgreSQL 里用一个高权限角色跑业务，其实问题不大。”

我现在越来越不认这句话。

不是因为它在任何场景下都绝对错误，而是因为它太容易把团队带向一个危险习惯：前面的安全动作越做越多，越容易觉得最后这一跳不用再认真设计。

恰恰相反。

越接近数据，越不该偷懒。

## 给后端团队的一份最小检查清单

如果你已经在用 PostgreSQL，我建议回去至少对照一下这张表：

- 业务流量是不是还在长期使用高权限账号？
- schema 是不是还只承担分类作用，没有承担边界隔离？
- 敏感数据是不是还和普通业务数据混在同一个可见范围里？
- 字段权限是不是默认整表开放？
- 那些“和用户有关但不该给用户角色直接看”的字段，有没有被收回来？
- 最敏感的业务表，有没有至少一张开始用 RLS？
- 连接角色、业务角色、管理角色，是不是已经分开？
- `pg_hba.conf` 和数据库审计，是不是已经补齐？

如果这几条里你已经踩中两三条，那数据库安全这件事，大概率还没真正做完。

## 结尾

我现在越来越觉得，很多团队不是不重视安全，而是把安全理解成了外围工程。

浏览器隔离、接口鉴权、内网部署、TLS 加密，这些都重要。但如果请求真的落到 PostgreSQL 时，后面还是一个几乎无所不能的高权限角色，那前面那套安全体系，最后还是会在最接近数据的地方漏掉。

这也是我为什么越来越看重 PostgreSQL 这套权限模型。

它真正值得学的，不是又多几个功能点，而是它提醒我：数据库不是终点站，它自己也该是最后一道权限判断。

性能掉一点，还能慢慢补。

安全最后一步失守，前面全白做。
