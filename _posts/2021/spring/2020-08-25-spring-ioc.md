---
layout: post
title: Spring IOC 理论推导
category: spring
tags: [spring]
keywords: spring
excerpt: spring框架的7大模块了解，IOC的原型由我们自行控制对象的创建，控制反转的本质是获得对象的方式反转了，将对象的创建转移给第三方
lock: noneed
---

<mark>Spring是一个轻量级的控制反转(IoC)和面向切面(AOP)的容器（框架）</mark>

## 1、Spring组成

![img](/assets/images/2020/spring/640.png)

Spring 框架是一个分层架构，由 7 个定义良好的模块组成。Spring 模块构建在核心容器之上，核心容器定义了创建、配置和管理 bean 的方式 ，如下图

![img](/assets/images/2020/spring/640-seven.png)

组成 Spring 框架的每个模块（或组件）都可以单独存在，或者与其他一个或多个模块联合实现。每个模块的功能如下：

- **核心容器**：核心容器提供 Spring 框架的基本功能。核心容器的主要组件是 BeanFactory，它是工厂模式的实现。BeanFactory 使用*控制反转*（IOC） 模式将应用程序的配置和依赖性规范与实际的应用程序代码分开。
- **Spring 上下文**：Spring 上下文是一个配置文件，向 Spring 框架提供上下文信息。Spring 上下文包括企业服务，例如 JNDI、EJB、电子邮件、国际化、校验和调度功能。
- **Spring AOP**：通过配置管理特性，Spring AOP 模块直接将面向切面的编程功能 , 集成到了 Spring 框架中。所以，可以很容易地使 Spring 框架管理任何支持 AOP的对象。Spring AOP 模块为基于 Spring 的应用程序中的对象提供了事务管理服务。通过使用 Spring AOP，不用依赖组件，就可以将声明性事务管理集成到应用程序中。
- **Spring DAO**：JDBC DAO 抽象层提供了有意义的异常层次结构，可用该结构来管理异常处理和不同数据库供应商抛出的错误消息。异常层次结构简化了错误处理，并且极大地降低了需要编写的异常代码数量（例如打开和关闭连接）。Spring DAO 的面向 JDBC 的异常遵从通用的 DAO 异常层次结构。
- **Spring ORM**：Spring 框架插入了若干个 ORM 框架，从而提供了 ORM 的对象关系工具，其中包括 JDO、Hibernate 和 iBatis SQL Map。所有这些都遵从 Spring 的通用事务和 DAO 异常层次结构。
- **Spring Web 模块**：Web 上下文模块建立在应用程序上下文模块之上，为基于 Web 的应用程序提供了上下文。所以，Spring 框架支持与 Jakarta Struts 的集成。Web 模块还简化了处理多部分请求以及将请求参数绑定到域对象的工作。
- **Spring MVC 框架**：MVC 框架是一个全功能的构建 Web 应用程序的 MVC 实现。通过策略接口，MVC 框架变成为高度可配置的，MVC 容纳了大量视图技术，其中包括 JSP、Velocity、Tiles、iText 和 POI。

## 2、SpringBoot与SpringCloud

- Spring Boot 是 Spring 的一套快速配置脚手架，可以基于Spring Boot 快速开发单个微服务;
- Spring Cloud是基于Spring Boot实现的；
- Spring Boot专注于快速、方便集成的单个微服务个体，Spring Cloud关注全局的服务治理框架；
- Spring Boot使用了约束优于配置的理念，很多集成方案已经帮你选择好了，能不配置就不配置 , Spring Cloud很大的一部分是基于Spring Boot来实现，Spring Boot可以离开Spring Cloud独立使用开发项目，但是Spring Cloud离不开Spring Boot，属于依赖的关系。
- SpringBoot在SpringClound中起到了承上启下的作用，如果你要学习SpringCloud必须要学习SpringBoot。

## 3、IOC理论推导

新建一个空白的maven项目

### 分析实现

我们先用我们原来的方式写一段代码 .

1、先写一个UserDao接口

```java
public interface UserDao {
   public void getUser();
}
```

