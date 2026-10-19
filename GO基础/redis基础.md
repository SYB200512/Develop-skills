# Redis 基础（阶段一：入门）

> 本阶段目标：**会装、会连、会敲常用命令，掌握 5 大数据类型和过期时间（TTL）**。
> 学习方式：**bash 命令 + Go 代码（go-redis/v9）双线进行**——bash 让你看懂 Redis 本身，Go 让你能在项目里落地。
> 属于《Redis 学习路线》的第①阶段，学完即可进入「核心原理（持久化 / 淘汰策略）」。

---

## 一、Redis 是什么，为什么快

Redis（Remote Dictionary Server）是一个**开源的内存型键值数据库**，数据主要存放在内存中，读写速度极快（读约 10 万次/秒级别）。

| 对比项 | Redis | MySQL |
| ------ | ----- | ----- |
| 存储位置 | 内存（快） | 磁盘（慢） |
| 定位 | 缓存中间件 | 主数据库（持久化） |
| 数据模型 | key-value | 表 + 行 + SQL |
| 默认端口 | 6379 | 3306 |

**为什么快（3 个关键点）**：

1. **内存存储**：CPU 直接访问内存，远快于磁盘 I/O；
2. **单线程**：命令在单线程中逐个执行，天然没有并发竞争、不需要加锁；
3. **I/O 多路复用**：一个线程同时监听大量连接，处理效率高。

> 定位要记牢：**Redis 是缓存中间件，不是主数据库**。生产环境通常与 MySQL/ES 配合——Redis 扛"热数据"，MySQL/ES 负责最终存储。

---

## 二、Windows 安装与启动

### 方案 1：Docker（推荐，最省事）

```bash
# 拉取并启动 redis 容器（-d 后台运行，-p 映射 6379 端口）
docker run -d --name redis -p 6379:6379 redis

# 进入容器执行 redis-cli
docker exec -it redis redis-cli
```

### 方案 2：Memurai（Windows 原生版）

1. 官网下载 Memurai（Redis 的 Windows 兼容实现），安装后作为 Windows 服务运行；
2. 打开 PowerShell，直接输入 `redis-cli` 即可连接本机 Redis。

### 验证连接

```bash
redis-cli ping
# 返回 PONG 说明连接正常
```

---

## 三、Go 环境准备（go-redis/v9）

go-redis 是 Go 官方的 Redis 客户端。你项目里的 `article_stat.go` 用的就是它。

```bash
# 新建 demo 目录并初始化 Go 模块
mkdir redis-demo
cd redis-demo
go mod init redis-demo

# 安装 go-redis v9
go get github.com/redis/go-redis/v9
```

**连接代码**（每个 demo 的基础模板）：

```go
package main

import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
)

func main() {
	client := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379", // Redis 地址
		Password: "",               // 密码，没有留空
		DB:       0,                // 使用第 0 号数据库
	})
	ctx := context.Background()

	// 连通性测试：返回 PONG
	pong, err := client.Ping(ctx).Result()
	fmt.Println(pong, err)
}
```

> ⚠️ **v9 的硬性要求：所有命令方法的第一个参数必须是 `context.Context`**。你项目里 `article_stat.go` 之前编译报错，就是因为少传了 ctx。

---

## 四、基础操作（bash + Go）

### 写入与读取

**bash**：

```bash
SET key value          # 写入键值对
GET key                # 读取，不存在返回 (nil)
DEL key                # 删除
EXISTS key             # 是否存在，返回 1 / 0
TYPE key               # 查看 key 的类型
```

**Go**：

```go
client.Set(ctx, "key", "value", 0)          // 0 表示永不过期
val, err := client.Get(ctx, "key").Result() // 读取
fmt.Println(val, err)                       // value <nil>

// 重点：err == redis.Nil 表示 key 不存在，这不是报错！
val, err = client.Get(ctx, "no_key").Result()
if err == redis.Nil {
	fmt.Println("key 不存在")
} else if err != nil {
	panic(err)
}

client.Del(ctx, "key")                       // 删除
n, _ := client.Exists(ctx, "key").Result()   // 是否存在：1 / 0
```

