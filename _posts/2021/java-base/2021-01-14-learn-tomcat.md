---
layout: post
title: Tomcat的架构分析，进入源码哦
category: java
tags: [java]
keywords: java
excerpt: 说说Tomcat的启动过程，生命周期
lock: noneed
---

## 前言

感谢鸭血粉丝的分享，作为一个 Java 资深开发人员，对 Tomcat 那是再熟悉不过了，bin目录、conf目录、webapps目录，对这些目录熟悉的简直不能再熟悉了。一言不合就是一个shutdown.sh，或者来个shutdown.bat，但是你知道你的启动startup.bat，和startup.sh他们的启动过程是什么过程么？接下来我们就开始进入分析吧。

## 1、Tomcat的架构

![](\assets\images\2021\javabase\tomcat-arch.jpg)

- Server：整个服务器。
- Service：具体的服务。
- Connector：提供Socket和request，response的连接。
- Container：用于封装和管理Servlet，以及具体的处理请求。（想起docker的容器，两者其实没有关系）

这个图就把里面的包含关系说的是明明白白了，为什么这么说呢？因为一个Server中可以存在多个服务，也就是多个Service，而一个Service中可以存在多个Connector，但是只能存在一个Container，是不是就非常的清晰了呢？

## 2、Tomcat的启动过程

接下来我们就来看看源码里面的启动过程吧，Bootstrap类中的启动过程。这个类的位置是在tomcat的catalina的包里面，大家看一下主方法也就是所谓的main方法，代码如下

```java
public static void main(String[] args) {
        //对象初始化
        if (daemon == null) {
            Bootstrap bootstrap = new Bootstrap();
            try {
                bootstrap.init();
            } catch (Throwable var3) {
                handleThrowable(var3);
                var3.printStackTrace();
                return;
            }
            daemon = bootstrap;
        } else {
            Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
        }

        try {
            String command = "start";
            if (args.length > 0) {
                command = args[args.length - 1];
            }

            if (command.equals("startd")) {
                args[args.length - 1] = "start";
                //加载
                daemon.load(args);
                //启动
                daemon.start();
            } else if (command.equals("stopd")) {
                args[args.length - 1] = "stop";
                //停止
                daemon.stop();
            } else if (command.equals("start")) {
                daemon.setAwait(true);
                //加载并且启动
                daemon.load(args);
                daemon.start();
                if (null == daemon.getServer()) {
                    System.exit(1);
                }
            } else if (command.equals("stop")) {
                daemon.stopServer(args);
            } else if (command.equals("configtest")) {
                daemon.load(args);
                if (null == daemon.getServer()) {
                    System.exit(1);
                }
                System.exit(0);
            } else {
                log.warn("Bootstrap: command \"" + command + "\" does not exist.");
            }
        } catch (Throwable var4) {
            Throwable t = var4;
            if (var4 instanceof InvocationTargetException && var4.getCause() != null) {
                t = var4.getCause();
            }
            handleThrowable(t);
            t.printStackTrace();
            System.exit(1);
        }
    }
```

main方法里面的存在也很简单，先进行init的操作，然后再执行start，也就是说，启动过程中，首先要进行初始化，然后接下来再进行启动，最后阶段在来个stop,这样才算完整嘛。

- load方法：其实说白了load方法就是根据server.xml文件创建Server并且调用Server的init方法进行初始化。
- start方法：start方法很直白，启动服务器。
- stop方法：stop方法同样，停止服务器。

在这里的start方法和stop方法调用的分别就是调用了Server内部的start和stop方法，而这三个方法都是按照图中的层级结构来的，先从Server的load，start，stop，然后Server的start再调用Service的start，而Service的start调用的就是Connector和Container的start方法了，这从外到内的启动，就可以把Tomcat完整的启动起来了。

我们在接下来就继续从外到内的启动开始分析一波。

### Catalina启动过程

上面的启动入口我们已经成功找到了，那么我们就继续来，对象初始化完成后，执行了init的方法

```java
Bootstrap bootstrap = new Bootstrap();
try {
  bootstrap.init();
} catch (Throwable var3) {
```

就是上面的这个，如果参数为空了，那么就开始调用start了

```java
public void start() throws Exception {
  if (this.catalinaDaemon == null) {
    this.init();
  }

  Method method = this.catalinaDaemon.getClass().getMethod("start", (Class[])null);
  method.invoke(this.catalinaDaemon, (Object[])null);
}
```

