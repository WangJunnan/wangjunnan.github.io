---
title: Java类加载机制
tags:
  - JVM
categories:
  - java基础
  - JVM
date: 2019-04-02 15:35:40
---

## 引言

介绍一下java类加载的过程

## 类加载的时机

类从被加载到java虚拟机内存中开始，被分为7个阶段，包括`加载`，`验证`，`准备`，`解析`，`初始化`，`使用`，`卸载`

![image.png](https://upload-images.jianshu.io/upload_images/2717496-793f9be59d3a7ea2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



这里我们暂且主要看加载阶段和初始化阶段，java虚拟机主要是做了三件事情

## 加载

1. 通过类的全限定名获取定义此类的二进制字节流
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
3. 在内存中生成一个代表这个类的`Class`对象

注意这里虚拟机并没有规定类的二进制字节流的来源，所以我们其实通过自定义类加载器来实现从其他途径（如网络中获取）中获取Class文件的二进制字节流

## 初始化

初始化阶段是类加载的最后一个阶段，可以理解成`初始静态块及静态变量`，在准备阶段，静态变量已经赋了初始值，初始化阶段赋的就是具体的值。

**初始化的必要条件，并且有且只有这5种情况才会触发类的初始化**

1. 遇到`new`,`getstatic`,`putstatic`,`invokestatic`这四个字节码时，如果还未执行类的初始化则会执行初始化
2. 使用`java.lang.reflect`包进行反射调用时，如果还没有进行初始化，则会执行初始化
3. 当初始化一个类时，发现其父类还没有初始化，则会初始化其父类
4. 当虚拟机启动时，用户需要指定一个主类，这个主类会被初始化
5. 使用jdk7动态语言支持时，会执行初始化

以上5中场景，称为对一个类的主动引用。除这5种方式之外，任何其他引用都被称为被动引用，并且都不会触发类的初始化。

下面举三种情况来说明被动引用

* 通过子类引用父类静态字段，不会触发父类的初始化

```
public class BoshiCat extends Cat {

    static {
        System.out.println("BoshiCat init");
    }
}

public class Cat {
    static {
        System.out.println("Cat init");
    }
    public static int value = 10;
}

public class Test {
    public static void main(String[] args) {
        System.out.println(BoshiCat.value);
    }
}
// 输出 10
```

* 通过数组来引用类，不会触发引用类的初始化

```
public class Test {
    public static void main(String[] args) {
        Cat []  cats = new Cat[10];
    }
}

// 输出 10
```

* 引用`static final`修饰的常量，因为常量在编译阶段就被写进了常量池

```
public class Cat {
    static {
        System.out.println("Cat init");
    }
    public static final int value = 10;
}

public class Test {
    public static void main(String[] args) {
        System.out.println(Cat.value);
    }
}
// 输出 10
```
