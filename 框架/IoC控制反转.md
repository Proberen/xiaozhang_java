# IoC 控制反转

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210406141954143.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

## IoC 基本概念

1、控制反转（IoC，Inversion of Control）：**是一个理论、概念、思想**

- 描述的是**：把对象的创建、赋值、管理工作都交给代码之外的容器实现，也就是对象的创建是由其他外部资源完成**

- **控制**：创建对象、对象的属性赋值、对象之间的关系管理

- **反转**：把原来开发人员管理、创建对象的权限转移给代码之外的容器实现，由容器代替开发人员管理对象、创建对象、给属性赋值
  - 正转：开发人员在代码中使用new构造方法创建对象，开发人员主动管理对象

2、**容器**：是一个服务器软件，一个框架（Spring）

> 为什么要使用IoC？

减少对代码的改动，也可以实现不同的功能，实现解耦合

> Java中创建对象的方式？

1、构造方法

2、反射

3、反序列化

4、clone方法

**5、ioc：容器创建对象**

> ioc的体现

**servlet的实现**

- 创建类继承HttpServelt

- 在web.xml文件中注册servlet，使用如下方式

  ```xml
  <servlet-name>test</servlet-name>
  <servlet-class>com.tao.test</servlet-class>
  ```

- 没有使用new的方式创建过对象，是Tomact服务器创建的，**Tomcat也称为容器**

  - Tomcat容器存放：Servlet对象、Listener、Filter对象等

  ```xml
  <servlet-name>test</servlet-name>
  <servlet-class>com.tao.test1</servlet-class> //修改class就可以改变对象
  ```

> IoC的技术实现——DI

**DI（依赖注入）是IoC的技术实现**

- 我们只需要在程序中**提供要使用的对象名称**就可以，至于对象如何在容器中创建、赋值、查找都由容器内部实现

Spring是使用的DI实现了IoC的功能，Spring底层创建对象，使用的是**反射机制**

> Bean

Spring 容器是一个超级大工厂，负责创建、管理所有的 Java 对象，这些 Java 对象被称 为 **Bean**。Spring 容器管理着容器中 Bean 之间的依赖关系，Spring 使用“依赖注入”的方式 来管理 Bean 之间的依赖关系，使用 IoC 实现对象之间的解耦和。

## 例子

**目标：使用ioc，由spring创建对象**

实现步骤：

- 创建maven项目
- 加入maven依赖
- 创建普通的一个类（接口和它的实现类）
- 创建配置文件，声明类的信息，这些类由spring创建和管理
- 测试

### 创建一个普通的类

```java
public interface SomeService {
  void hello();
}

public class SomeServiceImpl implements SomeService {
  @Override
  public void hello() {
    System.out.println("aaaa");
  }
}
```

### 配置文件

```xml
<!--
    beans：根标签，spring把java对象成为bean
    spring-beans.xsd：约束文件
-->

<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--
        id：自定义名称
        class：类的权限定名称（不能是接口，因为使用反射机制）

        spring就完成 SomeService service = new SomeServiceImpl();
        spring是把创建好的对象放入map中，spring框架有一个map存放对象
            springMap.put(id的值，对象)

        一个bean标签声明一个对象
    -->
    <bean id="someService" class="com.tao.service.impl.SomeServiceImpl"/>
</beans>
```

### 测试

```java
//指定spring配置文件的名称
String config = "beans.xml";

//创建表示spring容器的对象，ApplicationContext：表示spring容器，通过容器获取对象
//ClassPathXmlApplicationContext：表示从类路径加载spring的配置文件
ApplicationContext ac = new ClassPathXmlApplicationContext(config);

//从容器中获取某个对象，你要调用对象的方法
//getBean("配置文件中bean的id值")
SomeService sc = (SomeService) ac.getBean("someService");

//使用spring创建的方式
sc.hello();
```

### 细节

> 对象什么时候创建的？

`ApplicationContext ac = new ClassPathXmlApplicationContext(config);`

