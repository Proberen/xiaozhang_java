先说结论：<span style="background: yellow;">按值传递</span>

方法的参数传递包括<span style="background: yellow;">基本数据类型的传递</span>和<span style="background: yellow;">引用类型的传递</span>

### 1、基本数据类型的传递

举个例子：

```java
int a = 1;
int b = 2;

test(a,b);

System.out.println(a);
System.out.println(b);

public static void test(int aa,int bb){
	aa=bb;
}
```
结果：

```java
1
2
```
分析：

进入test，aa初始化为a的一个副本，bb初始化为b的一个副本，在test方法里，aa被赋值为bb，那么aa就等于2，bb等于2；方法结束后，aa和bb作为参数变量也被一起丢弃了，因此最后a和b的值并不会发生改变

### 2、引用数据类型的传递
举个例子：

```java
person a = new person(1);
test(a);
System.out.println(a.age);

public class person{
	int age;
}

public static void test(person x){
	x.age = 10;
}
```
结果：

```java
10
```
分析：

进入test方法后，x初始化为a的一个副本，a其实就是一个对象引用，真正的对象存储在堆内存中，在方法内，该对象的age被置为10，那么结束方法后这个x不再被使用，但是a还是指向那个堆内存中的对象，因此a的age为10

### 总结
1、一个方法不能修改一个基本数据类型的参数（即数值型或布尔型）。

2、一个方法可以改变一个对象参数的状态。

3、一个方法不能让对象参数引用一个新的对象。