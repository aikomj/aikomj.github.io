---
layout: post
title: String的长度限制,Java的常见陷阱，生成随机数的4种方法
category: java
tags: [java]
keywords: java
excerpt: Integer.parseInt的转换失败陷阱，Bigdecimal.divide的除不尽陷阱，Collection.emptyList、Array.asList不支持添加/删除元素操作，List使用Iterator迭代遍历时删除，Java生成随机数的4种方式
lock: noneed
---

## 1、String的长度限制

本人就遇到过面试的时候问这个的，而且在之前开发的中也真实地遇到过这个String长度限制的场景（将某固定文件转码成Base64的形式用字符串存储，在运行时需要的时候在转回来，当时文件比较大），那这个规范限制到底是怎么样的。

String其实是使用的一个char类型的数组来存储字符串中的字符的，看源码

![](\assets\images\2020\java\string-1.jpg)

String既然是数组存储，那数组会有长度的限制吗？有限制，看String中返回长度的方法length()。

![](\assets\images\2020\java\string-2.jpg)

返回值类型是int，Java中定义数组时可以给数组指定长度的，不指定的话默认会根据数组元素来指定：

```java
int[] arr1 = new int[10]; // 定义一个长度为10的数组
int[] arr2 = {1,2,3,4,5}; // 那么此时数组的长度为5
```

我们通过源码来看看int类型对应的包装类Integer，其长度最大限制为2^31 -1，所以数组的最大长度是2^31-1 = 2147483647 = 4GB），但实际容量没有那么大

![](\assets\images\2020\java\string-4.jpg)

以上是我通过定义字面量的形式构造的10万个字符的字符串，编译之后虚拟机报错，说我们的字符串长度过长，为什么才10万个就报错了呢？

因为这里涉及到JVM编译规范的限制了，如果我们将字符串定义成字面量的形式，JVM在编译时会将其存放在常量池中并对它进行限制，JVM手册是如何说的。

<img src="\assets\images\2020\java\string-5.jpg" style="zoom:50%;" />

常量池中，每个 cp_info 项的格式必须相同，它们都以一个cp_info 类型的单字节tag项开头，后面 info[]项的内容由tag的类型所决定，常量类型如下：

<img src="\assets\images\2020\java\string-6.jpg" style="zoom:60%;" />

String类型的表示是CONSTANT_String ，具体是如何定义的。

<img src="\assets\images\2020\java\string-7.jpg" style="zoom:60%;" />

string_index 表示的是常量池的有效索引，其类型是CONSTANT_Utf8_info 结构体，表示如下：

![](\assets\images\2020\java\string-8.jpg)

在class文件中u2表示的是无符号数占2个字节，我们知道1个字节占8位 ，那么2个字节能表示的范围就是2^16- 1 = 65535 。说明：

- class文件中文件内容类型解释

  定义一组私有数据类型来表示 Class 文件的内容，它们包括 u1，u2 和 u4，分别代 表了 1、2 和 4 个字节的无符号数。每个 Class 文件都是由 8 字节为单位的字节流组成，所有的 16 位、32 位和 64 位长度的数据将被构造成 2 个、4 个和 8 个 8 字节单位来表示。那么2个字节能表示的范围就是2^16- 1 = 65535。所以 length的最大值是65535。

- 程序异常处理的有效范围解释

  start_pc 和 end_pc 两项的值表明了异常处理器在 code[]数组中的有效范围。start_pc 必须是对当前 code[]数组中某一指令的操作码的有效索引，end_pc 要 么是对当前 code[]数组中某一指令的操作码的有效索引，要么等于 code_length 的值，即当前 code[]数组的长度。start_pc 的值必须比 end_pc 小。

当程序计数器在范围`[start_pc, end_pc)`内时，异常处理器就将生效。即设 x 为 异常句柄的有效范围内的值，x 满足：`start_pc ≤ x < end_pc`。实际上，end_pc 值本身不属于异常处理器的有效范围这点属于 Java 虚拟机历史上 的一个设计缺陷：如果 Java 虚拟机中的一个方法的 code 属性的长度刚好是 65535 个字节，并且以一个 1 个字节长度的指令结束，那么这条指令将不能被异常处理器 所处理。不过编译器可以通过限制任何方法、实例初始化方法或类初始化方法的`code[]`数组最大长度为 65534，这样可以间接弥补这个 BUG。测试：

<img src="\assets\images\2020\java\string-9.jpg" style="zoom:50%;" />

**总结**：

<mark>String的最大长度是65534</mark>

