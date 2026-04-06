---
title: "Redis 最容易踩的 7 个坑，问题往往不在 Redis"
date: 2026-04-04T12:10:00+08:00
draft: false
author: "任博"
description: "Redis 本身不难，难的是很多问题都不是命令问题，而是设计问题。大 Key、缓存穿透、雪崩、热 Key、持久化、淘汰策略和集群倾斜，这 7 个坑我按线上后果重新梳理了一遍。"
tags: ["Redis", "性能优化", "后端开发", "架构设计", "技术总结"]
categories: ["技术实战"]
featured: false
toc: true
video_source: "https://www.douyin.com/video/7515007852897946880"
---

很多人觉得 Redis 没什么门槛。

命令不难，接入也快，压测时还总能打出很漂亮的数字。于是项目上线后，大家默认它“应该不会出事”。

但我看过不少 Redis 问题，最后都不是 Redis 自己不行，而是前面的设计太随便：Key 怎么拆，TTL 怎么设，热点怎么扛，内存打满以后怎么办，这些事一开始没想清楚，后面都会用线上故障补课。

我最近刷到一条讲 Redis 避坑的短视频，里面提到的点很典型。我没有照着转写，而是按“线上最容易出事故的顺序”重新整理成这篇文章。

如果你只记一句话，那就是：

**Redis 真正危险的坑，大多不是命令层面的，而是设计层面的。**

![Redis 常见架构陷阱配图：中心是 Redis，周围是 Big Key、TTL、Hot、OOM 等风险信号](/images/redis-pitfalls-cover.svg)

<!--more-->

---

## 1. 大 Key：看着只是大一点，用起来会越来越重

很多人第一次听到“大 Key”，会误以为只是 value 稍微大一点。

其实麻烦根本不在“占空间”，而在它会把读、写、删、迁移这几件事一起拖慢。

日常排查里，我一般会先用一个很粗但实用的判断标准：

- 字符串值超过 10KB
- Hash、List、Set、ZSet 的元素数量超过 5000

这不是官方红线，但拿来做预警很够用。

### 它为什么危险

大 Key 最常见的三个后果：

1. 一次读取就把网络带宽拉高
2. 集群里数据分布失衡，某个节点明显更重
3. 删除时阻塞主线程，尤其是直接 `DEL`

### 很多人会这样写

```bash
# ❌ 把整份用户资料塞进一个字段里
HSET user:1001 profile "巨大的 JSON 字符串..."

# ✅ 拆开存，至少让读写和更新有边界
HSET user:1001:name "张三"
HSET user:1001:email "zhangsan@example.com"
HSET user:1001:phone "13800138000"
```

很多人嘴上说 Redis 快，实际却把它当对象仓库乱塞。快不起来，很正常。

### 先查，再改

```bash
redis-cli --bigkeys
```

这条命令一点都不高级，但很值得养成习惯。它能很快告诉你：现在库里最大的那些雷，具体是谁。

处理方式通常就四种：

- 拆分存储
- 按需压缩，但别把压缩当万能药
- 用 `HSCAN` 代替 `HGETALL`
- 删除大 Key 优先用 `UNLINK`

如果你上线前只能做一次 Redis 巡检，我建议先跑它。

---

## 2. 缓存穿透：你在为不存在的数据持续付账

缓存穿透的本质不复杂：

请求查的是一个根本不存在的数据。Redis 没有，数据库也没有。于是每次请求都穿过去，最后由数据库兜底。

这类问题最糟糕的地方，是它很容易被恶意放大。

### 一个常见错误

```python
def get_user(user_id):
    data = redis.get(f"user:{user_id}")
    if not data:
        data = db.query("SELECT * FROM users WHERE id = ?", user_id)
    return data
```

如果数据库里也没有，下次还会继续穿。

### 最省事的办法：空值也缓存

```python
def get_user(user_id):
    data = redis.get(f"user:{user_id}")
    if data is None:
        data = db.query("SELECT * FROM users WHERE id = ?", user_id)
        if data:
            redis.setex(f"user:{user_id}", 300, data)
        else:
            redis.setex(f"user:{user_id}", 300, "")
    return data
```

