#  Redis

## 问题

### 如何保证mysql与缓存一致性？

#### 读取缓存

读取缓存的方案都是按照下面的流程进行操作的：

<img src="https://img-blog.csdnimg.cn/58f44406193a455087c25bdc5454e002.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:40%;" />





#### 更新一致性方案——设置缓存过期时间

所有的写操作都以数据库为准，如果数据库写入成功但是缓存更新失败，只要缓存到期时间之后后面读缓存时自然会去数据库读取新的缓存然后更新

#### 更新一致性方案——先更新数据库再更新缓存

> 被普遍反对！

原因：

- **数据安全角度**：如果请求A和请求B同时进行操作，A先更新了数据库的一条数据，随后B马上有更新了该条数据，但是可能因为网络延迟等原因，B却比A先更新了缓存，就会出现一种什么情况呢？缓存中的数据并不最新的B更新过的数据，就导致了数据不一致的情况。
- **业务场景角度**：如果是写多读少的业务，这种方案会导致数据还没读缓存就被频繁更新，浪费性能

#### 更新一致性方案——先删除缓存再更新数据库

**1、问题：脏数据**

- A进行更新操作，B进行查询操作；A进行写操作前删除了缓存，B读的时候发现没有缓存就会查询数据库，如果A事务没有提交B就会查询到旧值存储到缓存中，就会导致数据不一致问题

**2、解决：延时双删**

<img src="https://img-blog.csdnimg.cn/b85b7ad8935c4b4d83df7fd62c356ad6.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:40%;" />

过程：

- 删除Redis缓存
- 更新数据库
- 等待一段时间，删除Redis缓存（**等待时间：读数据业务逻辑耗时+几百ms**）

**目的：确保B读请求结束A写请求能够删除B读请求存储的脏数据**

**3、问题：**第二次删除缓存失败，还是会造成缓存和数据库的不一致

#### 更新一致性方案——先更新数据库再删除缓存

缓存更新方案：Cache-Aside Pattern

- 应用程序应该从缓存中获取数据，获取成功就直接返回，获取失败就从数据库中读取，成功后放入缓存
- 更新数据时先把数据库存储到数据库中成功后再让缓存失效

**问题：**

有两个请求A和B，A进行查询同时B进行更新，假设发生下述情况：

①此时缓存刚好失效

②请求A 就会去查询数据库得到一个旧的值

③请求B将新的值写入数据库

④请求B写入成功后删除缓存

⑤请求A将查到的机制写入缓存，产生脏数据...

> 但是这种情况的概率很小

#### 如果删除缓存失败怎么办？

解决：重试机制

<img src="https://img-blog.csdnimg.cn/f6abfeb75421436fa93f47935b2154d6.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:40%;" />

解决：读取binlog异步删除缓存

### 单线程为什么快？

**1、为什么不使用多线程？**

- 减少多线程带来的额外开销和多线程同时访问共享资源的并发问题

**2、为什么单线程快？**

- Redis大部分操作在内存实现
- 采用高效的数据结构
- 多路I/O复用，能够并发处理大量客户端请求

**3、多路I/O复用**

- 一个线程处理多个IO流，**select/epoll机制**
- 该机制允许内核中同时存在多个监听socket和已连接socket；内核会一直监听这些socket上的连接请求或数据请求；一旦有请求到达就交给redis线程处理
- select/epoll 提供了基于事件的**回调机制**，即针对不同事件的发生，调用相应的处理函数。
  - **回调机制**：select/epoll 一旦监测到 FD 上有请求到达时，就会触发相应的事件；这些事件会被放进一个事件队列，redis对事件队列不断处理，Redis处理的时候会调用相应的处理函数，这就实现了基于事件的回调

<img src="https://img-blog.csdnimg.cn/d52e5e9dbd494cb19f6f09039f8aebeb.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

### select、poll、epoll

#### 基本的socket编程模型

1、服务端和客户端进行通信时，在服务器端通过以下三步来创建监听客户端连接的**监听套接字（listening socket）**

- 调用`socket`函数，创建一个套接字（**主动套接字**）
- 调用`bind`函数，将主动套接字和当前服务器的IP和监听端口进行绑定
- 调用`listen`函数，将主动套接字转为监听套接字，开始监听客户端的连接

在完成上述三步之后，服务器端就可以接收客户端的连接请求了，可以运行一个循环流程，在流程中调用`accept`函数，用于接收客户端连接请求

- `accept`函数是阻塞函数，如果此时一直没有客户端连接请求，那么服务器端执行流程会一直阻塞在accept函数

最后，服务器端可以通过调用`recv`或者`send`函数在刚刚返回的已连接套接字上，接收并处理读写请求，或者将数据发送给客户端

```c
listenSocket = socket(); //调用socket系统调用创建一个主动套接字
bind(listenSocket);  //绑定地址和端口
listen(listenSocket); //将默认的主动套接字转换为服务器使用的被动套接字，也就是监听套接字
while (1) { //循环监听是否有客户端连接请求到来
   connSocket = accept(listenSocket); //接受客户端连接
   recv(connsocket); //从客户端读取数据，只能同时处理一个客户端
   send(connsocket); //给客户端返回数据，只能同时处理一个客户端
}
```

2、在基本的 Socket 编程模型中，accept 函数只能在一个监听套接字上监听客户端的连接，recv 函数也只能在一个已连接套接字上，等待客户端发送的请求。

**3、IO多路复用机制**

- 可以让程序通过调用多路复用函数，同时监听多个套接字上的请求，既包括监听套接字上的请求也包括已连接套接字的请求

#### select和poll机制实现IO多路复用

**1、select机制**

- 参数：
  - `__nfds`：监听的文件描述符数量
  - `*__readfds、*__writefds、*__exceptfds`：被监听描述符的三个集合（fd_set结构）
  - `*__timeout`：监听时阻塞等待的超时时长
- 监听的事件：**读数据事件、写数据事件、异常事件**
- **对于每一个描述符集合，都可以监听1024个描述符**
- 如何使用select机制实现网络通信
  - 调用前，创建好传递给select函数的描述符集合，再创建监听套接字，将套接字的描述符加入到创建好的描述符集合中
  - 调用select函数，把创建好的描述符集合作为参数传递给select函数；程序调用后会发生阻塞，当select函数检测到有描述符就绪后，就会结束阻塞，并返回就绪的文件描述符数量
  - 在描述符集合中查找哪些描述符就绪了，对这些描述符对应的套接字进行处理，在该套接字上进行读写操作

<img src="https://img-blog.csdnimg.cn/dc90ad8c8c8a407bb14fe43a22d13335.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:40%;" />

```c
int sock_fd,conn_fd; //监听套接字和已连接套接字的变量
sock_fd = socket() //创建套接字
bind(sock_fd)   //绑定套接字
listen(sock_fd) //在套接字上进行监听，将套接字转为监听套接字

fd_set rset;  //被监听的描述符集合，关注描述符上的读事件
 
int max_fd = sock_fd

//初始化rset数组，使用FD_ZERO宏设置每个元素为0 
FD_ZERO(&rset);
//使用FD_SET宏设置rset数组中位置为sock_fd的文件描述符为1，表示需要监听该文件描述符
FD_SET(sock_fd,&rset);

//设置超时时间 
struct timeval timeout;
timeout.tv_sec = 3;
timeout.tv_usec = 0;
 
while(1) {
   //调用select函数，检测rset数组保存的文件描述符是否已有读事件就绪，返回就绪的文件描述符个数
   n = select(max_fd+1, &rset, NULL, NULL, &timeout);
 
   //调用FD_ISSET宏，在rset数组中检测sock_fd对应的文件描述符是否就绪
   if (FD_ISSET(sock_fd, &rset)) {
       //如果sock_fd已经就绪，表明已有客户端连接；调用accept函数建立连接
       conn_fd = accept();
       //设置rset数组中位置为conn_fd的文件描述符为1，表示需要监听该文件描述符
       FD_SET(conn_fd, &rset);
   }

   //依次检查已连接套接字的文件描述符
   for (i = 0; i < maxfd; i++) {
        //调用FD_ISSET宏，在rset数组中检测文件描述符是否就绪
       if (FD_ISSET(i, &rset)) {
         //有数据可读，进行读数据处理
       }
   }
}
```

- 缺点：
  - 对单个进程监听的文件描述符数量有限
  - 当select函数返回后需要遍历描述符集合找到具体就绪的描述符

**2、poll机制**

- 参数：

  - `*__fds`：pollfd 结构体数组，包含要监听的描述符，以及描述符上要监听的事件类型

    ```c
    struct pollfd {
        int fd;         //进行监听的文件描述符
        short int events;       //要监听的事件类型
        short int revents;      //实际发生的事件类型
    };
    ```

  - `_nfds`：表示*__fds数据元素的个数

  - `_timeout`：表示poll函数阻塞的超时事件

