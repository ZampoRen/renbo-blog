---
title: "我用 PostgreSQL 替换了整个技术栈，省了 6 个微服务"
description: "现代软件工程已经变成了订阅管理模拟器。我们被云厂商洗脑了，以为即使构建一个基础应用，也需要拼凑一个脆弱的分布式网络。我刚刚用 PostgreSQL 替换了整个技术栈。这是一个过度工程化、价格离谱的陷阱。"
date: 2026-04-17T20:00:00+08:00
draft: false
categories: ["技术"]
tags: ["PostgreSQL", "架构设计", "微服务", "数据库", "工程实践"]
slug: "replaced-entire-stack-with-postgresql"
images: ["/images/postgresql-replaces-stack/cover.png"]
toc: true
---

现代软件工程已经基本变成了"订阅管理模拟器"。

我们被云厂商洗脑了，以为即使构建一个基础应用，也需要拼凑一个脆弱的分布式网络：
- Redis 做缓存
- Confluent Kafka 集群做后台任务
- Elasticsearch 只是为了一个简单的搜索框
- 再搞个专用向量数据库为了那个临时加上的 AI 功能

等你终于把应用部署给你那"要求极高"的用户群（你自己和你妈）时，你已经欠了十几家 Y Combinator 支持的 SaaS 初创公司的钱，就为了让灯亮着。

**这是一个过度工程化、价格离谱的陷阱。**

但如果我告诉你，你可以把那些闪亮的云依赖全部扔进焚化炉，用一个经过 30 年验证的开源软件替换它们呢？

**科技行业不想让你知道的秘密：一个久经考验的工具可以吞噬你的整个架构。**

我刚刚用 PostgreSQL 替换了整个技术栈。

---

## 一、PostgreSQL 为什么能吞噬整个栈

PostgreSQL 是一个开源对象关系数据库系统，已经活跃开发了 30 多年。

**开箱即用的能力：**

- 坚如磐石的 ACID 合规性——当你的廉价云服务器崩溃时，用户数据不会损坏
- **可扩展性**——这才是它能吞噬整个栈的真正原因

**PostgreSQL 不只是行列存储。**

它还能存储：
- JSONB（半结构化数据）
- 向量（AI 嵌入）
- 全文搜索索引
- 地理空间数据（PostGIS）
- 图数据（关系遍历）
- 时间序列数据
- 键值对（作为 Redis 替代）

**PostgreSQL 不是数据库，是一个数据平台。**

---

## 二、实战替换：Redis（缓存 + 键值存储）

### 传统架构

```
应用 → Redis（缓存） → PostgreSQL（持久化）
```

**问题：**
- 两个服务要运维
- 缓存和数据库数据可能不一致
- Redis 挂了要处理降级

### PostgreSQL 方案

**使用 UNLOGGED 表做高速缓存：**

```sql
-- UNLOGGED 表：不写 WAL，崩溃后自动清空，性能接近 Redis
CREATE UNLOGGED TABLE session_cache (
    key TEXT PRIMARY KEY,
    value JSONB,
    expires_at TIMESTAMPTZ
);

-- 创建索引加速查询
CREATE INDEX idx_expires ON session_cache (expires_at);
```

**UNLOGGED 表的特点：**
- 不写 Write-Ahead Log（WAL），性能接近纯内存
- 崩溃后自动清空（正好做缓存失效）
- 支持 SQL 查询、JOIN、过期清理

**使用 pg_prewarm 预热缓存：**

```sql
-- 数据库重启后预热缓存
SELECT pg_prewarm('session_cache');
```

**性能对比：**

| 指标 | Redis | PostgreSQL UNLOGGED |
|------|-------|---------------------|
| 读取延迟 | 亚毫秒 | 毫秒级 |
| 写入延迟 | 亚毫秒 | 毫秒级 |
| 运维负担 | 独立服务 | 无额外负担 |
| 数据一致性 | 最终一致 | ACID 事务 |

**边界说明：**

- ✅ 适合：会话缓存、API 响应缓存、热点数据
- ❌ 不适合：亚毫秒级需求、超高并发（>10 万 QPS）

---

## 三、实战替换：Kafka（消息队列 + 后台任务）

### 传统架构

```
应用 → Kafka → 消费者服务 → PostgreSQL
```

**问题：**
- Kafka 集群运维复杂
- 需要单独的消费服务
- 数据一致性难保证

### PostgreSQL 方案

**使用 SKIP LOCKED 做任务队列：**

```sql
-- 创建任务表
CREATE TABLE jobs (
    id SERIAL PRIMARY KEY,
    status TEXT DEFAULT 'pending',
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    worker_id INTEGER
);

-- 获取并锁定下一个待处理任务（多 worker 并发安全）
UPDATE jobs
SET status = 'processing', worker_id = pg_backend_pid()
WHERE id = (
    SELECT id FROM jobs
    WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED
);
```

