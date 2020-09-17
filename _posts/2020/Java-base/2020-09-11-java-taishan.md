---
layout: post
title: 阿里Java开发手册泰山版-个人精简
category: java
tags: [java]
keywords: java
excerpt: 会当凌绝顶，一览众山小
lock: noneed
---

![](/assets/images/2020/icoding/scrum/java-taishan.jpg)

## 1、编程规约

### （1）命名风格

1. <font color = red>【强制】</font>POJO类中的任何布尔类型的变量，都不要加is前缀，否则部分框架解析会引起序列化错误。

2. <font color=red>【强制】</font>所有编程相关的命名严禁使用拼音与英文混合的方式，更不允许直接使用中文的方式，命名要易于理解，避免歧义

   ```sh
   正例：ali/alibaba /taobao / cainiao/ aliyun/ youku / hangzhou 等国际通用的名称，可视同英文
   ```

3. <font color=red>【强制】</font>类名使用UpperCamelCase风格，但以下情形例外：DO / BO / DTO / VO/ AO/ PO/ UID等。

   ```sh
   正例：ForceCode    /    UserDO    /    HtmlDTO    /    XmlService    /    TcpUdpDeal / TaPromotion
   反例：forcecode    /    UserDo    /    HTMLDto    /    XMLService    /   TCPUDPDeal / TAPromotion
   ```

4. <font color=red>【强制】</font>方法名、参数名、成员变量、局部变量都统一使用lowerCamelCase风格。

   ```sh
   正例：localValue / getHttpMessage() / inputUserId
   ```

5. <font color=red>【强制】</font>抽象类命名使用Abstract或Base开头；异常类命名使用Exception结尾；测试类命名以它要测试的类的名称开始，以Test结尾。

6. <font color=red>【强制】</font>POJO类中的任何布尔类型的变量，都不要加is前缀，否则部分框架解析会引起序列化错误。

7. <font color=red>【强制】</font>包名统一使用小写，点分隔符之间有且仅有一个自然语义的英语单词。包名统一使用单数形式，但是类名如果有复数含义，类名可以使用复数形式。

   ```sh
   正例：应用工具类包名为com.alibaba.ei.kunlun.aap.util、类名为MessageUtils（此规则参考spring的框架结构）
   ```

8. <font color='red'>【强制】</font>杜绝完全不规范的缩写，避免望文不知义。

   ```sh
   反例：AbstractClass“缩写”命名成AbsClass；condition“缩写”命名成condi，此类随意缩写严重降低了代码的可阅读性。
   ```

9. <font color='\#FFB800'>【推荐】</font>为了达到代码自解释的目标，任何自定义编程元素在命名时，使用尽量完整的单词组合来表达。

   ```sh
   正例：在JDK中，对某个对象引用的volatile字段进行原子更新的类名为：AtomicReferenceFieldUpdater。反例：常见的方法内变量为int a;的定义方式。
   ```



### （2）常量定义



### （3）代码格式



### （4）OOP规约



### （5）日期时间



### （6）集合处理



### （7）并发处理



### （8）控制语句



### （9）注释规约



## 2、异常日志

### （1）错误码



### （2）异常处理



### （3）日志规约



## 3、单元测试



## 4、安全规约



## 5、Mysql数据库

### （1）建表规约



### （2）索引规约



### （3）SQL语句



### （4）ORM映射



## 6、工程结构

### （1）应用分层



### （2）二方库依赖



### （3）服务器



## 7、设计规约



## 附3 错误码列表







