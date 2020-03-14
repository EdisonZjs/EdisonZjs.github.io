# 字典




## 1.字典的实现

``redis``中的字典是一种用于保存键值对映射关系的抽象数据结构，字典中每一个键都是独一无二的，可以根据键来操作与之关联的值。

``redis``中字典的应用非常广泛，比如``redis``的数据库、哈希键等等。例如执行命令：``set zjs "handsome boy"``，在数据库中创建一个键为”zjs“，值为”handsome boy"的键值对，这个键值对就保存在数据库字典中。

字典使用哈希表作为底层实现，一个哈希表可以有多个哈希节点，每个哈希节点就保存了一个键值对。

### 1.1 哈希表

``redis``字典所使用的哈希表由``dict.h/dictht``结构定义：

```c
typedef struct dictht {
	
	// 哈希表数组
	dictEntry **table;
	
	// 哈希表大小
	unsigned long size;
	
	// 哈希大小掩码，用于计算索引值。总是等于size-1
	unsigned long sizemask;
	
	// 哈希表已有节点的数量
	unsigned long used;
} dictht;
```

``table``属性是一个数组，数组中的每个元素都是一个指向``dict.h/dictEntry``结构的指针，每个``dictEntry``结构保存一个键值对。``size``属性记录了哈希表的大小，也就是table数组的大小。而used属性则记录了哈希表目前已经有的节点(键值对)的数量。``sizemask``属性和哈希值一起决定一个键应该被放在table数组的哪个索引上。

### 1.2 哈希表节点

哈希表节点使用``dictEntry``结构表示，每个``dictEntry``结构都保存着一个键值对：

```c
typedef struct dictEntry {
	
	// 键
	void *key;
	
	// 值
	union {
		void *val;
		uint64_tu64;
		int64_ts64;
	} v;
	
	// 指向下个哈希表节点，形成链表
	strcut dictEntry *next;
} dictEntry;
```

``key``属性保存着键值对中的键，而``v``属性则保存着键值对中的值，其中键值对的值可以是一个指针，或者是一个``uint64_t``整数，又或者是一个``int64_t``整数。``next``属性则是指向下一个哈希表节点的指针，这个指针可以将多个哈希值相同的键值对连接在一起形成一个链表，以此来解决键冲突的问题。

下图就展示了一个哈希表，键``k1``和``k2``哈希值相同，通过``next``指针相连。

