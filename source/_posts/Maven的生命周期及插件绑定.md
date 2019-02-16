---
title: Maven的生命周期及插件绑定
tags:
  - maven
categories:
  - 构建工具
  - maven
date: 2016-12-21 16:17:30
---

######何为生命周期？

一个完整的项目构建过程通常包括清理、编译、测试、打包、集成测试、验证、部署等步骤，Maven从中抽取了一套完善的、易扩展的生命周期。Maven的生命周期是抽象的，其中的具体任务都交由插件来完成。Maven为大多数构建任务编写并绑定了默认的插件，如针对编译的插件：maven-compiler-plugin。用户也可自行配置或编写插件。

***Maven的生命周期是抽象的，意味着生命周期本身不做任何实际工作，在Maven的设计中，实际的工作都交由插件来完成***


    Maven定义了三套生命周期：clean、default、site，每个生命周期都包含了一些阶段（phase）。三套生命周期相互独立，但各个生命周期中的phase却是有顺序的，且后面的phase依赖于前面的phase。执行某个phase时，其前面的phase会依顺序执行，但不会触发另外两套生命周期中的任何phase。

1 . 1 clean生命周期


![image.png](http://upload-images.jianshu.io/upload_images/2717496-45accb65ab14f44e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1 . 2 default生命周期


![image.png](http://upload-images.jianshu.io/upload_images/2717496-f73a6fdd0d3e10b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1 . 3 site生命周期


![image.png](http://upload-images.jianshu.io/upload_images/2717496-0a19f53d41fe0deb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

######（阶段）phase （目标）goal
Phase 之前说过，指的就是生命周期的各个阶段
Goal 指便是 插件的目标（通俗的说就是插件所具有的功能）


![image.png](http://upload-images.jianshu.io/upload_images/2717496-a72832b58a926875.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

######自定义插件配置
![image.png](http://upload-images.jianshu.io/upload_images/2717496-c7494e9f1be79261.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
