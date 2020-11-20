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

5. <font color=red>【强制】</font>POJO类中的任何布尔类型的变量，都不要加is前缀，否则部分框架解析会引起序列化错误。

   ```sh
   说明：在本文MySQL规约中的建表约定第一条，表达是与否的值采用is_xxx的命名方式，所以，需要在
   <resultMap>设置从is_xxx到xxx的映射关系。 
   反例：定义为基本数据类型Boolean isDeleted的属性，它的方法也是isDeleted()，框架在反向解析的时
   候，“误以为”对应的属性名称是deleted，导致属性获取不到，进而抛出异常。
   ```

   

6. <font color=red>【强制】</font>包名统一使用小写，点分隔符之间有且仅有一个自然语义的英语单词。包名统一使用单数形式，但是类名如果有复数含义，类名可以使用复数形式。

   ```sh
   正例：应用工具类包名为com.alibaba.ei.kunlun.aap.util、类名为MessageUtils（此规则参考spring的框架结构）
   ```

7. <font color='red'>【强制】</font>杜绝完全不规范的缩写，避免望文不知义。

   ```sh
   反例：AbstractClass“缩写”命名成AbsClass；condition“缩写”命名成condi，此类随意缩写严重降低了代码的可阅读性。
   ```

8. <font color='\#FFB800'>【推荐】</font>为了达到代码自解释的目标，任何自定义编程元素在命名时，使用尽量完整的单词组合来表达。

   ```sh
   正例：在JDK中，对某个对象引用的volatile字段进行原子更新的类名为：AtomicReferenceFieldUpdater。反例：常见的方法内变量为int a;的定义方式。
   ```

9. <font color='\#FFB800'>【推荐】</font>在常量与变量的命名时，表示类型的名词放在词尾，以提升辨识度。 

   ```sh
   正例：startTime / workQueue / nameList / TERMINATED_THREAD_COUNT 
   ```

10. <font color='\#FFB800'>【推荐】</font>如果模块、接口、类、方法使用了设计模式，在命名时需体现出具体模式，有利于阅读者快速理解架构设计理念

    ```sh
    正例： public class OrderFactory; 
           public class LoginProxy; 
           public class ResourceObserver
    ```

11. <font color='\#FFB800'>【推荐】</font>接口类中的方法和属性不要加任何修饰符号（public 也不要加），保持代码的简洁性，并加上有效的Javadoc注释。尽量不要在接口里定义变量，如果一定要定义变量，确定与接口方法相关，并且是整个应用的基础常量。

    ```sh
    正例：接口方法签名 void commit(); 
    接口基础常量 String COMPANY = "alibaba";
    ```

12. <font color='red'>【强制】</font>对于Service和DAO类，基于SOA的理念，暴露出来的服务一定是接口，内部的实现类用Impl的后缀与接口区别。 

    ```sh
    正例：CacheServiceImpl实现CacheService接口。 
    ```

     <font color="#FFB800">【推荐】</font>如果是形容能力的接口名称，取对应的形容词为接口名（通常是–able的形容词）。 

    ```sh
    正例：AbstractTranslator实现 Translatable接口。 
    ```

13.  <font color="#FFB800">【推荐】</font>枚举类名带上Enum后缀，枚举成员名称需要全大写，单词间用下划线隔开。 

14.  <font color="#FFB800">【推荐】</font>各层命名规约

     Service/DAO层方法命名规约 
     1） 获取单个对象的方法用get做前缀。 
     2） 获取多个对象的方法用list做前缀，复数结尾，如：listObjects。 
     3） 获取统计值的方法用count做前缀。 
     4） 插入的方法用save/insert做前缀。 
     5） 删除的方法用remove/delete做前缀。 
     6） 修改的方法用update做前缀。

     领域模型命名规约 
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
   正例:
   StringBuilder sb = new StringBuilder(); 
   // 超过120个字符的情况下，换行缩进4个空格，并且方法前的点号一起换行  
   sb.append("zi").append("xin")... 
           .append("huang")... 
           .append("huang")... 
           .append("huang"); 
   反例:
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

1. <font color=red>【强制】</font>所有的覆写方法，必须加@Override注解。 
   说明：getObject()与get0bject()的问题。一个是字母的O，一个是数字的0，加@Override可以准确判断是否覆盖成功。另外，如果在抽象类中对方法签名进行修改，其实现类会马上编译报错。 

2. <font color=red>【强制】</font>外部正在调用或者二方库依赖的接口，不允许修改方法签名，避免对接口调用方产生影响。接口过时必须加@Deprecated注解，并清晰地说明采用的新接口或者新服务是什么。尽量不使用过时方法

3. <font color=red>【强制】</font>所有整型包装类对象之间值的比较，全部使用equals方法比较。 
   说明：对于Integer var = ? 在-128至127之间的赋值，Integer对象是在 IntegerCache.cache产生，会复用已有对象，这个区间内的Integer值可以直接使用==进行判断，但是这个区间之外的所有数据，都会在堆上产生，并不会复用已有对象，这是一个大坑，推荐使用equals方法进行判断

4. <font color=red>【强制】</font>浮点数之间的等值判断，基本数据类型不能用==来比较，包装数据类型不能用equals来判断。

   ```java
   说明：浮点数采用“尾数+阶码”的编码方式，类似于科学计数法的“有效数字+指数”的表示方式。二进
   制无法精确表示大部分的十进制小数，具体原理参考《码出高效》。 
   反例： 
   float a = 1.0f - 0.9f; 
   float b = 0.9f - 0.8f; 
   if (a == b) { 
       // 预期进入此代码快，执行其它业务逻辑 
       // 但事实上a==b的结果为false 
   } 
   Float x = Float.valueOf(a); 
   Float y = Float.valueOf(b); 
   if (x.equals(y)) { 
       // 预期进入此代码快，执行其它业务逻辑 
       // 但事实上equals的结果为false 
   } 
   正例： 
   (1) 指定一个误差范围，两个浮点数的差值在此范围之内，则认为是相等的。 
   float a = 1.0f - 0.9f; 
   float b = 0.9f - 0.8f; 
   float diff = 1e-6f; 
   if (Math.abs(a - b) < diff) { 
       System.out.println("true"); 
   } 
    (2) 使用BigDecimal来定义值，再进行浮点数的运算操作。 
   BigDecimal a = new BigDecimal("1.0"); 
   BigDecimal b = new BigDecimal("0.9"); 
   BigDecimal c = new BigDecimal("0.8"); 
   BigDecimal x = a.subtract(b); 
   BigDecimal y = b.subtract(c); 
   if (x.equals(y)) { 
       System.out.println("true"); 
   } 
   ```

5.  <font color=red>【强制】</font>禁止使用构造方法BigDecimal(double)的方式把double值转化为BigDecimal对象。 

   ```java
   说明:BigDecimal(double)存在精度损失风险,在精确计算或值比较的场景中可能会导致业务逻辑异常
   如:BigDecimal g = new BigDecimal(0.1f); 实际的存储值为:0.10000000149 
   正例:优先推荐入参为String的构造方法,或使用BigDecimal的valueOf方法,此方法内部其实执行了
   Double的toString,而Double的toString按double的实际能表达的精度对尾数进行了截断
       
   BigDecimal recommend1 = new BigDecimal("0.1"); 
   BigDecimal recommend2 = BigDecimal.valueOf(0.1);
   ```

6.  关于基本数据类型与包装数据类型的使用标准如下： 

     1）<font color=red>【强制】</font>所有的POJO类属性必须使用包装数据类型。 
     2） <font color=red>【强制】</font>RPC方法的返回值和参数必须使用包装数据类型。 
     3） <font color="#FFB800">【推荐】</font>所有的局部变量使用基本数据类型。 
    说明：POJO类属性没有初值是提醒使用者在需要使用时，必须自己显式地进行赋值，任何NPE问题，或
    者入库检查，都由使用者来保证。

    ```java
    正例：数据库的查询结果可能是null，因为自动拆箱，用基本数据类型接收有NPE风险。 
    ```

7. <font color=red>【强制】</font>定义DO/DTO/VO等POJO类时，不要设定任何属性默认值。 

   ```java
   反例：POJO类的createTime默认值为new Date()，但是这个属性在数据提取时并没有置入具体值，在
   更新其它字段时又附带更新了此字段，导致创建时间被修改成当前时间
   ```

8. <font color=red>【强制】</font>序列化类新增属性时，请不要修改serialVersionUID字段，避免反序列失败；如果
   完全不兼容升级，避免反序列化混乱，那么请修改serialVersionUID值。

9. <font color=red>【强制】</font>构造方法里面禁止加入任何业务逻辑，如果有初始化逻辑，请放在init方法中

