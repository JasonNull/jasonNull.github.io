---
title: ThreadLocal分析
date: 2018-06-23 18:10:59
tags:
- java
- 多线程
- 源码
categories: 
- java
- 多线程
toc: true
---
# 原理
Thread对象中有一个[ThreadLocalMap](/2018/06/23/Thread%E5%88%86%E6%9E%90/)对象，ThreadLocal对象被存储在线程的ThreadLocalMap中，叫做threadLocals。
<!-- more -->
# 主要方法
## get
```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

// 设置初始的Map
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

// 初始化Map，或者设置当前的初始值
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);

    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    
    return value;
}

public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程的threadLocals
    ThreadLocalMap map = getMap(t);

    // 如果当前map不为空
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 设置初始值
    return setInitialValue();
}
```
## set
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);

    if (map != null)
        // 直接设置map的初始值
        map.set(this, value);
    else
        createMap(t, value);
}
```
## ThreadLocalMap
```java
static class ThreadLocalMap {
    // 特别重要的一点，防止内存泄漏
    // WeakReference
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    private static final int INITIAL_CAPACITY = 16;
    private Entry[] table;
    private int size = 0;
    private int threshold;

    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }

    private static int nextIndex(int i, int len) {
        return ((i + 1 < len) ? i + 1 : 0);
    }

    private static int prevIndex(int i, int len) {
        return ((i - 1 >= 0) ? i - 1 : len - 1);
    }

    private Entry getEntry(ThreadLocal<?> key) {
        // 获取当前的key的存储地方
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];

        // 当前Entry的key存在，并且等于想要查找的值
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
    }

    // 在指定位置没有查找到，使用线性探查法
    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;

        while (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == key)
                return e;
            if (k == null)
                // 清空无效Entry
                expungeStaleEntry(i);
            else
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null;
    }

    // 将当前ThreadLocal对象放入到当前map中
    private void set(ThreadLocal<?> key, Object value) {
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);

        // 遍历当前不为空的entry，查找key是否存在
        for (Entry e = tab[i];
                e != null;
                e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();

            if (k == key) {
                e.value = value;
                return;
            }

            if (k == null) {
                // 直接使用当前Entry
                replaceStaleEntry(key, value, i);
                return;
            }
        }

        tab[i] = new Entry(key, value);
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }

    // 删除当前key
    private void remove(ThreadLocal<?> key) {
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);

        // 遍历当前Entry[]数组
        for (Entry e = tab[i];
                e != null;
                e = tab[i = nextIndex(i, len)]) {
            if (e.get() == key) {
                e.clear();
                expungeStaleEntry(i);
                return;
            }
        }
    }

    // 这段代码的主要意思我可以猜一下
    // 主要了为了保证在发生冲突的时候，Entry可以尽量离index近一些
    // 里面如果查找到的第一个需要回收的Entry不是当前Entry
    // 则会将两个交换值
    private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                    int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;
        Entry e;

        // 从当前Slot开始查找最早的可删除Entry
        int slotToExpunge = staleSlot;
        for (int i = prevIndex(staleSlot, len);
                (e = tab[i]) != null;
                i = prevIndex(i, len))
            if (e.get() == null)
                slotToExpunge = i;

        for (int i = nextIndex(staleSlot, len);
                (e = tab[i]) != null;
                i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();

            if (k == key) {
                e.value = value;

                // 交换了位置
                tab[i] = tab[staleSlot];
                tab[staleSlot] = e;

                if (slotToExpunge == staleSlot)
                    slotToExpunge = i;
                // 有一个清理操作
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                return;
            }
            // 如果
            if (k == null && slotToExpunge == staleSlot)
                slotToExpunge = i;
        }

        // for gc
        tab[staleSlot].value = null;
        tab[staleSlot] = new Entry(key, value);

        // 继续回收内存
        if (slotToExpunge != staleSlot)
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    }

    // 删除当前staleSlot的无效Entry，其中还涉及到一个交换过程
    // 如果当前Entry是占用了其他Entry的位置的话，会将其他Entry交换回来
    private int expungeStaleEntry(int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;

        // expunge entry at staleSlot
        tab[staleSlot].value = null;
        tab[staleSlot] = null;
        size--;

        Entry e;
        int i;
        for (i = nextIndex(staleSlot, len);
                (e = tab[i]) != null;
                i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null;
                tab[i] = null;
                size--;
            } else {
                int h = k.threadLocalHashCode & (len - 1);
                if (h != i) {
                    tab[i] = null;
                    while (tab[h] != null)
                        h = nextIndex(h, len);
                    tab[h] = e;
                }
            }
        }
        return i;
    }

    // 清除一些无效的Entry
    private boolean cleanSomeSlots(int i, int n) {
        boolean removed = false;
        Entry[] tab = table;
        int len = tab.length;
        do {
            i = nextIndex(i, len);
            Entry e = tab[i];
            if (e != null && e.get() == null) {
                n = len;
                removed = true;
                i = expungeStaleEntry(i);
            }
        } while ( (n >>>= 1) != 0);
        return removed;
    }
    
    // 执行整理Entry数组，以及扩容操作
    private void rehash() {
        expungeStaleEntries();

        // Use lower threshold for doubling to avoid hysteresis
        if (size >= threshold - threshold / 4)
            resize();
    }

    // 扩容
    private void resize() {
        Entry[] oldTab = table;
        int oldLen = oldTab.length;
        int newLen = oldLen * 2;
        Entry[] newTab = new Entry[newLen];
        int count = 0;

        for (int j = 0; j < oldLen; ++j) {
            Entry e = oldTab[j];
            if (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null; // Help the GC
                } else {
                    int h = k.threadLocalHashCode & (newLen - 1);
                    while (newTab[h] != null)
                        h = nextIndex(h, newLen);
                    newTab[h] = e;
                    count++;
                }
            }
        }

        setThreshold(newLen);
        size = count;
        table = newTab;
    }

    private void expungeStaleEntries() {
        Entry[] tab = table;
        int len = tab.length;
        for (int j = 0; j < len; j++) {
            Entry e = tab[j];
            if (e != null && e.get() == null)
                expungeStaleEntry(j);
        }
    }
}
```
# 内存泄漏
大概率发生在一个情况下：   
线程长时间存活，并且在使用过ThreadLocal之后，没有后续任何使用，并且也没有调用ThreadLocal.remove()方法，
导致value对象内存泄漏。