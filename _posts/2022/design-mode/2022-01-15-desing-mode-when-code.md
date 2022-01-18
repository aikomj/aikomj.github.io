---
layout: post
title: 实战！工作中常用到哪些设计模式
category: design-mode
tags: [design-mode]
keywords: design-mode
excerpt: 策略模式Strategy,责任链模式chain of responsiblity,模版方法模式template method,观察者模式Observer，工厂模式factory,单例模式singleton
lock: noneed
---

以下文章来源于捡田螺的小男孩。

平时我们写代码，多数情况都是流水线式的写代码，基本就可以实现业务逻辑了。**如何在写代码中找乐趣？**我觉得最好的方式是：**使用设计模式优化自己的业务代码**，今天来聊聊，我工作中，都用过哪些设计模式：

![](/assets/images/2022/design-mode-used-on-work.jpg)

## 1、策略模式

### 业务场景

假设有这样的业务场景，大数据系统把文件推送过来，根据不同类型采用不同的解析方式。多数的小伙伴会写出以下代码：

```java
if(type == 'A'){
  // 按照A模式解析
}else if(type == 'B'){
	// 按照B模式解析
}else{
	// 按照默认格式解析
}
```

这样写代码会存在什么问题呢？

- 如果if分支变多，这里的代码就会变得**臃肿,难以维护，可读性低**
- 如果你要接入一种新的解析类型，只能在**原代码上修改**

说得专业点，以下代码，违背了面向对象编程的**开闭原则**和**单一原则**

- **开闭原则**（对于扩展是开放的，对于修改是封闭的）：增加或删除某个逻辑，都需要修改到原代码
- **单一原则**（规定一个类应该只有一个发生变化的原因）：修改任何类型的分支逻辑代码，都需要改动到当前类的代码

像这种多个if else 分支，我们可以用**策略模式**来优化

### 策略模式定义

策略模式定义了算法族，分别封装起来，让它们之间可以相互替换，此模式让算法的变化独立于使用算法的客户。我们来打个比喻：

> 假设你跟不同性格类型的小姐姐约会，要不同的策略，有的请电影比较好，有的则去吃小吃效果不错，有的去逛街买买最合适。当然，目的都是为了得到小姐姐的芳心，请看电影、吃小吃、逛街就是不同的策略

策略模式针对一组算法，将每一个算法封装到具有共同接口的独立类中，从而使得它们可以相互替换。

### 策略模式使用

怎么使用，这样实现：

- 一个接口或抽象类，里面有两个方法，一个方法匹配类型，一个可替换的逻辑实现方法
- 不同策略的差异化实现
- 使用策略模式

> 第一步：一个接口，两个方法

```java
public interface IFileStrategy{
  // 属于哪种文件解析类型
  FileTypeResolveEnum gainFileType();
  
  // 封装的公用算法（具体的解析方法）
  void resolve(Object objectparam);
}
```

> 第二步：不同策略的差异化实现

A类型策略具体实现

```java
@Component
public class AFileResolve implements IFileStrategy{
  @Override
  public FileTypeResolveEnum gainFileType(){
    return FileTypeResolveEnum.File_A_RESOLVE;
  }
  
  @Overide
  public void resolve(Object objectparam){
    logger.info("A类型解析文件，参数{}",objectparam);
    // A类型解析具体逻辑
  }
}
```

B类型策略具体实现

```java
@Component
public class BFileResolve implements IFileStrategy{
  @Override
  public FileTypeResolveEnum gainFileType(){
    return FileTypeResolveEnum.File_B_RESOLVE;
  }
  
  @Overide
  public void resolve(Object objectparam){
    logger.info("B类型解析文件，参数{}",objectparam);
    // B类型解析具体逻辑
  }
}
```

默认类型策略具体实现

```java
@Component
public class DefaultFileResolve implements IFileStrategy{
  @Override
  public FileTypeResolveEnum gainFileType(){
    return FileTypeResolveEnum.File_DEFAULT_RESOLVE;
  }
  
  @Overide
  public void resolve(Object objectparam){
    logger.info("默认类型解析文件，参数{}",objectparam);
    // 默认类型解析具体逻辑
  }
}
```

