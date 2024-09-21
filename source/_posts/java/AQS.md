---
title: AQS源码阅读（个人理解）
date: 2023-08-22 17:11:04
tags: JAVA
categories: JAVA
---

### 1、介绍

AQS是JAVA并发编程中的一个重要组件，它是一个抽象类，主要用于构建锁和其他同步器组件。

（1）核心思想：使用CLH（Craig, Landin, and Hagersten）队列锁的实现，这是一个虚拟的双向队列，通过这个队列来管理等待获取资源的线程。AQS通过内置的FIFO（First In, First Out，先进先出）队列来完成资源获取线程的排队工作，并通过一个int类型变量表示持有锁的状态。AQS使用CAS（Compare and Swap，比较并交换）操作来原子地修改这个同步状态的值。

（2）核心方法：AQS提供了几种核心方法，如acquire()、tryAcquire()、release()、tryRelease()、acquireShared()、tryAcquireShared()和releaseShared()，这些方法分别用于尝试获取锁、非阻塞地尝试获取锁、释放锁、非阻塞地释放锁、尝试获取共享资源、非阻塞地尝试获取共享资源和释放共享资源。此外，AQS还提供了其他辅助方法，如tryAcquireNanos()和hasQueuedThreads()，分别用于在指定时间内尝试获取锁和判断是否有线程在等待队列中等待。

（3）实现类：Java并发编程中的常用类，如ReentrantLock、Semaphore、ReentrantReadWriteLock、CountDownLatch等，都是基于AQS实现的。AQS的设计基于模板方法模式，允许开发者通过继承AQS并实现其中的关键方法来定义自己的同步器，而无需担心线程等待队列的维护等复杂细节。

### 2、ReentrantLock源码分析

个人理解：AQS本质是提供一个抽象同步队列，加锁的逻辑AQS内部是没有的，只提供了抽象方法，如**tryAcquire**和**tryRelease**。具体的加锁逻辑由实现类完成，如没有重写方法，则会抛出**UnsupportedOperationException**异常。

```java
// 获取独占锁
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
// 释放独占锁
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
// 获取共享锁   主要是ReentrantReadWriteLock.Sync中的类进行重写。尝试获取读锁。
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}
// 释放共享锁
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}
```

本次主要对**ReentrantLock**实现类进行讲解。

#### 2.1、继承关系

**ReentrantLock.FairSync**和**ReentrantLock.NonfairSync**分别继承自**ReentrantLock.Sync**,而**ReentrantLock.Sync**又继承自AQS抽象同步队列。相关的加锁逻辑则在子类中进行重写。

```java
abstract static class Sync extends AbstractQueuedSynchronizer{...}
static final class FairSync extends Sync{...}
static final class NonfairSync extends Sync{...}
```

#### 2.2、锁状态位

**AbstractQueuedSynchronizer**类中的**state**变量。如果state变量为0 则表示当前没有任何线程加锁。如果不为0则表示有线程已经加锁成功。

```java
// AQS类中的静态代码块，将内部的state变量的内存偏移量赋值給stateOffset。
static{
    stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
}
// 即可通过stateOffset去更新state变量的值。该方法通过在compareAndSetState(0, 1)中使用
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

#### 2.3、加锁类型

**ReentrantLock**有两种加锁类型，即公平锁和非公平锁。公平锁则严格按照线程获取锁的时间顺序进行加锁，而非公平锁则可以在每次加锁的时候尝试获取锁，如果尝试成功则直接加锁完成，失败则加入阻塞队列队尾。

```java
// NonfairSync中lock方法
final void lock() {
    // 尝试通过cas将锁状态标识更改。
    if (compareAndSetState(0, 1))
        // 如更改成功则尝试加锁完成，并将独占线程设置为自身
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}

// FairSync中lock方法
final void lock() {
    acquire(1);
}
```

#### 2.4、加锁方法

- 加锁方法**lock**

```java
// ReentrantLock类内部
final void lock() {
    acquire(1);
}
// AQS类内部 定义好的加锁模板，如果未加锁成功，则自动帮你将当前的线程加入等待队列末尾，并休眠等待其他线程唤醒。
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

