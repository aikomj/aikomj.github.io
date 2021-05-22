---
layout: post
title: String长度有限制吗？是多少？jdk中那些容易坑你的地方
category: java
tags: [java]
keywords: java
excerpt: 开发这么久了，我还真没想过这个问题
lock: noneed
---

## 前言

本人就遇到过面试的时候问这个的，而且在之前开发的中也真实地遇到过这个String长度限制的场景（将某固定文件转码成Base64的形式用字符串存储，在运行时需要的时候在转回来，当时文件比较大），那这个规范限制到底是怎么样的。

## 1、String

String其实是使用的一个char类型的数组来存储字符串中的字符的。点击源码

![](\assets\images\2020\java\string-1.jpg)

那么String既然是数组存储那数组会有长度的限制吗？是的有限制，我们看看String中返回length的方法。

![](\assets\images\2020\java\string-2.jpg)

由此我们看到返回值类型是int类型，Java中定义数组是可以给数组指定长度的，当然不指定的话默认会根据数组元素来指定：

```java
int[] arr1 = new int[10]; // 定义一个长度为10的数组
int[] arr2 = {1,2,3,4,5}; // 那么此时数组的长度为5
```

整数在java中是有限制的，我们通过源码来看看int类型对应的包装类Integer可以看到，其长度最大限制为2^31 -1，那么说明了数组的长度是0~2^31-1，那么计算一下就是（2^31-1 = 2147483647 = 4GB）

但实际容量没有那么大，

![](\assets\images\2020\java\string-4.jpg)

以上是我通过定义字面量的形式构造的10万个字符的字符串，编译之后虚拟机提示报错，说我们的字符串长度过长，不是说好了可以存21亿个吗？为什么才10万个就报错了呢？

其实这里涉及到了JVM编译规范的限制了，JVM在编译时，如果我们将字符串定义成了字面量的形式，编译时JVM是会将其存放在常量池中，这时候JVM对这个常量池存储String类型做出了限制，接下来我们先看下手册是如何说的。

![](\assets\images\2020\java\string-5.jpg)

常量池中，每个 cp_info 项的格式必须相同，它们都以一个表示 cp_info 类型的单字节 “tag”项开头。后面 info[]项的内容由tag 的类型所决定，常量类型如下

![](\assets\images\2020\java\string-6.jpg)

我们可以看到 String类型的表示是 CONSTANT_String ，我们来看下CONSTANT_String具体是如何定义的。

![](\assets\images\2020\java\string-7.jpg)

string_index 表示的是常量池的有效索引，其类型是CONSTANT_Utf8_info 结构体表示的。

![](\assets\images\2020\java\string-8.jpg)

在class文件中u2表示的是无符号数占2个字节单位，我们知道1个字节占8位，2个字节就是16位 ，那么2个字节能表示的范围就是2^16- 1 = 65535 。

说明：

1、class文件中文件内容类型解释

定义一组私有数据类型来表示 Class 文件的内容，它们包括 u1，u2 和 u4，分别代 表了 1、2 和 4 个字节的无符号数。每个 Class 文件都是由 8 字节为单位的字节流组成，所有的 16 位、32 位和 64 位长度的数据将被构造成 2 个、4 个和 8 个 8 字节单位来表示。

那么2个字节能表示的范围就是2^16- 1 = 65535。所以 length的最大值是65535。

2、程序异常处理的有效范围解释

start_pc 和 end_pc 两项的值表明了异常处理器在 code[]数组中的有效范围。

start_pc 必须是对当前 code[]数组中某一指令的操作码的有效索引，end_pc 要 么是对当前 code[]数组中某一指令的操作码的有效索引，要么等于 code_length 的值，即当前 code[]数组的长度。start_pc 的值必须比 end_pc 小。

当程序计数器在范围`[start_pc, end_pc)`内时，异常处理器就将生效。即设 x 为 异常句柄的有效范围内的值，x 满足：`start_pc ≤ x < end_pc`。

