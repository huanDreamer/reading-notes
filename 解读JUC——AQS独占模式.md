---
title: 解读 JUC —— AQS 独占模式
date: 2019-05-10 15:21:06
tags: [源码,Java]
published: true
hideInList: false
feature: https://user-gold-cdn.xitu.io/2019/5/8/16a939a72c2cb3ce?imageView2/0/w/1280/h/960/ignore-error/1
---
> 本文转载自 [https://juejin.im/post/5cd1d1876fb9a031eb58ae4d](https://juejin.im/post/5cd1d1876fb9a031eb58ae4d) 

# 1. 前言

说起 JUC，我们常常会想起其中的线程池（ExecutorService）。然而，我们今天来看看另一个核心模块 AQS。

AQS 是 AbstractQueuedSynchronizer 的简称，在 JUC 中作为各种同步器的基石。举个例子，常见的 ReentrantLock 就是由它实现的。

# 2. 如何实现一个锁？

我们知道，java 有一个关键字 synchronized 来给一段代码加锁，可是这是 JVM 层面的事情。那么问题来了，如何在 java 代码层面来实现模拟一个锁？

即实现这样一个接口：
```
package java.util.concurrent.locks;

public interface Lock {
    void lock();

    void unlock();
}
```

## 自旋锁

一个简单的想法是：让所有线程去竞争一个变量owner，确保只有一个线程成功，并设置自己为owner，其他线程陷入死循环等待。这便是所谓的**自旋锁**。

一个简单的代码实现：
```
import java.util.concurrent.atomic.AtomicReference;

public class SpinLock {
   private AtomicReference<Thread> owner = new AtomicReference<Thread>();

   public void lock() {
       Thread currentThread = Thread.currentThread();

       // 如果锁未被占用，则设置当前线程为锁的拥有者
       while (!owner.compareAndSet(null, currentThread)) {
       }
   }

   public void unlock() {
       Thread currentThread = Thread.currentThread();

       // 只有锁的拥有者才能释放锁
       owner.compareAndSet(currentThread, null);
   }
}
```

扯这个自旋锁，主要是为了引出 AQS 背后的算法 `CLH锁`。

关于`CLH锁`更多细节可以参考篇文章：

[自旋锁、排队自旋锁、MCS锁、CLH锁](https://link.juejin.im?target=https%3A%2F%2Fcoderbee.net%2Findex.php%2Fconcurrent%2F20131115%2F577)

# 3. AQS 的实现

CLH锁的思想，简单的说就是：一群人去ATM取钱，头一个人拿到锁，在里面用银行卡取钱，其余的人在后面**排队等待**；前一个人取完钱出来，**唤醒**下一个人进去取钱。

关键部分翻译成代码就是：

* 排队 -> 队列
* 等待/唤醒 -> wait()/notify() 或者别的什么 api 

## 3.1 同步队列

AQS 使用节点为 Node 的双向链表作为**同步队列**。拿到锁的线程可以继续执行代码，没拿到的线程就进入这个队列排队。
```
public abstract class AbstractQueuedSynchronizer ... {
    // 队列头
    private transient volatile Node head;
    // 队列尾
    private transient volatile Node tail;

    static final class Node {
        /** 共享模式，可用于实现 CountDownLatch */
        static final Node SHARED = new Node();
        /** 独占模式，可用于实现 ReentrantLock */
        static final Node EXCLUSIVE = null;
	
        /** 取消 */
        static final int CANCELLED =  1;
        /** 意味着它的后继节点的线程在排队，等待被唤醒 */
        static final int SIGNAL    = -1;
        /** 等待在条件上（与Condition相关，暂不解释） */
        static final int CONDITION = -2;
        /**
         * 与共享模式相关，暂不解释
         */
        static final int PROPAGATE = -3;
	
        // 可取值：CANCELLED, 0, SIGNAL, CONDITION, PROPAGATE
        volatile int waitStatus;
	
        volatile Node prev;
	
        volatile Node next;
	
        volatile Thread thread;
	    
        Node nextWaiter;
    }
}
```

这个队列大体上长这样：[图片来源](https://link.juejin.im?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fa8d27ba5db49)

![sync queue](https://user-gold-cdn.xitu.io/2019/5/8/16a939a72c2cb3ce?imageView2/0/w/1280/h/960/ignore-error/1)

条件队列是为了支持 Lock.newCondition() 这个功能，暂时不care，先跳过。

## 3.2 独占模式的 api

AQS 支持独占锁（Exclusive）和共享锁（Share）两种模式：

* 独占锁：只能被一个线程获取到 (ReentrantLock)；
* 共享锁：可以被多个线程同时获取 (CountDownLatch、ReadWriteLock 的读锁)。

这边我们只看独占模式，它对外提供一套 api：

* acquire(int n)：获取n个资源（锁）
* release(int n)：释放n个资源（锁）

简单看一眼怎么用的 (ReentrantLock 的例子):
```
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;
    
    abstract static class Sync extends AbstractQueuedSynchronizer {...}

    public void lock() {
        sync.acquire(1);
    }

    public void unlock() {
        sync.release(1);
    }
}
```

可以看到，AQS 封装了排队、阻塞、唤醒之类的操作，使得实现一个锁变的如此简洁。

## 3.2.1 acquire(int)

获取资源
```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &amp;&amp;
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

这个函数很短，其中 tryAcquire(int) 为模板方法，留给子类实现。类似 Activity.onCreate()。

根据 tryAcquire(arg) 的结果，分两种情况：

	* 返回 true: 该线程拿到锁，由于短路，直接跳出 if，该线程可以往下执行自己的业务代码。
	* 返回 false: 该线程没有拿到锁，会继续走 acquireQueued()，执行排队等待逻辑。

### 3.2.1.1 addWaiter(Node)

这一步把当前线程（Thread.currentThread()）作为一个Node节点，加入同步队列的尾部，并标记为独占模式。

当然，加入队列这个动作，要保证**线程安全**。
```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // 1处尝试更快地 enq(), 成功的话直接 return。失败的话, 在2处退化为完整版的 enq()，相对更慢些
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) { // 1
            pred.next = node;
            return node;
        }
    }
    enq(node); // 2
    return node;
}

private Node enq(final Node node) {
    // 神奇的死循环 + CAS
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

可以看到，这边有一个**死循环 + CAS**的神奇操作，这是非阻塞算法的经典操作，可自行查阅相关资料。简单的说，非阻塞算法就是在多线程的情况下，不加锁同时保证某个变量（本例中为双向链表）的线程安全，而且通常比 synchronized 的效率要高。

### 3.2.1.2 acquireQueued(Node，int)

这个函数主要做两件事：

	* 查看prev的waitStatus，看是不是需要阻塞，需要的话阻塞该线程
	* 排在队首的家伙调用了release()，会唤醒老二。老二尝试去获得锁，成功的话自己变成队首，跳出循环。

结合这张图来看，每次出队完需要确保 head 始终指向占用资源的线程：

![sync queue](https://user-gold-cdn.xitu.io/2019/5/8/16a939a72c3d316d?imageView2/0/w/1280/h/960/ignore-error/1)

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 这又是一个死循环 + CAS，这次CAS比较隐蔽，在 shouldParkAfterFailedAcquire()里边
        for (;;) {
            final Node p = node.predecessor();
            if (p == head &amp;&amp; tryAcquire(arg)) { // 排在队首的后面，看看能不能获得锁，成功的话自己变成队首
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &amp;&amp;
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        // 这里失败会做一些回滚操作，不分析
        if (failed)
            cancelAcquire(node);
    }
}
```

这边的 interrupted 主要是保证这样一个功能。线程在排队的时候不响应中断，直到出来以后，如果等待的过程中被中断过，作为弥补，立即相应中断（即调用selfInterrupt()）。

### shouldParkAfterFailedAcquire()

查看prev的waitStatus，看是不是需要阻塞。可以预见的是，经过几次死循环，全部都会变成SIGNAL状态。之后全部陷入阻塞。
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus; // 查看前驱节点的状态
    if (ws == Node.SIGNAL) // SIGNAL: 可以安全的阻塞
        return true;
    if (ws > 0) { // CANCEL: 取消排队的节点，直接从队列中清除。
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else { // 0 or PROPAGATE: 需要变成 SIGNAL，但不能立即阻塞，需要重走外层的死循环二次确认。
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

值得一提的是，阻塞和唤醒没有使用常说的 wait()/notify()，而是使用了 LockSupport.park()/unpark()。只有使用 synchronized 获取锁的对象才能使用 wait，ReentrantLock.newCondition 提供的 await/signal 也是 LockSupport.park()/unpark() 实现的。
参考 [https://pqpo.me/2019/01/30/learn-java-lock-block/](https://pqpo.me/2019/01/30/learn-java-lock-block/)

## 3.2.2 release(int)

释放资源
```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null &amp;&amp; h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * 唤醒next节点对应的线程，通常就是老二（直接后继）。
     * 如果是null，或者是cancel状态（出现异常如线程遇到空指针挂掉了），
     * 那么跳过cancel节点，找到后继节点。
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null &amp;&amp; t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 唤醒 node.next
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

释放的逻辑比较简单。注意一点，对于 next 节点 unpark()，相当于在把 next 节点从 acquireQueued() 中的死循环中解放出来。

回到 ATM 的例子，相当于，他取完钱，轮到后一个人取钱了。这样逻辑全部都串起来了。

# 4. 总结

这样，顺着独占锁这条线，AQS 的独占模式就分析完了。其他还有用于实现闭锁的共享模式，用于实现 Condition 的条件队列就不展开了。

# 5. 参考

[Java并发编程实战（chapter_4）（AQS源码分析）](https://link.juejin.im?target=https%3A%2F%2Fmy.oschina.net%2FUBW%2Fblog%2F2995774)

[JUC源码分析—AQS](https://link.juejin.im?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fa8d27ba5db49)

[自旋锁、排队自旋锁、MCS锁、CLH锁](https://link.juejin.im?target=https%3A%2F%2Fcoderbee.net%2Findex.php%2Fconcurrent%2F20131115%2F577)

《JAVA并发编程实践》
