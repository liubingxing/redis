# 字典和哈希表

## 目录

- 数据结构
- 整体流程
- 实现与设计
- 算法描述

### 数据结构

- 哈希表节点

  哈希表节点使用dictEntry结构表示，dictEntry结构用来保存键值对。该结构体定义如下：

```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; //链表解决冲突
} dictEntry;
```
- 哈希表

  Redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个节点，每个节点保存字典中的一个键值对。哈希表的结构体定义在dict.h中：
 
 ```c
 typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```
- 字典

  字典结构体定义如下：
 
 ```c
 typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;
```
- 字典与哈希表的关系图


