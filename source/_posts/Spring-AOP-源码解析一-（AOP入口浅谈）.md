---
title: Spring AOP 源码解析一 （AOP入口浅谈）
tags:
  - Spring
categories:
  - java框架
  - Spring
date: 2018-11-30 15:55:08
---

## 引言

在进行aop的源码解析之前，我们首先要知道`Spring AOP`是怎么用的，只有知道怎么用的，我们才能更好的理解aop，才能进行下一步解析


## 使用AOP

好了，废话不多说，我们直接动手来用一下`spring`的`AOP`实现

* 首先新建一个`MyPointCut`接口，并且实现它

```java
/**
 * MyPointCut
 *
 * @author wangjn
 * @date 2018/12/7
 */
public interface MyPointCut {

    void sayHello();
}
```

```java
/**
 * MyPointCutImpl
 *
 * @author wangjn
 * @date 2018/12/10
 */
public class MyPointCutImpl implements MyPointCut {

    @Override
    public void sayHello() {
        System.out.println("hello world");
    }
}
```

* 新建一个`MyAspect`接口，接口里的方法是我们之后要切入的aop逻辑方法

```
/**
 * MyAspect
 */
public interface MyAspect {

    void logBefore();
    void logAfter();
}
```

```
/**
 * MyAspectImpl
 */
public class MyAspectImpl implements MyAspect {

    @Override
    public void logBefore() {
        System.out.println("log before");
    }

    @Override
    public void logAfter() {
        System.out.println("log after");
    }
}
```

* `spring xml` 配置

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean id="myPointCut" class="com.wangjn.demo.impl.MyPointCutImpl"></bean>
    <bean id="aspectTest" class="com.wangjn.demo.MyAspectImpl"/>
    <aop:config>
        <aop:aspect id="log" ref="aspectTest">
            <aop:pointcut id="addAllMethod" expression="execution(* com.wangjn.demo.MyPointCut.*(..))" />
            <aop:before method="logBefore" pointcut-ref="addAllMethod" />
            <aop:after method="logAfter" pointcut-ref="addAllMethod" />
        </aop:aspect>
    </aop:config>

</beans>
```

* 写个main方法

```
/**
 * Start
 */
public class Start {

    public static void main(String[] args) throws IOException {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"spring.xml"});
        context.start();
        // 取得bean myPointCut
        MyPointCut myPointCut = (MyPointCut)context.getBean("myPointCut");
        myPointCut.sayHello();
    }
}
```

执行结果输出，可知`myPointCut`的方案前后都额外执行了aop要织入的逻辑，这就是aop的神奇之处，**可以在原代码毫无感知的情况下执行额外逻辑**

```
log before
hello world
log after
```

* 通过上面代码简单的演示，我们大致知道了aop是怎么使用的，不过显然知道怎么用并不是我们所要追求的，接下来就我们从aop的源码入口一步一步的解析它的实现原理

## AOP源码入口

要想知道aop的实现原理，我们首先要了解的就是`spinng ioc`容器是怎么解析我们的aop配置的，这里我们暂且只看xml形式的配置解析

好了，回到上面的`spring配置文件`
```
<aop:config>
        <aop:aspect id="log" ref="aspectTest">
            <aop:pointcut id="addAllMethod" expression="execution(* com.wangjn.demo.MyPointCut.*(..))" />
            <aop:before method="logBefore" pointcut-ref="addAllMethod" />
            <aop:after method="logAfter" pointcut-ref="addAllMethod" />
        </aop:aspect>
    </aop:config>
```
可以看到都是`spring aop`的自定义标签，在之前解析`spring ioc源码`的时候我们已经知道，spring是支持自定义的标签解析的，自定义标签的入口文件在`META-INF/spring.handlers`中，我们找到`spring aop`jar包下的这个目录，打开这个文件

```
http\://www.springframework.org/schema/aop=org.springframework.aop.config.AopNamespaceHandler
```
可以看到这里配置了一个aop命名空间的解析器 `org.springframework.aop.config.AopNamespaceHandler`，现在我们就从这个入口文件来进行源码分析

## AopNamespaceHandler

```
public class AopNamespaceHandler extends NamespaceHandlerSupport {

