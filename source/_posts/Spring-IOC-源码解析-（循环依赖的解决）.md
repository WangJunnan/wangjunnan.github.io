---
title: Spring IOC 源码解析 （循环依赖的解决）
tags:
  - Spring
categories:
  - java框架
  - Spring
date: 2018-12-21 15:59:42
---

## 引言

之前的几篇对`Spring IOC`源码分析的文章，大体上把`IOC`容器内部实现做了分析，但在有些细节上并没有很深入的去分析。本篇文章主要是分析`Spring IOC`容器对`Bean之间的循环依赖`是如何解决的

## 什么是循环依赖

那么什么是循环依赖呢？简单的理解一下，A依赖B，B又依赖A，这就构成了一个最简单的循环依赖，为了帮助大家理解，新建两个互相依赖的类（儿子和爸爸互相依赖没有错吧 ..）

```
public class Father {
    private String name;
    private Son son;
}

public class Son {
    private String name;
    private Father father;
}
```
这两个bean交给Spring管理

```
<bean id="son" class="com.wangjn.demo.impl.Son">
    <property name="name" value="son"></property>
    <property name="father" ref="father"></property>
</bean>

<bean id="father" class="com.wangjn.demo.impl.Father">
    <property name="name" value="father"></property>
    <property name="son" ref="son"></property>
</bean>
```

启动`Spring IOC`容器，用`getBean`方法可以成功获取`son`对象，并且也注入了`father`对象，可见`Spring`为我们解决了循环依赖的问题。可是按照正常创建`Bean`的流程来说，这个过程将会是一个死循环，因为在创建`son`对象为`son`注入`father`属性时，就会去获取`father`对象，而在获取`father`对象赋值`son`属性的时候，又会去获取`son`对象，从而就陷入了死循环，然后程序崩溃。

可是结果并不是我们预料的那样，接下来就来分析`Spring`是如何解决这个问题的

## Spring 如何解决循环依赖

之前对`IOC`源码分析的文章中有分析过`Bean`的创建过程，下面我将对循环依赖实现的某些细节作分析


## Spring 用缓存解决循环依赖

让我们回到`AbstractBeanFactory`的`doGetBean`方法，`doGetBean`方法就是我们通过容器`getBean`方法实际调用的逻辑，我们在这里着重关注`getSingleton`方法，之前的分析中有提到，调用`getSingleton(beanName)`方法的目的是为了从缓存中直接获取已经创建的Bean，而不必重复去创建。现在让我们进到`getSingleton`方法里面去看看它都做了啥，从哪个缓存取到了Bean对象

```
protected <T> T doGetBean(
		final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
		throws BeansException {

	final String beanName = transformedBeanName(name);
	Object bean;

	// 从缓存中获取 bean
	Object sharedInstance = getSingleton(beanName);
	... 省略其他创建bean的代码
}
```

### getSingleton方法

```
public Object getSingleton(String beanName) {
	// 默认都是允许提前暴露对象
	return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {	
	// 从创建完成的bean缓存中获取bean
	Object singletonObject = this.singletonObjects.get(beanName);
	// 判断该bean是否仍在创建中，意思是Bean已经完成实例化，但还不完整。属性还未完全注入
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		synchronized (this.singletonObjects) {
			// 从提前暴露的Bean缓存容器（earlySingletonObjects）中获取
			singletonObject = this.earlySingletonObjects.get(beanName);
			// 仍未获取到则从singletonFactories缓存中获取
			if (singletonObject == null && allowEarlyReference) {
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
					singletonObject = singletonFactory.getObject();
					// 加入到提前暴露Bean缓存（earlySingletonObjects）中
					this.earlySingletonObjects.put(beanName, singletonObject);
					// 从singletonFactories缓存中移除
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	// 返回对象，这里返回的不一定是完全创建的对象
	return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

从`getSingleton`方法中我们需要着重关注几个Bean的缓存，标题已经说了，缓存是解决循环依赖的关键，下面我介绍一下上面代码中提到了三种缓存 

```
/** Cache of singleton objects: bean name --> bean instance */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

