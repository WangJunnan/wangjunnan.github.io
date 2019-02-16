---
title: Spring IOC 源码解析五（非延迟加载bean的初始化）
tags:
  - Spring
categories:
  - java框架
  - Spring
date: 2018-11-23 15:53:34
---

## 引言

> 上期分析了`refresh`方法中大部分，这期我们只分析`finishBeanFactoryInitialization(beanFactory)`这一个方法，分析这个方法的过程中我们会顺藤摸瓜把`bean`初始化的整个流程都分析一遍

## finishBeanFactoryInitialization(beanFactory)

提前初始化一些非延迟加载的单例类型的`bean`，这里主要看最后一行 `beanFactory.preInstantiateSingletons()`,这里是初始化非延迟加载单例的入口

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// 初始化类型转化的bean
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}

	// Register a default embedded value resolver if no bean post-processor
	// (such as a PropertyPlaceholderConfigurer bean) registered any before:
	// at this point, primarily for resolution in annotation attribute values.
	// 注册默认的值解析器
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
			@Override
			public String resolveStringValue(String strVal) {
				return getEnvironment().resolvePlaceholders(strVal);
			}
		});
	}

	// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}

	// Stop using the temporary ClassLoader for type matching.
	beanFactory.setTempClassLoader(null);

	// Allow for caching all bean definition metadata, not expecting further changes.
	beanFactory.freezeConfiguration();

	// Instantiate all remaining (non-lazy-init) singletons.
	beanFactory.preInstantiateSingletons();
}
```
## preInstantiateSingletons

方法在`DefaultListableBeanFactory`中，用于初始化所有非延迟加载的单例

```java
public void preInstantiateSingletons() throws BeansException {
	if (this.logger.isDebugEnabled()) {
		this.logger.debug("Pre-instantiating singletons in " + this);
	}

	// 所有注册完成的 beanDefinitionNames 
	List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

	// 遍历所有非延迟加载的单例类型
	for (String beanName : beanNames) {
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			// 判断类型是否为 FactoryBean
			if (isFactoryBean(beanName)) {
				final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
				boolean isEagerInit;
				if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
					isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
						@Override
						public Boolean run() {
							return ((SmartFactoryBean<?>) factory).isEagerInit();
						}
					}, getAccessControlContext());
				}
				else {
					isEagerInit = (factory instanceof SmartFactoryBean &&
							((SmartFactoryBean<?>) factory).isEagerInit());
				}
				// 是否需要立即加载
				if (isEagerInit) {
					getBean(beanName);
				}
			}
			else {
				getBean(beanName);
			}
		}
	}

	// Trigger post-initialization callback for all applicable beans...
	for (String beanName : beanNames) {
		Object singletonInstance = getSingleton(beanName);
		if (singletonInstance instanceof SmartInitializingSingleton) {
			final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged(new PrivilegedAction<Object>() {
					@Override
					public Object run() {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}
				}, getAccessControlContext());
			}
			else {
				smartSingleton.afterSingletonsInstantiated();
			}
		}
	}
}
```

可以看到这段代码主要做了2个事情

1. 判断bean的类型是否是`factoryBean`，如果是的话判断类型是不是`SmartFactoryBean`，再获取是否需要立即加载的`isEagerInit`属性，执行`getBean`方法
2. 在单例`bean`都初始化完成后，循环判断bean的类型是否是`SmartInitializingSingleton`，是的话会在这时候执行`afterSingletonsInstantiated`方法

接下来就是获取bean的流程代码

## getBean

`getBean`方法最终都是走的`doGetBean`方法

```java
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}
```

## doGetBean

```java
protected <T> T doGetBean(
		final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
		throws BeansException {

	final String beanName = transformedBeanName(name);
	Object bean;

	// 首先检查缓存是否有，有则直接取
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		if (logger.isDebugEnabled()) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
						"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
		// 这里是取FactoryBean 
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}
	
	// 缓存里没有 第一次初始化Bean
	else {
		// 判断是不是处于循环依赖当中，有则直接报错
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// 检查父容器是否存在此Bean的定义
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			// 返回原始的名称 加上FactoryBean符号
			String nameToLookup = originalBeanName(name);
			if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}

		if (!typeCheckOnly) {
			// 把Bean的状态改为已创建，并从mergedBeanDefinitions 中移除
			markBeanAsCreated(beanName);
		}

		try {
			// 合并BeanDefinition转化成RootBeanDefinition
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);

			// 获取配置的 depends-On 依赖,优先加载
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
					if (isDependent(beanName, dep)) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
					}
					registerDependentBean(dep, beanName);
					try {
						// 递归获取
						getBean(dep);
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
					}
				}
			}

			// Create bean instance.
			// 如果是单例
			if (mbd.isSingleton()) {
				// 这里会把创建完成的bean加入缓存
				sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
					@Override
					public Object getObject() throws BeansException {
						try {
							// 创建bean的主逻辑
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					}
				});
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}
			
			// 判断类型 prototype 有状态的bean
			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				Object prototypeInstance = null;
				try {
					beforePrototypeCreation(beanName);
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
					afterPrototypeCreation(beanName);
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}

			else {
				// 其他类型的scope
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; consider " +
							"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// Check if required type matches the type of the actual bean instance.
	// 如果有指定的类型，则检查是否与实际bean实例的类型匹配
	if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
		try {
			return getTypeConverter().convertIfNecessary(bean, requiredType);
		}
		catch (TypeMismatchException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to convert bean '" + name + "' to required type '" +
						ClassUtils.getQualifiedName(requiredType) + "'", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}

```

看完上面的代码，我们作总结：
首先，创建`bean`之前会检查单例缓存里有没有，有的话会直接返回。然后会检查父容器是否有这个bean的定义，一直找到最顶层的父容器（有点类型双亲委派），最后会在父容器执行这个bean的获取，之后获取`depends-On`如果有配置的话会优先获取（递归获取直到依赖的所有`depends-On`），最后才是根据`scope`调用`AbstractAutowireCapableBeanFactory`的`createBean`方法

## createBean

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
	if (logger.isDebugEnabled()) {
		logger.debug("Creating instance of bean '" + beanName + "'");
	}
	RootBeanDefinition mbdToUse = mbd;

	// Make sure bean class is actually resolved at this point, and
	// clone the bean definition in case of a dynamically resolved Class
	// which cannot be stored in the shared merged bean definition.
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}

	// Prepare method overrides.
	try {
		mbdToUse.prepareMethodOverrides();
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
				beanName, "Validation of method overrides failed", ex);
	}

	try {
		// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
		// 扩展点，可以通过实现InstantiationAwareBeanPostProcessor 接口一个返回代理而不是目标bean实例
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
				"BeanPostProcessor before instantiation of bean failed", ex);
	}

	Object beanInstance = doCreateBean(beanName, mbdToUse, args);
	if (logger.isDebugEnabled()) {
		logger.debug("Finished creating instance of bean '" + beanName + "'");
	}
	return beanInstance;
}
```
这段代码主要看一下`resolveBeforeInstantiation(beanName, mbdToUse)`，这里是spring提供的一个扩展点，可以通过实现`InstantiationAwareBeanPostProcessor`接口在这里返回bean的代理
然后就可以直接看最后的`doCreateBean(beanName, mbdToUse, args)`方法

## doCreateBean

```java
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

这段代码主要看这三个方法

1. `createBeanInstance(beanName, mbd, args)` 创建bean
2. `populateBean(beanName, mbd, instanceWrapper)` 组装bean的依赖
3. `initializeBean(beanName, exposedObject, mbd)` 调用bean的一些初始化方法，以及扩展接口

接下来我们着重来看这三个方法 

### createBeanInstance

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
	// Make sure bean class is actually resolved at this point.
	// 获取bean class
	Class<?> beanClass = resolveBeanClass(mbd, beanName);

	// 不是 pullic 直接抛异常
	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}
	// 使用工厂方法实例化bean
	if (mbd.getFactoryMethodName() != null)  {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}

	// Shortcut when re-creating the same bean...
	// 判断bean是否创建过
	boolean resolved = false;
	boolean autowireNecessary = false;
	if (args == null) {
		synchronized (mbd.constructorArgumentLock) {
			if (mbd.resolvedConstructorOrFactoryMethod != null) {
				resolved = true;
				autowireNecessary = mbd.constructorArgumentsResolved;
			}
		}
	}
	// 如果已经加载过
	if (resolved) {
		if (autowireNecessary) {
			// 有参构造方法
			return autowireConstructor(beanName, mbd, null, null);
		}
		else {
			// 使用无参构造
			return instantiateBean(beanName, mbd);
		}
	}

	// Need to determine the constructor...
	// 寻找一个合适的构造方法
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null ||
			mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
			mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
		return autowireConstructor(beanName, mbd, ctors, args);
	}

	// No special handling: simply use no-arg constructor.
	return instantiateBean(beanName, mbd);
}
```

这个方法的主要逻辑其实就是在找一个合适的初始化方法

1. 首先判断`RootBeanDefinition`是否含有`factory-method`配置，有的话使用工厂方法实例化bean
2. 之后的逻辑就是寻找一个合适的构造方法用于bean的实例化
	* 如果判断是有参的构造方法，则通过`autowireConstructor`实例化，初始化的过程很繁琐，其中对构造方法参数的依赖也是通过递归`getBean()`来实现的
	* 无参的则使用`instantiateBean`实例化

到这里执行完`bean`已经创建了(spring默认是使用`CGLIB`)，但是这时候的`bean`是不完整的，相关的依赖项还并没有被注入

### populateBean(beanName, mbd, instanceWrapper)
组装bean的逻辑都在这里

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
	// 获取的配置的property依赖
	PropertyValues pvs = mbd.getPropertyValues();

	if (bw == null) {
		if (!pvs.isEmpty()) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
		}
		else {
			// Skip property population phase for null instance.
			return;
		}
	}

	// 扩展点 执行实现了InstantiationAwareBeanPostProcessors接口方法
	// 在设置属性依赖之前可以有机会修改bean的属性配置
	boolean continueWithPropertyPopulation = true;

	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					continueWithPropertyPopulation = false;
					break;
				}
			}
		}
	}
	
	// 判断是否需要继续组装bean属性
	if (!continueWithPropertyPopulation) {
		return;
	}
	
	// 判断自动配的类型（按名称和按类型）
	if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
			mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
		MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

		// Add property values based on autowire by name if applicable.
		// 按bean的名称注入
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
			autowireByName(beanName, mbd, bw, newPvs);
		}

		// Add property values based on autowire by type if applicable.
		// 按bean的类型注入
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			autowireByType(beanName, mbd, bw, newPvs);
		}

		pvs = newPvs;
	}

	// 判断是否需要依赖检查
	boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
	boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

	if (hasInstAwareBpps || needsDepCheck) {
		PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
		if (hasInstAwareBpps) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
					if (pvs == null) {
						return;
					}
				}
			}
		}
		if (needsDepCheck) {
			// 检查依赖
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}
	}
	
	// 组装属性
	applyPropertyValues(beanName, mbd, bw, pvs);
}
```
通过查看代码，可知上面不止做了组装`bean`这一件事，主要做了下面几件事情:

1. 获取spring配置项中配置的`property`属性依赖
2. 执行实现了`InstantiationAwareBeanPostProcessors`接口的方法，该接口可以在属性被注入前做一些修改，并且可以直接返回，不执行下面的属性注入逻辑
3. 判断是否有配置自动注入，有的话会按配置执行按名称（`autowireByName`方法）或类型（`autowireByType`方法）注入
4. 判断是否执行依赖检查（按配置的依赖检查等级check）
5. 最后执行bean属性的最终装配 （`applyPropertyValues(beanName, mbd, bw, pvs)`）

我们可以简单看一下 `autowireByName`的实现

#### autowireByName

```java
protected void autowireByName(
		String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
	
	// 返回需要自动依赖注入的属性名称
	String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
	for (String propertyName : propertyNames) {
		if (containsBean(propertyName)) {
			Object bean = getBean(propertyName);
			// 将获取的依赖bean放回 待装配
			pvs.add(propertyName, bean);
			registerDependentBean(propertyName, beanName);
			if (logger.isDebugEnabled()) {
				logger.debug("Added autowiring by name from bean name '" + beanName +
						"' via property '" + propertyName + "' to bean named '" + propertyName + "'");
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
						"' by name: no matching bean found");
			}
		}
	}
}
```

可知如果是按`name`注入的话，首先会先获取需要自动依赖注入的bean名称，然后是通过`getBean(beanName)`获取依赖项的。同理`autowireByType`

#### applyPropertyValues

在执行`applyPropertyValues`方法之前，我们已经准备好了所有的依赖项了，接下来的事情就是把这些依赖项装配到`bean`对应的属性中

```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
	// 待装配的属性为空 直接返回
	if (pvs == null || pvs.isEmpty()) {
		return;
	}

	MutablePropertyValues mpvs = null;
	List<PropertyValue> original;

	if (System.getSecurityManager() != null) {
		if (bw instanceof BeanWrapperImpl) {
			((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
		}
	}
	// 
	if (pvs instanceof MutablePropertyValues) {
		mpvs = (MutablePropertyValues) pvs;
		if (mpvs.isConverted()) {
			// Shortcut: use the pre-converted values as-is.
			try {
				bw.setPropertyValues(mpvs);
				return;
			}
			catch (BeansException ex) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Error setting property values", ex);
			}
		}
		original = mpvs.getPropertyValueList();
	}
	else {
		original = Arrays.asList(pvs.getPropertyValues());
	}

	TypeConverter converter = getCustomTypeConverter();
	if (converter == null) {
		converter = bw;
	}
	// 用于加载beanDefinition所对应的值
	BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

	// 创建一个PropertyValue的深拷贝
	List<PropertyValue> deepCopy = new ArrayList<PropertyValue>(original.size());
	boolean resolveNecessary = false;
	for (PropertyValue pv : original) {
		if (pv.isConverted()) {
			deepCopy.add(pv);
		}
		else {
			String propertyName = pv.getName();
			Object originalValue = pv.getValue();
			// 加载对应属性值 value
			Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
			Object convertedValue = resolvedValue;
			boolean convertible = bw.isWritableProperty(propertyName) &&
					!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
			if (convertible) {
				convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
			}
			// Possibly store converted value in merged bean definition,
			// in order to avoid re-conversion for every created bean instance.
			if (resolvedValue == originalValue) {
				if (convertible) {
					pv.setConvertedValue(convertedValue);
				}
				deepCopy.add(pv);
			}
			else if (convertible && originalValue instanceof TypedStringValue &&
					!((TypedStringValue) originalValue).isDynamic() &&
					!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
				pv.setConvertedValue(convertedValue);
				deepCopy.add(pv);
			}
			else {
				resolveNecessary = true;
				deepCopy.add(new PropertyValue(pv, convertedValue));
			}
		}
	}
	if (mpvs != null && !resolveNecessary) {
		mpvs.setConverted();
	}

	// Set our (possibly massaged) deep copy.
	try {
		// 设置属性信息
		bw.setPropertyValues(new MutablePropertyValues(deepCopy));
	}
	catch (BeansException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Error setting property values", ex);
	}
}
```

这段代码做的主要事情就是通过配置的属性去加载属性对应的值，真正为bean设置属性的逻辑在`bw.setPropertyValues(new MutablePropertyValues(deepCopy))`中，这里其实就是通过反射调用`set`方法进行属性的设置的。
代码执行到这里，`bean`的创建已经基本完整了,`populateBean`方法的工作已经完成了，下面要做的就是一些初始化的工作


### initializeBean(beanName, exposedObject, mbd)

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
	// 安全管理器 判断执行权限
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged(new PrivilegedAction<Object>() {
			@Override
			public Object run() {
				invokeAwareMethods(beanName, bean);
				return null;
			}
		}, getAccessControlContext());
	}
	else {
		// 执行实现了一系列Aware的接口方法
		invokeAwareMethods(beanName, bean);
	}

	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		// 执行BeanPostProcessor接口postProcessBeforeInitialization方法
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	try {
		// 先执行InitializingBean接口afterPropertiesSet方法
		// 再执行自定义配置的初始化方法(init-method)
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}

	if (mbd == null || !mbd.isSynthetic()) {
		// 执行BeanPostProcessor接口postProcessAfterInitialization方法
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}
```
上述代码做的都是bean初始化的事情

1. `invokeAwareMethods`方法，如果有`bean`又实现`Aware`相关的接口，则调用这些接口的方法给bean注入一些信息(`beanName`，`BeanClassLoader`，`BeanFactory`)
2. `applyBeanPostProcessorsBeforeInitialization` 执行`BeanPostProcessor`接口`postProcessBeforeInitialization`方法，`BeanPostProcessor`接口在上一篇解析中有讲到过，是在`refresh`方法的`registerBeanPostProcessors(beanFactory)`中注册的，是spring提供的非常重要的一个自定义扩展点
3. `invokeInitMethods`方法，这里是判断是否有实现`InitializingBean`接口，有则执行`afterPropertiesSet`初始化方法，并且如果有配置自定义的`init-method`方法则也一并执行
4. `applyBeanPostProcessorsBeforeInitialization` 执行`BeanPostProcessor`接口`postProcessAfterInitialization`方法

### 尾言
> 随着解析进行到这里，我们已经把`bean`的整个生命周期都走了个大概。相信大家对`spring ioc` 的具体实现已经有了个基本的了解了


