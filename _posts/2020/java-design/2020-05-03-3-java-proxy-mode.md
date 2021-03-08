---
layout: post
title: 23种设计模式之代理模式
category: java-design
tags: [java-design]
keywords: java
excerpt: 代理模式的概念,静态代理,JDK动态代理,CGLIB动态代理,Javassist动态代理,ASM动态代理
lock: noneed
---

<mark>代理模式Proxy</mark>

![](\assets\images\2021\javabase\proxy-mini.png)

动态代理在 Java 中有着广泛的应用，比如 **AOP 的实现原理、RPC远程调用、Java 注解对象获取、日志框架、全局性异常处理、事务处理等**。

## 1、代理模式的概念

`代理模式(Proxy Pattern)`是 23 种设计模式的一种，属于`结构型模式`。他指的是一个对象（代理对象）本身不做实际的操作，而是通过其他对象（目标对象）来得到自己想要的结果。这样做的好处是可以在**目标对象实现的基础上，增强额外的功能操作，即扩展目标对象的功能**。代理模式就是加一层的架构思想。

这里能体现出一个非常重要的编程思想：不要随意去改源码，如果需要修改，<font color=red>可以通过代理的方式来扩展该方法</font>，无侵入式的统一结果返回，就是拦截返回结果再进行封装返回给Response。

![](\assets\images\2021\javabase\proxy-pattern.png)

如上图所示，用户不能直接使用目标对象，而是构造出一个代理对象，由代理对象作为中转，代理对象负责调用目标对象真正的行为，从而把结果返回给用户。

代理的关键点就是**代理对象和目标对象的关系**。

代理其实就和经纪人一样，比如你是一个明星，有很多粉丝。你的流量很多，经常会有很多金主来找你洽谈合作等，你自己肯定忙不过来，因为你要处理的不只是谈合作这件事情，你还要懂才艺、拍戏、维护和粉丝的关系、营销等。为此，你找了一个经纪人，你让他负责和金主谈合作这件事，经纪人做事很认真负责，他圆满的完成了任务，于是，金主找你谈合作就变成了金主和你的经纪人谈合作，你就有更多的时间来忙其他事情了。如下图所示

![](\assets\images\2021\javabase\proxy-pattern-2.png)

这是一种静态代理，因为这个`代理(经纪人)`是你自己亲自挑选的。

但是后来随着你的业务逐渐拓展，你无法选择每个经纪人，所以你索性交给了代理公司来帮你做。如果你想在 B 站火一把，那就直接让代理公司帮你找到负责营销方面的代理人，如果你想维护和粉丝的关系，那你直接让代理公司给你找一些托儿就可以了，那么此时的关系图会变为如下

![](\assets\images\2021\javabase\proxy-pattern-3.png)

此时你几乎所有的工作都是由代理公司来进行打理，而他们派出谁来帮你做这些事情你就不得而知了，这得根据实际情况来定，因为代理公司也不只是负责你一个明星，而且每个人所擅长的领域也不同，所以你只有等到有实际需求后，才会给你指定对应的代理人，这种情况就叫做`动态代理`。从编程角度来看，这些代理人就是是代理接口下的所有类，所以`动态代理`是代理接口/抽象类的。

## 2 、静态代理

在编译阶段就能确定最终的执行方法的代理模式，这是静态代理

### 租房子的例子

 ![](/assets/images/2020/java/proxy-example.gif)

静态代理，角色分析：

- 抽象角色：一般会使用接口和抽象类实现
- 真实的角色：被代理的角色
- 代理角色：代理真实角色，代理后，一般会做一些附属操作（增强）
- 客户：访问代理的人

代码实现

1、抽象角色Rent接口

```java
public interface Rent {
	void rent();
}
```

2、真实的角色-房东

```java
public class Host implements Rent {
  // 真实角色要做的操作
	@Override
	public void rent() {
		System.out.println("房东要出租房子。。。。");
	}
}
```

3、代理角色-中介

```java
// 房东不想去找客户租房子，让中介代理
public class HostProxy implements Rent{
  // 具体的目标对象（真实角色）
	private Host host;

	public HostProxy(){}

	public HostProxy(Host host){
		this.host = host;
	}

	@Override
	public void rent() {
		// 房东出租房子
		host.rent();
		// 中介带你看房子
		seeHouse();
		// 合适就签合同
		signRentContract();
		// 收中介费
		fee();
	}

	// 代理的一些附属操作：
	// 看房
	public void seeHouse() {
		System.out.println("中介带你看房子");
	}

	// 签合同
	public void signRentContract() {
		System.out.println("签租赁合同");
	}

	// 收中介费
	public void fee() {
		System.out.println("收中介费");
	}
}
```

4、客户：你要租房子，找到代理

```java
public class Client {
	public static void main(String[] args) {
		Host host = new Host();
		HostProxy proxy = new HostProxy(host);
		proxy.rent();
	}
}
```

执行结果：

![](\assets\images\2020\java\proxy-rent-1.jpg)

### 修改用户的例子

1、抽象角色UserService接口

```java
public interface UserService {
	void add();

	void delete();

	void update();

	void query();
}
```

2、真实的角色UserServiceImpl实现类

```java
public class UserServiceImpl implements UserService{
	@Override
	public void add() {
		System.out.println("新增了一个用户");
	}

	@Override
	public void delete() {
		System.out.println("删除了一个用户");
	}

	@Override
	public void update() {
		System.out.println("更新了一个用户");
	}

	@Override
	public void query() {
		System.out.println("查询了多个用户");
	}
}
```

现在我们想在调用CRUD（增删改查）前记录一下日志，在不入侵原有代码的原则下，加一层代理。

3、代理角色UserServiceProxy类

