---
title: FutureTask分析
date: 2018-06-23 17:15:27
tags:
- java
- 多线程
- 源码
categories:
- java
- 多线程
toc: true
---
主要用于获取向线程池内提交的任务的执行状态以及执行结果，并且还可以操作任务的执行状态。
**[`awaitDone`](#get)的设计比较精妙，具体看[get](#get)**

# 成员属性
1. `Callable<V> callable`，需要执行的任务对象
2. `Object outcome`，任务的运行结果
3. `Thread runner`，运行任务的线程
4. `WaitNode waiters`，表示当前参与等待执行结果的队列（链表），无锁数据结构
5. `int state`，表示任务的状态
    - NEW，刚创建，但是还没开始执行
    - COMPLETING，正在执行过程当中
    - NORMAL，执行结束，没有异常
    - EXCEPTIONAL，执行结束，发生异常
    - CANCELLED，被取消
    - INTERRUPTING，任务正在被中断
    - INTERRUPTED，任务中断完成  
    **可能的状态转换：**
    1. NEW --> COMPLETETING --> NORMAL
    2. NEW --> COMPLETEING --> EXCEPTIONAL
    3. NEW --> CANCEL
    4. NEW --> INTERRUPTING --> INTERRUPTED
***
<!-- more -->

# 主要方法
## RunnableAdapter
主要用于将Runnable转换成Callable，比较重要
```java
class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;

    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }

    public T call() {
        task.run();
        return result;
    }
}
```
## run && runAndReset
两个方法的区别在于，一个用于普通线程池（只期望执行一次），一个用于定时任务线程池（会执行多次）。
代码层面上面唯一的区别**只有对于`set()`的调用**。
```java
// 设置异常状态
protected void setException(Throwable t) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = t;
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        finishCompletion();
    }
}
// 设置正常状态
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}

// 正在被中断的话，等待中断过程结束再返回
private void handlePossibleCancellationInterrupt(int s) {
    if (s == INTERRUPTING) {
        while (state == INTERRUPTING)
            Thread.yield(); 
}

// 运行任务
public void run() {
    // 查看状态，防止任务同时被启动两次
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                        null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            // 运行结果
            V result;
            // 是否成功运行
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            // 设置结果
            if (ran)
                set(result);
        }
    } finally {
        runner = null;
        int s = state;
        if (s >= INTERRUPTING)
            // 已经结束
            handlePossibleCancellationInterrupt(s);
    }
}

// 运行任务，并且清空所有属性值
protected boolean runAndReset() {
    // 这个就比较奇葩一些，不会更改任何相关状态
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                        null, Thread.currentThread()))
        return false;
    boolean ran = false;
    int s = state;
    try {
        Callable<V> c = callable;
        if (c != null && s == NEW) {
            try {
                c.call(); // don't set result
                ran = true;
            } catch (Throwable ex) {
                setException(ex);
            }
        }
    } finally {
        runner = null;
        s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
    return ran && s == NEW;
}
```
## get
等待任务执行完成，主要是分为以下两种状态
1. 任务未完成，加入等待链表，执行
2. 任务已完成（包含异常，以及正常），直接返回结果

```java
// 任务完成之后会调用这个函数，将所有线程从等待队列放出来
private void finishCompletion() {
    for (WaitNode q; (q = waiters) != null;) {
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

public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}

public V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
        throw new NullPointerException();
    int s = state;
    if (s <= COMPLETING &&
        (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
        throw new TimeoutException();
    return report(s);
}
```

一个比较精妙的地方是在于那么多else，一共三个步骤：
1. 新建等待队列节点
2. 加入到等待队列
3. 进行等待
这三个步骤之前都加入了是否已经完成的判断，最后才会等待，一共**自旋三次**，进行三次检测。
```java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    // 表示当前线程是否已经被加入到等待队列当中
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        // 如果不在运行
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        // 如果正在运行，自旋
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        // 如果是第一次尽力，添加到等待链表
        else if (q == null)
            q = new WaitNode();
        // 加入到等待队列中
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                    q.next = waiters, q);
        // 下面是等待操作
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

## cancel
```java
public boolean cancel(boolean mayInterruptIfRunning) {
    // 如果是new状态，则更改为Interrrupted或者Cancel状态
    if (!(state == NEW &&
            UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    
        // 进行中断当前线程
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();
    }
    return true;
}
```