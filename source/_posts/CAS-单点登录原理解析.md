---
title: CAS 单点登录原理解析
tags:
  - SSO
categories:
  - 第三方框架
  - SSO
date: 2017-05-19 16:12:12
---

## 引言

CAS是耶鲁大学发起的一个开源单点登录项目，也是用的最为广泛的开源项目。对于学习SSO有非常好的参考价值

## 什么是单点登录

单点登录（Single Sign On），简称为 SSO。多用于多系统共存的环境下，用户在一处登录后，就不用在其他系统中登录，最简单的单点登入实现可以完全基于`cookie`，通过往浏览器写带有登入信息的token来达到多点登入的目的，有兴趣的可以自己动手实现一个

## CAS单点登入的实现

先盗个图 ：

![cas单点登录.png](http://upload-images.jianshu.io/upload_images/2717496-6247a78311962768.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**从结构上看，分成了三部分：CAS Client，CAS Server，浏览器**

CAS Client 与受保护的客户端应用部署在一起，以 Filter 方式保护 Web 应用的受保护资源，过滤从客户端过来的每一个Web请求

下面我将分为两部分（用户第一次登录验证和之后的登录验证）详细解析其登入原理

### 用户第一次登录过程原理

1. `CAS Client` 会先根据`JSESSIONID`（存在浏览器cookie中） 来判断当前访问用户是否已经登录，如果没有`JSESSIONID` 会继续分析 HTTP 请求中是否包含请求 `Service Ticket`( ST 上图中的 Ticket) ，如果没有，则说明该用户是没有经过认证的；于是 `CAS Client` 会重定向用户请求到 `CAS Server` ，并传递 `Service` （要访问的目的资源地址）
2. 这时还没有登录过`CAS Server`，进入用户认证过程（返回登入界面） 
3. 如果用户认证成功，`CAS Server` 将随机产生一个相当长度、唯一、不可伪造的 `Service Ticket` ，并缓存以待将来验证，并且重定向用户到 Service 所在地址（附带刚才产生的 Service Ticket ） , 并为客户端浏览器设置一个`Ticket Granted Cookie`（ TGC，该Cookie设置的是`SSO Server`的域名）
4. `CAS Client` 在拿到 `Service` 和新产生的 `Ticket` 过后，与 `CAS Server` 通过拿到的`Ticket`进行身份核实，以确保 `Service Ticket` 的合法性。

以上是第一次验证时基本流程

### 当第二次验证时分两种情况

1. 同一个服务再次登录
这里浏览器`cookie`中会有 `sessionid`，凭借这个`sessionid`就不再需要去`CAS Server` 继续认证
2. 另外一个未登录过的服务登录
在上面步骤中的第二歩，因为之前已经在`CAS Server` 登录成功过，`CAS Server` 会读取浏览器中的 `TGC` （cookie），表明已经登入过，就不需要去返回登录界面了，直接返回CAS Client 一个唯一 票据（ticket），并进入第三歩。这样就实现了单点登录，多点无需再次手动登录


## 关于跨域的实现

跨域的关键在于浏览器端的Cookie:`TGC`，因为这个Cookie是设置在`SSO Server`端的，也就保证了统一票据和`Client`端域名无关。

## 总结

cas 单点登录核心就是 `单个cookie`， `N个session`

* 单个`cookie（TGC）`
在cas为各应用登录时使用，实现了只须一次录入用户密码，此`cookie`只于`cas server`相关，这也是`cas`可以实现跨域的关键
* N个`session`
各应用会创建自己的`session`表示是否登录，如已登录，就不会去CAS Server 去验证`TGC`

在该协议中，所有与 CAS Server 的交互均采用 `SSL` 协议，以确保 `ST` 和 `TGC` 的安全性。
