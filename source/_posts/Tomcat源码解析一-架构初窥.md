---
title: Tomcat源码解析一 架构初窥
tags:
  - Tomcat
categories:
  - web容器
  - Tomcat
date: 2019-01-20 16:39:01
---

## 引言

`Tomcat`是一个非常复杂的`Servlet`容器，也是在日常工作中与我们接触非常多的`Http服务器`，作为一个成熟的软件，它的整体设计，代码结构都十分的优秀，作为开发者，非常有必要研读`Tomcat`源码。阅读源码总是开头难，但是一旦对整体设计理念有一定了解后，再进行深入分析，就会有不一样的收获。


## 整体架构

先上一张总体架构图

![image.png](http://upload-images.jianshu.io/upload_images/2717496-68dd25ac0647213d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这张图可以说每篇介绍`Tomcat`的博客都是必放，既然每个人都认可这张图，说明此图非常重要。接下来我们就围绕这张图对`Tomcat`的整体架构进行分析。

* Server
服务器：可以看成代表Tomcat服务器本身，一个Tomcat实例只会有一个Server

* Service
查看上图，可知一个`Server`可以包含多个`Service`，在这里可以把`Service`看成是`Connector`和`Container`组合层，`Connector`和`Container`是Tomcat两个主要的模块

	* Connector 连接器，一个`Service`可以有多个`Connector`（http AJP 等），监听用户请求端口
	* Container 按照层级有`Engine`，`Host`，`Context`，`Wrapper`四种，一个`Service`只有一个`Engine`，其主要作用是执行业务逻辑

这里我再贴一个Tomcat的配置文件，可以根据此配置文件再回过头去看上图

```
<Server port="8005" shutdown="SHUTDOWN">
  <Service name="Catalina">

    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />

    <Engine name="Catalina" defaultHost="localhost">
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
      
    </Engine>
    
  </Service>
```
配置文件的第一个节点就是Server，并且配置了`shutdown`端口为`8005`，然后就配置了一个名为`Catalina`的`Service`，这里其实可以配置多个`Service`。再看`Service`节点中，配置了多个`Connector`连接器，并且配置了一个`Engine`，`Engine`对应`Container`的最顶层，一个`Service`只有一个`Engine`。

下面我们再进行逐个分析
## Server

`Server`是Tomcat最顶层的容器，一个`Tomcat`实例只会有一个`Server`。一个`Server`至少需要包含一个`Service`，在配置文件中的体现就是至少需要包含一个`Service`节点。这里我们可以看下Tomcat对于`Server`的标准实现类`org.apache.catalina.core.StandardServer`，其中`Lifecycle`这个接口是`Tomcat`对于生命周期维护的重要接口，每个组件都必须实现，从而保证了统一启动/统一关闭的效果

![image.png](http://upload-images.jianshu.io/upload_images/2717496-eb27865ccfb2bce0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## Service

前文说过，`Service`就是用于组装`Connector`和`Container`的对应关系的。通过查看配置文件，可知`Service`节点包含了`Connector`和`Container`，其中`Connector`监听用户请求，并解析请求参数，`Container`执行具体业务逻辑

## Connector

`Connector`是连接器，用于接受请求并将请求封装成`Request`和`Response`，然后交给`Container`进行处理，`Container`处理完之后再交给`Connector`返回给客户端

## Container

`Tomcat`中的`Servlet`容器必须实现`Container`接口，也就是说`Servlet容器`都是`Container`接口的实例，接下来将会介绍四种类型的容器

* Engine: 表示整个`Servlet`引擎
* Host: 表示包含一个或多个`Context`容器的虚拟主机
* Contetx: 表示一个Web应用程序。一份Context可以有多个Wrapper
* Wrapper: 表示一个独立的servlet

上述的每一个概念都是实现了`org.apache.catalina.Container`接口的，这四个实现的标准类分别对应`StandardEngine`,`StandardHost`,`StandardContetx`,`StandardWrapper`。我们可以通过下图理清它们的层次关系。一般情况下，一个容器可以有0个或多个低层次的子容器

![image.png](http://upload-images.jianshu.io/upload_images/2717496-6bf0f595524707ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### Engine

`Engine`容器表示`Tomcat`的整个`servlet`引擎。如果使用了`Engine`容器，那么它总是处于容器层级的最顶层。添加到Engine容器中的子容器通常是`Host`或`Context`的实现。默认情况下，`tomcat`会使用`Engine`容器，并且有一个`Host`容器作为其子容器。

### Host

`Host`表示虚拟主机，如果你想在同一个`Tomcat`部署上运行多个`Context`容器的话，你就需要使用`Host`容器。理论上，当你只有一个`Context实例`时，不需要使用`Host`实例。但是在`Tomcat`的实际部署中，总是会使用一个`Host`容器

### Context

对应一个独立的`Web应用程序`，也就是对应我们日常编写的单独Web应用

`Context`容器的父容器通常是Host容器，也有可能是其他实现，或则如果不必要，就可以不使用父容器

### Wrapper

`Wrapper`封装了一个具体的`Servlet`


## 尾言

本文主要分析解读了`tomcat`的总体设计架构，`tomcat`的源码非常非常的多，如果我们不对其整体架构层次进行分析，直接进行源码阅读，会感觉仿若置身大海，非常的浪费时间。所以若大家有想法阅读`tomcat`源码，还是有必要先理清其架构体系


