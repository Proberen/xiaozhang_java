# Fork/Join框架

> 什么是ForkJoin？

JDK1.7提出，用于并行执行任务，提高效率，大数据量

==把大任务分割成若干个小任务，最终汇总每个小任务的结果后得到大任务结果的框架==

- Fork：把大任务切分为若干个字任务并行的执行
- Join：合并这些子任务的执行结果

## 工作窃取
> ForkJoin的特点：工作窃取（work-stealing）

1、概念：指某个线程从其他队列里窃取任务来执行

2、比如：A线程负责A队列的任务，B线程的任务已经结束后去A队列中窃取一个任务来执行，这时访问同一个队列，因此为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用==双端队列==，被窃取的任务线程永远从头部拿任务，窃取任务的线程从尾部拿线程

3、优点和缺点
- 优点：充分利用线程进行并行计算，减少了线程之间的竞争
- 缺点：在某些情况下还是存在竞争，比如双端队列只有一个任务时。并且该算法消耗大量的稀土资源，比如创建多个线程、创建多个双端队列

## Fork/Join框架设计
步骤1：分割任务

步骤2：执行任务并合并结果

使用两个类来完成以上两件事情：
- `ForkJoinTask`：使用Fork/join框架，必须先创建一个Fork/Join任务。它提供在任务中执行fork和join操作的机制。通常情况下，我们只需要继承它的子类:
`RecursiveAction`：用于没有返回结果的任务
`RecursiveTask`：用于有返回结果的任务
- `ForkJoinPool`：`ForkJoinTask`需要通过`ForkJoinPool`来执行
## 使用Fork/Join

```java
public class add extends RecursiveTask<Long>{
    private long start;
    private long end;

    //临界值
    private long temp = 10000L;

    public add(long start, long end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        long sum=0L;
        if((end-start)<temp){
            for(long i=start;i<=end;i++){
                sum+=i;
            }
        }else {
            //forkjoin
            long middle = (start+end)/2;
            add task1 = new add(start,middle);
            add task2 = new add(middle+1,end);
            //执行子任务
            task1.fork();
            task2.fork();
            //等待结果
            long result1 = task1.join();
            long result2 = task2.join();
            sum = result1+result2;
        }
        return sum;
    }

    public static void main(String[] args) {
        //1、ForkJoinPool：通过它来执行
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        //2、生产一个计算任务
        add test = new add(1,12345);
        //3、执行一个任务
        Future<Long> result = forkJoinPool.submit(test);
        try {
            System.out.println(result.get());
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```
## 框架的实现原理
`ForkJoinPool`由`ForkJoinTask`数组和`ForkJoinWorkerThread`数组组成：
- `ForkJoinTask`数组：负责将存放程序==提交==给`ForkJoinPool`任务
- `ForkJoinWorkerThread`数组负责==执行==这些任务

1、`ForkJoinTask`的fork方法实现原理



```java
public final ForkJoinTask<V> fork() {
    Thread t;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        ((ForkJoinWorkerThread)t).workQueue.push(this);
    else
        ForkJoinPool.common.externalPush(this);
    return this;
}
```

2、`ForkJoinTask`的join方法实现原理

join方法主要作用：阻塞当前线程并等待获取结果

```java
public final V join() {
        int s;
        if ((s = doJoin() & DONE_MASK) != NORMAL)
            reportException(s);
        return getRawResult();
    }
    
private void reportException(int s) {
    if (s == CANCELLED)
        throw new CancellationException();
    if (s == EXCEPTIONAL)
        rethrow(getThrowableException());
}
```
1、首先使用`dojoin()`方法得到当前任务的状态
- 已完成（NORMAL）：直接返回任务结果
- 被取消（CANCELLED）：抛出异常
- 信号（SIGNAL）
- 出现异常（EXCEPTIONAL）：抛出相应的异常

```java
private int doJoin() {
    int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
    return (s = status) < 0 ? s :
        ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
        (w = (wt = (ForkJoinWorkerThread)t).workQueue).
        tryUnpush(this) && (s = doExec()) < 0 ? s :
        wt.pool.awaitJoin(w, this, 0L) :
        externalAwaitDone();
}
```

查看任务状态，看任务是否已经完成，如果完成则返回任务状态；如果没有完成，则从任务数组中取出任务并执行。

如果任务顺利完成，状态为NORMAL；如果出现异常则记录异常，状态为EXCEPTIONAL
