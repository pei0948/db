## redisObject 简介
> redis 版本：v2.0.4-stable-0-g1c1450736
### 1 redisObject 的作用
- redisObject就是Redis对外暴露的第一层面的数据结构：string, list, hash, set, sorted set，而每一种数据结构的底层实现所对应的是哪些第二层面的数据结构（dict, sds, ziplist, quicklist, skiplist等），则通过不同的encoding来区分。可以说，robj是联结两个层面的数据结构的桥梁。
- 为多种数据类型提供一种统一的表示方式。
- 允许同一类型的数据采用不同的内部表示，从而在某些情况下尽量节省内存。
- 支持对象共享和引用计数。当对象被共享的时候，只占用一份内存拷贝，进一步节省内存。
### 2 redisObject 的定义
```
/* The actual Redis Object */
typedef struct redisObject {
    void *ptr;
    unsigned char type;
    unsigned char encoding;
    unsigned char storage;  /* If this object is a key, where is the value?
                             * REDIS_VM_MEMORY, REDIS_VM_SWAPPED, ... */
    unsigned char vtype; /* If this object is a key, and value is swapped out,
                          * this is the type of the swapped out object. */
    int refcount;
    /* VM fields, this are only allocated if VM is active, otherwise the
     * object allocation function will just allocate
     * sizeof(redisObjct) minus sizeof(redisObjectVM), so using
     * Redis without VM active will not have any overhead. */
    struct redisObjectVM vm;
} robj;
```

```
/* Object types */
#define REDIS_STRING 0
#define REDIS_LIST 1
#define REDIS_SET 2
#define REDIS_ZSET 3
#define REDIS_HASH 4
```

```
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define REDIS_ENCODING_RAW 0    /* Raw representation */
#define REDIS_ENCODING_INT 1    /* Encoded as integer */
#define REDIS_ENCODING_ZIPMAP 2 /* Encoded as zipmap */
#define REDIS_ENCODING_HT 3     /* Encoded as an hash table */
```
### 3 redisObject 的编解码
#### 3.1 编码过程
编码过程做了件事：
- 检查 encoding。确保只对 REDIS_ENCODING_RAW 类型的对象进行操作
- 检查 refcount。引用计数大于1的共享对象，在多处被引用。由于编码过程结束后robj的对象指针可能会变化（我们在前一篇介绍sdscatlen函数的时候提到过类似这种接口使用模式），这样对于引用计数大于1的对象，就需要更新所有地方的引用，这不容易做到。因此，对于计数大于1的对象不做编码处理
- 试图将字符串转成 long，转换失败则返回
- 如果值在[0, 10000) 之内 并且当前线程为主线程，则增加共享对象的引用计数；否则，则将其转换为 REDIS_ENCODING_INT 编码
```
/* Try to encode a string object in order to save space */
static robj *tryObjectEncoding(robj *o) {
    long value;
    sds s = o->ptr;

    if (o->encoding != REDIS_ENCODING_RAW)
        return o; /* Already encoded */

    /* It's not safe to encode shared objects: shared objects can be shared
     * everywhere in the "object space" of Redis. Encoded objects can only
     * appear as "values" (and not, for instance, as keys) */
     if (o->refcount > 1) return o;

    /* Currently we try to encode only strings */
    redisAssert(o->type == REDIS_STRING);

    /* Check if we can represent this string as a long integer */
    if (isStringRepresentableAsLong(s,&value) == REDIS_ERR) return o;

    /* Ok, this object can be encoded...
     *
     * Can I use a shared object? Only if the object is inside a given
     * range and if this is the main thread, since when VM is enabled we
     * have the constraint that I/O thread should only handle non-shared
     * objects, in order to avoid race conditions (we don't have per-object
     * locking). */
    if (value >= 0 && value < REDIS_SHARED_INTEGERS &&
        pthread_equal(pthread_self(),server.mainthread)) {
        decrRefCount(o);
        incrRefCount(shared.integers[value]);
        return shared.integers[value];
    } else {
        o->encoding = REDIS_ENCODING_INT;
        sdsfree(o->ptr);
        o->ptr = (void*) value;
        return o;
    }
}
```
#### 3.2 解码过程
解码过程做了这些事：
- 如果编码是 REDIS_ENCODING_RAW，则增加引用计数返回
- 如果为字符串类型，并且编码为 REDIS_ENCODING_INT，则创建字符串返回
```
/* Get a decoded version of an encoded object (returned as a new object).
 * If the object is already raw-encoded just increment the ref count. */
static robj *getDecodedObject(robj *o) {
    robj *dec;

    if (o->encoding == REDIS_ENCODING_RAW) {
        incrRefCount(o);
        return o;
    }
    if (o->type == REDIS_STRING && o->encoding == REDIS_ENCODING_INT) {
        char buf[32];

        ll2string(buf,32,(long)o->ptr);
        dec = createStringObject(buf,strlen(buf));
        return dec;
    } else {
        redisPanic("Unknown encoding type");
    }
}
```