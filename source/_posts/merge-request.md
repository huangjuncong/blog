---
title: 高并发环境下的请求合并
date: 2018-01-22 19:10:45
tags:
---

在高并发系统中，我们经常遇到这样的需求：系统产生大量的请求，但是这些请求实时性要求不高。我们就可以将这些请求合并，达到一定数量我们统一提交。最大化的利用系统性IO,提升系统的吞吐性能。

所以请求合并框架需要考虑以下两个需求：

1. 当请求收集到一定数量时提交数据
2. 一段时间后如果请求没有达到指定的数量也进行提交

我们就聊聊一如何实现这样一个需求。

阅读这篇文章你将会了解到:

* ScheduledThreadPoolExecutor
* 阻塞队列
* 线程安全的参数
* LockSuppor的使用

<!--more-->

## 设计思路和实现 

我们就聊一聊实现这个东西的具体思路是什么。希望大家能够学习到分析问题，设计模块的一些套路。

1. 底层使用什么数据结构来持有需要合并的请求？
    * 既然我们的系统是在高并发的环境下使用，那我们肯定不能使用，普通的`ArrayList`来持有。我们可以使用阻塞队列来持有需要合并的请求。
    * 我们的数据结构需要提供一个 add() 的方法给外部，用于提交数据。当外部add数据以后，需要检查队列里面的数据的个数是否达到我们限额？达到数量提交数据，不达到继续等待。
    * 数据结构还需要提供一个timeOut()的方法，外部有一个计时器定时调用这个timeOut方法，如果方法被调用，则直接向远程提交数据。
    * 条件满足的时候线程执行提交动作，条件不满足的时候线程应当暂停，等待队列达到提交数据的条件。所以我们可以考虑使用 `LockSuppor.park()`和`LockSuppor.unpark` 来暂停和激活操作线程。

经过上面的分析，我们就有了这样一个数据结构：

```java
private static class FlushThread<Item> implements Runnable{

        private final String name;

        //队列大小
        private final int bufferSize;
        //操作间隔
        private int flushInterval;

        //上一次提交的时间。
        private volatile long lastFlushTime;
        private volatile Thread writer;

        //持有数据的阻塞队列
        private final BlockingQueue<Item> queue;

        //达成条件后具体执行的方法
        private final Processor<Item> processor;

        //构造函数
        public FlushThread(String name, int bufferSize, int flushInterval,int queueSize,Processor<Item> processor) {
            this.name = name;
            this.bufferSize = bufferSize;
            this.flushInterval = flushInterval;
            this.lastFlushTime = System.currentTimeMillis();
            this.processor = processor;

            this.queue = new ArrayBlockingQueue<>(queueSize);

        }

        //外部提交数据的方法
        public boolean add(Item item){
            boolean result = queue.offer(item);
            flushOnDemand();
            return result;
        }

        //提供给外部的超时方法
        public void timeOut(){
            //超过两次提交超过时间间隔
            if(System.currentTimeMillis() - lastFlushTime >= flushInterval){
                start();
            }
        }
        
        //解除线程的阻塞
        private void start(){
            LockSupport.unpark(writer);
        }

        //当前的数据是否大于提交的条件
        private void flushOnDemand(){
            if(queue.size() >= bufferSize){
                start();
            }
        }

        //执行提交数据的方法
        public void flush(){
            lastFlushTime = System.currentTimeMillis();
            List<Item> temp = new ArrayList<>(bufferSize);
            int size = queue.drainTo(temp,bufferSize);
            if(size > 0){
                try {
                    processor.process(temp);
                }catch (Throwable e){
                    log.error("process error",e);
                }
            }
        }

        //根据数据的尺寸和时间间隔判断是否提交
        private boolean canFlush(){
            return queue.size() > bufferSize || System.currentTimeMillis() - lastFlushTime > flushInterval;
        }

        @Override
        public void run() {
            writer = Thread.currentThread();
            writer.setName(name);

            while (!writer.isInterrupted()){
                while (!canFlush()){
                    //如果线程没有被打断，且不达到执行的条件，则阻塞线程
                    LockSupport.park(this);
                }
                flush();
            }

        }

    }
```

2. 如何实现定时提交呢？

通常我们遇到定时相关的需求，首先想到的应该是使用 `ScheduledThreadPoolExecutor`定时来调用FlushThread 的 timeOut 方法,如果你想到的是 `Thread.sleep()`...那需要再努力学习，多看源码了。

3. 怎样进一步的提升系统的吞吐量？

我们使用的`FlushThread` 实现了 `Runnable` 所以我们可以考虑使用线程池来持有多个`FlushThread`。

所以我们就有这样的代码：

```java

public class Flusher<Item> {

    private final FlushThread<Item>[] flushThreads;

    private AtomicInteger index;

    //防止多个线程同时执行。增加一个随机数间隔
    private static final Random r = new Random();

    private static final int delta = 50;

    private static ScheduledExecutorService TIMER = new ScheduledThreadPoolExecutor(1);

    private static ExecutorService POOL = Executors.newCachedThreadPool();

    public Flusher(String name,int bufferSiz,int flushInterval,int queueSize,int threads,Processor<Item> processor) {

        this.flushThreads = new FlushThread[threads];


        if(threads > 1){
            index = new AtomicInteger();
        }

        for (int i = 0; i < threads; i++) {
            final FlushThread<Item> flushThread = new FlushThread<Item>(name+ "-" + i,bufferSiz,flushInterval,queueSize,processor);
            flushThreads[i] = flushThread;
            POOL.submit(flushThread);
            //定时调用 timeOut()方法。
            TIMER.scheduleAtFixedRate(flushThread::timeOut, r.nextInt(delta), flushInterval, TimeUnit.MILLISECONDS);
        }
    }

    // 对 index 取模，保证多线程都能被add
    public boolean add(Item item){
        int len = flushThreads.length;
        if(len == 1){
            return flushThreads[0].add(item);
        }

        int mod = index.incrementAndGet() % len;
        return flushThreads[mod].add(item);

    }

    //上文已经描述
    private static class FlushThread<Item> implements Runnable{
        ...省略
    }
}

```

4. 面向接口编程，提升系统扩展性：

```java
public interface Processor<T> {
    void process(List<T> list);
}
```

## 使用

我们写个测试方法测试一下：

```java
//实现 Processor 将 String 全部输出
public class PrintOutProcessor implements Processor<String>{
    @Override
    public void process(List<String> list) {

        System.out.println("start flush");

        list.forEach(System.out::println);

        System.out.println("end flush");
    }
}

```

```java

public class Test {

    public static void main(String[] args) throws InterruptedException {

        Flusher<String> stringFlusher = new Flusher<>("test",5,1000,30,1,new PrintOutProcessor());

        int index = 1;
        while (true){
            stringFlusher.add(String.valueOf(index++));
            Thread.sleep(1000);
        }
    }
}

```

执行的结果：

```shell

start flush
1
2
3
end flush
start flush
4
5
6
7
end flush

```

我们发现并没有达到10个数字就触发了flush。因为出发了超时提交，虽然还没有达到规定的5
个数据，但还是执行了 flush。

如果我们去除 `Thread.sleep(1000);` 再看看结果：

```shell
start flush
1
2
3
4
5
end flush
start flush
6
7
8
9
10
end flush
```

每5个数一次提交。完美。。。。

## 总结

一个比较生动的例子给大家讲解了一些多线程的具体运用。学习多线程应该多思考多动手，才会有比较好的效果。希望这篇文章大家读完以后有所收获，欢迎交流。

github地址:[https://github.com/diaozxin007/framework](https://github.com/diaozxin007/framework)