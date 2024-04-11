## quicklist
- [quicklist](#quicklist)
  - [1 quicklist 简介](#1-quicklist-简介)
    - [1.1 quicklist 是什么？](#11-quicklist-是什么)
  - [2 quicklist 实现](#2-quicklist-实现)
    - [2.1 quicklist 结构](#21-quicklist-结构)
      - [2.1.1 quicklist 示例](#211-quicklist-示例)
      - [2.1.2 quicklist 结构体](#212-quicklist-结构体)
      - [2.1.3 quicklistNode 结构体](#213-quicklistnode-结构体)
      - [2.1.4 quicklistLZF 结构体](#214-quicklistlzf-结构体)
    - [2.2 创建](#22-创建)
    - [2.3 增加](#23-增加)
      - [2.3.1 push](#231-push)
      - [2.3.2 任意位置插入](#232-任意位置插入)
    - [2.4 删除](#24-删除)
  - [3 quicklist 配置](#3-quicklist-配置)
    - [3.1 配置问题](#31-配置问题)
    - [3.2 配置内容](#32-配置内容)
      - [3.2.1 list-max-ziplist-size](#321-list-max-ziplist-size)
      - [3.2.2 list-compress-depth](#322-list-compress-depth)
  - [4 相关链接](#4-相关链接)

### 1 quicklist 简介
#### 1.1 quicklist 是什么？
> 头部和尾部插入或删除数据，时间复杂度为 O(1)
> 在任意中间位置的存取操作，需要遍历，时间复杂度为 O(n)
quicklist是一个基于ziplist的双向链表，quicklist的每个节点都是一个ziplist，比如，一个包含3个节点的quicklist，如果每个节点的ziplist又包含4个数据项，那么对外表现上，这个list就总共包含12个数据项。quicklist的结构为什么这样设计呢？总结起来，大概又是一个空间和时间的折中：
- 双向链表便于在表的两端进行push和pop操作，但是它的内存开销比较大。首先，它在每个节点上除了要保存数据之外，还要额外保存两个指针；其次，双向链表的各个节点是单独的内存块，地址不连续，节点多了容易产生内存碎片。
- ziplist由于是一整块连续内存，所以存储效率很高。但是，它不利于修改操作，每次数据变动都会引发一次内存的realloc。特别是当ziplist长度很长的时候，一次realloc可能会导致大批量的数据拷贝，进一步降低性能。
### 2 quicklist 实现
#### 2.1 quicklist 结构
##### 2.1.1 quicklist 示例
![quicklist 结构](../../images/quicklist%E7%BB%93%E6%9E%84.png)
图中例子对应的ziplist大小配置和节点压缩深度配置:
```
list-max-ziplist-size 3
list-compress-depth 2
```
注意：
- 两端各有2个橙黄色的节点，是没有被压缩的。它们的数据指针zl指向真正的ziplist。中间的其它节点是被压缩过的，它们的数据指针zl指向被压缩后的ziplist结构，即一个quicklistLZF结构。
- 左侧头节点上的ziplist里有2项数据，右侧尾节点上的ziplist里有1项数据，中间其它节点上的ziplist里都有3项数据（包括压缩的节点内部）。这表示在表的两端执行过多次push和pop操作后的一个状态。
##### 2.1.2 quicklist 结构体
- head: 指向头节点（左侧第一个节点）的指针。
- tail: 指向尾节点（右侧第一个节点）的指针。
- count: 所有ziplist数据项的个数总和。
- len: quicklist节点的个数。
- fill: 16bit，ziplist大小设置，存放list-max-ziplist-size参数的值。
- compress: 16bit，节点压缩深度设置，存放list-compress-depth参数的值。
```
/* quicklist is a 40 byte struct (on 64-bit systems) describing a quicklist.
 * 'count' is the number of total entries.
 * 'len' is the number of quicklist nodes.
 * 'compress' is: 0 if compression disabled, otherwise it's the number
 *                of quicklistNodes to leave uncompressed at ends of quicklist.
 * 'fill' is the user-requested (or default) fill factor.
 * 'bookmarks are an optional feature that is used by realloc this struct,
 *      so that they don't consume memory when not used. */
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all listpacks */
    unsigned long len;          /* number of quicklistNodes */
    signed int fill : QL_FILL_BITS;       /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;
```

##### 2.1.3 quicklistNode 结构体
1）字段：
- prev: 指向链表前一个节点的指针。
- next: 指向链表后一个节点的指针。
- entry: 数据指针。如果当前节点的数据没有压缩，那么它指向一个ziplist结构；否则，它指向一个quicklistLZF结构。
- sz: 表示zl指向的ziplist的总大小（包括zlbytes, zltail, zllen, zlend和各个数据项）。需要注意的是：如果ziplist被压缩了，那么这个sz的值仍然是压缩前的ziplist大小。
- count: 表示ziplist里面包含的数据项个数。这个字段只有16bit。稍后我们会一起计算一下这16bit是否够用。
- encoding: 表示ziplist是否压缩了（以及用了哪个压缩算法）。目前只有两种取值：2表示被压缩了（而且用的是LZF压缩算法），1表示没有压缩。
- container: 是一个预留字段。本来设计是用来表明一个quicklist节点下面是直接存数据，还是使用ziplist存数据，或者用其它的结构来存数据（用作一个数据容器，所以叫container）。但是，在目前的实现中，这个值是一个固定的值2，表示使用ziplist作为数据容器。
- recompress: 当我们使用类似lindex这样的命令查看了某一项本来压缩的数据时，需要把数据暂时解压，这时就设置recompress=1做一个标记，等有机会再把数据重新压缩。
- attempted_compress: 这个值只对Redis的自动化测试程序有用。我们不用管它。
- extra: 其它扩展字段。目前Redis的实现里也没用上。

```
/* quicklistNode is a 32 byte struct describing a listpack for a quicklist.
 * We use bit fields keep the quicklistNode at 32 bytes.
 * count: 16 bits, max 65536 (max lp bytes is 65k, so max count actually < 32k).
 * encoding: 2 bits, RAW=1, LZF=2.
 * container: 2 bits, PLAIN=1 (a single item as char array), PACKED=2 (listpack with multiple items).
 * recompress: 1 bit, bool, true if node is temporary decompressed for usage.
 * attempted_compress: 1 bit, boolean, used for verifying during testing.
 * extra: 10 bits, free for future use; pads out the remainder of 32 bits */
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *entry;
    size_t sz;             /* entry size in bytes */
    unsigned int count : 16;     /* count of items in listpack */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* PLAIN==1 or PACKED==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int dont_compress : 1; /* prevent compression of entry that will be used later */
    unsigned int extra : 9; /* more bits to steal for future usage */
} quicklistNode;
```

2）count字段这16bit是否够用？
ziplist大小受到list-max-ziplist-size参数的限制。按照正值和负值有两种情况：
- 当这个参数取正值的时候，就是恰好表示一个quicklistNode结构中zl所指向的ziplist所包含的数据项的最大值。list-max-ziplist-size参数是由quicklist结构的fill字段来存储的，而fill字段是16bit，所以它所能表达的值能够用16bit来表示。
- 当这个参数取负值的时候，能够表示的ziplist最大长度是64 Kb。而ziplist中每一个数据项，最少需要2个字节来表示：1个字节的prevrawlen，1个字节的data（len字段和data合二为一；详见上一篇）。所以，ziplist中数据项的个数不会超过32 K，用16bit来表达足够了。
##### 2.1.4 quicklistLZF 结构体

quicklistLZF结构表示一个被压缩过的ziplist。其中：
- sz: 表示压缩后的ziplist大小。
- compressed: 是个柔性数组（flexible array member），存放压缩后的ziplist字节数组。
```
/* quicklistLZF is a 8+N byte struct holding 'sz' followed by 'compressed'.
 * 'sz' is byte length of 'compressed' field.
 * 'compressed' is LZF data with total (compressed) length 'sz'
 * NOTE: uncompressed length is stored in quicklistNode->sz.
 * When quicklistNode->entry is compressed, node->entry points to a quicklistLZF */
typedef struct quicklistLZF {
    size_t sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;
```
#### 2.2 创建
- 当我们使用lpush或rpush命令第一次向一个不存在的list里面插入数据的时候，Redis会首先调用quicklistCreate接口创建一个空的quicklist。
- quicklist是一个不包含空余头节点的双向链表（head和tail都初始化为NULL）。
```
/* Create a new quicklist.
 * Free with quicklistRelease(). */
quicklist *quicklistCreate(void) {
    struct quicklist *quicklist;

    quicklist = zmalloc(sizeof(*quicklist));
    quicklist->head = quicklist->tail = NULL;
    quicklist->len = 0;
    quicklist->count = 0;
    quicklist->compress = 0;
    quicklist->fill = -2;
    quicklist->bookmark_count = 0;
    return quicklist;
}
```
#### 2.3 增加
##### 2.3.1 push
不管是在头部还是尾部插入数据，都包含两种情况：
- 如果头节点（或尾节点）上ziplist大小没有超过限制（即_quicklistNodeAllowInsert返回1），那么新数据被直接插入到ziplist中（调用ziplistPush）。
- 如果头节点（或尾节点）上ziplist太大了，那么新创建一个quicklistNode节点（对应地也会新创建一个ziplist），然后把这个新创建的节点插入到quicklist双向链表中（调用_quicklistInsertNodeAfter）。

1）quicklistPush
```
/* Wrapper to allow argument-based switching between HEAD/TAIL pop */
void quicklistPush(quicklist *quicklist, void *value, const size_t sz,
                   int where) {
    /* The head and tail should never be compressed (we don't attempt to decompress them) */
    if (quicklist->head)
        assert(quicklist->head->encoding != QUICKLIST_NODE_ENCODING_LZF);
    if (quicklist->tail)
        assert(quicklist->tail->encoding != QUICKLIST_NODE_ENCODING_LZF);

    if (where == QUICKLIST_HEAD) {
        quicklistPushHead(quicklist, value, sz);
    } else if (where == QUICKLIST_TAIL) {
        quicklistPushTail(quicklist, value, sz);
    }
}
```
2）quicklistPushHead
```
/* Add new entry to head node of quicklist.
 *
 * Returns 0 if used existing head.
 * Returns 1 if new head created. */
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist->head;

    if (unlikely(isLargeElement(sz))) {
        __quicklistInsertPlainNode(quicklist, quicklist->head, value, sz, 0);
        return 1;
    }

    if (likely(
            _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
        quicklist->head->entry = lpPrepend(quicklist->head->entry, value, sz);
        quicklistNodeUpdateSz(quicklist->head);
    } else {
        quicklistNode *node = quicklistCreateNode();
        node->entry = lpPrepend(lpNew(0), value, sz);

        quicklistNodeUpdateSz(node);
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
    }
    quicklist->count++;
    quicklist->head->count++;
    return (orig_head != quicklist->head);
}
```
3）quicklistPushTail
```
/* Add new entry to tail node of quicklist.
 *
 * Returns 0 if used existing tail.
 * Returns 1 if new tail created. */
int quicklistPushTail(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_tail = quicklist->tail;
    if (unlikely(isLargeElement(sz))) {
        __quicklistInsertPlainNode(quicklist, quicklist->tail, value, sz, 1);
        return 1;
    }

    if (likely(
            _quicklistNodeAllowInsert(quicklist->tail, quicklist->fill, sz))) {
        quicklist->tail->entry = lpAppend(quicklist->tail->entry, value, sz);
        quicklistNodeUpdateSz(quicklist->tail);
    } else {
        quicklistNode *node = quicklistCreateNode();
        node->entry = lpAppend(lpNew(0), value, sz);

        quicklistNodeUpdateSz(node);
        _quicklistInsertNodeAfter(quicklist, quicklist->tail, node);
    }
    quicklist->count++;
    quicklist->tail->count++;
    return (orig_tail != quicklist->tail);
}
```
##### 2.3.2 任意位置插入
quicklist不仅实现了从头部或尾部插入，也实现了从任意指定的位置插入。quicklistInsertAfter和quicklistInsertBefore就是分别在指定位置后面和前面插入数据项。这种在任意指定位置插入数据的操作，情况比较复杂，有众多的逻辑分支：

- 当插入位置所在的ziplist大小没有超过限制时，直接插入到ziplist中就好了；
- 当插入位置所在的ziplist大小超过了限制，但插入的位置位于ziplist两端，并且相邻的quicklist链表节点的ziplist大小没有超过限制，那么就转而插入到相邻的那个quicklist链表节点的ziplist中；
- 当插入位置所在的ziplist大小超过了限制，但插入的位置位于ziplist两端，并且相邻的quicklist链表节点的ziplist大小也超过限制，这时需要新创建一个quicklist链表节点插入。
- 对于插入位置所在的ziplist大小超过了限制的其它情况（主要对应于在ziplist中间插入数据的情况），则需要把当前ziplist分裂为两个节点，然后再其中一个节点上插入数据。

#### 2.4 删除

quicklist的pop操作是调用quicklistPopCustom来实现的。quicklistPopCustom的实现过程基本上跟quicklistPush相反，先从头部或尾部节点的ziplist中把对应的数据项删除，如果在删除后ziplist为空了，那么对应的头部或尾部节点也要删除。删除后还可能涉及到里面节点的解压缩问题。

### 3 quicklist 配置
#### 3.1 配置问题
不过，这也带来了一个新问题：到底一个quicklist节点包含多长的ziplist合适呢？比如，同样是存储12个数据项，既可以是一个quicklist包含3个节点，而每个节点的ziplist又包含4个数据项，也可以是一个quicklist包含6个节点，而每个节点的ziplist又包含2个数据项。

这又是一个需要找平衡点的难题。我们只从存储效率上分析一下：
- 每个quicklist节点上的ziplist越短，则内存碎片越多。内存碎片多了，有可能在内存中产生很多无法被利用的小碎片，从而降低存储效率。这种情况的极端是每个quicklist节点上的ziplist只包含一个数据项，这就蜕化成一个普通的双向链表了。
- 每个quicklist节点上的ziplist越长，则为ziplist分配大块连续内存空间的难度就越大。有可能出现内存里有很多小块的空闲空间（它们加起来很多），但却找不到一块足够大的空闲空间分配给ziplist的情况。这同样会降低存储效率。这种情况的极端是整个quicklist只有一个节点，所有的数据项都分配在这仅有的一个节点的ziplist里面。这其实蜕化成一个ziplist了
#### 3.2 配置内容
##### 3.2.1 list-max-ziplist-size
可见，一个quicklist节点上的ziplist要保持一个合理的长度。那到底多长合理呢？这可能取决于具体应用场景。实际上，Redis提供了一个配置参数list-max-ziplist-size，就是为了让使用者可以来根据自己的情况进行调整。
```
list-max-ziplist-size -2
```
这个参数可以取正值，也可以取负值:

- 当取正值的时候，表示按照数据项个数来限定每个quicklist节点上的ziplist长度。比如，当这个参数配置成5的时候，表示每个quicklist节点的ziplist最多包含5个数据项。
- 当取负值的时候，表示按照占用字节数来限定每个quicklist节点上的ziplist长度。这时，它只能取-1到-5这五个值，每个值含义如下：
  - -5: 每个quicklist节点上的ziplist大小不能超过64 Kb。（注：1kb => 1024 bytes）
  - -4: 每个quicklist节点上的ziplist大小不能超过32 Kb。
  - -3: 每个quicklist节点上的ziplist大小不能超过16 Kb。
  - -2: 每个quicklist节点上的ziplist大小不能超过8 Kb。（-2是Redis给出的默认值）
  - -1: 每个quicklist节点上的ziplist大小不能超过4 Kb。

##### 3.2.2 list-compress-depth
> 这个参数表示一个quicklist两端不被压缩的节点个数
当列表很长的时候，最容易被访问的很可能是两端的数据，中间的数据被访问的频率比较低（访问起来性能也很低）。如果应用场景符合这个特点，那么list还提供了一个选项，能够把中间的数据节点进行压缩，从而进一步节省内存空间。Redis的配置参数list-compress-depth就是用来完成这个设置的。
```
list-compress-depth 0
```
注：这里的节点个数是指quicklist双向链表的节点个数，而不是指ziplist里面的数据项个数。实际上，一个quicklist节点上的ziplist，如果被压缩，就是整体被压缩的。

取值：
- 0: 是个特殊值，表示都不压缩。这是Redis的默认值。
- 1: 表示quicklist两端各有1个节点不压缩，中间的节点压缩。
- 2: 表示quicklist两端各有2个节点不压缩，中间的节点压缩。
- 3: 表示quicklist两端各有3个节点不压缩，中间的节点压缩。
- 依此类推…

Redis对于quicklist内部节点的压缩算法，采用的 [LZF](http://oldhome.schmorp.de/marc/liblzf.html)一种无损压缩算法。
### 4 相关链接
1）[Redis内部数据结构详解(5)——quicklist](http://zhangtielei.com/posts/blog-redis-quicklist.html) 