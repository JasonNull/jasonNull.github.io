---
title: gc介绍
date: 2018-06-22 19:36:16
tags:
- jvm
- gc
- g1
- cms
categories: 
- java
- jvm
toc: true
---

## GC分类
### partial gc
1. minor gc，回收年轻代
2. major gc，回收老年代
3. mixed gc，回收年轻代和一部分老年代，只有G1有

### full gc
回收整个java堆的gc

### full gc的触发的原因：
1. 由于young gc前的预估发现老年代存放不下新生代晋升的对象（PromotionFailure）
2. 创建大对象的时候，没有连续的内存存储，老年代不足以存放
3. 显示的调用了System.GC，并且配合上了jvm参数

**一般来说，Full GC == Major GC，因为发生major gc的时候一般都会先minor gc，但又不完全等价，Full GC需要回收年轻代和老年代，major gc不会**
<!-- more -->

## safe point
GC的时候，所有线程都需要进入safe point才可以进行gc，而safe point是由jit加入的
### 具体加入地点
1. 所有方法返回之前
2. 非计数循环的回跳之前
3. native方法可以视为自动拥有safe point，native代码的执行与jvm无关

## 对象存活判断
名称|含义|优点|缺点  
:-:|:-:|:-:|:-:
引用计数|给每个对象加上一个属性，值为其他对象指向当前对象的次数。<br/>如果该属性值为0，就代表对象需要被回收。 |实现简单，性能高| 对象直接循环引用，会造成内存泄漏。
可达性分析|从一组GC根对象出发，对堆内所有对象进行遍历，标记那些可以被到达的对象|准确度比较高，不会误判|耗时比较长，并且分析的过程中需要暂停程序的运行

## 根对象
gc根对象包含以下：
1. java方法栈内的对象
2. 本地方法栈内的对象
3. 方法区内类静态属性引用的对象
4. 方法区内常量池引用的对象

## 回收算法
名称|定义|优点|缺点  
:-:|:-:|:-:|:-:
标记复制（mark copy）|内存空间分为两块等大的区域，from和to区域。当from区域用完，先对需要回收的对象做标记，然后将未被标记的对象复制到另外to区域，最后清理from区域内存|速度比较快，只需要拷贝一遍对象就行了|占用内存比较大
标记删除（mark sweep）|内存空间是连续的一整块。当内存用完的时候，标记需要回收的对象，然后进行回收。|较为快速，内存空间的利用率是比较高的|内存碎片出现比较大
标记整理（mark compact）|在标记删除的最后，加上整理内存的步骤。|内存的连续性比较好，空间可以得到充分的利用。|时间比较长，整个步骤被分为，标记和复制两个步骤
分代收集|针对于95%以上的对象存活时间比较短，将内存划分成两块区域，新生代和老年代。针对不同的代使用不同的算法|回收速度，内存连续性，内存使用率上面达到了一个平衡|暂无

## 垃圾收集器
### 其他gc
名称|回收过程|参数|算法种类|优点|缺点
:-:|:-:|:-:|:-:|:-:|:-:
Serial/Serial Old|暂停所有用户线程，开始标记新生代和老年代，然后进行回收。|-XX:+UseSerialGC|**新生代**标记复制，老年代**标记整理**|简单而高效|大内存回收耗时比较长
ParNew|Serial收集器的多线程版本|-XX:+UseParNewGC，-XX:ParallelGCThreads|**新生代**收集器，标记复制|CMS的默认新生代收集器，多线程，效率较高|无
Parallel Scavenge|stw，并行，吞吐量优先|-XX:+UseParallelGC，-XX:MaxGCPauseMillis，-XX:GCTimeRatio，-XX:+UseAdptiveSizePolicy|**新生代**并行收集器，标记复制|多线程，吞吐量优先|回收不一定保证能完全回收
Parallel Old|同上|-XX:+UseParallelOldGC|老年代并行收集器，**标记整理**|同上|占用内存比较多

tip: 
1. 所有的新生代收集器，标记整理
2. SerialOld，ParallelOld，标记整理算法（只有这俩能够进行full gc）
3. cms，标记删除
4. g1，整体的标记复制