2、再去写Dao的实现类

```java
public class UserDaoImpl implements UserDao {
   @Override
   public void getUser() {
       System.out.println("获取用户数据");
  }
}
```

3、然后去写UserService的接口

```java
public interface UserService {
   public void getUser();
}
```

4、最后写Service的实现类

```java
public class UserServiceImpl implements UserService {
   private UserDao userDao = new UserDaoImpl();

   @Override
   public void getUser() {
       userDao.getUser();
  }
}
```

5、测试一下

```java
@Test
public void test(){
   UserService service = new UserServiceImpl();
   service.getUser();
}
```

那我们现在修改一下 ,把Userdao的实现类增加一个 .

```java
public class UserDaoMySqlImpl implements UserDao {
   @Override
   public void getUser() {
       System.out.println("MySql获取用户数据");
  }
}
```

紧接着我们要去使用MySql的话 , 我们就需要去service实现类里面修改对应的实现

```java
public class UserServiceImpl implements UserService {
   private UserDao userDao = new UserDaoMySqlImpl();

   @Override
   public void getUser() {
       userDao.getUser();
  }
}
```

如果我们再增加一个UserDao的实现类

```java
public class UserDaoOracleImpl implements UserDao {
   @Override
   public void getUser() {
       System.out.println("Oracle获取用户数据");
  }
}
```

那么我们要使用Oracle , 又需要去service实现类里面修改对应的实现 . 假设我们的这种需求非常大 , 这种方式就根本不适用了, 甚至反人类对吧 , 每次变动 , 都需要修改大量代码 . 这种设计的耦合性太高了, 牵一发而动全身 .

**那我们如何去解决呢 ?** 

我们可以在需要用到他的地方 , 不去实现它 , 而是留出一个接口 , 利用set , 我们去代码里修改下 service实现类.

```java
public class UserServiceImpl implements UserService {
   private UserDao userDao;
// 利用set实现
   public void setUserDao(UserDao userDao) {
       this.userDao = userDao;
  }

   @Override
   public void getUser() {
       userDao.getUser();
  }
}
```

测试一下

```java
@Test
public void test(){
   UserServiceImpl service = new UserServiceImpl();
   service.setUserDao( new UserDaoMySqlImpl() );
   service.getUser();
   //那我们现在又想用Oracle去实现呢
   service.setUserDao( new UserDaoOracleImpl() );
   service.getUser();
}
```

大家发现了区别没有 ? 可能很多人说没啥区别 . 但是同学们 , 他们已经发生了根本性的变化 , 很多地方都不一样了 . 仔细去思考一下 , 以前所有东西都是由程序去进行控制创建 , 而现在是由我们自行控制创建对象 , 把主动权交给了调用者 . 程序不用去管怎么创建,怎么实现了 . 它只负责提供一个接口 .这也是oop编程里多态的思想.

这种思想 , 从本质上解决了问题 , 我们程序员不再去管理对象的创建了 , 更多的去关注业务的实现 . 耦合性大大降低 . 这也就是IOC的原型 

### IOC本质

**控制反转IoC(Inversion of Control)，是一种设计思想，DI(依赖注入)是实现IoC的一种方法**，也有人认为DI只是IoC的另一种说法。没有IoC的程序中 , 我们使用面向对象编程 , 对象的创建与对象间的依赖关系完全硬编码在程序中，对象的创建由程序自己控制，控制反转后将对象的创建转移给第三方，<mark>个人认为所谓控制反转就是：获得依赖对象的方式反转了。</mark>

![](/assets/images/2020/spring/640-ioc.png)

**IoC是Spring框架的核心内容**，使用多种方式完美的实现了IoC，可以使用XML配置，也可以使用注解，新版本的Spring也可以零配置实现IoC（我个人理解是使用springboot自动配置）

Spring容器在初始化时先读取配置文件，根据配置文件或元数据创建与组织对象存入容器中，程序使用时再从Ioc容器中取出需要的对象。

![](/assets/images/2020/spring/640-ioc-2.png)

