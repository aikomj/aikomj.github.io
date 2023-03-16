---
layout: post
title: Java并发编程的10个注意事项
category: java
tags: [juc]
keywords: juc
excerpt: SimpleDateFormat线程不安全，双重检查锁的漏洞，volatile不保证原子性，死锁，没释放锁，HashMap导致内存溢出，使用默认线程池，@Async注解的陷阱，自旋锁浪费CPU资源，ThreadLocal用完没清空导致内存泄漏
lock: noneed
---

<img src="\assets\images\2022\java\juc-1.png" style="zoom:60%;" />

<img src="../../..\assets\images\2022\java\juc-1.png" style="zoom:60%;" />

## 1. SimpleDateFormat线程不安全

在java8之前，我们对时间的格式化处理，一般都是用的`SimpleDateFormat`类实现的。例如：

```java
@Service
public class SimpleDateFormatService {

    public Date time(String time) throws ParseException {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return dateFormat.parse(time);
    }
}
```

如果你真的这样写，是没问题的。

就怕哪天抽风，你觉得dateFormat是一段固定的代码，应该要把它抽取成常量。

于是把代码改成下面的这样：

```java
@Service
public class SimpleDateFormatService {

   private static SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public Date time(String time) throws ParseException {
        return dateFormat.parse(time);
    }
}
```

dateFormat对象被定义成了静态常量，这样就能被所有对象共用。

如果只有一个线程调用time方法，也不会出现问题。

但Serivce类的方法，往往是被Controller类调用的，而Controller类的接口方法，则会被`tomcat`的`线程池`调用。换句话说，可能会出现多个线程调用同一个Controller类的同一个方法，也就是会出现多个线程会同时调用Service层的time方法的情况。

而time方法会调用`SimpleDateFormat`类的`parse`方法：

```java
@Override
public Date parse(String text, ParsePosition pos) {
    ...
    Date parsedDate;
    try {
        parsedDate = calb.establish(calendar).getTime();
        ...
    } catch (IllegalArgumentException e) {
        pos.errorIndex = start;
        pos.index = oldStart;
        return null;
    }
   return parsedDate;
} 
```

该方法会调用`establish`方法：

```java
Calendar establish(Calendar cal) {
    ...
    //1.清空数据
    cal.clear();
    //2.设置时间
    cal.set(...);
    //3.返回
    return cal;
}
```

其中的步骤1、2、3是非原子操作。

但如果cal对象是局部变量还好，坏就坏在parse方法调用establish方法时，传入的calendar是`SimpleDateFormat`类的父类`DateFormat`的成员变量：

```java
public abstract class DateFormat extends Forma {
    ....
    protected Calendar calendar;
    ...
}
```

这样就可能会出现多个线程，同时修改同一个对象即：dateFormat，他的同一个成员变量即：Calendar值的情况。

<mark>这样可能会出现，某个线程设置好了时间，又被其他的线程修改了，从而出现时间错误的情况。</mark>

那么，如何解决这个问题呢？

1. SimpleDateFormat类的对象不要定义成静态的，可以改成方法的局部变量。
2. 使用ThreadLocal保存SimpleDateFormat类的数据。
3. 使用java8的DateTimeFormatter类。<mark>推荐</mark>.

## 2、双重检查锁的漏洞

单例模式的懒汉式使用volatie与双重检查锁

[/icoding-edu/2020/03/07/icoding-note-005.html](/icoding-edu/2020/03/07/icoding-note-005.html)

DCL，多线程下也保证了单例，但是反射可以破坏单例

```java
public class SimpleSingleton4 {

    private static SimpleSingleton4 INSTANCE;

    private SimpleSingleton4() {
    }

    public static SimpleSingleton4 getInstance() {
        if (INSTANCE == null) {
            synchronized (SimpleSingleton4.class) {
                if (INSTANCE == null) {
                    INSTANCE = new SimpleSingleton4();
                }
            }
        }
        return INSTANCE;
    }
}
```

需要在`synchronized`前后两次判断了INSTANCE == null 是否为空

但我要告诉你的是：这段代码有漏洞的。有什么问题？

```java
public static SimpleSingleton4 getInstance() {
    if (INSTANCE == null) {//1
        synchronized (SimpleSingleton4.class) {//2
            if (INSTANCE == null) {//3
                INSTANCE = new SimpleSingleton4();//4
            }
        }
    }
    return INSTANCE;//5
}
```

getInstance方法的这段代码，我是按1、2、3、4、5这种顺序写的，希望也按这个顺序执行。

但是java虚拟机实际上会做一些优化，对一些代码指令进行重排。重排之后的顺序可能就变成了：1、3、2、4、5，这样在多线程的情况下同样会**创建多次实例**。重排之后的代码可能如下：

