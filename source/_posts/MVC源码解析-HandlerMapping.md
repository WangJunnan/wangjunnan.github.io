---
title: MVC源码解析-HandlerMapping
tags:
  - Spring
categories:
  - Spring
date: 2019-08-20 16:07:09
---
## 引言

**什么是Spring mvc 的 HanderMapping? **
**
我们知道当一个http请求到服务器时，我们需要根据请求的上下文url来匹配到对应的处理类，而HanderMapping做的就是这个事情，它定义了我们的请求和对应的处理类的映射关系。

Spring mvc提供了多种HanderMapping类型，比较常用的有 `RequestMappingHandlerMapping`,`BeanNameUrlHandlerMapping`,`SimpleUrlHandlerMapping`

以下是这三个HanderMapping的继承结构：
![15680101158745.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/149106/1568030225719-c9a755b8-c582-4c6d-8373-b7b078b6ad0b.jpeg#align=left&display=inline&height=528&name=15680101158745.jpg&originHeight=528&originWidth=689&size=67913&status=done&width=689)

![15680196478301.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/149106/1568030238662-c338cb95-bf09-4657-bbeb-8dd96897e59f.jpeg#align=left&display=inline&height=624&name=15680196478301.jpg&originHeight=624&originWidth=672&size=70420&status=done&width=672)

![15680196756855.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/149106/1568030249196-ed212f57-bff0-47ba-8dd6-f8fc907b3d33.jpeg#align=left&display=inline&height=571&name=15680196756855.jpg&originHeight=571&originWidth=634&size=59515&status=done&width=634)

先来看一下这几个HandlerMapping的配置入口，如果是普通Spring Mvc应用的话，当我们在配置文件中配置了`annotation-driven`后，便会自动帮我们注入这几个类。具体的源码入口在`AnnotationDrivenBeanDefinitionParser`中，这里的逻辑非常简单，就是帮我们自动注入这几个常用类的`BeanDefinition`元数据（包括HandlerMapping，handlerAdapter，ExceptionResolver等等）

下面通过阅读源码依次分别看一下这几个HanderMapping的作用及实现原理

## RequestMappingHandlerMapping

我们先来看下 RequestMappingHandlerMapping 的用法

```
@Controller
public class TestAPI {
    @RequestMapping(value = "/api/sayHello")
    public String hello() {
        return "OK";
    }
    
    @RequestMapping(value = "/api/sayHello2")
    public String hello2() {
        return "OK";
    }
}
```

如上面的代码，这是一个我们非常常用的用法，一个`Controller`定义多个处理方法，并且在方法上注解RequestMapping，如此做之后就可以实现一个`Controller`根据不同的路径分别可以处理多个请求。下面我们来看下其实现原理。
我比较好奇的是，`RequestMappingHandlerMapping`是如何将请求路径与处理方法进行初始化映射的？

通过观察`RequestMappingHandlerMapping`的继承体系，可以知道其实现了`InitializingBean`接口

```java
@Override
public void afterPropertiesSet() {
	initHandlerMethods();
}

/**
 * 初始化 HandlerMethods
 */
protected void initHandlerMethods() {
	if (logger.isDebugEnabled()) {
		logger.debug("Looking for request mappings in application context: " + getApplicationContext());
	}
	// 获取容器中所有的Bean
	String[] beanNames = (this.detectHandlerMethodsInAncestorContexts ?
			BeanFactoryUtils.beanNamesForTypeIncludingAncestors(obtainApplicationContext(), Object.class) :
			obtainApplicationContext().getBeanNamesForType(Object.class));

	for (String beanName : beanNames) {
		if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
			Class<?> beanType = null;
			try {
				beanType = obtainApplicationContext().getType(beanName);
			}
			catch (Throwable ex) {
				// An unresolvable bean type, probably from a lazy bean - let's ignore it.
				if (logger.isDebugEnabled()) {
					logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
				}
			}
			// 判断该Bean是否被 @Controller 或则 @RequestMapping 所注解
			if (beanType != null && isHandler(beanType)) {
				// 初始化 HandlerMethods
				detectHandlerMethods(beanName);
			}
		}
	}
	handlerMethodsInitialized(getHandlerMethods());
}
```

上面的代码主要做了3件事情

1. 获取容器中所有的Bean
1. 筛选出被[@Controller ]() 或则 [@RequestMapping ]() 所注解的Bean
1. 根据筛选出的Bean初始化HandlerMethod

可以看到这里有一个很重要的类叫做HandlerMethod，我们来看看这个类的结构

```java
/** 目标Bean **/
private final Object bean;

@Nullable
private final BeanFactory beanFactory;

private final Class<?> beanType;
	
/** 对应的目标方法 **/
private final Method method;

private final Method bridgedMethod;
/** 方法参数类型 **/
private final MethodParameter[] parameters;

@Nullable
private HttpStatus responseStatus;

@Nullable
private String responseStatusReason;

@Nullable
private HandlerMethod resolvedFromHandlerMethod;

... 还提供了很多方法，像获取方法返回值等等
```

对于这个类我们现在可以不用深入去理解，只需知道它封装了处理类的目标方法

```java
protected void detectHandlerMethods(Object handler) {
	Class<?> handlerType = (handler instanceof String ?
			obtainApplicationContext().getType((String) handler) : handler.getClass());

	if (handlerType != null) {
		Class<?> userType = ClassUtils.getUserClass(handlerType);
		// 获取目标Bean中被RequestMapping注解的方法
		Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
				(MethodIntrospector.MetadataLookup<T>) method -> {
					try {
						return getMappingForMethod(method, userType);
					}
					catch (Throwable ex) {
						throw new IllegalStateException("Invalid mapping on handler class [" +
								userType.getName() + "]: " + method, ex);
					}
				});
		if (logger.isDebugEnabled()) {
			logger.debug(methods.size() + " request handler methods found on " + userType + ": " + methods);
		}
		// 注册 HandlerMethod
		methods.forEach((method, mapping) -> {
			Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
			registerHandlerMethod(handler, invocableMethod, mapping);
		});
	}
}
```

上面的代码主要做了两件事情

1. 获取目前Bean中被RequestMapping注解的方法
1. 把获取到的Method 封装成HandlerMethod并完成注册

到这里为止`RequestMappingHandlerMapping`就完成初始化了，下面再来看下当一个请求来时其实如何处理的，并选择正确的 HandlerMethod 处理的.

**当一个请求进来时，DispatcherServlet会分别调用每个HandlerMapping的getHandler的方法，直到有个HandlerMapping返回了一个hander**

`getHandler方法`在抽象父类`AbstractHandlerMapping.class`中

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	// 具体调用逻辑，由子类实现
	Object handler = getHandlerInternal(request);
	if (handler == null) {
		handler = getDefaultHandler();
	}
	if (handler == null) {
		return null;
	}
	if (handler instanceof String) {
		String handlerName = (String) handler;
		handler = obtainApplicationContext().getBean(handlerName);
	}

	HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
	if (CorsUtils.isCorsRequest(request)) {
		CorsConfiguration globalConfig = this.globalCorsConfigSource.getCorsConfiguration(request);
		CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
		CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
		executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
	}
	return executionChain;
}
```

看`RequestMappingHandlerMapping`的`getHandlerInternal`方法

```java
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
	// 根据 request获取到 path
	// 如请求是 localhost:8080/api/sayHello
	// 则这里返回  /api/sayHello
	String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
	if (logger.isDebugEnabled()) {
		logger.debug("Looking up handler method for path " + lookupPath);
	}
	this.mappingRegistry.acquireReadLock();
	try {
		// 获取到请求对应的 handlerMethod 
		HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
		if (logger.isDebugEnabled()) {
			if (handlerMethod != null) {
				logger.debug("Returning handler method [" + handlerMethod + "]");
			}
			else {
				logger.debug("Did not find handler method for [" + lookupPath + "]");
			}
		}
		return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
	}
	finally {
		this.mappingRegistry.releaseReadLock();
	}
}
```

以上就是`RequestMappingHandlerMapping`在整个MVC流程中所起的作用

## BeanNameUrlHandlerMapping

下面我们再简单看下`BeanNameUrlHandlerMapping`的使用和原理
顾名思义，`BeanNameUrlHandlerMapping`就是指通过BeanName和Url进行关联匹配，跟`RequestMappingHandlerMapping`不一样，`BeanNameUrlHandlerMapping`一个Bean就对应处理一个请求.
先来看个使用的例子

```
@Component("/test2API")
public class Test2API implements Controller {

    @Override
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        System.out.println("test2API");
        return null;
    }
}
```

如此定义一个类，我们便能通过`/test2API`这个路径来访问这个类。
如何实现的？我们还是打开`BeanNameUrlHandlerMapping`看看其实现，与`RequestMappingHandlerMapping`不同的是，它没有实现`InitializingBean`接口，不过它实现了`ApplicationContextAware`接口，在Spring容器完成初始化的时候回调用这个唤醒接口，也就是在这里进行了`BeanNameUrlHandlerMapping`的初始化。

初始化逻辑在 AbstractDetectingUrlHandlerMapping.detectHandlers() 方法中

```
protected void detectHandlers() throws BeansException {
	ApplicationContext applicationContext = obtainApplicationContext();
	if (logger.isDebugEnabled()) {
		logger.debug("Looking for URL mappings in application context: " + applicationContext);
	}
	String[] beanNames = (this.detectHandlersInAncestorContexts ?
			BeanFactoryUtils.beanNamesForTypeIncludingAncestors(applicationContext, Object.class) :
			applicationContext.getBeanNamesForType(Object.class));

	// Take any bean name that we can determine URLs for.
	for (String beanName : beanNames) {
		// 获取BeanName或别名是 "/" 开头的 Bean名称
		String[] urls = determineUrlsForHandler(beanName);
		if (!ObjectUtils.isEmpty(urls)) {
			// URL paths found: Let's consider it a handler.
			// 把获取到的 urls 注册到 handler
			registerHandler(urls, beanName);
		}
		else {
			if (logger.isDebugEnabled()) {
				logger.debug("Rejected bean name '" + beanName + "': no URL paths identified");
			}
		}
	}
}
```

`determineUrlsForHandler`是`BeanNameUrlHandlerMapping`类仅有实现的一个方法，主要逻辑就是获取BeanName或别名是 "/" 开头的 Bean名称。所以这里也可以知道在上文的使用例子中，BeanName我为何要以斜杠开头。

当一个请求来时`BeanNameUrlHandlerMapping`的处理逻辑类似，只不过匹配的容器换成了`handlerMap = new LinkedHashMap<>();`，匹配逻辑更加的简单直接。

## SimpleUrlHandlerMapping

最后再看下 SimpleUrlHandlerMapping的用法及实现

定义一个`SimpleUrlHandlerMapping`

```java
@Configuration
public class SampleConfig {

