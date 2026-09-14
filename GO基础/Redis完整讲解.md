# Redis完整讲解

## 一、Redis

Redis 是**开源的内存型键值数据库**，数据主要放在内存，读写速度极快；支持持久化（RDB/AOF）防止宕机丢数据。

> 定位：**缓存中间件**，不是主数据库。 默认端口：`6379` 单线程模型（命令执行单线程，IO 多路复用），所以高性能，不存在并发竞争问题

核心特性：

1. 内存存储：读写速度极高；支持 RDB/AOF 持久化，防止宕机丢失数据

2. 丰富数据结构：String、Hash、List、Set、ZSet

3. 高可用方案：主从复制、Sentinel 哨兵、Redis Cluster 集群

4. 单命令原子性：Redis 单命令执行是原子的；但多条命令默认不保证原子，想要事务需要 Lua 脚本或者 Redis 事务

   ```bash
   go get github.com/redis/go-redis/v9
   ```

   

## 二、Redis客户端连接配置

```go
func NewRedisClient() *redis.Client {
    client := redis.NewClient(&redis.Options{
        Addr:         "localhost:6379", // redis地址
        Password:     "",                // 密码，没有留空
        DB:           0,                 // 使用第0号数据库
        PoolSize:     100,               // 连接池最大连接数
        MinIdleConns: 10,                // 最小空闲连接，维持长连接，避免频繁创建连接
        MaxRetries:   3,                 // 命令失败最大重试次数
        DialTimeout:  5 * time.Second,   // 建立连接超时
        ReadTimeout:  3 * time.Second,   // 读取数据超时
        WriteTimeout: 3 * time.Second,   // 写入数据超时
        PoolTimeout:  4 * time.Second,   // 从连接池获取连接的超时时间
    })
    return client
}
```

重点概念：**连接池**

 Redis 建立 TCP 连接开销大，连接池预先创建一批连接复用。不要每次操作都新建客户端实例！ 测试连通性：`client.Ping(ctx).Result()`，返回`PONG`代表连接正常。

## 三、Redis 5 大基础数据结构

表格

| 类型             | 底层实现                | 典型场景                                 |
| ---------------- | ----------------------- | ---------------------------------------- |
| String           | SDS（简单动态字符串）   | 缓存普通数据、计数器、分布式锁、session  |
| Hash             | dict + ziplist          | 对象存储：用户信息、商品信息（类似 map） |
| List             | quicklist               | 消息队列、最新消息列表、日志             |
| Set              | intset + dict           | 标签、去重、交集 / 并集（好友共同关注）  |
| ZSet(Sorted Set) | ziplist + skiplist 跳表 | 排行榜、延时队列、权重排序               |

### 1.String字符串

```go
// SET key value 0：0代表不设置过期时间
client.Set(ctx, "key", "value", 0)
// 设置10秒过期
client.Set(ctx, "temp_key", "temp_value", 10*time.Second)

// GET读取
val, err := client.Get(ctx, "key").Result()
// 重点判断：err == redis.Nil 代表key不存在，不是系统报错！
if err == redis.Nil {
    fmt.Println("key不存在")
}

client.MSet(ctx,"k1","v1","k2","v2") //批量写
client.MGet(ctx,"k1","k2")           //批量读

client.Incr(ctx,"counter") //自增 +1，原子计数器
client.Expire(ctx,"key",5*time.Minute) //设置过期时间
client.Del(ctx,"key") //删除key
```

### 2.Hash（适合存储对象，类似 map）

`Hash：key -> field -> value`，适合用户、商品信息，**可以单独修改某个字段，不用序列化整个对象**

```go
// 单个字段写入
client.HSet(ctx, "user:1000", "name", "Alice")
client.HSet(ctx, "user:1000", "age", "25")

// 批量写入多个字段
client.HMSet(ctx, "user:1001", map[string]interface{}{
    "name":"Bob",
    "age":"30",
})

client.HGet(ctx,"user:1000","name") //获取单个字段
client.HGetAll(ctx,"user:1000")    //获取该hash全部字段和值，返回map
client.HExists(ctx,"user:1000","name") //判断字段是否存在
client.HDel(ctx,"user:1000","age") //删除hash里面某个字段
client.HLen(ctx,"user:1000")       //hash里面字段总数
```

### 3. List 双向链表

左右都可以插入弹出，做消息队列、最新消息列表

