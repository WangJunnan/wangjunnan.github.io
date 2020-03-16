---
title: Netty源码死磕二（Netty的启动流程）
tags:
  - netty
categories:
  - netty
date: 2020-03-16 10:45:47
---
## 引言
上一篇文章介绍了`Netty`的线程模型及`EventLoop`机制，相信大家对`Netty`已经有一个基本的认识。那么本篇文章我会根据Netty提供的`Demo`来分析一下`Netty`启动流程。

## 启动流程概览

开始之前，我们先来分析下`Netty`服务端的启动流程，下面是一个简单的流程图
![](http://img.souche.com/f2e/4282b07026b5590f3059617169921f58.jpg)


启动流程大致分为五步

1. 创建`ServerBootstrap`实例，`ServerBootstrap`是Netty服务端的启动辅助类，其存在意义在于其整合了Netty可以提供的所有能力，并且尽可能的进行了封装，以方便我们使用
2. 设置并绑定`EventLoopGroup`，`EventLoopGroup`其实是一个包含了多个`EventLoop`的NIO线程池，在上一篇文章我们也有比较详细的介绍过`EventLoop`事件循环机制，不过值得一提的是，Netty 中的`EventLoop`不仅仅只处理IO读写事件，还会处理用户自定义或系统的Task任务
3. 创建服务端Channel `NioServerSocketChannel`，并绑定至一个`EventLoop`上。在初始化`NioServerSocketChannel`的同时，会创建`ChannelPipeline`，`ChannelPipeline`其实是一个绑定了多个`ChannelHandler`的执行链，后面我们会详细介绍
4. 为服务端`Channel`添加并绑定`ChannelHandler`,`ChannelHandler`是Netty开放给我们的一个非常重要的接口，在触发网络读写事件后，Netty都会调用对应的`ChannelHandler`来处理，后面我们会详细介绍
5. 为服务端`Channel`绑定监听端口，完成绑定之后，Reactor线程(也就是第三步绑定的EventLoop线程)就开始执行`Selector`轮询网络IO事件了，如果`Selector`轮询到网络IO事件了，则会调用`Channel`对应的`ChannelPipeline`来依次执行对应的`ChannelHandler`

## 启动流程源码分析

下面我们就从启动源码来进一步分析 Netty 服务端的启动流程

### 入口

首先来看下常见的启动代码
```java
// 配置bossEventLoopGroup 配置大小为1
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
// 配置workEventLoopGroup 配置大小默认为cpu数*2
EventLoopGroup workerGroup = new NioEventLoopGroup();
// 自定义handler
final EchoServerHandler serverHandler = new EchoServerHandler();
try {
    // 启动辅助类 配置各种参数（服务端Channel类，EventLoopGroup，childHandler等）
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
     // 配置 channel通道，会反射实例化
     .channel(NioServerSocketChannel.class)
     .option(ChannelOption.SO_BACKLOG, 100)
     .handler(new LoggingHandler(LogLevel.INFO))
     .childHandler(new ChannelInitializer<SocketChannel>() {
         @Override
         public void initChannel(SocketChannel ch) throws Exception {
             ChannelPipeline p = ch.pipeline();
             //p.addLast(new LoggingHandler(LogLevel.INFO));
             p.addLast(serverHandler);
         }
     });

    // 绑定监听端口 启动服务器
    ChannelFuture f = b.bind(PORT).sync();

    // 等待服务器关闭
    f.channel().closeFuture().sync();
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```
1. 可以看到上面的代码首先创建了两个`EventLoopGroup`，在上一篇文章我们有介绍过`Netty`的线程模型有三种，而不同的`EventLoopGroup`配置对应了三种不同的线程模型。这里创建的两个`EventLoopGroup`则是用了`多线程Reactor模型`，其中`bossEventLoopGroup`对应的就是处理`Accept`事件的线程组，而`workEventLoopGroup`则负责处理IO读写事件。
2. 然后就是创建了一个启动辅助类`ServerBootstrap`，并且配置了如下几个重要参数
	* group 两个Reactor线程组（bossEventLoopGroup， workEventLoopGroup）
	* channel 服务端Channel
	* option 服务端socket参数配置 例如`SO_BACKLOG`指定内核未连接的Socket连接排队个数
	* handler 服务端Channel对应的Handler
	* childHandler 客户端请求Channel对应的Handler
3. 绑定服务端监听端口，启动服务 -> `ChannelFuture f = b.bind(PORT).sync();`

这篇文章主要是分析Netty的启动流程。so我们直接看`b.bind(PORT).sync()`的源码

`bind` 发现该方法内部实际调用的是`doBind(final SocketAddress localAddress)`方法

### doBind

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
// 初始化服务端Channel
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
    // 初始化一个 promise（异步回调）
        ChannelPromise promise = channel.newPromise();
        // 绑定监听端口
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } 
    .... // 省略其他代码
}
```
`doBind`主要做了两个事情
1. `initAndRegister()` 初始化Channel
2. `doBind0` 绑定监听端口

#### `initAndRegister()`

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
    // new一个新的服务端Channel
        channel = channelFactory.newChannel();
        // 初始化Channel
        init(channel);
    } catch (Throwable t) {
        ...
    }
    // 将Channel注册到EventLoopGroup中一个EventLoop上
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}
```

