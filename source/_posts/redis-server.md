---
title: Redis 数据库、键过期的实现
date: 2018-03-20 17:26:08
tags: [redis,db,redisdb,redis key,redis expire]
---

原文地址：[https://www.xilidou.com/2018/03/20/redis-server/](https://www.xilidou.com/2018/03/20/redis-server/)

之前的文章讲解了 Redis 的数据结构，这回就可以看看作为内存数据库，Redis 是怎么存储数据的。以及键是怎么过期的。

阅读这篇文章你将会了解到：

* Redis 的数据库实现
* Redis 键过期的策略

<!--more-->

# 数据库的实现

我们先看代码 `server.h/redisServer` 

```c
struct redisServer{
    ...

    //保存 db 的数组
    redisDb *db;
    
    //db 的数量
    int dbnum;

    ...
}

```

再看redisDb的代码：

```c
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
} redisDb;

```

总体来说redis的 server 包含若干个（默认16个） redisDb 数据库。

![db](https://xilidou.oss-cn-beijing.aliyuncs.com/img/redisServer.jpg)

Redis 是一个 k-v 存储的键值对数据库。其中字典 dict 保存了数据库中的所有键值对，这个地方叫做 `keyspace` 直译过来就是“键空间”。

所以我们就可以这么认为，在 redisDb 中我们使用 dict（字典）来维护键空间。

* keyspace 的 kay 是数据库的 key，每一个key 是一个字符串对象。注意不是字符串，而是字符串对象。
 
* keyspace 的 value 是数据库的 value，这个 value 可以是 redis 的，字符串对象，列表对象，哈希表对象，集合对象或者有序对象中的一种。

## 数据库读写操作

所以对于数据的增删改查，就是对 keyspace 这个大 map 的增删改查。

当我们执行：

```bash
>redis SET mobile "13800000000"
```

实际上就是为 keyspace 增加了一个 key 是包含字符串“mobile”的字符串对象，value 为包含字符“13800000000”的字符串对象。

看图：

![db](https://xilidou.oss-cn-beijing.aliyuncs.com/img/dbNoExpire.jpg)

对于删改查，没啥好说的。类似java 的 map 操作，大多数程序员应该都能理解。

需要特别注意的是，再执行对键的读写操作的时候，Redis 还要做一些额外的维护动作：

* 维护 hit 和 miss 两个计数器。用于统计 Redis 的缓存命中率。
* 更新键的 LRU 时间，记录键的最后活跃时间。
* 如果在读取的时候发现键已经过期，Redis 先删除这个过期的键然后再执行余下操作。
* 如果有客户对这个键执行了 WATCH 操作，会把这个键标记为 dirty，让事务注意到这个键已经被改过。
* 没修改一次 dirty 会增加1。
* 如果服务器开启了数据库通知功能，键被修改之后，会按照配置发送通知。

## 键的过期实现

Redis 作为缓存使用最主要的一个特性就是可以为键值对设置过期时间。就看看 Redis 是如果实现这一个最重要的特性的？

在 Redis 中与过期时间有关的命令 

* EXPIRE 设置 key 的存活时间单位秒
* EXPIREAT 设置 key 的过期时间点单位秒
* PEXPIRE 设置 key 的存活时间单位毫秒
* PEXPIREAT 设置 key 的过期时间点单位毫秒

其实这些命令，底层的命令都是由 REXPIREAT 实现的。

在 redisDb 中使用了 dict *expires，来存储过期时间的。其中 key 指向了 keyspace 中的 key（c 语言中的指针）， value 是一个 long long 类型的时间戳，标定这个 key 过期的时间点，单位是毫秒。

如果我们为上文的 mobile 增加一个过期时间。

```bash
>redis PEXPIREAT mobile 1521469812000
```

这个时候就会在过期的 字典中增加一个键值对。如下图：

![db](https://xilidou.oss-cn-beijing.aliyuncs.com/img/db.jpg)

对于过期的判断逻辑就很简单：

1. 在 字典 expires 中 key 是否存在。
2. 如果 key 存在，value 的时间戳是否小于当前系统时间戳。

接下来就需要讨论一下过期的键的删除策略。

key的删除有三种策略：

1. 定时删除，Redis定时的删除内存里面所有过期的键值对，这样能够保证内存友好，过期的key都会被删除，但是如果key的数量很多，一次删除需要CPU运算，CPU不友好。
2. 惰性删除，只有 key 在被调用的时候才去检查键值对是否过期，但是会造成内存中存储大量的过期键值对，内存不友好，但是极大的减轻CPU 的负担。
3. 定时部分删除，Redis定时扫描过期键，但是只删除部分，至于删除多少键，根据当前 Redis 的状态决定。

这三种策略就是对时间和空间有不同的倾向。Redis为了平衡时间和空间，采用了后两种策略 惰性删除和定时部分删除。

惰性删除比较简单，不做过多介绍。主要讨论一下定时部分删除。

过期键的定时删除的策略由 expire.c/activeExpireCycle() 函数实现，server.c/serverCron() 定时的调用 `activieExpireCycle()` 。

activeExpireCycle 的大的操作原则是，如果过期的key比较少，则删除key的数量也比较保守，如果，过期的键多，删除key的策略就会很激进。

```c
static unsigned int current_db = 0; /* Last DB tested. */
static int timelimit_exit = 0;      /* Time limit hit in previous call? */
static long long last_fast_cycle = 0; /* When last fast cycle ran. */
```

* 首先三个 `static` 全局参数分别记录目前遍历的 db下标，上一次删除是否是超时退出的，上一次快速操作是什么时候进行的。
* 计算 `timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;` 可以理解为 25% 的 cpu 时间。 

* 如果 db 中 expire 的大小为0 不操作
* expire 占总 key 小于 1% 不操作

* num = dictSize(db->expires)；num 是 expire 使用的key的数量。
* slots = dictSlots(db->expires); slots 是 expire 字典的尺寸大小。

* 已使用的key（num） 大于 ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP 则设置为 ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP。也就是说每次只检查 ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP 个键。

* 随机获取带过期的 key。计算是否过期，如果过期就删除。

* 然后各种统计，包括删除键的次数，平均过期时间。

* 每遍历十六次，计算操作时间，如果超过 timelimit 结束返回。

* 如果删除的过期键大于 ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP 的 1\4 就跳出循环，结束。

步骤比较复杂，总结一下：（这里都是以默认配置描述）

1. redis 会用最多 25% 的 cpu 时间处理键的过期。
2. 遍历所有的 redisDb
3. 在每个 redisDb 中如果数据中没有过期键或者过期键比例过低就直接进入下一个 redisDb。
4. 否则，遍历 redisDb 中的过期键，如果删除的键达到有过期时间的的key 的25% ，或者操作时间大于 cpu 时间的 25% 就结束当前循环，进入下一个redisDb。

# 后记

这篇文章主要解释了 Redis 的数据库是怎么实现的，同时介绍了 Redis 处理过期键的逻辑。看 Redis 的代码越多越发现，实际上 Redis 一直在做的一件事情就是平衡，一直在平衡程序的空间和时间。其实平时的业务设计，就是在宏观上平衡，平衡系统的时间和空间。所以我想说的是，看源码是让我们从微观学习系统架构，是每一位架构师的必经之路。大家加油。

我之前的三篇关于 Redis 的基础数据结构链接地址，欢迎大家阅读。


[Redis 的基础数据结构（一） 可变字符串、链表、字典](https://xilidou.com/2018/03/12/redis-data/?fromblog=redis-server)


[Redis 的基础数据结构（二） 整数集合、跳跃表、压缩列表](https://www.xilidou.com/2018/03/13/redis-data2/?blogfrom=redis-server)


[Redis 的基础数据结构（三）对象 ](https://www.xilidou.com/2018/03/15/redis-object/)

欢迎关注我的微信公众号：
![二维码](https://ws3.sinaimg.cn/large/006tNc79ly1fo3oqp60v3j307607674r.jpg)