### 批量操作

**bash**：

```bash
MSET k1 v1 k2 v2       # 批量写入
MGET k1 k2             # 批量读取
```

**Go**：

```go
client.MSet(ctx, "k1", "v1", "k2", "v2")
vals, _ := client.MGet(ctx, "k1", "k2").Result()
fmt.Println(vals)      // [v1 v2]
```

### 数据库选择

Redis 默认有 16 个数据库（编号 0~15），默认用第 0 个：

```bash
SELECT 1               # 切换到 1 号库（Go 在 NewClient 的 DB 字段配置）
DBSIZE                 # 查看当前库有多少个 key
FLUSHDB                # 清空当前库（慎用）
```

### key 命名规范（好习惯）

用「冒号分层」命名，方便归类：

```bash
SET article:1001:title "Go 入门"
SET user:42:name "张三"
```

---

## 五、TTL 过期时间（本阶段重点）

缓存的灵魂是**过期**——数据有生命周期，到期自动消失，避免内存无限增长。

**bash**：

```bash
EXPIRE key 60          # 设置 key 60 秒后过期
TTL key                # 查看剩余秒数；-1 永不过期，-2 key 不存在
PTTL key               # 查看剩余毫秒数
PERSIST key            # 取消过期时间（转为永久）

SET article:1001 "内容" EX 60     # 写入并 60 秒后过期
```

**Go**：

```go
client.Set(ctx, "temp", "v", 10*time.Second)     // 写入并 10 秒后过期
ttl, _ := client.TTL(ctx, "temp").Result()       // 查看剩余时间
fmt.Println(ttl)                                 // 10s

client.Expire(ctx, "key", 5*time.Minute)         // 给已有 key 设置过期
client.Persist(ctx, "key")                       // 取消过期时间
```

**为什么要过期**：

- 缓存数据不需要永久保留，过期后重新从数据库加载即可；
- 控制 Redis 内存占用，防止把内存写爆。

> 小知识：Redis 的过期删除有「惰性删除 + 定期删除」两种机制，属于阶段二的内容，这里先会用即可。

---

## 六、5 大数据类型

| 类型 | 底层实现 | 典型场景 |
| ---- | -------- | -------- |
| String | SDS 简单动态字符串 | 缓存、计数器、session |
| Hash | dict + ziplist | 存对象（用户、文章、浏览量） |
| List | quicklist | 最新消息列表、简单队列 |
| Set | intset + dict | 去重、标签、交集并集 |
| ZSet（Sorted Set） | ziplist + skiplist | 排行榜、权重排序 |

### 1. String 字符串

最常用、最基础的类型，value 是字符串，支持原子自增。

**bash**：

```bash
SET article:1001:views 0     # 浏览量初始为 0
INCR article:1001:views      # 自增 +1（原子操作，不会算错）
DECR article:1001:views      # 自减 -1
INCRBY article:1001:views 5  # 一次 +5
GET article:1001:views       # 读取
STRLEN key                   # 字符串长度
```

**Go**：

```go
client.Set(ctx, "article:1001:views", 0, 0)   // 浏览量初始为 0
client.Incr(ctx, "article:1001:views")        // 自增 +1（原子操作）
client.Decr(ctx, "article:1001:views")        // 自减 -1
client.IncrBy(ctx, "article:1001:views", 5)   // 一次 +5
views, _ := client.Get(ctx, "article:1001:views").Int64()
fmt.Println(views)
```

> 场景：**计数器**（浏览量、点赞数）、**简单缓存**、后续的分布式锁（阶段三）。

### 2. Hash 哈希

Hash 类似一个 `map`，适合存"一个对象的多个字段"。

**bash**：

```bash
HSET article:1001 title "Redis 入门" author "张三"   # 一次写多个字段
HGET article:1001 title        # 读单个字段
HGETALL article:1001           # 读所有字段
HINCRBY article:1001 views 1   # 浏览量字段 +1（项目里就是这么用的）
HEXISTS article:1001 author    # 字段是否存在
HDEL article:1001 author       # 删除字段
HKEYS article:1001             # 所有字段名
HVALS article:1001             # 所有字段值
```

