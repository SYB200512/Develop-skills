# Kafka完整讲解

Kafka是分布式事件流平台，常被当成高性能消息中间件。核心特点：**持久化存储、高吞吐、分区并行、多消费组回放数据**

典型场景：日志采集、用户行为埋点、系统解耦、流量削峰填谷、实时数据管道

## 一、整体框架

![image-20260918205429333](C:\Users\12472\AppData\Roaming\Typora\typora-user-images\image-20260918205429333.png)

### 1. Broker

> Kafka 的服务节点，一台部署 Kafka 的服务器就是一个 Broker。多个 Broker 组成 **Kafka 集群**。

- 职责：接收生产者消息、存储分区数据、处理消费者请求、维护副本。
- 集群：多台 Broker，分区分散在不同机器，实现横向扩容和容灾。

> 新版本 Kafka（KRaft 模式）不再依赖 Zookeeper；老版本依赖 Zookeeper 保存元数据（topic、分区、leader 信息）

### 2. Topic（主题）

消息的**逻辑分类**，相当于数据库里的一张表。生产者往 Topic 发消息，消费者订阅 Topic。

- 同一个 Topic 可以有多个生产者写入，多个消费组订阅。
- Topic 本身不存数据，数据存在它下面的**Partition（分区）**。

### 3. Partition（分区）

Topic 会拆成 1 个或多个分区，**物理存储单元**。

> 分区本质：一个**只能追加写入（append-only）**的日志文件，写完不能修改，只能往后追加新消息。 重点规则： ✅ **分区内消息有序；Topic 全局无序**。只有同一个分区内消息保证顺序；不同分区之间消息没有顺序。 ✅ 分区分散在集群不同 Broker，所以可以横向扩展吞吐量。 ✅ 分区数量：一般建议大于集群 Broker 数，分区越多并行度越高，但过多会增加元数据开销。

### 4. Offset（偏移量）

分区内**消息的唯一递增编号**，可以理解为消息在分区日志里的行号。

- 生产者写入消息，Kafka 自动分配 offset，从 0 开始不断递增。
- **消费者的 offset：书签**，记录这个消费者组读到分区哪个位置。
- offset 保存在 Kafka 内置主题 `__consumer_offsets`，不是存在客户端。

> 重要：消息被消费之后**不会立刻删除**，Kafka 会保留一段时间（由保留策略控制），所以消费者可以重置 offset，重新回放历史消息。

### 5. Replica（副本）

每个分区可以配置多个副本（`replication-factor` 副本因子），用来做高可用，防止 Broker 宕机丢数据。

- **Leader 副本**：分区主副本。**所有读写请求都走 Leader**。生产者写消息、消费者读消息，全部访问 Leader。
- **Follower 副本（从副本）**：只负责拉取 Leader 数据同步，不处理读写。Leader 挂掉，会从 Follower 里选举新 Leader。

#### ISR：In-Sync Replicas（同步副本集合）

和 Leader 保持同步、没有落后太多消息的副本集合。

- 生产者写消息成功，必须保证消息同步到 ISR 内所有副本（根据 acks 配置）。
- Follower 如果同步太慢，会被踢出 ISR；追上后重新加入。

> OSR：Out-of-Sync Replicas，不在 ISR 里、数据落后的副本。

### 6. Producer（生产者）

发送消息到 Kafka Topic 的客户端。 生产者发消息分区策略：

1. 指定`key`：对 key 做 hash 取模，固定落到同一个分区（**相同 key 消息保证分区有序**）。业务里订单 id、用户 id 常做 key。
2. 不指定 key：轮询（round-robin）分发到各个分区。
3. 自定义分区器。

生产者核心参数：

- ```
  acks
  ```

  ：消息确认机制

  - acks=0：发完不管，吞吐量最高，丢消息风险大。
  - acks=1：Leader 写入成功就返回成功（默认）。
  - acks=all：消息同步到 ISR 全部副本，可靠性最高，吞吐下降。

- `retries`：发送失败重试次数

- `batch.size`：批量发送缓冲区大小，攒一批消息一起发，提升吞吐。

### 7. Consumer（消费者）

订阅 Topic，读取消息的客户端进程。

### 8. Consumer Group（消费者组）

一组`group.id`相同的消费者实例。**最核心机制，面试高频**

> 规则：**同一个分区，只能被同一个消费组内一个消费者消费**。 举例：Topic 有 4 个分区，消费组启动 2 个消费者，每个消费者分配 2 个分区。

- 组内消费者数量 ≤ 分区数。如果消费者 > 分区数，多余消费者空闲，拿不到消息。
- 组内消费者宕机：触发**rebalance（再平衡）**，分区重新分配给组内其他存活消费者。
- 不同消费组：互相独立。同一个 Topic 消息，每个消费组都可以完整消费一遍（广播效果）。

#### Rebalance（再平衡）

消费组成员变化（新增 / 下线消费者、分区数变化），Kafka 触发分区重新分配给消费者。

