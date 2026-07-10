---
title: "你写了个日志系统选了 InnoDB，然后发现写入越来越慢"
slug: "lsm-vs-btree-storage-engine-decision"
cover: "/images/lsm-vs-btree-storage-engine-decision/cover.png"
date: 2026-07-10
draft: false
description: "LSM-Tree 和 B+Tree 都在写放大，只是放大的时机不同。写多读多、写少读多、写多读少——你的场景该选哪个。"
tags: ["存储引擎", "LSM-Tree", "B+Tree", "RocksDB", "InnoDB", "数据库"]
categories: ["后端", "数据库"]
---

你做了一个日志系统。

数据量每天 500GB，全是 APP 行为埋点。你选了 MySQL InnoDB，因为团队最熟。跑了一个月，写入越来越慢，到了夜里还要锁表清理。DBA 说：要不试试换 RocksDB？

你换了一版。写入快了很多，但查询偶尔会卡一下，有时候 p99 飙到 200ms+。老板问为什么，你说「Compaction 抖了一下」。

然后你开始琢磨：InnoDB 和 RocksDB，底层不都是「树」吗？怎么差异这么大？

**B+Tree 和 LSM-Tree 不是「哪个更快」的问题，是「你把写放大的代价付在什么时候」的问题。** 前者付在每次写入，后者付在后台合并。你的业务承受哪种代价，就选哪种设计。

---

## 先解决一个问题：B+Tree 在写入时到底发生了什么

大多数后端工程师都用过 MySQL InnoDB，知道数据存在索引里，索引是 B+Tree。但到底怎么「写进去」的，大部分人没细想过。

B+Tree 的结构简化版是这样的：

- 所有数据行（value）只存在叶子节点
- 内部节点只存 key，不存数据，作用是导航
- 叶子节点之间用双向链表连接，方便范围扫描

当你 INSERT 一行数据，InnoDB 做的事是：

1. 从 root 开始，顺着内部节点往下走到对应叶子页
2. 把这一页从磁盘读到 Buffer Pool（如果不在内存里）
3. 在页内的有序位置插入
4. 如果页满了，分裂成两页，重新分布键值
5. 如果分裂传到上层，级联更新内部节点

**问题出在第 4 步。**

你用自增 ID 做 PK 时，新插入的数据永远往最右侧叶子节点写，只有在最右侧满了才触发分裂，而且只影响右边界。

但是你用了 UUID 做 PK——新人常犯的经典错误——或者业务 key 本身就是分散的，那么每次插入的数据会随机落到任意一个叶子节点。**每次插入都可能触发一次页分裂。**

页分裂的代价是什么？InnoDB 默认页大小是 16KB。你写入一行 128 字节的数据，如果触发了页分裂，InnoDB 要读旧页、写两页，刷盘的数据可能是 32KB+。你写的是 128 字节，硬盘却干了 32KB 的活。**写放大。**

而且缓存在 Buffer Pool 里的热数据页，因为分裂被驱逐了，下次再读到同区域，又得从磁盘拉一次。

所以在随机 Key 写入场景下，InnoDB 的吞吐上限很快就被磁盘随机 I/O 和页分裂锁死了。这是数据结构的刚性成本。

**MySQL 本身没有问题，问题是你的写入模式撞上了 B+Tree 的设计边界。**

那有没有一种结构，把所有随机写统一变成顺序写？

有。LSM-Tree。

---

## LSM-Tree 是怎么把「随机写」变成「顺序写」的

LSM-Tree 的完整写入路径是四步链，每一步都为了消除随机写：

**第一步：WAL（Write-Ahead Log）。** 数据来了，先追加到日志文件。这是纯顺序写，一个文件的尾部追加。非常快，IO 效率接近理论极限。

**第二步：Memtable。** 同时把数据写入内存中的有序结构（通常是跳表或平衡树）。全内存操作，跟磁盘没关系。