10. <font color=red>【强制】</font>POJO类必须写toString方法。使用IDE中的工具：source> generate toString
    时，如果继承了另一个POJO类，注意在前面加一下super.toString。

    ```java
    说明：在方法执行抛出异常时，可以直接调用POJO的toString()方法打印其属性值，便于排查问题。 
    ```

    使用lombok可以简化我们的代码，它提供了下面的一些注解

    |          注解名称          | 功能                                                         |
    | :------------------------: | :----------------------------------------------------------- |
    |         `@Setter`          | 自动添加类中所有属性相关的 set 方法                          |
    |         `@Getter`          | 自动添加类中所有属性相关的 get 方法                          |
|         `@Builder`         | 使得该类可以通过 builder (建造者模式)构建对象                |
    | `@RequiredArgsConstructor` | 生成一个该类的构造方法，禁止无参构造                         |
|        `@ToString`         | 重写该类的`toString()`方法                                   |
    |    `@EqualsAndHashCode`    | 重写该类的`equals()`和`hashCode()`方法                       |
    |          `@Data`           | 等价于上面的`@Setter`、`@Getter`、`@RequiredArgsConstructor`、`@ToString`、`@EqualsAndHashCode` |
    
    使用lombok插件的@Data注解会包含toString方法，使用IDEA对class文件反编译得到最终的类，注意使用@Data的POJO类有继承父类的话，要加上
    
    ```
    @EqualsAndHashCode(callSuper = true)  // 调用父类的
    @ToString(callSuper = true)
    ```
    
    lombok生成子类的equals和toString方法默认是不会去调用父类的equals和toString方法，可以通过IDEA反编译class文件看最终源码知道。
    
    ```java
    @Data
    public class TestA {
        String oldName;
    }
    
    @Data
    @EqualsAndHashCode(callSuper = true)
    @ToString(callSuper = true)
    public class TestB extends TestA{
        private String name;
    
        private int age;
    }
    
    public static void main(String[] args) throws Exception {
      TestB t1 = new TestB();
      TestB t2 = new TestB();
    
      t1.setOldName("123");
      t2.setOldName("123");
    
      String name = "1";
    t1.setName(name);
      t2.setName(name);

      int age = 1;
      t1.setAge(age);
      t2.setAge(age);
      System.out.println(t1);
      System.out.println(t2);
      System.out.println(t1==t2); // 判断对象的地址是否相等
      System.out.println(t1.equals(t2)); // 判断对象内的属性是否相等，编译class的时候
      // lombok 会重写equals方法
    }
    ```
    
    ![](\assets\images\2020\java\lombok-data-equal-tostring.jpg)
    
    
    
11. <font color="#FFB800">【推荐】</font> 使用索引访问用String的split方法得到的数组时，需做最后一个分隔符后有无内容
    的检查，否则会有抛IndexOutOfBoundsException的风险。 

    ```java
    public static void main(String[] args) throws Exception {
      String str="1,2,3,,";
      String[] arr=str.split(",");
      System.out.println(arr.length); // 结果是 3
    }
    ```

12. <font color="#FFB800">【推荐】</font>类内方法定义的顺序依次是：公有方法或保护方法 > 私有方法 > getter / setter 
    方法。 

    ```java
    说明：公有方法是类的调用者和维护者最关心的方法，首屏展示最好；保护方法虽然只是子类关心，也可
    能是“模板设计模式”下的核心方法；而私有方法外部一般不需要特别关心，是一个黑盒实现；因为承载
    的信息价值较低，所有Service和DAO的getter/setter方法放在类体最后。
    ```

13. <font color="#FFB800">【推荐】</font>循环体内，字符串的连接方式，使用StringBuilder的append方法进行扩展，避免造成内存资源的浪费。

    ```java
    说明：下例中，反编译出的字节码文件显示每次循环都会new出一个StringBuilder对象，然后进行append
    操作，最后通过toString方法返回String对象，造成内存资源浪费。 
    反例： 
    String str = "start"; 
    for (int i = 0; i < 100; i++) { 
        str = str + "hello"; 
    } 
    ```

14. <font color="#FFB800">【推荐】</font>慎用Object的clone方法来拷贝对象。 
    说明：对象clone方法默认是浅拷贝（只拷贝基本类型的属性），若想实现深拷贝需覆写clone方法实现域对象的深度遍历式拷贝。 

15. <font color="#FFB800">【推荐】</font>类成员与方法访问控制从严： 
     1） 如果不允许外部直接通过new来创建对象，那么构造方法必须是private。 
     2） 工具类不允许有public或default构造方法。 
     3） 类非static成员变量并且与子类共享，必须是protected。 
     4） 类非static成员变量并且仅在本类使用，必须是private。 
     5） 类static成员变量如果仅在本类使用，必须是private。 
     6） 若是static成员变量，考虑是否为final。 

     7） 类成员方法只供类内部调用，必须是private。 
     8） 类成员方法只对继承类公开，那么限制为protected。 

    ```java
    说明：任何类、方法、参数、变量，严控访问范围。过于宽泛的访问范围，不利于模块解耦。思考：如果
    是一个private的方法，想删除就删除，可是一个public的service成员方法或成员变量，删除一下，不
    得手心冒点汗吗？变量像自己的小孩，尽量在自己的视线内，变量作用域太大，无限制的到处跑，那么你
    会担心的。 
    ```

### （5）日期时间

1. <font color=red>【强制】</font>获取当前毫秒数：System.currentTimeMillis(); 而不是new Date().getTime()。 
   说明：如果想获取更加精确的纳秒级时间值，使用System.nanoTime的方式。在JDK8中，针对统计时间
   等场景，推荐使用Instant类。

2. <font color=red>【强制】</font>不要在程序中写死一年为365天，避免在公历闰年时出现日期转换错误或程序逻辑错误。 

   ```java
   正例:
   // 获取今年的天数 
   int daysOfThisYear = LocalDate.now().lengthOfYear(); 
   // 获取指定某年的天数 
   LocalDate.of(2011, 1, 1).lengthOfYear(); 
   反例:
   // 第一种情况：在闰年366天时，出现数组越界异常 
   int[] dayArray = new int[365];  
   // 第二种情况：一年有效期的会员制，今年1月26日注册，硬编码365返回的却是1月25日 
   Calendar calendar = Calendar.getInstance(); 
   calendar.set(2020, 1, 26);  
   calendar.add(Calendar.DATE, 365); 
   ```

3. <font color="#FFB800">【推荐】</font>使用枚举值来指代月份。如果使用数字，注意Date，Calendar等日期相关类的月份
   month取值在0-11之间。 
   说明：参考JDK原生注释，Month value is 0-based. e.g., 0 for January. 
   正例： Calendar.JANUARY，Calendar.FEBRUARY，Calendar.MARCH等来指代相应月份来进行传参或
   比较。 

### （6）集合处理

1. <font color=red>【强制】</font>关于hashCode和equals的处理，遵循如下规则： 

   - 只要重写equals，就必须重写hashCode。 
   - 因为Set存储的是不重复的对象，依据hashCode和equals进行判断，所以Set存储的对象必须重写
     这两个方法。 

   - 如果自定义对象作为Map的键，那么必须覆写hashCode和equals。 
     说明：String因为重写了hashCode和equals方法，所以我们可以愉快地使用String对象作为key来使
     用。

2.  <font color=red>【强制】</font>判断所有集合内部的元素是否为空，使用isEmpty()方法，而不是size()==0的方式。 
   说明：前者的时间复杂度为O(1)，而且可读性更好

   ```java
   正例： 
   Map<String, Object> map = new HashMap<>(); 
   if(map.isEmpty()) { 
           System.out.println("no element in this map."); 
   } 
   ```

3. <font color=red>【强制】</font>在使用java.util.stream.Collectors类的toMap()方法转为Map集合时，一定要使
   用含有参数类型为BinaryOperator，参数名为mergeFunction的方法，否则当出现相同key
   值时会抛出IllegalStateException异常。 
   说明：参数mergeFunction的作用是当出现key重复时，自定义对value的处理策略。 

   ```java
   正例： 
   List<Pair<String, Double>> pairArrayList = new ArrayList<>(3); 
   pairArrayList.add(new Pair<>("version", 6.19)); 
   pairArrayList.add(new Pair<>("version", 10.24)); 
   pairArrayList.add(new Pair<>("version", 13.14)); 
   Map<String, Double> map = pairArrayList.stream().collect( 
   // 生成的map集合中只有一个键值对：{version=13.14} 
   Collectors.toMap(Pair::getKey, Pair::getValue, (v1, v2) -> v2)); 
   反例： 
   String[] departments = new String[] {"iERP", "iERP", "EIBU"}; 
   // 抛出IllegalStateException异常 
   Map<Integer, String> map = Arrays.stream(departments) 
       .collect(Collectors.toMap(String::hashCode, str -> str)); 
   ```

