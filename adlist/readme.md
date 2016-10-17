typedef struct list{    
   listNode *head;   //头节点    
   listNode *tail;   //尾节点    
   unsigned long len;    //节点个数    
   void *(*dup)(void *ptr); //复制ptr内容函数    
   void (*free)(void *ptr);  //释放ptr值函数    
   int (*match)(void *ptr，void *key); //匹配ptr和key内容的函数    
}list;  

##数据结构
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;
listNode是最基本的结构, 表示链表中的一个结点。 结点有向前(next)和向后 (prev)两个指针, 链表是双向链表。 保存的值是void *类型。
typedef struct list {
    listNode *head;
    
    listNode *tail;
    
    void *(*dup)(void *ptr);
    
    void (*free)(void *ptr);
    
    int (*match)(void *ptr, void *key);
    
    unsigned long len;
} list;
链表通过list定义, 提供头(head)、尾(tail)两个指针, 分别指向头部的结点和尾部的结点; 提供三个函数指针, 供用户传入自定义函数, 用于复制(dup)、释放(free)和匹配(match)链表中的结点的值(value); 通过无符号长整数len, 标示链表的长度。
其中dup和free，match函数可用于自定义函数；
在adlist.c中有类似如下代码段：
if(list->free) list-free(node->value);
else zfree(node);
表示如果有自定义的free函数，则调用，否则用zfree函数。
