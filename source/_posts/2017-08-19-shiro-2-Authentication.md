---
title: Shiro学习(2) —— 认证（Authentication）
toc: true
date: 2017-08-19 19:23:31
tags:
- Shiro
- Java Web
categories: 
- Web Framework
---

只要有用户参与的系统一般都要有权限管理，权限管理实现对用户访问系统的控制，按照安全规则或者安全策略控制用户可以访问而且只能访问自己被授权的资源。

权限管理包括用户认证和授权两部分。

本节将介绍Shiro关于认证(Authentication)部分的内容。

<!-- more -->

### 一、认证的流程

认证(Authentication)是指用户在访问系统时，系统对用户身份进行验证的过程。系统验证用户身份合法，用户方可访问系统的资源。

身份认证的主要流程如下所示：

![Procedure](/images/Shiro/2/AuthenticationProcedure.png)

1. 首先创建token，token中有用户提交的认证信息即principal和credential
2. 调用Subject.login(token)进行登录，其会自动委托给Security Manager，调用之前必须通过SecurityUtils. setSecurityManager()设置；
3. SecurityManager负责真正的身份验证逻辑；它会委托给Authenticator进行身份验证；
4. Authenticator才是真正的身份验证者，Shiro API中核心的身份认证入口点，此处可以自定义插入自己的实现；
5. Authenticator可能会委托给相应的AuthenticationStrategy进行多Realm身份验证，默认ModularRealmAuthenticator会根据AuthenticationStrategy进行多Realm身份验证；
6. Authenticator会把相应的token传入Realm，从Realm获取身份验证信息，如果没有返回/抛出异常表示身份验证失败了（详见下面注的代码中的assertCredentialsMatch()方法）。此处可以配置多个Realm，将按照相应的顺序及策略进行访问。


**注**：

其认证的具体流程如下：Authenticator的默认实现类ModularRealmAuthenticator调用Realm的实现类（Realm接口--> CachingRealm--> AuthenticatingRealm--> AuthorizingRealm，我们一般实现AuthorizingRealm抽象类来自定义Realm）来进行身份认证。其中AuthenticatingRealm中的getAuthenticationInfo()方法主要完成Credential验证的工作，如下图：

![AuthenticatingRealm](/images/Shiro/2/AuthenticatingRealm.png)

<br />



### 二、简单示例

首先，先在Maven工程中加入以下依赖：

``` xml
<!-- 单元测试 -->
<dependency>
  <groupId>junit</groupId>
  <artifactId>junit</artifactId>
  <version>4.12</version>
  <scope>test</scope>
</dependency>

<!-- 日志处理 -->
<dependency>
  <groupId>commons-logging</groupId>
  <artifactId>commons-logging</artifactId>
  <version>1.1.3</version>
</dependency>
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.7.25</version>
</dependency>

<!-- Shiro core -->
<dependency>
  <groupId>org.apache.shiro</groupId>
  <artifactId>shiro-core</artifactId>
  <version>1.2.3</version>
</dependency>
```

之后，创建.ini配置文件指定用户的principal和credential.

``` ini
[users]

zhang=111111
wang=123
```

最后，创建相应的测试样例。

``` Java
@Test
public void testFoo() {

    // 创建securityManager工厂，通过ini配置文件配置进行SecurityManager的初始化
    Factory<SecurityManager> factory = new IniSecurityManagerFactory(
    	"classpath:shiro-first.ini");

    // 获取SecurityManager
    SecurityManager securityManager = factory.getInstance();
    SecurityUtils.setSecurityManager(securityManager);

    // 创建与SecurityManager关联的Subject
    Subject subject = SecurityUtils.getSubject();

    // 在认证前准备好token
    UsernamePasswordToken token = new UsernamePasswordToken("zhangsan", "111111");

    try {
    	//执行认证提交
    	subject.login(token);
    } catch (AuthenticationException e) {
    	e.printStackTrace();
    }

    // 是否认证通过
    boolean isAuthenticated =  subject.isAuthenticated();

    System.out.println("是否认证通过：" + isAuthenticated);

    // 退出操作
    subject.logout();

    // 是否认证通过
    isAuthenticated =  subject.isAuthenticated();
    System.out.println("是否认证通过：" + isAuthenticated);
}
```

得到的结果为认证通过，修改token中的密码将得到不通过的结果，抛出的异常为IncorrectCredentialException.