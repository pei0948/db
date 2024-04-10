## <h3 id="redis_data_structure_4">集合对象(set)</h3>
> - set是一个存放不重复值的无序集合，可做全局去重的功能，提供了判断某个元素是否在set集合内的功能，这个也是list所不能提供的。基于set可以实现交集、并集、差集的操作，计算共同喜好，全部的喜好，自己独有的喜好等功能。
> - 集合对象的编码可以是intset和hashtable之一。
### 1 底层实现
#### 1.1 intset编码
intset编码的集合对象底层实现是整数集合，所有元素都保存在整数集合中。<br/>
![image](https://user-images.githubusercontent.com/87458342/132521935-98713a61-bc4e-47c5-a79d-168aaa2639f4.png)
 
#### 1.2 hashtable编码
hashtable编码的集合对象底层实现是字典，字典的每个键都是一个字符串对象，保存一个集合元素，不同的是字典的值都是NULL；可以参考java中的hashset结构。<br/>
![image](https://user-images.githubusercontent.com/87458342/132522046-75ce3940-7082-4a56-966b-bf223f2b5bf4.png)
 
#### 1.3 集合对象编码转换：
集合对象使用intset编码需要满足两个条件：
- 一是所有元素都是整数值；
- 二是元素个数小于等于512个
不满足任意一条都将使用hashtable编码。以上第二个条件可以在Redis配置文件中修改et-max-intset-entries选项。
 
### 2 应用场景：
1. 共同好友
2. 利用唯一性，统计访问网站的所有独立ip
