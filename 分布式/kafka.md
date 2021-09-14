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
  - **AR**：分区中所有的副本
  - **ISR**：与leader副本保持一定程度同步的副本（包括leader副本）
  - **HW**：高水位，标识特定的消息偏移量，消费者只能拉取这个offset之前的消息
  - **LEO**：标识当前日志文件下一条写入消息的offset

  > ISR、HW、LEO的关系
  >
  > - 初始状态：
  >   - Leader消息：0、1、2
  >   - follower1消息：0、1、2
  >   - follower2消息：0、1、2
  > - 发送消息3、4（情况1：follower2只收到3）
  >   - Leader消息：0、1、2、3、4
  >   - follower1消息：0、1、2、3、4
  >   - follower2消息：0、1、2、3
  >   - HW：4 ；LEO：5
  > - 发送消息3、4（情况2）
  >   - Leader消息：0、1、2、3、4
  >   - follower1消息：0、1、2、3、4
  >   - follower2消息：0、1、2、3、4
  >   - HW：5 ；LEO：5
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

# 2 kafka入门

## 2.1 工作流程

### 2.1.1 消息层次

**1、第一层：主题层**

- 每个topic可以配置M个partition，每个partition可以设置N个副本

**2、第二层：分区层**

- 每个partition的N个副本中只能有一个充当领导的角色，对外提供服务；其他N-1个副本是追随者副本，只是提供数据冗余之用

**3、第三层：消息层**

- partition包含若干条消息，每条消息的**位移**从0开始，依次递增

### 2.1.2 持久化数据

**1、消息日志（Log）**

- kafka使用log（**磁盘上一个只能追加写的物理文件，避免了缓慢的随机I/O，该用性能较好的顺序I/O**）来保存数据
- Kafka定期删除消息以回收磁盘
  - **日志段机制（Log Segment）**：将日志细分为多个日志段，消息被追加写入搭配当前最新的日志段中，写满一个日志段就自动切分一个新的日志段，将老的日志段封存起来，kakfa在后台有定时任务定期检查老的日志段是否能够被删除

<img src="https://img-blog.csdnimg.cn/75698a26aee845eaa0adcec32d0fc045.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

### 2.1.3 消费者

**1、点对点模型（P2P）**

- **消费者组（Consumer group）**：把多个消费者实例（可以是一个进程，也可以是一个线程）共同组成一个组来消费一组主题
  - **可以提高消费者端的吞吐量，多个消费者实例同时消费，加速整个消费端的吞吐量（TPS）**
  - **重平衡**：消费者组里所有的消费者彼此协助，如果某个实例挂了，就会把这个实例负责的分区转移给其他消费者，**kafka消费者实现高可用的重要手段**

**2、消费位移（offset）**

- 随时变化，表示消费者的消费进度
- 每个消费者都有自己的消费者位移



## 2.2 是消息引擎系统也是分布式流处理平台

**1、Kafka设计之初的目的是为了提供三个方面的特性**

- 提供一套API实现生产者和消费者
- 降低网络传输和磁盘存储开销
- 实现高伸缩性架构

**2、kafka Streams**

- 在大数据工程领域，Kafka在承接上下游、串联数据管道方面发挥重要的作用
- Kafka于0.10.0.0版本推出流处理组件**Kafka Streams**

**3、Kafka Streams的优势**

- **容易实现端到端的正确性**：处理一条消息有且只有一次机会能够影响系统状态
  - 其他框架只能保证在读取kafka消息之后的计算对于状态的影响只有一次，但是计算结果可能多次写入kafka
- **对于流式计算的定位**

4、**分布式存储系统**



# 3 kafka基本使用

## 3.1 Kafka线上集群部署方案

> 集群：多个kafka节点机器

### 3.1.1 操作系统

**1、部署在Linux系统**

- I/O模型的使用
- 数据网络传输效率
- 社区支持度

**2、I/O模型的使用**

- I/O模型就是操作系统执行I/O指令的方法
- **主流I/O模型：阻塞式I/O、非阻塞式I/O、I/O多路复用、信号驱动I/O、异步I/O**
  - java的Socket阻塞模式和非阻塞模式：阻塞式I/O、非阻塞式I/O
  - Linux系统的select函数：I/O多路复用
  - epoll系统调用：介于I/O多路复用、信号驱动I/O之间
- **kafka客户端使用java的selector，在linux的实现机制是epoll，在windows实现机制是select，因此部署在linux优势在于能够获得更高效的I/O性能**

**3、网络传输效率**

- kafka生产消费的消息都是通过网络传输的，消息保存在磁盘，因此kafka需要在磁盘和网络间大量数据传输
- **Linux 零拷贝（zero copy）技术**：避免将数据从磁盘复制到缓冲区，再将缓冲区数据发送到socket的性能损耗

> RocketMQ 选择了 `mmap + write` 这种零拷贝方式，适用于业务级消息这种小块文件的数据持久化和传输；
>
>  Kafka 采用的是 `sendfile` 这种零拷贝方式，**适用于系统日志消息这种高吞吐量的大块文件的数据持久化和传输**。但是值得注意的一点是，Kafka 的索引文件使用的是 `mmap + write` 方式，数据文件使用的是 `sendfile` 方式。
>
> <img src="https://img-blog.csdnimg.cn/ab51fbf1c25448d2add180fcc23cee26.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:50%;" />

### 3.1.2 磁盘

**1、机械磁盘 or 固态硬盘？**

- 机械磁盘：成本低且容量大，但是容易损坏
- 固态硬盘：性能优势大，但是成本高
- **选择机械硬盘即可**
  - kafka使用的方式多是顺序读写，一定程度上规避了机械磁盘的劣势（随机读写操作慢）
  - kafka在软件方面可以保证机械硬盘因易损坏造成的可靠性差等缺陷

**2、是否使用磁盘阵列（RAID）？**

- 使用RAID的优势：提供冗余的磁盘存储空间、提供负载均衡
- **kafka自身实现了冗余机制，通过分区也可以实现负载均衡**

**3、建议：**

- 追求性价比的公司可以不搭建 RAID，使用普通磁盘组成存储空间即可
- 使用机械磁盘完全能够胜任 Kafka 线上环境。

### 3.1.3 磁盘容量

**1、需要多大存储空间？**

- kafka需要把消息存储在磁盘，默认存储一段时间后自动删除
- kafka集群出了消息数据还有其它类型数据，比如索引数据等
- **例子**：假设消息平均大小1KB，每天1亿条1KB消息，保存两份且留存两周，总的空间大小等于200GB，加上为其他数据预留的10%磁盘空间，共220GB，保存两周后整体容量大约3TB左右，kafka支持消息压缩，假设压缩0.75，即需要2.25TB空间

**2、kafka规划磁盘容量需要考虑的因素**

