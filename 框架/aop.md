# AOP面向切面编程

## 动态代理

**1、动态代理**

**程序在整个运行过程中根本就不存在目标类的代理类**，目标对象的代理对象只是由代理生成工具（不是真实定义的类）在程序运行时由JVM根据反射机制动态生成的

**2、作用**

- 在目标类源代码不改变的情况，增加功能
- 减少代码的重复
- 专注业务逻辑代码
- 解耦合，让日志、事务等非业务功能和业务功能分离

### JDK 动态代理

1、实现方式：

- 使用JDK的proxy
- 通过CGLB生成代理

**2、jdk的动态代理要求目标对象必须实现接口，这是java设计上的要求**

3、三个类支持代理模式：

- Proxy
- Method
- InvocationHandler

### 例子

在程序的执行过程中，创建代理对象，通过代理对象执行方法，给目标类的方法增加额外的功能

实现步骤：

- 创建目标类，impl目标类，给它的`doSome`方法、`doOther`方法增加 输出时间、事务
- 创建`InvocationHandler`接口的实现类，在这个类实现给目标方法增加功能
- 使用jdk的类`Proxy`创建代理对象，实现创建对象的能力

```java
public class Tools implements InvocationHandler {
    static void doLog(){
        System.out.println(new Date());
    }
    static void doTrans(){
        //提交事务
        System.out.println("提交事务");
    }

    //目标对象
    //最终是impl类
    private Object target;

    public Tools(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //通过代理对象执行方法时，会调用执行这个invoke方法

        Tools.doLog();

        //执行目标类的方法，通过Method类实现
        //impl.doOther()、doSome()
        Object invoke = method.invoke(target,args);

        Tools.doTrans();

        return invoke;
    }
}
```



```java
public static void main(String[] args) {
    //创建代理对象
    impl target = new impl();

    //创建InvocationHandler对象
    Tools handler = new Tools(target);

    //使用Proxy创建代理
    SomeService proxy = (SomeService) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            handler);

    //通过代理执行方法，会调用handler中的invoke方法
    proxy.doSome();
    proxy.doOther();
}
```

### CGLIB动态代理

**原理：生成目标类的子类，而子类是增强过的，这个子类对象就是代理对象。所以，使用CGLB生成动态代理，要求目标类必须能够被继承，即不能是final类**

第三方工具库，创建代理对象，原理是继承，通过继承目标类创建子类，子类就是代理对象

## AOP概述

1、AOP 面向切面编程：**面向切面编程是从动态角度考虑程序运行过程**

- **切面**：给你的目标类增加的功能就是切面，例如日志、事务、统计信息、参数检查、权限认证等
  - 特点：一般都是非业务方法，可以独立使用

> 怎么理解面向切面编程？

- 需要在分析项目功能时，找出切面

- 合理的安排切面的执行时间（在目标方法前还是目标方法后）

- 合理的安排切面执行的位置，在哪个类，哪个方法增加增强功能

2、AOP 底层就是采用**动态代理模式**实现的，采用了两种代理：JDK 的动态代理、CGLIB 的动态代理

3、AOP是动态代理的规范化，把动态代理的实现步骤、方式都定义好了，让开发人员用一种统一的方式，就用动态代理

## 术语

1、切面（Aspect）

切面泛指交叉业务逻辑。上例中的事务处理、日志处理就可以理解为切面。常用的切面是**通知(Advice)**。实际就是对主业务逻辑的一种增强。

2、**JoinPoint**

连接点，连接业务方法和切面的位置，**就是某类中的业务方法，通常业务接口中的方法均为连接点。**

3、**Pointcut**

切入点，指多个连接点方法的集合，多个方法

4、**目标对象**

给哪个类的方法增加功能，这个类就是目标对象

5、**Advice**

通知，表示切面功能执行的时间

**切入点定义切入的位置，通知定义切入的时间。**

**一个切面有三个关键的要素：**

- 切面的功能代码，切面干什么
- 切面的执行位置，使用pointcut表示切面执行的位置
- 切面的执行时间，使用Advice表示时间，在目标方法之前还是目标方法之后

## aspectJ的使用

1、AOP的实现

- AOP是一个规范，是动态的一个规范化，一个标准
- AOP的技术实现框架：
  - Spring：spring在内部实现了aop规范，能做aop的工作；**主要在事务处理使用AOP**；项目开发中很少使用spring的aop实现，因为spring的aop比较笨重。
  - **aspectJ：一个开源的专门做aop的框架**，spring框架中已经继承了aspectJ框架，通过spring就能使用aspectJ的功能了
    - 使用xml的配置文件
    - 使用注解（5个）

### 通知

1、切面的执行时间，在规范中叫做Advice（通知，增强）

2、在aspectJ框架中使用注解表示，也可以使用xml配置文件中的标签

- `@Before`
- `@AfterReturning`
- `@Around`
- `@AfterThrowing`
- `@After`

### 位置

表示切面执行的位置，使用切入点表达式

