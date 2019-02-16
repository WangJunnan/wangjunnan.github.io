---
title: Tomcat源码解析三 Connector连接器
tags:
  - Tomcat
categories:
  - web容器
  - Tomcat
date: 2019-02-13 16:41:26
---

## 引言

上文分析了Tomcat的启动流程，我们已经大致理清了Tomcat启动的整个流程，本文将会对`Connector`连接器的创建进行分析


## 整体架构

![image.png](https://upload-images.jianshu.io/upload_images/2717496-502a745b5956d6bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上图完整了概括了整个`Connector`的架构体系，先简单的介绍一下各个组件的功能

* Endpoint 用来处理底层Socket的网络连接
* Processor 用来实现HTTP协议的
* Adapter 将请求适配到Servlet容器进行具体的处理

## org.apache.catalina.connector.Connector

我们先来看下`org.apache.catalina.connector.Connector`这个主体类的构造方法，`Connector`类初始化是在`Tomcat`读取配置文件时就完成的

```
public Connector(String protocol) {
    boolean aprConnector = AprLifecycleListener.isAprAvailable() &&
            AprLifecycleListener.getUseAprConnector();
	  // 判断协议类型
    if ("HTTP/1.1".equals(protocol) || protocol == null) {
        if (aprConnector) {
            protocolHandlerClassName = "org.apache.coyote.http11.Http11AprProtocol";
        } else {
            protocolHandlerClassName = "org.apache.coyote.http11.Http11NioProtocol";
        }
    } else if ("AJP/1.3".equals(protocol)) {
        if (aprConnector) {
            protocolHandlerClassName = "org.apache.coyote.ajp.AjpAprProtocol";
        } else {
            protocolHandlerClassName = "org.apache.coyote.ajp.AjpNioProtocol";
        }
    } else {
        protocolHandlerClassName = protocol;
    }

    // Instantiate protocol handler
    ProtocolHandler p = null;
    try {
        Class<?> clazz = Class.forName(protocolHandlerClassName);
        p = (ProtocolHandler) clazz.getConstructor().newInstance();
    } catch (Exception e) {
        log.error(什么.getString(
                "coyoteConnector.protocolHandlerInstantiationFailed"), e);
    } finally {
        this.protocolHandler = p;
    }

    // Default for Connector depends on this system property
    setThrowOnFailure(Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE"));
}
```
这里其实是拿到`server.xml`中`Connector`的协议配置，利用反射创建`ProtocolHandle`，`Connector`就是使用`ProtocolHandler`来处理请求的，不同的`ProtocolHandle`r代表不同的连接类型。

因为我这里使用的是`tomcat9`的源码版本，可以看到其已经淘汰了`BIO`。默认的http1.1协议处理类已经是`org.apache.coyote.http11NioProtocol`了，下面我们就以`http11NioProtocol`继续往下分析

## org.apache.coyote.http11NioProtocol

通过查看`Http11NioProtocol`的构造方法，可知`Endpoint`的实现类是`NioEndpoint`
```
public Http11NioProtocol() {
    super(new NioEndpoint());
}
```
`Endpoint`上文有说过是用来处理底层Socket网络连接的，下面就让我们来看下`NioEndpoint`的实现

## NioEndpoint

还是先看下启动方法 `startInternal`中的实现
```
public void startInternal() throws Exception {
        .....
        // 初始化Poller数组，启动Poller线程
        pollers = new Poller[getPollerThreadCount()];
        for (int i=0; i<pollers.length; i++) {
            pollers[i] = new Poller();
            Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
            pollerThread.setPriority(threadPriority);
            pollerThread.setDaemon(true);
            pollerThread.start();
        }
        // 启动 Acceptor 线程
        startAcceptorThreads();
    }
}
```
这里我省略了其他代码，可以看到在这里初始化了多个`Poller`类，并单独启动了线程，这里的每个`Poller`其实都绑定了一个`Selector`选择器（）。并且调用`startAcceptorThreads`方法启动了Acceptor线程，用来接收新的请求。下面我们继续看`startAcceptorThreads`方法

```
protected void startAcceptorThreads() {
	  // 获取Acceptor线程数 默认是1
    int count = getAcceptorThreadCount();
    acceptors = new ArrayList<>(count);

    for (int i = 0; i < count; i++) {
        Acceptor<U> acceptor = new Acceptor<>(this);
        String threadName = getName() + "-Acceptor-" + i;
        acceptor.setThreadName(threadName);
        acceptors.add(acceptor);
        Thread t = new Thread(acceptor, threadName);
        t.setPriority(getAcceptorThreadPriority());
        t.setDaemon(getDaemon());
        t.start();
    }
}
```
上面的代码根据配置启动了多个`Acceptor`线程，下面就去看下`Acceptor`类的`run`方法

## Acceptor

```
public void run() {

    int errorDelay = 0;

    // Loop until we receive a shutdown command
    while (endpoint.isRunning()) {
				 ....
            try {
                // 接收新的请求
                socket = endpoint.serverSocketAccept();
            } catch (Exception ioe) {
                // We didn't get a socket
                endpoint.countDownConnection();
                if (endpoint.isRunning()) {
                    // Introduce delay if necessary
                    errorDelay = handleExceptionWithDelay(errorDelay);
                    // re-throw
                    throw ioe;
                } else {
                    break;
                }
            }
            // Successful accept, reset the error delay
            errorDelay = 0;

            // Configure the socket
            if (endpoint.isRunning() && !endpoint.isPaused()) {
            	// 设置新的Socket连接与Poller绑定，并设置相关参数
                if (!endpoint.setSocketOptions(socket)) {
                    endpoint.closeSocket(socket);
                }
            } else {
                endpoint.destroySocket(socket);
            }
        } catch (Throwable t) {
     
    }
    state = AcceptorState.ENDED;
}

// NioEndpoint.class 中
protected boolean setSocketOptions(SocketChannel socket) {
    // Process the connection
    try {
        // 设置此Socket连接未非阻塞
        socket.configureBlocking(false);
        Socket sock = socket.socket();
        // 设置此Socket的相关参数值
        socketProperties.setProperties(sock);

        NioChannel channel = nioChannels.pop();
        if (channel == null) {
            SocketBufferHandler bufhandler = new SocketBufferHandler(
                    socketProperties.getAppReadBufSize(),
                    socketProperties.getAppWriteBufSize(),
                    socketProperties.getDirectBuffer());
            // 判断是否开启ssl
            if (isSSLEnabled()) {
                channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
            } else {
                channel = new NioChannel(socket, bufhandler);
            }
        } else {
            channel.setIOChannel(socket);
            channel.reset();
        }
        // 绑定 Poller 其实就是绑定选择器 Selector
        getPoller0().register(channel);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        try {
            log.error(sm.getString("endpoint.socketOptionsError"), t);
        } catch (Throwable e) {
            ExceptionUtils.handleThrowable(e);
        }
        // Tell to close the socket
        return false;
    }
    return true;
}
```
上面代码逻辑主要做了两件事情

* 调用`NioEndpoint`的`serverSocketAccept`方法来接收新的请求，**注意这里是阻塞的**
* 调用`NioEndpoint`的`setSocketOptions`方法对新接收的`Socket`请求，配置相关信息，并绑定`Poller`(绑定选择器 Selector)

## Poller

接下来我们将会分析`Poller`类，是`NioEndpoint`的内部类

```
public void register(final NioChannel socket) {
    socket.setPoller(this);
    NioSocketWrapper ka = new NioSocketWrapper(socket, NioEndpoint.this);
    socket.setSocketWrapper(ka);
    ka.setPoller(this);
    ka.setReadTimeout(getConnectionTimeout());
    ka.setWriteTimeout(getConnectionTimeout());
    ka.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
    ka.setSecure(isSSLEnabled());
    PollerEvent r = eventCache.pop();
    ka.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
    if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
    else r.reset(socket,ka,OP_REGISTER);
    addEvent(r);
}
```
register方法就是将新的`Socket`连接与`Selector`进行绑定，并注册监听读事件

```
public void run() {
    // Loop until destroy() is called
    while (true) {

        boolean hasEvents = false;

        try {
            if (!close) {
                hasEvents = events();
                if (wakeupCounter.getAndSet(-1) > 0) {
                    //if we are here, means we have other stuff to do
                    //do a non blocking select
                    keyCount = selector.selectNow();
                } else {
                    keyCount = selector.select(selectorTimeout);
                }
                wakeupCounter.set(0);
            }
            if (close) {
                events();
                timeout(0, false);
                try {
                    selector.close();
                } catch (IOException ioe) {
                    log.error(什么.getString("endpoint.nio.selectorCloseFail"), ioe);
                }
                break;
            }
        } catch (Throwable x) {
            ExceptionUtils.handleThrowable(x);
            log.error(什么.getString("endpoint.nio.selectorLoopError"), x);
            continue;
        }
        //either we timed out or we woke up, process events first
        if ( keyCount == 0 ) hasEvents = (hasEvents | events());

        Iterator<SelectionKey> iterator =
            keyCount > 0 ? selector.selectedKeys().iterator() : null;
        // Walk through the collection of ready keys and dispatch
        // any active event.
        while (iterator != null && iterator.hasNext()) {
            SelectionKey sk = iterator.next();
            NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
            // Attachment may be null if another thread has called
            // cancelledKey()
            if (attachment == null) {
                iterator.remove();
            } else {
                iterator.remove();
                processKey(sk, attachment);
            }
        }//while

        //process timeouts
        timeout(keyCount,hasEvents);
    }//while

    getStopLatch().countDown();
}
```

run方法中的一大堆代码，多是与NIO相关，主要逻辑就是调用`selector`的`select()`函数，监听就绪事件。这里我们可以直接看`processKey`方法，这里是根据`SelectionKey`来分别执行具体逻辑

```
protected void processKey(SelectionKey sk, NioSocketWrapper attachment) {
    try {
        if ( close ) {
            cancelledKey(sk);
        } else if ( sk.isValid() && attachment != null ) {
            if (sk.isReadable() || sk.isWritable() ) {
                if ( attachment.getSendfileData() != null ) {
                    processSendfile(sk,attachment, false);
                } else {
                    unreg(sk, attachment, sk.readyOps());
                    boolean closeSocket = false;
                    // 如果是可读事件就绪
                    if (sk.isReadable()) {
                    	  // 执行具体逻辑的地方
                        if (!processSocket(attachment, SocketEvent.OPEN_READ, true)) {
                            closeSocket = true;
                        }
                    }
                    // 如果是可写事件就绪
                    if (!closeSocket && sk.isWritable()) {
                        if (!processSocket(attachment, SocketEvent.OPEN_WRITE, true)) {
                            closeSocket = true;
                        }
                    }
                    if (closeSocket) {
                        cancelledKey(sk);
                    }
                }
            }
        } else {
            //invalid key
            cancelledKey(sk);
        }
    } catch ( CancelledKeyException ckx ) {
        cancelledKey(sk);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        log.error(sm.getString("endpoint.nio.keyProcessingError"), t);
    }
}
```
`processKey`方法也是直接调用了`AbstractEndpoint`的`processSocket`方法

```
public boolean processSocket(SocketWrapperBase<S> socketWrapper,
        SocketEvent event, boolean dispatch) {
    try {
        if (socketWrapper == null) {
            return false;
        }
        SocketProcessorBase<S> sc = processorCache.pop();
        if (sc == null) {
        		 // 创建一个 SocketProcessor 实例
            sc = createSocketProcessor(socketWrapper, event);
        } else {
            sc.reset(socketWrapper, event);
        }
        Executor executor = getExecutor();
        if (dispatch && executor != null) {
            executor.execute(sc);
        } else {
            // 执行
            sc.run();
        }
    } catch (RejectedExecutionException ree) {
        getLog().warn(什么.getString("endpoint.executor.fail", socketWrapper) , ree);
        return false;
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        // This means we got an OOM or similar creating a thread, or that
        // the pool and its queue are full
        getLog().error(什么.getString("endpoint.process.fail"), t);
        return false;
    }
    return true;
}
```

## SocketProcessor

```
protected void doRun() {
        NioChannel socket = socketWrapper.getSocket();
        SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector()); 
             .... 省略
             
             if (handshake == 0) {
                SocketState state = SocketState.OPEN;
                // Process the request from this socket
                if (event == null) {
                		// 获取 ConnectionHandler并调用process执行具体逻辑
                    state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
                } else {
                    state = getHandler().process(socketWrapper, event);
                }
                if (state == SocketState.CLOSED) {
                    close(socket, key);
                }
            } else if (handshake == -1 ) {
                close(socket, key);
            } else if (handshake == SelectionKey.OP_READ){
                socketWrapper.registerReadInterest();
            } else if (handshake == SelectionKey.OP_WRITE){
                socketWrapper.registerWriteInterest();
            }
        }
    }
}
```
SocketProcessor逻辑比较简单，doRun方法继续会往下调用，最终`http协议`的解析是在`Http11Processor`的`service`中进行，`Http11Processor`就对应上文架构图的`Process`模块，在`Process`完成`Http`协议解析之后，会由适配器进行适配后再交给`Servlet`容器进行具体处理

## 总结

本文分析`Tomcat`的`Connector`连接器的部分源码，`Connector`是`Tomcat`的核心组件，`Connector`组件用于等待用户的请求，包括支持`http1.1`，`http2`等协议，解析用户请求，封装请求信息，最后才交给我们熟悉的	`Servlet`处理。阅读此源码对于理解`http`协议也有很大的帮助。



