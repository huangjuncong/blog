---
title: redis-recommend
date: 2018-03-30 12:45:46
tags:
---


# Redis 命令的执行过程

之前写了一系列文章，已经很深入的探讨了 Redis 的数据结构，数据库的实现，key的过期策略以及 Redis 是怎么处理事件的。所以距离 Redis 的单机实现只差最后一步了，就是 Redis 是怎么处理 client 发来的命令并返回结果的，所以我们就仔细讨论一下 Redis 是怎么执行命令的。

# Redis client（客户端）

Redis 是单线程应用，它是如何与多个客户端简历网络链接并处理命令的？
由于 Redis 是基于 I/O 多路复用技术，为了能够处理多个客户端的请求，Redis 在本地为每一个链接到 Redis 服务器的客户端创建了一个 redisClient 的数据结构，这个数据结构包含了每个客户端各自的状态和执行的命令。 Redis 服务器使用一个链表来维护多个 redisClient 数据结构。

在服务器端用一个链表来管理所有的 redisClinet。

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

## flags

flags 表示了目前客户端的角色，以及目前所处的状态。他比较特殊可以单独表示一个状态或者多个状态。

## querybuf

querybuf 是一个 sds 动态字符串类型，所谓 buf 说明是一个缓冲区，用于存储没有被解析的命令。 

## argc & argv 

上文的 querybuf 是一个没有处理过的命令，当 Redis 将 querybuf 命令解析以后，会将得出的参数个数和以及参数分别保存在 argc 和 argv 中。argv 是一个 redisObject 的数组。

对于命令 ：
```bash
SET mobile 1380000000
```
argc = 3,

argv 就如图所示：

## cmd

cmd 是一个 redisCommend。 Redis 是用一个字典保存了所有的 redisCommend。key 是 redisCommend 的名字，值就是一个 redisCommend 结构，这个结构保存了命令的实现函数，命令的标志，命令应该给定的参数个数，命令的执行次数和总消耗时长等统计信息

当 Redis 解析出 argv 和 argc 后，会根据数组 argv[0]，去 redisCommend 的字典中查询出对应的 redisCommend。上文的例子中就回去查找 `SET` 这个命令。

## buf & bufpos & reply

buf 是一个长度为 REDIS_REPLY_CHUNK_BYTES 的数组。Redis 执行相应的操作以后，就会将需要返回的返回的数据存储到 buf 中，bufpos 用于记录 buf 中已用的字节数数量，当需要恢复的数据大于 REDIS_REPLY_CHUNK_BYTES 时，redis 就会是用 reply 这个链表来保存数据。

## 其他参数

其他参数大家看注释就能明白，就是字面的意思。省略的参数基本上都是涉 Redis 集群管理的参数，在之后的文章中会继续讲解。

## 客户端的链接和断开

上文说过 redisServer 是用一个链表来维护所有的 redisClient 状态，每当有一个客户端发起链接以后，就会在 Redis 中生成一个对应的 redisClient 数据结构，增加到`clients`这个链表之后。

一个客户端很可能被多种原因断开。

总体分为几种类型：

* 客户端主动退出或者被 kill。
* timeout 超时。
* Redis 为了自我保护，会断开发送或者回复的数据超过限制大小的客户端。

## 调用总结

当客户端和服务器端的嵌套字变得可读的时候，服务器将会调用命令请求处理器来执行以下操作：

1. 读取嵌套字中的数据，写入 querybuf。
2. 解析 querybuf 中的命令，记录到 argc 和 argv 中。
3. 根据 argv[0] 查找对应的 recommend。
4. 执行 recommend 对应的执行函数。
5. 执行以后将结过存入 buf & bufpos & reply 中，返回给调用方。
 

