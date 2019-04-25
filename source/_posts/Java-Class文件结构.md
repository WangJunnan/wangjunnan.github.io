---
title: Java Class文件结构
tags:
  - JVM
  - java字节码
categories:
  - java基础
  - JVM
  - java字节码
date: 2019-04-04 15:47:26
---

## 引言

我们都知道java是跨平台的，原因就在于各个平台的java虚拟机可以载入和执行同一种平台无关的字节码文件，也就是说java虚拟机不与包括Java在内的任何语言绑定，只于`Class`文件这种二进制格式文件所关联。

基于这样的设计，到目前为止已经出现了很多基于Java虚拟机的语言

如`groovy`最终都会编译成`class`文件



## Class文件结构

一个Class文件唯一对应一个类或接口

现在让我们来看下`Class`文件的基本结构

Class文件以8位字节为基本单位的二进制文件，**各个数据项目严格的按照顺序紧凑的排列在`Class`文件之中，中间没有任何分隔符，这使得整个`Class`文件中存储的内容几乎全部是运行时的必要数据**

`Class`文件的二进制文件只有两种数据类型：无符号数和表，后面的解析都会以这两种数据类型为基础

* 无符号数

无符号数是最基本的数据类型，以`u1`,`u2`,`u4`,`u8`来分别代表1个字节，2个字节，4个字节，8个字节的无符号数，无符号数用来描述数字，索引引用，数量值，以及以UTF-8编码的字符串

* 表

表则是由多个无符号数或**其他表组成的复合数据结构**，表的名称一般以`_info`结尾。**所以整个`Class`文件其实就是一张特殊的表**

下面表中所列的就是一个`Class`按顺序排列的数据结构

类型 | 名称 | 数量
--- | --- | ---
u4 | magic 魔数 标识Class文件 | 1
u2 | minor_version 次版本号 | 1
u2 | major_version 主版本号 | 1
u2 | constant_pool_count 常量表集合数量 | 1
cp_info | constant_pool 常量表 | constant_pool_count - 1
u2 | access_flag 访问标识 | 1
u2 | this_class 类索引 | 1
u2 | super_class 父类索引 | 1
u2 | interfaces_count 接口索引数量 | 1
u2 | interfaces 接口索引| interfaces_count
u2 | fields_count 字段表集合数量 | 1
field_info | fields 字段表 | fields_count
u2 | methods_count 方法表集合数量 | 1
method_info | methods 方法表 | methods_count
u2 | attributes_count 属性表集合数量 | 1
attribute_info | attributes 属性表 | attributes_count


下面依次来解读表中每个类型

### 魔数和版本号

头4个字节称为`Class`文件的魔数，魔数的作用是标识此文件能被Java虚拟机接受的`Class`文件，其实不止`Class`文件有魔数这个概念，包括其他很多文件格式出于安全的考虑也都会有魔数这个概念，魔数都是固定不变的，如`Class`文件的魔数就是`cafebabe`

紧接着魔数之后的是版本号，第5 6个字节表示的是次版本号，第7 8个字节表示的是主版本号。版本号都是向下兼容的

### 常量池表

读懂常量池表对于阅读Class字节码非常重要，下面我们将以大篇幅分析常量池表

常量池是`Class`文件中出现的第一个表结构类型，同时也是占用`Class`文件最大空间的类型之一。由于常量池表的数量不是固定的，所以在常量池的入口有一项`u2`类型的数据，来代表常量池的数量。并且常量池比较特殊，容量计数是从1开始而不是从0开始，所以实际的常量池数量是`constant_pool_count - 1`

常量池中主要存放两大类变量: 字面量和符号引用。字面量类似常量的概念，而符号引用则引至编译原理的概念，包括三类(类和接口的全限定名，字段的名称和描述符，方法的名称和描述符)，这里要注意的是，Java在`javac`编译的时候不会进行`Class`文件的动态连接，**只有在运行时才会进行具体的`Class`文件的解析操作**

