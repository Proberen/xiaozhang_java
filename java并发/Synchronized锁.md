# synchronized锁
阻塞式的解决方案：synchronized、Lock

非阻塞式的解决方案：原子变量

语法：

```java
synchronized(对象){
  临界区
}
```

<img src="https://img-blog.csdnimg.cn/20210319130659428.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

<img src="https://img-blog.csdnimg.cn/20210319130743969.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

1、因此，synchronized是用**对象锁**保证了临界区内代码的**原子性**，临界区内的代码对外是不可分割的，不会被其他线程打断

2、**保证可见性的原理：**执行synchronized时，会对应lock原子操作，会刷新工作内存中共享变量的值

3、有序性：依然会发生重排序，但是可以保证只有一个线程在同步代码块

## 三种使用方式

- **修饰代码块** ：指定加锁对象，对给定对象/类加锁。synchronized(this object) 表示进入同步代码库前要获得给定对象的锁。synchronized(类.class) 表示进入同步代码前要获得当前class的锁

```java
Object o = new Object();
public void test(){
  synchronized(o){
    临界区
  }
}
```

- **修饰实例方法:** 作用于当前对象实例加锁，进入同步代码前要获得<span style="background: yellow;">当前对象实例</span>的锁

```java
// 相当于以下代码
synchronized(this){
  临界区
}
```

- **修饰静态方法**: 也就是给当前类加锁，会作用于类的所有对象实例 ，进入同步代码前要获得<span style="background: yellow;">当前class 的锁</span>。

```java
// 相当于以下代码
class test{
  synchronized(test.class){
    临界区
  }
}
```

## 八锁问题

关于锁的8个问题：

 **1、标准情况下，两个线程都打印输出（中间sleep 1s），哪个先打印？**

第一个线程先打印，因为锁的存在，两个方法用的是同一个锁，谁先拿到谁执行

**2、第一个线程的锁方法延迟4s，哪个先？**

第一个线程先打印

 **3、增加一个普通方法后，先执行普通方法还是锁方法（sleep 4s）？**

先执行普通方法，不受锁的影响

 **4、两个对象，先打印哪个？**

先打印没有延迟的那个

 **5、两个静态的同步方法，一个对象，先打印哪一个？**

先打印第一个线程

 **6、两个静态的同步方法，两个对象，先打印哪一个？**

先打印第一个线程，`static`锁的是class，class全局唯一

 **7、一个静态同步方法（有延迟），一个普通同步方法，一个对象，先打印哪一个？**

 先打印普通同步方法，两个方法锁的不是一个（一个是class，一个是对象）

 **8、一个静态同步方法（有延迟），一个普通同步方法，两个对象，先打印哪一个？**

先打印普通同步方法，两个方法锁的不是一个（一个是class，一个是对象）

**小结：**

new：this 具体的一个对象

static：class 唯一的模版 



## 线程安全问题

### 成员变量和静态变量是否线程安全？

1、如果没有被共享，则线程安全

2、如果被共享了，根据它们的状态能否被改变，又分两种情况：

- 如果只有读操作：线程安全
- 如果**有读写操作**，则这段代码是临界区，需要考虑线程安全

### 局部变量是否线程安全？

1、局部变量是线程安全的

2、局部变量引用的对象不一定线程安全：

- 如果该对象没有逃离方法的作用范围，它是线程安全的
- 如果该**对象逃离方法的作用范围**，需要考虑线程安全

### 局部变量线程安全分析

```java
public static void test1(){
  int i=0;
  i++;
}
```

每个线程调用test1方法时局部变量i，会在每个线程的栈帧内存中被创建多份，因此不存在共享

<img src="https://img-blog.csdnimg.cn/20210319133722305.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />

如图：

<img src="https://img-blog.csdnimg.cn/20210319133749932.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />

**局部变量的引用稍有不同：**

```java
class test1{
  public void method1(int num){
    ArrayList<String> list = new ArrayList<>();
    for(int i=0;i<num;i++){
      method2(list);
      method3(list);
    }
}
  private void method2(ArrayList<String> list){
    list.add("1");
  }
  private void method3(ArrayList<String> list){
    list.remove(0);
  }
}
```



