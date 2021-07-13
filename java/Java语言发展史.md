
## 1.1 Java语言发展史
1、Java语言
- Java语言是美国 <span style="background: yellow;">sun公司</span>在1995年推出的
- Java之父：詹姆斯.高斯林

2、Java语言发展历史

<img src="https://img-blog.csdnimg.cn/20210307203148237.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

- 5.0：更新力度非常大，使得java发展进入快车道
- 8.0：公司使用最多
- 11.0：学习使用的版本

## 1.2 Java语言跨平台原理——JVM
1、基本概念
- 平台：指的是操作系统平台（windows、macOS、Linux）
- 跨平台：Java程序可以在任意操作系统上运行

2、<span style="background: yellow;">跨平台原理</span>
- 针对不同的操作系统提供不同的JVM
- 方法：在需要运行Java应用程序的操作系统上安装一个与操作系统对应的Java虚拟机（JVM）即可完成跨平台

3、<span style="background: yellow;">JVM</span>：运行 <span style="background: yellow;">Java 字节码</span>的虚拟机。JVM 有针对不同系统的特定实现（Windows，Linux，macOS），目的是使用相同的字节码，它们都会给出相同的结果。

> 什么是字节码?采用字节码的好处是什么?

- 含义：在 Java 中，JVM 可以理解的代码就叫做字节码（即扩展名为 .class 的文件），它不面向任何特定的处理器，只面向虚拟机。
- 好处：Java 语言通过字节码的方式，在一定程度上解决了传统解释型语言执行效率低的问题，同时又保留了解释型语言可移植的特点。所以 Java 程序运行时比较高效，而且，由于字节码并不针对一种特定的机器，因此，Java 程序无须重新编译便可在多种不同操作系统的计算机上运行。

4、java程序运行过程

Java 程序从源代码到运行一般有下面 3 步：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210307204848421.png)

<span style="background: yellow;">.class->机器码</span>
1. JVM 类加载器首先加载字节码文件
2. 然后通过解释器逐行解释执行，这种方式的执行速度会相对比较慢。而且，有些方法和代码块是经常需要被调用的(也就是所谓的热点代码)，所以后面引进了 <span style="background: yellow;">JIT 编译器</span>，而 JIT 属于运行时编译。当 JIT 编译器完成第一次编译后，其会将字节码对应的机器码保存下来，下次可以直接使用。而我们知道，机器码的运行效率肯定是高于 Java 解释器的。这也解释了我们为什么经常会说 Java 是编译与解释共存的语言。

> HotSpot 采用了惰性评估(Lazy Evaluation)的做法，根据二八定律，消耗大部分系统资源的只有那一小部分的代码（热点代码），而这也就是 JIT 所需要编译的部分。JVM 会根据代码每次被执行的情况收集信息并相应地做出一些优化，因此执行的次数越多，它的速度就越快。
> JDK 9 引入了一种新的编译模式 AOT(Ahead of Time Compilation)，它是直接将字节码编译成机器码，这样就避免了 JIT 预热等各方面的开销。JDK 支持分层编译和 AOT 协作使用。但是 ，AOT 编译器的编译质量是肯定比不上 JIT 编译器的。

## 1.3 JRE和JDK
1、**JRE**（Java Runtime Environment）
- 含义：java程序的运行时的环境，<span style="background: yellow;">包含JVM和运行时所需要的核心类库</span>
- 我们想要<span style="background: yellow;">运行</span>一个已有的java程序，只需要按照JRE即可

2、**JDK**（Java Development kit）
- 含义：java程序开发工具包，<span style="background: yellow;">包含JRE和开发人员使用的工具</span>
- 开发工具：编译工具（javac.exe）和运行工具（java.exe）
- 想要开发一个全新的Java程序，必须安装JDK

3、JDK、JRE、JVM的关系
<img src="https://img-blog.csdnimg.cn/20210307204240270.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

## 1.4 Oracle JDK 和 OpenJDK 

1、Oracle JDK 大概每 6 个月发一次主要版本，而 OpenJDK 版本大概每三个月发布一次，但这不是固定的。

2、OpenJDK 是一个参考模型并且是完全<span style="background: yellow;">开源</span>的，而 Oracle JDK 是 OpenJDK 的一个实现，并<span style="background: yellow;">不是完全开源</span>的；

3、Oracle JDK 比 OpenJDK 更<span style="background: yellow;">稳定</span>。OpenJDK 和 Oracle JDK 的代码几乎相同，但 Oracle JDK 有更多的类和一些错误修复。因此，如果您想开发企业/商业软件，我建议您选择 Oracle JDK，因为它经过了彻底的测试和稳定。

4、在响应性和 JVM 性能方面，Oracle JDK 与 OpenJDK 相比提供了<span style="background: yellow;">更好的性能</span>；
Oracle JDK 不会为即将发布的版本提供长期支持，用户每次都必须通过更新到最新版本获得支持来获取最新版本；

5、Oracle JDK 根据<span style="background: yellow;">二进制代码许可协议</span>获得许可，而 OpenJDK 根据 <span style="background: yellow;">GPL v2 许可</span>获得许可。

## 1.5 Java 和 C++的区别
- 都是面向对象的语言，都支持封装、继承和多态
- Java 不提供指针来直接访问内存，程序内存更加安全
- Java 的类是单继承的，C++ 支持多重继承；虽然 Java 的类不可以多继承，但是接口可以多继承。
- Java 有<span style="background: yellow;">自动内存管理垃圾回收机制(GC)</span>，不需要程序员手动释放无用内存
- 在 C 语言中，字符串或字符数组最后都会有一个额外的字符'\0'来表示结束。但是，Java 语言中没有结束符这一概念。

## 1.6 Java——编译与解释并存
Java 语言既具有编译型语言的特征，也具有解释型语言的特征，因为 Java 程序要经过先编译，后解释两个步骤，由 Java 编写的程序需要先经过编译步骤，生成字节码（*.class 文件），这种字节码必须由 Java 解释器来解释执行。因此，我们可以认为 Java 语言编译与解释并存。