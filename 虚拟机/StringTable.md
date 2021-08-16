# StringTable

## 基本特性

1、String：字符串，使用一对""引起来表示。

- String sl = "hello"；//字面量的定义方式
- String s2 = new String（"hello"） ；

2、String声明为`final`的， 不可被继承

3、String实现了`Serializable`接口：表示字符串是支持序列化的。 

4、实现了`Comparable`接口：表示String可以比较大小

5、String在jdk8及以前内部定义了`final char value[]`，value用于存储字符串数据，jdk9时改为`byte[]`

- `结论： String再也不用char[] 来存储，改成了byte[] 加上编码标记，节约了一些空间。StringBuffer和StringBuilder也做了一些修改`

6、String：代表**不可变的字符序列**，简称：不可变性。

- **当对字符串重新赋值时**，需要重写指定内存区域赋值，不能使用原有的value进行赋值。
- **当对现有的字符串进行连接操作时**，也需要重新指定内存区域赋值，不能使用原有的value进行赋值。
- **当调用String的replace（）方法修改指定字符或字符串时**，也需要重新指定内存区域赋值，不能使用原有的value进行赋值。

**7、通过字面量的方式（区别于new）给一个字符串赋值，此时的字符串值声明在字符串常量池中。**

```java
//字面量
String a = "aa";
```

**8、字符串常量池中是不会存储相同内容的字符串的**

- String的String Pool 是一个固定大小的`Hashtable`，默认值大小长度是1009。如果放进StringPool的String非常多， 就会造成Hash冲突严重，从而导致链表会很长，而链表长了后直接会造成的影响就是当调用`String. intern`时性能会大幅下降。
- 使用`-XX:StringTableSize`可设置StringTable的长度
- 在jdk6中StringTable是固定的，就是1009的长度，所以如果常量池中的字符串过多就会导致效率下降很快。StringTableSize设 置没有要求
- 在jdk7中，StringTable的长度默认值是60013
- jdk8开始,1009是StringTable长度可设置的最小值

## String的内存分配

1、在Java语言中有8种基本数据类型和一种比较特殊的类型String。这些类型为了使它们在运行过程中速度更快、更节省内存，都提供了一种**常量池**的概念。

2、常量池就类似一个Java系统级别提供的缓存，8种基本数据类型的常量池都是系统协调的，**String类型的常量池比较特殊**，它的主要使用方法有两种：

- **直接使用双引号声明出来的String对象会直接存储在常量池中**
  - 比如： `String info = "abc"` ；
- **如果不是用双引号声明的String对象，可以使用String提供的`intern（）`方法**

3、变化

- Java 6及以前，字符串常量池存放在永久代
- Java 7中Oracle的工程师对字符串池的逻辑做了很大的改变，即将**字符串常量池的位置调整到Java堆内**
  - 所有的字符串都保存在堆（Heap）中，和其他普通对象一样，这样在进行调优应用时仅需要调整堆大小就可以了
  - 字符串常量池概念原本使用得比较多，但是这个改动使得我们有足够的理由让我们重新考虑在Java 7中使用`String. intern（）`

- Java8元空间，字符串常量在**堆**

> 为什么字符串常量池需要进行改变？

1、永久代默认情况下比较小

2、永久代的回收效率较低，垃圾回收频率低

## String的基本操作

例一：

<img src="https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bac675fab2b1f~tplv-t2oaga2asx-watermark.awebp" alt="img" style="zoom:50%;" />

<hr/>

例2:

```java
class Memory {
  public static void main(String[] args) {//line 1
    int i = 1;//line 2
    Object obj = new Object();//line 3
    Memory mem = new Memory();//line 4
    mem.foo(obj);//line 5
  }//line 9

  private void foo(Object param) {//line 6
    String str = param.toString();//line 7
    System.out.println(str);
  }//line 8
}
```



<img src="https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bac6cb643c9df~tplv-t2oaga2asx-watermark.awebp" alt="img" style="zoom:50%;" />

## String的拼接操作

1、常量与常量的拼接结果在常量池，原理是**编译期优化**

2、常量池中不会存在相同内容的常量。

3、**只要其中有一个是变量，结果就在堆中**，变量拼接的原理是StringBuilder

**4、如果拼接的结果调用intern（）方法，则主动将常量池中还没有的字符串对象放入池中，并返回此对象地址**。

```java
@Test
public void test1(){
  String s1 = "a" + "b" + "c";//编译期优化：等同于"abc"
  String s2 = "abc"; //"abc"一定是放在字符串常量池中，将此地址赋给s2
  /*
   * 最终.java编译成.class,再执行.class
   * String s1 = "abc";
   * String s2 = "abc"
   */
  System.out.println(s1 == s2); //true
  System.out.println(s1.equals(s2)); //true
}
```

