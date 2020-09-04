---
typora-copy-images-to: img
typora-root-url: img
---

# 并发编程

并发编程脑图：https://www.processon.com/view/link/5d81dec7e4b04c14c4e7aac8

## ThreadPool



<img src="/图片1.png" alt="图片1" style="zoom:288%;" />

大概执行逻辑：

a. executor.execute（runnable)的时候 将线程封装程一个worker对象，如果当前存在的`线程数 < 核心线程数corepoolsize` 则直接创建线程，执行worker任务。

b. 当`线程数 > 核心线程数` 时， 新来的任务将会被丢到阻塞队列中。

c. 当`阻塞队列中任务数 > 阻塞队列长度` ，新来的任务将会被丢给**非核心线程数**（maximumPoolSize）。

d. 如果再来多余的任务则会执行RejectedExecutionHandler逻辑。



>  状态码， int类型32位，前三位表示状态，后29位表示任务数量

```java

//11100000000000000000000000000000
private static final int RUNNING    = -1 << COUNT_BITS;
//00000000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//00100000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;
//01000000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;
//10000000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;
```

> 代码分析

```java
// 线程池参数
int corePoolSize,   //核心线程数
int maximumPoolSize, //最大线程数
long keepAliveTime, //空闲时间
TimeUnit unit, //时间单位
BlockingQueue<Runnable> workQueue, //存放最大线程数外的任务
ThreadFactory threadFactory, // 
RejectedExecutionHandler handler
```

线程池threadePoolExecutor.submit 最终执行的方法是execute(runnable)， 参考上方的 `大概执行逻辑`

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
    // 获取当前ctl的值， ctl的值，前三位表示执行状态，后29位表示执行的任务数
        int c = ctl.get();
    //workerCountof(c) 就是c & (1 << 29 - 1)   即 c与二进制29个1做&运算。
        if (workerCountOf(c) < corePoolSize) {
            // 如果当前运行的任务数 < 核心线程数   将新增一个worker用于处理当前command任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
    // 线程池状态为running 则把任务添加到阻塞队列中，等待核心线程从阻塞队列中获取任务。
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //重新判断线程池的状态，不是running的话删除任务
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //如果worker的数量为0的话，新增一个null的非核心线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
    //添加非核心线程worker，如果没添加成功，则执行拒绝逻辑。表示已经超过了最大线程数
        else if (!addWorker(command, false))
            reject(command);
    }
```

线程池的任务执行主要在addworker方法中， 该方法有两个参数

a. runnable 线程

b. boolean  core  是否为核心线程。 任务数在小于 线程数+阻塞队列长度 的时候都为核心线程任务。

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
          // 如果线程池状态为非运行状态
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            //cas给ctl+1 失败，则重新加
            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    //如果为核心线程则与corepoolsize比较，否则与maximumpoolsize比较， 小于
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

