---
title: Java字节码常量池
tags:
  - JVM
  - java字节码
categories:
  - java基础
  - JVM
  - java字节码
date: 2019-04-17 15:48:48
---

## 引言

上篇文章简单介绍了`java Class` 字节码文件的基本格式。本文我们直接通过阅读字节码文件来进一步理解字节码中的常量池结构


首先我们新建一个最简单的Java文件

```java
public class Test {
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```

编译之后使用文本编辑器打开，可以看到`Class`文件最原始的16进制格式。最明显的`Class`文件的魔数`cafe babe`一眼就能看到

```
cafe babe 0000 0034 0022 0a00 0600 1409
0015 0016 0800 170a 0018 0019 0700 1a07
001b 0100 063c 696e 6974 3e01 0003 2829
5601 0004 436f 6465 0100 0f4c 696e 654e
756d 6265 7254 6162 6c65 0100 124c 6f63
616c 5661 7269 6162 6c65 5461 626c 6501
0004 7468 6973 0100 144c 636f 6d2f 796d
6d2f 6167 656e 742f 5465 7374 3b01 0004
6d61 696e 0100 1628 5b4c 6a61 7661 2f6c
616e 672f 5374 7269 6e67 3b29 5601 0004
6172 6773 0100 135b 4c6a 6176 612f 6c61
6e67 2f53 7472 696e 673b 0100 0a53 6f75
7263 6546 696c 6501 0009 5465 7374 2e6a
6176 610c 0007 0008 0700 1c0c 001d 001e
0100 0b68 656c 6c6f 2077 6f72 6c64 0700
1f0c 0020 0021 0100 1263 6f6d 2f79 6d6d
2f61 6765 6e74 2f54 6573 7401 0010 6a61
7661 2f6c 616e 672f 4f62 6a65 6374 0100
106a 6176 612f 6c61 6e67 2f53 7973 7465
6d01 0003 6f75 7401 0015 4c6a 6176 612f
696f 2f50 7269 6e74 5374 7265 616d 3b01
0013 6a61 7661 2f69 6f2f 5072 696e 7453
7472 6561 6d01 0007 7072 696e 746c 6e01
0015 284c 6a61 7661 2f6c 616e 672f 5374
7269 6e67 3b29 5600 2100 0500 0600 0000
0000 0200 0100 0700 0800 0100 0900 0000
2f00 0100 0100 0000 052a b700 01b1 0000
0002 000a 0000 0006 0001 0000 0009 000b
0000 000c 0001 0000 0005 000c 000d 0000
0009 000e 000f 0001 0009 0000 0037 0002
0001 0000 0009 b200 0212 03b6 0004 b100
0000 0200 0a00 0000 0a00 0200 0000 0b00
0800 0c00 0b00 0000 0c00 0100 0000 0900
1000 1100 0000 0100 1200 0000 0200 13
```

## 如何从上面的文件中找到常量池所在的位置呢

在上篇文章中我们已经分析过`Class`文件文件格式了，常量池的位置紧随在版本号之后，也就是从第9个字节开始就是常量池的内容了，下面就让我们一起来看下常量池的具体内容

### constant_pool_count u2

从第9个字节开始往后的两个字节，是`constant_pool_count`，我们把它叫做常量计数器，比较特殊的是它的下标是从1开始的，也就是说真正的常量池数量是`constant_pool_count - 1`个，那好现在让我们来看看上面的那个普通`Class`文件包含了多少个常量表（`cp_info`）

