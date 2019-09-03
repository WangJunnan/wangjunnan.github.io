---
title: 手写一个starter
tags:
  - Spring
  - SpringBoot
categories:
  - Spring
  - SpringBoot
date: 2019-08-28 16:47:15
---

前文也说过，springboot最大的优势在于自动配置，我么只需要引入一个个第三方提供的starter就可以快速集成第三方功能，既然starter这么方便，我们自己应该也可以写一个。

## 快速写一个starter

- 新建一个maven项目

```json
<groupId>com.walm</groupId>
    <artifactId>boot-test-starter</artifactId>
    <version>1.0-SNAPSHOT</version>


    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
            <version>2.0.6.RELEASE</version>
        </dependency>
    </dependencies>
```

- 新建一个config类

```java
@Configuration
public class HelloWorldConfig {

    @Bean
    public HelloWorld HelloWord() {
        return new HelloWorld();
    }
}
```

- 新建一个提供服务的service类

```java
public class HelloWorld {

    public void sayHello() {
        System.out.println("hello world !!!!");
    }
}
```

- 最重要的是一步，需要让spring能够扫描到我们的 config HelloWorldConfig 类
  - 在resource 文件夹下新建一个 META-INF 文件夹
  - 新建一个 spring.factories 文件



在spring.factories文件中配置我们的写的HelloWorldConfig类

```json
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.walm.boottest.HelloWorldConfig
```

这样一个简单的 springboot  starter 就完成了，最后本地安装 install我们的starter jar包就可以在其他项目中引用了

- 引用starter，只需要在项目引用 该jar包即可

```java
<dependency>
            <groupId>com.walm</groupId>
            <artifactId>boot-test-starter</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
```


下面我们简单分析一下 stater的原理

## starter 自动配置原理

首先自动配置的原理关键一定是在 spring.factories 文件是如何被加载到的这个点上去探究

需要从spring boot的启动过程一步一步的去看了

首先，我们启动一个spring-boot项目，肯定会有一个配置注解 `SpringBootApplication` 那么这个注解到底起什么作用呢？

- SpringBootApplication.class

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class)
	String[] excludeName() default {};

	/**
	 * Base packages to scan for annotated components. Use {@link #scanBasePackageClasses}
	 * for a type-safe alternative to String-based package names.
	 * @return base packages to scan
	 * @since 1.3.0
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	/**
	 * Type-safe alternative to {@link #scanBasePackages} for specifying the packages to
	 * scan for annotated components. The package of each class specified will be scanned.
	 * <p>
	 * Consider creating a special no-op marker class or interface in each package that
	 * serves no purpose other than being referenced by this attribute.
	 * @return base packages to scan
	 * @since 1.3.0
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};

}
```

这里着重看 EnableAutoConfiguration 这个注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	String[] excludeName() default {};

}
```

可以看到 @Import(AutoConfigurationImportSelector.class)这个注解，这里就是我们自动配置的入口
注意，解析Import注解其实是通过 BeanFactoryPostProcessor 接口来扩展的

下面我们来看下AutoConfigurationImportSelector.class 类的实现，AutoConfigurationImportSelector 实现了ImportSelector接口

- ImportSelector

```java
public interface ImportSelector {

	/**
	 * Select and return the names of which class(es) should be imported based on
	 * the {@link AnnotationMetadata} of the importing @{@link Configuration} class.
	 */
	String[] selectImports(AnnotationMetadata importingClassMetadata);

}
```

在解析import注解的时候会调用指定ImportSelector 接口实现 的 selectImports方法。下面我们看下AutoConfigurationImportSelector的 selectImports方法实现

- selectImports

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 获取  META-INF/spring.factories  文件中的配置信息
		List<String> configurations = getCandidateConfigurations(annotationMetadata,
				attributes);
		configurations = removeDuplicates(configurations);
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
		configurations = filter(configurations, autoConfigurationMetadata);
    // 触发事件
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return StringUtils.toStringArray(configurations);
	}
```

## 总结
写一个starter还是非常简单的，核心就是要让springboot自动扫描到我们的 配置类，核心逻辑其实就是通过扫描我们的自定义的META-INF/spring.factories文件来进行自动配置