采用XML方式配置Bean的时候，Bean的定义信息是和实现分离的，而采用注解的方式可以把两者合为一体，Bean的定义信息直接以注解的形式定义在实现类中，从而达到了零配置的目的。

**控制反转是一种通过描述（XML或注解）并通过第三方去生产或获取特定对象的方式。在Spring中实现控制反转的是IoC容器，其实现方法是依赖注入（Dependency Injection,DI）。**

IOC中文就是控制反转，依赖注入的意思，调用类对某个接口实现类的依赖调用由spring容器来实现，不再是调用类自己去实现，从而减少代码的耦合度。在没有引入ioc容器前，对象A依赖对象B，那么A对象在实例化运行到某一个点的时候，自己就必须主动创建对象B或者引用已经创建的对象B，控制权在我们自己手上。引入了IOC容器后，对象A实例化运行时，IOC容器会主动创建一个对象B注入到对象A所需要的地方。这时侯对象的生命周期交给了IOC容器处理，控制权颠倒了过来，这就是控制反转的由来。

## 4、IOC的主流程

### applicationContext接口

spring容器的顶层接口是：`BeanFactory`，但我们使用更多的是它的子接口：`ApplicationContext`。

通常情况下，如果我们想要手动初始化通过`xml文件`配置的spring容器时，代码是这样的：

```java
ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
User user = (User)applicationContext.getBean("name");
```

手动初始化通过`配置类`配置的spring容器时，

```java
AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(Config.class);
User user = (User)applicationContext.getBean("name");
```

这两个类应该是最常见的入口了，它们却殊途同归，最终都会调用`refresh`方法，该方法才是spring容器初始化的真正入口。如下图

![](\assets\images\2021\spring\application-context-refresh.png)

![](\assets\images\2021\spring\application-context-refresh-2.png)

其实调用`refresh`方法的类并非只有这两个，我们用一张图整体认识一下：

![](\assets\images\2021\spring\application-context-refresh-3.png)

虽说调用`refresh`方法的类有这么多，但我决定用`ClassPathXmlApplicationContext`类作为列子给大家讲解，因为它足够经典，而且难度相对来说要小一些。

### refresh方法

`refresh`方法是`spring ioc`的真正入口，它负责初始化spring容器。既然这个方法的作用是初始化spring容器，那方法名为啥不叫`init`？答案很简单，因为它不只被调用一次。

在`springboot`的`SpringAppication`类中的`run`方法会调用`refreshContext`方法，该方法会调用一次`refresh`方法。

在`springcloud`的`BootstrapApplicationListener`类中的`onApplicationEvent`方法会调用`SpringAppication`类中的`run`方法。也会调用一次`refresh`方法。

> 这是springboot项目中如果引入了springcloud，则refresh方法会被调用两次的原因。

在`springmvc`的`FrameworkServlet`类中的`initWebApplicationContext`方法会调用`configureAndRefreshWebApplicationContext`方法，该方法会调用一次`refresh`方法，不过会提前判断容器是否激活。

所以这里的`refresh`表示重新构建的意思。

下面重点看看`refresh`的关键步骤：

![](\assets\images\2021\spring\refresh-steps.png)

其实上图中一眼看过去好像有很多方法，但是真正的核心的方法不多，我主要讲其中最重要的：

- obtainFreshBeanFactory
- invokeBeanFactoryPostProcessors
- registerBeanPostProcessors
- finishBeanFactoryInitialization

### 解析xml配置文件

`obtainFreshBeanFactory`方法会解析xml的bean配置，生成`BeanDefinition`对象，并且注册到spring容器中（说白了就是很多map集合中）。

经过几层调用（细节不说，很简单），会调到`AbstractBeanDefinitionReader`类的`loadBeanDefinitions`方法：

![](\assets\images\2021\spring\application-context-refresh-4.png)

该方法会循环`locations`（applicationContext.xml文件路径）,调用另外一个`loadBeanDefinitions`方法，一个文件一个文件解析。

经过一些列的骚操作，会将location转换成inputSource和resource，然后再转换成Document对象，方面解析。

![](\assets\images\2021\spring\application-context-refresh-5.png)