![常量计数](http://upload-images.jianshu.io/upload_images/2717496-1e1c7e2eecf80bc1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到`constant_pool_count`标识位是`0022`，转为10进制就是34。也就是说包含`34 - 1 = 33`个常量表内容

### cp_info

上文经过分析我们已经确定了字节码文件包含的常量表个数，那么接下来我们要具体怎么看呢？

下面让我们来具体分析下`cp_info`的结构

![image.png](http://upload-images.jianshu.io/upload_images/2717496-e45775e88817026d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到每个`cp_info`的第一个字节位都是`tag`，用于标识常量表所属的具体类型，上篇文章我们说过，常量表之所以非常复杂，就是因为它的类型众多，有多大14种类型。那么这些类型的区分就是通过`tag`来区分。我们会逐个分析这14种类型

先来看下这14常量池类型

![image.png](http://upload-images.jianshu.io/upload_images/2717496-5f84affafe924089.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

还是回到上述字节码的16进制文件，我们接着往下看

![image.png](http://upload-images.jianshu.io/upload_images/2717496-e413777a45543d46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到`tag = 10`，根据上文我们列出的细分类型可以确定这第一个出现的常量表类型是`
constant_Methodref_info`。那好现在我们再去具体的看下`constant_Methodref_info`表的具体内容

![constant_Methodref_info.png](http://upload-images.jianshu.io/upload_images/2717496-10a980be4439b580.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到紧跟在`tag`之后其实是两项index索引值，分别指向`constant_Class_info`和`constant_NameAndType_info`。

继续往下阅读，可以知道具体的索引值分别为`0006`，`0014`。也就是指向常量池的第6项和第20项

下面为了方便阅读，我们直接使用`javap -v Test`命令生成方便我们阅读的字节码文件，当然如果有兴趣的话，直接阅读源文件也是oK的

```
public class com.ymm.agent.Test
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #5                          // com/ymm/agent/Test
  super_class: #6                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #21.#22        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #23            // hello world
   #4 = Methodref          #24.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #26            // com/ymm/agent/Test
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lcom/ymm/agent/Test;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               SourceFile
  #19 = Utf8               Test.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = Class              #28            // java/lang/System
  #22 = NameAndType        #29:#30        // out:Ljava/io/PrintStream;
  #23 = Utf8               hello world
  #24 = Class              #31            // java/io/PrintStream
  #25 = NameAndType        #32:#33        // println:(Ljava/lang/String;)V
  #26 = Utf8               com/ymm/agent/Test
  #27 = Utf8               java/lang/Object
  #28 = Utf8               java/lang/System
  #29 = Utf8               out
  #30 = Utf8               Ljava/io/PrintStream;
  #31 = Utf8               java/io/PrintStream
  #32 = Utf8               println
  #33 = Utf8               (Ljava/lang/String;)V
{
  public com.ymm.agent.Test();
    descriptor: ()V
.... 省略其他字节码
```

通过`javap` 生成的字节码文件，看起来就舒服多了。现在继续回到第一个常量池项`constant_Methodref_info`，可以知道它是由`constant_Class_info`，`constant_NameAndType_info`两项的索引值组成，根据索引值`6`,`20`我们可以找到具体常量池项内容。

先找到索引值为`6`的`constant_Class_info`的内容，可以发现其具体内容也是指向了一个索引值`27`，继续找索引值为`27`常量项

![image.png](http://upload-images.jianshu.io/upload_images/2717496-5c7eb023de534a0b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

找到索引值为`27`的常量项，可以发现是一个`constant_utf8_info`类型的字符串(java/lang/Object)，这也就是我们之前提到的类的全限定名称

![image.png](http://upload-images.jianshu.io/upload_images/2717496-22a9e1eb96234b8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



到目前为止大家应该已经找到了阅读常量池的感觉了，那么我们去查看索引项为`20`的`constant_NameAndType_info`的方法也是一样的

![image.png](http://upload-images.jianshu.io/upload_images/2717496-4332f33c24364301.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


`constant_NameAndType_info`指向索引值为`7`和`8`的内容

![image.png](http://upload-images.jianshu.io/upload_images/2717496-0e0729faf2be3012.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以知道方法名称为`init`，类型为`void`方法

到目前为止我们大概可以知道此常量池的第一项表示其实是一个无参的构造方法，并且是编译器为我们自动生成的

其实大多数复杂的常量池内容最终都会索引到一个`constant_utf8_info`字符串类型，用于描述类全限定名，方法或字段名称及描述符等等 ..

## 尾言

本篇文章我们只是通过一个示例来阅读`Class`文件的常量池内容，具体的常量池结构在上篇文章中我们都有列出，并且阅读的方法也是与本文类似。