```go
client.LPush(ctx,"queue","task1") //左边插入（头插）
client.RPush(ctx,"queue","task3") //右边插入（尾插）

client.LPop(ctx,"queue") //左边弹出
client.RPop(ctx,"queue") //右边弹出

client.LRange(ctx,"queue",0,-1) //读取全部元素，0开始，-1代表最后
client.LLen(ctx,"queue")        //获取列表长度

// BLPOP：阻塞弹出！队列没有数据时，阻塞等待，适合简单消息队列
client.BLPop(ctx,0,"queue")
```

### 4.Set 集合，无序，元素不可重复

适合标签、共同好友、去重

```go
client.SAdd(ctx,"tags","go","redis") //添加元素
client.SMembers(ctx,"tags")          //获取集合全部成员
client.SIsMember(ctx,"tags","go")    //判断元素是否存在
client.SCard(ctx,"tags")             //集合元素数量

client.SInter(ctx,"set1","set2") //交集：两个集合共同元素
client.SUnion(ctx,"set1","set2") //并集：所有元素，去重
client.SDiff(ctx,"set1","set2")  //差集：set1有，set2没有
```

### 5. ZSet 有序集合（Sorted Set）

每个成员带`score`分数，自动按分数排序；排行榜、延时队列经典方案

```go
// ZAdd 添加成员，redis.Z封装成员和分数
client.ZAdd(ctx, "rank",
    redis.Z{Score:100, Member:"player1"},
    redis.Z{Score:200, Member:"player2"},
)

client.ZRangeWithScores(ctx,"rank",0,-1) //从小到大，带分数返回
client.ZRank(ctx,"rank","player1")       //获取成员排名（从小到大）
client.ZIncrBy(ctx,"rank",50,"player1")  //给成员分数增加50
```

## 四、分布式锁

> 核心要点：
>
> 1. 获取锁：`SET key value NX EX`，原子操作：不存在才设置，同时设置过期时间（防止死锁）。`NX`=Not Exist。
> 2. 释放锁：**不能直接 DEL！** 需要 Lua 脚本，判断锁是当前持有者的才删除，防止误删别人的锁。
>
> - 为什么不能直接 DEL：锁过期之后，别的服务拿到锁，你把别人的锁删掉。
> - value 存唯一标识（当前请求唯一值，示例用时间戳），释放锁的时候校验 value。

```go
// 释放锁的Lua脚本
script := `
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
`
```

代码封装了`Lock()`、`Unlock()`、`TryLock()`（带重试）。使用的时候`defer lock.Unlock(ctx)`保证释放

注意：这个是简易版分布式锁；生产如果严格场景还要考虑锁续期（看门狗）

```go
package main

import (
	"context"
	"fmt"
	"github.com/redis/go-redis/v9"
	"sync"
	"time"
)

type WatchDogLock struct {
	client     *redis.Client
	key        string
	value      string
	ttl        time.Duration
	cancelFunc context.CancelFunc // 停止看门狗
	wg         sync.WaitGroup
}

func NewWatchDogLock(client *redis.Client, key string, ttl time.Duration) *WatchDogLock {
	return &WatchDogLock{
		client: client,
		key:    key,
		value:  fmt.Sprintf("%d", time.Now().UnixNano()),
		ttl:    ttl,
	}
}

// Lock 加锁并启动看门狗
func (l *WatchDogLock) Lock(ctx context.Context) (bool, error) {
	ok, err := l.client.SetNX(ctx, l.key, l.value, l.ttl).Result()
	if err != nil || !ok {
		return ok, err
	}

	// 启动看门狗协程
	watchCtx, cancel := context.WithCancel(ctx)
	l.cancelFunc = cancel
	l.wg.Add(1)//告诉等待组，新增一个需要等待的协程
	go func() {
		defer l.wg.Done()
		ticker := time.NewTicker(l.ttl / 3) // 每1/3过期时间续期一次
		defer ticker.Stop()
		for {
			select {
			case <-watchCtx.Done():
				// 业务结束，关闭看门狗
				return
			case <-ticker.C:
				// Lua脚本：校验value，然后延长过期时间
				script := `
					if redis.call("get", KEYS[1]) == ARGV[1] then
						return redis.call("expire", KEYS[1], ARGV[2])
					else
						return 0
					end
				`
				l.client.Eval(watchCtx, script, []string{l.key}, l.value, l.ttl.Seconds())
			}
		}
	}()
	return true, nil
}

// Unlock 停止看门狗 + 释放锁
func (l *WatchDogLock) Unlock(ctx context.Context) error {
	// 停止看门狗协程
	if l.cancelFunc != nil {
		l.cancelFunc()
		l.wg.Wait()
	}
	// Lua解锁脚本
	script := `
		if redis.call("get", KEYS[1]) == ARGV[1] then
			return redis.call("del", KEYS[1])
		else
			return 0
		end
	`
	_, err := l.client.Eval(ctx, script, []string{l.key}, l.value).Result()
	return err
}

func main() {
	client := NewRedisClient()
	ctx := context.Background()
	lock := NewWatchDogLock(client, "lock:order:1001", 10*time.Second)

	locked, err := lock.Lock(ctx)
	if err != nil || !locked {
		fmt.Println("获取锁失败")
		return
	}
	defer lock.Unlock(ctx)

	fmt.Println("拿到锁，业务开始，耗时15秒（超过锁原始10s，看门狗自动续期）")
	time.Sleep(15 * time.Second)
	fmt.Println("业务结束")
}
```

