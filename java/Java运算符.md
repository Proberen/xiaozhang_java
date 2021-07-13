> 基本概念：
> - 运算符：对常量或者变量进行操作的符号
> 
> - 表达式：用运算符把常量或者变量连接起来符合Java语法的式子可以称为表达式，不同运算符连接的表达式体现的是不同类型的表达式

## 3.1 算术运算符
1、算数运算符：加号（+）、减号（-）、除号（/）、乘号（*）以及取模操作符（%）

2、注意：
- 整数相除只能得到整数，若想得到小数，必须有浮点数参与运算

### 字符的+操作

```java
int i = 10;
char c = 'A' //65
c = 'a' //97
c = '0' //48
```
字符参与加操作，拿字符在计算机底层的<span style="background: yellow;">对应数值</span>来计算

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210308164511281.png)

算术表达式中包含多个基本数据类型的值的时候，整个算术表达式的类型会进行<span style="background: yellow;">自动提升</span>

提升规则：
- byte类型、short类型、char类型将被提升到int类型
- 整个表达式的类型自动提升到表达式中最高等级的类型，顺序：byte、short、char -> int -> long -> float -> double

### 字符串的+操作

```java
"zhang"+"tao" //输出zhangtao
"zhang"+666  //输出zhang666
"zhang"+7+66 //输出zhang766
1+99+"zhang" //输出100zhang
```
当“+”操作出现字符串时，这个+是<span style="background: yellow;">字符串连接符</span>，而不是算术运算
- `"zhang"+7+66`：如果出现了字符串，就是连接运算符，否则是算术运算；当连续进行+操作时，从左到右逐个执行
- `1+99+"zhang"`：从左到右逐个执行

## 3.2 赋值运算符

<img src="https://img-blog.csdnimg.cn/20210308170457637.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

注意：隐含<span style="background: yellow;">强制类型转换</span>

## 3.3 自增自减运算符

<img src="https://img-blog.csdnimg.cn/20210308170530678.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NjUwODk5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />


- 前缀递增和前缀递减（如++a或–a），会先执行运算，再生成值

- 后缀递增和后缀递减（如a++或a–），会先生成值，再执行运算

⚠️注意：
- ++和--可以放在变量的后边，也可以放在变量的前边
- <span style="background: yellow;">单独使用</span>，放在前边和后边结果一样（<span style="background: yellow;">最常见的用法==）
- <span style="background: yellow;">参与操作</span>时，如果放在变量后边，先拿变量进行操作，后拿变量做++或者--；如果放在变量前边，先拿变量做++或者--，后进行操作
例如：
```java
int i =10
int j = i++ 
// 结果：j = 10,i =11
```
## 3.4 关系运算符
关系操作符：比较两个变量之间的关系，生成一个boolean值（true or false）

| 操作符 | 含义       |
| ------ | ---------- |
| >      | 大于       |
| >=     | 大于或等于 |
| <      | 小于       |
| <=     | 小于或等于 |
| ==     | 等于       |
| !=     | 不等       |

## 3.5 逻辑运算符

逻辑运算符，是用来连接关系表达式的运算符，也可以直接连接布尔类型的常量或者变量

| 关键字     | 简介             | 具体含义                                                     |
| ---------- | ---------------- | :----------------------------------------------------------- |
| &、 &&  | 长路与 、短路与   | 相同点：两边都为真时为真；任意一个为假结果为假；     不同点：长路与两侧都会被运算；短路与如果第一个为假则结果为假 |
| ｜、｜｜ | 长路或  、 短路或 | 相同点：两边都为假时为假；任意一个为真结果为真；     不同点：长路与两侧都会被运算；短路与如果第一个为真则结果为真 |
| ！         | 取反             | 真变假；假变真                                               |
| ^          | 异或             | 不同时返回真；相同时返回假                                   |

最常用的逻辑运算符：`&&`	、`||`、`!`

## 3.6 三元运算符

格式：`关系表达式？表达式1:表达式2`

规则：首先计算关系表达式的值，如果为true，表达式1则为运算结果，反之表达式2为运算结果

### 案例：两只老虎

> 需求：动物园有两只老虎，分别是180kg和200kg，用程序实现判断两只老虎的体重是否相同

代码：
```java
int weight1 = 180;
int weight2 = 200;
System.out.println(weight1==weight2?true:false);
```
### 案例：三个和尚

> 需求：有三个和尚，身高分别为150cm、210cm、165cm，用程序获取三个和尚的最高身高

分析：
- 定义三个变量保存身高
- 用三元运算符获取前两个身高的较高身高，用临时变量保存
- 用三元运算符获得临时身高与第三个和尚的身高的较高身高

```java
 int height1 = 150;
 int height2 = 210;
 int height3 = 165;
 int temp  = height1>height2 ? height1:height2;
 int max = temp>height3 ?temp:height3;
 System.out.println(max);
```
