---
title: "PostgreSQL 入门：为什么说它能装下世间万物？"
date: 2026-04-07T12:30:00+08:00
draft: false
author: "任博"
description: "PostgreSQL 的能力早已超越了传统关系型数据库的范围。从 JSON 支持到全文检索，从向量数据库到定时任务，从缓存表到 RESTful API——很多原本需要多个组件协作才能完成的功能，PostgreSQL 一个数据库就能搞定。"
tags: ["PostgreSQL", "数据库", "后端开发", "技术入门", "架构设计"]
categories: ["技术实战"]
featured: false
toc: true
---

这是世界上最先进的开源关系型数据库。

PostgreSQL 的能力早已超越了传统关系型数据库的范围。现代开发中，为了解决各种问题，各类花哨的工具层出不穷——缓存用 Redis、全文检索用 Elasticsearch、文档存储用 MongoDB、向量数据库用专门的方案、定时任务跑在 K8s 上。

但有时候，仅仅一个 PostgreSQL，就能覆盖大部分后端开发需求。

<!--more-->

## 为什么是 PostgreSQL？

与 MySQL"快速易用"的卖点不同，PostgreSQL 的开发理念是打造出功能最丰富、最符合 SQL 标准的数据库。凭借着每年一个版本的稳定迭代、高标准的 SQL 兼容性、高可用性、高扩展性和自定义能力，在近几年 Stack Overflow 的开发者调查中，PostgreSQL 已渐渐超越 MySQL，登顶最受开发者欢迎的数据库。

核心差异在于：**PostgreSQL 比 MySQL 更符合 SQL 标准**，尤其是在 SQL 语言特性和高级功能方面。比如 PostgreSQL 支持把表结构修改语句纳入事务管理、支持部分索引、支持可延迟约束等。

下面我们从安装开始，一步步看看 PostgreSQL 到底强在哪里。

## 安装与远程连接

以 Ubuntu 系统为例：

```bash
# 更新 apt 索引
sudo apt update

# 安装 PostgreSQL 及官方插件包
sudo apt install postgresql postgresql-contrib
```

安装完成后系统会自动创建一个 `postgres` 用户，切换到该用户并进入命令行环境：

```bash
sudo -i -u postgres
psql
```

设置初始密码：

```sql
\password postgres
```

### 配置远程连接

为了方便本地调试，需要放开数据库的远程连接功能。

在 `psql` 中查看配置文件路径：

```sql
SHOW config_file;
SHOW hba_file;
```

退出 `psql`，使用 `vi` 修改第一个配置文件，找到 `listen_addresses`，改为：

```
listen_addresses = '*'
```

表示监听所有可用网络。

修改第二个配置文件（pg_hba.conf），在 IPv4 部分添加：

```
host    all             all             0.0.0.0/0               md5
```

表示允许所有 IP 地址的客户端连接。

最后重启数据库：

```bash
sudo systemctl restart postgresql
```

**注意：** 还需在服务器防火墙放行 5432 端口。

### 使用 DBeaver 连接

DBeaver 是一款开源免费的数据库工具。创建 PostgreSQL 连接时，输入服务器 IP、端口 5432、用户名 postgres 和刚才设置的密码即可。

连接成功后会发现，PostgreSQL 比 MySQL 多了一个层级：**Database → Schema → Table**（三级结构），而 MySQL 只有 Database → Table（两级）。这是两者的显著区别之一。

## 对象关系型数据库

PostgreSQL 官网对自己的定义是：**Open Source Object-Relational Database**（开源对象关系型数据库）。

关键词是"对象"与"关系"。PostgreSQL 既有传统数据库中表、行、列等严谨的关系形式，又吸收了面向对象编程的思想，在数据库层面直接提供对象支持。

### 丰富的内置数据类型

PostgreSQL 提供了上百种内置数据类型，每种数据类型就是一个内置的数据库对象。仅时间相关的就有多种：带时区/不带时区、带日期/不带日期等。

一些特色数据类型：

