---
title: Spring的@Configuration @Bean注解
tags:
  - Spring
categories:
  - Spring
date: 2019-09-03 16:47:31
---
## 引言

之前Spring 一直用的xml配置文件比较多，现在发现新公司用的注解比较多。也就是通过定义一个配置类(以@Configuration注解的类)内部在方法上注解 [[@Bean](#)](#) 注解来完成Bean的依赖注入Spring容器。用注解来配置的话，其实就是减少了配置文件的工作，但总体来说代码可读性其实是会下降的。不过也省去了一大堆的xml配置文件

## 直接看源码

下面就从源码层面来看看  通过注解配置有什么不同

之前有分析过Spring对Bean的解析，**是通过把我们配置的Bean信息抽象成了一个BeanDefiniion对象。这个对象持有了我们配置的Bean的元数据**

那么其实 [[@Bean](#)](#) 的注解的原理也是类似，也是通过将我们配置的Bean信息抽象成一个BeanDefiniion对象

这里我们分两种情况讨论，一种是 Spring 通过 xml文件配置实现注解配置，一种就是我们现在非常流行的 SpringBoot实现的注解配置。其实两者实现原理一致，只是配置入口略有不同。

先看 普通 Spring 应用的配置:

如果我们要在  普通 Spring 应用中实现注解配置。那么我们需要在Spring的配置文件中配置以下几个配置

```java

<context:annotation-config>

<context:component-scan>

```

这两个配置的意思是什么呢？

我们直接看Spring是如何解析这两个标签的吧，通过在`META-INF/spring.handlers`目录下根据`spring.handlers`文件可以找到 Context 标签  解析入口类 `ContextNamespaceHandler`

```java

public class ContextNamespaceHandler extends NamespaceHandlerSupport {

@Override

public void init() {

registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());

registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());

registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());

registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());

registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());

registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());

registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());

registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());

}

}

```

看了上面的解析源码，可以知道annotation-config由AnnotationConfigBeanDefinitionParser解析。component-scan由ComponentScanBeanDefinitionParser解析。这里就是Spring XML 注解配置的入口，下面我们来依次分析这两个解析类干了什么事情？

- ComponentScanBeanDefinitionParser

```java

public BeanDefinition parse(Element element, ParserContext parserContext) {

String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);

basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);

String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,

ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);

// Actually scan for bean definitions and register them.

ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);

  // 这里扫描了我们配置的 base-package 包下全部 beanDefinitions，并抽象成了 BeanDefinitionHolder类

Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);

  // 将抽象的 beanDefinitions 全部注册进 Spring容器中，注 这里包含了被@Configuration注解的类

registerComponents(parserContext.getReaderContext(), beanDefinitions, element);

return  null;

}

```

- AnnotationConfigBeanDefinitionParser

主要逻辑在 AnnotationConfigUtils 工具类的 registerAnnotationConfigProcessors方法中

```java

public BeanDefinition parse(Element element, ParserContext parserContext) {

Object source = parserContext.extractSource(element);

// 初始化一些 BeanFactoryPostProcessor 进Spring容器，会在getBean之前全部执行。

  // 一般用来在容器实例化Bean前，对BeanDefinition做修改，或添加新的BeanDefinition等前置修改

  // 主要逻辑就在这里

Set<BeanDefinitionHolder> processorDefinitions =

AnnotationConfigUtils.registerAnnotationConfigProcessors(parserContext.getRegistry(), source);

// Register component for the surrounding <context:annotation-config> element.

CompositeComponentDefinition compDefinition = new CompositeComponentDefinition(element.getTagName(), source);

parserContext.pushContainingComponent(compDefinition);

// Nest the concrete beans in the surrounding component.

for (BeanDefinitionHolder processorDefinition : processorDefinitions) {

parserContext.registerComponent(new BeanComponentDefinition(processorDefinition));

}

// Finally register the composite component.

parserContext.popAndRegisterContainingComponent();

return  null;

}

// AnnotationConfigUtils.class

// 添加支持 annotation-config 配置的一些BeanFactoryPostProcessor 实现类

public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(

BeanDefinitionRegistry registry, @Nullable Object source) {

DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);

if (beanFactory != null) {

if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {

beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);

}

if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {

beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());

}

}

Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

// @Configuration 注解解析

if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {

RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);

def.setSource(source);

beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));

}

  // @Autowired 注解解析

if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {

RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);

def.setSource(source);

beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));

}

// @Required 注解解析

if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {

RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);

def.setSource(source);

beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));

}

// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.

  // 对 JSR-250做支持 解析注解 @Resource @PostConstruct @PreDestroy 

if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {

RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);

def.setSource(source);

beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));

}

// Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.

if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {

RootBeanDefinition def = new RootBeanDefinition();

try {

def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,

AnnotationConfigUtils.class.getClassLoader()));

}

catch (ClassNotFoundException ex) {

throw new IllegalStateException(

"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);

}

def.setSource(source);

beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));

}

if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {

RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);

def.setSource(source);

beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));

}

if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {

RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);

def.setSource(source);

beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));

}

return beanDefs;

}

```

可以看到 annotation-config 配置其实就是添加了几个 BeanFactoryPostProcessor 实现类，以此来实现 Autowired Configuration @Resource @PostConstruct @PreDestroy 等注解的实现。其他的注解暂且不看，我们直接看今天的重点  Configuration 注解是如何解析，@Configuration是通过 BeanFactoryPostProcessor 的实现类 ConfigurationClassPostProcessor 类在Spring启动的时候执行的（具体BeanFactoryPostProcessor的执行时机可以看我另一篇博文[[ [链接](https://www.yuque.com/wangjunnan/pnhnfo/bx15ai)](https://www.yuque.com/wangjunnan/pnhnfo/bx15ai)]()）

- ConfigurationClassPostProcessor

```java

public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {

int registryId = System.identityHashCode(registry);

  // 判断是否已经执行过

if (this.registriesPostProcessed.contains(registryId)) {

throw new IllegalStateException(

"postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);

}

if (this.factoriesPostProcessed.contains(registryId)) {

throw new IllegalStateException(

"postProcessBeanFactory already called on this post-processor against " + registry);

}

this.registriesPostProcessed.add(registryId);

  // 正式执行逻辑

processConfigBeanDefinitions(registry);

}

public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {

List<BeanDefinitionHolder> configCandidates = new ArrayList<>();

String[] candidateNames = registry.getBeanDefinitionNames();

// 遍历所有 BeanDefinition name

for (String beanName : candidateNames) {

BeanDefinition beanDef = registry.getBeanDefinition(beanName);

  // 判断是否已经处理过

if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||

ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {

if (logger.isDebugEnabled()) {

logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);

}

}

 // 条件筛选

else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {

configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));

}

}

// 栓选完为空  直接返回

if (configCandidates.isEmpty()) {

return;

}

// 根据 @Order 注解排序

configCandidates.sort((bd1, bd2) -> {

int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());

int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());

return Integer.compare(i1, i2);

});

// Detect any custom bean name generation strategy supplied through the enclosing application context

  // 获取 BeanName 生成策略

SingletonBeanRegistry sbr = null;

if (registry instanceof SingletonBeanRegistry) {

sbr = (SingletonBeanRegistry) registry;

if (!this.localBeanNameGeneratorSet) {

BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);

if (generator != null) {

this.componentScanBeanNameGenerator = generator;

this.importBeanNameGenerator = generator;

}

}

}

if (this.environment == null) {

this.environment = new StandardEnvironment();

}

// Parse each @Configuration class

  // 解析 @Configuration 注解

ConfigurationClassParser parser = new ConfigurationClassParser(

this.metadataReaderFactory, this.problemReporter, this.environment,

this.resourceLoader, this.componentScanBeanNameGenerator, registry);

Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);

Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());

do {

parser.parse(candidates);

parser.validate();

Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());

configClasses.removeAll(alreadyParsed);

// Read the model and create bean definitions based on its content

  // 读取并解析注册 BeanDefinition

if (this.reader == null) {

this.reader = new ConfigurationClassBeanDefinitionReader(

registry, this.sourceExtractor, this.resourceLoader, this.environment,

this.importBeanNameGenerator, parser.getImportRegistry());

}

this.reader.loadBeanDefinitions(configClasses);

alreadyParsed.addAll(configClasses);

candidates.clear();

 // 处理新加入的

if (registry.getBeanDefinitionCount() > candidateNames.length) {

String[] newCandidateNames = registry.getBeanDefinitionNames();

Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));

Set<String> alreadyParsedClasses = new HashSet<>();

for (ConfigurationClass configurationClass : alreadyParsed) {

alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());

}

for (String candidateName : newCandidateNames) {

if (!oldCandidateNames.contains(candidateName)) {

BeanDefinition bd = registry.getBeanDefinition(candidateName);

if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&

!alreadyParsedClasses.contains(bd.getBeanClassName())) {

candidates.add(new BeanDefinitionHolder(bd, candidateName));

}

}

}

candidateNames = newCandidateNames;

}

}

while (!candidates.isEmpty());

// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes

if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {

sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());

}

if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {

// Clear cache in externally provided MetadataReaderFactory; this is a no-op

// for a shared cache since it'll be cleared by the ApplicationContext.

((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();

}

}

```

具体的执行逻辑在ConfigurationClassParser 类的 parse 方法中

- ConfigurationClassParser

核心解析方法是 doProcessConfigurationClass方法，下面直接看这个方法的实现

```java

protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)

throws IOException {

// Recursively process any member (nested) classes first

processMemberClasses(configClass, sourceClass);

// Process any @PropertySource annotations

  // 解析 @PropertySource 

for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(

sourceClass.getMetadata(), PropertySources.class,

org.springframework.context.annotation.PropertySource.class)) {

if (this.environment instanceof ConfigurableEnvironment) {

processPropertySource(propertySource);

}

else {

logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +

"]. Reason: Environment must implement ConfigurableEnvironment");

}

}

// Process any @ComponentScan annotations

  // 解析 @ComponentScan 

Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(

sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);

if (!componentScans.isEmpty() &&

!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {

for (AnnotationAttributes componentScan : componentScans) {

// The config class is annotated with @ComponentScan -> perform the scan immediately

Set<BeanDefinitionHolder> scannedBeanDefinitions =

this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());

// Check the set of scanned definitions for any further config classes and parse recursively if needed

for (BeanDefinitionHolder holder : scannedBeanDefinitions) {

BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();

if (bdCand == null) {

bdCand = holder.getBeanDefinition();

}

if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {

parse(bdCand.getBeanClassName(), holder.getBeanName());

}

}

}

}

// Process any @Import annotations

  // 解析 @Import

processImports(configClass, sourceClass, getImports(sourceClass), true);

// Process any @ImportResource annotations

  // 解析 @ImportResource

AnnotationAttributes importResource =

AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);

if (importResource != null) {

String[] resources = importResource.getStringArray("locations");

Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");

for (String resource : resources) {

String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);

configClass.addImportedResource(resolvedResource, readerClass);

}

}

// Process individual @Bean methods

  // 解析 @Bean

Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);

for (MethodMetadata methodMetadata : beanMethods) {

configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));

}

// Process default methods on interfaces

processInterfaces(configClass, sourceClass);

// Process superclass, if any

if (sourceClass.getMetadata().hasSuperClass()) {

String superclass = sourceClass.getMetadata().getSuperClassName();

if (superclass != null && !superclass.startsWith("java") &&

!this.knownSuperclasses.containsKey(superclass)) {

this.knownSuperclasses.put(superclass, configClass);

// Superclass found, return its annotation metadata and recurse

return sourceClass.getSuperClass();

}

}

// No superclass -> processing is complete

return  null;

}

```

可以看到上面的代码依次解析了 @PropertySource，@ComponentScan [[@Bean](#)](#) [[@Import](#)](#) 等注解。这里我们重点关注 [[@Bean](#)](#) 的实现

```java

// 获取 @bean 注解修饰的  方法元数据信息

Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);

for (MethodMetadata methodMetadata : beanMethods) {

  // 封装成 BeanMethod 

configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));

}

```

下面我们回到 ConfigurationClassPostProcessor ，解析完 元数据之后，就是加载注册 BeanDefinition了

```java

this.reader.loadBeanDefinitions(configClasses);

public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {

TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();

for (ConfigurationClass configClass : configurationModel) {

loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);

}

}

private void loadBeanDefinitionsForConfigurationClass(

ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

if (trackedConditionEvaluator.shouldSkip(configClass)) {

String beanName = configClass.getBeanName();

if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {

this.registry.removeBeanDefinition(beanName);

}

this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());

return;

}

if (configClass.isImported()) {

registerBeanDefinitionForImportedConfigurationClass(configClass);

}

for (BeanMethod beanMethod : configClass.getBeanMethods()) {

  // 解析之前 解析 @bean 加入的 beanMethod信息

loadBeanDefinitionsForBeanMethod(beanMethod);

}

loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());

loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());

}

```

看看 loadBeanDefinitionsForBeanMethod 方法的实现，其实就是从BeanMethod中提取信息组装成 BeanDefinition

```java

private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {

ConfigurationClass configClass = beanMethod.getConfigurationClass();

MethodMetadata metadata = beanMethod.getMetadata();

String methodName = metadata.getMethodName();

// 判断是否需要跳过

if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {

configClass.skippedBeanMethods.add(methodName);

return;

}

if (configClass.skippedBeanMethods.contains(methodName)) {

return;

}

AnnotationAttributes bean = AnnotationConfigUtils.attributesFor(metadata, Bean.class);

Assert.state(bean != null, "No @Bean annotation attributes");

// 考虑别名 名称

List<String> names = new ArrayList<>(Arrays.asList(bean.getStringArray("name")));

String beanName = (!names.isEmpty() ? names.remove(0) : methodName);

// 注册别名 

for (String alias : names) {

this.registry.registerAlias(beanName, alias);

}

// Has this effectively been overridden before (e.g. via XML)?

  // 判断是否在 xml配置文件中  已经配置过

if (isOverriddenByExistingDefinition(beanMethod, beanName)) {

if (beanName.equals(beanMethod.getConfigurationClass().getBeanName())) {

throw new BeanDefinitionStoreException(beanMethod.getConfigurationClass().getResource().getDescription(),

beanName, "Bean name derived from @Bean method '" + beanMethod.getMetadata().getMethodName() +

"' clashes with bean name for containing configuration class; please make those names unique!");

}

return;

}

ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);

beanDef.setResource(configClass.getResource());

beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));

  // 这里可以知道 @bean 配置是 通过 FactoryMethod 来初始化的

if (metadata.isStatic()) {

// static @Bean method

beanDef.setBeanClassName(configClass.getMetadata().getClassName());

beanDef.setFactoryMethodName(methodName);

}

else {

// instance @Bean method

beanDef.setFactoryBeanName(configClass.getBeanName());

beanDef.setUniqueFactoryMethodName(methodName);

}

beanDef.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR);

beanDef.setAttribute(RequiredAnnotationBeanPostProcessor.SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);

AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata);

  // 设置autowire属性

Autowire autowire = bean.getEnum("autowire");

if (autowire.isAutowire()) {

beanDef.setAutowireMode(autowire.value());

}

// 设置initMethod 初始化方法

String initMethodName = bean.getString("initMethod");

if (StringUtils.hasText(initMethodName)) {

beanDef.setInitMethodName(initMethodName);

}

// 设置destroyMethod 销毁方法

String destroyMethodName = bean.getString("destroyMethod");

beanDef.setDestroyMethodName(destroyMethodName);

// 获取 scope

ScopedProxyMode proxyMode = ScopedProxyMode.NO;

AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);

if (attributes != null) {

beanDef.setScope(attributes.getString("value"));

proxyMode = attributes.getEnum("proxyMode");

if (proxyMode == ScopedProxyMode.DEFAULT) {

proxyMode = ScopedProxyMode.NO;

}

}

// Replace the original bean definition with the target one, if necessary

  // 如有必要，将原始Bean定义替换为目标Bean定义

BeanDefinition beanDefToRegister = beanDef;

if (proxyMode != ScopedProxyMode.NO) {

BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(

new BeanDefinitionHolder(beanDef, beanName), this.registry,

proxyMode == ScopedProxyMode.TARGET_CLASS);

beanDefToRegister = new ConfigurationClassBeanDefinition(

(RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata);

}

if (logger.isDebugEnabled()) {

logger.debug(String.format("Registering bean definition for @Bean method %s.%s()",

configClass.getMetadata().getClassName(), beanName));

}

this.registry.registerBeanDefinition(beanName, beanDefToRegister);

}

```

代码解析到这里为止，@Bean配置的Bean信息就被全部抽象成了一个 BeanDefinition对象，接下来的Bean加载等等一大堆逻辑走的就是Spring的同一套逻辑了。

以上是通过在配置文件中配置 ` <context:annotation-config> <context:component-scan>`。

## Springboot怎么做的

那么Spring Boot并不需要配置文件，它又是在哪里配置了入口呢？

其实在之前的分析Spring boot的启动过程时 已经有提到了 [[链接](https://www.yuque.com/wangjunnan/pnhnfo/qtapgd)](https://www.yuque.com/wangjunnan/pnhnfo/qtapgd) ，其实是在初始化 Spring容器的构造方法中进行了配置的加载，并且最终也是调用了 AnnotationConfigUtils 工具类的 registerAnnotationConfigProcessors方法

## 总结

终于写完了，其实分析来分析去，不论是Spring Boot Spring mvc 其实最核心不变的还是 Spring Framework，只要把核心搞清楚了，下次Spring 又推出什么 Spring xxx 也能应付自如