<img src="https://img-blog.csdnimg.cn/20210319134530424.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

同样不存在共享，各不干扰

如果将method改为`public`，出现以下情况：

```java
class test2 extends test1{
  @Override
  public void method3(ArrayList<String> list){
    new Thread(()->{
      list.remove(0);
    }).start();
  }
}
```

这样，两个线程就会访问一个对象，会出现线程安全问题

因此，使用`private`或者`final`提供了安全

### 常见线程安全类

- String
- Integer等包装类
- StringBuffer
- Random
- Vector
- Hashtable
- **java.util.concurrent包下的类**

线程安全是指：多个线程调用它们的同一个实例的某个方法时，是线程安全的。

```java
Hashtable a = new Hashtable();

new Thread(()->{
  a.put(1,2)
}).start();

new Thread(()->{
  a.put(1,2)
}).start();

public synchronized V put(K key, V value) {}
```

也可以说：

- 它们的每一个方法是原子的
- **但注意：它们多个方法的组合不是原子的**

#### 线程安全类方法的组合

分析下面这个代码：

```java
Hashtable a = new Hashtable();
//线程1，线程2
if(a.get("key")==null){
  a.put("key",value);
}
```

<img src="https://img-blog.csdnimg.cn/2021031914040244.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom: 67%;" />

#### 不可变类线程安全性

String、Integer等都是不可变类，因为其内部的状态不可改变，因此是线程安全的



### 实例分析

<img src="https://img-blog.csdnimg.cn/20210319141155990.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />

map：不是线程安全的

s1：线程安全

s2：线程安全

D1：不是线程安全的

D2：不是线程安全的

<img src="https://img-blog.csdnimg.cn/20210319141458804.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />

userService：不是线程安全的，多个线程共享使用

count：不是线程安全的，共享资源



## 特性

### 可重入

一个线程可以多次执行synchronized，重复获取一把锁

原理：**synchronized的锁对象中有一个计数器（recursions变量），会记录线程获得几次锁，在执行完同步代码块时，计数器的数量会减1，计数器数量为0就释放锁**

好处：可以避免死锁、可以让我们更好的封装代码

### 不可中断

一个线程获得锁后另外一个线程想要获得锁必须处于阻塞或者等待状态，如果第一个线程不释放锁，第二个线程会一直阻塞或等待，不可被中断。

synchronized属于不可被中断

Lock的lock方法不可被中断，trylock方法可中断的

## Monitor原理

### 同步代码块底层原理

<img src="https://img-blog.csdnimg.cn/20210319144520377.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />

对应的字节码：

<img src="https://img-blog.csdnimg.cn/20210319144812812.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

 `monitorenter` 指令

- monitor才是真正的锁，JVM会创建一个monitor C++对象
- 两个成员变量
  - owner：拥有锁的线程
  - recursions：记录获取锁的次数（允许重入），为0释放锁

 `monitorexit` 指令

- 能执行这个指令的一定是拥有当前对象锁的线程
- **为什么会有两个monitorexit？**
  - 出现异常会走第二个monitorexit，释放锁

### 同步方法底层原理

同步方法在反汇编后，会增加`ACC_SYNCHRONIZED`修饰，隐式调用monitorenter和monitorexit，执行前调用monitorenter，执行后调用monitorexit



## synchronized和lock的区别

1、synchronized是关键字，lock是一个接口（接口有相应的实现类）

2、synchronized会自动释放锁，lock需要手动释放锁

3、synchronized是不可中断的，lock可以中断（trylock方法）可以不中断（lock方法）

4、lock可以知道线程有没有拿到锁（trylock），synchronized不可以

5、lock可以提高读的效率（读写锁）

6、synchronized是非公平锁，lock可以控制公平还是非公平



## 深入JVM源代码

### monitor监视器锁

<img src="https://img-blog.csdnimg.cn/20210328161829858.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:70%;" />

waitSet：处于wait状态的线程

_cxq：多个线程竞争锁的单向链表（保存没有抢到锁的线程）

entrylist：上一次在cxq等待没有抢到锁的线程

