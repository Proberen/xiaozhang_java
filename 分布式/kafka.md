# 1 引入

1、Kafka是一个多分区、多副本且基于 ZooKeeper 协调的分布式消息系统，是一个分布式流式处理平台，以高吞吐、可持久化、可水平扩展、支持数据处理等多种特性被广泛使用

2、三大角色：

- **消息系统：**系统解耦、冗余存储、流量削峰、缓冲、异步通信、扩展性、可恢复性等功能，还提供了消息顺序性保障和回溯消费功能
- **存储系统：**持久化功能和多副本机制
- **流式处理平台：**提供一个完整的流式处理类库，比如窗口、连接、变换、聚合等操作

## 1.1 基本概念

Kafka体系结构：

- 若干Producer：将数据发送到Broker
- 若干Broker：将收到的消息存储到磁盘中
- 若干Conusmer：从Broker订阅并消费信息
- 一个Zookeeper集群：负责集群元数据的管理、控制器的选举等操作

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210708151913315.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

























































