4. <font color=red>【强制】</font>ArrayList的subList结果不可强转成ArrayList，否则会抛出 ClassCastException异
   常：java.util.RandomAccessSubList cannot be cast to java.util.ArrayList。 

   ```java
   说明：subList 返回的是ArrayList的内部类SubList，并不是ArrayList而是ArrayList 的一个视图，对
   于SubList子列表的所有操作最终会反映到原列表上。 
   ```

5. <font color=red>【强制】</font>使用Map的方法keySet()/values()/entrySet()返回集合对象时，不可以对其进行添加元素操作，否则会抛出UnsupportedOperationException异常。

6. <font color=red>【强制】</font>使用集合转数组的方法，必须使用集合的toArray(T[] array)，传入的是类型完全一致、长度为0的空数组。  

   ```java
   反例：直接使用toArray无参方法存在问题，此方法返回值只能是Object[]类，若强转其它类型数组将出现
   ClassCastException错误。
   正例： 
   List<String> list = new ArrayList<>(2); 
   list.add("guan"); 
   list.add("bao"); 
   String[] array = list.toArray(new String[0]); 
       说明：使用toArray带参方法，数组空间大小的length， 
   
   1） 等于0，动态创建与size相同的数组，性能最好。 
   2） 大于0但小于size，重新创建大小等于size的数组，增加GC负担。 
   3） 等于size，在高并发情况下，数组创建完成之后，size正在变大的情况下，负面影响与2相同。 
   4） 大于size，空间浪费，且在size处插入null值，存在NPE隐患。 
   ```

7.  <font color=red>【强制】</font>使用工具类Arrays.asList()把数组转换成集合时，不能使用其修改集合相关的方法，
   它的add/remove/clear方法会抛出UnsupportedOperationException异常。 
   说明：asList的返回对象是一个Arrays内部类，并没有实现集合的修改方法。

   ```java
   Arrays.asList体现的是适配器模式，只是转换接口，后台的数据仍是数组。 
       String[] str = new String[] { "yang", "hao" }; 
       List list = Arrays.asList(str); 
   第一种情况：list.add("yangguanbao"); 运行时异常。 
   第二种情况：str[0] = "changed"; 也会随之修改，反之亦然
   ```

8.  <font color=red>【强制】</font>要在foreach循环里进行元素的remove/add操作。remove元素请使用Iterator
   方式，如果并发操作，需要对Iterator对象加锁。 

   ```java
   正例： 
   List<String> list = new ArrayList<>(); 
   list.add("1"); 
   list.add("2"); 
   Iterator<String> iterator = list.iterator(); 
   while (iterator.hasNext()) { 
       String item = iterator.next(); 
       if (删除元素的条件) { 
           iterator.remove(); 
       } 
   } 
   反例： 
   for (String item : list) { 
       if ("1".equals(item)) { 
           list.remove(item); 
       } 
   } 
   说明：以上代码的执行结果肯定会出乎大家的意料，那么试一下把“1”换成“2”，会是同样的结果吗
   ```

9.  <font color=red>【强制】</font>集合初始化时，指定集合初始值大小。 
   说明：HashMap使用HashMap(int initialCapacity) 初始化，如果暂时无法确定集合大小，那么指定默认值（16）即可。 
   正例：initialCapacity = (需要存储的元素个数 / 负载因子) + 1。注意负载因子（即loader factor）默认为0.75，如果暂时无法确定初始值大小，请设置为16（即默认值）

10. <font color=red>【强制】</font>高度注意Map类集合K/V能不能存储null值的情况，如下表格：

    

### （7）并发处理

1. <font color=red>【强制】</font>创建线程或线程池时请指定有意义的线程名称，方便出错时回溯。 
   正例：自定义线程工厂，并且根据外部特征进行分组，比如，来自同一机房的调用，把机房编号赋值给whatFeaturOfGroup

   ```java
   public class UserThreadFactory implements ThreadFactory { 
       private final String namePrefix; 
       private final AtomicInteger nextId = new AtomicInteger(1); 
    
       // 定义线程组名称，在jstack问题排查时，非常有帮助 
       UserThreadFactory(String whatFeaturOfGroup) { 
           namePrefix = "From UserThreadFactory's " + whatFeaturOfGroup + "-Worker-"; 
       } 
    
       @Override 
       public Thread newThread(Runnable task) { 
           String name = namePrefix + nextId.getAndIncrement(); 
           Thread thread = new Thread(null, task, name, 0, false); 
           System.out.println(thread.getName()); 
           return thread; 
       } 
   }
   ```

2. <font color=red>【强制】</font>线程池不允许使用Executors去创建，而是通过ThreadPoolExecutor的方式，这
   样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。

   ```java
   说明：Executors返回的线程池对象的弊端如下： 
   1） FixedThreadPool和SingleThreadPool： 
     允许的请求队列长度为Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM。 
   2） CachedThreadPool： 
     允许的创建线程数量为Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM
   ```

3. <font color=red>【强制】</font>SimpleDateFormat 是线程不安全的类，一般不要定义为static变量，如果定义为static必须加锁，或者使用DateUtils工具类。 

4. <font color=red>【强制】</font>必须回收自定义的ThreadLocal变量，尤其在线程池场景下，线程经常会被复用，
   如果不清理自定义的 ThreadLocal变量，可能会影响后续业务逻辑和造成内存泄露等问题。
   尽量在代理中使用try-finally块进行回收。 因为ThreadLocal的key是弱引用，只能生存到下次GC.

5. <font color=red>【强制】</font>对多个资源、数据库表、对象同时加锁时，需要保持一致的加锁顺序，否则可能会造成死锁。 

   ```java
   说明：线程一需要对表A、B、C依次全部加锁后才可以进行更新操作，那么线程二的加锁顺序也必须是A、
   B、C，否则可能出现死锁。 
   ```

6. <font color=red>【强制】</font>在使用阻塞等待获取锁的方式中，必须在try代码块之外，并且在加锁方法与try代
   码块之间没有任何可能抛出异常的方法调用，避免加锁成功后，在finally中无法解锁。 

   ```java
   说明一：如果在lock方法与try代码块之间的方法调用抛出异常，那么无法解锁，造成其它线程无法成功
   获取锁。 
   说明二：如果lock方法在try代码块之内，可能由于其它方法抛出异常，导致在finally代码块中，unlock
   对未加锁的对象解锁，它会调用AQS的tryRelease方法（取决于具体实现类），抛出
   IllegalMonitorStateException异常。 
   说明三：在Lock对象的lock方法实现中可能抛出unchecked异常，产生的后果与说明二相同。 
   正例： 
   Lock lock = new XxxLock(); 
   // ... 
   lock.lock(); 
   try { 
       doSomething(); 
       doOthers(); 
   } finally { 
       lock.unlock(); 
   } 
   反例： 
   Lock lock = new XxxLock(); 
   try { 
       // 如果此处抛出异常，则直接执行finally代码块 
       doSomething(); 
       // 无论加锁是否成功，finally代码块都会执行 
       lock.lock(); 
       doOthers(); 
   } finally { 
       lock.unlock(); 
   } 
   ```

7. <font color=red>【强制】</font>并发修改同一记录时，避免更新丢失，需要加锁。要么在应用层加锁，要么在缓存加锁，要么在数据库层使用乐观锁，使用version作为更新依据。 
   说明：如果每次访问冲突概率小于20%，推荐使用乐观锁，否则使用悲观锁。乐观锁的重试次数不得小于3次。 

8. <font color="FFB800">【推荐】</font>资金相关的金融敏感信息，使用悲观锁策略。 
   说明：乐观锁在获得锁的同时已经完成了更新操作，校验逻辑容易出现漏洞，另外，乐观锁对冲突的解决策略有较复杂的要求，处理不当容易造成系统压力或数据异常，所以资金相关的金融敏感信息不建议使用乐观锁更新。

   ```java
   正例：悲观锁遵循一锁二判三更新四释放的原则 
   ```

9. <font color="FFB800">【推荐】</font>volatile解决多线程内存不可见问题。对于一写多读，是可以解决变量同步问题，但
   是如果多写，同样无法解决线程安全问题。 

   ```java
   说明：如果是count++操作，使用如下类实现：
     AtomicInteger count = new AtomicInteger(); 
   	count.addAndGet(1); 
   如果是JDK8，推荐使用LongAdder对象，比AtomicLong性能更好（减少乐观锁的重试次数）。
   ```

10. <font color=red>【参考】</font>HashMap在容量不够进行resize时由于高并发可能出现死链，导致CPU飙升，在
    开发过程中注意规避此风险。

11.  <font color=red>【参考】</font>ThreadLocal对象使用static修饰，ThreadLocal无法解决共享对象的更新问题。 
    说明：这个变量是针对一个线程内所有操作共享的，所以设置为静态变量，所有此类实例共享此静态变量，也就是说在类第一次被使用时装载，只分配一块存储空间，所有此类的对象(只要是这个线程内定义的)都可以操控这个变量。

