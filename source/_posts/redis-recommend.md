---
title: Redis 命令的执行过程
date: 2018-03-30 12:45:46
tags: [redis]
---

原文地址：[https://www.xilidou.com/2018/03/30/redis-recommend/](https://www.xilidou.com/2018/03/30/redis-recommend/)

之前写了一系列文章，已经很深入的探讨了 Redis 的数据结构，数据库的实现，key的过期策略以及 Redis 是怎么处理事件的。所以距离 Redis 的单机实现只差最后一步了，就是 Redis 是怎么处理 client 发来的命令并返回结果的，所以我们就仔细讨论一下 Redis 是怎么执行命令的。

阅读这篇文章你将会了解到：

* Redis 是怎么执行远程客户端发来的命令的

<!--more-->

# Redis client（客户端）

Redis 是单线程应用，它是如何与多个客户端简历网络链接并处理命令的？
由于 Redis 是基于 I/O 多路复用技术，为了能够处理多个客户端的请求，Redis 在本地为每一个链接到 Redis 服务器的客户端创建了一个 redisClient 的数据结构，这个数据结构包含了每个客户端各自的状态和执行的命令。 Redis 服务器使用一个链表来维护多个 redisClient 数据结构。

在服务器端用一个链表来管理所有的 redisClient。

```c

struct redisServer {

    //...
    list *clients;              /* List of active clients */
    //...
}
```

所以我就看看 redisClient 包含的数据结构和重要参数：

```c
typedef struct redisClient {

    // 客户端状态标志
    int flags;              /* REDIS_SLAVE | REDIS_MONITOR | REDIS_MULTI ... */
    
    // 套接字描述符
    int fd;

    // 当前正在使用的数据库
    redisDb *db;

    // 当前正在使用的数据库的 id （号码）
    int dictid;

    // 客户端的名字
    robj *name;             /* As set by CLIENT SETNAME */

    // 查询缓冲区
    sds querybuf;

    // 查询缓冲区长度峰值
    size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size */

    // 参数数量
    int argc;

    // 参数对象数组
    robj **argv;

    // 记录被客户端执行的命令
    struct redisCommand *cmd, *lastcmd;

    // 请求的类型：内联命令还是多条命令
    int reqtype;

    // 剩余未读取的命令内容数量
    int multibulklen;       /* number of multi bulk arguments left to read */

    // 命令内容的长度
    long bulklen;           /* length of bulk argument in multi bulk request */

    // 回复链表
    list *reply;

    // 回复链表中对象的总大小
    unsigned long reply_bytes; /* Tot bytes of objects in reply list */

    // 已发送字节，处理 short write 用
    int sentlen;            /* Amount of bytes already sent in the current
                               buffer or object being sent. */

    // 回复偏移量
    int bufpos;
    // 回复缓冲区
    char buf[REDIS_REPLY_CHUNK_BYTES];
    // ...
}

```

这里需要特别的注意，redisClient 并非指远程的客户端，而是一个 Redis 服务本地的数据结构，我们可以理解这个 redisClient 是远程客户端的一个映射或者代理。

## flags

flags 表示了目前客户端的角色，以及目前所处的状态。他比较特殊可以单独表示一个状态或者多个状态。

## querybuf

querybuf 是一个 sds 动态字符串类型，所谓 buf 说明是它只是一个缓冲区，用于存储没有被解析的命令。 

## argc & argv 

上文的 querybuf 是一个没有处理过的命令，当 Redis 将 querybuf 命令解析以后，会将得出的参数个数和以及参数分别保存在 argc 和 argv 中。argv 是一个 redisObject 的数组。

## cmd

 Redis 使用一个字典保存了所有的 redisCommand。key 是 redisCommand 的名字，值就是一个 redisCommand 结构，这个结构保存了命令的实现函数，命令的标志，命令应该给定的参数个数，命令的执行次数和总消耗时长等统计信息，cmd 是一个 redisCommand。

当 Redis 解析出 argv 和 argc 后，会根据数组 argv[0]，到字典中查询出对应的 redisCommand。上文的例子中 Redis 就会去字典去查找 `SET` 这个命令对应的 redisCommand。redis 会执行 redisCommand 中命令的实现函数。

## buf & bufpos & reply

buf 是一个长度为 REDIS_REPLY_CHUNK_BYTES 的数组。Redis 执行相应的操作以后，就会将需要返回的返回的数据存储到 buf 中，bufpos 用于记录 buf 中已用的字节数数量，当需要恢复的数据大于 REDIS_REPLY_CHUNK_BYTES 时，redis 就会是用 reply 这个链表来保存数据。

## 其他参数

其他参数大家看注释就能明白，就是字面的意思。省略的参数基本上涉及 Redis 集群管理的参数，在之后的文章中会继续讲解。

## 客户端的链接和断开

上文说过 redisServer 是用一个链表来维护所有的 redisClient 状态，每当有一个客户端发起链接以后，就会在 Redis 中生成一个对应的 redisClient 数据结构，增加到`clients`这个链表之后。

一个客户端很可能被多种原因断开。

总体分为几种类型：

* 客户端主动退出或者被 kill。
* timeout 超时。
* Redis 为了自我保护，会断开发的数据超过限制大小的客户端。
* Redis 为了自我保护，会断需要返回的数据超过限制大小的客户端。

## 调用总结

当客户端和服务器端的嵌套字变得可读的时候，服务器将会调用命令请求处理器来执行以下操作：

1. 读取嵌套字中的数据，写入 querybuf。
2. 解析 querybuf 中的命令，记录到 argc 和 argv 中。
3. 根据 argv[0] 查找对应的 recommand。
4. 执行 recommand 对应的实现函数。
5. 执行以后将结果存入 buf & bufpos & reply 中，返回给调用方。
 

# Redis Server (服务端)

上文是从 redisClient 的角度来观察命令的执行，文章接下来的部分将会从 Redis 的代码层面，微观的观察 Redis 是怎么实现命令的执行的。


## redisServer 的启动

在了解redisServer 的工作机制的工作机制之前，需要了解 redisServer 的启动做了什么：

可以继续观察 Redis 的 main() 函数。

```c
int main(int argc, char **argv) {

    //...

    // 创建并初始化服务器数据结构
    initServer();

    //...
}
```

我们只关注 `initServer()` 这个函数，他负责初始化服务器的数据结构。继续跟踪代码:

```c

void initServer() {

    //...

    //创建eventLoop
    server.el = aeCreateEventLoop(server.maxclients+REDIS_EVENTLOOP_FDSET_INCR);

    /* Create an event handler for accepting new connections in TCP and Unix
     * domain sockets. */
    // 为 TCP 连接关联连接应答（accept）处理器
    // 用于接受并应答客户端的 connect() 调用
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                redisPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }

    // 为本地套接字关联应答处理器
    if (server.sofd > 0 && aeCreateFileEvent(server.el,server.sofd,AE_READABLE,
        acceptUnixHandler,NULL) == AE_ERR) redisPanic("Unrecoverable error creating server.sofd file event.");

    //...

}

```

篇幅限制，我们省略了很多与本编文章无关的代码，保留了核心逻辑代码。

在上一篇文章中 [《Redis 中的事件驱动模型》](https://www.xilidou.com/2018/03/22/redis-event/) 我们讲解过，redis 使用不同的事件处理器，处理不同的事件。

在这段代码里面：

* 初始化了事件处理器的 eventLoop
* 向 eventLoop 中注册了两个事件处理器 `acceptTcpHandler` 和 `acceptUnixHandler`，分别处理远程的链接和本地链接。

## redisClient 的创建

当有一个远程客户端连接到 Redis 的服务器，会触发 `acceptTcpHandler` 事件处理器.

`acceptTcpHandler` 事件处理器，会创建一个链接。然后继续调用 `acceptCommonHandler`。

`acceptCommonHandler` 事件处理器的作用是：

* 调用 `createClient()` 方法创建 redisClient
* 检查已经创建的 redisClient 是否超过 server 允许的数量的上限
* 如果超过上限就拒绝远程连接
* 否则创建 redisClient 创建成功
* 并更新连接的统计次数，更新 redisClinet 的 flags 字段

这个时候 Redis 在服务端创建了 redisClient 数据结构，这个时候远程的客户端就在 redisServer 中创建了一个代理。远程的客户端就与 Redis 服务器建立了联系，就可以向服务器发送命令了。

## 处理命令

在 `createClient()` 行数中：

```c
// 绑定读事件到事件 loop （开始接收命令请求）
if (aeCreateFileEvent(server.el,fd,AE_READABLE,readQueryFromClient, c) == AE_ERR)
```

向 eventLoop 中注册了 `readQueryFromClient`。  `readQueryFromClient` 的作用就是从client中读取客户端的查询缓冲区内容。

然后调用函数 `processInputBuffer` 来处理客户端的请求。在 `processInputBuffer` 中有几个核心函数：

* `processInlineBuffer` 和 `processMultibulkBuffer` 解析 querybuf 中的命令，记录到 argc 和 argv 中。
* `processCommand` 根据 argv[0] 查找对应的 recommen,执行 recommend 对应的执行函数。在执行之前还会验证命令的正确性。将结果存入 buf & bufpos & reply 中

## 返回数据

万事具备了，执行完了命令就需要把数据返回给远程的调用方。调用链如下

processCommand -> addReply -> prepareClientToWrite

在 `prepareClientToWrite` 中我们有见到了熟悉的代码：

```c
aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,sendReplyToClient, c) == AE_ERR) return REDIS_ERR;
```

向 eventloop 绑定了 `sendReplyToClient` 事件处理器。

在 `sendReplyToClient` 中观察代码发现，如果 bufpos 大于 0，将会把 buf 发送给远程的客户端，如果链表 reply 的长度大于0，就会将遍历链表 reply，发送给远程的客户端，这里需要注意的是，为了避免 reply 的节点特别多，如果一直会写就会过度的占用资源引起相应慢，会设置 REDIS_MAX_WRITE_PER_EVENT 。当写入的总数量大于 REDIS_MAX_WRITE_PER_EVENT ，临时中断写入，将处理时间让给其他客户端，剩余的内容等下次写入就绪再继续写入。这样的套路我们一路走来看过太多了。

# 总结

1. 远程客户端连接到 redis 后，redis服务端会为远程客户端创建一个 redisClient 作为代理。
2. redis 会读取嵌套字中的数据，写入 querybuf 中。
3. 解析 querybuf 中的命令，记录到 argc 和 argv 中。
4. 根据 argv[0] 查找对应的 recommend。
5. 执行 recommend 对应的执行函数。
6. 执行以后将结果存入 buf & bufpos & reply 中。
7. 返回给调用方。返回数据的时候，会控制写入数据量的大小，如果过大会分成若干次。保证 redis 的相应时间。

Redis 作为单线程应用，一直贯彻的思想就是，每个步骤的执行都有一个上限（包括执行时间的上限或者文件尺寸的上限）一旦达到上限，就会记录下当前的执行进度，下次再执行。保证了 Redis 能够及时响应不发生阻塞。


大家还可以阅读我的 Redis 相关的文章：

[Redis 的基础数据结构（一） 可变字符串、链表、字典](https://www.xilidou.com/2018/03/12/redis-data/?fromblog=redis-server)


[Redis 的基础数据结构（二） 整数集合、跳跃表、压缩列表](https://www.xilidou.com/2018/03/13/redis-data2/?blogfrom=redis-server)


[Redis 的基础数据结构（三）对象 ](https://www.xilidou.com/2018/03/15/redis-object/)


[Redis 数据库、键过期的实现](https://www.xilidou.com/2018/03/20/redis-server/)


[Redis 中的事件驱动模型](https://www.xilidou.com/2018/03/22/redis-event/)

欢迎关注我的微信公众号：
![二维码](https://ws3.sinaimg.cn/large/006tNc79ly1fo3oqp60v3j307607674r.jpg)


