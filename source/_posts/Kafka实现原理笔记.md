---
title: Kafka实现原理笔记
date: 2017-10-24 19:04:15
tags: [Kafka,分布式]
---

最近想了解一下分布式消息系统是怎么组成的于是就花了一些时间研究了kafka的实现原理。记录下来方便自己复习和回忆。kafka的设计思想很精妙，可以借鉴到大部分的分布式系统中。

<!--more-->

## kafka可以解决什么问题？

* kafka可以支持大量数据吞吐。
* 可以优雅的处理数据堆积问题。
* 低延迟
* 支持分布式

## 设计理念

### 持久化

1. 尽量线性的读写磁盘。一个硬盘的顺序读写速度一般是4k读写的千倍以上。线性的读写是可以被预测，也能被操作系统大幅的优化的。
2. 以pagecache为中心的设计风格，使用文件系统并依赖于pagecache要优于维护内存中缓存或其他结构。一方面避免 JVM 中的 gc带来的性能损耗。同时简化了代码实现。
3. 持久化队列，只需要简单的在文件后面追加写入即可。而不用考虑建立一个索引文件（BTree）。查询和写入的复杂度由 BTree的 O(logN) 减小为线性的 O(1)。大幅提升数据的吞吐量，有利于处理海量数据，且对存储系统的性能要求不高，降低成本。
4. 考虑将多条消息聚合在一次。减少平均每条消息的开销。
5. 使用 [zero-copy](https://www.ibm.com/developerworks/linux/library/j-zerocopy/)减少字符拷贝时候的开销。
6. 开启压缩协议，减少数据所占的空间。

### 生产者 （Producer）

1. Producer 向 Leader Partition 发送消息。
2. Producer 可以向任何一个 Partition 询问整个集群的状态，以及谁是 Leader Partition
3. Producer 自己决定写入到哪个 Partition。Producer 可以考虑使用何种负载的策略。随机，轮询，按照key分区都可以。
4. 支持批量操作。消息攒够一定数量再发送，使用适当的延迟换来更高的数据吞吐量。

### 消费者 （Consumer）

1. 消费者直接向 Leader Partition发送一个 fatch 的请求，并制定消费的起始位置（offset），取回offset后的一段数据进行处理。
2. Consumer 自己决定 Offset，自己决定从什么地方进行消费。
3. **Push 和 Pull** 的问题。 消息到底是推还是拉？ kafka 采取的机制是，Producer 向 Broker push 消息。 Consumer 向 Broker Pull 消息。这样做有几个好处。第一，消息消费的速率由 Consumer自己决定。第二，可以聚合的数据批量处理数据，如果使用 push，Broker需要考虑到底要等到多条数据，还是及时发送，Consumer可以尽可能多的拉取数据，保证消息尽可能及时被消费。
4. 如何记录那些消息被**有效消费**？Topic 被划分为多个有序的分区，保证每个分区任何时候只会被同一个Group里面的 Consumer消费。只需要记录消费的偏移量。同时这个位置可以作为CkeckPonit，定时检查。保证ACK的代价很小。
5. 如果我们可以使用某个 Consumer 消费数据后，存储到 类似Hadoop的平台上持久化。

### kafka 消息的语义

1. 消息系统系统一般有以下的语义：
    * At most once：消息可能丢失，但不会重复投递
    * At least once：消息不会丢失，但可能会重复投递
    * Exactly once：消息不丢失、不重复，会且只会被分发一次（真正想要的）
2. Producer 发送消息以后，有一个commit的概念，如果commit成功，则意味着消息不会丢失，但是Producer有可能提交成功后，没有收到commit的消息。这有可能造成 at least once 语义。
3. 从 Consumer 角度来看，我们知道 Offset 是由 Consumer 自己维护。所以何时更新 Offset 就决定了 Consumer 的语义。如果收到消息后更新 Offset，如果 Consumer crash，那新的 Cunsumer再次重启消费，就会造成 At most once 语义（消息会丢，但不重复）。
4. 如果 Consumser 消费完成后，再更新 Offset。如果 Consumer crash，别的 Consumer 重新用这个 Offser 拉取消息，这个时候就会造成 at least once 的语义（消息不丢，但多次被处理）。

所以结论：默认Kafka提供at-least-once语义的消息分发，允许用户通过在处理消息之前保存位置信息的方式来提供at-most-once语义。如果我们可以实现消费是**幂等**的，这个时候就可以认为整个系统是Exactly once的了。


### 备份

1. kafka 对每个 topic 的 partiotion 进行备份，份数由用户自己设置。
2. 默认情况下 kafka 有一个 Leader 和 0至多个 Follower。
3. 我们可以认为 Follower 也是一个 Consumer，通过消费 Leader 上的日志然后备份到本地。 
4. 所有的读写都是在 Leader 上进行的，所以 Follower 真的就只是备份。
5. kafka 如何确认一个 Follower 是活的？
    * 和 zookeeper 保持联系。
    * Follower 复制 Leader 上的消息，且落后的不多（可配置）。
6. 消息同步到所有的 Follower 才认为是提交成功，提交成功才能被消费。所以 Leader 宕机不会造成消息丢失（注意之前的Producer的 at least once 语义）。

### 选举
1. Leader宕机以后，需要在Follower中选出一个新 Leader。 Kafak动态维护一个同步备份集合（ISR）。这个集合中的 Follower 都能成为 Leader。 一个写入，要同步到所有的 ISR 中才能算做 Commit 成功。同时 ISR 会被持久化到 ZK 中。
2. 如果全部节点都故障了，kafka会选择第一副本（无需在ISR中） 作为Leader。这个时候会造成丢消息。
3. Producer 可以选择是否等待备份响应。所谓的备份相应，是指 ISR 集合中的备份响应。