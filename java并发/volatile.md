# Volatile

Volatile是JVM提供的轻量级同步机制

## 保证可见性

1、可见性：当一个线程修改了这个变量的值，新值对于其他线程都是立即可知的

```java
private volatile static int number = 0;
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

3、如何保证可见性的？

Java代码：

```java
instance = new Singlenton(); //instance是volatile变量
```

转为汇编代码，如下：

```java
0x01a3deld: movd $0x0,0x1104800(%esi);
0x01a3de24:lock add1 $0x0,(%esp);
```

对volatile变量修饰的共享变量进行写操作时会多出第二行汇编代码，**lock前缀的指令**在多核处理器下会引发两件事情：

- 将当前处理器缓存行的数据写回系统内存
- 这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效

## 不保证原子性

> 有人说：“volatile变量对所有线程是立即可见的，对volatile变量所有的写操作都能够立即反映到其他线程之中。也就是说volatile变量的运算在并发下是线程安全的”

事实上，volatile变量在各个线程的工作内存中是不存在一致性问题的，但是Java的运算操作符是非原子操作，这导致volatile变量的运算在并发下一样是不安全的

对Voliate变量的**单次读写**是可以保证原子性的，如long和double类型变量，但是不能保证自增自减这种操作，因为本质上是读、写两次操作

也就是说，<span style="background: yellow;">volatile是不保证原子性的！！</span>

### 案例演示

```java
private volatile static int number = 0;

public static void test_add(){
    number++;
}
public static void main(String[] args) throws InterruptedException {
  	//20个线程
    for(int i=0;i<20;i++){
        new Thread(()->{
            for(int j=0;j<1000;j++){
                test_add();
            }
        }).start();
    }
  	//需要等待上面20个线程都计算完成后再用main线程取得最后的结果
  	//如果线程数量大于2，说明上面20个线程没有计算完
    while (Thread.activeCount()>2){
        Thread.yield();
    }
    System.out.println(Thread.currentThread().getName()+number);
}
```

**输出结果：**

```java
main19325
```

### 原理

number++被拆分成3个指令

- 执行GETFIELD：拿到主内存中的原始值number

- 执行IADD：进行加1操作

- 执行PUTFIELD：把工作内存中的值写回主内存中

当多个线程并发执行PUTFIELD指令的时候，会出现写回**主内存覆盖问题**，所以才会导致最终结果不为20000，volatile不能保证原子性。

### 解决

- 加`Synchronized`或者`lock`
- 使用原子类保证原子性

<img src="https://img-blog.csdnimg.cn/20210318225446165.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

```java
private volatile static AtomicInteger number = new AtomicInteger(0);