重点不是空字符串本身，而是别让同一个不存在的数据反复穿库。

### 请求量再大，就上布隆过滤器

```python
from pybloom_live import BloomFilter

bloom = BloomFilter(capacity=1000000, error_rate=0.001)

for user_id in existing_ids:
    bloom.add(user_id)

def get_user(user_id):
    if user_id not in bloom:
        return None
    # 再走正常缓存和数据库流程
```

它不能保证“存在”，但能非常高效地挡掉“肯定不存在”的请求。

---

## 3. 缓存雪崩：不是缓存失效，而是失效得太整齐

很多雪崩事故的起点都很普通：大家偷懒，把一批 key 全写成同一个 TTL。

```python
redis.setex("product:1", 3600, data)
redis.setex("product:2", 3600, data)
redis.setex("product:3", 3600, data)
```

一小时后，它们会一起失效。然后数据库开始替缓存收拾残局。

### 最便宜的修法：TTL 加随机抖动

```python
import random

def set_cache(key, data, base_ttl=3600):
    random_ttl = base_ttl + random.randint(-360, 360)
    redis.setex(key, random_ttl, data)
```

这招不高级，但非常值。

### 另一种思路：核心热点数据不过期，改后台刷新

```python
redis.set("product:1", data)

def refresh_cache():
    products = db.query("SELECT * FROM products")
    for p in products:
        redis.set(f"product:{p.id}", p.data)
```

但这招别乱用。

“永不过期”听上去高级，实际如果刷新链路不稳，就是把脏数据永久化。只有那些允许短时间旧值、而且你能稳定刷新它的数据，才适合这么做。

---

## 4. 热 Key：不是 Redis 扛不住，是你把压力打在了一个点上

热 Key 的场景每个系统都有：

- 热搜榜
- 秒杀商品
- 活动页核心配置
- 突发事件详情页

问题不是 Redis 不够快，而是所有请求都打在同一个 key、同一个分片、同一个节点上。再快也顶不住你这么打。

### 常见做法一：读副本分流

```python
hot_data = redis.get("hot:product:1")
redis.set("hot:product:1:replica:1", hot_data)
redis.set("hot:product:1:replica:2", hot_data)

import random

def get_hot_product():
    replica = random.randint(1, 2)
    return redis.get(f"hot:product:1:replica:{replica}")
```

### 常见做法二：应用层本地缓存

```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def get_hot_data(key):
    return redis.get(key)
```

如果一个热点 key 在 1 秒内被打几万次，你就别只想着 Redis。应用层缓存、CDN、静态化、预计算，都应该一起考虑。

热 Key 本质上不是缓存问题，而是流量分配问题。

---

## 5. 持久化：RDB 和 AOF 都不是“打开就万事大吉”

我见过两种很常见的误区：

一种是把 Redis 当纯缓存，默认“丢了也没事”；另一种是只要开了持久化，就觉得安全问题已经解决了。

这两种都不太靠谱。

### RDB 的问题

- 快照之间的数据会有丢失窗口
- 数据量大时，`fork` 也可能带来卡顿

### AOF 的问题

- 文件会膨胀，需要定期重写
- 恢复时要回放日志，速度不一定理想

### 一个更稳妥的配置思路

```conf
appendonly yes
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

save 900 1
save 300 10
save 60 10000
```

真正该先想清楚的，不是“我要不要开持久化”，而是：

**这批数据到底能丢多少。**

能接受的数据丢失窗口不一样，配置就不该一样。

---

## 6. 内存淘汰策略：不配，等于把风险推给高峰期

Redis 默认是 `noeviction`。

意思很简单：内存满了以后，不淘汰，写请求直接报错。

开发环境里你可能感觉不到，线上一到高峰期就很要命。

### 常见选择

```conf
# 只淘汰设置了过期时间的 key
maxmemory-policy volatile-lru

# 所有 key 都参与淘汰
maxmemory-policy allkeys-lru
```

