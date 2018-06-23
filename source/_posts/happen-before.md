---
title: happen-before
date: 2018-06-23 15:54:35
tags: 
- jvm
- jmm
categories:
- java
- jvm
toc: true
---
1. 相互之间有依赖的代码，前面的代码先执行，后面的代码再执行
2. 锁定规则，lock操作发生在unlock操作之前
3. volatile变量的读取发生在写入操作之后（storeLoad和storeStore屏障）
4. A先发生于B，B先发生于C，则A先发生于C
5. 线程的start方法发生于后续的状态控制方法之前，所有的非stop()方法都发生在stop()方法之前
6. 线程的interrupt方法发生于检测到中断之前
7. 线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行
8. 一个对象的初始化早于finalize之前