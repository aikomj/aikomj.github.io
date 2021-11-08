---
layout: post
title: Spring AOP 切面编程的思想，简化开发
category: spring
tags: [spring]
keywords: spring
excerpt: AOP 切面编程，简化开发，无侵入业务层
lock: noneed
---

spring简化开发的4大核心思想：

1. Spring Bean，生命周期由spring 容器管理的ava对象
2. IOC，控制反转的思想，所有的对象都去Spring容器getbean
3. AOP，切面编程降低侵入。
4. xxxTemplate模版技术，如RestTemplate,RedisTemplate  

## 1、什么是AOP

AOP是Spring框架面向切面的编程思想，它将涉及多业务流程的**通用功能抽取**并单独封装，形成独立的切面，在合适的时机将这些切面横向切入到业务流程指定的位置中，从而让业务层只关注业务本身，降低程序的耦合度，是函数式编程的一种衍生。

在业务系统里除了实现业务功能之外，还要实现如权限拦截、性能监控、事务管理等非业务功能。通常的作法是把非业务代码穿插在业务代码中，从而导致业务功能与非业务功能的耦合。AOP 切面编程思想，就是解决这个耦合而诞生的，把非业务代码通过横向切割的方式抽取到一个独立的模块中。

通用功能：非业务逻辑功能，如日志记录（renren-fast有实现），权限验证（renren-fast有实现），事务处理，异常处理等

切面:相当于应用对象间的横切点，我们可以将其抽象为单独的模块

![](\assets\images\2020\java\spring-aop.png)

## 2、AOP术语

AOP 领域中的特性术语：

- 切面（Aspect）: 切面是通知和切点的结合。
- 切点（PointCut）: 可以插入增强处理的连接点。
- 通知（Advice）: AOP 框架中的增强处理。
- 连接点（join point）: 连接点表示应用执行过程中能够插入切面的一个点，这个点可以是方法的调用、异常的抛出。在 Spring AOP 中，连接点总是方法的调用。
- 引入（Introduction）：引入允许我们向现有的类添加新的方法或者属性。
- 织入（Weaving）: 将增强处理添加到目标对象中，并创建一个被增强的对象，这个过程就是织入

### 切面

定义一个切面类

```java
@Component
@Aspect
public class LogAspect {
    private static final Logger logger = LoggerFactory.getLogger(LogAspect.class);
    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();
}
```

### 切点

加入切点的声明

```java
@Component
@Aspect
public class LogAspect {
	private static final Logger logger = LoggerFactory.getLogger(LogAspect.class);
  private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();
  
  // 切点表达式
	@Pointcut("execution(public * com.xhx.springboot.controller.*.*(..))")
  public void pointCut(){}
}
```

![](/assets/images/2020/java/spring-aop-1.png)

> execution(方法修饰符 返回类型 方法全限定名(参数))  

**说明：**上面代码中的切点表达式,

1、..两个点表明多个，*代表一个

2、上面表达式代表切入com.xhx.springboot.controller包下的所有类的所有方法，方法参数不限，返回类型不限。  其中访问修饰符可以不写，不能用\*，

3、第一个\*代表返回类型不限，第二个\*表示所有类，第三个*表示所有方法，..两个点表示方法里的参数不限。 

4、然后用@Pointcut切点注解，放在一个空方法上面，一会儿在Advice通知中，直接调用这个空方法就行了。 

```java
// 匹配所有get开头的，第一个参数是Long类型的方法
@Pointcut("execution(* *..get*(Long,..))")
```

> within(类路径)  用来限定类

```java
// 匹配com.xhx.springboot包及其子包下的所有类方法
@Pointcut("within(com.xhx.springboot..*)")

// @within(annotationType) 匹配带有指定注解的类
// 下面匹配含有 @Component注解的类
@Pointcut("@within(org.springframework.stereotype.Component)")
```

> @annotation(annotationType) 匹配带有指定注解的方法

```java
// 限定在com.xx.yy.controller包下的所有类方法，且方法要带有注解@OperationLog
@Pointcut("within(com.xx.yy.controller..*) && @annotation(com.api.annotation.OperationLog)")
public void logAspect() {}
```

> @args(annotationType) 匹配使用指定注解标注的类作为参数的方法

**注意：**

可以使用`&&和||和!`三种运算符来组合切点表达式，表示与或非的关系。



### 通知增强

主要包括五个注解：

