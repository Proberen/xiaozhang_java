# JVM和Java体系结构

## 虚拟机与Java虚拟机

1、所谓虚拟机，就是一台虚拟的计算机，大体上分为**系统虚拟机和程序虚拟机**

2、Java虚拟机是一台执行**Java字节码**的虚拟计算机，拥有独立的运行机制，其运行的Java字节码也未必由java语言编译而成

3、Java技术的核心就是**Java虚拟机**，所有的Java程序都运行在Java虚拟机内部

4、特点：

- 一次编译，到处运行
- 自动内存管理
- 自动垃圾回收功能

<img src="https://img-blog.csdnimg.cn/2021032819422653.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

JVM是运行在操作系统之上的，与硬件没有直接交互

## JVM整体结构

HotSpot VM是高性能虚拟机的代表作之一

<img src="https://img-blog.csdnimg.cn/20210328194500996.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:60%;" />

1、类装载器子系统：将字节码文件加载到内存中，生成class对象（加载、链接、初始化）

2、运行时数据区：方法区、Java栈、本地方法栈、堆、程序计数器

3、执行引擎：高级语言翻译成机器语言



## Java代码执行流程

<img src="https://img-blog.csdnimg.cn/20210328195719927.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:60%;" />

## JVM的架构模型

Java编译器输入的指令流基本上是一种**基于栈的指令集架构**

特点：

- 设计和实现更简单，适用于资源受限的系统
- 避开了寄存器的分配难题，使用零地址指令方式分配
- 指令流中的指令大部分是零地址指令，其执行过程依赖于操作栈
- 不需要硬件的支持，可移植性更好，更好实现**跨平台**

**跨平台、指令集小、指令多、执行性能比寄存器差**

## JVM的生命周期

**启动**

通过引导类加载器创建一个初始类来完成，这个类由虚拟机的具体实现指定

<hr/>

**执行**

- 一个运行中的java虚拟机有一个清晰的任务：执行Java程序
- 程序开始执行他才开始运行，程序结束就停止
- 执行一个Java程序的时候，真正执行的是Java虚拟机的进程

<hr/>

**退出**

如下几种情况：

- 程序正常执行结束

- 异常终止

- 由于操作系统导致终止

- 线程调用Runtime类或System类的exit方法等

  