![常量池分类类型](https://upload-images.jianshu.io/upload_images/2717496-9a9644b61564b4b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 常量池表结构

上文两大类的常量池类型细分之后，到JDK1.7之后增加到了14种。之所以说常量池是最复杂的结构，就是因为这14种不同的类型都有不同的表结构，下面我们来简单看下这14种结构

每种常量类型的起始位都有一个`u1`类型的tag标识符，用于标识当前的常量类型

![截图至深入理解java虚拟机](https://upload-images.jianshu.io/upload_images/2717496-100be34bc99676c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![截图至深入理解java虚拟机](https://upload-images.jianshu.io/upload_images/2717496-16c3df5c9fded87b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 访问标志 

访问标志用于识别类或接口层面的信息，标识是否为`public`，`abstract`，`final`，`注解`，`枚举`等

### 类索引 父类索引 接口索引 

类索引和父类索引都是一个`u2`类型的数据，而接口索引则是一组`u2`类型的集合，所以接口索引入口的第一项为一个`u2`类型的计数，表示有几个接口索引

类索引和接口索引的具体值是一个`u2`的数据项，并且指向一个`CONSTANT_Class_info`常量池表类型在常量池中的偏移量

### 字段表集合

字段表用于描述接口或类中声明的变量，包括**类级变量和实际级变量，不包括在方法内部声明的局部变量**

包含的信息主要有这几种: `字段作用域(private,protact,public)`，`是否static修饰`，`可变性`，`volatile 修饰`，`可否被序列化`，`字段类型(基本类型，引用，数组)`，`字段名称`

字段表结构

类型 | 名称 | 数量 
--- | --- | --- 
u2 | access_flas | 1 
u2 | name_index | 1
u2 | descriptor_index | 1
u2 | attributes_count | 1
attribute_info | 属性表 | attributes_count

#### access_flas

我们来看下字段`access_flas`访问标识可选的类型

标志名称 | 标志值 | 含义
--- | --- | ---
ACC_PUBLIC | 0x0001 | 字段是否public 
ACC_PRIVARE | 0x0002 | 字段是否private
ACC_PROTECTED | 0x0004 | 字段是否protected
ACC_STATIC | 0x0008 | 字段是否static
ACC_FINAL | 0x0010 | 字段是否final
ACC_VOLATILE | 0x0040 | 字段是否volatile
ACC_TRANSIENT | 0x0080 | 是否 transient
ACC_SYNTHETIC | 0x1000 | 字段由编译器自动产生
ACC_ENUM | 0x4000 | 字段是否为枚举类型

看个例子，如果`access_flas`为`0x0019`，则标识了`ACC_PUBLIC`，`ACC_STATIC`，`ACC_FINAL`三种类型

#### name_index和descriptor_index

跟在`access_flas`之后是`name_index(简单名称)`和`descriptor_index(描述符)`。包括之前出现的`全限定名`，这里解释一下这几个名称。全限定名称一般来说是指`org/xxx/TestClass`这种类型的名称，可以理解为类的路径，简单名称就更加容易理解了，例如方法`inc()`的简单名称值的就是`inc`

描述符相对于上面的两种稍复杂一些，**描述符的作用是用来描述字段的数据类型，方法的参数列表和返回值**，根据描述符规则，基本数据类型（byte,char,int,long,float,double,short,boolean）以及代表无返回值的`void`类型都用一个大写字符来表示，而对象类型则用`L`加对象的全限定名来表示

具体的列在了下表中

标识字符 | 含义
--- | ---
B | byte
C | char
D | double
F | float
I | int
J | long
S | short
Z | boolean
V | void
L | 对象类型 如 Ljava/lang/Object

对于数组类型，每一纬度使用一个前置的`[`来表示，如定义一个`java.lang.String []`的数组类型，将被记录为`[[Ljava/lang/String;`

用描述符来描述方法时，按照先参数列表后返回值的顺序描述，参数列表严格按照顺序放在`()`内，如方法`void inc()`的描述符为`()V`，方法`int inc(int i, double)`的描述符为`(ID)I`

在描述符之后，紧跟着是一个属性表集合，属性表集合可以为空，

### 方法表

**方法表的组成与属性表的组成是完全一致的**，访问标识符的取值略有不同

标志名称 | 标志值 | 含义
--- | --- | ---
ACC_PUBLIC | 0x0001 | 方法是否public 
ACC_PRIVARE | 0x0002 | 方法是否private
ACC_PROTECTED | 0x0004 | 方法是否protected
ACC_STATIC | 0x0008 | 方法是否static
ACC_FINAL | 0x0010 | 方法是否final
ACC_SYNCHRONIZED | 0x0020 | 方法是否同步
ACC_BRIDGE | 0x0040 | 方法是否由编译器产生的桥接方法
ACC_VARARGS | 0x0080 | 方法是否接受不定蚕食
ACC_NATIVE | 0x0100 | 方法是否为native
ACC_ABSTRACT | 0x0400 | 方法是否为abstract
ACC_STRICTFP | 0x0800 | 方法是否为strictfp
SYNTHETIC | 0x1000 | 方法由编译器自动产生

那么这里大家可能会有疑问，方法里的java代码去哪了呢？ 答案就是在方法表的属性表集合中，有一个`code`属性，那里存放了编译成字节码的Java代码。对于属性表，在下文会提到

### 属性表

属性表，前文已经提到了多次。包括`Class`文件本身，`方法表`，`字段表`都有携带自己的属性表集合，用于描述专有场景信息

并且属性表与`Class`文件其他数据项要求不同，各个属性不要求严格的顺序，并且只要不与已有属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息

下面我们来看几个关键属性

#### code 属性

java程序方法体中的代码`javac`编译器处理后，最终变为字节码子令存储在`Code`属性内

#### Exceptions 属性

列举方法可能会抛出的异常

#### LineNumberTable 属性

描述java源码行数和字节码行数的对应关系，前者是字节码行，后者是源码行

#### LocalVariableTable 属性

描述栈局部变量和源码中定义的变量的关系.这项是可选的，可使用`javac -g:none`或`javac -g:vars`命令关闭生成这项信息

#### SourceFile 属性

用于记录Class源码文件的文件名称，这个属性是可选的。可使用`javac -g:none`或`javac -g:source`命令关闭生成这项信息
#### ConstantValue 属性

通知虚拟机为静态变量赋值

#### InnerClasses 属性

用于记录内部类与宿主类之间的关系

#### Deprecated Synthetic 属性

`Deprecated` 标识某个类，字段或方法过期
`Synthetic` 标识此字段不由Java源码直接产生，由编译器自动添加

#### StackMapTable 属性

这个属性会在虚拟机加载完字节码后的验证阶段被使用

#### Signature 属性

`Signature`在`JDK1.5`之后被添加，用于记录泛型签名信息。之所以要用这么一个属性去记录泛型信息，是因为Java语言的泛型采用的是擦除法实现的伪泛型，在`Code`属性中，泛型信息在编译之后统统都被擦除掉了。使用的擦除法的原因是这样子实现比较简单，只需要修改`javac`编译器就可以实现了，运行时也可以节省一些空间。坏处就是运行时无法拿到泛型信息。`Signature`就是为了弥补这个缺陷而设置的，现在的Java API反射能够获取到的泛型信息也来自这个属性

#### BootStrapMathods 属性

`BootStrapMathods`是JDK 1.7之后增加到规范中的，这个属性用于保存`invokedynamic`指令引用的引导方法限定符。本篇文章暂不赘述这个指令

