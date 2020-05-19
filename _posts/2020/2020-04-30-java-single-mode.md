---
layout: post
title: 23种设计模式之单例模式
category: java-design
tags: [java-design]
keywords: java
excerpt: 懒汉式、饿汉式、静态内部类、枚举、双重校验锁
lock: noneed
---

<mark>单例模式</mark>

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

- **懒汉式写法（线程安全）-加悲观锁**

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

- 饿汉式写法

  ```java
  public class Singleton {
    private static Singleton instance = new Singleton();
    private Singleton() {}
    
    public static Singleton getInstance() {
      return instance;
    }
  }
  ```

- **静态内部类**

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

- 双重校验锁

  ```java
  public class Singleton {
    private volatile static Singleton singleton;
    private Singleton() {}
    public static Singleton getInstance() {
      if (singleton == null) {
        synchronized (Singleton.class) {
          if (singleton == null) {
            singleton = new Singleton();
          }
        }
      }
    }
  }
  ```

我个人比较喜欢静态内部类写法和饿汉式写法，其实这两种写法能够应付绝大多数情况了。其他写法也可以选择，主要还是看业务需求吧。



> 本文为转载文章，来源方志朋的公众号