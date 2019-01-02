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
 ## 数据库的切换（id）       
 * redis服务器初始化时，会创建出redis.h/REDIS_DEFAULT_DBNUM 个数据库，并将所有数据库保存到 redis.h/redisServer.db 数组中，每个数据库的 id 为从 0 到REDIS_DEFAULT_DBNUM - 1 的值。     
切换数据库时程序直接使用 redisServer.db[number] 来切换数据库。    
 * 而一些内部程序，如AOF,RDB，需要知道当前数据库的号码，如果没有id域，程序只能在当前使用的数据库的指针，去 redisServer.db 数组中遍历。有id域后，程序就可以通过id域了解自己在使用哪个数据库。   
 ## 数据库键空间（dict）及键空间的操作                              
 * redis本身是一个键值对数据库，所以它的数据库本身也是一个字典    
    * 字典的键是一个字符串对象    
    * 字典的值可以是五大数据类型中的任意一种     
 * 看下图，redisDb结构体内的属性，dict保存着所有键值对数据     
 * 主要记录下取值：    
    在数据库中取值实际上就是在字典空间中取值，再加上一些额外的类型检查：     
         1. 键不存在，返回空回复；   
         2. 键存在，且类型正确，按照通讯协议返回值对象；   
         3. 键存在，但类型不正确，返回类型错误。        
     ![](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/image/get.png)     
      * 当客户端执行 GET message 时，服务器返回 "hello moto" 。    
      * 当客户端执行 GET not-exists-key 时，服务器返回空回复。    
      * 当服务器执行 GET book 时，服务器返回类型错误     
## 键的过期时间（expires）    
 * 通过expire,pexpire,expireat,pexpireat四个命令，客户端可以给某个存在的键设置过期时间，当键的过期时间到达时，键就不在可用。   
 * 命令TTL和PTTL则用于返回给定键距离过期还有多长时间    
 ### 过期时间的保存及判定   
 * 在数据库中，所有键的过期时间都被保存在redisDb的expire字典里。expire字典的键是一个指向键空间里某个键的指针，字典的值是键所指向的数据库键的到期时间，值以long long类型表示，如图：       
 ![](https://github.com/FantasmYi/CodeMonkeyNote/blob/master/image/expire.png)     
 * 通过expire字典，可以检查某个键是否过期    
    1. 检查键是否存在于expire字典；如果存在，取出键的过期时间。    
    2. 检查当前UNIX时间戳是否大于键的过期时间，如果是，则键已经过期；   
 ### 过期键的清除    
   * 定时删除：在设置键的过期时间时，创建定时事件，当过期时间到达时，由事件处理器自动执行删除键的操作。缺点：占用大量CPU时间。    
   * 惰性删除：每次从键空间中取出值时，都要检查这个键是否过期，过期就删除，返回Null，没过期就返回键值。缺点：对内存不友好。   
   * 定期删除：每隔一段时间，对expire字典进行检查（抽查），删除里面的过期键。      
   * redis采用惰性+定期删除策略。    
   * 实现惰性删除策略的核心是db.c/expireIfNeeded函数---所有命令在读取和写入之前，都会调用这个函数对输入键进行检查。expireIfNeeded的作用是：如果输入的键已过期，那么将键、值保存在expire中的过期时间及键值对全部删掉。伪代码描述：    
   ```C
   def expireIfNeeded(key):
   # 对过期键执行以下操作 。。。
   if key.is_expired():
   # 从键空间中删除键值对
   db.dict.remove(key)
   # 删除键的过期时间
   db.expires.remove(key)
   # 将删除命令传播到 AOF 文件和附属节点
   propagateDelKeyToAofAndReplication(key)
   ```     
   * 实现定期删除策略的核心是redis.c/activeExpireCycle函数执行---每当redis的例行处理程序serverCron执行时，该函数都会被调用，这个函数在限定时间内，尽可能的遍历各个数据库的expires字典，随机地抽查一部分键的过期时间，并删除其中的过期键。伪代码描述： 
   ```C
   def activeExpireCycle():
   # 遍历数据库（不一定能全部都遍历完，看时间是否足够）
   for db in server.db:
   # MAX_KEY_PER_DB 是一个 DB 最大能处理的 key 个数
   # 它保证时间不会全部用在个别的 DB 上（避免饥饿）
   i = 0
   while (i < MAX_KEY_PER_DB):
     # 数据库为空，跳出 while ，处理下个 DB
     if db.is_empty(): break
     # 随机取出一个带 TTL 的键
     key_with_ttl = db.expires.get_random_key()
     # 检查键是否过期，如果是的话，将它删除
     if is_expired(key_with_ttl):
       db.deleteExpiredKey(key_with_ttl)
     # 当执行时间到达上限，函数就返回，不再继续,这确保删除操作不会占用太多的 CPU 时间
     if reach_time_limit(): return
     i += 1
   ```   
###  过期键对AOF、RDB和复制的影响     
 * 在创建新的RDB文件时，程序会对键进行检查，过期的键不会被写入到更新后的RDB文件中。因此，过期键不会对更新后的RDB文件有影响。    
 * 当过期键被惰性删除、或者定期删除之后，程序会向 AOF 文件追加一条 DEL 命令，来显式地记录该键已被删除。
 
### 小结
 * expire的某个键和dict的某个键共同指向同一个字符串对象。    
 * 过期的键对更新后的AOF和重写后的RDB没有影响    
 
   
   
 
      