上面的start方法就是直接使用invoke的方法映射到了catalinaDaemon，也就是到了Catalina的start的方法上，而这个Catalina的start无非也就是调用了同样的方法，setAwait方法，load方法，start方法，

- setAwait方法：用于设置Server启动完成时，是否进入等待，如果是true，那就等待，如果不是false，那就不进入等待。
- load方法：创建并且初始化Server，
- start方法：同样是启动服务器

同样的setAwait方法比较少，阿粉就不给大家看了，无非就是个判断，而load方法一定得看，

```java
if (!this.loaded) {
  this.loaded = true;
  long t1 = System.nanoTime();
  try {
    inputSource.setByteStream((InputStream)inputStream);
    digester.push(this);
    digester.parse(inputSource);
    break label242;
  } catch (SAXParseException var28) {
    log.warn("Catalina.start using " + this.getConfigFile() + ": " + var28.getMessage());
    return;
  } catch (Exception var29) {
    log.warn("Catalina.start using " + this.getConfigFile() + ": ", var29);
  }
} finally {
  if (inputStream != null) {
    try {
      ((InputStream)inputStream).close();
    } catch (IOException var23) {
      ;
    }
  }
}
return;
}

try {
  //此处同样调用的Server的init方法，
  this.getServer().init();
} catch (LifecycleException var24) {
  if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
    throw new Error(var24);
  }
  log.error("Catalina.start", var24);
}

long t2 = System.nanoTime();
if (log.isInfoEnabled()) {
  log.info("Initialization processed in " + (t2 - t1) / 1000000L + " ms");
}

}
```

而从这里就开始进入下一步了，Server的启动过程，因为从Catalina里面已经找到了getServer的初始化方法，接下来就是要进行Server的初始化，然后加载，然后启动的过程了。

### Server的启动过程

Server是Tomcat里面的接口，而不是类，那么我们就只能去找实现它的子类来于是就找到了StandardServer extends LifecycleMBeanBase implements Server。

LifecycleMBeanBase

```java
public abstract class LifecycleMBeanBase extends LifecycleBase implements JmxEnabled {
```

LifecycleBase

```java
public abstract class LifecycleBase implements Lifecycle {
```

找到抽象类LifecycleBase里的方法

```java
public abstract class LifecycleBase {
    public final synchronized void init() throws LifecycleException{
        if(!this.state.equals(LifecycleState.NEW)){
          this.invalidTransition("before_init");
        }
      	try{
          this.setStateInternal(LifecycleState.INITIALIZING, (Object)null,false);
          this.initInternal();
          this.setStateInternal(LifecycleState.INITIALIZED, (Object)null,false);
        }catch(Throwable var2){
          ExceptionUtils.handleThrowable(var2);
          this.setStateInternal(LifecycleState.FAILED, (Object)null,false);
          throw new LifecycleException(...);
        }
    }
  
  public final synchronized void start() throws LifecycleException {
    if(!LifecycleState.STARTING_PREP.equals(this.state) && !LifecycleState.STARTING.equals(this.state)){
      if(this.state.equals(LifecycleState.NEW)){
        this.init();
      } else if(this.state.equals(LifecycleState.NEW)){
        this.stop();
      } else if(!this.state.equals(LifecycleState.INITIALIZED && !this.state.equals(LifecycleState.STOPPED)){
        this.invalidTransition("before_start");
      }
                
         try{
           this.setStateInternal(LifecycleState.STARTING_PREP，null,false);
           this.startInternal();
           ...
         }       
    }
  }
}
```

这init方法和start方法又调用了initInternal()和startInternal()，**模板方法，是有自己的子类具体实现**，回到StandardServer自己的init和start方法，

```java
 protected void startInternal() throws LifecycleException {
        this.fireLifecycleEvent("configure_start", (Object)null);
        this.setState(LifecycleState.STARTING);
        this.globalNamingResources.start();
        Object var1 = this.servicesLock;
        synchronized(this.servicesLock) {
            for(int i = 0; i < this.services.length; ++i) {
                this.services[i].start();
            }
        }
    }
```

总得来说就是，StandardServer继承自LifecycleMBeanBase，而LifecycleMBeanBase继承自LifecycleBase，而LifecycleBase类中的模板方法，又让自己的子类去进行具体的实现，但是我们要知道他的Tomcat生命周期中存在这些内容才行。

图中都说了，Server里面有Service，那么一定就有，我们得去找找看，于是阿粉再次去找并且去看它到底是个什么意思。