- @Before 在切点方法之前执行
- @After 在切点方法之后执行
- @AfterReturning 切点方法返回后执行
- @AfterThrowing 切点方法抛异常执行
- @Around 属于环绕增强，能控制切点执行前，执行后，用这个注解后，程序抛异常，会影响@AfterThrowing这个注解

## 3、AOP例子

```java
package com.xhx.springboot.config;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.reflect.SourceLocation;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;
import java.util.Arrays;

@Component
@Aspect
public class LogAspect {

  private static final Logger logger = LoggerFactory.getLogger(LogAspect.class);
  private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

  @Pointcut("execution(public * com.xhx.springboot.controller.*.*(..))")
  public void pointCut(){}

  // 1、在切点方法之前执行
  @Before(value = "pointCut()")
  public void before(JoinPoint joinPoint){
    logger.info("@Before通知执行");
    //获取目标方法参数信息
    Object[] args = joinPoint.getArgs();
    Arrays.stream(args).forEach(arg->{ 
      try {
        logger.info(OBJECT_MAPPER.writeValueAsString(arg));
      } catch (JsonProcessingException e) {
        logger.info(arg.toString());
      }
    });

    //aop代理对象
    Object aThis = joinPoint.getThis();
    logger.info(aThis.toString()); //com.xhx.springboot.controller.HelloController@69fbbcdd

    //被代理对象
    Object target = joinPoint.getTarget();
    logger.info(target.toString()); //com.xhx.springboot.controller.HelloController@69fbbcdd

    //获取连接点的方法签名对象
    Signature signature = joinPoint.getSignature();
    logger.info(signature.toLongString()); //public java.lang.String com.xhx.springboot.controller.HelloController.getName(java.lang.String)
    logger.info(signature.toShortString()); //HelloController.getName(..)
    logger.info(signature.toString()); //String com.xhx.springboot.controller.HelloController.getName(String)
    //获取方法名
    logger.info(signature.getName()); //getName
    //获取声明类型名
    logger.info(signature.getDeclaringTypeName()); //com.xhx.springboot.controller.HelloController
    //获取声明类型  方法所在类的class对象
    logger.info(signature.getDeclaringType().toString()); //class com.xhx.springboot.controller.HelloController
    //和getDeclaringTypeName()一样
    logger.info(signature.getDeclaringType().getName());//com.xhx.springboot.controller.HelloController

    //连接点类型
    String kind = joinPoint.getKind();
    logger.info(kind);//method-execution

    //返回连接点方法所在类文件中的位置  打印报异常
    SourceLocation sourceLocation = joinPoint.getSourceLocation();
    logger.info(sourceLocation.toString());
    //logger.info(sourceLocation.getFileName());
    //logger.info(sourceLocation.getLine()+"");
    //logger.info(sourceLocation.getWithinType().toString()); //class com.xhx.springboot.controller.HelloController

    ///返回连接点静态部分
    JoinPoint.StaticPart staticPart = joinPoint.getStaticPart();
    logger.info(staticPart.toLongString());  //execution(public java.lang.String com.xhx.springboot.controller.HelloController.getName(java.lang.String))

    //attributes可以获取request信息 session信息等
    ServletRequestAttributes attributes =
      (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
    HttpServletRequest request = attributes.getRequest();
    logger.info(request.getRequestURL().toString()); //当前请求的URL是http://127.0.0.1:8080/hello/getName
    logger.info(request.getRemoteAddr()); //127.0.0.1
    logger.info(request.getMethod()); //GET

    logger.info("@Before通知执行结束");
  }

  /**
     * 2.1 切点方法返回后执行
     * 如果第一个参数为JoinPoint，则第二个参数为返回值的信息
     * 如果第一个参数不为JoinPoint，则第一个参数为returning中对应的参数
     * returning：限定了只有目标方法返回值与通知方法参数类型匹配时才能执行后置返回通知，否则不执行，
     *            参数为Object类型将匹配任何目标返回值
     */
  @AfterReturning(value = POINT_CUT,returning = "result")
  public void doAfterReturningAdvice1(JoinPoint joinPoint,Object result){
    logger.info("@AfterReturning第一个后置返回通知的返回值："+result);
  }

  // 2.2 切点方法返回后执行
  @AfterReturning(value = POINT_CUT,returning = "result",argNames = "result")
  public void doAfterReturningAdvice2(String result){
    logger.info("@AfterReturning第二个后置返回通知的返回值："+result);
  }

  /**
     * 3、切点方法抛异常执行
     *  定义一个名字，该名字用于匹配通知实现方法的一个参数名，当目标方法抛出异常返回后，将把目标方法抛出的异常传给通知方法；
     *  throwing:限定了只有目标方法抛出的异常与通知方法相应参数异常类型时才能执行后置异常通知，否则不执行，
     *            对于throwing对应的通知方法参数为Throwable类型将匹配任何异常。
     */
  @AfterThrowing(value = POINT_CUT,throwing = "exception")
  public void doAfterThrowingAdvice(JoinPoint joinPoint,Throwable exception){
    logger.info(joinPoint.getSignature().getName());
    if(exception instanceof NullPointerException){
      logger.info("@AfterThrowing发生了空指针异常!!!!!");
    }
  }

  // 4、在切点方法之后执行
  @After(value = POINT_CUT)
  public void doAfterAdvice(JoinPoint joinPoint){
    logger.info("@After后置通知执行了!");
  }

  /**
     * 5、环绕通知：
     *   注意:Spring AOP的环绕通知会影响到AfterThrowing通知的运行,不要同时使用
     *   环绕通知非常强大，可以决定目标方法是否执行，什么时候执行，执行时是否需要替换方法参数，执行完毕是否需要替换返回值。
     *   环绕通知第一个参数必须是org.aspectj.lang.ProceedingJoinPoint类型
     */
  @Around(value = POINT_CUT)
  public Object doAroundAdvice(ProceedingJoinPoint proceedingJoinPoint){
    logger.info("@Around环绕通知："+proceedingJoinPoint.getSignature().toString());
    Object obj = null;
    try {
      obj = proceedingJoinPoint.proceed(); //可以加参数
      logger.info(obj.toString());
    } catch (Throwable throwable) {
      throwable.printStackTrace();
    }
    logger.info("@Around环绕通知执行结束");
    return obj;
  }
}
```

