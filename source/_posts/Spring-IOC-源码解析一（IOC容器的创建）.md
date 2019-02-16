---
title: Spring 源码解析一（IOC容器的创建）
tags:
  - Spring
categories:
  - java框架
  - Spring
date: 2018-02-28 17:43:57
---

### 引言

> **`Spring源码`值得每一个程序员去研读，不仅仅因为她是最常用的框架，更因为她所体现的编码思想** 本文主要介绍web应用启动时`Spring`容器的创建过程

*****
### Spring 启动监听

**最终入口**

```
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

### ContextLoaderListener(org.springframework.web.context.ContextLoaderListener) 

 这里的`contextInitialized` 方法会在web容器启动时被调用，从而启动整个spring容器的创建 
``` 
@Override
public void contextInitialized(ServletContextEvent event) {
	initWebApplicationContext(event.getServletContext());
}
```

### ContextLoader(org.springframework.web.context.ContextLoader) 

主要有 `initWebApplicationContext`，`determineContextClass`，`createWebApplicationContext`，`configureAndRefreshWebApplicationContext` 这四个方法的执行

##### initWebApplicationContext 

初始化过程入口（主要是创建容器的过程）
```
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
	if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
		throw new IllegalStateException(
				"Cannot initialize context because there is already a root application context present - " +
				"check whether you have multiple ContextLoader* definitions in your web.xml!");
	}

	Log logger = LogFactory.getLog(ContextLoader.class);
	servletContext.log("Initializing Spring root WebApplicationContext");
	if (logger.isInfoEnabled()) {
		logger.info("Root WebApplicationContext: initialization started");
	}
	long startTime = System.currentTimeMillis();

	try {
		// Store context in local instance variable, to guarantee that
		// it is available on ServletContext shutdown.
		if (this.context == null) {
			this.context = createWebApplicationContext(servletContext);
		}
		if (this.context instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
			if (!cwac.isActive()) {
				// The context has not yet been refreshed -> provide services such as
				// setting the parent context, setting the application context id, etc
				if (cwac.getParent() == null) {
					// The context instance was injected without an explicit parent ->
					// determine parent for root web application context, if any.
					ApplicationContext parent = loadParentContext(servletContext);
					cwac.setParent(parent);
				}
				configureAndRefreshWebApplicationContext(cwac, servletContext);
			}
		}
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

		ClassLoader ccl = Thread.currentThread().getContextClassLoader();
		if (ccl == ContextLoader.class.getClassLoader()) {
			currentContext = this.context;
		}
		else if (ccl != null) {
			currentContextPerThread.put(ccl, this.context);
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
					WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
		}
		if (logger.isInfoEnabled()) {
			long elapsedTime = System.currentTimeMillis() - startTime;
			logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
		}

		return this.context;
	}
	catch (RuntimeException ex) {
		logger.error("Context initialization failed", ex);
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
		throw ex;
	}
	catch (Error err) {
		logger.error("Context initialization failed", err);
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
		throw err;
	}
}
```

##### createWebApplicationContext

创建容器，需要知道我们默认创建的spring容器是哪个？ 在`determineContextClass`中获取
```
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
	// 获取容器class对象
	Class<?> contextClass = determineContextClass(sc);
	if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
		throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
				"] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
	}
	return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

##### determineContextClass

先尝试从servlet上下文获取容器类名，一般我们都不会去配。所以一般都在这里`return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());`获取到实际的容器，这里实际上会去读取配置文件`ContextLoader.properties`，找到实际启动的容器类为`org.springframework.web.context.support.XmlWebApplicationContext
`
```
protected Class<?> determineContextClass(ServletContext servletContext) {
	String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
	if (contextClassName != null) {
		try {
			return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
		}
		catch (ClassNotFoundException ex) {
			throw new ApplicationContextException(
					"Failed to load custom context class [" + contextClassName + "]", ex);
		}
	}
	else {
		contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
		try {
			return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
		}
		catch (ClassNotFoundException ex) {
			throw new ApplicationContextException(
					"Failed to load default context class [" + contextClassName + "]", ex);
		}
	}
}
```

##### configureAndRefreshWebApplicationContext

看名字，顾名思义。这个方法是在容器创建完成之后，进行相关配置并初始化刷新容器。其中最重要的就是获取我们在web.xml中配置的Spring配置文件信息，然后执行 `refresh` 方法刷新容器
```
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
	if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
		// The application context id is still set to its original default value
		// -> assign a more useful id based on available information
		String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
		if (idParam != null) {
			wac.setId(idParam);
		}
		else {
			// Generate default id...
			wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
					ObjectUtils.getDisplayString(sc.getContextPath()));
		}
	}

	wac.setServletContext(sc);
	// 找到配置文件 我们在web.xml中配置的Spring配置文件位置
	String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
	if (configLocationParam != null) {
		wac.setConfigLocation(configLocationParam);
	}

	// The wac environment's #initPropertySources will be called in any case when the context
	// is refreshed; do it eagerly here to ensure servlet property sources are in place for
	// use in any post-processing or initialization that occurs below prior to #refresh
	ConfigurableEnvironment env = wac.getEnvironment();
	if (env instanceof ConfigurableWebEnvironment) {
		((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
	}

	customizeContext(sc, wac);
	wac.refresh();
}
```
*****
### 尾言

> 到这里为止，我们知道了Spring 启动的容器类是 `XmlWebApplicationContext`，在下一节我们会着重解析容器的 `refresh` 方法，这个方法是整个spring ioc容器的精髓所在