- **网络类型**：`cidr`、`inet`、`macaddr` 等，用于存储网段、IP 地址、MAC 地址
- **几何类型**：点、线、线段、矩形、路径、圆形等，支持几何运算和查询
- **数组类型**：支持一维或多维数组作为字段类型

示例：使用 `cidr` 类型存储网段

```sql
CREATE TABLE networks (
    id serial PRIMARY KEY,
    network cidr
);

INSERT INTO networks (network) VALUES ('192.168.1.0/24');

-- 查询包含某个子网的网段（>> 是包含运算符）
SELECT * FROM networks WHERE network >> '192.168.1.5'::inet;
```

### 自定义类型

除了内置类型，PostgreSQL 还支持使用 `CREATE TYPE` 自定义类型：

```sql
CREATE TYPE employee AS (
    name text,
    age integer,
    skills text[]
);

CREATE TABLE employees (
    id serial PRIMARY KEY,
    info employee
);

INSERT INTO employees (info) VALUES (ROW('张三', 30, ARRAY['Java', 'Python'])::employee);
```

查询时可以直接展开自定义类型中的字段，技能字段本身又是一个数组。这种设计让数据库能够更好地存储复杂的数据结构。

### 表继承

传统关系型数据库中，数据存储在扁平的二维表格里。而面向对象编程中，对象之间通过组合、继承形成复杂的网状层次关系。把复杂对象"压扁"到二维表格中，是一个繁琐、易出错且效率低下的工作——这个问题被称为**阻抗失配（Impedance Mismatch）**。

PostgreSQL 在数据库层面引入对象概念，支持表之间定义继承关系：

```sql
CREATE TABLE employees (
    id serial PRIMARY KEY,
    name text,
    age integer
);

CREATE TABLE developers (
    programming_languages text[]
) INHERITS (employees);

INSERT INTO developers (name, age, programming_languages) 
VALUES ('李四', 28, ARRAY['Go', 'Rust']);
```

`developers` 表继承了 `employees` 表的所有字段，并添加了自己的独特字段。插入到 `developers` 表的数据也会同步出现在 `employees` 表中。这个特性模拟了面向对象编程中的多态性，更好地反映了类之间的继承关系。

## JSON 支持：替代 MongoDB 的可能

SQL 数据库与 NoSQL 数据库的一个核心争论点是：NoSQL 可以定义非结构化数据（如每条记录的 JSON 格式都可能不同），而传统 SQL 数据库有固定的列格式。

但 PostgreSQL 原生支持 JSON 数据类型，很大程度上能够替代 MongoDB 的使用场景。

```sql
CREATE TABLE http_logs (
    id serial PRIMARY KEY,
    data jsonb
);

-- 插入灵活的 JSON 数据（可嵌套、可包含数组）
INSERT INTO http_logs (data) VALUES (
    '{"remote_addr": "192.168.1.1", "status": 200, "request_data": {"method": "GET"}}'
);

-- 查询 JSON 内部字段（->> 是提取运算符）
SELECT * FROM http_logs WHERE data->>'remote_addr' = '192.168.1.1';
```

`jsonb` 是二进制类型的 JSON，是 PostgreSQL 中存储 JSON 数据的最佳类型。

### JSON 索引优化

对于频繁查询的 JSON 字段，可以建立索引提升效率：

```sql
-- 表达式索引：对提取的字段建索引
CREATE INDEX idx_remote_addr ON http_logs ((data->>'remote_addr'));

-- GIN 索引：对整个 JSON 字段建索引，大幅加快内部数据查询
CREATE INDEX idx_data_gin ON http_logs USING GIN (data);

-- 使用 GIN 索引查询
SELECT * FROM http_logs WHERE data @> '{"status": 200}';
SELECT * FROM http_logs WHERE data ? 'request_data';
```

当然，是否用 PostgreSQL 替换 MongoDB，还需根据项目具体情况决定。

## 全文检索：原生方案与插件

