
## redis数据结构实现--字典(set)
#### 3.1 字典的实现

* * *
字典（set）是一种保存键值对的抽象数据结构。
set key value 将存在数据库字典中，键不可重复。
哈希键的底层实现之一就是字典。

* * *

>Redis的字典使用哈希表作为底层实现，一个哈希表中有多个哈希节点，而每个节点中就保存了字典的一个键值对。


哈希表结构定义：
```
typeof struct dictht{

    //哈希表数组
    dictEntry **table;
    
    //哈希表大小
    unsigned long size;
    
    //哈希表大小掩码，用于计算索引值
    //总是等于size-1
    unsigned long sizemask;
    
    //该哈希表已有节点数
    unsigned long used;
    
}dictht;
```

table属性是一个数组，数组中每一个元素都是指向dictEntry结构的指针，每个dictEntry保存着一对键值。
哈希表大小掩码sizemask和key的哈希值一起决定一个键应该放在table数组上的哪个索引上.

* * *
![0e293ac3e5105638459e0e00ab39eb68.png](evernotecid://B6C9DE5E-ABC3-4C69-93C8-83BFE0EBF884/appyinxiangcom/21964413/ENResource/p182)

哈希节点的结构定义：

```
typeof struct dictEntry{
    //键
    void *key;
    
    //值
    union{
        void *val;
        uint64_tu64;
        int64_ts64;
    }v;
    
    //指向下个哈希表节点，形成链表
    struct dictEntry *next;
}dictEntry;
```
v中保存着键值中的值，可以是一个指针，也可以是一个int64整数，也可以是uint64整数。
*next是用来指向另个哈希表节点的指针，用来解决键哈希值冲突问题（类似于HashMap）。

字典的结构定义：
```
typeof struct dict{

    //类型特定函数
    dictType *type;
    
    //私有数据
    void *privdata;
    
    //哈希表
    dictht ht[2];
    
    //rehash索引，当rehash不在进行时值为-1
    in rehashidx;
    
}dict;

```

* type属性和privdata属性是为了针对不同类型的键值对，为创建多态字典而设置的。

    * [ ] type属性是一个指向dictType结构的指针，每个dictType保存了一簇用于操作特定类型键值对的函数，redis为不同用途的字典设置不同类型特定的函数。
    * [ ] privdata保存了要传给那些类型特定函数的可选参数。

* * *

* ht属性是一个包含两个项的数组，每一项都是一个dictht哈希表，一般情况下只会用ht[0]，ht[1]只会在ht[0]进行rehash的时候用。
* rehashidx记录当前rehash的进度，如果没有rehash在进行则为-1

![edb33098db103b88313c7e8592823bd7.png](evernotecid://B6C9DE5E-ABC3-4C69-93C8-83BFE0EBF884/appyinxiangcom/21964413/ENResource/p183)

* * *

#### 3.2 哈希算法

    添加新键值对时，先用字典设置的哈希函数根据key算出哈希值，再用哈希值和sizemask按位与运算得出包含此键值对的哈希节点在哈希数组中的索引值。

#### 3.3 键冲突
    
    Redis用链地址法解决键冲突，每个哈希节点都有一个next指针，发生键冲突时就用next指针构成单向链表，跟hashmap一样。

#### 3.4 rehash

    为了让加载因子 loadfactor维持在合理范围内，当保存的哈希节点过多或过少的时候，需要对哈希表进行扩展或缩容。
    loadfactor = 存放节点数/哈希表大小
    
    规定：拓展操作给ht[1]分配第一个大于等于ht[0].used*2的2的n次方幂的空间
            缩容操作给ht[1]分配第一个大于等于ht[0].used的2的n次方幂的空间
            
    rehash是指重新计算ht[0]上节点的哈希值和索引值并放置到ht[1]上去
    当ht[0]上所有节点都转移到ht[1]上以后，释放ht[0],并将ht[1]改为ht[0]
    最后在ht[1]上创建空哈希表
    
 * 没有在执行BGSAVE或BGREWRITEAOF命令时，负载因子大于1进行扩容
 * 在执行BGSAVE或BGREWRITEAOF命令时，负载因子大于5进行扩容
 * 负载因子小于0.1时进行缩容

   **因为在BGSAVE或BGREWRITEAOF命令时，Redis会创建子进程，应避免在子进程存在期间进行扩展操作，来节约内存。**
   
  * * *
  ##### 渐进式rehash详解：
  如果不采取渐进式rehash，而是一股脑全部rehash，很有可能给服务器带来巨大压力
  字典结构中有一个 rehashidx 用来监控rehash进度。
  
  1. rehash开始，rehashidx设置为0；
  2. 期间，将ht[0]中在rehashidx索引上的节点rehash到ht[1]上，rehashidx自增
  3. 所有节点都rehash到ht[1]上后，rehashidx设置为-1，表示rehash结束

    在rehash期间如果有搜索操作，将会先搜索ht[0]，找不到再去ht[1]
    如果是新增节点将直接增到ht[1]中
    