怎么选，不取决于教程怎么写，而取决于你 Redis 里到底放了什么：

- 如果它主要承担缓存，`allkeys-lru` 往往更省心
- 如果你明确区分了可淘汰缓存和不能乱动的数据，`volatile-lru` 更可控

我个人很反感那种“先不配，真满了再说”的做法。内存策略不是优化项，是上线前配置项。

### 至少盯住这两个指标

```bash
redis-cli INFO memory
CONFIG SET maxmemory 2gb
```

前者看现状，后者划边界。没有边界，后面很多所谓优化都只是安慰自己。

---

## 7. 集群倾斜：最后会把分布式用成单点受罪

很多团队一上 Redis Cluster，就默认容量和流量都会自动均匀摊开。

现实没这么理想。

如果大 Key、热 Key、业务分片规则都没设计好，最后常常会出现这样的场景：

```bash
127.0.0.1:7000> INFO memory
used_memory_human:8.0G

127.0.0.1:7001> INFO memory
used_memory_human:2.0G
```

这时候表面看像集群问题，根源其实还是 key 设计问题。

### 怎么排查

```bash
redis-cli --cluster check 127.0.0.1:7000
redis-cli --cluster reshard 127.0.0.1:7000
```

但要实话实说：`reshard` 只是补救，不是答案。

真正该提前做的是：

- 别把数据天然集中到某一类 key 规则里
- 提前控制大 Key
- 对热点流量有预案
- 定期看分布，而不是等报警后才看

---

## 上线前，最少过一遍这 6 项

```bash
# 1. 查大 Key
redis-cli --bigkeys

# 2. 看慢查询
SLOWLOG GET 10

# 3. 看内存
INFO memory

# 4. 看连接数
INFO clients

# 5. 看持久化状态
INFO persistence

# 6. 看集群状态
CLUSTER INFO
```

这份清单不花哨，但很有用。

很多 Redis 事故，并不是技术做不到，而是最基础的检查从来没人认真跑过。

---

## 再补 3 条很实用的建议

### 1. 能批量就别单条发

```python
# ❌ 逐条写
for i in range(1000):
    redis.set(f"key:{i}", value)

# ✅ 用 pipeline
pipe = redis.pipeline()
for i in range(1000):
    pipe.set(f"key:{i}", value)
pipe.execute()
```

Redis 很快，但网络往返次数太多，一样会被拖慢。

### 2. 别在生产里随手敲 `KEYS`

```python
# ❌ 容易阻塞
keys = redis.keys("user:*")

# ✅ 用 SCAN 逐步遍历
keys = []
for key in redis.scan_iter("user:*"):
    keys.append(key)
```

`KEYS` 在本地调试很好用，在线上往往是事故前奏。

### 3. TTL 别偷懒写成统一模板

```python
redis.setex("session:abc", 1800, data)      # 30 分钟
redis.setex("captcha:xyz", 300, data)       # 5 分钟
redis.setex("config:global", 86400, data)   # 24 小时
```

Session、验证码、全局配置，本来就不是一类东西。TTL 一刀切，通常只是因为前面没想清楚。

---

## 最后

Redis 最容易误导人的地方，是它在一切正常的时候看起来太轻松了。

命中率高、QPS 好看、延迟也不高，于是团队很容易误以为“这块已经稳了”。

但很多真正危险的问题，平时是看不出来的。只有流量上来、缓存同时失效、内存逼近上限、热点集中到某个节点的时候，它们才会一起露头。

所以别只看 Redis 快不快，也别只看命中率漂不漂亮。

更重要的是把这几件事提前想清楚：

- 有没有大 Key
- 缓存失效会不会一起爆
- 热点到底打在哪
- 内存打满以后怎么办
- 哪些数据可以丢，哪些不能丢

这些问题想明白了，Redis 才算真的用明白。

---

## 参考资料

- [Redis 官方文档](https://redis.io/docs/)
- [Redis 设计与实现](http://redisbook.com/)
- 视频来源：[程序员 Mike - 抖音](https://www.douyin.com/video/7515007852897946880)
