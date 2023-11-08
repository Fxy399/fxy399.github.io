---
title: AQS 的加锁及解锁原理分析
date: 2023-11-08 18:59:03
tags: Java并发编程
description: AQS 的加锁及解锁流程源码分析
---
# AQS 简介

AQS 全称 **AbstractQueuedSynchronizer**，是一个构建锁或者其他同步组件的的基础框架，它使用了一个 int 成员变量来表示同步状态，通过内置的 FIFO 同步队列来控制等待线程。

使用AQS能简单且高效地构造出应用广泛的大量的同步器，比如ReentrantLock，Semaphore，其他的诸如ReentrantReadWriteLock，SynchronousQueue，FutureTask等等皆是基于AQS的。

## AQS 结构分析

我们从源码开始，先看 AQS 的基本属性

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    // 等待队列节点类型
    static final class Node {
        //...省略
    }

    //队列头节点，同时也是当前	持有锁的线程
    private transient volatile Node head;

    //队列尾节点，每个新进来的等待线程，都插入到队列尾部，形成链表
    private transient volatile Node tail;

    //volatile修饰， 标识同步状态
    //state 为 0 表示锁空闲，state>0 表示锁被持有，可以大于 1 ，表示被重入
    private volatile int state;
    
    //利用CAS操作更新state值
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
    
}
```

以上包含了 AQS 的基本属性字段，比较核心的内容是：

* 等待队列：FIFO 队列，记录头尾指针

* 同步状态：通过 CAS 设置，获取，更新锁状态


我们可以看到等待队列是由 Node 类组成的一个链表，Node 类是 AQS 内定义的静态内部类，我们直接查看 Node 类的属性:

```java
static final class Node {
    // 标识节点当前在共享模式下
    static final Node SHARED = new Node();
    // 标识节点当前在独占模式下
    static final Node EXCLUSIVE = null;

    // ======== 下面的几个int常量是给waitStatus用的 ===========
    /** waitStatus value to indicate thread has cancelled */
    //此线程已取消争抢这个锁
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    //官方的描述是，其表示当前node的后继节点对应的线程需要被唤醒
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    static final int CONDITION = -2;
    
    
    static final int PROPAGATE = -3;

    
    volatile int waitStatus;

    //前驱节点的引用
    volatile Node prev;

    //后驱节点的引用
    volatile Node next;

    //线程本尊
    volatile Thread thread;

    //下一个进入等待的节点
    Node nextWaiter;
}
```

从以上代码可以得知，Node 的数据结构比较简单，就是由 thread(当前线程)+waitStatus(状态)+pre(前置节点)+next(后置节点) 等几个属性而已。

## lock() - 线程抢锁

我们对 AQS 的使用一般是通过 ReentrantLock 相关类，调用 lock() 以及 unlock() 方法实现加锁和解锁。并且 ReentrantLock 在内部用了内部类 Sync 来管理锁，所以真正的获取锁和释放锁是由 Sync 的实现类来控制的。所以我们直接进入 FairSync 类，查看其如何实现线程抢锁的

```java
//公平锁的同步内部静态类
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    // 争锁
    final void lock() {
        acquire(1);
    }

    //尝试直接获取锁，返回值是boolean，代表是否获取到锁
    //返回true：1.没有线程在等待锁；2.重入锁，线程本来就持有锁，也就可以理所当然可以直接获取
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // state == 0 此时此刻没有线程持有锁
        if (c == 0) {
             // 虽然此时此刻锁是可以用的，但是这是公平锁，既然是公平，就得讲究先来后到，
            // 看看有没有别人在队列中等了半天了
            if (!hasQueuedPredecessors() &&
                // 如果没有线程在等待，那就用CAS尝试一下，成功了就获取到锁了，
                // 不成功的话，只能说明一个问题，就在刚刚几乎同一时刻有个线程抢先了 =_=
                // 因为刚刚还没人的，我判断过了
                compareAndSetState(0, acquires)) {
                // 到这里就是获取到锁了，标记一下，告诉大家，现在是我占用了锁
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 会进入这个else if分支，说明是重入了，需要操作：state=state+1
        // 这里不存在并发问题
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        // 如果到这里，说明前面的if和else if都没有返回true，说明没有获取到锁
        // 回到上面一个外层调用方法继续看:
        // if (!tryAcquire(arg) 
        //        && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
        //     selfInterrupt();
        return false;
    }
}

// 我们看到，这个方法，如果tryAcquire(arg) 返回true, 也就结束了。
// 否则，acquireQueued方法会将线程压到队列中
public final void acquire(int arg) { // 此时 arg == 1
    // 首先调用tryAcquire(1)一下，名字上就知道，这个只是试一试
    // 因为有可能直接就成功了呢，也就不需要进队列排队了，
    // 对于公平锁的语义就是：本来就没人持有锁，根本没必要进队列等待(又是挂起，又是等待被唤醒的)
    if (!tryAcquire(arg) &&
        // tryAcquire(arg)没有成功，这个时候需要把当前线程挂起，放到阻塞队列中。
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
          selfInterrupt();
    }
}

//此方法位于 Node 类中
//创建节点并加入等待队列中
//参数：mode – Node.EXCLUSIVE 表示独占，Node.SHARED 表示共享
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    //等待队列尾节点
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            //加入等待队列成功,提前返回结果
            return node;
        }
    }
    //尾节点为空或加入失败,再次通过循环的方式竞争加入,直到加入成功
    enq(node);
    return node;
}


