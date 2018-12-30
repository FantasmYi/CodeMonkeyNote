# 五大数据类型    
   ## 字符串      
 redis中使用最为广泛的数据类型，是除了set、get等命令的操作对象之外，数据库中所有的键，以及执行命令时提供给redis的参数，都是用这种类型保存的。    
 * 字符串编码：    
   ![](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/image/StringEncoding.png)    
    由图：字符串类型使用两种编码类型   
          1. REDIS_ENCODING_INT 使用 long 类型来保存 long 类型值    
          2. REDIS_ENCODING_RAW 则使用 sdshdr 结构来保存 sds （也即是 char* )、long long 、double 和 long double 类型值     
    注：新创建的字符串默认使用REDIS_ENCODING_RAW，在将字符串作为键或值保存到数据库是会尝试转换成REDIS_ENCODING_INT编码。   

   ## 哈希表   
   ![](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/image/hashEncoding.png)     
   * 由图：哈希表使用两种编码类型   
          1. REDIS_ENCODING_ZIPLIST  压缩列表   
          2. REDIS_ENCODING_HT  字典    
   * 当哈希表使用字典编码时，程序将哈希表的键保存为字典的键，将哈希表的值保存为字典的值。  
   * 哈希表所使用的键和值都是字符串对象    
   * 当使用 REDIS_ENCODING_ZIPLIST 编码哈希表时，程序通过将键和值一同推入压缩列表，从而形成保存哈希表所需的键 -值对结构,新添加的 key-value 对会被添加到压缩列表的表尾。当进行查找/删除或更新操作时，程序先定位到键的位置，然后再通过对键的位置来定位值的位置。    
   * 编码的选择：默认选择压缩列表   
       1. 当某个键或某个值的长度大于64，压缩列表转化为字典     
       2. 压缩列表中的节点数量大于512，压缩列表转化为字典    

   ## 列表   
   ![](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/image/listEncoding.png)    
   * 由图：列表使用两种编码类型   
          1. REDIS_ENCODING_ZIPLIST 压缩列表   
          2. REDIS_ENCODING_LINKEDLIST 双端链表   
   * 编码的选择及转换：同hash   
   * 阻塞：BLPOP 、BRPOP 和 BRPOPLPUSH 三个命令都可能造成客户端被阻塞    
       ![](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/image/listBlock.png)    
       由上图：这些命令被用于空列表时，会阻塞客户端    

   ## 集合   
   ![](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/image/setEncoding.png)    
   * 由图：集合使用两种编码类型，第一个添加到集合的元素，决定了创建集合时所使用的编码   
          1. REDIS_ENCODING_INTSET  如果第一个元素是一个整数，那么集合初始编码是intset    
          2. REDIS_ENCODING_HT  否则初始编码是字典   
   * 如果一个集合使用 REDIS_ENCODING_INTSET 编码，那么当以下任何一个条件被满足时，这个集合会被转换成 REDIS_ENCODING_HT 编码：     
          1. intset 保存的整数值个数超过 server.set_max_intset_entries （默认值为 512 ）。    
          2. 试图往集合里添加一个新元素，并且这个元素不能被表示为 long long 类型（也即是，它不是一个整数）。     

   ## 有序集合   
   ![](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/image/sortSetEncoding.png)    
   * 编码选择: 在通过 ZADD 命令添加第一个元素到空 key 时，程序通过检查输入的第一个元素来决定该创建什么编码的有序集    
         如果第一个元素符合以下条件的话，就创建一个 REDIS_ENCODING_ZIPLIST 编码的有序集：    
             • 服务器属性 server.zset_max_ziplist_entries 的值大于 0 （默认为 128 ）。
             • 元素的 member 长度小于服务器属性 server.zset_max_ziplist_value 的值（默认为 64）。     
   * 编码转换：同hash     
   * ZIPLIST 编码的有序集：  
          1. 当使用 REDIS_ENCODING_ZIPLIST 编码时，有序集将元素保存到 ziplist 数据结构里面   
          2. 每个有序集元素以两个相邻的 ziplist 节点表示，第一个节点保存元素的 member 域，第二个元素保存元素的 score 域  
          3. 多个元素之间按 score 值从小到大排序，如果两个元素的 score 相同，那么按字典序对 member进行对比，决定那个元素排在前面，那个元素排在后面。    
   * SKIPLIST 编码的有序集：   
          ```C       
 typedef struct zset {      
    dict *dict;   // 字典  方便按成员查找等       
    zskiplist *zsl;    // 跳跃表  方便顺序查找等             
 } zset;         
          ```         
       用结构体可见，zset 同时使用字典和跳跃表两个数据结构来保存有序集元素。  

# 参考资料   
  * redis设计与实现（第一版） 黄健宏          



   
   