执行结果如下：

```java
@Around环绕通知
@Before通知执行
@Before通知执行结束
@Around环绕通知执行结束
@After后置通知执行了!
@AfterReturning第一个后置返回通知的返回值：18
```

**参数对象**

- JoinPoint joinPoint：连接点对象，它可以获取当前切入的方法的参数、代理类等信息

  ```java
  //aop代理对象
  Object aThis = joinPoint.getThis();
  logger.info(aThis.toString()); //com.xhx.springboot.controller.HelloController@69fbbcdd
  
  //被代理对象
  Object target = joinPoint.getTarget();
  logger.info(target.toString()); //com.xhx.springboot.controller.HelloController@69fbbcdd
  ```

  this表示当前切入点表达式所指代的方法的对象的实例;

  target表示当前切入点表达式所指代的方法的目标对象的实例 ;

  生成代理对象时会有两种方法，一个是CGLIB一个是jdk动态代理。

  用下面三个例子进行说明：   

  - this(SomeInterface)或target(SomeInterface)：这种情况下，无论是对于Jdk代理还是Cglib代理，其目标对象和代理对象都是实现SomeInterface接口的（Cglib生成的目标对象的子类也是实现了SomeInterface接口的），因而this和target语义都是符合的，此时这两个表达式的效果一样；
  - this(SomeObject)或target(SomeObject)，这里SomeObject没实现任何接口：这种情况下，Spring会使用Cglib代理生成SomeObject的代理类对象，由于代理类是SomeObject的子类，子类的对象也是符合SomeObject类型的，因而this将会被匹配，而对于target，由于目标对象本身就是SomeObject类型，因而这两个表达式的效果一样；
  - this(SomeObject)或target(SomeObject)，这里SomeObject实现了某个接口：对于这种情况，虽然表达式中指定的是一种具体的对象类型，但由于其实现了某个接口，因而Spring默认会使用Jdk代理为其生成代理对象，Jdk代理生成的代理对象与目标对象实现的是同一个接口，但代理对象与目标对象还是不同的对象，由于代理对象不是SomeObject类型的，因而此时是不符合this语义的，而由于目标对象就是SomeObject类型，因而target语义是符合的，此时this和target的效果就产生了区别；这里如果强制Spring使用Cglib代理，因而生成的代理对象都是SomeObject子类的对象，其是SomeObject类型的，因而this和target的语义都符合，其效果就是一致的。

- ProceedingJoinPoint proceedingJoinPoint：JoinPoint的子类，多了两个方法

  ```java
  // 调用下一个advice或者执行目标方法，返回值为目标方法返回值，因此可以通过更改返回值来修改方法的返回值
  public Object proceed() throws Throwable;
  
  // 参数为目标方法的参数  因此可以通过修改参数改变方法入参
  public Object proceed(Object[] args) throws Throwable;
  ```