```java
@Test
public void test2(){
  String s1 = "javaEE";
  String s2 = "hadoop";

  String s3 = "javaEEhadoop";
  String s4 = "javaEE" + "hadoop";//编译期优化
  //如果拼接符号的前后出现了变量，则相当于在堆空间中new String()，具体的内容为拼接的结果：javaEEhadoop
  String s5 = s1 + "hadoop";
  String s6 = "javaEE" + s2;
  String s7 = s1 + s2;

  System.out.println(s3 == s4);//true
  System.out.println(s3 == s5);//false
  System.out.println(s3 == s6);//false
  System.out.println(s3 == s7);//false
  System.out.println(s5 == s6);//false
  System.out.println(s5 == s7);//false
  System.out.println(s6 == s7);//false
  //intern():判断字符串常量池中是否存在javaEEhadoop值，如果存在，则返回常量池中javaEEhadoop的地址；
  //如果字符串常量池中不存在javaEEhadoop，则在常量池中加载一份javaEEhadoop，并返回次对象的地址。
  String s8 = s6.intern();
  System.out.println(s3 == s8);//true
}
```

### 底层原理

**1、字符串拼接操作不一定使用StringBuilder**

- 情况一：拼接符号左右两边都是字符串常量：`"a"+"b"`
- 情况二：拼接符号左右两边都是常量引用：`final String a= "a";final String b="b"; a+b;`

**2、针对final修饰类、方法、基本数据类型、引用数据类型时，能使用final就使用final**

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bac74e1331d1d~tplv-t2oaga2asx-watermark.awebp)

**3、append的方式比拼接字符串更加高效**

- 使用append的方式，从始至终只需要创建一个stringbuilder对象
- 使用字符串拼接，每次都需要创建stringbuilder对象、String对象，占用内存过多

4、改进空间

- 在实际开发中，如果基本确定需要添加字符串的长度，就可以自定义长度实例化stringbuilder

```java
StringBuilder a = new StringBuilder(10)；
```

## intern()方法

```java
public native String intern();
```

1、如果不是用双引号声明的String对象，可以使用String提供的`intern`方法： `intern`方法会从字符串常量池中查询当前字符串是否存在，若不存在就会将当前字符串放入常量池中。

- 比如： `String myInfo = new String("I love u").intern()；`
- 也就是说，如果在任意字符串上调用String. intern方法，那么其返回结果所指向的那个类实例，必须和直接以常量形式出现的字符串实例完全相同。

2、因此，下列表达式的值必定是true： 

`（"a" + "b" + "c"）.intern（）== "abc";`
通俗点讲，Interned String就是确保字符串在内存里只有一份拷贝，这样可以节约内存空间，加快字符串操作任务的执行速度。注意，这个值会被存放在字符串内部池（String Intern Pool）。

### new String()创建几个对象？

```java
public class StringNewTest {
  public static void main(String[] args) {
    String str1 = new String("ab");
    String str2 = new String("a") + new String("b");
  }
}
```



<img src="https://img-blog.csdnimg.cn/20210401145814662.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom: 30%;" />

**注意：使用toString方法不会在常量池中生成**



### 面试题

```java
/**
 * 如何保证变量s指向的是字符串常量池中的数据呢？
 * 有两种方式：
 * 方式一： String s = "shkstart";//字面量定义的方式
 * 方式二： 调用intern()
 *         String s = new String("shkstart").intern();
 *         String s = new StringBuilder("shkstart").toString().intern();
 */
public class StringIntern {
  public static void main(String[] args) {
    String s = new String("1");
    String s1 = s.intern();//调用此方法之前，字符串常量池中已经存在了"1"
    String s2 = "1";
    //s  指向堆空间"1"的内存地址
    //s1 指向字符串常量池中"1"的内存地址
    //s2 指向字符串常量池已存在的"1"的内存地址  所以 s1==s2
    System.out.println(s == s2);//jdk6：false   jdk7/8：false
    System.out.println(s1 == s2);//jdk6: true   jdk7/8：true
    System.out.println(System.identityHashCode(s));//491044090
    System.out.println(System.identityHashCode(s1));//644117698
    System.out.println(System.identityHashCode(s2));//644117698

    //s3变量记录的地址为：new String("11")
    String s3 = new String("1") + new String("1");
    //执行完上一行代码以后，字符串常量池中，是否存在"11"呢？答案：不存在！！

    // jdk6:创建了一个新的对象"11",也就有新的地址。
    // jdk7:此时常量中并没有创建"11",而是创建一个指向堆空间中new String("11")的地址
    s3.intern();
    //s4变量记录的地址：使用的是上一行代码代码执行时，在常量池中生成的"11"的地址
    String s4 = "11";
    System.out.println(s3 == s4);//jdk6：false  jdk7/8：true
  }

}
```

<img src="https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bac960ae887a7~tplv-t2oaga2asx-watermark.awebp" alt="6" style="zoom:50%;" />

<img src="https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bac97d066c47d~tplv-t2oaga2asx-watermark.awebp" alt="7" style="zoom:50%;" />

