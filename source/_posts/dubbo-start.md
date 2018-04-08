---
title: dubbo 源码学习（一）开篇
date: 2018-04-01 16:51:26
tags: [dubbo,rpc,java]
---

今天开始将开启 dubbo 的源码研究。

dubbo 是什么？

dubbo 是阿里巴巴开发的一个基于 java 的开源的 RPC 框架。所谓 RPC 指的的是 Remote Procedure Call Protocol 远程过程调用协议。

<!--more-->

# 阅读代码前的准备

1. 下载代码：

```git
git clone https://github.com/apache/incubator-dubbo.git
```

2. IDE 支持

```shell
mvn idea:idea
```

然后就可以自由的玩耍了。

# 架构

我们看代码包的结构：

![dubbo code](https://xilidou.oss-cn-beijing.aliyuncs.com/img/dubbo.jpeg)

* dubbo-common 公共逻辑模块：包括 Util 类和通用模型。
* dubbo-remoting 远程通讯模块：相当于 Dubbo 协议的实现，如果 RPC 用 RMI协议则不需要使用此包。
* dubbo-rpc 远程调用模块：抽象各种协议，以及动态代理，只包含一对一的调用，不关心集群的管理。
* dubbo-cluster 集群模块：将多个服务提供方伪装为一个提供方，包括：负载均衡, 容错，路由等，集群的地址列表可以是静态配置的，也可以是由注册中心下发。
* dubbo-registry 注册中心模块：基于注册中心下发地址的集群方式，以及对各种注册中心的抽象。
* dubbo-monitor 监控模块：统计服务调用次数，调用时间的，调用链跟踪的服务。
* dubbo-config 配置模块：是 Dubbo 对外的 API，用户通过 Config 使用D ubbo，隐藏 Dubbo 所有细节。
* dubbo-container 容器模块：是一个 Standlone 的容器，以简单的 Main 加载 Spring 启动，因为服务通常不需要 Tomcat/JBoss 等 Web 容器的特性，没必要用 Web 容器去加载服务。

# 依赖关系

这张图是从 dubbo 的官网上下载下来的：

![dubbo-architecture](https://xilidou.oss-cn-beijing.aliyuncs.com/img/dubbo-architecture.png)

顺着序号我们来看看 dubbo 的各个模块是怎么工作的。

名词解释：

* Container 服务容器，可以类比 tomcat 或者 jetty
* Provider 服务的提供方
* Consumer 服务的消费方，或者称为调用方
* Registry 注册中心，用于提供服务发现，注册等功能
* Monitor 监控方，用于监控整个集群的工作状态

所以按照序号我们看看 dubbo 各个模块都干什么了？

0. container 是 dubbo 运行的容器，容器启动以后会初始化服务的提供方（Provider）。
1. Provider 在启动成功以后，会向注册中心（Registry）告知，某ip，某端口，提供某服务。
2. Comsumer 启动以后会向注册中心订阅自己关心的服务的状态。
3. 服务中心会向 Comsumer 发送通知，告知它关心的服务的动向。
4. Comsumer 获取了服务提供方（Provider）的相关信息后，就会远程调用服务方提供的方法。完成远程调用。
5. Comsumer 和 Provider 会定时的上报自己运行的情况。

# 总结

以上就是对 dubbo 的代码结构和运行步骤的简单介绍。dubbo 的源码学习也就算打开了一个序幕。

下一篇文章就会从 dubbo-container 这个包开始逐步的介绍 dubbo 的源码实现。敬请期待。



