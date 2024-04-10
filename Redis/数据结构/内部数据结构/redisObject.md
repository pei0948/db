## redisObject 简介
> redis 版本：7.2 分支
### 1 redisObject 的作用
- redisObject就是Redis对外暴露的第一层面的数据结构：string, list, hash, set, sorted set，而每一种数据结构的底层实现所对应的是哪些第二层面的数据结构（dict, sds, ziplist, quicklist, skiplist等），则通过不同的encoding来区分。可以说，robj是联结两个层面的数据结构的桥梁。
- 为多种数据类型提供一种统一的表示方式。
- 允许同一类型的数据采用不同的内部表示，从而在某些情况下尽量节省内存。
- 支持对象共享和引用计数。当对象被共享的时候，只占用一份内存拷贝，进一步节省内存。
### 2 redisObject 的定义
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

### 3 redisObject 的编解码
#### 3.1 编码过程
编码过程做了件事：
- 检查 type。只对字符串对象（OBJ_STRING）进行处理
- 检查 encoding。确保只对 REDIS_ENCODING_RAW 和 OBJ_ENCODING_EMBSTR 类型的对象进行操作
- 检查 refcount。引用计数大于1的共享对象，在多处被引用。由于编码过程结束后robj的对象指针可能会变化（我们在前一篇介绍sdscatlen函数的时候提到过类似这种接口使用模式），这样对于引用计数大于1的对象，就需要更新所有地方的引用，这不容易做到。因此，对于计数大于1的对象不做编码处理
- 试图将字符串转成64位的long。64位的long所能表达的数据范围是-2^63到2^63-1，用十进制表达出来最长是20位数（包括负号）。string2l如果将字符串转成long转成功了，那么会返回1并且将转好的long存到value变量里。
- 在转成long成功时，又分为两种情况：
  - 1）如果Redis的配置不要求运行LRU替换算法，且转成的long型数字的值又在 [0, 10000) 内，那么会使用共享数字对象来表示。之所以这里的判断跟LRU有关，是因为LRU算法要求每个robj有不同的lru字段值，所以用了LRU就不能共享robj。shared.integers是一个长度为10000的数组，里面预存了10000个小的数字对象。这些小数字对象都是encoding = OBJ_ENCODING_INT的string robj对象。
  - 2）如果前一步不能使用共享小对象来表示，那么将原来的robj编码成encoding = OBJ_ENCODING_INT，这时ptr字段直接存成这个long型的值。注意ptr字段本来是一个void *指针（即存储的是内存地址），因此在64位机器上有64位宽度，正好能存储一个64位的long型值。这样，除了robj本身之外，它就不再需要额外的内存空间来存储字符串值。
- 对于那些不能转成64位long的字符串进行处理。最后再做两步处理：
  - 1）如果字符串长度足够小（小于等于OBJ_ENCODING_EMBSTR_SIZE_LIMIT，定义为44），那么调用createEmbeddedStringObject编码成encoding = OBJ_ENCODING_EMBSTR；
  - 2）如果前面所有的编码尝试都没有成功（仍然是OBJ_ENCODING_RAW），且sds里空余字节过多，那么做最后一次努力，调用sds的sdsRemoveFreeSpace接口来释放空余字节。
```
/* Try to encode a string object in order to save space */
robj *tryObjectEncodingEx(robj *o, int try_trim) {
    long value;
    sds s = o->ptr;
    size_t len;

    /* Make sure this is a string object, the only type we encode
     * in this function. Other types use encoded memory efficient
     * representations but are handled by the commands implementing
     * the type. */
    serverAssertWithInfo(NULL,o,o->type == OBJ_STRING);

    /* We try some specialized encoding only for objects that are
     * RAW or EMBSTR encoded, in other words objects that are still
     * in represented by an actually array of chars. */
    if (!sdsEncodedObject(o)) return o;

    /* It's not safe to encode shared objects: shared objects can be shared
     * everywhere in the "object space" of Redis and may end in places where
     * they are not handled. We handle them only as values in the keyspace. */
     if (o->refcount > 1) return o;

    /* Check if we can represent this string as a long integer.
     * Note that we are sure that a string larger than 20 chars is not
     * representable as a 32 nor 64 bit integer. */
    len = sdslen(s);
    if (len <= 20 && string2l(s,len,&value)) {
        /* This object is encodable as a long. Try to use a shared object.
         * Note that we avoid using shared integers when maxmemory is used
         * because every object needs to have a private LRU field for the LRU
         * algorithm to work well. */
        if ((server.maxmemory == 0 ||
            !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS)) &&
            value >= 0 &&
            value < OBJ_SHARED_INTEGERS)
        {
            decrRefCount(o);
            return shared.integers[value];
        } else {
            if (o->encoding == OBJ_ENCODING_RAW) {
                sdsfree(o->ptr);
                o->encoding = OBJ_ENCODING_INT;
                o->ptr = (void*) value;
                return o;
            } else if (o->encoding == OBJ_ENCODING_EMBSTR) {
                decrRefCount(o);
                return createStringObjectFromLongLongForValue(value);
            }
        }
    }

    /* If the string is small and is still RAW encoded,
     * try the EMBSTR encoding which is more efficient.
     * In this representation the object and the SDS string are allocated
     * in the same chunk of memory to save space and cache misses. */
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT) {
        robj *emb;

        if (o->encoding == OBJ_ENCODING_EMBSTR) return o;
        emb = createEmbeddedStringObject(s,sdslen(s));
        decrRefCount(o);
        return emb;
    }

    /* We can't encode the object...
     * Do the last try, and at least optimize the SDS string inside */
    if (try_trim)
        trimStringObjectIfNeeded(o, 0);

    /* Return the original object. */
    return o;
}
```
#### 3.2 解码过程
解码过程做了这些事：
- 如果编码是 REDIS_ENCODING_RAW 或者 OBJ_ENCODING_EMBSTR，则增加引用计数返回
- 如果为字符串类型，并且编码为 REDIS_ENCODING_INT，则创建字符串返回
```
/* Get a decoded version of an encoded object (returned as a new object).
 * If the object is already raw-encoded just increment the ref count. */
robj *getDecodedObject(robj *o) {
    robj *dec;

    if (sdsEncodedObject(o)) {
        incrRefCount(o);
        return o;
    }
    if (o->type == OBJ_STRING && o->encoding == OBJ_ENCODING_INT) {
        char buf[32];

        ll2string(buf,32,(long)o->ptr);
        dec = createStringObject(buf,strlen(buf));
        return dec;
    } else {
        serverPanic("Unknown encoding type");
    }
}
```