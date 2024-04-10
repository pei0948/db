## <h3 id="redis_data_structure_3">哈希对象(hash)</h3>
> 哈希对象的编码可以是 OBJ_ENCODING_ZIPLIST 和 OBJ_ENCODING_HT 之一。
### 1 底层实现
#### 1.1 OBJ_ENCODING_ZIPLIST 编码
ziplist编码的哈希对象底层实现是压缩列表，在ziplist编码的哈希对象中，key-value键值对是以紧密相连的方式放入压缩链表的，先把key放入表尾，再放入value；键值对总是向表尾添加。<br/>
![image](https://user-images.githubusercontent.com/87458342/132521588-ec193f11-3ee4-41b2-b9a4-7f30405d4977.png)

#### 1.2 OBJ_ENCODING_HT 编码
- hashtable编码的哈希对象底层实现是字典，哈希对象中的每个key-value对都使用一个字典键值对来保存。
- 字典键值对即是，字典的键和值都是字符串对象，字典的键保存key-value的key，字典的值保存key-value的value。
![image](https://user-images.githubusercontent.com/87458342/132521715-b3cc011a-1f38-473b-821c-ee66364d82a5.png)

#### 1.3 哈希对象编码转换：
哈希对象使用ziplist编码需要满足两个条件：
- 一是所有键值对的键和值的字符串长度都小于64字节
- 二是键值对数量小于512个；不满足任意一个都使用hashtable编码
以上两个条件可以在Reids配置文件中修改hash-max-ziplist-value选项和hash-max-ziplist-entries选项。
 
### 2 应用场景：
存储、读取、修改对象属性，比如：用户（姓名、性别、爱好），文章（标题、发布时间、作者、内容）
 
