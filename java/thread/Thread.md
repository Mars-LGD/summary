## 方法简述

### Thread类

- sleep：暂停当前正在执行的线程（类方法）

- yield：暂停当前正在执行的线程，并执行其他线程（类方法）

- join：等待该线程终止

- interrupt：中断该线程，当线程调用wait（），sleep（），join（），或I/O操作时，将收到InterruptedException或ClosedByInterruptException；

- stop（已废弃）：这个方法会解除被加锁的对象的锁，因而可能造成这些对象处于不一致的状态，而且这个方法造成的ThreadDeath异常不像其他的检查期异常一样被捕获。

  可以使用interrupt方法代替。事实上，如果一个方法不能被interrupt，那stop方法也不会起作用。

- Thread.suspend, Thread.resume （已废弃）：这俩方法有造成死锁的危险。使用suspend时，并不会释放锁；而如果我想先获取该锁，再进行resume，就会造成死锁。

  可以使用object的wait和notify方法代替。wait方法会释放持有的锁。

### Object类

- wait：暂停当前正在执行的线程，知道调用notify（）或notifyAll（）方法或超时，退出等待状态；
- notify：唤醒在该对象上等待的一个线程；
- notifyAll：唤醒在该对象上等待的所有线程；

## 详细分析

### sleep vs wait

sleep（）和wait（）方法都是暂停当前正在执行的线程，出让CPU资源；但wait会释放锁，sleep不会

### wait&&notify

调用对象的wait（）、notify（）、notifyAll（）方法的线程，必须是作为此对象监视器的所有者。常见的场景就是synchronized关键字的语句块内部使用者3个方法，如果直接在线程中使用wait（）、notify（）、notifyAll（）方法，那么会抛出异常IllegalMonitorException，抛出的异常表明某一线程已经试图等待对象的监视器，或者试图通知其他正在等待对象的监视器而本身没有指定监视器的线程。

调用wait（）方法的线程，在调用该线程的interrupt（）方法，则会重新尝试获取对象锁。只有当获取到对象锁，才开始抛出相应的异常，则执行该线程之后的程序。

### interrupt

interrupt（）方法的工作仅仅是改变中断状态，并不是直接中断正在运行的线程。中断的真正原理是当线程被Object.wait（）、Thread.join（）或Thread.sleep（）方法阻塞时，调用interrupt（）方法后改变中断状态，而wait/join/sleep这些方法内部会不断地检查线程的中断状态值，当发现中断状态值时则抛出InterruptException异常；对于没有阻塞的线程，调用interrupt（）方法时没有任何作用的。

### yield

yield()方法使当前线程出让CPU执行时间，当并不会释放当前线程所持有的锁。执行完yield（）方法后，线程从Running状态转变为Runable状态，既然是Runnable状态，那么也很可能马上会被CPU调度再次进入Running状态。



