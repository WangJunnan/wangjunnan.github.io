---
title: MVC源码解析-HandlerAdapter
tags:
  - Spring
categories:
  - Spring
date: 2019-08-25 16:08:54
---
## 引言

上篇文章分析了`handerMapping`的实现。本篇文章会继续分析和`handerMapping`相辅相成的`handerAdapter`。

**什么是`handerAdapter`？**
在理解了`handerMapping`的基础上，我们应该发现了一个问题。就是每个`handerMapping`所匹配的到的hander实现是不一样的。比如`RequestMappingHandlerMapping`的hander的实现是一个`HandlerMethod`实现，而`BeanNameUrlHandlerMapping`的hander则是一个`Controller`接口实现类。所以如果我们要是不引入`handerAdapter`的话，就无法统一的去执行各自的hander

上文所说`handerAdapter`和`handerMapping`是相辅相成的。以下列出了几个的对应关系

- `RequestMappingHandlerMapping` -> `RequestMappingHandlerAdapter`
- `BeanNameUrlHandlerMapping` -> `SimpleControllerHandlerAdapter`
- `SimpleUrlHandlerMapping` -> `SimpleControllerHandlerAdapter`
- `SimpleUrlHandlerMapping` -> `HttpRequestHandlerAdapter`

其中`SimpleUrlHandlerMapping`比较特殊，可以对应多个`Adapter`，主要原因是`SimpleUrlHandlerMapping`中的`handler`类型也可以有多种，比较常见的有`Controller`接口实现类，`HttpRequestHandler`接口实现类。

## HandlerAdapter接口

先看下HandlerAdapter接口的几个方法

```java
public interface HandlerAdapter {

	/***
	 * 判断指定hander是否可以被当前Adapter支持
	 * 
	 * @param handler handler object to check
	 * @return whether or not this object can use the given handler
	 */
	boolean supports(Object handler);

	/**
	 * 执行hander方法
	 * returned {@code true}.
	 * @throws Exception in case of errors
	 * @return ModelAndView object with the name of the view and the required
	 * model data, or {@code null} if the request has been handled directly
	 */
	@Nullable
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;

	/**
	 * 与HttpServlet的{@code getLastModified}方法的约定相同
	 * @param request current HTTP request
	 * @param handler handler to use
	 * @return the lastModified value for the given handler
	 * @see javax.servlet.http.HttpServlet#getLastModified
	 * @see org.springframework.web.servlet.mvc.LastModified#getLastModified
	 */
	long getLastModified(HttpServletRequest request, Object handler);
}
```

核心是`supports`和`hander`方法

1. `supports`用于判断指定hander是否可以被当前Adapter支持
1. `hander`方法用于执行指定的hander的核心逻辑

下面简单来看下各个HandlerAdapter接口的实现类，看一下对这两个核心方法的实现

## RequestMappingHandlerAdapter

- supports

```java
public final boolean supports(Object handler) {
	return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
}
```

方法比较简单，判断`handler`的类型是不是`HandlerMethod`，如果是的话，则说明该handler可以被该adapter所适配，那么就会直接返回该Adapter。

- hander

```
protected ModelAndView handleInternal(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
	
	// 定义 ModelAndView
	ModelAndView mav;
	checkRequest(request);

	// 处于一个session时判断是否需要同步
	if (this.synchronizeOnSession) {
		// 获取当前session
		HttpSession session = request.getSession(false);
		if (session != null) {
			Object mutex = WebUtils.getSessionMutex(session);
			synchronized (mutex) {
			   // 执行 handerMethod 实际逻辑
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No HttpSession available -> no mutex necessary
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}
	}
	else {
		// 
		mav = invokeHandlerMethod(request, response, handlerMethod);
	}

	if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
		if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
			applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
		}
		else {
			prepareResponse(response);
		}
	}

	return mav;
}
```

## SimpleControllerHandlerAdapter

- supports

```java
public boolean supports(Object handler) {
	return (handler instanceof Controller);
}
```

判断是否是handler的类型是否是Controller

- handler

```java
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
		throws Exception {

	return ((Controller) handler).handleRequest(request, response);
}
```

是的话，则直接执行Controller的handleRequest方法

## HttpRequestHandlerAdapter

- supports

```java
public boolean supports(Object handler) {
	return (handler instanceof HttpRequestHandler);
}
```

- handler

```java
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
		throws Exception {

	((HttpRequestHandler) handler).handleRequest(request, response);
	return null;
}
```

实现逻辑与`SimpleControllerHandlerAdapter`类似

对于以上几个Adapter的匹配顺序，Spring会根据各个Adapter的Order值来进行排序，依次匹配直到找到合适的为止。

## 总结

`HandlerAdapter`的核心逻辑就是承接不同的`HandlerMapping`做适配。本身的逻辑并不复杂，其引入的适配中间层设计模式是一个非常好的案例。

