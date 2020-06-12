---
---

# 线程池状态

```java
  /*
    ctl变量高3位记录线程池状态，低29位记录当前Worker总数，因此线程池最大线程数为
    2^29 - 1 = 536870911
  */
  private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
  private static final int COUNT_BITS = Integer.SIZE - 3;
  private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

  // 线程池状态定义
  private static final int RUNNING    = -1 << COUNT_BITS;
  private static final int SHUTDOWN   =  0 << COUNT_BITS;
  private static final int STOP       =  1 << COUNT_BITS;
  private static final int TIDYING    =  2 << COUNT_BITS;
  private static final int TERMINATED =  3 << COUNT_BITS;

  // 状态转换辅助方法
  private static int runStateOf(int c)     { return c & ~CAPACITY; }
  private static int workerCountOf(int c)  { return c & CAPACITY; }
  private static int ctlOf(int rs, int wc) { return rs | wc; }
```

线程池5种状态，分别是：

1. `RUNNING`: 接受新提交任务并能处理队列中的任务
2. `SHUTDOWN`: 不再接受新提交的任务但仍继续处理队列中已存在的任务
3. `STOP`: 不接受新提交的任务、不处理剩余任务同时中断正在执行中的任务
4. `TIDYING`: `STOP`或`SHUTDOWN`状态下当任务队列及线程池均为空时转为该状态
5. `TERMINATED`: `TIDYING`状态执行完`terminated()`方法后由转为该状态

# Worker类

线程池中工作线程被封装成`Worker`对象。`Worker`类继承AQS实现了**不可重入**锁，根据锁状态来判断该线程是否有正在执行的任务。

不使用`ReentrantLock`是因为线程池`interruptIdleWorkers()`方法通过`worker.tryLock()`判断是否当前线程正在运行，如果可重入会导致正在运行的形成被中断。

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {

    // 执行任务的线程
    final Thread thread;
    // 创建线程时候指定的第一个任务，可以为空
    Runnable firstTask;
    // 已完成任务的计数
    volatile long completedTasks;

    Worker(Runnable firstTask) {
        /*
          只有当线程开始执行任务后才响应中断，因此初始化state为负值
        */
        setState(-1);
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    // AQS methods...

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {}
        }
    }
}
```

# `execute()`方法

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    int c = ctl.get();
    // 1. 当前线程少于核心线程，则新建一个线程执行任务
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true)) // true 表示添加线程上限为核心线程数
            return;
        c = ctl.get();
    }
    // 2. 当前已经达到核心线程数，尝试将任务添加到队列中
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 重新检查线程池状态和当前线程数量，如当前线程池处于非运行状态，将任务移出队列成功后，拒绝该任务
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 已存在的线程可能已经被销毁，需要保证至少能有一个线程执行队列中的任务
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 3. 无法将任务添加到队列中时，尝试新建线程执行任务，失败则拒绝该任务
    else if (!addWorker(command, false))
        reject(command);
}
```

# `addWorker()`方法

`addWorker()`会根据当前线程池的状态以及线程数配置决定是否新建工作线程。

以下几种情况时无法新增工作线程：

* 线程池已经停止
* 线程池处于`SHUTDOWN`状态，任务队列为空或请求处理新任务
* 创建线程失败，通常是由于`threadFactory`为空或虚拟机内存不足

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 线程池处于非运行状态，或者当线程池处于SHUTDOWN状态仍请求处理
        // 新任务时，无法添加worker线程
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            // 当前线程数已经到达上限，无法添加worker
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize)) // core参数指定上限为核心线程数还是最大线程数
                return false;
            // 增加工作线程计数，如果CAS失败则重新开始
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            // 确保线程池工作状态没有变化
            if (runStateOf(c) != rs)
                continue retry;
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
                int rs = runStateOf(ctl.get());

                // 线程池处于运行状态或新worker用来处理任务队列中的任务
                // 将worker添加到集合中并记录下该线程池出现过的最大线程数
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
                t.start();  // 开始执行任务
                workerStarted = true;
            }
        }
    } finally {
        // 如果添加失败，则需要进行相应的清理工作，如递减工作线程数、将worker从集合中移除
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

# `runWorker()`方法

当新建工作线程成功之后，`worker`便开始循环执行任务，直到任务队列被清空

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // runWorker方法执行表示worker开始真正执行任务，调用unlock重置worker的state
    // 通过unlock将state置为0，使其能够响应中断
    w.unlock();
    // 是否非正常的结束了主循环（被中断、钩子方法及任务执行抛出异常）
    // 如果为true则需要在结束时修改worker计数
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // 每次循环都确保线程池停止时中断线程，而线程池运行时保证
            // 线程的正常运行
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;  // 置空task，确保下次循环从任务队列中取任务
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // worker已经完成了自己的使命，将其移出集合等待gc；同时还要保证
        // 线程池中线程数量满足最小线程数的配置，线程数不足则调用addWorker
        // 添加工作线程
        processWorkerExit(w, completedAbruptly);
    }
}
```

# `getTask()`方法

除了从队列中获取任务之外，该方法还利用阻塞队列超时获取功能实现超出核心线程数线程存活时间的处理。

```java
private Runnable getTask() {
    boolean timedOut = false;   // 标记是否获取任务超时

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 线程池处于STOP状态或SHUTDOWN状态时任务队列为空，则返回null提示
        // worker主循环结束
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 是否需要进行超时判断，allowCoreThreadTimeOut 设置是否允许核心线程超时
        // 在这里可以看出，如果线程池线程数量少于核心线程数且不允许核心线程超时的情况下，
        // 工作线程可以一直阻塞到任务队列中添加了新任务；而超出了核心线程数的线程如果一
        // 定时间后还未获取到任务则会被销毁，从而完成线程池超核心线程的时间控制
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