整体功能：拿到锁之后启动后台协程，**自动延长锁过期时间**。 解决基础分布式锁的痛点：业务执行时间 > 锁 TTL，锁提前过期，多个进程同时执行业务。

## 结构体定义

```
type WatchDogLock struct {
	client     *redis.Client
	key        string
	value      string
	ttl        time.Duration
	cancelFunc context.CancelFunc // 停止看门狗
	wg         sync.WaitGroup
}
```

- `client`：redis 客户端
- `key`：锁的 key，例如 `lock:order:1001`
- `value`：唯一标识，用来证明锁是谁持有的（这里用纳秒时间戳，生产推荐 UUID）
- `ttl`：锁初始过期时间
- `cancelFunc`：context 取消函数，用来关闭看门狗协程
- `wg`：等待组，解锁时等待看门狗协程完全退出，防止协程泄漏

```
func NewWatchDogLock(client *redis.Client, key string, ttl time.Duration) *WatchDogLock {
	return &WatchDogLock{
		client: client,
		key:    key,
		value:  fmt.Sprintf("%d", time.Now().UnixNano()),
		ttl:    ttl,
	}
}
```

构造函数，创建锁实例，生成唯一 value。

## Lock () 加锁函数

```
func (l *WatchDogLock) Lock(ctx context.Context) (bool, error) {
	ok, err := l.client.SetNX(ctx, l.key, l.value, l.ttl).Result()
	if err != nil || !ok {
		return ok, err
	}
```

`SetNX` = `SET key value NX EX`，原子加锁：

- NX：key 不存在才设置（互斥）
- 传入 ttl，自动设置过期时间，防止死锁 加锁失败直接返回，不启动看门狗。

```
	// 启动看门狗协程
	watchCtx, cancel := context.WithCancel(ctx)
	l.cancelFunc = cancel
	l.wg.Add(1)
	go func() {
		defer l.wg.Done()
		ticker := time.NewTicker(l.ttl / 3) // 每1/3过期时间续期一次
		defer ticker.Stop()
```

✅看门狗规则：**每 `ttl/3` 续一次** 举例：ttl=10 秒，每 3 秒多就去续期。

> 为什么是 1/3？留足余量：哪怕一次续期网络抖动失败，还有两次机会。

```
		for {
			select {
			case <-watchCtx.Done():
				// 业务结束，关闭看门狗
				return
			case <-ticker.C:
				// Lua脚本：校验value，然后延长过期时间
				script := `
					if redis.call("get", KEYS[1]) == ARGV[1] then
						return redis.call("expire", KEYS[1], ARGV[2])
					else
						return 0
					end
				`
				l.client.Eval(watchCtx, script, []string{l.key}, l.value, l.ttl.Seconds())
			}
		}
	}()
	return true, nil
}
```

### 看门狗协程逻辑

1. ```
   select
   ```

    监听两个信号

   - `watchCtx.Done()`：解锁的时候触发，看门狗退出
   - `ticker.C`：定时器触发，执行续期 Lua 脚本

2. Lua 脚本含义：

   - 先 GET 锁 key，判断锁当前持有者是不是自己（value 相等）
   - ✅相等：调用`expire`重置过期时间，锁续命成功
   - ❌不相等：说明锁已经被别人抢走，不再续期，返回 0

> 重点：**如果锁已经不属于当前客户端，看门狗不会继续续锁！** 避免锁已经丢失，看门狗还在疯狂续命。

## Unlock () 解锁函数

```
func (l *WatchDogLock) Unlock(ctx context.Context) error {
	// 停止看门狗协程
	if l.cancelFunc != nil {
		l.cancelFunc()
		l.wg.Wait()
	}
```

解锁第一步：**先关闭看门狗**

1. `cancelFunc()`：取消 watchCtx，看门狗协程收到 Done 信号退出循环
2. `wg.Wait()`：阻塞等待看门狗协程完全退出，防止协程泄露。