```java
public void addService(Service service) {
  service.setServer(this);
  Object var2 = this.servicesLock;
  synchronized(this.servicesLock) {
    Service[] results = new Service[this.services.length + 1];
    System.arraycopy(this.services, 0, results, 0, this.services.length);
    results[this.services.length] = service;
    this.services = results;
    if (this.getState().isAvailable()) {
      try {
        service.start();
      } catch (LifecycleException var6) {
        ;
      }
    }

    this.support.firePropertyChange("service", (Object)null, service);
  }
}
```

位置是在Server的接口中出现了增加和删除Service的方法，Server的init方法和start方法循环去调用每个Service的init方法和start方法。接下来我们看看Service的具体实现，找到StandardService:

```java
 protected void initInternal() throws LifecycleException {
        super.initInternal();
        if (this.engine != null) {
            this.engine.init();
        }

        Executor[] arr$ = this.findExecutors();
        int len$ = arr$.length;

        int len$;
        for(len$ = 0; len$ < len$; ++len$) {
            Executor executor = arr$[len$];
            if (executor instanceof JmxEnabled) {
                ((JmxEnabled)executor).setDomain(this.getDomain());
            }

            executor.init();
        }

        this.mapperListener.init();
        Object var11 = this.connectorsLock;
        synchronized(this.connectorsLock) {
            Connector[] arr$ = this.connectors;
            len$ = arr$.length;

            for(int i$ = 0; i$ < len$; ++i$) {
                Connector connector = arr$[i$];

                try {
                    connector.init();
                } catch (Exception var9) {
                    String message = sm.getString("standardService.connector.initFailed", new Object[]{connector});
                    log.error(message, var9);
                    if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
                        throw new LifecycleException(message);
                    }
                }
            }

        }
    }
```

而在方法中主要调用Executor,mapperListener,executor的init方法。connector之前已经有了，而这个mapperListener就是Mapper的监听器，用来监听container容器的变化。

### 总结

由外部到内部的启动过程

![](\assets\images\2021\javabase\tomcat-init-start.jpg)

## 3、Tomcat的生命周期

大家可以随便找一个zip版本的Tomcat，然后直接启动起来，我们来看看是个什么样子的，

```sh
一月 11, 2021 10:16:24 上午 org.apache.coyote.AbstractProtocol init
信息: Initializing ProtocolHandler ["http-bio-8080"]
一月 11, 2021 10:16:24 上午 org.apache.coyote.AbstractProtocol init
信息: Initializing ProtocolHandler ["ajp-bio-8009"]
一月 11, 2021 10:16:24 上午 org.apache.catalina.startup.Catalina load
信息: Initialization processed in 470 ms
一月 11, 2021 10:16:24 上午 org.apache.catalina.core.StandardService startInternal
信息: Starting service Catalina
一月 11, 2021 10:16:24 上午 org.apache.catalina.core.StandardEngine startInternal
信息: Starting Servlet Engine: Apache Tomcat/7.0.88
一月 11, 2021 10:16:24 上午 org.apache.catalina.startup.HostConfig deployDirectory
```

结合上面说的，整个启动流程就是： load，然后start，最后stop

### Lifecycle

上面提到Lifecycle的多个子接口，Tomcat是通过Lifecycle接口统一管理生命周期，所有生命周期的组件都要实现Lifecycle接口，以便提供一致的机制去启动和停止组件。

我们直接到tomcat的catalina的jar包寻找Lifecycle接口，主要包含了4部分内容

- 定义了13个String类型的变量
- 定义了3个管理监听器的方法
- 定义了4个生命周期
- 定义了2个获取当前状态的方法

> 13个String类型的变量

```java
String BEFORE_INIT_EVENT = "before_init";
String AFTER_INIT_EVENT = "after_init";
String START_EVENT = "start";
String BEFORE_START_EVENT = "before_start";
String AFTER_START_EVENT = "after_start";
String STOP_EVENT = "stop";
String BEFORE_STOP_EVENT = "before_stop";
String AFTER_STOP_EVENT = "after_stop";
String AFTER_DESTROY_EVENT = "after_destroy";
String BEFORE_DESTROY_EVENT = "before_destroy";
String PERIODIC_EVENT = "periodic";
String CONFIGURE_START_EVENT = "configure_start";
String CONFIGURE_STOP_EVENT = "configure_stop";
```

这些常量是LifecycleEvent事件的type属性，如初始化前，初始后，启动等，为了表示组件发出时的状态。

> 3个管理监听器的方法

```java
// 添加，查找和删除LifecycleListener类型的监听器
void addLifecycleListener(LifecycleListener var1);

LifecycleListener[] findLifecycleListeners();

void removeLifecycleListener(LifecycleListener var1);
```

