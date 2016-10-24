# 字典和哈希表

## 目录
- [1.数据结构](#1)
- [2.整体流程](#2)
- [3.代码设计与实现](#3)
- [4.反向二进制迭代器](#4)

<h2 id="1">1.数据结构</h2>

### 哈希表节点

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
### 哈希表

Redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个节点，每个节点保存字典中的一个键值对。哈希表的结构体定义在dict.h中：
 
 ```c
 typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```
### 字典

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
### 字典与哈希表的关系图

![image](https://github.com/liubingxing/redis/raw/master/dict/1.png)

<h2 id="2">2.整体流程</h2>

### 哈希算法

将一个新的键值对添加到字典中时，首先根据键计算出哈希值，然后根据哈希值计算出索引值，最后根据索引值，将包含新键值对的哈希表节点存储到哈希表数组中的指定索引上。当两个以上的键计算得到的哈希值一样时，称这些键发生了冲突。Redis的哈希表使用链接法来解决键冲突。因dictEntry节点组成的链表没有指向链表表尾的指针，为了速度考虑，总是将新节点添加到链表的表头位置。

```c
//使用哈希函数，计算key的哈希值
hash = dict->type->hashFunction(key);
//使用哈希表的sizemask属性，根据哈希值计算出索引值
index = hash & dict->ht[x].sizemask;
```
### rehash

随着操作的不断进行，哈希表保存的键值对会逐渐地增多或者减少，为了让哈希表的负载因子维持在一个合理的范围之内，当哈希表保存的键值对数量太多或太少时，需要对哈希表的大小进行相应的扩展或者收缩。这就是通过执行rehash操作来完成，rehash的步骤如下：

A：为字典的哈希表ht[1]分配空间，分配的空间大小取决于要执行的操作，以及ht[0].used 的值，分为扩展和收缩操作：

a.扩展操作，那么ht[1]的大小为第一个大于等于2\*(ht[0].used)的2^n（2的n次幂）；

b.收缩操作，那么ht[1]的大小为第一个大于等于ht[0].used的2^n（2的n次幂）；

B：重新计算ht[0]中每个键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上。

C：当ht[0]上所有键值对都迁移到ht[1]之后，释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备。

### rehash条件
Redis哈希表的负载因子计算方法是：
```c
//负载因子 = 哈希表已保存的节点数量 / 哈希表大小
load_factor = ht[0].used / ht[0].size
```
A.当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作;

B.当以下条件中的任意一个被满足时，程序会自动开始对哈希表执行扩展操作：

a：若服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，且哈希表的负载因子大于等于1。

b：若服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，且哈希表的负载因子大于等于5。

当redis在执行BGSAVE命令或者BGREWRITEAOF命令的过程中，Redis要用fork创建当前服务器进程的子进程，大多数操作系统在fork时都采用写时复制技术来优化内存的使用效率。因此，在子进程存在期间，通过提高扩展操作所需的负载因子，减少进行哈希表扩展操作的可能，避免不必要的内存写人操作，从而最大限度地节约内存。

### 渐进式rehash
为了避免rehash对服务器性能造成影响，服务器不是一次性将ht[0]里面的所有键值对全部rehash到ht[1]，而是分多次、渐进式地将ht[0]里面的键值对慢慢地rehash到ht[1]。以下是哈希表渐进式rehash的详细步骤：

a：为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表。

b：在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash正式开始。

c：在rehash进行期间，每次对字典执行添加、删除、查找或者更新操作时，除了执行指定的操作以外，还会顺带将ht[0]在rehashidx索引上的所有键值对rehash到ht[1]上，当rehash工作完成之后，rehashidx++。需要注意的是 rehashidx 记录的实际上是 rehash 进行到的索引，比如如果 rehash 进行到第 10 个元素，那么 rehashidx 的值就为 9，以此类推，如果没有在进行 rehash ，rehashidx 的值就为 -1 。

d：随着字典操作的不断执行，最终在某个时间点上，ht [0]的所有键值对都会被rehash至ht[1]，这时程序将rehashidx属性的值设为-1，表示rehash操作已完成。

<h2 id="3">3.代码设计与实现</h2>

### 字典创建流程
创建新字典执行的调用链是： dictCreate -> \_dictInit -> \_dictReset

其中 dictCreate 函数为 dict 结构分配了空间，然后将新的 dict 传给 \_dictInit 函数，让它初始化 dict 结构的相关属性，而 \_dictInit 又调用 \_dictReset ，对字典的 ht 属性（也即是两个哈希表）进行常量属性的设置。注意， \_dictReset 只是为字典所属的两个哈希表进行常量属性的设置(size、 sizemask 和 used），但并不为哈希表的链表数组分配内存。

### 0号哈希表的创建流程
从上一节可以知道， dictCreate 并不为哈希表的链表数组分配内存( d->ht[0]->table 和 d->ht[1]->table 都被设为 NULL），那么，什么时候哈希表的链表数组会被初始化呢？

答案是，当首次通过 dictAdd 向字典添加元素的时候， 0 号哈希表的链表数组会被初始化。

首次向字典增加元素将执行以下的调用序列： dictAdd -> dictAddRaw -> \_dictKeyIndex -> dictExpandIfNeeded -> dictExpand

其中 dictAdd 是 dictAddRaw 的调用者， dictAddRaw 是添加元素这一工作的底层实现，而 dictAddRaw 为了计算新元素的 key 的地址索引，会调用 \_dictKeyIndex ：
```c
dictEntry *dictAddRaw(dict *d, void *key)  
{  
    // 被省略的代码...   
  
    // 计算 key 的 index 值  
    // 如果 key 已经存在，_dictKeyIndex 返回 -1   
    if ((index = _dictKeyIndex(d, key)) == -1)  
        return NULL;  
  
    // 被省略的代码...   
}  
```
然后 \_dictKeyIndex 会在计算地址索引前，会先调用 \_dictExpandIfNeeded 检查两个哈希表是否有空间容纳新元素：
```c
static int _dictKeyIndex(dict *d, const void *key)  
{  
    // 被省略的代码...  
  
    /* Expand the hashtable if needed */  
    if (_dictExpandIfNeeded(d) == DICT_ERR)  
        return -1;  
  
    // 被省略的代码...  
}  
```
到 \_dictExpandIfNeeded 这步， \_dictExpandIfNeeded 会检测到 0 号哈希表还没有分配任何空间，于是它调用 dictExpand ，传入 DICT_HT_INITIAL_SIZE 常量作为 0 号哈希表的初始大小（目前的版本 DICT_HT_INITIAL_SIZE = 4 )，为 0 号哈希表分配空间：
```c
static int _dictExpandIfNeeded(dict *d)  
{  
    // 被忽略的代码...  
  
    /* If the hash table is empty expand it to the intial size. */  
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);  
  
    // 被忽略的代码...  
}  
```
dictExpand 会创建一个分配了链表数组的新哈希表，然后进行判断，决定是该将新哈希表赋值给 0 号哈希表还是 1 号哈希表。

这里因为我们的 0 号哈希表的 size 还是 0 ，因此，这里会执行 if 语句的第一个 case ，将新哈希表赋值给 0 号哈希表：
```c
int dictExpand(dict *d, unsigned long size)  
{  
    // 被省略的代码...  
  
    // 计算哈希表的(真正)大小  
    unsigned long realsize = _dictNextPower(size);  
  
    // 创建新哈希表  
    dictht n;  
    n.size = realsize;  
    n.sizemask = realsize-1;  
    n.table = zcalloc(realsize*sizeof(dictEntry*));  // 分配链表数组  
    n.used = 0;  
  
    // 字典的 0 号哈希表是否已经初始化？  
    // 如果没有的话，我们将新建哈希表作为字典的 0 号哈希表  
    if (d->ht[0].table == NULL) {  
        d->ht[0] = n;  
    } else {  
    // 否则，将新建哈希表作为字典的 1 号哈希表，并将它用于 rehash  
        d->ht[1] = n;  
        d->rehashidx = 0;  
    }  
  
    // 被省略的代码...  
}  
```
### 字典的扩展和 1 号哈希表的创建
在 0 号哈希表创建之后，我们就有了一个可以执各式各样操作（添加、删除、查找，诸如此类）的字典实例了。但是这里还有一个问题： 这个最初创建的 0 号哈希表非常小（当前版本的 DICT_HT_INITIAL_SIZE = 4)，它很快就会被元素填满，这时候， rehash 操作就会被激活。

（字典的）哈希表的完整 rehash 操作分为两步：

1） 首先就是创建一个比现有哈希表更大的新哈希表（expand）

2） 然后将旧哈希表的所有元素都迁移到新哈希表去（rehash）

当使用 dictAdd 对字典添加元素的时候， \_dictExpandIfNeeded 会一直对 0 号哈希表的使用情况进行检查，当 rehash 条件被满足的时候，它就会调用 dictExpand 函数，对字典进行扩展：
```c
static int _dictExpandIfNeeded(dict *d)  
{  
    // 被省略的代码...  
  
    // 当 0 号哈希表的已用节点数大于等于它的桶数量，  
    // 且以下两个条件的其中之一被满足时，执行 expand 操作：  
    // 1) dict_can_resize 变量为真，正常 expand  
    // 2) 已用节点数除以桶数量的比率超过变量 dict_force_resize_ratio ，强制 expand  
    // (目前版本中 dict_force_resize_ratio = 5)  
    if (d->ht[0].used >= d->ht[0].size &&  
        (dict_can_resize ||  
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))  
    {  
        return dictExpand(d, ((d->ht[0].size > d->ht[0].used) ?  
                                    d->ht[0].size : d->ht[0].used)*2);  
    }  
  
    // 被省略的代码...  
}  
```

### 渐增式 rehash 和平摊操作
当 expand 执行完毕之后，字典同时使用两个哈希表，并且字典的 rehashidx 属性从 -1 被改为 0 ，这是一个重要的改动，它标志着 rehash 可以进行了。

但是 rehash 操作在那里？我们什么时候调用它？嗯。。。源码里的确有一个 dictRehash 函数，它可以将指定数量的元素从 0 号哈希表 rehash 到 1 号哈希表，但是，redis 并不是使用它一下子将 0 号哈希表的所有元素全都 rehash 到 1 号哈希表，因为这样集中式的 rehash 会引起大量的计算工作 ，进而影响整个系统的性能。

以下是 dictFind 函数，它是其中一个平摊 rehash 操作的函数：
```c
dictEntry *dictFind(dict *d, const void *key)  
{  
    // 被忽略的代码...  
  
    // 检查字典(的哈希表)能否执行 rehash 操作  
    // 如果可以的话，执行平摊 rehash 操作  
    if (dictIsRehashing(d)) _dictRehashStep(d);  
  
    // 被忽略的代码...  
}  
```
将一个元素从 0 号哈希表转移到 1 号哈希表，代码中的 iterators == 0 表示在 rehash 时不能有迭代器，因为迭代器可能会修改元素，所以不能在有迭代器的情况下进行 rehash 
```c
static void _dictRehashStep(dict *d) {  
    if (d->iterators == 0) dictRehash(d,1);  
}  
```

就这样， 0 号哈希表的元素被逐个逐个地，从 0 号 rehash 到 1 号，最终整个 0 号哈希表被清空，这时 \_dictRehashStep 再调用 dictRehash ，被清空的 0 号哈希表就会被删除，然后原来的  1 号哈希表成为新的 0 号哈希表（当有需要再次进行 rehash 的时候，这个循环就会再次开始）：
```c
int dictRehash(dict *d, int n) {  
    // 被忽略的代码...  
  
    while(n--) {  
        dictEntry *de, *nextde;  
  
        // 0 号哈希表的所有元素 rehash 完毕？  
        if (d->ht[0].used == 0) {  
            zfree(d->ht[0].table);  // 清空 0 号  
            d->ht[0] = d->ht[1];   // 用原来的 1 号代替 0 号  
  
            _dictReset(&d->ht[1]);  // 重置 1 号哈希表  
            d->rehashidx = -1;      // 重置字典的 rehash flag  
  
            return 0;  
        }  
    // 被忽略的代码...  
    }  
  
    // 被忽略的代码...  
}  
```
当 rehashidx 不等于 -1 ，也即是 dictIsRehashing 为真时，所有新添加的元素都会直接被加到 1 号数据库，这样 0 号哈希表的大小就会只减不增，最终 rehash 总会有完成的一刻
```c
dictEntry *dictAddRaw(dict *d, void *key)  
{  
    // 被省略的代码...  
  
    // 如果字典正在进行 rehash ，那么将新元素添加到 1 号哈希表，  
    // 否则，使用 0 号哈希表  
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];  
  
    // 被省略的代码...  
}  
```
<h2 id="4">4.反向二进制迭代器</h2>

distScan函数是用来遍历字典的，但是在遍历的过程中，redis因为有rehash过程，字典可能会扩展和收缩，字典中的元素所在的位置相应的会发生变化，那如何保证字典中原有的元素都可以被遍历？又如何能尽可能少的重复迭代呢？

计算一个哈希表节点索引的方法是hashkey&mask，其中，mask的值永远是哈希表大小减1。哈希表长度为8，则mask为111，因此，节点的索引值就取决于hashkey的低三位，假设是abc。如果哈希表长度为16，则mask为1111，同样的节点计算得到的哈希值不变，而索引值是?abc，其中?既可能是0，也可能是1，也就是说，该节点在长度为16的哈希表中，索引是0abc或者1abc。以此类推，如果哈希表长度为32，则该节点的索引是00abc，01abc，10abc或者11abc中的一个。

首先看一下cursor的演变过程，也是该算法的核心所在。这里cursor的演变是采用了reverse binary iteration方法，也就是每次是向cursor的最高位加1，并向低位方向进位。
>
000 --> 100 --> 010 --> 110 --> 001 --> 101 --> 011 --> 111 --> 000     

0000 --> 1000 --> 0100 --> 1100 --> 0010 --> 1010 --> 0110 --> 1110 --> 0001 --> 1001 --> 0101 --> 1101 --> 0011 --> 1011 --> 0111 --> 1111 --> 0000     

在Redis中，字典的哈希表长度始终为2的n次方。因此m0始终是一个低n位全为1，其余为全为0的数。整个计算过程，都是在v的低n位数中进行的，比如长度为16的哈希表，则n=4，因此v是从0到15这几个数之间的转换。下面解释一下计算过程：

第一步：v |= ~m0;                 //用于保留v的低n位数，其余位全置为1：

![image](https://github.com/liubingxing/redis/raw/master/dict/distscan-1.png)

第二步：v = versebits2(v);    //将v的二进制位进行翻转，所以，v的低n位数成了高n位数，并且进行了翻转：

![image](https://github.com/liubingxing/redis/raw/master/dict/distscan-2.png)

第三步：v++;

![image](https://github.com/liubingxing/redis/raw/master/dict/distscan-3.png)

最后一步：v =versebits2(v);                 //再次翻转

![image](https://github.com/liubingxing/redis/raw/master/dict/distscan-4.png)

因此，最终得到的新v，就是向最高位加1，且向低位方向进位。

哈希表长度为8时，第i个cursor（0 <= i <=7），扩展到长度为16的哈希表中，对应的cursor是2i和2i+1，它们是相邻的。假设当前字典哈希表长度为8，在迭代完索引为010的bucket之后，下一个cursor为110。假设在下一次迭代前，字典哈希表长度扩展成了16，110这个cursor，在长度为16的情况下，就成了0110和1110（0110在前），因此开始迭代索引为0110的bucket中的节点。

在长度为8时，已经迭代过的cursor分别是：000，100，010。哈希表长度扩展到16后，在这些索引的bucket中的节点，分布到新的bucket中，新bucket的索引将会是：0000，1000，0100，1100，0010，1010。而这些，正好是将要迭代的0110之前的索引，从0110开始，按照长度为16的哈希表cursor变化过程迭代下去，这样既不会漏掉节点，也不会迭代重复的节点。

 再看一下字典哈希表缩小的情况，也就是由16缩小为8。在长度为16时，迭代完0100的cursor之后，下一个cursor为1100，假设此时哈希表长度缩小为8。1100这个cursor，在长度为8的情况下，就成了100。因此开始迭代索引为100的bucket中的节点。
 
 在长度为16时，已经迭代过的cursor是：0000，1000，0100，哈希表长度缩小后，这些索引的bucket中的节点，分布到新的bucket中，新bucket的索引将会是：000和100。现在要从索引为100的bucket开始迭代，这样不会漏掉节点，但是之前长度为16时，索引为0100中的节点会被重复迭代，然而，也就仅0100这一个bucket中的节点会重复而已。
 
原哈希表长度为x，缩小后长度为y，则最多会有x/y – 1个原bucket的节点会被重复迭代。比如由16缩小为8，则最多就有1个bucket节点会重复迭代，要是由32缩小为8，则最多会有3个。当然也有可能不产生重复迭代，还是从16缩小为8的情况，如果已经迭代完1100，下一个cursor为0010，此时长度缩小为8，cursor就成了010。

所以说这种算法，保证了：能迭代完所有节点而不会漏掉；又能尽可能较少的重复遍历。

如果按照正常的顺序迭代，下面分别是长度为8和16对应的cursor变化过程：
>
000 --> 001 --> 010 --> 011 --> 100 --> 101 --> 110 --> 111 --> 000     

0000 --> 0001 --> 0010 --> 0011 --> 0100 --> 0101 --> 0110 --> 0111 --> 1000 --> 1001 --> 1010 --> 1011 --> 1100 --> 1101 --> 1110 --> 1111 --> 0000  

字典扩展的情况，当前字典哈希表长度为8，假设在迭代完cursor为010的bucket之后，下一个cursor为011。迭代011之前，字典长度扩展成了16，011这个cursor，在长度为16的情况下，就成了0011，因此开始迭代索引为0011的bucket中的节点。

在长度为8时，已经迭代过的cursor是：000，001，010。哈希表长度扩展到16后，这些索引的bucket中的节点，会分布到新的bucket中，新bucket的索引将会是：0000，1000，0001，1001，0010和1010。现在要开始迭代的cursor为0011，而1000，1001，1010这些bucket中的节点在后续还是会遍历到，这就产生了重复遍历。

长度缩小的情况，长度由16缩小为8。在长度为16时，迭代完0100的cursor之后，下一个cursor为0101，此时长度缩小为8。0101这个cursor，在长度为8的情况下，就成了101。

在长度为16时，尚未迭代过的cursor是：0101，0110，0111，1000，1001，1010，1011，1100，1101，1110，1111。这些cursor，在哈希表长度缩小后，分配到新的bucket中，索引将会是：000，001，010，011，100，101，110，111。现在要开始迭代的cursor为101，那101之前的000，001，010，011，100这些cursor就不会迭代了，这样，原来的某些节点就被漏掉了。

另外，还是从16缩小为8的情况，如果已经迭代完1100，下一个cursor为1101，在长度为8的情况下，就成了101。

长度为16时，已经迭代过的cursor为0000，0001，0010，0011，0100，0101，0110，0111，1000，1001，1010，1011，1100。这些cursor，在哈希表长度缩小后，分配到新的bucket中，索引分别是：000，001，010，011，100，101，110，111。长度变为8后，从101开始，很明显，原来已经迭代过的0101，0110，0111就会产生重复迭代。

 因此，顺序迭代不是一个满足要求的迭代方法。

最后说一下distScan是如何遍历大小两个表的：
```c
/* Iterate over indices in larger table that are the expansion 
 * of the index pointed to by the cursor in the smaller table */  
do {//扫描大点的表里面的槽位，注意这里是个循环，会将小表没有覆盖的slot全部扫描一次的  
    /* Emit entries at cursor */  
    de = t1->table[v & m1];  
    while (de) {  
        fn(privdata, de);  
        de = de->next;  
    }  
    /* Increment bits not covered by the smaller mask */  
    //下面的意思是，还需要扩展小点的表，将其后缀固定，然后看高位可以怎么扩充。  
    //其实就是想扫描一下小表里面的元素可能会扩充到哪些地方，需要将那些地方处理一遍。  
    //后面的(v & m0)是保留v在小表里面的后缀。  
    //((v | m0) + 1) & ~m0) 是想给v的扩展部分的二进制位不断的加1，来造成高位不断增加的效果。  
    v = (((v | m0) + 1) & ~m0) | (v & m0);  
    /* Continue while bits covered by mask difference is non-zero */  
} while (v & (m0 ^ m1));//终止条件是 v的高位区别位没有1了，其实就是说到头了。  
```
这个遍历最主要的一行就是：

v = (((v | m0) + 1) & ~m0) | (v & m0);

下面简单分析一下它到底干了什么：

前面部分：(((v | m0) + 1) & ~m0) ， v|m0就是将v的低位全部设置为1（这里所说的低位指t0的mask覆盖的位，高位指m1相对于m0独有的位。((v | m0) + 1)后面的+1 就是将(v | m0) 的值加1，也就是给v的高位部分加1。

后面的& ~m0效果就是去掉v的前面的二进制位。最后的(v & m0) 其实就是提取出v的低位部分。两边或起来，其实语义就是：保留v的低位，高位不断加1，赋值给v；这样V能带着低位不变，高位每次加1。

循环条件v &(m0 ^ m1)，表示直到v的低m1-m0位到低m1位之间全部为0为止。
