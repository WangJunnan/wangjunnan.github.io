---
title: Java AQS 源码解析
tags:
  - 并发
categories:
  - java基础
  - 并发编程
date: 2019-03-12 16:21:26
---

## 引言

`AQS`全称`java.util.concurrent.locks.AbstractQueuedSynchronizer`，是Java 并发包中的一个抽象类，我们一般把它叫做抽象队列同步器，如我们常用的可重入锁`ReentrantLock`的内部实现就是基于`AQS`，理解`AQS`的内部源码实现对于我们深入理解使用java并发包中的各个功能非常重要。

## 实现原理

`AQS`内部维护了一个双向链表队列来管理多个线程。简单介绍一下就是，当有一个新的线程去尝试获取锁，这是如果获取失败，`AQS`则会将此线程封装成一个`Node`节点，并将此节点加入到内部维护的队列中。

上面简单的描述了线程获取锁的过程，那么`AQS`是使用什么来确认当前线程是否可以尝试获取锁呢？

其实`AQS`内部为了一个`volatile int state`变量来表示当前资源是否已经被其他线程占有，对于独占锁来说，只有当`state = 0`时才说明该资源当前没有线程占有，后续线程可以开始争抢改资源，后续线程需要通过`CAS`操作来确保修改`state`变量成功才能成功抢占该资源。

