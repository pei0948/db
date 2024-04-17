## intset

### 1.intset 介绍
- intset 是一个由**整数**组成的**有序集合**，从而便于进行二分查找，用于快速地判断一个元素是否属于这个集合。
- 它在内存分配上与ziplist有些类似，是连续的**一整块内存空间**
- 对于大整数和小整数（按绝对值）采取了不同的**编码**，尽量对内存的使用进行了优化。
### 2.intset 实现
#### 2.1 intset 定义
**1）介绍**
- encoding: 数据编码，表示intset中的每个数据元素用几个字节来存储。它有三种可能的取值：**INTSET_ENC_INT16** 表示每个元素用2个字节存储，**INTSET_ENC_INT32** 表示每个元素用4个字节存储，**INTSET_ENC_INT64** 表示每个元素用8个字节存储。因此，intset中存储的整数最多只能占用64bit。
- length: 表示intset中的元素个数。encoding和length两个字段构成了intset的头部（header）。
- contents: 是一个柔性数组（flexible array member），表示intset的header后面紧跟着数据元素。这个数组的总长度（即总字节数）等于encoding * length。柔性数组在Redis的很多数据结构的定义中都出现过（例如sds, quicklist, skiplist），用于表达一个偏移量。contents需要单独为其分配空间，这部分内存不包含在intset结构当中。

intset可能会随着数据的添加而改变它的数据编码：
- 最开始，新创建的intset使用占内存最小的INTSET_ENC_INT16（值为2）作为数据编码。
- 每添加一个新元素，则根据元素大小决定是否对数据编码进行升级(INTSET_ENC_INT32、INTSET_ENC_INT64)。
```
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```
**2）例子**
- 新创建的intset只有一个header，总共8个字节。其中encoding = 2, length = 0。
- 添加13, 5两个元素之后，因为它们是比较小的整数，都能使用2个字节表示，所以encoding不变，值还是2。
- 当添加32768的时候，它不再能用2个字节来表示了（2个字节能表达的数据范围是-215~215-1，而32768等于215，超出范围了），因此encoding必须升级到INTSET_ENC_INT32（值为4），即用4个字节表示一个元素。
- 在添加每个元素的过程中，intset始终保持从小到大有序。
- 与ziplist类似，intset也是按小端（little endian）模式存储的（参见维基百科词条Endianness）。比如，在上图中intset添加完所有数据之后，表示encoding字段的4个字节应该解释成0x00000004，而第5个数据应该解释成0x000186A0 = 100000。
![intset 改变编码示例](<../../images/intset 改变编码示例.png>)
#### 2.2 intset 查找
- intsetFind在指定的intset中查找指定的元素value，找到返回1，没找到返回0。
- _intsetValueEncoding函数会根据要查找的value落在哪个范围而计算出相应的数据编码（即它应该用几个字节来存储）。
- 如果value所需的数据编码比当前intset的编码要大，则它肯定在当前intset所能存储的数据范围之外（特别大或特别小），所以这时会直接返回0；否则调用intsetSearch执行一个二分查找算法。
- intsetSearch在指定的intset中查找指定的元素value，如果找到，则返回1并且将参数pos指向找到的元素位置；如果没找到，则返回0并且将参数pos指向能插入该元素的位置。
intsetSearch是对于二分查找算法的一个实现，它大致分为三个部分：
  - 特殊处理intset为空的情况。
  - 特殊处理两个边界情况：当要查找的value比最后一个元素还要大或者比第一个元素还要小的时候。实际上，这两部分的特殊处理，在二分查找中并不是必须的，但它们在这里提供了特殊情况下快速失败的可能。
  - 真正执行二分查找过程。注意：如果最后没找到，插入位置在min指定的位置。
- 代码中出现的intrev32ifbe是为了在需要的时候做大小端转换的。前面我们提到过，intset里的数据是按小端（little endian）模式存储的，因此在大端（big endian）机器上运行时，这里的intrev32ifbe会做相应的转换。
- 这个查找算法的总的时间复杂度为O(log n)。

```
/* Determine whether a value belongs to this set */
uint8_t intsetFind(intset *is, int64_t value) {
    uint8_t valenc = _intsetValueEncoding(value);
    return valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,NULL);
}
```

```
/* Search for the position of "value". Return 1 when the value was found and
 * sets "pos" to the position of the value within the intset. Return 0 when
 * the value is not present in the intset and sets "pos" to the position
 * where "value" can be inserted. */
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    /* The value can never be found when the set is empty */
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        if (value > _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }

    while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```
#### 2.3 intset 添加
- intsetAdd在intset中添加新元素value。如果value在添加前已经存在，则不会重复添加，这时参数success被置为0；如果value在原来intset中不存在，则将value插入到适当位置，这时参数success被置为0。
- 如果要添加的元素value所需的数据编码比当前intset的编码要大，那么则调用intsetUpgradeAndAdd将intset的编码进行升级后再插入value。
- 调用intsetSearch，如果能查到，则不会重复添加。
- 如果没查到，则调用intsetResize对intset进行内存扩充，使得它能够容纳新添加的元素。因为intset是一块连续空间，因此这个操作会引发内存的realloc（参见http://man.cx/realloc）。这有可能带来一次数据拷贝。同时调用intsetMoveTail将待插入位置后面的元素统一向后移动1个位置，这也涉及到一次数据拷贝。值得注意的是，在intsetMoveTail中是调用memmove完成这次数据拷贝的。memmove保证了在拷贝过程中不会造成数据重叠或覆盖，具体参见http://man.cx/memmove。
- intsetUpgradeAndAdd的实现中也会调用intsetResize来完成内存扩充。在进行编码升级时，intsetUpgradeAndAdd的实现会把原来intset中的每个元素取出来，再用新的编码重新写入新的位置。
- 注意一下intsetAdd的返回值，它返回一个新的intset指针。它可能与传入的intset指针is相同，也可能不同。调用方必须用这里返回的新的intset，替换之前传进来的旧的intset变量。类似这种接口使用模式，在Redis的实现代码中是很常见的，比如我们之前在介绍sds和ziplist的时候都碰到过类似的情况。

```
/* Insert an integer in the intset */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found. */
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1);
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    _intsetSet(is,pos,value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```
### 3.intset 与 ziplist 对比
- ziplist可以存储任意二进制串，而intset只能存储整数。
- ziplist是无序的，而intset是从小到大有序的。因此，在ziplist上查找只能遍历，而在intset上可以进行二分查找，性能更高。
- ziplist可以对每个数据项进行不同的变长编码（每个数据项前面都有数据长度字段len），而intset只能整体使用一个统一的编码（encoding）。
