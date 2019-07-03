> 显式锁 ReentrantLock 和同步工具类的实现基础都是 AQS，所以合起来一齐研究。

### 什么是 AQS

AQS 即是 AbstractQueuedSynchronizer，一个用来构建锁和同步工具的框架，包括常用的 ReentrantLock、CountDownLatch、Semaphore 等。

AQS 没有锁之类的概念，它有个 state 变量，是个 int 类型，在不同场合有着不同含义。本文研究的是锁，为了好理解，姑且先把 state 当成锁。

AQS 围绕 state 提供两种基本操作 “获取” 和“释放”，有条双向队列存放阻塞的等待线程，并提供一系列判断和处理方法，简单说几点：

*   state 是独占的，还是共享的；
*   state 被获取后，其他线程需要等待；
*   state 被释放后，唤醒等待线程；
*   线程等不及时，如何退出等待。

至于线程是否可以获得 state，如何释放 state，就不是 AQS 关心的了，要由子类具体实现。

直接分析 AQS 的代码会比较难明白，所以结合子类 ReentrantLock 来分析。AQS 的功能可以分为独占和共享，ReentrantLock 实现了独占功能，是本文分析的目标。

### ReentrantLock 对比 synchronized

```java
Lock lock = new ReentranLock();
lock.lock();
try{
    //do something
}finally{
    lock.unlock();
}
```

ReentrantLock 实现了 Lock 接口，加锁和解锁都需要显式写出，注意一定要在适当时候 unlock。

和 synchronized 相比，ReentrantLock 用起来会复杂一些。在基本的加锁和解锁上，两者是一样的，所以无特殊情况下，推荐使用 synchronized。==ReentrantLock 的优势在于它更灵活、更强大，除了常规的 lock()、unlock() 之外，还有 lockInterruptibly()、tryLock() 方法，支持中断、超时==。

### 公平锁和非公平锁

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

ReentrantLock 的内部类 Sync 继承了 AQS，分为公平锁 FairSync 和非公平锁 NonfairSync。

*   公平锁：线程获取锁的顺序和调用 lock 的顺序一样，FIFO；
*   非公平锁：线程获取锁的顺序和调用 lock 的顺序无关，全凭运气。

ReentrantLock 默认使用非公平锁是基于性能考虑，公平锁为了保证线程规规矩矩地排队，需要增加阻塞和唤醒的时间开销。如果直接插队获取非公平锁，跳过了对队列的处理，速度会更快。

### 尝试获取锁

```java
final void lock() { acquire(1);}

public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

先来看公平锁的实现，lock 方法很简单的一句话调用 AQS 的 acquire 方法：

```java
protected boolean tryAcquire(int arg) {    
        throw new UnsupportedOperationException();
}
```

噢，AQS 的 tryAcquire 不能直接调用，因为是否获取锁成功是由子类决定的，直接看 ReentrantLock 的 tryAcquire 的实现。

```java
protected final boolean tryAcquire(int acquires) {
   final Thread current = Thread.currentThread();
   int c = getState();
   // 判断AQS的state是否等于0，等于0表示锁没有人占有
   if (c == 0) {
       // hasQueuedPredecessors 判断队列是否有排在前面的线程在等待锁，没有的话调用 compareAndSetState 使用 cas 的方式修改 state，传入的 acquires 写死是 1
       if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
           // 将线程记录为独占锁的线程
           setExclusiveOwnerThread(current);
           return true;
       }
   }
   // 获取锁的线程是否为当前线程，实现重入
   else if (current == getExclusiveOwnerThread()) {
       int nextc = c + acquires;
       if (nextc < 0)
           throw new Error("Maximum lock count exceeded");
       setState(nextc);
       return true;
   }
   return false;
}
```

### 线程进入等待队列

AQS 内部有一条双向的队列存放等待线程，节点是 Node 对象。每个 Node 维护了线程、前后 Node 的指针和等待状态等参数。

线程在加入队列之前，需要包装进 Node，调用方法是 addWaiter：

```java
/** Marker to indicate a node is waiting in shared mode */
static final Node SHARED = new Node();
/** Marker to indicate a node is waiting in exclusive mode */
static final Node EXCLUSIVE = null;

private Node addWaiter(Node mode) {
   Node node = new Node(Thread.currentThread(), mode);
   // Try the fast path of enq; backup to full enq on failure
   Node pred = tail;
   if (pred != null) {
       node.prev = pred;
       if (compareAndSetTail(pred, node)) {
           pred.next = node;
           return node;
       }
   }
   enq(node);
   return node;
}
```

每个 Node 需要标记是独占的还是共享的，由传入的 mode 决定，ReentrantLock 自然是使用独占模式 Node.EXCLUSIVE。

创建好 Node 后，如果队列不为空，使用 cas 的方式将 Node 加入到队列尾。注意，这里只执行了一次修改操作，并且可能因为并发的原因失败。因此修改失败的情况和队列为空的情况，需要进入 enq。

```
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

enq 是个死循环，保证 Node 一定能插入队列。注意到，当队列为空时，会先为头节点创建一个空的 Node，因为**头节点代表获取了锁的线程**，现在还没有，所以先空着。关于enq代码重复的问题参见：https://www.jianshu.com/p/0ad6b6f2099d 。（至今仍无法理解，建议不要太较真）

### 阻塞等待线程

线程加入队列后，下一步是调用 acquireQueued 阻塞线程。

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //1
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //2
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

