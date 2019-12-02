---
title: "Redis RDB 存储协议"
date: 2019-12-02T23:38:26+08:00
draft: false
tags: ["Redis"]
categories: ["Redis"]
---

 


内容       |       字节数      |       案例        |      备注
---|---|---|---
RDB版本号  |       9          |       REDIS0006   |
数据库类型  |       1          |       254         | 254 代表后面跟着的存储是数据第几个数据库，比如，0,1-16
第几个数据库 |     1-5         |        0          | 参数Redis   RDB数据不定长数据存储协议假如那一个数据库没有数据，则不会存储
数据类型是否过期key | 1         |       252         | 假如key是没有配置expair 的，则没有这个
key过期时间 |       8          |    1574393426366   |
val 数据类型 |      1           |                   |    #define REDIS_RDB_TYPE_STRING 0<br/>#define REDIS_RDB_TYPE_LIST   1<br/>#define REDIS_RDB_TYPE_SET    2#<br/>define REDIS_RDB_TYPE_ZSET   3<br/>#define REDIS_RDB_TYPE_HASH   4<br/>/* Object types for encoded objects. */<br/>#define REDIS_RDB_TYPE_HASH_ZIPMAP    9#<br/>define REDIS_RDB_TYPE_LIST_ZIPLIST  10<br/>#define REDIS_RDB_TYPE_SET_INTSET    11<br/>#define REDIS_RDB_TYPE_ZSET_ZIPLIST  12<br/>#define REDIS_RDB_TYPE_HASH_ZIPLIST  13
Key 数据存储 |      N          |                    | 参考 RDB Key数据存储
Val 数据存储 |      N          |                    | 因为val有可能是int，也有可能是 list　等类型，假如是 list 
等类型都是优先转成string之后再参考 RDB String 存储
....        |       |       |
结束标识     |      1          | 255                |
Cksum值     |      8         |                       |  当开启了 rdbchecksum yes 才有

 

#### Redis RDB数据不定长数据存储协议

假如接下来要存储的长度为 x
x < 64 代表这一个字节的长度就是接下来存储数据的长度 就是x 
64<= x < 16384 , 则接下来第一个字节存储的是64，并且接下来的另一个字节以  len&0xFF 数据存储. 意味着，当接下来的数据长度 64<= x < 16384 , 是用2个字节来存储的
x >=16384 ，则则第一个字节存储的是255，并且接下来的4个字节存储的是  x，代表数据长度 x >=16384 是用 5个字节来存储的
 
 
#### RDB Key数据存储
 
##### Key 是int类型
 
当 -128 <= x <=127 ，采用2个字节存储
 
标题 | 长度 | 案例 | 备注
---|---|---|---
第1个字节 | 1 | 192 | (3<<6)或者0
第2个字节 | 1 |     | X & 255 value&0xFF


当 -32768 <= x <=32767，采用3个字节存储

标题 | 长度 | 案例 | 备注
---|---|---|---
第1个字节 | 1 | 193| 193 (3<<6)或者 1
第2-3个字节| 2 | x |  
       
 
当 -2147483648<= x <=2147483647，采用5个字节存储

标题          | 长度    |   案例      | 备注
---|---|---|---
第1个字节      | 1      |   194       | (3<<6)或2
第2-5个字节    |4       | 2147483647  |  
       
 
当 x 超过了有符号的4个字节存储的大小，则转成string 存储 , 请参考 RDB String存储
 
 
#### RDB String 存储
 
当string 长度 小于或等于 11 会尝试转成 32位的数字 进行存储 参考  RDB Key数据存储
 
否则进入下面逻辑

假如 rdbcompression yes 开启了，则在 len > 20的时候，会进行LZF 压缩算法
 
##### RDB String 存储 - LZF 压缩

标题          | 长度    |   案例      | 备注
---|---|---|---
类型存储       |  1     | 195         | (3<<6)或3 ，表示接下来的数据是LZF压缩的
压缩过后的长度  |  N     | comprlen    | 参考 Redis RDB数据不定长数据存储协议
压缩前的长度    |  N     |             | 参考 Redis RDB数据不定长数据存储协议
压缩过后的内容  | comprlen |           | 长度是comprlen,压缩过后的内容
       
 
 
 
##### RDB String 存储 - 普通存储