- 新增消息数量
- 消息留存时间
- 平均消息大小
- 备份数量
- 是否启用压缩

### 3.1.4 带宽

1、**带宽容易成为瓶颈**

- kafka通过网络大量进行数据传输
- **在真实案例当中，带宽资源不足导致 Kafka 出现性能问题的比例至少占 60% 以上，如果还涉及跨机房传输，那么情况可能就更糟了**

> 一般公司网络使用普通以太网的1Gbps千兆网络，如何进行带宽资源的规划？

**真正需要规划的是kafka服务器的数量**

假设机房环境是千兆网络，即 1Gbps，现在的业务目标是在 1 小时内处理 1TB 的业务数据。需要多少台 Kafka 服务器来完成这个业务呢？

- **带宽1Gbps，即每秒处理1Gb数据**，假设每台kafka服务器都在专属机器上，没有其他服务。通常kafka会用到70%的带宽资源（需要为其他进程留资源，超70%的阈值会出现网络丢包），也就是说**kafka服务器最多使用大约700Mb的带宽资源**
- kafka服务器不能常规性使用这么多带宽资源，通常需要额外流出2/3的资源，即**每台服务器使用带宽240Mbps**
- 如果需要1小时处理1TB数据，每秒需要处理2336Mb数据，除以240，约等于10台服务器，如果消息还需要额外复制两份，那么总的服务器需要乘以3，即30台

### 3.1.5 总结

<img src="https://img-blog.csdnimg.cn/3f2237c2b1a84ae087daeba33f8750ca.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:50%;" />

## 3.2 集群参数配置

<img src="https://img-blog.csdnimg.cn/81fc0670ce294627aba9212935e91dc1.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:40%;" />

<img src="https://img-blog.csdnimg.cn/2f21c4f5351441898ac30e526c6b614f.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:40%;" />



# 4 生产者

