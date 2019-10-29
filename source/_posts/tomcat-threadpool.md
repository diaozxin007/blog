---
title: 那些有趣的代码(一)--有点萌的 Tomcat 的线程池
date: 2019-10-15 00:39:17
tags: [java,线程池,多线程]
---

最近抓紧时间看看了看tomcat 和 jetty 的源代码。发现了一些有趣的代码，这里和大家分享一下。

Tomcat 作为一个老牌的 servlet 容器，处理多线程肯定得心应手，为了能保证多线程环境下的高效，必然使用了线程池。

但是，Tomcat 并没有直接使用 j.u.c 里面的线程池，而是对线程池进行了扩展，首先我们回忆一下，j.u.c 中的线程池的几个核心参数是怎么配合的：

1. 如果当前运行的线程，少于corePoolSize，则创建一个新的线程来执行任务。
2. 如果运行的线程等于或多于 corePoolSize，将任务加入 BlockingQueue。
3. 如果 BlockingQueue 内的任务超过上限，**则创建新的线程来处理任务。**
4. 如果创建的线程超出 maximumPoolSize，任务将被拒绝策略拒绝。

这个时候我们来仔细看看 Tomcat 的代码：

首先写了一个 TaskQueue 继承了非阻塞无界队列 `LinkedBlockingQueue<Runnable>` 并重写了的 offer 方法：

<!--more-->

```java

@Override
public boolean offer(Runnable o) {
    //we can't do any checks
    if (parent==null) return super.offer(o);
    //we are maxed out on threads, simply queue the object
    if (parent.getPoolSize() == parent.getMaximumPoolSize()){
        return super.offer(o);
    }
    //we have idle threads, just add it to the queue
    if (parent.getSubmittedCount()<=(parent.getPoolSize())) {
        return super.offer(o);
    }
    //if we have less threads than maximum force creation of a new thread
    if (parent.getPoolSize()<parent.getMaximumPoolSize()) {
    return false;
    }  
    //if we reached here, we need to add it to the queue
    return super.offer(o);
}
```

在提交任务的时候，增加了几个分支判断。

首先我们看看 parent 是什么:

```java
    private transient volatile ThreadPoolExecutor parent = null;
```

这里需要特别注意这里的 ThreadPoolExecutor 并不是 jdk里面的 java.util.concurrent.ThreadPoolExecutor 而是 tomcat 自己实现的。

我们分别来看 offer 中的几个 if 分支。

首先我们需要明确一下，**当一个线程池需要调用阻塞队列的 offer 的时候，说明线程池的核心线程数已经被占满了。（记住这个前提非常重要）**

要理解下面的代码，首先需要复习一下线程池的 getPoolSize() 获取的是什么？我们看源码：

```java
/**
 * Returns the current number of threads in the pool.
 *
 * @return the number of threads
 */
public int getPoolSize() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // Remove rare and surprising possibility of
        // isTerminated() && getPoolSize() > 0
        return runStateAtLeast(ctl.get(), TIDYING) ? 0
            : workers.size();
    } finally {
        mainLock.unlock();
    }
}
```

需要注意的是，workers.size()  **包含了 coreSize 的核心线程和临时创建的小于 maxSize 的临时线程。**

先看第一个 if

```java
    // 如果线程池的工作线程数等于 线程池的最大线程数，这个时候没有工作线程了，就尝试加入到阻塞队列中
    if (parent.getPoolSize() == parent.getMaximumPoolSize()){
        return super.offer(o);
    }
```

经过第一个 if 之后，线程数必然在核心线程数和最大线程数之间。

```java
if (parent.getSubmittedCount()<=(parent.getPoolSize())) {
    return super.offer(o);
}
```

对于 parent.getSubiitedCount() ,我们要先搞清楚 submiitedCount 是什么

```java
/**
 * The number of tasks submitted but not yet finished. This includes tasks
 * in the queue and tasks that have been handed to a worker thread but the
 * latter did not start executing the task yet.
 * This number is always greater or equal to {@link #getActiveCount()}.
 */
private final AtomicInteger submittedCount = new AtomicInteger(0);
```

