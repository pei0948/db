### 1 简介
sds全称是Simple Dynamic String，具有如下显著的特点：
- 可动态扩展内存。sds表示的字符串其内容可以修改，也可以追加。
- 采用预分配冗余空间的方式来减少内存的频繁分配，从而优化字符串的增长操作
- 二进制安全（Binary Safe）。sds能存储任意二进制数据，而不仅仅是可打印字符。
- 与传统的C语言字符串类型兼容。
### 2 实现
#### 2.1 存在 5 种定义
 sds一共有5种类型的header。之所以有5种，是为了能让不同长度的字符串可以使用不同大小的header。这样，短字符串就能使用较小的header，从而节省内存。
```
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
sds字符串的完整结构，由在内存地址上前后相邻的header和字符数组两部分组成：
- header：除了sdshdr5，一个header结构都包含3个字段：字符串的真正长度len、字符串的最大容量alloc和flags，flags总是占用一个字节。其中的最低3个bit用来表示header的类型。
- 字符数组：字符数组的长度等于最大容量+1，存放了真正有效的字符串数据。在真正的字符串数据之后，是空余未用的字节（一般以字节0填充），允许在不重新分配内存的前提下让字符串数据向后做有限的扩展。在真正的字符串数据之后，还有一个NULL结束符，即ASCII码为0的’\0’字符。这是为了和传统C字符串兼容。之所以字符数组的长度比最大容量多1个字节，就是为了在字符串长度达到最大容量时仍然有1个字节存放NULL结束符。
#### 2.2 sds 定义的优点
sds字符串的header，其实隐藏在真正的字符串数据的前面（低地址方向），这样的一个定义，有如下几个好处：
- header和数据相邻，从而不需要分成两块内存空间来单独分配，有利于减少内存碎片，提高存储效率
- 虽然header有多个类型，但sds可以用统一的char *来表达。且它与传统的C语言字符串保持类型兼容。如果一个sds里面存储的是可打印字符串，那么我们可以直接把它传给C函数，比如使用strcmp比较字符串大小，或者使用printf进行打印。

#### 2.3 编解码过程
##### 2.3.1 编码过程
- 当我们执行Redis的set命令的时候，Redis首先将接收到的value值（string类型）表示成一个type = OBJ_STRING并且encoding = OBJ_ENCODING_RAW的robj对象，然后在存入内部存储之前先执行一个编码过程，试图将它表示成另一种更节省内存的encoding方式。这一过程的核心代码，是object.c中的tryObjectEncoding函数。
##### 2.3.2 解码过程
- 当我们需要获取字符串的值，比如执行get命令的时候，我们需要执行与前面讲的编码过程相反的操作——解码。这一解码过程的核心代码，是object.c中的getDecodedObject函数。

### 3 使用 sds 的优点
- 性能高：
sds 数据结构主要由 len、alloc、buf[] 三个属性组成，其中 buf[] 为实际保存字符串的 char 类型数组；len 表示 buf[] 数组所保存的字符串的长度。由于使用 len 记录了保存的字符串长度，所以在获取字符串长度的时候，不需要从前往后遍历数组，直接获取 len 的值就可以了

- 内存预分配，优化字符串的增长操作：
当需要修改数据时，首先会检查 sds 空间 len 是否满足，不满足则自动扩容空间，然后再执行修改。sds 在分配空间时，还会分配额外的未使用空间 free，下次再修改就先检查未使用空间 free 是否满足，如果满足则不用再扩展空间。通过空间预分配策略，有效较少在字符串连续增长情况下产生的内存重分配次数。
> 额外分配 free 空间的规则：
> - 如果 sds 字符串修改后，len 值小于 1M，则额外分配未使用空间 free 的大小为 len 
> - 如果 sds 字符串修改后，len 值大于等于 1M，则额外分配未使用空间 free 的大小为1M
- 惰性空间回收，优化字符串的缩短操作：
当缩短 SDS 字符串后，并不会立即执行内存重分配来回收多余的空间，如果后续有增长操作，则可直接使用。