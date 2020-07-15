## redis数据结构实现--跳表（zskiplist）
#### 4 跳表的实现
* * *
结构图：
![25cf940fbeb9cda105d00121f46f8ad4.png](evernotecid://B6C9DE5E-ABC3-4C69-93C8-83BFE0EBF884/appyinxiangcom/21964413/ENResource/p181)



跳表由zskiplistNode组成zskiplist构成

zskiplist结构：
```
    header: 指向跳跃表的头节点
    tail: 指向跳跃表的尾节点
    level: 跳跃表中层数最大节点的层数（表头的层数不计入）
    length: 跳表保存的节点数（空表头不计入）
```
zskiplistNode结构：
```
    level数组: 层，每次创建一个新的跳表节点都会根据幂次定律计算出level数组的大小，也就是次层的高度。每一层带有两个属性-前进指针和跨度。
            前进指针用于访问表尾方向的其他指针；跨度用于记录当前节点与前进指针所指节点的距离
   backward: 后退指针，指向当前节点的前一个节点
   score：分值，用来排序，如果分值相同看成员变量在字典序大小排序
   obj: 成员对象是一个指针，指向一个字符串对象里面保存着一个sds；
            在跳表中各个节点的成员对象必须唯一，分值可以相同
```