**Go**：

```go
// 写入多个字段
client.HSet(ctx, "article:1001", "title", "Redis 入门", "author", "张三")

// 读单个字段
title, _ := client.HGet(ctx, "article:1001", "title").Result()
fmt.Println(title)   // Redis 入门

// 读所有字段
all, _ := client.HGetAll(ctx, "article:1001").Result()
fmt.Println(all)     // map[author:张三 title:Redis 入门]

// 浏览量 +1（对应 bash 的 HINCRBY）
client.HIncrBy(ctx, "article:1001", "views", 1)

// 字段是否存在
ok, _ := client.HExists(ctx, "article:1001", "author").Result()
fmt.Println(ok)      // true
```

> 场景：**存对象**（用户信息、文章信息）。go-blog 的 `article_stat.go` 就是用 Hash 存文章浏览量。

### 3. List 列表

List 是有序的字符串列表，可以从两端插入/弹出。

**bash**：

```bash
LPUSH list:news "文章1" "文章2"     # 从左侧插入
RPUSH list:news "文章3"             # 从右侧插入
LRANGE list:news 0 -1               # 取全部（0 开始，-1 到末尾）
LPOP list:news                      # 弹出最左侧元素
RPOP list:news                      # 弹出最右侧元素
LLEN list:news                      # 列表长度
```

**Go**：

```go
client.LPush(ctx, "list:news", "文章1", "文章2")  // 左侧插入
client.RPush(ctx, "list:news", "文章3")           // 右侧插入
items, _ := client.LRange(ctx, "list:news", 0, -1).Result()
fmt.Println(items)    // [文章2 文章1 文章3]

v, _ := client.LPop(ctx, "list:news").Result()   // 弹出最左侧
fmt.Println(v)        // 文章2

n, _ := client.LLen(ctx, "list:news").Result()   // 长度
fmt.Println(n)        // 2
```

> 场景：**最新消息列表**（后进先出）、**简单队列**（左进右出）。

### 4. Set 集合

Set 是无序、**去重**的字符串集合，支持集合运算。

**bash**：

```bash
SADD set:tags "Go" "Redis" "Go"    # 添加（重复的只存一份）
SMEMBERS set:tags                  # 查看全部元素
SISMEMBER set:tags "Go"            # 是否包含某元素
SCARD set:tags                     # 元素数量
SREM set:tags "Redis"              # 删除元素
SINTER set:a set:b                 # 交集（共同关注）
SUNION set:a set:b                 # 并集
```

**Go**：

```go
client.SAdd(ctx, "set:tags", "Go", "Redis", "Go")  // 重复的只存一份
members, _ := client.SMembers(ctx, "set:tags").Result()
fmt.Println(members)   // [Go Redis]

ok, _ := client.SIsMember(ctx, "set:tags", "Go").Result()
fmt.Println(ok)        // true

count, _ := client.SCard(ctx, "set:tags").Result()
fmt.Println(count)     // 2（去重后的数量）

client.SRem(ctx, "set:tags", "Redis")              // 删除元素
```

> 场景：**去重**（标签、已读用户）、**共同关注**（交集）。

### 5. ZSet 有序集合

ZSet = Set + 分数（score），按分数排序，是**排行榜**的标配。

**bash**：

```bash
ZADD rank:hot 100 "文章A" 80 "文章B"     # 写入成员和分数
ZRANGE rank:hot 0 -1                     # 按分数从小到大取全部
ZREVRANGE rank:hot 0 2                   # 按分数从大到小取前 3（排行榜）
ZSCORE rank:hot "文章A"                  # 查看某个成员分数
ZINCRBY rank:hot 10 "文章A"              # 给文章A 加 10 分
ZCARD rank:hot                           # 成员数量
ZREM rank:hot "文章B"                    # 删除成员
```

**Go**：

```go
// 写入成员和分数
client.ZAdd(ctx, "rank:hot",
	redis.Z{Score: 100, Member: "文章A"},
	redis.Z{Score: 80, Member: "文章B"},
)

// 按分数从大到小取前 3（排行榜）
top, _ := client.ZRevRangeWithScores(ctx, "rank:hot", 0, 2).Result()
for _, z := range top {
	fmt.Println(z.Member, z.Score)   // 文章A 100 / 文章B 80
}

// 给文章A 加 10 分
client.ZIncrBy(ctx, "rank:hot", 10, "文章A")
```