**SKIP LOCKED 的妙用：**
- 多个 worker 并发消费，不会抢同一个任务
- 无需分布式锁
- 无需轮询等待

**PostgreSQL 14+ 的 LISTEN/NOTIFY：**

```sql
-- 发布事件（应用层监听）
SELECT pg_notify('job_created', '123');

-- 应用层监听（无需轮询）
LISTEN job_created;
```

**性能对比：**

| 指标 | Kafka | PostgreSQL SKIP LOCKED |
|------|-------|------------------------|
| 吞吐量 | 百万级/秒 | 万级/秒 |
| 延迟 | 毫秒级 | 毫秒级 |
| 运维负担 | 高（Zookeeper + Broker） | 无额外负担 |
| 适用场景 | 流处理、日志收集 | 任务队列、事件驱动 |

**边界说明：**

- ✅ 适合：任务队列、后台作业、事件驱动架构
- ❌ 不适合：流处理、日志收集、百万级/秒吞吐量

---

## 四、实战替换：Elasticsearch（全文搜索）

### 传统架构

```
应用 → Elasticsearch → 同步管道 → PostgreSQL
```

**问题：**
- ES 集群运维复杂
- 需要数据同步管道
- 数据一致性延迟

### PostgreSQL 方案

**使用 GIN 索引 + tsvector：**

```sql
-- 创建全文搜索索引
CREATE INDEX idx_search ON products
USING GIN (to_tsvector('english', name || ' ' || description));

-- 全文搜索
SELECT * FROM products
WHERE to_tsvector('english', name || ' ' || description)
      @@ to_tsquery('laptop & portable');
```

**高级搜索功能：**

```sql
-- 带权重搜索（标题权重更高）
SELECT * FROM products
WHERE to_tsvector('english', 
      setweight(to_tsvector('english', name), 'A') ||
      setweight(to_tsvector('english', description), 'B')
      ) @@ to_tsquery('laptop & portable');

-- 高亮显示匹配词
SELECT ts_headline('english', description, 
       to_tsquery('laptop & portable'),
       'StartSel=<b>, StopSel=</b>') AS highlighted
FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('laptop & portable');
```

**性能对比：**

| 指标 | Elasticsearch | PostgreSQL GIN |
|------|--------------|----------------|
| 索引规模 | 亿级 | 百万级 |
| 搜索延迟 | 毫秒级 | 毫秒级 |
| 运维负担 | 高（集群管理） | 无额外负担 |
| 适用场景 | 电商搜索、复杂分面 | 简单搜索、中小规模 |

**边界说明：**

- ✅ 适合：百万级记录、简单查询、关键词搜索
- ❌ 不适合：亿级数据、复杂分面、模糊匹配、相关性调优

---

## 五、实战替换：向量数据库（AI 嵌入存储）

### 传统架构

```
应用 → Pinecone/Weaviate → 向量检索
```

**问题：**
- 专用向量数据库成本高
- 需要额外维护一个服务
- 数据一致性难保证

### PostgreSQL 方案（pgvector 扩展）

**安装扩展：**

```sql
CREATE EXTENSION vector;
```

**创建向量表：**

```sql
CREATE TABLE embeddings (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)  -- OpenAI embedding 维度
);
```

**创建索引加速检索：**

```sql
-- ivfflat 索引：适合大规模向量检索
CREATE INDEX ON embeddings USING ivfflat (embedding vector_cosine_ops);
```

**向量相似度搜索：**

```sql
-- 余弦相似度搜索（找最相似的 5 条）
SELECT content, 1 - (embedding <=> '[0.1, 0.2, ...]') AS similarity
FROM embeddings
ORDER BY similarity DESC
LIMIT 5;

-- 带过滤条件的向量搜索
SELECT content, 1 - (embedding <=> '[0.1, 0.2, ...]') AS similarity
FROM embeddings
WHERE category = 'tech'
ORDER BY similarity DESC
LIMIT 5;
```

**性能对比：**

| 指标 | Pinecone | PostgreSQL pgvector |
|------|----------|---------------------|
| 向量维度 | 不限 | 建议<2000 |
| 检索延迟 | 毫秒级 | 毫秒级 |
| 运维负担 | 托管服务 | 无额外负担 |
| 成本 | 按向量数收费 | 包含在 PG 内 |

**边界说明：**

- ✅ 适合：中小规模向量检索（百万级）、AI 嵌入存储
- ❌ 不适合：十亿级向量、超低延迟需求

---

