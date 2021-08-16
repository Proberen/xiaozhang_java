# 1 引入

1、Kafka是一个多分区、多副本且基于 ZooKeeper 协调的分布式消息系统，是一个**分布式**流式处理平台，以高吞吐、可持久化、可水平扩展、支持数据处理等多种特性被广泛使用

2、Kafka是一个**分布式**的**基于发布/订阅模式**的消息队列

2、三大角色：

- **消息系统：**系统解耦、冗余存储、流量削峰、缓冲、异步通信、扩展性、可恢复性等功能，还提供了消息顺序性保障和回溯消费功能
- **存储系统：**持久化功能和多副本机制
- **流式处理平台：**提供一个完整的流式处理类库，比如窗口、连接、变换、聚合等操作

## 1.1 消息队列

1、应用场景—异步处理

![在这里插入图片描述](https://img-blog.csdnimg.cn/c4101c2f41a84adfa9c87e26d6b6ed99.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

**2、好处**

- 解耦
  - 允许你独立的扩展或修改两边的处理过程，只需要确保它们遵守同样的接口约束
- 可恢复性
  - 系统的一部分组件失效时，不会影响整个系统，消息队列降低了进程间的耦合度，所以即使一个处理消息的进程挂掉，加入队列的消息仍然可以在系统恢复后被处理
- 缓冲
  - 有助于控制和优化数据流经过系统的速度，解决生产和消费处理速度不一致的问题
- 灵活性 & 峰值处理能力
  - 在访问量剧增的情况下，应用仍然需要继续发挥作用，但是这样的突发流量并不常见。使用消息队列能够使关键组件顶住突发的访问压力，而不会因为突发的超负荷的请求而崩溃
- 异步通信
  - 用户不需要立刻处理信息，消息队列提供了异步处理机制，允许用户把一个消息放入队列，但是不立刻处理。

**3、消息队列的两种模式**

- **点对点模式**（一对一，消费者主动拉取数据，消息收到后消息清除）

  - 消息生产者生产消息后发送到queue中，然后消费者从queue中取出并且消费消息。消息被消费后queue中不再有存储，所以**消费者不能消费到已经被消费的消息**。queue支持存在多个消费者，但是对于一个消息只有一个消费者可以消费

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/279331ccc8a64fbc8c43f3163fc7e93e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

- **发布/订阅模式**（一对多，消费者消费后不会清除消息）

  - 消息生产者（发布）将消息发布到topic中，同时有多个消息消费者（订阅）消费该消息，和点对点的方式不同，发布到topic的消息会被所有订阅者消费
  - 两种方式：
    - 类似于公众号，topic队列进行主动推送
    - 消费者**主动拉取**（Kafka），消费者可以控制自己的消费速度，但是需要长轮询来询问是否有新消息，比较浪费资源

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/c2dff56beed64df381d1846e10913f1b.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

## 1.2 Kafka基本概念

Kafka体系结构：

- 若干**生产者**（Producer）：将数据发送到Broker
- **消费者**：从Broker订阅并消费信息
  - **消费者组**：同一个消费者组内的不同消费者不能消费同一个分区数据
    - 好处：提高消费能力
- **Kafka 集群**（Kafka Cluster）包含若干**Broker**：将收到的消息存储到**磁盘**中（默认保存7天）
  - Topic **主题**
  - Partition **分区**：为了实现拓展性，一个topic可以分布到多个broker上，每个partition是一个有序队列
  - Replica **副本**：为了保证集群中某个节点发生故障时，该节点上的partition数据不丢失，且kafka能过继续工作，一个topic的每个分区都有若干副本（**一个Leader和多个Follower**）
    - Leader：主，生产者发送数据的对象和消费者消费数据的对象
    - Follower：从，实时从leader同步数据，保持和leader数据同步
- **Zookeeper**：
  - 帮助kafka集群存储一些信息
  - 帮助消费者存储消费的位置信息offset
    - 0.9版本之前存储在zk
    - 0.9版本之后存储在kafka本地

![在这里插入图片描述](https://img-blog.csdnimg.cn/23a2ec6a4edd4847a930ca7c31ad5eba.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

## 1.3 mac下Kafka的安装

1、`brew install kafka`

2、修改配置文件，目录：`/usr/local/etc/kafka/server.properties`

```properties
# broker的全局唯一编号，不能重复，int类型
broker.id=0

# 允许日志存储路径
log.dirs=/usr/local/var/lib/kafka-logs

# 存储时间：168个小时
log.retention.hours=168

# 配置zookeeper
zookeeper.connect=localhost:2181/kafka
```

3、启动zookeeper

```bash
zkServer.sh start
```

4、启动kafka

```bash
kafka-server-start -daemon /usr/local/etc/kafka/server.properties
```

5、常见命令

```bash
-- 列出所有topic
kafka-topics --zookeeper localhost:2181/kafka --list
```

```bash
-- 创建topic
kafka-topics --zookeeper localhost:2181/kafka --create --replication-factor 1 --partitions 3 --topic test

--topic 定义topic名字
--replication-factor 定义副本数
--partitions 分区数
```

```bash
-- 删除topic
kafka-topics --zookeeper localhost:2181/kafka --delete --topic test
```

```bash
-- 详情
kafka-topics --zookeeper localhost:2181/kafka --describe --topic test
```

```bash
-- 生产者
kafka-console-producer --topic test --broker-list localhost:9092
```

```bash
-- 消费者
kafka-console-consumer --topic test --bootstrap-server localhost:9092 --from-beginning

--from-beginning 从开头进行消费
```

# 2 高级

## 2.1 工作流程

kafka中的消息以topic进行分类，生产者生产消息，消费者消费消息，都是面向topic的。

**topic**是**逻辑**上的概念，而**partition**是**物理**上的概念，每个partition对应一个log文件，该log文件存储的就是producer生产的数据。producer生产的数据会被不断追加到log文件，且每条数据有自己的offset。消费组中的每个消费者都会实时记录自己消费到了哪个offset。

![在这里插入图片描述](https://img-blog.csdnimg.cn/75698a26aee845eaa0adcec32d0fc045.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

由于生产者生产的消息会不断追加到log文件末尾，为了防止log文件过大导致数据定位效率低下，kafka采取了**分片**和**索引**机制，将每个partition分为多个segment。每个segement对应两个文件：**.index文件和.log文件**。

这些文件位于一个文件夹下，该文件夹的命名规则为：topic名称+分区序号

> 例如：test1这个topic有三个分区，其对应的文件夹为test1-0,test1-1,test1-2

![在这里插入图片描述](https://img-blog.csdnimg.cn/47a83bf3ac7848f5bcf06a0f83e18141.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

index和log文件以当前segment的第一条消息的offset命名

通过二分查找法找到index，根据index找到消息数据，根据消息内容区log中定位数据位置

## 2.2 生产者

### 2.2.1 分区策略

**1、分区的原因**

- 方便在集群中扩展，每个partition可以通过调整以适应它所在的机器，而一个topic可以有多个partition组成，因此整个集群就可以适应任意大小的数据了；
- 可以提高并发，可以以partition为单位读写

**2、分区的原则**

将producer发送的数据封装成一个**producerRecord**对象

- 指明partition的情况下，直接将指明的值作为partition值
- 没有指明partition但是有key的情况下，将key的hash值与topic的partition数进行取余得到partition值
- 都没有的情况下，第一次调用时随机生成一个整数（后面每次调用在这个整数上自增），将这个值与topic可用的partition总数取余得到partition值，**round-robin算法**

### 2.2.2 数据可靠性保证

为了保证producer发送的数据，能可靠的发送到指定的topic，topic的每个partition收到数据后都需要向producer发送ack，如果producer收到ack，就会进行下一轮发送，否则重新发送数据。

> 问题：
>
> - 何时发送ack？
>   - 确保有follower与leader同步完成，leader再发送ack，这样才能保证leader挂掉之后能在follower中选举出新的leader
> - 多少个follower同步完成后发送ack？
>   - 方案一：半数以上
>   - 方案二：全部

**1、副本数据同步策略**

| 方案                      | 优点                                               | 缺点                                                |
| ------------------------- | -------------------------------------------------- | --------------------------------------------------- |
| 半数以上完成同步就发送ack | 延迟低                                             | 选举新的leader时，容忍n台节点的故障，需要2n+1个副本 |
| 全部完成同步发送ack       | 选举新的leader时，容忍n台节点的故障，需要n+1个副本 | 延迟高                                              |

Kafka选择第二种方案，原因如下：

- 同样为了容忍n台节点故障，第一种方案需要2n+1个副本，第二个方案只需要n+1个副本，kafka的每个分区都有大量数据，第一种方案会造成大量数据冗余
- 网络延迟对kafka的影响较小



**2、ISR**

> 问题：如果副本挂了或者一直没有同步成功，就不会发送ack
>
> 解决：ISR（同步副本）

```bash
/usr/local/Cellar/kafka/2.8.0> kafka-topics --zookeeper localhost:2181/kafka --describe --topic test1

Topic: test1	TopicId: 5s6UOUhjRPGrOlWi-eQqRA	PartitionCount: 3	ReplicationFactor: 1	Configs:
	Topic: test1	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
	Topic: test1	Partition: 1	Leader: 0	Replicas: 0	Isr: 0
	Topic: test1	Partition: 2	Leader: 0	Replicas: 0	Isr: 0
```

Leader维护了一个动态的ISR，意为**和leader保持同步的follower集合**。

当ISR中的follower完成数据同步之后，leader就会给follower发送ack。如果follower长时间没有向leader同步数据，该follower就会被踢出ISR，该时间阈值由`replica.lag.time.max.ms`参数设定。

**Leader发生故障之后，就会从ISR中选举新的leader**





















































