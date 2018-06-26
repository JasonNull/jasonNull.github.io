---
title: AQS分析
date: 2018-06-24 20:24:52
tags:
- java
- 并发
- 源码
categories:
- java
- 并发
toc: true
---

# CLH队列
## 原理
<img src="/images/CLH.jpg" height="80%" width="60%">
<!-- more -->
## 示例
```java
public class CLHLock {
    @Data
    class Node {
        private volatile boolean locking;
        private volatile int lockedCount;

        Node() {
            lockedCount = 1;
        }
    }

    private AtomicReference<Node> tail;
    private ThreadLocal<Node> myself = ThreadLocal.withInitial(() -> new Node());

    public CLHLock() {
        tail = new AtomicReference<>();
    }

    public void lock() {
        Node current = myself.get();

        if (current.locking) {
            ++current.lockedCount;
            return;
        } else {
            current.locking = true;
        }

        Node previous = tail.getAndSet(current);
        while (previous != null && previous.locking) {
            Thread.yield();
        }
    }

    public void unlock() {
        Node current = myself.get();

        if (--current.lockedCount == 0) {
            current.locking = false;
            myself.remove();
        }
    }

    static volatile int count = 0;

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(1);
        int max = 1000000;
        CLHLock clhLock = new CLHLock();

        Stopwatch stopwatch = Stopwatch.createStarted();
        for (int i = 0; i < max; i++) {
            executorService.submit(() -> {
                clhLock.lock();
                clhLock.lock();

                if (count % 100 == 0) {
                    System.out.println(count);
                }
                ++count;

                clhLock.unlock();
                clhLock.unlock();
            });
        }

        while (count != max) ;
        executorService.shutdownNow();
        System.out.println("lock is right, count: " + count);
        System.out.println("used time: " + stopwatch.elapsed());
    }
}
```

# AQS
## 原理
1. 数据结构等待链表，使用的是尾插法
2. Lock等方法，都是不会主动抛出`InterruptedException`的，除非调用可中断版本

![AQS原理](/images/AQS.jpg)
## 静态属性
1. `long spinForTimeoutThreshold = 1000L`，超过此时间将会采用LockSupport.parkNanos()，而不是自旋

## 成员属性
1. `volatile Node head`，链表的头结点，傀儡节点（prev，thread属性都为空）
2. `volatile Node tail`，链表的尾结点，
3. `volatile int state`，状态变量，用于控制获取的

## 主要方法
### 可中断等待
1. `LockSupport.park()`会从`Thread.interrupt()`中被唤醒
2. 添加`Thread.interrupted()`检测

下面的代码展示了如何实现中断的。
```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
### Node
1. `int waitStatus`，当前节点的等待状态
    - SIGNAL，代表当前节点需要被唤醒
    - CANCELLED，代表当前节点已经被取消
    - CONDITION，代表当前节点在等待一个条件
    - PROPAGATE，代表当前节点正在被多个线程同时操作，并且是shared方式获取，需要向后传播
    - 0，代表当前节点正常，没有任何操作进行
2. `Node prev`，前一个节点
3. `Node next`，后一个节点
4. `Thread thread`，等待的线程
5. `Node nextWaiter`，下一个Condition链表中的节点，或者表示当前是否在独占模式等待

### hasQueuedPredecessors
**这个方法重中之重，用于判断公平性**
```java
// 用于判断等待链中的第一个等待线程是不是自己
public final boolean hasQueuedPredecessors() {
    Node t = tail; 
    Node h = head;
    Node s;
    return h != t 
        && ((s = h.next) == null 
             || s.thread != Thread.currentThread());
}
```
### tryAcquire, acquire
```java
// 摘自于ReentrantLock的FairSync
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();

    // 如果锁没有被加锁
    if (c == 0) {
        // 不存在前驱节点，并且成功获取了许可
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            // 将当前线程设置为拥有者
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果当前线程是拥有者
    else if (current == getExclusiveOwnerThread()) {
        // 尝试更新拥有许可计数
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

// 检查
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 查看上一个节点状态
    int ws = pred.waitStatus;
    // 上一个节点是SIGNAL
    if (ws == Node.SIGNAL)
        return true;
    // 寻找到上个在等待的节点，或者是已经获取锁的节点
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 更改上个节点状态，便于后续的判断
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

// 进入阻塞，等待被唤醒
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}

// 获取许可
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 当前线程是第一个等待线程，并且获取成功
            if (p == head && tryAcquire(arg)) {
                // 重新设置头结点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                // 返回是否被中断
                return interrupted;
            }
            // 需要等待（上个有效节点不是头结点)
            // 并且尝试等待
            if (shouldParkAfterFailedAcquire(p, node) 
                    && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        // 如果异常终止，则取消当前获取
        if (failed)
            cancelAcquire(node);
    }
}

