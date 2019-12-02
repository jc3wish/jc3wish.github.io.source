---
title: "Redis启动概述"
date: 2019-12-02T22:55:26+08:00
draft: false
tags: ["Redis"]
categories: ["Redis"]
---

![image](/images/redis/initserver_1.jpg)

#### Redis 启动的时候 

./redis-server 0.0.0.0:6379 

做了如下事情:

##### 1. tcp 端口 和 unix socket 文件监控

##### 2. 默认情况下设置一个100ms的定时器

可以通过配置文件的 hz 配置来修改这个值,默认是10,即每秒触发多少次

这个定时器是往  epoll 中注册超时事件来实现的,并不是开启额外的线程来实现

定时器主要做了如下事情:

1). 连接检查

2). Signal信号捕捉

3). aof,rdb 数据持久化处理(比如是否开子进程，子进程结束判断等)

4). 每5秒统计一次redis使用情况，比如连接数，总键值数

5). 尝试resize每个db，resize是让每个db的dict结构进入rehash状态，rehash是为了扩容dict或者缩小dict

6). 每秒RDB是否持久化刷盘

7). 每秒AOF是否持久化刷盘

8). 从库每秒判断是否要重新连接master及发送心跳给master

9). 其他任务


##### 3. 从磁盘中恢复数据

假如 RDB 和 AOF 都同时开启了的情况下,只会加载 AOF 的数据

##### 4. 启动2个子线程

Redis 所谓的单线程是指处理请求等事件,是只有一个单线程处理,并不是指 Redis 整个进程只有一个线程

Redis 额外起了2个子线程

第一个子线程的工作是  " 调用flush从内核刷盘磁盘的线程 "

第一个子线程的工作是  " close句柄的线程 "

对, 没有错, Redis 开了 一个线程专门用于 flush 数据到磁盘, 也专门开了一个线程close句柄


##### 5. Redis eventLoop 死循环等待

在 Redis 所有初始化完之后,则进行事件循环处理上

![image](/images/redis/initserver_2.jpg)

![image](/images/redis/initserver_3.jpg)

![image](/images/redis/initserver_4.jpg)



