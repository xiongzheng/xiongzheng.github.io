---
layout: post
title: "ThreadPoolExecutor 类"
description: ""
category: 每日一C
tags: []
---
{% include JB/setup %}


话说小八的国庆过的还是很哈皮的，回武汉，参加了大学同学的婚礼，见了有两年没见的同学们，都带着小孩参加，咿咿呀呀、打打闹闹，好生热闹。然后去老婆家玩了两天，跟老婆的妹妹和妹夫们打打小麻将--红中赖子杠；去二妹家的新房子做客，房子很大、环境便利、伯伯的菜烧的不错。回到家里，去了趟宜家逛逛，话说家门口慢慢变得热闹了，附近就有园博会、宜家、还有大的购物中心，挺好。提前返杭后，正值结婚纪念日，找到借口逛街、购物、看电影，非常完美的一天。

国庆将过，收心工作。工作前为大家带来的一篇写Class的文章。

> ThreadPoolExecutor

#### 提问 & 回答
很久很久以前，有一次团队分享会，会上抛了个问题：线程池增加线程的机制是怎么样的？
A.线程池线程在corePoolSize~maximumPoolSize之间增减，如果超过maximumPoolSize，则放入缓冲队列等待执行。
B.线程池线程如果大于等于corePoolSize，则放入缓冲队列，缓冲队列如果有界被填满，则任务直接创建新线程执行并归入线程池管理，当超过maximumPoolSize时，启动拒绝策略。

答案是B，但当时我以及很多同事都是以为是A的，不读源码害死人啊。如果错误的理解成A，那么corePoolSize、maximumPoolSize以及任务缓冲队列的容量很可能会设置为非常奇怪的不恰当的值(比如corePoolSize=1，maximumPoolSize=100，队列容量=1000，这性能是要多烂)，给线上应用留下隐患。

> 线程池线程管理机制：
> 
> a.当前线程数小于corePoolSize时，任务直接新建线程运行，并归入线程池管理；当运行线程数大于等于corePoolSize时，任务优先加入任务队列中。
> 
> b.当队列中任务数达到设定的容量边界时，则新增的任务直接新建线程运行，并归入线程池管理；直到线程数达到maximumPoolSize，如果还有新增任务，就会拒绝，使用拒绝策略处理。
> 
> c.等线程执行完成任务以后，再从任务队列中取新任务执行。
> 
> d.当线程空闲时间，大于设定的值keepAliveTime(单位：unit)以后，线程就会回收。
> 
> e.默认创建线程池时，corePoolSize个线程不会马上创建，采用懒加载方式当有任务到来时加载线程。可以通过prestartAllCoreThreads()方法将核心线程都初始化好，避免线程池冷启动时的性能过低。
> 
> f.默认核心线程一旦初始化，除非线程异常，否则都在线程池中存在，执行任务或者等待执行。可以通过设置allowCoreThreadTimeOut，使得核心线程也在空闲时间(keepAliveTime)结束后回收。
> 
> g.largestPoolSize，该变量记录了线程池在整个生命周期中曾经出现的最大线程个数。
> 
> h.线程池创建之后，可以调用setCorePoolSize()改变运行的核心线程数，调用setMaximumPoolSize()改变运行的最大线程数。

#### 参数

```
public ThreadPoolExecutor(
int corePoolSize,					// 核心线程数
int maximumPoolSize,				// 最大线程数
long keepAliveTime,					// 超过核心线程数时，闲置线程的存活时间
TimeUnit unit,						// keepAliveTime的单位
BlockingQueue<Runnable> workQueue,	// 任务队列
ThreadFactory threadFactory,		// 线程创建工厂
RejectedExecutionHandler handler	// 拒绝策略的处理者
)
```

设置样例：

```
new ThreadPoolExecutor(5, 20, 1, TimeUnit.MINUTES, new LinkedBlockingQueue<Runnable>(200), 
Executors.defaultThreadFactory(), new AbortPolicy())
```

> 注意：这里LinkedBlockingQueue要设置容量，如果不设置容量则为无界队列。当高并发、任务执行时间过长时，可能引起内存溢出。

#### 队列
阻塞队列用于对线程池任务进行缓冲，平衡生产端和消费端处理速度。

JDK自带的队列有：

SynchronousQueue 		同步阻塞队列；在某次添加元素后必须等待其他线程取走后才能继续添加。

ArrayBlockingQueue		有界队列，数组实现；在生产者放入数据和消费者获取数据，都是共用同一个锁对象，由此也意味着两者无法真正并行运行。

