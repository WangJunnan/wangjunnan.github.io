---
title: SpringBoot的启动过程
tags:
  - Spring
  - SpringBoot
categories:
  - Spring
  - SpringBoot
date: 2019-08-27 16:44:29
---
之前文章中有分析过Spring的启动过程，因为Spring boot用的更多了现在，简单看下Springboot的启动过程。其实毕竟核心还是 Spring Framework 那一套。个人觉得 Spring boot没啥特别核心的东西

```java
public ConfigurableApplicationContext run(String... args) {
    // 统计启动时间
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // 设置 java.awt.headless 参数
		configureHeadlessProperty();
    // 获取监听器
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
            // 创建 context容器
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
            // 准备容器
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
            // 刷新容器
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```

- createApplicationContext
```java
protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
		if (contextClass == null) {
			try {
				switch (this.webApplicationType) {
				case SERVLET:
					contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
					break;
				case REACTIVE:
					contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
					break;
				default:
					contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
				}
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Unable create a default ApplicationContext, "
								+ "please specify an ApplicationContextClass",
						ex);
			}
		}
    // 这里会引入一些BeanFactoryProcess，比如 ConfigurationClassPostProcessor 解析Configuration注解的
		return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
	}
```


注: 引入的BeanFactoryProcess 的逻辑在DEFAULT_SERVLET_WEB_CONTEXT_CLASS的构造方法中

```java
// 看下 AnnotationConfigServletWebServerApplicationContext 的构造方法
public AnnotationConfigServletWebServerApplicationContext() {
    	// 相当于xml中配置了 <context:annotation-config>
		this.reader = new AnnotatedBeanDefinitionReader(this);
    	// 相当于xml中配置了 <context:component-scan>
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
```


我们在看下 AnnotatedBeanDefinitionReader 的构造方法

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
		this(registry, getOrCreateEnvironment(registry));
	}

	/**
	 * Create a new {@code AnnotatedBeanDefinitionReader} for the given registry and using
	 * the given {@link Environment}.
	 * @param registry the {@code BeanFactory} to load bean definitions into,
	 * in the form of a {@code BeanDefinitionRegistry}
	 * @param environment the {@code Environment} to use when evaluating bean definition
	 * profiles.
	 * @since 3.1
	 */
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
        // 在这里 注册了一堆的 BeanFactoryProcess 
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}
```

下面主要prepareContext和refreshContext方法

- prepareContext

```java
private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
		context.setEnvironment(environment);
		postProcessApplicationContext(context);
		applyInitializers(context);
		listeners.contextPrepared(context);
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}

		// Add boot specific singleton beans
		context.getBeanFactory().registerSingleton("springApplicationArguments",
				applicationArguments);
		if (printedBanner != null) {
			context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
		}

		// Load the sources
		Set<Object> sources = getAllSources();
		Assert.notEmpty(sources, "Sources must not be empty");
        // 这里引入了 启动类
		load(context, sources.toArray(new Object[0]));
		listeners.contextLoaded(context);
	}
```

**注： 这里的主要作用就是引入了启动类，加入到 BeanFactory中**

- refreshContext
  - 后面其实 就是 Spring 统一的一套了
  - 可以看我的之前写的 Spring 源码解析系列



## 总结
Springboot的核心在于自动配置 快速集成 。一些原本需要在Spring xml文件配置的东西，Spring 只需要一个注解就帮我们都做了。不过前文也都说了，核心还是 Spring Framework。


