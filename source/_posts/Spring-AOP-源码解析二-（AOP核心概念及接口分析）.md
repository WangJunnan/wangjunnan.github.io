---
title: Spring AOP 源码解析二 （AOP核心概念及接口分析）
tags:
  - Spring
categories:
  - java框架
  - Spring
date: 2018-12-10 15:56:32
---

## 引言

上期对`Spring AOP`的配置入口做了一个很简单的源码解读，相当于一道开胃菜，有些概念也并没有细说。这期我们主要来理一下`AOP`的整体概念，以及相关的接口分析


## AOP概念

面向切面编程`AOP`其实是对`OOP`编程方式的一种补充，`OOP`中模块化的关键单元是类，而在`AOP`中，模块化单元则是`Aspect`，我们又把它叫做切面。对于`Spring IOC`来说，`Spring AOP`是对`IOC`的强有力增强。

接下来我们将对`AOP`的核心接口做详细的分析

## AOP核心接口

### Pointcut 切点

`切入点`，顾名思义，就是我们要切入某个点的具体位置，对于切入点的位置选择，我们常用`aspect表达式`来进行定位。我们先来看一下`Pointcut`接口，看看它定义了哪些东西

```java
public interface Pointcut {

	/**
	 * 返回一个类型过滤器
	 * @return the ClassFilter (never {@code null})
	 */
	ClassFilter getClassFilter();

	/**
	 * 返回一个方法匹配器
	 * @return the MethodMatcher (never {@code null})
	 */
	MethodMatcher getMethodMatcher();

	Pointcut TRUE = TruePointcut.INSTANCE;

}
```

可以看到`Pointcut`接口定义了两个方法，一个用于返回一个类型过滤器，一个返回了一个方法选择器，我们可以再具体的看一下这两个接口的内容

```java
public interface ClassFilter {

	/**
	 * 判断切入点于给定的接口或目标类是否类型匹配
	 */
	boolean matches(Class<?> clazz);

	ClassFilter TRUE = TrueClassFilter.INSTANCE;

}

public interface MethodMatcher {

	/**
	 * 检查是否有此方法的静态匹配
	 */
	boolean matches(Method method, Class<?> targetClass);

	/**
	 * 是否运行时
	 */
	boolean isRuntime();

	/**
	 * 检查是否有此方法的运行时(动态)匹配
	 */
	boolean matches(Method method, Class<?> targetClass, Object... args);

	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;

}
```

可以看到，上面两个接口的核心都是`matches`方法，都是用于判断目标类型或目标方法与此切入点是否匹配。
在上一篇文章中我们有提到一个`Pointcut`接口的具体实现，（`AspectJExpressionPointcut.class`），这个是我们最常用的`Pointcut`接口实现，里面封装了对`aspect表达式`用于匹配切入点方法的具体实现

### Advice 通知

`Advice 通知`，翻译成中文是`建议`的意思，在上面我们知道了`pointcut`是用来选择切入点的，那选定了切入点之后自然就是要执行具体织入的`AOP逻辑`，并且还要在一个合适的位置去执行，而这个合适的位置就是我们常说的**在目标方法前 目标方法后或则环绕通知执行**，好了，还是让我们看一下`Advice`接口，看看Spring为我们定义了哪几种通知类型


Advice 是一个空接口，具体的类型在实现中
```java
public interface Advice {

}
```
上篇文章有提到`Advice`的具体实现类型，这里我就直接把每种类型通知对应的实现类列出来了

1. 前置通知 `AspectJMethodBeforeAdvice.class` 在目标方法执行前执行
2. 后置通知 `AspectJAfterAdvice.class` 在目标方法执行后执行（无论是否抛异常都会执行）
3. 环绕通知 `AspectJAroundAdvice.class` 在目标方法执行前后执行
4. 异常通知 `AspectJAfterThrowingAdvice.class` 目标方法执行异常时执行
5. 后置正常返回通知 `AspectJAfterReturningAdvice.class` 目标方法正常返回结果后执行

### Aspect 切面

`Aspect切面`，但看这个单词其实不好理解，但是当我们知道了`Pointcut`和`Advice`的作用之后，理解切面就会清晰很多。简单的理解切面就是`Pointcut`和`Advice`两者的结合，把切入点和通知连接起来就组成了一个切面。而在`Spring`代码中的体现，切面指的就是`Advisor`接口及其实现，来看一下`Advisor`接口的内容

```java
public interface Advisor {
	Advice getAdvice();
	boolean isPerInstance();

}
```
`Advisor`接口`getAdvice()`方法返回了该切面对应的通知

```java
public interface PointcutAdvisor extends Advisor {

	/**
	 * Get the Pointcut that drives this advisor.
	 */
	Pointcut getPointcut();

}

```
`PointcutAdvisor`接口继承了`Advisor`接口`，并且新增了`getPointcut()`方法，通过这个接口，就把切入点和通知两者连接了起来，完成了切面的配置

### Weaving 织入

当完成了以上工作之后，我们可以说是万事俱备，只欠东风，剩下的事情，就是把具体的切面逻辑织入到我们的切入点中去，那么`Spring`对于`AOP`代理逻辑织入是怎么做的，又是在何时做的呢？

这时候就是`Spring IOC` 容器大展身手的时候，我们在之前分析`Spring IOC`容器的时候有提到过扩展点`BeanPostProcessor`接口，上篇文章我们也有提到一个代理创建的核心类`AspectJAwareAdvisorAutoProxyCreator`，我们来看一下这个类的结构

![AspectJAwareAdvisorAutoProxyCreator.class](https://upload-images.jianshu.io/upload_images/2717496-9f1ab92940b81d43.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过查看该类的接口，可以知道`AspectJAwareAdvisorAutoProxyCreator`是`BeanPostProcessor`接口的具体实现，所以我们知道代理类创建必定是在Bean创建前后被创建的。具体的创建逻辑是在Bean完成初始化，执行`BeanPostProcessor`接口的`postProcessAfterInitialization`方法时执行

## 总结

本篇文章我们主要从接口层面分析了`AOP`的实现，并没有涉及到很多的`AOP`源码，但是
这些基本的接口是`AOP`实现的核心，理解这些对于`AOP`源码分析有很大的帮助。下篇文章我将会对代理的创建(AspectJAwareAdvisorAutoProxyCreator.class)做分析