```java
public static SimpleSingleton4 getInstance() {
    if (INSTANCE == null) {//1
       if (INSTANCE == null) {//3
           synchronized (SimpleSingleton4.class) {//2
                INSTANCE = new SimpleSingleton4();//4
            }
        }
    }
    return INSTANCE;//5
}
```

原来如此，那有什么办法可以解决呢？

**可以在定义INSTANCE是加上`volatile`关键字**。具体代码如下：

```java

public class SimpleSingleton7 {

    private volatile static SimpleSingleton7 INSTANCE;

    private SimpleSingleton7() {
    }

    public static SimpleSingleton7 getInstance() {
        if (INSTANCE == null) {
            synchronized (SimpleSingleton7.class) {
                if (INSTANCE == null) {
                    INSTANCE = new SimpleSingleton7();
                }
            }
        }
        return INSTANCE;
    }
}
```

`volatile`关键字可以保证多个线程的`可见性`，但是不能保证`原子性`。同时它也能`禁止指令重排`。

双重检查锁的机制既保证了线程安全，又比直接上锁提高了执行效率，还节省了内存空间。



## 3、volatile不保证原子性

`volatile`关键字能保证变量在多个线程的可见性，能禁止CPU指令重排，但不能保证原子性

请看 [/icoding-edu/2020/03/07/icoding-note-005.html](/icoding-edu/2020/03/07/icoding-note-005.html)

可以使用3种方式保证原子性

- synchronized 关键字
- lock加锁
- java.util.concurrent.atomic 原子类

## 4、死锁

死锁可能是大家都不希望遇到的问题，因为一旦程序出现了死锁，如果没有外力的作用，程序将会一直处于资源竞争的假死状态中。

