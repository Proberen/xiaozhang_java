## 18.1 Math
1、Math包含执行基本数字运算的方法

2、Math没有构造方法，如何使用类中的成员呢？
- 看类的成员是否都是静态的，如果是，通过类名就可以直接调用

<img src="https://img-blog.csdnimg.cn/20210312144420224.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

## 18.2 System
System类包含几个有用的类字段和方法，它不能被实例化

<img src="https://img-blog.csdnimg.cn/20210312144803924.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

## 18.3 Object
1、Object是类层次结构的根，每个类都可以将Object作为超类，所有类都直接或者间接的继承自Object

2、构造方法：	`public Object()`

在面向对象中，为什么说子类的构造方法默认访问父类的无参构造方法？
- 因为他们的顶级父类只有无参构造方法

### toString()

```java
//返回类的名字@实例的哈希码的16进制的字符串。建议Object所有的子类都重写这个方法。
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```
### equals()

```java
//用于比较2个对象的内存地址是否相等
//String类对该方法进行了重写用户比较字符串的值是否相等。
public boolean equals(Object obj) {
        return (this == obj);
    }
```

## 18.4 Arrays
### 18.4.1 冒泡排序
1、排序：将一组数据按照固定的规则进行排列

2、冒泡排序：一种排序的方式，对要进行排序的数据中相邻的数据进行两两比较，将较大的数据放在后面，依次对所有的数据进行操作，直至所有数据按要求完成排序


- 如果有n个数据进行排序，总共需要比较n-1次
- 每一次比较完毕，下一次的比较就会少一个数据参与

3、动画演示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210312160610180.gif)

4、代码

```java
main{
	//定义一个数组
	int[] arr = {24,69,80,57,13};
	System.out.println(arrayToString(arr));

    //第一次比较
    for(int i=0;i<arr.length-1;i++){
        if(arr[i] > arr[i+1]){
            int temp = arr[i];
            arr[i] = arr[i+1];
            arr[i+1] = temp;
        }
    }
    System.out.println(arrayToString(arr));

    //第二次比较
    for(int i=0;i<arr.length-1-1;i++){
        if(arr[i] > arr[i+1]){
            int temp = arr[i];
            arr[i] = arr[i+1];
            arr[i+1] = temp;
        }
    }
    System.out.println(arrayToString(arr));

    //第三次比较
    for(int i=0;i<arr.length-1-2;i++){
        if(arr[i] > arr[i+1]){
            int temp = arr[i];
            arr[i] = arr[i+1];
            arr[i+1] = temp;
        }
    }
    System.out.println(arrayToString(arr));

    //第四次比较
    for(int i=0;i<arr.length-1-3;i++){
        if(arr[i] > arr[i+1]){
            int temp = arr[i];
            arr[i] = arr[i+1];
            arr[i+1] = temp;
        }
    }
    System.out.println(arrayToString(arr));

	//优化
	for(int x = 0;x < arr.length-1;x++){
		for(int i = 0;i < arr.length-1-x;i++){
	        if(arr[i] > arr[i+1]){
	            int temp = arr[i];
	            arr[i] = arr[i+1];
	            arr[i+1] = temp;
	        }
	    }
	}
	System.out.println(arrayToString(arr));
}

//把数组中的元素按照指定的规则组成一个字符串
public static String arrayToString(int[] arr){
	StringBuilder sb = new StringBuilder();
	sb.append("[");
	for(int i = 0;i<arr.length;i++){
		if(i == arr.length-1){
			sb.append(arr[i]);
		}else{
			sb.append(arr[i]).append(",");
		}
	}
	sb.append("]");
	String s = sb.toString();
	return s;
}

```
最终冒泡排序的代码：

```java
//优化
for(int x = 0;x < arr.length-1;x++){
	for(int i = 0;i < arr.length-1-x;i++){
        if(arr[i] > arr[i+1]){
            int temp = arr[i];
            arr[i] = arr[i+1];
            arr[i+1] = temp;
        }
    }
}
System.out.println(arrayToString(arr));
```
### 18.4.2 Arrays类的概述和常用方法

Arrays类包含用于操作数组的各种方法

<img src="https://img-blog.csdnimg.cn/20210312161152401.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

```java
// toString(int[] a)
int[] arr = {24,69,80,57,13};
System.out.println(Arrays.toString(arr));
Arrays.sort(arr);
System.out.println(Arrays.toString(arr));
```
输出:

```java
[24,69,80,57,13]
[13,24,57,69,80]
```

有构造方法：`private Arrays(){}`

工具类的设计思想：
- 构造方法用`private`修饰
- 成员用`public` `static`修饰

## 18.5 基本类型包装类
### 18.5.1 概述

引入：

> 需求：判断一个数据是否在int范围内

```java
System.out.println(Integer.MIN_VALUE);
System.out.println(Integer.MAX_VALUE);
```
`java.lang.Integer`继承于`java.lang.Number`继承于`java.lang.Object`

