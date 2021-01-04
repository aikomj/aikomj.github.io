---
layout: post
title: Spring boot整合Quartz实现Java定时任务的动态配置，集群环境配置
category: springboot
tags: [springboot]
keywords: springboot
excerpt: Java定时任务使用过的解决方案，Timer定时器，@Scheduled注解，Quartz框架
lock: noneed
---

## 0、while+sleep方案

最简单定时，while+sleep方案，就是定义一个线程，然后 while 循环，通过 sleep 延迟时间来达到周期性调度任务。

```java
public static void main(String[] args) {
    final long timeInterval = 5000;
    new Thread(new Runnable() {
        @Override
        public void run() {
            while (true) {
                System.out.println(Thread.currentThread().getName() + "每隔5秒执行一次");
                try {
                    Thread.sleep(timeInterval);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }).start();
}
```

实现上非常简单，如果我们想在创建一个每隔3秒钟执行一次任务，怎么办呢？

同样的，也可以在创建一个线程，然后间隔性的调度方法；但是如果创建了大量这种类型的线程，这个时候会发现大量的定时任务线程在调度切换时性能消耗会非常大，而且整体效率低！

面对这种在情况，大佬们也想到了，于是想出了用一个线程将所有的定时任务存起来，事先排好序，按照一定的规则来调度，这样不就可以极大的减少每个线程的切换消耗吗？

正因此，JDK 中的 Timer 定时器由此诞生了！

## 1、Timer定时器

### Timer

首先看一个timer的例子

```java
public static void main(String[] args) {
    Timer timer = new Timer();
    //每隔1秒调用一次
    timer.schedule(new TimerTask() {
        @Override
        public void run() {
            System.out.println("test1");
        }
    }, 1000, 1000);
    //每隔3秒调用一次
    timer.schedule(new TimerTask() {
        @Override
        public void run() {
            System.out.println("test2");
        }
    }, 3000, 3000);
}
```

发现两个线程任务共用一个timer调度器，一起来看看源码

- 进入Timer.schedule()方法

  ```java
  public void schedule(TimerTask task, long delay, long period) {
      if (delay < 0)
          throw new IllegalArgumentException("Negative delay.");
      if (period <= 0)
          throw new IllegalArgumentException("Non-positive period.");
      sched(task, System.currentTimeMillis()+delay, -period);
  }
  ```

  从方法上可以看出，这里主要做参数验证，其中`TimerTask`是一个线程任务，`delay`表示延迟多久执行（单位毫秒），`period`表示多久执行一次（单位毫秒）

- 接着看`sched()`方法

  ```java
  private void sched(TimerTask task, long time, long period) {
      if (time < 0)
          throw new IllegalArgumentException("Illegal execution time.");
  
      // Constrain value of period sufficiently to prevent numeric
      // overflow while still being effectively infinitely large.
      if (Math.abs(period) > (Long.MAX_VALUE >> 1))
          period >>= 1;
  
      synchronized(queue) {
          if (!thread.newTasksMayBeScheduled)
              throw new IllegalStateException("Timer already cancelled.");
  
          synchronized(task.lock) {
              if (task.state != TimerTask.VIRGIN)
                  throw new IllegalStateException(
                      "Task already scheduled or cancelled");
              task.nextExecutionTime = time;
              task.period = period;
              task.state = TimerTask.SCHEDULED;
          }
  
          queue.add(task);
          if (queue.getMin() == task)
              queue.notify();
      }
  }
  ```

  这步操作中，可以很清晰的看到，在同步代码块里，会将`task`对象加入到`queue`对象

- 我们继续来看`queue`对象

  ```java
  public class Timer {
      
      private final TaskQueue queue = new TaskQueue();
  
      private final TimerThread thread = new TimerThread(queue);
  
      public Timer() {
          this("Timer-" + serialNumber());
      }
  
      public Timer(String name) {
          thread.setName(name);
          thread.start();
      }
  
      //...
  }
  ```

  任务会进入到`TaskQueue`队列中，同时在`Timer`初始化阶段会将`TaskQueue`作为参数传入到`TimerThread`线程中，并且启动线程。

  而`TaskQueue`其实是一个最小堆的数据实体类，源码如下

  > 每当有新元素加入的时候，会对原来的数组进行重排，会将即将要执行的任务排在数组的前面

  ```java
  class TaskQueue {
      
      private TimerTask[] queue = new TimerTask[128];
  
  
      private int size = 0;
  
      void add(TimerTask task) {
          // Grow backing store if necessary
          if (size + 1 == queue.length)
              queue = Arrays.copyOf(queue, 2*queue.length);
  
          queue[++size] = task;
          fixUp(size);
      }
  
      private void fixUp(int k) {
          while (k > 1) {
              int j = k >> 1;
              if (queue[j].nextExecutionTime <= queue[k].nextExecutionTime)
                  break;
              TimerTask tmp = queue[j];
     queue[j] = queue[k];
     queue[k] = tmp;
              k = j;
          }
      }
   
   //....
  }
  ```

- 最后我们来看看`TimerThread`

  ```java
  class TimerThread extends Thread {
      boolean newTasksMayBeScheduled = true;
      private TaskQueue queue;
  
      TimerThread(TaskQueue queue) {
          this.queue = queue;
      }
  
      public void run() {
          try {
              mainLoop();
          } finally {
              // Someone killed this Thread, behave as if Timer cancelled
              synchronized(queue) {
                  newTasksMayBeScheduled = false;
                  queue.clear();  // Eliminate obsolete references
              }
          }
      }
  
      /**
       * The main timer loop.  (See class comment.)
       */
      private void mainLoop() {
          while (true) {
              try {
                  TimerTask task;
                  boolean taskFired;
                  synchronized(queue) {
                      // Wait for queue to become non-empty
                      while (queue.isEmpty() && newTasksMayBeScheduled)
                          queue.wait();
                      if (queue.isEmpty())
                          break; // Queue is empty and will forever remain; die
  
                      // Queue nonempty; look at first evt and do the right thing
                      long currentTime, executionTime;
                      task = queue.getMin();
                      synchronized(task.lock) {
                          if (task.state == TimerTask.CANCELLED) {
                              queue.removeMin();
                              continue;  // No action required, poll queue again
                          }
                          currentTime = System.currentTimeMillis();
                          executionTime = task.nextExecutionTime;
                          if (taskFired = (executionTime<=currentTime)) {
                              if (task.period == 0) { // Non-repeating, remove
                                  queue.removeMin();
                                  task.state = TimerTask.EXECUTED;
                              } else { // Repeating task, reschedule
                                  queue.rescheduleMin(
                                    task.period<0 ? currentTime   - task.period
                                                  : executionTime + task.period);
                              }
                          }
                      }
                      if (!taskFired) // Task hasn't yet fired; wait
                          queue.wait(executionTime - currentTime);
                  }
                  if (taskFired)  // Task fired; run it, holding no locks
                      task.run();
              } catch(InterruptedException e) {
              }
          }
      }
  }
  ```

  `TimerThread`其实就是一个任务调度线程，首先从`TaskQueue`里面获取排在最前面的任务，然后判断它是否到达任务执行时间点，如果已到达，就会立刻执行任务

