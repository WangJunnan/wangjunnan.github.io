---
title: Spring 源码解析三（BeanDefinitions的载入注册）
tags:
  - Spring
categories:
  - java框架
  - Spring
date: 2018-10-20 14:12:47
---


## 引言

> 这期主要是解析`BeanDefinitions`的载入和注册的整个过程，整个过程看似非常复杂，其实我们可以分成几个部分去看: 1.载入 2.注册和解析
> 如果没有看过前面几期的话，建议先过一下前面几期

* 为了帮助大家理清整个调用过程，简单的画了一个图
![image.png](http://upload-images.jianshu.io/upload_images/2717496-bc31933caf9f1fb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* 关键类
	* `XmlBeanDefinitionReader` `AbstractBeanDefinitionReader` xml配置形式的BeanDefinition的读取器
	* `XmlReaderContext` 解析过程的上下文对象，实际持有`BeanFactory`对象用于`BeanDefinitions`的回调注册
	* `DefaultBeanDefinitionDocumentReader` 解析xml文件抽象成Docment对象
	* `BeanDefinitionParserDelegate` 主要解析过程
	* `DefaultNamespaceHandlerResolver` 自定义配置文件解析器加载器

## BeanDefinitions的载入
	
### loadBeanDefinitions(String... locations)

* 这个方法是比较简单在`AbstractBeanDefinitionReader`中，只是提供了一个入口

```java
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
    Assert.notNull(locations, "Location array must not be null");
    int counter = 0;
    String[] var3 = locations;
    int var4 = locations.length;

    for(int var5 = 0; var5 < var4; ++var5) {
        String location = var3[var5];
        // 开始载入配置信息
        counter += this.loadBeanDefinitions(location);
    }

    return counter;
}
```

### loadBeanDefinitions(String location, Set<Resource> actualResources)

* 这个方法同样在`AbstractBeanDefinitionReader`中被调用，看代码可知就是获取持有的`ResourceLoader`对象并调用`getResources`方法。返回可以被`BeanDefinitionReader`解析的`Resource`类型。 *这里要注意持有的`ResourceLoader`对象是在创建`XmlBeanDefinitionReader`时所传入的容器本身*

```java
public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
	  // 获取 ResourceLoader
    ResourceLoader resourceLoader = this.getResourceLoader();
    if(resourceLoader == null) {
        throw new BeanDefinitionStoreException("Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
    } else {
        int loadCount;
        // 判断 resourceLoader 的类型是否属于ResourcePatternResolver类型
        if(!(resourceLoader instanceof ResourcePatternResolver)) {
            Resource resource = resourceLoader.getResource(location);
            // 调用子类的 loadBeanDefinitions(resource) 方法
            loadCount = this.loadBeanDefinitions((Resource)resource);
            if(actualResources != null) {
                actualResources.add(resource);
            }

            if(this.logger.isDebugEnabled()) {
                this.logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
            }

            return loadCount;
        } else {
            try {
                Resource[] resources = ((ResourcePatternResolver)resourceLoader).getResources(location);
                // 循环遍历载入
                loadCount = this.loadBeanDefinitions(resources);
                if(actualResources != null) {
                    Resource[] var6 = resources;
                    int var7 = resources.length;

                    for(int var8 = 0; var8 < var7; ++var8) {
                        Resource resource = var6[var8];
                        actualResources.add(resource);
                    }
                }

                if(this.logger.isDebugEnabled()) {
                    this.logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
                }

                return loadCount;
            } catch (IOException var10) {
                throw new BeanDefinitionStoreException("Could not resolve bean definition resource pattern [" + location + "]", var10);
            }
        }
    }
}
```

### loadBeanDefinitions(EncodedResource encodedResource)

* 获取到XML文件的inputStream流

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
	Assert.notNull(encodedResource, "EncodedResource must not be null");
	if (logger.isInfoEnabled()) {
		logger.info("Loading XML bean definitions from " + encodedResource.getResource());
	}

	Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
	if (currentResources == null) {
		currentResources = new HashSet<EncodedResource>(4);
		this.resourcesCurrentlyBeingLoaded.set(currentResources);
	}
	if (!currentResources.add(encodedResource)) {
		throw new BeanDefinitionStoreException(
				"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
	}
	try {
	// 获取到XML文件的inputStream流
		InputStream inputStream = encodedResource.getResource().getInputStream();
		try {
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			// 进行具体的xml文件流解析
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		}
		finally {
			inputStream.close();
		}
	}
	catch (IOException ex) {
		throw new BeanDefinitionStoreException(
				"IOException parsing XML document from " + encodedResource.getResource(), ex);
	}
	finally {
		currentResources.remove(encodedResource);
		if (currentResources.isEmpty()) {
			this.resourcesCurrentlyBeingLoaded.remove();
		}
	}
}
```

#### doLoadBeanDefinitions(InputSource inputSource, Resource resource)

* `doLoadDocument(inputSource, resource)` 解析返回`Document`对象
* `registerBeanDefinitions(doc, resource)` 使用 开始注册 BeanDefinitions 的过程

```java
class = org.springframework.beans.factory.xml.XmlBeanDefinitionReader

protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
		throws BeanDefinitionStoreException {
	try {
		// 解析xml并返回Document对象，这里解析由DefaultDocumentLoader完成
		Document doc = doLoadDocument(inputSource, resource);
		// 解析 Document对象并注册
		return registerBeanDefinitions(doc, resource);
	}
	catch (BeanDefinitionStoreException ex) {
		throw ex;
	}
	catch (SAXParseException ex) {
		throw new XmlBeanDefinitionStoreException(resource.getDescription(),
				"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
	}
	catch (SAXException ex) {
		throw new XmlBeanDefinitionStoreException(resource.getDescription(),
				"XML document from " + resource + " is invalid", ex);
	}
	catch (ParserConfigurationException ex) {
		throw new BeanDefinitionStoreException(resource.getDescription(),
				"Parser configuration exception parsing XML from " + resource, ex);
	}
	catch (IOException ex) {
		throw new BeanDefinitionStoreException(resource.getDescription(),
				"IOException parsing XML document from " + resource, ex);
	}
	catch (Throwable ex) {
		throw new BeanDefinitionStoreException(resource.getDescription(),
				"Unexpected exception parsing XML document from " + resource, ex);
	}
}
```

## BeanDefinitions的解析与注册

### registerBeanDefinitions(doc, resource)

* 创建 `DefaultBeanDefinitionDocumentReader`，解析xml资源文件
* 创建 `XmlReaderContext` 上下文对象，持有资源本身，持有`DefaultNamespaceHandlerResolver`对象，用来加载解析凭据（要怎么解析由解析凭据（NamespaceHandler）决定）

```java
class = org.springframework.beans.factory.xml.XmlBeanDefinitionReader

public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	// 创建 DefaultBeanDefinitionDocumentReader 解析xml资源文件
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	int countBefore = getRegistry().getBeanDefinitionCount();
	// 具体的解析开始
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	return getRegistry().getBeanDefinitionCount() - countBefore;
}

/**
 * 创建上下文对象
 */
public XmlReaderContext createReaderContext(Resource resource) {
	// 创建上下文对象 持有XmlBeanDefinitionReader 对象
	return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
			this.sourceExtractor, this, getNamespaceHandlerResolver());
}

