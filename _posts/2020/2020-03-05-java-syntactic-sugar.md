---
layout: post
title: java的语法糖
category: java
tags: [java]
keywords: java
excerpt: 专为开发者设计，提高开发效率，代码更优雅易读
lock: noneed
---
## 什么是语法糖

​	语法糖（Syntactic Sugar），也称糖衣语法，是由英国计算机学家 Peter.J.Landin 发明的一个术语，指在计算机语言中添加的某种语法，这种语法对语言的功能并没有影响，更方便程序员使用。<font color="red">简而言之，语法糖让程序更加简洁，有更高的可读性</font>。

​	我们所熟知的编程语言中几乎都有语法糖，java 从1.7开始就一直在添加各种糖，未来还会向着“高糖”的方向发展。

**解语法糖**

​	java虚拟机其实并不支持语法糖的，所以在编译阶段语法糖会被还原成简单的集成语法结构，这个过程就是解语法糖。如果你去看com.sun.tools.javac.main.JavaCompiler的源码，就会发现在compile()中有一个步骤是调用desugar()，这个方法就是负责解语法糖的。

![image-20200305140952384](/assets/images/2020/java-compile-desugar.png)

### 糖块1. switch 支持 String 与枚举

java7中switch开始支持String，对于编译器来说，<font color="red">switch中其实只能使用整型，任何类型的比较都要转换成整型</font>,比如byte。short，char(转换成ackii码是整型)以及int。

```java
public class switchDemoString {
    public static void main(String[] args) {
        String str = "world";
        switch (str) {
        case "hello":
            System.out.println("hello");
            break;
        case "world":
            System.out.println("world");
            break;
        default:
            break;
        }
    }
}
```

// 反编译后得到

```java
public class switchDemoString
{
    public switchDemoString()
    {
    }
    public static void main(String args[])
    {
        String str = "world";
        String s;
        switch((s = str).hashCode())
        {
        default:
            break;
        case 99162322:
            if(s.equals("hello"))
                System.out.println("hello");
            break;
        case 113318802:
            if(s.equals("world"))
                System.out.println("world");
            break;
        }
    }
}
```

看到这个代码，字符串的switch是通过equals()和hashCode()来实现的，hashCode()方法返回的是int，而不是long。进行switch的实际是哈希值，然后通过equals方法比较进行安全检查，这个检查是必要的，因为哈希可能会发生碰撞。因此它的性能不如使用枚举进行switch或者使用纯整数switch，但这也不是很差。

### 糖块2. 泛型

java虚拟机在编译阶段通过类型擦除的方式来解语法糖。

类型擦除的过程：

1. 将所有泛型参数用最顶级的父类型来替代
2. 移除所有类型参数

例子：

```java
Map<String, String> map = new HashMap<String, String>();  
map.put("name", "hollis");  
map.put("wechat", "Hollis");  
map.put("blog", "www.dalbll.com");  

public static <A extends Comparable<A>> A max(Collection<A> xs) {
    Iterator<A> xi = xs.iterator();
    A w = xi.next();
    while (xi.hasNext()) {
        A x = xi.next();
        if (w.compareTo(x) < 0)
            w = x;
    }
    return w;
}
```

类型擦除后：

```java
Map map = new HashMap();  
map.put("name", "hollis");  
map.put("wechat", "Hollis");  
map.put("blog", "www.dalbll.com");  

public static Comparable max(Collection xs){
    Iterator xi = xs.iterator();
    Comparable w = (Comparable)xi.next();
    while(xi.hasNext())
    {
        Comparable x = (Comparable)xi.next();
        if(w.compareTo(x) < 0)
            w = x;
    }
    return w;
}

虚拟机中没有泛型，只有普通类和普通方法，所以泛型的类型参数在编译阶段会被擦除，如List<String>.class或List<Integer>是不存在的，只有List.class存在。
```

### 糖块3.自动装箱和拆箱

自动装箱就是将基本类型转换成封装类，如int的变量转换成Integer对象，反之就是拆箱。

基本类型byte, short, char, int, long, float, double 和 boolean 

对应的封装类为Byte, Short, Character, Integer, Long, Float, Double, Boolean

例子：

```java
// 装箱
public static void main(String[] args) {
    int i = 10;
    Integer n = i;
}
// 拆箱
public static void main(String[] args) {
    Integer i = 10;
    int n = i;
}
```

反编译后

```java
public static void main(String args[])
{
    int i = 10;
    Integer n = Integer.valueOf(i);
}
public static void main(String args[])
{
    Integer i = Integer.valueOf(10);
    int n = i.intValue();
}
```

从源码看出，装箱过程是调用包装器的valueOf方法实现的，拆箱过程是调用包装器的xxxValue方法实现的。