![image.png](https://i.loli.net/2020/03/11/ZYrHQkvTKyUhEf7.png)

### 1.4 字典

``redis``中的字典由``dict.h/dict``结构表示：

```c
typedef struct dict {
	
	// 类型特定函数
	dictType *type;
	
	// 私有数据
	void *privdata;
	
	// 哈希表
	dictht ht[2];
	
	// rehash索引，当rehash不在进行时，值为-1
	in trehashidx;
} dict;
```

``ht``属性是一个包含两个哈希表的数组。一般情况下，字典只使用``ht[0]``哈希表，``ht[1]``哈希表只有在哈希表进行``rehash``时使用。``rehashidx``属性记录了rehash目前的进度，如果目前没有在进行``rehash``，那么它的值为-1。

下图是一个常规状态下(没有正在rehash)的字典

![image.png](https://i.loli.net/2020/03/11/sJN6XKcuqGk7xDi.png)

## 2.解决键冲突

当有两个或以上数量的键被分配到了哈希表数组的同一个索引上面时，我们称这些键发生了冲突。``redis``的哈希表使用链地址法来解决键冲突，每个哈希表节点都有一个next指针，多个哈希表节点可以用next指针构成一个单向链表，被分配到同一个索引上的多个节点可以用这个单向链表连接起来，这就解决了键冲突的问题。



![image.png](https://i.loli.net/2020/03/11/czaU5OCmNnXLGEk.png)

举个例子，假设要将键值对``k2``和``v2``添加到上图的哈希表中。经过计算得出``k2``的索引值为1，那么键``k1``和``k2``将产生冲突，而解决冲突的办法就是使用next指针将键``k1``和``k2``所在的节点连接起来，如下图：

![image.png](https://i.loli.net/2020/03/11/iwLRDjBtUIOzJsQ.png)



## 3.渐进式的rehash

随着操作的不断执行，哈希表保存的键值对会逐渐的增多或减少，为了让哈希表的负载因子维持在一个合理的范围之内，当哈希表保存的键值对数量太多或太少时，程序需要对哈希表的大小进行相应的扩展或收缩。

扩展和收缩哈希表可以通过执行rehash操作来完成，``redis``对字典的哈希表执行rehash的步骤如下：

1. 为字典的ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对数量(即ht[0].used属性的值)
   - 如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于``ht[0].used * 2``的``2^n(2的n次方)``
   - 如果执行的是收缩操作，那么ht[1]的大小为第一个大于等于``ht[0].used``的``2^n``
2. 将保存在ht[0]中的所有键值对rehash到ht[1]上
3. 当ht[0]包含的所有键值对都迁移到了ht[1]之后，释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备

下图就展示了一个rehash过程：

​	![image.png](https://i.loli.net/2020/03/12/Fiaw9eJz8tPgAhl.png)												



1. ht[0].used的值为4，4*2=8，第一个大于等于8的2^n就是恰好就是8(2³)，所以ht[1]哈希表的大小设置为8。下图是在ht[1]分配空间之后，字典的样子。

   ![image.png](https://i.loli.net/2020/03/12/zDiYQkqA7uXBIOh.png)

2. 将ht[0]包含的四个键值对都rehash到ht[1]

   ![image.png](https://i.loli.net/2020/03/12/sh4pdGkrbu1UxJP.png)

3. 释放ht[0]，并将ht[1]设置为ht[0]，然后为ht[1]分配一个空白哈希表。至此，对哈希表的扩展操作执行完毕。

![image.png](https://i.loli.net/2020/03/12/yxAkif3weq541oj.png)

当满足以下任意一个条件时，程序会自动开始对哈希表执行扩展操作：

1. 服务器目前没有执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1。
2. 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5。
   - 其中哈希表的负载因子可以通过公式：``load_factor = ht[0].used ÷ ht[0].size``计算得出



在把``ht[0]``哈希表中的所有键值对重新rehash到``ht[1]``里面，这个rehash动作并不是一次性、集中式的完成的，而是分多次、渐进式的完成的。因为如果``ht[0]``里面有几千万个键值对，那一次性全部rehash到``ht[1]``里面不得卡死。

以下是哈希表渐进式rehash的详细步骤：

1. 为``ht[1]``分配空间，让字典同时持有``ht[0]``和``ht[1]``两个哈希表。
2. 在字典中维持一个索引计数器变量``rehashidx``，并将它的值设置为0，表示``rehash``工作正式开始。
3. 在``rehash``进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将``ht[0]``哈希表在``rehashidx``索引上的所有键值对``rehash``到``ht[1]``上，当``rehash``工作完成之后，程序将``rehashidx``属性的值加一。
4. 随着字典操作的不断执行，最终在某个时间点上，``ht[0]``的所有键值对都会被``rehash``至``ht[1]``，这时程序将``rehashidx``属性的值设置为-1，表示``rehash``操作完成。

渐进式``rehash``的好处就是在于它采用分而治之的思想，将``rehash``键值对所需的计算工作均摊到每个添加、删除、查找和更新操作上，从而避免了集中式``rehash``带来的庞大计算量。

下图展示了一个渐进式rehash的过程：

![image.png](https://i.loli.net/2020/03/12/5kLupwbjFNEZrRK.png)

&nbsp;&nbsp;&nbsp;

![image.png](https://i.loli.net/2020/03/12/GHD8o95RbVes1TE.png)

&nbsp;&nbsp;

![image.png](https://i.loli.net/2020/03/12/kUj4s1y2vpne786.png)

&nbsp;&nbsp;

![image.png](https://i.loli.net/2020/03/12/aG1z7uJ5dWMcFKg.png)

![image.png](https://i.loli.net/2020/03/12/UvF2DBp9LomcWk8.png)

![image.png](https://i.loli.net/2020/03/12/n7KB8UTPpjHGgEd.png)

### 3.1 渐进式rehash执行期间的哈希表操作

因为在进行渐进式rehash过程中，字典会同时使用ht[0]和ht[1]两个哈希表，所以在渐进式rehash进行期间，字典的删除、查找、更新等操作会在两个哈希表中进行。例如要查找一个键，程序会先在ht[0]里进行查找，如果找到就返回，没找到就继续到ht[1]里进行查找。

另外，在渐进式rehash执行期间，新添加的键值对一律保存到ht[1]里面，而ht[0]不再进行任何添加操作，这一措施保证了ht[0]的键值对数量只减不增，并随着rehash操作的执行最终变成空表。



## 参考

[Redis设计与实现]: http://redisbook.com/

[Redis设计与实现]