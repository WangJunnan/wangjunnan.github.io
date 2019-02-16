---
title: Maven 学习总结
tags:
  - maven
categories:
  - 构建工具
  - maven
date: 2016-12-20 16:16:16
---

#Maven 简介
Maven是什么？

>Maven 这个词可以翻译为“知识的积累”，也可以翻译为“专家”或“内涵”。本文将介绍maven这一跨平台的项目管理使用工具。做为apache组织中一个颇为成功的开源项目，maven主要服务于基于java平台的项目构建，依赖管理和项目信息管理，无论是小型的开源类库项目，还是大型的企业级应用；无论是传统的瀑布式开发，还是流行的敏捷模式，Maven都能大显身手
                                                                                              -- maven实战

#何为构建
除了编写源代码，我们每天有相当一部分时间花在了编译、运行单元测试、生成文档、打包和部署等烦琐且不起眼的工作上，这就是构建
如果我们现在还手工这样做，那成本也太高了，于是有人用软件的方法让这一系列工作完全自动化，使得软件的构建可以像全自动流水线一样，只需要一条简单的命令，所有烦琐的步骤都能够自动完成，很快就能得到最终结果

#maven不仅仅是构建工具
依赖管理

#与ant的区别
我看到很多项目的ant脚本中的命名基本上都是一致的，比如：编译一般叫build或者compile；打包一般叫jar或war；生成文档一般命名为 javadoc或javadocs；但其实最常用的就2，3个：比如javac javadoc jar等。
>这是因为Ant所提供的可重用的****task****粒度太小，虽然灵活性很强，但是我们需要纠缠很多细节的东西
ANT是没有依赖管理的，而对于Maven用户来说，依赖管理是理所当然的，Maven不仅内置了依赖管理，更有一个可能拥有全世界最多Java开源软件包的中央仓库，Maven用户无须进行任何配置就可以直接享用。

##Maven的优点：
>1.拥有约定，知道你的代码在哪里，放到哪里去
2.拥有一个生命周期，例如执行 mvn install就可以自动执行编译，测试，打包等构建过程
3.只需要定义一个pom.xml,然后把源码放到默认的目录，Maven帮你处理其他事情
4.拥有依赖管理，仓库管理

#Maven 的安装配置

首先官网下载 http://maven.apache.org/download.cgi
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-bc300fe55da729c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

说明需要JDK 1.7及以上，这里我们使用JDK1.8

这里我下载的是maven 3.3.9

1.将安装文件解压到指定的目录中，如：
>D:\maven\apache-maven-3.3.9

2.设置环境变量

>将D:\maven\apache-maven-3.3.9\bin 设进环境变量中
 
打开cmd 执行 
>mvn -v

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-79c6544b870e30f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**注意事项**

设置MAVEN_OPTS环境变量，通常需要设置MAVEN_OPTS的值为-Xms128m-Xmx512m，因为Java默认的最大可用内存往往不能够满足Maven运行的需要，比如在项目较大时，使用Maven生成项目站点需要占用大量的内存，如果没有该配置，则很容易得到java.lang.OutOfMemeoryError。因此，一开始就配置该变量是推荐的做法。

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-65e21d083093bafc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#maven使用入门
一、编写pom文件
就像Make的Makefile、Ant的build.xml一样，Maven项目的核心是pom.xml。POM（Project Object Model，项目对象模型）定义了项目的基本信息，用于描述项目如何构建，声明项目依赖，等等。现在先为Hello World项目编写一个最简单的pom.xml。

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.pikaq</groupId>
  <artifactId>maventest</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>
  <name>maventest</name>
</project>
```

这段代码中最重要的是包含groupId、artifactId和version的三行。这三个元素定义了一个项目基本的坐标，在Maven的世界，任何的jar、pom或者war都是以基于这些基本的坐标进行区分的。
groupId定义了项目属于哪个组，这个组往往和项目所在的组织或公司存在关联。譬如在googlecode上建立了一个名为myapp的项目，那么groupId就应该是com.googlecode.myapp，如果你的公司是mycom，有一个项目为myapp，那么groupId就应该是com.mycom.myapp。本书中所有的代码都基于groupId com.juvenxu.mvnbook。
本书其他章节代码会分配其他的artifactId。而在前面的groupId为com.googlecode.myapp的例子中，你可能会为不同的子项目（模块）分配artifactId，如myapp-util、myapp-domain、myapp-web等
顾名思义，version指定了Hello World项目当前的版本——1.0-SNAPSHOT。SNAPSHOT意为快照，说明该项目还处于开发中，是不稳定的版本。随着项目的发展，version会不断更新，如升级为1.0、1.1-SNAPSHOT、1.1、2.0等。6.5节会详细介绍SNAPSHOT，第13章会介绍如何使用Maven管理项目版本的升级发布。
最后一个name元素声明了一个对于用户更为友好的项目名称，虽然这不是必须的，但还是推荐为每个POM声明name，以方便信息交流。

##编写主代码

```
package com.pikaq.maventest;

public class Helloworld{
	public String sayHello(){
		return "hello";
	}
	
	public static void main(String [] args){
		System.out.println(new Helloworld().sayHello());
	}
	
}
```

这里需要注意，在大多数情况下，我们应该遵循maven约定把代码放在src/main/java 目录下，无需其他配置，maven就会自动去搜寻主代码，代码编写完之后，使用
>mvn clean compile 编译



![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-b5c57cbc20a036fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##编写测试代码

和编写主代码类似，测试代码的目录对应的应该在 src/test/java 下，因此应该先创建目录，
修改项目 pom 文件，增加Junit依赖

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.pikaq</groupId>
  <artifactId>maventest</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>

  <name>maventest</name>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.7</version>
      <scope>test</scope>
    </dependency>
 </dependencies>
  
</project>
```
编写测试类

```
package com.pikaq.maventest;
import static org.junit.Assert.assertEquals;
import org.junit.Test;

public class HelloworldTest{
	
	@Test
	public void testSayHello(){
		Helloworld h = new Helloworld();
		String result = h.sayHello();
		assertEquals("hello",result);
	}
	
}
```

>运行 mvn clean test

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-54e6daa34225e09b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#打包和运行

>mvn clean package


![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-bba6247b3094451b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

maven 在打包之前执行了编译和测试操作

**但是如何让其他maven 项目引用这个jar包呢：**
>mvn clean install

通过执行install 命令将jar 安装到了本地仓库中


#使用Archetype 生成项目骨架

前面我们只是为了我们进一步熟悉maven，真正用的时候我们当然不会这么复杂，一个一个目录的去创建，这显然是重复劳动！我们只需要一个简单的命令就能生成我们约定好的项目骨架！

>mvn archetype:generate

注意这里极有可能会卡主，因为这个命令会默认去远程请求所有的骨架

所以我们这里可以选择 >mvn archetype:generate -DarchetypeCatalog=internal 

这个命令执行只会列出常用的10个
我们只需要按照提示选择一个即可，并依次输入groupID等

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-0dee0e2cfc2ad134.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

提示创建成功，我们发现已经生成了一个maven项目，是不是非常简单！

####maven 的依赖管理问题

![image.png](http://upload-images.jianshu.io/upload_images/2717496-103204113f831f86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

maven会自动选择路径最短的
