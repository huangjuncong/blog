---
title: JAVA 中的 CAS
date: 2018-02-01 13:56:06
tags: [java,cas,高并发,锁]
---

CAS 是现代操作系统，解决并发问题的一个重要手段，最近在看 `eureka` 的源码的时候。遇到了很多 CAS 的操作。今天就系统的回顾一下 Java 中的CAS。

阅读这篇文章你将会了解到：

* 什么是 CAS
* CAS 实现原理是什么？
* CAS 在现实中的应用
  * 自旋锁
  * 原子类型
  * 限流器
* CAS 的缺点

<!--more-->

# 什么是 CAS

CAS: 全称Compare and swap，字面意思:”比较并交换“，一个 CAS 涉及到以下操作：

> 我们假设内存中的原数据V，旧的预期值A，需要修改的新值B。
> 1. 比较 A 与 V 是否相等。（比较）
> 2. 如果比较相等，将 B 写入 V。（交换）
> 3. 返回操作是否成功。

当多个线程同时对某个资源进行CAS操作，只能有一个线程操作成功，但是并不会阻塞其他线程,其他线程只会收到操作失败的信号。可见 CAS 其实是一个乐观锁。

# CAS 是怎么实现的

跟随AtomInteger的代码我们一路往下，就能发现最终调用的是 `sum.misc.Unsafe` 这个类。看名称 Unsafe 就是一个不安全的类，这个类是利用了 Java 的类和包在可见性的的规则中的一个恰到好处处的漏洞。Unsafe 这个类为了速度，在Java的安全标准上做出了一定的妥协。

再往下寻找我们发现 Unsafe的`compareAndSwapInt` 是 Native 的方法：

```java
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

也就是说，这几个 CAS 的方法应该是使用了本地的方法。所以这几个方法的具体实现需要我们自己去 jdk 的源码中搜索。

于是我下载一个 OpenJdk 的源码继续向下探索，我们发现在 `/jdk9u/hotspot/src/share/vm/unsafe.cpp` 中有这样的代码：

```c
{CC "compareAndSetInt",   CC "(" OBJ "J""I""I"")Z",  FN_PTR(Unsafe_CompareAndSetInt)},
```

这个涉及到，JNI 的调用，感兴趣的同学可以自行学习。我们搜索 `Unsafe_CompareAndSetInt`后发现:

```c
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSetInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x)) {
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *)index_oop_from_field_offset_long(p, offset);

  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
} UNSAFE_END
```

最终我们终于看到了核心代码 `Atomic::cmpxchg`。

继续向底层探索，在文件`java/jdk9u/hotspot/src/os_cpu/linux_x86/vm/atomic_linux_x86.hpp`有这样的代码:

```c
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value, cmpxchg_memory_order order) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}
```

我们通过文件名可以知道，针对不同的操作系统,JVM 对于 Atomic::cmpxchg 应该有不同的实现。由于我们服务基本都是使用的是64位linux，所以我们就看看linux_x86 的实现。

我们继续看代码：

* `__asm__` 的意思是这个是一段内嵌汇编代码。也就是在 C 语言中使用汇编代码。
* 这里的 `volatile`和 JAVA 有一点类似，但不是为了内存的可见性，而是告诉编译器对访问该变量的代码就不再进行优化。
* `LOCK_IF_MP(%4)` 的意思就比较简单，就是如果操作系统是多核的，那就增加一个 LOCK。
* `cmpxchgl` 就是汇编版的“比较并交换”。但是我们知道比较并交换，有三个步骤，不是原子的。所以在多核情况下加一个 LOCK，由CPU硬件保证他的原子性。
* 我们再看看 LOCK 是怎么实现的呢？我们去Intel的官网上看看，可以知道LOCK在的早期实现是直接将 cup 的总线阻塞，这样的实现可见效率是很低下的。后来优化为X86 cpu 有锁定一个特定内存地址的能力，当这个特定内存地址被锁定后，它就可以阻止其他的系统总线读取或修改这个内存地址。

关于 CAS 的底层探索我们就到此为止。我们总结一下 JAVA 的 cas 是怎么实现的：

* java 的 cas 利用的的是 unsafe 这个类提供的 cas 操作。
* unsafe 的cas 依赖了的是 jvm 针对不同的操作系统实现的 Atomic::cmpxchg
* Atomic::cmpxchg 的实现使用了汇编的 cas 操作，并使用 cpu 硬件提供的 lock信号保证其原子性

# CAS 的应用

了解了 CAS 的原理我们继续就看看 CAS 的应用：

## 自旋锁

```java
public class SpinLock {

  private AtomicReference<Thread> sign =new AtomicReference<>();

  public void lock(){
    Thread current = Thread.currentThread();
    while(!sign .compareAndSet(null, current)){
    }
  }

  public void unlock (){
    Thread current = Thread.currentThread();
    sign .compareAndSet(current, null);
  }
}
```

所谓自旋锁，我觉得这个名字相当的形象，在lock()的时候，一直while()循环，直到 cas 操作成功为止。

## AtomicInteger 的 incrementAndGet()

```java
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

