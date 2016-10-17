
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
>
```c
if(list->free) list-free(node->value);
else zfree(node);
```

表示如果有自定义的free函数，则调用，否则用zfree函数。
>
```c
typedef struct listIter {
    listNode *next;
    int direction;
} listIter;
```

listIter是访问链表的迭代器, 指针(next)指向链表的某个结点, direction表示迭代访问的方向(宏AL_START_HEAD表示向前,AL_START_TAIL表示向后)。

##函数使用
链表创建时(listCreate), 通过Redis自己实现的zmalloc()分配堆空间。 链表释放(listRelease)或删除结点(listDelNode)时, 如果定义了链表(list)的指针函数free, Redis会使用它释放链表的每一个结点的值(value), 否则需要用户手动释放。 结点的内存使用Redis自己实现的zfree()释放。

对于迭代器, 通过方法listGetIterator()、listNext()、 listReleaseIterator()、listRewind()和listRewindTail()使用, 例如对于链表list,要从头向尾遍历,可通过如下代码:
>
```c
iter = listGetIterator(list, AL_START_HEAD); // 获取迭代器
while((node = listNext(iter)) != NULL)
{
        dosomething;
}
listReleaseIterator(iter);
```

listDup()用于复制链表, 如果用户实现了dup函数, 则会使用它复制链表结点的value。 listSearchKey()通过循环的方式在O(N)的时间复杂度下查找值, 若用户实现了match函数, 则用它进行匹配, 否则使用按引用匹配。
