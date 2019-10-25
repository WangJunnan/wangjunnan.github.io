---
title: Sentinel源码解析三（滑动窗口流量统计）
tags:
  - 并发
categories:
  - java框架
  - Sentinel
date: 2019-10-25 16:55:44
---

## 前言

`Sentinel`的核心功能之一是流量统计，例如我们常用的指标QPS，当前线程数等。上一篇文章中我们已经大致提到了提供数据统计功能的`Slot（StatisticSlot）`，`StatisticSlot`在`Sentinel`的整个体系中扮演了一个非常重要的角色，后续的一系列操作（限流，熔断）等都依赖于`StatisticSlot`所统计出的数据。

本文所要讨论的重点就是`StatisticSlot`是如何做的流量统计？

其实在之前介绍常用限流算法[常用限流算法](http://wangjunnan.club/2019/10/09/%E6%B5%85%E8%B0%88%E5%B8%B8%E7%94%A8%E7%9A%84%E9%99%90%E6%B5%81%E7%AE%97%E6%B3%95/)的时候已经有提到过一个算法`滑动窗口限流`，该算法的滑动窗口原理其实跟`Sentinel`所提供的流量统计原理是一样的，都是基于时间窗口的滑动统计


## 回到StatisticSlot

```java
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {

	...
	// 当前请求线程数加一
	node.increaseThreadNum();
	// 新增请求数
	node.addPassRequest(count);
	...
}
```
可以看到`StatisticSlot`主要统计了两种类型的数据

1. 线程数
2. 请求数（QPS）

对于线程数的统计比较简单，通过内部维护一个`LongAdder`来进行当前线程数的统计，每进入一个请求加1，每释放一个请求减1，从而得到当前的线程数

对于请求数QPS的统计则相对比较复杂，其中有用到滑动窗口原理（也是本文的重点），下面根据源码来深入的分析

## DefaultNode 和 StatisticNode

```java
public void addPassRequest(int count) {
	  // 调用父类（StatisticNode）来进行统计
    super.addPassRequest(count);
    // 根据clusterNode 汇总统计（背后也是调用父类StatisticNode）
    this.clusterNode.addPassRequest(count);
}
```
最终都是调用了父类`StatisticNode`的`addPassRequest`方法

```java
/**
 * 按秒统计，分成两个窗口，每个窗口500ms，用来统计QPS
 */
private transient volatile Metric rollingCounterInSecond = new ArrayMetric(SampleCountProperty.SAMPLE_COUNT,
    IntervalProperty.INTERVAL);

/**
 * 按分钟统计，分成60个窗口，每个窗口 1000ms
 */
private transient Metric rollingCounterInMinute = new ArrayMetric(60, 60 * 1000, false);

public void addPassRequest(int count) {
    rollingCounterInSecond.addPass(count);
    rollingCounterInMinute.addPass(count);
}
```

代码比较简单，可以知道内部是调用了`ArrayMetric`的`addPass`方法来统计的，并且统计了两种不同时间维度的数据(秒级和分钟级)

## ArrayMetric

```java
private final LeapArray<MetricBucket> data;

public ArrayMetric(int sampleCount, int intervalInMs) {
    this.data = new OccupiableBucketLeapArray(sampleCount, intervalInMs);
}

public ArrayMetric(int sampleCount, int intervalInMs, boolean enableOccupy) {
    if (enableOccupy) {
        this.data = new OccupiableBucketLeapArray(sampleCount, intervalInMs);
    } else {
        this.data = new BucketLeapArray(sampleCount, intervalInMs);
    }
}

public void addPass(int count) {
    // 1. 获取当前窗口
    WindowWrap<MetricBucket> wrap = data.currentWindow();
    // 2. 当前窗口加1
    wrap.value().addPass(count);
}
```
`ArrayMetric`其实也是一个包装类，内部通过实例化`LeapArray`的对应实现类，来实现具体的统计逻辑，`LeapArray`是一个抽象类，`OccupiableBucketLeapArray`和`BucketLeapArray`都是其具体的实现类
`OccupiableBucketLeapArray`在1.5版本之后才被引入，**主要是为了解决一些高优先级的请求在限流触发的时候也能通过(通过占用未来时间窗口的名额来实现)** 也是默认使用的LeapArray实现类

而统计的逻辑也比较清楚，分成了两步：

1. 定位到当前窗口
2. 获取到当前窗口`WindowWrap`的`MetricBucket`并执行`addPass`逻辑

这里我们先看下第二步中的`MetricBucket`类，看看它做了哪些事情

### MetricBucket

```java
/**
 * 存放当前窗口各种类型的统计值(类型包括 PASS BLOCK EXCEPTION 等)
 */
private final LongAdder[] counters;
public MetricBucket() {
    MetricEvent[] events = MetricEvent.values();
    this.counters = new LongAdder[events.length];
    for (MetricEvent event : events) {
        counters[event.ordinal()] = new LongAdder();
    }
    initMinRt();
}
// 统计pass数
public void addPass(int n) {
    add(MetricEvent.PASS, n);
}
// 统计可占用的pass数
public void addOccupiedPass(int n) {
    add(MetricEvent.OCCUPIED_PASS, n);
}
// 统计异常数
public void addException(int n) {
    add(MetricEvent.EXCEPTION, n);
}
// 统计block数
public void addBlock(int n) {
    add(MetricEvent.BLOCK, n);
}
....
```
MetricBucket通过定义了一个`LongAdder`数组来存储不同类型的流量统计值，具体的类型则都定义在`MetricEvent`枚举中。
执行`addPass`方法对应`LongAdder`数组索引下表为0的值递增


下面再来看下`data.currentWindow()`的内部逻辑

### OccupiableBucketLeapArray

`OccupiableBucketLeapArray`继承了抽象类`LeapArray`，核心逻辑也是在`LeapArray`中

```java
/**
 * 时间窗口大小 单位ms
 */
protected int windowLengthInMs;
/**
 * 切分的窗口数
 */
protected int sampleCount;
/**
 * 统计的时间间隔 intervalInMs = windowLengthInMs * sampleCount
 */ 
protected int intervalInMs;

/**
 * 窗口数组 数组大小 = sampleCount
 */
protected final AtomicReferenceArray<WindowWrap<T>> array;

/**
 * update lock 更新窗口时需要上锁
 */
private final ReentrantLock updateLock = new ReentrantLock();

/**
 * @param sampleCount  需要划分的窗口数
 * @param intervalInMs 间隔的统计时间
 */
public LeapArray(int sampleCount, int intervalInMs) {
    this.windowLengthInMs = intervalInMs / sampleCount;
    this.intervalInMs = intervalInMs;
    this.sampleCount = sampleCount;
    this.array = new AtomicReferenceArray<>(sampleCount);
}

/**
 * 获取当前窗口
 */
public WindowWrap<T> currentWindow() {
    return currentWindow(TimeUtil.currentTimeMillis());
}
```
以上需要着重理解的是几个参数的含义：

1. sampleCount 定义的窗口的数
2. intervalInMs 统计的时间间隔
3. windowLengthInMs 每个窗口的时间大小 = intervalInMs / sampleCount

`sampleCount` 比较好理解，就是需要定义几个窗口（默认秒级统计维度的话是两个窗口），`intervalInMs` 指的就是我们需要统计的时间间隔，例如我们统计QPS的话那就是1000ms，`windowLengthInMs` 指的每个窗口的大小，是由`intervalInMs`除以`sampleCount`得来

类似下图
![](http://img.souche.com/f2e/35184370429c595e41100e9e70018761.jpg)￼


理解了上诉几个参数的含义后，我们直接进入到`LeapArray`的`currentWindow(long time)`方法中去看看具体的实现

```java
public WindowWrap<T> currentWindow(long timeMillis) {
    if (timeMillis < 0) {
        return null;
    }
    // 根据当前时间戳计算当前所属的窗口数组索引下标
    int idx = calculateTimeIdx(timeMillis);
    // 计算当前窗口的开始时间戳
    long windowStart = calculateWindowStart(timeMillis);

    /*
     * 从窗口数组中获取当前窗口项，分为三种情况
     *
     * (1) 当前窗口为空还未创建，则初始化一个
     * (2) 当前窗口的开始时间和上面计算出的窗口开始时间一致，表明当前窗口还未过期，直接返回当前窗口
     * (3) 当前窗口的开始时间 小于 上面计算出的窗口开始时间，表明当前窗口已过期，需要替换当前窗口
     */
    while (true) {
        WindowWrap<T> old = array.get(idx);
        if (old == null) {
            /*
             * 第一种情况，新建一个窗口项
             */
            WindowWrap<T> window = new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
            if (array.compareAndSet(idx, null, window)) {
                // Successfully updated, return the created bucket.
                return window;
            } else {
                // Contention failed, the thread will yield its time slice to wait for bucket available.
                Thread.yield();
            }
        } else if (windowStart == old.windowStart()) {
            /*
             * 第二种情况 直接返回
             */
            return old;
        } else if (windowStart > old.windowStart()) {
            /*
             * 第三种情况 替换窗口
             */
            if (updateLock.tryLock()) {
                try {
                    // Successfully get the update lock, now we reset the bucket.
                    return resetWindowTo(old, windowStart);
                } finally {
                    updateLock.unlock();
                }
            } else {
                // Contention failed, the thread will yield its time slice to wait for bucket available.
                Thread.yield();
            }
        } else if (windowStart < old.windowStart()) {
            // 第四种情况，讲道理不会走到这里
            return new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
        }
    }
}
/**
 * 根据当前时间戳计算当前所属的窗口数组索引下标
 */
private int calculateTimeIdx(/*@Valid*/ long timeMillis) {
    long timeId = timeMillis / windowLengthInMs;
    return (int)(timeId % array.length());
}
/**
 * 计算当前窗口的开始时间戳
 */
protected long calculateWindowStart(/*@Valid*/ long timeMillis) {
    return timeMillis - timeMillis % windowLengthInMs;
}
```

上面的方法就是整个滑动窗口逻辑的核心代码，注释其实也写的比较清晰了，简单概括下可以分为以下几步：

1. 根据当前时间戳 和 窗口数组大小 获取到当前的窗口数组索引下标`idx`，如果窗口数是2，那其实`idx`只有两种值(0 或 1)
2. 根据当前时间戳（`windowStart`） 计算得到当前窗口的开始时间戳值。通过`calculateWindowStart`计算来得到，这个方法还蛮有意思的，通过当前时间戳和窗口时间大小取余来得到 与当前窗口开始时间的 偏移量。比我用定时任务实现高级多了 ... 😆 可以去对比一下我之前文章中的蠢实现 [滑动窗口算法定时任务实现](https://github.com/WangJunnan/learn/blob/master/algorithm/src/main/java/com/walm/learn/algorithm/ratelimit/SlidingWindowRateLimit.java)
3. 然后就是根据上面得到的两个值 来获取当前时间窗口，这里其实又分为三种情况
 	* 当前窗口为空还未创建，则初始化一个
 	* 当前窗口的开始时间和上面计算出的窗口开始时间(`windowStart`)一致，表明当前窗口还未过期，直接返回当前窗口
 	* 当前窗口的开始时间 小于 上面计算出的窗口(`windowStart`)开始时间，表明当前窗口已过期，需要替换当前窗口

## 总结

总的来说，`currentWindow`方法的实现还是非常巧妙的，因为我在看`Sentinel`的源码前也写过一篇限流算法的文章，恰好其中也实现过一个滑动窗口限流算法，不过相比于`Sentinel`的实现，我用了定时任务去做窗口的切换更新，显然性能上更差，实现的也不优雅，大家也可以去对比一下。[常用限流算法](http://wangjunnan.club/2019/10/09/%E6%B5%85%E8%B0%88%E5%B8%B8%E7%94%A8%E7%9A%84%E9%99%90%E6%B5%81%E7%AE%97%E6%B3%95/)

Sentinel系列

* [Sentinel源码解析一](http://wangjunnan.club/2019/10/11/Sentinel%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E4%B8%80/)

* [Sentinel源码解析二（slot总览）](http://wangjunnan.club/2019/10/16/Sentinel%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E4%BA%8C%EF%BC%88slot%E6%80%BB%E8%A7%88%EF%BC%89/)

* [Sentinel源码解析二（滑动窗口流量统计）](http://wangjunnan.club/2019/10/25/Sentinel%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E4%B8%89%EF%BC%88%E6%BB%91%E5%8A%A8%E7%AA%97%E5%8F%A3%E6%B5%81%E9%87%8F%E7%BB%9F%E8%AE%A1%EF%BC%89/)