<img src="https://img-blog.csdnimg.cn/20210319143834219.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

### monitor竞争

monitorenter

<img src="https://img-blog.csdnimg.cn/20210328162631713.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />

对于重量级锁，会调用slow_enter

最终调用ObjectMonitor::enter，如下：

<img src="https://img-blog.csdnimg.cn/20210328162936500.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />

<img src="https://img-blog.csdnimg.cn/20210328163017741.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210328163052109.png)

流程如下：

- 通过CAS把monitor设置为当前线程
- 如果设置之前的owner指向当前线程，说明线程重入，执行recursions++
- 如果第一次进入monitor，设置recursions为1，onwer为当前线程，该线程成功获得锁并返回
- 如果获取锁失败，则等待锁的释放

### monitor等待

<img src="https://img-blog.csdnimg.cn/20210329110039318.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

<img src="https://img-blog.csdnimg.cn/20210328163517419.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

<img src="https://img-blog.csdnimg.cn/20210328163700322.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

<img src="https://img-blog.csdnimg.cn/20210328163700322.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

<img src="https://img-blog.csdnimg.cn/20210328163642615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

流程：

- 当前线程封装，状态设置为TS_CXQ
- 在for循环中，通过CAS把节点push到_cxq中，同一时刻可能有多个线程把自己的节点push到 _cxq列表中
- 进入cxq中之后，尝试自旋获取锁，如果还没有获取就挂起，等待被唤醒
- 当该线程被唤醒时，会从刮起的点继续执行，使用trylock尝试获取锁

### monitor释放

1、推出同步代码块会让recursions减1，减为0说明线程释放了锁

2、根据不同的策略，唤醒cxq或者entrylist中的线程

### monitor是重量级锁

**因为用户态和内核态的切换会消耗大量的系统资源**

<img src="https://img-blog.csdnimg.cn/20210328165949320.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

linux操作系统的体系架构分为：用户空间和内核

- 内核：本质上可以理解为一种软件，控制计算机的硬件资源，并提供上层应用程序运行的环境
- 用户空间：上层应用程序活动的空间，必须依托内核提供的资源
- 系统调用：为了使上层应用能够访问到这些资源，内核必须为上层应用提供访问的接口，即系统调用

所有进程初始都运行于用户空间，此时为**用户态**

当调用系统调用执行某些操作时，例如I/O调用，此时需要内核，就称为进程处于**内核运行态**

## JDK6 synchronized优化

### CAS

作用：将比较和交换转换为原子操作，这个原子操作由CPU保证，依赖于三个值：内存中的值V、旧的预估值X，要修改的新值B

Unsafe类：Unsafe对象不能直接调用，只能通过反射获得

这是一种完全依赖于`硬件`的功能，通过它实现了原子操作。原语的执行时连续的，在执行过程中不允许被中断，也就是说CAS是一条CPU的原子指令，不会造成数据不一致。

### 锁升级过程

无锁--》偏向锁--〉轻量级锁--》重量级锁

### Java对象布局

JVM中，对象在内存中的布局分为三块区域：**对象头、实例数据、对齐填充**

**普通对象头**

<img src="https://img-blog.csdnimg.cn/20210319142439473.png" alt="在这里插入图片描述" style="zoom:67%;" />

**数组对象头**

<img src="https://img-blog.csdnimg.cn/20210319142650205.png" alt="在这里插入图片描述" style="zoom:67%;" />

其中，Mark Word的结构为：

<img src="https://img-blog.csdnimg.cn/20210328172012240.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />

偏向锁第三位是1

<hr/>

**klass Word**

<img src="https://img-blog.csdnimg.cn/20210328172201393.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />



### 偏向锁

锁会偏向于第一个获得它的线程，会在对象头存储偏向锁的线程ID，以后该线程进入和退出同步块只需要检查是否是偏向锁、锁标志位、ThreadID即可

**一旦出现多个线程竞争就必须撤销偏向锁**，所以撤销偏向锁消耗的性能必须小于之前节省的CAS原子操作的性能

<img src="https://img-blog.csdnimg.cn/20210328172803293.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

<img src="https://img-blog.csdnimg.cn/20210328173142294.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:67%;" />

