---
title: Java线程池分析
tags:
  - 并发
  - 线程池
categories:
  - java基础
  - 并发编程
date: 2018-12-20 16:37:46
---

## 引言

在并发编程中，我们经常会使用到线程池，当然我们也可以手动一个一个创建线程，那么为何我们还是推崇大家使用线程池进行并发编程呢？借用《Java并发编程的艺术》提到的来说一下使用线程池的优点有3个

* 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
* 提高响应速度。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
* 提高线程的可管理性。**线程是稀缺资源**，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。


## ThreadPoolExecutor类

在日常使用中，大多数情况我们都会使用JDK提供的`Executors`去创建线程池，`Executors`利用工厂模式向我们提供了4种线程池实现方式，但是最新的阿里Java开发手册中明确说明了不建议使用`Executors`去创建线程池，原因是使用`Executors`创建线程池会使用很多默认值，默认使用的参数有时候是不合理，但是开发者往往会忽略。所以我们应该尽量使用`ThreadPoolExecutor`类来显示的创建线程池。

下面我们先来看下`ThreadPoolExecutor`类的继承结构

![image.png](https://upload-images.jianshu.io/upload_images/2717496-0812461a560d9dc0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
￼

* Executor接口只含有一个`execute(Runnable command)`方法，加入一个新的线程
* ExecutorService接口相比Executor多个几个方法

 ```
 public interface ExecutorService extends Executor {
    void shutdown();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
 
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
 ```

我们直接看`ThreadPoolExecutor`所提供的构造方法

```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

现在我们来解释一下各个参数的含义

* corePoolSize 核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系
* maximumPoolSize 线程池最大线程数
* keepAliveTime 空闲线程的存活时间，默认只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用
* unit 参数keepAliveTime的时间单位
* workQueue 一个阻塞队列，用来存储等待执行的任务
* threadFactory 线程工厂，主要用来创建线程
* handler 表示当拒绝处理任务时的策略，有以下四种取值

```
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
```

下面再简单再介绍一下`ThreadPoolExecutor`中几个重要的方法

* execute 这个方法其实就是对`Executor`接口的实现，也是最核心的方法，用于向线程池中提交一个任务执行
* submit 该方法也是用于向线程池里提交任务，区别在于`submit`提交可以有返回值，会返回一个`Futrue`类型来异步获取执行结果
* shutdown 用于关闭线程池
* shutdownNow 也是关闭线程池，与`shutdown`区别在于`shutdownNow`会尝试关闭正在执行的线程任务

## 通过源码分析 ThreadPoolExecutor

上文我们简单的分析了`ThreadPoolExecutor`，下面我们将根据源码来进一步分析`ThreadPoolExecutor`核心功能的实现

先从其核心方法`execute`开始解读

```
public void execute(Runnable command) {
	  // 如果传入空对象 则抛出空指针异常
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    
    /**
     * 如果当前正在执行任务的线程数小于 corePoolSize
     */
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    /**
     * 如果当前正在执行任务的线程数大于corePoolSize，且workQueue未满
     */
    if (isRunning(c) && workQueue.offer(command)) {
    		// 重复检查一次，防止正好此时线程池被shutdown
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    /**
     * 如果当前正在执行的任务线程数大于corePoolSize，且workQueue已满，则使用		* maximumPoolSize参数创建线程，如果已经到了maximumPoolSize最大值，则启用拒绝		* 策略
     */
    else if (!addWorker(command, false))
        reject(command);
}
```
上面的代码简单总结一下就是

1. 当前正在执行任务的线程数小于`corePoolSize`时，正常加入执行
2. 当前正在执行任务的线程数大于`corePoolSize`，且`workQueue`未满时，则将任务加入等待队列`workQueue`中
3. 如果当前正在执行的任务线程数大于`corePoolSize`，且`workQueue`已满，则会临时扩充线程数，根据`maximumPoolSize`最大线程数值
4. 如果当前正在执行的任务线程数大于`maximumPoolSize` 这时已经超出最大可承受的线程数值了，会启用拒绝策略，也就是上文所配置的四种策略之一，默认是抛出异常，加入任务失败


## workQueue 任务队列

上文有提到任务队列，也就是在等待执行的任务队列。`workQueue`的本质是一种阻塞队列BlockingQueue<Runnable>，列举三种常用类型

1. 基于数组的先进先出队列，此队列创建时必须指定大小
2. 基于链表的先进先出队列
3. 同步队列 该队列不存储元素，每个插入操作必须等待另一个线程调用移除操作，否则插入操作会一直阻塞


## 核心功能分析

上文中我们通过查看源码知道了新任务加入线程池的策略，下面我们继续往下看，再分析之前，我们可以先思考几个问题

1. 任务等待队列是在何时被执行？
2. 线程是如何实现重复利用的？，毕竟这是线程池最重要的功能了
3. 空闲线程根据`keepAliveTime`参数是在哪里被回收的？

接下来我们继续通过源码来解读这三个问题
根据上文`execute`方法源码，可以看到调用了`addWorker`方法

```
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            // 这里判断当前线程数是否大于最大值，大于了则返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
    		// 新建了一个 Worker
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // 加锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 把当前这个工作线程加入到 workers 集合中
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
            	  // 开始执行此worker
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

* 根据参数core 判断当前线程数是否已经大于`corePoolSize`或`maximumPoolSize`，大于则直接返回`false`
* 初始化了一个`Worker`对象，并且加入到了`workers`集合中，`workers`是一个`Set`类型的集合
* 启动当前这个`Worker`线程任务

经过上面的分析，可能大概会有疑惑，`Worker`对象是做啥的？下面我们就来看下`Worker`类的实现

```
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }
    
    ...
}
```

* `Worker`实现了`Runnable`接口
* 构造方法中通过`ThreadFactory`初始化了一个新的线程对象
* run()方法调用了`ThreadPoolExecutor`的runWorker()

重点在`runWorker`方法中

### 线程池重复利用机制

```
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    // 获取第一个任务
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
    		// 从任务队列中循环获取任务执行 getTask方法有可能会阻塞
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
            	  // 空实现，可以通过继承`ThreadPoolExecutor`重写此方法
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                		// 执行
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

代码解析到这里，基本可以回答上文所提出的三个问题了

* 加入到任务队列的任务会在这里被执行，因为`Worker`对象执行完一个任务后，并不会立刻结束，而是会通过循环调用`getTask()`方法从任务队列中获取最新任务来执行，这样子就实现了线程的重复利用，而不必每次都重新创建一个`Worker`对象
* 当`getTask()`方法无法获取到最新的任务后，该`Worker`线程才会被回收。所以第三个问题`keepAliveTime`的机制必然是在`getTask()`中实现的


### getTask 超时策略

```
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // workers大于corePoolSize，或则允许corePoolSize设置空闲超时时间
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
			
			// 当前线程数已经大于maximumPoolSize或则已经超时过一次，则直接返回null
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
        		 // 获取任务，如果timed为true，则等待一定时间（keepAliveTime）未返回的话，会返回null，如果timed未设置为true，则会一直阻塞，直到有数据
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

* 先判断当前线程池状态，如果`runState`大于`SHUTDOWN`（即为STOP或者TERMINATED），则直接返回null
* 判断当前线程池的线程数是否已经大于核心池大小`corePoolSize`或者允许为核心池中的线程设置空闲存活时间，则调用`workQueue.poll(time,timeUnit)`来取任务，这个方法会阻塞等待一定的时间，如果取不到任务就返回null，否则则会调用`workQueue.take()`来取任务，这个方法会`wait`释放CPU一直阻塞。

> 总结一下，只有当我们设置`allowCoreThreadTimeOut`（核心池线程空闲超时时间）或则当前线程数大于`corePoolSize`时，`keepAliveTime`机制才会生效

整个流程到这里大概就分析完了，下图基本绘制了线程池提交任务，执行任务的整个流程

![image.png](https://upload-images.jianshu.io/upload_images/2717496-2bcb9a98382451a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## Executors 中几个常用的线程池

虽然阿里Java开发规约中不建议我们使用`Executors`类直接创建线程池，但还是有必要简单介绍几个其中常用的方法

* `Executors.newCachedThreadPool()` 创建一个缓冲池，缓冲池容量大小为Integer.MAX_VALUE. 将`corePoolSize`设置为0，将`maximumPoolSize`设置为`Integer.MAX_VALUE`，`workQueue`使用的`SynchronousQueue`，也就是说来了任务就创建线程运行，**当线程空闲超过60秒，就销毁线程**
* `Executors.newSingleThreadExecutor()` 创建容量为1的缓冲池，将`corePoolSize`和`maximumPoolSize`都设置为1，`workQueue`使用的`LinkedBlockingQueue`
* `Executors.newFixedThreadPool(int)` 创建固定容量大小的缓冲池，空闲线程不会销毁。`corePoolSize`和`maximumPoolSize`值是相等的，`workQueue`使用的是`LinkedBlockingQueue`


## 总结

本篇文章主要从对线程池的配置使用，以及源码实现做了分析，总体上比较全面的分析了Java线程池的实现。