LinkedBlockingQueue 	无界队列，链表实现，设置容量使之有界；因为其对于生产者端和消费者端分别采用了独立的锁来控制数据同步，这也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。

PriorityBlockingQueue	优先级队列；可定义任务优先级别，按照优先级别排队。



#### 拒绝策略
ThreadPoolExecutor中包含四种处理策略，也可自行继承RejectedExecutionHandler扩展：

CallerRunsPolicy：被拒绝的任务直接交由生产者本身执行。此策略提供简单的反馈控制机制，能够减缓新任务的提交速度。

AbortPolicy：处理程序遭到拒绝将抛出运行时RejectedExecutionException。

DiscardPolicy：被拒绝的任务直接被丢弃。

DiscardOldestPolicy：如果执行程序尚未关闭，则位于工作队列头部的任务将被删除，然后重试执行程序（如果再次失败，则重复此过程）。



#### 附录：测试用例

```
package daily.c;

import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.ThreadPoolExecutor.AbortPolicy;
import java.util.concurrent.TimeUnit;

public class ThreadPoolExecutorTest {
	public static void main(String[] args) {
		ThreadPoolExecutor threadPool = new ThreadPoolExecutor(2, 4, 1, TimeUnit.MINUTES,
				new LinkedBlockingQueue<Runnable>(2), Executors.defaultThreadFactory(), new AbortPolicy());
		pInfo(threadPool);
		/**
		 * CorePoolSize:2 MaximumPoolSize:4 PoolSize:0 QueueSize:0 
		 * 初始时没有创建线程。
		 */

		threadPool.execute(new Task());
		pInfo(threadPool);
		/**
		 * CorePoolSize:2 MaximumPoolSize:4 PoolSize:1 QueueSize:0
		 * 当调用execute方法执行一个任务时，由于线程数小于核心线程数，所以直接创建线程执行任务，并归入线程池管理。
		 */

		threadPool.execute(new Task());
		threadPool.execute(new Task());
		pInfo(threadPool);
		/**
		 * CorePoolSize:2 MaximumPoolSize:4 PoolSize:2 QueueSize:1
		 * 再增加两个任务时，任务数(3个)大于核心线程数2，则后一个任务被放入任务队列。
		 */

		try {
			TimeUnit.SECONDS.sleep(10);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		pInfo(threadPool);
		/**
		 * CorePoolSize:2 MaximumPoolSize:4 PoolSize:2 QueueSize:0
		 * 待任务执行完，任务队列消减为0，线程数==核心线程数。
		 */

		for (int i = 0; i < 7; i++) {
			try {
				threadPool.execute(new Task());
				pInfo(threadPool);
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
		/**
		 * CorePoolSize:2 MaximumPoolSize:4 PoolSize:2 QueueSize:1
		 * CorePoolSize:2 MaximumPoolSize:4 PoolSize:2 QueueSize:2
		 * CorePoolSize:2 MaximumPoolSize:4 PoolSize:3 QueueSize:0
		 * CorePoolSize:2 MaximumPoolSize:4 PoolSize:3 QueueSize:1
		 * CorePoolSize:2 MaximumPoolSize:4 PoolSize:3 QueueSize:2
		 * CorePoolSize:2 MaximumPoolSize:4 PoolSize:4 QueueSize:2
		 * java.util.concurrent.RejectedExecutionException
		 * 当需要执行的任务数超过最大线程数+任务队列容量时，拒绝策略AbortPolicy开始处理，抛出异常。
		 * 
		 * 上面的输出序列不确定，由于打印是在主线程执行，任务是在线程池执行。但可以看出当QueueSize填满后，PoolSize会增长，
		 * 但不超过MaximumPoolSize。
		 */

		threadPool.shutdown();
		/**
		 * 等待所有任务执行完毕，关闭线程池。
		 */
	}

	private static void pInfo(ThreadPoolExecutor threadPool) {
		StringBuilder sb = new StringBuilder();
		sb.append("CorePoolSize:");
		sb.append(threadPool.getCorePoolSize());
		sb.append("\t");
		sb.append("MaximumPoolSize:");
		sb.append(threadPool.getMaximumPoolSize());
		sb.append("\t");
		sb.append("PoolSize:");
		sb.append(threadPool.getPoolSize());
		sb.append("\t");
		sb.append("QueueSize:");
		sb.append(threadPool.getQueue().size());
		sb.append("\t");

		System.out.println(sb.toString());
	}

	public static class Task implements Runnable {
		@Override
		public void run() {
			try {
				TimeUnit.SECONDS.sleep(2);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```         
               