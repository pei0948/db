## <h3 id="redis_data_structure_2">列表对象(list)</h3>
> - Redis3.2及之后的底层实现方式：quicklist
列表对象的编码可以是ziplist和linkedlist之一。
> - list 的实现为一个双向链表，经常被用作队列使用，支持在链表两端进行push和pop操作，时间复杂度为O(1)；同时也支持在链表中的任意位置的存取操作，但是需要对list进行遍历，时间复杂度为O(n)。

### 1 底层实现
linkedlist编码底层采用双端链表实现，每个双端链表节点都保存了一个字符串对象，在每个字符串对象内保存了一个列表元素。<br/>
![image](https://user-images.githubusercontent.com/87458342/132521303-9a8b7ea6-2995-4e9e-85ad-d9692e4427fc.png)
 
### 2 应用场景：
- 做简单的消息队列的功能；最新消息排行等功能（比如朋友圈的时间线）
- 微博的关注列表，粉丝列表，消息列表等功能都可以用Redis的 list 结构来实现。可以利用lrange命令，做基于redis的分页功能。
