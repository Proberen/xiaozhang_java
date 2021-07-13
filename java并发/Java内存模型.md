# Java内存模型

## 硬件的效率和一致性

1、**高速缓存（Cache）**：内存与CPU之间的缓冲，将运算所需要使用的数据复制到缓冲中，让运算能快速进行，当运算结束后再从缓存同步回内存之中，这样处理器就不需要等待缓慢的内存读写了

引出了新的问题：**缓存一致性**，为了解决该问题，需要各个处理器访问缓存时遵循一些协议（MSI、MESI、MOSI等）

2、**乱序执行优化**：为了使处理器内部的运算单元能够被尽量充分利用，处理器可能会对输入代码进行乱序执行优化，处理器会在计算之后将乱序执行的结果重组，保证结果与顺序执行的结果一致，但并不保证程序中各个语句计算的先后顺序与输入代码中的顺序一致。因此，如果存在一个计算任务依赖另外一个计算任务的中间结果，那么<span style="background: yellow;">其顺序性不能靠代码的先后顺序保证</span>。

Java虚拟机类似的操作：<span style="background: yellow;">指令重排序优化</span>

## JMM

1、JMM 即 Java Memory Model，它定义了<span style="background: yellow;">主存</span>、<span style="background: yellow;">工作内存</span>抽象概念，底层对应着CPU寄存器、缓存、硬件内存、CPU指令优化等

2、JMM的**主要目的**：定义程序中各种变量的访问规则，即关注在虚拟机中把变量存储到内存和从内存中取出变量值这样的底层细节

### 主内存与工作内存

1、**变量**：实例字段、静态字段、构成数组对象的元素，**不包含局部变量和方法参数**（线程私有的，不会被共享）

2、**主内存**：存储所有的变量

- 与物理上的主内存对比？在物理上，这里的主内存只是虚拟机内存的一部分

3、**工作内存**：每个线程都有自己的工作内存

- 与高速缓存类比
- 保存了被该线程使用的变量的**主内存副本**
- 线程对变量的所有操作都在工作内存中进行，而不能直接读写主内存的数据

<img src="https://img-blog.csdnimg.cn/20210318213002745.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

### 内存间交互操作

主内存与工作内存之间的具体的交互协议，即一个变量如何从主内存拷贝到工作内存、如何从工作内存同步到主内存这一类的实现细节

<img src="https://img-blog.csdnimg.cn/20210318205404626.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom: 67%;" />

> **8个操作**：

1、**lock**（锁定）：作用于**主内存的变量**，它把一个变量标识为一条线程独占的状态

2、**unlock**（解锁）：作用于**主内存的变量**，它把一个处于锁定状态的变量释放出来，释放后的变量可以被其他线程锁定

3、**read**（读取）：作用于**主内存的变量**，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的load操作

4、**load**（载入）：作用于**工作内存的变量**，它把read操作从主内存得到的变量值放入工作内存的变量副本中

5、**use**（使用）：作用于**工作内存的变量**，它把工作内存中的一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用变量的值的字节码指令时将会执行这个操作

6、**assign**（赋值）：作用于**工作内存的变量**，它把执行引擎接受的值赋值给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时将会执行这个操作

7、**store**（存储）：作用于**工作内存的变量**，它把工作内存中的一个变量的值传送到主内存中，以便随后的write操作使用

8、**write**（写入）：作用于**主内存的变量**，它把store操作从工作内存得到的变量值放到主内存的变量中

> **8个需要满足的规则**：

1、不允许read和load、store和write操作之一单独出现，即不允许一个变量从主内存读取了但工作内存不接受，或者工作内存发起store，主内存不write的情况出现

2、不允许一个线程丢弃最近的assign操作，即变量在工作内存改变了之后必须把变化同步回主内存

3、不允许一个线程没有原因（没有assign操作）的把数据从工作内存同步回主内存

4、一个变量只能在主内存诞生，不允许在工作内存直接使用一个没有被初始化的变量，也就是说在对一个变量实施use、store之前必须执行assign和load

5、一个变量在同一时刻只运行一个线程lock

6、对一个变量执行lock，**会清空工作内存此变量的值**，在执行引擎使用这个变量前，需要重新load或者assign操作以初始化

7、unlock前必须lock

8、执行unlock前必须store、write



## 特性一：可见性

**问题**

```java
		private static int number = 0;
    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            while (number==0){
            }
        },"t").start();

        TimeUnit.SECONDS.sleep(1);
        number=1;
        System.out.println(number);
    }
```

**上面的程序存在一个问题：**线程t不会停下来

为什么呢？分析一下：

1、初始状态，t线程刚开始从主内存读取了number的值到工作内存

<img src="https://img-blog.csdnimg.cn/20210319112448555.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

2、因为t线程要频繁从主内存中读取number的值，JIT编译器会将number的值缓存至自己的工作内存中的高速缓存中，减少对run的访问，提高效率

<img src="https://img-blog.csdnimg.cn/20210319112624346.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />



3、1s之后，main线程更改了number的值，并同步回主内存，而t是从自己的工作内存中的高速缓存中读取这个变量的值，结果永远是旧值

<img src="https://img-blog.csdnimg.cn/20210319112749121.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

### 解决：Volatile关键字

为了解决上面的这个问题，就需要把变量声明为`Volatile`

- 修饰成员变量和静态成员变量

- 指示JVM这个变量是共享的且不稳定的，<span style="background: yellow;">每次使用它都要去主存中读取</span>

```java
private volatile static int number = 0;
```

**当然，也可以使用synchronized解决这个问题**：

```java
private static int number = 0;
final static Object lock = new Object();
public static void main(String[] args) throws InterruptedException {
  new Thread(()->{
    synchronized(lock){
      while (number==0){
        
      }
    }
  },"t").start();

  TimeUnit.SECONDS.sleep(1);
  number=1;
  System.out.println(number);
}
```



## 特性二：原子性

**什么是原子性？**

不可分割，完整性，也即某个线程正在做具体业务时，中间不可以被分割或者加塞，需要整体完整

要么同时成功，要么同时失败



## 特性三：有序性