### 糖块4.可变数量的参数

这个是在java1.5引入的特性。

```java
public static void main(String[] args)
    {
        print("Holis", "公众号:dalbll", "博客：www.dalbll.com", "QQ：2274840169");
    }

public static void print(String... strs)
{
    for (int i = 0; i < strs.length; i++)
    {
        System.out.println(strs[i]);
    }
}
```

反编译后：

```java
public static void main(String args[])
{
    print(new String[] {
        "dalbll", "\u516C\u4F17\u53F7:Hollis", "\u535A\u5BA2\uFF1Awww.dalbll.com.com", "QQ\uFF1A907607222"
    });
}

public static transient void print(String strs[])
{
    for(int i = 0; i < strs.length; i++)
        System.out.println(strs[i]);
}
```

从编译代码看出，可变参数被使用的时候，会创建一个包含实参数值的数组，作为参数传递给调用方法的。

### 糖块5. 枚举

枚举是一个特殊的类，特殊之处就是它的构造函数的调用可以自己定义名字去使用;

例子

```java
public enum t {
    SPRING,SUMMER;
}
```

反编译后

```java
public final class T extends Enum
{
    private T(String s, int i)
    {
        super(s, i);
    }
    public static T[] values()
    {
        T at[];
        int i;
        T at1[];
        System.arraycopy(at = ENUM$VALUES, 0, at1 = new T[i = at.length], 0, i);
        return at1;
    }

    public static T valueOf(String s)
    {
        return (T)Enum.valueOf(demo/T, s);
    }

    public static final T SPRING;
    public static final T SUMMER;
    private static final T ENUM$VALUES[];
    static
    {
        SPRING = new T("SPRING", 0);
        SUMMER = new T("SUMMER", 1);
        ENUM$VALUES = (new T[] {
            SPRING, SUMMER
        });
    }
}
```

当我们使用enum定义一个枚举，编译器会创建一个final类型的类型继承Enum类。

### 糖块6.内部类

内部类可以理解为外部类的一个普通成员，它只是编译时的一个概念

```java
public class Outter {
    private String userName;

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public static void main(String[] args) {

    }

    class Inner{
        private String name;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
    }
}
```

编译后会生成两个class文件：Outter.class和Outter$Inner.class

反编译后

```java
public class Outter{
    public String getUserName()
    {
        return userName;
    }
    public void setUserName(String userName){
        this.userName = userName;
    }
    public static void main(String args1[])
    {
    }
    private String userName;
  
    class Inner
    {
        public String getName()
        {
            return name;
        }
        public void setName(String name)
        {
            this.name = name;
        }
        private String name;
        final OutterClass this$0;

        InnerClass()
        {
            this.this$0 = OutterClass.this;
            super();
        }
    }
}
```

### 糖块7. 断言

java在执行的时候默认不开启断言检查，需要开启-enableassertions或-ea

代码：

```java
public class AssertTest {
    public static void main(String args[]) {
        int a = 1;
        int b = 1;
        assert a == b;
        System.out.println("公众号：dalbll");
        assert a != b : "Hollis";
        System.out.println("博客：www.dalbll.com");
    }
}
```

反编译后

```java
public class AssertTest {
   public AssertTest()
    {
    }
    public static void main(String args[]){
    int a = 1;
    int b = 1;
    if(!$assertionsDisabled && a != b)
        throw new AssertionError();
    System.out.println("\u516C\u4F17\u53F7\uFF1AHollis");
    if(!$assertionsDisabled && a == b)
    {
        throw new AssertionError("dalbll");
    } else
    {
        System.out.println("\u535A\u5BA2\uFF1Awww.dalbll.com");
        return;
    }
}

static final boolean $assertionsDisabled = !com/dalbll/suguar/AssertTest.desiredAssertionStatus();
}
```

从代码看出，断言底层实现是if语言，如果断言结果为true，什么都不做，程序继续执行，为false，就抛出AssertError来打断程序的执行。

### 糖块8.数值字面量

java7中，允许在整数或浮点数间插入任意多个下划线，这些下划线不会对数值产生影响，目的是方便阅读

例子

```java
public class Test {
    public static void main(String... args) {
        int i = 10_000;
        System.out.println(i);
    }
}
```

反编译后

```java
public class Test
{
  public static void main(String[] args)
  {
    int i = 10000;
    System.out.println(i);
  }
}
```

可以看到编译阶段会把下划线_去掉

### 糖块9.for-each

for-each是用来增强for循环的，我们看这个语法糖背后是如何实现的