**拓展**

```java
public class StringIntern1 {
  public static void main(String[] args) {
    String s3 = new String("1") + new String("1");//new String("11")
    //执行完上一行代码以后，字符串常量池中，是否存在"11"呢？答案：不存在！！
    String s4 = "11";//在字符串常量池中生成对象"11"
    String s5 = s3.intern();
    System.out.println(s3 == s4);//false
    System.out.println(s5 == s4);//true
  }
}
```



### 总结

jdk6中，将这个字符串对象尝试放入串池

- 如果串池中有，则不会放入，返回已有的地址
- 如果没有，将此字符串对象复制一份放入，并返回对象地址

jdk7后，将这个字符串对象尝试放入串池

- 如果有，不会放入，返回已有的地址
- 如果没有，将字符串对象的引用地址复制一份放入，返回引用地址

<hr/>

1、两个String拼接，因为最后stringBuilder调用了toString方法，所以返回的是一个new String（）地址

2、如果常量池里面没有拼接后的值，调用intern方法，常量池中放的是new String()的地址

3、如果常量池有拼接后的值，调用intern方法，就会返回常量池这个值的地址

### 练习

```java
public class StringExer1 {
  public static void main(String[] args) {
    String s = new String("a") + new String("b");//new String("ab")
    //在上一行代码执行完以后，字符串常量池中并没有"ab"

    String s2 = s.intern();//jdk6中：在串池中创建一个字符串"ab"
    //jdk8中：串池中没有创建字符串"ab",而是创建一个引用，指向new String("ab")，将此引用返回

    System.out.println(s2 == "ab");//jdk6:true  jdk8:true
    System.out.println(s == "ab");//jdk6:false  jdk8:true
  }
}
```

**jdk6**

<img src="https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bac9a78f97a95~tplv-t2oaga2asx-watermark.awebp" alt="8" style="zoom: 33%;" />

**jdk7/8**

<img src="https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bacadb3c78aec~tplv-t2oaga2asx-watermark.awebp" alt="9" style="zoom: 33%;" />

```java
public class StringExer1 {
  public static void main(String[] args) {
    String x = "ab";
    String s = new String("a") + new String("b");//new String("ab")
    String s2 = s.intern();

    System.out.println(s2 == x);//true 
    System.out.println(s == x);//false
  }
}
```



<img src="https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bacaf267e7da0~tplv-t2oaga2asx-watermark.awebp" alt="10" style="zoom: 33%;" />



```java
public class StringExer2 {
  public static void main(String[] args) {
    String s1 = new String("ab");//执行完以后，会在字符串常量池中会生成"ab"
    //        String s1 = new String("a") + new String("b");////执行完以后，不会在字符串常量池中会生成"ab"
    s1.intern();
    String s2 = "ab";
    System.out.println(s1 == s2); //false
  }
}
```



## intern()空间效率

大的网站平台，需要内存中存储大量的字符串。比如社交网站，很多人都存储：北京市、海淀区等信息。这时候如果字符串都调用 intern（）方法，就会明显降低内存的大小。

## G1中的String去重操作

- 背景：对许多Java应用（有大的也有小的）做的测试得出以下结果：
  - 堆存活数据集合里面String对象占了25%
  - 堆存活数据集合里面重复的String对象有13.5%
  - String对象的平均长度是45
- 许多大规模的Java应用的瓶颈在于内存，测试表明，在这些类型的应用里面**，Java堆中存活的数据集合差不多25%是String对象**。更进一步，这里面差不多一半String对象是重复的，重复的意思是说： `string1. equals （string2）=true`。**堆上存在重复的string对象必然是一种内存的浪费**。这个项目将在G1垃圾收集器中实现自动持续对重复的String对象进行去重，这样就能避免浪费内存。

### 实现

- 当垃圾收集器工作的时候，会访问堆上存活的对象。对每一个访问的对象都会检查是否是候选的要去重的String对象。
- 如果是，把这个对象的一个引用插入到队列中等待后续的处理。一个去重的线程在后台运行，处理这个队列。处理队列的一个元素意味着从队列删除这个元素，然后尝试去重它引用的String对象。
- 使用一个hashtable来记录所有的被String对象使用的不重复的char数组。 当去重的时候，会查这个hashtable，来看堆上是否已经存在一个一模一样的char数组。
- 如果存在，String对象会被调整引用那个数组，释放对原来的数组的引用，最终会被垃圾收集器回收掉。
- 如果查找失败，char数组会被插入到hashtable，这样以后的时候就可以共享这个数组了。

### 命令行选项

- UseStringDeduplication （bool） ：开启String去重，默认是不开启的，需要手动开启。
- PrintStringDedupl icationStatistics （bool） ：打印详细的去重统计信息，
- StringDedupl icationAgeThreshold （uintx） ：达到这个年龄的string对象被认.为是去重的候选对象







