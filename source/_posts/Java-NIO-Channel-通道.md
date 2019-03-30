---
title: Java NIO Channel 通道
tags:
  - NIO
categories:
  - java基础
  - NIO
date: 2019-01-10 16:35:06
---

## 引言

上一篇文章主要介绍了`Java NIO Buffer`的使用及内部实现原理，本期我们将会认识到`Java NIO` 另一个非常重要的特性`Channel`（通道）


## 对比传统IO流

在认识`Java NIO Channel`通道之前，我们有必要对比一下`NIO Channel`与传统I/O流的区别，以及它的优势在哪里:

* 既可以从通道中读取数据，又可以写数据到通道。**但流的读写通常是单向的**。
* 通道可以异步地读写。（不会阻塞进程）
* 通道中的数据总是要先读到一个Buffer，或者总是要从一个Buffer中写入(面向缓冲区)。

## Channel 分类

下面列举的是`JAVA NIO Channel`的一些主要实现：

* FileChannel 文件通道
* SocketChannel 通过TCP读写网络中的数据。
* ServerSocketChannel TCP服务端，可以读写网络中的数据，可以监听新进来的TCP连接
* DatagramChannel 通过UDP读写网络中的数据

接下来我将主要对**文件通道**和**Socket通道**的使用做分析

## 文件通道 FileChannel

我们无法直接打开一个`FileChannel`，需要通过使用一个`InputStream`、`OutputStream`或`RandomAccessFile`来获取一个FileChannel实例。并且**`FileChannel`无法设置为非阻塞模式，它总是运行在阻塞模式下,也就是说FileChannel的read write方法都是阻塞的**。

### 读写数据

下面是通过RandomAccessFile打开FileChannel进行数据读写的示例：

写数据至指定文件
```
RandomAccessFile file = new RandomAccessFile("hello.txt", "rw");
FileChannel fileChannel = file.getChannel();
ByteBuffer buffer = ByteBuffer.allocate(512);
// 将要写的数据写入缓冲区
buffer.put("见到你真好".getBytes("utf-8"));

// 切换缓冲区至可读模式
buffer.flip();

// 开始写入文件
fileChannel.write(buffer); // 返回当前写入的字节数
file.close();
fileChannel.close();
```

从文件中读取数据
```
RandomAccessFile file = new RandomAccessFile("hello.txt", "rw");
FileChannel fileChannel = file.getChannel();
ByteBuffer buffer = ByteBuffer.allocate(512);
fileChannel.read(buffer);

// 切换缓冲区至可写模式
buffer.flip();
byte data [] = new byte[buffer.limit()];
// 将数据写至 byte数组
buffer.get(data);
System.out.println(new String(data));
file.close();
fileChannel.close();
```


## Socket 通道

同样，相对于传统`BIO`的`Socket`，`ServerSocket`。`NIO`也提供了对`TCP/IP`协议封装的套接字通道：

* SocketChannel 
* ServerSocketChannal

下面我们来看一下它们的基本使用

### SocketChannel 打开通道

```
// 打开一个通道
SocketChannel channel = SocketChannel.open();

// 与Server建立连接
channel.connect(new InetSocketAddress("127.0.0.1", 8890));
```
### ServerSocketChannel 打开通道接受连接

```
// 打开通道
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
// 绑定端口
serverSocketChannel.socket().bind(new InetSocketAddress(8890));
// 接受一个Socket连接 默认是阻塞模式
SocketChannel channel = serverSocketChannel.accept();
```

### 要点

`SocketChannel`,`ServerSocketChannel` 也可以设置成非阻塞模式，设置成非阻塞模式的话，`accept`，`read`，`write` 方法会立即返回， 因此需要我们在代码不断的轮询去判断连接和数据是否就绪

## 尾言

本篇文章主要介绍了`Java NIO`中`Channel`(通道)的使用，其使用与传统I/O流相比其实更加的易用。下期将会介绍`Java NIO`的核心`Selector`选择器的基本使用


