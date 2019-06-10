---
title: ReentrantLock可重入锁源码分析
tags:
  - 并发
categories:
  - java基础
  - 并发编程
date: 2019-03-25 16:19:02
---

## 引言
`ReentrantLock`可重入锁，也是日常使用除了`synchronized`关键字最多的锁。从字面上理解可重入的意思就是**同一个线程**可以重复加锁，当然`synchronized`关键字也是支持可重入的，其实它们的主要功能也是类似的，但是相比较来说，`ReentrantLock`功能更灵活丰富，例如`ReentrantLock`支持设置公平锁或非公平锁，支持分组唤醒需要唤醒的线程们，而不是像`synchronized`要么随机唤醒一个线程要么唤醒全部线程。

整个`ReentrantLock`的实现基于`AQS`独占锁，这个在上篇文章已经详细分析过


## 类结构

* ReentrantLock.class

![image.png](http://upload-images.jianshu.io/upload_images/2717496-de0aa2e9894c7a7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](http://upload-images.jianshu.io/upload_images/2717496-94b5fd02f6a75420.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`ReentrantLock`实现了`Lock`接口，但是通过查看源码可知`ReentrantLock`的具体实现其实都委托给了`NonfairSyns（非公平锁）`或则`FairSync（公平锁）`两个类实现，`NonfairSyns`和`FairSync`继承了抽象静态类`Sync`，`Sync`则是`AQS`的子类


## 公平锁 非公平锁

`ReentrantLock`支持公平锁，那么什么是公平锁呢？简单的理解就是多个线程获取锁的顺序应该是按照时间的先后，先来的先获取。也就是说公平锁绝对满足先进先出的队列形式。

我们从`AQS`同步队列来理解的话，可以认为`公平锁`每次都是从同步队列中的第一个节点获取到锁，而非公平锁则不一定，有可能占据锁的线程刚释放锁，同步队列中的线程还未来得及获取到锁，就被新请求锁的线程获取了。当然这里要注意，对于已经加入`AQS`同步队列的线程来说，无论`公平锁`还是`非公平锁`都是一样的，按照队列先后顺序获取。

下面我们来看下`ReentrantLock`的构造方法

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```
* `ReentrantLock`无参构造默认是非公平锁
* 可以通过设置参数将`ReentrantLock`设置为公平锁

从代码层面看实现区别

```java

/**
 * 非公平锁获取锁
 */
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}

/**
 * 公平锁获取锁
 */
final void lock() {
    acquire(1);
}
```

可以看到很明显的区别，`非公平锁`获取锁，只要当前没有持锁线程，新加入的线程都有机会获取到锁

## 可重入

上文说过，可重入的概念只针对同一个线程，那么`ReentrantLock`是如何实现同一个线程可重入的呢？

这就又要涉及到`AQS state`的概念，在`AQS`内部维护了一个状态值`state`，当有一个线程获取到锁，`state`值就会`+1`，对于独占锁来说，不同的线程要想获取锁，`state`值必须为0。下面我们就通过源码来看看`ReentrantLock`是如何实现可重入性的

以非公平锁为例

```java
/**
 * 尝试获取锁
 */
final void lock() {
    // 如果当前获取锁的线程为0，则直接获取
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
    		// 调用AQS模板方法，会调用子类的tryAcquire
        acquire(1);
}

protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

/**
 * 非公平方法尝试获取锁
 */
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
    		// 如果当前获取锁的线程为0，则直接获取锁
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果当前获取锁的线程为当前请求线程，则将state + 1
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
* 首先会判断`state`状态是否为0(也就是当时还没有持锁线程)，如果为0的话则直接获取锁
* `state`状态不为0.则判断与当前持锁线程是否为同一个线程，是的话，则`state`自增1.获取锁成功


## 总结

总的来说，`ReentrantLock`和`synchronized`非常相似，但是也还是比较大的区别，`ReentrantLock`的功能更丰富灵活，`synchronized`只支持非公平锁，`ReentrantLock`可以支持`公平锁`，并且`ReentrantLock`支持`Condition`等，可以批量分组唤醒等待线程。

`
