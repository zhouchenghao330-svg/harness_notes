# Redis 基础、使用理念与 Python 客户端

> 目标：不是死背 Redis 命令，而是建立正确的数据模型、使用边界和故障意识；具体代码可以借助 AI 生成，但必须能够判断它是否正确。

## 一、Redis 的核心心智模型

Redis 可以先理解为一台独立运行的、速度很快的“远程字典服务器”。

更准确地说，Redis 是数据结构服务器：key 用于定位数据，value 可以是 String、Hash、List、Set、Sorted Set 等数据结构。

```text
Python 服务 A ─┐
Python 服务 B ─┼── 网络请求 ──> Redis
Java 服务 C   ─┘
```

示例数据：

```text
key                         value
------------------------------------------------
user:1:name                 "张三"
article:1001:views          99
sms:13800138000             "482913"
article:1001:likes          {user1, user2, user3}
rank                        [(张三, 100), (李四, 200)]
```

Redis 命令的基本形式是：

```text
命令 key 参数
```

例如：

```redis
SET user:1:name 张三
GET user:1:name
```

- `SET`、`GET` 是 Redis 命令。
- `user:1:name` 是 key。
- `张三` 是 value。
- key 中的冒号只是常用命名约定，不是真正的目录层级。

普通 Python 字典只属于当前 Python 进程；Redis 是独立服务，多个程序和多台机器都可以通过网络访问它。

---

## 二、必须掌握的基础命令

### 写入和读取

```redis
SET user:1:name 张三
GET user:1:name
```

### 判断、删除和查看类型

```redis
EXISTS user:1:name
DEL user:1:name
TYPE user:1:name
```

### 设置和查看过期时间

```redis
EXPIRE user:1:name 300
TTL user:1:name
```

`TTL` 常见返回值：

- 正数：还剩多少秒。
- `-1`：key 存在，但没有过期时间。
- `-2`：key 不存在。

写入时直接设置过期时间：

```redis
SET sms:13800138000 482913 EX 300
```

含义是保存验证码 `482913`，并在 300 秒后自动过期。

线上不要随便使用：

```redis
KEYS *
```

数据量大时它会遍历全部 key，并可能阻塞服务。生产环境一般使用渐进式的 `SCAN`。

---

## 三、五种最重要的数据类型

### 1. String

可以存普通字符串、数字、JSON 文本和二进制数据。

```redis
SET user:1:name 张三
GET user:1:name
```

计数器：

```redis
SET article:1001:views 0
INCR article:1001:views
GET article:1001:views
```

`INCR` 是 Redis 的原子操作。并发请求不应该先 `GET`、在应用里加一、再 `SET`，那样容易出现竞争。

### 2. Hash

Hash 类似 Python 字典，适合保存对象的多个字段：

```redis
HSET user:1 name 张三 age 18 city 北京
HGET user:1 name
HGETALL user:1
```

它可以单独修改一个字段，而不必重新写入整个对象。

### 3. List

List 有顺序、允许重复，支持从两端插入和删除：

```redis
LPUSH queue task1
RPUSH queue task2
LPOP queue
RPOP queue
LRANGE queue 0 -1
```

可以做简单队列，但如果业务需要可靠确认、重试和消费组，通常考虑 Redis Stream、RabbitMQ 或 Kafka。

### 4. Set

Set 是无序、不重复的集合：

```redis
SADD article:1001:likes user1 user2 user3
SISMEMBER article:1001:likes user1
SMEMBERS article:1001:likes
```

集合交集：

```redis
SINTER user:1:follows user:2:follows
```

适合点赞去重、黑名单、标签、抽奖去重和共同好友。

### 5. Sorted Set / ZSet

ZSet 的成员不能重复，每个成员都有一个用于排序的 score：

```redis
ZADD rank 100 张三
ZADD rank 200 李四
ZADD rank 150 王五
ZRANGE rank 0 9 REV WITHSCORES
```

适合排行榜、热搜榜、成绩排名和部分延迟任务场景。

旧资料常用 `ZREVRANGE`，较新的 Redis 更推荐 `ZRANGE ... REV`。

---

## 四、Redis 为什么快

Redis 快不只是因为数据在内存里，还包括：

- 主要数据存放在内存中。
- 底层数据结构针对具体操作做了优化。
- 大多数命令执行时间很短。
- 使用事件驱动和 I/O 多路复用处理大量客户端连接。
- 核心命令通常串行执行，减少锁竞争和线程切换。

