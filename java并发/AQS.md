# AQS

## LockSupport

> `LockSupport`是用来创建锁和其他同步类的基本线程阻塞原语。

1、线程阻塞工具类，都是静态方法，调用底层native代码

2、LockSupport和每一个使用它的线程都有一个许可关联，permit相当于1、0开关，默认是0

- 调用unpark变为1
- 调用park变为0，通过park立即返回
- 如果再次调用park会阻塞，直到permit变为1
- 每个线程都只有一个相关的permit，最多一个，重复调用unpark不会积累凭证

### park

```java
public static void park() {
  UNSAFE.park(false, 0L);
}
```

permit默认为0，所以一开始调用park方法，当前线程就会阻塞，直到别的线程将当前线程的permit设置为1，park会被唤醒，然后将permit设置为0并返回

### unpark

```java
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

### 例子

```java
public static void main(String[] args) throws InterruptedException {
  Thread a = new Thread(()->{
    System.out.println(Thread.currentThread().getName()+"-> "+"come in");
    LockSupport.park();
    System.out.println(Thread.currentThread().getName()+"-> 被唤醒");
  });
  a.start();

  TimeUnit.SECONDS.sleep(3);

  new Thread(()->{
    LockSupport.unpark(a);
    System.out.println(Thread.currentThread().getName()+"-> 通知");
  }).start();
}
```

## AQS

> AQS是JUC框架中重要的类，通过它来实现独占锁和共享锁的，内部很多类都是通过AQS来实现的，比如CountDownLatch、ReentrantLock、ReentrantReadWriteLock、Semaphore。

AQS：抽象的队列同步器（AbstractQueuedSynchronized）

是用来构建锁或者其他同步器组件的重量级基础框架及整个JUC体系的基石，通过内置的**FIFO队列**来完成资源获取线程的排队工作，并通过一个**int类型变量**表示持有锁的状态

<img src="https://img-blog.csdnimg.cn/20210327175333720.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />



用处：加锁会导致堵塞，有阻塞就需要排队，实现排队就需要有某种形式的队列进行管理

**CLH队列：将请求共享资源的线程封装为队列的结点Node，通过CAS、自旋、LockSupport.park的方式维护state的状态，使并发达到同步的控制效果**

### AQS源码体系

<img src="https://img-blog.csdnimg.cn/20210327195233424.png" alt="在这里插入图片描述" style="zoom:50%;" />



内部体系架构：

<img src="https://img-blog.csdnimg.cn/20210327200931838.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021032720314341.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

**1、AQS自身：**

- int变量：state（1表示被占用，只能去排队）
- CLH队列：双向队列（有首节点、尾节点）

**2、内部类Node**

- 有两个模式：共享、排它
- 有前节点和后节点（pre、next）
- **有一个重要的int变量：waitStatus**，表示当前节点在队列中的状态
  - 0：默认
  - 1：表示线程获取锁的请求取消
  - -2：表示节点在等待队列中等待被唤醒
  - -3：当前线程在共享情况下
  - -1：表示线程已经准备好了，就等待释放资源了

```java
static final class Node {
  //共享
  static final Node SHARED = new Node();
  //独占
  static final Node EXCLUSIVE = null;
 
  static final int CANCELLED =  1;
  static final int SIGNAL    = -1;
  static final int CONDITION = -2;
  static final int PROPAGATE = -3;

  // 当前节点在队列中的状态
  // 默认0
  // 1：取消
  // -1：线程准备好了
  // -2：等待唤醒
  // -3：当前线程在共享情况下
  volatile int waitStatus;
  //前指针
  volatile Node prev;
  //后指针
  volatile Node next;
  //线程
  volatile Thread thread;

  Node nextWaiter;

  final boolean isShared() {
    return nextWaiter == SHARED;
  }

  final Node predecessor() throws NullPointerException {
    Node p = prev;
    if (p == null)
      throw new NullPointerException();
    else
      return p;
  }

  //构造方法
  Node() {}

  Node(Thread thread, Node mode) {   
    this.nextWaiter = mode;
    this.thread = thread;
  }

  Node(Thread thread, int waitStatus) { 
    this.waitStatus = waitStatus;
    this.thread = thread;
  }
}
```

### 源码

> Lock接口的实现了基本上都是通过聚合了一个队列同步器的子类完成线程的访问控制的

**ReentrantLock:**

<img src="https://img-blog.csdnimg.cn/20210414155640974.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />![在这里插入图片描述](https://img-blog.csdnimg.cn/20210414155726983.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)



**公平锁和非公平锁**

区别：公平锁需要判断队列中是否存在有效节点

- 公平锁讲究先来先到，如果这个锁的等待队列中已经有线程在等待，那么当前线程就会进入等待队列
- 非公平锁如果可以获取锁，就立刻占有

```java
//公平锁

