# CAS

>  对于并发控制而言，锁是一种悲观的策略，它总是假设每一次的临界区操作会产生冲突，因此会对每一次的操作都小心翼翼。如果有多个线程访问临界区资源，就宁可牺牲性能也不会让线程等待，所以说锁会阻塞线程执行。
>
> **无锁**是一种乐观的策略，它会假设对资源的访问是没有冲突的。遇到冲突怎么办？无锁的策略使用一种**比较交换的技术（CAS，Compare And Swap）**来鉴别线程冲突，一旦检测到冲突产生，就重试当前的操作直到没有冲突为止。



## 什么是CAS

CAS 表示 Compare And Swap，比较并交换，需要三个操作数

包含三个参数CAS（V，A，B）

- V：表示要更新的变量，内存位置
- A：表示旧的预期值
- B：表示新值

CAS 指令执行时，当且仅当 V 符合 A 时，处理器才会用 B 更新 V 的值，否则它就不执行更新。但不管是否更新都会返回 V 的旧值，这些处理过程是原子操作，执行期间不会被其他线程打断。

**这是一种完全依赖于`硬件`的功能**，通过它实现了原子操作。原语的执行时连续的，在执行过程中不允许被中断，也就是说CAS是一条CPU的原子指令，不会造成数据不一致。

在 JDK 5 后，Java 类库中才开始使用 CAS 操作，该操作由 Unsafe 类里的 `compareAndSwapInt` 等几个方法包装提供。HotSpot 在内部对这些方法做了特殊处理，即时编译的结果是一条平台相关的处理器 CAS 指令。Unsafe 类不是给用户程序调用的类，因此 JDK9 前只有 Java 类库可以使用 CAS，譬如 juc 包里的 AtomicInteger类中 `compareAndSet`等方法都使用了Unsafe 类的 CAS 操作实现。

```java
main{
  AtomicInteger number = new AtomicInteger(2020);
  number.compareAndSet(2020,2021);
}

public class AtomicInteger extends Number implements java.io.Serializable {
  //如果达到期望的值，就更新，否则就不更新
  public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
  }
}

public final class Unsafe {
  //处理器 CAS 指令
  public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
}
```



## CAS底层原理

```java
main{
  AtomicInteger number = new AtomicInteger(2020);
  number.getAndIncrement();
}
```

**1、Unsafe类**

是CAS的核心类，由于Java方法无法直接访问底层系统，需要通过本地方法来访问，Unsafe相当于一个后门，基于该类可以直接操作特定内存的数据。

<span style="background: yellow;">**Unsafe类存在于sun.misc包中**</span>，其内部方法操作可以像C的指针（通过内存偏移量）一样操作内存，因为Java中CAS操作的执行依赖于Unsafe类的方法。

**2、valueOffset**

表示该变量值在内存中的<span style="background: yellow;">偏移地址</span>，因为Unsafe就是根据内存偏移地址来获取数据的。

```java
public class AtomicInteger extends Number implements java.io.Serializable {
  private static final Unsafe unsafe = Unsafe.getUnsafe();
  private static final long valueOffset;
  static {
        try {
          	//获取内存偏移量
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
  private volatile int value;
}
```

**3、变量value使用volatile修饰，保证了多线程之间的内存可见性**



> 问：CAS是什么？

CAS是一条**<span style="background: yellow;">CPU并发原语</span>**，它判断内存某个位置的值是否为预期值，如果是更改为新的值，这个过程是原子的

CAS并发原语体现在JAVA语言中就是Unsafe类中的各个方法。调用这些方法，JVM可以帮我们实现**CAS汇编指令**，这是一种完全依赖于硬件的功能，通过它实现了原子操作。

再次强调，CAS是一种系统原语，原语属于操作系统用语范畴，是由若干条指令组成的，用于完成某个功能的一个过程，并且原语的执行必须是连续的，执行过程中不允许被打断，<span style="background: yellow;">也就是说CAS是一条CPU的原子指令，不会造成数据不一致的问题</span>。



### 例一源码分析

主函数：

```java
main{
  AtomicInteger number = new AtomicInteger(2020);
  number.getAndIncrement();
}
```

可以看到调用了`getAndIncrement()`方法，如下：

```java
public class AtomicInteger{
  public final int getAndIncrement() {
    //this：代表当前对象
    //valueOffset：内存偏移量
    //1：自增
    return unsafe.getAndAddInt(this, valueOffset, 1);
  }
}
```

这个方法又调用了Unsafe类的一个方法`getAndAddInt()`方法，如下：

```java
public final class Unsafe {
  
  //先获取，再加
  //var1:this
  //var2:内存地址偏移量
  //var4:1
  public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
      //获取当前这个对象这个地址上的值
      var5 = this.getIntVolatile(var1, var2);
    }
    //比较并交换
    //var1:this，var2:内存地址偏移量，var5:当前值，var5+var4:新值
    //修改成功跳出循环；不成功就返回false，一直循环
    while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
    return var5;
  }
  
  //获取这个值的方法
  public native int getIntVolatile(Object var1, long var2);
  
  //处理器 CAS 指令
  //var1和var2：要更新的变量
  //var4：旧的预期值
  //var5：新的值
  public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
}
```

### 例二

假设线程A和线程B同时执行`getAndAddInt()`操作（分别跑在不同的CPU上）：

