---
title: 从 LongAdder 中窥见并发组件的设计思路
date: 2018-11-27 20:42:20
tags: [java,LongAdder,并发]
---

![](https://ws4.sinaimg.cn/large/006tNbRwly1fxn57b96lsj31hc0u07tv.jpg)

最近在看阿里的 [Sentinel](https://github.com/alibaba/Sentinel) 的源码的时候。发现使用了一个类 LongAdder 来在并发环境中计数。这个时候就提出了疑问，JDK 中已经有 AtomicLong 了，为啥还要使用 LongAdder ？ AtomicLong 已经是基于 CAS 的无锁结构，已经有很好的并发表现了，为啥还要用 LongAdder ？于是赶快找来源码一探究竟。

<!--more-->

# AtomicLong 的缺陷

大家可以阅读我之前写的 [JAVA 中的 CAS](https://xilidou.com/2018/02/01/java-cas/) 详细了解 AtomicLong 的实现原理。需要注意的一点是，AtomicLong 的 Add() 是依赖自旋不断的 CAS 去累加**一个** Long 值。如果在竞争激烈的情况下，CAS 操作不断的失败，就会有大量的线程不断的自旋尝试 CAS 会造成 CPU 的极大的消耗。

# LongAdder 解决方案

通过阅读 LongAdder 的 Javadoc 我们了解到：
> This class is usually preferable to {@link AtomicLong} when multiple threads update a common sum that is used for purposes such as collecting statistics, not for fine-grained synchronization control.  Under low update contention, the two classes have similar characteristics. But under high contention, expected throughput of this class is significantly higher, at the expense of higher space consumption.

大概意思就是，LongAdder 功能类似 AtomicLong ，在低并发情况下二者表现差不多，在高并发情况下 LongAdder 的表现就会好很多。

LongAdder 到底用了什么黑科技能做到高性比 AtomicLong 还要好呢呢？对于同样的一个 add() 操作，上文说到 AtomicLong 只对一个 Long 值进行 CAS 操作。而 LongAdder 是针对 Cell 数组的某个 Cell 进行 CAS 操作 ，把线程的名字的 hash 值，作为 Cell 数组的下标，然后对 Cell[i] 的 long 进行 CAS 操作。简单粗暴的分散了高并发下的竞争压力。

# LongAdder 的实现细节

虽然原理简单粗暴，但是代码写得却相当细致和精巧。

在 `java.util.concurrent.atomic` 包下面我们可以看到 LongAdder 的源码。首先看 add() 方法的源码。

```java
    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }
```

看到这个 add() 方法，首先需要了解 Cell 是什么？

Cell 是 `java.util.concurrent.atomic` 下 `Striped64` 的一个内部类。

```java
    @sun.misc.Contended static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }

        // unsafe 机制
        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```

首先 Cell 被 @sun.misc.Contended 修饰。意思是让Java编译器和JRE运行时来决定如何填充。不理解不要紧，不影响理解。

**其实一个 Cell 的本质就是一个 volatile 修饰的 long 值，且这个值能够进行 cas 操作。**

回到我们的 add() 方法。

这里涉及四个额外的方法 casBase() , getProbe() , a.cas() , longAccumulate();

我们看名字就知道 casBase() 和 a.cas() 都是对参数的 cas 操作。 

getProbe() 的作用，就是根据当前线程 hash 出一个 int 值。

longAccumlate() 的作用比较复杂，之后我们会讲解。

所以这个 add() 操作归纳以后就是：

1. 如果 cells 数组不为空，对参数进行 casBase 操作，如果 casBase 操作失败。可能是竞争激烈，进入第二步。
2. 如果 cells 为空，直接进入 longAccumulate();
3. m = cells 数组长度减一，如果数组长度小于 1，则进入 longAccumulate()
4. 如果都没有满足以上条件，则对当前线程进行某种 hash 生成一个数组下标，对下标保存的值进行 cas 操作。如果操作失败，则说明竞争依然激烈，则进入 longAccumulate().

可见，操作的核心思想还是基于 cas。但是 cas 失败后，并不是傻乎乎的自旋，而是逐渐升级。升级的 cas 都不管用了则进入 longAccumulate() 这个方法。

下面就开始揭开 longAccumulate 的神秘面纱。

```java
    final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            //如果操作的cell 为空，double check 新建 cell
            if ((as = cells) != null && (n = as.length) > 0) {
                if ((a = as[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }

                // cas 失败 继续循环
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash

                // 如果 cell cas 成功 break
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;

                // 如果 cell 的长度已经大于等于 cpu 的数量，扩容意义不大，就不用标记冲突，重试
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                // 获取锁，上锁扩容，将冲突标记为否，继续执行    
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                // 没法获取锁，重散列，尝试其他槽
                h = advanceProbe(h);
            }

            // 获取锁，初始化 cell 数组
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }

            // 表未被初始化，可能正在初始化，回退使用 base。
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```

longAccumulate 看上去比较复杂。我们慢慢分析。

回忆一下，什么情况会进入到这个 longAccumulate 方法中， 
* cell[] 数组为空，
* cell[i] 数据的某个下标元素为空，
* casBase 失败，
* a.cas 失败，
* cell.length - 1 < 0

在 longAccumulate 中有几个标记位，我们也先理解一下
* `cellsBusy` cells 的操作标记位，如果正在修改、新建、操作 cells 数组中的元素会,会将其 cas 为 1，否则为0。
* `wasUncontended` 表示 cas 是否失败，如果失败则考虑操作升级。
* `collide` 是否冲突，如果冲突，则考虑扩容 cells 的长度。

整个 for(;;) 死循环，都是以 cas 操作成功而告终。否则则会修改上述描述的几个标记位，重新进入循环。

所以整个循环包括如下几种情况：
1. cells 不为空
    1. 如果 cell[i] 某个下标为空，则 new 一个 cell，并初始化值，然后退出
    2. 如果 cas 失败，继续循环
    3. 如果 cell 不为空，且 cell cas 成功，退出
    4. 如果 cell 的数量，大于等于 cpu 数量或者已经扩容了，继续重试。（扩容没意义）
    5. 设置 collide 为 true。
    6. 获取 cellsBusy 成功就对 cell 进行扩容，获取 cellBusy 失败则重新 hash 再重试。

2. cells 为空且获取到 cellsBusy ，init cells 数组，然后赋值退出。

3. cellsBusy 获取失败，则进行 baseCas ，操作成功退出，不成功则重试。

至此 longAccumulate 就分析完了。之所以这个方法那么复杂，我认为有两个原因
1. 是因为并发环境下要考虑各种操作的原子性，所以对于锁都进行了 double check。
2. 操作都是逐步升级，以最小的代价实现功能。

最后说说 LongAddr 的 sum() 方法，这个就很简单了。

```java
    public long sum() {
        Cell[] as = cells; Cell a;
        long sum = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

就是遍历 cell 数组，累加 value 就行。LongAdder 余下的方法就比较简单，没有什么可以讨论的了。

# LongAdder VS AtomicLong

看上去 LongAdder 性能全面超越了 AtomicLong。为什么 jdk 1.8 中还是保留了 AtomicLong 的实现呢？

其实我们可以发现，LongAdder 使用了一个 cell 列表去承接并发的 cas，以提升性能，但是 LongAdder 在统计的时候如果有并发更新，可能导致统计的数据有误差。

如果用于自增 id 的生成，就不适合使用 LongAdder 了。这个时候使用 AtomicLong 就是一个明智的选择。

而在 Sentinel 中 LongAdder 承担的只是统计任务，且允许误差。

# 总结

LongAdder 使用了一个比较简单的原理，解决了 AtomicLong 类，在极高竞争下的性能问题。但是 LongAdder 的具体实现却非常精巧和细致，分散竞争，逐步升级竞争的解决方案，相当漂亮，值得我们细细品味。




