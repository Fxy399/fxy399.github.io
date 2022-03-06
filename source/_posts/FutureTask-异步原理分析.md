---
title: FutureTask 源码解析
date: 2022-03-06 14:35:39
tags: Java并发编程
description: FutureTask 获取任务状态、取消任务、阻塞获取结果的源码实现分析
---

# 前言

**首先我们必须要知道一点：`FutureTask` 本身可以说就是为了线程池而设计的。**

在 JDK1.5 之前， Java 中还没有提供 Future 和 Callable 相关接口，创建线程只能通过 Thread 类或者是 Runnable 接口，这两种方式最大的缺点是在线程异步执行完任务之后无法获取线程的执行结果。 JDK1.5 之后，Doug Lea 大师给我们提供了 Future 和 Callable 几个相关工具类，方便我们异步获取线程的执行结果。

# Future 接口

`FutureTask` 实现了 Future 接口类，我们先来看看 Future 的接口方法

```java
public interface Future<V> {
	//取消任务
	boolean cancel(boolean mayInterruptIfRunning);
	//判断任务是否已被取消
	boolean isCancelled();
	//判断任务是否完成
	boolean isDone();
	//获取计算结果，若计算未完成则阻塞等待，直到任务结束返回结果或者线程中断
	V get() throws InterruptedException, ExecutionException;
	//返回计算结果，若计算未完成则阻塞等待，直到任务结束或线程中断或任务超时
	V get(long timeout, TimeUnit unit) throws InterruptedException,
	ExecutionException, TimeoutException;
}
```

从以上接口方法，我们可以判断出 Future 的实现类必须实现的几种功能：

1. 对异步任务进行控制（取消等操作）
2. 获取异步任务的状态
3. 阻塞获取异步任务的计算结果

接下来我们就进入 `FutureTask` 源码看看是如何实现以上功能的。

# FutureTask 实现

首先，我们发现 `FutureTask` 实现了 `RunnableFuture` 类，而`RunnableFuture`接口又继承了`Runnable`和`Future`接口。因此`FutureTask`本身就是一个`Runnable`对象，可以作为一个任务提交到线程池中，更不用说 `FutureTask` 本身就是为了 线程池而设计。

我们从 `AbstractExecutorService` 的 `newTaskFor()` 方法中可以得知， `FutureTask` 来源于Callable 或者 Runnable , 其内核是一个 Callable 类，FutureTask 所实现的一切方法都是基于对 Callable 的处理（可以说是套壳的 Callable，手动狗头）。

## run() 方法

话不多说直接进入 `FutureTask` 的 run() 方法,该方法实现任务的执行操作。

```java
public class FutureTask<V> implements RunnableFuture<V> {
		/**
     * 任务的运行状态（初始化状态为 NEW）
     */
    private volatile int state; //用 volatile 修饰，保持内存可见性
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

		/** The underlying callable; nulled out after running */
    private Callable<V> callable;
    /** The result to return or exception to throw from get() */
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters;

		public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }

  public FutureTask(Runnable runnable, V result) {
      this.callable = Executors.callable(runnable, result);
      this.state = NEW;       // ensure visibility of callable
  }    

 public void run() {
      //第一步先判断状态是否是 NEW ,防止并发执行
      if (state != NEW ||
          !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                       null, Thread.currentThread()))
          return;
      try {
          Callable<V> c = callable;
          if (c != null && state == NEW) {
              V result;
              boolean ran;
              try {
								//执行任务
                result = c.call();
                ran = true;
              } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
              }
              if (ran)
                //如果任务执行成功，则将执行结果传入该方法
                set(result);
          }
      } finally {
          // runner must be non-null until state is settled to
          // prevent concurrent calls to run()
          runner = null;
          // state must be re-read after nulling runner to prevent
          // leaked interrupts
          int s = state;
          if (s >= INTERRUPTING)
              handlePossibleCancellationInterrupt(s);
      }
  }

		/**
      * 1. 先将任务状态从 NEW 转到 COMPLETING，再将结果赋值到 outcome 中
			* 2. 将任务状态从 COMPLETING 转到 NORMAL
      * 3. 进入 finishCompletion() 方法
			*/
		protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
		/**
     * 对线程栈中对象逐个调用 LockSupport.unpark(t) 唤醒阻塞的线程
		 * 并出栈（使用CAS操作实现并发出栈）
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {4444444
            //在不使用锁的情况下通过cas并发出栈
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
    ...
}
```