**在创建Spring容器时，会创建配置文件中所有的对象**

> 其他的方法？

```java
//获取容器中对象的数量
int count = ac.getBeanDefinitionCount();

//获取容器中对象的名称
String[] names = ac.getBeanDefinitionNames();

System.out.println(count);
System.out.println(Arrays.toString(names));
```

> 创建已经存在的某个类的对象（非自定义类，比如String）？

```xml
<bean id="MyString" class="java.lang.String"/> 
```

**默认调用无参构造方法，必须要有无参构造方法！！**

## 基于XML的DI

> DI ：依赖注入，表示创建对象，给属性赋值
>
> 实现语法有两种：
>
> - 在配置文件中使用标签和属性完成：基于XML的DI实现
> - 使用spring的注解完成：基于注解的DI实现
>
> DI的语法分类：
>
> - set注入（设值注入）：spring调用类的set方法，在set方法中可以实现属性的赋值
> - 构造注入：spring调用类的有参数构造方法创建对象，在构造方法中完成赋值

### 注入分类

#### set注入

**1、简单类型的注入**

- **简单类型**：spring中java基本数据类型和字符串都是简单类型
- 调用的是对象的`set方法`，**必须要有set方法**
- **一个property只能给一个属性赋值**
- **不能重复给一个属性赋值**
- 当spring看到name为XX时，会去寻找XX对应的set方法：`setXX`方法，有就不会报错

```xml
<!--
	<bean id="xx" class="yy">
			<property name="属性名字" value="属性的值"/>
	</bean>
-->

<bean id="MyStu" class="com.tao.service.impl.student" >
  <property name="age" value="1"/>
  <property name="name" value="zhangsan"/>
</bean>
```

**2、引用类型的注入**

- 调用类的set方法

```xml
<!--
   <bean id="xx" class="yy">
         <property name="属性名字" ref="bean的id"/>
   </bean>
-->

<bean id="MyStu" class="com.tao.service.impl.student" >
  <property name="age" value="1"/>
  <property name="name" value="zhangsan"/>
  <!--引用数据类型-->
  <property name="s" ref="Mys"/>
</bean>

<!--声明SomeServiceImpl对象-->
<bean id="Mys" class="com.tao.service.impl.SomeServiceImpl">
  <property name="a" value="12"/>
</bean>
```

#### 构造注入

在构造调用者实例的同时完成实例化，即使用构造器设置依赖关系

- **index和name选择一个即可**

```xml
<!--
    construct-arg标签：
        - name：属性名称
        - index：参数的位置，0、1、2...
        - value：简单类型
        - ref：引用类型

    <bean id="xx" class="yy">
        <construct-arg name="属性名字" index="0" ref="bean的id"/>
        <construct-arg name="属性名字" index="1" value="bean的id"/>
    </bean>
-->

<bean id="MyStu" class="com.tao.service.impl.student" >
  <constructor-arg name="name" value="zhang"/>
  <constructor-arg name="age"  value="12"/>
  <constructor-arg name="s" ref="Mys"/>
</bean>

<bean id="Mys" class="com.tao.service.impl.SomeServiceImpl">
  <property name="a" value="12"/>
</bean>
```





### 引用类型属性自动注入

> 引用类型自动注入：spring框架根据某些规则可以给引用类型赋值
> 使用的规则常用的是byName、byType

#### byName自动注入

要求bean的id和属性一致

- 如：student对象有一个属性为s，所以需要id为s的SomeServiceImpl对象

```xml
<!--
    1、byName：按名称注入
    java类中引用类型的属性名和spring容器中<bean>的id名称一样，且数据类型一样，这样的容器的bean，spring可以赋值给引用类型
    语法：
    <bean id="xx" class="yy" autowire="byName">
        简单类型赋值
    </bean>
-->
<bean id="stu" class="com.tao.service.impl.student" autowire="byName">
    <property name="name" value="zhang"/>
    <property name="age" value="12"/>
</bean>
<bean id="s" class="com.tao.service.impl.SomeServiceImpl">
    <property name="a" value="12"/>
</bean>
```