字符串的内容是由一个字符数组 char[] 来存储的，数组的长度及索引是int，且String类中返回字符串长度的方法length() 的返回值也是int ，查看java源码中的类Integer的最大范围是2^31 -1，数组是从0开始的，所以最大长度是2^31=4GB。但是通过翻阅java虚拟机手册对class文件格式的定义以及常量池中对String类型的结构体定义，我们知道对于索引定义了u2，就是无符号占2个字节，2个字节可以表示的最大范围是2^16 -1 = 65535。但是由于JVM需要1个字节表示结束指令，所以最大范围就是65534了。超出这个范围在编译时期是会报错的，但是运行时拼接或者赋值的话范围是在整形的最大范围2^31=4GB

## 2、JDK的常见陷阱

### String.valueOf()的陷阱

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

### Integer.parseInt()的陷阱

事故现场：一次业务场景为拉取订单,打出订单列表记录,财务人员需要拉出对账,结果总是发现很奇怪的一个现象,每次拉取少很多数据,。还好财务发现了,要不然和第三方财务对账就会亏很多钱，最终发现是订单的一个字段值转Integer出错了,那个订单下的字段值是120.0通过Integer.parseInt()直接报错了,恰好开发人员认为这段开发肯定没问题,因此就没有catch异常,最后找了很久才发现(因为涉及到第三方,还让别人查了半天...). 知道真相的我们都有点汗颜,这么丁点的错误排查了很久,实在是不应该啊。

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

### Bigdecimal.divide()的陷阱

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

### Collections.emptyList()的陷阱

```java
public static void main(String[] args) {
  List<String> list = Collections.emptyList();
  list.add("hello");
}
```

执行结果

![](\assets\images\2021\javabase\empty-list.jpg)

不支持add、remove等操作，那是因为Collections.emptyList()返回的是一个内部类EmptyList，该类不具有add、remove等操作

![](\assets\images\2021\javabase\empty-list-2.jpg)

就跟`Arrays.asList()`返回的ArrayList 也是自定义的内部类，本质上还是数组，不是`java.util.ArrayList`

![](\assets\images\2021\javabase\arrays-aslist-7.png)

不支持添加、删除元素操作，并不是`java.util.List`

### List遍历时删除

一边遍历一边删除，必须使用`Iterator.remove`迭代器的方式删除，规避异常

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

## 3、生成随机数的4种方式

### Random

Random 类诞生于 JDK 1.0，它产生的随机数其实是伪随机数，也就是有规则的随机数。它使用的随机算法是`linear congruential pseudorandom number generator `，简称LGC线性同余法伪随机数。它的原理是以一个起源数字为种子数，在它基础上进行一定的变换，从而产生需要的随机数字。

<mark>Random 对象在种子数相同的情况下，相同次数生成的随机数是相同的</mark>。比如两个种子数相同的 Random 对象，第一次生成的随机数字完全相同，第二次生成的随机数字也完全相同。默认情况下 new Random() 使用的是当前纳秒时间作为种子数的。

```java
Random random = new Random(); // 当前纳秒时间作为种子数
for (int i = 0; i < 10; i++) {
    // 生成 0-9 随机整数
    int number = random.nextInt(10); // 生成一个从 0 到 10 的随机数（不包含 10）
    System.out.println("生成随机数：" + number);
}
```

执行结果

<img src="\assets\images\2021\javabase\random-1.jpg" style="zoom:50%;" />

<mark>优点</mark>

- Random 使用 LGC 算法生成伪随机数的执行效率比较高

<mark>缺点</mark>

- 如果 Random 的随机种子一样的话，每次生成的随机数都是可预测的，相同次数生成的随机数是相同的

  如下代码所示，当我们给两个线程设置相同的种子数的时候，会发现每次产生的随机数也是相同的

  ```java
   // 创建两个线程
  for (int i = 0; i < 2; i++) {
      new Thread(() -> {
          // 创建 Random 对象，设置相同的种子
          Random random = new Random(1024);
          // 生成 3 次随机数
          for (int j = 0; j < 3; j++) {
              // 生成随机数
              int number = random.nextInt();
              // 打印生成的随机数
              System.out.println(Thread.currentThread().getName() + ":" +
                                 number);
              // 休眠 200 ms
              try {
                  Thread.sleep(200);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
              System.out.println("---------------------");
          }
      }).start();
  }
  ```

  执行结果：

  <img src="\assets\images\2021\javabase\random-2.jpg" style="zoom:50%;" />

> Random是线程安全的

线程安全指的是在多线程的场景下，程序的执行结果和预期的结果一致，就叫线程安全的，否则则为非线程安全的（也叫线程安全问题）。比如有两个线程，第一个线程执行 10 万次 ++ 操作，第二个线程执行 10 万次 -- 操作，那么最终的结果应该是没加也没减，如果程序最终的结果和预期不符，则为非线程安全的。注意，两个线程都在操作共享资源才会产生线程安全问题。

我们来看 Random 的实现源码：

