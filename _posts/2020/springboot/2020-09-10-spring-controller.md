---
layout: post
title: Spring的Controller是单例还是多例？怎么保证并发的安全
category: spring
tags: [spring]
keywords: spring
excerpt: Spring的Controller默认是单例的，线程不安全，不要使用非静态的成员变量，如何多例
lock: noneed
---

## 1、验证controller单例

```java
/**
 * @author riemann
 * @date 2019/07/29 22:56
 */
@Controller
public class ScopeTestController {
    private int num = 0;

    @RequestMapping("/testScope")
    public void testScope() {
        System.out.println(++num);
    }

    @RequestMapping("/testScope2")
    public void testScope2() {
        System.out.println(++num);
    }
}
```

我们首先访问 http://localhost:8080/testScope，得到的答案是1；

然后我们再访问 http://localhost:8080/testScope2，得到的答案是 2。

<strong style="color:rgb(171, 25, 66)">得到的不同的值，这是线程不安全的。</strong>

> @Scope("prototype")多例

```java
/**
 * @author riemann
 * @date 2019/07/29 22:56
 */
@Controller
@Scope("prototype")
public class ScopeTestController {
    private int num = 0;

    @RequestMapping("/testScope")
    public void testScope() {
        System.out.println(++num);
    }

    @RequestMapping("/testScope2")
    public void testScope2() {
        System.out.println(++num);
    }
}
```

我们依旧首先访问 http://localhost:8080/testScope，得到的答案是1；

然后我们再访问 http://localhost:8080/testScope2，得到的答案还是 1;

对于请求线程来说得到的初始值都是1，说明线程安全的。

**结论**

- 单例会导致属性重复使用

**解决方案**

- 不要在controller中定义成员变量。
- 万一必须要定义一个非静态成员变量时候，则通过注解@Scope(“prototype”)，将其设置为多例模式。
- 在Controller中使用ThreadLocal变量（但ThreadLocal 使用不好会容易导致内存泄漏，见下篇文章）

> spring bean作用域

- singleton: 单例模式，当spring创建applicationContext容器的时候，spring会初始化所有的该作用域实例，加上lazy-init就可以避免预处理；

- prototype： 原型模式，每次通过getBean获取该bean就会新产生一个实例，创建后spring将不再对其管理；

- request： 每次请求都新产生一个实例，和prototype不同就是创建后，接下来的管理，spring依然在监听；

- session: 每次会话，同上；

- global session: 全局的web域，类似于servlet中的application。

> 此文章为转载文章，转自方志鹏公众号