```java
public static void main(String... args) {
    String[] strs = {"Hollis", "公众号：dalbll", "博客：www.dalbll.com"};
    for (String s : strs) System.out.println(s);
    List<String> strList = ImmutableList.of("Hollis", "公众号：dalbll", "博客：www.dalbll.com");
    for (String s : strList) {
        System.out.println(s);
    }
}
```

反编译后

```java
public static transient void main(String args[])
{
    String strs[] = {
        "dalbll", "\u516C\u4F17\u53F7\uFF1Adalbll", "\u535A\u5BA2\uFF1Awww.dalbll.com"
    };
    String args1[] = strs;
    int i = args1.length;
    for(int j = 0; j < i; j++)
    {
        String s = args1[j];
        System.out.println(s);
    }
    List strList = ImmutableList.of("dalbll", "\u516C\u4F17\u53F7\uFF1Adalbll", "\u535A\u5BA2\uFF1Awww.dalbll.com");
    String s;
    for(Iterator iterator = strList.iterator(); iterator.hasNext(); System.out.println(s))
        s = (String)iterator.next();

}
```

可以看出，for-each是用普通的for循环和迭代器实现的。

### 糖块10.try-with-resource

java里，操作文件IO流、数据库连接的开销是非常昂贵的资源，用完之后必须通过close方法关闭，否则可能会导致内存泄露问题，我们传统的写法是使用try-catch-finally

```java 
public static void main(String[] args) {
    BufferedReader br = null;
    try {
        String line;
        br = new BufferedReader(new FileReader("d:\\hollischuang.xml"));
        while ((line = br.readLine()) != null) {
            System.out.println(line);
        }
    } catch (IOException e) {
        // handle exception
    } finally {
        try {
            if (br != null) {
                br.close();
            }
        } catch (IOException ex) {
            // handle exception
        }
    }
}
```

java7开始支持更好的写法，使用try-with-resource语句

```java
public static void main(String... args) {
    try (BufferedReader br = new BufferedReader(new FileReader("d:\\ hollischuang.xml"))) {
        String line;
        while ((line = br.readLine()) != null) {
            System.out.println(line);
        }
    } catch (IOException e) {
        // handle exception
    }
}
```

是不是更简洁优雅了，反编译看它的背后原理

```java
public static transient void main(String args[])
    {
        BufferedReader br;
        Throwable throwable;
        br = new BufferedReader(new FileReader("d:\\ hollischuang.xml"));
        throwable = null;
        String line;
        try
        {
            while((line = br.readLine()) != null)
                System.out.println(line);
        }
        catch(Throwable throwable2)
        {
            throwable = throwable2;
            throw throwable2;
        }
        if(br != null)
            if(throwable != null)
                try
                {
                    br.close();
                }
                catch(Throwable throwable1)
                {
                    throwable.addSuppressed(throwable1);
                }
            else
                br.close();
            break MISSING_BLOCK_LABEL_113;
            Exception exception;
            exception;
            if(br != null)
                if(throwable != null)
                    try
                    {
                        br.close();
                    }
                    catch(Throwable throwable3)
                      {
                        throwable.addSuppressed(throwable3);
                    }
                else
                    br.close();
        throw exception;
        IOException ioexception;
        ioexception;
    }
}
```

可以看出是编译器帮我们做了关闭资源的操作，所以语法糖的作用是方便程序员的是使用，最终会转成编译器认识的语言。

### 糖块11.Lambda表达式

这个应该要经常使用熟悉，作为java程序员标配要懂的知识。lambda表达式是依赖JVM底层提供的api实现的。

```java
public static void main(String... args) {
    List<String> strList = ImmutableList.of("dalbll", "公众号：dalbll", "博客：www.dalbll.com");

    strList.forEach( s -> { System.out.println(s); } );
}
```

反编译后

```java
public static /* varargs */ void main(String ... args) {
    ImmutableList strList = ImmutableList.of((Object)"dalbll", (Object)"\u516c\u4f17\u53f7\uff1adalbll", (Object)"\u535a\u5ba2\uff1awww.dalbll.com");
    strList.forEach((Consumer<String>)LambdaMetafactory.metafactory(null, null, null, (Ljava/lang/Object;)V, lambda$main$0(java.lang.String ), (Ljava/lang/String;)V)());
}

private static /* synthetic */ void lambda$main$0(String s) {
    System.out.println(s);
}
```

可以看出forEach方法中，调用了java.lang.invoke.LambdaMetafactory.metafactory方法，该方法的第五个参数调用了lambda$main$0方法进行了输出。



### 总结

语法糖是提供给开发人员方便开发的语法，可以提升开发效率，要想被执行，需要进行解糖转换为jvm认识的语法，解糖后会发现，它其实是由其他更简单的语法构成的。

