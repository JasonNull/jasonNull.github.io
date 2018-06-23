---
title: ThreadPoolExecutor分析
date: 2018-06-23 18:18:15
tags:
- java
- 多线程
- 源码
categories: 
- java
- 多线程
toc: true
---
先思考一个问题，为什么需要是使用线程池？
1. 性能上，创建线程的开销挺大的，时间需要100us，空间也需要在1M左右。
2. 开发上，有任务需要异步执行，但是不需要在当前线程立刻知道任务的执行结果。
3. 系统上，系统可以创建的线程数量是比较固定的。

<!-- more -->
# 注意点
1. 线程池的任务不一定是顺序执行：
 当任务提交进去之后，如果当前核心线程池已满，队列已满，最大线程数未满的话，此时的任务会优先执行
2. 使用线程池的时候，execute需要注意不要抛出异常：
 抛出异常可能导致线程死亡，需要使用ThreadGroup中的异常处理机器

# 成员属性
1. `BlockingQueue<Runnable> workQueue`，任务存储的阻塞队列
2. `ReentrantLock mainLock`，用于控制对worker的操作（创建，删除等等）
3. `HashSet<Worker> workers`，存储Worker的地方，访问被mainLock保护
4. `Condition termination`，关闭线程池时，会唤醒这个信号量
5. `int largestPoolSize`，线程池的线程数量的最大值（没什么用）
6. `long completedTaskCount`，已经完成的任务的数量
7. `ThreadFactory threadFactory`，创建线程的工厂
8. `RejectedExecutionHandler handler`，任务队列满时候的处理策略
9. `long keepAliveTime`，Worker最大空闲存活时间，
10. `boolean allowCoreThreadTimeOut`，控制核心线程在空闲时是否关闭
11. `int corePoolSize`，核心线程数量
12. `int maximumPoolSize`，最大线程数量（≥largestPoolSize）
13. `AtomicInteger ctl`，当前线程池的状态，以及线程数量，低29位是线程数量，高3位是状态
    RUNNING    = -1 << COUNT_BITS(29);
    SHUTDOWN   =  0 << COUNT_BITS;
    STOP       =  1 << COUNT_BITS;
    TIDYING    =  2 << COUNT_BITS;
    TERMINATED =  3 << COUNT_BITS;

# 主要方法
## 丢弃策略
1. Abort，抛出异常，RejectedExecutionException
2. Discard，直接被丢弃
3. DiscardOldest，丢弃提交最早的那个
4. CallerRun，提交者运行

## 线程池状态
主要过程是RUNNING --> SHUTDOWN --> STOP --> TIDYING --> TERMINATED
1. RUNNING -> SHUTDOWN
调用shutdown()
2. (RUNNING or SHUTDOWN) -> STOP
调用shutdownNow()
3. SHUTDOWN -> TIDYING
线程池和执行队列都空的时候
4. STOP -> TIDYING
当队列为空
5. TIDYING -> TERMINATED
执行完terminated()方法

## allowsCoreThreadTimeOut
这个方法是jdk**1.6**之后加的，可以让核心线程永久存活。

## shutdown，shutdownNow，tryTerminate，awaitTermination
1. shutdown: RUNNING --> SHUTDOWN, **中断所有空闲线程**，但是会等待所有已提交任务执行完成
2. shutdownNow: RUNNING --> STOP，**中断所有线程**，并且返回未执行的任务
3. tryTerminate: 尝试设置Terminate状态，
    如果暂时不需要Terminate的话，则尽快返回。
    如果所有的队列都已空，并且没有存活线程，则触发信号量。
