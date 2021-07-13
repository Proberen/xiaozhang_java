# ThreadLocal

## 概述

ThreadLocal类用来提供线程内部的局部变量，不同线程之间不会相互干扰，这种变量在线程的生命周期内起作用，减少同一个线程内多个函数或组件之间一些公共变量传递的复杂度。

线程隔离

## 基本使用

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210328083051319.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

```java

public static void main(String[] args) throws ExecutionException, InterruptedException {
  get g = new get();
  for(int i=0;i<5;i++){
    new Thread(()->{
      g.setA(Thread.currentThread().getName()+"set");
      System.out.println(Thread.currentThread().getName()+"----"+g.getA());
    }).start();
  }
}

class get{
  ThreadLocal<String> t = new ThreadLocal();
  String a;

  public String getA() {
    return t.get();
  }

  public void setA(String a) {
    t.set(a);
  }
}
```

输出：

```java
Thread-0----Thread-0set
Thread-4----Thread-4set
Thread-3----Thread-3set
Thread-2----Thread-2set
Thread-1----Thread-1set
```

## 与synchronized的区别

ThreadLocal和synchronized都是用来处理多线程并发访问变量的问题

**1、synchronized**

- 原理：以时间换空间，只提供了一个变量，让不同的线程排队访问
- 侧重点：多个线程之间访问资源的同步



**2、ThreadLocal**

- 原理：以空间换时间，每一个线程都有一份变量的副本，从而实现同时访问不干扰
- 侧重点：多线程中让每个线程直接的数据相互隔离

**总结：在上述案例中虽然使用两者都可以解决问题，但是使用ThreadLocal会有更高的并发性**

## 运用场景

转账案例

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210328090928555.png)

## 内部结构

在JDK8中，每个Thread维护一个ThreadLocalMap，这个Map的key是ThreadLocal本身，value是真正要存储的值Object

<img src="https://img-blog.csdnimg.cn/2021032809125924.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

### 对比

<img src="https://img-blog.csdnimg.cn/20210328091329488.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

JDK8的设计方案好处：

- 每个Map存储的Entry数量变少（之前Entry的数量是由Thread数量决定的，现在是由ThreadLocal数量决定的，尽量避免哈希冲突的发生）
- 当Thread销毁的时候，ThreadLocalMap也会销毁，减少内存的使用

## 核心方法源码

### set方法

```java
public void set(T value) {
  //获取当前线程对象
  Thread t = Thread.currentThread();
  //获取当前线程对象中维护的ThreadLocalMap
  ThreadLocalMap map = getMap(t);
  //判断Map是否存在
  if (map != null)
    //存在就调用map.set方法设置此实体Entry
    map.set(this, value);
  else
    //不存在就进行ThreadLocalMap的初始化
    createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
  //Thread类：ThreadLocal.ThreadLocalMap threadLocals = null;
  return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
  t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

### get方法

```java
public T get() {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null) {
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  //情况1:map不存在
  //情况2:map存在，但是不存在与当前ThreadLocal关联的entry
  return setInitialValue();
}

private T setInitialValue() {
  //调用initialValue获取初始化的值
  T value = initialValue();
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
  return value;
}
```

### remove方法

```java
public void remove() {
  ThreadLocalMap m = getMap(Thread.currentThread());
  if (m != null)
    m.remove(this);
}
```

流程：

- 获取当前线程，根据当前线程获取一个Map
- 获取的Map不为空就移除对应的entry

### initialValue方法

```java
protected T initialValue() {
    return null;
}
```

返回该线程局部变量的初始值

- 这个方法是一个延迟调用方法，set方法没有调用就调用了get方法才会执行，并且只执行一次
- 这个方法缺省值返回null
- 如果想要一个除null之外的初始值，可以重写此方法

## ThreadLocalMap源码

**没有实现Map接口，内部的Entry也是独立实现的**

<img src="/Users/zhangtao/Desktop/截屏2021-03-28 上午9.34.28.png" alt="截屏2021-03-28 上午9.34.28" style="zoom:50%;" />

### 成员变量

```java
//初始容量，必须是2的次幂
private static final int INITIAL_CAPACITY = 16;
//存放数据的table
private Entry[] table;
//数组里面entry的个数，用来判断是否超过阈值
private int size = 0;
//进行扩容的阈值
private int threshold; // Default to 0
```

### 存储结构

```java
//继承WeakReference，用ThreadLocal作为key，key是弱引用，目的是将ThreadLocal对象的生命周期和线程生命周期解绑
static class Entry extends WeakReference<ThreadLocal<?>> {
  Object value;