可以把它和 Python event loop 类比：

```text
客户端 A 的网络数据还没到
→ 不一直等待 A
→ 先处理 B、C
→ A 的数据准备好后再处理 A
```

“Redis 是单线程”是面试中的简化表达。更准确地说，Redis 的核心命令执行长期采用单线程模型，但整个 Redis 进程并非只有一个线程，网络 I/O 和后台持久化等工作也可能涉及其他线程或进程。

因为命令执行通常是串行的，一条特别慢的命令仍可能拖住其他请求。

---

## 五、Redis 作为缓存时的基本流程

```text
请求商品 42
    ↓
先读取 Redis 的 product:42
    ↓
缓存命中？
├── 是：直接返回
└── 否：查询 MySQL
         ↓
       写入 Redis，并设置 TTL
         ↓
       返回结果
```

这种模式叫 Cache Aside，也叫旁路缓存。

伪代码：

```python
product = await redis.get("product:42")

if product is None:
    product = await mysql.get_product(42)
    await redis.set("product:42", serialize(product), ex=300)

return product
```

在这种架构中，MySQL 通常是事实数据源，Redis 是可以重建的加速层。

---

## 六、缓存穿透、击穿和雪崩

| 问题 | 含义 | 核心关键词 |
| --- | --- | --- |
| 缓存穿透 | 查询 Redis 和数据库里都不存在的数据 | 不存在 |
| 缓存击穿 | 一个超级热点 key 突然失效 | 一个热点 |
| 缓存雪崩 | 大量 key 同时失效或 Redis 整体不可用 | 大批量 |

### 缓存穿透

每次查询不存在的数据都会继续访问数据库。

常见方案：参数校验、短暂缓存空结果、布隆过滤器和接口限流。

### 缓存击穿

一个热点 key 过期，大量并发请求一起查询数据库。

常见方案：互斥重建、提前刷新、逻辑过期和热点数据不设置物理过期。

### 缓存雪崩

大量 key 在同一时刻过期，或者 Redis 集体不可用，请求集中落到数据库。

常见方案：给 TTL 增加随机值、多级缓存、限流降级和 Redis 高可用。

---

## 七、过期删除和内存淘汰

### 过期删除

key 被设置了 TTL，使用期限到达以后，Redis 通过访问时检查和定期检查等方式清理。

### 内存淘汰

Redis 达到 `maxmemory` 限制后，即使一些 key 尚未过期，也可能需要释放空间。

常见策略：

- `noeviction`：不删除，新的写入报错。
- `allkeys-lru`：从全部 key 中淘汰最近较少使用的 key。
- `allkeys-lfu`：从全部 key 中淘汰访问频率较低的 key。
- `volatile-lru`：只从设置了过期时间的 key 中选择。

```text
过期删除：key 的使用期限到了
内存淘汰：Redis 内存不够，需要腾位置
```

---

## 八、RDB 和 AOF 持久化

### RDB

RDB 是某一时刻的内存快照，像定期拍照。

优点：文件紧凑、适合备份、恢复较快。

缺点：两次快照之间的数据可能丢失；大数据集执行 `fork()` 时也可能产生延迟。

### AOF

AOF 记录会修改数据的写命令，重启时通过重放命令恢复状态，像操作流水。

常见的每秒 `fsync` 策略在极端故障时可能丢失大约一秒数据。Redis 7 之后使用多文件 AOF 机制。

RDB 和 AOF 可以同时开启；纯缓存场景也可以不使用持久化。

---

## 九、分布式锁的正确认识

基础加锁形式：

```redis
SET lock:order:1001 随机唯一值 NX PX 30000
```

- `NX`：key 不存在时才设置成功。
- `PX 30000`：30 秒后自动过期。
- 随机唯一值：标记锁属于哪个客户端。

释放锁时不能无条件 `DEL`。自己的锁可能已经过期，另一个客户端已经获得新锁；此时直接删除会误删别人的锁。

正确做法是原子地完成：

```text
锁里的唯一值仍然是我的？
→ 是：删除
→ 否：不能删除
```

旧版本通常使用 Lua 脚本；Redis 8.4 还提供 `DELEX ... IFEQ`。实际项目优先使用成熟客户端实现，不要随手拼接分布式锁。

---

## 十、主从、Sentinel 和 Cluster