**总结**

相比 **while + sleep** 方案，多了一个线程来管理所有的任务，优点就是**减少了线程之间的性能开销，提升了执行效率**；但是同样也带来的了一些缺点，整体的新加任务写入效率变成了 O(log(n))。

缺点：

- **串行阻塞**：调度线程只有一个，长任务会阻塞短任务的执行，例如，A任务跑了一分钟，B任务至少需要等1分钟才能跑
- **容错能力差**：没有异常处理能力，一旦一个任务执行故障，后续任务都无法执行

### ScheduledThreadPoolExecutor

鉴于 Timer 的上述缺陷，从 Java 5 开始，推出了基于线程池设计的 ScheduledThreadPoolExecutor 调度线程池。

![](\assets\images\2021\juc\scheduled-thread-pool-executor.jpg)

其设计思想是，每一个被调度的任务都会由线程池来管理执行，因此任务是并发执行的，相互之间不会受到干扰。需要注意的是，只有当任务的执行时间到来时，ScheduledThreadPoolExecutor 才会真正启动一个线程，其余时间 ScheduledThreadPoolExecutor 都是在轮询任务的状态。

**例子**

```java
public static void main(String[] args) {
    ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(3);
    //启动1秒之后，每隔1秒执行一次
    executor.scheduleAtFixedRate((new Runnable() {
        @Override
        public void run() {
            System.out.println("test3");
        }
    }),1,1, TimeUnit.SECONDS);
    //启动1秒之后，每隔3秒执行一次
    executor.scheduleAtFixedRate((new Runnable() {
        @Override
        public void run() {
            System.out.println("test4");
        }
    }),1,3, TimeUnit.SECONDS);
}
```

同样的，我们首先打开源码，看看里面到底做了啥

- 进入`scheduleAtFixedRate()`方法

  ```java
  public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                long initialDelay,
                                                long period,
                                                TimeUnit unit) {
      if (command == null || unit == null)
          throw new NullPointerException();
      if (period <= 0)
          throw new IllegalArgumentException();
      ScheduledFutureTask<Void> sft =
          new ScheduledFutureTask<Void>(command,
                                        null,
                                        triggerTime(initialDelay, unit),
                                        unit.toNanos(period));
      RunnableScheduledFuture<Void> t = decorateTask(command, sft);
      sft.outerTask = t;
      delayedExecute(t);
      return t;
  }
  ```

  首先是校验基本参数，然后将任务作为封装到`ScheduledFutureTask`线程中，`ScheduledFutureTask`继承自`RunnableScheduledFuture`，并作为参数调用`delayedExecute()`方法进行预处理

- 进入`delayedExecute()`方法

  ```java
  private void delayedExecute(RunnableScheduledFuture<?> task) {
      if (isShutdown())
          reject(task);
      else {
          super.getQueue().add(task);
          if (isShutdown() &&
              !canRunInCurrentRunState(task.isPeriodic()) &&
              remove(task))
              task.cancel(false);
          else
     //预处理
              ensurePrestart();
      }
  }
  ```

  可以很清晰的看到，当线程池没有关闭的时候，会通过`super.getQueue().add(task)`操作，将任务加入到队列，同时调用`ensurePrestart()`方法做预处理

  其中`super.getQueue()`得到的是一个自定义的`new DelayedWorkQueue()`阻塞队列，数据存储方面也是一个最小堆结构的队列，这一点在初始化`new ScheduledThreadPoolExecutor()`的时候，可以看出！

  ```java
  public ScheduledThreadPoolExecutor(int corePoolSize) {
      super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
            new DelayedWorkQueue());
  }
  ```

  打开源码可以看到，`DelayedWorkQueue`其实是`ScheduledThreadPoolExecutor`中的一个静态内部类，在添加的时候，会将任务加入到`RunnableScheduledFuture`数组中，同时线程池中的`Woker`线程会通过调用任务队列中的`take()`方法获取对应的`ScheduledFutureTask`线程任务，接着执行对应的任务线程

  ```java
  static class DelayedWorkQueue extends AbstractQueue<Runnable>
          implements BlockingQueue<Runnable> {
  
      private static final int INITIAL_CAPACITY = 16;
      private RunnableScheduledFuture<?>[] queue =
          new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
      private final ReentrantLock lock = new ReentrantLock();
      private int size = 0;   
  
      //....
  
      public boolean add(Runnable e) {
          return offer(e);
      }
  
      public boolean offer(Runnable x) {
          if (x == null)
              throw new NullPointerException();
          RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
          final ReentrantLock lock = this.lock;
          lock.lock();
          try {
              int i = size;
              if (i >= queue.length)
                  grow();
              size = i + 1;
              if (i == 0) {
                  queue[0] = e;
                  setIndex(e, 0);
              } else {
                  siftUp(i, e);
              }
              if (queue[0] == e) {
                  leader = null;
                  available.signal();
              }
          } finally {
              lock.unlock();
          }
          return true;
      }
  
      public RunnableScheduledFuture<?> take() throws InterruptedException {
          final ReentrantLock lock = this.lock;
          lock.lockInterruptibly();
          try {
              for (;;) {
                  RunnableScheduledFuture<?> first = queue[0];
                  if (first == null)
                      available.await();
                  else {
                      long delay = first.getDelay(NANOSECONDS);
                      if (delay <= 0)
                          return finishPoll(first);
                      first = null; // don't retain ref while waiting
                      if (leader != null)
                          available.await();
                      else {
                          Thread thisThread = Thread.currentThread();
                          leader = thisThread;
                          try {
                              available.awaitNanos(delay);
                          } finally {
                              if (leader == thisThread)
                                  leader = null;
                          }
                      }
                  }
              }
          } finally {
              if (leader == null && queue[0] != null)
                  available.signal();
              lock.unlock();
          }
      }
  }
  ```

- 回到我们最开始说到的`ScheduledFutureTask`任务线程类，最终执行任务的其实就是它

  ```java
  private class ScheduledFutureTask<V>
              extends FutureTask<V> implements RunnableScheduledFuture<V> {
  
      /** Sequence number to break ties FIFO */
      private final long sequenceNumber;
  
      /** The time the task is enabled to execute in nanoTime units */
      private long time;
  
      /**
       * Period in nanoseconds for repeating tasks.  A positive
       * value indicates fixed-rate execution.  A negative value
       * indicates fixed-delay execution.  A value of 0 indicates a
       * non-repeating task.
       */
      private final long period;
  
      /** The actual task to be re-enqueued by reExecutePeriodic */
      RunnableScheduledFuture<V> outerTask = this;
  
      /**
       * Overrides FutureTask version so as to reset/requeue if periodic.
       */
      public void run() {
          boolean periodic = isPeriodic();
          if (!canRunInCurrentRunState(periodic))
              cancel(false);
          else if (!periodic)
              ScheduledFutureTask.super.run();
          else if (ScheduledFutureTask.super.runAndReset()) {
              setNextRunTime();
              reExecutePeriodic(outerTask);
          }
      }
   //...
  }
  ```

  `ScheduledFutureTask`任务线程，才是真正执行任务的线程类，只是绕了一圈，做了很多包装，`run()`方法就是真正执行定时任务的方法。

