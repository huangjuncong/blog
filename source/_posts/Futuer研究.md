---
title: Future研究
date: 2017-10-24 00:00:07
tags: [java,Future,多线程]
---

## Future是什么？

最近写了一些关于`netty`的相关代码，发现类似`netty` 的这种异步框架大量的使用一个Future的类。利用这个future类可以实现，代码的异步调用，程序调用耗时的网络或者IO相关的方法的时候，首先获得一个Future的代理类，同时线程并不会被阻塞。继续执行之后的逻辑，直到真正要使用远程调用返回的结果的时候，才需要调用future的`get()`方法。这样可以提高代码的执行效率。
于是就花了一点时间研究future是如何实现的。调用方式如何知道，结果什么时候返回的呢？如果使用一个线程去轮询`flag` 标记，那么就很难及时的感知对象的改变，同时还很难降低开销。。所以我们需要了解java的等待通知机制。利用这个机制来构建一个节能环保的Future。

 <!--more-->

## 等待通知机制

一个线程修改了一个对象的值，另一个线程感知到了变化，然后进行相应的操作。一个线程是生产者，另一个线程是消费者。这种模式做到了解耦，隔离了“做什么”和“做什么”。如果要实现这个功能，我们可以利用java内对象内置的等待通知机制来实现。
我们知道'java.lang.Object'有以下方法

| 方法名称 |描述  |
| --- | --- | 
| notify（） | 随机选择通知一个在对象上等待的的线程，解除其阻塞状态。 |
| notfiyAll（）|解除所有那些在该对象上调用wait方法的线程的阻塞状态 |
| wait（）| 导致线程进入等待状态。 |
| wait（long）|同上，同时设置一个超时时间，线程等待一段时间。|
| wait（long，int）|同上，且为超时时间设置一个单位。|

ps：敲黑板，面试中面试官可能会问，你了解'Object'的哪些方法？如果只答出 `toString()`的话。估计得出门右转慢走不送了。

所谓等待通知机制，就是某个线程A调用了对象 O 的`wait()`方法，另一个线程B调用对象 O 的 `notify()` 或者 `notifyAll()` 方法。 线程 A 接收到线程 B 的通知，从wait状态中返回，继续执行后续操作。两个线程通过对象 O 来进行通信。
我们看damo：

```java

public class waitAndNotify {

    private static Object object = new Object();
    private static boolean flag = true;

    public static void main(String[] args) throws InterruptedException {

        Thread a = new Thread(new waitThread(),"wait");
        a.start();

        TimeUnit.SECONDS.sleep(5);

        Thread b = new Thread(new notifyThread(),"notify");
        b.start();

    }

    static class waitThread implements Runnable{
        @Override
        public void run() {
            synchronized (object){
                while (flag){
                    try {
                        System.out.println(Thread.currentThread() + "flag is true wait @" +
                                new SimpleDateFormat("HH:mm:ss").format(new Date()));
                        object.wait();
                    }catch (InterruptedException e){
                    }
                }

                System.out.println(Thread.currentThread() + "flag is false go on @"+
                        new SimpleDateFormat("HH:mm:ss").format(new Date()));
            }

        }
    }

    static class notifyThread implements Runnable{

        @Override
        public void run() {
            synchronized (object){
                System.out.println(Thread.currentThread() + "lock the thread and change flag" +
                        new SimpleDateFormat("HH:mm:ss").format(new Date()));
                object.notify();
                flag = false;
                try {
                    TimeUnit.SECONDS.sleep(5);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            synchronized (object){
                System.out.println(Thread.currentThread() + "lock the thread again@" +
                        new SimpleDateFormat("HH:mm:ss").format(new Date()));
                try {
                    TimeUnit.SECONDS.sleep(5);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }

    }
}
```

程序的输出：
>Thread[wait,5,main]flag is true wait @22:46:52
Thread[notify,5,main]lock the thread and change flag22:46:57
Thread[notify,5,main]lock the thread again@22:47:02
Thread[wait,5,main]flag is false go on @22:47:07

1. `wait()` 和 `notify()`以及`notifyAll()` 需要在对象被加锁以后会使用。
2. 调用`notify()` 和`notifyAll()` 后，对象并不是立即就从`wait()`返回。而是需要对象的锁释放以后，等待线程才会从`wait()`中返回。

## 等待通知经典范式

通过以上的代码我们可以把等待通知模式进行抽象。
wait线程：
1. 获取对象的锁。
2. 条件不满足，调用对象`wait()`方法。
3. 等待另外线程通知，如果满足条件，继续余下操作执行。
伪码如下：

```java
lock(object){
    while(condition){
        object.wait();
    }
    doOthers();
}
```

notify线程：
1. 获取对象的锁。
2. 修改条件。
3. 调用对象的`notify()`或者`notifyAll()`方法通知等待的线程。
4. 释放锁.
伪码如下:

```java
lock(object){
    change(condition);
    objcet.notify();
}
```

## Future的实现原理：

了解了java的等待通知机制，我们来看看如何利用这个机制实现一个简单的Future。
首先我们定义一个Future的接口：

```java
public interface IData {

    String getResult();

}
```

假设我们有一个很耗时的远程方法：

```java
public class RealData implements IData {

    private String result;
    RealData(String str){
        StringBuilder sb = new StringBuilder();
        //假设一个相当耗时的远程方法
        for (int i = 0; i < 20; i++) {
            sb.append("i").append(i);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        result = sb.append(str).toString();
    }

    @Override
    public String getResult() {
        return result;
    }
}
```

同时还要有一个实现了`IData`的`RealData`包装类:

```java
public class FutureData implements IData {

    private RealData realData;
    private volatile boolean isReal = false;

    @Override
    public synchronized String getResult() {

        while (!isReal){
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        return realData.getResult();
    }

    public synchronized void setReault(RealData realData){
        if(isReal){
            return;
        }

        this.realData = realData;
        isReal = true;
        notifyAll();

    }
}
```

可以看出来我们的这个包装类就是一个相当标准的等待通知机制的类。

再看看我们Service类，在Service中的getData方法被调用的时候，程序只接返回了一个FutureData的代理类，同时起了一个新的线程去执行真正耗时的`RealData`。

```java
public class Service {

    public IData getData(final String str){

        final FutureData futureData = new FutureData();
        new Thread(new Runnable() {
            @Override
            public void run() {
                RealData realData = new RealData(str);
                futureData.setReault(realData);
            }
        }).start();
        return futureData;
    }

}
``` 

最后看看是如何使用的：

```java
public class Clinet {

    public static void main(String[] args) {

        Service service = new Service();

        IData data = service.getData("test");

        System.out.println("a");
        System.out.println("b");

        System.out.println("result is " + data.getResult());

    }

}
```

执行的结果是：

```
a
b
result is i0i1i2i3i4i5i6i7i8i9i10i11i12i13i14i15i16i17i18i19test
```

可见程序并没有因为调用耗时的方法阻塞，先打印了a和b，在程序调用`getReslut()`才打印出真正的结果。

## 总结：

通过以上的讲解，我们总结一下future，首先使用future可以实现异步调用，实现future我们使用了java的等待通知机制。这个时候们回过头再来看netty的future就很简单了。

## 参考

《java并发编程的艺术》
[漫谈并发编程：Future模型（Java、Clojure、Scala多语言角度分析）](http://dantezhao.com/2017/04/23/concurrency-and-parallelism-future/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