public static void test_add(){
    number.getAndIncrement();
}
```

## 禁止指令重排

**1、重排序分类**

计算机在执行程序时，为了提高性能，编译器和处理器常常会对指令进行重排，分为以下三种：

- **编译器优化的重排序**：编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序
- **指令级并行的重排序**：现代处理器采用了指令级并行技术（ILP）将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序
- **内存系统的重排序**：由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去是在乱序执行

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210320104649190.png)

第一个属于**编译器重排序**，第二个和第三个属于**处理器重排序**

对于处理器重排序，JMM的处理器重排序会要求java编译器在生成指令序列时，插入特定类型的**内存屏障指令**，<span style="background: yellow;">通过内存屏障指令来禁止特定类型的处理器重排序</span>

**2、分析**

- 单线程环境下，确保程序最终执行结果和代码顺序执行的结果一致
- 处理器在进行重排序要考虑指令之间的<span style="background: yellow;">数据依赖性</span>
- **多线程环境下，线程交替执行，由于编译器重排的存在，两个线程使用的变量能否保证一致性是无法确定的，结果无法预测**

### 数据依赖性

如果两个操作访问同一个变量，且两个操作中有一个写操作，此时两个操作存在数据依赖性，分为三个类型

- 写后读

```java
a=1
b=a
```

- 写后写

```java
a=1;
a=2;
```

- 读后写

```java
a=b;
b=1;
```

以上三种情况，只要进行重排序，结果就会发生改变

编译器和处理器在重排序时，**会遵守数据依赖性**

### as-if-serial语义

1、as-if-serial语义：不管怎么重排序，<span style="background: yellow;">单线程程序的执行结果不能被改变</span>；编译器、runtime和处理器都必须尊重as-if-serial语义。

2、为了遵守as-if-serial语义，编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变执行结果

### happens-before语义

先行发生原则，JMM 定义的两项操作间的偏序关系，是判断数据是否存在竞争的重要手段。

JMM 将 happens-before 要求禁止的重排序按是否会改变程序执行结果分为两类。对于会改变结果的重排序 JMM 要求编译器和处理器必须禁止，对于不会改变结果的重排序，JMM 不做要求。

JMM 存在一些天然的 happens-before 关系，无需任何同步器协助就已经存在。如果两个操作的关系不在此列，并且无法从这些规则推导出来，它们就没有顺序性保障，虚拟机可以对它们随意进行重排序。

- **程序次序规则：**一个线程内写在前面的操作先行发生于后面的。
- **管程锁定规则：** unlock 操作先行发生于后面对同一个锁的 lock 操作。
- **volatile 规则：**对 volatile 变量的写操作先行发生于后面的读操作。
- **线程启动规则：**线程的 `start` 方法先行发生于线程的每个动作。
- **线程终止规则：**线程中所有操作先行发生于对线程的终止检测。
- **对象终结规则：**对象的初始化先行发生于 `finalize` 方法。
- **传递性：**如果操作 A 先行发生于操作 B，操作 B 先行发生于操作 C，那么操作 A 先行发生于操作 C 。

#### 区别

as-if-serial 保证单线程程序的执行结果不变，happens-before 保证正确同步的多线程程序的执行结果不变。

这两种语义的目的都是为了在不改变程序执行结果的前提下尽可能提高程序执行并行度。



1、**内存屏障（Memory Barriers）**

- 又称为内存栅栏，是一个CPU指令

作用：

- 保证特定操作的执行顺序
- 强制刷出各种CPU的缓存数据，保证某些变量的内存**可见性**（利用这个特性实现volatile的内存可见性）

编译器和处理器会对指令进行重排序，JMM 为了保证在不同的编译器和 CPU 上有相同的结果，通过<span style="background: yellow;">插入特定类型的内存屏障禁止在内存屏障前后的指令执行重排序优化</span>，插入一条内存屏障会告诉编译器和 CPU：不管什么指令都不能和这条 Memory Barrier 指令重排序。

**对Voliate变量进行写操作时**，会在写操作后加入一条store屏障指令，将工作内存中的共享变量值刷新回主内存

<img src="https://img-blog.csdnimg.cn/20210320112153961.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

**对Voliate变量进行读操作时**，会在读操作前加入一条load屏障指令，从主内存中读取共享变量

<img src="https://img-blog.csdnimg.cn/20210320112310138.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />



## 你在哪里用过Volatile？

### 单例模式DCL代码

共6种

DCL机制不一定线程安全，因为有指令重排序的存在，加入volatile可以禁止指令重排

原因：某一个线程执行到第一次检测，读取到instance不为null时，instance的引用对象可能没有完成初始化。

`instance = new add();`分为3步完成：

```java
//1、分配对象内存空间
memory = add();
//2、初始化对象
instance(memory)；
//3、设置instance指向刚分配的内存地址
instance = memory;
```

步骤2和步骤3不存在数据依赖关系，而且无论重排前还是重排后程序的执行结果在单线程没有改变

因此，这种重排优化是允许的

```java
//1、分配对象内存空间
memory = add();
//3、设置instance指向刚分配的内存地址 但是对象还没有初始化完成！
instance = memory;
//2、初始化对象
instance(memory)；
```





```java
public class add{
    private static volatile add instance = null;
    private add(){
        System.out.println(Thread.currentThread().getName()+"\t构造方法");
    }
    //DCL(双重检查)
    public static add getInstance(){
        if(instance==null){
            synchronized (add.class){
                if(instance==null){
                    instance = new add();
                }
            }
        }
        return instance;
    }

    public static void main(String[] args){
        //并发多线程
        for (int i =0;i<10;i++){
            new Thread(()->{
                add.getInstance();
            }).start();
        }
    }
}
```