**原理：**当锁对象第一次被线程获取，虚拟机会把对象头中的标志位设为01，即偏向模式，同时使用CAS操作把获取的这个线程ID记录在对象头中，如果CAS操作成功，以后这个线程进入和这个锁相关的同步块时，虚拟机不需要进行任何操作

### 轻量级锁

目的：**在多线程交替执行同步块的情况下**，尽量避免重量级锁引起的性能消耗，但是如果多个线程在同一时刻进入临界区，会导致轻量级锁膨胀升级重量级锁

#### 原理

**栈帧：一个进入栈的方法**

关闭了偏向锁或者多个线程竞争偏向锁导致偏向锁升级为轻量级锁，则为尝试获取轻量级锁，步骤如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021032817452319.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)



**轻量级锁是为了在没有竞争的前提下减少重量级锁使用操作系统互斥量产生的性能消耗。**

在代码即将进入同步块时，如果同步对象没有被锁定，虚拟机将在当前线程的栈帧中建立一个锁记录空间，存储锁对象目前 Mark Word 的拷贝。然后虚拟机使用 CAS 尝试把对象的 Mark Word 更新为指向锁记录的指针，如果更新成功即代表该线程拥有了锁，锁标志位将转变为 00，表示处于轻量级锁定状态。

如果更新失败就意味着至少存在一条线程与当前线程竞争。虚拟机检查对象的 Mark Word 是否指向当前线程的栈帧，如果是则说明当前线程已经拥有了锁，直接进入同步块继续执行，否则说明锁对象已经被其他线程抢占。如果出现两条以上线程争用同一个锁，轻量级锁就不再有效，将膨胀为重量级锁，锁标志状态变为 10，此时Mark Word 存储的就是指向重量级锁的指针，后面等待锁的线程也必须阻塞。

解锁同样通过 CAS 进行，如果对象 Mark Word 仍然指向线程的锁记录，就用 CAS 把对象当前的 Mark Word 和线程复制的 Mark Word 替换回来。假如替换成功同步过程就顺利完成了，如果失败则说明有其他线程尝试过获取该锁，就要在释放锁的同时唤醒被挂起的线程。

### 自旋锁

因为执行同步代码块的时间是比较短的，但是第二个线程获取不到锁会立刻阻塞，这是不划算的，为了让线程等待，只需要让这个线程执行自旋，就实现了自旋锁

自旋锁在 JDK1.4 就已引入，默认关闭，在 JDK6 中改为默认开启。

自旋不能代替阻塞，虽然避免了线程切换开销，但要占用处理器时间，如果锁被占用的时间很短，自旋的效果就会非常好，反之只会白白消耗处理器资源。如果自旋超过了限定的次数仍然没有成功获得锁，就应挂起线程，自旋默认限定次数是 10。

#### 适应性自旋锁

JDK6 对自旋锁进行了优化，**自旋时间不再固定**，而是由前一次的自旋时间及锁拥有者的状态决定。

如果在同一个锁上，自旋刚刚成功获得过锁且持有锁的线程正在运行，虚拟机会认为这次自旋也很可能成功，进而允许自旋持续更久。如果自旋很少成功，以后获取锁时将可能直接省略掉自旋，避免浪费处理器资源。

有了自适应自旋，随着程序运行时间的增长，虚拟机对程序锁的状况预测就会越来越精准。

### 锁消除

锁消除指即时编译器对检测到不可能存在共享数据竞争的锁进行消除。

主要判定依据来源于逃逸分析，如果判断**一段代码中堆上的所有数据都只被一个线程访问，就可以当作栈上的数据对待，认为它们是线程私有的而无须同步。**

### 锁粗化

原则需要将同步块的作用范围限制得尽量小，只在共享数据的实际作用域中进行同步，这是为了使等待锁的线程尽快拿到锁。

但如果一系列的连续操作都对同一个对象反复加锁和解锁，甚至加锁操作是出现在循环体之外的，即使没有线程竞争也会导致不必要的性能消耗。因此如果虚拟机探测到有一串零碎的操作都对同一个对象加锁，将会把同步的范围扩展到整个操作序列的外部。





