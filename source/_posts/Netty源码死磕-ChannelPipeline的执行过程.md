---
title: Netty源码死磕(ChannelPipeline的执行过程)
tags:
  - netty
categories:
  - netty
date: 2020-04-14 10:07:58
---

## 引言 
上文有提到如果`Selector`轮询到网络IO事件了，则会调用该`Channel`对应的`ChannelPipeline`来依次执行对应的`ChannelHandler`。 

## `ChannelPipeline`和`ChannelHandler`的关系

那么这里的`ChannelPipeline`和`ChannelHandler`之间到底是什么关系呢？
其实我们可以理解为类似于一个过滤器模式，其中`ChannelPipeline`是一个存放各种过滤器的管道容器，而`ChannelHandler`则对应着单个过滤器实体

## 执行流程

我们来看下`ChannelPipline`的工作流程

1. `NioEventLoop` 触发读事件，会调用`SocketChannel`所关联的`ChannelPipline`
2. 由上一步读取到的消息会在`ChannelPipline`中依次被多个`ChannelHandler`处理
3. 处理完消息会调用`ChannelHandlerContext`的`write`方法发送消息，发送的消息同样也会经过`ChannelPipline`中的多个`ChannelHandler`处理

上面的流程其实主要分成了两步，触发读事件和写事件。对应这两种事件，`Netty`把`ChannelHandler`也分为了两大类 
1. `ChannelInboundHandler` 顾明思义，负责处理链路读事件的`Handler`。通常由IO线程触发
2. `ChannelOutboundHandler`，负责链路写事件的`Handler`
也就是说当IO线程触发读事件时，只会调用`ChannelInboundHandler`的实现`handler`

下面我们看下`ChannelHandler`在`ChannelPipline`中是一个什么样的结构

![](￼http://img.souche.com/f2e/232af441946b83599c5521d190535d08.jpg)

可以看到`ChannelHandler`在加入`ChannelPipline`之前会被封装成一个`ChannelHandlerContext`节点类加入到一个双向链表结构中。除了头尾两个特殊的`ChannelHandlerContext`实现类，我们自定义加入的`ChannelHandler`最终都会被封装成一个`DefaultChannelHandlerContext`类。

![](http://img.souche.com/f2e/00fd4d708cd18e28ee677bbeb920c5de.jpg)


当有读事件被触发时，`ChannelHandler`(会筛选类型为`ChannelInboundHandler`的Handler) 的触发顺序是 `HeaderContext` -> `TailContext`
当有写事件被触发时，`ChannelHandler`(会筛选类型为`ChannelOutboundHandler`的Handler) 的触发顺序与读事件相反是 `TailContext` -> `HeaderContext`

## 总结
本文比较简短，主要是为了理清`ChannelPipeline`和`ChannelHandler`的关系，以及`ChannelHandler`在`ChannelPipeline`中的执行过程是怎样的