```java
public class testProducer {
    public static final String brokerList = "localhost:9092";
    public static final String topic = "test";

    public static Properties intConfig() {
        Properties props = new Properties();
        props.put("bootstrap.servers", brokerList);
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("client.id", "producer.client.id.demo");
        return props;
    }

    public static void main(String[] args) {
        Properties props = intConfig();
        KafkaProducer<String, String> producer = new KafkaProducer<>(props);
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, "hello, Kafka 1");
        try {
            RecordMetadata recordMetadata = producer.send(record).get();
            System.out.println(recordMetadata.partition());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

## 4.1 发送消息

1、生产者发送消息的三种方式：Fire and Fogret、同步、异步

```java
public Future<RecordMetadata> send(ProducerRecord<K, V> record) 
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback)
```

- **Fire and Fogret**

  ```java
  producer.send(new ProducerRecord<String, String>("my-topic", Integer.toString(i), Integer.toString(i)));
  ```

- **同步**

  ```java
  方式一：producer.send(record).get()
  方式二：  
  Future<RecordMetadata> future = producer.send(record) ; 
  RecordMetadata metadata = future.get();
  System out.println(metadata top () + "-" + metadata.partition() + ":" + metadata.offset() );
  ```

- **异步**

  ```java
  producer.send(myRecord,new Callback() {
    public void onCompletion(RecordMetadata metadata, Exception e) {
      if(e != null) {
        e.printStackTrace();
      } else {
        System.out.println("The offset of the record we just sent is: " + metadata.offset());
      }
    }
  });
  ```

**源码分析**

- 异步：调用dosend方法（**同步的send，callback传了null，还是走了这个方法**）

  ```java
  @Override
  public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    // intercept the record, which can be potentially modified; this method does not throw exceptions
    ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
    return doSend(interceptedRecord, callback);
  }
  ```

  ```java
  private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    TopicPartition tp = null;
    try {
      throwIfProducerClosed();
      // first make sure the metadata for the topic is available
      long nowMs = time.milliseconds();
      ClusterAndWaitTime clusterAndWaitTime;
      try {
        //这个方法就是一直在等一个条件，这个条件达到了就返回，否则一直等待超时退出。而这个条件就是当前的版本号要大于上个版本号。
        clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);
      } catch (KafkaException e) {
        .....
      }
      nowMs += clusterAndWaitTime.waitedOnMetadataMs;
      long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
      Cluster cluster = clusterAndWaitTime.cluster;
      
      //序列化key、value
      byte[] serializedKey;
      serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
      byte[] serializedValue;
      serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
      int partition = partition(record, serializedKey, serializedValue, cluster);
      
      //分区
      tp = new TopicPartition(record.topic(), partition);
  
      setReadOnly(record.headers());
      Header[] headers = record.headers().toArray();
  
      int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                                                                         compressionType, serializedKey, serializedValue, headers);
      ensureValidRecordSize(serializedSize);
      long timestamp = record.timestamp() == null ? nowMs : record.timestamp();
      
      // callback
      Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);
      
      //result，添加到消息累加器
      RecordAccumulator.RecordAppendResult result = accumulator.append
        (tp, timestamp, serializedKey,serializedValue, headers, interceptCallback, remainingWaitMs, true, nowMs);
  
      //如果是新的批次
      if (result.abortForNewBatch) {
        int prevPartition = partition;
        partitioner.onNewBatch(record.topic(), cluster, prevPartition);
        partition = partition(record, serializedKey, serializedValue, cluster);
        tp = new TopicPartition(record.topic(), partition);
        
        interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);
        
        //result，添加到消息累加器
        result = accumulator.append(tp, timestamp, serializedKey,
                                    serializedValue, headers, interceptCallback, remainingWaitMs, false, nowMs);
      }
  
      。。。。。。
      return result.future;
  }
  ```

2、如果想使用同步方式，其实是**通过异步方式间接实现**，因为异步方式返回的是一个future对象，**在这对象上调用get方法**，将被阻塞直到返回结果。

- **Future**：表示一个任务的生命周期，并提供相应的方法判断任务是否已经完成或取消，以及获取任务的结果和取消任务等。

3、producer一般有两种类型异常：可重试异常、不可重试异常

- 可重试异常：网络异常、分区leader副本不可用
- 不可重试异常：消息太大

4、通常一个producer不会只负责发送单条消息，更多是发送多条消息，发送完这些消息后需要调用kafkaProducer的close方法回收资源

```c
public void close()
public void close(long timeout , TimeUnit timeUnit)
```

## 4.2 序列化

生产者需要使用序列化器（serializer）把对象转换成字节数组才能通过网络发送给kafka；消费者需要使用反序列化器（deserializer）把从kafka收到的字节数组转换成相应的对象

## 4.3 分区器

> 消息通过send方法发往broker过程中，有可能需要经过**拦截器（非必需）、序列化器、分区器**的一系列作用之后才能被真正发往broker

1、消息通过序列化之后需要确定它发往的分区，**如果消息ProducerRecord没有指定partition字段，那么就需要依赖分区器，根据key这个字段来计算partition的值**。

- 分区器的作用：为消息分配分区

2、默认分区器：`org.apache.kafka.clients.producer.internals.DefaultPartitioner`，实现了`org.apache.kafka.clients.producer.Partitioner`接口，定义了2个方法

```java
public int partition(String topic,Object key,byte[] keyBytes,Object value,byte[] valueBytes,Cluster cluster);
public void close();
```

- partition方法：计算分区号，返回int类型
- close方法：关闭分区器的时候回收一些资源

## 4.4 拦截器

1、两种拦截器

- 生产者拦截器：在消息发送前做一些准备工作、发送回调逻辑前做一些定制化需求
- 消费者拦截器

## 4.5 整体架构

<img src="https://img-blog.csdnimg.cn/afdbd91d7e194646a651da2cc6c56a14.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_16,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:80%;" />

1、整个生产者客户端由两个线程协调运行：**主线程、sender线程**

- 主线程：KafkaProducer创建消息，通过可能的拦截器、序列化器、分区器的作用后缓存到消息累加器
- sender线程：从消息累加器中获取消息发送到kafka中

**2、消息累加器**

- 缓存消息以便sender线程可以批量发送，减少网络传输的资源消耗
- 缓存大小可以通过生产者客户端参数buffer.memory配置，默认32MB
- 如果生产者发送速度超过发往服务器的速度，就会导致生产者空间不足，这个时候send方法要么阻塞要么异常
- 主线程发送的消息会被追加到消息累加器的某个双端队列（Deque）中，队列的内容就是ProducerBatch，包含多个ProducerRecord

**3、sender线程**

- 获取消息后，进一步将原来`<分区,Deque<ProducerBatch>>`转换为`<Node,List<ProducerBatch>>`的形式，其中node表示kafka的broker节点
- sender进一步封装成`<Node,Request>`的形式，发往各个node
- sender发送之前会保存到InFightRequest中，保持对象的形式是`Map<NodeId,Deque<Request>>`，主要是**缓存已经发出去但是还没有收到响应的请求**

![在这里插入图片描述](https://img-blog.csdnimg.cn/d72981631bb74f6abcd20a2b31414c2b.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16)

## 4.6 重要参数

**1、acks**

- 指定分区中必须有多少副本收到这条消息，之后生产者才认为消息是成功写入的
- 1：只要leader写入就会响应
- 0：不需要等待服务端响应
- -1或者all：ISR中副本都成功写入就会响应

**2、max.request.size**

- 限制生产者发送消息的最大值，默认1MB

**3、retries**

- 生产者重试次数，默认0

**4、linger.ms**

- 生产者发送ProducerBatch之前等待更多消息（ProducerRecord）加入的时间，默认0



## 4.7 生产者消息分区机制

### 4.7.1 为什么分区？

1、kafka的消息组织方式

- 主题：主题下每条消息只会保存在某一个分区（**同一topic消息不保证数据顺序性**）
- 分区
- 消息

<img src="https://img-blog.csdnimg.cn/2f3e26b9e3224fe98075ec7aff19277f.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:50%;" />

2、分区的作用：**负载均衡、实现系统的高伸缩性**

- 不同分区被放置到不同节点的机器上，数据的读写操作也是针对分区这个粒度进行的，每个节点的机器能够独立执行各自分区的读写请求

### 4.7.2 分区策略

> 分区策略：决定生产者把消息发送到哪个分区的算法

**1、自定义配置分区策略**

- 编写生产者程序时，编写具体的类实现`org.apache.kafka.clients.producer.Partitioner`接口
  - 两个方法：`partition()`和`close()`
  - topic、key、keyBytes、value和valueBytes都属于消息数据，cluster则是集群信息（比如当前 Kafka 集群共有多少主题、多少 Broker 等）

```java
int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);
```

**2、轮询策略**

- 比如3个分区，那么第一条消息发送到分区0，第二条到分区1，第三条道分区2，以此类推。
- **Kafka Java 生产者API的默认分区策略**
- **有非常优秀的负载均衡表现，最常用的策略之一**

<img src="https://img-blog.csdnimg.cn/e4fc46dc7c4b40c8a0f7f231f7854574.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:50%;" />

**3、随机策略**

- 随机把消息放置到任意一个分区

- 实现的partition方法：获取分区List，返回一个小于分区数量的随机数

  ```java
  List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
  return ThreadLocalRandom.current().nextInt(partitions.size());
  ```

- **把数据均匀打散，逊色于轮询策略**

**4、按消息键保序策略**

- 允许为每个消息定义消息key

- 保证一个key的消息放入一个分区，分区下的消息处理都是有序的

- 实现

  ```java
  List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
  return Math.abs(key.hashCode()) % partitions.size();
  ```

- **Kafka默认分区策略**

  - 如果指定key，默认按消息键保序策略
  - 不指定key，使用轮询策略

> 如何保证消息的顺序？
>
> - 设置单分区，保证全局的顺序性，但是丧失了多分区的高吞吐量和负载均衡优势
> - 按照消息的特点设置自定义分区策略，保证一个特点的消息发送到同一个分区，保证了消息的顺序

**5、其他分区策略**

- 按照地理位置分区
  - 针对跨城市、跨国家的集群

## 4.8 生产者压缩算法

**1、压缩（compression）**

- 时间换空间
- 用cpu时间换磁盘空间或网络I/O传输量

### 4.8.1 怎么压缩？

1、Kafka有两大消息格式：V1版本、V2版本（Kafka 0.11.0.0 中正式引入）

**2、Kafka消息层次**

- 消息集合（message set）
- 消息（message）
- **Kafka通常不会直接操作具体一条条消息，总是在消息集合这个层面进行写入操作**

**3、V1和V2版本区别**

- v1：message、message set
- v2：record、record batch
- v2把消息的公共部分抽取出来放到外层消息集合，这样就不用每条消息都保存这些信息了

> V1版本每条消息都需要执行CRC校验，但有些情况下消息的 CRC 值是会发生变化的，对每条消息都执行 CRC 校验就有点没必要了，**不仅浪费空间还耽误 CPU 时间**，因此在 V2 版本中，**消息的 CRC 校验工作就被移到了消息集合这一层**

- **保存压缩消息的方法发生了变化**

> V1 版本中保存压缩消息的方法是把多条消息进行压缩然后保存到外层消息的消息体字段中；而 V2 版本的做法是对整个消息集合进行压缩

4、对两个版本分别做了一个简单的测试，结果显示，在相同条件下，不论是否启用压缩，V2 版本都比 V1 版本节省磁盘空间。当启用压缩时，这种节省空间的效果更加明显。

### 4.8.2 什么时候压缩？

1、可能发生的两个地方：生产者、Broker

**2、生产者端压缩**

- 配置 compression.type 参数

  ```java
   Properties props = new Properties();
   props.put("bootstrap.servers", "localhost:9092");
   props.put("acks", "all");
   props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
   props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
   // 开启GZIP压缩
   props.put("compression.type", "gzip");
   
   Producer<String, String> producer = new KafkaProducer<>(props);
  ```

- 比较关键的代码行是 props.put(“compression.type”, “gzip”)，它表明该 Producer 的压缩算法使用的是 GZIP。**这样 Producer 启动后生产的每个消息集合都是经 GZIP 压缩过的**，故而能很好地节省网络传输带宽以及 Kafka Broker 端的磁盘占用。

**3、Broker端压缩**

- 出现的情况：
  - Broker与Producer使用的压缩算法不同，Broker会解压然后重新压缩;
  - Broker 端发生了消息格式转换（为了兼容老版本引入）

### 4.8.3 什么时候解压？

**1、Producer端压缩、Broker端保持、Consumer端解压**

- kafka将使用的消息压缩算法封装到消息集合中

**2、Broker端解压**

- 目的：对消息执行各种验证

### 4.8.4 压缩算法对比

1、2.1.0 版本之前，Kafka 支持 3 种压缩算法：**GZIP、Snappy 和 LZ4**。

2、从 2.1.0 开始，Kafka 正式支持 **Zstandard** 算法（简写为 zstd）。它是 Facebook 开源的一个压缩算法，能够提供超高的**压缩比**（compression ratio）。

### 4.8.5 最佳实践

1、启用压缩的比较合适的时机

- Producer程序运行机器上的cpu资源充足
- 带宽资源有限
- 如果客户端机器 CPU 资源有很多富余，强烈建议你开启 **zstd 压缩**，这样能极大地节省网络资源消耗

2、解压

- 对不可抗拒的解压缩无能为力，但至少能规避掉那些意料之外的解压缩（比如：**为了兼容老版本引入的解压操作**）

## 4.9 Java生产者管理TCP连接

### 4.9.1 为何采用TCP？

> Apache Kafka通信都是基于TCP的

1、原因

- 从社区角度来看，在开发客户端时，人们能够利用 TCP 本身提供的一些高级功能，比如**多路复用请求以及同时轮询多个连接的能力**。

**2、多路复用请求**

- 将多个数据流合并到底层单一物理连接
- TCP的多路复用请求会在一条物理连接上创建若干虚拟连接，每个虚拟连接负责流转对应的数据流

### 4.9.2 Kafka 生产者程序概览

Kafka的Java生产者API主要对象就是`KafkaProducer`，通常开发生产者的步骤有4步：

- 构造生产者对象所需参数
- 利用参数创建`KafkaProducer`对象实例
- 使用`KafkaProducer`的send方法发送消息
  - **同步生产者**：这个生产者写一条消息的时候，它就立马发送到某个分区去。follower还需要从leader拉取消息到本地，follower再向leader发送确认，leader再向客户端发送确认。由于这一套流程之后，客户端才能得到确认，所以很慢。
  - **异步生产者**：这个生产者写一条消息的时候，先是写到某个缓冲区，这个缓冲区里的数据还没写到broker集群里的某个分区的时候，它就返回到client去了。虽然效率快，但是不能保证消息一定被发送出去了。
- 调用`KafkaProducer`的close方法关闭生产者并释放资源

> 当我们开发一个 Producer 应用时，生产者会向 Kafka 集群中指定的主题（Topic）发送消息，这必然涉及与 Kafka Broker 创建 TCP 连接。那么，Kafka 的 Producer 客户端是如何管理这些 TCP 连接的呢？

### 4.9.3 何时创建TCP连接？

1、时间：**创建`KafkaProducer`对象实例的时候建立与Broker的TCP连接**

- **在创建 KafkaProducer 实例时，生产者应用会在后台创建并启动一个名为 Sender 的线程，该 Sender 线程开始运行时首先会创建与 Broker 的连接**
- 创建时会与`bootstrap.servers`（producer的一个参数）所有的broker建立连接
  - 通常指定3～4台即可，因为一旦连接到一台broker就可以拿到整个集群的broker消息（**发送METADATA请求，metadata消息**）

2、`KafkaProducer`类是**线程安全**的

- KafkaProducer 实例创建的线程和前面提到的 Sender 线程共享的可变数据结构只有 RecordAccumulator 类，故维护了 RecordAccumulator 类的线程安全，也就实现了 KafkaProducer 类的线程安全。
  - 主要的数据结构是一个 ConcurrentMap。TopicPartition 是 Kafka 用来表示主题分区的 Java 对象，本身是不可变对象。而 RecordAccumulator 代码中用到 Deque 的地方都有锁的保护，所以基本上可以认定 RecordAccumulator 类是线程安全的

**3、总结：TCP连接的创建时间**

- 创建`KafkaProducer`实例时
- 更新元数据后（可能创建）
  - 发现新的broker加入集群，**需要和没有创建连接的Broker建立TCP连接**
  - 场景：
    - 当producer尝试给一个不存在的topic发送信息，broker会告诉producer这个topic不存在，producer会发送**METADATA请求**给kafka集群获取最新的元数据信息
    - Producer 通过 metadata.max.age.ms 参数定期地去更新元数据信息（默认5分钟）
- 消息发送时（可能创建）
  - 发现broker还没有建立连接

### 4.9.4 何时关闭TCP连接？

1、关闭方式

- 用户主动关闭：`producer.close()`
- kafka自动关闭

2、Kafka自动关闭

- 与参数connections.max.idle.ms 的值有关（默认9分钟）
- 9分钟没有任何请求经过某个TCP连接，Kafka会主动关闭（Broker端发起），客户端会出现大量CLOSE_WAIT连接（客户端可能会hold住这个连接）

### 4.9.5 小结

1、KafkaProducer 实例创建时启动 Sender 线程，从而创建与 bootstrap.servers 中所有 Broker 的 TCP 连接。

2、KafkaProducer 实例首次更新元数据信息之后，还会再次创建与集群中所有 Broker 的 TCP 连接。

3、如果 Producer 端发送消息到某台 Broker 时发现没有与该 Broker 的 TCP 连接，那么也会立即创建连接。

4、如果设置 Producer 端 connections.max.idle.ms 参数大于 0，则步骤 1 中创建的 TCP 连接会被自动关闭；如果设置该参数 =-1，那么步骤 1 中创建的 TCP 连接将无法被关闭，从而成为“僵尸”连接。



## 4.10 幂等生产者和事务生产者

> Kafka消息交付可靠性保障以及精确处理一次语义的实现

**1、可靠性保障**：kafka对生产者和消费者需要处理的消息提供的承诺

- **最多一次：**消息可能会丢失，但是绝对不会重复
- **至少一次**：消息不会丢失，可能会重复（**Kafka默认**）
- **精确一次**：消息不会丢失，不会重复

**2、至少一次（at least once）**

- 只有生产者获得broker的应答才算发送成功，如果没有应答就会重试发送，可能会出现消息重复发送

**3、最多一次（at most once）**

- 禁止producer重试

**4、精确一次（exactly once）**

- Kafka通过两个机制来实现
  - **幂等性（Idempotence）**
  - **事务（Transaction）**

### 4.11.1 什么是幂等性？

1、概念

- 在命令式编程语言（比如 C）中，若一个子程序是幂等的，那它必然不能修改系统状态。这样不管运行这个子程序多少次，与该子程序关联的那部分系统状态保持不变。
- 在函数式编程语言（比如 Scala 或 Haskell）中，很多纯函数（pure function）天然就是幂等的，它们不执行任何的 side effect

2、好处

- **可以安全的重试，反正也不会破坏系统状态**

### 4.11.2 幂等性 Producer

1、在Kafka 中，Producer 默认不是幂等性的

**2、创建幂等性 Producer**

- `props.put(“enable.idempotence”, ture)`
- Kafka自动帮你做消息的重复去重

**3、底层原理**

- 空间换时间：在Broker多保存一些字段：`ProducerID`、`SequenceNumber`
  - `ProducerID`：每个producer初始化时会被分配一个唯一的id，对客户端不可见
  - `SequenceNumber`：producer发送的每个topic和partition都对应一个从0开始递增的值

4、作用范围

- 只能保证某个topic的一个partition不出现重复消息，无法实现多个分区的幂等
- 只能实现单会话的幂等

### 4.12.3 事务型 Producer

**1、事务**

- Kafka 自 0.11 版本开始也提供了对事务的支持，目前主要是在 **read committed 隔离级别**上做事情。**它能保证多条消息原子性地写入到目标分区，同时也能保证 Consumer 只能看到事务成功提交的消息**。

2、设置方法

- 开启 `enable.idempotence = true`。

- 设置 Producer 端参数 `transactional.id`，**可以穿越多次对话**

- **实现机制：两阶段提交（2PC）、事务协调器**

- 代码调整

  ```java
  //初始化事务
  producer.initTransactions();
  try {
    //开启事务
    producer.beginTransaction();
    producer.send(record1);
    producer.send(record2);
    //提交事务
    producer.commitTransaction();
  } catch (KafkaException e) {
    //终止事务
    producer.abortTransaction();
  }
  ```

- 上述代码，Record1和Record2被当作一个事务提交，同时写入成功或失败，但是kafka还是会写入底层日志，也就是说消费者还是可以看见这些消息，因此需要设置消费者读取消息的参数：

  - `read_uncommitted`：这是默认值，表明 Consumer 能够读取到 Kafka 写入的任何消息，不论事务型 Producer 提交事务还是终止事务，其写入的消息都可以读取。很显然，如果你用了事务型 Producer，那么对应的 Consumer 就不要使用这个值。
  - `read_committed`：表明 Consumer 只会读取事务型 Producer 成功提交事务写入的消息。当然了，它也能看到非事务型 Producer 写入的所有消息。

# 5 Kafka副本机制

1、副本机制的概念

- 分布式系统在多台网络互联的机器上保存有相同的数据拷贝

2、副本机制好处

- **提供数据冗余**
- **提供高伸缩性**：支持横向扩展，能够通过增加机器的方式来提升读性能
- **改善数据局部性**：允许将数据放入与用户地理位置相近的地方

> Kafka的副本机制**只提供了数据冗余**

## 5.1副本定义

1、副本的概念实际上是在分区层级下定义的，每个分区有若干副本

**2、本质：只能追加写消息的提交日志**

- 同一个分区下的所有副本保存相同的消息序列，分散在不同的broker上

3、相关概念

- **AR**：分区中所有副本
- **ISR**：与leader保持同步的副本集合
- **LEO**：标识每个分区最后一条消息的下一个位置，每个副本都有自己的LEO
- **HW**：ISR最小的LEO，消费者只能拉取到HW之前的消息

## 5.2 副本角色

> 如何保证副本中所有消息都一致？
>
> 解决：**基于领导者的副本机制**

1、工作原理

<img src="https://img-blog.csdnimg.cn/bdee31404be1498fb034ac4291f608fa.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:50%;" />

- **副本分为两类**：领导者副本（每个分区创建时选举）、追随者副本
- **追随者副本不对外提供服务**，**唯一的任务**：从领导者副本异步拉取消息，写入自己的提交日志
- 当领导者副本宕机时，zookeeper提供的监控功能能够实时感知到，并立即开始新一轮的领导者选举

**2、为什么要这么设计？**

- **方便实现“Read-your-writes”**
  - Read-your-writes：当使用生产者API向kafka成功写入消息后，马上使用消费者API去读取生产的消息

- **方便实现单调读**
  - 单调读：对于一个消费者用户在多次消费消息时不会看到某条消息一会村庄一会不存在（一致性）

## **5.3 In-sync Replicas（ISR）**

> 追随者副本异步拉取领导者副本的消息，**异步就会存在不可能与Leader实时同步的风险**

**1、ISR副本集合**

- ISR副本都是和Leader同步的副本，不在ISR的副本就是不同步的
- Leader副本必然在ISR中
- 动态调整

**2、判断是否同步的标准**

- Broker参数：`replica.lag.time.max.ms`，表示追随者副本能够落后领导者副本的最长时间间隔，默认10s；**只要不连续超过10s就认为是同步的**

## 5.4 Unclean 领导者选举

> ISR是动态调整的，自然会出现ISR为空的情况，即leader副本挂了，**kafka需要选举一个新的leader**，但是ISR为空，如何选举呢？

1、Kafka把所有不在ISR的存活副本称为非同步副本，在kafka中选举这种副本的过程称为**Unclean领导者选举**

2、设置：Broker参数`unclean.leader.election.enable`（建议不开启）

3、可能会造成数据丢失，但是至少不至于停止对外服务，**因此提高了高可用性**

4、CAP理论

- C：一致性
- A：可用性
- P：分区容错性



# 6 Controller

1、Controller是kafka最核心的组件

- 为集群所有主题分区选举领导者副本
- 承载集群的全部元数据信息，并负责将这些信息同步到其他broker上

**2、Controller启动**：kafka集群中**每个broker启动时**会实例化一个kafkaController类，该类会执行一系列业务逻辑，选举出主题分区的Leader节点，步骤如下：

- 第一个启动的代理节点，会在Zookeeper系统里面创建一个临时节点/controller，并写入该节点的注册信息，使该节点成为控制器；
- 其他的代理节点陆续启动时，也会尝试在Zookeeper系统中创建/controller节点，但是由于/controller节点已经存在，所以会抛出“创建/controller节点失败异常”的信息。创建失败的代理节点会根据返回的结果，判断出在Kafka集群中已经有一个控制器被成功创建了，所以放弃创建/controller节点，这样就确保了**Kafka集群控制器的唯一性**；
- 其他的代理节点，会在控制器上注册相应的监听器，各个监听器负责监听各自代理节点的状态变化。当监听到节点状态发生变化时，会触发相应的监听函数进行处理。

**3、Kafka Contoller本质上就是一个broker**

4、Broker选举的过程是在zk创建/controller临时节点，每个broker会对/controller节点添加监听器，以此坚挺此节点的数据变化，当/controller节点发生变化就会触发选举

**5、分区Leader选举**

- 基本策略：按照AR集合（全部副本）中副本的顺序查找第一个存活的副本，并且这个副本在ISR集合中
- AR副本顺序不变，ISR副本顺序可能会改变



# 7 消费者

```java
public class testConsumer {
  public static final String brokerList = "localhost:9092";
  public static final String topic = "test";
  public static final String groupid = "group.demo";
  public static final AtomicBoolean isRunning =new AtomicBoolean(true);