### （8）控制语句

1. <font color=red>【强制】</font>在一个switch块内，每个case要么通过continue/break/return等来终止，要么
   注释说明程序将继续执行到哪一个case为止；在一个switch块内，都必须包含一个default语句并且放在最后，即使它什么代码也没有。 

   当switch括号内的变量类型为String并且此变量为外部参数时，必须先进行null
   判断。

   ```java
   public class SwitchString { 
       public static void main(String[] args) { 
           method(null); 
       } 
    
       public static void method(String param) { 
           switch (param) { 
               // 肯定不是进入这里 
               case "sth": 
                   System.out.println("it's sth"); 
                   break; 
               // 也不是进入这里 
               case "null": 
                   System.out.println("it's null"); 
                   break; 
               // 也不是进入这里 
               default: 
                   System.out.println("default"); 
           } 
       } 
   } 
   ```

2. <font color=red>【强制】</font>在高并发场景中，避免使用”等于”判断作为中断或退出的条件。 
   说明：如果并发控制没有处理好，容易产生等值判断被“击穿”的情况，<font color=red>使用大于或小于的区间判断条件来代替。</font>

   ```java
   反例：判断剩余奖品数量等于0时，终止发放奖品，但因为并发处理错误导致奖品数量瞬间变成了负数，
   这样的话，活动无法终止。
   ```

3. <font color="FFB800">【推荐】</font>表达异常的分支时，少用if-else方式，这种方式可以改写成：

   ```java
   if (condition) {     
       ... 
       return obj; 
   } 
   // 接着写else的业务逻辑代码 
   ```

4. <font color="FFB800">【推荐】</font> 循环体中的语句要考量性能，以下操作尽量移至循环体外处理，如定义对象、变量、获取数据库连接，进行不必要的try-catch操作（这个try-catch是否可以移至循环体外）

5. <font color="FFB800">【推荐】</font> 接口入参保护，这种场景常见的是用作批量操作的接口。 
   反例：某业务系统，提供一个用户批量查询的接口，API文档上有说最多查多少个，但接口实现上没做任何保护，导致调用方传了一个1000的用户id数组过来后，查询信息后，内存爆了。

### （9）注释规约

1. <font color=red>【强制】</font>类、类属性、类方法的注释必须使用Javadoc规范，使用/**内容*/格式，不得使用
   // xxx方式。 

   ```java
   说明：在IDE编辑窗口中，Javadoc方式会提示相关注释，生成Javadoc可以正确输出相应注释；在IDE
   中，工程调用方法时，不进入方法即可悬浮提示方法、参数、返回值的意义，提高阅读效率。 
   ```

2. <font color=red>【强制】</font>所有的抽象方法（包括接口中的方法）必须要用Javadoc注释、除了返回值、参数、
   异常说明外，还必须指出该方法做什么事情，实现什么功能。 
   说明：对子类的实现要求，或者调用注意事项，请一并说明。 

3. <font color=red>【强制】</font>所有的类都必须添加创建者和创建日期。

   ```java
   /** 
    * @author yangguanbao 
    * @date 2016/10/31 
    */ 
   ```

4. <font color="red">【参考】</font>谨慎注释掉代码。在上方详细说明，而不是简单地注释掉。如果无用，则删除。 
   说明：代码被注释掉有两种可能性：1）后续会恢复此段代码逻辑。2）永久不用。前者如果没有备注信息，
   难以知晓注释动机。后者建议直接删掉即可，假如需要查阅历史代码，登录代码仓库即可。

5. <font color="red">【参考】</font>特殊注释标记，请注明标记人与标记时间。注意及时处理这些标记，通过标记扫描，经常清理此类标记。线上故障有时候就是来源于这些标记处的代码

   ```java
   1） 待办事宜（TODO）:（标记人，标记时间，[预计处理时间]） 
      表示需要实现，但目前还未实现的功能。这实际上是一个Javadoc的标签，目前的Javadoc还没 
      有实现，但已经被广泛使用。只能应用于类，接口和方法（因为它是一个Javadoc标签）。 
    2） 错误，不能工作（FIXME）:（标记人，标记时间，[预计处理时间]） 
      在注释中用FIXME标记某代码是错误的，而且不能工作，需要及时纠正的情况。 
   ```

6. <font color=red>【强制】</font>避免用Apache Beanutils进行属性的copy。 
   说明：Apache BeanUtils性能较差，可以使用其他方案比如Spring BeanUtils, Cglib BeanCopier，注意均是浅拷贝

7. <font color=red>【强制】</font>如果想获取整数类型的随机数，直接使用Random对象的nextInt或者nextLong方法。 

8. <font color="FFB800">【推荐】</font>任何数据结构的构造或初始化，都应指定大小，避免数据结构无限增长吃光内存。

9. <font color="FFB800">【推荐】</font>及时清理不再使用的代码段或配置信息。

   ```java
   正例：对于暂时被注释掉，后续可能恢复使用的代码片断，在注释代码上方，统一规定使用三个斜杠(///)
   来说明注释掉代码的理由。如： 
      public static void hello() { 
       /// 业务方通知活动暂停 
       // Business business = new Business(); 
       // business.active(); 
       System.out.println("it's finished"); 
   } 
   ```

   

## 2、异常日志

### （1）错误码

1. 正例：错误码回答的问题是谁的错？错在哪？1）错误码必须能够快速知晓错误来源，可快速判断是谁的问题。2）错误码易于记忆和比对（代码中容易equals）。3）错误码能够脱离文档和系统平台达到线下轻量化地自由沟通的目的

2. <font color=red>【强制】</font>全部正常，但不得不填充错误码时返回五个零：00000。 

3. <font color=red>【强制】</font>错误码为字符串类型，共5位，分成两个部分：错误产生来源+四位数字编号。 

   ```java
   说明：错误产生来源分为A/B/C，A表示错误来源于用户，比如参数错误，用户安装版本过低，用户支付
   超时等问题；B表示错误来源于当前系统，往往是业务逻辑出错，或程序健壮性差等问题；C表示错误来源
   于第三方服务，比如CDN服务出错，消息投递超时等问题；四位数字编号从0001到9999，大类之间的
   步长间距预留100
   ```

4. <font color=red>【强制】</font>错误码不能直接输出给用户作为提示信息使用。 
   说明：堆栈（stack_trace）、错误信息(error_message)、错误码（error_code）、提示信息（user_tip）是一个有效关联并互相转义的和谐整体，但是请勿互相越俎代庖

5. <font color="FFB800">【推荐】</font>错误码之外的业务独特信息由error_message来承载，而不是让错误码本身涵盖过
   多具体业务属性。

6. <font color=red>【参考】</font>错误码分为一级宏观错误码、二级宏观错误码、三级宏观错误码。 

   ```java
   说明：在无法更加具体确定的错误场景中，可以直接使用一级宏观错误码，分别是：A0001（用户端错误）、
   B0001（系统执行出错）、C0001（调用第三方服务出错）。   
   ```

### （2）异常处理

1. <font color=red>【强制】</font>finally块必须对资源对象、流对象进行关闭，有异常也要做try-catch。 
   说明：如果JDK7及以上，可以使用try-with-resources方式
   
2. <font color=red>【强制】</font>在调用RPC、二方包、或动态生成类的相关方法时，捕捉异常必须使用Throwable
   类来进行拦截。 
   
3. <font color="FFB800">【推荐】</font>防止NPE，是程序员的基本修养，注意NPE产生的场景： 
   
    1） 返回类型为基本数据类型，return包装数据类型的对象时，自动拆箱有可能产生NPE。 
       反例：public int f() { return Integer对象}， 如果为null，自动解箱抛NPE。 
    
    2） 数据库的查询结果可能为null。 
    
    3） 集合里的元素即使isNotEmpty，取出的数据元素也可能为null。 
   
   4） 远程调用返回对象时，一律要求进行空指针判断，防止NPE。 
   
   5） 对于Session中获取的数据，建议进行NPE检查，避免空指针。 
   
   6） 级联调用obj.getA().getB().getC()；一连串调用，易产生NPE。 正例：使用JDK8的Optional类来防止NPE问题

### （3）日志规约

1. <font color="FFB800">【推荐】</font> 使用日志框架

   ```java
   slf4j:
   import org.slf4j.Logger; 
   import org.slf4j.LoggerFactory; 
   private static final Logger logger = LoggerFactory.getLogger(Test.class); 
   
   jcl:
   import org.apache.commons.logging.Log; 
   import org.apache.commons.logging.LogFactory; 
   private static final Log log = LogFactory.getLog(Test.class); 
   ```