实际上，end_pc 值本身不属于异常处理器的有效范围这点属于 Java 虚拟机历史上 的一个设计缺陷：如果 Java 虚拟机中的一个方法的 code 属性的长度刚好是 65535 个字节，并且以一个 1 个字节长度的指令结束，那么这条指令将不能被异常处理器 所处理。

不过编译器可以通过限制任何方法、实例初始化方法或类初始化方法的`code[]`数组最大长度为 65534，这样可以间接弥补这个 BUG。

**总结：**

注意：这里对个人认为比较重要的点做了标记，首先第一个加粗说白了就是说数组有效范围就是【0-65565】但是第二个加粗的地方又解释了，因为虚拟机还需要1个字节的指令作为结束，所以其实真正的有效范围是【0-65564】，这里要注意这里的范围仅限编译时期，如果你是运行时拼接的字符串是可以超出这个范围的。

测试：

![](\assets\images\2020\java\string-9.jpg)

> 总结

**问：字符串有长度限制吗？是多少？**

答：首先字符串的内容是由一个字符数组 char[] 来存储的，由于数组的长度及索引是整数，且String类中返回字符串长度的方法length() 的返回值也是int ，所以通过查看java源码中的类Integer我们可以看到Integer的最大范围是2^31 -1,由于数组是从0开始的，所以数组的最大长度可以使【0~2^31】通过计算是大概4GB。

但是通过翻阅java虚拟机手册对class文件格式的定义以及常量池中对String类型的结构体定义我们可以知道对于索引定义了u2，就是无符号占2个字节，2个字节可以表示的最大范围是2^16 -1 = 65535。但是由于JVM需要1个字节表示结束指令，所以这个范围就为65534了。超出这个范围在编译时期是会报错的，但是运行时拼接或者赋值的话范围是在整形的最大范围。

<mark>所以，String的最大长度是65534</mark>

## 2、String.valueOf()的陷阱

看源码，我们知道对象为null时会返回`null`字符串

```java
public static String valueOf(Object obj) {
  return (obj == null) ? "null" : obj.toString();
}
```

场景代码：

```java
// 调用用户服务根据用户id获取用户信息
Map<String, Object> userInfo = userService.getUserInfoById(userId);
Object userNameObject = userInfo.get("name");
String userName = String.valueOf(userNameObject);
// 判空
if(userName!=null&&userName.length()>0) {
    String message = getMessage(userName);
    smsService.send(message);
}
```

问题点：当userNameObject为null时，userName就是`null`字符串了，符合发非空判断，发出短信<font color="red">尊敬的null 你好,xxx等</font>，必然会收到客户的投诉。正确的写法应该是

```java
Object userNameObject = userInfo.get("name");
// 判空
if(userNameObject!=null) {
    String message = getMessage(userNameObject.toString());
    smsService.send(message);
}
```

`String.valueOf()`要看具体场景使用。

特殊场景：数据库字符格式无法兼容emoji，当用户名字带有emoji表情符号，会导致数据写入失败。

## 3、Integer.parseInt()的陷阱

**事故现场**

一次业务场景为拉取订单,打出订单列表记录,财务人员需要拉出对账,结果总是发现很奇怪的一个现象,每次拉取少很多数据,。还好财务发现了,要不然和第三方财务对账就会亏很多钱...最终发现是订单的一个字段值转Integer出错了,那个订单下的字段值是120.0通过Integer.parseInt()直接报错了,恰好开发人员认为这段开发肯定没问题,因此就没有catch异常,最后找了很久才发现(因为涉及到第三方,还让别人查了半天...). 知道真相的我们都有点汗颜,这么丁点的错误排查了很久,实在是不应该啊。

`Integer.parseInt()`方法用于将字符串转化为Integer类型的方法,当传入50.0、20L、30d、40f这类数据的情况下，它都会报`NumberFormatException`异常，事实上我们应该自动转换成功的，而不是抛出异常，怎么做到？推荐使用<mark>hutool的NumberUtil.parseInt()方法</mark>,充分考虑到了浮点、long、小数等类型数据可能带来的解析异常的问题。