	/**
	 * Register the {@link BeanDefinitionParser BeanDefinitionParsers} for the
	 * '{@code config}', '{@code spring-configured}', '{@code aspectj-autoproxy}'
	 * and '{@code scoped-proxy}' tags.
	 */
	@Override
	public void init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		// 支持 @AspectJ 声明式配置
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace as of 2.1
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}

}
```

这里我们可以看到对`config`属性的解析是由`ConfigBeanDefinitionParser`完成的。`ConfigBeanDefinitionParser`是`BeanDefinitionParser`接口的实现，其中的`parse`方法封装了对`<aop:config>`属性标签的解析逻辑

## ConfigBeanDefinitionParser的parse方法

```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
	CompositeComponentDefinition compositeDef =
			new CompositeComponentDefinition(element.getTagName(), parserContext.extractSource(element));
	parserContext.pushContainingComponent(compositeDef);

	configureAutoProxyCreator(parserContext, element);

	List<Element> childElts = DomUtils.getChildElements(element);
	for (Element elt: childElts) {
		String localName = parserContext.getDelegate().getLocalName(elt);
		// pointCut标签
		if (POINTCUT.equals(localName)) {
			parsePointcut(elt, parserContext);
		}
		// advisor标签
		else if (ADVISOR.equals(localName)) {
			parseAdvisor(elt, parserContext);
		}
		// aspect标签
		else if (ASPECT.equals(localName)) {
			parseAspect(elt, parserContext);
		}
	}

	parserContext.popAndRegisterContainingComponent();
	return null;
}
```

可以清晰的看到，这里分别对`pointcut`,`advisor`,`aspect`三种标签进行解析


## configureAutoProxyCreator(parserContext, element)

这里配置了代理创建类，调用了`AopNamespaceUtils.registerAspectJAutoProxyCreatorIfNecessary`方法完成配置

```
private void configureAutoProxyCreator(ParserContext parserContext, Element element) {
	AopNamespaceUtils.registerAspectJAutoProxyCreatorIfNecessary(parserContext, element);
}
```

我们来看下 `AopNamespaceUtils.registerAspectJAutoProxyCreatorIfNecessary`的实现

```
public static void registerAspectJAutoProxyCreatorIfNecessary(
		ParserContext parserContext, Element sourceElement) {
	// 获取代理创建类 AspectJAwareAdvisorAutoProxyCreator并命名为org.springframework.aop.config.internalAutoProxyCreator
	
	BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAutoProxyCreatorIfNecessary(
			parserContext.getRegistry(), parserContext.extractSource(sourceElement));
	// 判断是否使用 CGLIB 创建代理，指定proxy-target-class=true则使用CGLIB，否则使用JDK动态代理
	useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
	// 往 spring ioc 容器注册AspectJAwareAdvisorAutoProxyCreator代理创建器
	registerComponentIfNecessary(beanDefinition, parserContext);
}
```

可以看到这段代码主要做了两件事情

1. 注册代理创建器 `AspectJAwareAdvisorAutoProxyCreator`，这个类很重要，是 `aop`的核心实现类
2. 指定创建代理的方式，其中如果`<aop:config>`节点配置了`proxy-target-class=true`，则使用`CGLIB`创建代理，否则默认使用`JDK 动态代理`


## parsePointcut

解析aop的切入点配置，对应 `<aop:pointcut>`标签
```
private AbstractBeanDefinition parsePointcut(Element pointcutElement, ParserContext parserContext) {
	// 获取我们配置的 pointCut 切入点 id
	String id = pointcutElement.getAttribute(ID);
	// 获取要切入的 expression 表达式
	String expression = pointcutElement.getAttribute(EXPRESSION);

	AbstractBeanDefinition pointcutDefinition = null;

	try {
		// 记录当前处理节点
		this.parseState.push(new PointcutEntry(id));
		// 创建 AspectJExpressionPointcut 类型的 beanDefinition，并且把expression属性赋给此bean
		pointcutDefinition = createPointcutDefinition(expression);
// 设置源数据		pointcutDefinition.setSource(parserContext.extractSource(pointcutElement));
		
		// 设置 bean的id名称
		String pointcutBeanName = id;
		if (StringUtils.hasText(pointcutBeanName)) {
		// 注册beanDefinution 至spring容器
		parserContext.getRegistry().registerBeanDefinition(pointcutBeanName, pointcutDefinition);
		}
		else {
			pointcutBeanName = parserContext.getReaderContext().registerWithGeneratedName(pointcutDefinition);
		}

		parserContext.registerComponent(
				new PointcutComponentDefinition(pointcutBeanName, pointcutDefinition, expression));
	}
	finally {
		this.parseState.pop();
	}

	return pointcutDefinition;
}
```

这里主要是配置了aop切入点的bean `AspectJExpressionPointcut`


## parseAspect

```
private void parseAspect(Element aspectElement, ParserContext parserContext) {
	// 获取切面id
	String aspectId = aspectElement.getAttribute(ID);
	// 获取切面要执行的具体逻辑类 这个是必须设置的属性
	String aspectName = aspectElement.getAttribute(REF);

	try {
		this.parseState.push(new AspectEntry(aspectId, aspectName));
		List<BeanDefinition> beanDefinitions = new ArrayList<BeanDefinition>();
		List<BeanReference> beanReferences = new ArrayList<BeanReference>();

		List<Element> declareParents = DomUtils.getChildElementsByTagName(aspectElement, DECLARE_PARENTS);
		for (int i = METHOD_INDEX; i < declareParents.size(); i++) {
			Element declareParentsElement = declareParents.get(i);
			beanDefinitions.add(parseDeclareParents(declareParentsElement, parserContext));
		}

		// We have to parse "advice" and all the advice kinds in one loop, to get the
		// ordering semantics right.
		// 解析advice标签
		NodeList nodeList = aspectElement.getChildNodes();
		boolean adviceFoundAlready = false;
		for (int i = 0; i < nodeList.getLength(); i++) {
			Node node = nodeList.item(i);
			// 判断 advice 的类型是否是限定的几种（after before around）
			if (isAdviceNode(node, parserContext)) {
				if (!adviceFoundAlready) {
					adviceFoundAlready = true;
					// 如果没有配置 ref 属性，抛出异常
					if (!StringUtils.hasText(aspectName)) {
						parserContext.getReaderContext().error(
								"<aspect> tag needs aspect bean reference via 'ref' attribute when declaring advices.",
								aspectElement, this.parseState.snapshot());
						return;
					}
					beanReferences.add(new RuntimeBeanReference(aspectName));
				}
				// 开始解析advice节点
				AbstractBeanDefinition advisorDefinition = parseAdvice(
						aspectName, i, aspectElement, (Element) node, parserContext, beanDefinitions, beanReferences);
				beanDefinitions.add(advisorDefinition);
			}
		}

		AspectComponentDefinition aspectComponentDefinition = createAspectComponentDefinition(
				aspectElement, aspectId, beanDefinitions, beanReferences, parserContext);
		parserContext.pushContainingComponent(aspectComponentDefinition);
		// 解析 aspect 标签中的 pointcut
		List<Element> pointcuts = DomUtils.getChildElementsByTagName(aspectElement, POINTCUT);
		for (Element pointcutElement : pointcuts) {
			parsePointcut(pointcutElement, parserContext);
		}

		parserContext.popAndRegisterContainingComponent();
	}
	finally {
		this.parseState.pop();
	}
}
```

## parseAdvice

`parseAdvice`这段代码的主要逻辑其实就是解析 `advice`标签，并最终包装成`advisor`。`advisor`其实相当于整合了`advice 通知`和`pointcut 切点`，相当于通过 `advisor`把切点和要执行的切面逻辑连接了起来

```
private AbstractBeanDefinition parseAdvice(
		String aspectName, int order, Element aspectElement, Element adviceElement, ParserContext parserContext,
		List<BeanDefinition> beanDefinitions, List<BeanReference> beanReferences) {

	try {
		this.parseState.push(new AdviceEntry(parserContext.getDelegate().getLocalName(adviceElement)));

		// create the method factory bean
		// 创建切面增强方法对应的 methodDefinition，并设置 methodName targetBeanName属性
		RootBeanDefinition methodDefinition = new RootBeanDefinition(MethodLocatingFactoryBean.class);
		methodDefinition.getPropertyValues().add("targetBeanName", aspectName);
		methodDefinition.getPropertyValues().add("methodName", adviceElement.getAttribute("method"));
		methodDefinition.setSynthetic(true);

		// create instance factory definition
		// 创建切面类 FactoryBean
		RootBeanDefinition aspectFactoryDef =
				new RootBeanDefinition(SimpleBeanFactoryAwareAspectInstanceFactory.class);
		aspectFactoryDef.getPropertyValues().add("aspectBeanName", aspectName);
		aspectFactoryDef.setSynthetic(true);

		// register the pointcut
		// 选择合适的 advice 并注册 pointcut
		AbstractBeanDefinition adviceDef = createAdviceDefinition(
				adviceElement, parserContext, aspectName, order, methodDefinition, aspectFactoryDef,
				beanDefinitions, beanReferences);

		// configure the advisor
		// 配置 advisor，对应的类型是 AspectJPointcutAdvisor.class
		RootBeanDefinition advisorDefinition = new RootBeanDefinition(AspectJPointcutAdvisor.class);
		advisorDefinition.setSource(parserContext.extractSource(adviceElement));
		advisorDefinition.getConstructorArgumentValues().addGenericArgumentValue(adviceDef);
		if (aspectElement.hasAttribute(ORDER_PROPERTY)) {
			advisorDefinition.getPropertyValues().add(
					ORDER_PROPERTY, aspectElement.getAttribute(ORDER_PROPERTY));
		}

		// register the final advisor
		parserContext.getReaderContext().registerWithGeneratedName(advisorDefinition);

		return advisorDefinition;
	}
	finally {
		this.parseState.pop();
	}
}

```
**这里划一下上面代码提到的几个重要的类:**

* `Pointcut` 对应 `AspectJExpressionPointcut.class`
* `Advice` 对应 `AspectJMethodBeforeAdvice`,`AspectJAfterAdvice`， `AspectJAfterReturningAdvice`, `AspectJAfterThrowingAdvice`,`AspectJAroundAdvice`, 五种通知类型
* `Advisor` 对应 `AspectJPointcutAdvisor.class`


## 总结 

到这里，对`AOP`的配置信息已经解析完成，并成功的放入`Spring`容器中。看到这里也许大家对`aop`的实现还是很模糊，并没有一个整体的概念。那是因为我们对`Aspect`， `Pointcut` ，`Advice`，`Advisor`这几个aop的主要概念还没有一个系统的认识，下篇文章我将会对这几个概念好好的聊一下。