/** Cache of singleton factories: bean name --> ObjectFactory */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

/** Cache of early singleton objects: bean name --> bean instance */
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
```

1.  `singletonObjects` 用于存放创建完成的单例对象
2.  `singletonFactories` 用于存放对象工厂类，这里是解决循环依赖的
3.  `earlySingletonObjects` 用于存放提前暴露的单例对象。指的是已经完成Bean的实例化，但还未完成属性注入的不完整对象

再来说上面代码中取缓存的步骤，首先肯定是从`singletonObjects`中获取完全创建完成的Bean对象，如果获取不到，则从提前暴露对象缓存（`earlySingletonObjects`）中获取，还获取不到再到`singletonFactories`中获取

到这里为止，我们只分析了取Bean缓存的过程，所以接下来我们要分析的就是放缓存的过程代码 


### 提前暴露Bean

现在让我们去到创建Bean的过程。如果缓存没取到，会执行创建Bean的逻辑，找到`AbstractAutowireCapableBeanFactory`类的`doCreateBean`方法，这个方法在之前的文章中有做过分析，但没有对Bean缓存处理做分析。这里我们着重看中间解决循环依赖的那部分

```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
		throws BeanCreationException {

	// Instantiate the bean.
	// 封装bean的容器
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
		// 这里是创建 BeanWrapper 
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
	Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
	mbd.resolvedTargetType = beanType;

	// Allow post-processors to modify the merged bean definition.
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			try {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			}
			catch (Throwable ex) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Post-processing of merged bean definition failed", ex);
			}
			mbd.postProcessed = true;
		}
	}

	// 判断是否需要提前暴露对象的引用，用于解决循环依赖
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isDebugEnabled()) {
			logger.debug("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
		addSingletonFactory(beanName, new ObjectFactory<Object>() {
			@Override
			public Object getObject() throws BeansException {
                                 // 这里会与AOP相关
				return getEarlyBeanReference(beanName, mbd, bean);
			}
		});
	}

	// Initialize the bean instance.
	Object exposedObject = bean;
	try {
		// 依赖注入的主逻辑
		populateBean(beanName, mbd, instanceWrapper);
		if (exposedObject != null) {
			//	执行一些初始化的方法
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
	}
	catch (Throwable ex) {
		if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
			throw (BeanCreationException) ex;
		}
		else {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
		}
	}

	if (earlySingletonExposure) {
		Object earlySingletonReference = getSingleton(beanName, false);
		if (earlySingletonReference != null) {
			if (exposedObject == bean) {
				exposedObject = earlySingletonReference;
			}
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
				for (String dependentBean : dependentBeans) {
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException(beanName,
							"Bean with name '" + beanName + "' has been injected into other beans [" +
							StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
							"] in its raw version as part of a circular reference, but has eventually been " +
							"wrapped. This means that said other beans do not use the final version of the " +
							"bean. This is often the result of over-eager type matching - consider using " +
							"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
				}
			}
		}
	}

	// Register bean as disposable.
	try {
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
	}

	return exposedObject;
}
```

通过看代码，可知在Bean完成实例化之后，注入属性之前，`Spring`就将这个不完整的`Bean`放到了`singletonFactories`缓存中，从而让这个`Bean`提前进行了暴露。这样子在后续的属性注入操作中，如果存在循环依赖，就会从缓存中获取到这个提前暴露的`Bean`，从而可以顺利完成依赖注入。**但是要注意这时候注入的对象是不完整的**，但是因为依赖方已经持有它的引用，所以后续对象的完整性是可以保证的

## 总结

本篇文章主要从`Spring`对`Bean`的缓存层面分析了其对`循环依赖`的解决，虽然是`Spring`帮我们解决了这个问题，但是对于实现的逻辑我们仍然应该去了解，譬如，通过查看源码可知`Spring`仅仅对单例类型的循环依赖进行解决，对于有状态的`Bean`Spring并没有去做处理，而是直接跑出异常，这些都是需要注意的。

