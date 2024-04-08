## Redis五种数据类型

### [字符串对象(string)](https://github.com/pei0948/db/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/string.md)

### [列表对象(list)](https://github.com/pei0948/db/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/list.md)

### [哈希对象(hash)](https://github.com/pei0948/db/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/hash.md)

### [集合对象(set)](https://github.com/pei0948/db/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/set.md)

### [有序集合对象(zset)](https://github.com/pei0948/db/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/zset.md)

### <h3 id="redis_data_structure_6">总结</h3>

在Redis的五大数据对象中，string对象是唯一个可以被其他四种数据对象作为内嵌对象的；<br/>
列表（list）、哈希（hash）、集合（set）、有序集合（zset）底层实现都用到了压缩列表结构，并且使用压缩列表结构的条件都是在元素个数比较少、字节长度较短的情况下；<br/>

四种数据对象使用压缩列表的优点：
* 1. 节约内存，减少内存开销，Redis是内存型数据库，所以一定情况下减少内存开销是非常有必要的。
* 2. 减少内存碎片，压缩列表的内存块是连续的，并分配内存的次数一次即可。
* 3. 压缩列表的新增、删除、查找操作的平均时间复杂度是O(N)，在N再一定的范围内，这个时间几乎是可以忽略的，并且N的上限值是可以配置的。
* 4. 四种数据对象都有两种编码结构，灵活性增加。
