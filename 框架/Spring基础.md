# Spring基础

## 概述

> Spring出现在2002年左右，帮助企业降低开发的难度，减轻对项目模块之间的管理、类和类之间的管理，帮助开发人员创建对象，管理对象之间的关系，能实现模块之前、类之间的解耦合

**1、优点**

- **轻量**

  使用的jar都比较小，一般在1M以下，Spring的核心功能所需要的jar共3M左右

- **针对接口编程，接耦合**

  提供了IOC控制反转，由容器管理对象，对象的依赖关系，原来在程序代码中的对象创建方式现在由容器完成，对象之间的依赖解耦合

- **AOP编程的支持**

- **方便集成各种优秀的框架**

2、体系结构

<img src="https://img-blog.csdnimg.cn/20210405160312415.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

Spring由20多个模块组成，它们可以分为数据访问/集成(Data Access/Integration)、 Web、面向切面编程(AOP, Aspects)、提供JVM的代理(Instrumentation)、消息发送(Messaging)、 核心容器(Core Container)和测试(Test)。