final void lock() {
  acquire(1);
}

//非公平锁
final void lock() {
  if (compareAndSetState(0, 1))
    setExclusiveOwnerThread(Thread.currentThread());
  else
    acquire(1);
}

//最终都会调用acquire方法
public final void acquire(int arg) {
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

### lock

加锁三个阶段：

- 尝试加锁
- 加锁失败，进入队列
- 线程入队列后，进入阻塞状态

```java
//main方法代码
ReentrantLock lock = new ReentrantLock();
lock.lock();
```

```java
//ReentrankLock类：lock方法
public void lock() {
  sync.lock();
}

//ReentrankLock类中的sync内部类的抽象方法：lock
abstract void lock();

//ReentrankLock类中的非公平锁内部类的方法：lock
final void lock() {
  //CAS：更改AQS的state
  //如果state为0，就将state设置为1
  if (compareAndSetState(0, 1))
    //设置队列的当前线程为这个线程
    setExclusiveOwnerThread(Thread.currentThread());
  else
    //如果CAS失败，说明已经有线程抢占了锁，那么就需要进入acquire方法
    acquire(1);
}
```

#### acuqire

```java
//AQS类的acquire方法：传入的参数为1，arg为1
public final void acquire(int arg) {
  if (!tryAcquire(arg) && //返回false，取反为true；如果返回true（重入），因为是短路与，所以后面就不会进行
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

#### tryAcquire—抢占、重入

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210414155726983.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

```java
//AQS的方法
protected boolean tryAcquire(int arg) {
  throw new UnsupportedOperationException();
}

//ReentrankLock类中的非公平锁内部类的方法：tryAcquire
protected final boolean tryAcquire(int acquires) {
  return nonfairTryAcquire(acquires);
}

//ReentrankLock类中的sync内部类的方法，acquires为1
final boolean nonfairTryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  //获取state
  int c = getState();
  //如果state为0，直接AQS，抢占锁
  if (c == 0) {
    if (compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  //如果请求的线程和当前的线程相同，说明重入
  else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0) // overflow
      throw new Error("Maximum lock count exceeded");
    //设置state为重入的次数
    setState(nextc);
    //返回true
    return true;
  }
  //如果都不是，就返回false
  return false;
}
```

> ReentrantLock重入的原理

在第二次获取锁的时候，CAS失败，进入tryacquire方法，会进行判断请求的线程和当前拥有资源的线程是否相同，如果相同，state+1

#### addwaiter—创建node并加入到队列

```java
private Node addWaiter(Node mode) {
  //创建一个新的node
  Node node = new Node(Thread.currentThread(), mode);
  // 获取tail，如果是第一个线程，就为null
  Node pred = tail;
  // 如果tail不是null
  if (pred != null) {
    //CAS交换
    node.prev = pred;
    if (compareAndSetTail(pred, node)) {
      pred.next = node;
      return node;
    }
  }
  enq(node);
  //返回线程
  return node;
}

//Node的enq方法
private Node enq(final Node node) {
  //自旋
  for (;;) {
    Node t = tail;
    //如果tail为null，就进行初始化
    if (t == null) {
      //使用CAS进行初始化，new一个空node作为头节点（Thread=null，waitStatus=0）：傀儡节点/哨兵节点，只是用来占位
      if (compareAndSetHead(new Node()))
        tail = head;
    } else {
      //如果tail不为null，将传入的需要增加的节点加入到队列中
      node.prev = t;
      //CAS设置tail为需要增加的节点
      if (compareAndSetTail(t, node)) {
        //设置t的next为需要增加的节点
        t.next = node;
        //返回t
        return t;
      }
    }
  }
}
```

步骤：

1、addWriter方法会首先新建这个node

2、如果当前队列不为空，就将自己放到尾节点，去6；

3、如果当前队列为空，就需要进入Node的enq方法；去4；

4、再次判断队列是否为空，如果为空就进行初始化（傀儡节点），等待队列的第一个节点是傀儡节点

5、初始化后，tail不为null了，就可以使用CAS传入线程（同2）

6、返回这个节点

<img src="https://img-blog.csdnimg.cn/20210414170752692.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

#### acquireQueued—使用locksupport阻塞节点

```java
//node：插入的新node
//arg：1
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    //被打断
    boolean interrupted = false;

    for (;;) {
      //获得上一个节点
      final Node p = node.predecessor();
      //如果p=head，再抢一次（看看state是不是0）
      if (p == head && tryAcquire(arg)) {
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
      }
      //抢占失败，进入shouldParkAfterFailedAcquire方法，将上一个节点的waitstatus变为-1
      //再次进入，进入parkAndCheckInterrupt方法，使用locksupport挂起当前节点，直到unpark才结束
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}