  Entry(ThreadLocal<?> k, Object v) {
    super(k);
    value = v;
  }
}
```

### 弱引用和内存泄漏

**1、内存泄露**

- Memory overflow：内存溢出，没有足够的内存供申请者使用
- Memory leak：内存泄漏，已动态分配的堆内存因为某种原因无法释放，造成内存浪费

**2、弱引用**

Java的引用有4种类型：强、软、弱、虚

- **强引用：**最常见的普通对象引用，垃圾回收器不会回收这种对象
- **弱引用：**垃圾回收器一旦发现只具有弱引用的对象，不管内存是否足够，都会回收它的内存

**3、如果key使用强引用**

如果ThreadLocalMap的key使用强引用，会出现内存泄露吗？

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210328094407827.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

4、如果key使用弱引用

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210328094927312.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210328095431367.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

### hash冲突的解决

threadLocal的set方法

```java
public void set(T value) {
  //获取当前线程对象
  Thread t = Thread.currentThread();
  //获取当前线程对象中维护的ThreadLocalMap
  ThreadLocalMap map = getMap(t);
  //判断Map是否存在
  if (map != null)
    //存在就调用map.set方法设置此实体Entry
    map.set(this, value);
  else
    //不存在就进行ThreadLocalMap的初始化
    createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
  //Thread类：ThreadLocal.ThreadLocalMap threadLocals = null;
  return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
  t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

首先获得当前线程，根据线程获取map

获取的map不为空，则调用threadlocalmap的set方法

获取的map为空，则调用threadlocalmap的构造方法

#### 构造方法

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
  //默认容量16
  table = new Entry[INITIAL_CAPACITY];
  //计算索引
  int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
  //设置值
  table[i] = new Entry(firstKey, firstValue);
  size = 1;
  //设置阈值
  setThreshold(INITIAL_CAPACITY);
}
```

阈值是初始容量的2/3

```java
private final int threadLocalHashCode = nextHashCode();

private static int nextHashCode() {
  return nextHashCode.getAndAdd(HASH_INCREMENT);
}

private static AtomicInteger nextHashCode = new AtomicInteger();
//与斐波那契数列有关，尽量避免哈希冲突
private static final int HASH_INCREMENT = 0x61c88647;

//AtomicInteger的一个方法
public final int getAndAdd(int delta) {
  return unsafe.getAndAddInt(this, valueOffset, delta);
}
```

#### set方法

```java
private void set(ThreadLocal<?> key, Object value) {
  Entry[] tab = table;
  int len = tab.length;
  int i = key.threadLocalHashCode & (len-1);

  //线性探测法
  for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
    //获得key
    ThreadLocal<?> k = e.get();
    //如果key存在且相同
    if (k == key) {
      //覆盖
      e.value = value;
      return;
    }
    //如果key是空的，value不为空，说明之前的对象已经被回收了
    if (k == null) {
      //替换，包括垃圾清理动作，防止内存泄漏
      replaceStaleEntry(key, value, i);
      return;
    }
  }
  
  //如果没有遍历成功，就创建新值 
  tab[i] = new Entry(key, value);
  int sz = ++size;
  if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
}

private void rehash() {
  expungeStaleEntries();
  //如果size大于3/4的阈值，扩容
  if (size >= threshold - threshold / 4)
    resize();
}


private void resize() {
  Entry[] oldTab = table;
  int oldLen = oldTab.length;
  //扩容两倍
  int newLen = oldLen * 2;
  Entry[] newTab = new Entry[newLen];
  int count = 0;
  //遍历旧数组
  for (int j = 0; j < oldLen; ++j) {
    Entry e = oldTab[j];
    if (e != null) {
      ThreadLocal<?> k = e.get();
      if (k == null) {
        e.value = null; // Help the GC
      } else {
        int h = k.threadLocalHashCode & (newLen - 1);
        while (newTab[h] != null)
          h = nextIndex(h, newLen);
        newTab[h] = e;
        count++;
      }
    }
  }

  setThreshold(newLen);
  size = count;
  table = newTab;
}
```

**使用信息探测法解决哈希冲突：**

该方法一次探测下一个地址，直到有空的地址后插入，如果整个空间都没有空余地址，则溢出

假设当前table为16，key计算出的hash值为14，如果位置上已经有值，并且key与当前key不一样，就发生了哈希冲突，这时候将14加1得到15，判断15的位置，如果有冲突就会回到0，以此类推，直到可以插入