> 场景：**热度排行榜**、**延时队列**、按权重排序。阶段三会用它做博客的文章热度榜。

---

## 七、通关 Demo：博客浏览统计 + 热度排行（完整 Go 程序）

把下面的代码保存为 `redis-demo/main.go`，然后 `go run main.go`。

```go
package main

import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
)

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	ctx := context.Background()

	// 先确认 Redis 已启动
	if _, err := client.Ping(ctx).Result(); err != nil {
		panic("Redis 未启动：" + err.Error())
	}

	// 1. Hash：记录 3 篇文章的浏览量
	client.HSet(ctx, "article:1", "title", "Go 入门", "views", 0)
	client.HSet(ctx, "article:2", "title", "Redis 入门", "views", 0)
	client.HSet(ctx, "article:3", "title", "MySQL 优化", "views", 0)

	// 2. 模拟 5 次浏览（浏览量 +1，原子操作）
	for i := 0; i < 5; i++ {
		client.HIncrBy(ctx, "article:1", "views", 1)
	}
	client.HIncrBy(ctx, "article:2", "views", 3)
	client.HIncrBy(ctx, "article:3", "views", 2)

	// 3. 把浏览量同步到 ZSet 热度榜
	for _, id := range []string{"article:1", "article:2", "article:3"} {
		views, _ := client.HGet(ctx, id, "views").Int64()
		client.ZAdd(ctx, "rank:hot", redis.Z{Score: float64(views), Member: id})
	}

	// 4. 输出排行榜
	top, _ := client.ZRevRangeWithScores(ctx, "rank:hot", 0, -1).Result()
	for i, z := range top {
		fmt.Printf("第%d名 %s 浏览量 %d\n", i+1, z.Member, int64(z.Score))
	}
}
```

运行结果：

```
第1名 article:1 浏览量 5
第2名 article:2 浏览量 3
第3名 article:3 浏览量 2
```

这个 Demo 就是你 go-blog 里 `article_stat.go` 的简化版：**Hash 记浏览量 + ZSet 排热度**。

---

## 八、常用管理与排查命令

```bash
PING                 # 连通性测试，返回 PONG
INFO                 # 查看服务器信息（内存、连接数、命令统计）
DBSIZE               # 当前库 key 数量
KEYS *               # 匹配所有 key（生产环境慎用，会阻塞！用 SCAN 代替）
SCAN 0               # 游标式遍历 key，不阻塞
```

> 记住：`KEYS *` 生产环境不要用，Redis 单线程执行，会卡住其他命令；排查用 `SCAN`。

---

## 九、阶段一练习清单（动手才记得住）

bash 和 Go 各做一遍，以下是本阶段的通关任务：

- [ ] 用 Docker 启动 Redis，`ping` 返回 PONG；Go 里 `client.Ping(ctx)` 返回 `pong`
- [ ] 完成 `SET / GET / DEL / EXISTS / TYPE`（bash + Go 各 3 遍）
- [ ] 给一个 key 设置 10 秒过期，用 `TTL` 观察它消失；Go 里 `Set(ctx, ..., 10*time.Second)`
- [ ] String：用 `INCR` 实现"文章浏览量+1"
- [ ] Hash：用 `HSET/HGETALL` 存一篇"假文章"（标题、作者、内容）
- [ ] List：`LPUSH` 存 3 篇文章标题，`LRANGE` 取出来
- [ ] Set：往标签集合里重复添加同名字，验证去重
- [ ] ZSet：给 3 篇文章打分，用 `ZREVRANGE` 排出热度榜前 3
- [ ] 跑通第七节的完整 Demo

**通关小目标**：用 bash 和 Go 各实现一遍「模拟博客文章浏览统计 + 热度排行」，做完阶段一就算掌握了。

> 下一阶段预告：RDB/AOF 持久化、过期删除与内存淘汰——见《Redis 完整讲解》。