```java
public class UserServiceProxy implements UserService{
	private UserService userService;

	public void setUserService(UserService userService){
		this.userService = userService;
	}

	@Override
	public void add() {
		log("add");
		userService.add();
	}

	@Override
	public void delete() {
		log("delete")
		userService.delete();
	}

	@Override
	public void update() {
		log("update");
		userService.update();
	}

	@Override
	public void query() {
		log("query");
		userService.query();
	}

	private void log(String msg){
		System.out.println("【Debug】使用了"+ msg +"方法");
	}
}
```

4、客户UserClient类

```java
public class UserClient {
	public static void main(String[] args) {
		UserServiceImpl userService = new UserServiceImpl();
		UserServiceProxy proxy = new UserServiceProxy();
		proxy.setUserService(userService);
		proxy.add();
	}
}
```

### 小结

- 静态代理类：由程序员创建或者由第三方工具生成，再进行编译；在程序运行之前，代理类的 .class 文件已经存在了。
- 静态代理事先知道要代理的是什么。
- 静态代理类通常只代理一个类。

优点

- **可以使真实角色的操作更加纯粹，不用关注公共业务**
- 公共业务交给代理角色，实现业务分工，降低耦合性
- 公共业务发生扩展的时候，方便集中管理

缺点

- 一个真实角色会产生一个代理角色，代码量翻倍，开发效率变低！

代理类UserServiceProxy本身不做用户数据的CRUD,内部是调用实现类UserServiceImpl的CRUD，所以JVM在编译阶段就已经确定了最终的执行方法，这种代理模式叫做静态代理。

相关知识点，AOP：

![](/assets/images/2020/java/mode-aop.gif)

代理模式具有无侵入式的优点，但是静态代理模式的缺点是每个实现类都有一个对应的代理类的代码要写，当代理类很多且要修改的时候就会很烦。所以使用动态代理来代理一个接口，而不是代理一个具体的目标类。

## 3、动态代理

- 动态代理和静态代理的角色一样
- 代理类是动态生成的
- 动态代理分为两大类：基于接口的动态代理 + 基于类的动态代理

**动态代理的实现技术有4种：**

- 基于接口--使用JDK动态代理实现，内置在 JDK 中

- 基于类--- Cglib实现

- Java字节码实现--Javassist

  Cglib 和 Javassist 都是高级的字节码生成库，总体性能比 JDK 自带的动态代理好，而且功能十分强大。

- ASM代理

  SM 是低级的字节码生成工具，使用 ASM 已经近乎于在使用字节码编程，对开发人员要求最高。当然，也是`性能最好`的一种动态代理生成工具。但 ASM 的使用很繁琐，而且性能也没有数量级的提升，与 CGLIB 等高级字节码生成工具相比，ASM 程序的维护性较差，如果不是在对性能有苛刻要求的场合，还是推荐 CGLIB 或者 Javassist。



### JDK动态代理

需要了解反射相关的两个类：Proxy、InvocationHandler，动态代理的处理类事先继承InvocationHandler接口，使用 Proxy 类中的 newProxyInstance 方法动态的创建代理类。

![](/assets/images/2020/java/invocation-handler.gif)

![](/assets/images/2020/java/invocation-handler3.gif)

新建InvocationHandler的实现类HostInvocationHandler

```java
public class HostInvocationHandler implements InvocationHandler {
  // 被代理的接口
  private Rent rent;

  public void setRent(Rent rent) {
    this.rent = rent;
  }

  // 动态生成得到代理类
  public Object getProxy() {
    return Proxy.newProxyInstance(this.getClass().getClassLoader(),rent.getClass().getInterfaces(),this);
  }

  // 处理代理实例，并返回结果
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable{
    // 动态代理的本质，就是使用反射机制实现
    Object result = method.invoke(rent,args);
    seeHouse();
    signRentContract();
    fee();
    return result;
  }

  // 看房
  public void seeHouse() {
    System.out.println("动态代理-中介带你看房子");
  }

  // 签合同
  public void signRentContract() {
    System.out.println("动态代理-签租赁合同");
  }

  // 收中介费
  public void fee() {
    System.out.println("动态代理-收中介费");
  }
}
```

客户Client

```java
public class Client {
  public static void main(String[] args) {
    // 真实角色：房东
    Host host = new Host();
    // 代理角色没有，动态生成一个
    HostInvocationHandler handler = new HostInvocationHandler();
    // 要代理的目标对象
    handler.setRent(host);
    Rent proxy =(Rent) handler.getProxy();
    proxy.rent();
  }
}
```

运行结果：

![](\assets\images\2020\java\proxy-rent-2.jpg)

HostInvocationHandler是动态代理类，里面有一个接口Rent，方法getProxy() 就是通过类加载器、接口数组获取代理类实例，proxy.rent()执行是接口方法，最终是由invoke触发的执行的是隐藏在代理对象后面的真实对象host的rent()方法。JVM在编译期阶段是无法确定最终的执行方法的。

> 封装成工具类写法

新建ProxyInvocationHandler

```java
public class ProxyInvocationHandler implements InvocationHandler {
	// 被代理的接口
	private Object target;

	public void setTarget(Object target){
		this.target = target;
	}

	// 生成得到代理类
	public Object getProxy() {
		return Proxy.newProxyInstance(this.getClass().getClassLoader(),target.getClass().getInterfaces(),this);
	}

  // 处理代理实例,并返回结果
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable{
    // 通过反射获取方法名
    log(method.getName());
    Object result = method.invoke(target,args);
    return result;
  }
  
  private void log(String msg){
    System.out.println("【Debug】使用了"+ msg +"方法");
  }
}
```

客户角色Client类

```java
public class Client {
  public static void main(String[] args) {
    // 真实角色
    UserServiceImpl userService = new UserServiceImpl();
    // 代理角色，不存在，通过handler生成
    ProxyInvocationHandler handler = new ProxyInvocationHandler();
    // 设置要代理的对象
    handler.setTarget(userService);
    UserService proxy = (UserService) handler.getProxy();
    proxy.add();
    proxy.query();
  }
}
```

运行结果应该很清晰啦，就不截图了

JDK动态代理的特点：

