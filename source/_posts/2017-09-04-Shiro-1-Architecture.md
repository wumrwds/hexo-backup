---
title: Shiro学习(1) —— Shiro的架构
toc: true
date: 2017-09-04 16:48:58
tags: 
- Shiro
categories: 
- Framework
---

Apache Shiro是Java的一个开源安全框架，其主要提供认证(Authentication)， 授权(Authorization)， 加密(Cryptography)， 会话管理(Session Management)等功能。

本节将简单介绍Shiro的主要功能及架构。

<!-- more -->

### 一、 Shiro的功能模块 ###

![Shiro module](/images/Shiro/1/shiro_module.png "Shiro 结构图")

**Authentication**：认证，验证用户是不是拥有相应的身份；

**Authorization**：授权，即权限验证，验证某个已认证的用户是否拥有某个权限；

**Session Manager**：会话管理，即用户登录后就是一次会话，在没有退出之前，它的所有信息都在会话中；会话可以是普通JavaSE环境的，也可以是如Web环境的；

**Cryptography**：加密，保护数据的安全性，如密码加密存储到数据库，而不是明文存储；

**Web Support**：Web支持，可以非常容易的集成到Web环境；

**Caching**：缓存，比如用户登录后，其用户信息、拥有的角色/权限不必每次去查，这样可以提高效率；

**Concurrency**：shiro支持多线程应用的并发验证，即如在一个线程中开启另一个线程，能把权限自动传播过去；

**Testing**：提供测试支持；

**Run As**：允许一个用户假装为另一个用户（如果他们允许）的身份进行访问；

**Remember Me**：记住我，这个是非常常见的功能，即一次登录后，下次再来的话不用登录了。

<br />

### 二、 Shiro架构 ###

从外部应用的角度观察Shiro的工作流程，将如下图所示：

![Shiro Frame 1](/images/Shiro/1/shiro_frame_01.png "从外部看Shiro架构")

这里可以看出外部程序主要通过Subject与Shiro框架的核心SecurityManger进行交互，而SecurityManager又通过Realm去获取相关的安全数据（principal、credential等）。



而从整体来看Shiro框架，其内部结构如下所示：

![Shiro Frame 2](/images/Shiro/1/shiro_frame_02.png "从内部看Shiro架构")

<br />

### 三、关键词的说明 ###

**Subject**：主体。外部程序要与 `SecurityManager` 交互，必须通过`Subject`来进行。`Subject`是一个抽象的概念，它代表了一个接入的外部程序用户的状态及可进行的一些安全操作（Authentication、Authorization、Session Access等） 。

**SecurityManager**：安全管理器，Shiro核心，有点类似于 `DispatcherServlet` 在Spring MVC中的位置，管理所有`Subject`，负责前后的沟通。`Subject`进行认证和授权都 是通过`SecurityManager`进行

**Realm**：域，类似安全数据库。Shiro从`Realm`获取安全数据（用户、密码、权限等）并返回验证信息 `AuthenticationInfo` 。在`Realm`中将存储授权和认证的逻辑。

**Authenticator**：认证器，主体进行认证最终通过`Authenticator`进行的。

**Authorizer**：授权器，主体进行授权最终通过`Authorizer`进行的。

**SessionManager**：会话管理器，web应用中一般是由web容器对session进行管理，但是Shiro也提供一套session管理的方式。Shiro有一个默认的`DefaultWebSessionManager`可供使用，使用时将其注入`SecurityManager`即可。

**SessionDao**： 通过`SessionDao`管理session数据，针对个性化的session数据存储需要使用`sessionDao`。

**CacheManager**：缓存管理器，主要对session和授权数据进行缓存，比如将授权数据通过`CacheManager`进行缓存管理，和ehcache整合对缓存数据进行管理。

**Cryptography**：加密，提供了一套加密/解密的组件，方便开发。比如提供常用的散列、加/解密等功能。比如md5散列算法等。

<br />

参考：[http://jinnianshilongnian.iteye.com/blog/2018936/](http://jinnianshilongnian.iteye.com/blog/2018936/)