public final void acquire(int arg) {
    // 尝试获取，不成功则进入阻塞获取阶段
    if (!tryAcquire(arg)
            && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
    }
}

// 唤醒链表中最前面的等待线程
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}

// 取消获取，并且设置状态
private void cancelAcquire(Node node) {
    if (node == null)
        return;

    node.thread = null;

    // 跳过被取消的节点
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    Node predNext = pred.next;
    // 当前节点状态被设置为取消
    node.waitStatus = Node.CANCELLED;

    // 如果当前节点是尾结点
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        // 如果前面的节点不是头节点
        // 并且在等待唤醒（SIGNAL，PROPAGATE，CONDITION），线程不为空
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL 
                || (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) 
                && pred.thread != null) {
            Node next = node.next;
            // 从链表中删除当前节点
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            // 直接唤醒待取消节点的下一个节点
            unparkSuccessor(node);
        }
        // 将当前节点脱链
        node.next = node;
    }
}
```
### tryRelease, release
```java
// ReentrantLock的释放
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    // 可重入锁的计数，计数为0，代表，已经释放所有许可
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
public final boolean release(int arg) {
    // 释放成功
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            // 唤醒下一个等待线程
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
### tryAcquireShared, acquireShared
```java
// Semaphore的获取
protected int tryAcquireShared(int arg) {
    for (; ; ) {
        int available = getState();
        int left = available - arg;

        // 许可不足，或者获取成功
        if (left < 0 || compareAndSetState(available, left)) {
            return left;
        }
    }
}
// 共享方式获取许可
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        // 尝试阻塞获取
        doAcquireShared(arg);
}

private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);

    // 保证传播性，如果下个节点也在等待
    if (propagate > 0 || h == null || h.waitStatus < 0 
            || (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // 下个节点是共享节点，唤醒下一个节点去获取许可
        if (s == null || s.isShared())
            doReleaseShared();
    }
}

// 要点是获取许可成功之后，还会唤醒下一个等待线程去尝试获取
private void doAcquireShared(int arg) {
    // 把当前线程添加到等待链当中
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 前面没有任何线程等待s
            if (p == head) {
                // 尝试再次获取
                int r = tryAcquireShared(arg);
                // 获取成功
                if (r >= 0) {
                    // 更新头结点，并且尝试唤醒下一个节点
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 阻塞自己
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
### tryReleaseShared, releaseShared
```java
@Override
// Semaphore的尝试释放
protected boolean tryReleaseShared(int arg) {
    // 释放，死循环，知道陈宫
    for (; ; ) {
        int available = getState();
        int left = available + arg;

        if (left < 0) {
            throw new Error("maximum limits get");
        } else if (compareAndSetState(available, left)) {
            return true;
        }
    }
}
// 
public final boolean releaseShared(int arg) {
    // 尝试释放
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

// 唤醒下一个等待线程
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            // 头结点
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                // 更新头结点状态为0
                // 这个自旋是为了保证释放的节点是需要被唤醒的节点
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;
                // 唤醒下一个节点，也是为了保证传播性
                unparkSuccessor(h);
            }
            // 多个线程同时进行释放许可操作，继续
            else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        // 如果释放成功一个节点，
        if (h == head) 
            break;
    }
}
```
### Condition
如上面的图表示的，Condition内部含有一个链表下面是await和signal的简介。
```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 加入Condition等待链表
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 当前在等待Condition
    while (!isOnSyncQueue(node)) {
        // 当前线程被调用signal唤醒
        LockSupport.park(this);
        // 当前线程被中断
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 当前线程没被中断的话，尝试以独占模式获取许可 
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 尝试删除被取消的节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    // 抛出异常
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

 public final void signal() {
     // 检测是否独占模式获取
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    // 唤醒链表中的第一个节点（如果存在的话）
    if (first != null)
        doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
                (first = firstWaiter) != null);
}

// 将当前节点从Condition中脱链，加入到AQS的链表中（复用了一个节点）
final boolean transferForSignal(Node node) {
    // 节点已经被取消
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    // 将当前节点从Condition的等待队列转移到AQS的等待队列
    Node p = enq(node);
    int ws = p.waitStatus;
    // 将节点状态，改成SIGNAL
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        // 唤醒节点的等待线程
        LockSupport.unpark(node.thread);
    return true;
}
```
## 调用关系
acquire先调用tryAcquire，**不成功**，调用doAcquire()阻塞等待
release先调用tryRelease，**成功了**，调用doRelease()释放下一个等待线程

***
# ReentrantLock
内部一个Sync类，继承了AQS用于实现独占锁的功能，主要是实现了tryAcquire和tryRelease功能
## 公平性
由`hasQueuedPredecessors()`来保证的，如果存在前置节点，则返回尝试加锁失败
## Sync
```java
private static final long serialVersionUID = -5179523762034025860L;
    abstract void lock();

    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 还未加锁
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 当前线程是拥有者线程，此次是重入加锁
        else if (current == getExclusiveOwnerThread()) {
            // 多次加锁
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        // 用于保证多次释放
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

    final ConditionObject newCondition() {
        return new ConditionObject();
    }
}

