> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 https://www.jianshu.com/p/1161d33fc1d0

> 在了解了 [AQS 独占锁模式](https://www.jianshu.com/p/71449a7d01af)以后，接下来再来看看共享锁的实现原理。

搞清楚 AQS 独占锁的实现原理之后，再看共享锁的实现原理就会轻松很多。两种锁模式之间很多通用的地方本文只会简单说明一下，就不在赘述了，具体细节可以参考我的上篇文章[深入浅出 AQS 之独占锁模式](https://www.jianshu.com/p/71449a7d01af)

#### 一、执行过程概述

获取锁的过程：

1.  当线程调用 acquireShared() 申请获取锁资源时，如果成功，则进入临界区。
2.  当获取锁失败时，则创建一个共享类型的节点并进入一个 FIFO 等待队列，然后被挂起等待唤醒。
3.  当队列中的等待线程被唤醒以后就重新尝试获取锁资源，如果成功则**唤醒后面还在等待的共享节点并把该唤醒事件传递下去，即会依次唤醒在该节点后面的所有共享节点**，然后进入临界区，否则继续挂起等待。

释放锁过程：

1.  当线程调用 releaseShared() 进行锁资源释放时，如果释放成功，则唤醒队列中等待的节点，如果有的话。

#### 二、源码深入分析

基于上面所说的共享锁执行流程，我们接下来看下源码实现逻辑：
首先来看下获取锁的方法 acquireShared()，如下

```java
   public final void acquireShared(int arg) {
        //尝试获取共享锁，返回值小于0表示获取失败
        if (tryAcquireShared(arg) < 0)
            //执行获取锁失败以后的方法
            doAcquireShared(arg);
    }
```

这里 tryAcquireShared() 方法是留给用户去实现具体的获取锁逻辑的。关于该方法的实现有两点需要特别说明：

1. 该方法必须自己检查当前上下文是否支持获取共享锁，如果支持再进行获取。

2. 该方法返回值是个重点。其一、由上面的源码片段可以看出返回值小于 0 表示获取锁失败，需要进入等待队列。其二、如果返回值等于 0 表示当前线程获取共享锁成功，但它后续的线程是无法继续获取的，也就是不需要把它后面等待的节点唤醒。最后、如果返回值大于 0，表示当前线程获取共享锁成功且它后续等待的节点也有可能继续获取共享锁成功，也就是说此时需要把后续节点唤醒让它们去尝试获取共享锁。

有了上面的约定，我们再来看下 doAcquireShared 方法的实现：

```java
    //参数不多说，就是传给acquireShared()的参数
    private void doAcquireShared(int arg) {
        //添加等待节点的方法跟独占锁一样，唯一区别就是节点类型变为了共享型，不再赘述
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                //表示前面的节点已经获取到锁，自己会尝试获取锁
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    //注意上面说的， 等于0表示不用唤醒后继节点，大于0需要
                    if (r >= 0) {
                        //这里是重点，获取到锁以后的唤醒操作，后面详细说
                        setHeadAndPropagate(node, r);
                        p.next = null;
                        //如果是因为中断醒来则设置中断标记位
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                //标记1：挂起逻辑跟独占锁一样，不再赘述 
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            //获取失败的取消逻辑跟独占锁一样，不再赘述
            if (failed)
                cancelAcquire(node);
        }
    }
```

独占锁模式获取成功以后设置头结点然后返回中断状态，结束流程。而共享锁模式获取成功以后，调用了 setHeadAndPropagate 方法，从方法名就可以看出除了设置新的头结点以外还有一个传递动作，一起看下代码：

```java
    //两个入参，一个是当前成功获取共享锁的节点，一个就是tryAcquireShared方法的返回值，注意上面说的，它可能大于0也可能等于0
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; //记录当前头节点
        //设置新的头节点，即把当前获取到锁的节点设置为头节点
        //注：这里是获取到锁之后的操作，不需要并发控制
        setHead(node);
        //这里意思有两种情况是需要执行唤醒操作
        //1.propagate > 0 表示调用方指明了后继节点需要被唤醒
        //2.头节点后面的节点需要被唤醒（waitStatus<0），不论是老的头结点还是新的头结点
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            //如果当前节点的后继节点是共享类型或者没有后继节点，则进行唤醒
            //这里可以理解为除非明确指明不需要唤醒（后继等待节点是独占类型），否则都要唤醒
            if (s == null || s.isShared())
                //后面详细说
                doReleaseShared();
        }
    }

    private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }
```

最终的唤醒操作也很复杂，专门拿出来分析一下

注：这个唤醒操作在 releaseShare() 方法里也会调用。

```java
private void doReleaseShared() {
        for (;;) {
            //唤醒操作由头结点开始，注意这里的头节点已经是上面新设置的头结点了
            //其实就是唤醒上面新获取到共享锁的节点的后继节点
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                //表示后继节点需要被唤醒
                if (ws == Node.SIGNAL) {
                    //这里需要控制并发，因为入口有setHeadAndPropagate跟release两个，避免两次unpark
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;      
                    //执行唤醒操作，唤起上面的标记1，继续向下传递  
                    unparkSuccessor(h);
                }
                //如果后继节点暂时不需要唤醒，则把当前节点状态设置为PROPAGATE确保以后可以传递下去
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                
            }
            //如果头结点没有发生变化，表示设置完成，退出循环
            //如果头结点发生变化，比如说其他线程获取到了锁，为了使自己的唤醒动作可以传递，必须进行重试
            if (h == head)                   
                break;
        }
    }
```

接下来看下释放共享锁的过程：

```java
public final boolean releaseShared(int arg) {
        //尝试释放共享锁
        if (tryReleaseShared(arg)) {
            //唤醒过程，详情见上面分析
            doReleaseShared();
            return true;
        }
        return false;
    }
```

注：上面的 setHeadAndPropagate() 方法表示等待队列中的线程成功获取到共享锁，这时候它需要唤醒它后面的共享节点（如果有），但是当通过 releaseShared（）方法去释放一个共享锁的时候，接下来等待独占锁跟共享锁的线程都可以被唤醒进行尝试获取。

#### 三、总结

跟独占锁相比，共享锁的主要特征在于当一个在等待队列中的共享节点成功获取到锁以后（它获取到的是共享锁），既然是共享，那它必须要依次唤醒后面所有可以跟它一起共享当前锁资源的节点，毫无疑问，这些节点必须也是在等待共享锁（这是大前提，如果等待的是独占锁，那前面已经有一个共享节点获取锁了，它肯定是获取不到的）。当共享锁被释放的时候，可以用读写锁为例进行思考，当一个读锁被释放，此时不论是读锁还是写锁都是可以竞争资源的。