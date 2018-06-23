---
title: io
date: 2018-06-23 16:19:21
tags:
- java
- 基础
- io
categories:
- java
- 基础
toc: true
---
#### nio:
new io，同步非阻塞，底层使用linux的epoll，用于实现一个线程可以同时监听多个channel，基于时间驱动的思想
用于连接数多，但时间比较短的地方

#### bio：
阻塞io，同步并阻塞，jdk1.4之前常用，性能较差，一个线程只能操作一个端口，

#### aio
异步io，异步非阻塞，和bio的区别是数据的操作是异步完成的，当前线程不需要等待，用于连接数目比较多，时间比较长的地方

io中有两个模式reactor模式：
netty用nio实现