> 4个生命周期

```java
// 初始化
void init() throws LifecycleException;
// 启动
void start() throws LifecycleException;
// 停止
void stop() throws LifecycleException;
// 销毁
void destroy() throws LifecycleException;
```

> 2个获取状态的方法

```java
LifecycleState getState();

String getStateName();
```

下面来了解接口Lifecycle的实现类LifecycleBase

### LifecycleBase

LifecycleBase是抽象类，是tomcat中所有组件类的基类,他实现了Lifecycle，但是Tomcat下的很多子类也同样的继承了它，所以他也是非常重要的，

```java
public abstract class LifecycleBase implements Lifecycle {
  private LifecycleSupport lifecycle = new LifecycleSupport(this);
  // 源组件的当前状态,不同状态触发不同事件
  private volatile LifecycleState state;
  public LifecycleBase() {
    this.state = LifecycleState.NEW;
  }
}
```

点击LifecycleSupport类的源码

```java
public final class LifecycleSupport {
    private Lifecycle lifecycle = null;
    private LifecycleListener[] listeners = new LifecycleListener[0];
    private final Object listenersLock = new Object();

    public LifecycleSupport(Lifecycle lifecycle) {
        this.lifecycle = lifecycle;
    }

  // 添加监听器
    public void addLifecycleListener(LifecycleListener listener) {
        Object var2 = this.listenersLock;
        synchronized(this.listenersLock) {
            LifecycleListener[] results = new LifecycleListener[this.listeners.length + 1];

            for(int i = 0; i < this.listeners.length; ++i) {
                results[i] = this.listeners[i];
            }

            results[this.listeners.length] = listener;
            this.listeners = results;
        }
    }

    public LifecycleListener[] findLifecycleListeners() {
        return this.listeners;
    }

  // 执行事件
    public void fireLifecycleEvent(String type, Object data) {
        LifecycleEvent event = new LifecycleEvent(this.lifecycle, type, data);
        LifecycleListener[] interested = this.listeners;

        for(int i = 0; i < interested.length; ++i) {
            interested[i].lifecycleEvent(event);
        }
    }

// 删除监听器  
    public void removeLifecycleListener(LifecycleListener listener) {
        Object var2 = this.listenersLock;
        synchronized(this.listenersLock) {
            int n = -1;

            for(int i = 0; i < this.listeners.length; ++i) {
                if (this.listeners[i] == listener) {
                    n = i;
                    break;
                }
            }

            if (n >= 0) {
                LifecycleListener[] results = new LifecycleListener[this.listeners.length - 1];
                int j = 0;

                for(int i = 0; i < this.listeners.length; ++i) {
                    if (i != n) {
                        results[j++] = this.listeners[i];
                    }
                }

                this.listeners = results;
            }
        }
    }
}
```

LifecycleSupport中定义了一个LifecycleListener数组类型的属性来保存所有的监听器，然后在里面分别定义了添加，删除，查找，执行监听器的方法

LifecycleSupport的生命周期方法

**init方法**

```java
public final synchronized void init() throws LifecycleException {
  //这里表示只有NEW状态下是可以使用的，
  if (!this.state.equals(LifecycleState.NEW)) {
    this.invalidTransition("before_init");
  }
  //在这里通过不同的状态，然后去触发不同的事件，
  try {
    //设置生命周期状态为INITIALIZING
    this.setStateInternal(LifecycleState.INITIALIZING, (Object)null, false);
    //执行方法
    this.initInternal();
    //设置生命周期状态为INITIALIZED
    this.setStateInternal(LifecycleState.INITIALIZED, (Object)null, false);
  } catch (Throwable var2) {
    ExceptionUtils.handleThrowable(var2);
    this.setStateInternal(LifecycleState.FAILED, (Object)null, false);
    throw new LifecycleException(sm.getString("lifecycleBase.initFail", new Object[]{this.toString()}), var2);
  }
}
```

**start方法**

