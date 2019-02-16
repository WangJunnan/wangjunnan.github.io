---
title: Spring 源码解析二（BeanFactory的创建）
tags:
  - Spring
categories:
  - java框架
  - Spring
date: 2018-03-09 14:12:20
---


## 引言

> 在第二期介绍容器的`refresh`方法开始之前，首先大家应该对Spring容器的整个继承体系有个大概的了解，不然就会有雾里看花的感觉

* 为了帮助大家理清整个继承体系，我将接下来所要涉及到的几个重要类及接口的继承关系贴上来，在阅读中如有疑惑的话，可以回过头来看看这几张图

##### XmlWebApplicationContext
由web容器启动的Spring容器类，注意与`DefaultListableBeanFactory`的区别
![image.png](http://upload-images.jianshu.io/upload_images/2717496-d858bd2adb034202.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### DefaultListableBeanFactory
实际实现的bean工厂类，获取bean操作的方法都在这里
![image.png](http://upload-images.jianshu.io/upload_images/2717496-79886eca0986f5d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### XmlBeanDefinitionReader
读取我们配置文件信息
![image.png](http://upload-images.jianshu.io/upload_images/2717496-55584bcef001c6d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



##### BeanFactory 接口及其实现

由以上图中可以看出，`BeanFactory` 接口定义了IOC容器最基本的功能规范，是每一个IOC容器都要遵守的基本规范。其中最重要的就是定义了`getBean`方法，让我们可以从容器中根据名字拿到所需要的bean。 **而`DefaultListableBeanFactory`就是其一个重要实现，与我们接触也是最多的，而`XmlWebApplicationContext`可以理解为一个更高级的IOC容器，除支持BeanFactory的基本功能外，还提供了很多扩展功能**

## 容器 refresh 方法

接着上节继续往下讲
`refresh` 是整个容器启动的核心方法，在`AbstractApplicationContext`中，其做的事情主要有三个：

* BeanFactory的创建(DefaultListableBeanFactory)
* BeanDefinition的Resource定位，载入，和注册
* 在实例化Bean之前对BeanDefinition进行修改(调用实现了接口`BeanFactoryPostProcessor`，`BeanDefinitionRegistryPostProcessor`的类)

```
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

			// 执行 BeanFactoryPostProcessors
			invokeBeanFactoryPostProcessors(beanFactory);

			// 注册 BeanPostProcessors
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			initMessageSource();

			// Initialize event multicaster for this context.
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			onRefresh();

			// Check for listener beans and register them.
			registerListeners();

			// 提前实例化化全部延迟加载的单例类型
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
## obtainFreshBeanFactory()

`obtainFreshBeanFactory()` 也在 `AbstractApplicationContext`类中，主要逻辑在`refreshBeanFactory` 方法中

```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	refreshBeanFactory();
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (logger.isDebugEnabled()) {
		logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
	}
	return beanFactory;
}
```

### refreshBeanFactory() (AbstractRefreshableApplicationContext)

这里做的事情主要有2个：

*1.创建BeanFactory (createBeanFactory())*
*2.载入BeanDefinitions (loadBeanDefinitions(beanFactory))*

```
protected final void refreshBeanFactory() throws BeansException {
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		customizeBeanFactory(beanFactory);
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
```

### createBeanFactory()

从这里可以知道容器所使用的BeanFactory实现类是 `DefaultListableBeanFactory`
```
protected DefaultListableBeanFactory createBeanFactory() {
	return new DefaultListableBeanFactory(getInternalParentBeanFactory());
}
```

### loadBeanDefinitions(beanFactory)  (XmlWebApplicationContext)

这个方法是在`XmlWebApplicationContext`实现的，Spring随处可见父类调用子类的设计模式（模板方法设计模式）

> 在解析`loadBeanDefinitions`方法之前，有必要先了解`BeanDefinition`在Spring中所表示的概念 （通俗的理解可以说是bean的抽象数据结构，它包括属性参数，构造器参数，以及其他具体的参数，就是我们从配置信息中读出来Bean的抽象数据结构）有兴趣的可以去看看BeanDefinition的数据结构。

```
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	// 创建一个XmlBeanDefinitionReader，用来读取配置文件配置的Bean信息，解析成beanDefinition，并配置beanFactory 方便回调
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	// 配置XmlBeanDefinitionReader的一些基本配置
	beanDefinitionReader.setEnvironment(getEnvironment());
	// 配置ResourceLoader（XmlWebApplicationContext 是实现了ResourceLoader接口的，不清楚的可以上图的继承链）
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	// Allow a subclass to provide custom initialization of the reader,
	// 这里没做任何事情
	initBeanDefinitionReader(beanDefinitionReader);
	loadBeanDefinitions(beanDefinitionReader);
}
```
### loadBeanDefinitions(XmlBeanDefinitionReader reader)
XmlBeanDefinitionReader 根据配置文件位置调用 loadBeanDefinitions
```
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
	String[] configLocations = getConfigLocations();
	if (configLocations != null) {
		for (String configLocation : configLocations) {
			reader.loadBeanDefinitions(configLocation);
		}
	}
}
```

## 尾言

> 到目前为止，我们知道了IOC容器的创建，以及BeanDefinition是在哪里被加载的。接下来要讲的自然就是配置信息是如何被解析加载的，以及如何注册进BeanFactory的。see => 下期
