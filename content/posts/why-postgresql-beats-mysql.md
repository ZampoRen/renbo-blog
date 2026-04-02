---
title: "为什么 PostgreSQL 能超越 MySQL？这是我见过最全面的对比"
date: 2026-04-02T15:47:00+08:00
draft: false
tags: ["PostgreSQL", "MySQL", "数据库", "后端开发", "技术选型", "开源"]
description: "从索引、数据一致性、性能、扩展性四大维度，全面对比 PostgreSQL 与 MySQL。附详细对比表和选型建议。"
keywords: ["PostgreSQL vs MySQL", "数据库对比", "后端技术选型", "MySQL迁移PostgreSQL", "开源数据库"]
---

用了 MySQL 好几年，最近深入研究 PostgreSQL，才发现自己一直在将就。

不是 MySQL 不好，而是 PostgreSQL 在太多地方更胜一筹。这篇文章从四个维度把它说清楚：**索引、数据一致性、性能、扩展性**。

---

## 一、索引：PostgreSQL 的玩法更多

两者的基础索引都是 B+ 树，但 PostgreSQL 的花样也太多了。

### 聚簇 vs 非聚簇

MySQL（InnoDB）是**聚簇索引**：数据按主键顺序存在叶子节点上，查主键很快，但非主键字段要先查二级索引再回表查一遍。

PostgreSQL 的所有索引都是**非聚簇索引**：数据单独存放在堆表里，索引只存指针。主键和普通索引没区别，插入数据直接追加到末尾，不用担心页分裂。

### GIN 索引：倒排索引

PostgreSQL 的 GIN 索引简直是 JSON 数据的克星：

```sql
CREATE INDEX idx_attr ON products USING GIN (attributes);
SELECT * FROM products WHERE attributes @> '{"color": "red"}';
```

一句 `@>` 操作符就能查包含特定键值对的记录。**MySQL 根本不支持。**

### GiST 索引：什么都能索引

GiST 是一个索引框架，任何数据类型实现了接口就能用。

地理坐标、时间范围、IP 区间……这些用 PostgreSQL 原生的 `daterange` 加上 GiST 索引，查询重叠和距离效率极高。

```sql
SELECT * FROM reservations WHERE daterange '[2026-01-08,2026-01-13]' @> '2026-01-10'::date;
```

**MySQL 也不行。**

### 部分索引 + 表达式索引

逻辑删除的表想保持唯一索引？PostgreSQL 一行代码：

```sql
CREATE UNIQUE INDEX idx_phone ON users (phone) WHERE active = true;
```

邮箱大小写不敏感查询？直接对 lower() 结果建索引：

```sql
CREATE INDEX idx_email ON users (lower(email));
```

MySQL 8.0.13 才支持表达式索引，而且实现方式完全不同——PostgreSQL 是真的索引表达式，MySQL 只是生成了一个隐藏虚拟列。

---

## 二、数据一致性：PostgreSQL 的严谨你想不到

### DDL 也能回滚

在 PostgreSQL 眼里，**建表改表本质上也是写数据**，都存系统表里。

这意味着你可以把一堆 DDL 操作放进事务：
```sql
BEGIN;
ALTER TABLE users ADD COLUMN first_name TEXT;
ALTER TABLE users ADD COLUMN last_name TEXT;
UPDATE users SET first_name = split_part(name, ' ', 1);
-- 中间任何一步出错，整体回滚，不留脏数据
COMMIT;
```

MySQL 没有这个能力。DDL 就是 DDL，没法回滚。

### 可延迟约束

想给两个用户对调名字？但名字字段有唯一约束。

PostgreSQL 告诉你什么叫"先把鸡和蛋都准备好，再一起提交"：
```sql
CREATE TABLE users (name TEXT UNIQUE DEFERRABLE INITIALLY DEFERRED);
BEGIN;
UPDATE users SET name = 'Bob' WHERE id = 1;
UPDATE users SET name = 'Alice' WHERE id = 2;
COMMIT;  -- 提交时才检查唯一性，此时没有冲突
```

### 隔离级别：MySQL 那个坑

MySQL 默认的"可重复读"有个很离谱的 bug：

- SELECT 读的是事务开始的快照
- UPDATE 读的却是当前最新数据

同一个事务里，查不到但能改？这不符合直觉，还容易出 bug。

PostgreSQL 的默认隔离级别是"读已提交"，读写一致，不整活。

---

## 三、性能：MySQL 的优势早没了

油管博主 Anton Putra 做了实测：

**高并发写入**：PostgreSQL 是 MySQL 的 **2 倍**
**高并发查询**：PostgreSQL 是 MySQL 的 **1.5 倍**

更残酷的是版本趋势：
- MySQL 5.7 → 8.0 → 8.4，性能越来越差
- PostgreSQL 17 比 6 年前的 PostgreSQL 10，提升了 **30%~70%**

一句话：MySQL 以前引以为傲的性能，如今已被反超。

---

## 四、扩展性：PostgreSQL 才是真正的"开源"

| | MySQL | PostgreSQL |
|---|---|---|
| 自定义数据类型 | ❌ | ✅ |
| 自定义运算符 | ❌ | ✅ |
| 自定义索引类型 | ❌ | ✅ |
| 第三方插件 | 47 个 | **375 个** |

MySQL 的可插拔存储引擎听起来很美好，但现实是 99% 的人都在用 InnoDB，其他引擎基本没人维护。

PostgreSQL 的扩展是彻底的。以 PostGIS 为例——没改一行核心代码，就实现了企业级地理信息系统。数据类型、函数、索引、运算符，四大扩展点全用上了：

```sql
-- 查距离最近的三个地点
SELECT name FROM locations
ORDER BY geom <-> (SELECT geom FROM locations WHERE name = '电话亭')
LIMIT 3;
```

### 开源协议

- **PostgreSQL**：类 MIT/BSD，最宽松，改完闭源没人管你
- **MySQL**：GPL v2，有传染性，链接即开源，想闭源找 Oracle 买授权

这也是为什么 MariaDB 会诞生——MySQL 创始人 2009 年自己搞了个分支，就因为不想受制于 Oracle。

---

## 总结

| 维度 | MySQL | PostgreSQL |
|---|---|---|
| 索引 | B+ 树、全文索引 | B+ 树、GIN、GiST、部分索引、表达式索引 |
| DDL 事务 | ❌ | ✅ |
| 可延迟约束 | ❌ | ✅ |
| 读写一致性 | SELECT 和 UPDATE 快照不同 | 行为一致 |
| 性能 | 逐版本下降 | 逐版本提升 |
| 扩展性 | 有限 | 极强 |
| 开源协议 | GPL 传染性 | MIT 最宽松 |

如果你在选型，或者纠结要不要从 MySQL 迁移，这篇文章应该能帮你做出判断。

**我的结论：MySQL 不是不能用，但在索引灵活性、数据一致性、扩展性、开源自由度这几个关键维度上，PostgreSQL 更现代、更严谨、更自由。**

---

*如果你觉得这篇文章有帮助，点个赞再走？有问题评论区见。*