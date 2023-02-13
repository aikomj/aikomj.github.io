---
layout: post
title: 实战！工作中常用到哪些设计模式
category: design-mode
tags: [design-mode]
keywords: design-mode
excerpt: 策略模式Strategy,责任链模式chain of responsiblity,模版方法模式template method,观察者模式Observer，工厂模式factory,单例模式singleton
lock: noneed
---

平时我们写代码，多数情况都是流水线式的写代码，基本就可以实现业务逻辑了。**如何在写代码中找乐趣？**我觉得最好的方式是：**使用设计模式优化自己的业务代码**，今天来聊聊，我工作中，都用过哪些设计模式：

![](/assets/images/2022/design-mode/design-mode-used-on-work.jpg)

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

<img src="/assets/images/2022/design-mode/chain.jpg" style="zoom:67%;" />

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
// 参数校验对象
@Component
@Order(1) // 顺序排第1，最先校验
public class CheckParamHandler extends AbstractHandler {
  @Override
  public void doFilter(Request request,Rsponse response){
    System.out.println("非空参数检查");
  }
}

// 安全校验对象
@Component
@Order(2)
public class CheckSecurityHandler extends AbstractHandler {
  @Override
  public void doFilter(Request request,Rsponse response){
    // invoke Security check
    System.out.println("安全调用校验");
  }
}

// 黑名单校验对象
@Component
@Order(3)
public class CheckBlackHandler extends AbstractHandler {
  @Override
  public void doFilter(Request request,Rsponse response){
    // invoke black list check
    System.out.println("校验黑名单");
  }
}

// 规则拦截对象
@Component
@Order(4)
public class CheckRuleHandler extends AbstractHandler {
  @Override
  public void doFilter(Request request,Rsponse response){
    // check rule
    System.out.println("check rule");
  }
}
```

> 对象链连起来(初始化)与使用

```java
@Component
public class ChainPatternDemo{
  // 自动注入各个责任链的对象
  @Autowired
  private List<AbstractHandler> abstractHandleList;
  
  private AbstractHandler abstractHandler;
  
  // spring 注入后自动执行，责任链的对象连接起来
  @PostConstract
  public void initalizeChainFilter(){
    for(int i=0;i<abstractHandleList.size();i++){
      if(i==0){
        abstractHandler = abstractHandleList.get(0);
      }else{
        AbstractHandler currentHandler = abstractHandlerList.get(i-1);
        AbstractHandler nextHandler = abstractHandlerList.get(i);
        currentHandler.setNextHandler(nextHandler);
      }
    }
  }
  
  // 直接调用这个方法使用
  public Response exec(Request request,Response response){
    abstractHandler.filter(request,response);
    return response;
  }
  
  public AbstractHandler getAbstractHandler(){
    return abstractHandler;
  }
  
  public void setAbstractHandler(AbstractHandler abstractHandler){
    this.abstractHandler = abstractHandler;
  }
}
```

运行结果如下：

```sh
非空参数检查
安全调用校验
校验黑名单
check rule
```

## 3、模板方式模式

### 业务场景

假设我们有这么一个业务场景：内部系统不同商户，调用我们系统接口，去跟外部第三方系统交互http方式，流程大概如下：

![](/assets/images/2022/design-mode/template-method.jpg)

这里，有的商户可能是走代理出去的，有的是走直连，假设当前有A,B商户接入，有小伙伴可能会这么写，伪代码如下：

```java
// 商户A处理
CompanyAHandler implements RequestHandler {
     Resp hander(req){
   //查询商户信息
   queryMerchantInfo();
   //加签
   signature();
   //http请求（A商户假设走的是代理）
   httpRequestbyProxy()
   //验签
   verify();
   }
}