2. <font color=red>【强制】</font>在日志输出时，字符串变量之间的拼接使用占位符的方式。 
   说明：因为String字符串的拼接会使用StringBuilder的append()方式，有一定的性能损耗。使用占位符仅是替换动作，可以有效提升性能。 

   ```java
   正例：logger.debug("Processing trade with id: {} and symbol: {}", id, symbol); 
   ```

3. <font color=red>【强制】</font>应用中的扩展日志（如打点、临时监控、访问日志等）命名方式：
   appName_logType_logName.log。logType:日志类型，如stats/monitor/access等；logName:日志描述。这种命名的好处：通过文件名就可知道日志文件属于什么应用，什么类型，什么目的，也有利于归类查找。 
   说明：推荐对日志进行分类，如将错误日志和业务日志分开存放，便于开发人员查看，也便于通过日志对系统进行及时监控

4. <font color=red>【强制】</font>异常信息应该包括两类信息：案发现场信息和异常堆栈信息。如果不处理，那么通过关键字throws往上抛出。 

   ```java
   // 正例：
   logger.error(各类参数或者对象toString() + "_" + e.getMessage(), e); 
   ```

5. <font color="FFB800">【推荐】</font>可以使用warn日志级别来记录用户输入参数错误的情况，避免用户投诉时，无所适
   从。如非必要，请不要在此场景打出error级别，避免频繁报警。  
   说明：注意日志输出的级别，error级别只记录系统逻辑出错、异常或者重要的错误信息。 

## 3、单元测试

1. <font color=red>【强制】</font>好的单元测试必须遵守AIR原则。 

   ```sh
   说明：单元测试在线上运行时，感觉像空气（AIR）一样并不存在，但在测试质量的保障上，却是非常关键
   的。好的单元测试宏观上来说，具有自动化、独立性、可重复执行的特点。
   ```

   - <font color="#1E9FFF">A</font>：Automatic（自动化） 
   - <font color="#1E9FFF">I</font>：Independent（独立性） 
   - <font color="#1E9FFF">R</font>：Repeatable（可重复）
   
2. <font color="red">【强制】</font>对于单元测试，要保证测试粒度足够小，有助于精确定位问题。单测粒度至多是类级
   别，一般是方法级别。 

   ```sh
   说明：只有测试粒度小才能在出错时尽快定位到出错位置。单测不负责检查跨类或者跨系统的交互逻辑，那是集成测试的领域。 
   ```

3. <font color="red">【强制】</font>单元测试代码必须写在如下工程目录：src/test/java，不允许写在业务代码目录下。 

   ```sh
   说明：源码编译时会跳过此目录，而单元测试框架默认是扫描此目录
   ```

4. <font color="FFB800">【推荐】</font>编写单元测试代码遵守BCDE原则，以保证被测试模块的交付质量。 
   - <font color="#1E9FFF">B</font>：Border，边界值测试，包括循环边界、特殊取值、特殊时间点、数据顺序等。 
   - <font color="#1E9FFF">C</font>：Correct，正确的输入，并得到预期的结果。 
   - <font color="#1E9FFF">D</font>：Design，与设计文档相结合，来编写单元测试。 
   - <font color="#1E9FFF">E</font>：Error，强制错误信息输入（如：非法数据、异常流程、业务允许外等），并得到预期的结果。

5. <font color="FFB800">【推荐】</font>对于数据库相关的查询，更新，删除等操作，不能假设数据库里的数据是存在的，或
   者直接操作数据库把数据插入进去，请使用程序插入或者导入数据的方式来准备数据。 

   ```sh
   反例：删除某一行数据的单元测试，在数据库中，先直接手动增加一行作为删除目标，但是这一行新增数
   据并不符合业务插入规则，导致测试结果异常
   ```

6. <font color="FFB800">【推荐】</font>和数据库相关的单元测试，可以设定自动回滚机制，不给数据库造成脏数据。或者对
   单元测试产生的数据有明确的前后缀标识
7.  <font color="FFB800">【推荐】</font>在设计评审阶段，开发人员需要和测试人员一起确定单元测试范围，单元测试最好覆
   盖所有测试用例（UC）
8.  <font color="FFB800">【参考】</font>不要对单元测试存在如下误解： 
   - 那是测试同学干的事情。本文是开发手册，凡是本文内容都是与开发同学强相关的。 
   - 单元测试代码是多余的。系统的整体功能与各单元部件的测试正常与否是强相关的。 
   - 单元测试代码不需要维护。一年半载后，那么单元测试几乎处于废弃状态。 
   -  单元测试与线上故障没有辩证关系。好的单元测试能够最大限度地规避线上故障。 

## 4、安全规约

1. <font color="red">【强制】</font>隶属于用户个人的页面或者功能必须进行权限控制校验。 

   ```sh
   说明：防止没有做水平权限校验就可随意访问、修改、删除别人的数据，比如查看他人的私信内容。
   ```

2. <font color="red">【强制】</font>用户敏感数据禁止直接展示，必须对展示数据进行脱敏。 

   ```sh
   说明：中国大陆个人手机号码显示为:137****0969，隐藏中间4位，防止隐私泄露。
   ```

3. <font color="red">【强制】</font>用户输入的SQL参数严格使用参数绑定或者METADATA字段值限定，防止SQL注入，
   禁止字符串拼接SQL访问数据库。 

   ```sh
   反例：某系统签名大量被恶意修改，即是因为对于危险字符 # --没有进行转义，导致数据库更新时，where
   后边的信息被注释掉，对全库进行更新。
   ```

4. <font color="red">【强制】</font>用户请求传入的任何参数必须做有效性验证。 
   说明：忽略参数校验可能导致： 

   - page size过大导致内存溢出 

   - 恶意order by导致数据库慢查询 
   - 缓存击穿 
   - SSRF 
   - 任意重定向 
   - SQL注入，Shell注入，反序列化注入 
   - 正则输入源串拒绝服务ReDoS 

5. <font color="red">【强制】</font>表单、AJAX提交必须执行CSRF安全验证。 
   说明：CSRF(Cross-site request forgery)跨站请求伪造是一类常见编程漏洞。对于存在CSRF漏洞的应用/
   网站，攻击者可以事先构造好URL，只要受害者用户一访问，后台便在用户不知情的情况下对数据库中用
   户参数进行相应修改。

## 5、Mysql数据库

### （1）建表规约

1. <font color="red">【强制】</font>表达是与否概念的字段，必须使用is_xxx的方式命名，数据类型是unsigned tinyint
   （1表示是，0表示否）

   ```sh
   说明：任何字段如果为非负数，必须是unsigned。 
   注意：POJO类中的任何布尔类型的变量，都不要加is前缀，所以，需要在<resultMap>设置从is_xxx到
   Xxx的映射关系。数据库表示是与否的值，使用tinyint类型，坚持is_xxx的命名方式是为了明确其取值含
   义与取值范围。 
   正例：表达逻辑删除的字段名is_deleted，1表示删除，0表示未删除。 
   ```

2. <font color="red">【强制】</font>表名、字段名必须使用小写字母或数字，禁止出现数字开头，禁止两个下划线中间只
   出现数字。数据库字段名的修改代价很大，因为无法进行预发布，所以字段名称需要慎重考虑。 

   ```sh
   说明：MySQL在Windows下不区分大小写，但在Linux下默认是区分大小写。因此，数据库名、表名、
   字段名，都不允许出现任何大写字母，避免节外生枝。 
   正例：aliyun_admin，rdc_config，level3_name 
   反例：AliyunAdmin，rdcConfig，level_3_name 
   ```

3. <font color="red">【强制】</font>禁用保留字，如desc、range、match、delayed等，请参考MySQL官方保留字。 主键索引名为pk\_字段名；唯一索引名为uk\_字段名；普通索引名则为idx\_字段名

   ```sh
   说明：pk_ 即primary key；uk_ 即 unique key；idx_ 即index的简称。
   ```

4. <font color="red">【强制】</font>小数类型为decimal，禁止使用float和double。 
   说明：在存储的时候，float 和 double 都存在精度损失的问题，很可能在比较值的时候，得到不正确的
   结果。如果存储的数据范围超过 decimal 的范围，建议将数据拆成整数和小数并分开存储。

5. <font color="red">【强制】</font>如果存储的字符串长度几乎相等，使用char定长字符串类型。varchar是可变长字符串，不预先分配存储空间，长度不要超过5000，如果存储长度大于此值，定义字段类型为text，独立出来一张表，用主键来对应，避免影响其它字段索引效率

6. <font color="red">【强制】</font>表必备三字段：id, gmt_create, gmt_modified。 

   ```sh
   说明：其中id必为主键，类型为bigint unsigned、单表时自增、步长为1。gmt_create, gmt_modified
   的类型均为datetime类型，前者现在时表示主动式创建，后者过去分词表示被动式更新。 
   还可以加上is_delete删除标记，version版本两个字段
   ```