- 通信流程

  - 创建pollfd数组和监听套接字，进行绑定
  - 将监听套接字加入pollfd数组，并设置其监听读事件，也就是客户端的连接请求
  - 循环调用poll函数，检测pollfd数组就绪的文件描述符
    - 如果是连接套接字就绪，表明有客户端连接，使用accept接收连接，并创建已连接套接字，加入pollfd数组，监听读事件
    - 如果是已连接套接字就绪，表明客户端有读写请求，调用recv/send函数处理

  ```c
  int sock_fd,conn_fd; //监听套接字和已连接套接字的变量
  sock_fd = socket() //创建套接字
  bind(sock_fd)   //绑定套接字
  listen(sock_fd) //在套接字上进行监听，将套接字转为监听套接字
  
  //poll函数可以监听的文件描述符数量，可以大于1024
  #define MAX_OPEN = 2048
  
  //pollfd结构体数组，对应文件描述符
  struct pollfd client[MAX_OPEN];
  
  //将创建的监听套接字加入pollfd数组，并监听其可读事件
  client[0].fd = sock_fd;
  client[0].events = POLLRDNORM; 
  maxfd = 0;
  
  //初始化client数组其他元素为-1
  for (i = 1; i < MAX_OPEN; i++)
      client[i].fd = -1; 
  
  while(1) {
     //调用poll函数，检测client数组里的文件描述符是否有就绪的，返回就绪的文件描述符个数
     n = poll(client, maxfd+1, &timeout);
     //如果监听套件字的文件描述符有可读事件，则进行处理
     if (client[0].revents & POLLRDNORM) {
         //有客户端连接；调用accept函数建立连接
         conn_fd = accept();
  
         //保存已建立连接套接字
         for (i = 1; i < MAX_OPEN; i++){
           if (client[i].fd < 0) {
             client[i].fd = conn_fd; //将已建立连接的文件描述符保存到client数组
             client[i].events = POLLRDNORM; //设置该文件描述符监听可读事件
             break;
            }
         }
         maxfd = i; 
     }
     
     //依次检查已连接套接字的文件描述符
     for (i = 1; i < MAX_OPEN; i++) {
         if (client[i].revents & (POLLRDNORM | POLLERR)) {
           //有数据可读或发生错误，进行读数据处理或错误处理
         }
     }
  }
  ```

- **和select函数相比，poll函数允许一次监听超过1024个文件描述符，但是仍然需要遍历每个文件描述符**

#### epoll机制实现IO多路复用

1、epoll机制使用`epoll_event`结构体来记录待监听的文件描述符及其监听的事件类型

- `events`：整数类型变量，取值使用不同的宏定义值
- `epoll_data_t`：联合体变量
  - fd：记录文件描述符

```c
typedef union epoll_data
{
  ...
  int fd;  //记录文件描述符
  ...
} epoll_data_t;


struct epoll_event
{
  uint32_t events;  //epoll监听的事件类型
  epoll_data_t data; //应用程序数据
};
```

2、`epoll_create`函数

- 创建一个epoll实例，维护两个结构，分别是**记录要监听的文件描述符和已经就绪的文件描述符**，对于已经就绪的文件描述符会被返回给用户程序处理
- 所以就不需要遍历查询哪些文件描述符已经就绪

3、`epoll_ctl`函数

- 给被监听的文件描述符添加事件类型

4、`epoll_wait`函数

- 获取就绪的文件描述符

<img src="https://img-blog.csdnimg.cn/48eca7c0f422484a90b8c0aedf4efe79.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:50%;" />

```c
int sock_fd,conn_fd; //监听套接字和已连接套接字的变量
sock_fd = socket() //创建套接字
bind(sock_fd)   //绑定套接字
listen(sock_fd) //在套接字上进行监听，将套接字转为监听套接字
    
epfd = epoll_create(EPOLL_SIZE); //创建epoll实例，
//创建epoll_event结构体数组，保存套接字对应文件描述符和监听事件类型    
ep_events = (epoll_event*)malloc(sizeof(epoll_event) * EPOLL_SIZE);

//创建epoll_event变量
struct epoll_event ee
//监听读事件
ee.events = EPOLLIN;
//监听的文件描述符是刚创建的监听套接字
ee.data.fd = sock_fd;

//将监听套接字加入到监听列表中    
epoll_ctl(epfd, EPOLL_CTL_ADD, sock_fd, &ee); 
    
while (1) {
   //等待返回已经就绪的描述符 
   n = epoll_wait(epfd, ep_events, EPOLL_SIZE, -1); 
   //遍历所有就绪的描述符     
   for (int i = 0; i < n; i++) {
       //如果是监听套接字描述符就绪，表明有一个新客户端连接到来 
       if (ep_events[i].data.fd == sock_fd) { 
          conn_fd = accept(sock_fd); //调用accept()建立连接
          ee.events = EPOLLIN;  
          ee.data.fd = conn_fd;
          //添加对新创建的已连接套接字描述符的监听，监听后续在已连接套接字上的读事件      
          epoll_ctl(epfd, EPOLL_CTL_ADD, conn_fd, &ee); 
                
       } else { //如果是已连接套接字描述符就绪，则可以读数据
           ...//读取数据并处理
       }
   }
}
```



## 😊 复习

> 8大数据类型

string、hash、list、set、sortedSet、Bitmap、HyperLogLog、GEO

> String类型使用场景

**1、命令**

最常用：`set key value`、`get key`

设置多个：`mset k1 v1 k2 v2`、`mget k1 k2`

数值增减：

- 递增：`incr k1`
- 增加指定：`incrby k1 2`
- 递减：`decr k1`
- 减少指定：`decrby k1 2`

获取字符串长度：`strlen k1`

`setnx k1 v1`、`set key value [EX seconds][PX milliseconds] [NX|XX]`、`setex key seconds value`

- EX：key在多少秒后过期
- PX：key在多少毫秒后过期

- NX：当key不存在时，才创建key，效果等同于setnx
- XX：当key存在时，覆盖key

**2、应用场景**

- 商品编号、订单号采用incr命令生成



> hash应用场景

`Map<String,Map<Object,Object>>`

**1、命令**

一次设置一个字段值：`hset k1 f1 v1`

一次获取一个字段值：`hget k1 f1`

一次设置多个字段值：`hmset k1 f1 v1 f2 v2`

一次获取多个字段值：`hmget k1 f1 f2`

获取所有字段值：`hgetall k1`

获取某个key内全部数量：`hlen k1`

删除一个key：`hdel k1`

增加：`hincrby k1 f1 2`

**2、应用场景**

- 购物车

> list应用场景

**1、命令**

左边添加：`lpush k1 1 2 3 4` 

右边添加：`rpush k1 1 2 3 4`

查看列表（类似于分页）：`lrange k1 0 -1`

获取元素个数：`llen k1`

**2、应用场景**

- 点赞

> set应用场景

**1、命令**

添加元素：`sadd k1 m1 m2...`

删除元素：`srem k1 m1 m2...`

获取集合中的所有元素：`smembers k1`

判断元素是否在集合中：`sismember k1 m1`

获取集合中元素的个数：`scard k1`

从集合中随机弹出一个元素，元素不删除：`srandmember k1 [数字]`

从集合中随机弹出一个元素，出一个删一个：`spop k1 [数字]`

**2、集合运算**

交：`sinter key1 key2`

并：`sunion key1 key2`

差（属于k1不属于k2）：`sdiff key1 key2`

**3、应用场景**

- 微信抽奖小程序

  - 参与：`sadd key 用户ID`

  - 显示参与人数：`scard key`
  - 抽奖：`spop k1 1`

- 朋友圈点赞

  - 新增点赞：`sadd pub:msgID 点赞用户1 点赞用户2 `
  - 取消点赞：`srem pub:msgID 点赞用户1`
  - 展示所有点赞过的用户：`smembers pub:msgID`
  - 点赞用户统计：`scard pub:msgID`

- 好友社交关系
  - 共同关注：`sinter s1 s2`
- qq可能认识的人
  - `sdiff s1 s2`

> zset应用场景

向有序集合中加入元素和分数

**1、命令**

添加元素：`zadd k1 score m1`

按照元素分数从小到大顺序返回索引从start到stop之间的所有元素：`zrange k1 start stop [withscores]`

获取元素的分数：`zscore k1 m1`

获取排名：`zrank k1 m1`、`zrevrank k1 m1`

增加：`zincrby key 1 m1`

**2、场景**

- 热搜

- 排序显示

> 分布式锁



















## 😊 入门

**问题现象**

- 海量用户
- 高并发

**关系型数据库**

- 性能瓶颈：磁盘IO性能低下
- 扩展瓶颈：数据关系复杂、扩展性差、不便于大规模集群

**解决思路**

- 降低磁盘IO次数
- 去除数据间关系，越简单越好

### Nosql

1、Nosql：非关系型数据库，作为关系型数据库的补充

2、**作用：**应对基于海量用户和海量数据前提下的数据处理问题

3、特征：

- 可扩容、可伸缩
- 大数据量下高性能
- 灵活的数据模型
- 高可用
- **不支持ACID**

**4、常见Nosql数据库：**

- Redis
  - 数据在内存，支持持久化
  - 支持多种数据结构的存储
- memcache
  - 数据在内存，不支持持久化
- MongoDB
  - 文档型数据库
  - 对value（尤其是json）提供丰富的查询功能