static final class NonfairSync extends Sync {
    final void lock() {
        // 非公评锁，直接加锁（被唤醒的线程还未得到时间片运行，等运行时会进行再次书面）
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 公平性重点!!!!!!!!
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 重复加锁
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

***
# Semaphore
基本和ReentrantLock一样，主要实现了tryAcquiredShared()和tryReleaseShared()。
## 公平性
由`hasQueueProdecessors()`来决定的，如果存在前置节点就失败
## 缺陷
由于队列是FIFO的结构，导致了如果首元结点获取失败，则不会向后传播，下面是例子
```java
// 第一个线程启动后，尝试获取11个许可，但是由于获取失败，一直会阻塞。
// 第二个启动后，由于公平性原因，获取直接失败，导致直接入队，此时如果队列头部一直获取失败的话，队列是不能有任何元素出队的。
Semaphore semaphore = new Semaphore(10, true);
new Thread(() -> { semaphore.acquire(11); semaphore.release(11); }).start();
new Thread(() -> { semaphore.acquire(1), semaphore.release(1); }).start();
```
## Sync
```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    final int nonfairTryAcquireShared(int acquires) {
        // 尝试去获取，不处理有线程等待的情况
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 
                    || compareAndSetState(available, remaining))
                return remaining;
        }
    }

    protected final boolean tryReleaseShared(int releases) {
        // 利用死循环去释放，知道释放成功
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))
                return true;
        }
    }
}

static final class NonfairSync extends Sync {
    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}

static final class FairSync extends Sync {
    protected int tryAcquireShared(int acquires) {
        for (;;) {
            // 公平性重点
            // 已经有线程在等待，直接失败
            if (hasQueuedPredecessors())
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```

***
# CountDownLatch
不可以**被重置**，并不代表技术上做不到，只是CountDownLatch就是用来用于一次性任务的，支持重置的话，可能会导致多组获取紊乱。
## Sync
内部使用state作为初始计数，每次获取则计数减一，到获取完后直接是释放所有线程
```java
private static final class Sync extends AbstractQueuedSynchronizer {
    protected int tryAcquireShared(int acquires) {
        // 这招太损了
        // 
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

***
# CyclicBarrier
## 思考：为什么不基于AQS实现
没有办法单纯的只基于AQS就实现，**需要添加控制变量**。

## 成员属性
1. `final ReentrantLock lock = new ReentrantLock();`，用于保护await操作
2. `final Condition trip = lock.newCondition();`用于等待被唤醒
3. `final int parties;`最多支持的线程数量
4. `final Runnable barrierCommand;`barrier被打破时执行的命令
5. `Generation generation = new Generation();`代表当前barrier的状态
6. `int count;`**没有volatile**，为什么不用volatile，count只在doAwait()方法内部访问，而doAwait是全程加锁的，也就是代表count的访问永远都是单线程的访问。

## 主要方法
### doAwait
```java
// 处理几种异常
// 1. 超时
// 2. 被中断
// 3. 执行command时异常
private int dowait(boolean timed, long nanos) throws InterruptedException, BrokenBarrierException, TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        final Generation g = generation;
        // 如果屏障已经解除，这个照理来说不可能发生的
        // 除非某个线程被中断了，并且唤醒了所有等待线程
        if (g.broken)
            throw new BrokenBarrierException();
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
        int index = --count;
        // index为0，代表需要解除屏障
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                // 运行命令
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                // 将所有线程唤醒，并且更新generation，但是不更新count
                nextGeneration();
                return 0;
            } finally {
                // 运行command过程出错
                if (!ranAction)
                    breakBarrier();
            }
        }
        for (;;) {
            // 超时等待，并且处理被中断的情况
            try {
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                // 返回当前线程的等待序号
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```

### reset
```java
// 构建新的屏障
private void nextGeneration() {
    trip.signalAll();
    count = parties;
    generation = new Generation();
}

// 清除当前屏障
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}

public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        breakBarrier();
        nextGeneration();
    } finally {
        lock.unlock();
    }
}
```


# ReadWriteLock
内部原理是state状态变量的低16位代表写锁数量，高16位代表读锁数量。
`ReadLock.lock()`调用`tryAcquireShared`
`WriteLock.lock()`调用`tryAcquire`
#### 写锁降级
原理是先获取读锁（直接获取会成功），释放写锁（同时虽然会唤醒头等待线程，但是没有用，获取不到）
#### 读锁升级
直接加写锁呗