---
layout: post
title: 代理模式
category: java-design
tags: [java-design]
keywords: java
excerpt: 静态代理和动态代理
lock: noneed
---

## 1 、静态代理

租房子的例子：

 ![](/assets/images/2020/java/proxy-example.gif)

 

静态代理，角色分析：

- 抽象角色：一般会使用接口和抽象类实现
- 真实的角色：被代理的角色
- 代理角色：代理真实角色，代理后，一般会做一些附属操作
- 客户：访问代理的人，

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

> 静态代理的优点

- **可以使真实角色的操作更加纯粹，不用关注公共业务**
- 公共业务交给代理角色，实现业务分工，降低耦合性
- 公共业务发生扩展的时候，方便集中管理

缺点

- 一个真实角色会产生一个代理角色，代码量翻倍，开发效率变低！



再来一个demo，

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

相关知识点，AOP：

![](/assets/images/2020/java/mode-aop.gif)



## 2、动态代理

- 动态代理和静态代理的角色一样

- 代理类是动态生成的

- 动态代理分为两大类：基于接口的动态代理 + 基于类的动态代理

  - 基于接口--使用JDK动态代理实现

  - 基于类--- cglib实现

  - java字节码实现--javassist

    > 百度百科：
    >
    > Javassist是一个[开源](https://baike.baidu.com/item/开源/20720669)的分析、编辑和创建Java[字节码](https://baike.baidu.com/item/字节码)的类库。是由[东京工业大学](https://baike.baidu.com/item/东京工业大学/2567295)的数学和计算机科学系的 Shigeru Chiba （千叶 滋）所创建的。它已加入了开放源代码[JBoss](https://baike.baidu.com/item/JBoss/5611767) [应用服务器](https://baike.baidu.com/item/应用服务器/4971773)项目，通过使用Javassist对字节码操作为JBoss实现动态"AOP"框架。
    >
    > 关于java字节码的处理，有很多工具，如bcel，[asm](https://baike.baidu.com/item/asm)。不过这些都需要直接跟[虚拟机](https://baike.baidu.com/item/虚拟机)指令打交道。如果你不想了解虚拟机指令，
    >
    > 可以采用javassist。javassist是[jboss](https://baike.baidu.com/item/jboss)的一个子项目，其主要的优点，在于简单，而且快速。直接使用java编码的形式，而不需要了解虚拟机指令，就能动态改变类的结构，或者动态生成类。

需要了解反射相关的两个类：Proxy、InvocationHandler

![](/assets/images/2020/java/invocation-handler.gif)

![](/assets/images/2020/java/invocation-handler3.gif)

上栗子，改造上面租房的代码：

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
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		// 动态代理的本质，就是使用反射机制实现
		Object result = method.invoke(rent,args);
		seeHouse();
		signRentContract();
		fee();
		return result;
	}

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

客户Client3

```java
public class Client3 {
	public static void main(String[] args) {
		// 真实角色：房东
		Host host = new Host();

		// 代理角色： 没有，动态生成
		HostInvocationHandler handler = new HostInvocationHandler();
		handler.setRent(host);
		Rent proxy =(Rent) handler.getProxy();
		proxy.rent();
	}
}
```

运行结果：

> 房东要出租房子。。。
>
> 中介带你看房子
>
> 签租赁合同
>
> 收中介费



## 3、工具类写法

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
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
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

运行结果

> 【Debug】使用了add方法
> 新增了一个用户



动态代理的好处：

- 一个动态代理类代理的是一个接口，一般对应一类业务，而静态代理类代理的是一个真实的角色对象。
- 一个动态可以代理多个类，只要是实现同一个接口。