### CMS
concurrent mark sweep
#### 介绍
老年代并发收集器，注重缩短收集时的用户线程的停顿时间
#### 参数
-XX:+UseConcMarkSweepGC，-XX:CMSFullGCsBeforeCompaction，-XX:+CMSParallelRemarkEnabled，
-XX+UseCMSCompactAtFullCollection-XX:+UseCMSInitiatingOccupancyOnly，-XX:CMSInitiatingOccupancyFraction=70，
-XX:CMSInitiatingPermOccupancyFraction，-XX:+CMSIncrementalMode，-XX:+CMSClassUnloadingEnabled
#### 步骤
1. 初始标记，stw，标记一下Gc Roots能关联到的对象
2. 并发标记，在和用户线程并发运行的过程中，使用初始标记的结果作为集合去标记对象
3. 重新标记，stw，修正并发标记的误差，因为有些对象可能没有被标记出来，时间比初始标记长
4. 并发清除，回收所有垃圾对象

#### 算法
标记清除，会产生内存碎片，但是可以打开标记整理。
#### 触发条件
老年代使用率大于CMSInitiatingOccupancyFraction
#### Concurrent Mode Failure
- 由于整个时间操作都是比较长的，并且在CMS运行过程中,，预留空间不能够满足使用（大小不够或者连续容量不够）
- 此时在并发阶段出现ConcurrentModeFailure的情况，CMS会启用**Serial Old**作为backup，配合ParNew去执行FullGC
- cms会在老年代占用比例超过CMSInitiatingOccupancyFraction的时候进行回收老年代，发生major gc
- major gc的重新标记步骤，**可能会触发一次新生代gc**（根据参数,CMSMaxAbortablePrecleanTime），触发了并且这次新生代gc成功，就会进行重新标记
- 触发了，但是执行不成功，会转为由Serial Old来进行老年代的回收（完全的stw）
#### 优点
CMS的优点是，在老年代是使用到一定的比例就会开始进行老年代的回收，并且回收发生的绝大部分时间之内程序都是可以运行的。

### G1
#### 特性
1. 更高的回收效率
2. 更少的内存碎片
3. 可预测的停顿模型

#### 算法
标记复制算法，小区域的复制，但是大范围上看起来是标记整理，不会产生内存碎片。
#### 介绍
将整个堆内存划分成最大2048个区域，年轻代和老年代可能会物理相邻。(region的大小从1MB开始，为2的倍数，最大32MB)。   
在内存使用率大于InitiatingHeapOccupancyPercent开始进行初始标记，剩余内存小于G1ReservePercent时，进行回收。  
整个回收的过程可能是young gc，或者是mixed gc。
mixed gc回收多少老年代则主要取决于预测模型的决定。  
预测模型会根据MaxGCPauseMillis来进行决定怎么回收，具体对哪一块进行回收（回收的效益会达到最高）。  
Humongous对象会直接进入H区，H区的扫描比较简单的。  
G1的young gc，mixed gc都会stw。  

#### tip 
1. 回收过程中发生concurrent mode failure（晋升失败）或者 发生promotion failure，会启用serial old来stw做full gc
2. mixed gc时候，通常需要也先触发一次young gc，这时候可以复用根扫描这个操作，并且可以大幅度减少需要扫描的对象

#### 回收过程
**young gc:**
1. 根扫描， 静态和本地对象被扫描
2. 更新RS，处理dirty card队列更新RS
3. 处理RS，检测从年轻代指向年老代的对象
4. 对象拷贝，拷贝存活的对象到survivor/old区域
5. 处理引用队列，软引用，弱引用，虚引用处理  
如图：![young区域回收](/images/g1内存示意.png)

**mixed gc:**
1. 初始标记（会复用young gc的标记）
2. 并发标记
3. 再次标记
4. 清除垃圾
5. 拷贝

**tip**
- RS中保存了`新生代的哪些对象被老年代引用了`
- 在新生代扫描的过程当中，根扫描是一个比较耗时的工作，引入了remember set，用来记录哪些对象被老年代引用了
- 主要还是为了避免扫描整个堆（老年代扫描比较慢）

#### 参数
-XX:UseG1GC,-XX:+UseG1GC,-XX:MaxGCPauseMillis=400,-XX:InitiatingHeapOccupancyPercent=40
-XX:SurvivorRatio=1,-XX:MaxTenuringThreshold=15,-XX:G1ReservePercent=15
-XX:+UnlockExperimentalVMOptions,-XX:G1NewSizePercent=40,-XX:G1MaxNewSizePercent=80

Parallel收集器和G1收集器不是在传统的hotspot垃圾收集器框架里面开发的，所以不能配合和其他收集器使用。