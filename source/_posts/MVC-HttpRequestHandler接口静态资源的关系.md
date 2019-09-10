---
title: MVC-HttpRequestHandler接口静态资源的关系
tags:
  - Spring
categories:
  - Spring
date: 2019-08-30 16:10:15
---
在Spring Mvc，一般我们都会配置对静态资源的拦截，因为静态资源处理有别与普通请求，需要进行特殊配置。

比较常用的配置有这几种

1. Tomcat defaultServelt 激活tomcat自带的defaultServelt 用来处理静态资源

```xml
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.jpg</url-pattern>
</servlet-mapping>
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.js</url-pattern>
</servlet-mapping>
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.css</url-pattern>
</servlet-mapping>
```

2. 使用 `<mvc:resources/>` Spring mvc自己提供的静态资源处理器

例如做如下配置，就可以将`static`开头的路径全部访问至`static`目录

```xml
<mvc:resources mapping="/static/**" location="/static/" />
```

那么这个配置的实现原理是什么？

通过阅读源码其实可以知道实际上是 Spring mvc 自动帮我们注册了一个`SimpleUrlHandlerMapping`，在之前分析`SimpleUrlHandlerMapping`时也有提到它其实可以当成拦截器用，这里其实就是一个实例。注册了`HandlerMapping`必然要确定`handler`处理器，这里的`handler`处理器其实就是`HttpRequestHandler`接口的实现类（ResourceHttpRequestHandler）

3. 使用 `<mvc:default-servlet-handler/>` 激活Spring mvc提供的默认静态资源处理器

使用这个配置其实原理与第二个配置非常相似，也是利用了`SimpleUrlHandlerMapping`，并且`handler`处理器也是`HttpRequestHandler`接口的实现类，只是把具体的实现类换成了`DefaultServletHttpRequestHandler`，相比第二种配置，此种配置无法自定义。

- 写在后面

不过我们要是使用Spring boot开发，也就省去了这一堆繁琐的配置，包括静态资源的配置也都是自动完成配置。只要按照约定将静态资源放置在classpath下的这几个目录下（`public`,`resource`,`static`），就不需要任何配置，不得不说，约定优于配置还是可以极大的提升开发效率的。