> 缺点：rebalance 期间消费暂停；频繁 rebalance 会造成消费卡顿。尽量避免频繁启停消费者。

### 9. Offset Commit（offset 提交）

消费者读完消息，提交 offset 告诉 Kafka：“这个位置之前消息我已经处理完了”。 两种提交方式：

1. **自动提交（enable.auto.commit=true）**：定时提交。风险：消息处理中途程序崩溃，已经提交 offset，消息丢失；或者消息处理成功，提交前宕机，消息重复消费。
2. **手动提交**：业务逻辑处理完成之后再提交 offset（推荐生产环境），分同步提交、异步提交。

> Kafka 消息模型没有 “删除消息” ACK，**只有 offset 标记消费位置**。因此 Kafka 天然会出现：
>
>  ✅ 重复消费（最常见） 
>
> ❗ 很难出现消息丢失（配置得当） 
>
> ❗ 不支持消息修改

### 10. Message / Record（消息 / 记录）

Kafka 里最小数据单元，结构：

- key：可选，用来路由分区
- value：业务数据（二进制字节，序列化，json/protobuf）
- timestamp：消息时间戳（生产者时间或 Broker 时间）
- headers：消息头

### 11. Log Retention 日志保留策略

Kafka 不会消费完就删消息，按策略清理旧日志：

1. 按时间：`retention.ms` 默认 7 天，到期删除日志段。
2. 按大小：`retention.bytes`，分区日志总大小超过阈值删除旧段。
3. Log Compaction（日志压缩）：保留同一个 key 的最新一条消息，老的同 key 消息删除。适合状态数据（比如用户最新信息）。

### 12. 五大 API

1. **Producer API**：发送消息到 Topic
2. **Consumer API**：订阅 Topic 消费消息
3. **Admin API**：创建 / 删除 topic、修改分区、查看集群元数据（替代命令行）
4. **Kafka Connect**：数据同步连接器，快速把 MySQL、文件、ES 数据导入 / 导出 Kafka（不用自己写生产者消费者）
5. **Kafka Streams**：流处理 API，在 Kafka 内部做实时计算（过滤、聚合、join）

## 二、消息发送&消费完整流程

### 生产者发送流程

1. Producer 拿到消息，选分区（key hash / 轮询）
2. 消息放入生产者本地缓冲区，批量打包
3. 发送到该分区对应的 Leader Broker
4. Leader 写入本地日志，Follower 拉取同步
5. 根据 acks 配置，返回发送成功给生产者

### 消费者消费流程

1. Consumer 指定 group.id，订阅 topic
2. 加入消费组，触发分区分配
3. 从分区 Leader 拉取消息（从自己已提交 offset 开始读）
4. 执行业务处理
5. 提交 offset（自动 / 手动）
6. 持续拉取下一批消息

## 三、生产环境核心问题

### 1. 消息丢失场景

- 生产者：acks=0/1，Leader 写完还没同步，Broker 宕机。
- 消费者：自动提交 offset，业务还没处理完就提交 offset，进程 crash → 消息丢失。 ✅ 解决方案：生产者 acks=all；消费者**手动提交 offset，业务处理成功再提交**。

### 2. 消息重复消费（最常见）

业务处理成功，但是 offset 提交失败（网络抖动、进程挂掉），重启后消费者从旧 offset 重新消费，消息重复。 ✅ 解决方案：业务做**幂等**，同一个消息多次执行结果不变（唯一消息 id 做数据库唯一键）。

### 3. 消息积压

消费者处理速度远低于生产者写入速度，分区消息堆积，磁盘持续上涨。 排查：看消费组 offset 与分区最新 offset 差值（lag）。 解决：

1. 优化消费者业务逻辑，提升单实例速度
2. 增加消费者实例（不超过分区总数）
3. 增加分区数（扩容并行度）

### 4. 顺序保证

- 全局顺序：Topic 只设置 1 个分区，吞吐极低，一般不用。
- 业务分区顺序：消息带上业务 key（订单 ID），相同 key 落到同一个分区，分区内有序。

## 四、完整流程

### 第一步：创建 order-topic

打开终端，执行 kafka 命令（如果是 docker 部署的 kafka，要先进容器 `docker exec -it kafka bash`）

```bash
kafka-topics --bootstrap-server localhost:9092 \
--create \
--topic order-topic \
--partitions 3 \
--replication-factor 1
```

参数解释

- `--topic order-topic`：主题名字
- `--partitions 3`：3 个分区，最多支持 3 个消费者并行消费
- `--replication-factor 1`：副本数 1，测试环境用；生产环境≥2

验证是否创建成功

```bash
# 查看所有topic
kafka-topics --bootstrap-server localhost:9092 --list

# 查看topic详细信息
kafka-topics --bootstrap-server localhost:9092 --describe --topic order-topic
```

### 第二步：初始化 Go 项目，安装 sarama 依赖

新建文件夹，进入文件夹执行：

```bash
mkdir kafka-demo && cd kafka-demo
go mod init kafka-demo
go get github.com/Shopify/sarama
```

