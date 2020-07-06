---
title: Java并发编程--同步工具类
date: 2020-07-01 23:08:00
tags: Java并发编程
description: 闭锁、FutureTask、信号量、栅栏等同步工具类的用法详解
---
## 内容摘要
同步工具类：根据自身的状态来协调线程的控制流。
所有的同步工具类都包含一些特定的结构化属性：封装一些状态，这些状态将决定执行同步工具类的线程是继续执行还是等待，此外还提供了一些方法对状态进行操作。

* [闭锁 (CountDownLatch)](#闭锁)
* [FutureTask](#futuretask)
* [信号量 (Semaphore)](#信号量)
* [栅栏（CyclicBarrier）](#栅栏)

## 闭锁

**主要方法**：

| 方法 | 返回值 | 描述 |
|:---:|:---:|:---:|
| countDown() | void | 计数器递减，如果计数达到零，释放所有等待的线程 |
| await() | void | 阻塞，直到计数器变为零，释放所有等待线程 |
| getCount() | long | 返回当前计数 |


**闭锁的作用** ：
> 延迟线程的进度，等待线程到达终止状态。

CountDownLatch 可以使一个或多个线程等待一组事件发生。闭锁状态包括一个计数器，该计数器初始化为一个正数表示需要等待的事件数量。countDown() 方法递减计数器，表示有一个事件已经发生，而 await() 方法等待计数器达到零，这表示所有需要等待的事件都已经发生。如果计数器的值非零，那么 await() 方法会一直阻塞直到计数器为零，或者等待中的线程中断，或者等待超时。

**CountDownLatch 应用场景实例**：
我要实现一次性下载多个文件并打包所有文件的功能，为了提高下载速率，使用了线程并发执行下载方法。但是线程执行的速度有快有慢，我们需要使用闭锁等待所有线程下载完毕之后再同时将所有文件打包。

示例代码如下：

```java
/**
 * 1. 新建10个线程，初始化 CountDownLatch 数量为10
 * 2. 每个线程执行下载任务
 * 3. 每个线程执行完后使用 CountDownLatch 的 countDown() 方法减一
 * 4. CountDownLatch 的 await() 方法等待所有线程下载完成
 * 5. 最后将所有文件打包
 * @throws Exception
 * @author Judy.Feng on 2020/07/02
 */
public ZipOutputStream countDownLatchTest() throws Exception {
	int count = 10;
	CountDownLatch countDownLatch = new CountDownLatch(10);
	List<InputStream> fileList = new ArrayList<>();
	for (int i = 0; i < count ; i++) {
		try {
			new Thread(() -> {
				InputStream file = downloadFile();
				fileList.add(file);
			}).start();
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			countDownLatch.countDown();
		}
	}
	//需要 await 等待所有文件下载完成才可以继续打包
	countDownLatch.await();
	return packageFile(fileList);
}
```
## FutureTask

**主要方法**：

| 方法 | 返回值 | 描述 |
|:---:|:---:|:---:|
| get() | V | 阻塞方法，等待任务完成之后才返回结果 |

**FutureTask的作用**：
> FutureTask 是创建线程的方式之一，同时也可用作闭锁。

FutureTask 实现了 Future 语义，表示一种抽象的可生成结果的 Runnable，并且可以处于以下 3 种状态： 等待运行，正在运行和运行完成。
Future.get() 是阻塞方法，如果任务已经完成，那么 get() 会立即返回结果，否则 get() 将阻塞直到任务进入完成状态，然后返回结果或者抛出异常。

**适用场景**：
FutureTask 适用于需要异步获取任务结果的场景，或者需要等待其他线程任务执行完毕才能继续进行下一步的场景。

我们将上面打包文件的代码修改一下：
```java
/**
 * 1. 使用 FutureTask 创建10个线程
 * 2. 线程分别下载文件
 * 3. FutureTask.get() 方法阻塞获取已下载完成的文件
 * 4. 对所有已下载文件进行打包
 * @throws Exception
 * @author Judy.Feng on 2020/07/02
 */
public ZipOutputStream futureTaskTest() throws Exception {
	int count = 10;
	List<FutureTask<InputStream>> fileList = new ArrayList<>();
	List<InputStream> inputStreamList = new ArrayList<>();
	for (int i = 0; i < count ; i++) {
		FutureTask<InputStream> task = new FutureTask<InputStream>(() -> {
			InputStream file = downloadFile();
			return file;
		});
		fileList.add(task);
		new Thread(task).start();
	}
	for (FutureTask<InputStream> file : fileList) {
		inputStreamList.add(file.get()); //阻塞获取所有已下载的文件
	}
	return packageFile(inputStreamList);
}
```

## 信号量

**主要方法**：

| 方法 | 返回值 | 描述 |
|:---:|:---:|:---:|
| acquire() | void | 从信号量池中获取许可，如果没有则阻塞 |
| release() | void | 释放许可，将其返回到信号量 |

**计数信号量的作用**：
> 在Java并发编程中，信号量的作用是控制线程并发的数量。

线程能更好地利用多核 CPU 的性能，提高资源利用率，但是如果线程过多反而会引起过多的上下文切换，造成极大的性能开销。这时使用信号量来控制线程并发数量就能更好地提高线程的效率。

**适用场景**：
信号量可以用来实现某种资源池，或者对容器施加边界。例如数据连接池，我们可以构造一个固定长度的资源池，当池为空时，任务会阻塞，直到资源池不为空。这样可以将数据库连接控制在一定范围内，避免资源浪费。


依然是打包文件的代码，我们有没有考虑过，如果要同时下载20个，30个甚至更多的文件，那么就要开启几十个线程，这样必然带来极大的性能开销，这个时候我们就需要信号量将同时下载的线程数量控制在一定数量内。

代码如下：
```java
/**
 * 1. 创建50个线程
 * 2. 新建许可数量为10的 Semaphore 信号量
 * 3. 每个线程启动时需要通过 Semaphore.qcquire() 方法获取许可，如果池为空，则阻塞
 * 4. 线程完成任务后需要调用 Semaphore.release() 方法释放许可
 * 5. 所有任务下载完毕之后对所有已下载文件进行打包
 * @throws Exception
 * @author Judy.Feng on 2020/07/04
 */
public ZipOutputStream countDownLatchTest() throws InterruptedException {
	int count = 100;
	Semaphore semaphore = new Semaphore(10);
	CountDownLatch countDownLatch = new CountDownLatch(100);
	List<InputStream> fileList = new ArrayList<>();
	for (int i = 0; i < count ; i++) {
		try {
			new Thread(() -> {
				try {
				    //请求许可
					semaphore.acquire();
					InputStream file = downloadFile();
					fileList.add(file);
				} catch (InterruptedException e) {
					Thread.currentThread().interrupt();
				} finally {
				    //释放许可
					semaphore.release();
				}
			}).start();
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			countDownLatch.countDown();
		}
	}
	countDownLatch.await();
	return packageFile(fileList);
}
```

## 栅栏

**主要方法**：

| 方法 | 返回值 | 描述 |
|:---:|:---:|:---:|
| await() | int | 阻塞等待所有线程都已经调用该方法|
| reset() | void | 将屏障重置为初始状态 |


**栅栏的作用**：
> 栅栏类似于闭锁，它能阻塞一组线程直到某个事件发生。

**适用场景**：
CyclicBarrier可以用于多线程计算数据，最后合并计算结果的应用场景。并且CyclicBarrier 可以重置计数器，可以处理更加复杂的业务场景，在并行迭代算法中非常有用。

**栅栏和闭锁的区别**:
栅栏与闭锁的关键区别在于，所有线程必须同时到达栅栏位置才能继续执行。闭锁用于等待事件，而栅栏用于等待其他线程。

代码如下：
```java
/**
 * 1. 新建10个线程，初始化 CyclicBarrier 栅栏数量为10
 * 2. 每个线程执行下载任务
 * 3. 每个线程执行完任务后调用 CyclicBarrier.await() 方法阻塞等待其他线程到达栅栏
 * 4. 最后将所有文件打包
 * @throws Exception
 * @author Judy.Feng on 2020/07/05
 */
public ZipOutputStream countDownLatchTest() throws InterruptedException {
	int count = 10;
	CyclicBarrier barrier = new CyclicBarrier(10);
	List<InputStream> fileList = new ArrayList<>();
	for (int i = 0; i < count ; i++) {
		new Thread(() -> {
			InputStream file = downloadFile();
			fileList.add(file);
			try {
				barrier.await(); //阻塞等待所有线程达到栅栏位置
			} catch (Exception ex) {
				ex.printStackTrace();
			}
		}).start();
	}
	return packageFile(fileList);
}
```
