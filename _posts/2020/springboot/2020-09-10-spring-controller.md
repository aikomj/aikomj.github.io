---
layout: post
title: Spring的Controller是单例还是多例？怎么保证并发的安全
category: spring
tags: [spring]
keywords: spring
excerpt: Spring的Controller默认是单例的，线程不安全的，尽量不使用成员变量，ThreadLocal成员变量，Request请求会复用线程池里线程，所以也不能达到请求并发安全，使用ConcurrentHashMap等并发安全类，微服务的并发安全考虑使用可共享信息的分布式缓存中间件如Redis
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

### Scope注解多例模式

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

  缺点：增加了创建销毁bean的服务器资源开销

- 在Controller中使用ThreadLocal变量（但ThreadLocal 使用不好会容易导致内存泄漏，见下篇文章）

> spring bean作用域

- singleton: 单例模式，当spring创建applicationContext容器的时候，spring会初始化所有的该作用域实例，加上lazy-init就可以避免预处理；

- prototype： 原型模式，每次通过getBean获取该bean就会新产生一个实例，创建后spring将不再对其管理；

- request： 每次请求都新产生一个实例，和prototype不同就是创建后，接下来的管理，spring依然在监听；

- session: 每次会话，同上；

- global session: 全局的web域，类似于servlet中的application。

### 线程隔离类ThreadLocal

```java
@Controller
public class HomeController {
    private ThreadLocal<Integer> i = new ThreadLocal<>();
    @GetMapping("testsingleton1")
    @ResponseBody
    public int test1() {
        if (i.get() == null) {
            i.set(0);
        }
        i.set(i.get().intValue() + 1);
        log.info("{} -> {}", Thread.currentThread().getName(), i.get());
        return i.get().intValue();
    }
}
```

多次访问url测试，打印日志如下：

```sh
[INFO ] 2019-12-03 11:49:08,226 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-1 -> 1
[INFO ] 2019-12-03 11:49:16,457 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-2 -> 1
[INFO ] 2019-12-03 11:49:17,858 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-3 -> 1
[INFO ] 2019-12-03 11:49:18,461 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-4 -> 1
[INFO ] 2019-12-03 11:49:18,974 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-5 -> 1
[INFO ] 2019-12-03 11:49:19,696 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-6 -> 1
[INFO ] 2019-12-03 11:49:22,138 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-7 -> 1
[INFO ] 2019-12-03 11:49:22,869 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-9 -> 1
[INFO ] 2019-12-03 11:49:23,617 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-8 -> 1
[INFO ] 2019-12-03 11:49:24,569 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-10 -> 1
[INFO ] 2019-12-03 11:49:25,218 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-1 -> 2
[INFO ] 2019-12-03 11:49:25,740 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-2 -> 2
[INFO ] 2019-12-03 11:49:43,308 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-3 -> 2
[INFO ] 2019-12-03 11:49:44,420 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-4 -> 2
[INFO ] 2019-12-03 11:49:45,271 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-5 -> 2
[INFO ] 2019-12-03 11:49:45,808 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-6 -> 2
[INFO ] 2019-12-03 11:49:46,272 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-7 -> 2
[INFO ] 2019-12-03 11:49:46,489 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-9 -> 2
[INFO ] 2019-12-03 11:49:46,660 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-8 -> 2
[INFO ] 2019-12-03 11:49:46,820 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-10 -> 2
[INFO ] 2019-12-03 11:49:46,990 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-1 -> 3
[INFO ] 2019-12-03 11:49:47,163 com.cjia.ds.controller.HomeController.test1(HomeController.java:50)
http-nio-8080-exec-2 -> 3
```

从日志分析出，二十多次的连续请求得到的结果有1有2有3等等，而我们期望不管我并发请求有多少，每次的结果都是1；同时可以发现web服务器默认的请求线程池大小为10，<mark>这10个核心线程可以被之后不同的Http请求复用</mark>，所以这也是为什么相同线程名的结果不会重复的原因。

总结：

**ThreadLocal的方式可以达到线程隔离，但还是无法达到并发安全**

### 使用并发安全的类

如果非要在单例bean中使用成员变量，可以考虑使用并发安全的容器，如ConcurrentHashMap、ConcurrentHashSet、CopyOnWriteArrayList等，将我们的成员变量（一般可以是当前运行中的任务列表等这类变量）包装到这些并发安全的容器中进行管理即可。

### 分布式或微服务的并发安全

分布式，一个服务会有多个服务实例，就要考虑全局的并发安全，就要借助于可以共享某些信息的分布式缓存中间件如Redis等，这样即可保证同一服务的所有服务实例都拥有同一份共享信息（如当前运行中的任务列表等这类变量）