## redis数据结构实现--链表（list）
#### 2.链表和链表节点的实现


* * *
每个链表节点由一个listNode实现
```
typeof struct listNode{

    //前置节点
    
    struct listNode *prev;
    
    //前置节点
    
    struct listNode *next;
    
    //值
    
    void *value;
    
}listNode;
```
用list来持有链表
```
typeof struct list{

    //表头节点
    
    listNode *head;
    
    //表尾节点
    
    listNode *tail;
    
    //链表长度
    
    unsigned long len;
    
    //节点值复制函数
    
    void *(*dup)(void *ptr);
    
    //节点值释放函数
    
    void (*free)(void *ptr)
    
    //节点值对比函数
    
    int (*match)(void *ptr,void *key)
    
}
```
redis链表的实现特性总结如下：
* 双端：链表节点带有prev和next指针
* 无环：表头节点的prev指针和表尾节点的next指针都指向null，访问链表以null为终点
* list中带头指针和表尾指针：通过list结构中的head和tail指针访问表头尾节点时间复杂度为*O*(1)
* 链表长度计数器：list结构中有len属性来记录链表长度，获取链表长度时间复杂度为*O*(1)
* 链表节点void* 指针来保存节点值，可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。




