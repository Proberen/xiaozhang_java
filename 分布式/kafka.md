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
zookeeper.connect=localhost:2181
```

3、启动zookeeper

```c
nohup zookeeper-server-start /usr/local/etc/kafka/zookeeper.properties &
```



















































































