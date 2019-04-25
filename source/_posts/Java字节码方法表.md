---
title: Java字节码方法表
tags:
  - JVM
  - java字节码
categories:
  - java基础
  - JVM
  - java字节码
date: 2019-04-25 15:50:02
---

## 引言

因为字段表和方法表的结构类似，所以我们直接分析Java字节码的方法表内容，理解了方法表，自然就理解了字段表

## 方法表

方法表在`Class`文件中的位置是在字段表之后的，具体的结构我们根据下表再来回顾一下

类型 | 名称 | 数量 
--- | --- | --- 
u2 | access_flas | 1 
u2 | name_index | 1
u2 | descriptor_index | 1
u2 | attributes_count | 1
attribute_info | 属性表 | attributes_count

还是跟上篇文章一样，我们写一个简单的Java类

```java

public class Test {

 public String sayHello() {

 String sayStr = "hello world";

 return sayStr;

 }

 public static void main(String[] args) {

 Test test = new Test();

 System.out.println(test.sayHello());

 }

}

```

使用`javap`翻译字节码文件

```

public class com.ymm.agent.Test

 minor version: 0

 major version: 52

 flags: (0x0021) ACC_PUBLIC, ACC_SUPER

 this_class: #3 // com/ymm/agent/Test

 super_class: #8  // java/lang/Object

 interfaces: 0, fields: 0, methods: 3, attributes: 1

Constant pool:

 #1 = Methodref #8.#27  // java/lang/Object."<init>":()V

 #2 = String  #28 // hello world

 #3 = Class #29 // com/ymm/agent/Test

 #4 = Methodref #3.#27  // com/ymm/agent/Test."<init>":()V

 #5 = Fieldref  #30.#31 // java/lang/System.out:Ljava/io/PrintStream;

 #6 = Methodref #3.#32  // com/ymm/agent/Test.sayHello:()Ljava/lang/String;

 #7 = Methodref #33.#34 // java/io/PrintStream.println:(Ljava/lang/String;)V

 #8 = Class #35 // java/lang/Object

 #9 = Utf8  <init>

 #10 = Utf8  ()V

 #11 = Utf8  Code

 #12 = Utf8  LineNumberTable

 #13 = Utf8  LocalVariableTable

 #14 = Utf8  this

 #15 = Utf8  Lcom/ymm/agent/Test;

 #16 = Utf8  sayHello

 #17 = Utf8  ()Ljava/lang/String;

 #18 = Utf8  sayStr

 #19 = Utf8  Ljava/lang/String;

 #20 = Utf8  main

 #21 = Utf8  ([Ljava/lang/String;)V

 #22 = Utf8  args

 #23 = Utf8  [Ljava/lang/String;

 #24 = Utf8  test

 #25 = Utf8  SourceFile

 #26 = Utf8  Test.java

 #27 = NameAndType #9:#10  // "<init>":()V

 #28 = Utf8  hello world

 #29 = Utf8  com/ymm/agent/Test

 #30 = Class #36 // java/lang/System

 #31 = NameAndType #37:#38 // out:Ljava/io/PrintStream;

 #32 = NameAndType #16:#17 // sayHello:()Ljava/lang/String;

 #33 = Class #39 // java/io/PrintStream

 #34 = NameAndType #40:#41 // println:(Ljava/lang/String;)V

 #35 = Utf8  java/lang/Object

 #36 = Utf8  java/lang/System

 #37 = Utf8  out

 #38 = Utf8  Ljava/io/PrintStream;

 #39 = Utf8  java/io/PrintStream

 #40 = Utf8  println

 #41 = Utf8  (Ljava/lang/String;)V

{

 public com.ymm.agent.Test();

 descriptor: ()V

 flags: (0x0001) ACC_PUBLIC

 Code:

 stack=1, locals=1, args_size=1

 0: aload_0

 1: invokespecial #1 // Method java/lang/Object."<init>":()V

 4: return

 LineNumberTable:

 line 9: 0

 LocalVariableTable:

 Start Length Slot Name  Signature

 0  5  0 this  Lcom/ymm/agent/Test;

 public java.lang.String sayHello();

 descriptor: ()Ljava/lang/String;

 flags: (0x0001) ACC_PUBLIC

 Code:

 stack=1, locals=2, args_size=1

 0: ldc  #2 // String hello world

 2: astore_1

 3: aload_1

 4: areturn

 LineNumberTable:

 line 12: 0

 line 13: 3

 LocalVariableTable:

 Start Length Slot Name  Signature

 0  5  0 this  Lcom/ymm/agent/Test;

 3  2  1 sayStr  Ljava/lang/String;

 public static void main(java.lang.String[]);

 descriptor: ([Ljava/lang/String;)V

 flags: (0x0009) ACC_PUBLIC, ACC_STATIC

 Code:

 stack=2, locals=2, args_size=1

 0: new  #3 // class com/ymm/agent/Test

 3: dup

 4: invokespecial #4 // Method "<init>":()V

 7: astore_1

 8: getstatic  #5 // Field java/lang/System.out:Ljava/io/PrintStream;

 11: aload_1

 12: invokevirtual #6 // Method sayHello:()Ljava/lang/String;

 15: invokevirtual #7 // Method java/io/PrintStream.println:(Ljava/lang/String;)V

 18: return

 LineNumberTable:

 line 17: 0

 line 18: 8

 line 19: 18

 LocalVariableTable:

 Start Length Slot Name  Signature

 0 19  0 args  [Ljava/lang/String;

 8 11  1 test  Lcom/ymm/agent/Test;

}

SourceFile: "Test.java"

```