1. `channelFactory.newChannel()`其实就是通过反射创建配置的服务端Channel类，在这里是`NioServerSocketChannel`

2. 创建完成的`NioServerSocketChannel`进行一些初始化操作，例如将我们配置的`Handler`加到服务端`Channel`的`pipeline`中
3. 将`Channel`注册到`EventLoopGroup`中一个EventLoop上

下面我们来看下`NioServerSocketChannel`类的构造方法，看看它到底初始化了哪些东西，先看下其继承结构

##### NioServerSocketChannel初始化
![](http://img.souche.com/f2e/25faa82af6e7f37636915bad607caf21.jpg)


下面是它的构造方法的调用顺序，依次分为了四步
```java
// 1
public NioServerSocketChannel() {
// 通过 SelectProvider来初始化一个Java NioServerChannel
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}

// 2.
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    // 创建一个配置类，持有Java Channel
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}

// 3.
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
    // 设置Channel为非阻塞
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            ch.close();
        } catch (IOException e2) {
            logger.warn(
                        "Failed to close a partially initialized socket.", e2);
        }

        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}

// 4
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    // 生成一个channel Id
    id = newId();
    // 创建一个 unSafe 类，unsafe封装了Netty底层的IO读写操作
    unsafe = newUnsafe();
    // 创建一个 pipeline类
    pipeline = newChannelPipeline();
}
```
可以看到`NioServerSocketChannel`的构造函数主要是初始化并绑定了以下3类

1. 绑定一个Java `ServerSocketChannel`类
2. 绑定一个`unsafe`类，unsafe封装了Netty底层的IO读写操作
3. 绑定一个`pipeline`，每个Channel都会唯一绑定一个`pipeline`


##### init(Channel channel)
```java
void init(Channel channel) {
// 设置Socket参数
    setChannelOptions(channel, newOptionsArray(), logger);
    setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));

    ChannelPipeline p = channel.pipeline();
    // 子EventLoopGroup用于完成Nio读写操作
    final EventLoopGroup currentChildGroup = childGroup;
    // 为workEventLoop配置的自定义Handler
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }
    // 设置附加参数
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);
	  // 为服务端Channel pipeline 配置 对应的Handler
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```
这里主要是为服务端Channel配置一些参数，以及对应的处理器`ChannelHandler`，注意这里不仅仅会把我们自定义配置的`ChannelHandler`加上去，同时还会自动帮我们加入一个系统`Handler `(`ServerBootstrapAcceptor`)，这就是Netty用来接收客户端请求的`Handler`，在`ServerBootstrapAcceptor`内部会完成`SocketChannel`的连接，`EventLoop`的绑定等操作，之后我们会着重分析这个类


##### Channel的注册

```java
// MultithreadEventLoopGroup
public ChannelFuture register(Channel channel) {
// next()会选择一个EventLoop来完成Channel的注册
    return next().register(channel);
}

// SingleThreadEventLoop
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
// AbstractChannel
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    ObjectUtil.checkNotNull(eventLoop, "eventLoop");
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
                new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }

    AbstractChannel.this.eventLoop = eventLoop;

    if (eventLoop.inEventLoop()) {
    // 注册逻辑
        register0(promise);
    } 
    ....
}
```

完成注册流程
1. 完成实际的Java `ServerSocketChannel`与`Select`选择器的绑定
2. 并触发`channelRegistered`以及`channelActive`事件

到这里为止，其实Netty服务端已经基本启动完成了，就差绑定一个监听端口了。可能读者会很诧异，怎么没有看到Nio线程轮询 IO事件的循环呢，讲道理肯定应该有一个死循环才对？那我们下面就把这段代码找出来

在之前的代码中，我们经常会看到这样一段代码

```java
// 往EventLoop中丢了一个异步任务（其实是同步的，因为只有一个Nio线程，不过因为是事件循环机制（丢到一个任务队列中），看起来像是异步的）
eventLoop.execute(new Runnable() {
    @Override
    public void run() {
        ...
    }
});
```
`eventLoop.execute`到底做了什么事情？
```java
private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();
    // 把当前任务添加到任务队列中
    addTask(task);
    // 不是Nio线程自己调用的话，则表明是初次启动
    if (!inEventLoop) {
    // 启动EventLoop的Nio线程
        startThread();
        ...
    }
    ...
}

/**
 * 启动EventLoop的Nio线程
 */
private void doStartThread() {
    assert thread == null;
    // 启动Nio线程
    executor.execute(new Runnable() {
        @Override
        public void run() {
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;
            updateLastExecutionTime();
            try {
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
            ... 
}

``` 
通过上面的代码可以知道这里主要做了两件事情
1. 创建的任务被丢入了一个队列中等待执行
2. 如果是初次创建，则启动Nio线程
3. SingleThreadEventExecutor.this.run(); 调用子类的Run实现（执行IO事件的轮询）
看下 `NioEventLoop`的`Run`方法实现

```java
protected void run() {
    int selectCnt = 0;
    for (;;) {
        try {
            int strategy;
            try {
            // 获取IO事件类型
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                ... 
                default:
                }
            } catch (IOException e) {
                // 出现异常 重建Selector
                rebuildSelector0();
                selectCnt = 0;
                handleLoopException(e);
                continue;
            }
            selectCnt++;
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            boolean ranTasks;
            if (ioRatio == 100) {
                try {
                    if (strategy > 0) {
                    // 处理对应事件，激活对应的ChannelHandler事件
                        processSelectedKeys();
                    }
                } finally {
                    // 处理完事件了才执行全部Task
                    ranTasks = runAllTasks();
                }
            }
            ...
        }   
    }
}
```
到这里的代码是不是就非常熟悉了，熟悉的死循环轮询事件

1. 通过Selector来轮询IO事件
2. 触发Channel所绑定的Handler处理对应的事件
3. 处理完IO事件了 会执行系统或用户自定义加入的Task

#### doBind0

实际的Bind逻辑在	`NioServerSocketChannel`中执行，我们直接省略前面一些冗长的调用，来看下最底层的调用代码，发现其实就是调用其绑定的Java Channel来执行对应的监听端口绑定逻辑
```java
protected void doBind(SocketAddress localAddress) throws Exception {
// 如果JDK版本大于7
    if (PlatformDependent.javaVersion() >= 7) {
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```

## 尾言

本篇文章把Netty的启动流程粗略的捋了一遍，目的不是为了抠细节，而是大致能够清楚Netty服务端启动时主要做了哪些事情，所以有些地方难免会比较粗略一笔带过。在后面的文章我会把一些细节的源码单独拎出来深入分析





