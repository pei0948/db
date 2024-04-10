## <h3 id="redis_data_structure_1">字符串对象(string)</h3>
> - String的内部存储结构一般是sds（Simple Dynamic String），但是如果一个String类型的value的值是数字，那么Redis内部会把它转成long类型来存储，从而减少内存的使用。
### 1 底层实现
- 确切地说，String在Redis中是用一个robj来表示的。
- 用来表示String的robj可能编码成3种内部表示：OBJ_ENCODING_RAW，OBJ_ENCODING_EMBSTR，OBJ_ENCODING_INT。其中前两种编码使用的是sds来存储，最后一种OBJ_ENCODING_INT 编码直接把string存成了long型。
- 在对string进行incr, decr等操作的时候，如果它内部是OBJ_ENCODING_INT编码，那么可以直接进行加减操作；如果它内部是OBJ_ENCODING_RAW或OBJ_ENCODING_EMBSTR编码，那么Redis会先试图把sds存储的字符串转成long型，如果能转成功，再进行加减操作。
- 对一个内部表示成long型的string执行append, setbit, getrange这些命令，针对的仍然是string的值（即十进制表示的字符串），而不是针对内部表示的long型进行操作。比如字符串”32”，如果按照字符数组来解释，它包含两个字符，它们的ASCII码分别是0x33和0x32。当我们执行命令setbit key 7 0的时候，相当于把字符0x33变成了0x32，这样字符串的值就变成了”22”。而如果将字符串”32”按照内部的64位long型来解释，那么它是0x0000000000000020，在这个基础上执行setbit位操作，结果就完全不对了。因此，在这些命令的实现中，会把long型先转成字符串再进行相应的操作。
#### 1.1 OBJ_ENCODING_INT 编码
字符串保存的是整数值，并且这个正式可以用long类型来表示，那么其就会直接保存在redisObject的ptr属性里，并将编码设置为int，如图：<br/>
![image](https://user-images.githubusercontent.com/87458342/132519704-605f9566-20c2-45c4-b5a3-23faad04d111.png)

#### 1.2 OBJ_ENCODING_RAW 编码
字符串保存的大于32字节的字符串值，则使用简单动态字符串（SDS）结构，并将编码设置为raw，此时内存结构与SDS结构一致，内存分配次数为两次，创建redisObject对象和sdshdr结构，如图：<br/>
![image](https://user-images.githubusercontent.com/87458342/132519802-72780b33-00a3-440a-a036-169675c1a079.png)

#### 1.3 OBJ_ENCODING_EMBSTR 编码
字符串保存的小于等于32字节的字符串值，使用的也是简单的动态字符串（SDS结构），但是内存结构做了优化，用于保存顿消的字符串；内存分配也只需要一次就可完成，分配一块连续的空间即可，如图：<br/>
![image](https://user-images.githubusercontent.com/87458342/132519975-152ef3c0-f2e8-4bdb-94f0-15d80070b8d1.png)

### 2 字符串对象总结：
   * 在Redis中，存储long、double类型的浮点数是先转换为字符串再进行存储的。
   * raw与embstr编码效果是相同的，不同在于内存分配与释放，raw两次，embstr一次。
   * embstr内存块连续，能更好的利用缓存在来的优势
   * int编码和embstr编码如果做追加字符串等操作，满足条件下会被转换为raw编码；embstr编码的对象是只读的，一旦修改会先转码到raw。
 
### 3 应用场景：
   1. 访问量统计：每次访问博客和文章使用 INCR 命令进行递增。
   2. 一般做一些复杂的技术功能的缓存。