```text
主从复制
→ 数据副本

Sentinel 哨兵
→ 主节点故障后的自动切换

Redis Cluster
→ 数据分片、容量扩展和集群高可用
```

Redis Cluster 把 key 映射到 16384 个哈希槽，再由不同节点负责不同槽位。

Redis 复制通常是异步的，因此从节点的数据可能短暂落后；故障切换也不能直接等同于严格零丢失。

---

## 十一、Redis 和 MySQL 的一致性

Cache Aside 中常见的写入方式是：

```text
更新 MySQL
→ 删除 Redis 缓存
→ 下一次查询时重新加载缓存
```

它不是严格强一致方案，仍可能遇到：

- 数据库更新成功，但删除缓存失败。
- 并发请求在特殊时序下写回旧数据。
- 网络中断或服务崩溃。

生产系统可能还要加入删除失败重试、消息队列、Outbox、binlog/CDC 监听和合理 TTL。

面试时应表述为：

> 先更新数据库再删除缓存，是 Cache Aside 中常见的最终一致性基线；要配合重试或消息机制处理缓存删除失败，不能宣称它绝对强一致。

---

## 十二、Python 的 Redis SDK：redis-py

Python 官方推荐的主流 Redis 客户端是 `redis-py`：

```bash
pip install redis
```

### 同步客户端

```python
import redis

r = redis.Redis(
    host="localhost",
    port=6379,
    decode_responses=True,
)

r.set("user:1:name", "张三")
name = r.get("user:1:name")
```

`decode_responses=True` 会把常见返回值从 bytes 解码成字符串。

### asyncio 异步客户端

```python
import redis.asyncio as redis

async def example():
    async with redis.Redis(
        host="localhost",
        port=6379,
        decode_responses=True,
    ) as r:
        await r.set("sms:13800138000", "482913", ex=300)
        code = await r.get("sms:13800138000")
        return code
```

`redis.asyncio` 的 API 和同步客户端基本对应，只是 Redis 命令需要 `await`。

在 FastAPI 等长期运行的异步应用中，推荐：

- 应用启动时创建一个 Redis client。
- 多个请求共享这个 client 及其内部连接池。
- 不要每个请求都新建一个 `Redis()`。
- 应用关闭时执行 `await client.aclose()`。
- 配置连接和命令超时、连接池上限，并处理连接失败。

---

## 十三、redis-py 和 ORM 是不是一回事

不是。

### redis-py 更像数据库驱动

```python
await r.set(...)
await r.get(...)
await r.hset(...)
await r.zadd(...)
```

它把 Python 方法转换成 Redis 命令，并管理连接池、编码、响应和错误。

它更接近 MySQL 的数据库驱动，而不是 SQLAlchemy ORM。

### Redis OM 才更像对象映射层

Redis 官方生态里有 Redis OM for Python，可以使用 Python/Pydantic 风格的模型保存和查询文档。

但是它不能替代对 Redis 数据结构的理解：

- Redis 的核心价值包括 TTL、计数器、集合、排行榜、队列和原子命令。
- 很多 Redis 用法并不是“把对象做 CRUD”。
- JSON 文档查询等 Redis OM 场景还可能依赖 Redis Stack、JSON 和 Search 能力。
- 对缓存和中间件场景，直接使用 redis-py 通常更透明、更常见。

因此学习顺序应该是：

```text
先理解 key、数据类型、TTL 和原子命令
→ 使用 redis-py
→ 真的需要文档模型和复杂查询时，再评估 Redis OM
```

不要为了看起来像 MySQL 而强行给 Redis 套 ORM。

---

## 十四、AI 时代应该学什么

结论：不需要死记大量 API，但不能完全不看代码。

AI 很擅长生成：

- 连接 Redis 的样板代码。
- `get`、`set`、`hset` 等方法调用。
- FastAPI 生命周期集成。
- Docker Compose 和基础配置。
- 测试代码。

人必须理解并决定：

1. Redis 在这个业务中解决什么问题。
2. 数据应该选择 String、Hash、Set 还是 ZSet。
3. key 如何命名，是否需要 TTL。
4. 哪些操作必须原子化，是否存在并发竞争。
5. Redis 是事实数据源还是可重建缓存。
6. 缓存失效、连接失败、超时和重试时会发生什么。
7. 内存满了以后采用什么淘汰策略。
8. 是否需要持久化、主从、Sentinel 或 Cluster。
9. 客户端是否复用连接池，是否设置超时和资源上限。
10. 线上操作会不会出现大 key、热 key 或阻塞命令。