    @Bean
    public SimpleUrlHandlerMapping simpleUrlHandlerMapping() {
        SimpleUrlHandlerMapping simpleUrlHandlerMapping = new SimpleUrlHandlerMapping();
        Map<String, Object> urlMap = new HashMap<>();
        urlMap.put("/**", new Test2API());
        // ... 可配置多个映射关系
        simpleUrlHandlerMapping.setUrlMap(urlMap);
        simpleUrlHandlerMapping.setOrder(1);
        return simpleUrlHandlerMapping;
    }
}
```

以上配置会拦截所有的请求，然后转交给 Test2API处理（Test2API可以是Controller接口实现类，也可以是一个 HttpRequestHandler接口实现类），可以看到这里`SimpleUrlHandlerMapping`起的作用非常像一个拦截器起的作用，其实也确实是这样，像对静态资源的拦截就可以采用`SimpleUrlHandlerMapping`来实现。

具体实现其实跟上文的两种mapping实现类似，简单看下源码

初始注册逻辑

```java
@Override
public void initApplicationContext() throws BeansException {
	super.initApplicationContext();
	registerHandlers(this.urlMap);
}
protected void registerHandlers(Map<String, Object> urlMap) throws BeansException {
	if (urlMap.isEmpty()) {
		logger.warn("Neither 'urlMap' nor 'mappings' set on SimpleUrlHandlerMapping");
	}
	else {
		urlMap.forEach((url, handler) -> {
			// 确保路径是以斜杠开头的
			if (!url.startsWith("/")) {
				url = "/" + url;
			}
			// 去除空格
			if (handler instanceof String) {
				handler = ((String) handler).trim();
			}
			// 注册
			registerHandler(url, handler);
		});
	}
}
```

逻辑还是比较简单的，就是将我们手动配置的多个映射关系进行初始注册，以备后面逻辑使用。
## 总结

总的来说HanderMapping的职责是非常清晰的，就是存储并匹配请求url和处理类的映射关系。整个处理流程的源码相对来说也比较直观，易阅读。读完其源码对MVC的url匹配流程会有更加深刻的认识

