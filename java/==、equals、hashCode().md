
# equals()
1、equals() 的作用是 <span style="background: yellow;">用来判断两个对象是否相等</span>。

2、equals() 定义在JDK的Object.java中。通过判断两个对象的地址是否相等(即，是否是同一个对象)来区分它们是否相等。

3、Java对equals（）的要求：
- 对称性：如果`x.equals(y)`为`true`，那么`y.equals(x)`也为`true`
- 反射性：`x.equals(x)`为`true`
- 类推性：`x.equals(y)`为`true`，`y.equals(z)`为`true`，则`x.equals(z)`为`true`
- 一致性：如果`x.equals(y)`为`true`，只要x和y的内容不变，不管重复多少次都是`true`
- 非空性：`x.equals(null)`为`false`，`x.equals(不同类型的对象)`为`false`

# == 和 equals()
面试官：请问 equals() 和 "==" 有什么区别？

应聘者：

- equals()方法用来比较的是两个对象的内容是否相等，由于所有的类都是继承自`java.lang.Object`类的，所以适用于所有对象，如果没有对该方法进行覆盖的话，调用的仍然是Object类中的方法，而Object中的equals方法返回的却是==的判断；

- "==" 比较的是变量(栈)内存中存放的对象的(堆)内存地址，用来判断两个对象的地址是否相同，即是否是指相同一个对象。

## 题目

```java
String s1 = new String("zs");
String s2 = new String("zs");
System.out.println(s1 == s2); //false

String s3 = "zs";
String s4 = "zs";
System.out.println(s3 == s4); //true，常量池地址
System.out.println(s3 == s1); //false，s1指向堆，s3指向常量池

String s5 = "zszs";
String s6 = s3+s4;
System.out.println(s5 == s6); //false，s3+s4构建新的对象

final String s7 = "zs";
final String s8 = "zs";
String s9 = s7+s8;
System.out.println(s5 == s9); //true，final关键字，表示是常量

final String s10 = s3+s4;
System.out.println(s5 == s10); //false
```
# hashCode()的作用
1、hashCode() 的作用是获取<span style="background: yellow;">哈希码</span>，也称为<span style="background: yellow;">散列码</span>；它实际上是返回一个<span style="background: yellow;">int整数</span>。这个哈希码的作用是<span style="background: yellow;">确定该对象在哈希表中的索引位置</span>。

2、hashCode() 定义在JDK的Object.java中，这就意味着<span style="background: yellow;">Java中的任何类都包含有hashCode() 函数</span>。

