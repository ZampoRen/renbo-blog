---
title: "我为什么从 MySQL 投奔 PostgreSQL（以及没有全盘迁移）"
description: "一篇真实的迁移记录：什么场景下宁可多花两周也要换，什么场景保留 MySQL 不动。连接风暴、DDL 阻塞、autovacuum 陷阱、混合架构——都是付过学费才得到的判断。"
date: 2026-06-27T10:00:00+08:00
draft: false
author: "任博"
tags: ["MySQL", "PostgreSQL", "数据库", "迁移实战", "工程决策"]
categories: ["技术实战"]
cover: "/images/mysql-to-pg-migration/cover.png"
toc: true
---

上一篇《MySQL 千疮百孔，为什么没人能替代它》发出去之后，收到的留言比我预想的多。大部分不是技术讨论，是一种很微妙的共鸣：我知道它有问题，但我确实不知道该不该换。

这其实才是真正的困惑。

你看完 MySQL 那堆坑——ENUM 排序按定义位置、级联外键不触发 trigger、DDL 隐式提交——你的第一反应不该是"PostgreSQL 好，我要换"。你的第一反应应该是：我当前的项目，这些坑到底有没有咬到我？

如果没咬到，别动。

如果咬到了，也不能说换就换。你得知道换了之后会面临什么全新的疼法。

这篇就是讲这个的。

---

## 让我的手指终于放在键盘上的那个瞬间

先说我的触发点。

不是什么宏大的"我们要用更先进的数据库"。是一个凌晨两点半的告警电话。

线上 MySQL 实例的活跃连接数冲破了 500。`thread_cache_size` 已经调过一轮了，`max_connections` 也放宽过。但 MySQL 的 thread-per-connection 模型到了这个量级，上下文切换已经压垮了 CPU。新连接进不来，已有查询跑不完，连接池开始拒绝请求。回滚也回不干净，因为 DML 都被堵在了锁等待队列里。

我试过调参数、重启连接池、杀掉慢查询。能做的都做了，只够撑到天亮。

天亮以后我查了接下来半年要上线的功能排期。三个和搜索有关的，两个需要地理查询，还有一个说想在用户标签里做 JSON 过滤。MySQL 的 JSON 你是知道的——每次解析全文，索引靠虚拟列绕路，这不是能用，是将就。

那天中午我开了个会。结论不是"PG 更好"，是"在这个点上，不改的成本已经超过改的成本了"。

但我仍然没有做全盘迁移。这篇就是告诉你我换了什么、没换什么、以及换完之后才知道的学费。

---

## 第一块：什么让我决定换

### 连接数瓶颈——不是 PG 更好，是 PgBouncer 把它变成了可控问题

MySQL 的 thread-per-connection 模型在 <100 连接时没什么感觉。到了 200+，你就开始频繁看 `Too many connections`。到 500+，运维基本靠祈祷。

这问题技术上不是 MySQL 不能处理高并发——淘宝、Facebook 都在用——但那是建立在应用层做了极其精细的连接管理之上的。大部分团队没有那个资源。

PostgreSQL 本身也是 fork-per-connection，每个连接 2-5 MB 基础内存，直接连 500 个也扛不住——甚至比 MySQL 更差。但 PG 加 PgBouncer 的组合把问题结构化了：你在应用层维持几百上千个连接，PgBouncer 用事务池把它们复用到 PG 端的几十个连接上。

Percona 的基准测试说得很清楚：56 并发时直接连接比 PgBouncer 快 2.5 倍（PgBouncer 有 ~2-3ms 的开销），但到了 300 并发，PgBouncer 的 TPS 比直接连高 150%，P95 延迟低 40%。到 600，直接连接已经不可用了。

我自己的经验分界线：**当应用端持久连接超过 200 个，且你不想在应用层实现连接复用时，PG + PgBouncer 的结构化优势就出现了。**

但对于低于 50 并发的应用，PgBouncer 是纯包袱。直接连就好。

### MVCC 写放大——换个角度看同一件事

MySQL 的 MVCC 用 Undo Log 记录行的"差异"。好处是写放大很小——改一列只记一列。坏处是 Undo 空间膨胀不可控，长事务会 pin 住 Undo 的清理。

PostgreSQL 的 MVCC 在堆表里存完整行版本。每个 UPDATE 把整行复制一份，标记旧版本为死元组。写放大比 MySQL 大得多——这是事实，也是取舍。

但这里有一个很多人忽略的对比维度：**读写互斥**。

MySQL 的 Undo Log 需要计算才能重建旧版本，而且写操作在某些隔离级别下可以阻塞读。PostgreSQL 的 `xmin`/`xmax` 标签让读从不阻塞写，写从不阻塞读——这不是微优化，这是真实负载下才体现得出的差距。

