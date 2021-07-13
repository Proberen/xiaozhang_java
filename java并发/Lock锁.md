# Lock锁
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210317200101446.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)
1、格式：
```java
 Lock l = ...;
 //加锁
 l.lock();
 try {
   // access the resource protected by this lock
 } finally {
 	//解锁
   l.unlock();
 }
```
2、实现类：
- `ReentrantLock`：可重入锁
- `ReentrantReadWriteLock.ReadLock`
- `ReentrantReadWriteLock.WriteLock`

3、ReentrantLock

构造方法：
```java
public ReentrantLock() {
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
	sync = fair ? new FairSync() : new NonfairSync();
}
```
- 公平锁：十分公平，先来后到
- **非公平锁：不公平，可以插队（默认）**

例子：
```java
class ticket{
	private int number = 30;
	//创建锁
	Lock lock = new ReentrantLock();

	public void sale(){
		//加锁
		lock.lock();
		//尝试获取锁
		lock.tryLock();
		try{
			.....
		}catch(Exception e){
		}finally{
			//解锁
			lock.unlock();	
		}
	}
}
```
## Synchronized和Lock区别

1、原始构成

- `Synchronized` 是内置的Java关键字，属于JVM层面，底层通过monitor对象完成，其实wait/notify也依赖于monitor对象

- `Lock` 是一个Java类，是api层面的锁

2、`Synchronized` 无法判断获取锁的状态，`Lock` 可以判断是否取到了锁

3、`Synchronized` 会自动释放锁，`Lock` 必须要手动释放锁（不释放会死锁）

4、`Synchronized` 线程1（获得锁，阻塞）线程2（等待）；`Lock`锁不一定会等待下去，线程可以不用一直等待就结束了；

5、`Synchronized` 可重入锁，不可以中断的，非公平的；`Lock` 可重入锁，可以判断锁，可以自己设置公不公平

6、`Synchronized` 适合锁少量的代码同步问题，`Lock`适合大量的代码块同步

## ReentrantLock

**聚合关系总结**:

1. ReentrantLock实现了Lock,Serializable接口
2. ReentrantLock.Sync(内部类)继承了AQS
3. ReentrantLock.NonfairSync和ReentrantLock.FairSync继承了ReentrantLock.Sync
4. ReentrantLock持有ReentrantLock.Sync对象(实现锁功能)

**锁实现总结**:

1. 由Node节点组成一条同步队列(有head,tail两个指针,并且**head初始化时指向空节点**)
2. int state标记锁使用数量(独占锁时,通常为1,发生重入时>1)
3. lock()时加到队列尾部
4. unlock()时,释放head节点,并指向下一个节点head=head.next,然后唤醒当前head节点

### 构造方法

```java
//默认非公平锁
public ReentrantLock() {
  sync = new NonfairSync();
}

//设置为公平锁
public ReentrantLock(boolean fair) {
  sync = fair ? new FairSync() : new NonfairSync();
}
```

### lock方法

```java
public void lock() {
  sync.lock();
}

//sync内部类
abstract void lock();
```

分为公平锁、非公平锁

### unlock方法

```java
public void unlock() {
    sync.release(1);
}
```

### tryLock方法

```java
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}
```

