---
title: Queue分析
date: 2018-06-23 18:38:47
tags:
- java
- 集合
- 源码
categories:
- java
- 集合
toc: true
---
# 接口
## queue
- add，无法添加时候抛出异常
- offer，无法添加时候返回false
- poll，队列为空或者超时返回空
- remove，删除某个元素，如果元素不存在，则抛出异常

## blocking queue
- put，无法添加则等待可以添加
- take，队列为空，则等待到可取出元素为止
- offer，超时等待版本
- poll，超时等待版本
<!-- more -->

# PriorityQueue
## 数据结构
小根堆
## 适用场景
本质上是一个队列，但是是一个带有优先级的队列。元素插入的时候会进行比较，保证队列内的元素都是有序的。
比如线程调度的时候，需要根据线程优先级去调度的。
## 静态属性
1. MAX_ARRAY_SIZE, Integer.MAX_VALUE - 8，**不是**队列的最大容量，最大仍为Integer.MAX_VALUE 

## 属性
1. `int size = 0;`队列当前容量
2. `Object[] queue;`队列存储的数组
3. `final Comparator<? super E> comparator;`比较优先级使用的Comparator
4. `int modCount = 0;`非幂等操作的计数

## 特点
1. 使用Comparator来做优先级比较，是小根堆
2. 是用了大量的`Arrays.copyOf`来进行扩容

## 主要方法
### grow
```java
 private void grow(int minCapacity) {
    int oldCapacity = queue.length;
    // Double size if small; else grow by 50%
    // 队列比较小的情况下，2倍增长，比较大则1.5倍
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                        (oldCapacity + 2) :
                                        (oldCapacity >> 1));
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    queue = Arrays.copyOf(queue, newCapacity);
}
```
### add
```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        // 扩容
        grow(i + 1);
    size = i + 1;
    if (i == 0)
        queue[0] = e;
    else
        // 堆调整
        siftUp(i, e);
    return true;
}

private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}

@SuppressWarnings("unchecked")
private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        // 父节点
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        // 比父节点大，则break
        if (key.compareTo((E) e) >= 0)
            break;
        // 比父节点小，则交换值
        queue[k] = e;
        k = parent;
    }
    queue[k] = key;
}

@SuppressWarnings("unchecked")
private void siftUpUsingComparator(int k, E x) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        if (comparator.compare(x, (E) e) >= 0)
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = x;
}
```
### poll
```java
 public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    E result = (E) queue[0];
    E x = (E) queue[s];
    queue[s] = null;
    if (s != 0)
        // 调整堆
        siftDown(0, x);
    return result;
}

private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}

@SuppressWarnings("unchecked")
private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>)x;
    int half = size >>> 1;        // loop while a non-leaf
    while (k < half) {
        int child = (k << 1) + 1; // assume left child is least
        Object c = queue[child];
        int right = child + 1;
        // 选出左右叶子节点中比较大的一个
        if (right < size &&
            ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        // 叶子节点都比自己大
        if (key.compareTo((E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    queue[k] = key;
}

@SuppressWarnings("unchecked")
private void siftDownUsingComparator(int k, E x) {
    int half = size >>> 1;
    while (k < half) {
        int child = (k << 1) + 1;
        Object c = queue[child];
        int right = child + 1;
        if (right < size &&
            comparator.compare((E) c, (E) queue[right]) > 0)
            c = queue[child = right];
        if (comparator.compare(x, (E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    queue[k] = x;
}
```
****
# ArrayDeque
双端队列，底层是数组
## 特点
1. 默认长度是16，扩容每次扩两倍
2. 拥有head和tail两个指针，用于对队列做循环操作
3. 用了很多Arrays.copyOf和System.arraycopy

***
# ArrayBlockingQueue
阻塞队列
## 成员属性
1. `Object[] items`，存储元素的数组
2. `int takeIndex`，队列的头部
3. `int putIndex`，队列的尾部
4. `int count`，队列中元素的数量
5. `ReentrantLock lock`，互斥锁，用于锁住非幂等操作使用
6. `Condition notEmpty`，用于唤醒等待取数据的线程
7. `Itrs itrs`，用于统一存储一些迭代器，便于控制迭代的准确性

## 特点
1. 为了线程安全，性能比较差，内部使用了ReentrantLock
2. 容量是固定的，无法使用扩容

## 主要方法
### put
```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 等待信号量，使用while为了保证不是signalAll触发的
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    // 唤醒信号量
    notEmpty.signal();
}
```
### poll
```java
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return x;
}
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}
```
### Itrs
```java
// 这边把这个类的方法放出来
// 从下面的方法可以看出，Itrs中保存了ArrayBlockingQueue对象中的所有迭代器
// 内部是一个链表，用于保存所有的Iterator
// 每当queue中做一些非幂等操作的时候，都会通知迭代器做响应的改变
// 比如当`ArrayBlockingQueue.clear()`被调用的时候，
// 都会顺带调用一下itrs.queueIsEmpty方法，把所有的保存的迭代器关闭并清空
class Itrs {
    private class Node extends WeakReference<Itr> {
        Node next;
        Node(Itr iterator, Node next) {
            super(iterator);
            this.next = next;
        }
    }
    int cycles = 0;
    private Node head;
    private Node sweeper = null;
    private static final int SHORT_SWEEP_PROBES = 4;
    private static final int LONG_SWEEP_PROBES = 16;

    Itrs(Itr initial) {
        register(initial);
    }

    void doSomeSweeping(boolean tryHarder) {    }

    void register(Itr itr) {
        head = new Node(itr, head);
    }
    void takeIndexWrapped() {    }
    void removedAt(int removedIndex) {    }
    void queueIsEmpty() {    }
    void elementDequeued() {    }
}
```
***
# LinkedBlockingQueue
和ArrayBlockinqQueue的区别就是底层是链表而已，**容量固定**
> 稍微一些不同，LinkedBlockingQueue用了**两把锁，putLock和takeLock**，用于增加吞吐量。