一个叫 Boozt 的时尚平台在迁移报告里写了这么一句话，我看了很有共鸣："迁移到 PostgreSQL 之后，Slack 里的锁等待超时告警完全消失了。"

**我的分界线：当"LOCK WAIT TIMEOUT"已经成为运维周报上的常态项，当你的负载是混合读写 OLTP 而非单纯的写多读少，PG 的 MVCC 并发模型带来的运维简化很值得。**

### DDL 阻塞——这个体感差异是"用了就回不去"级别的

MySQL 做 schema 变更是一种特定形式的痛苦。线上加字段、加索引，哪怕你用 gh-ost 或 pt-online-schema-change，也要经过排练、窗口、回滚脚本。

最让人崩溃的是那种"改到一半发现问题，但回不去了"的场景。MySQL 的 DDL 会隐式提交当前事务，你没法 `ROLLBACK`。

PostgreSQL 支持事务性 DDL。这意味着：

```sql
BEGIN;
ALTER TABLE users ADD COLUMN email TEXT;
CREATE INDEX idx_email ON users(email);
-- 发现问题？
ROLLBACK;  -- 以上所有操作全部回退，表回到之前的状态
```

我团队里有一个 DBA 在第一次体验了这个功能后，在群里发了很长一串"卧槽"。这不是夸张。当你每个月要做 5-8 次线上 schema 变更，每次都要花至少半天排练，这个功能带来的心流差异是巨大的。

PG 也不是所有 DDL 都无锁。`ALTER TABLE ... ADD COLUMN` 加非空默认值在 PG 11+ 是瞬间完成的（纯元数据操作）。`CREATE INDEX CONCURRENTLY` 不会阻塞写，但会跑得更久。`ALTER TABLE ... SET NOT NULL` 会扫描全表。需要区分对待。

**分界线：如果你的团队每月线上 schema 变更次数超过 3 次，且大部分变更需要窗口排练，事务性 DDL 给你的不仅是效率，还有犯错的安全网。**

### 扩展性——当你发现要引入第二个专用系统的时候

MySQL 的 plugin 能做的事情有限。不能添加新的数据类型、新的索引方法、新的运算符。

PostgreSQL 的 Extension 系统是引擎级别的。pgvector 添加的 `vector` 类型是原生的一等公民——可以参与事务、MVCC、备份、复制。PostGIS 添加的 GiST 索引和 B-tree 索引共享同一个查询计划器框架。

我做了一个简单的估算：我们团队同时维护了 MySQL（主业务）、Elasticsearch（搜索）、MongoDB（文档/地理）、InfluxDB（时序）。如果用 PG + Extensions，可以将这个四数据库架构压缩为 PG + Redis。这意味着一套备份策略、一组监控面板、一种查询语言。

当然，Extension 不是银弹。pgvector 的 HNSW 索引和 Pinecone 这种专用向量数据库比还有差距。但如果你不需要那 5% 的极致性能差异，少维护一套系统的工程收益是巨大的。

---

## 第二块：换了之后没想到的成本

这部分我花最多的篇幅，因为它才是迁移的真正学费。

### autovacuum 是 PG 最大的坑——不是功能问题，是运维习惯要重学

我把这行放在最前面，因为这真的是我花最多时间调教的东西。

MySQL 的运维人员习惯了"无需显式回收空间"——InnoDB 的 PURGE 线程是自动的，你不会每天去想死元组的事。

PostgreSQL 的 VACUUM 是你必须正视的运维职责。

最糟的情况是什么？死元组堆积导致表膨胀、索引膨胀、顺序扫描变慢、统计信息过时、规划器做出错误计划——最终触发 Anti-wraparound 紧急 VACUUM，强行锁表清扫。

这是可以避免的，但前提是你知道 PG 的 autovacuum 默认值是一个陷阱。

默认的 `autovacuum_vacuum_scale_factor = 0.2` 意味着 200M 行的表会等到 40M 死元组才开始清扫。这太慢了。

我学到的最重要教训：**不要全局调 autovacuum，用 per-table 设置。**

```sql
-- 热表单独调优
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- 1% 死元组就触发
    autovacuum_vacuum_threshold = 10000,
    autovacuum_vacuum_cost_limit = 2000
);
```

调优之后：autovacuum 每 5-10 分钟跑一次，每次 30-90 秒，业务几乎感知不到。

另外一个差点让我翻车的：**空闲事务 pin xmin horizon**。一个长期不提交的事务会阻止 VACUUM 清扫任何比它新的死元组。Trigger.dev 有一篇复盘文章，他们的 PG 实例因为 xmin 被 pin 住，表从 2GB 膨胀到 20GB，CPU 从 30% 飙到 90%，最终靠故障切换到只读副本解决。

