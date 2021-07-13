## 9.1 String
1、String概述：
- String类在`java.lang`包下，所以使用的时候不需要导包
- String类表示字符串，Java程序中所有的字符串文字（例如"abc"）都被实现为此类的实例，也就是说，<span style="background: yellow;">Java程序中所有双引号字符串，都是String类的对象</span>

### 字符串的特点
- <span style="background: yellow;">字符串不可变</span>，它们的值在创建后不可以被更改
- 虽然String值不可变，但是可以被共享
- 字符串效果上相当于字符数组（char[]），但是底层原理是<span style="background: yellow;">字节数组</span>(byte[])，<span style="background: yellow;">jdk8及以前是字符数组，jdk9及以后是字节数组</span>

### String构造方法
- `public String()`：创建一个空白字符串对象，不含任何内容
- `public String(char[] chs)`：根据字符数组的内容，来创建字符串对象
- `public String(byte[] bys)`：根据字节数组的内容，来创建字符串对象
- `String s = "abc"`：直接赋值的方式创建字符串对象（推荐）

例子：输出abc
```java
String s1 = new String();

char[] chs = {'a','b','c'};
String s2 = new String(chs);

byte[] bys = {97,98,99};
String s3 = new String(bys)

String s4 = "abc";
```

### String对象的特点
- 通过`new`创建的字符串对象，每一次new都会申请一个内存空间，但是地址不同
- 以`""`的方式给出的字符串，只要字符序列相同，无论在程序中出现几次都只会建立一个String对象，并在字符池中维护

### String对象的内存空间
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210310220105206.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

### 字符串的比较
- `==`
基本类型：比较的是数据值是否相同
引用类型：比较的是地址值是否相同
- `equals`
字符串public boolean equals(Object anObject)：将此字符串与指定对象进行比较，由于比较的是字符串对象，所以参数直接传递字符串。

## 9.2 StringBuilder
概述：StringBuilder是一个可变的字符串类，可以看作一个容器（<span style="background: yellow;">可变是指StringBuilder对象中的内容是可变的</span>）

### String的加法操作

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210310223752477.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70)

### String和StringBuilder的区别
String内容不可变，StringBuilder内容可变

### StringBuilder构造方法
- `public StringBuilder()`：创建一个空白可变字符串对象，不含有任何内容
- `public StringBuilder(String str)`：根据字符串的内容，来创建可变字符串对象

```java
// 创建一个空白可变字符串
StingBuilder sb = new StringBuilder();
System.out.println(sb);

// 根据字符串内容创建一个可变字符串
StingBuilder sb2 = new StringBuilder("hello");
System.out.println(sb2);
```
### StringBuilder的添加和反转方法
- `public StringBuilder append(任意类型)`：在对象本身上添加数据，并<span style="background: yellow;">返回对象本身</span>
- `public StringBuilder reverse()`：对象本身进行相反的字符序列

```java
StingBuilder sb = new StringBuilder();
StingBuilder sb2 = sb.append("hello")
System.out.println(sb);
System.out.println(sb2);
System.out.println(sb == sb2);
```
输出：

```java
hello
hello
true
```
注意：这两个方法都会改变对象本身～～

### String和StringBuilder的相互转换
#### StingBuilder转为String
`public String toString()`：通过toString()就可以实现把StringBuilder转换为String

```java
StingBuilder sb = new StringBuilder("abc");
String s = sb.toString();
```

#### Sting转为StringBuilder
`public StringBuilder(String s)`：通过构造方法实现String转为StringBuilder

```java
String a = "abc";
StringBuilder b = new StringBuilder(a);
```
## 面试题
1、String不可变的好处？
- 可以缓存 hash 值
因为 String 的 hash 值经常被使用，例如 String 用做 HashMap 的 key。不可变的特性可以使得 hash 值也不可变，因此只需要进行一次计算。
- String Pool 的需要
如果一个 String 对象已经被创建过了，那么就会从 String Pool 中取得引用。只有 String 是不可变的，才可能使用 String Pool。
- 安全性
String 经常作为参数，String 不可变性可以保证参数不可变。例如在作为网络连接参数的情况下如果 String 是可变的，那么在网络连接过程中，String 被改变，改变 String 的那一方以为现在连接的是其它主机，而实际情况却不一定是。
- 线程安全
String 不可变性天生具备线程安全，可以在多个线程中安全地使用。

==========================================================

2、浅谈一下String, StringBuffer，StringBuilder的区别？

- String是不可变类，每当我们对String进行操作的时候，总是会创建新的字符串。操作String很耗资源,所以Java提供了两个工具类来操作String - StringBuffer和StringBuilder。

- StringBuffer和StringBuilder是可变类，<span style="background: yellow;">StringBuffer是线程安全的</span>，<span style="background: yellow;">StringBuilder则不是线程安全的</span>。所以在多线程对同一个字符串操作的时候，我们应该选择用StringBuffer。由于不需要处理多线程的情况，StringBuilder的效率比StringBuffer高。

==========================================================

3、什么是字符串池？

字符串常量池就是用来存储字符串的，它存在于Java 堆内存。

==========================================================

4、String是线程安全的吗？

String是不可变类，一旦创建了String对象，我们就无法改变它的值。因此，它是线程安全的，可以安全地用于多线程环境中。

==========================================================

5、为什么我们在使用HashMap的时候总是用String做key？

因为字符串是不可变的，当创建字符串时，它的它的hashcode被缓存下来，不需要再次计算。因为HashMap内部实现是通过key的hashcode来确定value的存储位置，所以相比于其他对象更快。这也是为什么我们平时都使用String作为HashMap对象。