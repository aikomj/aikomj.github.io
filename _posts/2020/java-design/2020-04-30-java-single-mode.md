---
layout: post
title: 23种设计模式之单例模式
category: java-design
tags: [java-design]
keywords: java
excerpt: 适用场景，与其他模式的关系，四种写法-懒汉式、饿汉式、静态内部类、枚举、双重校验锁
lock: noneed
---

<mark>单例模式 Singleton</mark>

![](\assets\images\2021\javabase\design-singleton.png)

## 1、意图

以下内容来源于网站 Refactoring Guru  : [https://refactoringguru.cn/design-patterns/singleton](https://refactoringguru.cn/design-patterns/singleton)

创建型模式：提供创建对象的机制， 增加已有代码的灵活性和可复用性

**单例模式**是一种创建型设计模式， 让你能够保证一个类只有一个实例， 并提供一个访问该实例的全局节点。

![](\assets\images\2021\javabase\singleton.png)

### 问题

1. 保证一个类只有一个实例

   为什么会有人想要控制一个类所拥有的实例数量？ 最常见的原因是控制某些共享资源 （例如数据库或文件） 的访问权限。Spring的controller类默认是单例的，在程序启动时会创建一个controller单例，每次request请求访问的都是同一个controller实例。

   它的运作方式是这样的： 如果你创建了一个对象， 同时过一会儿后你决定再创建一个新对象， 此时你会获得之前已创建的对象， 而不是一个新对象。

   ![](\assets\images\2021\javabase\singleton-comic-1-zh.png)

   

2. 为该实例提供一个全局访问节点

   和全局变量一样， 单例模式也允许在程序的任何地方访问特定对象。 但是它可以保护该实例不被其他代码覆盖。

### 解决方案

所有单例的实现都包含以下两个相同的步骤： 

- 将默认构造函数设为私有， 防止其他对象使用单例类的 `new`运算符。 
- 新建一个静态构建方法作为构造函数。 该函数会 “偷偷” 调用私有构造函数来创建对象， 并将其保存在一个静态成员变量中。 此后所有对于该函数的调用都将返回这一缓存对象。

### 真实世界类比

政府是单例模式的一个很好的示例。 一个国家只有一个官方政府。 不管组成政府的每个人的身份是什么，  “某政府” 这一称谓总是鉴别那些掌权者的全局访问节点。

### 单例模式结构

![](/assets/images/2021/javabase/singleton-structure.jpg)

### 伪代码

在本例中， 数据库连接类即是一个**单例**。 该类不提供公有构造函数， 因此获取该对象的唯一方式是调用 `获取实例`方法。 该方法将缓存首次生成的对象， 并为所有后续调用返回该对象。

```java
// 数据库类会对`getInstance（获取实例）`方法进行定义以让客户端在程序各处
// 都能访问相同的数据库连接实例。
class Database is
    // 保存单例实例的成员变量必须被声明为静态类型。
    private static field instance: Database

    // 单例的构造函数必须永远是私有类型，以防止使用`new`运算符直接调用构
    // 造方法。
    private constructor Database() is
        // 部分初始化代码（例如到数据库服务器的实际连接）。
        // ...

    // 用于控制对单例实例的访问权限的静态方法。
    public static method getInstance() is
        if (Database.instance == null) then
            acquireThreadLock() and then
                // 确保在该线程等待解锁时，其他线程没有初始化该实例。
                if (Database.instance == null) then
                    Database.instance = new Database()
        return Database.instance

    // 最后，任何单例都必须定义一些可在其实例上执行的业务逻辑。
    public method query(sql) is
        // 比如应用的所有数据库查询请求都需要通过该方法进行。因此，你可以
        // 在这里添加限流或缓冲逻辑。
        // ...

class Application is
    method main() is
        Database foo = Database.getInstance()
        foo.query("SELECT ...")
        // ...
        Database bar = Database.getInstance()
        bar.query("SELECT ...")
        // 变量 `bar` 和 `foo` 中将包含同一个对象。
```

### 单例模式适用场景

-  如果程序中的某个类对于所有客户端只有一个可用的实例， 可以使用单例模式。

- 如果你需要更加严格地控制全局变量， 可以使用单例模式。

  单例模式与全局变量不同， 它保证类只存在一个实例。 除了单例类自己以外， 无法通过任何方式替换缓存的实例。 

  请注意， 你可以随时调整限制并设定生成单例实例的数量， 只需修改 `获取实例`方法， 即 getInstance 中的代码即可实现。

### 实现方式

1. 在类中添加一个私有静态成员变量用于保存单例实例。 

2. 声明一个公有静态构建方法用于获取单例实例。 

3. 在静态方法中实现"延迟初始化"。 该方法会在首次被调用时创建一个新对象， 并将其存储在静态成员变量中。 此后该方法每次被调用时都返回该实例。 

4. 将类的构造函数设为私有。 类的静态方法仍能调用构造函数， 但是其他对象不能调用。 

5. 检查客户端代码， 将对单例的构造函数的调用替换为对其静态构建方法的调用。

### 优缺点

![](\assets\images\2021\javabase\design-singleton-good.png)

违反“单一职责”原则，指的是多个线程（客户端）共享单个调用类实例处理业务，按常理应该是每个客户端应该new一个调用类实例处理业务的，这叫做单一职责。

### 与其他模式的关系

- [外观模式](https://refactoringguru.cn/design-patterns/facade)类通常可以转换为[单例模式](https://refactoringguru.cn/design-patterns/singleton)类， 因为在大部分情况下一个外观对象就足够了。
- 如果你能将对象的所有共享状态简化为一个享元对象， 那么[享元模式](https://refactoringguru.cn/design-patterns/flyweight)就和[单例](https://refactoringguru.cn/design-patterns/singleton)类似了。 但这两个模式有两个根本性的不同。

  1. 只会有一个单例实体， 但是*享元*类可以有多个实体， 各实体的内在状态也可以不同。 

  2. 单例对象可以是可变的。 享元对象是不可变的。
- [抽象工厂模式](https://refactoringguru.cn/design-patterns/abstract-factory)、 [生成器模式](https://refactoringguru.cn/design-patterns/builder)和[原型模式](https://refactoringguru.cn/design-patterns/prototype)都可以用[单例](https://refactoringguru.cn/design-patterns/singleton)来实现。

### 代码示例

在Java中使用

复杂度：1

流行度：3

Java 核心程序库中仍有相当多的单例示例：

- [`java.lang.Runtime#getRuntime()`](http://docs.oracle.com/javase/8/docs/api/java/lang/Runtime.html#getRuntime--)
- [`java.lang.System#getSecurityManager()`](http://docs.oracle.com/javase/8/docs/api/java/lang/System.html#getSecurityManager--)

看下面的写法

## 2、四种写法

简单点说，就是一个应用程序中，某个类的实例对象只有一个，你没有办法去new，因为构造器是被private修饰的，一般通过getInstance()的方法来获取它们的实例。

getInstance()的返回值是一个对象的引用，并不是一个新的实例，所以不要错误的理解成多个对象。单例模式实现起来也很容易，直接看demo吧

```java
public class Singleton {
  private static Singleton singleton;
  
  private Singleton() { 
  }
  
  public static Singleton getInstance() {
    if(singleton == null) {
      singleton = new Singleton();
    }
    return singleton;
  }
}
```

按照我的习惯，我恨不得写满注释，怕你们看不懂，但是这个代码实在太简单了，所以我没写任何注释，如果这几行代码你都看不明白的话，那你可以洗洗睡了，等你睡醒了再来看我的博客说不定能看懂。

上面的是最基本的写法，也叫懒汉写法（线程不安全）下面我再公布几种单例模式的写法：

### 懒汉式写法（线程安全）-加同步锁

```java
public class Singleton {
  private static Singleton singleton;
  
  private Singleton() {}
  
  public static synchronized Singleton getInstance() {
    if(singleton == null) {
      singleton = new Singleton();
    }
    return singleton;
  }
}
```

### 饿汉式写法

```java
public class Singleton {
  private static Singleton instance = new Singleton();
  private Singleton() {}
  
  public static Singleton getInstance() {
    return instance;
  }
}
```

### 静态内部类

```java
public class Singleton {
  private static class SingletonHolder {
    private static final Singleton instance = new Singleton();
  }
  private Singleton() {}
  public static final Singleton getInstance() {
    return SingletonHolder.instance;
  }
}
```

### 双重校验锁DCL

```java
public class Singleton {
  private volatile static Singleton singleton;
  private Singleton() {}
  public static Singleton getInstance() {
    if (singleton == null) { // 减少竞争锁的线程
      synchronized (Singleton.class) {
        if (singleton == null) {
          singleton = new Singleton();
        }
      }
    }
    return singleton;
  }
}
```

我个人比较喜欢静态内部类写法和饿汉式写法，其实这两种写法能够应付绝大多数情况了。其他写法也可以选择，主要还是看业务需求吧。

> 本文为转载文章，来源方志朋的公众号