Redis 的危险点往往不是“代码能不能跑”，而是：

- 没有 TTL，内存持续增长。
- 在异步服务里用了同步客户端，阻塞 event loop。
- 每个请求创建一个新连接，连接数暴涨。
- 用 `GET` 加一再 `SET`，产生并发丢失更新。
- 分布式锁直接 `DEL`，误删其他客户端的锁。
- 使用 `KEYS *` 或大范围操作，阻塞服务。
- 把缓存误当成绝对可靠的事实数据源。
- 只考虑正常流程，没有考虑 Redis 超时或不可用。

这些问题 AI 生成的代码未必会主动替业务做出正确选择。

比较合理的学习精力分配：

```text
50%：使用理念、数据建模和业务边界
30%：并发、一致性、故障和线上注意点
20%：常用命令和客户端代码
```

代码至少要达到“能读、能审、能改、能验证”的水平，而不是完全交给 AI 后不理解。

---

## 十五、是否还需要读原文

不需要逐字精读或死背。当前这份笔记已经覆盖了原文主要内容，并修正了几个问题：

- 原文验证码示例的代码块内容错位。
- `ZREVRANGE` 是较旧写法，当前更推荐 `ZRANGE ... REV`。
- “Redis 是单线程”需要理解成简化表述。
- 分布式锁的 value 必须是每次加锁唯一的 token，不能使用固定值。
- “更新数据库再删除缓存”只是最终一致性基线，不是绝对一致。

原文适合以后作为 29 道问题的检查清单快速浏览。更值得投入时间的是亲手完成一次小练习，而不是再读一遍相似的八股答案。

---

## 十六、建议的实践顺序

### 第一阶段：Redis CLI

亲手执行：

```text
SET / GET / DEL / EXISTS
EXPIRE / TTL
INCR
HSET / HGET / HGETALL
LPUSH / RPOP / LRANGE
SADD / SISMEMBER / SMEMBERS
ZADD / ZRANGE ... REV WITHSCORES
```

目标不是背命令，而是看到业务场景时能选择合适的数据结构。

### 第二阶段：Python 小练习

用 `redis-py` 实现：

1. 验证码：String + TTL。
2. 阅读量：`INCR`。
3. 用户对象：Hash。
4. 点赞用户：Set。
5. 排行榜：ZSet。
6. Cache Aside：Redis 未命中后查询模拟数据库并回填。

### 第三阶段：理解故障

至少能回答：

- Redis 不可用时接口怎么办？
- 缓存删除失败怎么办？
- 热点 key 过期怎么办？
- 为什么不能直接删除分布式锁？
- 为什么不能每次请求都创建 Redis client？
- 为什么不能在异步服务里使用同步 redis-py 做慢请求？

### 第四阶段：再学高级主题

- Pipeline 和事务。
- Lua 脚本与原子性。
- Pub/Sub 和 Stream。
- RDB/AOF 更深的实现细节。
- 主从复制、Sentinel 和 Cluster。
- 大 key、热 key、慢命令和监控。

---

## 十七、当前阶段的最低掌握标准

如果已经能清楚回答下面这些问题，就已经达到“理解 Redis 基础”的程度：

1. Redis 为什么适合做缓存？
2. String、Hash、List、Set、ZSet 分别解决什么问题？
3. TTL 有什么作用？
4. `INCR` 为什么比 `GET` 后自己加一更安全？
5. 缓存穿透、击穿和雪崩分别是什么？
6. RDB 和 AOF 的区别是什么？
7. 为什么分布式锁要有 TTL 和唯一 token？
8. 主从、Sentinel 和 Cluster 分别解决什么问题？
9. Redis 和 MySQL 为什么很难做到绝对强一致？
10. redis-py 同步客户端和异步客户端应该怎么选？

---

## 参考资料

- Redis 八股原文：https://blog.csdn.net/Tyyccc_/article/details/160743052
- Redis 数据类型：https://redis.io/docs/latest/develop/data-types/
- Redis 持久化：https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- Redis 分布式锁：https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/
- redis-py 官方指南：https://redis.io/docs/latest/develop/clients/redis-py/
- redis-py 异步指南：https://redis.io/docs/latest/develop/clients/redis-py/async/
- Redis OM for Python：https://redis.io/docs/latest/integrate/redisom-for-python/