对文本进行全文检索是常见需求。比如搜索包含"Powerful"这个单词的句子，SQL 的 `LIKE` 语法无法满足需求（无法利用索引，只能全表扫描，性能极低）。

通常的解决方案是 Elasticsearch，但它部署繁琐且增加系统复杂度。PostgreSQL 提供了不错的原生替代方案。

### 原生全文检索

```sql
CREATE TABLE documents (
    id serial PRIMARY KEY,
    content text,
    tsv tsvector GENERATED ALWAYS AS (to_tsvector('english', content)) STORED
);

-- 在 tsvector 列上建 GIN 索引
CREATE INDEX idx_tsv ON documents USING GIN (tsv);

-- 搜索（@@ 表示包含）
SELECT * FROM documents WHERE tsv @@ to_tsquery('english', 'Powerful');
```

`tsvector` 是 PostgreSQL 独有的文本搜索向量类型。`to_tsvector` 函数会自动进行分词、转小写、移除冠词/介词等意义不大的单词，还能进行词干提取（还原成词根形式）。

**限制：** 原生分词器只支持英文、法语、德语、西班牙语等，不支持中文。

### 中文全文检索：pg_trgm 插件

PostgreSQL 有非常完善的插件生态。安装 `pg_trgm` 插件可实现中文全文检索：

```bash
# 安装插件（以 Ubuntu 24.04 为例）
sudo apt install postgresql-16-pgtrgm

# 在数据库中启用
CREATE EXTENSION pg_trgm;

# 在文本字段上直接创建索引
CREATE INDEX idx_content_trgm ON documents USING GIN (content gin_trgm_ops);

-- 使用 % 操作符进行模糊匹配
SELECT * FROM documents WHERE content % '引擎';
```

通过安装插件，PostgreSQL 轻松添加了中文全文搜索能力。

## 向量数据库：支撑 RAG 系统

当前大模型知识库最常用的方法是 **RAG（检索增强生成）**。整个流程是：

1. 将资料拆分成文本块
2. 使用嵌入模型对文本块进行向量化（变成长数字序列）
3. 将向量及对应文本保存在向量数据库中
4. 用户提问时，将问题也向量化，与向量数据库进行相似度匹配
5. 选出匹配度最高的原文片段，连同问题一起发给大模型做归纳总结

在 RAG 系统中，向量数据库的选择至关重要。PostgreSQL 通过 `pgvector` 插件可以胜任这一角色。

### 安装与使用

```bash
# 安装 pgvector 插件
sudo apt install postgresql-16-pgvector

# 在数据库中启用
CREATE EXTENSION vector;
```

创建向量表：

```sql
CREATE TABLE embeddings (
    id serial PRIMARY KEY,
    embedding vector(1024)  -- 1024 维向量
);

INSERT INTO embeddings (embedding) VALUES ('[0.1, 0.2, 0.3, ...]');
```

### 配合 LangChain 搭建 RAG

在 LangChain 框架中，`ChatLangChain` 模块提供了使用 PostgreSQL 向量数据库的示例：

```bash
# 安装依赖
pip install langchain-postgres langchain-openai
```

配置环境变量（API Key、API 端点、模型名称、数据库连接地址），创建向量存储库，添加测试数据后即可进行相似度搜索：

```python
results = vector_store.similarity_search_with_score("可爱动物在水里", k=10)
```

分数越低表示向量距离越近、相似度越高。找到最相似的几条数据后，发给大模型做总结，一个 RAG 系统就搭建完成了。

## 定时任务：一行 SQL 搞定

实际开发中经常需要编写定时任务（如定时数据清理）。传统方案是用 Java/Python 写定时脚本，打包成 Docker 镜像，跑在 K8s 上，还要分配专人维护。

PostgreSQL 使用 `pg_cron` 插件，一行 SQL 就能搞定：

```bash
# 安装 pg_cron（注意版本号）
sudo apt install postgresql-16-pg-cron

# 修改 postgresql.conf，添加：
shared_preload_libraries = 'pg_cron'
cron.database_name = 'postgres'

# 修改 pg_hba.conf，添加本地免密验证
host    all             all             127.0.0.1/32            trust

# 重启数据库后启用插件
CREATE EXTENSION pg_cron;
```

