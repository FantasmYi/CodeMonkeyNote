# 1-redis
 ## 内部数据结构   
 redis从使用者的层面来说，有5种数据类型，String,list,hash,set,sortset   
 从内部实现的层面（更底层的实现）来说，有多种数据类型，sds,字典，双端链表及笔记2中提到的内部映射数据结构等.   
 例如：String类型内部实现为sds，hash对象的编码之一为压缩列表，list类型内部实现之一为双端链表，set类型内部实现之一为intset等
**简单动态字符串（sds）**
* sds是redis底层所使用的字符串表示，它被用在几乎所有redis模块中，sds可以高效地实现追加和长度计算，且二进制安全  
  在redis中，aof缓存，返回给客户端的回复等内容都是由sds类型保存的  
 * 作用：  
             1.实现字符串对象   
             2.在redis程序内部用作char* 的替代品
*  sds实现：
      ```C
        typedef char *sds;                                                                                 
        struct sdshdr {                                                                                   
        int len;     // buf 已占用长度 
        int free;    // buf 剩余可用长度
        char buf[];  // 实际保存字符串数据的地方
       }; 
      ```
* sds的追加操作  
  首先set命令创建并保存hello world到一个sdshdr中 如下  
   ```C
  struct sdshdr {  
      len = 11;  
      free = 0;  
      buf = "hello world\0";  \\buf的实际长度为len+1 (\0算一个)
   }
  ```
  追加一个字符串 again  
  ```C  
  struct sdshdr {
       len = 18;
       free = 18;
       buf = "hello world again!\0 "; // 空白的地方为预分配空间，共 18 + 18 + 1 个字节
    }
  ``` 
    注：如果新字符串总长度小于1MB，为字符串分配2倍于所需长度的空间，否则分配所需长度+1MB的空间  
    目的是将来再次对同一个sdshdr进行追加操作，只要追加内容的长度不超过free的值，就不需要对buf进行内存重分配。        
  
 **双端链表**
 *  应用场景：  
    1.实现列表类型（重点）        
    2.事务模块使用双端链表来按顺序保存输入的命令  
    3.服务器模块使用双端链表来保存多个客户端  
    4.订阅/发送模块使用双端链表来保存订阅模式的多个客户端  
 *  实现：  
    由listNode（双端链表的节点）和list（双端链表的本身）两个数据结构构成  
 *  特性：  
    节点带有前驱和后继指针，访问前驱和后继节点的复杂度O(1)，且对链表的迭代可以从表头到表尾和从表尾到表头两个方向进行  
    对表头和表尾进行处理的复杂度为O(1)  
    O(1)时间内返回链表节点数量  
    注：所有特性由来可看listNode和list的定义  
   ```C
     typedef struct listNode {
        struct listNode *prev;  // 前驱节点
        struct listNode *next;  // 后继节点
        void *value; // 值
     } listNode;
     typedef struct list {
        listNode *head;  // 表头指针
        listNode *tail;  // 表尾指针
        unsigned long len;  // 节点数量
        void *(*dup)(void *ptr);  // 复制函数
        void (*free)(void *ptr);  // 释放函数
        int (*match)(void *ptr, void *key);  // 比对函数
     } list;
   ```
**字典**  
  * 又名映射或关联数组，由一集键值对组成  
  * 主要用途：  
    1.实现数据库键空间   
      redis中的键由字典保存，每个数据库都有与之对应的字典，称为键空间    
      维护key和value的映射关系   
    2.用作Hash类型键的一种底层实现  
      程序创建新Hash键时，默认使用压缩列表作为底层实现，到某个条件时，压缩列表转换为字典  
  * 实现方式：  
     1.链表或数组  
     2.哈希表 redis选择哈希表作为字典的底层实现  
     3.平衡树    
 ```C
    typedef struct dict {
      dictType *type;  // 特定于类型的处理函数
      void *privdata;  // 类型处理函数的私有数据
      dictht ht[2];  // 哈希表（2 个）
      int rehashidx;  // 记录 rehash 进度的标志，值为-1 表示 rehash 未进行
      int iterators;  // 当前正在运作的安全迭代器数量
} dict;
 ```
    注意：一个字典用2个哈希表，0号是字典主要使用的哈希表，1号只有在程序对0号哈希进行rehash时才使用
  *  添加键值到字典  
     1.字典为空，那么需要对0号哈希表初始化  
     2.添加新键值对时发生碰撞处理（链地址法、使用链表将多个哈希值相同的节点串联在一起）   
     3.添加新键值对时触发了rehash操作 
        解释：为了在字典的键值对不断增多的情况下仍然保持良好的性能，字典需要对所使用的哈希表进行rehash操作，即对哈希表进行扩容。   

 **跳跃表**      
  *  主要用途：  
       实现有序集数据类型   
  *  实现方式：  
       跳跃表将指向有序集的score值和member域的指针作为元素，以score值（可重复）作为索引，对有序集进行排序。注：对比同一个元素需要同时检查它的score和member。每个节点带有高度为1的指针，可以实现从表尾到表头方向迭代。        
      ![image](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/JumpTable.png)    
      * 表头：负责维护跳跃表的节点指针  
      * 跳跃表节点：保存着元素值，以及多个层  
      * 层：保存着指向其他元素的指针。高层的指针越过的元素数量大于等于低层的指针，为了提高查找的效率，程序总是从高层先开始访问，然后随着元素值范围的缩小，慢慢降低层次  
      * 表尾：全部由 NULL 组成，表示跳跃表的末尾
          
