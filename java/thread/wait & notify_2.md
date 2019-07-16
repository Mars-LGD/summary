> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 https://blog.csdn.net/u013332124/article/details/84647915 版权声明：本文为博主原创文章，未经博主允许不得转载。如有疑问，请在下方评论或邮件本人, 本人邮箱: kongtrio@sina.com https://blog.csdn.net/u013332124/article/details/84647915

在 java 语言中，可以通过 3 种方式让线程进入休眠状态，分别是`Thread.sleep()`、`Object.wait()`、`LockSupport.park()`方法。这三种方法的表现和原理都各有不同，今天稍微研究了下这几个方法的区别。

### Thread.sleep() 方法

`Thread.sleep(time)`方法必须传入指定的时间，线程将进入休眠状态，通过 jstack 输出线程快照的话此时该线程的状态应该是`TIMED_WAITING`，表示休眠一段时间。

另外，该方法会抛出 InterruptedException 异常，这是受检查异常，调用者必须处理。

**通过 sleep 方法进入休眠的线程不会释放持有的锁，因此，在持有锁的时候调用该方法需要谨慎。**

### Object.wait() 方法

我们都知道，java 的每个对象都隐式的继承了 Object 类。因此每个类都有自己的 wait() 方法。我们通过 object.wait() 方法也可以让线程进入休眠。wait() 有 3 个重载方法:

```java
public final void wait() throws InterruptedException;
public final native void wait(long timeout) throws InterruptedException;
public final native void wait(long timeout) throws InterruptedException;
```

如果不传 timeout，wait 将会进入无限制的休眠当中，直到有人唤醒他。使用 wait() 让线程进入休眠的话，无论有没有传入 timeout 参数，线程的状态都将是`WAITING`状态。

另外，必须获得对象上的锁后，才可以执行该对象的 wait 方法。否则程序会在运行时抛出`IllegalMonitorStateException`异常。

```java
Object waitObject = new Object();
try {
    //没获取到waitObject的锁，调用该方法抛出IllegalMonitorStateException异常
      waitObject.wait();
} catch (InterruptedException e) {
      e.printStackTrace();
}

//正确的调用方式  
Object waitObject = new Object();
try {
    //先获取到waitObject的锁
    synchronized (waitObject){
        waitObject.wait();
    }
} catch (InterruptedException e) {
      e.printStackTrace();
}
```

再调用 wait() 方法后，线程进入休眠的同时，**会释放持有的该对象的锁**，这样其他线程就能在这期间获取到锁了。

调用 Object 对象的 notify() 或者 notifyAll() 方法可以唤醒因为 wait() 而进入等待的线程。

### LockSupport.park() 方法

通过`LockSupport.park()`方法，我们也可以让线程进入休眠。**它的底层也是调用了 Unsafe 类的 park 方法**：

```java
//Unsafe.java类
//唤醒指定的线程
public native void unpark(Thread jthread);
//isAbsolute表示后面的时间是绝对时间还是相对时间，time表示时间，time=0表示无限阻塞下去
public native void park(boolean isAbsolute, long time);
```

调用 park 方法时，还允许设置一个 blocker 对象，主要用来给监视工具和诊断工具确定线程受阻塞的原因。

调用 park 方法进入休眠后，线程状态为 **WAITING**。

#### 实现原理

LockSupport.park() 的实现原理是通过二元信号量做的阻塞，**要注意的是，这个信号量最多只能加到 1**。我们也可以理解成获取释放许可证的场景。unpark() 方法会释放一个许可证，park() 方法则是获取许可证，如果当前没有许可证，则进入休眠状态，知道许可证被释放了才被唤醒。**无论执行多少次 unpark() 方法，也最多只会有一个许可证。**

#### 和 wait 的不同

park、unpark 方法和 wait、notify() 方法有一些相似的地方。都是休眠，然后唤醒。但是 wait、notify 方法有一个不好的地方，就是我们在编程的时候必须能保证 wait 方法比 notify 方法先执行。如果 notify 方法比 wait 方法晚执行的话，就会导致因 wait 方法进入休眠的线程接收不到唤醒通知的问题。而 park、unpark 则不会有这个问题，我们可以先调用 unpark 方法释放一个许可证，这样后面线程调用 park 方法时，发现已经许可证了，就可以直接获取许可证而不用进入休眠状态了。

**另外，和 wait 方法不同，执行 park 进入休眠后并不会释放持有的锁**。

#### 对中断的处理

park 方法不会抛出`InterruptedException`，但是它也会响应中断。当外部线程对阻塞线程调用 interrupt 方法时，park 阻塞的线程也会立刻返回。

```java
Thread parkThread = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("park begin");

                //等待获取许可
                LockSupport.park();
                //输出thread over.true
                System.out.println("thread over." + Thread.currentThread().isInterrupted());

            }
        });
        parkThread.start();

        Thread.sleep(2000);
        // 中断线程
        parkThread.interrupt();

        System.out.println("main over");
```

上面的 demo 最终会输出

```
park begin
main over
thread over.true
```

说明因 park 进入休眠的线程收到中断通知后也会立刻返回，并且可以手动通过`Thread.currentThread().isInterrupted()`获取到中断位。

### 总结

![image-20190715175300877](../../assets/image-20190715175300877.png)

### 题外话：关于 java 进程的关闭

在 linux 中，我们通常用 kill 命令来关闭一个进程。众所周知，kill 有 - 9 和 - 15 两种参数，默认是 - 15。如果是 - 15 参数，系统就发送一个关闭信号给进程，然后等待进程关闭。在这个过程中，目标进程可以释放手中的资源，以及进行一些关闭操作。

正是有了这个概念，我曾经很大一段时间对 java 进程的关闭流程有所误解。在我原先的理解中，java 进程接收到关闭信号后，会逐一给阻塞中的进程发送中断信号，并等待线程处理完。**但其实这是错误的**。

java 进程收到关闭信号后，不会去关心运行中的那些线程是否运行完，也不会给阻塞中的线程发送中断信号。我们只能通过绑定关闭钩子来中断目标线程并等待线程执行完。

```java
final Thread waitThread = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("thread begin");

                //等待获取许可
                try {
                    Thread.sleep(1000000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                //输出thread over.true
                System.out.println("thread over." + Thread.currentThread().isInterrupted());

                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        waitThread.start();
        //绑定钩子
        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    waitThread.interrupt();
                    waitThread.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("shutdown success");
            }
        }));
```

java 进程在收到关闭信号后，会执行所有绑定了 shutdownHook 的线程，确保这些绑定的线程都执行完了才真正关闭。因此，我们要释放资源就要在 shutdownHook 的线程内操作，然后在线程内等待其他释放资源的线程执行完成。

**注意，所有绑定了 shutdownHook 的线程也是并行执行的，不是顺序执行。另外，用 - 9 参数的 kill 不会等 shutdownHook 线程执行完就退出。**