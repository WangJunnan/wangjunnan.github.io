---
title: Spring IOC 源码解析四（加载非延迟单例前的操作）
tags:
  - Spring
categories:
  - java框架
  - Spring
date: 2018-11-20 15:48:35
---

## 引言

> 上一期讲述了`BeanDefinitions`的载入和注册，`beanDefinition`相当于spring对`bean`的数据抽象定义，这一期我们继续顺着的 `refresh` 方法往下讲


## 回到`refresh`方法

上一期其实主要是介绍了`obtainFreshBeanFactory()` 这一步，这一期我们继续接着往下看

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		prepareRefresh();

		// 创建BeanFactory,并载入BeanDefinitions（重要）
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
			postProcessBeanFactory(beanFactory);

			// 执行 BeanFactoryPostProcessors（实例化bean前修改bean定义信息）
			invokeBeanFactoryPostProcessors(beanFactory);

			// 注册 BeanPostProcessors（初始化bean前后执行）
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			// 国际化相关
			initMessageSource();

			// Initialize event multicaster for this context.
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			// 初始化主题信息
			onRefresh();

			// Check for listener beans and register them.
			registerListeners();

			// 提前实例化化全部非延迟加载的单例类型
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```

## prepareBeanFactory(beanFactory)

先看一下`prepareBeanFactory`的方法实现

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	// 配置容器的 classLoader
	beanFactory.setBeanClassLoader(getClassLoader());
	// 这是一个表达是语言处理器
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
	// 属性编辑器
	beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

	// 配置了BeanPostProcessor的一个实现类
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
	// 配置被忽略的依赖项（各种 Aware 实现类）
	beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

	// 修正依赖
	beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);

	// Register early post-processor for detecting inner beans as ApplicationListeners.
	// 注册AOP相关的
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

	// Detect a LoadTimeWeaver and prepare for weaving, if found.
	if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		// Set a temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}

	// 配置Spring 默认环境变量bean
	if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
	}
}

```

这个方法做了什么事情呢，我们简单的来看一下

* 首先是配置了容器的默认 `classLoader`
* 配置了一个语言器和一个属性编辑器
* 注册了一个`BeanPostProcessor`接口的一个实现类，这个接口的具体用处下面马上会讲到
* 配置要自动注入时要被忽略的bean类型，可以看到这里配置的都是一些`Aware`实现类，这些依赖项spring容器会统一设置
* 注册`aop`相关的
* 配置`Spring`默认环境变量`bean`(systemProperties, systemEnvironment, environment)

## postProcessBeanFactory(beanFactory)

方法比较简短，通过查看代码，可知也是对`beanFactory`做一些配置的工作

```java
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext, this.servletConfig));
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    beanFactory.ignoreDependencyInterface(ServletConfigAware.class);
    WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
    WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext, this.servletConfig);
}
```

## invokeBeanFactoryPostProcessors(beanFactory)

