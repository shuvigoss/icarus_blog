---
title: 【ReentrantLock源码分析】公平锁&非公平锁
date: 2017-02-09 16:22:27
tags: ReentrantLock
categories: Java
---

首先，先看一下ReentrantLock类结构。
![](/images/ReentrantLock.gif)
这里可以看到，在ReentrantLock内部，有个Sync内部静态抽象类，该类继承自`AbstractQueuedSynchronizer(AQS)`,并且有2个内部静态类的实现`NonfairSync`与`FairSync`，从类的名字就能看出来公平锁与非公平锁是通过Sync实现的。

<!--more-->
> 这里先不对大名鼎鼎的AQS进行介绍，在ReentrantLock内,AQS中state代表着锁的数量,初始值为0,如果有线程获得锁会变为1,由于ReentrantLock是可重入锁,获得锁的线程还是可以继续获得锁,相应的state会在1的基础上继续做++操作,由于对state的操作都是原子的,对它的修改都是通过compareAndSetState(int expect, int update)实现的(CAS)

直接看`Sync`
``` java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    /**
     * Performs {@link Lock#lock}. The main reason for subclassing
     * is to allow fast path for nonfair version.
     */
    abstract void lock();

    /**
     * Performs non-fair tryLock.  tryAcquire is
     * implemented in subclasses, but both need nonfair
     * try for trylock method.
     */
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

    //...省略
}

```

其中`nonfairTryAcquire`是专门为非公平锁获取锁实现。

> 这里不明白为什么Doug Lea大神不把实现放到NonfairSync中去~

接下来从代码层面来走一遍lock以及unlock的流程
``` java
class X {
  private final ReentrantLock lock = new ReentrantLock();
  // ...
  public void m() {
    lock.lock();  // block until condition holds
    try {
      // ... method body
    } finally {
      lock.unlock()
    }
  }
}
```

``` java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```
`ReentrantLock`构造方法默认是非公平模式。


``` java
public void lock() {
    sync.lock();
}
```
lock方法很简单，调用sync(NonfairSync、FairSync)的lock方法，具体调用哪个要看创建的是哪种模式的锁(公平、非公平)

### lock
#### NonfairSync
[CLH队列](http://blog.csdn.net/aesop_wubo/article/details/7533186)
``` java
final void lock() {
    //很霸道的直接尝试加锁,并不会考虑是否需要加入CHL(Craig,Landin,Hagersten)队列
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        //如果直接加锁失败,尝试获取
        acquire(1);
}
```
#### FairSync
``` java
final void lock() {
    //尝试获取锁
    acquire(1);
}
```

### acquire
acquire方法是`AbstractQueuedSynchronizer`的方法
``` java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
tryAcquire(尝试获取)->失败->acquireQueued(加入队列)

### tryAcquire
#### NonfairSync
``` java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    //如果当前没有锁,设置锁线程拥有者为自己并返回成功(明显的一个插队现象)
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //如果自己获得得了锁,再次对state进行++操作,重入
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
#### FairSync
``` java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    //与NonfairSync不同在于,它会判断队列中是否还有排队线程,
    //如果没有才会设置自己为线程拥有者
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //重入
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

``` java
//h != t queue非空
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

*如果tryAcquire成功,代表加锁成功 end*

### addWaiter
`acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`
``` java
//放入队列尾部,模式是独占锁模式
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

``` java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // 初始化节点,初始化完成后再设置head节点的next节点为等待节点
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

### acquireQueued
``` java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //如果当前节点pre节点为head节点并且tryAcquire成功,
            //认为加锁成功,将自己设置为头结点,并返回不是被interrupted.
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //是否要进行Park操作
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

``` java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    //pre节点告诉后节点,你需要等待一个Signal,所以你先Park吧~
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    //pre节点已经Cancelled,那么当前节点要对前边为CANCELLED的进行剔除,
    //通过do while 循环找到一个waitStatus>0的节点并设置为自己的pre节点
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //如果pre节点状态为0或者PROPAGATE,将pre节点设置为SIGNAL方便下次进来时执行Park
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

``` java
//park直到被unpark并且返回自己是否是被interrupted唤醒的.
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

``` java
//这个方法目的在于如果自己是被interrupted,
//那么将自己标记为CANCELLED,并判断是否需要唤醒后继线程
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;
    node.thread = null;
    // Skip cancelled predecessors
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    Node predNext = pred.next;
    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    node.waitStatus = Node.CANCELLED;
    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
```

### selfInterrupt
``` java
//!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg) 
//!没有获得锁并且获取队列过程中是被interrupted,那么最终标记线程interrupt
private static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

### unlock
``` java
//调用sync的release
public void unlock() {
    sync.release(1);
}
```

``` java
public final boolean release(int arg) {
    //尝试释放锁,如果成功,判断是否有等待的头结点,如果有唤醒
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

``` java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    //判断是否是线程的拥有者
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

``` java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    //将waitStatus设置为0
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    //筛选后继结点waitStatus<=0的节点
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //唤醒后继节点
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

### Fair Flow
![](/images/ReentrantLock-Fair.png)

### Nonfair Flow
![](/images/ReentrantLock-Nonfair.png)

### end
`ReentrantLock`在性能上与synchronized(重量级锁)比较的话(大并发),会高一些(个人认为),毕竟在获取锁时的CAS,以及可重入等等会有那么一些性能上的优势,毕竟synchronized在重量级锁的环境下是没有CAS的,并且锁的竞争一直存在.

`ReentrantLock`相较synchronized关键字的优势有以下几点:
* 顺序,能保证线程获取锁的顺序(CHL).
* Condition,使ReentrantLock更加灵活.



