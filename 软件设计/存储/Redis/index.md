---
tags: [moc, 软件设计, Redis]
created: 2026-05-15
publish: true
---

# Redis 专题

> 深入理解 Redis 原理，掌握生产环境最佳实践

## 目录

- [数据结构](#数据结构)
- [持久化](#持久化)（待补充）
- [高可用](#高可用)（待补充）
- [常见问题](#常见问题)（待补充）
- [练习题](#练习题)

---

## 数据结构

> 核心规律：**小数据用内存紧凑结构（ziplist/intset），大数据用高效检索结构（hashtable/skiplist）**，Redis 会在达到阈值时自动升级。

### 1. String

**底层实现**：SDS（Simple Dynamic String）

Redis 不直接用 C 字符串，原因：
- C 字符串获取长度 O(N)，SDS 有 `len` 字段，O(1)
- C 字符串遇 `\0` 截断，无法存二进制；SDS 用 `len` 判断结尾，可存图片/序列化数据
- C 字符串修改需手动 `realloc`，SDS 有预分配机制，减少内存重分配

```
SDS 结构：
┌──────┬──────┬───────────────┐
│  len │ free │    buf[]      │
└──────┴──────┴───────────────┘
  已用   预留    实际数据
```

**场景一：分布式锁（防止超卖）**

```bash
SET lock:item:1001 "order:9527" NX PX 3000
```

| 参数 | 含义 |
|------|------|
| `lock:item:1001` | key，标识锁住的资源 |
| `"order:9527"` | value，记录加锁方身份，释放时校验 |
| `NX` | Not eXist，key 不存在才写入，保证只有一人抢到锁 |
| `PX 3000` | 过期时间 3000 毫秒，防止加锁方崩溃后锁永不释放 |

释放锁必须用 Lua 脚本保证原子性（先 GET 校验，再 DEL），不能分两步：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
end
return 0
```

**场景二：计数器（文章浏览量）**

```bash
INCR article:view:123
```

| 参数 | 含义 |
|------|------|
| `article:view:123` | key，标识文章123的浏览量 |

`INCR` 是原子操作，key 不存在时从 0 开始自增，Redis 单线程保证线程安全。

---

### 2. Hash

**底层实现**：
- 字段数 ≤ 128 且所有字段/值长度 ≤ 64 字节 → **ziplist**（连续内存，O(N) 查找）
- 超过阈值 → **hashtable**（O(1) 查找）

ziplist 存 Hash 时 key/value 相邻存储：`[key1][val1][key2][val2]...`

**场景：用户信息存储（支持单字段更新）**

```bash
HSET user:1001 name "alice" age 18 vip 1
HGET user:1001 name
HMGET user:1001 name age
HGETALL user:1001
HINCRBY user:1001 age 1
```

| 命令 | 参数说明 |
|------|---------|
| `HSET user:1001 name "alice" age 18` | key=`user:1001`，后跟任意多个 field-value 对 |
| `HGET user:1001 name` | key=`user:1001`，field=`name`，只读一个字段 |
| `HMGET user:1001 name age` | M=Multi，批量读取多个字段，返回对应值列表 |
| `HGETALL user:1001` | 返回所有 field 和 value，数据量大时慎用 |
| `HINCRBY user:1001 age 1` | 对 field=`age` 做整数加法，第三个参数为增量值 |

> 与 String 存 JSON 相比，Hash 可单独更新某字段，避免并发读写整个对象的竞态问题。

---

### 3. List

**底层实现**：**quicklist**（Redis 3.2+）= 双向链表，每个节点是一个 ziplist

```
quicklist：
node1(ziplist) <-> node2(ziplist) <-> node3(ziplist)
  [a,b,c,d]         [e,f,g,h]         [i,j,k]
```

纯链表每个节点需要 prev/next 指针，小数据时指针开销比数据本身还大；quicklist 将连续元素打包成 ziplist 节点，大幅减少指针数量，内存和缓存命中率更好。

**场景一：消息通知列表（最新50条）**

```bash
LPUSH notify:uid:1001 "你的订单已发货"
LTRIM notify:uid:1001 0 49
LRANGE notify:uid:1001 0 9
```

| 命令 | 参数说明 |
|------|---------|
| `LPUSH notify:uid:1001 "消息"` | 从左端（头部）插入，可一次插多个值 |
| `LTRIM notify:uid:1001 0 49` | 裁剪列表，只保留下标 0~49，即最新50条 |
| `LRANGE notify:uid:1001 0 9` | 读取下标 0~9 的元素，不删除；`-1` 表示最后一个 |

**场景二：任务队列（消费者阻塞等待）**

```bash
LPUSH queue:task "task:001"   # 生产者入队
BRPOP queue:task 5            # 消费者出队
```

| 命令 | 参数说明 |
|------|---------|
| `BRPOP queue:task 5` | B=Blocking，阻塞弹出右端元素；可传多个 key，哪个有数据先返回；`5` 为超时秒数，`0` 表示永久阻塞 |

---

### 4. Set

**底层实现**：
- 全为整数且数量 ≤ 512 → **intset**（有序整数数组，二分查找，内存极省）
- 否则 → **hashtable**

**场景一：点赞去重**

```bash
SADD like:article:100 "uid:1001"
SISMEMBER like:article:100 "uid:1001"
SCARD like:article:100
```

| 命令 | 参数说明 |
|------|---------|
| `SADD like:article:100 "uid:1001"` | 向 Set 添加元素，重复自动忽略，可一次加多个 |
| `SISMEMBER like:article:100 "uid:1001"` | 判断元素是否在 Set 中，返回 0 或 1，O(1) |
| `SCARD like:article:100` | 返回 Set 元素数量 |

**场景二：社交关系运算**

```bash
SINTER follow:uid:1 follow:uid:2    # 共同关注
SDIFF follow:uid:2 follow:uid:1     # uid:2 关注但 uid:1 没关注（推荐好友）
SUNION follow:uid:1 follow:uid:2    # 合并关注列表
```

| 命令 | 含义 |
|------|------|
| `SINTER` | 交集 |
| `SDIFF` | 差集，以第一个 key 为基准，减去后面 key 的元素 |
| `SUNION` | 并集 |

**场景三：抽奖**

```bash
SRANDMEMBER lucky:activity:1 3   # 随机取3个，不删除（查看中奖名单）
SPOP lucky:activity:1 1          # 随机弹出1个，会删除（抽奖消耗）
```

---

### 5. ZSet（有序集合）

**底层实现**：
- 元素数 ≤ 128 且所有 member 长度 ≤ 64 字节 → **ziplist**
- 超过阈值 → **skiplist + hashtable** 双结构

为什么同时用两种结构？
- **skiplist**：按 score 范围查询，O(logN)
- **hashtable**：按 member 直接查 score，O(1)
- 两者共存，互补短板

**跳表（skiplist）结构**：

```
层3：  1 ──────────────────────→ 50
层2：  1 ──────────→ 20 ───────→ 50
层1：  1 ──→ 10 ──→ 20 ──→ 35 → 50
```

查找35：从最高层开始，逐层缩小范围，平均 O(logN)。相比红黑树：实现更简单，范围查询更直观。

**场景一：排行榜**

```bash
ZADD rank:week 100 "uid:1001"
ZINCRBY rank:week 50 "uid:1001"
ZREVRANGE rank:week 0 9 WITHSCORES
ZREVRANK rank:week "uid:1001"
ZRANGEBYSCORE rank:week 1000 2000 WITHSCORES LIMIT 0 10
```

| 命令 | 参数说明 |
|------|---------|
| `ZADD rank:week 100 "uid:1001"` | 添加 member，`100` 为 score |
| `ZINCRBY rank:week 50 "uid:1001"` | 对 member 的 score 增加50，原子操作 |
| `ZREVRANGE rank:week 0 9 WITHSCORES` | 按 score 从高到低取下标 0~9；`WITHSCORES` 同时返回 score |
| `ZREVRANK rank:week "uid:1001"` | 返回 member 从高到低的排名（0-based），+1 即名次 |
| `ZRANGEBYSCORE rank:week 1000 2000 WITHSCORES LIMIT 0 10` | 取 score 在 1000~2000 之间的元素；`LIMIT 0 10` 跳过0条取10条，类似 SQL LIMIT |

**场景二：延迟队列**

```bash
ZADD delay:queue 1716000000 "send_email:order:9527"   # score=执行时间戳
ZRANGEBYSCORE delay:queue 0 <当前时间戳>              # 取到期任务
ZREM delay:queue "send_email:order:9527"              # 执行后删除
```

---

### 6. HyperLogLog

**解决的问题**：统计 UV（独立访客数），数据量上亿时不能用 Set 存所有 uid（内存不够）。

**原理**：概率算法，误差约 0.81%，无论多少数据，**最多只用 12KB 内存**。

**场景：页面 UV 统计**

```bash
PFADD uv:page:home "uid:1001" "uid:1002" "uid:1003"
PFCOUNT uv:page:home
PFMERGE uv:site uv:page:home uv:page:about
```

| 命令 | 参数说明 |
|------|---------|
| `PFADD uv:page:home "uid:1001"` | PF=Philippe Flajolet（发明者）；添加元素，重复不增加计数；返回 1 表示估算值有变化 |
| `PFCOUNT uv:page:home` | 返回估算的不重复元素数量；可传多个 key 合并统计 |
| `PFMERGE uv:site uv:page:home uv:page:about` | 合并多个 HyperLogLog；第一个参数为目标 key，后面为源 key |

---

### 7. Bitmap

**解决的问题**：用户签到记录，一年365天，每天仅占1位，365天只需 46 字节。

**场景：用户签到**

```bash
SETBIT sign:uid:1001:2024 180 1
GETBIT sign:uid:1001:2024 180
BITCOUNT sign:uid:1001:2024
BITCOUNT sign:uid:1001:2024 0 30
BITPOS sign:uid:1001:2024 1
```

| 命令 | 参数说明 |
|------|---------|
| `SETBIT sign:uid:1001:2024 180 1` | 设置第 `180` 位的值为 `1`（已签到）；位偏移从0开始 |
| `GETBIT sign:uid:1001:2024 180` | 获取第 `180` 位的值，返回 0 或 1 |
| `BITCOUNT sign:uid:1001:2024` | 统计所有值为1的位数，即全年签到天数 |
| `BITCOUNT sign:uid:1001:2024 0 30` | 统计第0到第30**字节**（注意是字节不是位）内值为1的数量，即前 248 天 |
| `BITPOS sign:uid:1001:2024 1` | 返回第一个值为 `1` 的位的位置，即今年第一次签到是第几天 |

---

### 8. Stream

**解决 List 队列的缺陷**：消息被消费后即删除，不支持多消费者，无 ACK 机制。Stream 是 Redis 5.0 引入的轻量级消息队列，对标 Kafka。

**核心概念**：
- 每条消息有唯一 ID：`时间戳-序号`，如 `1716000000000-0`
- **消费者组**：同一条消息只投递给组内一个消费者，实现水平扩展
- **ACK 机制**：消费者处理完后确认，未确认消息留在 PEL（Pending Entry List）可重新投递

**场景：订单消息队列**

```bash
# 生产者写入消息
XADD orders * product "iphone" qty 2

# 查看消息
XLEN orders
XRANGE orders - +

# 消费者（普通读取）
XREAD COUNT 10 STREAMS orders 0

# 消费者组模式
XGROUP CREATE orders group:pay 0
XREADGROUP GROUP group:pay consumer1 COUNT 5 STREAMS orders >
XACK orders group:pay 1716000000000-0
```

| 命令 | 参数说明 |
|------|---------|
| `XADD orders * product "iphone" qty 2` | key=`orders`；`*` 让 Redis 自动生成消息 ID；后跟 field-value 对作为消息内容 |
| `XRANGE orders - +` | 按 ID 范围读取；`-` 表示最小 ID（从头），`+` 表示最大 ID（到尾） |
| `XREAD COUNT 10 STREAMS orders 0` | 读取消息不删除；`COUNT 10` 最多读10条；`0` 从 ID>0 开始即从头读；`$` 表示只读新消息 |
| `XGROUP CREATE orders group:pay 0` | 创建消费者组 `group:pay`；`0` 从头消费历史消息，`$` 只消费新消息 |
| `XREADGROUP GROUP group:pay consumer1 COUNT 5 STREAMS orders >` | 以消费者组身份读取；`consumer1` 为消费者名称（自动创建）；`>` 只读未投递给任何消费者的新消息 |
| `XACK orders group:pay 1716000000000-0` | 确认消息处理完成，从 PEL 中移除；参数依次为 key、组名、消息 ID |

**与其他方案对比**：

| 特性 | List | Stream | Kafka |
|------|------|--------|-------|
| 消费后消息 | 删除 | 保留 | 保留 |
| 多消费者 | 不支持 | 支持（消费者组）| 支持 |
| ACK 机制 | 无 | 有 | 有 |
| 消息回溯 | 不支持 | 支持 | 支持 |
| 适用场景 | 简单队列 | 轻量级消息队列 | 高吞吐、大数据 |

---

### 数据结构选型总结

| 场景 | 推荐类型 | 理由 |
|------|---------|------|
| 缓存、计数器、分布式锁 | String | 简单 key-value，原子操作 |
| 对象存储、部分字段更新 | Hash | 字段级操作，避免整体读写 |
| 最新列表、简单队列 | List | 有序、支持阻塞消费 |
| 去重、集合运算 | Set | 天然去重，支持交并差 |
| 排行榜、延迟队列 | ZSet | 按 score 排序，范围查询高效 |
| 大数据量 UV 统计 | HyperLogLog | 固定内存，允许误差 |
| 签到、状态标记 | Bitmap | 位级操作，极省内存 |
| 可靠消息队列 | Stream | 支持 ACK、消费者组、消息回溯 |

---

## 持久化

（待补充）

## 高可用

（待补充）

## 常见问题

（待补充）

---

## 练习题

### 数据结构专项

**Q1.** 以下场景该用哪种数据类型，说明理由：
- 统计某活动页面每天的独立访客数（UV），日访问量可能达到1000万
- 记录用户最近浏览的商品（最多保留20条，有重复时保留最新）
- 实现一个延迟任务：用户下单后30分钟未支付，自动取消订单

**Q2.** `SET key value NX EX 60` 和 `SET key value XX EX 60` 有什么区别？分别适用于什么场景？

**Q3.** ZSet 在数据量大时为什么同时维护 skiplist 和 hashtable 两种结构？只用一种行不行？

**Q4.** 以下命令有什么问题，应该如何修改：
```bash
# 需求：判断用户是否点赞，没有则添加点赞
GET like:article:100
# 如果返回值不包含 uid:1001，则执行：
SET like:article:100 "uid:1001,uid:1002"
```

**Q5.** `BITCOUNT key 0 6` 中的 `0 6` 是什么单位？统计的是哪个范围的位？

**Q6.** Stream 的消费者组中，`XREADGROUP` 命令里的 `>` 参数是什么含义？如果改成某个具体的消息 ID 会发生什么？

**Q7.** 用 Redis 实现一个抽奖系统，要求：
- 每个用户只能参与一次
- 支持随机抽取N名中奖者
- 中奖后该用户从奖池中移除

写出完整的 Redis 命令并说明每个参数。

**Q8.** 渐进式 rehash 是什么？为什么 Redis 要这样设计而不是一次性完成 rehash？