#### byType自动注入

```xml
<!--
    2、byType：按类型注入
    java类中引用类型的数据类型和spring容器中<bean>的class属性是同源关系，这样的bean能够赋值给引用类型
    同源：
     - 和class的值一样
     - 和class的值是父子类关系
     - 和class的值接口是实现类关系
-->
<bean id="stu" class="com.tao.service.impl.student" autowire="byType">
    <property name="name" value="zhang"/>
    <property name="age" value="12"/>
</bean>
<bean id="s" class="com.tao.service.impl.SomeServiceImpl">
    <property name="a" value="15"/>
</bean>
```





### 主配置文件

**1、多个配置文件的优势**

- 每个文件的大小比一个文件要小很多，效率高
- 避免多人竞争带来的冲突

**如果项目中有多个模块，一个模块一个配置文件**

2、多配置文件的分配方式

- 按功能模块，一个模块一个配置文件
- 按类的功能，数据库相关的配置一个配置文件，做事务的功能一个配置文件，service功能一个配置文件

3、结构

- **主配置文件**：包含其他配置文件，不定义对象

  - 语法：`<import resource="其他配置文件的路径"/>`
  - 关键字："`classpath:`"表示类路径（class文件所在的目录），在spring的配置文件中要指定其他文件的位置，需要使用classpath去哪里加载读取文件

  ```xml
  <!--加载的是文件列表-->
  <import resource="classpath:beans.xml"/>
  <import resource="classpath:beans2.xml"/>
  <!--使用通配符（*，表示任意字符）,主配置文件不能包含在通配符内-->
  <import resource="classpath:beans*.xml"/>
  ```

## 基于注解的DI

> 通过注解完成java对象的创建，属性赋值

1、使用注解的步骤：

- 加入maven的依赖 spring-context，在加入这个依赖的同时，会自动加入spring-aop依赖（使用注解必须要这个依赖）
- 在类中加入spring的注解（多个不同功能的注解）
- 在spring的配置文件中加入一个**组件扫描器**的标签，说明注解在项目中的位置
- 测试

2、注解

- `@Component`：调用无参构造方法创建指定value的对象
- `@Respotory`
- `@Service`
- `@Controller`

- `@Value`
- `Autowired`
- `@Resource`

### 组件扫描器

```xml
<!--
    声明组件扫描器(component-scan),组件就是java对象
      - base-package：指定注解在项目中的包名
      - 工作方式：spring扫描遍历base-package指定的包，把包中和子包中所有的类，找到类的注解，按照注解功能创建对象或者给属性赋值

    加入component-scan标签，配置文件产生的变化：
      - 会加入一个新的约束文件spring-context.xsd
      - 给新的约束文件起个命名空间的名称
-->
<context:component-scan base-package="com.tao.service.impl"/>
```

**指定多个包的三种方式：**

  - 多次使用组件扫描器标签
  - 使用分隔符（ ; 或者 , ）来分割多个包名
  - 指定父包

### 创建对象：@Component

**调用无参构造方法创建对象**

- 可以省略value
- 可以不写value，Spring提供默认名称，默认名称是类名的首字母小写

```java
/**
 * 用来创建对象，等同于bean的功能
 *  - 属性：value 对象的名称，也就是bean的id值
 *         value的值是唯一的，创建的对象在整个spring容器中就一个
 *  - 位置要求：在类的上面
 *
 * @Component(value = "myStudent")等同于
 * <bean id="myStudent" class="com.tao.service.impl.student"/>
 *
 */
@Component(value = "myStudent")
//可以省略value
@Component("myStudent") 
//Spring提供默认名称，默认名称是类名的首字母小写
@Component
public class student {
  private String name;
  private int age;
}
```

测试

