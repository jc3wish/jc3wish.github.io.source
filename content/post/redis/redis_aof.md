---
title: "Redis AOF 持久化 实现原理"
date: 2019-12-02T23:38:26+08:00
draft: false
tags: ["Redis"]
categories: ["Redis"]
---

### 1. AOF 概念

From now on, every time Redis receives a command that changes the dataset (e.g. SET) 

it will append it to the AOF. 

When you restart Redis it will re-play the AOF to rebuild the state.


写入的命令以追加到文件的形式保存到文件中
重启的时候对命令进行重现

AOF 是解决 RDB 会丢失一部分数据则出来的持久久，每写一条命令，就往文件中追加一次

但这样有可能会就会造成一个文件非常的大，并且在重启的时候，对命令重现的时候，非常的久

所以 AOF 也有类似的镜像模式


### 2. AOF 配置

```
appendonly no      #是否开启AOF，默认no
appendfilename "appendonly.aof"     #aof文件

appendfsync   
#always 每条命令都刷盘
#everysec 每秒刷盘
#no 内核决定什么时候刷

no-appendfsync-on-rewrite no  #是否开启rdb类似方式rewrite镜像写
auto-aof-rewrite-percentage 100   #AOF文件是上一次重写的时候的1倍大小
auto-aof-rewrite-min-size 64mb    #并于AOF文件要大于64M

aof-load-truncated yes  #当数据恢复的时候，假如遇到数据损坏，是继续还是直接退出

aof-rewrite-incremental-fsync yes #当 aof rewrite 的时候开启32MB数据，刷一次盘
```

### 3. AOF 在实现过程中重要成员

#### 3.1 AOF 状态

```
/* AOF states */

#define REDIS_AOF_OFF 0             /* AOF 关闭 */

#define REDIS_AOF_ON 1              /* AOF 开启 */

#define REDIS_AOF_WAIT_REWRITE 2    /* 在线修改 为 appendonly yes 的时候 正在rewrite */
```

#### 3.2 重要变量

```
# 要刷到AOF文件的数据，string类型

server.aof_buf      

# rewrite 期间，执行的命令缓存 ，是一个链表

server.aof_rewrite_buf_blocks

#距上次rewrite之后 AOF 刷了多少数据，每次rewrite之后清空

server.aof_current_size

#上一次rewrite的时候，AOF数据大小

server.aof_rewrite_base_size
```

### 4. AOF rewrite 


假如 一直往一个文件中追加命令，那想想，那一个文件不是有可能无限大了？

再想想，假如一个key写一1000万次，那这key就得重现1000万次，那感觉是不是不爽？

所以作者也实现了  AOF rewrite，在一定情况下，将内存中的数据镜像的形式刷到文件，然后在那个时间点之后再进行 命令追加的形式保存

#### 4.1 AOF rewrite 开启一个子进程

和 RDB 开启一个子进程一样，以 copy on write 的技术开启一个子进程

![image](/images/redis/aof_child_process_code_1.png)

#### 4.2 AOF rewrite 刷盘过程

![image](/images/redis/aof_rewrite_process.png)

4.2.1 创建一个 temp-rewriteaof-%d.aof 文件，并打开

4.2.2 将 aof rewrite 镜像数据以二进制形式追加到文件 temp-rewriteaof-%d.aof（这个过程是到内核）

4.2.3 每 32MB 数据将数据从内核fsync方式刷到磁盘

4.2.4 关闭句柄

4.2.5 将 temp-rewriteaof-%d.aof 改名为 temp-rewriteaof-bg-$pid.aof

#### 4.3 AOF rewrite 子进程结束之后回调

在 aof rewrite 子进程结束之后，由<font color="#FOO">主进程</font>回调函数，根据  appendfsync 配置进行一次刷盘调用

![image](/images/redis/aof_child_process_code_2.png)

![image](/images/redis/aof_child_process_code_3.png)


1). 在rewrite期间新增的数据 追加到 temp-rewriteaof-bg-$pid.aof

![image](/images/redis/aof_child_process_code_4.png)

2). 将AOF文件名修改为 appendonly.aof 

3). aways 刷盘模式而主线程立马调aof_fsync 刷盘

4). everysec 刷盘模式而提交给另外一个线程异步刷盘

5). 提交给关闭句柄的线程，关闭 旧的AOF文件句柄


#### 4.4 AOF rewrite 触发

**4.4.1 定时触发**

```
  /* Trigger an AOF rewrite if needed */
         if (server.rdb_child_pid == -1 &&
             server.aof_child_pid == -1 &&
             server.aof_rewrite_perc &&
             server.aof_current_size > server.aof_rewrite_min_size)
         {
            long long base = server.aof_rewrite_base_size ?
                            server.aof_rewrite_base_size : 1;
            long long growth = (server.aof_current_size*100/base) - 100;
            if (growth >= server.aof_rewrite_perc) {
                redisLog(REDIS_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
                rewriteAppendOnlyFileBackground();
            }
         }
```

1). 默认100ms 检测一次

2). 没有rdb,aof rewrite 正在执行

3). 当前AOF数据大小 大于 设置的最小rewrite大小 ，默认64MB（auto-aof-rewrite-min-size参数）

4).  (server.aof_current_size*100/base) - 100 >= server.aof_rewrite_perc
    当前AOF数据大小 是否是上一次的多少倍，默认为 1 倍 （auto-aof-rewrite-percentage 参数）