```
	// Lua解锁脚本
	script := `
		if redis.call("get", KEYS[1]) == ARGV[1] then
			return redis.call("del", KEYS[1])
		else
			return 0
		end
	`
	_, err := l.client.Eval(ctx, script, []string{l.key}, l.value).Result()
	return err
}
```

Lua 解锁脚本：**先判断锁归属，再删除**

- 如果锁是自己的 → DEL 删除锁，释放
- 如果锁不是自己的 → 不做任何操作，防止误删别人锁

## main 主函数调用演示

```
func main() {
	client := NewRedisClient()
	ctx := context.Background()
	lock := NewWatchDogLock(client, "lock:order:1001", 10*time.Second)

	locked, err := lock.Lock(ctx)
	if err != nil || !locked {
		fmt.Println("获取锁失败")
		return
	}
	defer lock.Unlock(ctx) // 函数退出自动解锁

	fmt.Println("拿到锁，业务开始，耗时15秒（超过锁原始10s，看门狗自动续期）")
	time.Sleep(15 * time.Second)
	fmt.Println("业务结束")
}
```

执行流程：

1. 调用 Lock，SetNX 拿到锁，启动看门狗
2. 业务 sleep 15 秒（> 初始锁 10 秒）
3. 看门狗每约 3.3 秒执行一次续期，锁一直保持有效，不会过期
4. sleep 结束，函数退出，触发`defer lock.Unlock()`
5. Unlock：停止看门狗，等待协程退出，执行 Lua 脚本删除锁

## 测试场景演示

场景：锁 TTL=10s，业务执行 15s

- 不带看门狗版本：第 10 秒锁自动过期，其他进程抢锁，并发冲突
- 带看门狗版本：后台持续续期，15 秒内锁一直属于当前程序，业务执行完才释放





## 五、CacheAside旁路缓存模式

封装了 CacheAside 结构体：Get/Set/Delete 方法。 业务逻辑：

1. 查询用户：先查 Redis 缓存，命中直接返回
2. 缓存未命中，查询数据库
3. 数据库查到数据，序列化写入 Redis，设置过期时间
4. 更新用户：**先更新数据库，再删除缓存**（不是更新缓存！让读请求懒加载）

> 为什么更新时删除缓存而不是更新缓存： 高并发场景，更新 DB + 更新缓存会出现数据不一致。删除缓存，下次读请求自动加载最新数据，更安全。

```go
type CacheAside struct {
	client *redis.Client
	// db 数据库对象
}

// Get 查询：先缓存，后DB，回填缓存
func (c *CacheAside) Get(ctx context.Context, key string) (*User, error) {
	// 1.查redis
	val, err := c.client.Get(ctx, key).Result()
	if err == nil {
		// 缓存命中，反序列化返回
		var u User
		json.Unmarshal([]byte(val), &u)
		return &u, nil
	}
	// 缓存未命中，查询数据库
	user, err := c.db.QueryUser(ctx, key)
	if err != nil {
		return nil, err
	}
	if user == nil {
		return nil, nil
	}
	// 回填缓存，设置过期时间
	data, _ := json.Marshal(user)
	c.client.Set(ctx, key, data, 10*time.Minute)
	return user, nil
}

// Update 更新：先更新DB，再删除缓存
func (c *CacheAside) Update(ctx context.Context, key string, newUser *User) error {
	// 1.先更新数据库
	err := c.db.UpdateUser(ctx, key, newUser)
	if err != nil {
		return err
	}
	// 2.删除缓存，而不是set新数据
	return c.client.Del(ctx, key).Err()
}

// Delete 删除：删DB，删缓存
func (c *CacheAside) Delete(ctx context.Context, key string) error {
	err := c.db.DeleteUser(ctx, key)
	if err != nil {
		return err
	}
	return c.client.Del(ctx, key).Err()
}
```

## 六、Pipeline和Redis事务

### Pipeline（管道）

Pipeline 本质：批量发送多条 Redis 命令，只进行 1 次网络往返（1 次 RTT）
Redis 客户端默认：发一条命令 → 等 Redis 返回结果，再发下一条。多条命令就有多次网络 IO。
Pipeline：客户端先把所有命令缓存到本地，最后一次性打包发给 Redis；Redis 全部执行完，一次性把所有结果返回。

一次网络请求批量发送多条命令，减少网络 IO 开销，**命令之间不保证原子性**。适合批量操作。

