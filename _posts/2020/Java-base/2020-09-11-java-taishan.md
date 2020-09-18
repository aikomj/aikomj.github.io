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

10. <font color='\#FFB800'>【推荐】</font>在常量与变量的命名时，表示类型的名词放在词尾，以提升辨识度。 

    ```sh
    正例：startTime / workQueue / nameList / TERMINATED_THREAD_COUNT 
    ```

11. <font color='\#FFB800'>【推荐】</font>如果模块、接口、类、方法使用了设计模式，在命名时需体现出具体模式，有利于阅读者快速理解架构设计理念

    ```sh
    正例： public class OrderFactory; 
           public class LoginProxy; 
           public class ResourceObserver
    ```

12. <font color='\#FFB800'>【推荐】</font>接口类中的方法和属性不要加任何修饰符号（public 也不要加），保持代码的简洁性，并加上有效的Javadoc注释。尽量不要在接口里定义变量，如果一定要定义变量，确定与接口方法相关，并且是整个应用的基础常量。

    ```sh
    正例：接口方法签名 void commit(); 
    接口基础常量 String COMPANY = "alibaba";
    ```

13. <font color='red'>【强制】</font>对于Service和DAO类，基于SOA的理念，暴露出来的服务一定是接口，内部的实现类用Impl的后缀与接口区别。 

    ```sh
    正例：CacheServiceImpl实现CacheService接口。 
    ```

     <font color="#FFB800">【推荐】</font>如果是形容能力的接口名称，取对应的形容词为接口名（通常是–able的形容词）。 

    ```sh
    正例：AbstractTranslator实现 Translatable接口。 
    ```

14.  <font color="#FFB800">【推荐】</font>枚举类名带上Enum后缀，枚举成员名称需要全大写，单词间用下划线隔开。 
15.  <font color="#FFB800">【推荐】</font>各层命名规约
    - Service/DAO层方法命名规约 
         1） 获取单个对象的方法用get做前缀。 
         2） 获取多个对象的方法用list做前缀，复数结尾，如：listObjects。 
         3） 获取统计值的方法用count做前缀。 
         4） 插入的方法用save/insert做前缀。 
         5） 删除的方法用remove/delete做前缀。 
         6） 修改的方法用update做前缀。
    - 领域模型命名规约 
         1） 数据对象：xxxDO，xxx即为数据表名。 
         2） 数据传输对象：xxxDTO，xxx为业务领域相关的名称。 
         3） 展示对象：xxxVO，xxx一般为网页名称。 
         4） POJO是DO/DTO/BO/VO的统称，禁止命名成xxxPOJO。 

### （2）常量定义

1. <font color='red'>【强制】</font>在long或者Long赋值时，数值后使用大写的L，不能是小写的l，小写容易跟数字
   混淆，造成误解。 

   ```sh
   反例：Long a = 2l; 写的是数字的21，还是Long型的2。 
   ```

2. <font color="#FFB800">【推荐】</font>不要使用一个常量类维护所有常量，要按常量功能进行归类，分开维护，大而全的常量类，杂乱无章，使用查找功能才能定位到修改的常量，不利于理解，也不利于维护

   ```sh
   正例：缓存相关常量放在类CacheConsts下；系统配置相关常量放在类ConfigConsts下。
   ```

3. <font color="#FFB800">【推荐】</font> 如果变量值仅在一个固定范围内变化用enum类型来定义。

   ```java
   // 季节
   public enum SeasonEnum { 
       SPRING(1), SUMMER(2), AUTUMN(3), WINTER(4); 
    
       private int seq; 
       SeasonEnum(int seq) { 
           this.seq = seq; 
       } 
       public int getSeq() { 
          return seq; 
       } 
   } 
   ```

### （3）代码格式

1. <font color=red>【强制】</font>如果是大括号内为空，则简洁地写成{}即可，大括号中间无需换行和空格；如果是非空代码块则： 
    1） 左大括号前不换行。 
    2） 左大括号后换行。 
    3） 右大括号前换行。 
    4） 右大括号后还有else等代码则不换行；表示终止的右大括号后必须换行。

2. <font color=red>【强制】</font>if/for/while/switch/do等保留字与括号之间都必须加空格。采用4个空格缩进，禁止使用tab字符（因为tab在不同编辑器的长度不一）。 如果使用tab缩进，必须设置1个tab为4个空格。IDEA设置tab为4个空格时（默认4个空格），请勿勾选Use tab character；而在eclipse中，必须勾选insert spaces for tabs。 

   ![](\assets\images\tools\idea-tab-4-space.jpg)

   ```java
   // 正例： （涉及1-5点） 
   public static void main(String[] args) { 
       // 缩进4个空格 
       String say = "hello"; 
       // 运算符的左右必须有一个空格 
       int flag = 0; 
       // 关键词if与括号之间必须有一个空格，括号内的f与左括号，0与右括号不需要空格 
       if (flag == 0) { 
           System.out.println(say); 
       } 
    
       // 左大括号前加空格且不换行；左大括号后换行 
       if (flag == 1) { 
           System.out.println("world"); 
           // 右大括号前换行，右大括号后有else，不用换行 
       } else { 
           System.out.println("ok"); 
           // 在右大括号后直接结束，则必须换行 
       } 
   } 
   ```

   

3. <font color=red>【强制】</font>任何二目、三目运算符的左右两边都需要加一个空格。 

4. <font color=red>【强制】</font>注释的双斜线与注释内容之间有且仅有一个空格。 

    ```java
    // 正例： 这是示例注释，请注意在双斜线之后有一个空格 
    String commentString = new String(); 
    ```

5. <font color=red>【强制】</font>单行字符数限制不超过120个，超出需要换行，换行时遵循如下原则： 

   1）第二行相对第一行缩进4个空格，从第三行开始，不再继续缩进，参考示例。 
    2）运算符与下文一起换行。 
    3）方法调用的点符号与下文一起换行。 
    4）方法调用中的多个参数需要换行时，在逗号后进行。 
    5）在括号前不要换行，见反例。 

   ```java
   正例： 
   StringBuilder sb = new StringBuilder(); 
   // 超过120个字符的情况下，换行缩进4个空格，并且方法前的点号一起换行  
   sb.append("zi").append("xin")... 
           .append("huang")... 
           .append("huang")... 
           .append("huang"); 
   反例： 
   StringBuilder sb = new StringBuilder(); 
   // 超过120个字符的情况下，不要在括号前换行  
   sb.append("you").append("are")...append 
       ("lucky"); 
   // 参数很多的方法调用可能超过120个字符，逗号后才是换行处  
   method(args1, args2, args3, ... 
       , argsX);  
   ```

6. <font color="#FFB800">【推荐】</font>单个方法的总行数不超过80行

   正例：代码逻辑分清红花和绿叶，个性和共性，绿叶逻辑单独出来成为额外方法，使主干代码更加清晰；共性逻辑抽取成为共性方法，便于复用和维护。 

7. <font color="#FFB800">【推荐】</font>不同逻辑、不同语义、不同业务的代码之间插入一个空行分隔开来以提升可读性。 说明：任何情形，没有必要插入多个空行进行隔开

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