- AutomicInteger里面的value原始值是3，即主内存中AutomicInteger的value为3，根据JMM模型，线程A和线程B各自持有一份值为3的value的副本分别到各自的工作内存
- 线程A通过`getIntVolatile(var1,var2)`拿到value值为3，这时线程A被挂起
- 线程B通过`getIntVolatile(var1,var2)`拿到value值为3，刚好B<span style="background: yellow;">没有被挂起</span>，并执行`compareAndSwapInt`方法，比较内存值也是3，成功修改内存值为4，线程B收工
- 这时A恢复，执行`compareAndSwapInt`方法，发现自己的值和主内存的值不一致，说明这个值已经被修改过了，那A修改失败，<span style="background: yellow;">只能重新读取</span>
- 线程A重新获取value值，因为变量value被volatile修饰，所以其他线程对它的修改，线程A总是可以看见，线程A继续执行`compareAndSwapInt`方法进行比较替换，直到成功

**<span style="background: yellow;">Unsafe类+CAS思想（自旋）</span>**

## CAS缺点

**1、循环时间长，开销大**

这个方法的do while，如果CAS失败，会一直尝试，如果长时间不成功会带来很大的开销

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
      var5 = this.getIntVolatile(var1, var2);
    }while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
    return var5;
  }
```

**2、只能保证一个共享变量的原子操作**

对于多个共享变量操作，循环CAS无法保证一致性，只能使用加锁

**3、ABA问题**

### ABA问题

> 原子类AutomicInteger的ABA问题？原子更新引用知道吗？

ABA问题：狸猫换太子

CAS算法实现一个重要前提：取出内存中某时刻的数据并在当下时刻比较并替换，那么这个时间差会导致数据的变化。

**1、ABA问题如何产生的？**

比如：一个线程1从内存位置V取出A，这时候另外一个线程2也从内存中取出A，并且线程2进行了一些操作将值变成了B，然后线程2又将V位置的数据变回了A；这时候线程1进行CAS操作，发现内存中仍然是A，线程1操作成功。

<span style="background: yellow;">虽然线程1的CAS操作成功，但是这不代表这个过程没有问题</span>

#### 原子引用

Atomic：原子

在`java.util.concurrent.atomic`下有这些类：

<img src="https://img-blog.csdnimg.cn/20210319202553593.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom: 50%;" />

其中，`AtomicReference<V>`为原子引用：

<img src="https://img-blog.csdnimg.cn/20210319202821686.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:33%;" />

例子：

```java
class User{
  String userName;
  int age;
}
main{
  User u1 = new User("zhangsan",18);
  User u2 = new User("lisi",20);
  
  AtomicReference<User> at = new AtomicReference<>();
  at.set(u1);
  
  System.out.println(at.compareAndSet(u1,u2));//true
  System.out.println(at.compareAndSet(u1,u2));//false
}
```

### ABA解决

原子引用+新增机制（修改版本号，类似于时间戳）

```java
线程			 工作内存			1s 				 2s					
线程1				100	1								2019 2

线程2				100	1			101 2			102 3
```

**时间戳原子引用类：**



<img src="https://img-blog.csdnimg.cn/20210319203431276.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

一个 `AtomicStampedReference`保持随着整数“邮票”一个对象的引用，可以自动更新。 

实现说明：此实现通过创建表示“框”[参考，整数]对的内部对象来保持标记的引用。

**对比：**

1、 `AtomicReference`产生ABA问题

```java
static AtomicReference<Integer> a1 = new AtomicReference<>(100);
public static void main(String[] args) throws InterruptedException {
  new Thread(()->{
    a1.compareAndSet(100,101);
    a1.compareAndSet(101,100);
  },"t1").start();

  new Thread(()->{
    //暂停1s，保证t1线程完成了一次ABA操作
    try {
      TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println(a1.compareAndSet(100, 2019)+"\t"+a1.get());
  },"t2").start();
}
```

输出：

```java
true	2019
```

2、使用 `AtomicStampedReference`解决问题

```java
static AtomicStampedReference<Integer> a2 = new AtomicStampedReference<>(100,1);
public static void main(String[] args) throws InterruptedException {
  new Thread(()->{
    //获得初始版本号
    int stamp = a2.getStamp();
    System.out.println(Thread.currentThread().getName()+"\t第一次版本号："+stamp);
    //暂停1s,保证t2获取的版本号和t1一致
    try {
      TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    a2.compareAndSet(100,101,a2.getStamp(),a2.getStamp()+1);
    System.out.println(Thread.currentThread().getName()+"\t第二次版本号："+a2.getStamp());
    a2.compareAndSet(101,100,a2.getStamp(),a2.getStamp()+1);
    System.out.println(Thread.currentThread().getName()+"\t第三次版本号："+a2.getStamp());
  },"t1").start();

  new Thread(()->{
    //获得初始版本号
    int stamp = a2.getStamp();
    System.out.println(Thread.currentThread().getName()+"\t第一次版本号："+stamp);
    //暂停3s，保证t3完成一次ABA操作
    try {
      TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println(a2.compareAndSet(100, 2019,stamp,stamp+1));
  },"t2").start();
}
```

输出：

```java
t1	第一次版本号：1
t2	第一次版本号：1
t1	第二次版本号：2
t1	第三次版本号：3
false
```