4. shutdown, shutwdownNow，都不会等待线程结束的。**awaitTermination才是具体的等待操作**    
```java
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)
            // 主要就是中断线程
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}

public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 状态更改为SHUTDOWN
        advanceRunState(SHUTDOWN);
        // 中断所有空闲的Worker线程
        interruptIdleWorkers();
        onShutdown(); // 空方法
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}

public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 状态更改成STOP
        advanceRunState(STOP);
        // 中断所有Worker线程
        interruptWorkers();
        // 会把当前正在等待的任务都取出来
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}

// tryTerminate会在很多地方被调用
// 1. shutdown
// 2. shutdownNow
// 3. addWorkerFailed
// 4. worker执行过程中发生异常
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        // 如果正在运行
        // 不处于TIDYING状态
        // 处于SHUTDOWN，但是任务队列还有数据
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && !workQueue.isEmpty()))
            return;
        // 只有Worker当中正在执行任务，但是Worker队列中也不存在数据了
        // 这边采用链式触发的感觉，让Worker逐步一个一个退出
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 这边只有在状态为STOP的情况下才会进来，或者等到所有任务执行完后
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated();
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
    }
}

// 很简单的一个操作，等待终止信号量的完成
public boolean awaitTermination(long timeout, TimeUnit unit)
    throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();

    try {
        for (;;) {
            if (runStateAtLeast(ctl.get(), TERMINATED))
                return true;
            if (nanos <= 0)
                return false;
            nanos = termination.awaitNanos(nanos);
        }
    } finally {
        mainLock.unlock();
    }
}
```
## execute和submit
两者主要的区别是submit提交的任务都是包装过的，基于FutureTask的封装。
1. 如果核心线程池未满，则直接新增核心线程，并且执行当前任务
2. 如果工作队列未满，则添加进入工作队列，并且再次检查需要新增Worker（因为可能有Worker刚执行异常退出了）
3. 如果工作队列已满，则尝试新增一个Worker去执行当前任务
4. 如果在添加的过程当中线程池被关闭，则会尝试移除当前任务
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
        
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 工作队列未满
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 如果不在运行，则从队列中移除成功。
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 进入这边代表正在运行，工作队列未满，需要添加新的Worker
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 工作队列已满，则增加线程数量
    // 如果已经达到最大线程数量，则拒绝当前任务。
    else if (!addWorker(command, false))
        reject(command);
}

// 都是通过FutureTask去封装的
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}

public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```
## 构造函数解析
```java
public ThreadPoolExecutor(int corePoolSize,
                        int maximumPoolSize,
                        long keepAliveTime,
                        TimeUnit unit,
                        BlockingQueue<Runnable> workQueue,
                        ThreadFactory threadFactory,
                        RejectedExecutionHandler handler) { 
}
```
1. corePoolSize 
  核心线程数量，当任务队列未满时，线程数量一直为corePoolSize
2. maximumPoolSize
  最大线程数量，当任务队列已满的时候，线程数量会增加到maximumPoolSize
3. keepAliveTime, unit
  线程最大空闲等待时间，用于从任务队列中poll任务的参数
4. workQueue
  任务队列，任务都是提交到任务队列里面去的。
5. threadFactory
  线程工厂，用来创建线程的
6. handler
  任务队列已满，线程数量达到最大，任务提交失败的时候执行的handler

## Worker的高可用保障
1. 通过runWorker内部的trycatch和completedAbruptly来
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
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
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```
2. 通过ProcessWorkerExit函数来保证Worker不能被非正常的减少
```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate();

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```
3. 通过FutureTask内部的try-catch来保证
```java
public void run() {
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
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
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
```

## Worker
这个设计有些操蛋。
```java
private final class Worker extends AbstractQueuedSynchronizer
    implements Runnable {
    private static final long serialVersionUID = 6138294804551838833L;
    final Thread thread;
    Runnable firstTask;
    volatile long completedTasks;

    Worker(Runnable firstTask) {
        setState(-1); 
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() {
        runWorker(this);
    }

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}

final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 释放1个许可，此时当前线程线程可以被中断
    // @see Worker构造函数
    w.unlock();
    // 代表当前线程是否正常退出
    // true代表异常
    boolean completedAbruptly = true;

    try {
        while (task != null || (task = getTask()) != null) {
            // 这个就更操蛋了，单线程中自己对自己加锁。
            // 为什么这么做的原因是防止在执行过程中被其他的线程调用interruptIfStarted()
            w.lock();
            // 如果线程池正在推出，保证线程被中断
            // 否则，消除任何其他中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted())
                wt.interrupt();
            try {
                // 回调
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
                    // 回调
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}

private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 如果异常退出，会重新添加新的线程
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate();

    int c = ctl.get();
    // 没有停止
    if (runStateLessThan(c, STOP)) {
        // 正常退出
        if (!completedAbruptly) {
            // 这段代码又让人看不懂了
            // 主要意思是：
            // 如果设置了allowCoreTimeOut，则重新添加新的线程
            // 如果没有设置，但是队列还有任务，重新添加新线程
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

## addWorker
主要是根据不同的情况添加新的线程
1. 如果当前已经终止，任务队列为空，不添加，不为空添加
2. 如果当前未终止，但是任务队列为空
3. 添加时还需要注意是否刚更改Worker计数，然后整个线程池就被Shutdown或者STOP了，需要再次检测
```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 当前线程池已经在终止，并且没有需要执行的任务
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
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

// 线程数可能已满，无法再添加等等
// 或者添加过程中线程被终止
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        decrementWorkerCount();
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```