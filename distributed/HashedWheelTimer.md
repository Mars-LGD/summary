本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 https://albenw.github.io/posts/ec8df8c/

## HashedWheelTimer时间轮原理分析

 发表于 2019-03-28 |  分类于 [java ](https://albenw.github.io/categories/java/)， [netty ](https://albenw.github.io/categories/java/netty/)|  阅读次数: 33

HashedWheelTimer时间轮是一个高性能，低消耗的数据结构，它适合用非准实时，延迟的短平快任务，例如心跳检测。

## 概要

时间轮是一种非常惊艳的数据结构。其在Linux内核中使用广泛，是Linux内核定时器的实现方法和基础之一。Netty内部基于时间轮实现了一个HashedWheelTimer来优化I/O超时的检测，本文将详细分析HashedWheelTimer的使用及原理。

## 背景

由于Netty动辄管理100w+的连接，每一个连接都会有很多超时任务。比如发送超时、心跳检测间隔等，如果每一个定时任务都启动一个Timer，不仅低效，而且会消耗大量的资源。
在Netty中的一个典型应用场景是判断某个连接是否idle，如果idle（如客户端由于网络原因导致到服务器的心跳无法送达），则服务器会主动断开连接，释放资源。得益于Netty NIO的优异性能，基于Netty开发的服务器可以维持大量的长连接，单台8核16G的云主机可以同时维持几十万长连接，及时掐掉不活跃的连接就显得尤其重要。

看看官方文档说明：

> A optimized for approximated I/O timeout scheduling.
> You can increase or decrease the accuracy of the execution timing by
>
> - specifying smaller or larger tick duration in the constructor. In most
> - network applications, I/O timeout does not need to be accurate. Therefore,
> - the default tick duration is 100 milliseconds and you will not need to try
> - different configurations in most cases.

大概意思是一种对“适当”I/O超时调度的优化。因为I/O timeout这种任务对时效性不需要准确。

这种方案也不是Netty凭空造出来的，而是根据George Varghese和Tony Lauck在1996年的论文实现的，有兴趣的可以阅读一下。
[论文下载](http://cseweb.ucsd.edu/users/varghese/PAPERS/twheel.ps.Z)
[论文PPT](https://www.cse.wustl.edu/~cdgill/courses/cs6874/TimingWheels.ppt)

## 应用场景

HashedWheelTimer本质是一种类似延迟任务队列的实现，那么它的特点就是上述所说的，适用于对时效性不高的，可快速执行的，大量这样的“小”任务，能够做到高性能，低消耗。
例如：

- 心跳检测
- session、请求是否timeout
  业务场景则有：
- 用户下单后发短信
- 下单之后15分钟，如果用户不付款就自动取消订单

## 简单使用

如果之前没用过，先看看用法有一个大体的感受，

```
@Slf4j
public class HashedWheelTimerTest {

    private CountDownLatch countDownLatch = new CountDownLatch(2);

    @Test
    public void test1() throws Exception {
        //定义一个HashedWheelTimer，有16个格的轮子，每一秒走一个一个格子
        HashedWheelTimer timer = new HashedWheelTimer(1, TimeUnit.SECONDS, 16);
        //把任务加到HashedWheelTimer里，到了延迟的时间就会自动执行
        timer.newTimeout((timeout) -> {
            log.info("task1 execute");
            countDownLatch.countDown();
        }, 500, TimeUnit.MILLISECONDS);
        timer.newTimeout((timeout) -> {
            log.info("task2 execute");
            countDownLatch.countDown();
        }, 2, TimeUnit.SECONDS);
        countDownLatch.await();
        timer.stop();
    }
}
```



需要引入`netty-all.jar`包
使用上跟`ScheduledExecutorService`差不多。

## 实现原理

源码基于`netty-all.4.1.34.Final`

### 数据结构

时间轮其实就是一种环形的数据结构，可以想象成时钟，分成很多格子，一个格子代码一段时间（这个时间越短，Timer的精度越高）。并用一个链表报错在该格子上的到期任务，同时一个指针随着时间一格一格转动，并执行相应格子中的到期任务。任务通过取摸决定放入那个格子。如下图所示：

[![upload successful](../assets/HashedWheelTimer%E6%97%B6%E9%97%B4%E8%BD%AE%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90__0-20190802172837900.png)](https://albenw.github.io/images/HashedWheelTimer时间轮原理分析__0.png)

假设一个格子是1秒，则整个wheel能表示的时间段为8s，假如当前指针指向2，此时需要调度一个3s后执行的任务，显然应该加入到(2+3=5)的方格中，指针再走3次就可以执行了；如果任务要在10s后执行，应该等指针走完一个round零2格再执行，因此应放入4，同时将round（1）保存到任务中。检查到期任务时应当只执行round为0的，格子上其他任务的round应减1。

再回头看看构造方法的三个参数分别代表

- tickDuration
  每一tick的时间
- timeUnit
  tickDuration的时间单位
- ticksPerWheel
  就是轮子一共有多个格子，即要多少个tick才能走完这个wheel一圈。

对于HashedWheelTimer的数据结构在介绍完源码之后有图解。

### 初始化

HashedWheelTimer整体代码不难，慢慢看应该都可以看懂
我们从HashedWheelTimer的构造方法入手，先说明一下

#### 构造方法

```
//附上文档说明，自行阅读
/**
 * Creates a new timer.
 *
 * @param threadFactory        a {@link ThreadFactory} that creates a
 *                             background {@link Thread} which is dedicated to
 *                             {@link TimerTask} execution.
 * @param tickDuration         the duration between tick
 * @param unit                 the time unit of the {@code tickDuration}
 * @param ticksPerWheel        the size of the wheel
 * @param leakDetection        {@code true} if leak detection should be enabled always,
 *                             if false it will only be enabled if the worker thread is not
 *                             a daemon thread.
 * @param  maxPendingTimeouts  The maximum number of pending timeouts after which call to
 *                             {@code newTimeout} will result in
 *                             {@link java.util.concurrent.RejectedExecutionException}
 *                             being thrown. No maximum pending timeouts limit is assumed if
 *                             this value is 0 or negative.
 * @throws NullPointerException     if either of {@code threadFactory} and {@code unit} is {@code null}
 * @throws IllegalArgumentException if either of {@code tickDuration} and {@code ticksPerWheel} is &lt;= 0
 */
//threadFactory默认是用Executors.defaultThreadFactory()，太懒了
//tickDuration，unit，ticksPerWheel核心参数，之前已经说过了
//leakDetection内存泄漏检查
//maxPendingTimeouts准备执行的任务数，默认是-1，即不限制。如果并发量真的很高，可以设置一下，防止OOM
public HashedWheelTimer(
        ThreadFactory threadFactory,
        long tickDuration, TimeUnit unit, int ticksPerWheel, boolean leakDetection,
        long maxPendingTimeouts) {

    if (threadFactory == null) {
        throw new NullPointerException("threadFactory");
    }
    if (unit == null) {
        throw new NullPointerException("unit");
    }
    if (tickDuration <= 0) {
        throw new IllegalArgumentException("tickDuration must be greater than 0: " + tickDuration);
    }
    if (ticksPerWheel <= 0) {
        throw new IllegalArgumentException("ticksPerWheel must be greater than 0: " + ticksPerWheel);
    }

    //初始化时间轮（1）
    wheel = createWheel(ticksPerWheel);
    mask = wheel.length - 1;
    
    this.tickDuration = unit.toNanos(tickDuration);

    //要求tickDuration * wheel.length < Long.MAX_VALUE，我猜测是因为担心有类似的计算而导致溢出（但实际我没找到）
    if (this.tickDuration >= Long.MAX_VALUE / wheel.length) {
        throw new IllegalArgumentException(String.format(
                "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
                tickDuration, Long.MAX_VALUE / wheel.length));
    }
    //创建Work执行线程（2）
    workerThread = threadFactory.newThread(worker);
    //默认是启动内存泄露检测（我还不是很清楚具体原理）
    leak = leakDetection || !workerThread.isDaemon() ? leakDetector.track(this) : null;

    this.maxPendingTimeouts = maxPendingTimeouts;
    //HashedWheelTimer实例数限制，因为HashedWheelTimer是一个非常消耗内存的对象，如果超过64个则会警告
    if (INSTANCE_COUNTER.incrementAndGet() > INSTANCE_COUNT_LIMIT &&
        WARNED_TOO_MANY_INSTANCES.compareAndSet(false, true)) {
        reportTooManyInstances();
    }
}
```

#### createWheel

```
private static HashedWheelBucket[] createWheel(int ticksPerWheel) {
    if (ticksPerWheel <= 0) {
        throw new IllegalArgumentException(
                "ticksPerWheel must be greater than 0: " + ticksPerWheel);
    }
    if (ticksPerWheel > 1073741824) {
        throw new IllegalArgumentException(
                "ticksPerWheel may not be greater than 2^30: " + ticksPerWheel);
    }
    //格子数向2的N次方数靠齐
    ticksPerWheel = normalizeTicksPerWheel(ticksPerWheel);
    HashedWheelBucket[] wheel = new HashedWheelBucket[ticksPerWheel];
    for (int i = 0; i < wheel.length; i ++) {
        //初始化每一个bucket
        wheel[i] = new HashedWheelBucket();
    }
    return wheel;
}
//循环直到小于2的N次方
private static int normalizeTicksPerWheel(int ticksPerWheel) {
    int normalizedTicksPerWheel = 1;
    while (normalizedTicksPerWheel < ticksPerWheel) {
        normalizedTicksPerWheel <<= 1;
    }
    return normalizedTicksPerWheel;
}
```

wheel其实是一个bucket数组

#### HashedWheelBucket

```
/**
* Bucket that stores HashedWheelTimeouts. These are stored in a linked-list like datastructure to allow easy
* removal of HashedWheelTimeouts in the middle. Also the HashedWheelTimeout act as nodes themself and so no
* extra object creation is needed.
*/
private static final class HashedWheelBucket {
    // Used for the linked-list datastructure
    private HashedWheelTimeout head;
    private HashedWheelTimeout tail;
}
```

bucket的结构是一个带有头节点指针和尾节点指针的linked-list

### newTimeout

HashedWheelTimer初始化后，看看怎样增加一个任务（在HashedWheelTimer内部统一叫HashedWheelTimeout，缩写timeout，从名字已经可以看出HashedWheelTimer的作用）

```
public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
    if (task == null) {
        throw new NullPointerException("task");
    }
    if (unit == null) {
        throw new NullPointerException("unit");
    }
    //记录代处理数，这个数字没有实际作用，或可用于监控
    long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();
    //如果大于maxPendingTimeouts则报错，默认-1，即不限制
    if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {
        pendingTimeouts.decrementAndGet();
        throw new RejectedExecutionException("Number of pending timeouts ("
            + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
            + "timeouts (" + maxPendingTimeouts + ")");
    }
    //（1）
    start();

    //计算这个timeout的执行时间，公式=当前时间 + 延迟时间 - wheelTimer的启动时间，单位纳秒，很直观吧
    long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
    //初始化timeout
    HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
    //timeouts是一个MpscQueue队列。这里注意了，新加入的timeout不是立即加到wheel中的，而是先加入到一个队列，在tick的时候再从队列取出来加入到wheel。
    //原因是，我猜测一是做缓冲作用，二是避免在插入timeout和执行timeout时有并发的冲突，特别是对linked-list的操作，如果采用加锁的话，而执行该bucket所有timeout的时间不能保证，可能反而会阻塞到用户。
    timeouts.add(timeout);
    return timeout;
}

//（1）
//start方法为了保证wheelTimer在运行的状态（如果是关闭状态，那么直接抛异常出去），且wheelTimer已经初始化完成
//大家是不是很好奇wheelTimer的初始化为什么要在newTimeout，即加入第一个timeout后才去做。可以沿着延迟初始化或懒加载的思路去想，如果一个HashedWheelTimer初始化后一直没有timeout加入，在那里空转而白白浪费CPU资源就不好了。
public void start() {
    switch (WORKER_STATE_UPDATER.get(this)) {
        case WORKER_STATE_INIT:
            if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                //启动Worker
                workerThread.start();
            }
            break;
        case WORKER_STATE_STARTED:
            break;
        case WORKER_STATE_SHUTDOWN:
            throw new IllegalStateException("cannot be started once stopped");
        default:
            throw new Error("Invalid WorkerState");
    }

    //因为Worker是异步执行的，会一直等待Worker的初始化
    while (startTime == 0) {
        try {
            //是一个CountDownLatch
            startTimeInitialized.await();
        } catch (InterruptedException ignore) {
            // Ignore - it will be ready very soon.
        }
    }
}
```



#### HashedWheelTimeout

timeout的结构

```
private static final class HashedWheelTimeout implements Timeout {
    //timeout的状态有初始化，取消，已执行
    private static final int ST_INIT = 0;
    private static final int ST_CANCELLED = 1;
    private static final int ST_EXPIRED = 2;
    //用来支持CAS操作原子类
    private static final AtomicIntegerFieldUpdater<HashedWheelTimeout> STATE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(HashedWheelTimeout.class, "state");
    
    //所属timer
    private final HashedWheelTimer timer;
    //需要执行的Runnable
    private final TimerTask task;
    //执行时间
    private final long deadline;

    @SuppressWarnings({"unused", "FieldMayBeFinal", "RedundantFieldInitialization" })
    private volatile int state = ST_INIT;

    //时间轮的层数
    long remainingRounds;

    //在linked-list中该节点指向的前后节点
    HashedWheelTimeout next;
    HashedWheelTimeout prev;

    // The bucket to which the timeout was added
    HashedWheelBucket bucket;

    HashedWheelTimeout(HashedWheelTimer timer, TimerTask task, long deadline) {
        this.timer = timer;
        this.task = task;
        this.deadline = deadline;
    }
}
```



### Worker run

一个HashedWheelTimer只有一个Worker线程。看看Worker的初始化

```
private final Worker worker = new Worker();
```



Worker继承Runnable，我们看run方法

```
public void run() {
        //获取启动的时间作为开始时间
        startTime = System.nanoTime();
        if (startTime == 0) {
            // We use 0 as an indicator for the uninitialized value here, so make sure it's not 0 when initialized.
            startTime = 1;
        }

        // Notify the other threads waiting for the initialization at start().
        startTimeInitialized.countDown();

        do {
            //等待下一个tick（1）
            final long deadline = waitForNextTick();
            if (deadline > 0) {
                //找到该处理的bucket下标，因为wheel.length是2的N次方，mask为length - 1，所以这里相当于取模操作，性能比%高
                int idx = (int) (tick & mask);
                //处理掉已经取消的timeout（2）
                processCancelledTasks();
                //找到当前tick对应哪个bucket
                HashedWheelBucket bucket =
                        wheel[idx];
                //把队列的timeout放到wheel里（3）
                transferTimeoutsToBuckets();
                //处理当前bucket所有的timeout（4）
                bucket.expireTimeouts(deadline);
                //tick加一
                tick++;
            }
            //注意一下处理的顺序，也讲究的，先处理掉被取消的timeout，再把队列的加进来，再处理，后面两步不能反转，因为有可能队列里的timeout是下一tick执行的
        
        //循环直到wheelTimer被关闭
        } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);

        //wheelTimer被关闭后的处理
        //取出每一个bucket里还没被执行的timeout，放到unprocessedTimeouts中
        for (HashedWheelBucket bucket: wheel) {
            bucket.clearTimeouts(unprocessedTimeouts);
        }
        //把队列里的timeout放到unprocessedTimeouts中
        //PS：这个unprocessedTimeouts暂时只是做记录用，做监控时或可用到
        for (;;) {
            HashedWheelTimeout timeout = timeouts.poll();
            if (timeout == null) {
                break;
            }
            if (!timeout.isCancelled()) {
                unprocessedTimeouts.add(timeout);
            }
        }
        //还要处理掉中间被取消的timeout
        processCancelledTasks();
    }

    //（1）
    private long waitForNextTick() {
        //计算下一个tick的时间，很简单一看就懂
        long deadline = tickDuration * (tick + 1);

        for (;;) {
            final long currentTime = System.nanoTime() - startTime;
            //根据当前计算需要sleep的时间。这里加了999999是因为向上取整了1毫秒，假如距离下一个tick的时间为2000010纳秒，那如果sleep 2毫秒是不够的，所以需要多sleep 1毫秒。
            long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;

            //sleepTimeMs <=0 说明下一个tick的时间到了，说明上一个tick执行的时间“太久”了，所以直接返回就好了，不需要sleep
            if (sleepTimeMs <= 0) {
                //currentTime == Long.MIN_VALUE 这个判断不是很理解
                if (currentTime == Long.MIN_VALUE) {
                    return -Long.MAX_VALUE;
                } else {
                    return currentTime;
                }
            }

            // Check if we run on windows, as if thats the case we will need
            // to round the sleepTime as workaround for a bug that only affect
            // the JVM if it runs on windows.
            //
            // See https://github.com/netty/netty/issues/356
            //这里是为了处理在windows系统上的一个bug，如果sleep不够10ms则要取整
            if (PlatformDependent.isWindows()) {
                sleepTimeMs = sleepTimeMs / 10 * 10;
            }
            //直接sleep等待
            try {
                Thread.sleep(sleepTimeMs);
            } catch (InterruptedException ignored) {
                //Worker被中断，如果是关闭了则返回负数，表示不会执行下一个tick
                if (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
                    return Long.MIN_VALUE;
                }
            }
        }
    }

    //（2）
    private void processCancelledTasks() {
        for (;;) {
            //cancelledTimeouts也是一个MpscQueue，调用timeout的cancel方法会把timeout加进去
            //把取消的timeout取出来
            HashedWheelTimeout timeout = cancelledTimeouts.poll();
            if (timeout == null) {
                // all processed
                break;
            }
            try {
                //移除掉（2.1）
                timeout.remove();
            } catch (Throwable t) {
                if (logger.isWarnEnabled()) {
                    logger.warn("An exception was thrown while process a cancellation task", t);
                }
            }
        }
    }

    //（2.1）
    void remove() {
        HashedWheelBucket bucket = this.bucket;
        if (bucket != null) {
            //调用bucket的remove方法（2.2）
            bucket.remove(this);
        } else {
            timer.pendingTimeouts.decrementAndGet();
        }
    }

    //（2.2）
    //类似链表删除节点的操作
    public HashedWheelTimeout remove(HashedWheelTimeout timeout) {
        HashedWheelTimeout next = timeout.next;
        // remove timeout that was either processed or cancelled by updating the linked-list
        if (timeout.prev != null) {
            timeout.prev.next = next;
        }
        if (timeout.next != null) {
            timeout.next.prev = timeout.prev;
        }

        if (timeout == head) {
            // if timeout is also the tail we need to adjust the entry too
            if (timeout == tail) {
                tail = null;
                head = null;
            } else {
                head = next;
            }
        } else if (timeout == tail) {
            // if the timeout is the tail modify the tail to be the prev node.
            tail = timeout.prev;
        }
        // null out prev, next and bucket to allow for GC.
        timeout.prev = null;
        timeout.next = null;
        timeout.bucket = null;
        timeout.timer.pendingTimeouts.decrementAndGet();
        return next;
    }

    //（3）
    private void transferTimeoutsToBuckets() {
        //最多取队列的100000的元素出来
        for (int i = 0; i < 100000; i++) {
            HashedWheelTimeout timeout = timeouts.poll();
            if (timeout == null) {
                // all processed
                break;
            }
            //如果timeout被取消了则不做处理
            if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
                // Was cancelled in the meantime.
                continue;
            }
            //计算位于实践论的层数
            long calculated = timeout.deadline / tickDuration;
            timeout.remainingRounds = (calculated - tick) / wheel.length;
            //就是timeout已经到期了，也不能放到之前的tick中
            final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.
            //计算所在bucket下标，并放进去
            int stopIndex = (int) (ticks & mask);

            HashedWheelBucket bucket = wheel[stopIndex];
            //又是类似链表插入节点的操作
            bucket.addTimeout(timeout);
        }
    }

    //（4）
    public void expireTimeouts(long deadline) {
        HashedWheelTimeout timeout = head;
        //把bucket的所有timeout取出来执行
        while (timeout != null) {
            HashedWheelTimeout next = timeout.next;
            if (timeout.remainingRounds <= 0) {
                next = remove(timeout);
                if (timeout.deadline <= deadline) {
                    //timeout的真正执行
                    timeout.expire();
                } else {
                    // The timeout was placed into a wrong slot. This should never happen.
                    throw new IllegalStateException(String.format(
                            "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
                }
            //该timeout被取消了则移除掉    
            } else if (timeout.isCancelled()) {
                next = remove(timeout);
            //否则层数减一，等待下一轮的到来
            } else {
                timeout.remainingRounds --;
            }
            timeout = next;
        }
    }
```



### 图解

画了个图给大家体会一下
[![upload successful](../assets/HashedWheelTimer%E6%97%B6%E9%97%B4%E8%BD%AE%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90__2-20190802172838065.png)](https://albenw.github.io/images/HashedWheelTimer时间轮原理分析__2.png)

### MpscQueue队列

HashedWheelTimer用到的`timeouts`和`cancelledTimeouts`都是一种`MpscQueue`队列的数据结构。
MpscQueue全称Multi-Producer Single-Consumer Queue，从名字看出，是一种适合于多个生产者，单个消费者的高并发场景的高性能的，无锁的队列，原来Netty是自己实现了一个，但在最新的版本用了JCTools的，大家有兴趣可以了解一下。

### 多层时间轮

当时间跨度很大时，提升单层时间轮的 tickDuration 可以减少空转次数，但会导致时间精度变低，层级时间轮既可以避免精度降低，又避免了指针空转的次数。如果有时间跨度较长的定时任务，则可以交给层级时间轮去调度。
设想一下一个定时了 3 天，10 小时，50 分，30 秒的定时任务，在 tickDuration = 1s 的单层时间轮中，需要经过：3246060+106060+5060+30 次指针的拨动才能被执行。但在 wheel1 tickDuration = 1 天，wheel2 tickDuration = 1 小时，wheel3 tickDuration = 1 分，wheel4 tickDuration = 1 秒 的四层时间轮中，只需要经过 3+10+50+30 次指针的拨动。

如图所示:
[![upload successful](../assets/HashedWheelTimer%E6%97%B6%E9%97%B4%E8%BD%AE%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90__1-20190802172837963.png)](https://albenw.github.io/images/HashedWheelTimer时间轮原理分析__1.png)

## 缺点

HashedWheelTimer也有一些缺点，在使用场景上要注意一下

- Netty的HashedWheelTimer只支持单层的时间轮
- 当前一个任务执行时间过长的时候，会影响后续任务的到期执行时间的，也就是说其中的任务是串行执行的，所以，要求里面的任务都要短平快

## 延迟任务方案对比

HashedWheelTimer本质上也是一个延迟队列，我们跟其他延迟类解决方案对比一下

- 数据库轮询
  比较常用的一种方法，数据先保存在数据库中，然后启动一个定时Job，根据时间或状态把数据捞出来，处理后再更新回数据库。这种方式很简单，不会引入其他的技术，开发周期短。如果数据量比较大，千万级甚至更多，插入频率很高的话，上面的方式在性能上会出现一些问题，查找和更新对会占用很多时间，轮询频率高的话甚至会影响数据入库。如果数据量进一步增大，那扫数据库肯定就不行了。另一方面，对于订单这类数据，我们也许会遇到分库分表，那上述方案就会变得过于复杂，得不偿失。
  不过，优点是数据得到持久化，有问题可以查看。
- DelayQueue
  DelayQueue本质是PriorityQueue，每次插入或删除任务都要调整堆，复杂度是O(logN)，相对HashedWheelTimer的O(1)来说有性能消耗。
- ScheduledExecutorService
  其本质也是类似DelayQueue，不过ScheduledExecutorService是多线程的方式执行，可以基本保证其他任务的准时进行。ScheduledExecutorService封装较好，方便使用，还支持周期性任务。

## 总结

HashedWheelTimer时间轮是一个高性能，低消耗的数据结构，它适合用非准实时，延迟的短平快任务，例如心跳检测。

## 参考资料

https://zacard.net/2016/12/02/netty-hashedwheeltimer/
https://www.ctolib.com/topics-113116.html
https://www.cnkirito.moe/timer/