```java
execution(modifiers-pattern? ret-type-pattern
          declaring-type-pattern?name-pattern(param-pattern)
          throws-pattern?)
解释:
modifiers-pattern：访问权限类型
ret-type-pattern：返回值类型
declaring-type-pattern：包名类名 
name-pattern(param-pattern)：方法名(参数类型和参数个数) 
throws-pattern：抛出异常类型
?：表示可选的部分
```

以上表达式共 4 个部分：

**execution(访问权限 方法返回值 方法声明(参数) 异常类型)**

切入点表达式要匹配的对象就是目标方法的方法名。所以，execution 表达式中明显就是方法的签名。注意，表达式中黑色文字表示可省略部分，各部分间用空格分开。在其中可以使用以下符号:

<img src="https://img-blog.csdnimg.cn/20210407160532479.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

**举例:**

`execution(public * *(..))`

任意公共方法

`execution(* set*(..))`

任何一个以“set”开始的方法

`execution(* com.xyz.service.*.*(..))`

定义在 service 包里的任意类的任意方法

`execution(* com.xyz.service..*.*(..))`

定义在 service 包或者子包里的任意类的任意方法。“..”出现在类名中时，后 面必须跟“*”，表示包、子包下的所有类。

`execution(* *..service.*.*(..))`

指定所有包下的 serivce 子包下所有类(接口)中所有方法为切入点

### 例子

**1、使用aop目的是给已经存在的一些类和方法，增加额外的功能，前提是不改变原来类的代码**

2、使用aspectJ实现aop的基本步骤：

- 加入依赖
- 创建目标类：接口和实现类（目的：给类中的方法增加功能）
- 创建切面类：普通类
  - 在类的上面加注解：`@Aspect`
  - 在类中定义方法，方法就是切面要执行的功能代码
    - 在方法的上面加入aspectj的通知注解
    - 指定切入点表达式`excution()`
- 创建spring的配置文件，在文件中声明对象，把对象交给容器统一管理
  - 声明对象：配置文件或者xml配置文件`<bean>`
  - 声明对象包括：目标对象、切面类对象、aspectj框架的自动代理生成器标签（用来完成代理对象的创建功能）
- 创建测试类，从spring容器中获取目标对象（实际上就是代理对象），通过代理执行方法实现aop的功能增强

#### 前置通知 @Before

**目标类**

```java
//目标类
public class SomeServiceImpl implements SomeService {
    @Override
    public void doSome(String name, Integer age) {
        //给doSome方法增加功能：执行之前输出方法的执行时间
        System.out.println("=====doSome=====");
    }
}
```

**切面类**

```java
/**
 * @Aspect：是aspectJ框架中的注解
 *     - 作用：表示当前类是切面类
 *     - 切面类：用来给业务方法增加功能的类，在这个类中有切面的功能代码
 *     - 使用位置：在类定义的上面
 */
@Aspect
public class MyAspect {
    /**
     * 定义方法，实现切面功能
     * 要求：
     *   公共方法
     *   没有返回值
     *   名称自定义
     *   可以有参数也可以没有，参数不是自定义的，有几个参数类型可以使用
     */

    /**
     * @Before：前置通知注解
     *    属性：value，是切入点表达式，表示切面的功能执行的位置
     *    位置：在方法的上面
     * 特点：
     *    在目标方法之前先执行
     *    不会改变目标方法的执行结果
     *    不会影响目标方法的执行
     */
    @Before(value = "execution(public void com.tao.service.aspectJ.SomeServiceImpl.doSome(String,Integer))")
    public void myBefore(){
        //切面要执行的功能代码
        System.out.println("执行时间："+ new Date());
    }
}
```

**配置文件**

```xml
<!--声明目标对象-->
<bean id="impl" class="com.tao.service.aspectJ.SomeServiceImpl"/>
<!--声明切面类对象-->
<bean id="myAspect" class="com.tao.service.aspectJ.MyAspect"/>

<!--
    声明自动代理生成器：使用aspectj框架内部功能，创建代理对象
    创建代理对象是在内存中实现的，修改目标对象的内存结构，创建为代理对象，所以目标对象就是被修改后的代理对象
-->
<aop:aspectj-autoproxy/>
```

**测试结果：**

```java
执行时间：Wed Apr 07 16:49:42 CST 2021
=====doSome=====
```

>使用的是jdk的动态代理对象proxy： com.sun.proxy

#### 切入点表达式优化

```java
@Before(value = "execution(public void com.tao.service.aspectJ.SomeServiceImpl.doSome(String,Integer))")

优化为：
  
@Before(value = "execution(* *..SomeServiceImpl.do*(..))")
```

#### JoinPoint

指定通知方法的参数：`JoinPoint`，**代表的是业务方法**
 *      作用：可以在通知方法中获取方法执行时的信息，例如方法名称、方法的实参
 *      如果你的切面功能中需要用到方法的信息，就加入`JoinPoint`
 *      这个JoinPoint必须是第一个位置的参数

```java
@Before(value = "execution(* *..SomeServiceImpl.do*(..))")
public void myBefore(JoinPoint jp){
    //获取方法的完整定义
    System.out.println("方法的定义="+jp.getSignature());
    System.out.println("方法的名称="+jp.getSignature().getName());

    //获取方法的实参
    System.out.println("name:"+jp.getArgs()[0]);
    System.out.println("age:"+jp.getArgs()[1]);
    
    System.out.println("执行时间："+ new Date());
}
```

