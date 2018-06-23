---
title: List分析
date: 2018-06-23 18:38:29
tags:
- java
- 集合
- 源码
categories:
- java
- 集合
toc: true
---
# ArrayList
## 静态属性
1. DEFAULT_CAPACITY，默认大小，10
2. EMPTY_ELEMENTDATA，初始数组，空
3. DEFAULTCAPACITY_EMPTY_ELEMENTDATA，默认大小时使用的数组，空
4. MAX_ARRAY_SIZE，elementData最大大小，这边有歧义，数组最大大小还是Integer.MAX_VALUE

## 成员属性
1. elementData，存储元素的数组
2. size，大小
3. modCount，操作次数（在AbstractList里面），每次进行非幂等操作的时候，都会增加这个值
<!-- more -->

## 特点
1. 大小增长是1.5倍，默认大小时10
2. indexOf中使用了equals
3. 内部大量使用**Array.copyOf()** [创建一个全新数组], System.arraycopy() [数组之间元素的复制]
4. clone方法内部使用了Arrays.copyOf()
5. add(int index, E element)中用了System.arraycopy进行数组之间的复制
6. clear()只是清除了引用，但是没有回收数组空间
7. batchRemove()，里面有一个经典算法（如何在O(n)的时间范围内移除移除A数组中包含B数组的所有元素）
8. modCount在ListIterator和Iterator中用于保证failfast（ConcurrentModificationException)
9. ArrayListSpliterator用于并行分割计算
10. List有两种迭代器，Iterator和ListIterator（可以前后迭代）
11. SubList返回的不是一个全新的List，而是对原有List的一个引用（坑点）
12. modCount是带有局限性的

## 适用场景
适用于可动态扩容的数组

## 主要方法
### grow(扩容)
```java
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    // 1.5倍扩容
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
### contains
用==或者equals做比较
```java
// 采用equals做比较
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```
### add
```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```
### subList
很多人认为subList()返回的是一个全新的list，但**不是**。
```java
private class SubList extends AbstractList<E> implements RandomAccess {
    private final AbstractList<E> parent;
    private final int parentOffset;
    private final int offset;
    int size;

    SubList(AbstractList<E> parent,
            int offset, int fromIndex, int toIndex) {
        this.parent = parent;
        this.parentOffset = fromIndex;
        this.offset = offset + fromIndex;
        this.size = toIndex - fromIndex;
        this.modCount = ArrayList.this.modCount;
    }

    // 会修改原有的List
    public E set(int index, E e) {
        rangeCheck(index);
        checkForComodification();
        E oldValue = ArrayList.this.elementData(offset + index);
        ArrayList.this.elementData[offset + index] = e;
        return oldValue;
    }
}
```
### spliterator
可以使用此对list做切割，然后并行的对每个部分做操作
```java
static final class ArrayListSpliterator<E> implements Spliterator<E> {
    private final ArrayList<E> list;
    // fence
    private int index; // current index, modified on advance/split
    private int fence; // -1 until used; then one past last index
    private int expectedModCount; // initialized when fence set

    ArrayListSpliterator(ArrayList<E> list, int origin, int fence,
                            int expectedModCount) {
        this.list = list; // OK if null unless traversed
        this.index = origin;
        this.fence = fence;
        this.expectedModCount = expectedModCount;
    }

    private int getFence() { // initialize fence to size on first use
        int hi; // (a specialized variant appears in method forEach)
        ArrayList<E> lst;
        if ((hi = fence) < 0) {
            if ((lst = list) == null)
                hi = fence = 0;
            else {
                expectedModCount = lst.modCount;
                hi = fence = lst.size;
            }
        }
        return hi;
    }

    public ArrayListSpliterator<E> trySplit() {
        // 将现有的范围[index, fence)，分为
        // [index, (index + fence)/2) 和 [(index + fence)/2,fence]
        int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
        return (lo >= mid) ? null : // divide range in half unless too small
            new ArrayListSpliterator<E>(list, lo, index = mid,
                                        expectedModCount);
    }
}
```
### equals和hashCode
equals需要逐个元素的比较，hashCode会对所有的元素做hashCode

---
# LinkedList
不做详细分析，只写特点。
## 特点
1. 底层数据结构是链表
2. 插入删除复杂度为O(1)，查找复杂度为O(n)
3. 适用于那种插入删除频繁的地方，可用于构建可变长队列，栈

---
# CopyOnWriteArrayList
## 静态属性
无
## 成员属性
1. array
Object数组，用于存储元素
2. lock
可重入锁，用于控制非幂等操作的并发安全性问题。
## 特点
1. 初始长度为0，每次做非幂等操作（写入，删除）的时候都会创建一个全新的数组
2. 写入时候复制，即使每次长度只增加1（这样有一个好处是遍历的时候不会出现问题）
3. 内部大量使用Arrays.copyOf()[整段复制]和System.arrayCopy()[分段复制，native函数]
## 使用场景
特别适合那种写入少，读取量大的情况。
缺点是写入特别耗时间。
## 主要方法
### get
```java
private E get(Object[] a, int index) {
    return (E) a[index];
}
// getArray返回的是当前array
public E get(int index) {
    return get(getArray(), index);
}
```
### add
```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 复制整个数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
### remove
```java
private boolean remove(Object o, Object[] snapshot, int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;
        // snapshot不等于current，代表当前index可能已经过期了，需要再重新寻找
        // 至于为什么这么做，主要是为了减少加锁时间，能够提升并发操作的性能
        if (snapshot != current) findIndex: {
            int prefix = Math.min(index, len);
            for (int i = 0; i < prefix; i++) {
                if (current[i] != snapshot[i] && eq(o, current[i])) {
                    index = i;
                    break findIndex;
                }
            }
            if (index >= len)
                return false;
            if (current[index] == o)
                break findIndex;
            index = indexOf(o, current, index, len);
            if (index < 0)
                return false;
        }
        Object[] newElements = new Object[len - 1];
        System.arraycopy(current, 0, newElements, 0, index);
        System.arraycopy(current, index + 1,
                            newElements, index,
                            len - index - 1);
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

### toArray
```java
// 返回一个全新的数组
public Object[] toArray() {
    Object[] elements = getArray();
    return Arrays.copyOf(elements, elements.length);
}
// 原数组长度不够，创建新数组，返回新数组的信用
public <T> T[] toArray(T a[]) {
    Object[] elements = getArray();
    int len = elements.length;
    if (a.length < len)
        return (T[]) Arrays.copyOf(elements, len, a.getClass());
    else {
        System.arraycopy(elements, 0, a, 0, len);
        if (a.length > len)
            a[len] = null;
        return a;
    }
}
```
### equals和hashCode
同ArrayList

---
# Vector
jdk1.2推出的，使用synchronized加锁保证线程安全性

# Stack
继承于Vector，也是线程安全的，实现了栈而已。

# ListIterator
和Iterator的主要区别是多了previous，hasPrevious，nextIndex，set和add方法