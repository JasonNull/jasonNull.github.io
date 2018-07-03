---
title: DTS功能构思
date: 2018-06-26 19:03:37
tags:
- 中间件
- 开源
categories:
- 中间件
---

## 介绍
是将要做的一个开源项目，全称是distributed task scheduler(分布式任务调度器)。
原型是[xxl-job](https://github.com/xuxueli/xxl-job)。

## 功能点
以下是需要做的修改点。
1. 高性能
    - 合并对单个客户端的分发（较少网络IO消耗），考虑是否采用客户端pull的方式进行任务的消费
    - 多机联合分发（充分利用多机，单个任务的分发能够支持由多个机器联合）
    - 任务状态上报优化，减少上报次数
    - jetty短连接优化（netty）
2. 高可用
    - BASE原理，任务状态
    - 任务调度失败补偿
    - 任务丢失补偿
    - 子任务触发补偿
    - 子任务状态上报
3. 重构
    - 调度层抽象（以后可能提供多种调度引擎，quartz）
    - 通信层抽象（可能不使用netty，使用tomcat，或者netty）
    - 包结构优化（分为server，client，core，starter，example）
    - 代码内容优化，类结构更合理
4. 数据库优化
    - 分库分表sharding-jdbc，分布式关系型数据库tiDB
    - 表结构优化，减少冗余数据
5. 功能
    - 失败处理策略，执行失败告警，执行失败重试
    - 支持尝试停止任务（可以能通过自定义线程池以及`Thread.stop()`实现）
    - 日志页面优化（JobLog和TaskLog查询）

## 高可用
1. quartz高可用
    - requireRecovery
    - misfireFireAndProceeding
2. BASE
    - 任务状态(未调度，调度成功，调度失败，执行中，等待中，完成)
    - 状态上报
    - 状态监测