## ziplist 压缩列表

- [ziplist 压缩列表](#ziplist-压缩列表)
  - [1. 简介](#1-简介)
  - [2. 实现](#2-实现)
    - [2.1 数据结构](#21-数据结构)
      - [2.1.1 整体结构](#211-整体结构)
      - [2.1.2 entry 定义](#212-entry-定义)
      - [2.1.3 例子](#213-例子)
    - [2.2 ziplist 接口](#22-ziplist-接口)
      - [2.2.1 接口定义](#221-接口定义)
      - [2.2.2 插入接口](#222-插入接口)
  - [3. hash 与 ziplist](#3-hash-与-ziplist)
    - [3.1 实现](#31-实现)
    - [3.2 ziplist 实现 hash 的缺点](#32-ziplist-实现-hash-的缺点)
  - [4. 相关链接](#4-相关链接)


### 1. 简介
- ziplist本来就设计为各个数据项挨在一起组成连续的内存空间，这种结构并不擅长做修改操作。一旦数据发生改动，就会引发内存realloc，可能导致内存拷贝。
- ziplist是一个经过特殊编码的双向链表，它的设计目标就是为了提高存储效率。ziplist可以用于存储字符串或整数，其中整数是按真正的二进制表示进行编码的，而不是编码成字符串序列。它能以O(1)的时间复杂度在表的两端提供push和pop操作。
### 2. 实现
#### 2.1 数据结构
##### 2.1.1 整体结构
从宏观上看，ziplist的内存结构如下：
```
<zlbytes><zltail><zllen><entry>...<entry><zlend>
```

各个部分在内存上是前后相邻的，它们分别的含义如下：

- \<zlbytes\>: 32bit，表示ziplist占用的字节总数（也包括\<zlbytes\>本身占用的4个字节）。
- \<zltail\>: 32bit，表示ziplist表中最后一项（entry）在ziplist中的偏移字节数。\<zltail\>的存在，使得我们可以很方便地找到最后一项（不用遍历整个ziplist），从而可以在ziplist尾端快速地执行push或pop操作。
- \<zllen\>: 16bit， 表示ziplist中数据项（entry）的个数。zllen字段因为只有16bit，所以可以表达的最大值为2^16-1。这里需要特别注意的是，如果ziplist中数据项个数超过了16bit能表达的最大值，ziplist仍然可以来表示。那怎么表示呢？这里做了这样的规定：如果\<zllen\>小于等于2^16-2（也就是不等于2^16-1），那么\<zllen\>就表示ziplist中数据项的个数；否则，也就是\<zllen\>等于16bit全为1的情况，那么\<zllen\>就不表示数据项个数了，这时候要想知道ziplist中数据项总数，那么必须对ziplist从头到尾遍历各个数据项，才能计数出来。
- \<entry\>: 表示真正存放数据的数据项，长度不定。一个数据项（entry）也有它自己的内部结构，这个稍后再解释。
- \<zlend\>: ziplist最后1个字节，是一个结束标记，值固定等于255。
上面的定义中还值得注意的一点是：\<zlbytes\>, \<zltail\>, \<zllen\>既然占据多个字节，那么在存储的时候就有大端（big endian）和小端（little endian）的区别。ziplist采取的是小端模式来存储，这在下面我们介绍具体例子的时候还会再详细解释。
##### 2.1.2 entry 定义
```
<prevrawlen><len><data>
```

1）我们看到在真正的数据（<data>）前面，还有两个字段：

- \<prevrawlen\>: 表示前一个数据项占用的总字节数。这个字段的用处是为了让ziplist能够从后向前遍历（从后一项的位置，只需向前偏移prevrawlen个字节，就找到了前一项）。这个字段采用变长编码。
- \<len\>: 表示当前数据项的数据长度（即\<data\>部分的长度）。也采用变长编码。

2）那么\<prevrawlen\>和\<len\>是怎么进行变长编码的呢？
\<prevrawlen\>有两种可能，或者是1个字节，或者是5个字节：
- 如果前一个数据项占用字节数小于254，那么\<prevrawlen\>就只用一个字节来表示，这个字节的值就是前一个数据项的占用字节数。
- 如果前一个数据项占用字节数大于等于254，那么\<prevrawlen\>就用5个字节来表示，其中第1个字节的值是254（作为这种情况的一个标记），而后面4个字节组成一个整型值，来真正存储前一个数据项的占用字节数。

3）为什么没有255的情况呢？
这是因为 255 已经定义为ziplist结束标记\<zlend\>的值了。在ziplist的很多操作的实现中，都会根据数据项的第1个字节是不是255来判断当前是不是到达ziplist的结尾了，因此一个正常的数据的第1个字节（也就是\<prevrawlen\>的第1个字节）是不能够取255这个值的，否则就冲突了。

4）\<len\>字段根据第1个字节的不同，总共分为9种情况（下面的表示法是按二进制表示）：
- |00pppppp| - 1 byte。第1个字节最高两个bit是00，那么\<len\>字段只有1个字节，剩余的6个bit用来表示长度值，最高可以表示63 (2^6-1)。
- |01pppppp|qqqqqqqq| - 2 bytes。第1个字节最高两个bit是01，那么\<len\>字段占2个字节，总共有14个bit用来表示长度值，最高可以表示16383 (2^14-1)。
- |10__|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes。第1个字节最高两个bit是10，那么len字段占5个字节，总共使用32个bit来表示长度值（6个bit舍弃不用），最高可以表示2^32-1。需要注意的是：在前三种情况下，\<data\>都是按字符串来存储的；从下面第4种情况开始，\<data\>开始变为按整数来存储了。
- |11000000| - 1 byte。\<len\>字段占用1个字节，值为0xC0，后面的数据\<data\>存储为2个字节的int16_t类型。
- |11010000| - 1 byte。\<len\>字段占用1个字节，值为0xD0，后面的数据<data>存储为4个字节的int32_t类型。
- |11100000| - 1 byte。\<len\>字段占用1个字节，值为0xE0，后面的数据<data>存储为8个字节的int64_t类型。
- |11110000| - 1 byte。\<len\>字段占用1个字节，值为0xF0，后面的数据\<data\>存储为3个字节长的整数。
- |11111110| - 1 byte。\<len\>字段占用1个字节，值为0xFE，后面的数据\<data\>存储为1个字节的整数。
- |1111xxxx| - - (xxxx的值在0001和1101之间)。这是一种特殊情况，xxxx从1到13一共13个值，这时就用这13个值来表示真正的数据。注意，这里是表示真正的数据，而不是数据长度了。也就是说，在这种情况下，后面不再需要一个单独的\<data\>字段来表示真正的数据了，而是\<len\>和\<data\>合二为一了。另外，由于xxxx只能取0001和1101这13个值了（其它可能的值和其它情况冲突了，比如0000和1110分别同前面第7种第8种情况冲突，1111跟结束标记冲突），而小数值应该从0开始，因此这13个值分别表示0到12，即xxxx的值减去1才是它所要表示的那个整数数据的值。
  
##### 2.1.3 例子
![ziplist存储示例](../../images/ziplist%E5%AD%98%E5%82%A8.png)
1）这个ziplist里存了4个数据项，分别为：
- 字符串: “name”
- 字符串: “tielei”
- 字符串: “age”
- 整数: 20

2）逐项解读：
- 这个ziplist一共包含33个字节。字节编号从byte[0]到byte[32]。图中每个字节的值使用16进制表示。
- 头4个字节（0x21000000）是按小端（little endian）模式存储的\<zlbytes\>字段。什么是小端呢？就是指数据的低字节保存在内存的低地址中（参见维基百科词条Endianness）。因此，这里\<zlbytes\>的值应该解析成0x00000021，用十进制表示正好就是33。
- 接下来4个字节（byte[4..7]）是\<zltail\>，用小端存储模式来解释，它的值是0x0000001D（值为29），表示最后一个数据项在byte[29]的位置（那个数据项为0x05FE14）。
- 再接下来2个字节（byte[8..9]），值为0x0004，表示这个ziplist里一共存有4项数据。
- 接下来6个字节（byte[10..15]）是第1个数据项。其中，prevrawlen=0，因为它前面没有数据项；len=4，相当于前面定义的9种情况中的第1种，表示后面4个字节按字符串存储数据，数据的值为”name”。
- 接下来8个字节（byte[16..23]）是第2个数据项，与前面数据项存储格式类似，存储1个字符串”tielei”。
- 接下来5个字节（byte[24..28]）是第3个数据项，与前面数据项存储格式类似，存储1个字符串”age”。
- 接下来3个字节（byte[29..31]）是最后一个数据项，它的格式与前面的数据项存储格式不太一样。其中，第1个字节prevrawlen=5，表示前一个数据项占用5个字节；第2个字节=FE，相当于前面定义的9种情况中的第8种，所以后面还有1个字节用来表示真正的数据，并且以整数表示。它的值是20（0x14）。
最后1个字节（byte[32]）表示<zlend>，是固定的值255（0xFF）。
#### 2.2 ziplist 接口
##### 2.2.1 接口定义
- ziplist的数据类型，没有用自定义的struct之类的来表达，而就是简单的unsigned char *。这是因为ziplist本质上就是一块连续内存，内部组成结构又是一个高度动态的设计（变长编码），也没法用一个固定的数据结构来表达。
- ziplistNew: 创建一个空的ziplist（只包含\<zlbytes\>\<zltail\>\<zllen\>\<zlend\>）。
- ziplistMerge: 将两个ziplist合并成一个新的ziplist。
- ziplistPush: 在ziplist的头部或尾端插入一段数据（产生一个新的数据项）。注意一下这个接口的返回值，是一个新的ziplist。调用方必须用这里返回的新的ziplist，替换之前传进来的旧的ziplist变量，而经过这个函数处理之后，原来旧的ziplist变量就失效了。为什么一个简单的插入操作会导致产生一个新的ziplist呢？这是因为ziplist是一块连续空间，对它的追加操作，会引发内存的realloc，因此ziplist的内存位置可能会发生变化。实际上，我们在之前介绍sds的文章中提到过类似这种接口使用模式（参见sdscatlen函数的说明）。
- ziplistIndex: 返回index参数指定的数据项的内存位置。index可以是负数，表示从尾端向前进行索引。
- ziplistNext和ziplistPrev分别返回一个ziplist中指定数据项p的后一项和前一项。
- ziplistInsert: 在ziplist的任意数据项前面插入一个新的数据项。
- ziplistDelete: 删除指定的数据项。
- ziplistFind: 查找给定的数据（由vstr和vlen指定）。注意它有一个skip参数，表示查找的时候每次比较之间要跳过几个数据项。为什么会有这么一个参数呢？其实这个参数的主要用途是当用ziplist表示hash结构的时候，是按照一个field，一个value来依次存入ziplist的。也就是说，偶数索引的数据项存field，奇数索引的数据项存value。当按照field的值进行查找的时候，就需要把奇数项跳过去。
- ziplistLen: 计算ziplist的长度（即包含数据项的个数）。
```
unsigned char *ziplistNew(void);
unsigned char *ziplistMerge(unsigned char **first, unsigned char **second);
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where);
unsigned char *ziplistIndex(unsigned char *zl, int index);
unsigned char *ziplistNext(unsigned char *zl, unsigned char *p);
unsigned char *ziplistPrev(unsigned char *zl, unsigned char *p);
unsigned int ziplistGet(unsigned char *p, unsigned char **sval, unsigned int *slen, long long *lval);
unsigned char *ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen);
unsigned char *ziplistDelete(unsigned char *zl, unsigned char **p);
unsigned char *ziplistDeleteRange(unsigned char *zl, int index, unsigned int num);
unsigned char *ziplistReplace(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen);
unsigned int ziplistCompare(unsigned char *p, unsigned char *s, unsigned int slen);
unsigned char *ziplistFind(unsigned char *zl, unsigned char *p, unsigned char *vstr, unsigned int vlen, unsigned int skip);
unsigned int ziplistLen(unsigned char *zl);
size_t ziplistBlobLen(unsigned char *zl);
void ziplistRepr(unsigned char *zl);
typedef int (*ziplistValidateEntryCB)(unsigned char* p, unsigned int head_count, void* userdata);
int ziplistValidateIntegrity(unsigned char *zl, size_t size, int deep,
                             ziplistValidateEntryCB entry_cb, void *cb_userdata);
void ziplistRandomPair(unsigned char *zl, unsigned long total_count, ziplistEntry *key, ziplistEntry *val);
void ziplistRandomPairs(unsigned char *zl, unsigned int count, ziplistEntry *keys, ziplistEntry *vals);
unsigned int ziplistRandomPairsUnique(unsigned char *zl, unsigned int count, ziplistEntry *keys, ziplistEntry *vals);
int ziplistSafeToAdd(unsigned char* zl, size_t add);
```
##### 2.2.2 插入接口
ziplistPush和ziplistInsert都是插入，只是对于插入位置的限定不同。它们在内部实现都依赖一个名为__ziplistInsert的内部函数，其代码如下（出自ziplist.c）:
- 这个函数是在指定的位置p插入一段新的数据，待插入数据的地址指针是s，长度为slen。插入后形成一个新的数据项，占据原来p的配置，原来位于p位置的数据项以及后面的所有数据项，需要统一向后移动，给新插入的数据项留出空间。参数p指向的是ziplist中某一个数据项的起始位置，或者在向尾端插入的时候，它指向ziplist的结束标记\<zlend\>。
- 函数开始先计算出待插入位置前一个数据项的长度prevlen。这个长度要存入新插入的数据项的\<prevrawlen\>字段。
然后计算当前数据项占用的总字节数reqlen，它包含三部分：\<prevrawlen\>, \<len\>和真正的数据。其中的数据部分会通过调用zipTryEncoding先来尝试转成整数。
- 由于插入导致的ziplist对于内存的新增需求，除了待插入数据项占用的reqlen之外，还要考虑原来p位置的数据项（现在要排在待插入数据项之后）的\<prevrawlen\>字段的变化。本来它保存的是前一项的总长度，现在变成了保存当前插入的数据项的总长度。这样它的\<prevrawlen\>字段本身需要的存储空间也可能发生变化，这个变化可能是变大也可能是变小。这个变化了多少的值nextdiff，是调用zipPrevLenByteDiff计算出来的。如果变大了，nextdiff是正值，否则是负值。
- 现在很容易算出来插入后新的ziplist需要多少字节了，然后调用ziplistResize来重新调整大小。ziplistResize的实现里会调用allocator的zrealloc，它有可能会造成数据拷贝。
- 现在额外的空间有了，接下来就是将原来p位置的数据项以及后面的所有数据都向后挪动，并为它设置新的\<prevrawlen\>字段。此外，还可能需要调整ziplist的\<zltail\>字段。
- 最后，组装新的待插入数据项，放在位置p。
```
/* Insert item at "p". */
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen, newlen;
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789; /* initialized to avoid warning. Using a value
                                    that is easy to see if for some reason
                                    we use it uninitialized. */
    zlentry tail;

    /* Find out prevlen for the entry that is inserted. */
    if (p[0] != ZIP_END) {
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else {
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            prevlen = zipRawEntryLengthSafe(zl, curlen, ptail);
        }
    }

    /* See if the entry can be encoded */
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        /* 'encoding' is set to the appropriate integer encoding */
        reqlen = zipIntSize(encoding);
    } else {
        /* 'encoding' is untouched, however zipStoreEntryEncoding will use the
         * string length to figure out how to encode it. */
        reqlen = slen;
    }
    /* We need space for both the length of the previous entry and
     * the length of the payload. */
    reqlen += zipStorePrevEntryLength(NULL,prevlen);
    reqlen += zipStoreEntryEncoding(NULL,encoding,slen);

    /* When the insert position is not equal to the tail, we need to
     * make sure that the next entry can hold this entry's length in
     * its prevlen field. */
    int forcelarge = 0;
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
    if (nextdiff == -4 && reqlen < 4) {
        nextdiff = 0;
        forcelarge = 1;
    }

    /* Store offset because a realloc may change the address of zl. */
    offset = p-zl;
    newlen = curlen+reqlen+nextdiff;
    zl = ziplistResize(zl,newlen);
    p = zl+offset;

    /* Apply memory move when necessary and update tail offset. */
    if (p[0] != ZIP_END) {
        /* Subtract one because of the ZIP_END bytes */
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        /* Encode this entry's raw length in the next entry. */
        if (forcelarge)
            zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
        else
            zipStorePrevEntryLength(p+reqlen,reqlen);

        /* Update offset for tail */
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        /* When the tail contains more than one entry, we need to take
         * "nextdiff" in account as well. Otherwise, a change in the
         * size of prevlen doesn't have an effect on the *tail* offset. */
        assert(zipEntrySafe(zl, newlen, p+reqlen, &tail, 1));
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        /* This element will be the new tail. */
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    /* When nextdiff != 0, the raw length of the next entry has changed, so
     * we need to cascade the update throughout the ziplist */
    if (nextdiff != 0) {
        offset = p-zl;
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    /* Write the entry */
    p += zipStorePrevEntryLength(p,prevlen);
    p += zipStoreEntryEncoding(p,encoding,slen);
    if (ZIP_IS_STR(encoding)) {
        memcpy(p,s,slen);
    } else {
        zipSaveInteger(p,value,encoding);
    }
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}
```
### 3. hash 与 ziplist
#### 3.1 实现
- 实际上，hash随着数据的增大，其底层数据结构的实现是会发生变化的，当然存储效率也就不同。在field比较少，各个value值也比较小的时候，hash采用ziplist来实现；
- 而随着field增多和value值增大，hash可能会变成dict来实现。当hash底层变成dict来实现的时候，它的存储效率就没法跟那些序列化方式相比了。

1）当我们为某个key第一次执行 hset key field value 命令的时候，Redis会创建一个hash结构，这个新创建的hash底层就是一个ziplist。
```
robj *createHashObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_HASH, zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}
```
createHashObject 负责创建一个新的hash结构。可以看出，它创建了一个type = OBJ_HASH但encoding = OBJ_ENCODING_ZIPLIST的robj对象。

每执行一次hset命令，插入的field和value分别作为一个新的数据项插入到ziplist中（即每次hset产生两个数据项）。

2）hash底层的这个ziplist就可能会转成dict。那么到底插入多少才会转呢？
```
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
```
在如下两个条件之一满足的时候，ziplist会转成dict：
- 当hash中的数据项（即field-value对）的数目超过512的时候，也就是ziplist数据项超过1024的时候（请参考t_hash.c中的hashTypeSet函数）。
- 当hash中插入的任意一个value的长度超过了64的时候（请参考t_hash.c中的hashTypeTryConversion函数）。
#### 3.2 ziplist 实现 hash 的缺点
当ziplist变得很大的时候，它有如下几个缺点：
- 每次插入或修改引发的realloc操作会有更大的概率造成内存拷贝，从而降低性能。
- 一旦发生内存拷贝，内存拷贝的成本也相应增加，因为要拷贝更大的一块数据。
- 当ziplist数据项过多的时候，在它上面查找指定的数据项就会性能变得很低，因为ziplist上的查找需要进行遍历。

### 4. 相关链接
1）[Redis内部数据结构详解(4)——ziplist](http://zhangtielei.com/posts/blog-redis-ziplist.html)