标题          | 长度    |   案例      | 备注
---|---|---|---
长度存储       | 1-5    |   16        | 参考 Redis RDB数据不定长数据存储协议
String内容    | N      | MyTestVale  | 前面1-5个存储的值，就是这个内容的长度
       
       
       
#### RDB Value 存储
 
标题          | 长度    |   案例      | 备注
---|---|---|---
REDIS_STRING  | N       | MyTestVale | 参考 RDB String 存储
REDIS_LIST    |        |             | 参考 DB value 存储 - REDIS_LIST 
REDIS_SET     |        |             |      
REDIS_ZSET     |        |             |            
REDIS_HASH     |        |             |            
 

##### RDB value 存储 - REDIS_LIST
 
标题          | 长度    |   案例      | 备注
---|---|---|---
REDIS_ENCODING_ZIPLIST | N |        | 字符串长度
REDIS_ENCODING_LINKEDLIST |         | 参考 RDB value 存储 - REDIS_LIST - REDIS_ENCODING_LINKEDLIST 
       
       
       
 
##### RDB value 存储 - REDIS_LIST - REDIS_ENCODING_LINKEDLIST
 
标题          | 长度    |   案例      | 备注
---|---|---|---
List的个数    | 1-5     |            | 这里存的是 list 的个数，不是整个list 占的字节数参考 Redis RDB数据不定长数据存储协议
List数据内容  | N       |             | 遍历list的值，存储，每个值一条记录参考 RDB String 存储
       
 
##### RDB value 存储 - REDIS_SET
 
标题          | 长度    |   案例      | 备注
---|---|---|---
REDIS_ENCODING_HT | N  |            | 字符串长度
REDIS_ENCODING_INTSET | N           | 参考 RDB String 存储 
       
       
       
 
###### RDB value 存储 - REDIS_SET - REDIS_ENCODING_HT

标题          | 长度    |   案例      | 备注
---|---|---|---
字典的大小     | N       |            | 这里存的是 map里的个数，不是整个map占的字节数参考 Redis RDB数据不定长数据存储协议
字典的key      | N      |            | 遍历map的key，存储，每个值一条记录参考 RDB String 存储
       
 
##### RDB value 存储 - REDIS_ZSET
 
标题          | 长度    |   案例      | 备注
---|---|---|---
REDIS_ENCODING_ZIPLIST | N |         | 参考 RDB String 存储
REDIS_ENCODING_SKIPLIST| N |         | 参考RDB value 存储 - REDIS_ZSET- REDIS_ENCODING_SKIPLIST 
 

######RDB value 存储 - REDIS_ZSET- REDIS_ENCODING_SKIPLIST

标题          | 长度    |   案例      | 备注
---|---|---|---
字典的大小     | N      |             | 这里存的是 map里的个数，不是整个map占的字节数参考 Redis RDB数据不定长数据存储协议
字典的内容     |         |             | 这个字典存的都是double值

标题          | 长度    |   案例      | 备注
---|---|---|---
字典key       | N         |          | 参考 RDB String 存储
字典val       | 128       |          | Doule值
 
 
##### RDB value 存储 - REDIS_HASH
 
标题          | 长度    |   案例      | 备注
---|---|---|---
REDIS_ENCODING_ZIPLIST | N |        | 参考 RDB String 存储
REDIS_ENCODING_HT      |    |       | 参考RDB value 存储 - REDIS_HASH- REDIS_ENCODING_HT 
 

##### RDB value 存储 - REDIS_HASH- REDIS_ENCODING_HT

标题          | 长度    |   案例      | 备注
---|---|---|---
字典的大小     | N       |            |这里存的是 map里的个数，不是整个map占的字节数参考 Redis RDB数据不定长数据存储协议
字典的内容     |         |             |
    


标题          | 长度    |   案例      | 备注
---|---|---|---
字典key       | N       |             | 参考 RDB String 存储
字典val       | N       |             | 参考 RDB String 存储
 
