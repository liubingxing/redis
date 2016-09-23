###Redis 中用于替代char * 类型字符串的数据结构，有以下特点：

- 动态调整字符串 buffer 大小
- 记录实际使用长度len和已分配 alloc 的 buffer 大小
- 总是 null-terminated（结尾是‘\0’)
- binary safe，即字符串中间允许出现 \0
- 搭配了丰富的 API ，如 sdsnew(), sdsempty(), sdsdup() 等 

`sds` 是 `char*` 的别名，真正存储了字符串长度、分配空间大小信息的是` sdshdr（sdsheader）`。`sds` 实际是指向 `sdshdr`最后一个成员 `buf`

`redis 3.2.3` 非常注重节省空间，根据字符串长度不同，`sdshdr` 分为五种类型，sdshdr5 用来存储5个字节可以存储的长度的字符串，最大长度为 2^5 - 1 个字符。
同理 sdshdr8 能够存储最多 2^8 - 1 个字符，以此类推。当字符串长度超出当前 `header` 所能表示的最大长度时，会自动升级 `header` 。
 
###sds内存结构：
下图是两个sds字符串的内存结构，s1使用sdshdr8类型的header，s2使用sdshdr16类型的header。有如下特点：
![image](https://github.com/liubingxing/redis/raw/master/sds.h/redis_sds_structure.png)

- sds的字符指针（s1和s2）是指向数据开始的位置，而header位于内存地址较低的方向，代码中`flags = s[-1]` 即是向低地址偏移1个字节取得flags字段。

- sdshdr5 使用了 flags 的高位5个 bits 来存储字符串的长度，低三位来标明字符串的类型，而其他长度的 hdr 使用了低三位来标明字符串类型，忽略了高5位。

- 在各个header的定义中最后有一个char buf[]。我们注意到这是一个没有指明长度的字符数组，这是C语言中定义字符数组的一种特殊写法（ flexible array member），它在这里起到一个标记的作用，表示在flags字段后面就是一个字符数组。而程序在为header分配的内存的时候，它并不占用内存空间。例如计算sizeof(struct sdshdr16)的值，结果是5个字节，其中没有buf字段。

- ____attribute____((packed)) 是是 gcc 的拓展，表示以最少內存的方式生成结构体，避免对齐造成的空间浪费。保证header和sds的数据部分紧邻，先低地址偏移1个字节能获取flags字段。

- 虽然header有多个类型，但sds可以用统一的char *来表达。且它与传统的C语言字符串保持类型兼容。如果一个sds里面存储的是可打印字符串，那么我们可以直接把它传给C函数，比如使用strcmp比较字符串大小，或者使用printf进行打印。