//通过CAS更新尾节点
private final boolean compareAndSetTail(Node expect, Node update) {
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}

// 下面这个方法，参数node，经过addWaiter(Node.EXCLUSIVE)，此时已经进入阻塞队列
// 注意一下：如果acquireQueued(addWaiter(Node.EXCLUSIVE), arg))返回true的话，
// 意味着上面这段代码将进入selfInterrupt()，所以正常情况下，下面应该返回false
// 这个方法非常重要，应该说真正的线程挂起，然后被唤醒后去获取锁，都在这个方法里了
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //如果 当前节点的前一个节点是 head 说明当前节点已经是阻塞队列第一个,所以当前节点直接尝试获取锁
            if (p == head && tryAcquire(arg)) {
                //直接将当前节点设置为头节点,以此来将节点移除等待队列
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 到这里，说明上面的if分支没有成功，要么当前node本来就不是队头，
            // 要么就是tryAcquire(arg)没有抢赢别人，继续往下看
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

//会进入这个方法是因为当前node并不是队列头，或者尝试抢占锁失败，就需要查找到真正的队列头，设置其状态
//返回 true 表示前驱节点正常
//返回 false 说明前驱节点并不是正常状态，会在上一个 acquireQueued 方法中重走此方法，最终结果就会返回 true
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 前驱节点的 waitStatus == -1 ，说明前驱节点状态正常，当前线程需要挂起，直接可以返回true
    if (ws == Node.SIGNAL)
        return true;
    // 前驱节点 waitStatus大于0 ，之前说过，大于0 说明前驱节点取消了排队。
    // 这里需要知道这点：进入阻塞队列排队的线程会被挂起，而唤醒的操作是由前驱节点完成的。
    // 所以下面这块代码说的是将当前节点的prev指向waitStatus<=0的节点，
    // 简单说，就是为了找到仍然在等待的队列头
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
        // 进入这个分支说明找到状态小于 -1 的节点了，直接将该节点状态修改为 SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

// 这个方法很简单，因为前面返回true，所以需要挂起线程，这个方法就是负责挂起线程的
// 这里用了LockSupport.park(this)来挂起线程，然后就停在这里了，等待被唤醒
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}


//通过自旋的方式设置尾节点,因为可能会同时有多个线程竞争入队
//从这段代码可以得知,head 头节点其实是一个空的初始化的值,属于哨兵节点
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        //如果尾节点为空,初始化头节点
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //设置尾节点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

最后我们总结 lock() 方法抢夺锁的步骤:

* 当前线程尝试直接获取锁，获取成功则直接标记当前占有锁的线程为当前线程。
* 如果直接获取锁失败，则将当前线程加入等待队列
* 如果等待队列头为空，则表示队列为空，先设置一个空节点为队列头，再将当前线程入队，如果不为空则直接将当前线程入队尾
* 如果当前线程位于等待队列头，继续抢夺锁
* 如果当前线程不位于等待队列头，或者抢夺锁失败，就会将当前线程挂起，并遍历当前线程前面的所有 prev 节点，如果前置节点状态为 SIGNAL，则直接挂起当前线程，如果前置节点已经取消排队，则往上递归找到对应节点，并将节点状态修改为 SIGNAL，不管如何，最后都会将当前线程挂起。

## unlock() - 线程解锁

当线程占有锁，并执行完之后，需要执行解锁操作，释放资源，接下来我们直接进入 unlock() 的源码:

```java
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    //尝试解锁
    if (tryRelease(arg)) {
        Node h = head;
        //唤醒等待队列下一个线程
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    //如果 c == 0 说明没有嵌套锁了，可以释放了
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}

// 唤醒当前节点的后继节点，从上面代码我们可知当前节点其实是头节点
private void unparkSuccessor(Node node) {
    // 如果head节点当前waitStatus<0, 将其修改为0
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

     // 下面的代码就是唤醒后继节点，但是有可能后继节点取消了等待（waitStatus==1）
    // 从队尾往前找，找到waitStatus<=0的节点，该节点就是等待队列中排在最前面的未取消节点
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;节点
    }
    if (s != null)
        // 唤醒线程
        LockSupport.unpark(s.thread);
}
```

从以上代码我们总结一下 unlock() 解锁的步骤：

* 释放锁，清除当前占有锁的线程标记
* 唤醒当前节点之后排在最前面的等待线程

# 总结

在并发条件下，AQS 通过对以下几个部件的控制实现对资源的加锁和解锁：

* state： 同步状态，通过 state 状态判断锁是否被线程占用，它为 0 的时候代表没有线程占有锁，可以去争抢这个锁，用 CAS 将 state 设为 1，如果 CAS 成功，说明抢到了锁，这样其他线程就抢不到了，如果锁重入的话，state进行 +1 就可以，解锁就是减 1，直到 state 又变为 0，代表释放锁。
* 线程的挂起和唤醒：AQS 中采用了 LockSupport.park(thread) 来挂起线程，用 unpark 来唤醒线程。
* 阻塞队列：阻塞队列由 Node 类实现，其中包含了线程，waitStatus 线程状态等属性，因为争抢锁的线程可能有很多，但是只能有一个线程能拿到锁，所以必须建造一个阻塞队列来管理其他等待的线程。AQS 使用的是一个 FIFO 的队列，每个 node 都有前继和后继节点的引用。