- 动态代理通常是在程序运行时，通过`反射机制`动态生成的。
- 动态代理类通常代理`接口`下的所有类。
- 动态代理事先不知道要代理的是什么，只有在运行的时候才能确定。
- 动态代理的调用处理程序必须事先继承 InvocationHandler 接口，使用 Proxy 类中的 newProxyInstance 方法动态的创建代理类。

### CGLIB 动态代理

上面我们提到 JDK 动态代理是基于接口的代理，而 CGLIB 动态代理**是针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法** ，也就是说 CGLIB 动态代理采用类继承 -> 方法重写的方式进行的，下面我们先来看一下 CGLIB 动态代理的结构。

![](\assets\images\2021\javabase\proxy-pattern-4.png)

如上图所示，代理类继承于目标类，每次调用代理类的方法都会在拦截器中进行拦截，拦截器中再会调用目标类的方法。下面我们通过一个示例来演示一下 CGLIB 动态代理的使用，新建一个Springboot工程。

1、pom.xml导入依赖

```xml
<dependency>
  <groupId>cglib</groupId>
  <artifactId>cglib</artifactId>
  <version>3.2.5</version>
</dependency>
```

2、以上面租房子的例子为基础，Host类就是我们要代理的目标类，我们通过代理模式在执行Host.rent()方法的前后增加一些操作进行增强，同时不入侵目标类方法。

创建一个自定义方法拦截类

```java
public class AutoMethodInterceptor implements MethodInterceptor {

  public Object intercept(Object obj, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    System.out.println("---- 方法拦截 ----");
    Object object = methodProxy.invokeSuper(obj, args);
    return object;
  }
}
```

这里解释一下这几个参数都是什么含义

- Object obj: obj 是 CGLIB 动态生成代理类实例
- Method method: Method 为实体类所调用的被代理的方法引用
- Objectp[] args: 这个就是方法的参数列表
- MethodProxy methodProxy : 这个就是生成的代理类对方法的引用。

对于 `methodProxy` 参数调用的方法，在其内部有两种选择：`invoke()` 和 `invokeSuper()` ，二者的区别不在本文展开说明，感兴趣的读者可以参考本篇文章：Cglib源码分析 invoke和invokeSuper的差别

3、测试一下

```java
public class Client {
  public static void main(String[] args) {
    // 3、cglib动态代理
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(Host.class);
    enhancer.setCallback(new AutoMethodInterceptor());
    Host host1 = (Host)enhancer.create();
    host1.rent();
  }
}
```

执行结果：

![](\assets\images\2021\javabase\proxy-pattern-cglib-1.png)

实际项目中引入的cglib.jar是3.1版本的，百度上说是因为与asm版本冲突的原因

![](\assets\images\2021\javabase\proxy-pattern-cglib-2.png)

修改版本为3.2.5，maven重新引入jar包，重新执行成功

![](\assets\images\2021\javabase\proxy-pattern-cglib-3.png)

我们给拦截方法增加一些业务操作，如下

```java
public class AutoMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        System.out.println("---- 方法拦截 ----");
        Object object = methodProxy.invokeSuper(obj, args);
        seeHouse();
        signRentContract();
        fee();
        return object;
    }

    // 看房
    public void seeHouse() {
        System.out.println("cglib动态代理-中介带你看房子");
    }
    // 签合同
    public void signRentContract() {
        System.out.println("cglib动态代理-签租赁合同");
    }
    // 收中介费
    public void fee() {
        System.out.println("cglib动态代理-收中介费");
    }
}
```

执行一些Client.main方法，结果：

![](\assets\images\2021\javabase\proxy-pattern-cglib-4.png)

main方法中主要涉及 `Enhancer` 的使用，Enhancer 是一个非常重要的类，它允许为`非接口类型`创建一个 Java 代理，Enhancer 动态的创建给定类的子类并且拦截代理类的所有的方法，和 JDK 动态代理不一样的是不管是接口还是类它都能正常工作。

> 小结

- JDK 动态代理与 CGLIB 动态代理都是将真实对象<mark>隐藏</mark>在代理对象的后面，以达到 `代理` 的效果。与 JDK 动态代理所不同的是 CGLIB 动态代理使用 Enhancer 来创建代理对象，而 JDK 动态代理使用的是 Proxy.newProxyInstance 来创建代理对象；
- CGLIB 可以代理大部分类，而 JDK 动态代理只能代理实现了接口的类。



### Javassit代理

**百度百科：**

