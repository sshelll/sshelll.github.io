---
title: 一种高性能的 Kafka 消费模型
date: 2023-09-16 20:44:44
updated: 2023-09-16 20:45:44
excerpt: 本文介绍了一种基于 Golang 的高性能 Kafka 消费模型
categories: tech
math: true
tags: 
  - Golang
  - Kafka
---

> Kafka 无疑是现在最流行的消息队列。



## 闲话

对于消息队列的选型，在国内环境中， RocketMQ 是一个强大的对手，网络上也有各种各样的对比和指导思想，有兴趣的读者可以自行搜索查看，在本文中并不会让这两者分个高下。

网络上林林总总的文档中，比较有参考意义的是 Apache RocketMQ 官方文档中的对比：[Apache RocketMQ 5.0 中文文档](https://rocketmq.apache.org/zh/docs/)。

可以看到，相比之下RocketMQ 是一个更为“现代化”的消息队列中间件，原生支持了 Kafka 不具备的许多功能，同时也对 Kafka 已有的功能做了一些补充和改进。同时，从文中也可以看到 RocketMQ 相比 Kafka 有着“低延时”和“高可用”的优点，但 Kafka 相比之下拥有更高的“吞吐量”。

此外，官方文档中没有对两者的性能做出详细的说明，而阿里巴巴官方在外网发布了这篇文章作为补充：[基于 Topic 数量的性能压测](https://alibaba-cloud.medium.com/kafka-vs-rocketmq-multiple-topic-stress-test-results-d27b8cbb360f)。



但是，就笔者的经验来说，在大部分实际的技术选型中，有以下三个“反直觉”的真相：

1. 许多细节上的差异并不会给项目带来明显的优势。举一个实际的例子来说：在绝大部分情况下，将接口响应时间从 10s 优化至 100ms，对比将响应时间从 100ms 优化至 1ms ，给用户带来的体感是完全不同的（后者远小于前者），同时付出的成本也是完全不同的（后者远大于前者）。
2. 社区活跃度的重要程度往往被我们轻视——当你遇见问题的时候，更活跃的社区往往能给你带来更短的定位 / 解决时间。这里是一个自带活跃度对比的外网社区的统计结果：[社区活跃度的对比](https://stackshare.io/stackups/kafka-vs-rocketmq)。从中可以看到，RocketMQ 在全球范围内的社区影响力远低于 Kafka。
3. 不同的公司的技术选型思路有巨大的差异。举例来说，我在字节跳动工作期间，公司内部所使用的云平台是自研的“字节云”。在字节云中，从 Golang 的镜像，到云服务器的 Linux 内核，都是公司自研的——根据公司的实际情况做出了相应的优化。当时在云平台的官方文档中，基础架构的同学写下了一段大致如下的描述：“强烈推荐各业务方从 Kafka / BMQ 切换到 RocketMQ，当前字节云中的 RocketMQ 中间件从各方位对比前者都有显著的优势” —— 有时候，只需要这样一句话，就能打消业务方几乎所有的疑问和顾虑（同时也解决了 2 中所描述的社区问题）。

总的来说，所有技术选型都应该基于实际情况进行决策。就像 CAP 定理和“软件工程没有银弹”一样，世界上没有完美的解决方案，到底是否应该“以牺牲部分吞吐量为代价，来获得 RocketMQ 同步刷盘的能力”，还是“它们都太差了，我要自研一款属于我的 MQ”，是需要根据实际业务场景来决定的。

而我现在所处的公司，在决策时面临的困难显然不那么强烈——我们使用亚马逊的 AWS 来部署服务，而 AWS 官方并没有提供基于 RocketMQ 的 Paas 服务，但 AWS MSK （Managed Streaming for Apache Kafka）已经相当成熟了，我们也就自然而然的选择了 Kafka 作为我们的消息队列，事实也证明，即便是和阿里一样同样置身在金融行业中，我们使用了所谓相比之下没有那么“高可靠、低延时”的 Kafka，也并没有给我们带来任何的困扰。



## 正文

### 基于 Golang 的 Kafka SDK

相同的中间件在不同编程语言中的实现的 SDK 必然是有差异的，它们往往会根据语言自身的特点做出相应的适配和优化。我所采用的 sdk 是 segmentio 公司研发的 [kafka-go](https://github.com/segmentio/kafka-go)，是 golang 中非常流行的一个库，也是适配得较好的一个库。

这个库的实现原理大致如下：

![kafka-go sdk](/img/kafka-go/kafka-go-sdk.png)

1. kafka-go 会向 Kafka Broker 拉取一定数量的消息放入内存中的 msg chan（该 chan 的缓存大小是可配置的）
2. 然后应用程序通过调用 `FetchMessage` 方法来从这个 chan 中获取到一个消息并进行消费
3. 当消费成功后，应用程序通过调用 `CommitMessages` 方法来提交 commit request
4. 根据配置，kafka-go 会将 commit chan 中的请求立即提交，或将其 merge 合并为一个请求异步提交



### 简单的阻塞消费模型

最简单的消费模型其实和上面描述的过程是一致的，如下：

![一次消费一条消息](/img/kafka-go/simple.png)

当消费成功时直接提交；当消费失败时，根据需要进行无限重试，或是直接丢弃（提交 commit 请求视为成功）。

---

常用 RocketMQ 的用户可以思考一下为什么我们一定要阻塞在某一条消息的处理上，为什么不能使用一个 `sync.WaitGroup` 来进行一定程度的并发呢？

曾经我在字节工作期间使用 RocketMQ 时，确实是这么做的，就像在接到 HTTP 请求后开启协程来处理一样，在接到消息后也开启协程进行处理。但这是建立在 RocketMQ 本身是支持消息重试的基础上的，当 RocketMQ 消息消费失败后，我们可以显示地向 RocketMQ 提交一个 `NACK` 请求来表明消息处理失败了，此时消息会被 RocketMQ 丢入重试队列中等待二次消费，当失败次数达到阈值后，会被其放进死信队列中，消息永远是不会丢失的。

但对于 Kafka 而言，它的提交行为是不区分 `ACK ` / ` NACK` 的，它的 commit 请求中只包含一个 `offset` 参数来标识当前消费者的消费进度。与此同时，kafka-go sdk 又在内存中将消息缓存到 chan 中，当我开启若干个协程调用 FetchMessage 方法时，获取到的消息是完全不同的。也就是说，当我开启三个协程处理三条消息时，可能分别在处理 offset = `1` / `2` / `3` 的 3 条消息，当 1 和 2 处理失败但 3 处理成功时，一旦将 3 commit 到 Kafka 中，1 和 2 就永久的丢失了（再也不会被消费了）。

---

这种模型虽然很简单，但也十分低效，适合的场景也非常有限，举几个实际的例子：

1. 新增 / 注销账号产生的消息。类似的场景下消息产生的 QPS 很可能 < 1，因此使用该模型并不会造成性能瓶颈，反而在一定程度上可以提升开发效率。
2. 需要使用顺序消费的场景。也就是说，在消费某条消息之前，必须消费完这条消息之前的所有消息，例如涉及到某些状态流转的场景，阶段 A 不能直接转换为阶段 C，而是必须经过阶段 B，此时我们需要顺序消费 A->B、B->C 这两条消息。



### 高性能的批量消费模型

在 Kafka 不支持 `ACK` / `NACK` 的情况之下，如果我们仍需要保证消费的有序性，是不是就无法使用并发消费了呢？当然不是。接下来将介绍一种可配置、高并发的高性能消费模型：

#### 以 Partition 为维度并发

熟悉 Kafka 架构的同学可能知道，Kafka 同一个 Topic 下的消息是分为多个 Partition 进行存储的，每个 Partition 中的消息都是按照投递的顺序进行排序的，也就是说，我们在消费同一个 Topic 的情况下，至少可以进行 Partition 维度的并发——就像在 RocketMQ 中，每个 Queue 中的消息是局部有序的。

![根据 partition 并发处理](/img/kafka-go/by_partition.png)

在这种模型中，我们需要做的主要有两件事：

1. 实现一个简单的 Dispatcher，根据 Kafka Message 的 Partition 进行分组后，将这条消息投入对应 Partition 的 chan 中
2. 实现一个通用的 Worker，用于处理 Partition Chan 中的消息

以下是我写的一段核心代码：

``` go
type KafkaConsumer[T msgtype.KafkaMessage] struct {
	callback           func(msgs ...T)
	partitionMsgs      map[int]chan T
	partitionQueueSize int
  
	minBatchSize   int
	maxBatchSize   int
	forceFlushTime time.Duration
  
	closeCh chan struct{}
}

func (c *KafkaConsumer[T]) Run() {
	go func() {
		log.Info("kafka consumer started")
		for {
			msg := c.fetchMsg()
			// 优雅退出
			select {
			case <-c.closeCh:
				log.Info("kafka consumer stopped")
				return
			default:
			}
			c.parallelHandle(msg)
		}
	}()
}

func (c *KafkaConsumer[T]) Stop() {
	close(c.closeCh)
}

func (c *KafkaConsumer[T]) parallelHandle(msg T) {
	partition := msg.RawMessage().Partition

	if _, ok := c.partitionMsgs[partition]; !ok {
		c.startPartitionHandler(partition)
	}

	c.partitionMsgs[partition] <- msg
}

func (c *KafkaConsumer[T]) startPartitionHandler(partition int) {
	// 初始化分区消息队列
	c.partitionMsgs[partition] = make(chan T, c.partitionQueueSize)

	// 启动分区消费者
	go func() {
		flushTimer := time.NewTimer(c.forceFlushTime)

		for {
			msgBatch := make([]T, 0, c.maxBatchSize)

			// 先取一条消息, 避免直接进入计时
			msgBatch = append(msgBatch, <-c.partitionMsgs[partition])
			flushTimer.Reset(c.forceFlushTime)

			func() {
				for len(msgBatch) < c.maxBatchSize {
					select {
					// 超时退出
					case <-flushTimer.C:
						return
					// 接收到新的 msg
					case msg := <-c.partitionMsgs[partition]:
						msgBatch = append(msgBatch, msg)
					// 暂时没有新的消息到来，但也尚未超时，此时缓存的消息若满足最小接收量则退出
					default:
						if len(msgBatch) > c.minBatchSize {
							return
						}
						// 再次 select, 避免空转
						select {
						case <- flushTimer.C:
							return
						case msg := <-c.partitionMsgs[partition]:
							msgBatch = append(msgBatch, msg)
						}
					}
				}
			}()

			// callback 中根据业务自行实现重试保证成功或是 drop
			c.callback(msgBatch...)
			c.CommitMessages(msgBatch...)
		}
	}()
}
```

可以看到，在实际的代码实现中，会加入许多额外的细节来提供健壮性，其中使用了 `map[int]chan T`  结构来维护消息的分组过程，也就是实现了上述的 `Dispatcher`。除此之外，在 `startPartitionHandler(int)` 方法中，以异步的形式实现了前文中的`Worker`。

在 Worker 的具体实现中，我使用了 3 个可配置的选项来根据实际情况动态的调控消费行为，分别是：

1. `forceFlushTime` ，每次处理消息前最大的循环时间——防止一直接收不到足量的消息而阻塞
2. `maxBatchSize`，每次批量处理消息的最大数量——防止一次性接收过多消息
3. `minBatchSize`，每次批量处理消息的最小数量——某些时候上游已经暂时不再产生消息，此时为了避免持续空转到超时，可以提前返回

---

#### 从单条转向到批量处理消息

从上面的 `KafkaConsumer` 中可以看到，我们对于消息的处理从单条转向了批量。在批量处理的过程中，我们需要保证所有消息都处理成功后（至少是业务上认为处理成功），再返回成功，常见的处理方式就是无限重试。在处理完一批消息之后，我们需要批量的提交消息，kafka-go 本身就提供了批量处理消息的函数，但是我们也需要进行一些合理的封装来实现如下功能：

1. 给 kafka 对应的 partition 提交 offset
2. 合理拆分消息数组，避免一次性提交的消息大小超过 kafka 的配置限额
3. 对 kafka 的连接进行合理的检测

伪代码如下：

```go
type KafkaPartitionProducer[T msgtype.KafkaMessage] struct {
	brokers   []string
	topic     string
	partition int
	conn      *kafka.Conn
	codec     *kafka.CompressionCodec

	logger *logrus.Entry
}

func (pp *KafkaPartitionProducer[T]) ProduceMust(msgs ...T) {
	if pp.codec == nil {
		panic("codec is nil")
	}

	if pp.conn == nil {
		pp.connectMust()
	}

	// 构造 kafka 消息
	kmsgs := make([]kafka.Message, 0, len(msgs))
	for _, msg := range msgs {
		kmsg := pp.buildKafkaMessage(msg)
		if kmsg == nil {
			continue
		}
		kmsgs = append(kmsgs, *kmsg)
	}

	if len(kmsgs) == 0 {
		pp.logger.Warn("no messages to produce")
		return
	}

	// 无限重试直到发送成功
	for i := 0; ; i++ {
		nbytes, _, offset, _, err := pp.conn.WriteCompressedMessagesAt(*pp.codec, kmsgs...)

		if err == nil {
			log.Infof("write compressed messages success")
			break
		}

		switch err {
		case kafka.MessageSizeTooLarge:
			log.WithError(err).Errorf("kafka message size too large")
			if len(kmsgs) <= 1 {
				panic("kafka message size too large")
			}
			// 消息过多对半递归重发
			mid := len(kmsgs) / 2
			pp.ProduceMust(msgs[:mid]...)
			pp.ProduceMust(msgs[mid:]...)
		default:
			// 其它错误直接重连
			log.WithError(err).Errorf("failed to write compressed messages")
			pp.connectMust()
		}
	}
}

func (pp *KafkaPartitionProducer[T]) connect() error {
	if pp.conn != nil {
		pp.conn.Close()
		pp.conn = nil
	}

	for _, addr := range pp.brokers {
		log := pp.logger.WithField("addr", addr)

		ctx, canceler := context.WithTimeout(context.Background(), 5*time.Second)
		defer canceler()

		// 连接到指定 broker 的 topic / partition
		conn, err := kafka.DialLeader(ctx, "tcp", addr, pp.topic, pp.partition)
		if err != nil {
			log.WithError(err).Errorf("failed to dial leader for partition")
			continue
		}

		// 测试 kafka 连接是否正常
		first, err := conn.ReadFirstOffset()
		if err != nil {
			log.WithError(err).Error("failed to read first offset")
			conn.Close()
			continue
		}

		last, err := conn.ReadLastOffset()
		if err != nil {
			log.WithError(err).Error("failed to read last offset")
			conn.Close()
			continue
		}

		log.WithFields(logrus.Fields{
			"first": first,
			"last":  last,
		}).Info("dial conn success")

		pp.conn = conn
		break
	}

	// 没有可用的 broker 连接
	if pp.conn == nil {
		pp.logger.WithField("brokers", pp.brokers).Error("failed to dial leader from all brokers")
		return fmt.Errorf("failed to dial leader for partition %d", pp.partition)
	}

	return nil
}
```

## 后话

Kafka 的 Consumer Gourp 离不开 Rebalance 机制，所谓 Rebalance 指的就是将一个 Topic 下若干 Partition 通过协商的过程平均分配给同一个 Consumer Group 中不同 Consumer 的过程。也就是说，一个 Partition 只能被一个 Consumer Group 中的一个消费者消费，也就让我们可以容易的实现局部顺序消费，配合上 Producer 端的少许逻辑，就可以达成业务上的顺序性。

![consumer group 与 partition](/img/kafka-go/kafka-consumergroup.png)

在上图中，以订单的流转为例，只需要 Producer 在生产消息时根据订单 id 分配固定的 Partition 有序的发送消息（如对于订单221，将发起订单、流转订单、完成订单三个操作依次发送到 P1 中），下游的 Consumer 就能保证同一个订单的消息被按序处理，因为同一个 Partition 中的消息不会同时被两个 Consumer 消费。

但是有些极端的情况仍可能导致重复消费的错误，例如 C1 在消费完「发起订单」后，还没来得及 Commit 就挂掉了，Rebalance 后 C2 或 C3 又会重新接收到该消息，并尝试再次发起订单，因此业务方自行保证消息消费的幂等性是十分有必要的。