创建定时任务：

```sql
SELECT cron.schedule(
    'backup-documents',           -- 任务名
    '0 3 * * *',                  -- cron 表达式（每天凌晨 3 点）
    $$
        INSERT INTO documents_archive SELECT * FROM documents;
        TRUNCATE documents;
    $$
);
```

修改定时任务：

```sql
SELECT cron.update_job_schedule(
    'backup-documents',
    '* * * * *'  -- 改为每分钟执行一次
);
```

在 `cron` schema 中，`job` 表列出自定义的定时任务，`job_run_details` 表记录执行历史（命令、结果、返回值等）。

## 缓存表：替代 Redis 的可能

Redis 是高效的内存数据库，一般用作缓存。PostgreSQL 可以通过 `UNLOGGED` 表实现类似的轻量级高速缓存功能。

```sql
CREATE UNLOGGED TABLE cache_data (
    id serial PRIMARY KEY,
    key text,
    value text,
    expires_at timestamp
);
```

`UNLOGGED` 关键字让表不写入 WAL 日志，可提高约 5 倍写入速度。代价是一旦数据库断电或崩溃，表中数据全部丢失——这正好符合缓存的使用场景。

配合以下优化：

- 调大 PostgreSQL 的共享缓存区（`shared_buffers`），让经常读写的数据保存在内存中
- 搭配 `pg_cron` 定时清理过期缓存

这样就能获得一个性能不错的内存缓存表，在一定程度上替代 Redis 的功能。

## 数据库即 API：PostgREST

PostgreSQL 是开源软件，配合 PostgREST 可以直接把数据库转换成 RESTful API。数据库的表结构决定了 API 的端点和操作。

### 配置步骤

创建只读角色：

```sql
CREATE ROLE api_user;
GRANT USAGE ON SCHEMA public TO api_user;
GRANT SELECT ON documents TO api_user;
```

创建配置文件 `postgrest.conf`：

```
db-uri = "postgres://api_user:password@server_ip:5432/database_name"
db-schema = "public"
db-anon-role = "api_user"
```

启动 PostgREST：

```bash
./postgrest postgrest.conf
```

访问 `http://localhost:3000/documents` 即可通过 API 获取表中数据。配置 `UPDATE` 或 `INSERT` 权限后，还可以使用 PUT/POST 接口增改数据。

无需编写业务代码，直接把 PostgreSQL 转换成 RESTful API。PostgREST 还支持过滤、排序、分组、权限验证等进阶功能。

## 更多扩展能力

PostgreSQL 的扩展生态非常丰富，以下是一些值得关注的方向：

### pg_graphql

让 PostgreSQL 获得 GraphQL API 能力：

```sql
CREATE EXTENSION pg_graphql;

-- 使用 GraphQL 查询
SELECT graphql.resolve($$
    {
        documents {
            edges {
                node {
                    id
                    content
                }
            }
        }
    }
$$);
```

### AGE（Apache AGE）

允许使用 Cypher 语言建立图数据，处理好友关系网、电网关系等场景。有了这个扩展，无需再安装专门的图数据库。

### TimescaleDB

PostgreSQL 的时序数据库扩展，适合处理物联网、金融数据等对时序要求非常高的数据。

---

## 结语

PostgreSQL 的能力早已超越了传统关系型数据库的范围，更像一个**全能的后端聚合平台**。

从 JSON 支持到全文检索，从向量数据库到定时任务，从缓存表到 RESTful API，再到图数据和时序数据——很多原本需要多个组件协作才能完成的功能，PostgreSQL 一个数据库就能搞定。

这并不意味着要用 PostgreSQL 替换所有专用工具。但在很多场景下，它确实能简化架构、降低运维成本、减少技术栈复杂度。

下次设计系统架构时，不妨先问问自己：**这个问题，PostgreSQL 能不能解决？**

很多时候，答案会是肯定的。
