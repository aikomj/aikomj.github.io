---
layout: post
title: Spring源码第1节：Spring核心源码1
category: icoding-allen
tags: [icoding-allen]
keywords: spring
excerpt: 源码学习方法论，spring的核心注解，核心接口，
lock: noneed
---

## 1、前言

> 经验源码

基于**SpringBoot**，讲解**Spring**、**SpringMVC**、SpringBoot相关核心源码，讲解**MyBatis**核心源码，讲解**Tomcat**源码。

为什么？

Spring是源码教科书，大家必须懂。

Tomcat是架构经典，大家必须借鉴。

> 学习方法

流程图（时序图、类图）+ 文字总结 + 核心断点

源码永远讲不完，希望大家从源码分析课程，总结出一套适合自己的**源码方法论**，以及源码设计思想

纸上得来终觉浅、觉知此事要躬行。



## 2、核心注解

| 注解             | 作用                                                         | 备注                         |
| ---------------- | ------------------------------------------------------------ | ---------------------------- |
| @Bean            | 给spring容器中注册组件                                       |                              |
| @Primary         | 同类组件如果有多个，标注主组件(优先使用)                     |                              |
| @DependsOn       | 组件之间声明依赖关系                                         |                              |
| @Lazy            | 组件懒加载（最后使用的时候才创建组件）                       |                              |
| @Scope           | 声明组件的作用范围(SCOPE_PROTOTYPE,SCOPE_SINGLETON,)         |                              |
| @Configuration   | 声明这是一个配置类，替换以前的beans.xml                      | 理解proxyBeanMethods         |
| @Controller      |                                                              |                              |
| @Service         |                                                              |                              |
| @Repository      |                                                              |                              |
| @Component       |                                                              |                              |
| @Indexed         | 加速注解，所有标注了@Indexed的组件，直接会启动快速加载       |                              |
| @Order           | 数字越小优先级越高，越先工作                                 |                              |
| @ComponentScan   | 包扫描                                                       |                              |
| @ComponentScans  | 包扫描                                                       |                              |
| @Conditional     | 条件注入   1,2  @ConditionalOnMissingBean                    | 自定义                       |
| @Import          | 导入第三方jar包中的组件                                      |                              |
| @ImportResource  | 导入以前的xml配置文件，让其生效                              |                              |
| @Profile         | 基于多环境激活                                               |                              |
| @PropertySource  | 外部properties配置文件和JavaBean进行绑定.结合ConfigurationProperties |                              |
| @PropertySources |                                                              |                              |
| @Autowired       | 自动装配                                                     |                              |
| @Qualifier       | 精确指定                                                     |                              |
| @Value           | 取值、计算机环境变量、JVM系统。xxxx                          | SpringBoot支持各种外部化配置 |

> @Configuration 配置类

 查看源码，要注意它的一个属性proxyBeanMethods

![](/aikomj.github.io/assets/images/2020/annotation/configuration.jpg)

```java
/**
 * 这是一个配置类---beans.xml
 *
 * proxyBeanMethods: 代理bean方法
 * 为了Spring快速加载:
 * Lite Mode：每次直接调用方法获取这个组件
 * Full Mode：调用方法获取组件的时候，会先判断容器中有没有这个组件，如果有就获取容器中的
 *
 * proxyBeanMethods = true： Tomcat的jackDog和容器中的jackDog一样； Full Mode：
 * proxyBeanMethods = false： Tomcat的jackDog和容器中的jackDog不一样
 *
 */
@Configuration(proxyBeanMethods = false)  //默认true，使用重模式
public class ICodingConfig {
  ....
}
```

- 轻模式Lite Mode
- 重模式Full Mode

**例子**

1、 定义两个类

```java
import lombok.Data;
@Data
public class JackDog {
}

@Data
public class Tomcat {
    //Tomcat 有组件依赖关系,依赖组件jackdog
    private JackDog friend;
    private String name;
}
```

2、配置类中创建bean，放入spring容器

```java
@Configuration(proxyBeanMethods = true)  //默认true
public class ICodingConfig {
  @Bean
  public Tomcat tomcat(){
    Tomcat tomcat = new Tomcat();
    tomcat.setFriend(jackDog());
    return tomcat;
  }

  @Bean
  public JackDog jackDog(){
    JackDog jackDog = new JackDog();
    System.out.println("容器中的jackDog"+jackDog);
    return jackDog;
  }
}
```

3、启动类

```java
@SpringBootApplication
public class SpringSourcesApplication {
  public static void main(String[] args) throws InterruptedException {
    //1、返回ioc容器
    ConfigurableApplicationContext run = SpringApplication.run(SpringSourcesApplication.class, args);

    Tomcat tomcat = run.getBean("tomcat", Tomcat.class);
    System.out.println(tomcat.getFriend());

    JackDog jackDog = run.getBean("jackDog", JackDog.class);
    System.out.println(jackDog == tomcat.getFriend());
  }
}
```

启动运行：

![](/assets/images/2020/annotation/configuration-2.jpg)

 把proxyBeanMethods改为false

```java
@Configuration(proxyBeanMethods = false)  //默认true
public class ICodingConfig {
...
```

再次启动运行

![](/assets/images/2020/annotation/configuration-3.jpg)

现在是false 也就是说tomcat里面的组件jackdog与 spring容器中的组件jackdog不是同一个对象，也就是说堆中存在了两个jackdog对象