输出：

```java
方法的定义=void com.tao.service.aspectJ.SomeService.doSome(String,Integer)
方法的名称=doSome
name:heloo
age:20
执行时间：Wed Apr 07 17:31:54 CST 2021
=====doSome=====
```

#### 后置通知 @AfterReturning

1、后置通知：

 *   公共方法
 *   没有返回值
 *   名称自定义
 *   方法有参数，建议Object

2、`@AfterReturning`：后置通知注解

 * 属性

   - 属性1：`value`，是切入点表达式，表示切面的功能执行的位置

   - 属性2：`returning`，是自定义的变量，表示目标方法的返回值，自定义的变量名必须和通知方法的形参一样

 * 位置：在方法的上面

 * 特点：

   - 在目标方法之后执行

   - 能够获得目标方法的返回值，可以进行处理修改，**但是不影响结果**

3、执行过程：

- 执行doSome方法：`Object res = doSome()；`
- 执行myAfter方法

```java
@AfterReturning(value = "execution(* *..SomeServiceImpl.do*(..))",
                returning = "res")
public void myAfter(Object res){
    //res：目标方法的返回值
    System.out.println("返回值："+res);
    System.out.println("执行时间："+ new Date());
}
```

#### 环绕通知 @Around

1、定义方法，实现切面功能

- 要求：
  - 公共方法
  - 必须有返回值，推荐使用Object
  - 名称自定义
  - 方法有参数，固定的参数` ProceedingJoinPoint`

2、`@Around：`环绕通知

 *    属性：value 切入点表达式
 *    位置：在方法的定义上面
 * 特点：
     - 1、功能最强的通知
     - 2、在目标方法的前和后都有增强功能
     - 3、控制目标方法是否被调用执行
     - 4、修改原来的目标方法的执行结果，影响最后的调用结果

- 等同于jdk的动态代理

 *  参数：ProceedingJoinPoint等同于jdk动态代理中invoke的method
 *      作用：执行目标方法
 * 返回值：目标方法的执行结果，可以被修改

3、在目标方法执行之前之后执行。被注解为环绕增强的方法要有返回值，Object 类型。并 且方法可以包含一个 ProceedingJoinPoint 类型的参数。接口 ProceedingJoinPoint 其有一个 proceed()方法，用于执行目标方法。若目标方法有返回值，则该方法的返回值就是目标方法 的返回值。最后，环绕增强方法将其返回值返回。**该增强方法实际是拦截了目标方法的执行。**

```java
@Around(value = "execution(* *..SomeServiceImpl.doSome(..))")
public Object myAround(ProceedingJoinPoint pjp) throws Throwable {
  //获取参数
  Object[] args = pjp.getArgs();
  String t = (String) args[0];

  Object result = null;
  System.out.println("实现目标方法之前："+new Date());
  //控制目标方法的调用
  if(t.equals("zhangsan")){
    result = pjp.proceed();//method.invoke();
  }
  System.out.println("实现目标方法之后提交事务");
  
  //改变返回值结果
  result = "hello";
  return  result;
}
```

####  异常通知 @AfterThrowing

**1、在目标方法抛出异常后执行**

2、该注解的 `throwing` 属性用于指定所发生的异常类对象。

被注解为异常通知的方法可以包含一个参数 `Exception`，参数名称为` throwing` 指定的名称，表示发生的异常对象。

**3、特点：可以做异常的监控程序，监控目标方法执行时是不是有异常；如果有异常可以发送通知**

```java
@AfterThrowing(value = "execution(* *..SomeServiceImpl.doSome(..))",throwing = "ex")
public void myAfterThrowing(Exception ex){
    System.out.println("异常通知：方法异常时，执行："+ex.getMessage());
    //发送邮件、短信通知开发人员
}
```

运行出错结果：

```java
异常通知：方法异常时，执行：/ by zero
```

#### 最终通知 @After 

无论目标方法是否抛出异常，该增强均会被执行。

**一般是做资源清除的**

```java
@After(value = "execution(* *..SomeServiceImpl.doSome(..))")
public void myAfter(){
    System.out.println("最终通知");
}
```

#### 定义切入点 @Pointcut

```java
/**
 * @Pointcut：定义和管理切入点，如果项目中有多个切入点表达式是重复的，可以复用，可以使用@Pointcut
 *     属性：value 切入点表达式
 *     位置：在自定义的方法上面
 * 特点：
 *     当使用@Pointcut定义在一个方法的上面，此时这个方法的名称就是切入点表达式的别名，
 *     在其他的通知，value属性就可以使用这个方法名称，代替切入点表达式了
 */
@Pointcut(value = "execution(* *..SomeServiceImpl.doSome(..))")
public void mypt(){
    //无需代码
}
```

## 使用CGLIB

如果期望目标类有接口，使用cglib代理

```xml
<aop:aspectj-autoproxy proxy-target-class="true"/>
```

