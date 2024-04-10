## Redis五种数据类型
- [Redis五种数据类型](#redis五种数据类型)
  - [1 Redis 的设计原则](#1-redis-的设计原则)
  - [2 Redis 的两层数据结构](#2-redis-的两层数据结构)
    - [2.1 Redis 两层数据结构简介](#21-redis-两层数据结构简介)
    - [2.2 两层数据结构的关联](#22-两层数据结构的关联)
    - [2.3 两层数据结构的实现：redisobject](#23-两层数据结构的实现redisobject)
      - [2.3.1 什么是 redisobject](#231-什么是-redisobject)
      - [2.3.2 redisobject 的定义](#232-redisobject-的定义)
      - [2.3.3 redisobject 的作用](#233-redisobject-的作用)
  - [3 Redis 的五种数据类型](#3-redis-的五种数据类型)
  - [总结](#总结)
  - [5 相关链接](#5-相关链接)

### 1 Redis 的设计原则
- **存储效率**。Redis是专用于存储数据的，它对计算机资源的主要消耗就在于**内存**，因此节省内存是它非常重要的一个方面。这意味着Redis一定是非常精细地考虑了压缩数据、减少内存碎片等问题。
- **快速响应时间**。与快速响应时间相对的，是高吞吐量。Redis是用于提供在线访问的，对于单个请求的响应时间要求很高，因此，快速响应时间是比高吞吐量更重要的目标。
- **单线程**。Redis的性能瓶颈不在于CPU资源，而在于**内存访问和网络IO**。而采用单线程的设计带来的好处是，极大简化了数据结构和算法的实现。相反，Redis通过异步IO和pipelining等机制来实现高速的并发访问。显然，单线程的设计，对于单个请求的快速响应时间也提出了更高的要求。

### 2 Redis 的两层数据结构
#### 2.1 Redis 两层数据结构简介
> - 对外暴露的每种数据结构都是经过专门设计的，并都有一种或多种数据结构来支持，依赖这些灵活的数据结构，来提升读取和写入的性能
> - Redis 通过组合第二个层面的各种基础数据结构来实现第一个层面的更高层的数据结构
- 第一层数据结构：string、list、hash、set、zset
- 第二层数据结构：sds、quicklist、ziplist、skiplist、intset、dict
#### 2.2 两层数据结构的关联
![数据类型与底层数据结构的对应关系](<images/redis 数据类型与底层数据结构的对应关系.png>)
#### 2.3 两层数据结构的实现：redisobject
##### 2.3.1 什么是 redisobject
- 从Redis的使用者的角度来看，一个Redis节点包含多个database（非cluster模式下默认是16个，cluster模式下只能是1个），而一个database维护了从key space到object space的映射关系，这个映射关系的key是string类型，而value可以是多种数据类型，比如：string, list, hash、set、sorted set等
- 从Redis内部实现的角度来看，database内的这个映射关系是用一个dict来维护的。dict的key固定用一种数据结构来表达就够了，这就是动态字符串sds；而value则比较复杂，为了在同一个dict内能够存储不同类型的value，这就需要一个通用的数据结构，这个通用的数据结构就是robj，全名是redisObject

##### 2.3.2 redisobject 的定义
1）数据结构的定义
- type: 对象的数据类型。占4个bit。可能的取值有5种：OBJ_STRING, OBJ_LIST, OBJ_SET, OBJ_ZSET, OBJ_HASH，分别对应Redis对外暴露的5种数据结构
- encoding: 对象的内部表示方式（也可以称为编码）。占4个bit。可能的取值有10种，即前面代码中的10个OBJ_ENCODING_XXX常量。
- refcount: 引用计数。它允许robj对象在某些情况下被共享。
- 数据指针。指向真正的数据。比如，一个代表string的robj，它的ptr可能指向一个sds结构；一个代表list的robj，它的ptr可能指向一个quicklist。
- 下面代码为 7.2 分支
```
struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
};
```

```
/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```

```
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* No longer used: old hash encoding. */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* No longer used: old list/hash/zset encoding. */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of listpacks */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
#define OBJ_ENCODING_LISTPACK 11 /* Encoded as a listpack */
```
2）encoding字段的说明
- 对于同一个type，还可能对应不同的encoding，这说明同样的一个数据类型，可能存在不同的内部表示方式。而不同的内部表示，在内存占用和查找性能上会有所不同。
**encoding 取值：**
- OBJ_ENCODING_RAW: 最原生的表示方式。其实只有string类型才会用这个encoding值（表示成sds）。
- OBJ_ENCODING_INT: 表示成数字。实际用long表示。
- OBJ_ENCODING_HT: 表示成dict。
- OBJ_ENCODING_ZIPMAP: 是个旧的表示方式，已不再用。在小于Redis 2.6的版本中才有。
- OBJ_ENCODING_LINKEDLIST: 也是个旧的表示方式，已不再用。
- OBJ_ENCODING_ZIPLIST: 表示成ziplist。
- OBJ_ENCODING_INTSET: 表示成intset。用于set数据结构。
- OBJ_ENCODING_SKIPLIST: 表示成skiplist。用于sorted set数据结构。
- OBJ_ENCODING_EMBSTR: 表示成一种特殊的嵌入式的sds。
- OBJ_ENCODING_QUICKLIST: 表示成quicklist。用于list数据结构。

##### 2.3.3 redisobject 的作用
- redisObject就是Redis对外暴露的第一层面的数据结构：string, list, hash, set, sorted set，而每一种数据结构的底层实现所对应的是哪些第二层面的数据结构（dict, sds, ziplist, quicklist, skiplist等），则通过不同的encoding来区分。可以说，robj是联结两个层面的数据结构的桥梁。
- 为多种数据类型提供一种统一的表示方式。
- 允许同一类型的数据采用不同的内部表示，从而在某些情况下尽量节省内存。
- 支持对象共享和引用计数。当对象被共享的时候，只占用一份内存拷贝，进一步节省内存。


### 3 Redis 的五种数据类型
- [字符串对象(string)](https://github.com/pei0948/db/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/string.md)
- [列表对象(list)](https://github.com/pei0948/db/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/list.md)
- [哈希对象(hash)](https://github.com/pei0948/db/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/hash.md)
- [集合对象(set)](https://github.com/pei0948/db/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/set.md)
- [有序集合对象(zset)](https://github.com/pei0948/db/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/zset.md)


### <h3 id="redis_data_structure_6">总结</h3>
- 在Redis的五大数据对象中，string对象是唯一个可以被其他四种数据对象作为内嵌对象的；
- 列表（list）、哈希（hash）、集合（set）、有序集合（zset）底层实现都用到了压缩列表结构，并且使用压缩列表结构的条件都是在元素个数比较少、字节长度较短的情况下；

四种数据对象使用压缩列表的优点：
* 1. 节约内存，减少内存开销，Redis是内存型数据库，所以一定情况下减少内存开销是非常有必要的。
* 2. 减少内存碎片，压缩列表的内存块是连续的，并分配内存的次数一次即可。
* 3. 压缩列表的新增、删除、查找操作的平均时间复杂度是O(N)，在N再一定的范围内，这个时间几乎是可以忽略的，并且N的上限值是可以配置的。
* 4. 四种数据对象都有两种编码结构，灵活性增加。

### 5 相关链接
- [Redis的五种数据结构的底层实现原理](https://blog.csdn.net/a745233700/article/details/113449889)