## 六、实战替换：地理空间（PostGIS）

**PostGIS 扩展：**

```sql
-- 安装扩展
CREATE EXTENSION postgis;

-- 查找附近 5 公里的餐厅
SELECT name, ST_Distance(
    location,
    ST_MakePoint(114.0579, 22.5431)::geography
) AS distance
FROM restaurants
WHERE ST_DWithin(
    location::geography,
    ST_MakePoint(114.0579, 22.5431)::geography,
    5000
)
ORDER BY distance;
```

**边界说明：**

- ✅ 适合：LBS 应用、附近搜索、地理围栏
- ❌ 不适合：专业 GIS 分析（需要独立 GIS 系统）

---

## 七、架构对比

### 传统微服务架构

```
┌─────────┐   ┌─────────┐   ┌──────────┐   ┌──────────┐
│  Redis  │   │  Kafka  │   │Elasticsearch│  │  Pinecone │
└────┬────┘   └────┬────┘   └─────┬────┘   └─────┬────┘
     │             │              │              │
     └─────────────┴──────────────┴──────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   PostgreSQL      │
                    └───────────────────┘
```

**问题：**
- 6 个独立服务要运维
- 数据一致性难以保证
- 每家云厂商都要收费
- 故障排查复杂度指数级增长

### PostgreSQL 单一体架构

```
┌───────────────────────────────────────┐
│           PostgreSQL                  │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────────┐ │
│  │JSONB│ │Vector│ │GIS  │ │Full-text│ │
│  └─────┘ └─────┘ └─────┘ └─────────┘ │
└───────────────────────────────────────┘
```

**优势：**
- 1 个服务运维
- ACID 事务保证一致性
- 单一账单
- 故障排查简单

### 对比表格

| 维度 | 传统微服务 | PostgreSQL 单一体 |
|------|-----------|------------------|
| 服务数量 | 6 个 | 1 个 |
| 运维复杂度 | 高 | 低 |
| 数据一致性 | 最终一致 | ACID 事务 |
| 成本 | 多家 SaaS 订阅 | 单一数据库 |
| 故障排查 | 指数级复杂 | 简单 |
| 扩展性 | 水平扩展容易 | 垂直扩展为主 |

![架构对比图](/images/postgresql-replaces-stack/architecture-comparison.png)

---

## 八、什么时候不该用 PostgreSQL 单一体

### 适合的场景

- ✅ 初创公司（0→1 阶段）
- ✅ 中小规模应用（日活 < 100 万）
- ✅ 团队规模小（< 10 人）
- ✅ 快速迭代验证产品

### 不适合的场景

- ❌ 超大规模（日活 > 1000 万）
- ❌ 需要亚毫秒级缓存（高频交易）
- ❌ 复杂搜索需求（电商搜索、推荐系统）
- ❌ 全球分布式部署（多区域低延迟）

**核心判断：**

**过度工程化是初创公司最昂贵的自我欺骗。**

当你只有 10 个用户时，不需要 10 个微服务。

---

## 九、迁移检查清单

### 迁移前评估

```
[ ] 当前架构有多少独立服务？
[ ] 每个服务的运维成本（时间 + 金钱）？
[ ] 数据一致性要求多高？
[ ] 团队有多少人能运维这些服务？
[ ] 哪些服务可以被 PostgreSQL 替换？
```

### 5 步迁移步骤

1. **评估现有依赖** —— 列出所有外部服务
2. **识别可替换项** —— 对照本文的替换方案
3. **渐进式迁移** —— 先替换非核心服务（如缓存）
4. **性能基准测试** —— 迁移前后对比
5. **回滚方案** —— 保留原服务直到验证完成

### 迁移后验证

```
[ ] 查询延迟是否在可接受范围？
[ ] 数据库负载是否正常？
[ ] 运维复杂度是否降低？
[ ] 成本是否下降？
[ ] 团队是否适应新架构？
```

### 回滚方案

- 保留原服务直到验证完成
- 使用双写策略（同时写入新旧系统）
- 准备快速回滚脚本
- 设置监控告警

---

## 十、最后

回到开篇的判断：

**现代软件工程已经变成了订阅管理模拟器。**

我们被云厂商洗脑了，以为需要 Redis、Kafka、Elasticsearch、Pinecone……才能构建一个"专业"的应用。

但真相是：

**PostgreSQL 不是数据库，是一个数据平台。**

**一个久经考验的工具可以吞噬你的整个架构。**

下次设计架构前，先问自己：
- 我真的需要这个独立服务吗？
- PostgreSQL 能替换它吗？
- 我是在解决问题，还是在制造复杂度？

从订阅管理模拟器到简单有效，这才是架构设计该有的样子。