```java
// JDK 1.8.0_211
public Random() {
    this(seedUniquifier() ^ System.nanoTime());
}

public int nextInt() {
    return next(32);
}

protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed)); // CAS（Compare and Swap）生成随机数
    return (int)(nextseed >>> (48 - bits));
}
```

从以上源码可以看出，Random 底层使用的是 CAS（Compare and Swap，比较并替换）来解决线程安全问题的，因此对于绝大数随机数生成的场景，使用 Random 不乏为一种很好的选择。

Java 并发机制实现原子操作有两种：

- 一种是锁

  如Java的synchronized 关键字、ReentrantLock锁，Redis分布式锁

- 一种是 CAS

  CAS 是 Compare And Swap（比较并替换）的缩写，java.util.concurrent.atomic 中的很多类，如（AtomicInteger AtomicBoolean AtomicLong等）都使用了 CAS 机制来实现。

### ThreadLocalRandom

ThreadLocalRandom 是 JDK 1.7 新提供的类，它属于 JUC（java.util.concurrent）下的一员，为什么有了 Random 之后还会再创建一个 ThreadLocalRandom？

原因很简单，通过上面 Random 的源码我们可以看出，Random 在生成随机数时使用的 CAS 来解决线程安全问题的，然而<mark>CAS 在线程竞争比较激烈的场景中效率是非常低的</mark>，原因是 CAS 对比时老有其他的线程在修改原来的值，所以导致 CAS 对比失败，所以它要一直循环来尝试进行 CAS 操作。所以<mark>在多线程竞争比较激烈的场景可以使用 ThreadLocalRandom 来解决 Random 执行效率比较低的问题</mark>。

当我们第一眼看到 ThreadLocalRandom 的时候，一定会联想到一次类 ThreadLocal，确实如此，ThreadLocalRandom 的实现原理与 ThreadLocal 类似，它相当于给每个线程一个自己的本地种子，从而就可以避免因多个线程竞争一个种子，而带来的额外性能开销了。

![](\assets\images\2021\javabase\threadLocalRandom-1.jpg)

```java
// 得到 ThreadLocalRandom 对象
ThreadLocalRandom random = ThreadLocalRandom.current();
for (int i = 0; i < 10; i++) {
    // 生成 0-9 随机整数
    int number = random.nextInt(10);
    // 打印结果
    System.out.println("生成随机数：" + number);
}
```

> 实现原理

ThreadLocalRandom 的实现原理和 ThreadLocal 类似，它是让每个线程持有自己的本地种子，该种子在生成随机数时候才会被初始化，实现源码如下：

```java
public int nextInt(int bound) {
    // 参数效验
    if (bound <= 0)
        throw new IllegalArgumentException(BadBound);
    // 根据当前线程中种子计算新种子
    int r = mix32(nextSeed());
    int m = bound - 1;
    // 根据新种子和 bound 计算随机数
    if ((bound & m) == 0) // power of two
        r &= m;
    else { // reject over-represented candidates
        for (int u = r >>> 1;
             u + m - (r = u % bound) < 0;
             u = mix32(nextSeed()) >>> 1)
            ;
    }
    return r;
}

final long nextSeed() {
    Thread t; long r; // read and update per-thread seed
    // 获取当前线程中 threadLocalRandomSeed 变量，然后在种子的基础上累加 GAMMA 值作为新种子
    // 再使用 UNSAFE.putLong 将新种子存放到当前线程的 threadLocalRandomSeed 变量中
    UNSAFE.putLong(t = Thread.currentThread(), SEED,
                   r = UNSAFE.getLong(t, SEED) + GAMMA); 
    return r;
}
```

- 优点

  ThreadLocalRandom 结合了 Random 和 ThreadLocal 类，并被隔离在当前线程中。因此它通过避免竞争操作种子数，从而**在多线程运行的环境中实现了更好的性能**，而且也保证了它的**线程安全**。

  另外，不同于 Random， ThreadLocalRandom 明确不支持设置随机种子。它重写了 Random 的`setSeed(long seed)` 方法并直接抛出了 `UnsupportedOperationException`异常，因此**降低了多个线程出现随机数重复的可能性**。

  ```java
  public void setSeed(long seed) {
      // only allow call from super() constructor
      if (initialized)
          throw new UnsupportedOperationException();
  }
  ```

  只要程序中调用了 setSeed() 方法就会抛出 `UnsupportedOperationException` 异常

