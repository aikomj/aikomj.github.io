---
layout: post
title: Spring AOP 切面编程的思想，简化开发
category: java
tags: [java]
keywords: java
excerpt: AOP 切面编程，简化开发，无侵入业务层
lock: noneed
---

spring简化开发的4大核心思想：

1. Spring Bean，生命周期由spring 容器管理的ava对象
2. IOC，控制反转的思想，所有的对象都去Spring容器getbean
3. AOP，切面编程降低侵入。
4. xxxTemplate模版技术，如RestTemplate,RedisTemplate  

AOP是Spring框架面向切面的编程思想，它将涉及多业务流程的**通用功能抽取**并单独封装，形成独立的切面，在合适的时机将这些切面横向切入到业务流程指定的位置中，从而让业务层只关注业务本身，降低程序的耦合度，是函数式编程的一种衍生。

通用功能：非业务逻辑功能，如日志记录（renren-fast有实现），权限验证（renren-fast有实现），事务处理，异常处理等

![](\assets\images\2020\java\spring-aop.png)

AOP 领域中的特性术语：

- 切面（Aspect）: 切面是通知和切点的结合。
- 切点（PointCut）: 可以插入增强处理的连接点。
- 通知（Advice）: AOP 框架中的增强处理。
- 连接点（join point）: 连接点表示应用执行过程中能够插入切面的一个点，这个点可以是方法的调用、异常的抛出。在 Spring AOP 中，连接点总是方法的调用。
- 引入（Introduction）：引入允许我们向现有的类添加新的方法或者属性。
- 织入（Weaving）: 将增强处理添加到目标对象中，并创建一个被增强的对象，这个过程就是织入

> 切面

定义一个切面类

```java
@Component
@Aspect
public class LogAspect {
    private static final Logger logger = LoggerFactory.getLogger(LogAspect.class);
    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();
}
```

> 切点

加入切点的声明

```java
@Component
@Aspect
public class LogAspect {
	private static final Logger logger = LoggerFactory.getLogger(LogAspect.class);
  private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();
  
	@Pointcut("execution(public * com.xhx.springboot.controller.*.*(..))")
  public void pointCut(){}
}
```

![](/assets/images/2020/java/spring-aop-1.png)

**说明：**上面代码中的切点表达式中

1、..两个点表明多个，*代表一个

2、上面表达式代表切入com.xhx.springboot.controller包下的所有类的所有方法，方法参数不限，返回类型不限。  其中访问修饰符可以不写，不能用\*，

3、第一个\*代表返回类型不限，第二个\*表示所有类，第三个*表示所有方法，..两个点表示方法里的参数不限。 

4、然后用@Pointcut切点注解，放在一个空方法上面，一会儿在Advice通知中，直接调用这个空方法就行了。 



> 通知增强

主要包括五个注解：

- @Before 在切点方法之前执行

- @After 在切点方法之后执行

- @AfterReturning 切点方法返回后执行

- @AfterThrowing 切点方法抛异常执行

- @Around 属于环绕增强，能控制切点执行前，执行后，用这个注解后，程序抛异常，会影响@AfterThrowing这个注解

```java

```











https://recomm.cnblogs.com/blogpost/12851620

https://www.cnblogs.com/suphowe/p/12098042.html

