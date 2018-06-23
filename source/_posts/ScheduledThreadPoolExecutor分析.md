---
title: ScheduledThreadPoolExecutor分析
date: 2018-06-23 18:01:46
tags:
- java
- 多线程
- 源码
categories: 
- java
- 多线程
toc: true
---
类似ThreadPoolExecutor，但是加了三个方法，`schedule()`，`scheduleAtFixedRate()`和`scheduleWithFixedDelay()`。
<!-- more -->
# 主要方法
## DelayedWorkerQueue
```java
static class DelayedWorkQueue extends AbstractQueue<Runnable> implements BlockingQueue<Runnable> {
    private static final int INITIAL_CAPACITY = 16;
    private RunnableScheduledFuture<?>[] queue = new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
    // 内部一把锁，加锁
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition available = lock.newCondition();
    private int size = 0;
    // leader代表当前线程正在等待
    private Thread leader = null;

    // 这边解释一下helpIndex，用于代表当前任务在整个队列当中处于的具体位置
    private void setIndex(RunnableScheduledFuture<?> f, int idx) {
        if (f instanceof ScheduledFutureTask)
            ((ScheduledFutureTask)f).heapIndex = idx;
    }

    private void siftUp(int k, RunnableScheduledFuture<?> key) {
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            RunnableScheduledFuture<?> e = queue[parent];
            if (key.compareTo(e) >= 0)
                break;
            queue[k] = e;
            setIndex(e, k);
            k = parent;
        }
        queue[k] = key;
        setIndex(key, k);
    }

    private void siftDown(int k, RunnableScheduledFuture<?> key) {
        int half = size >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;
            RunnableScheduledFuture<?> c = queue[child];
            int right = child + 1;
            if (right < size && c.compareTo(queue[right]) > 0)
                c = queue[child = right];
            if (key.compareTo(c) <= 0)
                break;
            queue[k] = c;
            setIndex(c, k);
            k = child;
        }
        queue[k] = key;
        setIndex(key, k);
    }

    private void grow() {
        int oldCapacity = queue.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1); // grow 50%

        if (newCapacity < 0) // overflow
            newCapacity = Integer.MAX_VALUE;
        queue = Arrays.copyOf(queue, newCapacity);
    }

    // 删除
    public boolean remove(Object x) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int i = indexOf(x);
            if (i < 0)
                return false;

            setIndex(queue[i], -1);
            int s = --size;
            RunnableScheduledFuture<?> replacement = queue[s];
            queue[s] = null;
            if (s != i) {
                // 调整根堆
                siftDown(i, replacement);
                if (queue[i] == replacement)
                    siftUp(i, replacement);
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

    public int size() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return size;
        } finally {
            lock.unlock();
        }
    }

    public boolean isEmpty() {
        return size() == 0;
    }

    // 添加
    public boolean offer(Runnable x) {
        if (x == null)
            throw new NullPointerException();
        RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int i = size;
            // 尝试插入
            if (i >= queue.length)
                grow();
            size = i + 1;
            if (i == 0) {
                queue[0] = e;
                setIndex(e, 0);
            } else {
                // 调整小根堆
                siftUp(i, e);
            }
            // 队列头是e，代表数据刚插入，
            // 第一个线程在等待
            if (queue[0] == e) {
                leader = null;
                available.signal();
            }
        } finally {
            lock.unlock();
        }
        return true;
    }

    // 获取
    // 主要是几个点：
    // 1. 多个线程可能同时进入，但是第一个线程会设置leader，使其!=null，第一个线程在退出时候会将leader设置为null，并且会signal线程
    // 2. 其他线程则会无限期等待，每次插入数据的时候，会唤醒等待线程，去走1的逻辑
    public RunnableScheduledFuture<?> take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                RunnableScheduledFuture<?> first = queue[0];
                if (first == null)
                    // 等待
                    available.await();
                else {
                    // 获取需要等待的时间
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return finishPoll(first);
                    first = null; // don't retain ref while waiting
                    // 代表有线程在等待了
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            // 获取到了，并且队列中还有等待执行的数据
            if (leader == null && queue[0] != null)
                available.signal();
            lock.unlock();
        }
    }

    // 相比较take，poll在代码层面支持超时
    public RunnableScheduledFuture<?> poll(long timeout, TimeUnit unit)
            throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                RunnableScheduledFuture<?> first = queue[0];
                if (first == null) {
                    if (nanos <= 0)
                        return null;
                    else
                        nanos = available.awaitNanos(nanos);
                } else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return finishPoll(first);
                    if (nanos <= 0)
                        return null;
                    // 为了更好的gc，在等待的时候清楚了本地引用
                    first = null; // don't retain ref while waiting
                    if (nanos < delay || leader != null)
                        nanos = available.awaitNanos(nanos);
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        
                        try {
                            long timeLeft = available.awaitNanos(delay);
                            nanos -= delay - timeLeft;
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && queue[0] != null)
                available.signal();
            lock.unlock();
        }
    }

    public void clear() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            for (int i = 0; i < size; i++) {
                RunnableScheduledFuture<?> t = queue[i];
                if (t != null) {
                    queue[i] = null;
                    setIndex(t, -1);
                }
            }
            size = 0;
        } finally {
            lock.unlock();
        }
    }
}
```

## ScheduledFutureTask
```java
public void run() {
    boolean periodic = isPeriodic();
    // 不能运行
    if (!canRunInCurrentRunState(periodic))
        // 停止当前task，false代表不中断当前线程
        cancel(false);
    // 不是周期性执行
    else if (!periodic)
        ScheduledFutureTask.super.run();
    // 是周期性执行，并且执行结果正常
    // runAndReset只有在执行成功的时候才返回true
    else if (ScheduledFutureTask.super.runAndReset()) {
        // 计算下次执行时间
        setNextRunTime();
        // 再次进入DelayedWorkerQueue
        reExecutePeriodic(outerTask);
    }
}
```