> 小结

ScheduledExecutorService 相比 Timer 定时器，完美的解决上面说到的 Timer 存在的两个缺点！

<font style="color:red">在单体应用里面</font>，使用 ScheduledExecutorService 可以解决大部分需要使用定时任务的业务需求！

但是这是否意味着它是最佳的解决方案呢？

我们发现线程池中 ScheduledExecutorService 的排序容器跟 Timer 一样，都是采用最小堆的存储结构，新任务加入排序效率是`O(log(n))`，执行取任务是`O(1)`。

这里的写入排序效率其实是有空间可提升的，有可能优化到`O(1)`的时间复杂度，也就是我们下面要介绍的**时间轮实现**！

### 时间轮实现

所谓时间轮（RingBuffer）实现，从数据结构上看，简单的说就是循环队列，从名称上看可能感觉很抽象。

它其实就是一个环形的数组，如图所示，假设我们创建了一个长度为 8 的时间轮

![](\assets\images\2021\juc\scheudler-1.jpg)

> 1、插入、取值流程

- 1.当我们需要新建一个 1s 延时任务的时候，则只需要将它放到下标为 1 的那个槽中，2、3、...、7也同样如此。
- 2.而如果是新建一个 10s 的延时任务，则需要将它放到下标为 2 的槽中，但同时需要记录它所对应的圈数，也就是 1 圈，不然就和 2 秒的延时消息重复了
- 3.当创建一个 21s 的延时任务时，它所在的位置就在下标为 5 的槽中，同样的需要为他加上圈数为 2，依次类推...

因此，总结起来有两个核心的变量：

- 数组下标：表示某个任务延迟时间，从数据操作上对执行时间点进行取余
- 圈数：表示需要循环圈数

通过这张图可以更直观的理解！

![](\assets\images\2021\juc\scheudler-2.jpg)

当我们需要取出延时任务时，只需要每秒往下移动这个指针，然后取出该位置的所有任务即可，取任务的时间消耗为`O(1)`。

当我们需要插入任务式，也只需要计算出对应的下表和圈数，即可将任务插入到对应的数组位置中，插入任务的时间消耗为`O(1)`。

如果时间轮的槽比较少，会导致某一个槽上的任务非常多，那么效率也比较低，这就和 HashMap 的 hash 冲突是一样的，因此在设计槽的时候不能太大也不能太小。

> 2、代码实现

首先创建一个`RingBufferWheel`时间轮定时任务管理器

```java
public class RingBufferWheel {

    private Logger logger = LoggerFactory.getLogger(RingBufferWheel.class);


    /**
     * default ring buffer size
     */
    private static final int STATIC_RING_SIZE = 64;

    private Object[] ringBuffer;

    private int bufferSize;

    /**
     * business thread pool
     */
    private ExecutorService executorService;

    private volatile int size = 0;

    /***
     * task stop sign
     */
    private volatile boolean stop = false;

    /**
     * task start sign
     */
    private volatile AtomicBoolean start = new AtomicBoolean(false);

    /**
     * total tick times
     */
    private AtomicInteger tick = new AtomicInteger();

    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    private AtomicInteger taskId = new AtomicInteger();
    private Map<Integer, Task> taskMap = new ConcurrentHashMap<>(16);

    /**
     * Create a new delay task ring buffer by default size
     *
     * @param executorService the business thread pool
     */
    public RingBufferWheel(ExecutorService executorService) {
        this.executorService = executorService;
        this.bufferSize = STATIC_RING_SIZE;
        this.ringBuffer = new Object[bufferSize];
    }


    /**
     * Create a new delay task ring buffer by custom buffer size
     *
     * @param executorService the business thread pool
     * @param bufferSize      custom buffer size
     */
    public RingBufferWheel(ExecutorService executorService, int bufferSize) {
        this(executorService);

        if (!powerOf2(bufferSize)) {
            throw new RuntimeException("bufferSize=[" + bufferSize + "] must be a power of 2");
        }
        this.bufferSize = bufferSize;
        this.ringBuffer = new Object[bufferSize];
    }

    /**
     * Add a task into the ring buffer(thread safe)
     *
     * @param task business task extends {@link Task}
     */
    public int addTask(Task task) {
        int key = task.getKey();
        int id;

        try {
            lock.lock();
            int index = mod(key, bufferSize);
            task.setIndex(index);
            Set<Task> tasks = get(index);

            int cycleNum = cycleNum(key, bufferSize);
            if (tasks != null) {
                task.setCycleNum(cycleNum);
                tasks.add(task);
            } else {
                task.setIndex(index);
                task.setCycleNum(cycleNum);
                Set<Task> sets = new HashSet<>();
                sets.add(task);
                put(key, sets);
            }
            id = taskId.incrementAndGet();
            task.setTaskId(id);
            taskMap.put(id, task);
            size++;
        } finally {
            lock.unlock();
        }

        start();

        return id;
    }


    /**
     * Cancel task by taskId
     * @param id unique id through {@link #addTask(Task)}
     * @return
     */
    public boolean cancel(int id) {

        boolean flag = false;
        Set<Task> tempTask = new HashSet<>();

        try {
            lock.lock();
            Task task = taskMap.get(id);
            if (task == null) {
                return false;
            }

            Set<Task> tasks = get(task.getIndex());
            for (Task tk : tasks) {
                if (tk.getKey() == task.getKey() && tk.getCycleNum() == task.getCycleNum()) {
                    size--;
                    flag = true;
                    taskMap.remove(id);
                } else {
                    tempTask.add(tk);
                }

            }
            //update origin data
            ringBuffer[task.getIndex()] = tempTask;
        } finally {
            lock.unlock();
        }

        return flag;
    }

    /**
     * Thread safe
     *
     * @return the size of ring buffer
     */
    public int taskSize() {
        return size;
    }

    /**
     * Same with method {@link #taskSize}
     * @return
     */
    public int taskMapSize(){
        return taskMap.size();
    }

    /**
     * Start background thread to consumer wheel timer, it will always run until you call method {@link #stop}
     */
    public void start() {
        if (!start.get()) {

            if (start.compareAndSet(start.get(), true)) {
                logger.info("Delay task is starting");
                Thread job = new Thread(new TriggerJob());
                job.setName("consumer RingBuffer thread");
                job.start();
                start.set(true);
            }

        }
    }

    /**
     * Stop consumer ring buffer thread
     *
     * @param force True will force close consumer thread and discard all pending tasks
     *              otherwise the consumer thread waits for all tasks to completes before closing.
     */
    public void stop(boolean force) {
        if (force) {
            logger.info("Delay task is forced stop");
            stop = true;
            executorService.shutdownNow();
        } else {
            logger.info("Delay task is stopping");
            if (taskSize() > 0) {
                try {
                    lock.lock();
                    condition.await();
                    stop = true;
                } catch (InterruptedException e) {
                    logger.error("InterruptedException", e);
                } finally {
                    lock.unlock();
                }
            }
            executorService.shutdown();
        }


    }


    private Set<Task> get(int index) {
        return (Set<Task>) ringBuffer[index];
    }

    private void put(int key, Set<Task> tasks) {
        int index = mod(key, bufferSize);
        ringBuffer[index] = tasks;
    }

    /**
     * Remove and get task list.
     * @param key
     * @return task list
     */
    private Set<Task> remove(int key) {
        Set<Task> tempTask = new HashSet<>();
        Set<Task> result = new HashSet<>();

        Set<Task> tasks = (Set<Task>) ringBuffer[key];
        if (tasks == null) {
            return result;
        }

        for (Task task : tasks) {
            if (task.getCycleNum() == 0) {
                result.add(task);

                size2Notify();
            } else {
                // decrement 1 cycle number and update origin data
                task.setCycleNum(task.getCycleNum() - 1);
                tempTask.add(task);
            }
            // remove task, and free the memory.
            taskMap.remove(task.getTaskId());
        }

        //update origin data
        ringBuffer[key] = tempTask;

        return result;
    }

    private void size2Notify() {
        try {
            lock.lock();
            size--;
            if (size == 0) {
                condition.signal();
            }
        } finally {
            lock.unlock();
        }
    }

    private boolean powerOf2(int target) {
        if (target < 0) {
            return false;
        }
        int value = target & (target - 1);
        if (value != 0) {
            return false;
        }

        return true;
    }

    private int mod(int target, int mod) {
        // equals target % mod
        target = target + tick.get();
        return target & (mod - 1);
    }

    private int cycleNum(int target, int mod) {
        //equals target/mod
        return target >> Integer.bitCount(mod - 1);
    }

    /**
     * An abstract class used to implement business.
     */
    public abstract static class Task extends Thread {

        private int index;

        private int cycleNum;

        private int key;

        /**
         * The unique ID of the task
         */
        private int taskId ;

        @Override
        public void run() {
        }

        public int getKey() {
            return key;
        }

        /**
         *
         * @param key Delay time(seconds)
         */
        public void setKey(int key) {
            this.key = key;
        }

        public int getCycleNum() {
            return cycleNum;
        }

        private void setCycleNum(int cycleNum) {
            this.cycleNum = cycleNum;
        }

        public int getIndex() {
            return index;
        }

        private void setIndex(int index) {
            this.index = index;
        }

        public int getTaskId() {
            return taskId;
        }

        public void setTaskId(int taskId) {
            this.taskId = taskId;
        }
    }


    private class TriggerJob implements Runnable {

        @Override
        public void run() {
            int index = 0;
            while (!stop) {
                try {
                    Set<Task> tasks = remove(index);
                    for (Task task : tasks) {
                        executorService.submit(task);
                    }

                    if (++index > bufferSize - 1) {
                        index = 0;
                    }

                    //Total tick number of records
                    tick.incrementAndGet();
                    TimeUnit.SECONDS.sleep(1);

                } catch (Exception e) {
                    logger.error("Exception", e);
                }

            }

            logger.info("Delay task has stopped");
        }
    }
}
```