/**
 * 创建XMl命名空间解析代理类，解析XML文件的依据。可以解析自定义XML文件
 */
public NamespaceHandlerResolver getNamespaceHandlerResolver() {
	if (this.namespaceHandlerResolver == null) {
		this.namespaceHandlerResolver = createDefaultNamespaceHandlerResolver();
	}
	return this.namespaceHandlerResolver;
}

/**
 * 创建默认的命名空间解析代理类
 */
protected NamespaceHandlerResolver createDefaultNamespaceHandlerResolver() {
	return new DefaultNamespaceHandlerResolver(getResourceLoader().getClassLoader());
}

```

### doRegisterBeanDefinitions(Element root)

* 创建`BeanDefinitionParserDelegate` 解析器，并加载默认配置（）解析器持有上下文对象

```java
class = org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader

public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    this.logger.debug("Loading bean definitions");
    Element root = doc.getDocumentElement();
    // 解析根节点
    this.doRegisterBeanDefinitions(root);
}

protected void doRegisterBeanDefinitions(Element root) {
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = this.createDelegate(this.getReaderContext(), root, parent);
    // 判断节点所属命名空间是否为spring默认(http://www.springframework.org/schema/beans)
    if(this.delegate.isDefaultNamespace(root)) {
        String profileSpec = root.getAttribute("profile");
        if(StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(profileSpec, ",; ");
            if(!this.getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                if(this.logger.isInfoEnabled()) {
                    this.logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec + "] not matching: " + this.getReaderContext().getResource());
                }

                return;
            }
        }
    }
	 // 空实现
    this.preProcessXml(root);
    // 开始解析根节点
    this.parseBeanDefinitions(root, this.delegate);
    // 空实现
    this.postProcessXml(root);
    // 赋值生成的父对象。含有默认配置
    this.delegate = parent;
}
```

### parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate)

* 判断是否为Spring默认的命名空间配置，加载不同的解析方法。默认命名空间会执行`parseDefaultElement`方法，如果是用户自定义的则会执行 `BeanDefinitionParserDelegate`的 `parseCustomElement`方法

```java
class = org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader

protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	  // 判断是否为默认命名空间
    if(delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();

        for(int i = 0; i < nl.getLength(); ++i) {
            Node node = nl.item(i);
            if(node instanceof Element) {
                Element ele = (Element)node;
                if(delegate.isDefaultNamespace(ele)) {
                		// 默认解析
                    this.parseDefaultElement(ele, delegate);
                } else {
                		// 执行自定义解析方法
                    delegate.parseCustomElement(ele);
                }
            }
        }
    } else {
    		// 执行自定义解析方法
        delegate.parseCustomElement(root);
    }

}
```

### 默认解析 

#### parseDefaultElement

* spirng默认的解析规则，可以看到这里有很多我们熟悉的标签元素，比如`bean`, `import`等，对应这些标签，spring都有专门的解析规则。`bean` 标签的解析就由`processBeanDefinition`来完成解析过程，这里我们着重来看下`bean`的解析

```java
class = org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader

private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		// 解析import标签
    if(delegate.nodeNameEquals(ele, "import")) {
        this.importBeanDefinitionResource(ele);
    } else if(delegate.nodeNameEquals(ele, "alias")) {
    		// 解析alias标签
        this.processAliasRegistration(ele);
    } else if(delegate.nodeNameEquals(ele, "bean")) {
    		// 解析bean标签
        this.processBeanDefinition(ele, delegate);
    } else if(delegate.nodeNameEquals(ele, "beans")) {
    		// 解析beans标签
        this.doRegisterBeanDefinitions(ele);
    }

}
```

#### processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate)

* 解析`BeanDefinition`并设置到`BeanDefinitionHolder`中去
* 注册到beanFactory的BeanDefiniton容器中去(其实是一个map)

```java
class = org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader

protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 解析并包装成BeanDefinitionHolder
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if(bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);

        try {
        		// 向容器注册 BeanDefiniton，XmlReaderContext其实是持有容器的引用的
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, this.getReaderContext().getRegistry());
        } catch (BeanDefinitionStoreException var5) {
            this.getReaderContext().error("Failed to register bean definition with name '" + bdHolder.getBeanName() + "'", ele, var5);
        }

        this.getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }

}
```

#### parseBeanDefinitionElement

* 感兴趣的同学也可以继续把parseBeanDefinitionElement看一遍，可以发现大多数都是与Bean定义相关的标签解析


### 自定义解析 parseCustomElement

* 从上下文获取 `DefaultNamespaceHandlerResolver` 加载自定义解析凭据类(`NamespaceHandler`)，`DefaultNamespaceHandlerResolver`类通过查看源码可知其会去所有`META-INF/spring.handlers`目录下加载`spring.handlers`文件，`spring.handlers`配置的是对应的不同命名空间的解析类
* 调用`NamespaceHandler`的`parse`解析自定义xml配置
* 看到这里其实我们知道`spring`在这里是留了一个扩展点的，可以通过自定义配置文件来进行自定义解析

```java
class=org.springframework.beans.factory.xml.BeanDefinitionParserDelegate

public BeanDefinition parseCustomElement(Element ele) {
    return this.parseCustomElement(ele, (BeanDefinition)null);
}

public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
    // 获取所属的命名空间地址
    String namespaceUri = this.getNamespaceURI(ele);
    // 上下文中获取DefaultNamespaceHandlerResolver 并根据命名空间地址获取对应的 Handler
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    if(handler == null) {
        this.error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
        return null;
    } else {
    		// 执行解析（this.readerContext 中持有BeanFactory注册对象，可以回调注册BeanDefinition信息）
        return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
    }
}
```


## 尾言

> `BeanDefinitions` 到这里为止就被注册完成了，下期就是`Spring`重要的依赖注入了