与自旋锁有异曲同工之妙，就是一直while，直到操作成功为止。

## 令牌桶限流器

所谓令牌桶限流器，就是系统以恒定的速度向桶内增加令牌。每次请求前从令牌桶里面获取令牌。如果获取到令牌就才可以进行访问。当令牌桶内没有令牌的时候，拒绝提供服务。我们来看看 `eureka` 的限流器是如何使用 CAS 来维护多线程环境下对 token 的增加和分发的。

```java
public class RateLimiter {

    private final long rateToMsConversion;

    private final AtomicInteger consumedTokens = new AtomicInteger();
    private final AtomicLong lastRefillTime = new AtomicLong(0);

    @Deprecated
    public RateLimiter() {
        this(TimeUnit.SECONDS);
    }

    public RateLimiter(TimeUnit averageRateUnit) {
        switch (averageRateUnit) {
            case SECONDS:
                rateToMsConversion = 1000;
                break;
            case MINUTES:
                rateToMsConversion = 60 * 1000;
                break;
            default:
                throw new IllegalArgumentException("TimeUnit of " + averageRateUnit + " is not supported");
        }
    }

    //提供给外界获取 token 的方法
    public boolean acquire(int burstSize, long averageRate) {
        return acquire(burstSize, averageRate, System.currentTimeMillis());
    }

    public boolean acquire(int burstSize, long averageRate, long currentTimeMillis) {
        if (burstSize <= 0 || averageRate <= 0) { // Instead of throwing exception, we just let all the traffic go
            return true;
        }

        //添加token
        refillToken(burstSize, averageRate, currentTimeMillis);

        //消费token
        return consumeToken(burstSize);
    }

    private void refillToken(int burstSize, long averageRate, long currentTimeMillis) {
        long refillTime = lastRefillTime.get();
        long timeDelta = currentTimeMillis - refillTime;

        //根据频率计算需要增加多少 token
        long newTokens = timeDelta * averageRate / rateToMsConversion;
        if (newTokens > 0) {
            long newRefillTime = refillTime == 0
                    ? currentTimeMillis
                    : refillTime + newTokens * rateToMsConversion / averageRate;

            // CAS 保证有且仅有一个线程进入填充
            if (lastRefillTime.compareAndSet(refillTime, newRefillTime)) {
                while (true) {
                    int currentLevel = consumedTokens.get();
                    int adjustedLevel = Math.min(currentLevel, burstSize); // In case burstSize decreased
                    int newLevel = (int) Math.max(0, adjustedLevel - newTokens);
                    // while true 直到更新成功为止
                    if (consumedTokens.compareAndSet(currentLevel, newLevel)) {
                        return;
                    }
                }
            }
        }
    }

    private boolean consumeToken(int burstSize) {
        while (true) {
            int currentLevel = consumedTokens.get();
            if (currentLevel >= burstSize) {
                return false;
            }

            // while true 直到没有token 或者 获取到为止
            if (consumedTokens.compareAndSet(currentLevel, currentLevel + 1)) {
                return true;
            }
        }
    }

    public void reset() {
        consumedTokens.set(0);
        lastRefillTime.set(0);
    }
}

```

所以梳理一下 CAS 在令牌桶限流器的作用。就是保证在多线程情况下，不阻塞线程的填充token 和消费token。

## 归纳

通过上面的三个应用我们归纳一下 CAS 的应用场景：

* CAS 的使用能够避免线程的阻塞。
* 多数情况下我们使用的是 while true 直到成功为止。

# CAS 缺点

1. ABA 的问题，就是一个值从A变成了B又变成了A，使用CAS操作不能发现这个值发生变化了，处理方式是可以使用携带类似时间戳的版本AtomicStampedReference
2. 性能问题，我们使用时大部分时间使用的是 while true 方式对数据的修改，直到成功为止。优势就是相应极快，但当线程数不停增加时，性能下降明显，因为每个线程都需要执行，占用CPU时间。

# 总结

CAS 是整个编程重要的思想之一。整个计算机的实现中都有CAS的身影。微观上看汇编的 CAS 是实现操作系统级别的原子操作的基石。从编程语言角度来看 CAS 是实现多线程非阻塞操作的基石。宏观上看，在分布式系统中，我们可以使用 CAS 的思想利用类似`Redis`的外部存储，也能实现一个分布式锁。

从某个角度来说架构就将微观的实现放大，或者底层思想就是将宏观的架构进行微缩。计算机的思想是想通的，所以说了解底层的实现可以提升架构能力，提升架构的能力同样可加深对底层实现的理解。计算机知识浩如烟海，但是套路有限。抓住基础的几个套路突破，从思想和思维的角度学习计算机知识。不要将自己的精力花费在不停的追求新技术的脚步上，跟随‘start guide line’只能写一个demo，所得也就是一个demo而已。

停下脚步，回顾基础和经典或许对于技术的提升更大一些。

希望这篇文章对大家有所提升。

欢迎关注我的微信公众号
![二维码](https://ws3.sinaimg.cn/large/006tNc79ly1fo3oqp60v3j307607674r.jpg)
