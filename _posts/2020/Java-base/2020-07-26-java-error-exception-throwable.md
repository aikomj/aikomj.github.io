---
layout: post
title: Error 和 Exception,抽象类
category: java
tags: [java]
keywords: java
excerpt: error 一般指与JVM相关的问题如OOM就是error，exception是指程序中可以遇见性的异常
lock: noneed
---

## Throwable

Error 一般指与JVM相关的问题，如OOM就是error，如系统崩溃，JVM错误等，error无法恢复或不可能被捕获，将导致程序中断；

Exception 是指程序中可以遇见性的异常，如RuntimeException,IOException,SQLException等。Exception分为两类- 

- checked 检查性异常

  要求程序员必须注意该异常，要么声明抛出，要么try catch 处理，不能对该异常置之不理，否则就会在编译时发生错误，无法通过编译。checked异常体现了java的严谨性，增加了程序的健壮性。

- runtime 运行时异常

  不需要显示声明抛出，如果程序需要捕获Runtime异常，可以用try catch 实现。

## 抽象类

在面向对象的概念中，所有的对象都是通过类来描绘的。自己好少用抽象类(abtract class)，用接口多，现在算是明白抽象类的使用场景了，抽象类里的抽象方法与接口里的方法一样是没有方法体的，它的具体实现由它的子类确定，跟普通类一样，抽象类也拥有成员变量、构造方法、成员方法，但是它不能直接实例化对象，必须通过子类继承抽象类，实例化子类，那么抽象类里的成员变量、成员方法被子类继承了，才可以被调用。Java里类是单继承的extend，但是可以实现implement多个接口 (interface)

```java
abstract class Employee {
   private String name;
  
   public Employee(String name){
        System.out.println("Constructing an Employee");
        this.name = name;
    }
}

public class Testab {
    private Employee employee;

    public void setName(){
        this.employee = new Employee("jacob");
    }

    public String getName(){
        return wrapper.getName();
    }

    public static void main(String[] args) {
        Testab testab = new Testab();
        testab.setName();
        System.out.println(testab.getName());
    }
}
```

编译报错，抽象类不能直接被实例化

![](\assets\images\2020\java\abstract-class.jpg)

有两种方式

- 显示创建一个子类继承Employee，实例化子类对象

  ```java
  public class Salary extends Employee{
    ...
  }
  ```

- 给抽象类一个抽象方法，隐藏式创建子类，实现抽象方法

  ```java
  abstract class Employee {
     private String name;
    
     public Employee(String name){
          System.out.println("Constructing an Employee");
          this.name = name;
      }
    
    protected abstract String getStatementId(String var1);
  }
  
  public class Testab {
      private Employee employee;
  
    // 创建子类，并实例化
     this.employee = new Employee("jacob"){
              @Override
              protected String getStatementId(String var1) {
                  return "123";
              }
      };
  
      public String getName(){
          return wrapper.getName();
      }
  
      public static void main(String[] args) {
          Testab testab = new Testab();
          testab.setName();
          System.out.println(testab.getName());
      }
  }
  ```

  执行结果：

  ![](\assets\images\2020\java\abstract-class-2.jpg)

  使用场景：

  抽象类包含了子类集合的常见共用方法，如果需要某个特别方法由子类具体实现，可以定义为抽象方法。