**4.4.2 手工触发 - BGREWRITEAOF 命令**

```
void bgrewriteaofCommand(redisClient *c) {
    if (server.aof_child_pid != -1) {
        addReplyError(c,"Background append only file rewriting already in progress");
    } else if (server.rdb_child_pid != -1) {
        server.aof_rewrite_scheduled = 1;
        addReplyStatus(c,"Background append only file rewriting scheduled");
    } else if (rewriteAppendOnlyFileBackground() == REDIS_OK) {
        addReplyStatus(c,"Background append only file rewriting started");
    } else {
        addReply(c,shared.err);
    }
}
```

立马开启一个子进程进行rewrite

但命令结果返回给客户端之后不一定rewrite完了


**4.4.3 手工触发 - CONFIG appendonly yes**

```
  } else if (!strcasecmp(c->argv[2]->ptr,"appendonly")) {
        int enable = yesnotoi(o->ptr);

        if (enable == -1) goto badfmt;
        if (enable == 0 && server.aof_state != REDIS_AOF_OFF) {
            stopAppendOnly();
        } else if (enable && server.aof_state == REDIS_AOF_OFF) {
            if (startAppendOnly() == REDIS_ERR) {
                addReplyError(c,
                    "Unable to turn on AOF. Check server logs.");
                return;
            }
        }
    }
```

```
/* Called when the user switches from "appendonly no" to "appendonly yes"
 * at runtime using the CONFIG command. */
int startAppendOnly(void) {
    server.aof_last_fsync = server.unixtime;
    server.aof_fd = open(server.aof_filename,O_WRONLY|O_APPEND|O_CREAT,0644);
    redisAssert(server.aof_state == REDIS_AOF_OFF);
    if (server.aof_fd == -1) {
        redisLog(REDIS_WARNING,"Redis needs to enable the AOF but can't open the append only file: %s",strerror(errno));
        return REDIS_ERR;
    }
    if (rewriteAppendOnlyFileBackground() == REDIS_ERR) {
        close(server.aof_fd);
        redisLog(REDIS_WARNING,"Redis needs to enable the AOF but can't trigger a background AOF rewrite operation. Check the above logs for more info about the error.");
        return REDIS_ERR;
    }
    /* We correctly switched on AOF, now wait for the rerwite to be complete
     * in order to append data on disk. */
    server.aof_state = REDIS_AOF_WAIT_REWRITE;
    return REDIS_OK;
}
```

当原本是关闭的情况下

立马开一个子进程rewrite

<font color="#F00">假如rewrite失败了，每次定时任务都会尝试重试rewrite</font>


### 5. Redis 写命令

每条写命令在更新了内存之后，都会有一个buf 来保存每命令的数据，不管 appendfsync  配置的是什么参数，这个buf 都是 Redis 自已的，并不是刷到内核里

![image](/images/redis/aof_append_1.png)

1) . 只要开启了 AOF，就写到 server.aof_buf

2）. 假如开启了rewrite进程 期间，将数据额外追加一份到  server.aof_rewrite_buf_blocks。这份数据是用于在 AOF rewrite 进程退出之后，立马将这份数据刷到 aof 文件末尾的

### 6. appendfsync everysec

```
 /* AOF postponed flush: Try at every cron cycle if the slow fsync
     * completed. */
    if (server.aof_flush_postponed_start) flushAppendOnlyFile(0);

    /* AOF write errors: in this case we have a buffer to flush as well and
     * clear the AOF error in case of success to make the DB writable again,
     * however to try every second is enough in case of 'hz' is set to
     * an higher frequency. */
    run_with_period(1000) {
        if (server.aof_last_write_status == REDIS_ERR)
            flushAppendOnlyFile(0);
    }
```

1). 每秒触发一次调用刷盘函数

2). 每次调用函数刷盘，会判断上一次刷盘是否完成，

如果没有完成，则100ms后的定时任务继续调用


![image](/images/redis/aof_append_2.png)


3). 将数据从buf刷到内核

![image](/images/redis/aof_append_3.png)

4). 提交给另外一个线程 异步调用aof_fsync 将数据从内核刷到磁盘

### 7. appendfsync aways

每条命令都刷盘一次


![image](/images/redis/aof_append_4.png)

1). 每个事件处理之前 将server.aof_buf 里的数据刷到内核。记住并不是每条命令之后立马将数据落盘，而是等待下一个事件执行之前落盘


![image](/images/redis/aof_append_5.png)

![image](/images/redis/aof_append_6.png)

2). 主线程调用 aof_fsync 从内核刷到磁盘。这里是由主线程执行调用 fsync 刷到磁盘


### 8. appendfsync  no

1). 每个事件处理之前 将server.aof_buf 里的数据刷到内核

2). 将 buf 刷到内核之后，将由内核决定什么时候刷到磁盘，redis 进程不再接管flush这个事情


### 9. 小结


1). AOF 也会有rewrite刷盘,也是开子进程，cony on write

2). AOF 默认情况下是 当前AOF数据大小是上一次rewrite的倍的时候触发

3). AOF everysec 刷盘，是每秒从buf刷到内核后fsync刷到磁盘, 并不是每条命令就往内核里刷一次数据

4). AOF aways 刷盘，是写命令后的下一个事件执行之前由主线程执行刷盘的，并非写命令之后立马刷盘

