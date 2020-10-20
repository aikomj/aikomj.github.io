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

     使用lombok插件的@Data注解会包含toString方法，使用IDEA对class文件反编译得到最终的类，注意使用@Data的POJO类有继承父类的话，要加上

    ```
    @EqualsAndHashCode(callSuper = true)
    @ToString(callSuper = true)
    ```

    lombok生成子类的equals和toString方法都会去调用父类的equals和toString方法，默认是不会去调用的，可以通过IDEA反编译class文件看最终源码知道。

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
      System.out.println(t1==t2);
      System.out.println(t1.equals(t2));
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
    6） 级联调用obj.getA().getB().getC()；一连串调用，易产生NPE。 
   正例：使用JDK8的Optional类来防止NPE问题

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