通过上文的`interfaces: 0, fields: 0, methods: 3, attributes: 1`，我们可以知道该文件一共包含`3`个方法，分别对应着`无参构造`,`sayHello`,`main`三个方法

好了，现在让我们直接略过前面的一大推常量池项，

直接阅读我们自己写的`sayHello`方法的字节码内容

```

public java.lang.String sayHello();

 descriptor: ()Ljava/lang/String; /* 方法描述符 */

 flags: (0x0001) ACC_PUBLIC /* 访问标识 */

 Code: /* code属性  也就是我们写的具体代码翻译成的字节码指令内容 */

 stack=1, locals=2, args_size=1 /* 分别指操作数栈深度；本地变量所需的存储空间（slot为单位）；参数个数 */

 0: ldc  #2 // String hello world /*将一个常量加载到操作数栈*/

 2: astore_1 /* 将一个数值从操作数栈存储到局部变量表 */

 3: aload_1 /* 将一个数值从局部变量表加载到操作数栈 */

 4: areturn /* 将栈顶第一个元素返回 */

 LineNumberTable: /* 字节码与java代码行数对应关系 一般用于调试 */

 line 12: 0

 line 13: 3

 LocalVariableTable: /* 局部变量表 */

 Start Length Slot Name  Signature

 0  5  0 this  Lcom/ymm/agent/Test;

 3  2  1 sayStr  Ljava/lang/String;

```

根据上表我们知道方法表的第一个内容的是`access_flas`，占一个字节。表中的`access_flas`其实就对应着上文中的  `flags: (0x0001) ACC_PUBLIC`，标识这个方法是`public`的，接下在的`name_index`和`descriptor_index`，在上文中的体现分别对应着`sayHello`和`()Ljava/lang/String;`。

这里解释一下描述符的概念，在介绍`Class文件结构`的文章中有提到，这里再提一遍:

**描述符的作用是用来描述字段的数据类型，方法的参数列表和返回值**，根据描述符规则，基本数据类型（byte,char,int,long,float,double,short,boolean）以及代表无返回值的`void`类型都用一个大写字符来表示，而对象类型则用`L`加对象的全限定名来表示

最重要的内容其实还是在`attribute_info`属性表中的`code`内容，也就是我们写的具体代码翻译成的字节码指令内容，这块内容是我们阅读字节码文件的重中之重。下面我们就来阅读一下上文的字节码内容