这个数是一个原子类的整数，用于记录提交到线程中，且还没有结束的任务数。包含了在阻塞队列中的任务数和正在被执行的任务数两部分之和 。

所以这行代码的策略是，如果已提交的线程数小于等于线程池中的线程数，表明这个时候还有空闲线程，直接加入阻塞队列中。为什么会有这种情况发生？其实我的理解是，之前创建的临时线程还没有被回收，这个时候直接把线程加入到队里里面，自然就会被空闲的临时线程消费掉了。

我们继续往下看：

```java
//if we have less threads than maximum force creation of a new thread
if (parent.getPoolSize()<parent.getMaximumPoolSize()) {
    return false;
}
```

由于上一个 if 条件的存在，走到这个 if 条件的时候，提交的线程数已经大于核心线程数了，且没有空闲线程，所以返回一个 false 标明，表示任务添加到阻塞队列失败。线程池就会认为阻塞队列已经无法继续添加任务到队列中了，根据默认线程池的工作逻辑，线程池就会创建新的线程直到最大线程数。

回忆一下 jdk 默认线程池的实现，如果阻塞队列是无界的，任务会无限的添加到无界的阻塞队列中，线程池就无法利用核心线程数和最大线程数之间的线程数了。

Tomcat 的实现就是为了，线程池即使核心线程数满了以后，且使用无界队列的时候，线程池依然有机会创建新的线程，直到达到线程池的最大线程数。

Tomcat 对线程池的优化并没结束，Tomcat 还重写了线程池的 execute 方法：

```java
public void execute(Runnable command, long timeout, TimeUnit unit) {
    //提交任务数加一
    submittedCount.incrementAndGet();
    try {
        super.execute(command);
    } catch (RejectedExecutionException rx) {
        // 被拒绝以后尝试，再次向阻塞队列中提交任务
        if (super.getQueue() instanceof TaskQueue) {
            final TaskQueue queue = (TaskQueue)super.getQueue();
        try {
            if (!queue.force(command, timeout, unit)) {
                submittedCount.decrementAndGet();
                throw new RejectedExecutionException(sm.getString("threadPoolExecutor.queueFull"));
            }
        } catch (InterruptedException x) {
            submittedCount.decrementAndGet();
            throw new RejectedExecutionException(x);
        }
        } else {
            submittedCount.decrementAndGet();
            throw rx;
        }
    }
}
```

终于到整篇文章的萌点了，就是提交线程的时候，如果被线程池拒绝了，Tomcat 的线程池，还会厚着脸皮再次尝试，调用 force() 方法"强行"的尝试向阻塞队列中添加任务。

![tomcat](https://img.xilidou.com/img/tomcat.png)

在群里和朋友讲完 Tomcat 线程池的实现，帆哥给了一个特别厉害的例子。

总结一下：

Tomcat 线程池的逻辑：

1. 如果当前运行的线程，少于corePoolSize，则创建一个新的线程来执行任务。
2. 如果线程数大于 corePoolSize了，Tomcat 的线程不会直接把线程加入到无界的阻塞队列中，而是去判断，submittedCount（已经提交线程数）是否等于 maximumPoolSize。
3. 如果等于，表示线程池已经满负荷运行，不能再创建线程了，直接把线程提交到队列，
4. 如果不等于，则需要判断，是否有空闲线程可以消费。
5. 如果有空闲线程则加入到阻塞队列中，等待空闲线程消费。
6. 如果没有空闲线程，尝试创建新的线程。**（这一步保证了使用无界队列，仍然可以利用线程的 maximumPoolSize）。**
7. 如果总线程数达到 maximumPoolSize，则继续尝试把线程加入 BlockingQueue 中。
8. 如果 BlockingQueue 达到上限（假如设置了上限），被默认线程池启动拒绝策略，tomcat 线程池会 catch 住拒绝策略抛出的异常，再次把尝试任务加入中 BlockingQueue 中。
9. 再次加入失败，启动拒绝策略。

如此努力的 Tomcat 线程池，有点萌啊。