  public static Properties initConfig() {
    Properties props = new Properties();
    props.put("key.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
    props.put("value.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
    props.put("bootstrap.servers", brokerList);
    //消费组名称
    props.put("group.id", groupid);
    //对应客户端id
    props.put("client.id","consumer.client.id.demo");
    return props;
  }

  public static void main(String[] args) {
    Properties props = initConfig();
    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
    consumer.subscribe(Arrays.asList(topic));
    try{
      while (isRunning.get()){
        ConsumerRecords<String ,String> records = consumer.poll(Duration.ofMillis(1000));
        for(ConsumerRecord<String,String> record : records){
          System.out.println(record.value());
        }
      }
    }catch (Exception e){
      e.printStackTrace();
    }finally {
      consumer.close();
    }
  }
}
```

## 7.1 消费者组

**1、什么是消费者组？**

- Kafka提供的可扩展且具有容错性的消费者机制
- 组内多个消费者共享一个Group ID
- **消费者组内所有消费者消费topic的所有partition，但是每个partition只能被一个消费者消费**
- **同一个分区也可以被其他消费者组消费**

**2、消息引擎模型：P2P、发布/订阅模型**

- P2P：伸缩性差，下游多个消费者需要抢消息
- 发布/订阅模型：伸缩性不够，每个订阅者需要订阅所有分区
- **消费者组解决了这个缺陷**
  - 如果所有实例都是一个group（**每个消息只被组内的一个消费者消费**），那么实现的就是消息队列模型
  - 如果实例不是一个group（**不同组的消费者可以同时消费一个消息**），实现发布/订阅模型

**3、group下消费者的数量**

- 理想情况下，消费者数量和topic的partition数量一致

**4、针对消费组，kafka如何管理offset？**

- 消费者在消费过程中需要记录自己消费了多少数据，即消费位置信息
- **对于消费者组，offset是一组KV（key是分区，V是该分区的最新位移）**，但是kafka源码的数据结构比这个复杂的多

- 对于老版本，位移保存在Zookeeper中，减少了broker端的状态保存开销
- **在新版本中，位移保存在kafka内部topic中**（__consumer_offsets）

**5、重平衡（Rebalance）**

- 本质上是一种协议，规定了一个group下所有消费者如何达成一致，来分配订阅topic的每个分区；
  - 比如一个有20个consumer的group订阅了一个100个partition的topic，正常情况下kafka会为每个consumer分配5个partition，这个分配过程为重平衡
- **触发条件**
  - conusmer实例数量发生变化
  - 订阅的topic数量发生变化
  - 订阅的topic的partition数量发生变化
- **分配策略（consumer如何知道应该消费哪些分区）**
  - **Range分配策略**：对同一个topic的partition按照序号进行排序，按照消费者线程的字母顺序进行排序，然后用分区数量除以消费者线程数量，剩下的前面几个消费者多消费一个分区
  - **RoundRobin分配策略**：将消费者和partition按照字典序排序，然后通过轮询算法诸葛分配给每个消费者
  - **Sticky分配策略**

- **缺点**
  - 重平衡的时候所有消费者实例会停止消费
  - 目前的设计所有的消费者都需要参与重平衡，不够高效
  - 速度慢

## 7.2 订阅主题和分区

1、方法：`subscribe()`，可以订阅多个主题

```java
public void subscribe(Collect on<String> toplics, ConsumerRebalanceListener listener) 
//集合
public void subscribe(Collection<String> topics) 
  
public void subscribe (Pattern pattern, ConsumerRebalanceListener listener) 
//正则表达式
public void subscribe (Pattern pattern)
```

2、方法：`assign()`

```java
public void assign(Collection<TopicPartition> partition)
```

- 指定需要订阅的分区集合
- ToplicPartition：partition、topic属性

3、区别

- `subscribe()`方法支持消费者重平衡

## 7.3 消息消费

1、kafka消息消费是一个不断轮询的过程，消费者需要重复调用`poll()`方法，返回的是订阅的一组消息

```java
public ConsumerRecords<K, V> poll(final Duration timeout)
```

- timeout：控制方法的阻塞时间，在没有可用数据的时候会发生阻塞，**设为0会立刻返回，不管是否有消息**

**2、ConsumerRecord**

```java
public class ConsumerRecord<K, V> { 
  //主题名称
  private final String topic; 
  //分区编号
  private final int partition; 
  //消息偏移量
  private final long offset; 
  private final long timestamp; 
  private final TimestampType timestampType; 
  private final int serial zedKeySize;
  private final int serializedValueSize; 
  //消息头部内容
  private final Headers headers; 
  //消息的键
  private final K key ; 
  //消息的值
  private final V value ; 
  private volatile Long checksum; 
  //省略若干方法
}
```

**3、ConsumerRecords**

- poll方法返回的是`ConsumerRecords`，表示一次拉取操作获得的消息集，内部包含若干`ConsumerRecord`，提供了一个`iterator()`方法遍历消息内部的消息
- 可以按照分区维度获取消息，提供了`records(TopicPartition)`方法获取消息中指定分区的消息，`partitions()`获取消息集中所有分区

## 7.4 位移主题

1、主题名称：`__consumer_offsets`

**2、出现背景**

- 老版本的位移存储在zk中，但是zk不适合高频写的操作，因此新版本中使用位移主题管理位移

**3、位移管理机制**

- 将consumer的位移数据作为一条普通的kafka消息提交到`__consumer_offsets`中
- 这就是个普通的kafka主题，**但是消息格式是kafka自定义的**，用户不能修改，也就是说不能往这个主题随意写消息，如果写的话broker会出现崩溃（消息格式不一致）

**4、消息格式**

- **格式1：存储格式可以简单理解为KV对**
  - key保存的内容：**group id、topic、partition号**
  - value保存的内容：位移值、时间戳等元数据
- **格式2：保存消费者组信息的消息，注册group**
- **格式3：tombstone 消息（墓碑消息）**
  - 用于删除group过期位移、删除group的消息
  - 一旦group下的所有consumer都停止了，并且他们的位移数据都被删除时会写入这个消息，表示彻底删除group

**5、什么时候创建？**

- Kafka集群第一个consumer程序启动时自动创建
- 分区数：50；副本数：3

**6、consumer如何提交位移？**

- **自动提交**：只要consumer一直启动就会一直写入消息，无消费的时候也会一直写入这个位移
  - 默认每隔5s拉取每个分区最大的消息位移进行提交
  - 问题：延时提交、重复提交
- **手动提交**：调用API
  - 异步提交、同步提交

**7、删除过期消息**

- 方法：整理（compaction）
- 使用Compact策略删除位移主题的过期消息，**避免该主题无限膨胀**
- **如何定义过期？**
  - 同一个key的两条消息M1和M2，M1发送时间早，那么就是过期消息
- 过程
  - 扫描日志所有信息，剔除过期消息，整理剩下的消息
- **定期巡检等待整理主题的线程：Log Cleaner**

## 7.5 重平衡

1、新版的消费者客户端对此进行了重新设计，将全部消费组分成多个子集，每个消费组的子集在服务端对应一个 `GroupCoordinator `进行管理， `GroupCoordinator`是Kafka 服务端中用于管理消费组的组件。而消费者客户端中的 `ConsumerCoordinator `组件负责与 `GroupCoordinator`交互

- `ConsumerCoordinator`和`GroupCoordinator`之间最重要的职责就是负责执行消费者重平衡的操作，包括分区分配工作也是在重平衡期间完成

**2、触发重平衡的操作**

- 有新的消费者加入消费者组
- 消费者下线，不一定真的下线，可能是长时间GC、网络延迟导致消费者长时间没有向GroupCorrdinator发送心跳
- 消费者退出（`leaveGroupRequest`请求）
- 消费者的`GroupCoorinator`节点发生变化
- 消费者组订阅的任一topic或者partition数量发生变化

> 当有消费者加入消费者组时，消费者、消费组及协调器会经历以下阶段

**3、第一阶段（find_coordinator）**

- 目的：**消费者需要确定所属消费组对应的groupCoordinator所在的broker，并与该broker创建相互通信的网络连接**
- 步骤：
  - 如果消费者已经保存了对应协调器节点的信息就进入第二阶段
  - 向**集群负载最小的节点**发送`FindCoordinatorRequest`请求来查找对应的`GroupCoordinator`
    - `FindCoordinatorRequest`请求体：消费组名称（`group id`）
  - kafka收到`FindCoordinatorRequest`请求后**根据groupid查找对应的`GroupCoordinator`节点**，如果找到对应的就会返回其对应的node_id、host、port信息
    - 查找方法：
      - 根据`groupid`的哈希值计算`__consumer_offsets`中分区的编号
      - 寻找此分区leader副本所在的broker节点，该节点就是为这个groupid所对应的`GroupCoordinator`节点
  - **消费者组groupid最终的分区方案及组内消费者所提交的消费位移信息都会发送给此分区leader副本所在的broker节点**

**4、第二阶段（join_group）**

- 目的：**加入消费组**
- 步骤：
  - 向`groupCoordinator`发送`JoinGroupRequest`请求
    - 参数：
      - `groupId`
      - `session_timeout`（心跳时间）
      - `rebalance_timeout`（重平衡的时候GroupCoordinator等待各个消费者重新加入的最长等待时机）
      - `member_id`（消费者id，第一次发送时为null）
  - 服务端接收到请求后交给`GroupCoordinator`处理；对请求进行合法性校验，给第一次请求加入的消费者生成member_id
  - **选举消费组的leader**
    - 如果还没有leader，第一个加入消费组的消费者为leader
    - 如果leader退出了消费组，会进行重新选举，过程很随意
      - 在GroupCoordinator中消费者信息以HashMap存储，key为消费者id，value为元数据
      - 选举时取hashMap第一个键值对
  - **选举分区分配策略**
    - 每个消费者都可以设置自己的分区分配策略，消费组的策略需要进行选举投票，过程如下：
      - 收集各个消费者支持的所有分配策略，组成候选集candidates
      - 每个消费者从候选集中找出第一个自身支持的策略，投上一票
      - 得分多的策略为消费组的策略

**5、第三阶段（sync_group）**

- 目的：转发同步分区分配方案；各个消费者给GroupCoordinator发送SyncGroupRequest请求来同步分配方案

**6、第四阶段（heartbeat）**

- 目的：消费者确定拉取消息的其实位置
- 消费者通过向GroupCoordinator发送心跳来维持他们与消费组的从属关系，心跳线程是一个独立的线程；如果消费者停止发送心跳的时间足够长整个会话就会判定过期

## 7.6 能够避免重平衡吗？

1、在Rebalance 过程中，所有 Consumer 实例共同参与，在**协调者组件**的帮助下，完成订阅主题分区的分配，**整个过程中所有实例都不能消费任何消息**，因此它对 Consumer 的 TPS 影响很大。

**2、协调者（Coordinator）**

- 专门为group服务，负责group执行重平衡以及提供位移管理等
- consumer提交位移时是向coordinator所在的broker提交位移
- 当 Consumer 应用启动时，也是向 Coordinator 所在的 Broker 发送各种请求，然后由 Coordinator 负责执行消费者组的注册、成员管理记录等元数据管理操作
- **所有的broker都有自己的coordinator组件**
- **group如何确定为它服务的coordinator在哪个broker上？**
  - 确定位移主题__consumer_offsets的哪个分区保存该group的数据
  - 找到该分区leader副本所在的broker，该broker即对应的coordinator

3、目前，无法解决重平衡慢的问题，所以选择**避免重平衡**，重平衡发生的时机有三个：

- group成员数量变化
  - 增加：通常都是计划内的，不属于需要避免的不必要重平衡
  - 减少：除了计划内的，**某些情况下协调者会错误认为消费者已经停止**
    - consumer需要定期给coordinator发送心跳请求，如果不能及时发送就会被误认为这个consumer挂了，默认10s
    - consumer消费时间过长（默认5分钟），可以预留足够的时间避免这个情况
- topic数量变化
- partition数量变化

## 7.7 多线程开发消费者实例

### 7.7.1 Kafka Java Consumer设计原理

**1、单线程设计，从0.10.1.0版本开始变为双线程（用户主线程+心跳线程）**

- 用户主线程：启动Consumer应用程序main方法的那个线程
- 心跳线程：负责定期给broker机器发送心跳请求

2、单线程的设计容易实现

### 7.7.2 多线程方案

**1、KafkaConsumer类不是线程安全的**

- 所有的I/O处理都发生在用户主线程中，因此你需要确保线程安全

2、**方案一**：消费者启动多个线程，每个线程维护专属的KafkaConsumer实例，负责完整的消息获取、处理

3、**方案二**：消费者使用单或多线程获取消息，同时创建多个消费线程执行消费处理逻辑

<img src="https://img-blog.csdnimg.cn/26c1e20bb8014deeae4e4c244258cf9d.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:50%;" />

## 7.8 消费者管理TCP连接

### 7.8.1 什么时候创建TCP连接？

1、消费者的入口是KafkaConsumer类，**和生产者不同，创建实例时没有创建TCP连接**

2、**TCP连接在调用`KafkaConsumer.poll`方法时被创建**

3、在poll方法内部有3个时机可以创建TCP连接

- **发起`FindCoordinator`请求时**
  - 当消费者程序首次启用调用poll方法时，需要向kafka集群发送`FindCoordinator`请求，希望kafka告诉他哪个broker是管理它的协调者
- **连接协调者时**
  - broker处理完上一步发送的请求后会返回response，告诉消费者哪个是协调者，因此在这一步消费者会创建连向该broker的TCP连接
- **消费数据时**
  - 消费者会为每个要消费的分区创建与该分区leader副本所在broker连接的TCP

### 7.8.2 创建多少个TCP连接？

1、通常来说，消费者程序会创建 3 类 TCP 连接：

- 确定协调者和获取集群元数据。
- 连接协调者，令其执行组成员管理操作。
- 执行实际的消息获取。

### 7.8.3 删除TCP连接

和生产者类似，消费者关闭 Socket 也分为**主动关闭和 Kafka 自动关闭**

- **主动关闭**：是指你显式地调用消费者 API 的方法去关闭消费者，具体方式就是手动调用 `KafkaConsumer.close() `方法，或者是执行 Kill 命令，不论是 Kill -2 还是 Kill -9；
- **自动关闭**：是由消费者端参数 `connection.max.idle.ms` 控制的，该参数现在的默认值是 9 分钟，即如果某个 Socket 连接上连续 9 分钟都没有任何请求“过境”的话，那么消费者会强行“杀掉”这个 Socket 连接。
  - 和生产者有些不同的是，如果使用了循环的方式来调用 poll 方法消费消息，那么上面提到的所有请求都会被定期发送到 Broker，因此这些 Socket 连接上总是能保证有请求在发送，从而也就实现了“长连接”的效果。

针对上面提到的三类 TCP 连接，你需要注意的是，当第三类 TCP 连接成功创建后，消费者程序就会废弃第一类 TCP 连接，之后在定期请求元数据时，它会改为使用第三类 TCP 连接。也就是说，最终你会发现，第一类 TCP 连接会在后台被默默地关闭掉。对一个运行了一段时间的消费者程序来说，只会有后面两类 TCP 连接存在。

## 7.9 重要参数

**1、fetch.min.bytes**

- 拉取的最小数据量，默认1B
- 如果小于这个值就需要进行等待

**2、fetch.max.bytes**

- 拉取的最大数据量，默认50MB

**3、fetch.max.wait.ms**

- 指定kafka等待的时间，默认500ms

**4、max.poll.records**

- 一次拉取的最大消息数量，默认500条

**5、heartbeat.interval.ms**

- 使用kafka分组管理时，心跳到消费者协调器之间的预计时间
- 默认3000ms

**6、connections.max.idle.ms**

- 指定多久之后关闭限制的连接，默认9分钟

# 8 实现无消息丢失配置

>  kafka在什么情况下能保证消息不丢失？

**只对已提交的消息做有限度的持久化保证**

- **已提交**：kafka集群收到消息并写入日志
- **有限度**：在保证存放消息的broker至少有一个存活的前提下保证消息成功持久化

## 8.1 “消息丢失”案例

#### 生产者程序丢失数据

> 目前**Kafka Producer是异步发送消息**的，调用的是`producer.send(msg)`这个API，通常会立刻返回，但是此时不能认为消息成功发送

**导致发送失败的因素：**

- 网络抖动，消息未到broker
- 消息不合格，broker拒绝接受（比如消息过大等）

**解决方案**

- Producer永远要使用带回调通知的发送API，使用`producer.send(msg,callback)`这个API，`callback`可以准备告诉你消息是否提交成功

#### 消费者程序丢失数据

> Conusmer端要消费的消息不见了

Consumer 程序有个“**位移（offset）**”的概念，表示的是这个 Consumer 当前消费到的 Topic 分区的位置

<img src="https://img-blog.csdnimg.cn/c421cd69e6a04ed4afc2ab026fe0de18.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:40%;" />

- 如上图，对于Consumer A 而言，它当前的位移值就是 9；Consumer B 的位移值是 11

**1、使用位移的步骤（顺序不能颠倒）**

- 读取数据
- 更新位移
- **顺序颠倒会导致消息丢失**
- 问题：**消息重复处理**

**2、消费者程序丢失消息的原因**

- 先更新位移后读取数据
- 多线程异步处理消息，如果某个线程运行失败了，位移已经被更新了
  - 解决：手动提交位移

## 8.2 最佳实践

1、不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)。记住，**一定要使用带有回调通知的 send 方法**。

2、设置 `acks = all`。acks 是 Producer 的一个参数，代表了你对“已提交”消息的定义。如果设置成 all，则表明所有副本 Broker 都要接收到消息，该消息才算是“已提交”。这是最高等级的“已提交”定义。

3、设置 retries 为一个较大的值。这里的 retries 同样是 Producer 的参数，对应前面提到的 Producer 自动重试。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了 retries > 0 的 Producer 能够**自动重试消息发送**，避免消息丢失。

4、设置 replication.factor >= 3。这也是 Broker 端的参数。其实这里想表述的是，**最好将消息多保存几份**，毕竟目前防止消息丢失的主要机制就是冗余。

5、设置 min.insync.replicas > 1。这依然是 Broker 端参数，**控制的是消息至少要被写入到多少个副本才算是“已提交”**。设置成大于 1 可以提升消息持久性。在实际环境中千万不要使用默认值 1。

6、确保 replication.factor > min.insync.replicas。如果两者相等，那么只要有一个副本挂机，整个分区就无法正常工作了。我们不仅要改善消息的持久性，防止数据丢失，还要在不降低可用性的基础上完成。推荐设置成 replication.factor = min.insync.replicas + 1。

7、确保消息消费完成再提交。Consumer 端有个参数 enable.auto.commit，最好把它设置成 false，并采用手动提交位移的方式。就像前面说的，这对于单 Consumer 多线程处理的场景而言是至关重要的。

<img src="https://img-blog.csdnimg.cn/e9ef26374b784fe5a47b6b33b493b952.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:40%;" />