1. ldc #2 将常量池中索引为2的字符串加载到操作数栈顶

2. astore_1 将一个数值从操作数栈存储到局部变量表

3. aload_1 将一个数值从局部变量表加载到操作数栈

4. areturn 将栈顶第一个元素返回

可以看到上面的四个步骤的操作其实都是基于栈的操作，这里提一下java虚拟机栈的栈帧结构

![image.png](https://upload-images.jianshu.io/upload_images/2717496-0b89369329932afb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于字节码子令集可以参考[[java字节码子令集](https://blog.csdn.net/zqz_zqz/article/details/79484757)](https://blog.csdn.net/zqz_zqz/article/details/79484757)

## 有趣的例子 ---

下面我们再来看一个有趣的例子，大家思考一下这个执行这个方法的返回值会是多少？

```

public class Test {

 public int inc() {

 int x;

 try {

 x = 1;

 return x;

 } catch (Exception e) {

 x = 2;

 return x;

 } finally {

 x = 3;

 }

 }

}

```

代码非常简单，我想大家应该也都知道正确答案，当没有出现异常的时候，返回值为1，出现异常的话则为2（当然这里不会抛异常）。可是如果我们在finally快里加句代码`System.out.prinln("do it")`，然后再执行这个方法，其实是可以看到`do it`被打印了，也就是说在执行`return`之前，finally快中的代码是被执行了的，那么这里就有一个有趣的问题了，为何返回仍然是2而不是3呢？

下面我们就从字节码文件中来找出答案

```

public int inc();

 descriptor: ()I

 flags: (0x0001) ACC_PUBLIC

 Code:

 stack=1, locals=5, args_size=1

 0: iconst_1 // 将int = 1的值压入栈顶

 1: istore_1 // 弹出栈顶元素，存入位置为1的局部变量表

 2: iload_1 // 从位置为1的局部变量中取出元素压入栈顶

 3: istore_2 // 弹出栈顶元素，存入位置2的局部变量中

 4: iconst_3 // 将int = 3的值压入栈顶 （这里执行finally块中的代码了）

 5: istore_1 // 弹出栈顶元素，存入位置1的局部变量中

 6: iload_2 // 从位置为2的局部变量中取出元素压入栈顶

 7: ireturn // 返回栈顶元素2 (哈哈哈  看到没有，这里返回的是2，没有异常的话，这里方法就返回了)

 8: astore_2 // 将栈顶的异常引用，存入位置2的局部变量中  （这里就是异常捕获的代码了）

 9: iconst_2 // 将int = 2的值压入栈顶

 10: istore_1 // 弹出栈顶元素，存入位置1的局部变量中

 11: iload_1 // 从位置为1的局部变量中取出元素压入栈顶

 12: istore_3 // 弹出栈顶元素，存入位置3的局部变量中

 13: iconst_3 // 将int = 3的值压入栈顶

 14: istore_1 // 弹出栈顶元素，存入位置1的局部变量中

 15: iload_3 // 从位置为3的局部变量中取出元素压入栈顶

 16: ireturn // 返回栈顶元素 2

 17: astore 4 // 将栈顶异常引用存入位置为4的局部变量表中

 19: iconst_3 // 将int = 3的值压入栈顶

 20: istore_1 // 弹出栈顶元素，存入位置1的局部变量中

 21: aload  4 将位置为4的局部变量引用压入栈顶

 23: athrow // 将栈顶的异常抛出

 Exception table: // 异常表

 from to target type

 0  4  8  Class java/lang/Exception

 0  4 17  any

 8 13 17  any

 17 19 17  any

```

如果你已经看懂了上面的字节码，你应该会豁朗开朗。原因其实就是在于这里开辟了两个局部变量，`finally`块中的代码也确实执行了，只是将变量存入了局部变量表中的另一个位置，并且通过这个例子，也可以发现，无论什么情况`finally`块中的代码都会执行。

## 尾言

好了，本文到这里就差不多结束了。对于字节码的探索个人觉的还是非常有意思的，之后应该也会探索更多有意思的东西

