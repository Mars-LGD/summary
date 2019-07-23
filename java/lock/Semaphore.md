> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 https://www.jianshu.com/p/0090341c6b80

> 简书 [占小狼](https://www.jianshu.com/users/90ab66c248e6/latest_articles)  
> 转载请注明原创出处，谢谢！

### 前言

JDK 的并发包中提供了几个非常有用的工具类，这些工具类给我们在业务开发过程中提供了一种并发流程控制的手段，本文会基于实际应用场景介绍如何使用 Semaphore，以及内部实现机制。

### Semaphore 是什么

Semaphore 也叫信号量，在 JDK1.5 被引入，可以用来控制同时访问特定资源的线程数量，通过协调各个线程，以保证合理的使用资源。

Semaphore 内部维护了一组虚拟的许可，许可的数量可以通过构造函数的参数指定。

*   访问特定资源前，必须使用 acquire 方法获得许可，如果许可数量为 0，该线程则一直阻塞，直到有可用许可。
*   访问资源后，使用 release 释放许可。

Semaphore 和 ReentrantLock 类似，获取许可有公平策略和非公平许可策略，默认情况下使用非公平策略。

### 应用场景

Semaphore 可以用来做流量分流，特别是对公共资源有限的场景，比如数据库连接。  
假设有这个的需求，读取几万个文件的数据到数据库中，由于文件读取是 IO 密集型任务，可以启动几十个线程并发读取，但是数据库连接数只有 10 个，这时就必须控制最多只有 10 个线程能够拿到数据库连接进行操作。这个时候，就可以使用 Semaphore 做流量控制。

```java
public class SemaphoreTest {
    private static final int COUNT = 40;
    private static Executor executor = Executors.newFixedThreadPool(COUNT);
    private static Semaphore semaphore = new Semaphore(10);
    public static void main(String[] args) {
        for (int i=0; i< COUNT; i++) {
            executor.execute(new ThreadTest.Task());
        }
    }

    static class Task implements Runnable {
        @Override
        public void run() {
            try {
                //读取文件操作
                semaphore.acquire();
                // 存数据过程
                semaphore.release();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
            }
        }
    }
}
```

### 实现原理

本文代码源于 JDK1.8  
Semaphore 实现主要基于 java 同步器 AQS，不熟悉的可以移步这里 [深入浅出 java 同步器](https://www.jianshu.com/p/d8eeb31bee5c)。

内部使用 state 表示许可数量。  
**非公平策略**

1.  acquire 实现，核心代码如下：

```java
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

acquires 值默认为 1，表示尝试获取 1 个许可，remaining 代表剩余的许可数。

*   如果 remaining < 0，表示目前没有剩余的许可。
*   当前线程进入 AQS 中的 doAcquireSharedInterruptibly 方法等待可用许可并挂起，直到被唤醒。

2.  release 实现，核心代码如下：

```java
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

releases 值默认为 1，表示尝试释放 1 个许可，next 代表如果许可释放成功，可用许可的数量。

*   通过 unsafe.compareAndSwapInt 修改 state 的值，确保同一时刻只有一个线程可以释放成功。
*   许可释放成功，当前线程进入到 AQS 的 doReleaseShared 方法，唤醒队列中等待许可的线程。

**_也许有人会有疑问，非公平性体现在哪里？_**  
当一个线程 A 执行 acquire 方法时，会直接尝试获取许可，而不管同一时刻阻塞队列中是否有线程也在等待许可，如果恰好有线程 C 执行 release 释放许可，并唤醒阻塞队列中第一个等待的线程 B，这个时候，线程 A 和线程 B 是共同竞争可用许可，不公平性就是这么体现出来的，线程 A 一点时间都没等待就和线程 B 同等对待。

**公平策略**

1.  acquire 实现，核心代码如下：

```java
protected int tryAcquireShared(int acquires) {
    for (;;) {
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

acquires 值默认为 1，表示尝试获取 1 个许可，remaining 代表剩余的许可数。  
可以看到和非公平策略相比，就多了一个对阻塞队列的检查。

*   如果阻塞队列没有等待的线程，则参与许可的竞争。
*   否则直接插入到阻塞队列尾节点并挂起，等待被唤醒。

2.  release 实现，和非公平策略一样。

### 总结

通过本文的介绍，希望大家能够了解 Semaphore 的应用场景和工作机制。

END。  
我是占小狼。  
在魔都艰苦奋斗，白天是上班族，晚上是知识服务工作者。  
读完我的文章有收获，记得关注和点赞哦，如果非要打赏，我也是不会拒绝的啦！