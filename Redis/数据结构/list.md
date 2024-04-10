## <h3 id="redis_data_structure_2">列表对象(list)</h3>

列表对象的编码可以是ziplist和linkedlist之一。

* 1. ziplist编码
ziplist编码的列表对象底层实现是压缩列表，每个压缩列表节点保存了一个列表元素。<br/>
![image](https://user-images.githubusercontent.com/87458342/132521213-ac18414b-b7d4-4d29-9368-f5b267395a01.png)
 
* 2. linkedlist编码
linkedlist编码底层采用双端链表实现，每个双端链表节点都保存了一个字符串对象，在每个字符串对象内保存了一个列表元素。<br/>
![image](https://user-images.githubusercontent.com/87458342/132521303-9a8b7ea6-2995-4e9e-85ad-d9692e4427fc.png)
 
* 列表对象编码转换：
1. 列表对象使用ziplist编码需要满足两个条件：一是所有字符串长度都小于64字节，二是元素数量小于512，不满足任意一个都会使用linkedlist编码。
2. 两个条件的数字可以在Redis的配置文件中修改，list-max-ziplist-value选项和list-max-ziplist-entries选项。
3. 图中StringObject就是上一节讲到的字符串对象，字符串对象是唯一个在五大对象中作为嵌套对象使用的。
 
* 应用场景：
做简单的消息队列的功能；最新消息排行等功能（比如朋友圈的时间线）。