7.  <font color="FFB800">【推荐】</font>表的命名最好是遵循“业务名称_表的作用”；库名与应用名称尽量一致； 如果修改字段含义或对字段表示的状态追加时，需要及时更新字段注释。

   ```java
   正例：alipay_task / force_project / trade_config 
   ```

8.  <font color="FFB800">【推荐】</font>字段允许适当冗余，以提高查询性能，但必须考虑数据一致。冗余字段应遵循： 
    1） 不是频繁修改的字段。 
    2） 不是唯一索引的字段。 
    3） 不是varchar超长字段，更不能是text字段。 

    ```sh
   正例：各业务线经常冗余存储商品名称，避免查询时需要调用IC服务获取。 
   ```

9.  <font color="FFB800">【推荐】</font>单表行数超过500万行或者单表容量超过2GB，才推荐进行分库分表。 

   ```sh
   说明：如果预计三年后的数据量根本达不到这个级别，请不要在创建表时就分库分表。 
   ```

10. <font color="FFB800">【推荐】</font>合适的字符存储长度，不但节约数据库表空间、节约索引存储，更重要的是提升检索
    速度。 

    ```sh
    正例：无符号值可以避免误存负数，且扩大了表示范围。
    ```

    | 对象     | 年龄区间  | 类型              | 字节 | 表示范围                  |
    | -------- | --------- | ----------------- | ---- | ------------------------- |
    | 人       | 150岁之内 | tinyint unsigned  | 1    | 无符号值：0到255          |
    | 龟       | 数百岁    | smallint unsigned | 2    | 无符号值：0到65535        |
    | 恐龙化石 | 数千万年  | int unsigned      | 4    | 无符号值：0到约43亿       |
    | 太阳     | 约50亿年  | bigint unsigned   | 8    | 无符号值：0到约10的19次方 |

### （2）索引规约

1. <font color="red">【强制】</font>业务上具有唯一特性的字段，即使是组合字段，也必须建成唯一索引。 

```sh
说明：不要以为唯一索引影响了insert速度，这个速度损耗可以忽略，但提高查找速度是明显的；另外，
即使在应用层做了非常完善的校验控制，只要没有唯一索引，根据墨菲定律，必然有脏数据产生。 
```

2. <font color="red">【强制】</font>超过三个表禁止join。需要join的字段，数据类型保持绝对一致；多表关联查询时，保证被关联的字段需要有索引。

3. <font color="red">【强制】</font>在varchar字段上建立索引时，必须指定索引长度，没必要对全字段建立索引，根据
   实际文本区分度决定索引长度

   ```sh
   说明：索引的长度与区分度是一对矛盾体，一般对字符串类型数据，长度为20的索引，区分度会高达90%
   以上，可以使用count(distinct left(列名, 索引长度))/count(*)的区分度来确定。
   ```

4. <font color="red">【强制】</font>页面搜索严禁左模糊或者全模糊，如果需要请走搜索引擎来解决。 

   ```sh
   说明：索引文件具有B-Tree的最左前缀匹配特性，如果左边的值未确定，那么无法使用此索引
   ```

5. <font color="FFB800">【推荐】</font>如果有order by的场景，请注意利用索引的有序性。order by 最后的字段是组合索
   引的一部分，并且放在索引组合顺序的最后，避免出现file_sort的情况，影响查询性能。 

   ```sh
   正例：where a=? and b=? order by c; 索引：a_b_c 
   反例：索引如果存在范围查询，那么索引有序性无法利用，如：WHERE a>10 ORDER BY b; 索引a_b无法排序
   ```

6. <font color="FFB800">【推荐】</font>利用延迟关联或者子查询优化超多分页场景。 
   说明：MySQL并不是跳过offset行，而是取offset+N行，然后返回放弃前offset行，返回N行，那当
   offset特别大的时候，效率就非常的低下，要么控制返回的总页数，要么对超过特定阈值的页数进行SQL
   改写。 

   ```sh
   正例：先快速定位需要获取的id段，然后再关联： 
   SELECT a.* FROM 表1 a, (select id from 表1 where 条件 LIMIT 100000,20 ) b where a.id=b.id 
   ```

7. <font color="FFB800">【推荐】</font>SQL性能优化的目标：至少要达到 range 级别，要求是ref级别，如果可以是consts最好。 

   ```sh
   说明： 
    1） consts 单表中最多只有一个匹配行（主键或者唯一索引），在优化阶段即可读取到数据。 
    2） ref 指的是使用普通的索引（normal index）。 
    3） range 对索引进行范围检索。 
   反例：explain表的结果，type=index，索引物理文件全扫描，速度非常慢，这个index级别比较range还低，
   与全表扫描是小巫见大巫。 
   ```

8. <font color="FFB800">【推荐】</font>建组合索引的时候，区分度最高的在最左边。 
   
   ```sh
   正例：如果where a=? and b=?，a列的几乎接近于唯一值，那么只需要单建idx_a索引即可。 
   说明：存在非等号和等号混合判断条件时，在建索引时，请把等号条件的列前置。如：where c>? and d=? 
   那么即使c的区分度更高，也必须把d放在索引的最前列，即建立组合索引idx_d_c。
   ```

9. <font color="FFB800">【推荐】</font>利用延迟关联或者子查询优化超多分页场景。 

   ```sh
   说明：MySQL并不是跳过offset行，而是取offset+N行，然后返回放弃前offset行，返回N行，那当
   offset特别大的时候，效率就非常的低下，要么控制返回的总页数，要么对超过特定阈值的页数进行SQL
   改写。 
   正例：先快速定位需要获取的id段，然后再关联： 
         SELECT a.* FROM 表1 a, (select id from 表1 where 条件 LIMIT 100000,20 ) b where a.id=b.id 
   ```

### （3）SQL语句

1. <font color="red">【强制】</font>不要使用count(列名)或count(常量)来替代count(*)，count(*)是SQL92定义的标准统计行数的语法，跟数据库无关，跟NULL和非NULL无关。 

   ```sh
   说明：count(*)会统计值为NULL的行，而count(列名)不会统计此列为NULL值的行。
   ```

2. <font color="red">【强制】</font>count(distinct col) 计算该列除NULL之外的不重复行数，注意 count(distinct col1, col2) 如果其中一列全为NULL，那么即使另一列有不同的值，也返回为0。当某一列的值全是NULL时，count(col)的返回结果为0，但sum(col)的返回结果为NULL，因此使用sum()时需注意NPE问题。

   ```sh
   可以使用如下方式来避免sum的NPE问题：SELECT IFNULL(SUM(column), 0) FROM table; 
   ```

3. <font color="red">【强制】</font>使用ISNULL()来判断是否为NULL值。 

   ```sh
   说明：NULL与任何值的直接比较都为NULL。 
   1） NULL<>NULL的返回结果是NULL，而不是false。 
   2） NULL=NULL的返回结果是NULL，而不是true。 
   3） NULL<>1的返回结果是NULL，而不是true。 
   反例：在SQL语句中，如果在null前换行，影响可读性。select * from table where column1 is null and 
   column3 is not null; 而ISNULL(column)是一个整体，简洁易懂。从性能数据上分析，ISNULL(column)
   执行效率更快一些。 
   ```

4. <font color="red">【强制】</font>不得使用外键与级联，一切外键概念必须在应用层解决。 

   ```sh
   说明：（概念解释）学生表中的student_id是主键，那么成绩表中的student_id则为外键。如果更新学生表中的student_id，
   同时触发成绩表中的student_id更新，即为级联更新。外键与级联更新适用于单机低并发，不适合分布式、高并发集群；
   级联更新是强阻塞，存在数据库更新风暴的风险；外键影响数据库的插入速度。 
   ```

5. <font color="red">【强制】</font>数据订正（特别是删除或修改记录操作）时，要先select，避免出现误删除，确认无
   误才能执行更新语句。 

6. <font color="FFB800">【推荐】</font>SQL语句中表的别名前加as，并且以t1、t2、t3、...的顺序依次命名。 

   ```sh
   说明：1）别名可以是表的简称，或者是根据表出现的顺序，以t1、t2、t3的方式命名。2）别名前加as使别名更容易识别。 
   正例：select t1.name from table_first as t1, table_second as t2 where t1.id=t2.id;
   ```

7. <font color="FFB800">【推荐】</font>in操作能避免则避免，若实在避免不了，需要仔细评估in后边的集合元素数量，控制在1000个之内。

8. <font color="FFB800">【推荐】</font>因国际化需要，所有的字符存储与表示，均采用utf8字符集，那么字符计数方法需要注意。

   ```sh
   说明： 
   SELECT LENGTH("轻松工作")； 返回为12 
   SELECT CHARACTER_LENGTH("轻松工作")； 返回为4 
   如果需要存储表情，那么选择utf8mb4来进行存储，注意它与utf8编码的区别。 
   ```