// 商户B处理
CompanyBHandler implements RequestHandler {
     Resp hander(req){
   //查询商户信息
   queryMerchantInfo();
   //加签
   signature();
   //http请求（B商户走直连）
   httpRequestbyDirect()
   //验签
   verify();
   }
}
```

如果有一个C商户接入，你需要再实现一套这样的代码，显然，这样代码就**重复了，一些通用的方法，在每个子类都重新写了一遍**。

如何优化？使用**模板方法模式**

### 模板方法模式定义

定义一个操作中的算法的骨架流程，将一些步骤延迟到子类中实现，使得子类可以不改变一个算法的结构的情况下，可以重定义该算法中的某些特定步骤。

它的核心思想是：定义一个操作的一系列步骤，某些暂时不确定下来的步骤，留给子类去实现吧，这样不同的子类就可以定义出不同的步骤。

### 模板方法使用

1. 一个抽象类，定义骨架流程，抽象方法放一起；
2. 确定共同方法步骤，放到抽象类中；
3. 不确定步骤，给子类去差异化实现；

> 一个抽象类，定义骨架流程

```java
/**
 * 抽象类定义骨架流程（查询商户信息，加签，http请求，验签）
 */
abstract class AbstractMerchantService  { 

      //查询商户信息
      abstract queryMerchantInfo();
      //加签
      abstract signature();
      //http 请求
      abstract httpRequest();
       // 验签
       abstract verifySinature();
 
}
```

> 确定共同方法步骤，放到抽象类中

```java
abstract class AbstractMerchantService  { 

