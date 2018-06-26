---
title: Redis笔记
date: 2018-06-24 15:53:41
tags:
- 中间件
- redis
categories:
- 中间件
toc: true
---
### 数据类型
字符串，hash，list，set，sort set

#### 跳表
优点：
1. 实现更加简单，查找复杂度类似
2. 插入性能更高
<!-- more -->
当然也会坏处，占用内存更多，多了更多无用的节点
![跳跃表](/images/skip-list.jpg)

#### 字符串存储结构
SDS，simple dynamic string，简单动态字符串

#### 过期策略
1. 定时器删除
2. 惰性删除
3. 定期扫描删除
对key设置过期的时候，是将key加入到了过期字典当中。
redis采取了定期扫描删除和惰性删除结合的策略。