---
title: Thread分析
date: 2018-06-23 18:08:21
tags:
- java
- 多线程
- 源码
categories: 
- java
- 多线程
toc: true
---
# 成员属性
1. `String name`, 线程名称
2. `int priority`, 优先级
3. `Thread threadQ`, 不知道
4. `long eetop`, 不知道
5. `boolean single_step`, 不知道 
6. `boolean daemon = false`, 是否是后台线程
7. `boolean stillborn = false`, 表示中间状态
8. `Runnable target`, 表示当前正在执行的指令
<!-- more -->
9. `ThreadGroup group`, 线程组
10. `ClassLoader contextClassLoader`, 用于加载类的
11. `AccessControlContext inheritedAccessControlContext`, 不知道
12. `ThreadLocal.ThreadLocalMap threadLocals`, 线程本地存储
13. `ThreadLocal.ThreadLocalMap inheritableThreadLocals`, 继承的线程本地存储
14. `long stackSize`, 栈的大小
15. `long nativeParkEventPointer`, 不知道
16. `long tid`, 线程的id
17. `int threadStatus`, 线程状态
18. `Object parkBlocker`, 表示当前线程正在等待的对象
19. `Interruptible blocker`, 不知道
20. `Object blockerLock`, 不知道

# 静态属性
1. `long threadSeqNumber`, 线程id的生成器

# 主要方法
## start
不能start两次，否则抛出IllegalThreadStateException

## interrupt, isInterrupted, interrupted
1. interrupt中断线程的阻塞，（其实是释放了linux信号量，并且设置了是否被中断标记位）
2. isInterrupted则是检测当前线程的中断标记位是否被设置，但是不清除标记位的设置
3. interrupted检测线程是否被中断，并且清除标记位的设置

## tip
1. `Thread.interrupt`只在Object.wait() .Thread.join(), Thread.sleep()等几个native方法上面抛出异常
2. `LockSupport.parkNanos`，只是会让它立即返回
3. 在AQS上面是主动监测了标记位，然后跑出了异常
4. java的Thread对象会对应一个系统级别的线程。