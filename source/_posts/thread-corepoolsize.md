---
title: 线程池 execute() 的工作逻辑
date: 2018-02-09 23:42:27
tags: [java,线程池,多线程]
---

原文地址:[https://www.xilidou.com/2018/02/09/thread-corepoolsize/](https://www.xilidou.com/2018/02/09/thread-corepoolsize/)

最近在看[《Java并发编程的艺术》](https://union-click.jd.com/jdc?d=igYcKg)回顾线程池的原理和参数的时候发现一个问题，如果 corePoolSize = 0 且 阻塞队列是无界的。线程池将如何工作？

我们先回顾一下书里面描述线程池`execute()`工作的逻辑：

1. 如果当前运行的线程，少于corePoolSize，则创建一个新的线程来执行任务。
2. 如果运行的线程等于或多于 corePoolSize，将任务加入 BlockingQueue。
3. 如果 BlockingQueue 内的任务超过上限，则创建新的线程来处理任务。
4. 如果创建的线程数是单钱运行的线程超出 maximumPoolSize，任务将被拒绝策略拒绝。

看了这四个步骤，其实描述上是有一个漏洞的。如果核心线程数是0，阻塞队列也是无界的，会怎样？如果按照上文的逻辑，应该没有线程会被运行，然后线程无限的增加到队列里面。然后呢？

<!--more-->

于是我做了一下试验看看到底会怎样？

```java
public class threadTest {
    private final static ThreadPoolExecutor executor = new ThreadPoolExecutor(0,1,0, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>());
    public static void main(String[] args) {
        AtomicInteger atomicInteger = new AtomicInteger();
        while (true) {
            executor.execute(() -> {
                System.out.println(atomicInteger.getAndAdd(1));
            });
        }
    }
}
```

结果里面的`System.out.println(atomicInteger.getAndAdd(1));`语句执行了，与上面的描述矛盾了。到底发生了什么？线程池创建线程的逻辑是什么？我们还是从源码来看看到底线程池的逻辑是什么？

# ctl

要了解线程池，我们首先要了解的线程池里面的状态控制的参数 ctl。

* 线程池的ctl是一个原子的 AtomicInteger。
* 这个ctl包含两个参数 ：
  * workerCount 激活的线程数
  * runState 当前线程池的状态
* 它的低29位用于存放当前的线程数, 因此一个线程池在理论上最大的线程数是 536870911; 高 3 位是用于表示当前线程池的状态, 其中高三位的值和状态对应如下:
  * 111: RUNNING
  * 000: SHUTDOWN
  * 001: STOP
  * 010: TIDYING
  * 110: TERMINATED

为了能够使用 ctl 线程池提供了三个方法:

```java
    // Packing and unpacking ctl
    // 获取线程池的状态
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    // 获取线程池的工作线程数
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    // 根据工作线程数和线程池状态获取 ctl
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

# execute

外界通过 execute 这个方法来向线程池提交任务。

先看代码：

```java

 public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();

        //如果工作线程数小于核心线程数，
        if (workerCountOf(c) < corePoolSize) {
            //执行addWork，提交为核心线程,提交成功return。提交失败重新获取ctl
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }

        //如果工作线程数大于核心线程数，则检查线程池状态是否是正在运行，且将新线程向阻塞队列提交。
        if (isRunning(c) && workQueue.offer(command)) {

            //recheck 需要再次检查,主要目的是判断加入到阻塞队里中的线程是否可以被执行
            int recheck = ctl.get();

            //如果线程池状态不为running，将任务从阻塞队列里面移除，启用拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果线程池的工作线程为零，则调用addWoker提交任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }

        //添加非核心线程失败，拒绝
        else if (!addWorker(command, false))
            reject(command);
    }

```

# addWorker

```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            //获取线程池状态
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 判断是否可以添加任务。
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                //获取工作线程数量
                int wc = workerCountOf(c);
                //是否大于线程池上限，是否大于核心线程数，或者最大线程数
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //CAS 增加工作线程数
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                //如果线程池状态改变，回到开始重新来
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;

        //上面的逻辑是考虑是否能够添加线程，如果可以就cas的增加工作线程数量
        //下面正式启动线程
        try {
            //新建worker
            w = new Worker(firstTask);

            //获取当前线程
            final Thread t = w.thread;
            if (t != null) {
                //获取可重入锁
                final ReentrantLock mainLock = this.mainLock;
                //锁住
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    // rs < SHUTDOWN ==> 线程处于RUNNING状态
                    // 或者线程处于SHUTDOWN状态，且firstTask == null（可能是workQueue中仍有未执行完成的任务，创建没有初始任务的worker线程执行）
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        // 当前线程已经启动，抛出异常
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //workers 是一个 HashSet 必须在 lock的情况下操作。
                        workers.add(w);
                        int s = workers.size();
                        //设置 largeestPoolSize 标记workAdded
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                //如果添加成功，启动线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            //启动线程失败，回滚。
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

先看看 `addWork()` 的两个参数，第一个是需要提交的线程 Runnable firstTask，第二个参数是 boolean 类型，表示是否为核心线程。

execute() 中有三处调用了 `addWork()` 我们逐一分析。

* 第一次，条件 `if (workerCountOf(c) < corePoolSize)` 这个很好理解，工作线程数少于核心线程数，提交任务。所以 `addWorker(command, true)`。
* 第二次，如果 `workerCountOf(recheck) == 0` 如果worker的数量为0，那就 `addWorker(null,false)`。为什么这里是 `null` ？之前已经把 command 提交到阻塞队列了 `workQueue.offer(command)` 。所以提交一个空线程，直接从阻塞队列里面取就可以了。
* 第三次，如果线程池没有 RUNNING 或者 offer 阻塞队列失败，`addWorker(command,false)`，很好理解，对应的就是，阻塞队列满了，将任务提交到，非核心线程池。与最大线程池比较。

至此，重新归纳`execute()`的逻辑应该是：

1. 如果当前运行的线程，少于corePoolSize，则创建一个新的线程来执行任务。
2. 如果运行的线程等于或多于 corePoolSize，将任务加入 BlockingQueue。
3. 如果加入 BlockingQueue 成功，需要二次检查线程池的状态如果线程池没有处于 Running，则从 BlockingQueue 移除任务，启动拒绝策略。
4. 如果线程池处于 Running状态，则检查工作线程（worker）是否为0。如果为0，则创建新的线程来处理任务。如果启动线程数大于maximumPoolSize，任务将被拒绝策略拒绝。
5. 如果加入 BlockingQueue 。失败,则创建新的线程来处理任务。
6. 如果启动线程数大于maximumPoolSize，任务将被拒绝策略拒绝。

# 总结

回顾我开始提出的问题：
> 如果 corePoolSize = 0 且 阻塞队列是无界的。线程池将如何工作？

这个问题应该就不难回答了。

# 最后

[《Java并发编程的艺术》](https://union-click.jd.com/jdc?d=igYcKg)是一本学习 java 并发编程的好书，在这里推荐给大家。

同时，希望大家在阅读技术数据的时候要仔细思考，结合源码，发现，提出问题，解决问题。这样的学习才能高效且透彻。

欢迎关注我的微信公众号

![二维码](https://ws3.sinaimg.cn/large/006tNc79ly1fo3oqp60v3j307607674r.jpg)