```java
public class DeadLockTest {
    public static String OBJECT_1 = "OBJECT_1";
    public static String OBJECT_2 = "OBJECT_2";

    public static void main(String[] args) {
        LockA lockA = new LockA();
        new Thread(lockA).start();

        LockB lockB = new LockB();
        new Thread(lockB).start();
    }

}

class LockA implements Runnable {
    @Override
    public void run() {
        synchronized (DeadLockTest.OBJECT_1) {
            try {
                Thread.sleep(500);

                synchronized (DeadLockTest.OBJECT_2) {
                    System.out.println("LockA");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

class LockB implements Runnable {
    @Override
    public void run() {
        synchronized (DeadLockTest.OBJECT_2) {
            try {
                Thread.sleep(500);

                synchronized (DeadLockTest.OBJECT_1) {
                    System.out.println("LockB");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

一个线程在获取OBJECT_1锁时，没有释放锁，又去申请OBJECT_2锁。而刚好此时，另一个线程获取到了OBJECT_2锁，也没有释放锁，去申请OBJECT_1锁。由于OBJECT_1和OBJECT_2锁都没有释放，两个线程将一起请求下去，陷入死循环，即出现`死锁`的情况。

那么如果避免死锁问题呢？

### 缩小锁的范围

```java
class LockA implements Runnable {
    @Override
    public void run() {
        synchronized (DeadLockTest.OBJECT_1) {
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        synchronized (DeadLockTest.OBJECT_2) {
             System.out.println("LockA");
        }
    }
}

class LockB implements Runnable {
    @Override
    public void run() {
        synchronized (DeadLockTest.OBJECT_2) {
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        synchronized (DeadLockTest.OBJECT_1) {
             System.out.println("LockB");
        }
    }
}
```

在获取OBJECT_1锁的代码块中，不包含获取OBJECT_2锁的代码。同时在获取OBJECT_2锁的代码块中，也不包含获取OBJECT_1锁的代码。

### 保证锁的顺序

如果我们能保证每次获取锁的顺序都相同，就不会出现死锁问题。

```java
class LockA implements Runnable {
    @Override
    public void run() {
        synchronized (DeadLockTest.OBJECT_1) {
            try {
                Thread.sleep(500);

                synchronized (DeadLockTest.OBJECT_2) {
                    System.out.println("LockA");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

class LockB implements Runnable {
    @Override
    public void run() {
        synchronized (DeadLockTest.OBJECT_1) {
            try {
                Thread.sleep(500);

                synchronized (DeadLockTest.OBJECT_2) {
                    System.out.println("LockB");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

两个线程，每个线程都是先获取OBJECT_1锁，再获取OBJECT_2锁。

## 5、没释放锁

java加锁有两种方式，`synchronized`关键字和`Lock`锁，后者需要手动释放

```java
public class LockTest {
    private final ReentrantLock rLock = new ReentrantLock();

    public void fun() {
        rLock.lock();
        try {
            System.out.println("fun");
        } finally {
            rLock.unlock();
        }
    }
}
```

代码中先创建一个`ReentrantLock`类的实例对象rLock，调用它的`lock`方法加锁。然后执行业务代码，最后再`finally`代码块中调用`unlock`方法。

## 6、HashMap导致内存溢出

HashMap是有线程安全问题的，因为put方法没有加锁

请看[/icoding-edu/2020/03/01/icoding-note-003.html](/icoding-edu/2020/03/01/icoding-note-003.html)

```java
@Service
public class HashMapService {

    private Map<Long, Object> hashMap = new HashMap<>();

    public void add(User user) {
        hashMap.put(user.getId(), user.getName());
    }
}
```

在controller层的接口中调用add方法，会使用tomcat的线程池去处理请求，就相当于在多线程的场景下调用add方法。一般情况下，Service层不会有成员变量

在jdk1.7中，HashMap使用的数据结构是：`数组`+`链表`。如果在多线程的情况下，不断往HashMap中添加数据，它会调用`resize`方法进行扩容。该方法在复制元素到新数组时，采用的头插法，在某些情况下，会导致链表会出现死循环。

死循环最终结果会导致：`内存溢出`。

如果HashMap中数据非常多，会导致链表很长。当查找某个元素时，需要遍历某个链表，查询效率不太高。

jdk1.8之后，将HashMap的数据结构改成了：`数组`+`链表`+`红黑树`，提高查询效率

如果同一个数组元素中的数据项小于8个，则还是用链表保存数据。如果大于8个，则自动转换成红黑树。

**为什么要用红黑树？**

答：链表的时间复杂度是O(n)，而红黑树的时间复杂度是O(logn)，红黑树的复杂度是优于链表的

**为什么不直接使用红黑树？**

答：树节点所占存储空间是链表节点的两倍，节点少的时候，尽管在时间复杂度上，红黑树比链表稍微好一些。但是由于红黑树所占空间比较大，HashMap综合考虑之后，认为节点数量少的时候用占存储空间更多的红黑树不划算。

**jdk1.8中HashMap还会不会出现死循环？**

答：会，它在多线程环境中依然会出现死循环。在扩容的过程中，在链表转换为树的时候，for循环一直无法跳出，从而导致死循环。

**如果想多线程环境中使用HashMap该怎么办呢？**

答：使用`ConcurrentHashMap`。

## 7、使用默认线程池

jdk1.5之后提供了`Executors`类，给我们快速创建线程池。

该类中包含了很多静态方法：

- `newCachedThreadPool`：创建一个可缓冲的线程，如果线程池大小超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
- `newFixedThreadPool`：创建一个固定大小的线程池，如果任务数量超过线程池大小，则将多余的任务放到队列中。
- `newScheduledThreadPool`：创建一个固定大小，并且能执行定时周期任务的线程池。
- `newSingleThreadExecutor`：创建只有一个线程的线程池，保证所有的任务安装顺序执行。

在高并发的场景下，会有OOM问题

- `newFixedThreadPool`：允许请求的队列长度是Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM。
- `newSingleThreadExecutor`：允许请求的队列长度是Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM。
- `newCachedThreadPool`：允许创建的线程数是Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM。

[icoding-edu/2020/03/04/icoding-note-004.html](icoding-edu/2020/03/04/icoding-note-004.html)

**那我们该怎办呢？**

优先推荐使用`ThreadPoolExecutor`类，我们自定义线程池。

```java
ExecutorService threadPool = new ThreadPoolExecutor(
    8, //corePoolSize线程池中核心线程数
    10, //maximumPoolSize 线程池中最大线程数
    60, //线程池中线程的最大空闲时间，超过这个时间空闲线程将被回收
    TimeUnit.SECONDS,//时间单位
    new ArrayBlockingQueue(500), //队列
    new ThreadPoolExecutor.CallerRunsPolicy()); //拒绝策略
}
```

## 8、@Async注解的陷阱

[/icoding-edu/2020/03/22/icoding-note-012.html](/icoding-edu/2020/03/22/icoding-note-012.html)

1.在`springboot`的启动类上面加上`@EnableAsync`注解。

```java
@EnableAsync
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

2.在需要执行异步调用的业务方法加上`@Async`注解。

```java
@Service
public class CategoryService {
     @Async
     public void add(Category category) {
        //添加分类
     }
}
```

3.在controller方法中调用这个业务方法。

```java
@RestController
@RequestMapping("/category")
public class CategoryController {
     @Autowired
     private CategoryService categoryService;
  
     @PostMapping("/add")
     public void add(@RequestBody category) {
        categoryService.add(category);
     }
}
```

这样就能开启异步功能了

但是，用@Async注解开启的异步功能，会调用`AsyncExecutionAspectSupport`类的`doSubmit`方法。

![](\assets\images\2022\java\async-do-submit.png)

默认情况会走else逻辑，而else的逻辑最终会调用doExecute方法：

```java
protected void doExecute(Runnable task) {
  Thread thread = (this.threadFactory != null ? this.threadFactory.newThread(task) : createThread(task));
  thread.start();
}
```

我去，这不是每次都会创建一个新线程吗？

没错，使用@Async注解开启的异步功能，默认情况下，每次都会创建一个新线程。

如果在高并发的场景下，可能会产生大量的线程，从而导致OOM问题。

<mark>建议大家在@Async注解开启的异步功能时，请别忘了定义一个线程池。</mark>

## 9、自旋锁浪费cpu资源

[/icoding-edu/2020/03/07/icoding-note-005.html](/icoding-edu/2020/03/07/icoding-note-005.html)

自旋锁有个非常经典的使用场景就是：`CAS`（即比较和交换），它是一种无锁化思想（说白了用了一个死循环），用来解决高并发场景下，更新数据的问题。

而atomic包下的很多类，比如：AtomicInteger、AtomicLong、AtomicBoolean等，都是用CAS实现的。

我们以`AtomicInteger`类为例，它的`incrementAndGet`没有每次都给变量加1。

```java
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

它的底层就是用的自旋锁实现的：

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
  int var5;
  do {
      var5 = this.getIntVolatile(var1, var2);
  } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

在do...while死循环中，不停进行数据的比较和交换，如果一直失败，则一直循环重试。

如果在高并发的情况下，compareAndSwapInt会很大概率失败，因此导致了此处cpu不断的自旋，这样会严重浪费cpu资源。

**那么，如果解决这个问题呢？**

答：使用`LockSupport`类的`parkNanos`方法。

```java
private boolean compareAndSwapInt2(Object var1, long var2, int var4, int var5) {
     if(this.compareAndSwapInt(var1,var2,var4, var5)) {
          return true;
      } else {
          LockSupport.parkNanos(10);
          return false;
      }
 }
```

当cas失败之后，调用LockSupport类的parkNanos方法休眠一下，相当于调用了Thread.Sleep方法。这样能够有效的减少频繁自旋导致cpu资源过度浪费的问题。

## 10、ThreadLocal用完没清空

请看 [ThreadLocal的使用与内存泄漏](/java/2020/09/17/java-threadlocal.html)

java中保证线程安全的技术有很多，可以使用synchroized、Lock等关键字给代码块加锁。但是它们有个共同的特点，就是加锁会对代码的性能有一定的损耗。

在jdk中还提供了另外一种思想即：`用空间换时间`。使用`ThreadLocal`类就是对这种思想的一种具体体现。

ThreadLocal为每个使用变量的线程提供了一个独立的变量副本，这样每一个线程都能独立地改变自己的副本，而不会影响其它线程所对应的副本。

1. 先创建一个CurrentUser类，其中包含了ThreadLocal的逻辑。

   ```java
   public class CurrentUser {
       private static final ThreadLocal<UserInfo> THREA_LOCAL = new ThreadLocal();
       
       public static void set(UserInfo userInfo) {
           THREA_LOCAL.set(userInfo);
       }
       
       public static UserInfo get() {
          THREA_LOCAL.get();
       }
       
       public static void remove() {
          THREA_LOCAL.remove();
       }
   }
   ```

2. 在业务代码中调用CurrentUser类。

   ```java
   public void doSamething(UserDto userDto) {
      UserInfo userInfo = convert(userDto);
      CurrentUser.set(userInfo);
      ...
   
      //业务代码
      UserInfo userInfo = CurrentUser.get();
      ...
   }
   ```

在业务代码的第一行，将userInfo对象设置到CurrentUser，这样在业务代码中，就能通过CurrentUser.get()获取到刚刚设置的userInfo对象。特别是对业务代码调用层级比较深的情况，这种用法非常有用，可以减少很多不必要传参。

但在高并发的场景下，这段代码有问题，只往ThreadLocal存数据，数据用完之后并没有及时清理。

ThreadLocal即使使用了`WeakReference`（弱引用）也可能会存在`内存泄露`问题，因为 entry对象中只把key(即threadLocal对象)设置成了弱引用，但是value值没有。

**那么，如何解决这个问题呢？**

使用完后，调用remove清除线程变量的值

```java
public void doSamething(UserDto userDto) {
   UserInfo userInfo = convert(userDto);
   
   try{
     CurrentUser.set(userInfo);
     ...
     
     //业务代码
     UserInfo userInfo = CurrentUser.get();
     ...
   } finally {
      CurrentUser.remove();
   }
}
```

需要在`finally`代码块中，调用remove方法清理没用的数据。