- 缺点

  虽然 ThreadLocalRandom 不支持手动设置随机种子的方法，但并不代表 ThreadLocalRandom 就是完美的，当我们查看 ThreadLocalRandom 初始化随机种子的方法 initialSeed() 源码时发现，默认情况下它的随机种子也是以当前时间有关，源码如下：

  ```java
  private static long initialSeed() {
      // 尝试获取 JVM 的启动参数
      String sec = VM.getSavedProperty("java.util.secureRandomSeed");
      // 如果启动参数设置的值为 true，则参数一个随机 8 位的种子
      if (Boolean.parseBoolean(sec)) {
          byte[] seedBytes = java.security.SecureRandom.getSeed(8);
          long s = (long)(seedBytes[0]) & 0xffL;
          for (int i = 1; i < 8; ++i)
              s = (s << 8) | ((long)(seedBytes[i]) & 0xffL);
          return s;
      }
      // 如果没有设置启动参数，则使用当前时间有关的随机种子算法
      return (mix64(System.currentTimeMillis()) ^
              mix64(System.nanoTime()));
  }
  ```

  从上述源码可以看出，当我们设置了启动参数“-Djava.util.secureRandomSeed=true”时，ThreadLocalRandom 会产生一个随机种子，一定程度上能缓解随机种子相同所带来随机数可预测的问题，然而**默认情况下如果不设置此参数，那么在多线程中就可以因为启动时间相同，而导致多个线程在每一步操作中都会生成相同的随机数**。

### SecureRandom

SecureRandom 继承自 Random，该类提供加密强随机数生成器。<mark>SecureRandom 不同于 Random，它收集了一些随机事件，比如鼠标点击，键盘点击等，SecureRandom 使用这些随机事件作为种子。这意味着，种子是不可预测的</mark>，而不像 Random 默认使用系统当前时间的毫秒数作为种子，从而避免了生成相同随机数的可能性。

```java
// 创建 SecureRandom 对象，并设置加密算法
SecureRandom random = SecureRandom.getInstance("SHA1PRNG");
for (int i = 0; i < 10; i++) {
    // 生成 0-9 随机整数
    int number = random.nextInt(10);
    // 打印结果
    System.out.println("生成随机数：" + number);
}
```

SecureRandom 默认支持两种加密算法：

1. SHA1PRNG 算法，提供者 sun.security.provider.SecureRandom；
2. NativePRNG 算法，提供者 sun.security.provider.NativePRNG。

当然除了上述的操作方式之外，你还可以选择使用 `new SecureRandom()` 来创建 SecureRandom 对象，实现代码如下：

```java
SecureRandom secureRandom = new SecureRandom();
```

通过 new 初始化 SecureRandom，默认会使用 NativePRNG 算法来生成随机数，但是也可以配置 JVM 启动参数“-Djava.security”参数来修改生成随机数的算法，或选择使用 `getInstance("算法名称")` 的方式来指定生成随机数的算法。

### Math

Math 类诞生于 JDK 1.0，它里面包含了用于执行基本数学运算的属性和方法，如初等指数、对数、平方根和三角函数，当然它里面也包含了生成随机数的静态方法 `Math.random()` ，**此方法会产生一个 0 到 1 的 double 值**

```java
for (int i = 0; i < 10; i++) {
    // 产生随机数
    double number = Math.random();
    System.out.println("生成随机数：" + number);
}
```

执行结果：

<img src="\assets\images\2021\javabase\math-random.jpg" style="zoom:50%;" />

当然如果你想**用它来生成一个一定范围的 int 值**也是可以的，你可以这样写：

```java
for (int i = 0; i < 10; i++) {
    // 生成一个从 0-99 的整数
    int number = (int) (Math.random() * 100);
    System.out.println("生成随机数：" + number);
}
```

执行结果：

<img src="\assets\images\2021\javabase\math-random-2.jpg" style="zoom:50%;" />

> 实现原理

通过分析 `Math` 的源码我们可以得知：当第一次调用 `Math.random()` 方法时，自动创建了一个伪随机数生成器，实际上用的是 `new java.util.Random()`，当下一次继续调用 `Math.random()` 方法时，就会使用这个新的伪随机数生成器。源码如下：

```java
public static double random() {
    return RandomNumberGeneratorHolder.randomNumberGenerator.nextDouble();
}

private static final class RandomNumberGeneratorHolder {
    static final Random randomNumberGenerator = new Random();
}
```

### 总结

我们介绍了 4 种生成随机数的方法，

- 其中 Math 是对 Random 的封装，所以二者比较类似。Random 生成的是伪随机数，是以当前纳秒时间作为种子数的，并且在多线程竞争比较激烈的情况下因为要进行 CAS 操作，所以存在一定的性能问题，但**对于绝大数应用场景来说，使用 Random 已经足够了。**

- **当在竞争比较激烈的场景下可以使用 ThreadLocalRandom 来替代 Random，但如果对安全性要求比较高的情况下，可以使用 SecureRandom 来生成随机数**，因为 SecureRandom 会收集一些随机事件来作为随机种子，所以 SecureRandom 可以看作是生成真正随机数的一个工具类。