虽然，每个Java类都包含hashCode() 函数。但是，**仅仅当创建并某个“类的散列表”(关于“散列表”见下面说明)时，该类的hashCode() 才有用**(作用：确定该类的每一个对象在散列表中的位置；其它情况下(例如，创建类的单个对象，或者创建类的对象数组等等)，类的hashCode() 没有作用。

上面的<span style="background: yellow;">散列表</span>，指的是：**Java集合中本质是散列表的类**，如HashMap，Hashtable，HashSet。

>  也就是说：hashCode() 在散列表中才有用，在其它情况下没用。在散列表中hashCode() 的作用是获取对象的散列码，进而确定该对象在散列表中的位置。
## 散列表
## 散列码
> 我们都知道，散列表存储的是键值对(key-value)，它的特点是：能根据“键”快速的检索出对应的“值”。这其中就利用到了散列码！
> 散列表的本质是通过<span style="background: yellow;">数组</span>实现的。当我们要获取散列表中的某个“值”时，实际上是要获取数组中的某个位置的元素。而数组的位置，就是通过“键”来获取的；更进一步说，数组的位置，是通过“键”对应的散列码计算得到的。

下面，我们以HashSet为例，来深入说明hashCode()的作用。

假设，HashSet中已经有1000个元素。当插入第1001个元素时，需要怎么处理？因为HashSet是Set集合，它允许有重复元素。

“将第1001个元素逐个的和前面1000个元素进行比较”？显然，这个效率是相等低下的。散列表很好的解决了这个问题，它根据元素的散列码计算出元素在散列表中的位置，然后将元素插入该位置即可。对于相同的元素，自然是只保存了一个。

由此可知，<span style="background: yellow;">若两个元素相等，它们的散列码一定相等；但反过来确不一定</span>。在散列表中，
- 如果两个对象相等，那么它们的hashCode()值一定要相同；
- 如果两个对象hashCode()相等，它们并不一定相等。
注意：这是在散列表中的情况。在非散列表中一定如此！

# hashCode()和equals()
我们以“类的用途”来将“hashCode() 和 equals()的关系”分2种情况来说明。

## 不会创建“类对应的散列表”
这里所说的“不会创建类对应的散列表”是说：我们不会在HashSet, Hashtable, HashMap等等这些本质是散列表的数据结构中，用到该类。例如，不会创建该类的HashSet集合。

在这种情况下，<span style="background: yellow;">该类的“hashCode() 和 equals() ”没有半毛钱关系的！</span>

这种情况下，equals() 用来比较该类的两个对象是否相等。而hashCode() 则根本没有任何作用，所以，不用理会hashCode()。

下面，我们通过示例查看类的两个对象相等 以及 不等时hashCode()的取值。

```java
public class NormalHashCodeTest{
    public static void main(String[] args) {
        // 新建2个相同内容的Person对象，
        // 再用equals比较它们是否相等
        Person p1 = new Person("eee", 100);
        Person p2 = new Person("eee", 100);
        Person p3 = new Person("aaa", 200);
        System.out.printf("p1.equals(p2) : %s; p1(%d) p2(%d)\n", p1.equals(p2), p1.hashCode(), p2.hashCode());
        System.out.printf("p1.equals(p3) : %s; p1(%d) p3(%d)\n", p1.equals(p3), p1.hashCode(), p3.hashCode());
    }

    private static class Person {
        int age;
        String name;
        /**
         * @desc 覆盖equals方法
         */
        public boolean equals(Object obj){
            if(obj == null){
                return false;
            }
            //如果是同一个对象返回true，反之返回false
            if(this == obj){
                return true;
            }
            //判断是否类型相同
            if(this.getClass() != obj.getClass()){
                return false;
            }
            Person person = (Person)obj;
            return name.equals(person.name) && age==person.age;
        }
    }
}
```
结果：

```java
p1.equals(p2) : true; p1(1169863946) p2(1901116749)
p1.equals(p3) : false; p1(1169863946) p3(2131949076)
```
从结果也可以看出：p1和p2相等的情况下，hashCode()也不一定相等。

## 会创建“类对应的散列表”
这里所说的“会创建类对应的散列表”是说：我们会在HashSet, Hashtable, HashMap等等这些本质是散列表的数据结构中，用到该类。例如，会创建该类的HashSet集合。

在这种情况下，该类的“hashCode() 和 equals() ”是有关系的：
- 如果两个对象相等，那么它们的hashCode()值一定相同。
              这里的相等是指，通过equals()比较两个对象时返回true。
- 如果两个对象hashCode()相等，它们并不一定相等。
               因为在散列表中，hashCode()相等，即两个键值对的哈希值相等。然而哈希值相等，并不一定能得出键值对相等。补充说一句：“两个不同的键值对，哈希值相等”，这就是哈希冲突。

此外，在这种情况下。若要判断两个对象是否相等，除了要覆盖equals()之外，也要覆盖hashCode()函数。否则，equals()无效。

**例如，创建Person类的HashSet集合，必须同时覆盖Person类的equals() 和 hashCode()方法。**

<span style="background: yellow;">Test1:</span>

如果单单只是覆盖equals()方法，我们会发现，equals()方法没有达到我们想要的效果。

```java
public class ConflictHashCodeTest1{
    public static void main(String[] args) {
        // 新建Person对象，
        Person p1 = new Person("eee", 100);
        Person p2 = new Person("eee", 100);
        Person p3 = new Person("aaa", 200);
        // 新建HashSet对象
        HashSet set = new HashSet();
        set.add(p1);
        set.add(p2);
        set.add(p3);
        // 比较p1 和 p2， 并打印它们的hashCode()
        System.out.printf("p1.equals(p2) : %s; p1(%d) p2(%d)\n", p1.equals(p2), p1.hashCode(), p2.hashCode());
        // 打印set
        System.out.printf("set:%s\n", set);
    }
    
    private static class Person {
        int age;
        String name;
        /**
         * @desc 覆盖equals方法
         */
        @Override
        public boolean equals(Object obj){
            if(obj == null){
                return false;
            }
            //如果是同一个对象返回true，反之返回false
            if(this == obj){
                return true;
            }
            //判断是否类型相同
            if(this.getClass() != obj.getClass()){
                return false;
            }
            Person person = (Person)obj;
            return name.equals(person.name) && age==person.age;
        }
    }
}
```
结果：

```java
p1.equals(p2) : true; p1(1169863946) p2(1690552137)
set:[(eee, 100), (eee, 100), (aaa, 200)]
```
我们重写了Person的equals()。但是，很奇怪的发现：HashSet中仍然有重复元素：p1 和 p2。为什么会出现这种情况呢？

这是因为虽然p1 和 p2的内容相等，但是它们的hashCode()不等；所以，HashSet在添加p1和p2的时候，认为它们不相等。

<span style="background: yellow;">Test2:</span>
下面，我们同时覆盖equals() 和 hashCode()方法。

```java
public class ConflictHashCodeTest2{

    public static void main(String[] args) {
        // 新建Person对象，
        Person p1 = new Person("eee", 100);
        Person p2 = new Person("eee", 100);
        Person p3 = new Person("aaa", 200);
        Person p4 = new Person("EEE", 100);
        // 新建HashSet对象
        HashSet set = new HashSet();
        set.add(p1);
        set.add(p2);
        set.add(p3);
        // 比较p1 和 p2， 并打印它们的hashCode()
        System.out.printf("p1.equals(p2) : %s; p1(%d) p2(%d)\n", p1.equals(p2), p1.hashCode(), p2.hashCode());
        // 比较p1 和 p4， 并打印它们的hashCode()
        System.out.printf("p1.equals(p4) : %s; p1(%d) p4(%d)\n", p1.equals(p4), p1.hashCode(), p4.hashCode());
        // 打印set
        System.out.printf("set:%s\n", set);
    }
    private static class Person {
        int age;
        String name;
        /**
         * @desc重写hashCode
         */
        @Override
        public int hashCode(){
            int nameHash =  name.toUpperCase().hashCode();
            return nameHash ^ age;
        }
        /**
         * @desc 覆盖equals方法
         */
        @Override
        public boolean equals(Object obj){
            if(obj == null){
                return false;
            }
            //如果是同一个对象返回true，反之返回false
            if(this == obj){
                return true;
            }
            //判断是否类型相同
            if(this.getClass() != obj.getClass()){
                return false;
            }
            Person person = (Person)obj;
            return name.equals(person.name) && age==person.age;
        }
    }
}
```
结果：

```java
p1.equals(p2) : true; p1(68545) p2(68545)
p1.equals(p4) : false; p1(68545) p4(68545)
set:[aaa - 200, eee - 100]
```
这下，equals()生效了，HashSet中没有重复元素。
- 比较p1和p2，我们发现：它们的hashCode()相等，通过equals()比较它们也返回true。所以，p1和p2被视为相等。
- 比较p1和p4，我们发现：虽然它们的hashCode()相等；但是，通过equals()比较它们返回false。所以，p1和p4被视为不相等。

## 面试题

***1、为什么要有 hashCode？***

我们以“HashSet 如何检查重复”为例子来说明为什么要有 hashCode？

当你把对象加入 HashSet 时，HashSet 会先计算对象的 hashcode 值来判断对象加入的位置，同时也会与其他已经加入的对象的 hashcode 值作比较，如果没有相符的 hashcode，HashSet 会假设对象没有重复出现。但是如果发现有相同 hashcode 值的对象，这时会调用 equals() 方法来检查 hashcode 相等的对象是否真的相同。如果两者相同，HashSet 就不会让其加入操作成功。如果不同的话，就会重新散列到其他位置。这样我们就大大减少了 equals 的次数，相应就大大提高了执行速度。

<hr/>

***2、为什么重写 equals 时必须重写 hashCode 方法？***

通俗的讲，注释中第二点和第三点的含义就是equals（）和hashcode（）方法要保持相当程度的一致性，**equals（）方法相等，hashcode（）必须相等；反之，equals方法不相等，hashcode可以相等，可以不相等**。但是两者的一致有利于提高哈希表的性能。

equals（）相等的的两个等价对象因为hashCode不同，所以在hashmap中的table数组的下标不同，从而这两个对象就会同时存在于集合中，在调用hashmap集合中的方法时就会出现逻辑的错误，也就是，你的equals（）方法也“白白”重写了。

因此，对于“为什么重写equals()就一定要重写hashCode()方法？”这个问题应该是有个前提，就是你需要用到HashMap,HashSet等Java集合。用不到哈希表的话，其实仅仅重写equals()方法也可以吧。而工作中的场景是常常用到Java集合，所以Java官方建议重写equals()就一定要重写hashCode()方法。

<hr/>

***3、为什么两个对象有相同的 hashcode 值，它们也不一定是相等的？***

因为 hashCode() 所使用的杂凑算法也许刚好会让多个对象传回相同的杂凑值。越糟糕的杂凑算法越容易碰撞，但这也与数据值域分布的特性有关（所谓碰撞也就是指的是不同的对象得到相同的 hashCode。

我们刚刚也提到了 HashSet,如果 HashSet 在对比的时候，同样的 hashcode 有多个对象，它会使用 equals() 来判断是否真的相同。也就是说 hashcode 只是用来缩小查找成本。
# final
final修饰类，表示类不可变，不可继承，比如：String

final修饰方法，表示方法不可以重写

final修饰变量，这个变量就是常量

注意：
- 修饰的是基本数据类型，这个值本身不可修改

- 修饰的引用类型，引用的<span style="background: yellow;">指向不可以更改</span>，内容可以更改
下面这个代码是<span style="background: yellow;">可以</span>的：

```java
final Student student = new Student(1,"Andy");
student.setAge(18);
```