private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  //默认为0
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)
    return true;
  if (ws > 0) {
    do {
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
  } else {
    //更改上一个节点的waitStatus变为-1
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}

private final boolean parkAndCheckInterrupt() {
  LockSupport.park(this);
  return Thread.interrupted();
}
```

步骤：

- 将上一个节点的waitstatus改为-1
- 使用locksupport将自己进行阻塞
- **后面的线程都在park，在parkAndCheckInterrupt()方法这里等待unpark()**

### unlock

```java
public void unlock() {
    sync.release(1);
}
```

#### release

```java
//AQS类
public final boolean release(int arg) {
  //如果是true，说明state=0
  if (tryRelease(arg)) {
    //获取head
    Node h = head;
    //如果head不为null，waitstatus不等于0
    if (h != null && h.waitStatus != 0)
      unparkSuccessor(h);
    return true;
  }
  return false;
}
```

#### tryRelease—state减1，为0free就是true

```java
protected boolean tryRelease(int arg) {
  throw new UnsupportedOperationException();
}

//ReentrankLock的内部类
protected final boolean tryRelease(int releases) {
  //当前状态-1
  int c = getState() - releases;
  if (Thread.currentThread() != getExclusiveOwnerThread())
    throw new IllegalMonitorStateException();
  boolean free = false;
  //如果c=0
  if (c == 0) {
    free = true;
    //设置当前thread为null
    setExclusiveOwnerThread(null);
  }
  setState(c);
  return free;
}
```

```java
protected final void setState(int newState) {
    state = newState;
}
```

#### unparkSuccessor

```java
private void unparkSuccessor(Node node) {
  //获取waitStatus
  int ws = node.waitStatus;
  if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);
  //获取下一个节点
  Node s = node.next;
  //如果下一个节点等于null且waitstatus大于0
  if (s == null || s.waitStatus > 0) {
    s = null;
    for (Node t = tail; t != null && t != node; t = t.prev)
      if (t.waitStatus <= 0)
        s = t;
  }
  //如果下一个节点不等于null
  if (s != null)
    //唤醒下一个节点
    LockSupport.unpark(s.thread);
}
```

### 总结

**1、AQS的加锁操作**

- 首先调用lock方法

- lock方法调用了sync的lock方法

- sync的lock方法调用了非公平锁的lock方法，使用CAS更改state状态

  - 更改成功，就将当前占有锁的线程更改为这个线程

  - **更改不成功（说明没有线程占有锁）进入AQS类的`acquire`方法**

    ```java
    public final void acquire(int arg) {
      if (!tryAcquire(arg) && //返回false，取反为true；如果返回true（重入），因为是短路与，所以后面就不会进行
          acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
    }
    ```

- 首先进入ReentrankLock类中非公平锁的`tryAcquire`方法，接着进入sync的`nonfairTryAcquire`方法
  - **使用CAS转换state，转换成功就占有锁，返回true，整个流程结束**
  - 转换不成功，**如果两个线程相同，说明是重入，那么就state加1，返回true，整个流程结束**
  - 以上情况都不是，返回false，进入`addWaite`方法

- 接着进入`addWaiter`方法
  - 根据线程创建新的节点
  - 如果tail不是null，使用CAS转换，将当前线程设置为tail，返回这个节点
  - 如果tail为null，进入Node的`enq`方法
    - 再次判断队列是否为空，如果为空就进行初始化（**傀儡节点**），等待队列的第一个节点是傀儡节点
    - 初始化后，tail不为null了，就可以使用CAS将这个线程转为tail
    - 返回这个节点

- 再进入`acquireQueued`方法
  - 将上一个节点的`waitstatus`改为-1（傀儡节点，除了初始化的傀儡节点，获得锁的线程node会将node的thread设置为null，也是傀儡节点）
  - 使用`locksupport`将自己进行阻塞（park）
  - **后面的线程都在park，在parkAndCheckInterrupt()方法这里等待unpark()**

**2、AQS的unlock**

- 进入sync的`release`方法
- 首先进行`tryRelease`方法
  - 将state减1，如果为0，就返回true
  - **如果不为0，说明是重入了，返回false，流程结束**

- 如果不是重入，进入`unparkSuccessor`方法
  - unpark队列的下一个节点

> 如果实现非公平？

非公平锁相比较公平锁的 `tryAcquire`方法，少了一步**判断 AQS 队列中是否有等待的线程**的操作。

