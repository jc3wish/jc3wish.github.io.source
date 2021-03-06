---
title: "Redis slots 槽的数量为什么是16384,而不是65536,也是不8192 ?"
date: 2019-11-06T23:05:25+08:00
draft: false
tags: ["Redis"]
categories: ["Redis"]
---

昨天同事突然问了一个关于 Redis Cluster 的槽是怎么存储的,为什么槽的数量 是 16384

我突然有点蒙住了,因为以前没有认真关注过 Redis 3.0 的具体实现,以前也只去看过 Redis2.8 的主从及持久化相关的下结代码

然后去研究了一下 Redis 3.0 的 slot 是咋回事


### 1. 为什么 slots 不选择用 65536


Redis 每个实例节点上保存对应有哪些 slots 是一个 unsigned char slots[REDIS_CLUSTER_SLOTS/8] 类型

```
typedef struct clusterNode {
    mstime_t ctime; /* Node object creation time. */
    char name[REDIS_CLUSTER_NAMELEN]; /* Node name, hex string, sha1-size */
    int flags;      /* REDIS_NODE_... */
    uint64_t configEpoch; /* Last configEpoch observed for this node */
    unsigned char slots[REDIS_CLUSTER_SLOTS/8]; /* slots handled by this node */
    int numslots;   /* Number of slots handled by this node */
    int numslaves;  /* Number of slave nodes, if this is a master */
    struct clusterNode **slaves; /* pointers to slave nodes */
    struct clusterNode *slaveof; /* pointer to the master node */
    mstime_t ping_sent;      /* Unix time we sent latest ping */
    mstime_t pong_received;  /* Unix time we received the pong */
    mstime_t fail_time;      /* Unix time when FAIL flag was set */
    mstime_t voted_time;     /* Last time we voted for a slave of this master */
    mstime_t repl_offset_time;  /* Unix time we received offset for this node */
    long long repl_offset;      /* Last known repl offset for this node. */
    char ip[REDIS_IP_STR_LEN];  /* Latest known IP address of this node */
    int port;                   /* Latest known port of this node */
    clusterLink *link;          /* TCP/IP link with this node */
    list *fail_reports;         /* List of nodes signaling this as failing */
} clusterNode;

```

假如 slots 数量 是 65536

65536 / 8(每个字节8bit) / 1024(1024个字节1kB) = 8kB   (这个公式计算是因为存储是 bit来表示的)

假如 slots 数量 是 16384

16384 / 8(每个字节8bit) / 1024(1024个字节1kB) = 2kB

换算了一下  16384 个 slots 的情况下比 65535 内存就要省 6 kB 左右(每个关例节点变量,假如一个集群有100个节点,那每个实例里就省了600kB)


重点是 Redis 集群节点之间每秒都会发送 PING 的心跳命令的时候,并传输 slofs 信息,规律如下:

(1)每秒会随机选取5个最久没有通信的节点 并且发送 PING 消息

(2)每100毫秒(1秒10次) 扫描本地节点列表，如果发现节点最近一次接受 PONG 消息的时间大于cluster-node-timeout/2 则立刻发送 PING 消息,cluster-node-timeout 默认值 15 秒


假如一个 Redis 集群有 200 个节点,不考滤其他压缩及其他数据等的情况下, 粗略算一下 每秒发送 PING 消息网络流量

采用 65536 个 slots

200 * 5 * 8KB = 8000KB, 那约等于8MB,换成交换机流量 小b 计算的话,就是 64Mb

采用 16384 个 slots

200 * 5 * 2KB = 2000KB, 那约等于2MB,换成交换机流量 小b 计算的话,就是 16Mb

这样算下来,每秒可以省 48Mb 的网络流量


### 2. 为什么 slots 不选择用  8192 呢?

通过第一个问题的算法,就是省内存空间,省网络开销,那省一点是一点

这个问题 搜索引擎 我也没搜索出来,好像没人关注过一样,大家都只在关注为什么不是65536.

那为什么不用 8192 呢?

16384 / 8(每个字节8bit) / 1024(1024个字节1kB) = 1kB

8192 通过上面的公式得出来 只要开销 1kB

那再来看看 Redis 把 Key 换算成所属 slots 的方法

```
/* -----------------------------------------------------------------------------
 * Key space handling
 * -------------------------------------------------------------------------- */

/* We have 16384 hash slots. The hash slot of a given key is obtained
 * as the least significant 14 bits of the crc16 of the key.
 *
 * However if the key contains the {...} pattern, only the part between
 * { and } is hashed. This may be useful in the future to force certain
 * keys to be in the same node (assuming no resharding is in progress). */
unsigned int keyHashSlot(char *key, int keylen) {
    int s, e; /* start-end indexes of { and } */

    for (s = 0; s < keylen; s++)
        if (key[s] == '{') break;

    /* No '{' ? Hash the whole key. This is the base case. */
    if (s == keylen) return crc16(key,keylen) & 0x3FFF;

    /* '{' found? Check if we have the corresponding '}'. */
    for (e = s+1; e < keylen; e++)
        if (key[e] == '}') break;

    /* No '}' or nothing betweeen {} ? Hash the whole key. */
    if (e == keylen || e == s+1) return crc16(key,keylen) & 0x3FFF;

    /* If we are here there is both a { and a } on its right. Hash
     * what is in the middle between { and }. */
    return crc16(key+s+1,e-s-1) & 0x3FFF;
}
```

Redis 将 key 换算成 slots 的方法 是将 crc16(key) 之后再和 slots 的数量 进行 与 计算

这里说明下,这里为什么是用  0x3FFF(16383) 来计算,而不是16384,是因为 在不产生溢出的情况下 x % (2^n) 等价于 x & (2^n - 1) 即  x % 16384 == x & 16383


这里为什么不用8192


crc16 出来结果,理论上出现重复的概率为  1/65536

但实际结果重复概率可能比这个大不少,就像crc32 结果 理论上 1/40亿 分之一,但实际有人测下来10万碰撞的概率就比较大了

假如 slots 设置成 8192, 200个实例的节点情况下,理论值是 每40个不同key请求,命中就会失效一次,假如节点数增加到400,那就是20个请求

只是理论值,实际上可能概率会更大

并且1kb 并不会比 2k 省太多,性价比不是特别高,所以可能 选 16384 会更为通用一点


#### 备注:

以上代码来源于 Redis3.0.1 版本

感谢 @皇家救星 技术讨论与技术支持

