# callable接口

**1、与Runnable的区别：**

- Runnable没有返回值，callable有返回值
- run方法不会抛异常，call会抛异常

2、Callable接口代表一段可以调用并返回结果的代码;

Future接口表示异步任务，是还没有完成的任务给出的未来结果。

所以说Callable用于产生结果，Future用于获取结果。 

**3、FutureTask**

```java
public class FutureTask<V> implements RunnableFuture<V>
public interface RunnableFuture<V> extends Runnable, Future<V> 
```











```java
public class add{
    public static void main(String[] args){
        FutureTask<Integer> futureTask = new FutureTask<>(new MyThread());
        Thread t1 = new Thread(futureTask,"AA");
        t1.start();
    }
}

class MyThread implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        System.out.println("1");
        return 1024;
    }
}
```