**第三步：SSTable 刷盘。** Memtable 满了（默认大小可配，RocksDB 常见 64MB），整个数据结构序列化成磁盘上一个不可变的文件——SSTable（Sorted String Table）。批量顺序写，一次性落一个大文件。

**第四步：后台 Compaction。** 磁盘上随时间积累了大量 SSTable，它们之间有重复 key、有过期数据。后台线程定期把这些 SSTable 合并、重写、下沉到更深的层级。

**整个写入过程，只有 WAL 刷盘和 SSTable 刷盘涉及磁盘 I/O，而且全是顺序追加。** 没有随机写，没有页分裂。

代价在后面——读的时候。

---

## 读路径：LSM 付出的代价

B+Tree 读一行数据，路径是确定的：从 root 到叶子，O(log n) 次磁盘读取。因为有 Buffer Pool，热门的内节点几乎都在内存里，实际读次数很少。

LSM 读一行数据，要面对的问题是：数据可能在任何一层。

写入路径决定了数据流的时序：新数据在 Memtable，稍早的在 L0 层 SSTable，更久的在 L1、L2……

读一条 key：

1. 先查 Memtable（内存中，快）
2. 再到 L0 层查 SSTable（L0 层内可能有多个重叠的 SSTable，每个都要查）
3. 再到 L1、L2……一层层查下去

如果没有优化，读一次数据可能要打开多个文件、多次比较。

**优化手段来了：Bloom Filter。**

每个 SSTable 文件在写入时附带一个 Bloom Filter。它能斩钉截铁地告诉你「这个 key 肯定不在这个文件里」，但只能说「可能在」。

读路径变成：

1. 查 Memtable → 没找到
2. 对 L0 的每个 SSTable 用 Bloom Filter 快速过滤 → 99% 的文件被排除
3. 只在 Bloom Filter 说「可能在」的文件里做实际二分查找

读性能靠 Bloom Filter 兜回来的。Bloom Filter 的精度由 bits-per-key 控制（RocksDB 默认 10，大约是 1% 的误判率）。你愿意花更多内存来存 Bloom Filter，读性能就更好。

所以 LSM 的读性能不是「天生慢」，是你没把 Bloom Filter 配够。**这是吐预算换性能，不是结构缺陷。**

---

## 再说 Compaction：是什么，为什么抖

Compaction 是 LSM 的「垃圾回收 + 版本合并」后台线程。

当 L0 层 SSTable 数量超过阈值（RocksDB 默认 4 个），触发一次 compaction：把 L0 的文件和 L1 做归并排序，生成新的 L1 SSTable，丢弃被覆盖的旧版本数据。

Compaction 做的事本质上是**大规模归并排序 + 文件读写**。这会占 CPU、占磁盘带宽。如果你的业务写入量大，compaction 频率就高，磁盘带宽可能被吃满——这时候前台查 A 数据，磁盘正忙着重写 B 文件，读延迟就上去了。

这就是「抖动」的来源。

**但这是可以调的。** RocksDB 提供了大量调优手段：

- 调大计算 compaction 需要的线程数
- 限速（rate_limiter）：限制 compaction 占用的磁盘带宽
- 预分配 compaction 所需的空间
- 用更短的 compaction 策略（size-tiered）换更少的写放大

每个调优选项都在同一个三角里做取舍：**写放大 ↔ 读放大 ↔ 空间放大**。

Size-Tiered Compaction：写放大最小，但读放大和空间放大最大。适合写多读少、数据很快过时的场景（日志、时序）。

Leveled Compaction：读放大和空间放大最小，但写放大最大。适合读多写多的混合场景。

RocksDB 的默认策略是 Tiered + Leveled 混合——L0 用 size-tiered，L1+ 用 leveled。这也是工业实践里最常见的方案。

---

## 产品落地：这些引擎用在哪

理论够了，看实际。

### LevelDB（Google）