```go
pipe := client.Pipeline()
pipe.Set(ctx,"k1","v1",0)
pipe.Get(ctx,"k1")
pipe.Incr(ctx,"counter")
//一次性发送所有命令
results,err := pipe.Exec(ctx)
```

### Redis 事务 + Watch 乐观锁

`Watch`监听 key，如果事务执行前 key 被别人修改，事务直接失败。实现乐观锁。

Redis 事务最大重点：**不支持回滚！** 和 MySQL 这种数据库事务完全不一样。

- 如果**语法错误、命令不存在**：EXEC 的时候，整个事务所有命令都不会执行
- 如果**语法没问题，运行时出错**（比如对字符串执行 INCR）：错误的那条命令失败，**其他正常命令依旧执行，不会回滚**

```go
client.Watch(ctx, func(tx *redis.Tx) error {
    //读取数据
    n, _ := tx.Get(ctx, "counter").Int()
    n += 10
    //事务管道
    _, err := tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
        pipe.Set(ctx,"counter",n,0)
        return nil
    })
    return err
}, "counter") //监听counter这个key
```

1. `client.Watch(..., "counter")`：监听`counter`key
2. 回调函数内部：先读取当前 counter 的值
3. `tx.TxPipelined`：等价 MULTI + 批量命令 + EXEC，是 redis/v9 封装好的事务管道
4. 内部逻辑：读取 n，n+10，然后 set 回 redis
5. 执行判断：
   - 如果在【Get 读取数据】之后、【TxPipelined 事务提交】之前，有别的客户端修改了`counter` → Watch 检测到 key 变动，事务直接失败，返回 err
   - 无冲突：事务正常执行

> 注意：Watch 只对本次事务生效，EXEC 成功后自动取消监视；事务失败，业务一般**循环重试**。

##### 乐观锁场景举例（并发扣余额 / 计数器）

两个客户端同时执行上面代码，都读取 counter=100，都想 + 10

- 客户端 A：先提交事务，counter 被改成 110
- 客户端 B：Watch 发现 counter 已经被修改，事务执行失败，需要业务重试读取新值再执行

> Redis 事务最大重点：**不支持回滚！** 和 MySQL 这种数据库事务完全不一样。
>
> - 如果**语法错误、命令不存在**：EXEC 的时候，整个事务所有命令都不会执行
> - 如果**语法没问题，运行时出错**（比如对字符串执行 INCR）：错误的那条命令失败，**其他正常命令依旧执行，不会回滚**



## 七、集群模式

### 1. Redis Cluster 分片集群

数据分片存储在多个 master 节点，自动分片，水平扩容。go-redis 提供`redis.NewClusterClient`

```go
client := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs: []string{"localhost:7000","localhost:7001"},
})
```

### 2. Sentinel 哨兵模式

哨兵监控主从节点，主节点宕机自动故障转移，切换主库。客户端使用`NewFailoverClient`连接哨兵集群。

## 八、缓存三大经典问题

### 1. 缓存穿透

**现象**：查询**数据库不存在的数据**，缓存永远不会命中，请求直接打到数据库。

> 例子：查询 id=-1 的用户，Redis 没有，数据库也没有。大量这类请求直接压垮 DB。

- 解决方案：
  1. **缓存空值**：查询 DB 发现不存在，在 Redis 写入空标记，设置较短过期时间。
  2. **布隆过滤器**：提前把所有存在的 id 放入布隆过滤器；请求先过过滤器，不存在直接拦截，不去查 Redis 和 DB。

### 2. 缓存雪崩

**现象**：大量缓存 key**同一时间过期**，大量请求同时落到数据库，数据库瞬间压力暴增，宕机。

- 原因：上线批量设置 key 过期时间，过期时间完全一致。

- 解决方案：

  过期时间增加随机值

  ```
  // 原本30分钟，加上0~300秒随机值，打散过期时间
  expire := 30*time.Minute + time.Duration(rand.Intn(300))*time.Second
  redis.Set(ctx, "user:"+id, data, expire)
  ```

  其他方案：Redis 集群高可用、服务限流降级。

### 3. 缓存击穿

**现象**：**热点 key 过期瞬间**，大量并发请求同时打到数据库。

> 例子：秒杀商品，千万人访问，这个商品 key 刚好过期，大量请求同时进入查 DB 逻辑。 和雪崩区分：雪崩是大量 key 同时过期；击穿是**单个热点 key 过期**。

- 解决方案：
  1. **互斥锁**：缓存失效时，只允许一个请求去查 DB、回写缓存；其他请求等待。
  2. **热点 key 永不过期**：代码层面不设置过期时间，后台异步更新缓存。