### Redis简介

概念：用C语言开发的一个开源的高性能**键值对数据库**

特征：

- 数据间没有必然的关联关系
- 内部使用**单线程**
- 高性能
- 多数据类型支持
  - 字符串类型 string
  - 列表类型 list
  - 散列类型 hash
  - 集合类型 set
  - 有序集合类型 zset
- 数据类型都支持丰富操作，且是**原子性**的
- 持久化支持，可以进行数据灾难恢复
- 实现主从同步操作

### Redis应用

![在这里插入图片描述](https://img-blog.csdnimg.cn/084e21ceb94244d787ff17b4f2a8c438.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

### Redis安装

```bash
redis-benchmark：性能测试工具
redis-check-aof ：修复有问题的AOF文件
redis-check-rdb：修复有问题的RDB文件
redis-sentinel：Redis集群使用

redis-server：Redis服务器启动命令
redis-cli：客户端，操作入口
```

启动：

```bash
redis-server
```

### Redis相关知识

1、端口：6379

2、默认16个数据库，select 0/1/2

3、Redis是 **单线程+多路IO复用技术**

- **多路复用**：使用一个线程检查多个文件描述符（socket）的就绪状态，比如调用select和poll函数，传入多个文件描述符，如果有一个文件描述符就绪，则返回，否则阻塞直到超时。得到就绪状态后进行真正的操作可以在同一个线程里执行，也可以启用线程执行

 

## 😊 数据类型

### 介绍

业务数据的特殊性

**1、作为缓存使用**

- 原始业务功能设计：秒杀、双11、12306
- 运营平台监控到的突发高频访问数据：突发时政要闻
- 高频、复杂的统计数据：在线人数、投票排行榜

**2、附加功能**

- 系统功能优化或升级：单服务器升级集群、Session管理

**3、redis数据存储格式**

redis是一个Map，其中所有的数据都是采用key-value的形式存储

- **数据类型指的是存储的数据类型，也就是value部分的类型，key永远是字符串**

### 对key的操作

```mysql
key * ：查看当前库所有的key
exists key ：判断某个key是否存在
type key ：查看key的类型 

del key ：删除key的数据
unlink key ：根据value选择非阻塞删除，仅将keys从元数据中删除，真正的删除在后续异步完成

expire key 10 ：为给定key设置过期时间（单位：秒），不设置时间表示永远不过期
ttl key：查看剩余时间,-2表示过期，-1表示永不过期

dbsize：查看当前数据库key的数量
```

### string

**1、存储的数据：单个数据，最简单的数据存储类型，最常用**

**2、存储数据的格式：一个存储空间保存一个数据**

3、存储内容：通常使用字符串，如果以整数形式展示，也可以作为数字操作，value最大512M

4、基本操作

- 添加/修改数据：`set key value`
- 获取数据：`get key`
- 删除数据：`del key`

```sql
127.0.0.1:6379> del name                                                                  
(integer) 1 # 操作成功
127.0.0.1:6379> del name                                                                  
(integer) 0 # 操作失败
```

- 添加/修改多个数据：`mset key1 value1 key2 value2`
- 获取多个数据：`mget key1 key2`
- 获取数据字符串个数（字符串长度）：`strlen key`
- 追加信息到原始信息后部（如果原始信息存在就追加，否则新建）：`append key value`

> 单数据操作和多数据操作？

比如：操作三个数据

- 单数据操作：3次发送、3次回复、运行三次
- 多数据操作：1次发送，1次回复，运行三次

**结论：**没有固定结论，对于多数据操作，可能需要切割

**5、String类型数据的扩展操作**

> 业务场景：分布式ID
>
> 对于大型企业级应用，分表操作是基本操作，使用多张表存储同类型数据，但是对应的主键id必须保证统一性，不能重复，如何解决这个问题？

**解决方案**

- 设置数值数据增加指定范围的值

  ```sql
  # 加1
  incr key
  # 加指定数
  incrby key increment
  incrbyfloat key increment
  ```

- 设置数值减少指定范围的值

  ```sql
  decr key
  decrby key increment
  ```

Sting作为数值操作

- string在redis内部存储默认就是一个字符串，当遇到增减类操作incr、dec时会转为数值型进行计算
- **redis所有操作都是原子性的，采用单线程处理所有的业务，命令一个个执行**
- 注意：如果原始数据不能转为数值或者超越了数值上限，就会报错

**redis用于控制数据库主键id，为数据库表主键提供生成策略，保障数据库表的主键唯一性**

> 业务场景：控制时效性

**解决方案**

- 设置数据具有指定的生命周期

  ```sql
  setex key seconds value
  psetex key milliseconds value
  ```

redis控制数据的生命周期，通过数据是否失效控制业务行为，适用于所有具有时效性限定控制的操作

**6、数据结构**

String的数据结构为简单动态字符串（SDS），是可以修改的字符串，内部结构实现类似于java的arraylist，采用预分配冗余空间的方式来减少内存的频繁分配。

![在这里插入图片描述](https://img-blog.csdnimg.cn/77dffea9b4bd42509d140cead0bc18a8.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

内部为当前字符串实际分配的空间capacity，一般高于实际字符串长度len，当字符串长度小于1M时，扩容都是2倍，如果超过1M，扩容一次只多扩容1M空间。（最长512M）

> 为什么redis不使用char*？

- **char*的结构设计**

  - 一块连续的内存空间，依次存放字符串的每一个字符；最后一个字符是`\0`，标识字符串的结束

  - **影响**：如果保存的数据本身含有`\0`，那么数据会被截断，这就不符合redis希望能够保持任意数据的需求了

  - **操作函数的复杂度**：

    - 求数组长度，需要遍历每一个字符，复杂度是O（n）
    - 追加字符，需要遍历字符串获得末尾完成追加，还需要保证空间足够

    <img src="https://img-blog.csdnimg.cn/197459f2e1bc4a5e8a6b09a6389ae11d.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_18,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:50%;" />

- **原因：不符合redis对字符串高效操作的需求**



**SDS结构**

- 主要由`len`（buf数据所保存字符串的长度）、`free`（未使用的字节数量）、`buf[]`（保存字符串的数组）三个属性组成



<img src="https://img-blog.csdnimg.cn/e01985bdbd994d71aec1fe52945d27e1.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

- **本质还是数据，只是增加了额外的元数据**
- 追加操作：
  - 获取目标字符串长度，根据当前长度和需要追加的长度判断空间是否足够
  - 将数据拷贝到字符串结尾
  - 设置字符串的最新长度
- 优点
  - 效率高：使用`strlen`获取字符串长度在SDS中len属性记录了字符串长度，获取字符串长度的时间复杂度为O(1)
  - 数据溢出：SDS会自动扩容
- **内存重分配策略**：解决了字符串在增长和缩短时内存分配问题
  - **空间预分配策略**：当修改字符串时，会为SDS分配修改所必要的空间，还会为SDS分配额外使用的空间`free`，如果字符串修改后`len`小于1M，那么`free`和`len`相等，`len`值大于等于1M，那么`free`为1M
  - **惰性空间释放**：优化字符串缩短操作，使用free记录剩余空间



### List：单键多值

1、简介

- Redis列表是简单的字符串列表，按照**插入顺序排序**
- 可以添加一个元素到列表的头部或者尾部
- 底层实际上是个**双向列表**，对两端操作性能很高

**2、基本操作**

- 添加数据/修改数据

  ```sql
  lpush key value1 [value2]...
  rpush key value1 [value2]...
  ```

- 获取数据

  ```sql
  lrange key start stop    (lrange list1 0 -1)   
  lindex key index
  llen key
  ```

- 获取并移除数据

  ```sql
  lpop key
  rpop key
  ```

**3、扩展操作**

- 规定时间内获取并移除数据，阻塞式数据获取

  ```sql
  blpop key1 [key2] timeout
  brpop key1 [key2] timeout
  ```

> 业务场景：微信朋友圈点赞，要求按照点赞顺序显示点赞好友信息

<img src="https://img-blog.csdnimg.cn/20210417170350860.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

**从中间取消元素，移除指定元素**

```sql
lrem key count value

例子：
lrem list01 1 d
```

**4、注意事项**

- list中保存的数据是string类型
- 具有索引概念，但是操作数据通常以队列形式进行入队出队，或者栈的形式
- list可以对数据进行分页操作，通常第一页信息来自list，第2页及更多信息通过数据库形式加载

**5、数据结构**

List的数据结构为**快速链表**（quickList）

- 列表元素较少的情况下会使用一块连续的内存存储，这个结构是**zipList**，它将所有的元素紧挨着一起存储，分配的是一块连续的内存

- 数据量较多的情况下会使用**quickList**

  - 因为普通链表需要的附加指针空间太大，会比较浪费空间

  - Redis将链表和zipList结合起来组成quickList，将多个ziplist使用双向指针串起来使用

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/5eff04b5fdee4157af248456e0b2e81d.png)



### Set

> 对外提供功能与list类似的一个列表的功能，特殊之处在于set可以**自动排重**，当需要一个列表数据但不希望出现重复时，set是一个很好的选择，并且set提供了判断某个成员是否在集合内的接口
>
> Redis的set时string类型的无序集合，底层是value为null的**hash表**，复杂度为O（1 ）

<img src="https://img-blog.csdnimg.cn/20210417171747798.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

**1、基本操作**

- 添加数据

  ```sql
  sadd key m1 m2...
  ```

- 获取全部数据

  ```sql
  smembers key
  ```

- 删除数据

  ```sql
  srem key m1 m2
  ```

- 获取集合数据总量

  ```sql
  scard key
  ```

- 判断集合中是否包含指定数据

  ```sql
  sismember key member
  ```

**2、扩展操作**

> 每位用户首次使用今日头条会设置3个爱好的内容，后期为了增加用户的活跃度，需要让用户对其他信息类别逐渐产生兴趣，如何实现？

业务分析

- 系统分析出各个分类的最新或最热点信息条目并组织成set集合
- 随机挑选其中部分信息
- 配合用户关注信息分类中的热点信息组织成展示的全信息集合

解决方案

- 随机获取集合中指定数量的数据

  ```sql
  srandmember key count
  ```

- **随机获取**集合中某个数据并将数据移出集合

  ```sql
  spop key
  ```

**随机推荐类信息检索，如热点歌单推荐**

> qq好友推荐（共同好友）、微博推荐

解决方案

- 求两个集合的**交、并、差**

  ```sql
  sinter key1 key2
  sunion key1 key2
  sdiff key1 key2
  ```

- 求两个集合的交、并、差并存储到指定集合

  ```mysql
  sinterstore targetkey key1 key2
  sunionstore targetkey key1 key2
  sdiffstore targetkey key1 key2
  ```

- 将指定数据从原始集合移动到目标集合

  ```sql
  smove source targetkey member
  ```

**redis应用于同类信息的关联搜索，二度关联搜索、深度关联搜索**

**显示共同好友、共同关注**

3、注意事项

- set类型不允许数据重复
- set虽然和hash存储结构相同，无法启用hash中的存储值的空间

4、应用场景

> 集团有1000名员工，内部系统有700多个角色，3000多个业务操作，23000多种数据每位员工具有一个或多个角色，如何进行权限校验？

解决方案：

- 根据用户id获取用户所有角色
- 根据用户所有角色获取用户所有操作权限放入set集合（合并并存储）
- 根据用户所有角色获取用户所有数据全选放入set集合

> 统计网站的PV（访问量）、UV（独立访客）、IP（独立IP）
>
> PV：被访问的次数
>
> UV：不同用户访问次数
>
> IP：不同IP地址访问次数

解决方案：

- 建立string类型数据，利用incr统计日访问量（PV）
- 建立set模型，记录不同cookie数量（UV）
- 建立set模型，记录不同ip数量（IP）

> 黑白名单

解决方案：

- 周期性更新用户黑名单，加入set集合
- 用户行为信息到达后和黑名单进行比对，确认行为去向
- 黑名单过滤IP地址：应用于开放游客访问权限的信息源
- 黑名单过滤设备信息：应用于限定访问设备的信息源
- 黑名单过滤用户：应用于基于访问权限的信息源

**5、数据结构**

set的数据结构是dict字典，字典是用哈希表实现的

Java中的HashSet的内部实现使用HashMap，Redis的set结构也是一样的，内部使用hash结构，所有的value指向同一个内部值



### Hash

1、简介

- 键值对集合
- 是一个string类型的field和value的映射表，hash特别适合存储对象

<img src="https://img-blog.csdnimg.cn/20210417161600331.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

2、基本操作

- 添加/修改数据：`hset key field value`

- 获取数据：

  ```sql
  hget key field
  hgetall key
  ```

- 添加/修改多个数据：`hmset key field1 value1 field2 value2`

- 获取多个数据：`hmget key field1 field2`

- 获取哈希表中字段的数量：`hlen key`

- 获取哈希表中是否存在指定字段：`hexists key field`

**3、扩展操作**

- 获取哈希表所有的字段名或字段值

  ```sql
  hkeys key
  hvals key
  ```

- 设置指定字段的数值数据增加指定范围的值

  ```sql
  hincrby key field increment
  hincrbyfloat key field increment
  ```

**4、注意事项**

- hash类型的value只能存储字符串
- 每个hash可以存储2^32-1个
- hgetall操作可以获取全部属性，如果field过多，遍历效率会很低，有可能成为数据访问瓶颈

**4、应用场景**

> 购物车设计和实现

<img src="https://img-blog.csdnimg.cn/20210417162753154.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />

**业务分析**

- 添加、浏览、更改数量、删除、清空

**解决方案**

- 用户id为key，每位客户创建一个hash存储结构存储对应的购物车信息
- 商品编号为field，购买数量为value
- 添加商品：追加全新的field和数量value
- 浏览：遍历hash
- 更改数量：自增/自减，设置value值
- 删除商品：删除field
- 清空：删除key

**问题：当前只是将数据存储到了redis中，并没有起到加速的作用，商品信息还需要二次查询数据库**

- 每条商品记录保存为两条field
- field1用于保存购买数量
  - 命名格式：`商品id：nums`
  - 数据：数值

- field2用于保存购物车显示的信息，包括文字描述、图片地址等
  - 命名格式：`商品id：info`
  - 数据：json

```sql
127.0.0.1:6379> hmset 004 g01:nums 100 g01:info {...}                                     
OK                                                                                                                                                            
127.0.0.1:6379> hgetall 004                                                               
1) "g01:nums"                                                                             
2) "100"                                                                                  
3) "g01:info"                                                                             
4) "{...}"                                                                                
127.0.0.1:6379> 
```

**优化：商品相同，会出现信息重复，可以将field2独立成hash**

```sql
# 如果key中对应的field有vlaue，什么都不做，没有值就加进去
hsetnx key field value
```

> 抢购商品

<img src="https://img-blog.csdnimg.cn/2021041716452792.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

解决方案：

- 以商家id作为key
- 参与抢购的商品id作为field
- 数量为value
- 使用降值的方式控制

**5、数据结构**

Hash对应的数据结构有两种：ziplist、hashtable

- field-value长度较短且个数较少，使用ziplist
- field-value长度较长且个数较多，使用hashtable

#### Hash数据结构

> 两个问题：**哈希冲突、rehash**
>
> Redis解决方案：
>
> - 哈希冲突：链式哈希
> - rehash：渐进式rehash

**1、链式哈希**

- 用链表把映射到hash表同一个桶中的键连接起来

2、**rehash**

- redis准备了两个哈希表用于rehash时交替保存数据













### Zset有序集合

> 新的存储需求：数据排序有利于数据有效展示，需要提供一种可以根据自身特征进行排序的方式
>
> 需要的存储结构：新的存储模型，可以保存可排序的数据
>
> zset类型：在set的存储结构基础上添加可排序字段 score



<img src="https://img-blog.csdnimg.cn/20210417175132505.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

**1、基本操作**

- 添加数据

  ```sql
  zadd key score1 m1 score2 m2
  ```

- 获取全部数据，已经排序

  ```sql
  # 从小到大
  zrange key start stop 【withscores】
  # 从大到小
  zrevrange key start stop 【withscores】
  ```

- 删除数据

  ```sql
  zrem key m1 m2 m3...
  ```

- 按条件获取数据

  ```sql
  zrangebyscore key min max [withscores][limit]
  zrevrangebyscore key max min [withscores]
  ```

- 条件删除数据

  ```sql
  zremrangebyrank key start stop
  zremrangebyscore key min max
  ```

- 获取集合数据总量

  ```sql
  zcard key
  zcount key min max
  ```

- 集合交、并操作

  ```sql
  zinterstore target numkeys key1 key2
  zunionstore target numkeys key1 key2
  ```

**2、扩展操作**

> 电影TOP10
>
> - 对资源建立排序依据

解决方案

- 获取数据对应的索引（排名）

  ```sql
  zrank key member
  zrevrank key member
  ```

- score值获取与修改

  ```sql
  zscore key member
  zincrby key increment member
  ```

3、注意事项

- score有范围，64位
- score保存的数据可以是一个双精度double值，基于双精度浮点数的特征可能会丢失精度，使用时要慎重
- **zset底层存储还是基于set结构，因此数据不能重复，如果重复添加相同数据，score会被覆盖，保留最后一次修改的结果**

**4、应用场景**

> 基于时效性任务管理
>
> 试用VIP，VIP到期之后，如果有效管理此类信息，即便对于正式VIP用户也存在对应的管理方式

解决方案

- 将处理时间记录为score值
- 记录下一个要处理的时间，当到期后处理对应任务，移除redis中的记录，记录下一个要处理的时间
- 新任务加入时，判定并更新当前下一个要处理的任务时间
- 为了提升性能，通常将任务根据特征存储成若干个zset，例如1小时内、1天内等，操作时逐级提升，将即将操作的若干个任务纳入1小时内处理的队列中
- 获取当前系统时间：`time`

> 权重任务队列/消息队列

解决方案

- 对于带权重的任务，优先处理权重高的任务，采用score记录权重

**多条件任务权重**

- 如果权限过多，需要对排序score值进行处理

**5、数据结构**

zset是Redis提供的一个非常特别的数据结构，一方面等价于java的Map<String,Double>，可以给每个元素value赋予一个权重score，另一方面又类似于TreeSet，内部的元素会按照score进行排序，得到每个元素的名次，还可以根据score的范围获取元素的列表

底层使用了两个数据结构：

- **hash**，hash的作用是关联value和score，保障元素value的唯一性，可以通过元素value找到对应的score
- **跳表**，目的是给value排序，根据score的范围获取元素列表

> 为什么不用平衡二叉树？

- 更加节省内存
- 遍历更加友好
- 更容易实现和维护

#### 跳表

1、可以实现二分查找的有序链表

**2、查找的时间复杂度**

- 过程：从最高级索引开始，一层一层遍历最后下沉到原始链表
- 时间复杂度=索引高度*每层索引遍历元素的个数
- O（Logn）

<img src="https://img-blog.csdnimg.cn/d76ad8ea4e8149d0a806efab0223b060.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:50%;" />

**3、空间复杂度**

- 假设**每两个结点会抽出一个结点作为上一级索引的结点**，原始的链表有n个元素，则一级索引元素个数为n/2，二级索引元素个数n/4，所以索引节点的总和是：n/2 + n/4 + n/8 + … + 8 + 4 + 2 = n-2，**空间复杂度是 O(n)**。
- 如果**每三个结点抽一个结点做为索引**，索引总和数就是 n/3 + n/9 + n/27 + … + 9 + 3 + 1= n/2，减少了一半
- 因此，**可以通过减少索引数来减少空间复杂度**
- O（n）

**4、插入数据**

- 概率算法：告诉我们这个元素需要插入到几级索引中
- 最坏时间复杂度：O（logn）

<img src="https://img-blog.csdnimg.cn/cfd28470af644f7d9c9d2a961c70c1c1.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:30%;" />

<img src="https://img-blog.csdnimg.cn/0dcb1affc4be42b68b50f6d1f66a4772.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5b-r5LmQ55qE5Yay5rWq56CB5Yac,size_20,color_FFFFFF,t_70,g_se,x_16" alt="在这里插入图片描述" style="zoom:30%;" />

<img src="/Users/zhangtao/Desktop/截屏2021-09-06 下午3.32.42.png" alt="截屏2021-09-06 下午3.32.42" style="zoom:33%;" />

<img src="/Users/zhangtao/Desktop/截屏2021-09-06 下午3.32.48.png" alt="截屏2021-09-06 下午3.32.48" style="zoom:40%;" />





### Redis配置文件

```conf
################################## NETWORK #####################################

# 表示只能本地访问
bind 127.0.0.1 ::1

# 表示开启保护模式
protected-mode yes

port 6379

# 设置tcp的backlog，backlog是一个连接队列，队列总和=未完成三次握手队列+已经完成三次握手队列
tcp-backlog 511

# 超时时间 0表示永不超时
timeout 0

# 表示检查心跳的时间
tcp-keepalive 300

################################# TLS/SSL #####################################


################################# GENERAL #####################################
# redis后台启动
daemonize no

# 保存进程号
pidfile /var/run/redis_6379.pid

# 日志级别：
# debug (a lot of information, useful for development/testing)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (moderately verbose, what you want in production probably)
# warning (only very important / critical messages are logged)
loglevel notice

# 日志输出文件路径
logfile ""

# 数据库个数
databases 16

always-show-logo no

set-proc-title yes

################################ SNAPSHOTTING  ################################
stop-writes-on-bgsave-error yes

rdbcompression yes

rdbchecksum yes

dbfilename dump.rdb

dir /usr/local/var/db/redis/

################################# REPLICATION #################################

replica-serve-stale-data yes

replica-read-only yes

repl-diskless-sync no

repl-diskless-sync-delay 5

############################### KEYS TRACKING #################################


################################## SECURITY ###################################


################################### CLIENTS ####################################


############################## MEMORY MANAGEMENT ################################


############################# LAZY FREEING ####################################

lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no


lazyfree-lazy-user-del no

lazyfree-lazy-user-flush no

################################ THREADED I/O #################################

############################ KERNEL OOM CONTROL ##############################
oom-score-adj no
oom-score-adj-values 0 200 800


#################### KERNEL transparent hugepage CONTROL ######################

disable-thp yes

############################## APPEND ONLY MODE ###############################
appendonly no

no-appendfsync-on-rewrite no

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

aof-load-truncated yes

aof-use-rdb-preamble yes

################################ LUA SCRIPTING  ###############################

lua-time-limit 5000

################################ REDIS CLUSTER  ###############################


########################## CLUSTER DOCKER/NAT support  ########################


################################## SLOW LOG ###################################

slowlog-max-len 128

################################ LATENCY MONITOR ##############################


############################### GOPHER SERVER #################################


############################### ADVANCED CONFIG ###############################

```

### 发布和订阅

1、Reds发布订阅是一种消息通信模式

2、Redis客户端可以订阅任意数量的频道

3、命令行实现

- 打开一个客户端订阅channel1

  ```sql
  subscribe channel1
  ```

- 打开另外一个客户端，给channel1发布消息

  ```sql
  publish channel1 hello
  ```

  

### Bitmaps

1、简介

- 本身不是一种数据类型，实际上是字符串，但是可以对字符串的位进行操作
- 单独提供一套命令，在redis中使用bitmaps和 使用字符串的方法不同。可以把bitmaps想象成一个以位为单位的数组，数组的每个单元存储0和1

  

### HyperLogLog

### Geospatital



### 案例

> 微信接收消息默认将最近接收的消息置顶，多个好友及订阅号同时发消息，该排序会不停进行交替，同时还可以将重要的会话设置为置顶，一旦用户离线后，再次打开微信时消息怎么按顺序展示？

依赖list的数据具有顺序的特征

对置顶好友和普通好友分别创建独立的list

当某个list接受到用户消息后，将消息发送方的id从list的一侧加入list

多个相同id加入需要先删除

先推送置顶会话list



## 😊 通用命令

### key通用命令

#### key特征

1、key是一个字符串，通过key获取redis保存的数据

> key应该设计哪些操作？

对于key自身状态相关操作

对于key有效性控制相关操作

对于key快速查询

#### key基本操作

1、删除指定key

```sql
del key
```

2、获取key是否存在

```sql
exists key
```

3、获取key类型

```sql
type key
```

#### key扩展操作（时效性）

1、为指定key设置有效期

```sql
expire key seconds
pexpire key milliseconds
expireat key timestamp
pexpireat key milliseconds-timestamp
```

2、获取key有效期

- -2：不存在
- -1：存在
- 当前有效时长

```sql
ttl key
pttl key
```

3、切换key转换到永久性

```sql
persist key
```

#### key扩展操作（查询模式）

1、查询key

- `Keys *` 查询索引
- `Keys it*` 查询所有以it开头的
- `keys ??h` 查询前面两个字符任意，后面以h结尾
- `keys user:?` 查询所有以user：开头，最后一个字符任意

```sql
keys pattern
```

#### key其他操作

1、为key改名字

```sql
# 同名就覆盖
rename key newkey
# 如果不存在改名
renamenx key newkey
```

2、对所有key排序

```sql
sort
```

3、其他key 通用操作

```sql
help @generic
```

### 数据库通用命令

> key重复问题

1、切换数据库

```sql
select index
```

2、其他操作

```sql
quit
ping 
echo message
```

3、数据移动

```sql
move key db_index
```

## 😊 Jedis

```java
public void testJedis(){
  //1、连接redis
  Jedis jedis = new Jedis("192.168.164.134",6379);
  //2、操作redis
  jedis.select(2);
  jedis.set("name","zhang");
  System.out.println(jedis.get("name"));
  //3、关闭连接
  jedis.close();
}
```



## 😊 事务和锁

 Redis事务是一个**单独的隔离操作**，事务中的所有命令都会序列化、按顺序执行。事务在执行的过程中，不会被其他客户端发送的命令请求打断

Redis事务的主要作用是**串联多个命令**防止别的命令插队

### Multi、Exec、discard

从输入multi命令开始，输入的命令都会依次进入命令队列中，但不会执行，直到输入exec奇偶，redis将之前的命令依次执行。

组队过程中可以使用discard放弃组队

![在这里插入图片描述](https://img-blog.csdnimg.cn/0343f819374b40799612483619dcf49a.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

1. **如果组队中命令出现错误，那么所有的命令都不会执行**；
2. **如果执行出现运行错误，那么正确的命令会被执行，只有错误的命令不执行**。

### 事务冲突

![在这里插入图片描述](https://img-blog.csdnimg.cn/4879b58d97404d9781cfad06ca5d850f.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

#### 悲观锁

每次拿数据的时候认为别人会修改，所以每次在拿数据的时候会加上锁，这样别人想去拿数据就会阻塞。

![在这里插入图片描述](https://img-blog.csdnimg.cn/16d8ef1f15d94f808b824de927a27dad.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

#### 乐观锁

使用版本号机制

适用于多读的应用类型，这样可以提高吞吐量，redis就是使用这种check-and-set机制实现事务的

![在这里插入图片描述](https://img-blog.csdnimg.cn/8c46bbf3590449c2aa1497b01a3a1fac.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

#### watch key

在执行multi之前，先执行watch key1 [key2]，可以监视一个或多个key。

**如果在事务执行之前这些key被其他命令改动，那么事务被打断，监控一直持续到EXEC命令**

#### 事务三特性

1、单独的隔离操作

- 事务的所有命令都会序列化、按顺序的执行。事务在执行过程中不会被其他客户端发送的命令请求打断

2、没有隔离级别的概念

- 队列的命令在提交之前都不会执行

3、不保证原子性

- 事务中如果有一条命令执行失败，后面的命令仍然会执行，不回滚

### 秒杀案例

![在这里插入图片描述](https://img-blog.csdnimg.cn/e04b93e8a4b84ae0b156516c698b6357.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

>  set集合判断用户是否重复秒杀操作：`sismember`
>
> 库存-1：`decr`

问题：

- 连接超时问题：使用连接池解决
- **超卖问题**

![在这里插入图片描述](https://img-blog.csdnimg.cn/3f08040453ee433fa6fd3ddcd19c0cc6.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

解决方案：

- **乐观锁**

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/60d1b46a79e7450f855706ad747adb13.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

```java
//watch
jedis.watch(kcKey);
//获取库存，如果库存为null，秒杀开没有开始

//如果商品数量小于1，秒杀结束

//开始秒杀
//使用事务
Transaction multi = jedis.multi();
//组队操作
multi.decr(kcKey);
multi.sadd(userKey,uid);
//执行
List<Object> results = multi.exec();
if(results==null || results.size()==0){
  jedis.close(); 
}
```

**库存遗留问题**

- 乐观锁造成库存遗留问题
- 解决：
  - LUA脚本
    - 将复杂的或者多步的redis操作，写为一个脚本，一次提交给redis执行，减少反复连接redis的次数
    - lua脚本类似redis的事务，有一定的原子性，不会被其他命令插队
    - 利用lua脚本淘汰用户，解决超卖问题
    - 通过lua解决争抢问题，实际上是**redis利用单线程的特性，用任务队列的方式解决多任务并发问题**

### 分布式锁

> 超卖问题，避免一件商品被多个人修改

业务分析：

- 使用watch监控一个key有没有改变不能解决问题，需要监控的是具体数据
- 虽然redis是单线程的，但是**多个客户端对同一数据进行操作，如何避免不被同时修改？**

解决方案

- **使用setnx设置一个公共锁**：利用setnx命令返回值特征，有值就返回设置失败（无控制权），无值则返回设置成功（有控制权），操作完毕通过del操作释放锁

```sql
setnx lock-key value
```



> 业务场景
>
> 依赖分布式锁的机制，某个用户操作时对应客户端当即，且此时已经获取到锁，如何解决？

业务分析：

- 由于锁操作由用户控制加锁解锁，必定会存在加锁后不解锁的风险
- 需要解锁操作不能仅依赖用户控制，系统级别要给出保底处理方案

解决方案

- 使用expire为锁key添加时间限制

```sql
expire lock-key second
pexpire lock-key milliseconds
```



## 😊 持久化

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210417214135821.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

> 什么是持久化？

持久化：利用永久性存储介质将数据进行保存，在特定的时间将保存的数据进行恢复的工作机制

> 为什么要进行持久化？

防止数据的意外丢失，确保数据安全

> 持久化过程保存什么？

1、RDB：将当前数据状态进行保存，**快照形式**，存储数据结果，存储格式简单，关注点在数据（**全量快照**）

2、AOF：将数据的操作过程进行保存，**日志形式**，存储操作过程，存储格式复杂，关注点在数据的操作过程

### RDB

> 谁？什么时间？干什么事情？

- 谁：redis操作者
- 什么时间：即时
- 干什么事情：保存数据

#### save、bgsave

1、命令：

```sql
save
```

<img src="https://img-blog.csdnimg.cn/20210417214829822.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

2、save指令相关配置

<img src="https://img-blog.csdnimg.cn/20210417214936359.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

3、save工作原理

**save指令会阻塞当前Redis服务器，线上环境不建议使用**

> 数据量过大，单线程执行方式造成效率过低如何处理？

**后台执行**

- 谁：使用者
- 什么时间：合理的时间
- 干什么事：保存数据

1、命令

```sql
bgsave
```

2、作用

手动启动后台保存操作，但是不是立即执行的

```sql
127.0.0.1:6379> bgsave                                            
Background saving started 
```

3、工作原理

<img src="https://img-blog.csdnimg.cn/2021041722000451.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

**bgsave是针对save阻塞问题做的优化，Redis内部所涉及的RDB操作都采用bgsave方式**

> 快照时数据能够修改吗？

Redis借助操作系统提供的写时复制（COW），在执行快照的时候能够正常处理些操作

bgsave子进程是由主线程fork生成的，可以共享主线程的内存数据；如果主线程要修改一块数据，那么这个数据块就会被复制一份，主线程在这个数据副本上进行修改。bgsave可以继续把原来的数据写入rdb

这既保证了快照的完整性，也允许主线程同时对数据进行修改，避免了对正常业务的影响

<img src="https://img-blog.csdnimg.cn/77090b87a1134c398c20b07bc9c79f63.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />



#### 自动执行

> 反复执行保存指令，忘记了怎么办？不知道数据产生了多少变化？何时保存？

**自动执行**

- 谁：redis服务器发起指令（基于条件）
- 什么时间：满足条件
- 干什么事情：保存数据

1、配置

```sql
save second changes
```

作用：满足限定时间范围内（second）key的变化数量达到指定数量（changes）即进行持久化

2、工作原理

4、原理

<img src="https://img-blog.csdnimg.cn/20210417220936467.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

save配置要根据实际业务设置，随意设置会出现问题

使用的是bgsave

#### 对比

<img src="https://img-blog.csdnimg.cn/20210418100055166.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

#### 优点和缺点

1、优点

- 存储效率高
- 内部存储的是redis在某个时间点的数据快照
- **速度比AOF快**
- 应用：服务器中每X个小时执行bgsave备份，将rdb文件拷贝到远程机器中，**用于灾难恢复**

2、缺点

- 无法做到实时持久化，具有较大可能丢失数据
- bgsave需要fork操作创建子进程，要牺牲一些性能
- redis众多版本没有进行rdb的格式统一，可能出现兼容问题

### AOF

> RDB的弊端：
>
> - 数据量大，效率低、IO性能低
> - 基于fork创建子进程，内存产生额外消耗
> - 宕机带来的数据丢失风险
>
> 解决思路
>
> - 不写全数据，仅记录部分数据
> - 改记录数据为记录操作过程
> - 对所有操作均进行记录，排除丢失数据的风险

1、AOF概念：以独立日志的方式记录每次写命令，重启时再重新执行AOF中命令来恢复数据，与RDB相比可以简单描述为**改记录数据为记录操作过程**

2、主要作用：解决了数据持久化的实时性，目前已经是Redis持久化的主流方式

#### 写数据的过程

<img src="https://img-blog.csdnimg.cn/20210418093029908.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

4、AOF功能开启

- 配置：开启AOF持久化功能

```sql
appendonly yes
```

- 配置：AOF写数据策略

```sql
appendfsync always|everysec|no
```

#### 风险

1、如果刚执行完一个命令就宕机了，这个命令和数据会有丢失的风险

2、AOF日志在主线程执行，在把日志写入磁盘时，磁盘写压力大，导致写盘很慢，导致后面操作无法执行

#### 回写策略

三个写数据的策略

- **always**：每次写入操作均同步到AOF文件中，**数据零误差，性能较低**
- **everysec**：每秒将缓冲区的指令同步到AOF文件中，数据**准确性较高，性能较高**，在系统突然宕机的情况下丢失1s的数据
- **no**：操作系统控制每次同步到AOF文件的周期，**过程不可控制**

<img src="https://img-blog.csdnimg.cn/6a0844aeff924075ba596c47451a1474.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

#### AOF重写

随着命令不断写入AOF，文件会越来越大，为了解决这个问题，Redis引入了AOF重写机制压缩文件体积，AOF文件重写是将Redis进程内的数据转化为写命令同步到新AOF文件的过程。

**简单的说，就是将同一个数据的若干执行结果转化为最终结果数据对应的指令进行记录**

作用：

- 降低磁盘占用量
- 提高持久化效率
- 降低恢复数据的效率

规则：

- 进程内已超时的数据不再写入文件
- 忽略无效指令，重写时使用进程内数据直接生成，新的AOF只保留最终数据的写入命令
- 对同一个数据的多条命令进行合并
- 把rdb快照以二进制的形式附在新的aof头部

> 重写会阻塞吗？

重写过程由后台子进程来完成，这也是为了避免阻塞主线程，导致数据库性能下降

**一个拷贝，两处日志**

- **一个拷贝**：每次执行重写时候，主线程fork出后台的bgrewriteaof子进程。此时，fork 会把主线程的内存拷贝一份给 bgrewriteaof 子进程，这里面就包含了数据库的最新数据。然后，bgrewriteaof 子进程就可以在不影响主线程的情况下，逐一把拷贝的数据写成操作，记入重写日志
- **两处日志**：如果有写操作，第一处日志就是指正在使用的AOF日志，Redis会把这个操作写到它的缓冲区；第二处日志指的是重写日志

<img src="https://img-blog.csdnimg.cn/40fb5c3624894d1a9bd83240cdc7b5b3.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />



### RDB和AOF区别

<img src="https://img-blog.csdnimg.cn/20210418095510742.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />



#### 如何选择

**1、对数据非常敏感，建议使用默认的AOF持久化方案**

- AOF持久化策略使用everysecond，每秒钟fsync一次，该策略redis仍可以保持很好的处理性能，当出现问题，最多丢失1s数据
- 注意：AOF文件存储体积大，且恢复速度慢

**2、数据呈现阶段有效性，建议使用RDB持久化方案**

- 数据可以良好的做到阶段内无丢失，且恢复速度较快
- 注意：利用RDB实现紧凑数据持久化会降低Redis性能

3、综合比对

- RDB和AOF是一种权衡
- **不能承受分钟以内的数据丢失，对业务数据敏感，使用AOF**
- **能够承受分钟以内的数据丢失，且追求大数据集的恢复速度，使用RDB**
- **灾难恢复使用RDB**
- 双保险策略，同时开启RDB和AOF，重启后Redis**优先使用AOF**恢复数据

### 持久化应用场景

<img src="https://img-blog.csdnimg.cn/20210418100351741.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

不需要持久化：1、3、4、9、12（长期）、15

持久化：5、6、7、12（短期）、13、16



## 😊 删除策略

### 过期数据

Redis是一种内存级的数据库，所有数据均存放在内存中，内存中的数据可以通过TTL指令获取其状态

- XX：具有时效性的数据
- -1：永久有效的数据
- -2：**已经过期的数据**或**被删除的数据**或**未定义的数据**

### 删除策略

#### 时效性数据的存储结构

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210418111159381.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)



> 删除策略的目标

在内存占用和cpu占用之间寻找平衡点

#### 定时删除

**1、创建一个定时器，当key设置有过期时间，且过期时间到达，由定时器任务立即执行对键的删除操作**

2、优点：节约内存，快速释放

3、缺点：cpu压力很大，会影响redis服务器的响应时长、吞吐量

4、总结：时间换空间

#### 惰性删除

**1、数据到达过期时间不处理，等下次访问该数据时如果已经过期就删除**

2、`expirelfNeeded()`：检查数据过期

3、优点：节约cpu性能

4、缺点：内存压力很大，出现长期占用内存的数据

5、总结：存储空间换处理器性能

#### 定期删除

1、过程

- Redis服务器初始化时，读取配置server.hz的值，默认为10；
- 每秒钟执行server.hz次`serverCron()->databasesCron()-> activeExpireCycle()`
- `activeExpireCycle()`对每个expires[*]逐一检测，随机挑选w个key检测
  - 如果key超时就删除
  - 如果一轮中删除的key数量>w * 0.25循环该过程
  - 如果小于等于就检查下一个expires[*]，0-15循环
  - w值在配置文件中设置

- 参数current_db用于记录`activeExpireCycle()`进入哪一个expires[*]执行
- 如果`activeExpireCycle()`执行时间到期，下次从current_对比继续执行

**2、周期性轮询redis库中的时效性数据，采用随机抽取的策略，利用过期数据占比的方式控制删除频率**

3、特点：

- cpu性能占用设置有峰值，检测频率可以自定义设置
- 内存压力不是很大，长期占用内存的冷数据会被持续清理

**4、总结：周期性抽查**

#### 比对

**1、定时删除**

- 定义计时器，过期就删除
- 节约内存，无占用
- 不分时间占用cpu

**2、惰性删除**

- 下次访问就删除
- 内存占用严重
- cpu利用率高

**3、定期删除**

- 轮询redis库，随机抽查
- 每秒花费固定的cpu资源维护

### 内存淘汰机制

> 如果新数据进入redis时，如果内存不足怎么办？

redis使用内存存储数据，执行每一个命令前，会调用`freeMemoryifNeeded()`检测内存是否充足

**如果内存不满足新加入数据的最低存储要求，redis要临时删除一些数据，清理数据的策略为逐出算法**

#### 配置

最大可用内存

```sql
maxmemory
```

每次选取待删除数据个数

```sql
maxmemory-samples
```

删除策略

```sql
maxmemory-policy
```

#### 检测易失数据

**可能会过期的key**

- `voliate-lru`：挑选最近最少使用（时间上）的数据进行数据淘汰（LRU）
- `voliate-lfu`：挑选最近使用次数最少（次数上）的数据淘汰（LFU）
- `volatile-ttl`：挑选将要过期的数据淘汰
- `volatile-random`：任意选择数据淘汰

#### 检测全库数据

**所有key**

- `allkeys-lru`
- `allkeys-lfu`
- `allkeys-random`：任意选择数据淘汰

#### 放弃数据驱逐

- `noenviction`：禁止驱逐

### LRU算法

> 是什么？

Least Recently used，**常用页面置换算法**

**选择最近最久未使用的数据淘汰**

https://leetcode-cn.com/problems/lru-cache/

**核心：哈希链表**

- 本质：HashMap+DoubleLinkedList

#### 手写LRU

```java
public class LRUCache{
  //Node节点作为数据载体
  class Node<k,v>{
    k key;
    v value;
    Node<k,v> prev;
    Node<k,v> next;
    public Node(){
      this.prev=this.next=null;
    }
    public Node(k key,v value){
      this.key=key;
      this.value=value;
      this.prev=this.next=null;
    }
  }

  //双向链表，里面放node
  class DoubleLinkedList<k,v>{
    Node<k,v> head;
    Node<k,v> tail;

    public DoubleLinkedList() {
      head=new Node<>();
      tail=new Node<>();
      head.next=tail;
      tail.prev=head;
    }
    //添加到头
    public void addHead(Node<k,v> node){
      node.prev=head;
      node.next=tail.next;
      head.next.prev=node;
      head.next=node;
    }

    //删除节点
    public void removeNode(Node<k,v> node){
      node.next.prev=node.prev;
      node.prev.next=node.next;
      node.next=null;
      node.prev=null;
    }

    //获得最后一个节点
    public Node<k,v> getLast(){
      return tail.prev;
    }
  }

  //缓存坑位
  private int capacity;
  Map<Integer,Node<Integer,Integer>> map;
  DoubleLinkedList<Integer,Integer> doubleLinkedList;

  public LRUCache(int capacity){
    this.capacity=capacity;
    map=new HashMap<>();
    doubleLinkedList=new DoubleLinkedList<>();
  }

  public int get(int key){
    if(!map.containsKey(key)){
      return -1;
    }
    Node<Integer, Integer> node = map.get(key);
    doubleLinkedList.removeNode(node);
    doubleLinkedList.addHead(node);
    return node.value;
  }

  public void put(int key,int value){
    if(map.containsKey(key)){
      Node<Integer, Integer> node = map.get(key);
      node.value=value;
      map.put(key,node);
      doubleLinkedList.removeNode(node);
      doubleLinkedList.addHead(node);
    }else {
      if(map.size()==capacity){
        Node<Integer, Integer> last = doubleLinkedList.getLast();
        map.remove(last.key);
        doubleLinkedList.removeNode(last);
      }else {
        Node<Integer,Integer> newNode = new Node<>(key,value);
        map.put(key,newNode);
        doubleLinkedList.addHead(newNode);
      }
    }

  }
}
```

## 😊 主从复制

1、是什么？

- 主机数据更新后根据配置和策略，自动同步到备机的master/slaver机制，master以写为主，slave以读为主

<img src="https://img-blog.csdnimg.cn/de647e0d9da341d7a2e7d90aba4e7d67.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

2、用处

- 读写分离
- 容灾恢复

### 主从复制原理

1、从服务器连接上主服务器之后，从服务器向主服务器发送进行数据同步消息

2、主服务器接到从服务器发送过来的同步消息之后，把主服务器数据进行持久化，rdb文件，把rdb文件发送给从服务器，从服务器读取rdb文件

3、每次主服务器进行写操作后，和从服务器进行数据同步

#### 主从库如何进行第一次同步

**1、第一阶段**：主从库间建立连接、协商同步的过程。从库和主库建立起来连接，告诉主库即将进行同步，主库确认回复后主从库就可以开始同步了

- 从库给主库发送psync命令，表示要进行数据同步
- 主库收到psync命令后会用FULLRESYNC响应命令带上两个参数（**表示第一次复制采用全量复制**）

2、**第二阶段**：主库将所有数据同步给从库，从库收到数据后在本地完成数据加载

- 主库执行bgsave命令生成rdb文件，将文件发送给从库
- 从库清空数据库，加载rdb文件
- 同步过程中的写操作主库会在内存中用buffer记录写操作

3、**第三阶段**：主库把第二阶段执行过程中新收到的写命令发送给从库

#### 主-从-从模式

> 如果从库数量很多，都需要和主库进行全量复制，就会导致主库忙于fork子进程生成RDB文件，进行数据全量同步，“主-从-从”可以分担主库压力
>
> redis cluster模式下，无法使用主-从-从的级联模式

<img src="https://img-blog.csdnimg.cn/dee1641f08cb4982a6b6a9474cb48911.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

#### 主从库间网络断了怎么办

1、网络断后，主从库会采用增量复制的方式继续同步

2、主库会把断连期间收到的写操作命令写入 replication buffer，同时也会把这些操作命令也写入 repl_backlog_buffer 这个缓冲区。

3、repl_backlog_buffer 是一个环形缓冲区，主库会记录自己写到的位置，从库则会记录自己已经读到的位置。

### 哨兵机制

> 能够后台监控主机是否故障，如果故障了根据投票数量自动将从库转换为主库

<img src="https://img-blog.csdnimg.cn/5525ce7078a443c6a662aa90159176ab.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:33%;" />

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67310a227b284539810f3914ec1f3822~tplv-k3u1fbpfcp-watermark.awebp)

主要功能包括：

- 主节点生存检测，通过ping监控
- 自动故障转移：**当主节点无法正常工作时，Sentinel将启动自动故障转移操作。它将与发生故障的主节点处于主从关系的从节点之一升级到新的主节点，并将其他从节点指向新的主节点；**
- 通知：通知其他从库和客户端新的主库

#### 监控：主观下线和客观下线

1、哨兵进程会使用ping命令检测它自己和主、从库的网络连接情况，用来判断实例的状态

2、**主观下线**：发现从库超时了就标记为主观下线

3、误判一般会发生在集群网络压力较大、网络拥塞，或者是主库本身压力较大的情况，**如何减少误判？**

- **哨兵集群**：通常采用多实例组成的集群模式进行部署，引入多个哨兵实例一起来判断，就可以避免单个哨兵因为自身网络状态不好而误判主库下线的情况。

4、**客观下线**：只有大多数的哨兵实例，都判断主库已经“主观下线”了，主库才会被标记为“客观下线”

#### 选定新的主库

1、过程：筛选+打分

- 先按一定的照筛选条件把不符合条件的从库去掉，再按照一定的规则给剩下的从库打分

<img src="https://img-blog.csdnimg.cn/cbce5522e96c4db4b7975e387179b29f.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

**2、筛选条件**

- 检查从库的在线状态、判断之前的网络连接状态

**3、打分**

- 第一轮：优先级高的从库得分高（`slave-priority`配置项）
- 第二轮：和旧主库同步程度最接近的从库得分高
- 第三轮：ID号小的从库得分高（最早连接上）

### 哨兵集群

在配置哨兵集群的时候，配置哨兵的信息需要设置**主库的IP**和**端口**，**并没有配置其他哨兵的连接信息**

#### 基于pub/sub机制的哨兵集群组成

1、哨兵实例之间可以相互发现，归功于Redis提供的发布/订阅机制

**2、哨兵间如何互相知道彼此地址？**

- 哨兵只要和主库建立起了连接，就可以在主库上发布消息（比如发布自己的连接信息），同时也可以从主库上订阅消息，获得其他哨兵发布的连接信息
- 频道：`__sentinel__:hello`

<img src="https://img-blog.csdnimg.cn/46b627811f8941efbfa8735e9285dc45.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

**3、哨兵如何知道从库的IP地址和端口？**

- 哨兵向主库发送`INFO`命令，主库返回从库列表给哨兵

<img src="https://img-blog.csdnimg.cn/4881d351a5c34ef09b2c8f6206a25e95.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

#### 基于pub/sub机制的客户端事件通知

**1、哨兵是特殊的redis实例**

- 每个哨兵也提供发布/订阅机制，客户端可以从哨兵订阅信息

2、哨兵提供不同的消息订阅频道

<img src="https://img-blog.csdnimg.cn/9d8c9f1ecc2642149f2601d7d8ce2f04.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

#### 由哪个哨兵执行主从切换？

1、判断客观观下线

- 任何一个哨兵只要判断主库“主观下线”后就会给其他哨兵发送is-master-down-by-addr 命令，其他哨兵也会根据自己和主库的连接情况做出响应
- 一个哨兵获得了所需要的赞同票数后可以标记主库“客观下线”
- **此时这个哨兵会给其他哨兵发送命令，表示希望自己来执行主从切换，并让其他哨兵投票（Leader选取）**

**2、Leader选取**

- 需要满足两个条件：
  - 拿到半数以上的赞成票
  - 拿到的票数大于等于哨兵配置文件的quorum值

3、需要注意的是，如果哨兵集群只有 2 个实例，此时，一个哨兵要想成为 Leader，必须获得 2 票，而不是 1 票。所以，如果有个哨兵挂掉了，那么，此时的集群是无法进行主从库切换的。因此，**通常我们至少会配置 3 个哨兵实例**。这一点很重要，你在实际应用时可不能忽略了。

## 😊切片集群

1、概念

- 切片集群也叫分片集群，指启动多个Redis实例组成一个集群，然后按照一定的规则将收到的数据划分成多份，每一份用一个实例来保存

<img src="https://img-blog.csdnimg.cn/ba87fc1e7abf4de18a14decf5427853c.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

### 如何保存更多数据？

1、为了保存大量的数据，使用大内存云主机（纵向扩展）和切片集群（横向扩展）两种方法

- **纵向扩展**：升级单个redis实例的资源配置，包括增加内存容量、增加磁盘容量等
- **横向扩展**：增加redis实例的个数

2、使用横向扩展要解决的两个问题：

- 数据在多个实例如何分布？
- 客户端怎么确定想要访问的数据在哪里？

### 数据切片和实例的对应分布关系

#### Redis Cluster

1、在redis 3.0之后，官方提供了Redis Cluster的方案用于实现切片集群，规定了数据和实例的对应规则

**2、Slot**

- Redis Cluster一个切片集群由16384个哈希槽
- 哈希槽类似于数据分区，每个键值对都会根据key被映射到一个哈希槽中

**3、映射过程**

- 根据key按照CRC16算法计算一个16bit的值
- 用这个值对16384取模，得到哈希槽

**4、哈希槽的映射**

- 使用cluster create创建集群，此时redis会自动把这些槽平均分布在集群实例上

<img src="https://img-blog.csdnimg.cn/64bebe51c02d4b038cd2dc4e069a2731.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

### 客户端如何定位数据？

一般来说，客户端和集群实例建立连接后，实例就会把哈希槽的分配信息发给客户端。但是，**在集群刚刚创建的时候，每个实例只知道自己被分配了哪些哈希槽，是不知道其他实例拥有的哈希槽信息的**

## 😊 应用问题

### 缓存穿透

![在这里插入图片描述](https://img-blog.csdnimg.cn/d2193c72722a4411b4e39cbcbcdb4dd9.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

**1、背景**

- 应用服务器压力变大了
- redis命中率降低，一直查询数据库

**2、现象**

- redis查询不到数据
- 出现大量非正常url访问

3、解决方案

- **对空值缓存 **：如果一个查询返回的数据为空，把这个结果进行缓存
- **设置可访问的白名单**：使用bitmaps定义一个可访问的名单，名单id作为bitmaps的偏移量，每次访问和bitmap里面的id做比较
- **采用布隆过滤器**

### 缓存击穿

![在这里插入图片描述](https://img-blog.csdnimg.cn/d493f2a1b9ce446f93571be04667c5db.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

**1、现象**

- 数据库访问压力瞬间增加
- redis没有出现大量key过期
- redis正常运行

**2、原因**

- redis的某个key过期，大量访问使用这个key

**3、解决方案**

- 预先设置热门数据
- 事时调整：监控热门数据，实时调整key的过期时长
- 使用锁：
  - 缓存失效的时候，不是立即去load db
  - 先使用缓存工具的某些成功操作返回值的操作（redis的setnx）去set一个mutex key
  - 操作返回成功时，再进行load db的操作，并回设缓存，删除mutex key
  - 操作返回失败，说明有线程在load db，当前线程睡眠一段时间重试

### 缓存雪崩

![在这里插入图片描述](https://img-blog.csdnimg.cn/2a01fe182c544e6f9558186e364e94b0.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

**1、现象**

- 数据库压力变大，服务器崩溃
- 大量key集中过期

2、解决方案

- **构建多级缓存架构**：ngnix缓存+redis缓存+其他缓存
- **使用锁**：保证不会有大量线程对数据库一次性进行读写，从而避免失效时出现大量并发请求
- **设置过期标志更新缓存**：记录缓存数据是否过期，如果过期会触发通知另外的线程在后台更新实际key的缓存
- **分散过期时间**

### 分布式锁

1、使用setnx上锁，使用del释放锁

2、锁一直没有释放，设置key的过期时间，自动释放

```sql
setnx user 1
expire 10
```

3、上锁之后突然出现异常，无法设置过期时间

```sql
上锁，设置过期时间
set 10 nx ex 10
```