1、将基本数据类型封装成对象的<span style="background: yellow;">好处</span>：可以在对象中定义更多的功能方法操作该数据

2、常用操作之一：用于基本数据类型与字符串之间的转换

### 18.5.2 Integer类的概述和使用
1、Integer：包装一个对象中的原始类型int的值

2、方法：
- `public static Integer valueOf(int i)`：返回表示指定int值的Integer实例
- `public static Integer valueOf(String s)`：返回保存指定值的Integer对象String

### 18.5.3 int和String的相互转换

```java
int number = 100;
//int->string
String s1 = "" + number;
//int->string
String s2 = String.valueOf(number);

String s = "100";
//string->int
Integer i = Integer.valueOf(s);
int i1 = i.intValue();
//string->int
int i2 = Integer.parseInt(s);
```

1、int转为String
- `public static String valueOf(int i)`

2、String转为int
- `public static int parseInt(String s)`

## 18.6 日期类
### 18.6.1 Date类

Date代表特定的时间，精确到毫秒

`public class Date extends Object implements Serializable,Cloneable,Comparable<Date>`

1、构造方法：
- `public Date()`：分配一个Date对象，并初始化，以便它代表它被分配的时间，精确到毫秒
- `public Date(long date)` 分配一个Date对象，将其初始化为从标准基准时间（January 1, 1970, 00:00:00 GMT.）起指定的毫秒数

例：

```java
main{
	//public Date():分配一个Date对象，并初始化，以便它代表它被分配的时间，精确到毫秒
	Date d1 = new Date() // Fri Mar 12 20:39:41 CST 2021

	//public Date(long date):分配一个Date对象，并将其初始化为从标准基准时间起指定的毫秒数
	long date = 1000*60*60;
	Date d2 = new Date(date);// Thu Jan 01 09:00:00 CST 1970
	
	
}
```

2、常用方法
- `public long getTime()`：获取日期对象从1970年1月1日0点到现在的毫秒值
- `public void setTime(long time)`：设置时间，给的是毫秒值

例：

```java
main{
	//创建日期对象
	Date d = new Date();
	
	//public long getTime()
	d.getTime(); // 1615553049497毫秒
	
	//public void setTime(long time)
	d.setTime(0); //Thu Jan 01 08:00:00 CST 1970 由于时区不一样，所以结果不一样
}
```

### 18.6.2 SimpleDateFormat类

`public class SimpleDateFormat extends DateFormat`

1、SimpleDateFormat是一个具体类，用于以区域设置敏感的方式格式化和解析日期

2、常用的模式字母及对应关系：
- y	年
- M	月
- d	日
- H	时
- m	分
- s	秒

3、构造方法：
- `public SimpleDateFormat()`：使用默认模式和日期格式
- `public SimpleDateFormat(String pattern)`：使用给的的模式和日期格式

4、格式化和解析日期方法
- 格式化（Date到String）：`public final String format(Date date)`
- 解析（String到Date）：`public Date parse(String source)`

例：

```java
//格式化
Date d = new Date();

SimpleDateFormat sdf = new SimpleDateFormat();
String s = sdf.format(d);
System.out.println(s); //21-3-12 下午8:54

SimpleDateFormat sdf1 = new SimpleDateFormat("yyyy-MM-dd-HH-mm-ss");
String s = sdf1.format(d);
System.out.println(s); //2021-03-12-20-55-49

//从String到Date
String ss = "2021-03-12-20-55-49";
SimpleDateFormat sdf2 = new SimpleDateFormat("yyyy-MM-dd-HH-mm-ss");
Date dd = sdf2.parse(ss); //这里需要处理异常 throws ParseException
System.out.println(dd); //Fri Mar 12 20:55:49 CST 2021
```
### 18.6.3 Calendar类
Calendar类为某一时刻和一组日历字段之间的转换提供了一些方法，并为操作日历字段提供了一些方法

```java
//获取对象
Calendar c = Calendar.getInstance(); //多态的形式

//public int get(int field)
int year = c.get(Calendar.YEAR);
int month = c.get(Calendar.MONTH)+1;//从0开始的
int day = c.get(Calendar.DATE);
```
1、常用方法：
- `public int get(int field)`：返回给定日历字段的值
- `public abstract void add(int field,int amount)`：根据日历的规则，将指定的时间量添加或减去给定的日历字段
- `public final void set(int year,int month,int date)`：设置当前日历的年月日

```java
//获取对象
Calendar c = Calendar.getInstance(); 

//public abstract void add(int field,int amount)
c.add(Calendar.YEAR,1);//年份+1
System.out.println(c.get(Calendar.YEAR)); // 2022

//public final void set(int year,int month,int date)
c.set(2020,1,1);
int year = c.get(Calendar.YEAR);
int month = c.get(Calendar.MONTH)+1;//从0开始的
int day = c.get(Calendar.DATE);
System.out.println(year+"-"+month+"-"+day) // 2020-2-1
```