```java
public final synchronized void start() throws LifecycleException {
  //在这里验证生命周期状态，状态是这三种状态的是为不可用状态STARTING_PREP，STARTING，STARTED
  if (!LifecycleState.STARTING_PREP.equals(this.state) && !LifecycleState.STARTING.equals(this.state) && !LifecycleState.STARTED.equals(this.state)) {
    //如果是NEW状态，执行init方法
    if (this.state.equals(LifecycleState.NEW)) {
      this.init();
      //如果是FAILED状态，那么执行stop方法
    } else if (this.state.equals(LifecycleState.FAILED)) {
      this.stop();
      //如果是INITIALIZED状态，那么就会告诉你是个非法的操作
    } else if (!this.state.equals(LifecycleState.INITIALIZED) && !this.state.equals(LifecycleState.STOPPED)) {
      this.invalidTransition("before_start");
    }

    try {
      //设置启动状态为 STARTING_PREP
      this.setStateInternal(LifecycleState.STARTING_PREP, (Object)null, false);
      this.startInternal();
      //这里就非常的严谨，他会在启动之后，继续去看状态是什么，保证启动成功
      if (this.state.equals(LifecycleState.FAILED)) {
        this.stop();
      } else if (!this.state.equals(LifecycleState.STARTING)) {
        this.invalidTransition("after_start");
      } else {
        this.setStateInternal(LifecycleState.STARTED, (Object)null, false);
      }

    } catch (Throwable var2) {
      ExceptionUtils.handleThrowable(var2);
      this.setStateInternal(LifecycleState.FAILED, (Object)null, false);
      throw new LifecycleException(sm.getString("lifecycleBase.startFail", new Object[]{this.toString()}), var2);
    }
  } else {
    if (log.isDebugEnabled()) {
      Exception e = new LifecycleException();
      log.debug(sm.getString("lifecycleBase.alreadyStarted", new Object[]{this.toString()}), e);
    } else if (log.isInfoEnabled()) {
      log.info(sm.getString("lifecycleBase.alreadyStarted", new Object[]{this.toString()}));
    }

  }
}
```

**stop方法**

```java
public final synchronized void stop() throws LifecycleException {
  //同样的，和上面一样，三种状态下不可执行
  if (!LifecycleState.STOPPING_PREP.equals(this.state) && !LifecycleState.STOPPING.equals(this.state) && !LifecycleState.STOPPED.equals(this.state)) {
    //如果是NEW状态，状态直接修改为STOPPED
    if (this.state.equals(LifecycleState.NEW)) {
      this.state = LifecycleState.STOPPED;
    } else {
      //如果不是这2中状态，那么就直接异常
      if (!this.state.equals(LifecycleState.STARTED) && !this.state.equals(LifecycleState.FAILED)) {
        this.invalidTransition("before_stop");
      }
      try {
        if (this.state.equals(LifecycleState.FAILED)) {
          this.fireLifecycleEvent("before_stop", (Object)null);
        } else {
          this.setStateInternal(LifecycleState.STOPPING_PREP, (Object)null, false);
        }
        this.stopInternal();
        if (!this.state.equals(LifecycleState.STOPPING) && !this.state.equals(LifecycleState.FAILED)) {
          this.invalidTransition("after_stop");
        }
        this.setStateInternal(LifecycleState.STOPPED, (Object)null, false);
      } catch (Throwable var5) {
        ...
      } finally {
        ...
      }

    }
  } else {
    ...
  }
}
```

**destroy方法**

```java
public final synchronized void destroy() throws LifecycleException {
  //如果状态是启动失败的，也就是FAILED，那么会直接去调用stop方法，
  if (LifecycleState.FAILED.equals(this.state)) {
    try {
      this.stop();
    } catch (LifecycleException var3) {
      log.warn(sm.getString("lifecycleBase.destroyStopFail", new Object[]{this.toString()}), var3);
    }
  }
  //如果是这两种状态DESTROYING、DESTROYED，那么就不再进行执行了，直接进行return
  if (!LifecycleState.DESTROYING.equals(this.state) && !LifecycleState.DESTROYED.equals(this.state)) {
    if (!this.state.equals(LifecycleState.STOPPED) && !this.state.equals(LifecycleState.FAILED) && !this.state.equals(LifecycleState.NEW) && !this.state.equals(LifecycleState.INITIALIZED)) {
      this.invalidTransition("before_destroy");
    }

    try {
      this.setStateInternal(LifecycleState.DESTROYING, (Object)null, false);
      this.destroyInternal();
      this.setStateInternal(LifecycleState.DESTROYED, (Object)null, false);
    } catch (Throwable var2) {
      ExceptionUtils.handleThrowable(var2);
      this.setStateInternal(LifecycleState.FAILED, (Object)null, false);
      throw new LifecycleException(sm.getString("lifecycleBase.destroyFail", new Object[]{this.toString()}), var2);
    }
  } else {
    ...
      ｝
  }
```

### 总结

13个状态变量，4个生命周期方法

![](\assets\images\2021\javabase\tomcat-life-status.jpg)