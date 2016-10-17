
##数据结构

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;   
} listNode;
```
listNode是最基本的结构, 表示链表中的一个结点。 结点有向前(next)和向后 (prev)两个指针, 链表是双向链表。保存的值是void * 类型。

```c
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```
链表通过list定义, 提供头(head)、尾(tail)两个指针, 分别指向头部的结点和尾部的结点; 通过无符号长整数len, 标示链表的长度; 
提供三个函数指针, 供用户传入自定义函数, 用于复制(dup)、释放(free)和匹配(match)链表中的结点的值(value); 
在adlist.c中有类似如下代码段：
```c
if(list->free) list-free(node->value);
else zfree(node);
```
表示如果有自定义的free函数，则调用，否则用zfree函数。

```c
typedef struct listIter {
    listNode *next;
    int direction;
} listIter;
```
listIter是访问链表的迭代器, 指针(next)指向链表的某个结点, direction表示迭代访问的方向(宏AL_START_HEAD表示向前,AL_START_TAIL表示向后)。

