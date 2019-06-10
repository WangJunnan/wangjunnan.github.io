---
title: Java NIO Selector 多路复用选择器
tags:
  - NIO
categories:
  - java基础
  - NIO
date: 2019-01-20 16:36:41
---

## 引言

上篇文章我们简单的使用了`NIO`的`Channel`通道，本期我们主要来介绍一下选择器（Selector
）的使用，`Selector`是`Java NIO`核心组件中的一个。在之前介绍5种I/O模型的时候，有介绍过`多路复用模型`。多路复用模型使得我们可以使用一个线程来管理成千上万的连接，避免了线程上下文切换带来的开销，使得性能得到极大的提升

## 基本模型

选择器`Selector`使用的基本模式，跟传统BIO处理模型不一样。传统`BIO`往往我们会使用多线程来提升处理性能，也就是说每接入一个Client，Server端就会为其新开一个线程，以此来提升并发吞吐量，使用这种模式的弊端很明显，因为线程是系统非常重要的资源，当并发量少的时候，感觉不到，一旦并发量上来，就会出现瓶颈。

再来看`Selector`是如何处理的，首先每接入一个Client，我们可以通过`Selector`选择器注册感兴趣的事件，然后通过一个线程去不停的轮询检测各个Client是否有感兴趣的事件发生，有则顺序处理该Client就绪的各个事件。

![image.png](http://upload-images.jianshu.io/upload_images/2717496-86f2efcd8869bdcd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Selector 使用实例

我们先来看一个`Selector`非常常见的使用例子

```java
try {
	  // 打开一个通道
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.socket().bind(new InetSocketAddress(8890));
    // 打开 选择器 selector
    Selector selector = Selector.open();
    // 设置非阻塞
    serverSocketChannel.configureBlocking(false);
    // 为ServerSocketChannel注册 OP_ACCEPT 事件，返回一个SelectionKey 对象
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

    while (true) {
    		// 返回至少有一个事件就绪的通道数，该方法是阻塞的
        int readyNum = selector.select();
        if (readyNum == 0) {
            continue;
        }
			
			// 返回就绪的 SelectionKey集合
        Set<SelectionKey> selectionKeys = selector.selectedKeys();
        Iterator<SelectionKey> iteratorKeys = selectionKeys.iterator();
        
        // 遍历所有就绪的SelectionKey集合
        while (iteratorKeys.hasNext()) {
            SelectionKey key = iteratorKeys.next();
            iteratorKeys.remove();
				 // 判断就绪的具体事件
            if (key.isValid()) {
                if (key.isAcceptable()) {
                    ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
                    // 接受一个新连接
                    SocketChannel channel = serverChannel.accept();
                    channel.configureBlocking(false);
                    // 为该Channel注册可读事件
                    channel.register(selector, SelectionKey.OP_READ);
                } else if (key.isReadable()) {
                    ...
                }
            }
        }
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

通过查看上面的代码，可知我们通过`Selector.open()`创建了一个选择器，并且通过`SocketChannel`的`register`方法注册了选择器和事件，其中`register`方法会返回一个`SelectionKey`对象，该对象其实就是维护了`Channel`和`Selector`的对应关系

### SelectionKey

上文我们提到`SelectionKey`对象维护了`Channel`和`Selector`的对应关系，现在我们来看下
`SelectionKey`对象内部几个非常重要的属性和方法

* 属性
	* OP_READ OP_WRITE OP_CONNECT OP_ACCEPT 可注册的四个感兴趣的项目
	* interestOps 所有感兴趣的集合
	* readyOps 所有就绪的兴趣集合

* 方法
	* boolean isValid() 判断该`SelectionKey`是否有效
	* void cancel() 取消该`SelectionKey`中 通道与其选择器 的注册
	* int interestOps() 返回该`SelectionKey`所包含的所有兴趣（可读可写）集合
	* SelectionKey interestOps(int ops) 设置一个感兴趣的项目
	* int readyOps() 返回已经就绪的兴趣集合
	* boolean isWritable() 判断是否可写
	* boolean isReadable() 判断是否可读
	* boolean isConnectable() 判断是否可连接
	* boolean isAcceptable() 判断是否可接受
	* Object attach(Object ob) 在该`SelectionKey`中附加一个对象信息
	* Object attachment() 获取附加对象信息



## Selector

讲完`SelectionKey`和`Selector`的关系之后，我们再次回到`Selector`类，我们首先需要知道`Selector`中3个重要的`SelectionKey`集合

* `keys`：所有注册到Selector的Channel所表示的SelectionKey都会存在于该集合中。keys元素的添加会在Channel注册到Selector时发生。
* `selectedKeys`：该集合中的每个SelectionKey都是其对应的Channel在上一次操作selection期间被检查到至少有一种SelectionKey中所感兴趣的操作已经准备好被处理。该集合是keys的一个子集。
* `cancelledKeys`：执行了取消操作的SelectionKey会被放入到该集合中。该集合是keys的一个子集。

接下来将会介绍上文例子中所用到的几个方法

### Selector.open()

这个静态方法可以打开一个`Selector`选择器

```java
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}
```
通过查看源码可知在`Linux`系统下默认会使用`EpollSelectorImpl`来作为`Selector`实现类，注意各个系统下默认的实现类是不同的。

### selector.select()

`Selector`的`select()`方法会返回事件就绪的`SelectionKey`数目（也就是就绪的Channel数）

并且该方法会一直阻塞直到至少一个`channel`被选择(即，该`channel`注册的事件发生了)为止，除非当前线程发生中断或者`selector`的`wakeup`方法被调用

该方法还有一个重载`select(long timeout)`，可以自定义超时时间

### selector.selectNow()

该方法与上面方法类似，但该方法不会发生阻塞，即使没有一个`channel`被选择也会立即返回

### selector.selectedKeys()

返回已就绪的`SelectionKey`集合，该方法在执行了`selector.select()`后调用，因为在执行`selector.select()`后就表示至少有一个`SelectionKey`已经就绪


## 尾言

好了，本篇文章就介绍到这里了，篇幅有限加上本人对`NIO`的理解也有待加深，希望可以在之后更深入的对`Java NIO`的实现进行分析。



