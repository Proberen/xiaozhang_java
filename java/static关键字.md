# 静态字段
如果将一个字段定义为static，那么每个类都只有一个这样的字段。而对于非静态的实例字段，每一个对象都有自己的一个副本。

例子：

```java
class Employee{
	private static int nextID = 1;
	private int id;
	....
}
```
每一个Employee对象都有一个自己的ID字段，但这个类的所有实例将共享一个nextID字段。

如果有100个Employee对象，则有100个实例字段id，分别对应每一个对象；但是，只有一个nextID字段。即使没有Employee对象，静态字段nextID也存在。它属于类，不属于任何一个对象。

```java
Employee.nextID
```

# 静态常量

例如：

```java
public class Math{
	...
	public static final double PI = 3.1415926;
	...
}
```
在程序中，可以使用`Math.PI`来访问这个常量

如果省略关键字static，PI就变成了一个实例字段，就需要通过类的一个对象来访问PI

咱们多次使用过`System.out`这个静态常量，它在System中的声明如下：

```java
public class System{
	...
	public static final PrintStream out = ....;
	...
}
```

# 静态方法

1 、静态方法是不在对象上执行的方法

2、静态方法没有this参数的方法

3、<span style="background: yellow;">静态方法不能访问实例字段，但是可以访问静态字段</span>

```java
public static int getNextID(){
	return nextID;
}
```
可以这样调用这个方法：`Employee.getNextID();`

4、<span style="background: yellow;">可以使用对象调用静态方法，但是不建议使用</span>

5、下面两种状况可以使用静态方法：
- 方法不需要访问对象状态，因为它所需要的所有参数都通过显式参数提供
- 方法之需要访问类的静态字段

# 修饰类
static修饰的类<span style="background: yellow;">只能为内部类</span>，普通类无法用static关键字修饰。static修饰的内部类相当于一个普通的类，访问方式为（new 外部类名.内部类的方法() ）

# 工厂方法
静态方法还有一种常见的用途。类似`LocalDate`和`NumberFormat`的类使用静态工厂方法来构造对象。

为什么不使用构造器呢？
- 无法命名，构造器的名字必须和类名相同
- 使用构造器时，无法改变所构造对象的类型。