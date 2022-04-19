---
layout: post
title: ThreadLocal使用与内存泄漏
category: java
tags: [java]
keywords: juc
excerpt: threadLocal线程局部变量为我们解决了多线程访问变量的安全问题,key是弱引用，只能生存到下次GC前，如果很多线程使用Threadlocal可能会引起内存泄露，不同的请求获取相同的ThreadLocal值如何解决，InheritableThreadLocal可继承线程变量在线程池中执行被获取相同值的原因，因为线程变量没有被重新set值
lock: noneed
---

## 1、ThreadLocal解决了什么问题

被threadlocal声明的变量只能在线程内访问，解决了对象在被共享访问时带来的线程安全问题，原理就是每个线程都有一个threadlocal变量的副本，每个线程只能访问自己的副本。

我们平时是使用加锁的方式，如synchronized 和 ReentrantLock，保证线程安全（原子性操作），生活中的例子：现在公司所有人都要填写一个表格，但是只有一支笔，这个时候就只能上个人用完了之后，下个人才可以使用，为了保证"笔"这个资源的可用性，只需要保证在接下来每个人的获取顺序就可以了，这就是 lock 的作用，当这支笔被别人用的时候，我就加 lock ，你来了那就进入队列排队等待获取资源（非公平方式那就另外说了），这支笔用完之后就释放 lock ，然后按照顺序给下个人使用。

但是完全可以一个人一支笔对不对，这样的话，你填写你的表格，我填写我的表格，咱俩谁都不耽搁谁。这就是 ThreadLocal 在做的事情，因为每个 Thread 都有一个副本，就不存在资源竞争，所以也就不需要加锁，这不就是拿空间去换了时间嘛。数据结构如下图：

![](\assets\images\2020\juc\threadlocal.jpg)

可以看到，在 Thread 中持有一个 ThreadLocalMap ， ThreadLocalMap 是由 Entry 来组成的，在 Entry 里面有 ThreadLocal 和 value，

,ThreadLocalMap 是Thread的静态内部类，它通过调用creatMap()方法懒加载创建，这里也要说要一下ThreadLocal的子类InheritableThreadLocal，源码如下：

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    /**
     * Computes the child's initial value for this inheritable thread-local
     * variable as a function of the parent's value at the time the child
     * thread is created.  This method is called from within the parent
     * thread before the child is started.
     * <p>
     * This method merely returns its input argument, and should be overridden
     * if a different behavior is desired.
     *
     * @param parentValue the parent thread's value
     * @return the child thread's initial value
     */
    protected T childValue(T parentValue) {
        return parentValue;
    }

    /**
     * Get the map associated with a ThreadLocal.
     *
     * @param t the current thread
     */
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    /**
     * Create the map associated with a ThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the table.
     */
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

ThreadLocal调用set方法设置局部变量value，懒加载创建当前线程的ThreadLocalMap对象

```java
/**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
public void set(T value) {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
}
```

如果是InheritableThreadLocal则调用自己的createMap()方法，如果是ThreadLoca则调用它自己的createMap()方法

```java
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

都是给当前线程Thread的成员变量创建ThreadLocalMap对象，并把线程变量值放入ThreadLocalMap对象中，当线程结束，它里面的成员变量就会被回收

![](\assets\images\2021\javabase\Thread-threadLocals-threadLocalMap.jpg)

看ThreadLocalMap的构造方法

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
  table = new Entry[INITIAL_CAPACITY];
  int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
  table[i] = new Entry(firstKey, firstValue);
  size = 1;
  setThreshold(INITIAL_CAPACITY);
}
```

Entry 数组初始容量是16，ThreadLocal线程变量本身就是个key，Entry是ThreadLocalMap的静态内部类

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
  /** The value associated with this ThreadLocal. */
  Object value;

  Entry(ThreadLocal<?> k, Object v) {
    super(k);
    value = v;
  }
}
```

调用super

```java
public class WeakReference<T> extends Reference<T> {

    /**
     * Creates a new weak reference that refers to the given object.  The new
     * reference is not registered with any queue.
     *
     * @param referent object the new weak reference will refer to
     */
    public WeakReference(T referent) {
        super(referent);
    }