1、tryAcquire() 尝试获取锁，即执行加锁逻辑。通常为子类实现方法
2、加锁失败执行
    2.1、addWaiter() 将当前线程封装成节点Node加入链表末尾。
    2.2、acquireQueued() 如果当前节点为第一个入队的节点，则cas尝试获取锁，失败几次后线程进入阻塞状态
3、selfInterrupt() 将当前线程的锁中断标识设置为true

- 非公平锁的**tryAcquire**方法

```java
// NonfairSync类中方法
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
// Sync类中方法
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();    // 获取当前线程
    int c = getState(); // 获取AQS内部state变量
    // 如果等于0 则标识当前无线程加锁
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            // 通过cas去设置锁标识位，成功则将当前线程设置独占线程并返回。
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果不等于0，并且当前线程为加锁成功的哪个线程。（可重入，即多次加锁。但也要多次释放锁）
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        // 因加锁线程是当前线程，所以可以直接调用set方法，没有线程安全问题。
        setState(nextc);
        return true;
    }
    // 加锁失败
    return false;
}
```

- 公平锁的**tryAcquire** 方法

```java
protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 如果当前没有线程加入队列，则尝试cas一次。成功则加锁完成，设置当前线程为独占线程
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

- **addWaiter**方法

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode); // 封装node节点
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred; // 当前node节点的前置节点和 tail节点关联。此刻是release锁的时候为什么从后往前遍历的关键。
        if (compareAndSetTail(pred, node)) {
            // 如果cas设置当前节点为尾节点成功，则将原tail节点的后置设置为当前node。
            // 核心点在cas内部设置tail.next的属性,在cas外部设置node.prev节点的属性。如果cas失败则不影响其他节点访问tail。
            pred.next = node;
            return node;
        }
    }
    // cas失败则循环入队
    enq(node);
    return node;
}
```

- acquireQueued方法

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 本质如果当前node节点的前置位head 表明当前节点为第一个节点。则尝试获取锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 如果尝试几次后，还是没成功则shouldParkAfterFailedAcquire方法会返回成功。
            // 并调用parkAndCheckInterrupt方阻塞线程。
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

- **shouldParkAfterFailedAcquire**方法

```java
static final Node EXCLUSIVE = null; // 初始状态
/** waitStatus value to indicate thread has cancelled */
static final int CANCELLED =  1;  // 取消状态 
/** waitStatus value to indicate successor's thread needs unparking */
static final int SIGNAL    = -1; // 表示当前线程进行阻塞，需要其他线程 unparking
/** waitStatus value to indicate thread is waiting on condition */
static final int CONDITION = -2;
/**
    * waitStatus value to indicate the next acquireShared should
    * unconditionally propagate
*/
static final int PROPAGATE = -3;

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 如果为Node.SIGNAL 则表示已经执行过一次该方法。不为初始状态了。
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        // 小于0 则表示前置节点已取消。循环获取未取消的前置节点。
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        // 初始状态，则进入此刻执行。
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

#### 2.5、释放锁方法

- **unlock**方法

```java
// ReentrantLock类中方法
public void unlock() {
    sync.release(1);
}
// AQS类中方法
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        // 如果释放锁成功，则执行unparkSuccessor方法，唤醒其他等待线程
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

- **tryRelease**方法

```java
// ReentrantLock.Sync类中方法
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    // 如果当前释放锁的线程不是加锁成功的线程则抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        // 如果释放锁后，state等于0 则将独占锁清空。
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

- **unparkSuccessor**方法

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    // 将node的等待状态cas为0。
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next; // node节点的后置节点 
    if (s == null || s.waitStatus > 0) {
        //如果s为空或者状态为取消，则从tail节点开始往前找。为什么从tail往前找，不从head往后找，在addWaiter方法中可找到原因。
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 唤醒等待线程。
        LockSupport.unpark(s.thread);
}
```


