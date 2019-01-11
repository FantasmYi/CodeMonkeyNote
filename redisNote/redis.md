# redis  
## 数据类型与内部数据类型   
   redis的每一种数据类型，比如字符串，列表，有序集，它们都拥有不止一种底层实现（redis内部称之为编码,encoding）,所以每当对某种数据类型的键进行操作时，程序都必须根据键所采取的编码，进行不同的操作。因此,redis构建了自己的类型系统，系统主要包括：   
     * redisObject 对象。   
     * 基于 redisObject 对象的类型检查   
     * 基于 redisObject 对象的显式多态函数   
     * 对 redisObject 进行分配、共享和销毁的机制。

## redisObject 数据结构，以及 Redis 的数据类型    
  *  redisObject 是 Redis 类型系统的核心，数据库中的每个键、值，以及 Redis 本身处理的参数，都表示为这种数据类型    
   ```C
       /*
        * Redis 对象
       */
      typedef struct redisObject {
         unsigned type:4;  // 类型
         unsigned notused:2;  // 对齐位
         unsigned encoding:4;  // 编码方式
         unsigned lru:22;   // LRU 时间（相对于 server.lruclock）
         int refcount;    // 引用计数
         void *ptr;    // 指向对象的值
      } robj;
   ```     
 *  type 记录了对象所保存的值的类型，它的值可能是以下常量的其中一个   
  ```C
   #define REDIS_STRING 0 // 字符串   
   #define REDIS_LIST 1 // 列表   
   #define REDIS_SET 2 // 集合     
   #define REDIS_ZSET 3 // 有序集     
   #define REDIS_HASH 4 // 哈希表       
  ```
 * encoding 记录了对象所保存的值的编码，它的值可能是以下常量的其中一个    
  ```C
   #define REDIS_ENCODING_RAW 0 // 编码为字符串
   #define REDIS_ENCODING_INT 1 // 编码为整数
   #define REDIS_ENCODING_HT 2 // 编码为哈希表
   #define REDIS_ENCODING_ZIPMAP 3 // 编码为 zipmap
   #define REDIS_ENCODING_LINKEDLIST 4 // 编码为双端链表
   #define REDIS_ENCODING_ZIPLIST 5 // 编码为压缩列表
   #define REDIS_ENCODING_INTSET 6 // 编码为整数集合
   #define REDIS_ENCODING_SKIPLIST 7 // 编码为跳跃表
  ```
  * ptr 是一个指针，指向实际保存值的数据结构，这个数据结构由 type 属性和 encoding 属性决定      
     例如：如果一个redisObject的type属性为REDIS_LIST，encoding属性为REDIS_ENCODING_LINKEDLIST ，那么这个对象就是一个 Redis 列表，它的值保存在一个双端链表内，而 ptr 指针就指向这个双端链表；    
     上图：   
     ![](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/image/redisObject.png)    
     
###  执行一个处理数据类型的命令时，redis执行步骤：   
    1.根据给定key，在数据库字典中查找和它对应的redisObject对象，没找到返回null。   
    2.检查redisObject的type属性是否和执行命令所需的类型相符，不符返回类型错误。   
    3.根据redisObject的encoding属性所指定的编码，选择合适的操作函数处理底层的数据结构。    
    4.返回结果作为命令的返回值    

* [内部数据结构](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/redisNote/redis1.md)    
* [内部映射数据结构](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/redisNote/redis2.md)    
* [五大数据类型](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/redisNote/redis3.md)     


# 参考资料   
  * redis设计与实现（第一版） 黄健宏    