Javassist是一个[开源](https://baike.baidu.com/item/开源/20720669)的分析、编辑和创建Java[字节码](https://baike.baidu.com/item/字节码)的类库。是由[东京工业大学](https://baike.baidu.com/item/东京工业大学/2567295)的数学和计算机科学系的 Shigeru Chiba （千叶 滋）所创建的。它已加入了开放源代码[JBoss](https://baike.baidu.com/item/JBoss/5611767) [应用服务器](https://baike.baidu.com/item/应用服务器/4971773)项目，通过使用Javassist对字节码操作为JBoss实现动态"AOP"框架。关于java字节码的处理，有很多工具，如bcel，[asm](https://baike.baidu.com/item/asm)。不过这些都需要直接跟[虚拟机](https://baike.baidu.com/item/虚拟机)指令打交道。如果你不想了解虚拟机指令，可以采用javassist。javassist是[jboss](https://baike.baidu.com/item/jboss)的一个子项目，其主要的优点，在于简单，而且快速。直接使用java编码的形式，而不需要了解虚拟机指令，就能动态改变类的结构，或者动态生成类。

`Javassist`是在 Java 中编辑字节码的类库(直接修改字节码，更低层)；它使 Java 程序能够在运行时定义一个新类, 并在 JVM 加载时修改类文件。我们使用最频繁的动态特性就是 `反射`，而且反射也是动态代理的基础，我们之所以没有提反射对动态代理的作用是因为我想在后面详聊，反射可以在运行时查找对象属性、方法，修改作用域，通过方法名称调用方法等。实时应用不会频繁使用反射来创建，因为反射开销比较大，另外，还有一种具有和反射一样功能强大的特性那就是 `Javaassist`。

以上面租房子的例子，继续说明

1、pom.xml导入依赖

```xml
<dependency>
    <groupId>org.javassist</groupId>
    <artifactId>javassist</artifactId>
    <version>3.27.0-GA</version>
</dependency>
```

2、新建一个AssistByteCode 类

```java
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

/**
 * @Author xiejw17
 * @Date 2021/1/20 11:03
 */
public class AssistByteCode {
    public static void createByteCode() throws Exception {
        ClassPool classPool = ClassPool.getDefault();
        // 类路径
        CtClass ctClass = classPool.makeClass("proxy.Host2");
        // 设置接口
        CtClass ctInterface = classPool.get("proxy.Rent");
        ctClass.setInterfaces(new CtClass[]{ctInterface});

        // 创建方法
        CtMethod rent = CtMethod.make("public void rent(){}",ctClass);
        rent.setBody("System.out.println(\"房东要出租房子。。。。\");");
        ctClass.addMethod(rent);

        // 输出字节码文件
        Class c = ctClass.toClass();
      	ctClass.writeFile("D:/jacob/code/c-ccs/web-parent/ccs-common/src/test/java/");
    }
}
```

`ClassPool`：ClassPool 就是一个 CtClass 的容器，而一个 `CtClass` 对象就是一个 class 对象的实例，这个实例和 class 对象一样，包含属性、方法等。

那么上面代码主要做了哪些事儿呢？通过 ClassPool 来获取 CtClass 所需要的接口、抽象类的 CtClass 实例，然后通过 CtClass 实例添加自己的属性和方法，并通过它的 writeFile 把二进制流输出到当前项目的根目录路径下。writeFile 其内部是使用了 `DataOutputStream` 进行输出的。

我们在main方法里执行一下createByteCode()方法

```java
public static void main(String[] args) {
  try {
    AssistByteCode.createByteCode();
  } catch (Exception e) {
    e.printStackTrace();
  }
}
```

流写完后，我们打开这个 `.class` 文件如下所示

![](\assets\images\2021\javabase\proxy-javassist-1.png)

可以对比一下上面发现 Host2 发现编译器除了为我们添加了一个公有的构造器，其他基本一致。

![](\assets\images\2021\javabase\proxy-javassist-2.png)

上面演示了javassist动态的在JVM新增一个类的字节码文件。

javassist如何实现动态代理，创建一个代理工厂，代码如下：

```java
import javassist.util.proxy.ProxyFactory;

/**
 * @Author xiejw17
 * @Date 2021/1/20 15:07
 */
public class JavaassistProxyFactory {
    public Object getProxy(Class clazz) throws Exception{

        // 代理工厂
        ProxyFactory proxyFactory = new ProxyFactory();
        // 设置需要创建的子类
        proxyFactory.setSuperclass(clazz);
        proxyFactory.setHandler((self, thisMethod, proceed, args) -> {
            System.out.println("---- javassist动态代理,开始拦截 ----");
            Object result = proceed.invoke(self, args);
            System.out.println("---- javassist动态代理,结束拦截 ----");
            return result;
        });
        return proxyFactory.createClass().newInstance();
    }
}
```

上面我们定义了一个代理工厂，代理工厂里面创建了一个 handler，在调用目标方法时，Javassist 会回调 MethodHandler 接口方法拦截，来调用真正执行的方法，你可以在拦截方法的前后实现自己的业务逻辑。最后的 **proxyFactory.createClass().newInstance()** 就是使用字节码技术来创建了最终的子类实例，这种代理方式类似于 JDK 中的 InvocationHandler 接口。

在Client.main()方法执行一下

```java
public class Client {
    public static void main(String[] args) {
      // 4、javassist动态代理
      JavaassistProxyFactory proxyFactory = new JavaassistProxyFactory();
      Host proxy = null;
      try {
        proxy = (Host)proxyFactory.getProxy(Host.class);
      } catch (Exception e) {
        e.printStackTrace();
      }
      proxy.rent();
    }
}
```

![](D:\jacob\code\aikomj.github.io\assets\images\2021\javabase\proxy-javassist-3.png)

### ASM代理

ASM 是一套 Java 字节码生成架构，它可以动态生成二进制格式的子类或其它代理类，或者在类被 Java 虚拟机装入内存之前，动态修改类。

下面我们使用 ASM 框架实现一个动态代理，ASM 生成的动态代理

以下代码摘自 https://blog.csdn.net/lightj1996/article/details/107305662

```java
public class AsmProxy extends ClassLoader implements Opcodes {

    public static void createAsmProxy() throws Exception {

        // 目标类类名 字节码中类修饰符以 “/” 分割
        String targetServiceName = TargetService.class.getName().replace(".", "/");
        // 切面类类名
        String aspectServiceName = AspectService.class.getName().replace(".", "/");
        // 代理类类名
        String proxyServiceName = targetServiceName+"Proxy";
        // 创建一个 classWriter 它是继承了ClassVisitor
        ClassWriter classWriter = new ClassWriter(0);
        // 访问类 指定jdk版本号为1.8, 修饰符为 public，父类是TargetService
        classWriter.visit(Opcodes.V1_8, Opcodes.ACC_PUBLIC, proxyServiceName, null, targetServiceName, null);
        // 访问目标类成员变量 为类添加切面属性 “private TargetService targetService”
        classWriter.visitField(ACC_PRIVATE, "targetService", "L" + targetServiceName+";", null, null);
        // 访问切面类成员变量 为类添加目标属性 “private AspectService aspectService”
        classWriter.visitField(ACC_PRIVATE, "aspectService", "L" + aspectServiceName+";", null, null);

        // 访问默认构造方法 TargetServiceProxy()
        // 定义函数 修饰符为public 方法名为 <init>， 方法表述符为()V 表示无参数，无返回参数
        MethodVisitor initVisitor = classWriter.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
        // 从局部变量表取第0个元素 “this”
        initVisitor.visitVarInsn(ALOAD, 0);
        // 调用super 的构造方法 invokeSpecial在这里的意思是调用父类方法
        initVisitor.visitMethodInsn(INVOKESPECIAL, targetServiceName, "<init>", "()V", false);
        // 方法返回
        initVisitor.visitInsn(RETURN);
        // 设置最大栈数量，最大局部变量表数量
        initVisitor.visitMaxs(1, 1);
        // 访问结束
        initVisitor.visitEnd();

        // 创建有参构造方法 TargetServiceProxy(TargetService var1, AspectService var2)
        // 定义函数 修饰符为public 方法名为 <init>， 方法表述符为(TargetService, AspectService)V 表示无参数，无返回参数
        MethodVisitor methodVisitor = classWriter.visitMethod(ACC_PUBLIC, "<init>", "(L" + targetServiceName + ";L"+aspectServiceName+";)V", null, null);
        // 从局部变量表取第0个元素 “this”压入栈顶
        methodVisitor.visitVarInsn(ALOAD, 0);
        // this出栈 , 调用super 的构造方法 invokeSpecial在这里的意思是调用父类方法。 <init>的owner是AspectService, 无参无返回类型
        methodVisitor.visitMethodInsn(INVOKESPECIAL, targetServiceName, "<init>", "()V", false);
        // 从局部变量表取第0个元素 “this”压入栈顶
        methodVisitor.visitVarInsn(ALOAD, 0);
        // 从局部变量表取第1个元素 “targetService”压入栈顶
        methodVisitor.visitVarInsn(ALOAD, 1);
        // this 和 targetService 出栈, 调用targetService put 赋值给this.targetService
        methodVisitor.visitFieldInsn(PUTFIELD, proxyServiceName, "targetService", "L" + targetServiceName + ";");
        // 从局部变量表取第0个元素 “this”压入栈顶
        methodVisitor.visitVarInsn(ALOAD, 0);
        // 从局部变量表取第2个元素 “aspectService”压入栈顶
        methodVisitor.visitVarInsn(ALOAD, 2);
        // this 和 aspectService 出栈 将 targetService put 赋值给this.aspectService
        methodVisitor.visitFieldInsn(PUTFIELD, proxyServiceName, "aspectService", "L" + aspectServiceName + ";");
        // 方法返回
        methodVisitor.visitInsn(RETURN);
        // 设置最大栈数量，最大局部变量表数量
        methodVisitor.visitMaxs(2, 3);
        // 方法返回
        methodVisitor.visitEnd();

        // 创建代理方法 修饰符为public，方法名为 demoQuest
        MethodVisitor visitMethod = classWriter.visitMethod(ACC_PUBLIC, "demoQuest", "()I", null, null);
        // 从局部变量表取第0个元素 “this”压入栈顶
        visitMethod.visitVarInsn(ALOAD, 0);
        // this 出栈 将this.aspectService压入栈顶
        visitMethod.visitFieldInsn(GETFIELD, proxyServiceName, "aspectService", "L"+aspectServiceName+";");
        // 取栈顶元素出栈 也就是targetService 调用其preOperation方法， demoQuest的owner是AspectService, 无参无返回类型
        visitMethod.visitMethodInsn(INVOKEVIRTUAL, aspectServiceName,"preOperation", "()V", false);
        // 从局部变量表取第0个元素 “this”压入栈顶
        visitMethod.visitVarInsn(ALOAD, 0);
        // this 出栈, 取this.targetService压入栈顶
        visitMethod.visitFieldInsn(GETFIELD, proxyServiceName, "targetService", "L"+targetServiceName+";");
        // 取栈顶元素出栈 也就是targetService调用其demoQuest方法, demoQuest的owner是TargetService, 无参无返回类型
        visitMethod.visitMethodInsn(INVOKEVIRTUAL, targetServiceName, "demoQuest", "()I", false);
        // 方法返回
        visitMethod.visitInsn(IRETURN);
        // 设置最大栈数量，最大局部变量表数量
        visitMethod.visitMaxs(1, 1);
        // 方法返回
        visitMethod.visitEnd();

        // 生成字节码二进制流
        byte[] code = classWriter.toByteArray();
        // 自定义classloader加载类
        Class<?> clazz = (new AsmProxy()).defineClass(TargetService.class.getName() + "Proxy", code, 0, code.length);
        // 取其带参数的构造方法
        Constructor constructor = clazz.getConstructor(TargetService.class, AspectService.class);
        // 使用构造方法实例化对象
        Object object = constructor.newInstance(new TargetService(), new AspectService());

        // 使用TargetService类型的引用接收这个对象
        TargetService targetService;
        if (!(object instanceof TargetService)) {
            return;
        }
        targetService = (TargetService)object;

        System.out.println("生成代理类的名称: " + targetService.getClass().getName());
        // 调用被代理方法
        targetService.demoQuest();

        // 这里可以不用写, 但是如果想看最后生成的字节码长什么样子，可以写 "ascp-purchase-app/target/classes/"是我的根目录, 阅读者需要将其替换成自己的
        String classPath = "/Users/mr.l/cxuan-justdoit/";
        String path = classPath + proxyServiceName + ".class";
        FileOutputStream fos =
                new FileOutputStream(path);
        fos.write(code);
        fos.close();

    }
}
```

使用 ASM 生成动态代理的代码比较长，上面这段代码的含义就是生成类 TargetServiceProxy，用于代理TargetService ，在调用 targetService.demoQuest() 方法之前调用切面的方法 aspectService.preOperation();

测试类就直接调用 AsmProxy.createAsmProxy() 方法即可，比较简单。

下面是我们生成 TargetServiceProxy 的目标类

![](\assets\images\2021\javabase\proxy-asm-1.png)

### 小结

至此，我们已经介绍了四种动态代理的方式，分别是**JDK 动态代理、CGLIB 动态代理、Javaassist 动态代理、ASM 动态代理**，那么现在思考一个问题，为什么会有动态代理的出现呢？或者说动态代理是基于什么原理呢？

其实我们上面已经提到过了，没错，动态代理使用的就是`反射` 机制，反射机制是 Java 语言提供的一种基础功能，􏱥􏱩赋予程序在运行时动态修改属性、方法的能力。通过反射我们能够直接操作类或者对象，比如获取某个类的定义，获取某个类的属性和方法等。

另外还有需要注意的一点，从性能角度来讲，有些人得出结论说是 Java 动态代理要比 CGLIB 和 Javaassist 慢几十倍，其实，在主流 JDK 版本中，Java 动态代理可以提供相等的性能水平，**数量级的差距不是广泛存在的**。而且，在现代 JDK 中，反射已经得到了改进和优化。

动态代理的好处：

- 一个动态代理类代理的是一个接口，一般对应一类业务，而静态代理类代理的是一个真实的角色对象。
- 一个动态可以代理多个类，只要是实现同一个接口。
- 在无侵入式的代码扩展目标类的方法（增强），同时减少代码量，避免代理类泛滥成灾。

我们在选型中，性能考量并不是主要关注点，**可靠性、可维护性、编码工作量**同等重要。

## 4、意图

以下内容转载自 [https://refactoringguru.cn/design-patterns/proxy](https://refactoringguru.cn/design-patterns/proxy)

结构型设计模式：该模式将对象和类组装成较大的结构， 并同时保持结构的灵活和高效

**代理模式**是一种结构型设计模式， 让你能够提供对象的替代品或其占位符。 代理控制着对于原对象的访问， 并允许在将请求提交给对象前后进行一些处理。

![](\assets\images\2021\javabase\proxy.png)

### 问题

为什么要控制对于某个对象的访问呢？ 举个例子： 有这样一个消耗大量系统资源的巨型对象， 你只是偶尔需要使用它， 并非总是需要。如数据库的访问

![](\assets\images\2021\javabase\proxy-problem-zh.png)

### 解决方案

代理模式建议新建一个与原服务对象接口相同的代理类， 然后更新应用以将代理对象传递给所有原始对象客户端。 代理类接收到客户端请求后会创建实际的服务对象， 并将所有工作委派给它。动态代理

![](\assets\images\2021\javabase\proxy-solution-zh.png)

<mark>代理将自己伪装成数据库对象， 可在客户端或实际数据库对象不知情的情况下处理延迟初始化和缓存查询结果的工作。</mark>

这有什么好处呢？ 如果需要在类的主要业务逻辑前后执行一些工作， 你无需修改类就能完成这项工作。 由于代理实现的接口与原类相同， 因此你可将其传递给任何一个使用实际服务对象的客户端。

### 真实世界类比

![](\assets\images\2021\javabase\proxy-live-example-zh.png)

信用卡是银行账户的代理， 银行账户则是一大捆现金的代理。 它们都实现了同样的接口， 均可用于进行支付。 消费者会非常满意， 因为不必随身携带大量现金； 商店老板同样会十分高兴， 因为交易收入能以电子化的方式进入商店的银行账户中， 无需担心存款时出现现金丢失或被抢劫的情况。信用卡和银行账户支付都是现金支付的代理，它可以记录支付的相关信息，相当于增强了现金支付的功能，你使用现金支付是没有记录相关信息的。

### 代理模式结构

![](\assets\images\2021\javabase\proxy-structure.png)

代理对象可以在服务对象的基础上做增强，如记录日志、访问控制、缓存

### 伪代码

本例演示如何使用**代理**模式在第三方腾讯视频 （TencentVideo， 代码示例中记为 TV） 程序库中添加延迟初始化和缓存

![](\assets\images\2021\javabase\proxy-example-zh.png)

程序库提供了视频下载类。 但是该类的效率非常低。 如果客户端程序多次请求同一视频， 程序库会反复下载该视频， 而不会将首次下载的文件缓存下来复用。

代理类实现和原下载器相同的接口， 并将所有工作委派给原下载器。 不过， 代理类会保存所有的文件下载记录， 如果程序多次请求同一文件， 它会返回缓存的文件

```java
// 1、远程服务接口。
interface ThirdPartyTVLib is
    method listVideos()
    method getVideoInfo(id)
    method downloadVideo(id)

// 2、服务对象（目标对象）  
// 服务连接器的具体实现。该类的方法可以向腾讯视频请求信息。请求速度取决于
// 用户和腾讯视频的互联网连接情况。如果同时发送大量请求，即使所请求的信息
// 一模一样，程序的速度依然会减慢。
class ThirdPartyTVClass implements ThirdPartyTVLib is
    method listVideos() is
        // 向腾讯视频发送一个 API 请求。

    method getVideoInfo(id) is
        // 获取某个视频的元数据。

    method downloadVideo(id) is
        // 从腾讯视频下载一个视频文件。

// 3、代理对象  
// 为了节省网络带宽，我们可以将请求结果缓存下来并保存一段时间。但你可能无
// 法直接将这些代码放入服务类中。比如该类可能是第三方程序库的一部分或其签
// 名是`final（最终）`。因此我们会在一个实现了服务类接口的新代理类中放入
// 缓存代码。当代理类接收到真实请求后，才会将其委派给服务对象。
class CachedTVClass implements ThirdPartyTVLib is
    private field service: ThirdPartyTVLib
    private field listCache, videoCache
    field needReset

    constructor CachedTVClass(service: ThirdPartyTVLib) is
        this.service = service

    method listVideos() is
        if (listCache == null || needReset)
            listCache = service.listVideos()
        return listCache

    method getVideoInfo(id) is
        if (videoCache == null || needReset)
            videoCache = service.getVideoInfo(id)
        return videoCache

    method downloadVideo(id) is
        if (!downloadExists(id) || needReset)
            service.downloadVideo(id)

// 4、管理类
// 之前直接与服务对象交互的 GUI 类不需要改变，前提是它仅通过接口与服务对
// 象交互。我们可以安全地传递一个代理对象来代替真实服务对象，因为它们都实
// 现了相同的接口。
class TVManager is
    protected field service: ThirdPartyTVLib

    constructor TVManager(service: ThirdPartyTVLib) is
        this.service = service

    method renderVideoPage(id) is
        info = service.getVideoInfo(id)
        // 渲染视频页面。

    method renderListPanel() is
        list = service.listVideos()
        // 渲染视频缩略图列表。

    method reactOnUserInput() is
        renderVideoPage()
        renderListPanel()

// 5、客户端
// 程序可在运行时对代理进行配置。
class Application is
    method init() is
        aTVService = new ThirdPartyTVClass()
        aTVProxy = new CachedTVClass(aTVService)
        manager = new TVManager(aTVProxy)
        manager.reactOnUserInput()      
      
```

这个代码示例是静态代理的方式。

### 适合应用场景

1. 延迟初始化 （虚拟代理）。 如果你有一个偶尔使用的重量级服务对象， 一直保持该对象运行会消耗系统资源时， 可使用代理模式。

    <mark>你无需在程序启动时就创建该对象， 可将对象的初始化延迟到真正有需要的时候。</mark>

2. 访问控制 （保护代理）。 如果你只希望特定客户端使用服务对象， 这里的服务对象可以是操作系统中非常重要的部分， 而客户端则是各种已启动的程序 （包括恶意程序）， 此时可使用代理模式。

   <mark>代理可仅在客户端凭据满足要求时将请求传递给服务对象。</mark>

3.  本地执行远程服务 （远程代理）。 适用于服务对象位于远程服务器上的情形。

    在这种情形中， 代理通过网络传递客户端请求， 负责处理所有与网络相关的复杂细节。

4.  记录日志请求 （日志记录代理）。 适用于当你需要保存对于服务对象的请求历史记录时。 代理可以在向服务传递请求前进行记录。

5.  缓存请求结果 （缓存代理）。 适用于需要缓存客户请求结果并对缓存生命周期进行管理时， 特别是当返回结果的体积非常大时。

   代理可对重复请求所需的相同结果进行缓存， 还可使用请求参数作为索引缓存的键值。

6. 智能引用。 可在没有客户端使用某个重量级对象时立即销毁该对象。

   代理会将所有获取了指向服务对象或其结果的客户端记录在案。 代理会时不时地遍历各个客户端， 检查它们是否仍在运行。 如果相应的客户端列表为空， 代理就会销毁该服务对象， 释放底层系统资源。像GC垃圾回收，原理是懂了但是落地实现，还是不懂。

   代理还可以记录客户端是否修改了服务对象。 其他客户端还可以复用未修改的对象。

### 实现方式

1. 如果没有现成的服务接口， 你就需要创建一个接口来实现代理和服务对象的可交换性。 从服务类中抽取接口并非总是可行的， 因为你需要对服务的所有客户端进行修改， 让它们使用接口。 **备选计划是将代理作为服务类的子类， 这样代理就能继承服务的所有接口了**，CGLIB就是使用类继承的方式实现动态创建代理类的。
2. 创建代理类， 其中必须包含一个存储指向服务的引用的成员变量。 通常情况下， 代理负责创建服务并对其整个生命周期进行管理。 在一些特殊情况下， 客户端会通过构造函数将服务传递给代理。
3. 根据需求实现代理方法。 在大部分情况下， 代理在完成一些任务后应将工作委派给服务对象。
4. 可以考虑新建一个构建方法来判断客户端可获取的是代理还是实际服务。 你可以在代理类中创建一个简单的静态方法， 也可以创建一个完整的工厂方法。
5. 可以考虑为服务对象实现延迟初始化。

### 优缺点

- 你可以在客户端毫无察觉的情况下控制服务对象。
- 如果客户端对服务对象的生命周期没有特殊要求， 你可以对生命周期进行管理
- 即使服务对象还未准备好或不存在， 代理也可以正常工作。
- 开闭原则，你可以在不对服务或客户端做出修改的情况下创建新代理。

### 与其他模式的关系

- [适配器模式](https://refactoringguru.cn/design-patterns/adapter)能为被封装对象（目标对象，服务对象）提供不同的接口， [代理模式](https://refactoringguru.cn/design-patterns/proxy)能为对象提供相同的接口， [装饰模式](https://refactoringguru.cn/design-patterns/decorator)则能为对象提供加强的接口（目标对象和装饰器遵循同一接口）。

- [外观模式](https://refactoringguru.cn/design-patterns/facade)与[代理](https://refactoringguru.cn/design-patterns/proxy)的相似之处在于它们都缓存了一个复杂实体并自行对其进行初始化。 *代理*与其服务对象遵循同一接口， 使得自己和服务对象可以互换， 在这一点上它与外观不同。
- [装饰](https://refactoringguru.cn/design-patterns/decorator)和[代理](https://refactoringguru.cn/design-patterns/proxy)有着相似的结构， 但是其意图却非常不同。 这两个模式的构建都基于组合原则， 也就是说一个对象应该将部分工作委派给另一个对象。 两者之间的不同之处在于*代理*通常自行管理其服务对象的生命周期， 而*装饰*的生成则总是由客户端进行控制。

### 代码示例

复杂度：2

流行度：1

**使用示例：** 尽管代理模式在绝大多数 Java 程序中并不常见， 但它在一些特殊情况下仍然非常方便。 当你希望在无需修改客户代码的前提下于已有类的对象上增加额外行为时， 该模式是无可替代的。

**识别方法：**代理模式会将所有实际工作委派给一些其他对象。 除非代理是某个服务的子类， 否则每个代理方法最后都应该引用一个服务对象。

> 缓存代理

在本例中， 代理模式有助于实现延迟初始化， 并对低效的第三方 YouTube 集成程序库进行缓存

1)**some_cool_media_library/ThirdPartyYouTubeLib.java:**  远程服务接口

```java
package refactoring_guru.proxy.example.some_cool_media_library;

import java.util.HashMap;

public interface ThirdPartyYouTubeLib {
    HashMap<String, Video> popularVideos();

    Video getVideo(String videoId);
}
```

2) **some_cool_media_library/ThirdPartyYouTubeClass.java:**  远程服务实现

```java
public class ThirdPartyYouTubeClass implements ThirdPartyYouTubeLib {
  @Override
  public HashMap<String, Video> popularVideos() {
    connectToServer("http://www.youtube.com");
    return getRandomVideos();
  }

  @Override
  public Video getVideo(String videoId) {
    connectToServer("http://www.youtube.com/" + videoId);
    return getSomeVideo(videoId);
  }

  // -----------------------------------------------------------------------
  // Fake methods to simulate network activity. They as slow as a real life.

  private int random(int min, int max) {
    return min + (int) (Math.random() * ((max - min) + 1));
  }

  private void experienceNetworkLatency() {
    int randomLatency = random(5, 10);
    for (int i = 0; i < randomLatency; i++) {
      try {
        Thread.sleep(100);
      } catch (InterruptedException ex) {
        ex.printStackTrace();
      }
    }
  }

  private void connectToServer(String server) {
    System.out.print("Connecting to " + server + "... ");
    experienceNetworkLatency();
    System.out.print("Connected!" + "\n");
  }

  private HashMap<String, Video> getRandomVideos() {
    System.out.print("Downloading populars... ");

    experienceNetworkLatency();
    HashMap<String, Video> hmap = new HashMap<String, Video>();
    hmap.put("catzzzzzzzzz", new Video("sadgahasgdas", "Catzzzz.avi"));
    hmap.put("mkafksangasj", new Video("mkafksangasj", "Dog play with ball.mp4"));
    hmap.put("dancesvideoo", new Video("asdfas3ffasd", "Dancing video.mpq"));
    hmap.put("dlsdk5jfslaf", new Video("dlsdk5jfslaf", "Barcelona vs RealM.mov"));
    hmap.put("3sdfgsd1j333", new Video("3sdfgsd1j333", "Programing lesson#1.avi"));

    System.out.print("Done!" + "\n");
    return hmap;
  }

  private Video getSomeVideo(String videoId) {
    System.out.print("Downloading video... ");

    experienceNetworkLatency();
    Video video = new Video(videoId, "Some video title");

    System.out.print("Done!" + "\n");
    return video;
  }
}
```

some_cool_media_library/Video.java: 视频文件

```java
public class Video {
  public String id;
  public String title;
  public String data;

  Video(String id, String title) {
    this.id = id;
    this.title = title;
    this.data = "Random video.";
  }
}
```

代理Proxy

3)  **proxy/YouTubeCacheProxy.java:**  缓存代理

```java
public class YouTubeCacheProxy implements ThirdPartyYouTubeLib {
  private ThirdPartyYouTubeLib youtubeService;
  private HashMap<String, Video> cachePopular = new HashMap<String, Video>();
  private HashMap<String, Video> cacheAll = new HashMap<String, Video>();

  public YouTubeCacheProxy() {
    this.youtubeService = new ThirdPartyYouTubeClass();
  }

  @Override
  public HashMap<String, Video> popularVideos() {
    if (cachePopular.isEmpty()) {
      cachePopular = youtubeService.popularVideos();
    } else {
      System.out.println("Retrieved list from cache.");
    }
    return cachePopular;
  }

  @Override
  public Video getVideo(String videoId) {
    Video video = cacheAll.get(videoId);
    if (video == null) {
      video = youtubeService.getVideo(videoId);
      cacheAll.put(videoId, video);
    } else {
      System.out.println("Retrieved video '" + videoId + "' from cache.");
    }
    return video;
  }

  public void reset() {
    cachePopular.clear();
    cacheAll.clear();
  }
}
```

4) **downloader/YouTubeDownloader.java:**  媒体下载应用

```java
public class YouTubeDownloader {
    private ThirdPartyYouTubeLib api;

    public YouTubeDownloader(ThirdPartyYouTubeLib api) {
        this.api = api;
    }

    public void renderVideoPage(String videoId) {
        Video video = api.getVideo(videoId);
        System.out.println("\n-------------------------------");
        System.out.println("Video page (imagine fancy HTML)");
        System.out.println("ID: " + video.id);
        System.out.println("Title: " + video.title);
        System.out.println("Video: " + video.data);
        System.out.println("-------------------------------\n");
    }

    public void renderPopularVideos() {
        HashMap<String, Video> list = api.popularVideos();
        System.out.println("\n-------------------------------");
        System.out.println("Most popular videos on YouTube (imagine fancy HTML)");
        for (Video video : list.values()) {
            System.out.println("ID: " + video.id + " / Title: " + video.title);
        }
        System.out.println("-------------------------------\n");
    }
}
```

5) Demo测试

```java
public class Demo {
  public static void main(String[] args) {
    YouTubeDownloader naiveDownloader = new YouTubeDownloader(new ThirdPartyYouTubeClass());
    YouTubeDownloader smartDownloader = new YouTubeDownloader(new YouTubeCacheProxy());

    long naive = test(naiveDownloader);
    long smart = test(smartDownloader);
    System.out.print("Time saved by caching proxy: " + (naive - smart) + "ms");
  }

  private static long test(YouTubeDownloader downloader) {
    long startTime = System.currentTimeMillis();
    // User behavior in our app:
    downloader.renderPopularVideos();
    downloader.renderVideoPage("catzzzzzzzzz");
    downloader.renderPopularVideos();
    downloader.renderVideoPage("dancesvideoo");
    // Users might visit the same page quite often.
    downloader.renderVideoPage("catzzzzzzzzz");
    downloader.renderVideoPage("someothervid");

    long estimatedTime = System.currentTimeMillis() - startTime;
    System.out.print("Time elapsed: " + estimatedTime + "ms\n");
    return estimatedTime;
  }
}
```