从上面代码可以看到 run() 方法的执行流程，首先判断状态是否 NEW （开始状态），防止并发执行，然后 callable.call() 方法执行任务，任务成功执行之后进入 set(result) 方法， 该方法的任务是改变任务状态，并将任务执行结果赋值给 outcome 对象，最后通过 finishCompletion() 方法清空等待线程，实现方式是遍历等待线程，唤醒阻塞中的线程并将线程出栈。

## get() 方法

接下来我们进入 get() 方法，该方法能够通过阻塞获取任务执行结果。

```java
public V get() throws InterruptedException, ExecutionException {
		int s = state;
		if (s <= COMPLETING)
       //阻塞等待结果
	    s = awaitDone(false, 0L);
		return report(s);
}

/**
     * Awaits completion or aborts on interrupt or timeout.
     * 1. 判断线程是否中断，如果中断则移除等待线程
     * 2. 将当前线程设置为 WaitNode，并加入等待队列
     * 3. 判断当前任务是否超时，如果超时则直接唤醒线程并移除等待队列
     * 如果任务不超时，则使等待线程阻塞
     * 4. 最后当任务完成直接释放线程
     */
private int awaitDone(boolean timed, long nanos)
      throws InterruptedException {
      final long deadline = timed ? System.nanoTime() + nanos : 0L;
      WaitNode q = null;
      boolean queued = false;
      for (;;) {
        //如果线程中断则将本等待线程出栈
          if (Thread.interrupted()) {
              removeWaiter(q);
              throw new InterruptedException();
          }

          int s = state;
          if (s > COMPLETING) {
              if (q != null)
                  q.thread = null;
              return s;
          }
          //任务完成直接释放线程
          else if (s == COMPLETING) // cannot time out yet
              Thread.yield();
          else if (q == null)
              //将当前线程设置为 WaitNode
              q = new WaitNode();
          else if (!queued)
              //将当前线程加入等待队列
              queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                   q.next = waiters, q);                                     
           //判断当前任务是否超时，如果超时则直接唤醒线程并移除等待队列
          //如果任务不超时，则使等待线程阻塞
					else if (timed) {
              nanos = deadline - System.nanoTime();
              if (nanos <= 0L) {
                  removeWaiter(q);
                  return state;
              }
              LockSupport.parkNanos(this, nanos);
          }
          else
              LockSupport.park(this);
      }
}
```

## ****Treiber stack****

从上面方法中，我们发现调用 get() 时会将调用线程放进一个等待队列中，任务执行完成之后才会唤醒线程，或者如果任务被中断或超时，等待队列则会被移除。

看到对于等待线程栈 waiters 的 CAS 操作，由此我们引申出 Treiber Stack 这个概念。

- Treiber stack是一种 **栈** 数据结构，最早是由 **R. Kent Treiber** 在1986年的发布的论文《Systems Programming: Coping with Parallelism》中提出
，支持无锁入栈和出栈操作，在并发情况下有良好的访问性能。
- reiber stack结构上和普通的栈数据结构没有区别，唯一的不同是在入栈和出栈的时候，不同于阻塞的数据结构：需要加锁来保护栈顶指针，在Treiber stack的实现中，采用了CompareAndSwap机制来更新栈顶的指针以实现无锁入队和出队操作。[^1]



在 FutureTask 中，reiber stack 是 **`WaitNode`  waiters 队列**，并且通过`awaitDone()` 和 `finishCompletion()` 方法来实现通过 CAS 机制来入栈和出栈的操作。

```java
//等待线程链表
static final class WaitNode {
    volatile Thread thread;
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}

private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                    //遍历移除数据
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        //如果另外一个线程已经移除本节点，将回到 retry: 处从头遍历
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    //这里使用cas锁来实现出栈操作，将指针指向原顶节点的下一个节点
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }
```

# 线程安全的实现

从上面对 `FutureTask` 的源码解析中，我们也学习到了不少并发编程的知识，认识到 Doug Lea 大师是如何实现任务的线程安全共享。

1. **volatile 关键字：**FutureTask 通过 `volatile` 关键字修饰任务状态属性，以此来保持任务状态的线程安全共享。
2. ****Treiber stack：****通过 CAS 对栈顶指针进行无锁入栈出栈实现无锁并发栈。

# 结论

`FutureTask` 实现了 Future 接口，`FutureTask` 主要用于实现异步任务的计算，并且提供了一系列接口来访问异步计算结果，比如我们常用的 get() 方法。

通俗一点讲，`FutureTask` 的作用是将 Callable、Runnable 等任务执行类包装起来（包装方法位于 `AbstractExecutorService` 的 `newTaskFor()中`），让使用者可以获取任务状态、取消任务、阻塞获取任务结果。

参考文章：

[FutureTask实现原理](https://tech101.cn/2019/10/04/FutureTask%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)
