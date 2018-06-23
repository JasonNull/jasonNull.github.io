---
title: tcp
date: 2018-06-23 16:19:25
tags:
- java
- 基础
- tcp
- 网络
categories:
- java
- 基础
toc: true
---

#### tcp状态
![tcp状态](/images/tcp状态.gif)
<!-- more -->
1. CLOSE_WAIT: 被动关闭的这方，接收到FIN包之后就会进入Close_Wait状态，等待这边相关读写操作都完成(调用socket的close)，才会由CLOSE_WAIT切换到LAST_WAIT状态  
2. TIME_WAIT：主动关闭的这方会处于Time_Wait状态，为了防止最后一个ack丢失（被动方会重新发送FIN包），会等待大概4分钟（2msl，maximum segment lifetime）。server端应该减少关闭连接的次数。

#### tip
1. crawler上面就发现了这个问题
2. http-1.1使用Connection: keep alive来避免服务器端TIME_WAIT过多的情况
3. CLOSE_WAIT状态过多避免，被动关闭连接处理不当，没有调用close()方法，导致过多