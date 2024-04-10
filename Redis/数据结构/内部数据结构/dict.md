- [1 简介](#1-简介)
- [2 底层实现](#2-底层实现)
  - [2.1 dict 的数据结构定义](#21-dict-的数据结构定义)
  - [2.2 dict 的创建](#22-dict-的创建)
  - [2.2 dict 的查找](#22-dict-的查找)
  - [2.2 dict 的插入](#22-dict-的插入)
  - [2.2 dict 的删除](#22-dict-的删除)

## 1 简介
- dict是一个用于维护key-value映射关系的数据结构。Redis的一个database中所有key到value的映射，就是使用一个dict来维护的。
- 不过，这只是它在Redis中的一个用途而已，它在Redis中被使用的地方还有很多。比如，Redis的hash结构，当它的field较多时，便会采用dict来存储。再比如，Redis配合使用dict和skiplist来共同维护一个sorted set。
- dict本质上是为了解决算法中的查找问题，是一个基于哈希表的算法，在不要求数据有序存储，且能保持较低的哈希值冲突概率的前提下，查询的时间复杂度接近O(1)。(一般查找问题的解法分为两个大类：一个是基于各种平衡树，一个是基于哈希表)

## 2 底层实现

- 它采用某个哈希函数并通过计算key从而找到在哈希表中的位置，采用拉链法解决冲突，并在装载因子（load factor）超过预定值时自动扩展内存，引发重哈希（rehashing），为了避免扩容时一次性对所有key进行重哈希，Redis采用了一种称为增量式重哈希（incremental rehashing）的方法，将重哈希的操作分散到对于dict的各个增删改查的操作中去。这种方法能做到每次只对一小部分key进行重哈希，而每次重哈希之间不影响dict的操作。dict之所以这样设计，是为了避免重哈希期间单个请求的响应时间剧烈增加，这与前面提到的“快速响应时间”的设计原则是相符的。
- 当装载因子大于 1 的时候，Redis 会触发扩容，将散列表扩大为原来大小的 2 倍左右；当数据动态减少之后，为了节省内存，当装载因子小于 0.1 的时候，Redis 就会触发缩容，缩小为字典中数据个数的大约2 倍大小。

### 2.1 dict 的数据结构定义
> 7.2 分支

![dict 的定义](<../../images/dict 的定义.png>)
- 为了实现增量式重哈希（incremental rehashing），dict的数据结构里包含两个哈希表。在重哈希期间，数据从第一个哈希表向第二个哈希表迁移 

一个dict由如下若干项组成：
- 一个指向dictType结构的指针（type）。它通过自定义的方式使得dict的key和value能够存储任何类型的数据。
- 一个私有数据指针（privdata）。由调用者在创建dict的时候传进来。
- 两个哈希表（ht[2]）。只有在重哈希的过程中，ht[0]和ht[1]才都有效。而在平常情况下，只有ht[0]有效，ht[1]里面没有任何数据
- 当前重哈希索引（rehashidx）。如果rehashidx = -1，表示当前没有在重哈希过程中；否则，表示当前正在进行重哈希，且它的值记录了当前重哈希进行到哪一步了
- ht_size_exp 记录了两个表的 size: exponent of size. (size = 1<<exp)
- ht_used 记录了两个 dict 表中现有的数据个数，与 size 的比值为装填因子

- dict
```
struct dict {
    dictType *type;

    dictEntry **ht_table[2];
    unsigned long ht_used[2];

    long rehashidx; /* rehashing not in progress if rehashidx == -1 */

    /* Keep small vars at end for optimal (minimal) struct padding */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
    signed char ht_size_exp[2]; /* exponent of size. (size = 1<<exp) */

    void *metadata[];           /* An arbitrary number of bytes (starting at a
                                 * pointer-aligned address) of size as defined
                                 * by dictType's dictEntryBytes. */
};
```
- dictType

dictType结构包含若干函数指针，用于dict的调用者对涉及key和value的各种操作进行自定义。这些操作包含：
- hashFunction，对key进行哈希值计算的哈希算法。
- keyDup和valDup，分别定义key和value的拷贝函数，用于在需要的时候对key和value进行深拷贝，而不仅仅是传递对象指针。
- keyCompare，定义两个key的比较操作，在根据key进行查找时会用到。
- keyDestructor和valDestructor，分别定义对key和value的析构函数。
  
私有数据指针（privdata）就是在dictType的某些操作被调用时会传回给调用者。
```
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(dict *d, const void *key);
    void *(*valDup)(dict *d, const void *obj);
    int (*keyCompare)(dict *d, const void *key1, const void *key2);
    void (*keyDestructor)(dict *d, void *key);
    void (*valDestructor)(dict *d, void *obj);
    int (*expandAllowed)(size_t moreMem, double usedRatio);
    /* Flags */
    /* The 'no_value' flag, if set, indicates that values are not used, i.e. the
     * dict is a set. When this flag is set, it's not possible to access the
     * value of a dictEntry and it's also impossible to use dictSetKey(). Entry
     * metadata can also not be used. */
    unsigned int no_value:1;
    /* If no_value = 1 and all keys are odd (LSB=1), setting keys_are_odd = 1
     * enables one more optimization: to store a key without an allocated
     * dictEntry. */
    unsigned int keys_are_odd:1;
    /* TODO: Add a 'keys_are_even' flag and use a similar optimization if that
     * flag is set. */

    /* Allow each dict and dictEntry to carry extra caller-defined metadata. The
     * extra memory is initialized to 0 when allocated. */
    size_t (*dictEntryMetadataBytes)(dict *d);
    size_t (*dictMetadataBytes)(void);
    /* Optional callback called after an entry has been reallocated (due to
     * active defrag). Only called if the entry has metadata. */
    void (*afterReplaceEntry)(dict *d, dictEntry *entry);
} dictType;
```
- dictEntry
dictEntry 结构：
- dictEntry结构中包含k, v和指向链表下一项的next指针。k是void指针，这意味着它可以指向任何类型。v是个union，当它的值是uint64_t、int64_t或double类型时，就不再需要额外的存储，这有利于减少内存碎片。当然，v也可以是void指针，以便能存储任何类型的数据。

```
struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;     /* Next entry in the same hash bucket. */
    void *metadata[];           /* An arbitrary number of bytes (starting at a
                                 * pointer-aligned address) of size as returned
                                 * by dictType's dictEntryMetadataBytes(). */
};
```
### 2.2 dict 的创建
- dictCreate为dict的数据结构分配空间并为各个变量赋初值。其中两个哈希表ht[0]和ht[1]起始都没有分配空间，table指针都赋为NULL。这意味着要等第一个数据插入时才会真正分配空间。
```
/* Create a new hash table */
dict *dictCreate(dictType *type)
{
    size_t metasize = type->dictMetadataBytes ? type->dictMetadataBytes() : 0;
    dict *d = zmalloc(sizeof(*d) + metasize);
    if (metasize) {
        memset(dictMetadata(d), 0, metasize);
    }

    _dictInit(d,type);
    return d;
}
```
```
/* Initialize the hash table */
int _dictInit(dict *d, dictType *type)
{
    _dictReset(d, 0);
    _dictReset(d, 1);
    d->type = type;
    d->rehashidx = -1;
    d->pauserehash = 0;
    return DICT_OK;
}
```
```
/* Reset hash table parameters already initialized with _dictInit()*/
static void _dictReset(dict *d, int htidx)
{
    d->ht_table[htidx] = NULL;
    d->ht_size_exp[htidx] = -1;
    d->ht_used[htidx] = 0;
}
```
### 2.2 dict 的查找
上述dictFind的源码，根据dict当前是否正在重哈希，依次做了这么几件事：
- 如果当前正在进行重哈希，那么将重哈希过程向前推进一步（即调用_dictRehashStep）。实际上，除了查找，插入和删除也都会触发这一动作。这就将重哈希过程分散到各个查找、插入和删除操作中去了，而不是集中在某一个操作中一次性做完。
- 计算key的哈希值（调用dictHashKey，里面的实现会调用前面提到的hashFunction）。
- 先在第一个哈希表ht[0]上进行查找。在table数组上定位到哈希值对应的位置（如前所述，通过哈希值与sizemask进行按位与），然后在对应的dictEntry链表上进行查找。查找的时候需要对key进行比较，这时候调用dictCompareKeys，它里面的实现会调用到前面提到的keyCompare。如果找到就返回该项。否则，进行下一步。
- 判断当前是否在重哈希，如果没有，那么在ht[0]上的查找结果就是最终结果（没找到，返回NULL）。否则，在ht[1]上进行查找（过程与上一步相同）。
```
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    uint64_t h, idx, table;

    if (dictSize(d) == 0) return NULL; /* dict is empty */
    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);
    for (table = 0; table <= 1; table++) {
        idx = h & DICTHT_SIZE_MASK(d->ht_size_exp[table]);
        he = d->ht_table[table][idx];
        while(he) {
            void *he_key = dictGetKey(he);
            if (key == he_key || dictCompareKeys(d, key, he_key))
                return he;
            he = dictGetNext(he);
        }
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```

**_dictRehashStep 的实现：**
- dictRehash每次将重哈希至少向前推进n步（除非不到n步整个重哈希就结束了），每一步都将ht[0]上某一个bucket（即一个dictEntry链表）上的每一个dictEntry移动到ht[1]上，它在ht[1]上的新位置根据ht[1]的sizemask进行重新计算。rehashidx记录了当前尚未迁移（有待迁移）的ht[0]的bucket位置。

- 如果dictRehash被调用的时候，rehashidx指向的bucket里一个dictEntry也没有，那么它就没有可迁移的数据。这时它尝试在ht[0].table数组中不断向后遍历，直到找到下一个存有数据的bucket位置。如果一直找不到，则最多走n*10步，本次重哈希暂告结束。

- 最后，如果ht[0]上的数据都迁移到ht[1]上了（即d->ht[0].used == 0），那么整个重哈希结束，ht[0]变成ht[1]的内容，而ht[1]重置为空。

```
/* Performs N steps of incremental rehashing. Returns 1 if there are still
 * keys to move from the old to the new hash table, otherwise 0 is returned.
 *
 * Note that a rehashing step consists in moving a bucket (that may have more
 * than one key as we use chaining) from the old to the new hash table, however
 * since part of the hash table may be composed of empty spaces, it is not
 * guaranteed that this function will rehash even a single bucket, since it
 * will visit at max N*10 empty buckets in total, otherwise the amount of
 * work it does would be unbound and the function may block for a long time. */
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    unsigned long s0 = DICTHT_SIZE(d->ht_size_exp[0]);
    unsigned long s1 = DICTHT_SIZE(d->ht_size_exp[1]);
    if (dict_can_resize == DICT_RESIZE_FORBID || !dictIsRehashing(d)) return 0;
    if (dict_can_resize == DICT_RESIZE_AVOID && 
        ((s1 > s0 && s1 / s0 < dict_force_resize_ratio) ||
         (s1 < s0 && s0 / s1 < dict_force_resize_ratio)))
    {
        return 0;
    }

    while(n-- && d->ht_used[0] != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(DICTHT_SIZE(d->ht_size_exp[0]) > (unsigned long)d->rehashidx);
        while(d->ht_table[0][d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht_table[0][d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = dictGetNext(de);
            void *key = dictGetKey(de);
            /* Get the index in the new hash table */
            if (d->ht_size_exp[1] > d->ht_size_exp[0]) {
                h = dictHashKey(d, key) & DICTHT_SIZE_MASK(d->ht_size_exp[1]);
            } else {
                /* We're shrinking the table. The tables sizes are powers of
                 * two, so we simply mask the bucket index in the larger table
                 * to get the bucket index in the smaller table. */
                h = d->rehashidx & DICTHT_SIZE_MASK(d->ht_size_exp[1]);
            }
            if (d->type->no_value) {
                if (d->type->keys_are_odd && !d->ht_table[1][h]) {
                    /* Destination bucket is empty and we can store the key
                     * directly without an allocated entry. Free the old entry
                     * if it's an allocated entry.
                     *
                     * TODO: Add a flag 'keys_are_even' and if set, we can use
                     * this optimization for these dicts too. We can set the LSB
                     * bit when stored as a dict entry and clear it again when
                     * we need the key back. */
                    assert(entryIsKey(key));
                    if (!entryIsKey(de)) zfree(decodeMaskedPtr(de));
                    de = key;
                } else if (entryIsKey(de)) {
                    /* We don't have an allocated entry but we need one. */
                    de = createEntryNoValue(key, d->ht_table[1][h]);
                } else {
                    /* Just move the existing entry to the destination table and
                     * update the 'next' field. */
                    assert(entryIsNoValue(de));
                    dictSetNext(de, d->ht_table[1][h]);
                }
            } else {
                dictSetNext(de, d->ht_table[1][h]);
            }
            d->ht_table[1][h] = de;
            d->ht_used[0]--;
            d->ht_used[1]++;
            de = nextde;
        }
        d->ht_table[0][d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht_used[0] == 0) {
        zfree(d->ht_table[0]);
        /* Copy the new ht onto the old one */
        d->ht_table[0] = d->ht_table[1];
        d->ht_used[0] = d->ht_used[1];
        d->ht_size_exp[0] = d->ht_size_exp[1];
        _dictReset(d, 1);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```
### 2.2 dict 的插入
- dictAdd插入新的一对key和value，如果key已经存在，则插入失败。
- dictReplace也是插入一对key和value，不过在key存在的时候，它会更新value。
1）dictAdd
- 它也会触发推进一步重哈希（_dictRehashStep）。
- 如果正在重哈希中，它会把数据插入到ht[1]；否则插入到ht[0]。
- - 在对应的bucket中插入数据的时候，总是插入到dictEntry的头部。因为新数据接下来被访问的概率可能比较高，这样再次查找它时就比较次数较少。
- _dictKeyIndex在dict中寻找插入位置。如果不在重哈希过程中，它只查找ht[0]；否则查找ht[0]和ht[1]。
- _dictKeyIndex可能触发dict内存扩展（_dictExpandIfNeeded，它将哈希表长度扩展为原来两倍，具体请参考dict.c中源码）。
```
/* Add an element to the target hash table */
int dictAdd(dict *d, void *key, void *val)
{
    dictEntry *entry = dictAddRaw(d,key,NULL);

    if (!entry) return DICT_ERR;
    if (!d->type->no_value) dictSetVal(d, entry, val);
    return DICT_OK;
}

/* Low level add or find:
 * This function adds the entry but instead of setting a value returns the
 * dictEntry structure to the user, that will make sure to fill the value
 * field as they wish.
 *
 * This function is also directly exposed to the user API to be called
 * mainly in order to store non-pointers inside the hash value, example:
 *
 * entry = dictAddRaw(dict,mykey,NULL);
 * if (entry != NULL) dictSetSignedIntegerVal(entry,1000);
 *
 * Return values:
 *
 * If key already exists NULL is returned, and "*existing" is populated
 * with the existing entry if existing is not NULL.
 *
 * If key was added, the hash entry is returned to be manipulated by the caller.
 */
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    /* Get the position for the new key or NULL if the key already exists. */
    void *position = dictFindPositionForInsert(d, key, existing);
    if (!position) return NULL;

    /* Dup the key if necessary. */
    if (d->type->keyDup) key = d->type->keyDup(d, key);

    return dictInsertAtPosition(d, key, position);
}
```
2）dictReplace
```
/* Add or Overwrite:
 * Add an element, discarding the old value if the key already exists.
 * Return 1 if the key was added from scratch, 0 if there was already an
 * element with such key and dictReplace() just performed a value update
 * operation. */
int dictReplace(dict *d, void *key, void *val)
{
    dictEntry *entry, *existing;

    /* Try to add the element. If the key
     * does not exists dictAdd will succeed. */
    entry = dictAddRaw(d,key,&existing);
    if (entry) {
        dictSetVal(d, entry, val);
        return 1;
    }

    /* Set the new value and free the old one. Note that it is important
     * to do that in this order, as the value may just be exactly the same
     * as the previous one. In this context, think to reference counting,
     * you want to increment (set), and then decrement (free), and not the
     * reverse. */
    void *oldval = dictGetVal(existing);
    dictSetVal(d, existing, val);
    if (d->type->valDestructor)
        d->type->valDestructor(d, oldval);
    return 0;
}
```

### 2.2 dict 的删除
- dictDelete也会触发推进一步重哈希（_dictRehashStep）
- 如果当前不在重哈希过程中，它只在ht[0]中查找要删除的key；否则ht[0]和ht[1]它都要查找。
- 删除成功后会调用key和value的析构函数（keyDestructor和valDestructor）。
```
/* Remove an element, returning DICT_OK on success or DICT_ERR if the
 * element was not found. */
int dictDelete(dict *ht, const void *key) {
    return dictGenericDelete(ht,key,0) ? DICT_OK : DICT_ERR;
}
```

```
/* Search and remove an element. This is a helper function for
 * dictDelete() and dictUnlink(), please check the top comment
 * of those functions. */
static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
    uint64_t h, idx;
    dictEntry *he, *prevHe;
    int table;

    /* dict is empty */
    if (dictSize(d) == 0) return NULL;

    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);

    for (table = 0; table <= 1; table++) {
        idx = h & DICTHT_SIZE_MASK(d->ht_size_exp[table]);
        he = d->ht_table[table][idx];
        prevHe = NULL;
        while(he) {
            void *he_key = dictGetKey(he);
            if (key == he_key || dictCompareKeys(d, key, he_key)) {
                /* Unlink the element from the list */
                if (prevHe)
                    dictSetNext(prevHe, dictGetNext(he));
                else
                    d->ht_table[table][idx] = dictGetNext(he);
                if (!nofree) {
                    dictFreeUnlinkedEntry(d, he);
                }
                d->ht_used[table]--;
                return he;
            }
            prevHe = he;
            he = dictGetNext(he);
        }
        if (!dictIsRehashing(d)) break;
    }
    return NULL; /* not found */
}
```