![image.png](http://upload-images.jianshu.io/upload_images/2717496-b08b642e54d9c66a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 重要方法

`AQS`内部提供了三组核心方法用来实现一个同步组件。三组方法从字面意思上区分可以分为两种类型：独占锁相关，共享锁相关

1. 访问`state`有三种方法

*  `getState()`  获取同步状态

*  `setState()`  设置同步状态（非`CAS`操作）

*  `compareAndSetState()`  通过`CAS`操作修改

2.  一组抽象方法，需要子类实现，主要是获取同步锁并判断是否成功的操作

方法 | 描述
--- | ---
boolean tryAcquire(int arg) | 独占式获取同步状态
boolean tryRelease(int arg) | 独占式释放同步状态
int tryAcquireShared(int arg) | 共享式获取同步状态
boolean tryReleaseShared(int arg) | 共享式释放同步状态
boolean isHeldExclusively() | 检测当前线程是否获取独占锁

3.  模板方法，用于子类直接调用

方法 | 描述
---  |  ---
void acquire(int arg) | 获取独占锁。会调用`tryAcquire`方法，如果未获取成功，则会进入同步队列等待
void acquireInterruptibly(int arg) | 响应中断版的 `acquire`
boolean tryAcquireNanos(int arg,long nanos) | 超时+响应中断版的 acquire
void acquireShared(int arg) | 获取共享锁。共享式获取同步状态，同一时刻可能会有多个线程获得同步状态。比如**读写锁的读锁**就是就是调用这个方法获取同步状态的
void acquireSharedInterruptibly(int arg) | 响应中断版的`acquireShared`
boolean tryAcquireSharedNanos(int arg,long nanos) | 超时+响应中断版的`acquireShared`
boolean release(int arg) | 独占式释放同步状态
boolean releaseShared(int arg) | 释放共享锁
Collection getQueuedThreads() | 获取同步队列上的线程集合

## 源码分析

上文有提到，进入同步队列的线程会被封装成一个`Node`节点，下面我们先来看下`Node.class`

```java

static  final  class Node {

 /**

 * 用于标记一个节点在共享模式下等待

 */

 static final Node SHARED = new Node();

 /**

 * 用于标记一个节点在独占模式下等待

 */

 static final Node EXCLUSIVE = null;

 /**

 * 等待状态：取消状态

 */

 static final int CANCELLED = 1;

 /**

 * 等待状态：通知(指示后继线程需要阻塞)

 */

 static final int SIGNAL = -1;

 /**

 * 等待状态：条件等待

 */

 static final int CONDITION = -2;

 /**

 * 等待状态：传播

 */

 static final int PROPAGATE = -3;

 /**

 * 等待状态

 */

 volatile int waitStatus;

 /**

 * 前驱节点

 */

 volatile Node prev;

 /**

 * 后继节点

 */

 volatile Node next;

 /**

 * 节点对应的线程

 */

 volatile Thread thread;

 /**

 * 等待队列中的后继节点

 */

 Node nextWaiter;

 /**

 * 当前节点是否处于共享模式等待

 */

 final boolean isShared() {

 return nextWaiter == SHARED;

 }

 /**

 * 获取前驱节点，如果为空的话抛出空指针异常

 */

 final Node predecessor() throws NullPointerException {

 Node p = prev;

 if (p == null) {

 throw new NullPointerException();

 } else {

 return p;

 }

 }

 Node() {

 }

 /**

 * addWaiter会调用此构造函数

 */

 Node(Thread thread, Node mode) {

 this.nextWaiter = mode;

 this.thread = thread;

 }

 /**

 * Condition会用到此构造函数

 */

 Node(Thread thread, int waitStatus) {

 this.waitStatus = waitStatus;

 this.thread = thread;

 }

}

```

这里解释几个重要的属性

属性 | 意义
---  |  ---
prev | 前置节点
next | 后置节点
CANCELLED | 状态值 表示节点被取消
SIGNAL | 状态值  表示后置节点需要被阻塞
CONDITION | 状态值 表示节点进入 CONDITION
PROPAGATE | 状态值 共享锁传播状态

## 独占锁的获取

在`AQS`中包含了`head`和`tail`两个Node引用，其中`head`在逻辑上的含义是当前持有锁的线程，`tail`指示队尾节点。`head`和`tail`可以相等

```java

/**

* 获取独占锁，该方法会首先调用tryAcquire（由子类实现）来获取锁，

* 如果获取成功会直接返回，失败则会将当前抢锁失败的线程封装为Node节点加入队尾

 */

public final void acquire(int arg) {

 if (!tryAcquire(arg) &&

 acquireQueued(addWaiter(Node.EXCLUSIVE), arg))

 selfInterrupt();

}

/**

* 增加一个

 *

 */

private Node addWaiter(Node mode) {

 Node node = new Node(Thread.currentThread(), mode);

  // Try the fast path of enq; backup to full enq on failure

  // 快速执行一次加入队列尾部的操作，如果失败的话则执行 enq

  // 获取队尾节点

 Node pred = tail;

 if (pred != null) {

  // 获取队尾的前驱节点

 node.prev = pred;

  // CAS设置新的队尾，有可能失败

 if (compareAndSetTail(pred, node)) {

 pred.next = node;

 return node;

 }

 }

  // 如果还没有队尾（初始状态），或则入队失败，则执行下面的方法

 enq(node);

 return node;

}

private Node enq(final Node node) {

  // 通过CAS 自旋设置新的队尾

 for (;;) {

 Node t = tail;

  // 队尾节点为空，说明是初始状态，必须初始化头尾节点

 if (t == null) { // Must initialize

 if (compareAndSetHead(new Node()))

 tail = head;

 } else {

 // 这里注意一个细节

  // node.prev = t 操作是设置当前节点的前驱节点为现在的队尾节点

  // 但是该操作没有放到compareAndSetTail操作成功以后，也就是说代码为何不这样写

  // if (compareAndSetTail(t, node)) {

  // node.prev = t;

  // t.next = node;

  // return t;

 // }

// 如果这样子写的话，队尾节点在某一瞬间会是一个孤立的节点，但是如果先

// 执行 node.prev = t 就把当前节点的前驱指向了还未设置前的队尾，这样子当我们从后往前遍历的时候，就不会出现队列断裂的情况

 node.prev = t;

 if (compareAndSetTail(t, node)) {

 t.next = node;

 return t;

 }

 }

 }

}

/**

* 节点加入队列之后操作，这里分两种情况

* 1\. 如果当前节点的前驱节点为head 头节点，则尝试获取锁，并设置新的头节点

 * 2\. 阻塞自己

 */

final boolean acquireQueued(final Node node, int arg) {

 boolean failed = true;

 try {

 boolean interrupted = false;

  // 循环尝试获取锁

 for (;;) {

 // 获取前驱节点

 final Node p = node.predecessor();

  // 前驱节点为head节点 则尝试获取锁

 if (p == head && tryAcquire(arg)) {

 setHead(node);

 p.next = null; // help GC

 failed = false;

 return interrupted;

 }

  // 判断是否需要阻塞自己

 if (shouldParkAfterFailedAcquire(p, node) &&

 parkAndCheckInterrupt())

 interrupted = true;

 }

 } finally {

  // 如果抛异常失败，则调用 cancelAcquire 取消该节点

 if (failed)

 cancelAcquire(node);

 }

}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {

 int ws = pred.waitStatus;

  // 如果前驱节点的 waitStatus 为 -1，则说明当前线程需要阻塞

 if (ws == Node.SIGNAL)

 /*

 * This node has already set status asking a release

 * to signal it, so it can safely park.

 */

 return true;

  // waitStatus大于0，说明前驱节点为取消状态，需要重新设置前驱节点

 if (ws > 0) {

  // 向前寻找waitStatus<=0的节点，并设置为新的前驱节点

 do {

 node.prev = pred = pred.prev;

 } while (pred.waitStatus > 0);

 pred.next = node;

 } else {

 /*

 * 如果前驱节点状态为 0 -2 -3，则设置前驱节点 WaitStatus = -1，并重新循环

 */

 compareAndSetWaitStatus(pred, ws, Node.SIGNAL);

 }

  return  false;

}

/**

 * 调用 LockSupport.park(this) 阻塞当前线程

 */

private final boolean parkAndCheckInterrupt() {

 LockSupport.park(this);

 return Thread.interrupted();

}

/**

* 取消节点，也就是当前线程放弃获取锁

 */

private void cancelAcquire(Node node) {

 if (node == null)

 return;

 node.thread = null;

  // 获取前驱节点，会跳过状态为取消状态的节点 Skip cancelled predecessors

 Node pred = node.prev;

 while (pred.waitStatus > 0)

 node.prev = pred = pred.prev;

  // 获取后继节点

 Node predNext = pred.next;

  // 设置节点状态为取消

 node.waitStatus = Node.CANCELLED;

  // 如果当前节点是队尾节点，则设置前驱节点为队尾，并设置前驱节点的后继节点为null

 if (node == tail && compareAndSetTail(node, pred)) {

 compareAndSetNext(pred, predNext, null);

 } else {

 int ws;

  // 如果前驱节点不是头结点，且前驱节点的 WaitStatus = -1

 if (pred != head &&

 ((ws = pred.waitStatus) == Node.SIGNAL ||

 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&

 pred.thread != null) {

 Node next = node.next;

  // 将前置节点和后继节点相连

 if (next != null && next.waitStatus <= 0)

 compareAndSetNext(pred, predNext, next);

 } else {

 // 唤醒后继节点

  // 这里为何要唤醒后继节点？

  // 因为如果当前节点的前置节点为head节点，但是此时head节点的状态为0（当前节点还未来得及设置前置节点状态为 -1），这时head节点释放了锁因为状态还为0，所以不会去唤醒后继线程。这时队列就失去活性了！所以这里需要唤醒后继线程

 unparkSuccessor(node);

 }

 node.next = node; // help GC

 }

}

/**

* 唤醒后继节点

 */

private void unparkSuccessor(Node node) {

 /*

 * 如果节点 waitStatus < 0，则尝试将状态设为0

 */

 int ws = node.waitStatus;

 if (ws < 0)

 compareAndSetWaitStatus(node, ws, 0);

 /*

 * 获取当前节点的后继节点

 * 如果 s!=null 且 s未被取消 则直接唤醒

 */

 Node s = node.next;

 if (s == null || s.waitStatus > 0) {

 s = null;

  // 从尾节点向前查找离node最近的非取消节点

  // 注意这里为何要从后向前查找

  // 在 enq方法中有段注释，如果从前往后的话，有可能出现队列断裂

 for (Node t = tail; t != null && t != node; t = t.prev)

 if (t.waitStatus <= 0)

 s = t;

 }

 if (s != null)

 LockSupport.unpark(s.thread);

}

```

上文基本完成了对独占锁获取锁的流程的分析，这里简单画个图来更清晰的展示整个获取锁的过程：

![image.png](http://upload-images.jianshu.io/upload_images/2717496-376c9c2663fd07fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 独占锁的释放

```java

/**

 * 释放锁

 */

public  final  boolean release(int arg) {

  // 如果释放锁成功

 if (tryRelease(arg)) {

 Node h = head;

  // 且当前持锁的头结点状态不等于0，就去唤醒后继线程

  // 思考这里等于0 为何不行？

  // 因为当 waitStatus == 0时，说明后继线程还在运行中，还未来得及将waitStatus 状态设为-1

 if (h != null && h.waitStatus != 0)

  // 与上文unparkSuccessor方法一致

 unparkSuccessor(h);

 return true;

 }

  return  false;

}

```

## 共享锁

共享锁与独占锁最大的区别在于，共享锁允许多个线程持有锁，而独占锁在同一时间只有一个线程可以获取到锁

### 共享锁的获取

```java

/**

* 获取共享锁

 */

public final void acquireShared(int arg) {

  // 如果tryAcquireShared(arg) < 0表示获取共享锁失败

  // 获取失败的原因一般是由于该资源正由独占锁占有

 if (tryAcquireShared(arg) < 0)

 doAcquireShared(arg);

}

private void doAcquireShared(int arg) {

  // 加入到队列尾部

 final Node node = addWaiter(Node.SHARED);

 boolean failed = true;

 try {

 boolean interrupted = false;

 for (;;) {

 final Node p = node.predecessor();

 if (p == head) {

 int r = tryAcquireShared(arg);

  // 一旦共享获取成功，设置新的头结点，并且唤醒后继线程

 if (r >= 0) {

 setHeadAndPropagate(node, r);

 p.next = null; // help GC

 if (interrupted)

 selfInterrupt();

 failed = false;

 return;

 }

 }

 // 判断是否需要阻塞

 if (shouldParkAfterFailedAcquire(p, node) &&

 parkAndCheckInterrupt())

 interrupted = true;

 }

 } finally {

 if (failed)

 cancelAcquire(node);

 }

}

/**

* 在获取共享锁成功后，设置head节点

* 跟据调用tryAcquireShared返回的状态以及节点本身的等待状态来判断是否要需要唤醒后继线程。

 */

private void setHeadAndPropagate(Node node, int propagate) {

 Node h = head;

 setHead(node);

 /*

 * propagate是tryAcquireShared的返回值，这是决定是否传播唤醒的依据之一。

 * h.waitStatus为SIGNAL或者PROPAGATE时也根据node的下一个节点共享来决定是否传播唤醒，

 * 思考，这里为什么不能只用propagate > 0来决定是否可以传播

 * 因为此时有可能有其他线程释放了锁，但propagate还未更新，所以还需要判断h.waitStatus < 0

 */

 if (propagate > 0 || h == null || h.waitStatus < 0 ||

 (h = head) == null || h.waitStatus < 0) {

 Node s = node.next;

  // 后继节点为共享锁类型

 if (s == null || s.isShared())

 doReleaseShared();

 }

}

/**

* 这是共享锁中的核心唤醒函数，主要做的事情就是唤醒下一个线程或者设置传播状态。

 */

private void doReleaseShared() {

 for (;;) {

 Node h = head;

  // 如果队列中存在后继线程。

 if (h != null && h != tail) {

 int ws = h.waitStatus;

 if (ws == Node.SIGNAL) {

 if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))

 continue;

 unparkSuccessor(h);

 }

  // 如果h节点的状态为0，需要设置为PROPAGATE用以保证唤醒的传播。

 else if (ws == 0 &&

 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))

 continue;

 }

 if (h == head)

 break;

 }

}

```

### 共享锁的释放

```

public final boolean releaseShared(int arg) {

 if (tryReleaseShared(arg)) {

 // 与上文doReleaseShared一样

 doReleaseShared();

 return true;

 }

 return false;

}

```

## 总结

`AQS`的整体理念相对比较容易理解，但是`AQS`真正难懂的其实是它的细节，因为一个很简单的问题一旦被放在并发环境中就会变的非常抽象，难以理解。要理解`AQS`的细节实现，还是需要多看，我是前前后后看了有一周，对于有些细节还是略模糊。

本篇文章参考了

[[AbstractQueuedSynchronizer源码解读](https://www.cnblogs.com/micrari/p/6937995.html)](https://www.cnblogs.com/micrari/p/6937995.html)