可以看到主要逻辑在，`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors`中，这里其实是`spring`留的一个扩展点，我们可以通过实现`BeanFactoryPostProcessor`来实现在 `getBean`之前的自定义扩展。现在我们就来看看它是在什么时候执行的

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
	PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

	// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
	// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
	if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}
}
```

继续查看`PostProcessorRegistrationDelegate.class`

```java
public static void invokeBeanFactoryPostProcessors(
		ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

	// 存放已经执行的 BeanFactoryPostProcessor bean名字
	Set<String> processedBeans = new HashSet<String>();

	// 判断beanFactory类型是否为BeanDefinitionRegistry，只有BeanDefinitionRegistry类型的才支持BeanDefinitionRegistryPostProcessor的注册
	if (beanFactory instanceof BeanDefinitionRegistry) {
		BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
		// 存放手动加入的 BeanFactoryPostProcessor
		List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
		// 存放手动加入的 BeanDefinitionRegistryPostProcessor
		List<BeanDefinitionRegistryPostProcessor> registryProcessors = new LinkedList<BeanDefinitionRegistryPostProcessor>();

		for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
		// 如果是BeanDefinitionRegistryPostProcessor类型
			if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
				BeanDefinitionRegistryPostProcessor registryProcessor =
						(BeanDefinitionRegistryPostProcessor) postProcessor;
				// 执行 postProcessBeanDefinitionRegistry 方法
				registryProcessor.postProcessBeanDefinitionRegistry(registry);
				// 放入待执行容器中，之后会执行BeanFactoryPostProcessor接口的postProcessBeanFactory方法
				registryProcessors.add(registryProcessor);
			}
			else {
				// 放入待执行容器中，之后统一执行 
				regularPostProcessors.add(postProcessor);
			}
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		// Separate between BeanDefinitionRegistryPostProcessors that implement
		// PriorityOrdered, Ordered, and the rest.
		// 用于存放当前需要执行的 BeanDefinitionRegistryPostProcessor list
		List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();

		// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
		// 一 先执行类型为 PriorityOrdered 的BeanDefinitionRegistryPostProcessors
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}
		// 排序
		sortPostProcessors(currentRegistryProcessors, beanFactory);
		// 将当前需要执行全部放入 待执行容器
		registryProcessors.addAll(currentRegistryProcessors);
		// 执行
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
		// 清空已经执行过的
		currentRegistryProcessors.clear();

		// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
		// 二 在执行类型为Ordered的BeanDefinitionRegistryPostProcessor
		postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
			if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}
		sortPostProcessors(currentRegistryProcessors, beanFactory);
		registryProcessors.addAll(currentRegistryProcessors);
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
		currentRegistryProcessors.clear();

		// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
		// 最后执行普通的 BeanDefinitionRegistryPostProcessors
		boolean reiterate = true;
		// 循环的目的是未了防止执行过程中有新的 postProcessorNames 加入
		while (reiterate) {
			reiterate = false;
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
					reiterate = true;
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();
		}

		// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
		// 执行待执行容器的中的 BeanFactoryPostProcessor 
		invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
	}

	else {
		// Invoke factory processors registered with the context instance.
		// 直接执行手动加入的 BeanFactoryPostProcessor
		invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
	}


	// 以上执行完BeanDefinitionRegistryPostProcessor，现在开始执行BeanFactoryPostProcessor
	String[] postProcessorNames =
			beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

	// 执行顺序和BeanDefinitionRegistryPostProcessor基本一致，先PriorityOrdered，再Ordered，最后普通的
	List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
	// 存放 PriorityOrdered类型的BeanFactoryPostProcessor
	List<String> orderedPostProcessorNames = new ArrayList<String>();
	// 存放 Ordered类型的BeanFactoryPostProcessor
	List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
	for (String ppName : postProcessorNames) {
		if (processedBeans.contains(ppName)) {
			// 这里跳过的是已经执行过的 BeanDefinitionRegistryPostProcessor
			// skip - already processed in first phase above
		}
		else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
	// 先执行 PriorityOrdered
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

	// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
	// 再执行 Ordered
	List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
	for (String postProcessorName : orderedPostProcessorNames) {
		orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

	// Finally, invoke all other BeanFactoryPostProcessors.
	// 最后执行普通的
	List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
	for (String postProcessorName : nonOrderedPostProcessorNames) {
		nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

	// Clear cached merged bean definitions since the post-processors might have
	// modified the original metadata, e.g. replacing placeholders in values...
	beanFactory.clearMetadataCache();
}
```
这个方法代码非常的多，夹杂着各种判断，看的眼花缭乱。我们简单理一下逻辑

1. `BeanFactoryPostProcessor`有一个子接口叫做`BeanDefinitionRegistryPostProcessor`，它的优先级是比`BeanFactoryPostProcessor`高的
2. 方法的入参里有一个`List<BeanFactoryPostProcessor> beanFactoryPostProcessors`，这个是容器通过`addBeanFactoryPostProcessor`方法手动加入的。这里的`BeanDefinitionRegistryPostProcessor`类型是优先级最高的
3. 同时`BeanFactoryPostProcessor`和`BeanDefinitionRegistryPostProcessor`可以都实现两个接口(`PriorityOrdered`,`Ordered`)用来定义执行的顺序，其中`PriorityOrdered`优先级最高，其次`Ordered`，最后是没有实现任何Order接口的

综上三点，我们是可以得出上面一大坨代码的执行顺序了

* 首先是拿到手动加入的`BeanFactoryPostProcessor`，挑出里面的`BeanDefinitionRegistryPostProcessor`类型，优先执行。
* 再拿到容器中的全部`BeanDefinitionRegistryPostProcessors`，按照实现的排序接口顺序执行
* 执行`BeanDefinitionRegistryPostProcessors`父类型`BeanFactoryPostProcessor`的方法
* 最后再拿到容器中全部`BeanFactoryPostProcessors`并过滤到已经执行过的`BeanDefinitionRegistryPostProcessors`，按照顺序依次执行


## registerBeanPostProcessors(beanFactory)

这里也是`spring`的一个扩展点，通过实现`BeanPostProcessor`接口来实现自定义逻辑，具体的执行时机，会在分析`getBaan`时讲到，这里的逻辑仅仅是将`BeanPostProcessor`用户自定义的`BeanPostProcessor`注册到容器

```java
public static void registerBeanPostProcessors(
		ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

	// Register BeanPostProcessorChecker that logs an info message when
	// a bean is created during BeanPostProcessor instantiation, i.e. when
	// a bean is not eligible for getting processed by all BeanPostProcessors.
	int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
	beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

	// Separate between BeanPostProcessors that implement PriorityOrdered,
	// Ordered, and the rest.
	List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
	List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
	List<String> orderedPostProcessorNames = new ArrayList<String>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
	for (String ppName : postProcessorNames) {
		if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			priorityOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	// First, register the BeanPostProcessors that implement PriorityOrdered.
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

	// Next, register the BeanPostProcessors that implement Ordered.
	List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
	for (String ppName : orderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		orderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, orderedPostProcessors);

	// Now, register all regular BeanPostProcessors.
	List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
	for (String ppName : nonOrderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		nonOrderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

	// Finally, re-register all internal BeanPostProcessors.
	sortPostProcessors(internalPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, internalPostProcessors);

	// Re-register post-processor for detecting inner beans as ApplicationListeners,
	// moving it to the end of the processor chain (for picking up proxies etc).
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

这里的逻辑其实跟上面的逻辑非常的相似，其实也都在一个class中，唯一的区别是这里并没有执行，仅仅只是做了下排序后就注册进了容器中，留到后面的步骤中在执行。这里也简单的列一下这里的要点

1. `BeanPostProcessor`有一个子接口`MergedBeanDefinitionPostProcessor`,这个是最晚被注册进容器的
2. 同样可以实现(`PriorityOrdered`,`Ordered`)接口用来定义执行的顺序


## initMessageSource()

国际化相关，这里就不做仔细分析了

## initApplicationEventMulticaster()

顾名思义。就是`spring`用来初始化事情广播器的

```java
protected void initApplicationEventMulticaster() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
		this.applicationEventMulticaster =
				beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
		}
	}
	else {
		this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
		beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
					APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
					"': using default [" + this.applicationEventMulticaster + "]");
		}
	}
}
```

看一下这里的代码比较的简单，就是一个 `if else`，先尝试获取名字为`APPLICATION_EVENT_MULTICASTER_BEAN_NAME`类型为`ApplicationEventMulticaster`的 bean，如果没有获取到，则配置一个默认的bean (`SimpleApplicationEventMulticaster`)

## onRefresh()

这里是留给子类实现的，没有什么逻辑

## registerListeners()

注册一些监听器

```java
protected void registerListeners() {
	// Register statically specified listeners first.
	for (ApplicationListener<?> listener : getApplicationListeners()) {
		getApplicationEventMulticaster().addApplicationListener(listener);
	}

	// Do not initialize FactoryBeans here: We need to leave all regular beans
	// uninitialized to let post-processors apply to them!
	String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
	for (String listenerBeanName : listenerBeanNames) {
		getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
	}

	// Publish early application events now that we finally have a multicaster...
	Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
	this.earlyApplicationEvents = null;
	if (earlyEventsToProcess != null) {
		for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
			getApplicationEventMulticaster().multicastEvent(earlyEvent);
		}
	}
}
```
## 尾言

> 下篇会解析`finishBeanFactoryInitialization`方法，并且会着重讲解初始化bean的整个生命周期