> 参考代码

rcc 的SecurityUtils工具类使用了线程变量

```java
@Component
public class SecurityUtils {
  // 对比ThreadLocal，InheritableThreadLocal的可继承线程变量
    private static ThreadLocal<BaseRequest> THREAD_BASE_REQUEST = new InheritableThreadLocal<BaseRequest>() {
        protected BaseRequest initialValue() {
            return new BaseRequest();
        }
    };

    private SecurityUtils() {
    }

    public static void setBaseRequest(BaseRequest baseRequest) {
        THREAD_BASE_REQUEST.set(baseRequest);
    }

    public static BaseRequest getBaseRequest() {
        return (BaseRequest)THREAD_BASE_REQUEST.get();
    }

    public static void clear() {
        THREAD_BASE_REQUEST.remove();
    }

    public static String getOriginSystem() {
        return ((BaseRequest)THREAD_BASE_REQUEST.get()).getOriginSystem();
    }

    public static String getUserKey() {
        return ((BaseRequest)THREAD_BASE_REQUEST.get()).getUserKey();
    }

    public static String getIpAddr() {
        return ((BaseRequest)THREAD_BASE_REQUEST.get()).getIpAddr();
    }

    public static String getLanguage() {
        return ((BaseRequest)THREAD_BASE_REQUEST.get()).getLanguage();
    }

    public static Integer getTimeZone() {
        return ((BaseRequest)THREAD_BASE_REQUEST.get()).getTimeZone();
    }

    public static String getUserCode() {
        return ((BaseRequest)THREAD_BASE_REQUEST.get()).getUserCode();
    }

    public static String getUserName() {
        return ((BaseRequest)THREAD_BASE_REQUEST.get()).getUserName();
    }

    public static String getWorkIp() {
        return ((BaseRequest)THREAD_BASE_REQUEST.get()).getWorkIp();
    }
}
```

在AOP中给每个请求的线程变量重新赋值

```java
@Aspect
public class RpcRequestInterceptor {
    //作为中间变量，用于保存用户信息
    private ThreadLocal<BaseRequest> threadLocal = new ThreadLocal<BaseRequest>();
    //计数器，记录进入facade层数，用于退出相应层级时清空threadLocal里面值
    private ThreadLocal<Integer> counter = new ThreadLocal<Integer>();
  
   //对接口插入切面，否则在rpc层调用其它rpc模块时不能先从threadLocal中获取已塞进去的用户信息拿出来
    @Around("execution(* com.midea.ccs..*.facade..*.*(..))")
    public Object aroundFacadeImplMethod(ProceedingJoinPoint pjp) throws Throwable {
    	long start = System.currentTimeMillis();
        try {        	
            increase();
            logger.debug("进了RPC端的request interceptor(" + counter.get() + "次):" + pjp.getSignature().toString());

            Object[] args = pjp.getArgs();

            if (args != null && args.length > 0) {
                if (args[0] != null && args[0] instanceof BaseRequest) { // 左边第一个
                    BaseRequest arg = (BaseRequest) args[0];

                    if (threadLocal.get() == null) { // case1: Web invoke RPC
                        UserInfo userInfo = new UserInfo();
                        BeanUtils.copyProperties(arg, userInfo);
                        threadLocal.set(userInfo);
                    } else { // case2: RPC invoke RPC
                        BaseRequest source = threadLocal.get();
                        List<String> ignores = buildIgnoreList(source,arg);
                        String[] ignoreArray = ignores.toArray(new String[ignores.size()]);
                        BeanUtils.copyProperties(source, arg, ignoreArray);
                    }
                }
            }
            return pjp.proceed(args);
            
        } catch (Exception e) {
            logger.error("RPC调用时遇到异常:" + e.getMessage(), e);
            throw e;
        } finally {
            Integer cnt = minus();
            if (cnt <= 0) {
                threadLocal.remove();
                counter.remove();
            }
            
            try {
            	long end = System.currentTimeMillis();
                logger.debug("RPC invoke time " + (end -start) + " ms (" + pjp.getTarget().getClass().getName() + "." + pjp.getSignature().getName() +")");
            } catch (Exception ex) {            	
            }
        }
    }
}
```



参考：

[Spring AOP切点表达式用法总结](https://www.cnblogs.com/zhangxufeng/p/9160869.html)

[https://www.cnblogs.com/suphowe/p/12098042.html](https://www.cnblogs.com/suphowe/p/12098042.html)