9. <font color="FFB800">【注意】</font>TRUNCATE TABLE 比 DELETE 速度快，且使用的系统和事务日志资源少，但TRUNCATE无事务且不触发trigger(故没有回滚一说)，有可能造成事故，故不建议在开发代码中使用此语句。 

   ```sh
   说明：TRUNCATE TABLE 在功能上与不带 WHERE 子句的 DELETE 语句相同。
   ```

   

### （4）ORM映射

1. <font color="red">【强制】</font>在表查询中，一律不要使用 * 作为查询的字段列表，需要哪些字段必须明确写明。 

   ```sh
   说明：1）增加查询分析器解析成本。2）增减字段容易与resultMap配置不一致。3）无用字段增加网络消耗，尤其是text类型的字段。 
   ```

2. <font color="red">【强制】</font>POJO类的布尔属性不能加is，而数据库字段必须加is_，要求在resultMap中进行字段与属性之间的映射。 

   ```sh
   说明：参见定义POJO类以及数据库字段定义规定，在sql.xml增加映射，是必须的。需要在<resultMap>设置从is_xxx到xxx的映射关系。 
   反例：定义为基本数据类型Boolean isDeleted的属性，它的方法也是isDeleted()，框架在反向解析的时
   候，“误以为”对应的属性名称是deleted，导致属性获取不到，进而抛出异常。
   ```

3. <font color="red">【强制】</font>不要用resultClass当返回参数，即使所有类属性名与数据库字段一一对应，也需要定义\<resultMap>；反过来，每一个表也必然有一个\<resultMap>与之对应。 

   ```sh
   说明：配置映射关系，使字段与DO类解耦，方便维护。 
   ```

4. <font color="red">【强制】</font>sql.xml配置参数使用：#{}，#param# 不要使用${} 此种方式容易出现SQL注入。 

5. <font color="red">【强制】</font>不允许直接拿HashMap与Hashtable作为查询结果集的输出。 

   ```sh
   反例：某同学为避免写一个\<resultMap\>，直接使用HashTable来接收数据库返回结果，结果出现日常
   是把bigint转成Long值，而线上由于数据库版本不一样，解析成BigInteger，导致线上问题。 
   ```

6. <font color="red">【强制】</font>更新数据表记录时，必须同时更新记录对应的gmt_modified字段值为当前时间。

7. <font color="FFB800">【推荐】</font>不要写一个大而全的数据更新接口。传入为POJO类，不管是不是自己的目标更新字段，都进行update table set c1=value1,c2=value2,c3=value3; 这是不对的。执行SQL时，不要更新无改动的字段，一是易出错；二是效率低；三是增加binlog存储。

8. <font color="FFB800">【推荐】</font>@Transactional事务不要滥用。事务会影响数据库的QPS，另外使用事务的地方需要考虑各方面的回滚方案，包括缓存回滚、搜索引擎回滚、消息补偿、统计修正等。

9. <font color="FFB800">【推荐】</font>\<isEqual>中的compareValue是与属性值对比的常量，一般是数字，表示相等时带上此条件；\<isNotEmpty>表示不为空且不为null时执行；\<isNotNull>表示不为null值时执行。 

   

## 6、工程结构

### （1）应用分层

1. <font color="FFB800">【推荐】</font>图中默认上层依赖于下层，箭头关系表示可直接依赖，如：开放接口层可以依赖于Web层，也可以直接依赖于Service层，依此类推： 

   ![](\assets\images\2020\java\taishan-service-layer.jpg)

   - 开放接口层：可直接封装Service方法暴露成RPC接口；通过Web封装成http接口；网关控制层等。 
   - 终端显示层：各个端的模板渲染并执行显示的层。当前主要是velocity渲染，JS渲染，JSP渲染，移
     动端展示等。 
   - Web层：主要是对访问控制进行转发，各类基本参数校验，或者不复用的业务简单处理等。 
   - Service层：相对具体的业务逻辑服务层。 
   - Manager层：通用业务处理层，它有如下特征： 
        1） 对第三方平台封装的层，预处理返回结果及转化异常信息。 
        2） 对Service层通用能力的下沉，如缓存方案、中间件通用处理。 
        3） 与DAO层交互，对多个DAO的组合复用。 
   - DAO层：数据访问层，与底层MySQL、Oracle、Hbase、OB等进行数据交互。 
   - 外部接口或第三方平台：包括其它部门RPC开放接口，基础平台，其它公司的HTTP接口。

2. <font color="FFB800">【推荐】</font>（分层异常处理规约）在DAO层，产生的异常类型有很多，无法用细粒度的异常进行catch，使用catch(Exception e)方式，并throw new DAOException(e)，不需要打印日志，因为日志在Manager/Service层一定需要捕获并打印到日志文件中去，如果同台服务器再打日志，浪费性能和存储。<font color=red>在Service层出现异常时，必须记录出错日志到磁盘，尽可能带上参数信息，相当于保护案发现场。</font>Manager层与Service同机部署，日志方式与DAO层处理一致，如果是单独部署，则采用与Service一致的处理方式。Web层绝不应该继续往上抛异常，因为已经处于顶层，如果意识到这个异常将导致页面无法正常渲染，那么就应该直接跳转到友好错误页面，尽量加上友好的错误提示信息。开放接口层要将异常处理成错误码和错误信息方式返回
3. <font color="FFB800">【推荐】</font>分层领域模型规约：
   - DO（Data Object）：此对象与数据库表结构一一对应，通过DAO层向上传输数据源对象。 
   - DTO（Data Transfer Object）：数据传输对象，Service或Manager向外传输的对象。 
   - BO（Business Object）：业务对象，可以由Service层输出的封装业务逻辑的对象。 
   - Query：数据查询对象，各层接收上层的查询请求。注意超过2个参数的查询封装，禁止使用Map类
     来传输。 
   - VO（View Object）：显示层对象，通常是Web向模板渲染引擎层传输的对象 

### （2）二方库依赖

1.  <font color="red">【强制】</font>定义GAV遵从以下规则： 

   - <font color="#1E9FFF">G</font>roupID格式：com.{公司/BU }.业务线 [.子业务线]，最多4级。 
       说明：{公司/BU} 例如：alibaba/taobao/tmall/aliexpress等BU一级；子业务线可选。 
       正例：com.taobao.jstorm 或 com.alibaba.dubbo.register  

   - <font color="#1E9FFF">A</font>rtifactID格式：产品线名-模块名。语义不重复不遗漏，先到中央仓库去查证一下。 
       正例：dubbo-client / fastjson-api / jstorm-tool 

   - <font color="#1E9FFF">V</font>ersion

     命名方式：主版本号.次版本号.修订号

     1）主版本号：产品方向改变，或者大规模API不兼容，或者架构不兼容升级。  
      2） 次版本号：保持相对兼容性，增加主要功能特性，影响范围极小的API不兼容修改。 
      3） 修订号：保持完全兼容性，修复BUG、新增次要功能特性等

     ```sh
     说明：注意起始版本号必须为：1.0.0，而不是0.0.1。
     ```

   2.  <font color="red">【强制】</font>线上应用不要依赖SNAPSHOT版本（安全包除外）；正式发布的类库必须先去中央仓库进行查证，使RELEASE版本号有延续性，且版本号不允许覆盖升级。  

      ```sh
      说明：不依赖SNAPSHOT版本是保证应用发布的幂等性。另外，也可以加快编译时的打包构建。
      ```

   3. <font color="red">【强制】</font>二方库里可以定义枚举类型，参数可以使用枚举类型，但是接口返回值不允许使用枚举类型或者包含枚举类型的POJO对象。

   4. <font color="red">【强制】</font>依赖于一个二方库群时，必须定义一个统一的版本变量，避免版本号不一致。 

      ```sh
      说明：依赖springframework-core,-context,-beans，它们都是同一个版本，可以定义一个变量来保存版本：${spring.version}，定义依赖的时候，引用该版本
      ```

   5. <font color="red">【强制】</font>禁止在子项目的pom依赖中出现相同的GroupId，相同的ArtifactId，但是不同的Version。 

      ```sh
      说明：在本地调试时会使用各子项目指定的版本号，但是合并成一个war，只能有一个版本号出现在最后的lib目录中。曾经出现过线下调试是正确的，发布到线上却出故障的先例。
      ```

   6. <font color="FFB800">【推荐】</font>所有pom文件中的依赖声明放在\<dependencies>语句块中，所有版本仲裁放在\<dependencyManagement>语句块中。 

      ```sh
      说明：<dependencyManagement>里只是声明版本，并不实现引入，因此子项目需要显式的声明依赖，version和scope都读取自父pom。而<dependencies>所有声明在主pom的<dependencies>里的依赖都会自动引入，并默认被所有的子项目继承。
      ```

   7. <font color="FFB800">【推荐】</font>为避免应用二方库的依赖冲突问题，二方库发布者应当遵循以下原则： 
      1）精简可控原则。移除一切不必要的API和依赖，只包含 Service API、必要的领域模型对象、Utils类、常量、枚举等。如果依赖其它二方库，尽量是provided引入，让二方库使用者去依赖具体版本号；无log具体实现，只依赖日志框架。 
      2）稳定可追溯原则。每个版本的变化应该被记录，二方库由谁维护，源码在哪里，都需要能方便查到。除非用户主动升级版本，否则公共二方库的行为不应该发生变化