    /**
     * Creates a new weak reference that refers to the given object and is
     * registered with the given queue.
     *
     * @param referent object the new weak reference will refer to
     * @param q the queue with which the reference is to be registered,
     *          or <tt>null</tt> if registration is not required
     */
    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

Entry的key就是ThreadLocal的弱引用，发送GC时就被回收，导致ThreadLocal的value值无法访问，但是value不会被回收。

当前线程的另ThreadLocal变量调用set方法，当前线程的ThreadLocalMap已创建就直接调用它的set方法，把ThreadLocal变量值放进Entry数组

```java
private void set(ThreadLocal<?> key, Object value) {

  // We don't use a fast path as with get() because it is at
  // least as common to use set() to create new entries as
  // it is to replace existing ones, in which case, a fast
  // path would fail more often than not.

  Entry[] tab = table;
  int len = tab.length;
  int i = key.threadLocalHashCode & (len-1);

  for (Entry e = tab[i];
       e != null;
       e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();

    if (k == key) {
      e.value = value;
      return;
    }

    if (k == null) {
      replaceStaleEntry(key, value, i);
      return;
    }
  }

  tab[i] = new Entry(key, value);
  int sz = ++size;
  if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
}
```



## 2、ThreadLocalMap

点进ThreadLocal的源码，里面维护一个ThreadLocalMap

![](\assets\images\2020\juc\threadlocal3.jpg)

点进WeakReference

![](\assets\images\2020\juc\threadlocal4.jpg)

在调用 super(k) 时就将 ThreadLocal 实例包装成了一个 WeakReference

Java的4大引用，在我JUC的笔记中，coding老师也讲过

|           引用类型           |                           功能特点                           |
| :--------------------------: | :----------------------------------------------------------: |
| 强引用 ( Strong Reference )  |         被强引用关联的对象永远不会被垃圾回收器回收掉         |
|   软引用( Soft Reference )   | 软引用关联的对象，只有当系统将要发生内存溢出时，才会去回收软引用引用的对象 |
|  弱引用 ( Weak Reference )   |    只被弱引用关联的对象，只要发生垃圾收集事件，就会被回收    |
| 虚引用 ( Phantom Reference ) | 被虚引用关联的对象的唯一作用是能在这个对象被回收器回收时收到一个系统通知 |

为什么key不使用强引用，如果使用强引用，没有对threadlocal对象设置为null，ThreadLocalMap 没有对remove，在 GC 时进行可达性分析， ThreadLocal 依然可达，这样就不会对 ThreadLocal 进行回收

使用弱引用的话，虽然会出现内存泄漏的问题，但是在 ThreadLocal 生命周期里面，都有对 key 值为 null 时进行回收的处理操作

## 3、内存泄漏

Entry 继承了 WeakReference ，而 Entry 对象中的 key 使用了 WeakReference 封装，key是一个弱引用，只能生存到下次GC前。

如果一个线程调用了 ThreadLocalMap 的 set 设置变量，当前的 ThreadLocalMap 就会新增一条记录，但由于发生了一次垃圾回收， key 值被回收掉了，但是 value 值还在内存中，而且如果线程一直存在的话，那么它的 value 值就会一直存在。就会存在一条引用链：thread -> ThreadLocalMap -> Entry -> Value 

![](\assets\images\2020\juc\threadlocal2.jpg)

就是因为这条引用链的存在，就会导致如果Thread 还在运行，那么 Entry 不会被回收，进而 value 也不会被回收掉，但是 Entry 里面的 key 值已经被回收掉了。如果很多个线程就会造成内存泄漏，因为value值没有被回收掉。

**解决办法**：使用完 key 值之后，将 value 值通过 remove 方法 remove 掉

## 4、相同ThreadLocal值

两个不同的请求，在获取ThreadLocal里保存的值时，获取到了相同的值。

原因：这两个请求共用了一个线程

- http1.1协议中的keep-alive是默认开启的，同一个会话中，有限的请求是共用一个长连接的。
- tomcat默认使用线程池，所以一个线程的生命周期不能对等于一个请求的生命周期，线程池中的线程是可以被复用的。每一个浏览器的请求其实是被tomcat监听到，然后调用线程进行处理，线程是被复用的处理多个请求，当tomcat线程池满了，没有线程资源处理请求，请求就会被挂起，无响应。

<mark>解决方案</mark>

1）保证每次请求都用新的值覆盖线程变量，调用ThreadLocal的set方法

2）保证在每个请求结束后清空线程变量，调用ThreadLocal的remove方法

## 5、可继承的线程变量

InheritableThreadLocal 是指当子线程被创建时，父线程会把自己的InheritableThreadLocal 变量全部复制到子线程的ThreadLoaLMap中，具体怎么实现，从子线程被创建开始，点击Thread源码，找到构造方法

```java
public Thread() {
  init(null, null, "Thread-" + nextThreadNum(), 0);
}

public Thread(Runnable target) {
  init(null, target, "Thread-" + nextThreadNum(), 0);
}
```

init方法

```java
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize) {
  init(g, target, name, stackSize, null, true);
}

private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;

        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            /* Determine if it's an applet or not */

            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }

            /* If the security doesn't have a strong opinion of the matter
               use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();

        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();

        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }
```

里面有一段代码，创建子线程的inheritThreadLocals对象，入参是父线程的inheritThreadLocals对象

```java
 if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
```

跳到ThreadLocal的createInheritedMap方法

```java
    static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
        return new ThreadLocalMap(parentMap);
    }
```

构造方法

```java
private ThreadLocalMap(ThreadLocalMap parentMap) {
  Entry[] parentTable = parentMap.table;
  int len = parentTable.length;
  setThreshold(len);
  table = new Entry[len];

  for (int j = 0; j < len; j++) {
    Entry e = parentTable[j];
    if (e != null) {
      @SuppressWarnings("unchecked")
      ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
      if (key != null) {
        Object value = key.childValue(e.value);
        Entry c = new Entry(key, value);
        int h = key.threadLocalHashCode & (len - 1);
        while (table[h] != null)
          h = nextIndex(h, len);
        table[h] = c;
        size++;
      }
    }
  }
}
```

这是线程被创建的整个流程。

总结，使用InheritableThreadLocal 线程变量时，要注意在线程池中执行的任务获取的InheritableThreadLocal 的变量值大概率都是相同的，因为线程是被任务复用的。

参考:

- [https://blog.csdn.net/weixin_36210111/article/details/115780843](https://blog.csdn.net/weixin_36210111/article/details/115780843)
- [https://www.jianshu.com/p/09ceb962894d](https://www.jianshu.com/p/09ceb962894d)
- https://blog.csdn.net/v123411739/article/details/79117430