```java
@Test
public void test04(){
    String config = "zhujie.xml";
    ApplicationContext ac = new ClassPathXmlApplicationContext(config);

    student myStu = (student) ac.getBean("myStudent");
    System.out.println(myStu);
}

输出：
  student{name='null', age=0}
```

**用法：当一个类不是持久层、业务层、控制器，就使用@Component**

### 创建对象：@Repository

放在dao的实现类上，表示创建dao对象，dao对象是可以访问数据库的

### 创建对象：@Service

放在service的实现类上，创建service对象，service对象可以做业务处理，可以有事务等功能

### 创建对象：@Controller

放在控制器类上，创建控制器对象，控制器对象能够接受用户提交的参数，显示请求的处理结果

### 简单类型赋值：@Value



```java
@Component("myStudent")
public class student {

    /**
     * @Value：简单类型的赋值
     *  - 属性：value 是string类型的，表示简单类型的属性值
     *  - 位置：在属性定义的上面，无需set方法 或者 在set方法的上面
     */
    @Value("zhangsan")
    private String name;
    @Value("123")
    private int age;
}
```

测试

```java
@Test
public void test04(){
    String config = "zhujie.xml";
    ApplicationContext ac = new ClassPathXmlApplicationContext(config);

    student myStu = (student) ac.getBean("myStudent");
    System.out.println(myStu);
}

输出：
  student{name='zhangsan', age=123}
```

### 引用类型赋值：@Atuowired

**默认使用按类型自动装配Bean的方式**

```java
@Component("myStudent")
public class student {

  /**
     * @Value：简单类型的赋值
     *  - 属性：value 是string类型的，表示简单类型的属性值
     *  - 位置：在属性定义的上面，无需set方法 或者 在set方法的上面
     *
     */
  @Value("zhangsan")
  private String name;
  @Value("123")
  private int age;

  /**
     * 引用类型
     * spring中通过注解给引用类型赋值，使用自动注入原理，支持byName、byType
     *
     * @Autowired：spring框架提供的注解，实现引用类型的赋值，默认使用byType
     *      - 位置：在属性定义上面 或者 set方法上面
     */
  @Autowired
  private SomeServiceImpl s;
}

@Component("mySchool")
public class SomeServiceImpl implements SomeService {

  @Value("10086")
  public int a ;
}

测试结果：
  student{name='zhangsan', age=123,s=SomeServiceImpl{a=10086}}
```

#### required属性

`@Autowired `还有一个属性 required，默认值为 true，表示当匹配失败后，会终止程序运行。若将其值设置为 false，则匹配失败，将被忽略，未匹配的属性值为 null。

### 引用类型赋值：@Qualifier

**需要和`@Autowired`配合使用**

```java
/**
 *
 *  使用byName的方式：需要在属性上面加入@Autowired，在属性上面加入@Qulifier(value="bean的id")：表示使用指定名称的bean完成赋值
 */
@Autowired
@Qualifier("mySchool")
private SomeServiceImpl s;
```

### JDK注解：@Resource

1、**引用类型赋值**

2、**来自于jdk**

3、**可以按名称匹配 Bean， 也可以按类型匹配 Bean。**

4、**默认是按名称注入**

**5、默认处理：先使用byName自动注入，如果byName失败会使用byType**

**6、只使用byName方式：使用`name`，指定为对象名**

```java
@Resource(name = "s")
private SomeServiceImpl s;
```

## xml和注解对比

1、注解

- 优点：方便 、直观、高效(代码少，没有配置文件的书写那么复杂)。

- 弊端：以硬编码的方式写入到 Java 代码中，修改是需要重新编译代码的。

2、XML 

- 优点：配置和代码是分离的，在xml中做修改，无需编译代码，只需重启服务器即可将新的配置加载。 
- 缺点：编写麻烦，效率低，大型项目过于复杂

## IoC实现解耦合

IoC能够实现业务对象之间的解耦合，例如service和dao对象之间的解耦合