还有复制槽。无效的逻辑复制槽不清理，VACUUM 会一直跳过清理。这在开发/实验环境里很常见——创建了复制槽，项目结束了，忘了删。

所以，如果你计划迁移到 PG，先回答自己一个问题：你的团队有没有时间去学一套全新的空间回收运维体系？这不是 PG 比 MySQL 更难，是**不一样**。而"不一样"在运维领域就代表时间成本。

### 数据类型迁移陷阱——看起来像，跑起来不是

最烦人的不是那些明显不同的类型。是那些看起来一样的。

**陷阱一：`ON UPDATE CURRENT_TIMESTAMP`**

MySQL 里写 `updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` 是天经地义的事。PG 没有这个语法。你需要用 trigger：

```sql
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

这个极其容易被遗忘。我第一次迁移的时候就是，跑了一个星期发现所有表的 `updated_at` 都停在插入时间。线上数据已经写乱了才意识到。

**陷阱二：零日期**

MySQL 允许 `'0000-00-00'` 作为 DATE 值。PG 拒绝。直接迁移会在中间暴毙。这是一个必须在迁移前置脚本里检查的东西：

```sql
SELECT * FROM table WHERE date_column = '0000-00-00';
```

**陷阱三：UNSIGNED 不存在**

PG 没有 `UNSIGNED INT`。全部变成了 `INTEGER` + `CHECK (col >= 0)`。如果你的应用代码里某个列被声明为 UNSIGNED 但从来不真的用负数，这改起来就是纯体力活。但如果某个隐藏逻辑依赖了 UNSIGNED 溢出回绕这个 MySQL 行为（真的有人这么写），排查会让你崩溃。

### SQL 兼容性差异——迁移第一周被绊 checklist

如果你和我一样写了很多年 MySQL SQL，以下是你迁移第一周一定会被绊的清单：

| MySQL 写法 | PG 写法 | 绊倒频率 |
|-----------|---------|---------|
| `` `backtick` `` | `"double quote"` 或无引号 | 每小时 |
| `LIMIT 10, 20` | `LIMIT 20 OFFSET 10` | 每天 |
| `IFNULL(a, b)` | `COALESCE(a, b)` | 每天 |
| `IF(cond, a, b)` | `CASE WHEN cond THEN a ELSE b END` | 每天 |
| `GROUP_CONCAT(...)` | `STRING_AGG(...)` | 每周 |
| `INSERT ... ON DUPLICATE KEY UPDATE` | `INSERT ... ON CONFLICT ... DO UPDATE` | 每周 |
| `UPDATE ... JOIN` | `UPDATE ... FROM` | 每周 |
| `REPLACE INTO` | `ON CONFLICT ... DO UPDATE` | 数周一次 |

最让我改到手软的是 `UPDATE...JOIN`。MySQL 的方式很直观：

```sql
UPDATE users u
JOIN orders o ON u.id = o.user_id
SET u.total_spent = u.total_spent + o.amount
WHERE o.status = 'completed';
```

PG 必须换一种组织方式：

```sql
UPDATE users u
SET total_spent = u.total_spent + o.amount
FROM orders o
WHERE u.id = o.user_id AND o.status = 'completed';
```

语义等效，但每一处都要手动改。如果你有大量手写 SQL，先做一个存量评估。一个 400M 行迁移的真实案例里，47 个存储过程重写花了 3 周，23 个触发器全部重写。这不是危言耸听。

### 工具链切换——备份和监控不是"换个命令"那么简单

MySQL 时代你可能习惯了 `mysqldump`、`XtraBackup`、`SHOW SLAVE STATUS`、`performance_schema`。到了 PG，工具名都变了：`pg_dump`、`pg_basebackup`、WAL 归档、`pg_stat_statements`。

连接池也从"推荐有"变成了"必须有"。PgBouncer 不是可选项：超过 100 并发，你必须用它。而 PgBouncer 有三个模式——session、transaction、statement——选错了模式会导致你的 `LISTEN`/`NOTIFY`、临时表、prepared statements 全部无法正常工作。

如果你依赖 Prometheus 做监控，MySQL exporter 换成 PG exporter 也要重新配一遍。默认的 metric 集不同，告警规则的阈值也需要重新学习。

这些事加起来，不是一个周末能搞定的。Cadence 的迁移成本分析里说：一个熟练团队迁移 200GB 的数据库，估算 6-12 周。其中包括代码改写、数据迁移、测试和回滚预案。这不是给你泼冷水，是让你做预算时别太乐观。

### 团队学习曲线——不是更难，是"不一样"

我的团队在 PG 迁移上经历了两个阶段。

前两周是语法适应期。每个人都学会了一个新公式：`IFNULL` → `COALESCE`，`LIMIT a, b` → `LIMIT b OFFSET a`，backtick 去掉。

后四周才是痛苦的开始。autovacuum 调优、PgBouncer 配置、`EXPLAIN ANALYZE BUFFERS` 输出解读、BRIN 和 GIN 索引在什么场景下要用、复制槽管理——这些知识不是靠查文档消化得了的，需要在真实故障中才能形成肌肉记忆。

我大致估算：一个 MySQL 团队从零到生产级 PG 熟练，需要 4-8 周的集中投入。如果你赶业务交期，这不是好时机。

---

## 第三块：我没有全盘迁移的部分

到这里你可能觉得我是个 PG 布道者。其实不是。

我有很多系统**没有**迁移。有些是故意的，有些是迁移到一半停下来了的。

### 什么系统留在 MySQL 上不动

**简单 CRUD + 低并发。** 一个内部管理系统，用户不到 100 人，十几张表，全都是单表查询。迁移它的 ROIC（投资回报率）是负数。迁移带来的风险远大于收益。

**写密集型、小行高频 UPDATE。** 我们有一个计数器服务，每秒成千上万次 UPDATE 某行的 count 字段。PG 的行版本机制在这里是劣势——每个 UPDATE 复制整行，写放大明显。MySQL 的 Undo Log（只记录差异）更合适。

**已有超成熟运维体系的存量系统。** 我们有一个跑了 7 年的 MySQL 集群，运维手册几百页，什么故障模式都见过。迁移到 PG 意味着把这几百页扔了重写。不是 PG 不好，是迁移的 ROI 算不过来。

### 混合架构——我最终的选择

我没有全盘迁移。我用的是 MySQL 做主库 + PG 做特定场景的混合架构。

具体来说：

- **用户服务、订单服务**：留在 MySQL。事务完整性要求高，且 MySQL 已经完全够用。
- **搜索服务**：迁到 PG + pgvector。之前我们用 Elasticsearch，但为了一个全文检索功能维护一套 ES 集群太浪费了。pgvector 加上 pg_bm25 覆盖了 90% 的搜索场景。
- **分析服务**：迁到 PG + 窗口函数 + 物化视图。之前用 MySQL 跑分析查询，`GROUP BY` 嵌套子查询跑出分钟级响应，换到 PG 用窗口函数改写后降到秒级。
- **地理服务**：迁到 PG + PostGIS。之前用 MongoDB 做地理查询，现在少维护一套系统。
- **时间序列**：迁到 PG + BRIN 索引。BRIN 索引真是一个惊喜——对一个 50M 行的 events 表，B-tree 索引要 480 MB，BRIN 只要 4.2 MB，缩小了 99%。

这个"不是全有也不是全无"的方案比我想象的好。每个模块独立迁移，范围可控，风险隔离。出了问题也只影响一个服务。

### 一个判断框架

用了一年多混合架构之后，我对自己团队的判断框架大致变成了这样：

**应该迁的：**
1. MySQL 需要外挂 2 个以上专用系统来解决它不原生支持的需求（向量 + 全文检索 + GIS）
2. 每月至少发生 1 次线上 DDL 相关事故
3. 连接风暴导致每周至少 1 次 P1 告警
4. 分析类查询占比超过 20%，且窗口函数能显著简化代码
5. 团队有 4-8 周的集中投入时间学习 PG

**不该动的：**
1. 历史 SQL 超过 10 万行，且大量使用 MySQL 方言
2. 团队正在赶业务交期，没有学习带宽
3. 无法承受超过 2 小时的迁移停机窗口
4. 写密集型小行高频 UPDATE 的纯 OLTP 场景
5. 已经跑了好几年、不出故障的存量系统

---

## 结尾

写了这么多，我想说的其实很简单。

迁移不是最终答案。学会在 MySQL 和 PostgreSQL 之间做选择才是。

MySQL 在很多场景下完全够用。PostgreSQL 在另一些场景下能省掉你维护多套系统的时间。它们都不是完美的数据库。工程世界里永远没有一个"在所有维度上最优"的选项。

真正成熟的技术判断不是"我知道哪个更好"，而是"我知道在什么条件下选哪个、换哪个、不换哪个"。

这不仅是技术题。它是运维投入、团队时间、迁移窗口、风险承担能力之间的权衡。

所以我的建议：不要因为看了我这篇就去迁移。也不要因为 MySQL 有问题就否定 MySQL。

先回答一个问题：**你当前系统最痛的点，是不是换数据库就能解决的？**

如果是，再开始评估。如果只是觉得"可能 PG 更好"，先留着。真正需要你行动的时候，你的系统会告诉你的。那声凌晨的告警电话，比任何文章都诚实。