> 第三步：使用策略模式

我们借助`spring`的生命周期，使用`ApplicationContextAware`接口，把具体实现的策略，初始化到map里面，然后对外提供`resolveFile`方法即可。

```java
@Component
public class StrategyUseService implements ApplicationContextAware{
  private Map<FileTypeResolveEnum,IFileStrategy> iFileStrategyMap = new ConcurrentHashMap<>();
  
  public void resolveFile(FileTypeResolveEnum fileTypeResolveEnum,Object objectParam){
    IFileStrategy ifileStrategy = iFileStrategyMap.get(fileTypeResolveEnum);
    if(ifileStrategy!=null){
      ifileStrategy.resolve(objectParam);
    }
  }
  
  // 把不同策略放到map
  @Override
  public void setApplicationContext(ApplicationContext applicationContext) throws BeansException{
   Map<String,IFileStrategy> tempMap =  applicationContext.getBeansOfType(IFileStrategy.class);
    tempMap.values().foreach(strategyService -> iFileStrategyMap.put(strategyService.gainFileType,strategyService));
  }
}
```



## 2、责任链模式

### 业务场景

我们来看一个常见的业务场景，下订单，它的基本逻辑，一般有参数非空校验、安全校验、黑名单校验、规则拦截等等。很多小伙伴会使用异常来实现：

```java
public class Order{
  public void checkNullParam(Object param){
    // 参数非空校验
    throw new RuntimeException("xxx");
  }
  public void checkSecurity(){
    // 安全校验
    throw new RuntimeException("xxx");
  }
  public void checkBackList(){
    // 黑名单校验
    throw new RuntimeException("xxx");
  }
  public void checkRule(){
    // 规则拦截
    throw new RuntimeException("xx");
  }
  
  public static void main(String[] args){
    Order order = new Order();
    try{
      order.checkNullParam();
      order.checkSecurity();
      order.checkBackList();
      order.checkRule();
      System.out.println("order sucesss");
    }catch(Exception e){
      System.out.println("order fail");
    }
  }
}
```

如何优化这段代码，可以考虑**责任链模式**

### 责任链模式定义

当你想要让**一个以上的对象**有机会能够处理某个请求的时候，就使用**责任链模式**

> 责任链模式为请求创建了一个接受者对象的链，执行链上有多个对象节点，每个对象节点都有机会（条件匹配）处理请求事务，如果某个对象节点处理完了，就可以根据实际业务需求传递给下一个节点继续处理或者繁华处理完毕。

责任链模式实际上是一种处理请求的模式，它让多个处理器节点都有机会处理该请求，直到某个处理成功为止，它把请求串成链，然后让请求在链上传递，如下图：

<img src="/assets/images/2022/chain.jpg" style="zoom:67%;" />

### 责任链模式使用

怎么使用？三步

1. 一个接口或抽象类
2. 每个对象差异化处理
3. 对象链（数组）初始化（连起来）

> 一个接口或者抽象类

它需要：

- 有一个指向责任下一个对象的属性
- 一个设置下一个对象的set方法
- 给子类对象差异化实现的方法

```java
public abstract class AbstractHandler {
  // 责任链的下一个对象
  private AbstractHandler nextHandler;
  
  // 设置下一个对象
  public void setNextHandler(AbstractHandler nextHandler){
    this.nextHandler = nextHandler;
  }
  
  // 具体参数拦截逻辑，给子类去实现
  public void filter(Request request,Response response){
    doFilter(request,response);
    if(getNextHandler() !=null){
      getNextHandler().filter(request,response);
    }
  }
  
  public AbstractHandler getNextHandler(){
    return nextHandler;
  }
  
  abstract void doFilter(Request filterRequest,Response response);
}
```

> 每个对象差异化处理

责任链上，每个对象的**差异化**处理，如本小节的业务场景，就有参数校验对象，安全校验对象，黑名单校验对象，规则拦截对象

```java
```

> 对象链连起来与使用

```java

```

运行结果如下：

```sh
非空参数检查
安全调用校验
校验黑名单
check rule
```



## 3、模版方式模式



## 4、观察者模式



## 5、工厂模式



## 6、单例模式



