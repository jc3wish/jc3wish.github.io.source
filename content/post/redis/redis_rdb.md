---
title: "Redis RDB 持久化 实现原理"
date: 2019-12-02T23:38:26+08:00
draft: false
tags: ["Redis"]
categories: ["Redis"]
---

### 1. RDB 概念

**官方解析**

By default Redis saves snapshots of the dataset on disk, in a binary file called dump.rdb. 
You can configure Redis to have it save the dataset every N seconds 
if there are at least M changes in the dataset, 
or you can manually call the SAVE or BGSAVE commands.


**本人解析**

将内存中数据，以镜像的方式存储到 dump.rdb 二进制文件中

通过配置 N 秒刷了多少条数据进行触发

也可以通过 SAVE 或者 BGSAVE 命令进行触发


### 2. RDB 配置

```
save 900 1      #900 秒内如果至少有 1 个 key 的值变化，则保存
save 300 10     #300 秒内如果至少有 10 个 key 的值变化，则保存
save 60 10000  #60 秒内如果至少有 10000 个 key 的值变化，则保存

stop-writes-on-bgsave-error yes  
#当启用了RDB且最后一次后台保存数据失败，Redis是否停止接收数据

rdbcompression yes  #是否进行压缩存储
rdbchecksum yes  #使用CRC64算法来进行数据校验
dbfilename dump.rdb #快照的文件名

dir ./  #快照文件的存放路径
```


### 3. 定时检查是否刷开子进程刷盘


![image](/images/redis/rdb_1.png)

#### 1). 每次写命令执行的时候 变量 server.dirty ++ 

#### 2). 默认情况下每100ms 触发一次定时器，检查是否满足配置条件

save 900 1      
#900 秒内如果至少有 1 个 key 的值变化，则保存
save 300 10     
#300 秒内如果至少有 10 个 key 的值变化，则保存
save 60 10000  
#60 秒内如果至少有 10000 个 key 的值变化，则保存

#### 3). 开启一个子进程将数据镜像刷到磁盘

开启的子进程是采用 copy on write 的技术， 在有写命令的时候，才会复制2份

### 4. RDB 刷盘过程

![image](/images/redis/rdb_2.png)

#### 1). 创建一个 tmp-$pid.rdb 的临时文件

#### 2). 按 RDB 存储协议 追加数据到  tmp-$pid.rdb 文件中

RDB 文件 最开头 5 个字节 是存储的是 REDIS 5个字母

再接下来的5个字节是RDB版本号，例如：0006

接着就按 Redis 数据库顺序根据不同 存储的类型刷数据了

不过什么类型，过期时间在前面，然后是就是数据类型，再接着就是key值，和 val值

当所有 db 都遍历完之后，接着写一下 255 这个数字，占1个字节 

最后刷一个 cksum 值

#### 3). 当所有数据追加到  tmp-$pid.rdb 完之后，对句柄进行close

#### 4). 将 tmp-$pid.rdb 文件重命名为 dump.rdb

这里为什么要先写一个 tmp临时文件，写完之后再修改 dump.rdb 文件呢？

假如直接写 dump.rdb文件，假如中途写失败了呢？那文件不就损坏了么？！


### 5. RDB 刷盘失败

什么时候会刷盘失败？

A. 当上一次rdb刷盘还没有完成的时候，不会再开新的子进程刷盘
B. 开启子进程失败，子进程被kill 或者运行过程 异常退出

不管什么原因造成子进程退出 ，在子进程结束回调的时候都会修改server.lastbgsave_status=REDIS_ERR

### 6. stop-writes-on-bgsave-error

在 开启了  stop-writes-on-bgsave-error yes 

并且 上一次 RDB 刷盘之无，刷盘失败了，则会阻塞 写和PING 等命令

```
/* Don't accept write commands if there are problems persisting on disk
     * and if this is a master instance. */
    if (((server.stop_writes_on_bgsave_err &&
          server.saveparamslen > 0 &&
          server.lastbgsave_status == REDIS_ERR) ||
          server.aof_last_write_status == REDIS_ERR) &&
        server.masterhost == NULL &&
        (c->cmd->flags & REDIS_CMD_WRITE ||
         c->cmd->proc == pingCommand))
    {
        flagTransaction(c);
        if (server.aof_last_write_status == REDIS_OK)
            addReply(c, shared.bgsaveerr);
        else
            addReplySds(c,
                sdscatprintf(sdsempty(),
                "-MISCONF Errors writing to the AOF file: %s\r\n",
                strerror(server.aof_last_write_errno)));
        return REDIS_OK;
    }
```


### 7. 手动触发 RDB 镜像生成

#### 7.1 SAVE 命令

```
void saveCommand(redisClient *c) {
    if (server.rdb_child_pid != -1) {
        addReplyError(c,"Background save already in progress");
        return;
    }
    if (rdbSave(server.rdb_filename) == REDIS_OK) {
        addReply(c,shared.ok);
    } else {
        addReply(c,shared.err);
    }
}
```

SAVE 命令，是阻塞式的

是由当前主线程进行刷盘，而不是开子进程刷盘

刷完之后再返回给客户端


#### 7.2 BIGSAVE 命令

```
void bgsaveCommand(redisClient *c) {
    if (server.rdb_child_pid != -1) {
        addReplyError(c,"Background save already in progress");
    } else if (server.aof_child_pid != -1) {
        addReplyError(c,"Can't BGSAVE while AOF log rewriting is in progress");
    } else if (rdbSaveBackground(server.rdb_filename) == REDIS_OK) {
        addReplyStatus(c,"Background saving started");
    } else {
        addReply(c,shared.err);
    }
}
```

BIGSAVE 命令，是立马开启一下子进程，后台进行刷盘

立马返回信息给你客户端