```java
public static void main(String[] args) {
  String input="50.0"; 
  System.out.println(NumberUtil.parseNumber(input)); // parseInt会取整
  System.out.println("-----我是分割线------");
  System.out.println(Integer.parseInt(input));
}
```

![](\assets\images\2021\javabase\number-util-integer-parseInt.jpg)

```java
public static void main(String[] args) {
  String input="20L"; // 30d,40f
  System.out.println(NumberUtil.parseNumber(input));
  System.out.println("-----我是分割线------");
  System.out.println(Integer.parseInt(input));
}
```

![](\assets\images\2021\javabase\number-util-integer-parseInt-2.jpg)

![](\assets\images\2021\javabase\number-util-integer-parseInt-3.jpg)

```java
public static void main(String[] args) {
  String input="50.2f";
  System.out.println(NumberUtil.parseInt(input)); // 使用parseInt会取整
  System.out.println("-----我是分割线------");
  System.out.println(Integer.parseInt(input));
}
```

![](\assets\images\2021\javabase\number-util-integer-parseInt-4.jpg)

## 4、Bigdecimal.divide()的陷阱

BigDecimal是处理金额最有效的数据类型,一般进行财务报表计算的时候为了防止金额出现错误,都会采用Bigdecimal，而double、float都会存在些许的误差。但在做除法的时候要注意

```java
BigDecimal one = new BigDecimal("10");
BigDecimal two = new BigDecimal("2");
System.out.println(one.divide(two));
```

一般情况下没毛病，如果都是来自前端的参数，回传10,3 会怎样？

```java
public static void main(String[] args) {
  BigDecimal one = new BigDecimal("10");
  BigDecimal two = new BigDecimal("3");
  System.out.println(one.divide(two));
}
```

会抛出`ArithmeticException`异常

![](\assets\images\2021\javabase\bigdecimal-arithmetic-exception.jpg)

这是因为BidDecimal一旦返回的结果是无限循环小数,就会抛出ArithmeticException。正确的姿势应该是要做保留小数点处理

```java
public static void main(String[] args) {
  BigDecimal one = new BigDecimal("10");
  BigDecimal two = new BigDecimal("3");
  System.out.println(one.divide(two,2,BigDecimal.ROUND_HALF_UP));
}
```

![](\assets\images\2021\javabase\bigdecimal-arithmetic-exception-2.jpg)

## 5、Collections.emptyList()的陷阱

```java
public static void main(String[] args) {
  List<String> list = Collections.emptyList();
  list.add("hello");
}
```

执行结果

![](\assets\images\2021\javabase\empty-list.jpg)

不支持add、remove等操作，那是因为Collections.emptyList()返回的是一个内部类EmptyList，该类并具有add、remove等操作

![](\assets\images\2021\javabase\empty-list-2.jpg)

就跟`Arrays.asList()`返回的ArrayList 也是自定义的内部类，不是`java.util.ArrayList`

![](\assets\images\2021\javabase\arrays-aslist-7.png)

不支持添加、删除元素操作，并不是`java.util.List`

## 6、list一边遍历一边删除

可以，但必须使用`Iterator.remove`迭代器的方式删除，规避异常

![](\assets\images\2021\javabase\list-concurrent-modification-exception.png)

仔细翻阅源码会发现,每次remove之前会检查元素的条数,如果发现预期的modCount和当前的modCount不一致就会抛出这个异常.modCount是list中用来记录修改次数的一个属性,当对元素进行统计的时候就会对该元素加1,而当对list边遍历边删除的话,就会造成excepted与modCount不一致，从而抛出异常。

```java
final void checkForComodification() {
  if (modCount != expectedModCount)
    throw new ConcurrentModificationException();
}
```

正确写法

```java
List<Integer> list = new ArrayList<>();
list.add(1);
list.add(2);
list.add(3);
list.add(4);
Iterator<Integer> iterator = list.iterator();
while (iterator.hasNext()) {
    Integer integer = iterator.next();
    if (integer == 2) {
        iterator.remove();
    }
}
```



