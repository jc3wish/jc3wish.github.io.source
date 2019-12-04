---
title: "JAVA JVM 堆内存 GC 及 内存分配"
date: 2019-12-04T23:06:26+08:00
draft: false
tags: ["JAVA","JVM"]
categories: ["JAVA"]
---

JAVA 中 分堆内存 和 堆外内存

堆外内存不是由JVM控制的，这个得开发人员自己手工调用方法去释放

堆内内存是由JVM控制，可以由JVM 通过算法自动 GC

JAVA 在 JVM 启动的时候，就事先对堆内存进行了划分了几个模块

**<font color="#F00">默认</font>情况下 JVM 堆内存分配：**

<table>
    <tr>
        <th colspan="4">新生代 (占堆内存 1/3)</th>
        <th rowspan="3">老年代 (占堆内存 2/3)</th>
    </tr >
    <tr >
        <td>Eden</td>
        <td>Survivor0</td>
        <td>Survivor1</td>
        </td></td>
    </tr>
    <tr>
        <td>80%</td>
        <td>10%</td>
        <td>10%</td>
        </td></td>
    </tr>
</table>

JVM 将堆内存划分了2个大模块

新生代： 当我们使用了一个新对象，申请内存的时候，就是先从新生代内存里申请

老年代： 当新生代的内存经过了15次 GC 还没被 GC 掉 或者 当新申请的内存比较大的时候，直接从老年代里分配


### 新生代 GC

新生代里又分划分了 Eden , Survivor0, Survivor1 三个区域

翻译过来的 就是 伊甸园 , 存活0区 , 存活1区

伊甸园是《圣经》故事中人类的始祖亚当和夏娃居住的乐园。所以作者也很意思。

实际 Eden , Survivor0, Survivor1 三个区域职责如下：

**Eden** ： 当申请内存的时候，从 Eden 里申请(当申请的内存大于JVM启动参数 PretenureSizeThreshold 的值的时候，直接从 老年代里分配内存，默认值为0,所有都从Eden分配), 当 Eden 里不够内存的时候，会进行一次普通GC

**Survivor** : 当 Eden 内存不够的时候,普通GC, 将 Eden 里没被引用等的内存GC掉，把活下来的，转到 Survivor 区域

这里为什么不写是  Survivor0 或者 Survivor1 

因为 Survivor0 和 Survivor1 两个区域干的活是一样， Survivor0 和 Survivor1 相交替干活的

可以理解为，每次GC，只有 其中一个可用。

例如,当前 Survivor0 和 Survivor1 都会空

#### **第一次新生代GC**

Eden 里GC存活来的内存，转移到 Survivor0, 并且给那些活下来的对象 “寿命”+1

#### **第二次新生代GC**

将Eden 和  <font color="#F00">**Survivor0**</font>两个区域活下来的内存转移到 Survivor1

然后再将 Eden 和 <font color="#F00">**Survivor0**</font> 两个都区域的全清空

并且给那些活下来的对象 “寿命”+1

#### **第三次新生代GC**

将Eden 和  <font color="#F00">**Survivor1**</font> 两个区域活下来的内存转移到 Survivor0

然后再将 Eden 和 <font color="#F00">**Survivor1**</font> 两个都区域的全清空

并且给那些活下来的对象 “寿命”+1


#### **第 N 次新生代GC**

将 “寿命” >= 15 的对象，转移到 老年代。

假如 Eden 和 Survivor 区域活下来的对象需要的空间 大于 Survivor0 的空间。则找出哪一个 寿命及以上的 总合大于  Survivor0 的空间，则将那个年龄及对上的 对象转移到 老年代（其实就是按年龄倒序排序，直接那个 Survivor 中其中一个 区域够存为止）

每次新生代GC 都是只用 Survivor 其中一个区域，把另一个空出来，待下一次的时候，这就是大家常说的 复制GC

但这样可以理解为至少有一个 Survivor 区域内存是没有真正存数据的

### 老年代 GC

新生代 GC 后 Survivor0或者Survivor1 不够存 新生代 GC 之后活下来的对象 的时候

会新生代 将 “寿命” >= 15 的对象 或者 Survivor0 不够存按年龄大小倒序排出来的对象 转移到 老年代

但是老年代的空间，也是有限的，默认情况下占当前 JVM 堆空间的 2/3

#### 老年代普通GC

当老年代里的 <font color="#F00">连续内存空间</font> 大于 新生代转移过来的内存块的时候，则会进行一次 老年代普通 GC

#### Full　GC

当老年代里的 <font color="#F00">连续内存空间</font> 小于 新生代转移过来的内存块的时候，则会进行 Full GC

Full GC 是对整个堆空间进行GC

如果Full GC后还是无法给新创建的对象分配内存，或者无法移动那些需要进入老年代中的对象，那么JVM抛出 Out Of Memory Error


### 小结

1. JVM 是中整个堆当内存不够的时候，会进行 Full GC , 当Full GC 内存还是不够的时候，会抛出 Out Of Memory Error

2. JVM 新生代，老年代内存分配比例是可以通过启动参数控制的

3. Map ,Set , ArrayList 等对象中存存储的数据，需要开发人员手动 remove 掉，要不然GC 可能不会释放（除非WeakHaspMap等对象），很容易让堆占满，导致Full GC