标记 1 是线程唤醒后尝试获取锁的过程。如果前一个节点正好是 head，表示自己排在第一位，可以马上调用 tryAcquire 尝试。如果获取成功就简单了，直接修改自己为 head。这步是实现公平锁的核心，保证释放锁时，由下个排队线程获取锁。（看到线程解锁时，再看回这里啦）

标记 2 是线程获取锁失败的处理。这个时候，线程可能等着下一次获取，也可能不想要了，Node 变量 waitState 描述了线程的等待状态，一共四种情况：

```
static final int CANCELLED =  1;   //取消
static final int SIGNAL    = -1;     //下个节点需要被唤醒
static final int CONDITION = -2;  //线程在等待条件触发
static final int PROPAGATE = -3; //（共享锁）状态需要向后传播


```

shouldParkAfterFailedAcquire 传入当前节点和前节点，根据前节点的状态，判断线程是否需要阻塞。

```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)
      return true;
  if (ws > 0) {
      do {
          node.prev = pred = pred.prev;
      } while (pred.waitStatus > 0);
      pred.next = node;
  } else {
      compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}


```

*   前节点状态是 SIGNAL 时，当前线程需要阻塞；
*   前节点状态是 CANCELLED 时，通过循环将当前节点之前所有取消状态的节点移出队列；
*   前节点状态是其他状态时，需要设置前节点为 SIGNAL。

如果线程需要阻塞，由 parkAndCheckInterrupt 方法进行操作。

```
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}


```

parkAndCheckInterrupt 使用了 LockSupport，和 cas 一样，最终使用 UNSAFE 调用 Native 方法实现线程阻塞（以后有机会就分析下 LockSupport 的原理，park 和 unpark 方法作用类似于 wait 和 notify）。最后返回线程唤醒后的中断状态，关于中断，后文会分析。

到这里总结一下获取锁的过程：线程去竞争一个锁，可能成功也可能失败。成功就直接持有资源，不需要进入队列；失败的话进入队列阻塞，等待唤醒后再尝试竞争锁。

### 释放锁

通过上面详细的获取锁过程分析，释放锁过程大概可以猜到：头节点是获取锁的线程，先移出队列，再通知后面的节点获取锁。

```
public void unlock() {
    sync.release(1);
}


```

ReentrantLock 的 unlock 方法很简单地调用了 AQS 的 release：

```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}


```

和 lock 的 tryAcquire 一样，unlock 的 tryRelease 同样由 ReentrantLock 实现：

```
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}


```

因为锁是可以重入的，所以每次 lock 会让 state 加 1，对应地每次 unlock 要让 state 减 1，直到为 0 时将独占线程变量设置为空，返回标记是否彻底释放锁。

最后，调用 unparkSuccessor 将头节点的下个节点唤醒：

```
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}


```

寻找下个待唤醒的线程是从队列尾向前查询的，找到线程后调用 LockSupport 的 unpark 方法唤醒线程。被唤醒的线程重新执行 acquireQueued 里的循环，就是上文关于 acquireQueued 标记 1 部分，线程重新尝试获取锁。

### 中断锁

```
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}


```

在 acquire 里还有最后一句代码调用了 selfInterrupt，功能很简单，对当前线程产生一个中断请求。

为什么要这样操作呢？因为 LockSupport.park 阻塞线程后，有两种可能被唤醒。

第一种情况，前节点是头节点，释放锁后，会调用 LockSupport.unpark 唤醒当前线程。整个过程没有涉及到中断，最终 acquireQueued 返回 false 时，不需要调用 selfInterrupt。

第二种情况，LockSupport.park 支持响应中断请求，能够被其他线程通过 interrupt() 唤醒。但这种唤醒并没有用，因为线程前面可能还有等待线程，在 acquireQueued 的循环里，线程会再次被阻塞。parkAndCheckInterrupt 返回的是 Thread.interrupted()，不仅返回中断状态，还会清除中断状态，保证阻塞线程忽略中断。最终 acquireQueued 返回 true 时，真正的中断状态已经被清除，需要调用 selfInterrupt 维持中断状态。

因此普通的 lock 方法并不能被其他线程中断，ReentrantLock 是可以支持中断，需要使用 lockInterruptibly。

两者的逻辑基本一样，不同之处是 parkAndCheckInterrupt 返回 true 时，lockInterruptibly 直接 throw new InterruptedException()。

### 非公平锁

分析完公平锁的实现，还剩下非公平锁，主要区别是获取锁的过程不同。

```
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}


```

在 NonfairSync 的 lock 方法里，第一步直接尝试将 state 修改为 1，很明显，这是抢先获取锁的过程。如果修改 state 失败，则和公平锁一样，调用 acquire。

```
final boolean nonfairTryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {
      if (compareAndSetState(0, acquires)) {
          setExclusiveOwnerThread(current);
          return true;
      }
  }
  else if (current == getExclusiveOwnerThread()) {
      int nextc = c + acquires;
      if (nextc < 0) // overflow
          throw new Error("Maximum lock count exceeded");
      setState(nextc);
      return true;
  }
  return false;
}


```

nonfairTryAcquire 和 tryAcquire 乍一看几乎一样，差异只是缺少调用 hasQueuedPredecessors。这点体验出公平锁和非公平锁的不同，公平锁会关注队列里排队的情况，老老实实按照 FIFO 的次序；非公平锁只要有机会就抢占，才不管排队的事。

### 总结

从 ReentrantLock 的实现完整分析了 AQS 的独占功能，总的来讲并不复杂。别忘了 AQS 还有共享功能，下一篇是 -- [分析 CountDownLatch 的实现原理](https://www.jianshu.com/p/7c7a5df5bda6)。