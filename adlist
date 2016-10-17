typedef struct list{    
   listNode *head;   //头节点    
   listNode *tail;   //尾节点    
   unsigned long len;    //节点个数    
   void *(*dup)(void *ptr); //复制ptr内容函数    
   void (*free)(void *ptr);  //释放ptr值函数    
   int (*match)(void *ptr，void *key); //匹配ptr和key内容的函数    
}list;  
其中dup和free，match函数可用于自定义函数；
在adlist.c中有类似如下代码段：
if(list->free) list-free(node->value);
else zfree(node);
表示如果有自定义的free函数，则调用，否则用zfree函数。
