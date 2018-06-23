---
title: synchronized
date: 2018-06-23 15:59:24
tags:
- java
- jvm
- 关键字
categories:
- java
- jvm
toc: true
---
# 原理
1. 每个对象其实并不是一把监视器锁，这是一个概念上的误区，只不过是用起来像监视器锁
2. 主要是monitorEnter，monitorExit实现的
3. monitorEnter是查看自己能否获取当前对象，根据对象MarkWord来的，如果不能获取，则把自己加入到monitor record链表中，并且park当前线程
4. monitorExit和AQS类似，释放对象头当中的标记位置，然后唤醒monitor record中的第一个线程
5. monitorExit不会插入内存屏障

# 详解
对象头里面包好MarkWord，如下图：
![MarkWord](/images/MarkWord.png)

<!-- more -->

# 加锁
只有**第一个线程**可以成功以偏向锁模式加锁。
## 偏向锁加锁
1. 发现锁标志位为00，并且偏向标记为1，并且线程ID为0，即可进行偏向加锁。
2. 尝试CAS操作对象头，更新为线程ID即可，并且更新进入偏向模式，即成功获取偏向锁。

## 偏向锁唤醒
1. 发现还是偏向模式，不需要做任何操作。
2. 非偏向模式，并且存在线程在等待，则唤醒MarkWord中的等待线程。

## 轻量级锁加锁
1. 发现已经偏向锁模式开启，并且不偏向自己，退出偏向模式。
2. 已经加锁，那就直接进入重量级锁加锁。
3. 未加锁，进入轻量级锁加锁模式，将MarkWord拷贝到当前线程栈中作为LockRecord
4. CAS更新MarkWord为指向LockRecord

## 轻量级锁解锁
1. 如果MarkWord还保存当前线程的LockRecord指针，则CAS替换为原来的LockRecord
2. 否则，唤醒当前加锁线程

## 重量解锁加锁
1. 当前MarkWord是轻量级加锁并且自旋一段时间之后无法获取轻量级锁，则进入重量级锁加锁
2. 将MarkWord保存到栈，进入重量级锁等待，更新MarkWord为指向重量级锁的指针

## 重量级锁级解锁
1. 直接唤醒重量级锁对象。

## tip
1. 偏向锁的存在是为了消除锁以及CAS操作  
2. 轻量级锁的存在是为了消除重量级锁，并且加入了适应性自旋  
3. 重量级锁我个人则感觉和`LockSupport.park()`一样的效果