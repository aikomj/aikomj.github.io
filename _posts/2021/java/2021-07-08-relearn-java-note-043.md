---
layout: post
title: 重学Java第43讲：throw 和 throws
category: java
tags: [java]
keywords: java
excerpt: throw 主动抛出异常，throws 关键字写在方法上声明可能产生的异常，让调用方法通过try catch进行捕捉处理
lock: noneed
---

转载自沉默王二的《Java程序员进阶之路》

码云地址: [https://gitee.com/itwanger/toBeBetterJavaer](https://gitee.com/itwanger/toBeBetterJavaer)

GitHub 地址：[https://github.com/itwanger/toBeBetterJavaer](https://github.com/itwanger/toBeBetterJavaer])

进入正题。

## 1、throw和throws的区别

> throw

throw 关键字，用于主动地抛出异常；正常情况下，当除数为 0 的时候，程序会主动抛出 `ArithmeticException`；但如果我们想要除数为 1 的时候也抛出` ArithmeticException`，就可以使用 throw 关键字主动地抛出异常。

```java
throw new exception_class("error message");
```

语法也非常简单，throw 关键字后跟上 new 关键字，以及异常的类型还有参数即可

```java
public class ThrowDemo {
    static void checkEligibilty(int stuage){
        if(stuage<18) {
            throw new ArithmeticException("年纪未满 18 岁，禁止观影");
        } else {
            System.out.println("请认真观影!!");
        }
    }

    public static void main(String args[]){
        checkEligibilty(10);
        System.out.println("愉快地周末..");
    }
}
```

执行结果：

![](\assets\images\2021\javabase\throw-a-exception.jpg)

> throws

throws 关键字的作用就和 throw 完全不同，这里就涉及检查型异常和非检查型异常两个知识点，

- 检查性异常程序在编译阶段，编译器会提示你，你必须处理，否则无法通过编译，处理方式就是要么在方法上使用throws显示声明异常，要么代码放try-catch中

  `Class.forName()` 方法在执行的时候可能会遇到 `java.lang.ClassNotFoundException`异常，一个检查型异常

  ![](\assets\images\2021\javabase\throws-signature.png)

- 非检查型异常，也就是运行时异常RuntimeException，程序在运行过程产生的异常，通常使用try-catch捕捉异常

这里扩展异常处理机制的知识，Error和Exception的区别，他们的共同父类是Throwable

- Error通常与JVM相关，如OOM导致系统崩溃，会导致程序中断，是无法捕获的
- Exception可遇见性的异常，如RuntimeException,IOException,SQLException等，分上面说的检查型异常和非检查型异常两类。

![](\assets\images\2021\javabase\throwable.jpg)

**那什么情况下使用 throws 而不是 try-catch 呢？**

假设现在有这么一个方法 `myMethod()`，可能会出现 `ArithmeticException` 异常，也可能会出现 `NullPointerException`。这种情况下，可以使用 try-catch 来处理

```java
public void myMethod() {
    try {
        // 可能抛出异常 
    } catch (ArithmeticException e) {
        // 算术异常
    } catch (NullPointerException e) {
        // 空指针异常
    }
}
```

如果为每个方法都加上 try-catch，就会显得非常繁琐。代码就会变得又臭又长，可读性就差了

一个解决办法就是，使用 throws 关键字，在方法签名上声明可能会抛出的异常，然后在调用该方法的地方使用 try-catch 进行处理

```java
public static void main(String args[]){
    try {
        myMethod1();
    } catch (ArithmeticException e) {
        // 算术异常
    } catch (NullPointerException e) {
        // 空指针异常
    }
}
public static void myMethod1() throws ArithmeticException, NullPointerException{
    // 方法签名上声明异常
}
```

> 总结

1）throws 关键字用于声明异常，它的作用和 try-catch 相似；而 throw 关键字用于显式的抛出异常。

2）throws 关键字后面跟的是异常的名字；而 throw 关键字后面跟的是异常的对象。

示例

```java
throws ArithmeticException;

throw new ArithmeticException("算术异常");
```

3）throws 关键字出现在方法签名上，而 throw 关键字出现在方法体里。

4）throws 关键字在声明异常的时候可以跟多个，用逗号隔开；而 throw 关键字每次只能抛出一个异常。