     //模板方法流程
     Resp handlerTempPlate(req){
           //查询商户信息
           queryMerchantInfo();
           //加签
           signature();
           //http 请求
           httpRequest();
           // 验签
           verifySinature();
     }
      // Http是否走代理（提供给子类实现）
      abstract boolean isRequestByProxy();
}
```

> 不确定步骤，给子类去差异化实现

因为是否走代理流程是**不确定**的，所以给子类去实现。

商户A的请求实现：

```java
CompanyAServiceImpl extends AbstractMerchantService{
    Resp hander(req){
      return handlerTempPlate(req);
    }
    //走http代理的
    boolean isRequestByProxy(){
       return true;
    }
```

商户B的请求实现

```java
CompanyBServiceImpl extends AbstractMerchantService{
    Resp hander(req){
      return handlerTempPlate(req);
    }
    //公司B是不走代理的
    boolean isRequestByProxy(){
       return false;
    }
```

## 4、观察者模式

### 业务场景

登录注册的场景，用户注册成功，就给用户发一条消息，发个邮件，因此代码经常如下：

```java
void register(User user){
  // 插入注册记录
  insertRegister(user);
  // 发消息
  sendMessage(user);
  // 发邮件
  sendEmail(user);
}
```

如果产品加需求： 用户注册成功，再给用户发一条短信通知，于是你又修改register方法，这就违法了**开闭原则**

如果调**发短信的接口失败了**，又影响到用户注册了，所以的加个异步方法发短信

实际上，我们可以使用观察者模式进行优化。

### 观察者模式定义

观察者模式定义对象间的一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象得到通知并完成业务的更新。它的主要成员是：

- **被观察者**：也称目标对象，状态发生变化时，将通知所有的观察者；

- **观察者**：接受被观察者的状态变化通知，执行预定义的业务。

**使用场景**：完成某件事情后，异步通知场景，如登录成功，发个短信消息。

类比mq消息中间件的发布订阅模式，观察者模式是一种松耦合的关系，MQ则完全不存在耦合关系。

### 观察者模式使用

使用起来还是比较简单的：

1. 一个被观察者的类Observerable
2. 多个观察者Observer
3. 观察者的差异化实现
4. 经典观察者模式封装：EventBus

> 一个被观察者的类Observerable

```java
public class Observerable{
  private List<Observer> observers = ArrayList<Observer>();
  private int state;
  
  public int getState() {
      return state;
   }
 
   public void setState(int state) {
      notifyAllObservers(state);
   }
  
 	//添加观察者
   public void addServer(Observer observer){
      observers.add(observer);      
   }
   
   //移除观察者
   public void removeServer(Observer observer){
      observers.remove(observer);      
   }
   
  //通知
   public void notifyAllObservers(int state){
      if(state!=1){
         System.out.println(“不是通知的状态”);
         return ;
      }
      for (Observer observer : observers) {
         observer.doEvent();
      }
   }  
}
```

> 观察者的差异化实现

```java
 //观察者接口
interface Observer {  
    void doEvent();  
}  
//Im消息
IMMessageObserver implements Observer{
    void doEvent（）{
       System.out.println("发送IM消息");
    }
}
//手机短信
MobileNoObserver implements Observer{
    void doEvent（）{
       System.out.println("发送短信消息");
    }
}
//EmailNo
EmailObserver implements Observer{
    void doEvent（）{
       System.out.println("发送email消息");
    }
}
```

在doEvent方法上增加@Async注解，异步执行。

> EventBus实战

Guava EventBus帮我们封装好了观察者模式，它提供了一套基于注解的事件总线，下面来简单实战一下

1、声明一个EventBusCenter类，相当于上面的被观察者Observerable

```java
public class EventBusCenter {
  private static EventBus eventBus = new EventBus();

  private EventBusCenter() {
  }

  public static EventBus getInstance() {
    return eventBus;
  }
  //添加观察者
  public static void register(Object obj) {
    eventBus.register(obj);
  }
  //移除观察者
  public static void unregister(Object obj) {
    eventBus.unregister(obj);
  }
  //把消息推给观察者
  public static void post(Object obj) {
    eventBus.post(obj);
  }
}
```

2、声明观察者EventListener

```java
public class EventListener {
  @Subscribe // 加了订阅，标记这个方法上事件处理方法
  public void handle(NotifyEvent notifyEvent){
    System.out.println("发送IM消息" + notifyEvent.getImNo());
    System.out.println("发送短信消息" + notifyEvent.getMobileNo());
    System.out.println("发送Email消息" + notifyEvent.getEmailNo());
  }
}

// 通知事件类
public class NotifyEvent {
  private String mobileNo;
  private String emailNo;
  private String imNo;
  
  public NotifyEvent(String mobileNo, String emailNo, String imNo) {
    this.mobileNo = mobileNo;
    this.emailNo = emailNo;
    this.imNo = imNo;
  }
}
```

3、demo测试

```java
public class EventBusDemoTest{
  public static void mian(String[] args){
    EventListener eventListener = new EventListner();
    EventBusCenter.register(eventListener);
    EventBusCenter.post(new NotifyEvent("13334344573","123@qq.com","666"));
  }
}
```

运行结果

```sh
发送IM消息666
发送短信消息13372817283
发送Email消息123@qq.com
```

思想上跟mq一致，消费者订阅某个topic，发送者把消息发送到broker，消费者监听到有消息则拉下消息进行消费，这里则是通过消息总线注册监听者（观察者），总线主动发送消息，监听者则触发执行handle方法。



## 5、工厂模式

### 业务场景

工厂模式一般配合策略模式一起使用，可以优化大量的`if...else...`或`switch...case...`条件语句

取上面的文件解析类型，创建不同的解析对象

```java
 IFileStrategy getFileStrategy(FileTypeResolveEnum fileType){
     IFileStrategy  fileStrategy ;
     if(fileType=FileTypeResolveEnum.File_A_RESOLVE){
       fileStrategy = new AFileResolve();
     }else if(fileType=FileTypeResolveEnum.File_A_RESOLV){
       fileStrategy = new BFileResolve();
     }else{
       fileStrategy = new DefaultFileResolve();
     }
     return fileStrategy;
 }
```

其实这就是**工厂模式**，定义一个创建对象的接口，让其子类自己决定实例化哪一个工厂类，工厂模式使其创建过程延迟到子类进行。

策略模式的例子，没有使用上一段代码，而是借助spring的特性，搞了一个工厂模式，哈哈，小伙伴们可以回去那个例子细品一下，我把代码再搬下来，小伙伴们再品一下吧：

```java
/**
 *  @author 公众号：捡田螺的小男孩
 */
@Component
public class StrategyUseService implements ApplicationContextAware{

    private Map<FileTypeResolveEnum, IFileStrategy> iFileStrategyMap = new ConcurrentHashMap<>();

    //把所有的文件类型解析的对象，放到map，需要使用时，信手拈来即可。这就是工厂模式的一种体现啦
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        Map<String, IFileStrategy> tmepMap = applicationContext.getBeansOfType(IFileStrategy.class);
        tmepMap.values().forEach(strategyService -> iFileStrategyMap.put(strategyService.gainFileType(), strategyService));
    }
}
```

### 使用工厂模式

定义工厂模式也是比较简单的:

- 一个工厂接口，提供一个创建不同对象的方法。
- 其子类实现工厂接口，构造不同对象
- 使用工厂模式

> 一个工厂接口

```java
interface IFileResolveFactory{
   void resolve();
}
```

> 不同子类实现工厂接口

```java
class AFileResolve implements IFileResolveFactory{
   void resolve(){
      System.out.println("文件A类型解析");
   }
}

class BFileResolve implements IFileResolveFactory{
   void resolve(){
      System.out.println("文件B类型解析");
   }
}

class DefaultFileResolve implements IFileResolveFactory{
   void resolve(){
      System.out.println("默认文件类型解析");
   }
}
```

> 使用工厂模式

```java
//构造不同的工厂对象
IFileResolveFactory fileResolveFactory;
if(fileType=“A”){
    fileResolveFactory = new AFileResolve();
}else if(fileType=“B”){
    fileResolveFactory = new BFileResolve();
 }else{
    fileResolveFactory = new DefaultFileResolve();
}

fileResolveFactory.resolve();
```

一般情况下，对于工厂模式，你不会看到以上的代码。工厂模式会跟配合其他设计模式如策略模式一起出现的

## 6、单例模式

### 业务场景

单例模式，**保证一个类仅有一个实例**，并提供一个访问它的全局访问点。I/O连接、数据库连接，一般都使用单例模式实现的，Windows里面的Task Manager（任务管理器）也是很典型的单例模式。

### 经典写法

> 饿汉式

```java
public class EHanSingleton {
  private static EHanSingleton instance = new EHanSingleton();
  private EHanSingleton(){}
  
  public statice EHanSingleton getInstance(){
    return instance;
  }
}
```

程序启动时就创建好对象，没有线程安全问题。

> DCL双重校验锁

```java
public class DoubleCheckSingleton {
  // 禁止cpu指令重排
   private volatile static DoubleCheckSingleton instance;

   private DoubleCheckSingleton() { }
   
   public static DoubleCheckSingleton getInstance(){
       if (instance == null) { // 减少大量的锁竞争
           synchronized (DoubleCheckSingleton.class) {
               if (instance == null) {
                   instance = new DoubleCheckSingleton();
               }
           }
       }
       return instance;
   }
}
```

在synchronized关键字内外都加了一层  `if`条件判断，这样既保证了线程安全，又比直接上锁提高了执行效率，还节省了内存空间。

> 静态内部类

```java
public class InnerClassSingleton{
  private InnerClassSingleton(){}
  
  private static class InnerClassSingletonHolder{
    private static final InnerClassSingleton INSTANCE = new InnerClassSingleton();
  }
  
  public static final InnerClassSingleton getInstance(){
    return InnerClassSingletonHolder.INSTANCE;
  }
}
```

静态内部类的实现方式，效果有点类似双重校验锁。但这种方式只适用于静态域场景，双重校验锁方式可在实例域需要延迟初始化时使用。

### 枚举

枚举就是经典的单例

```java
public enum SingletonEnum {
    INSTANCE;
    public SingletonEnum getInstance(){
        return INSTANCE;
    }
}
```

枚举实现的单例，代码简洁清晰。并且它还自动支持序列化机制，绝对防止多次实例化。

反射可以破坏单例模式，但不能破坏枚举创建多个实例，是因为JDK内部做了异常抛出。
