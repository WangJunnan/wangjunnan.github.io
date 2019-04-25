---
title: Java CAS
tags:
  - 并发
categories:
  - java基础
  - 并发编程
date: 2019-03-02 16:15:23
---

## 引言

在介绍`CAS`之前，我们有必要先理解线程安全的三大特性

* 原子性: 对于涉及共享变量访问的操作，该操作从其执行线程以外的任意线程来看是不可分割的，从而可以让各个线程依次串行访问，但是原子性并不保证可见性
* 可见性: 修改共享变量时，立即将工作内存中的值同步到主存中，并使该修改对其他线程可见
* 有序性: 禁止读取共享变量后的代码、修改共享变量前的代码重排序

`CAS`即`compare and swap`的缩写，中文翻译成比较并交换。是一种用于在多线程环境下实现同步功能的机制。调用`Java CAS`需要三个操作数
1. 内存中值的内存位置
2. 预期值
3. 新值

具体实现是通过值的内存位置取到内存中的值并与预期值比较，若相等，则将内存位置处的值替换为新值，若不相等，则不做任何操作返回false。如果大家有了解过悲观锁和乐观锁，可以发现`CAS`其实是一种乐观锁的实现。

## 使用CAS的目的

对于实现线程安全，我们用的比较多的应该是`synchronized`关键字，`synchronized`其实是一种悲观锁，锁被占用的情况会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。`CAS`是一种乐观锁，每次取数据都不会加锁，更新的时候会进行数据比对，有冲突的话则会自旋重试。可以看到在读操作频繁，更新频率低，冲突概率低的情况下，用`CAS`的话会更加合理。当然JDK1.6之后，Java对`synchronized`关键字来了一大波优化（自旋锁，锁消除，锁粗化，偏向锁，轻量级锁），一般情况下使用`synchronized`是非常稳定的。


## CAS的底层实现

现在的CPU都是多核心的，多个核心通过总线来操作内存。那么这里就存在一个问题，就是如果多个核心同时操作一块内存区域，会发生什么问题呢？是的，这里数据就会出现混乱。不过这里我们可以从`intel`的使用手册中找到答案，对指令加`lock`前缀可以保证操作的原子性，可见性以及有序性。好了，底层的就不多说了，我们直接去看一下`java.util.concurrent.atomic`包下的原子类 `AtomicInteger`的源码实现

## AtomicInteger源码

```
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // value值的内存位置
    private static final long valueOffset;

    static {
        try {
            // 获取value的内存位置
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
	  
	  // value值 volatile 修饰 保证可见性
    private volatile int value;

    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    public AtomicInteger() {
    }

    /**
     * 获取value在内存中当前的值
     *
     * @return the current value
     */
    public final int get() {
        return value;
    }
    
    /**
     * 比较并替换 实现在unsafe.compareAndSwapInt中
     * @param expect 期望值
     * @param update 新值
     */
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
    
    /**
     * 自增 实现在unsafe.getAndAddInt中
     */
    public final int incrementAndGet() {
     		return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
	 }

}
```

* Unsafe.class

```
public final native boolean compareAndSwapInt(Object o, long valueOffset, int expected, int value);

/**
 * 获取当前值 并加1，返回的是加1前的值
 */
public final int getAndAddInt(Object o, long valueOffset, int addValue) {
    int currentValue;
    do {
        currentValue = this.getIntVolatile(o, valueOffset);
        // 比较当前内存的值和预期值currentValue是否一致，一致的话则设置新值。但是因为当前内存中的值有可能被其他线程修改，会有和预期值不一致的情况，所以这里会循环直到 compareAndSwapInt 返回成功为止，这里的操作也称为CAS自旋
    } while(!this.compareAndSwapInt(o, valueOffset, currentValue, value + addValue));
    return value;
}
```
`AtomicInteger`是`Java`对`Integer`类型原子性操作的实现，可以看到底层都是调用了CAS `compareAndSwapInt`native方法。
这里主要看一下`compareAndSwapInt(Object o, long valueOffset, int expect, int update)`的四个参数

* o 当前操作的对象
* valueOffset 操作值所在的内存位置 
* expect 期望值
* update 新值

具体实现是将内存位置处的数值与预期数值相比较，若相等，则将内存位置处的值替换为新值。若不相等，则不做任何操作。
`CAS自旋`指的是替换新值失败时会进入循环，重新获取期望值，直到期望值和内存位置处的数值相等。

## CAS的问题

### ABA问题

提到`CAS`存在的问题，就不得不提`ABA`问题，什么是`ABA`问题呢？
举个例子，A是个共享变量，原值是`10`，线程1从内存中拿到了A，此时值为`10`，当线程1要对变量A进行`CAS`操作前，因为其他线程的操作，A从`10`变为了`11`，又从`11`变回了`10`。此时线程1对变量A执行`CAS`操作照道理应该是要失败的，但实际却是成功的。这是因为经过了上面的流程，在线程1看来，变量A没有发生任何变化，所以它执行`CAS`操作是会成功的。

要解决`ABA`问题，通常的解决方案给对象加上版本号，每经过一次`CAS`操作就更新一次版本号


## 总结

本文的目的主要是让自己对java并发包的基础`CAS`有个简单的了解，以便进行后续的源码分析