接着，编写一个客户端，测试客户端

```java
public static void main(String[] args) {
    RingBufferWheel ringBufferWheel = new RingBufferWheel( Executors.newFixedThreadPool(2));
    for (int i = 0; i < 3; i++) {
        RingBufferWheel.Task job = new Job();
        job.setKey(i);
        ringBufferWheel.addTask(job);
    }
}

public static class Job extends RingBufferWheel.Task{
    @Override
    public void run() {
        System.out.println("test5");
    }
}
```

执行结果

```java
test5
test5
test5
```

如果要周期性执行任务，可以在任务执行完成之后，再重新加入到时间轮中。

详细源码分析地址：[https://crossoverjie.top/2019/09/27/algorithm/time%20wheel/]



## 2、定时注解@Schedule

Spring实现定时任务，首先说它支持的定时任务注解@Scheduled，支持cron表达式，代码内嵌简单的定时任务，例子：

```java
// 定时任务类
@Service
public class ScheduledService {

	// 每分钟执行一次
	@Scheduled(cron = "0 * * * * ?")
	public void hello(){
		System.out.println("Hello ......");
	}
} 
```

启动类开启定时任务

```java
@SpringBootApplication
@EnableAsync // 开启异步注解的支持
@EnableScheduling // 开启定时任务的支持
public class SpringBootDataStudyXjwApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBootDataStudyXjwApplication.class, args);
	}
}
```



## 3、定时框架Quartz

### Quartz的架构图

![](\assets\images\2021\juc\quartz.png)

从图中可以看出，Quartz 框架主要包括如下几个部分：

- **SchedulerFactory**：任务调度工厂，主要负责管理任务调度器
- **Scheduler**：任务调度控制器，主要是负责任务调度
- **Job**：任务接口，即被调度的任务
- **JobDetail**：Job 的任务描述类，job 执行时会依据此对象的信息反射实例化出 Job 的具体执行对象。
- **Trigger**：任务触发器，主要存放 Job 执行的时间策略。例如多久执行一次，什么时候执行，以什么频率执行等等
- **Calendar**：Trigger 扩展对象，可以排除或者包含某个指定的时间点（如排除法定节假日）
- **JobStore**：存储作业和任务调度期间的状态

看个简单的例子

```java
public class QuartzTest implements Job {
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        System.out.println(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()));
    }
    public static void main(String[] args) throws SchedulerException {
        // 创建一个Scheduler
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
        // 启动Scheduler
        scheduler.start();
        // 新建一个Job, 指定执行类是QuartzTest, 指定一个K/V类型的数据, 指定job的name和group
        JobDetail job = JobBuilder.newJob(QuartzTest.class)
                .usingJobData("jobData", "test")
                .withIdentity("myJob", "myJobGroup")
                .build();
        // 新建一个Trigger, 表示JobDetail的调度计划, 这里的cron表达式是 每1秒执行一次
        Trigger trigger = TriggerBuilder.newTrigger()
                .withIdentity("myTrigger", "myTriggerGroup")
                .startNow()
                .withSchedule(CronScheduleBuilder.cronSchedule("0/5 * * * * ?"))
                .build();
        // 让scheduler开始调度这个job, 按trigger指定的计划
        scheduler.scheduleJob(job, trigger);
    }
}
```

执行结果

```java
2020-11-09 21:38:40
2020-11-09 21:38:45
2020-11-09 21:38:50
2020-11-09 21:38:55
2020-11-09 21:39:00
2020-11-09 21:39:05
2020-11-09 21:39:10
```

整个代码虽然简单，但是五脏俱全，在应用方面使用最多的就是`Job`和`Trigger`。

### 分析源码

> 1、Job

打开`Job`源码，里面其实就是一个包含执行方法`void execute(JobExecutionContext context)`的接口，开发者只需实现接口来定义具体任务即可！

```java
public interface Job {
    void execute(JobExecutionContext context) throws JobExecutionException;
}
```

`JobExecutionContext` 类封装了获取上下文的各种信息，`Job`运行时的信息也保存在 `JobDataMap` 实例中！例如，我想要获取在上文初始化时使用到的`usingJobData("jobData", "test")`参数，可以通过如下方式进行获取！

```java
@Override
public void execute(JobExecutionContext context) throws JobExecutionException {
    //从context中获取instName，groupName以及dataMap
    String jobName = context.getJobDetail().getKey().getName();
    String groupName = context.getJobDetail().getKey().getGroup();
    JobDataMap dataMap = context.getJobDetail().getJobDataMap();
    //从dataMap中获取myDescription，myValue以及myArray
    String value = dataMap.getString("jobData");
    System.out.println("jobName:" + jobName + ",groupName:" + groupName + ",jobData:" + value);
}
```

输出结果：

```java
jobName:myJob,groupName:myJobGroup,jobData:test
```

> 2、Trigger

`Trigger`主要用于描述`Job`执行的时间触发规则，最常用的有`SimpleTrigger`和`CronTrigger`两个实现类型。

- **SimpleTrigger**：主要处理一些简单的调度规则，例如触发一次或者以固定时间间隔周期执行
- **CronTrigger**：调度处理更加灵活，可以通过`Cron`表达式定义出各种复杂时间规则的调度方案，例如每早晨9:00执行，周一、周三、周五下午5:00执行等；

用`SimpleTrigger`实现每2秒钟执行一次任务为例，代码如下：

```java
public static void main(String[] args) throws SchedulerException {
    //构建一个JobDetail实例...

    // 构建一个Trigger，指定Trigger名称和组，规定该Job立即执行，且两秒钟重复执行一次
    SimpleTrigger trigger = TriggerBuilder.newTrigger()
            .startNow() // 执行的时机，立即执行
            .withIdentity("myTrigger", "myTriggerGroup") // 不是必须的
            .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                    .withIntervalInSeconds(2).repeatForever()).build();

    // 让scheduler开始调度这个job, 按trigger指定的计划
    scheduler.scheduleJob(job, trigger);
}
```

执行结果

```java
2020-12-03 16:55:53
2020-12-03 16:55:55
2020-12-03 16:55:57
......
```

其中最关键的就是`withSchedule()`这个方法，通过`SimpleScheduleBuilder.simpleSchedule().withIntervalInSeconds(2).repeatForever()`来构建了一个简单的`SimpleTrigger`类型的任务调度规则，从而实现任务调度！

用`CronTrigger` 每隔5秒执行一次任务，代码如下：

```java
public static void main(String[] args) throws SchedulerException {
  //构建一个JobDetail实例...

  // 新建一个Trigger, 表示JobDetail的调度计划, 这里的cron表达式是 每5秒执行一次
  Trigger trigger = TriggerBuilder.newTrigger()
    .withIdentity("myTrigger", "myTriggerGroup")
    .startNow() // 立即执行
    .withSchedule(CronScheduleBuilder.cronSchedule("0/5 * * * * ?"))
    .build();

  // 让scheduler开始调度这个job, 按trigger指定的计划
  scheduler.scheduleJob(job, trigger);
```

`CronTrigger`相比`SimpleTrigger`，在配置调度规则方面，使用`cron`表达式更加灵活！

执行结果：

```java
2020-12-03 17:09:10
2020-12-03 17:09:15
2020-12-03 17:09:20
......
```

> 3、cron表达式

在我这篇文章[http://139.199.13.139/blog/icoding-edu/2020/03/22/icoding-note-012.html](http://139.199.13.139/blog/icoding-edu/2020/03/22/icoding-note-012.html)  @schedule注解也有使用cron表达式

```sh
.---------------------- seconds(0 - 59)
|  .------------------- minute (0 - 59)
|  |  .---------------- hour (0 - 23)
|  |  |  .------------- day of month (1 - 31)
|  |  |  |  .---------- month (1 - 12)
|  |  |  |  |  .------- Day-of-Week (1 - 7) 
|  |  |  |  |  |  .---- year (1970 - 2099) ...
|  |  |  |  |  |  |
*  *  *  *  *  ?  *
```

![](\assets\images\2021\juc\quartz-crontrigger.png)

在 cron 表达式中不区分大小写，更多配置和使用操作可以参考这里。

还可以在线解析`cron`表达式进行测试。

![](\assets\images\2021\juc\cron-expression-online.jpg)

### 监听器

quartz 除了提供能正常调度任务的功能之外，还提供了监听器功能！

所谓监听器，其实你可以把它理解为类似`Spring Aop`的功能，可以对全局或者局部实现监听！

监听器应用，在实际项目中并不常用，但是在某些业务场景下，可以发挥一定的作用，例如：你想在任务处理完成之后，去发送邮件或者发短信进行通知，但是你又不想改以前的代码，这个时候就可以在监听器里面完成改项任务！

quartz 监听器主要分三大类：

- SchedulerListener：任务调度监听器
- TriggerListener：任务触发监听器
- JobListener：任务执行监听器

> 1、SchedulerListener：任务调度监听器

`SchedulerListener`监听器，主要对任务调度`Scheduler`生命周期中关键节点进行监听，它只能全局进行监听，简单示例如下：

```java
public class SimpleSchedulerListener implements SchedulerListener {

    @Override
    public void jobScheduled(Trigger trigger) {
        System.out.println("任务被部署时被执行");
    }

    @Override
    public void jobUnscheduled(TriggerKey triggerKey) {
        System.out.println("任务被卸载时被执行");
    }

    @Override
    public void triggerFinalized(Trigger trigger) {
        System.out.println("任务完成了它的使命，光荣退休时被执行");
    }

    @Override
    public void triggerPaused(TriggerKey triggerKey) {
        System.out.println(triggerKey + "（一个触发器）被暂停时被执行");
    }

    @Override
    public void triggersPaused(String triggerGroup) {
        System.out.println(triggerGroup + "所在组的全部触发器被停止时被执行");
    }

    @Override
    public void triggerResumed(TriggerKey triggerKey) {
        System.out.println(triggerKey + "（一个触发器）被恢复时被执行");
    }

    @Override
    public void triggersResumed(String triggerGroup) {
        System.out.println(triggerGroup + "所在组的全部触发器被回复时被执行");
    }

    @Override
    public void jobAdded(JobDetail jobDetail) {
        System.out.println("一个JobDetail被动态添加进来");
    }

    @Override
    public void jobDeleted(JobKey jobKey) {
        System.out.println(jobKey + "被删除时被执行");
    }

    @Override
    public void jobPaused(JobKey jobKey) {
        System.out.println(jobKey + "被暂停时被执行");
    }

    @Override
    public void jobsPaused(String jobGroup) {
        System.out.println(jobGroup + "(一组任务）被暂停时被执行");
    }

    @Override
    public void jobResumed(JobKey jobKey) {
        System.out.println(jobKey + "被恢复时被执行");
    }

    @Override
    public void jobsResumed(String jobGroup) {
        System.out.println(jobGroup + "(一组任务）被恢复时被执行");
    }

    @Override
    public void schedulerError(String msg, SchedulerException cause) {
        System.out.println("出现异常" + msg + "时被执行");
        cause.printStackTrace();
    }

    @Override
    public void schedulerInStandbyMode() {
        System.out.println("scheduler被设为standBy等候模式时被执行");
    }

    @Override
    public void schedulerStarted() {
        System.out.println("scheduler启动时被执行");
    }

    @Override
    public void schedulerStarting() {
        System.out.println("scheduler正在启动时被执行");
    }

    @Override
    public void schedulerShutdown() {
        System.out.println("scheduler关闭时被执行");
    }

    @Override
    public void schedulerShuttingdown() {
        System.out.println("scheduler正在关闭时被执行");
    }

    @Override
    public void schedulingDataCleared() {
        System.out.println("scheduler中所有数据包括jobs, triggers和calendars都被清空时被执行");
    }
}
```

需要在任务调度器启动前，将`SimpleSchedulerListener`注册到`Scheduler`容器中！

```sh
// 创建一个Scheduler
Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
//添加SchedulerListener监听器
scheduler.getListenerManager().addSchedulerListener(new SimpleSchedulerListener());
// 启动Scheduler
scheduler.start();
```

运行`main`方法，输出结果如下：

```java
scheduler正在启动时被执行
scheduler启动时被执行
一个JobDetail被动态添加进来
任务被部署时被执行
2020-12-10 17:27:10
....
```

> 2、TriggerListener：任务触发监听器

`TriggerListener`，与触发器`Trigger`相关的事件都会被监听，它既可以全局监听，也可以实现局部监听。所谓局部监听，就是对某个`Trigger`的名称或者组进行监听，简单示例如下：

```java
public class SimpleTriggerListener implements TriggerListener {

    /**
     * Trigger监听器的名称
     * @return
     */
    @Override
    public String getName() {
        return "mySimpleTriggerListener";
    }

    /**
     * Trigger被激发 它关联的job即将被运行
     * @param trigger
     * @param context
     */
    @Override
    public void triggerFired(Trigger trigger, JobExecutionContext context) {
        System.out.println("myTriggerListener.triggerFired()");
    }

    /**
     * Trigger被激发 它关联的job即将被运行, TriggerListener 给了一个选择去否决 Job 的执行,如果返回TRUE 那么任务job会被终止
     * @param trigger
     * @param context
     * @return
     */
    @Override
    public boolean vetoJobExecution(Trigger trigger, JobExecutionContext context) {
        System.out.println("myTriggerListener.vetoJobExecution()");
        return false;
    }

    /**
     * 当Trigger错过被激发时执行,比如当前时间有很多触发器都需要执行，但是线程池中的有效线程都在工作，
     * 那么有的触发器就有可能超时，错过这一轮的触发。
     * @param trigger
     */
    @Override
    public void triggerMisfired(Trigger trigger) {
        System.out.println("myTriggerListener.triggerMisfired()");
    }

    /**
     * 任务完成时触发
     * @param trigger
     * @param context
     * @param triggerInstructionCode
     */
    @Override
    public void triggerComplete(Trigger trigger, JobExecutionContext context, Trigger.CompletedExecutionInstruction triggerInstructionCode) {
        System.out.println("myTriggerListener.triggerComplete()");
    }
}
```

同样，需要将`SimpleTriggerListener`注册到`Scheduler`容器中

```java
// 创建一个Scheduler
Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
// 创建并注册一个全局的Trigger Listener
scheduler.getListenerManager().addTriggerListener(new SimpleTriggerListener(), EverythingMatcher.allTriggers());
        
// 创建并注册一个局部的Trigger Listener
//scheduler.getListenerManager().addTriggerListener(new SimpleTriggerListener(), KeyMatcher.keyEquals(TriggerKey.triggerKey("myTrigger", "myJobTrigger")));
        
// 创建并注册一个特定组的Trigger Listener
//scheduler.getListenerManager().addTriggerListener(new SimpleTriggerListener(), GroupMatcher.groupEquals("myTrigger"));
// 启动Scheduler
scheduler.start();
```

> 3、JobListener 任务执行监听器

`JobListener`，与任务执行`Job`相关的事件都会被监听，和`Trigger`一样，既可以全局监听，也可以实现局部监听。

```java
public class SimpleJobListener implements JobListener {

    /**
     * job监听器名称
     * @return
     */
    @Override
    public String getName() {
        return "mySimpleJobListener";
    }

    /**
     * 任务被调度前
     * @param context
     */
    @Override
    public void jobToBeExecuted(JobExecutionContext context) {
        System.out.println("simpleJobListener监听器，准备执行："+context.getJobDetail().getKey());
    }

    /**
     * 任务调度被拒了
     * @param context
     */
    @Override
    public void jobExecutionVetoed(JobExecutionContext context) {
        System.out.println("simpleJobListener监听器，取消执行："+context.getJobDetail().getKey());
    }

    /**
     * 任务被调度后
     * @param context
     * @param jobException
     */
    @Override
    public void jobWasExecuted(JobExecutionContext context, JobExecutionException jobException) {
        System.out.println("simpleJobListener监听器，执行结束："+context.getJobDetail().getKey());
    }
}
```

同样的，将`SimpleJobListener`注册到`Scheduler`容器中，即可实现监听！

```java
// 创建一个Scheduler
Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
// 创建并注册一个全局的Job Listener
scheduler.getListenerManager().addJobListener(new SimpleJobListener(), EverythingMatcher.allJobs());

// 创建并注册一个指定任务的Job Listener
//scheduler.getListenerManager().addJobListener(new SimpleJobListener(), KeyMatcher.keyEquals(JobKey.jobKey("myJob", "myJobGroup")));

// 将同一任务组的任务注册到监听器中
//scheduler.getListenerManager().addJobListener(new SimpleJobListener(), GroupMatcher.jobGroupEquals("myJobGroup"));

// 启动Scheduler
scheduler.start();
```

**小结**

如果想多个同时监听，将其依次加入即可！

```java
Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();

//添加SchedulerListener监听器
scheduler.getListenerManager().addSchedulerListener(new SimpleSchedulerListener());

// 创建并注册一个全局的Trigger Listener
scheduler.getListenerManager().addTriggerListener(new SimpleTriggerListener(), EverythingMatcher.allTriggers());

// 创建并注册一个全局的Job Listener
scheduler.getListenerManager().addJobListener(new SimpleJobListener(), EverythingMatcher.allJobs());

// 启动Scheduler
scheduler.start();
```

### SpringBoot 整合

第二个用过的就是quartz，主要有三个核心概念

- scheduler 调度器
- trigger 触发器
- job 任务

> 单体应用介绍

1、pom.xml导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

2、编写具体任务

```java
public class TestTask implements Job {

  @Override
  public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
    System.out.println("testTask：" + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()));
  }
}
```

3、调度配置类

```java
@Configuration
public class TestTaskConfig {
    @Bean
    public JobDetail testQuartz() {
        return JobBuilder.newJob(TestTask.class)
                .usingJobData("jobData", "test")
                .withIdentity("myJob", "myJobGroup")
                .storeDurably()
                .build();
    }

    @Bean
    public Trigger testQuartzTrigger() {
        //5秒执行一次
        return TriggerBuilder.newTrigger()
                .forJob(testQuartz())
                .withIdentity("myTrigger", "myTriggerGroup")
                .startNow()
                .withSchedule(CronScheduleBuilder.cronSchedule("0/5 * * * * ?"))
                .build();
    }
}
```

4、启动测试

![](\assets\images\2021\juc\testQuartz.jpg)

如果需要创建多个定时任务，配置流程也类似！但是这个是静态的。我们需要改成动态配置的

> 动态配置定时任务

从上面的代码中我们可以分析中，定时任务的有三个核心变量，其他的方法都可以封装成公共的。

- 任务名称：例如`myJob`
- 任务执行类：例如`TestTask.class`
- 任务调度时间：例如`0/5 * * * * ?`

基于此规律，我们可以创建一个定时任务实体类，用于保存定时任务相关信息到数据库当中，然后编写一个定时任务工具库，用于创建、更新、删除、暂停任务操作，通过 restful 接口操作将任务存入数据库并管理定时任务！

这里ORM框架使用Jpa

1、导入Jpa依赖

```xml
<!--jpa 支持-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<!--mysql 数据源-->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

在`application.properties`中加入数据源配置

```properties
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/test
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
```

2、创建任务配置表

```sql
CREATE TABLE `tb_job_task` (
  `id` varchar(50)  NOT NULL COMMENT '任务ID',
  `job_name` varchar(100)  DEFAULT NULL COMMENT '任务名称',
  `job_class` varchar(200)  DEFAULT NULL COMMENT '任务执行类',
  `cron_expression` varchar(50)  DEFAULT NULL COMMENT '任务调度时间表达式',
  `status` int(4) DEFAULT NULL COMMENT '任务状态，0：启动，1：暂停，2：停用',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

3、编写实体类

```java
@Entity
@Table(name = "tb_job_task")
public class QuartzBean {

    /**
     * 任务ID
     */
    @Id
    private String id;

    /**
     * 任务名称
     */
    @Column(name = "job_name")
    private String jobName;

    /**
     * 任务执行类
     */
    @Column(name = "job_class")
    private String jobClass;

    /**
     * 任务调度时间表达式
     */
    @Column(name = "cron_expression")
    private String cronExpression;

    /**
     * 任务状态，0：启动，1：暂停，2：停用
     */
    @Column(name = "status")
    private Integer status;

    //get、set...
}
```

4、编写任务操作工具类

```java
public class QuartzUtils {

  private static final Logger log = LoggerFactory.getLogger(QuartzUtils.class);

  /**
     * 创建定时任务 定时任务创建之后默认启动状态
     * @param scheduler   调度器
     * @param quartzBean  定时任务信息类
     * @throws Exception
     */
  public static void createScheduleJob(Scheduler scheduler, QuartzBean quartzBean){
    try {
      //获取到定时任务的执行类  必须是类的绝对路径名称
      //定时任务类需要是job类的具体实现 QuartzJobBean是job的抽象类。
      Class<? extends Job> jobClass = (Class<? extends Job>) Class.forName(quartzBean.getJobClass());
      // 构建定时任务信息
      JobDetail jobDetail = JobBuilder.newJob(jobClass).withIdentity(quartzBean.getJobName()).build();
      // 设置定时任务执行方式
      CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(quartzBean.getCronExpression());
      // 构建触发器trigger
      CronTrigger trigger = TriggerBuilder.newTrigger().withIdentity(quartzBean.getJobName()).withSchedule(scheduleBuilder).build();
      scheduler.scheduleJob(jobDetail, trigger);
    } catch (ClassNotFoundException e) {
      log.error("定时任务类路径出错：请输入类的绝对路径", e);
    } catch (SchedulerException e) {
      log.error("创建定时任务出错", e);
    }
  }

  /**
     * 根据任务名称暂停定时任务
     * @param scheduler  调度器
     * @param jobName    定时任务名称
     * @throws SchedulerException
     */
  public static void pauseScheduleJob(Scheduler scheduler, String jobName){
    JobKey jobKey = JobKey.jobKey(jobName);
    try {
      scheduler.pauseJob(jobKey);
    } catch (SchedulerException e) {
      log.error("暂停定时任务出错", e);
    }
  }

  /**
     * 根据任务名称恢复定时任务
     * @param scheduler  调度器
     * @param jobName    定时任务名称
     * @throws SchedulerException
     */
  public static void resumeScheduleJob(Scheduler scheduler, String jobName) {
    JobKey jobKey = JobKey.jobKey(jobName);
    try {
      scheduler.resumeJob(jobKey);
    } catch (SchedulerException e) {
      log.error("暂停定时任务出错", e);
    }
  }

  /**
     * 根据任务名称立即运行一次定时任务
     * @param scheduler     调度器
     * @param jobName       定时任务名称
     * @throws SchedulerException
     */
  public static void runOnce(Scheduler scheduler, String jobName){
    JobKey jobKey = JobKey.jobKey(jobName);
    try {
      scheduler.triggerJob(jobKey);
    } catch (SchedulerException e) {
      log.error("运行定时任务出错", e);
    }
  }

  /**
     * 更新定时任务
     * @param scheduler   调度器
     * @param quartzBean  定时任务信息类
     * @throws SchedulerException
     */
  public static void updateScheduleJob(Scheduler scheduler, QuartzBean quartzBean){
    try {
      //获取到对应任务的触发器
      TriggerKey triggerKey = TriggerKey.triggerKey(quartzBean.getJobName());
      //设置定时任务执行方式
      CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(quartzBean.getCronExpression());
      //重新构建任务的触发器trigger
      CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);
      trigger = trigger.getTriggerBuilder().withIdentity(triggerKey).withSchedule(scheduleBuilder).build();
      //重置对应的job
      scheduler.rescheduleJob(triggerKey, trigger);
    } catch (SchedulerException e) {
      log.error("更新定时任务出错", e);
    }
  }

  /**
     * 根据定时任务名称从调度器当中删除定时任务
     * @param scheduler 调度器
     * @param jobName   定时任务名称
     * @throws SchedulerException
     */
  public static void deleteScheduleJob(Scheduler scheduler, String jobName) {
    JobKey jobKey = JobKey.jobKey(jobName);
    try {
      scheduler.deleteJob(jobKey);
    } catch (SchedulerException e) {
      log.error("删除定时任务出错", e);
    }
  }
}
```

5、编写Jpa数据操作（DAO层）

```java
public interface QuartzBeanRepository extends JpaRepository<QuartzBean,String> {
  /**
     * 修改任务状态
     * @param id
     * @param status
     */
  @Modifying
  @Transactional
  @Query("update QuartzBean m set m.status = ?2 where m.id =?1")
  void updateState(String id, Integer status);

  /**
     * 修改任务调度时间
     * @param id
     * @param cronExpression
     */
  @Modifying
  @Transactional
  @Query("update QuartzBean m set m.cronExpression = ?2 where m.id =?1")
  void updateCron(String id, String cronExpression);
}
```

6、编写WEB控制层controller服务

```java
@RestController
@RequestMapping("/quartz")
public class QuartzController {

    private static final Logger log = LoggerFactory.getLogger(QuartzController.class);

    @Autowired
    private Scheduler scheduler;

    @Autowired
    private QuartzBeanRepository repository;

    /**
     * 创建任务
     * @param quartzBean
     */
    @RequestMapping("/createJob")
    public HttpStatus createJob(@RequestBody  QuartzBean quartzBean)  {
        log.info("=========开始创建任务=========");
        QuartzUtils.createScheduleJob(scheduler,quartzBean);
        repository.save(quartzBean.setId(UUID.randomUUID().toString()).setStatus(0));
        log.info("=========创建任务成功，信息：{}", JSON.toJSONString(quartzBean));
        return HttpStatus.OK;
    }

    /**
     * 暂停任务
     * @param quartzBean
     */
    @RequestMapping("/pauseJob")
    public HttpStatus pauseJob(@RequestBody QuartzBean quartzBean) {
        log.info("=========开始暂停任务，请求参数：{}=========", JSON.toJSONString(quartzBean));
        QuartzUtils.pauseScheduleJob(scheduler, quartzBean.getJobName());
        repository.updateState(quartzBean.getId(), 1);
        log.info("=========暂停任务成功=========");
        return HttpStatus.OK;
    }

    /**
     * 立即运行一次定时任务
     * @param quartzBean
     */
    @RequestMapping("/runOnce")
    public HttpStatus runOnce(@RequestBody QuartzBean quartzBean) {
        log.info("=========立即运行一次定时任务，请求参数：{}", JSON.toJSONString(quartzBean));
        QuartzUtils.runOnce(scheduler,quartzBean.getJobName());
        log.info("=========立即运行一次定时任务成功=========");
        return HttpStatus.OK;
    }

    /**
     * 恢复定时任务
     * @param quartzBean
     */
    @RequestMapping("/resume")
    public HttpStatus resume(@RequestBody QuartzBean quartzBean) {
        log.info("=========恢复定时任务，请求参数：{}", JSON.toJSONString(quartzBean));
        QuartzUtils.resumeScheduleJob(scheduler,quartzBean.getJobName());
        repository.updateState(quartzBean.getId(), 0);
        log.info("=========恢复定时任务成功=========");
        return HttpStatus.OK;
    }

    /**
     * 更新定时任务
     * @param quartzBean
     */
    @RequestMapping("/update")
    public HttpStatus update(@RequestBody QuartzBean quartzBean)  {
        log.info("=========更新定时任务，请求参数：{}", JSON.toJSONString(quartzBean));
        QuartzUtils.updateScheduleJob(scheduler,quartzBean);
        repository.updateCron(quartzBean.getId(), quartzBean.getCronExpression());
        log.info("=========更新定时任务成功=========");
        return HttpStatus.OK;
    }

    /**
     * 删除定时任务
     * @param quartzBean
     */
    @RequestMapping("/delete")
    public HttpStatus delete(@RequestBody QuartzBean quartzBean)  {
        log.info("=========删除定时任务，请求参数：{}", JSON.toJSONString(quartzBean));
        QuartzUtils.deleteScheduleJob(scheduler,quartzBean.getJobName());
        repository.updateState(quartzBean.getId(), 2);
        log.info("=========删除定时任务成功=========");
        return HttpStatus.OK;
    }
}
```

**服务重启补偿**

在应用程序正常运行的时候，虽然没问题，但是当我们重启服务的时候，这个时候内存的里面的定时任务其实全部都被销毁，因此在应用程序启动的时候，还需要将正常的任务重新加入到服务中！

```java
@Component
public class TaskConfigApplicationRunner implements ApplicationRunner {

    @Autowired
    private QuartzBeanRepository repository;

    @Autowired
    private Scheduler scheduler;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        List<QuartzBean> list = repository.findAll();
        if(!CollectionUtils.isEmpty(list)){
            for (QuartzBean quartzBean : list) {
                //加载启动类型的定时任务
                if(quartzBean.getStatus().intValue() == 0){
                    QuartzUtils.createScheduleJob(scheduler,quartzBean);
                }
                //加载暂停类型的定时任务
                if(quartzBean.getStatus().intValue() == 1){
                    QuartzUtils.createScheduleJob(scheduler,quartzBean);
                    QuartzUtils.pauseScheduleJob(scheduler, quartzBean.getJobName());
                }
            }
        }
    }
}
```

在服务重启的时候，会重新将有效任务加入quartz 中！

7、接口服务测试

使用postman调用controller层的接口方法测试新增、更新任务等

8、添加监听器（选用）

当然，如果你想对某个任务实现监听，只需要添加一个配置类，将其注入即可！

```java
@Configuration
public class TestTaskConfig {

  @Primary
  @Bean
  public Scheduler initScheduler() throws SchedulerException {
    Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
    //添加SchedulerListener监听器
    scheduler.getListenerManager().addSchedulerListener(new SimpleSchedulerListener());

    // 创建并注册一个全局的Trigger Listener
    scheduler.getListenerManager().addTriggerListener(new SimpleTriggerListener(), EverythingMatcher.allTriggers());

    // 创建并注册一个全局的Job Listener
    scheduler.getListenerManager().addJobListener(new SimpleJobListener(), EverythingMatcher.allJobs());
    scheduler.start();
    return scheduler;
  }
}
```

**小结**

需要注意的是：在 quartz 任务暂停之后再次启动时，会立即执行一次，在更新之后也会立即执行一次任务调度！



> 使用示例

之前使用renren_fast框架中的定时器也是使用quartz框架，有界面可以动态启动、停止任务，任务被执行时用到了反射，下面是它的核心代码。

1、导入依赖

```xml
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.2.1</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
</dependency>
```

2、配置类QuartzConfigration

```java
@Configuration
public class QuartzConfigration {
    @Autowired
    private JobFactory jobFactory;

    @Bean
    public SchedulerFactoryBean schedulerFactoryBean() {
        SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
        try {
            schedulerFactoryBean.setOverwriteExistingJobs(true);
            schedulerFactoryBean.setQuartzProperties(quartzProperties());
            schedulerFactoryBean.setJobFactory(jobFactory);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return schedulerFactoryBean;
    }

    // 指定quartz.properties，可在配置文件中配置相关属性
    @Bean
    public Properties quartzProperties() throws IOException {
        PropertiesFactoryBean propertiesFactoryBean = new PropertiesFactoryBean();
        propertiesFactoryBean.setLocation(new ClassPathResource("/config/quartz.properties"));
        propertiesFactoryBean.afterPropertiesSet();
        return propertiesFactoryBean.getObject();
    }

    // 创建schedule
    @Bean(name = "scheduler")
    public Scheduler scheduler() {
        return schedulerFactoryBean().getScheduler();
    }
}
```



### 集群环境



