---
title: Dubbo(1) - 入门
toc: true
date: 2017-08-21 20:22:10
tags:
- Dubbo
- Java Web
categories:
- Web Framework
---

### 一、Dubbo的架构

![Dubbo Architecture](/images/Dubbo/1/dubbo-architecture.jpg)

**角色说明**：

- Provider: 暴露服务的服务提供方
- Consumer: 调用远程服务的服务消费方。
- Registry: 服务注册与发现的注册中心。
- Monitor: 统计服务的调用次调和调用时间的监控中心。
- Container: 服务运行容器。

<br/>

<!-- more -->

- **调用关系说明：**
- 服务容器负责启动，加载，运行服务提供者。
- 服务提供者在启动时，向注册中心注册自己提供的服务。
- 服务消费者在启动时，向注册中心订阅自己所需的服务。
- 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
- 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
- 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。

由以上结构可以看出Dubbo具有如下的几个特性：

(1) 连通性： 注册中心负责服务地址的注册与查找，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小 监控中心负责统计各服务调用次数，调用时间等，统计先在内存汇总后每分钟一次发送到监控中心服务器，并以报表展示 服务提供者向注册中心注册其提供的服务，并汇报调用时间到监控中心，此时间不包含网络开销 服务消费者向注册中心获取服务提供者地址列表，并根据负载算法直接调用提供者，同时汇报调用时间到监控中心，此时间包含网络开销 注册中心，服务提供者，服务消费者三者之间均为长连接，监控中心除外 注册中心通过长连接感知服务提供者的存在，服务提供者宕机，注册中心将立即推送事件通知消费者 注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表 注册中心和监控中心都是可选的，服务消费者可以直连服务提供者

(2) 健状性： 监控中心宕掉不影响使用，只是丢失部分采样数据 数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务 注册中心对等集群，任意一台宕掉后，将自动切换到另一台 注册中心全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯 服务提供者无状态，任意一台宕掉后，不影响使用 服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复

(3) 伸缩性： 注册中心为对等集群，可动态增加机器部署实例，所有客户端将自动发现新的注册中心 服务提供者无状态，可动态增加机器部署实例，注册中心将推送新的服务提供者信息给消费者

(4) 升级性： 当服务集群规模进一步扩大，带动IT治理结构进一步升级，需要实现动态部署，进行流动计算，现有分布式服务架构不会带来阻力：

![Dubbo Architucture Future](/images/Dubbo/1/dubbo-architecture-future.jpg)

- Deployer: 自动部署服务的本地代理。
- Repository: 仓库用于存储服务应用发布包。
- Scheduler: 调度中心基于访问压力自动增减服务提供者。
- Admin: 统一管理控制台。

<br/>

### 二、入门demo

#### dubbo-demo

首先创建一个maven工程dubbo-demo，packaging方式为pom. 之后导入Spring包、Dubbo包及日志包。

pom.xml文件如下：

``` xml
<properties>
  <maven.compiler.source>1.8</maven.compiler.source>
  <maven.compiler.target>1.8</maven.compiler.target>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
  <!-- spring版本号 -->
  <spring.version>4.2.5.RELEASE</spring.version>
  <slf4j.version>1.7.18</slf4j.version>
  <log4j.version>1.2.17</log4j.version>
</properties>
<dependencies>
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.11</version><!-- 指定范围，在测试时才会加载 -->
    <scope>test</scope>
  </dependency><!-- 添加spring核心依赖 -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>${spring.version}</version>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>${spring.version}</version>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
    <version>${spring.version}</version>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>${spring.version}</version>
  </dependency>
  <dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>${log4j.version}</version>
  </dependency>
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>${slf4j.version}</version>
  </dependency>
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>${slf4j.version}</version>
  </dependency><!-- log end -->
  <dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.9</version>
  </dependency><!-- dubbo -->
  <dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>dubbo</artifactId>
    <version>2.5.3</version>
    <exclusions>
      <exclusion>
        <groupId>org.springframework</groupId>
        <artifactId>spring</artifactId>
      </exclusion>
    </exclusions>
  </dependency>
  <dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.10</version>
  </dependency>
</dependencies>
```

而后，创建分别创建三个maven module： dubbo-api、dubbo-consumer、dubbo-provider.

其中dubbo-api用于定义service接口，dubbo-provider实现具体的service业务，供给dubbo-consumer调用。

#### dubbo-api

dubbo-api主要用于定义service接口。

``` Java

```



#### dubbo-provider

dubbo-provider主要实现具体的service业务。



```java

```



### dubbo-consumer

dubbo-consumer主要调用dubbo-provider提供的服务。



```java

```