***
# PriorityBlockingQueue
和ArrayBlockinqQueue的区别就是底层是小根堆而已，**容量不固定的**，有餐构造函数传入的是初始大小

*** 
# SynchronousQueue
阻塞队列的一种，但是类似于生产者者消费者，放入的元素一定要被其他线程取走才能接触阻塞。

## 数组结构
底层使用了无锁链表（通过cas来构建链表）实现了栈和队列，用来实现公平与非公平。

## 成员属性
1.  transferer，Transferer对象，主要是实现无锁链表的。

## 适用场景
适合非常极端的场景。一对一的生产者消费者模式。

## 重要方法
### transfer
```java
static final class SNode {
    volatile SNode next;        // next node in stack
    volatile SNode match;       // the node matched to this
    volatile Thread waiter;     // to control park/unpark
    Object item;                // data; or null for REQUESTs
    int mode;           // Note: item and mode 
}
// 下面介绍的是栈
// 链表内的节点有三种状态，request，data，fullfilling和canceled
// request和data可以与fullfilling进行叠加
// 下面是队列的一个例子
// request->request|canceled->request
// 加入一个新的data节点，
// 发现第一个是request节点，正好匹配，就尝试使用cas算法，将头结点移除，并且把数据给设置到match域中，并且再将睡眠的线程唤醒
// 发现第一个节点是data节点，操作同上
// 发现节点是一个request或者data节点，并且被取消了，因为时间到了，则直接将节点移除
E transfer(E e, boolean timed, long nanos) {
    SNode s = null; // constructed/reused as needed
    int mode = (e == null) ? REQUEST : DATA;

    // 这边是个死循环
    for (;;) {
        SNode h = head;
        // 链表为空，或者头结点的模式和自己是一样的，证明头结点也在等待
        if (h == null || h.mode == mode) {  // empty or same-mode
            // 已经超时了 
            if (timed && nanos <= 0) {      // can't wait
                // 将cancel掉的节点删除
                if (h != null && h.isCancelled())
                    casHead(h, h.next);     // pop cancelled node
                else
                    return null;
            // 将自己加入到链表中等待
            } else if (casHead(h, s = snode(s, e, h, mode))) {
                // 等待匹配
                SNode m = awaitFulfill(s, timed, nanos);
                // 证明超时
                if (m == s) {               // wait was cancelled
                    clean(s);
                    return null;
                }
                // 直接将两个节点都删除
                if ((h = head) != null && h.next == s)
                    casHead(h, s.next);     // help s's fulfiller
                return (E) ((mode == REQUEST) ? m.item : s.item);
            }
        // 当前头结点不在被匹配
        } else if (!isFulfilling(h.mode)) { // try to fulfill
            // 当前头结点已被删除
            if (h.isCancelled())            // already cancelled
                casHead(h, h.next);         // pop and retry
            else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                for (;;) { // loop until matched or waiters disappear
                    SNode m = s.next;       // m is s's match
                    if (m == null) {        // all waiters are gone
                        casHead(s, null);   // pop fulfill node
                        s = null;           // use new node next time
                        break;              // restart main loop
                    }
                    SNode mn = m.next;
                    if (m.tryMatch(s)) {
                        casHead(s, mn);     // pop both s and m
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    } else                  // lost match
                        s.casNext(m, mn);   // help unlink
                }
            }
        } else {                            // help a fulfiller
            // 帮助匹配
            SNode m = h.next;               // m is h's match
            if (m == null)                  // waiter is gone
                casHead(h, null);           // pop fulfilling node
            else {
                SNode mn = m.next;
                if (m.tryMatch(h))          // help match
                    casHead(h, mn);         // pop both h and m
                else                        // lost match
                    h.casNext(m, mn);       // help unlink
            }
        }
    }
}

SNode awaitFulfill(SNode s, boolean timed, long nanos) {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    int spins = (shouldSpin(s) ?
                    (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    // 这边使用自旋
    for (;;) {
        if (w.isInterrupted())
            s.tryCancel();
        SNode m = s.match;
        if (m != null)
            return m;
        if (timed) {
            nanos = deadline - System.nanoTime();
            // 时间过期
            if (nanos <= 0L) {
                s.tryCancel();
                continue;
            }
        }
        if (spins > 0)
            spins = shouldSpin(s) ? (spins-1) : 0;
        else if (s.waiter == null)
            s.waiter = w; // establish waiter so can park next iter
        // 如果没有超时的话，一直等待
        else if (!timed)
            LockSupport.park(this);
        // 超时等待
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}
```