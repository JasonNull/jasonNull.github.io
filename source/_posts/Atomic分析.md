---
title: Atomic分析
date: 2018-06-23 18:25:39
tags:
- java
- 并发
- 源码
categories:
- java
- 并发
toc: true
---
# AtomicInteger
粗暴一些，直接分析源码了
<!-- more -->
以下几个方法着重体现了，AtomicInteger主要是调用Unsafe实现的，具体请看[Unsafe](/2018/06/23/Unsafe分析)
```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            // 获取value属性的偏移量，通过Unsafe直接去操作
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

    public final void lazySet(int newValue) {
        // 深度思考，这边为什么使用count = newValue
        // 因为"count = newValue"会插入storeLoad屏障
        // 而putOrderedInt插入的是storeStore屏障
        unsafe.putOrderedInt(this, valueOffset, newValue);
    }

    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }

    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }

    public final int decrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
    }

    public final int addAndGet(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
    }
}
```
# AtomicFloat
实现可以使用`Float.floatToIntBits()`函数。