在解析xml文件时，需要判断是默认标签，还是自定义标签，处理逻辑不一样：

![](\assets\images\2021\spring\application-context-refresh-6.png)

spring的默认标签只有4种：

- `<import/>`
- `<alias/>`
- `<bean/>`
- `<beans/>`

对应的处理方法是：

![](\assets\images\2021\spring\application-context-refresh-7.png)

注意常见的：`<aop/>`、`<context/>`、`<mvc/>`等都是自定义标签。

从上图中处理`<bean/>`标签的`processBeanDefinition`方法开始，经过一系列调用，最终会调到`DefaultBeanDefinitionDocumentReader`类的`processBeanDefinition`方法。

![](\assets\images\2021\spring\application-context-refresh-8.png)

这个方法包含了关键步骤：解析元素生成BeanDefinition 和 注册BeanDefinition。

### 生成BeanDefinition

下面重点看看BeanDefinition是如何生成的。

上面的方法会调用`BeanDefinitionParserDelegate`类的`parseBeanDefinitionElement`方法：

![](\assets\images\2021\spring\application-context-refresh-9.png)

一个`<bean/>`标签会对应一个`BeanDefinition`对象。

该方法又会调用同名的重载方法：`processBeanDefinition`，真正创建`BeanDefinition`对象，并且解析一系列参数填充到对象中：

![](\assets\images\2021\spring\application-context-refresh-10.png)

其实真正创建BeanDefinition的逻辑是非常简单的，直接new了一个对象：

![](\assets\images\2021\spring\application-context-refresh-11.png)

真正复杂的地方是在前面的各种属性的解析和赋值上。

### 注册BeanDefinition

上面通过解析xml文件生成了很多`BeanDefinition`对象，下面就需要把`BeanDefinition`对象注册到spring容器中，这样spring容器才能初始化bean。

在`BeanDefinitionReaderUtils`类的`registerBeanDefinition`方法很简单，只有两个流程：

![](\assets\images\2021\spring\application-context-refresh-12.png)

先看看`DefaultListableBeanFactory`类的`registerBeanDefinition`方法是如何注册`beanName`的

![](\assets\images\2021\spring\application-context-refresh-13.png)

接下来看看`SimpleAliasRegistry`类的`registerAlias`方法是如何注册`alias`别名的：

![](\assets\images\2021\spring\application-context-refresh-14.png)

这样就能通过多个不同的`alias`找到同一个`name`，再通过`name`就能找到`BeanDefinition`。

### 修改BeanDefinition

上面`BeanDefinition`对象已经注册到spring容器当中了，接下来，如果想要修改已经注册的`BeanDefinition`对象该怎么办呢？

`refresh`方法中通过`invokeBeanFactoryPostProcessors`方法修改`BeanDefinition`对象。

经过一系列的调用，最终会到`PostProcessorRegistrationDelegate`类的`invokeBeanFactoryPostProcessors`方法：

![](\assets\images\2021\spring\application-context-refresh-15.png)

流程看起来很长，其实逻辑比较简单，主要是在处理`BeanDefinitionRegistryPostProcessor`和`BeanFactoryPostProcessor`。

而`BeanDefinitionRegistryPostProcessor`本身是一种特殊的`BeanFactoryPostProcessor`，它也会执行`BeanFactoryPostProcessor`的逻辑，只是加了一个额外的方法。

![](\assets\images\2021\spring\application-context-refresh-16.png)

`ConfigurationClassPostProcessor`可能是最重要的`BeanDefinitionRegistryPostProcessor`，它负责处理`@Configuration`注解。

### 注册BeanPostProcessor

处理完前面的逻辑，`refresh`方法接着会调用`registerBeanPostProcessors`注册`BeanPostProcessor`，它的功能非常强大，后面的文章会详细讲解。经过一系列的调用，最终会到`PostProcessorRegistrationDelegate`类的`registerBeanPostProcessors`方法：

![](\assets\images\2021\spring\application-context-refresh-17.png)

注意，这一步只是注册`BeanPostProcessor`，真正的使用在后面。