### 第三步：生产者代码 producer.go

```go
package main

import (
	"fmt"
	"github.com/Shopify/sarama"
	"log"
)

func main() {
	config := sarama.NewConfig()
	config.Producer.RequiredAcks = sarama.WaitForAll // acks=all，可靠性最高
	config.Producer.Retry.Max = 3
	config.Producer.Idempotent = true       // 幂等生产者，防止重复消息
	config.Producer.Return.Successes = true

	// 连接kafka地址
	producer, err := sarama.NewSyncProducer([]string{"localhost:9092"}, config)
	if err != nil {
		log.Fatal("生产者创建失败：", err)
	}
	defer producer.Close()

	fmt.Println("生产者启动成功，开始发送消息")
	// 循环发送5条订单消息
	for i := 1; i <= 5; i++ {
		msg := &sarama.ProducerMessage{
			Topic: "order-topic",
			Key:   sarama.StringEncoder(fmt.Sprintf("order_%d", i)),
			Value: sarama.StringEncoder(fmt.Sprintf("订单编号ORD%03d，金额：%d元", i, i*100)),
		}
		partition, offset, err := producer.SendMessage(msg)
		if err != nil {
			log.Printf("消息发送失败: %v\n", err)
		} else {
			fmt.Printf("发送成功！分区:%d, offset:%d\n", partition, offset)
		}
	}
	fmt.Println("全部消息发送完毕")
}
```

### 第四步：消费者代码 consumer.go

```go
package main

import (
	"context"
	"fmt"
	"github.com/Shopify/sarama"
	"log"
	"os"
	"os/signal"
	"syscall"
)

type ConsumerHandler struct{}

func (h *ConsumerHandler) Setup(_ sarama.ConsumerGroupSession) error {
	fmt.Println("=== Rebalance完成，消费者准备开始消费 ===")
	return nil
}

func (h *ConsumerHandler) Cleanup(_ sarama.ConsumerGroupSession) error {
	fmt.Println("=== 消费者会话结束 ===")
	return nil
}

func (h *ConsumerHandler) ConsumeClaim(sess sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	for msg := range claim.Messages() {
		fmt.Printf("【收到消息】topic:%s 分区:%d offset:%d key:%s value:%s\n",
			msg.Topic, msg.Partition, msg.Offset, string(msg.Key), string(msg.Value))

		// =========业务逻辑写在这里=========
		// 例如：解析订单JSON、入库
		// ==================================

		// 业务处理完成，标记offset
		sess.MarkMessage(msg, "")
	}
	return nil
}

func main() {
	config := sarama.NewConfig()
	config.Version = sarama.V3_5_0_0
	config.Consumer.Group.Rebalance.GroupStrategies = []sarama.BalanceStrategy{sarama.NewBalanceStrategyRoundRobin()}
	config.Consumer.Offsets.Initial = sarama.OffsetOldest // 新消费组从头读取消息

	groupID := "go-order-consumer-group"
	consumerGroup, err := sarama.NewConsumerGroup([]string{"localhost:9092"}, groupID, config)
	if err != nil {
		log.Fatal("消费者组创建失败：", err)
	}
	defer consumerGroup.Close()

	// 监听终止信号，优雅关闭
	ctx, cancel := context.WithCancel(context.Background())
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigChan
		fmt.Println("\n收到退出信号，准备关闭消费者")
		cancel()
	}()

	handler := &ConsumerHandler{}
	topics := []string{"order-topic"}
	fmt.Println("消费者启动，订阅topic order-topic")

	for {
		err := consumerGroup.Consume(ctx, topics, handler)
		if err != nil {
			log.Printf("消费异常：%v\n", err)
		}
		if ctx.Err() != nil {
			break
		}
	}
	fmt.Println("消费者退出")
}
```

### 第五步：运行测试（**新开两个终端窗口**）

终端 1：启动消费者，先启动！

```bash
go run consumer.go
```

看到打印 `消费者启动，订阅topic order-topic` 代表成功，等待接收消息

终端 2：启动生产者

```bash
go run producer.go
```

### 现象观察

1. 生产者终端：打印每条消息发送成功的分区号和 offset
2. 消费者终端：实时打印收到的订单消息，key、value、分区、offset

### 第六步：常用调试命令（测试用）

```bash
# 控制台生产者，手动发消息
kafka-console-producer --bootstrap-server localhost:9092 --topic order-topic

# 控制台消费者，从头消费
kafka-console-consumer --bootstrap-server localhost:9092 --topic order-topic --from-beginning

# 查看消费组状态，查看消息积压LAG
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group go-order-consumer-group
```

## 常见报错排查

1. `connection refused`：Kafka 没启动，或者地址不是[localhost:9092](https://localhost:9092)
2. `topic does not exist`：topic 创建命令执行失败，检查命令
3. 消费者收不到消息：确认先启动消费者；`OffsetOldest`只有新消费组第一次启动才会读历史消息