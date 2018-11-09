---
title: '关于xml文件 xsi:schemaLocation'
tags:
  - xml
categories:
  - 基础
  - xml
date: 2017-05-09 15:39:28
---

相信很多人对xml 头上一大堆得东西都是拿来主义，copy过来就行了，并不理解那是什么意思

先来一段
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
  xmlns:context="http://www.springframework.org/schema/context"
  xmlns:mvc="http://www.springframework.org/schema/mvc"
  xsi:schemaLocation="http://www.springframework.org/schema/beans  
  http://www.springframework.org/schema/beans/spring-beans-3.1.xsd  
  http://www.springframework.org/schema/context  
  http://www.springframework.org/schema/context/spring-context-3.1.xsd  
  http://www.springframework.org/schema/mvc  
  http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd">

  <context:component-scan base-package="com.pikaq"/>
  <bean id="xxx" class="xxx.xxx.xxx.Xxx">
  <property name="xxx" value="xxxx"/>
</bean>
</beans>
```

首先看到的就是  `xmlns`， `xmlnsXML` 是Namespace的缩写，可译为`“XML命名空间”`
##### 为什么需要xmlns？
因为xml文件有成千上万，谁也不能保证你的标签是独一无二的，总是会冲突的，这时就需要xmlns了！

##### 怎么使用xmlns 呢？

使用语法： `xmlns:namespace-prefix="namespaceURI"`。其中namespace-prefix为自定义前缀，只要在这个XML文档中保证前缀不重复即可；namespaceURI是这个前缀对应的XML Namespace的定义。例如：
>xmlns:context="http://www.springframework.org/schema/context"

这里的<component-scan/>元素就来自别名为context的XML Namespace，也就是在http://www.springframework.org/schema/context中定义的。
其实我们完全可以将前缀定义为abc：
>xmlns:abc="http://www.springframework.org/schema/context"

好了，看到这里，你也许会问 那  xmlns 和xmlns:context 有什么区别呢？

xmlns 没有带别名，就是表示那是默认的，如 
><bean id="xxx" class="xxx.xxx.xxx.Xxx">
  <property name="xxx" value="xxxx"/>
</bean>

这里的bean 属性就出自这个默认命名空间

##### xsi:schemaLocation 是干嘛的？
看到这里也许你已经知道了它是干嘛的了
schemaLocation不就是 xsi 命名空间的一个属性吗，如果之前我们把
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 的别名改成
xmlns:sb="http://www.w3.org/2001/XMLSchema-instance"

这里其实就变成 sb:schemaLocation，这里讲一下这个属性是干嘛的，这个属性的值由一个或多个URI引用对组成，两个URI之间以空白符分隔（空格和换行均可）
它定义了XML Namespace和对应的 XSD（Xml Schema Definition）文档的位置的关系，意思就是 这个命名空间对应的具体模板是哪个

例如我们打开 http://www.springframework.org/schema/mvc/ 这个 命名空间，可以看到有很多选择
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-cd92bf94a507aaaf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`xsi:schemaLocation` 这个属性就是跟他说我要选择哪一个
