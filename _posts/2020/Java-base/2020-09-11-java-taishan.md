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







