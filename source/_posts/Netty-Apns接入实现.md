---
title: Netty-Apns接入实现
date: 2017-10-24 00:00:39
tags: [Netty,Apns,java]
---
极光推送免费版每分钟600次的请求限制实在是把我恶心坏了，考虑到现在我们 Android 的推送已经全量接入了小米，所以接下来就是要把 iOS 的推送直接接入 APNS 这样就可以彻底摆脱极光的推送。不再受这个600次/分钟的限制了。APNS使用 HTTP2 协议进行通信所以自然就想到了使用Netty作为网络框架，进行开发。下面逐个给大家介绍使用 Netty 接入 APNS 的注意事项和接入的时候踩到的坑。

 <!--more-->

## APNS
APNS 是 Apple 提供的推送服务。[官方文档](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html#//apple_ref/doc/uid/TP40008194-CH8-SW1)
下面说一说自己对接入APNS的时候遇到的坑：
1. APNS 使用 HTTP/2 协议通信，必须使用 TLS 1.2 或以上的加密方式通信，如果使用JDK提供的加密方法，如果使用 JDK7 ，需要对 JVM 的启动参数进行设置，另一个比较简单的解决方法，就是使用 JDK 8 。JDK 8 就能很好的解决加密的问题，或者调用系统的Openssl 进行加解密。具体看大家的线上环境决定。
2. 在开发的时候还遇到一个比较奇葩的问题，就是使用开发证书可以推送到达，但是切换到线上后发现怎么推送都不能到达，后来仔细读了官方文档，发现在请求头中有一个 `apns-topic` 字段，如果你的推送证书中包括了不止一个应用，这个字段就是一个必填字段，且为应用的 `bundle ID`。
3. 由于使用了 HTTP/2 协议，APPLE 推荐尽量复用链接，因为使用了 TLS 加密，每次建立连接的握手会消耗大量的时间。同时还可以使用多个连接提升推送的效率，所以之后的实现中，我使用 `Common pool2` 作为连接池来提高推送的效率。可以使用 PING 来检查连接是否有效。所以在实现的时候使用了 Netty 的 `IdleStateEvent` 来检查连接。

## 代码的具体实现：
首先看看代码的结构：
![屏幕快照 2017-05-14 下午11.10.10](http://7u2r32.com1.z0.glb.clouddn.com/屏幕快照 2017-05-14 下午11.10.10.png)

Module 中的 Payload 和 PsuhsNotifcation 的是对 APNS 的数据结构的封装，具体可以参考 Apple 提供的文档
ApnsConfig 是对整个推送系统的相关设置，提供了一些默认参数。可以根据需要自己进行设置。
PingMessage 从名字就能看出，是对链接进行检测的 `PING frame`。

下面着重介绍一下Service相关的实现思路。

## 具体实现
 Netty 的 Client 的代码我参考了 Netty官方提供的 [Example](https://netty.io/4.1/xref/io/netty/example/http2/helloworld/client/package-summary.html)。
 

1. `ApnsConnection` 实现了 `Connection` 接口，主要负责维护和 APNS 的链接。
 ![屏幕快照 2017-05-14 下午11.20.37](http://7u2r32.com1.z0.glb.clouddn.com/屏幕快照 2017-05-14 下午11.20.37.png)
 
2. 接池的实现：

链接池相关的代码主要在 `ApnsConnectionPool` 中。整个链接池的使用了 `Commone Pool2`作为底层实现，实现的时候主要参考了`jedies`的链接池的实现。

3.`NettyApnsService` 向外暴露了推送的接口。直接调用`sendNotification()`方法就能推送消息了。

![屏幕快照 2017-05-14 下午11.26.31](http://7u2r32.com1.z0.glb.clouddn.com/屏幕快照 2017-05-14 下午11.26.31.png)

   
## 总结
代码地址：https://github.com/diaozxin007/Netty-apns

欢迎大家拍砖指点。

看开源的代码是快速学习的方法，在整个项目中，参考了很多开源的工程的具体实现。希望对大家有所帮助。