LSM-Tree 的「教科书实现」。单机、单线程 compaction，写入性能不错但调优空间小。Google 内部早期用在 Chrome、Bigtable 等场景。现在大部分新项目已经迁移到 RocksDB。

### RocksDB（Facebook/Meta）

LevelDB 的工程升级版。加了多线程 compaction、Bloom Filter 参数可调、列族（column family）支持、支持事务、100+ 配置项。跟 LevelDB 比就像「原型机」和「生产型」的差距。

**典型场景：**
- 时序数据库（InfluxDB、TDEngine 底层）
- 消息队列持久层（Kafka 的某些实现用 RocksDB 做状态存储）
- 流式计算（Flink、Spark 的状态后端）
- 图数据库（Neo4j 的某些存储层）
- MySQL 替代引擎 MyRocks（Facebook 内部大量使用）

### WiredTiger（MongoDB）

MongoDB 默认存储引擎，用 B-Tree（带 LSM 可选模式）。支持行级锁和压缩。MongoDB 3.2 后默认从 MMAPv1 切换到 WiredTiger，写入性能提升明显。

比较特殊的是 WiredTiger 同时支持 B-Tree 和 LSM 两种存储模式，同一份配置可以按集合粒度指定。这是「两个都要」的务实方案。

### InnoDB（MySQL）

MySQL 默认引擎，B+Tree 的工业级实现。16KB 页、Adaptive Hash Index、Double Write Buffer、Change Buffer。稳定、可控、生态成熟。

适用于 OLTP 混合读写场景：电商、金融、社交。在这些场景下，InnoDB 的延迟模型可预测，这在交易类场景里比「峰值吞吐高但偶尔卡一下」重要得多。

---

## 怎么选：不是 LSM 还是 B+Tree，是什么负载

来，直说。

### 写入密集型 → 选 LSM

时序数据、日志、埋点、消息队列。写入量大，写入密集，数据很少二次修改。读通常是范围扫描（查最近 N 分钟的数据），很少点查特定 key。

**推荐：RocksDB。** 做底层引擎，上层封装接口。如果不想从零写，直接上 TiKV 或 CockroachDB 这类用 RocksDB 做底层存储的分布式数据库。

### 读取密集型 → 选 B+Tree

商品详情、用户资料、内容管理。点查多、范围查少，写入量相对低。延迟稳定比峰值更重要。

**推荐：MySQL InnoDB 或 PostgreSQL。** 成熟稳定，运维成本低，DBA 好招。

### 混合读写 → 看取舍

OLTP 系统通常读写都有，但不同的业务对 p99 读延迟的容忍度不同。

- **能接受偶尔 50ms 读延迟抖动 → LSM 家族。** 用 RocksDB 调参，适当加 Bloom Filter 内存预算，把读性能拉到接近 B+Tree。
- **不能接受抖动（交易、支付、库存）→ B+Tree 家族。** InnoDB 的延迟模型更稳定，抖动来自 MVCC、死锁、长事务，而不是 Compaction。

### 大型分布式系统 → 混合部署

大型系统已经在走「两个都要」的路了。

- **TiKV**：Raft + RocksDB，用 LSM 做写入性能，用 Raft 做分布式一致性。
- **MySQL + MyRocks**：同一套 MySQL 协议，InnoDB 做 OLTP 表，MyRocks 做写密集型表。
- **WiredTiger**：单个引擎内可以按集合选 B-Tree 或 LSM。

---

## 结语

你写的不是一个数据库，是一个判断框架。

所有存储引擎都在同一个三角里做取舍：**写性能、读性能、空间效率。** 不存在全面胜出的结构。

B+Tree 把写放大的代价付在每次写入的原地更新上——随机 I/O、页分裂、尾部延迟不可预测，但读路径简单稳定。

LSM-Tree 把写放大的代价付在后台 Compaction 上——前台写入几乎全是顺序 I/O，吞吐高，但读路径复杂，Compaction 可能抖动。

**你选的不是「哪种更快」，是你觉得哪种代价你能赔得起。**