### （3）服务器

1. <font color="FFB800">【推荐】</font>高并发服务器建议调小TCP协议的time_wait超时时间。 
   说明：操作系统默认240秒后，才会关闭处于time_wait状态的连接，在高并发访问下，服务器端会因为
   处于time_wait的连接数太多，可能无法建立新的连接，所以需要在服务器上调小此等待值。 

   ```sh
   正例：在linux服务器上请通过变更/etc/sysctl.conf文件去修改该缺省值（秒）： 
    net.ipv4.tcp_fin_timeout = 30 
   ```

2. <font color="FFB800">【推荐】</font>调大服务器所支持的最大文件句柄数（File Descriptor，简写为fd）

   ```sh
   说明：主流操作系统的设计是将TCP/UDP连接采用与文件一样的方式去管理，即一个连接对应于一个fd。主流的linux服务器默认所支持最大fd数量为1024，当并发连接数很大时很容易因为fd不足而出现“open too many files”错误，导致新的连接无法建立。建议将linux服务器所支持的最大句柄数调高数倍（与服务器的内存数量相关）。 
   ```

3. <font color="FFB800">【推荐】</font>设置JVM启动参数 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/jvmdump

   JVM发生内存溢出（OOM）时，JVM会自动将堆转储dump文件，存放在-XX:HeapDumpPath指定的路径下。

   ```sh
   说明：OOM的发生是有概率的，甚至相隔数月才出现一例，出错时的堆内信息对解决问题非常有帮助。
   ```

4. <font color="FFB800">【推荐】</font>在线上生产环境，JVM的Xms和Xmx设置一样大小的内存容量，避免在GC 后调整堆大小带来的压力。

   ```sh
   -Xms 初始堆的大小，等价：-XX:InitialHeapSize，一般是物理内存的 1/64
   -Xmx 最大堆的大小 ，等价：-XX:MaxHeapSize，一般是物理内存的 1/4
   # 启动spring boot项目的jar包，并指定配置文件(application.properties文件与bat文件同文件夹放置)
   java -jar test3.jar --spring.config.location=application.properties
   ```

    服务器内部重定向必须使用forward；外部重定向地址必须使用URL Broker生成，否则因线上采用HTTPS协议而导致浏览器提示“不安全“。此外，还会带来URL维护不一致的问题。 



## 7、设计规约

1. <font color="red">【强制】</font>存储方案和底层数据结构的设计获得评审一致通过，并沉淀成为文档。 

   <font color="orange">说明：</font>有缺陷的底层数据结构容易导致系统风险上升，可扩展性下降，重构成本也会因历史数据迁移和系
   统平滑过渡而陡然增加，所以，存储方案和数据结构需要认真地进行设计和评审，生产环境提交执行后，
   需要进行double check。 

   ```sh
   正例：评审内容包括存储介质选型、表结构设计能否满足技术方案、存取性能和存储空间能否满足业务发展、表或字段之间的辩证关系、字段名称、字段类型、索引等；数据结构变更（如在原有表中新增字段）也需要进行评审通过后上线
   ```

2. <font color="red">【强制】</font>在需求分析阶段，如果与系统交互的User超过一类并且相关的User Case超过5个，使用<mark>用例图</mark>来表达更加清晰的结构化需求。 

3. <font color="red">【强制】</font>如果某个业务对象的状态超过3个，使用<mark>状态图</mark>来表达并且明确状态变化的各个触发条件。 
   <font color="orange">说明：</font>状态图的核心是对象状态，首先明确对象有多少种状态，然后明确两两状态之间是否存在直接转换
   关系，再明确触发状态转换的条件是什么。 

   ```sh
   正例：淘宝订单状态有已下单、待付款、已付款、待发货、已发货、已收货等。比如已下单与已收货这两
   种状态之间是不可能有直接转换关系的。
   ```

4. <font color="red">【强制】</font>如果系统中某个功能的调用链路上的涉及对象超过3个，使用<mark>时序图</mark>来表达并且明确各调用环节的输入与输出。 
   <font color="orange">说明：</font>时序图反映了一系列对象间的交互与协作关系，清晰立体地反映系统的调用纵深链路。

5. <font color="red">【强制】</font>如果系统中模型类超过5个，并且存在复杂的依赖关系，使用<mark>类图</mark>来表达并且明确类之间的关系。 

   ```sh
   说明：类图像建筑领域的施工图，如果搭平房，可能不需要，但如果建造蚂蚁Z空间大楼，肯定需要详细的施工图。
   ```

6. <font color="red">【强制】</font>如果系统中超过2个对象之间存在协作关系，并且需要表示复杂的处理流程，使用<mark>活动图</mark>来表示。 

   ```sh
   说明：活动图是流程图的扩展，增加了能够体现协作关系的对象泳道，支持表示并发等。 
   ```

7. <font color="FFB800">【推荐】</font>系统架构设计时明确以下目标： 

   - 确定系统边界。确定系统在技术层面上的做与不做。 
   - 确定系统内模块之间的关系。确定模块之间的依赖关系及模块的宏观输入与输出。 
   - 确定指导后续设计与演化的原则。使后续的子系统或模块设计在一个既定的框架内和技术方向上继
     续演化。 
   - 确定非功能性需求。非功能性需求是指安全性、可用性、可扩展性等。 

8. <font color="FFB800">【推荐】</font>需求分析与系统设计在考虑主干功能的同时，需要充分评估异常流程与业务边界。 

   ```sh
   反例：用户在淘宝付款过程中，银行扣款成功，发送给用户扣款成功短信，但是支付宝入款时由于断网演练产生异常，淘宝订单页面依然显示未付款，导致用户投诉。 
   ```

9. <font color="FFB800">【推荐】</font>类在设计与实现时要符合<mark>单一原则</mark> 
   说明：单一原则最易理解却是最难实现的一条规则，随着系统演进，很多时候，忘记了类设计的初衷。 

10. <font color="FFB800">【推荐】</font>

    - 系统设计阶段，根据依赖倒置原则，尽量依赖抽象类与接口，有利于扩展与维护。 
      说明：低层次模块依赖于高层次模块的抽象，方便系统间的解耦。

    - 系统设计阶段，注意对扩展开放，对修改闭合。

      说明：极端情况下，交付的代码是不可修改的，同一业务域内的需求变化，通过模块或类的扩展来实现。

    - 系统设计阶段，共性业务或公共行为抽取出来公共模块、公共配置、公共类、公共方法等，在系统中不出现重复代码的情况。

      说明：随着代码的重复次数不断增加，维护成本指数级上升。 

11. <font color="FFB800">【推荐】</font>避免如下误解：**敏捷开发 = 讲故事 + 编码 + 发布**。 
    说明：敏捷开发是快速交付迭代可用的系统，省略多余的设计方案，摒弃传统的审批流程，但核心关键点上
    的必要设计和文档沉淀是需要的。  

    ```sh
    反例：某团队为了业务快速发展，敏捷成了产品经理催进度的借口，系统中均是勉强能运行但像面条一样
    的代码，可维护性和可扩展性极差，一年之后，不得不进行大规模重构，得不偿失。 
    ```

12. <font color="FFB800">【参考】</font>设计文档的作用是明确需求、理顺逻辑、后期维护，次要目的用于指导编码。

    ```sh
    说明：避免为了设计而设计，系统设计文档有助于后期的系统维护和重构，所以设计结果需要进行分类归
    档保存。
    ```

13. <font color="FFB800">【参考】</font>可扩展性的本质是找到系统的变化点，并隔离变化点。 

    ```sh
    说明：世间众多设计模式其实就是一种设计模式即隔离变化点的模式。 
    正例：极致扩展性的标志，就是需求的新增，不会在原有代码交付物上进行任何形式的修改。 
    ```

14. <font color="FFB800">【参考】</font>设计的本质就是识别和表达系统难点。 
    说明：识别和表达完全是两回事，很多人错误地认为识别到系统难点在哪里，表达只是自然而然的事情，
    但是大家在设计评审中经常出现语焉不详，甚至是词不达意的情况。准确地表达系统难点需要具备如下能

    力： 表达规则和表达工具的熟练性。抽象思维和总结能力的局限性。基础知识体系的完备性。深入浅出的
    生动表达力。 

## 附3 错误码列表







