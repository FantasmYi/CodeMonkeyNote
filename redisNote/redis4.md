# 数据库：   
   redis中的每个数据库，都由一个redisDB结构表示：     
  ```C      
  typedef struct redisDb {                 
  int id;       // 保存着数据库以整数表示的号码         
  dict *dict;   // 保存着数据库中的所有键值对数据,这个属性也被称为键空间（key space）      
  dict *expires;  // 保存着键的过期信息        
  dict *blocking_keys;   // 实现列表阻塞原语，如 BLPOP    
  dict *ready_keys;     
  dict *watched_keys;    // 用于实现 WATCH 命令
  } redisDb;   
  ```    
 ## 数据库的切换   
 * redis服务器初始化时，会创建出redis.h/REDIS_DEFAULT_DBNUM 个数据库，并将所有数据库保存到 redis.h/redisServer.db 数组中，每个数据库的 id 为从 0 到REDIS_DEFAULT_DBNUM - 1 的值。     
切换数据库时程序直接使用 redisServer.db[number] 来切换数据库。    
 * 而一些内部程序，如AOF,RDB，需要知道当前数据库的号码，如果没有id域，程序只能在当前使用的数据库的指针，去 redisServer.db 数组中遍历。有id域后，程序就可以通过id域了解自己在使用哪个数据库。   
 ## 键空间的操作   
 主要记录下取值：    
    在数据库中取值实际上就是在字典空间中取值，再加上一些额外的类型检查：     
         1. 键不存在，返回空回复；   
         2. 键存在，且类型正确，按照通讯协议返回值对象；   
         3. 键存在，但类型不正确，返回类型错误。    
     ![](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/image/get.png)     
      * 当客户端执行 GET message 时，服务器返回 "hello moto" 。    
      * 当客户端执行 GET not-exists-key 时，服务器返回空回复。    
      * 当服务器执行 GET book 时，服务器返回类型错误     
  
      
