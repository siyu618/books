redis 设计与实现

***Chapter 2 简单动态字符串***

2.1 SDS（Simple Dynamic String）定义
   ```c
   sds.h/sdshdr
   struct sdshrd {
      int len; // 记录buf中已经使用的字节的数量，等于sds所保存字符串的长度
      int free; // 记录buff中未使用的字节的数量
      char buf[]; // 字节数组，用于保存字符串，总长度：len + free + 1 （'\0'）
   }
   ```

2.2 SDS 与C字符串的区别
   * STRLEN 常数复杂度获取字符串长度 O(1)
   * 杜绝缓冲区溢出，strcat首先看时候能否放得下，不能则先扩容
   * 减少修改字符串是带来的内存重分配次数
      * 空间预分配
         1. 对sds修改之后，如果sds长度（len属性）小于1MB，则程序分配与len属性相同大小的未使用空间，则buf长度为 len + len + 1
         1. 如果长度大于1MB， 则分配1MB未使用空间，buf长度为 len + 1MB + 1 byte
      * 惰性回收
   * 二进制安全
   * 兼容部分c函数


***Chapter 3 链表*** 

3.1 链表和链节点的实现
   ```
   adlist.h/listNode
   typedef struct listNode {
      struct listNode *prev; // 前置节点
      struct listNode *next; // 后置节点
      void *value;  // 节点的值
   } listNode;
   ```
   ```
   typedef struct list {
      listNode *head; // 头节点
      listNode *tail; // 尾节点
      unsigned long len; // 链表包含的节点数
      void* (*dup)(void *ptr); // 节点值复制函数
      void (*free)(void *ptr); // 节点值释放函数
      int (*match)(void *ptr, void *key); // 节点值对比函数
   } list;
   ```
   * 特性
      1. 双端链表
      1. 无环
      1. 带有头指针和尾指针
      1. 带链表长度计数器  
      1. 多态
   * 使用场景
      * 列表键，发布订阅，慢查询，监视器

***Chapter 4 字典***

4.1 字典的实现
   ```
   dict.h/dictht
   typedef struct dictht {
      dictEntry **table; //哈希表数组
      unsigned long size; //哈希表大小，table数组的大小
      unsigned long sizemask; //哈希表大小掩码，用于计算索引值，值总为 size-1
      unsigned long used; // 该哈希表已有的节点的数量
   } dictht;
   ```
   ```
   dict.h/dictEntry
   typedef struct dictEntry {
      void *key; // key
      union {
         void * val; //
         uint64_t u64; //
         int64_t s64; //
      } v; //   value
      struct dictEntry *next; // next
   } dictEntry;
   ```
   ```
   dict.h/dict
   typedef struct dict {
      dictType *type; // 特定类型函数
      void *privdata; // 似有数据
      dictht ht[2]; // 哈希表，平时只有ht[0]在使用，进行rehash时，两个同时使用
      int rehashidx; // rehash索引，没有在进行rehash时，值为-1；

   } dict;
   ```
   * type属性和privadata属性是针对笔筒类型的键值对，为创建多态字典而建设的
      * type属性是一个指向dictType结构的指针，每个dictType结构保存了一簇用于操作特定类型键对值的函数，redis会为不同用途不同的字典设置不同类型的特定函数
      * privdata属性则保存了需要传给那些特定类型函数的可选项
   ```
   typedef struct dictType {
      // 计算hash值的函数
      unsigned int (*hashFunction)(const void *key);
      // 复制key的函数
      void* （*keyDedup)(void *privdata, const void *key);
      // 复制value的函数
      void* (*valDup)(void *privdata, const void *obj);
      // 对比key的函数
      int (*keyCompare)(void *privdata, const void *key1, const void *key2);
      // 销毁key的函数
      void (*keyDestructor)(void *privdata, void *key);
      // 销毁value的函数
      void (*valDestructor)(void *privdata, void *obj);
   } dictType;
   ```

4.2 hash算法
   * 使用MurmurHash2哈希算法
   ```
   hash = dict->type->hashFunction(key);
   index = hash & dict->ht[x].sizemask;
   ``` 
 
4.3 解决key冲突
   * 采用链地址法（separate chaining）

4.4 rehash
   * rehash 步骤：
      1. 为字典的ht[1]哈希表分配空间，这个哈希表的大小取决于要执行的操作，以及ht[0]当前包含的键值对的数量（即ht[0].userd）    
         * 如果执行的是扩展操作，则ht[1]的大小为第一个大于等于ht[0].used * 2 的pow(2,n)
         * 如果执行的是收缩操作，那么ht[1]的大小为第一个大于等于ht[0].used的pow(2,n)
      1. 将保存在ht[0]中的所有的键值对rehash到ht[1]上面
         * rehash是指重新计算key的hash和index，然后将键值对放置到ht[1]中指定位置上
      1. 当ht[0]包含的所有键值对都迁移到ht[1]之后（ht[0]为空），释放ht[0], 将ht[1]设置为ht[0]，并在ht[1]新建一个空白hash表，为下一次rehash做准备
   * hash表的扩展与收缩
      * 扩展操作发生的条件（任一满足即可）：load_factor = ht[0].used / ht[0].size
         1. 服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载引子大于等于1
         1. 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令， 并且哈希表的负载引子大于等于5
   * 渐进式rehash
      * rehash不是一次性、集中式地完成的，而是分多次、渐进式的完成的
         * 原因在于，如果hash表很到的话，rehash需要很长时间，可能知道服务器在一段时间内停止服务
      * 步骤
         1. 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个hash表
         1. 在字典中维持一个索引计数器变量rehashidx， 并将其设置为0，表示rehash开始工作
         1. 在rehash进行期间，每次对字典执行添加、删除、查找或更新操作时， 程序除了执行置顶的操作意外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash工作完成之后，程序将rehashidx属性的值增1.
         1. 随着字典操作的部队执行，最终在某个时间点上， ht[0]的所有键值对都被rehash到ht[1]，这时程序将rehashidx属性设为-1，表示rehash操作结束
      * rehash执行期间的哈希表操作
         * 字典的delete、find、updatedd等操作会在两个哈希表上进行
   * 字典用于数据库和哈希键

***Chapter 5 跳跃表***

   跳跃表（skiplist） 是一种有序数据结构，通过在每个几点中维持多个指想其他节点指针，从而达到快速访问的目的。
   跳跃表支持平均O(logN)、最坏O(N)复杂度的节点查找，还可以通过顺序性来批量处理节点
   redis 中使用skiplist的地方：实现有序集合键、集群节点中用于内部数据结构

5.1 跳跃表的实现
   ```
   redis.h/zskiplistNode
   typedef struct zskiplistNode {
      struct zskiplistLevel {
        struct zskiplistNode *forward; // 前进执行
        unsigned int span; // 跨度
      } level[]; // 层 ： 一般来说层越多，方悦其他节点的速度就越快
      struct 在skiplistNode *backward; //后推指针
      double score; // 分值
      robj *obj;   // 成员对象
   } zskiplistNode;
   ```
   ```
   typedef struct zskiplist {
       struct zskiplistNode *head;
       struct zskiplistNode *tail;
       unsigned long length; // 节点数量
       int level; // 表中层数最大的节点数量
   } zskiplist;

   ```

***Chapter 6 整数集合***

6.1 整数集合的实现
   ```
   typedef struct intset {
       uint32_t encoding; // 编码方式
       uint32_t length; // 集合中包含元素数量
       int8_t contents[] // 保存元素的数组, 其中元素是有序的
   } intset;
   ```

6.2 升级
   * 当加入的新元素的类型比整数集合现有的类型都养长时，整数集合需要先进行升级（upgrade）
   * 升级的步骤
      1. 根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间。
      1. 将底层数组现有的所有元素都转换与新元素相同的类型，并将类型转换后的元素放置到正确的位上，而且在防止元素的过程中，需要继续维持底层数组的有序性值不变
      1. 将新元素添加到底层数组里面
   * 升级的好处
      * 提升灵活性
      * 节约内存

6.3 降级
   * 不支持降级操作！！！


***Chapter 7 压缩列表***   

   ziplist是列表键和哈希键的底层实现之一。
   其目的是redis为了节约内存而开发的， 是有一系列特殊编码的连续内存组成的顺序型（sequential）数据结构。

7.1 压缩列表构成
   * |zlbytes|zltail|zllen|entry1|entry2|...|entryN|zlend|
      * zlbytes：记录增个压缩列表占用的内存字节数
      * zltail：记录压缩列表尾节点距离压缩列表的其实地址有多少字节，程序可通过该变量快速定位尾节点
      * zllen：记录压缩列表包含的节点数量
      * entryX：压缩列表包含的各个几点，
      * zllen：一个字节，值为0XFF，用于标记压缩列表的末端

7.2 压缩列表节点的构成
   * |previous_entry_length|encoding|content|   
      * previous_entry_length : 一个字节或者五个字节
         * 长度小于254， 一个字节来存储
         * 长度大于等于254，5个字节来存储，0XFE + length
      * encoding：记录了数据类型及长度
         * 一字节、两字节或者五字节长
      * content：保存的是节点的值

7.3 连锁更新
   * 因为previous_entry_length 存在变长的问题，可能引起连锁的更新
      * 在添加或者删除的时候

***Chapter 8： 对象***      

8.0 
   * Redis 没有直接使用上面的数据结构来实现数据库，而是基于这些数据结构创建了一个对象系统，这个系统包括字符串对象、列表对象、集合对象和有序集合对象。
   使用对象的好处：
      * 判定一个命令是否可以执行
      * 针对不同的使用场景，为对象设置多种不同的数据结构实现，从而优化对象在不同场景下的使用效率
   * redis对象实现了基于引用计数的内存回收机制 + 对象共享机制
   * redis对象带有访问时间记录信息，噶信息可以使用计算数据库键d额空转时长，在服务器用了maxmemory功能的情况下，空转时长较大的那些键可能会被服务器优先删除

8.1 对象的类型与编码
   * 新建键值对的时候，会创建键对象 + 值对象
   ```
   typedef struct redisObject {
      unsigned type:4; // 类型
      unsigned encoding:4; // 编码
      void *ptrl; // 执行底层实现数据结构的指针
      int refcount; // 引用计数
      unsigned lru:22; // 最后一次被访问的时间
      ...

   } robj;
   ```   
   * type
    | 对象|对象的type属性的值|type 命令的输出|
    -------------------------------------
    |字符串对象|REDIS_STRING|string|
    |列表对象|REDIS_LIST|list|
    |哈希对象|REDIS_HASH|hash|
    |集合对象|REDIS_SET|set|
    |有序集合对象|REDIS_ZSET|zset|
   * 编码和底层实现
    |编码常量|编码所对应的底层数据结构|
    -----------------------------
    |REDIS_ENCODING_INT|long类型的整数| 
    |REDIS_ENCODING_EMBSTR|embstr编码的简单动态字符串| 
    |REDIS_ENCODING_RAW|简单动态字符串| 
    |REDIS_ENCODING_HT|字典| 
    |REDIS_ENCODING_LINKEDLIST|双端链表| 
    |REDIS_ENCODING_ZIPLIST|压缩列表| 
    |REDIS_ENCODING_INTSET|整数集合| 
    |REDIS_ENCODING_SKIPLIST|跳跃表和字典| 

    |类型|编码|对象|object encoding输出|
    --------------
    |REDIS_STRING|REDIS_ENCODING_INT|使用整型值实现的字符串对象|"int"|
    |REDIS_STRING|REDIS_ENCODING_EMBSTR|使用embstr编码的简单动态字符串实现的字符串对象|"embstr"|
    |REDIS_STRING|REDIS_ENCODING_RAW|使用简单动态字符串实现的字符串对象|"raw"|
    |REDIS_LIST|REDIS_ENCODING_ZIPLIST|使用压缩列表实现的列表对象|"ziplist"|
    |REDIS_LIST|REDIS_ENCODING_LINEDLIST|使用双端链表实现的列表对象|"linkedlist"|
    |REDIS_HASH|REDIS_ENCODING_ZIPLIST|使用压缩列表实现的哈希对象|"ziplist"|
    |REDIS_HASH|REDIS_ENCODING_HT|使用字典表实现的哈希对象|"hashtabel"|
    |REDIS_SET|REDIS_ENCODING_INTSET|使用整数集合实现的集合对象|"intset"|
    |REDIS_SET|REDIS_ENCODING_HT|使用字典实现的集合对象|"hashtable"|
    |REDIS_ZSET|REDIS_ENCODING_ZIPLIST|使用压缩列表实现的有序集合对象|"ziplilst"|
    |REDIS_ZSET|REDIS_ENCODING_SKIPLIST|使用跳跃表+字典表实现的有序集合对象|"skiplist"|
 
8.2 字符串对象
    * 编码可以是 int、embstr（<=32字节）或者raw（>32字节）
    * 对于raw编码，会调用两次内存分配来分别创建redisObject和sdshdr
    * 对于embstr则会调用一次内存分配函数来分配一块连续的内存（redisObject|sdshdr）
    * 编码转换规则
       * int -> str, 则直接用raw
       * embstr 不可变，embstr -> raw

8.3 列表对象
   * 编码可以是ziplist或者linkedlist
   * 编码转化
      * 同时满足以下两个条件时，使用ziplist编码  （限定值是可以修改的）
         * 列表对象报错的所有字符串元素的长度都小于64字节
         * 列表对象保存的元素量小元素512

8.4 哈希对象
   * 哈希队形的编码可以是ziplist或者hashtable
   * 编码转换
      * 当哈希对象可以同时满足以下两个条件是，哈希对象使用ziplist编码 （数值可配置）
         1. 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节
         1. 哈希对象保存的键值对数量小于512个

8.5 集合对象
   * 结合对象的编码可以是intset或者hashtable
   * 编码转换
      * 当结合对象可以同时满足以下两个条件时，对象使用intset编码：
         1. 集合对象保存的所有元素都是整数
         1. 节后对象保存的元素数量不超过512

8.6 有序结合对象
   * 有序结合的编码可以是ziplist或者skiplist
   * zset
   ```
   typedef struct zset {
       zskplist *zsl; // zrange, zrank
       dict *dict; // zscore
   } zset;
   ```
   * 编码的转换
      * 当有序集合对象可以同时满足以下两个条件时，对象使用ziplist编码 （具体的数字可以修改）
         1. 有序集合保存的元素数量小于128个
         1. 有序结合保存的所有元素成员的长度小于64字节

8.7 类型检查与命令多态
   * 类型检查的实现
   * 多态命令的实现

8.8 内存回收
   * redisObject： int refcount; // 引用计数

8.9 对象共享
   * redis会共享值为0到9999的字符串对象

8.10 对象的空转时长
   * lru:22 记录了对象的 最后一次被访问的时间 #object idletime 命令不会更新该时间
   * 空转时长 = 当前时间 - lru

### 第二部分：单机数据库的实现 ###

***第9张：数据库***

9.1 服务器中的数据库
   * redisServer
   ```
   typedef struct redisServer {
       redisDb *db;  // 数组，保存服务器中的所有数据库
       int dbnum;    // 服务器的数据库数量
   } redisServer;
   ```

9.2 切换数据库   
   * cmd： SELECT 0 # default is 0
   * redisClient中保存了目标数据库的信息
   ```
   typedef struct redisClient {
      redisDb *db; // 记录客户端当前在使用的数据库
   } redisClient;
   ```

9.3 数据库键空间
   * redisDb
   	```
   	typedef struct redisDb {
   	    dict *dict;
   	} redisDb;
   	```
   * 操作
      * 添加：SET
      * 删除：DEL
      * 更新：SET、HSET
      * 取值：GET、HGET
      * 其他：FLUSHDB
   * 读写键空间的维护
      * 读取一个键之后，会根据键是否存在来更新服务器的键空间命中次数（hit）和不命中次数（miss），可在info stats中查看
      * 读取一个键之后，会更新键的LRU时间，可以计算出键的空闲时间
      * 读取一个键的时候发现键已过期则先删除该键，然后执行后续操作
      * 如果有客户端使用watch命令监视了某个键，那么服务器在对键修改之后会将这个键标记为脏（dirty），从而让事务程序知道该键已经修改过
      * 服务器每次修改键之后，都会对脏（dirty）键计数器+1，该计数器会触发服务器的持久化以及复制操作
      * 如果服务器开启了数据控通知功能，那么在对键进行修改之后，服务器将按照配置发送相应的数据库通知。

9.4 设置键的生存时间和过期时间
   * 设置过期时间
      * EXPIRE <key> <ttl>
      * PEXPIRE <key> <ttl>
      * EXPIREAT <key> <timestamp>
      * PEXPIREAT <key> <timestamp>
   * 保存过期时间
      * redisDb中的过期字典，保存键的过期时间
      ```
      typedef struct redisDb {
         dict *expires; // map<string, long long>
      } redisDb;
      ```
   * 移除过期时间
      * PERSIST <key>； // 从redisDb.expireDB 中移除，如果有的话
   * 计算并返回剩余生存时间
   * 过期键的判定

9.5 过期键的删除
   * 常见的删除策略
      1. 定时删除：在设置建的过期时间是，创建一个timer，时间到的时候立即删除
      1. 惰性删除：放任键过期不管，但每次从键空间获取时，都检查键是否过期，如果过期就删除，没有过期则返回
      1. 定期删除：每个一段时间，程序就对数据库进行一次检查，删除里面的键。至于要删除多少键和检查多少数据库由算法决定。
   * 定时删除 
      * 是主动删除
      * 对内存友好，对CPU不友好
   * 惰性删除
      * 是被动删除
      * 对CPU友好，对内存不友好
      * 可能会有空间一直得不到释放
   * 定期删除
      * 是前面两者的整合和折中
         * 定期删除策略每个一段时间执行一次删除过期键操作，通过限制每次操作的时长和批量练减少删除操作对cpu时间的影响
         * 有效减少了内存浪费
      * 难点
         * 时长
         * 频率

9.6 redis的过期键删除策略
   * redis使用惰性删除和定期删除两种策略
   * expiredIfNeeded
   * 定期删除策略的实现
      * 策略：redis.c/activeExpireCycle函数实现
      * 每当服务周期性操作redis.c/serverCron函数执行时activeExpireCycle就会被调用

9.7  AOF、RDB和复制功能对过期键的处理
   * 生成RDB文件：SAVE、BGSAVE创建一个新的rdb文件， 过期键不会被写入
   * 载入RDB文件
      * 主从模式的主，载入rdb文件不会载入过期键
      * 主从模式的从，会载入过期键
   * AOF文件写入
      * 键删除之后会写入AOF文件
   * AOF重写
      * 已过期键不会写入
   * 复制
      * 当服务器运行在复制模式下时，从服务器的过期键删除动作由主服务器控制
         * 主服务器在删除一个过期键之后，会显示地向从服务器发送一个DEL命令
         * 从服务器在执行客户端发送的命令时，及时碰到过期键也不会删除，而是继续像处理未过期的键一样
         * 从服务器之后在接受到主服务器发来DEL命令之后，才会删除key

9.8 数据库通知
   * SUBSCRIBE __keyspace@0__:message
   * SUBSCRIBE __keyspace@0__:del


***第10章：RDB持久化***

10.1 RDB文件的创建和载入
   * SAVE 
      * 调用rdbSave()
      * 执行期间redis会被阻塞，客户端请求会被拒绝
   * BGSAVE
     * fork子进程来做
        * 子进程调用rdbSave()，然后signal_parent()
        * 父进程继续处理命令，并且通过轮询等待子进程信号
   * RDB文件载入时，服务器会一直阻塞

 10.2 自动间隔性保存
    * 配置，任意满足则触发bgsave
    ```
    save 900 1
    save 300 10
    save 60 10000
    redisSever{
       struct saveparam * saveparams;// 记录了保存条件数组
       long long dirty; //修改计数器，距离上次成功执行save或者bgsave之后服务器对于数据库状态（服务器中的所有数据库）进行多少次修改
       time_t lastsave;// 上次成功执行save或者bgsave的时间
    }
    typedef struct saveparam {
       time_t seconds;
       int changes;
    } saveparam;
    ```
    * 检查保存条件是否满足
       * serverCront默认每个100ms就会执行一次，该函数用于对正在运行的服务器进行维护，其中一项就是检查条件是否满足，如果满足则进项bgsave

 10.3 RDB文件结构
    * overall
    |REDIS|db_version|databases|EOF|checksum|
    * databases
    |SELECTDB|db_number|key_value_pairs|
    * key_value_pairs 
    |TYPE|key|value| 
    |EXPIRETIME_MS|ms|TYPE|key|value|
    * value 的编码
       * 字符串对象：|ENCODING|val|，有压缩和不压缩的差别
       * 列表对象 |list_length|item1|item2|...|itemN|
       * 集合对象：|set_size|elem1|elem2|...|elemN|
       * 哈希表对象：|hash_size|key_value_pair1 |key_value_pair2|...|keyvalue_pairN|
       * 有序集合对象：|sorted_set_size|elemtn1|elemtnt2|...|elementN|
       * intset编码的集合：整数集合字符串化
       * ziplist编码的列表、哈希表或者有序结合
          * 将压缩列表转换成一个字符串对象

10.4 分析RDB文件
   * od -cx dump.rdb

***Chapter 11: AOF持久化***

11.1 AOF （Append Only File）持久化的实现   
   * 命令追加
      * 执行写命令之后会以协议格式将被执行的命令追加到服务器的aof_buf缓存区的末尾
   ```
   struct redisServer {
       sds aof_buf; // AOF缓冲区
   };
   ```
   * AOF问价的写入与同步
   ```
   def eventLoop():
      while Ture:
         # 处理文件事件、接受命令请求以及发送命令回复
         # 处理命令请求时可能有新内容被最爱到aof_buf缓冲区中
         processFileEvents()

         # 处理事件事件
         processTimeEvents()

         # 考虑是否要将aof_buf中内容写入和保存到AOF文件里面
         flushAppendOnlyFile()
   ```
   * appendfsync 选项
      * always: 将aof_buf缓冲区中的所有内容写入并同步到AOF文件
      * everysec：将aof_buf缓冲区中的所有内容写入到AOF问价，如果上次同步AOF文件的时间鞠距离现在超过一秒钟，那么再次对AOF文件进行同步，并且这个同步由一个线程专门负责执行
      * no：将aof_buf缓冲区的所有内容写入AOF文件，但是何时同步将由操作系统同步
   * 文件的写入与同步
      * 现在OS中write()函数仅将内容写入缓冲区，等到缓冲区被填满或者超过指定的时限之后，才能将缓冲区中的内容写入磁盘
      * 有fsync和fdatassyanc两个同步函数，来强制系统将缓冲区中的内容写入磁盘

11.2 AOF文件的载入与数据还原
   * 载入步骤：
      1. 创建一个不带网络连接的伪客户端
      1. 从aof文件中分析并读取一条命令
      1. 使用为客户端执行被读出的写命令
      1. 重复前两个步骤直到处理完成

11.3 AOF重写
   * aof_rewrite : 阻塞，导致无法处理客户端需求
   * aof 后台重写
      * 为了保证一致性，重写期间服务器进程
         1. 执行客户端发来的命令
         2. 将执行后的命令追加到AOF缓冲区 （确保现有的aof正确）
         3. 将执行后的命令追加到AOF重写缓冲区（确保未来的aof正确）

***Chapter 12：事件***

   Redis服务器是一个事件驱动程序，主要处理两类事件
   * 文件事件（file event）
   * 时间事件（time event）
      * 比如serverCron

12.1 文件事件
   * 基于Reactor模式开发了自己的网络事件处理器，（file event handler）
      * 文件事件处理器使用I/O多路复用程序来同时监听多个套接字，并根据套接字目前执行的任务来为套接字关联不同的事件处理器
      * 当被监听的套接字准备好连接应答（accept）、读取（read）、写入（write）、关闭（close）等操作时，与操作相关联的文件事件就会产生，这是文件事件处理器就会调用套接字之前关联好的事件处理器来处理
   * 文件事件初期器的构成
      1. 套接字
      1. I/O多路复用程序
         * 将所有产生事件的套接字都放到一个队列里面，然后通过这个队列，以有序（sequentially）、同步（Synchronously）、每次一个套接字的方式向文件事件分配器传送
         * 当上一个套接字产生的时间呗处理完毕之后，I/O多路复用程序才会向文件事件分派器传送下一个套接字
      1. 文件事件分派器（dispatcher）
      1. 事件处理器
   * I/O多路复用程序的实现
      * select、epool、evport和kqueue
   * 事件的类型
      * AE_READABLE (优先处理)
      * AE_WRITEABLE
   * API
      * ae.c/aeXXXFileEvent
   * 文件事件处理器
      1. 连接应答处理器：networking.c/accetpTcpHandler：客户端connect时候
      1. 命令请求处理器：networking.c/readQueryFromClient
         * 在客户端链接服务器过程中，服务器会一直为客户端套接字的AE_READABLE事件关联命令处理器
      1. 命令回复处理器：networking.c/sendReplyToClient
         * 当服务器有命令回复需要传送给客户端时候，服务器会将客户端的AE_WRITEABLE事件和命令回复处理器关联起来；发送完毕之后服务器会接触该关联
      1. 一次完整的客户端与服务器连接事件示例

12.2 时间事件
   * 事件类型
      * 定时事件
      * 周期性事件
   * 时间事件的三个属性
      1. id：全局唯一ID，新的比旧的大
      1. when：毫秒精度的unix时间戳，记录事件的到达时间
      1. timeProc：时间事件处理器
   * 事件类型的判定
      * 如果事件处理器返回ae.h/AE_NOMORE，则为定时事件，事件到达之后会被删除，之后不会再到达
      * 如果返回一个非AE——NOMORE，则是一个周期性时间
   * time_events in redisServer
   * 实例：serverCron函数
      * 更新服务器的各类统计信息，比如时间、内存占用、数据库占用等
      * 清理数据库中的过期键值对
      * 关闭和清理链接失效的客户端
      * 尝试进行AOF或RDB的持久化
      * 如果服务器是主服务器，那么对从服务器进行定期同步
      * 如果处于集群模式，对集群进行定期同步和链接测试

12.3 事件的调度与执行
   * 时间的调度和执行由ae.c/aeProcessEvents函数负责
   ```
   def aeProcessEvents():
      # 获取到达时间离当前时间最接近的时间事件
      time_event = aeSearchNearestTimer()
     
      # 计算最接近的时间事件距离到达时间还有多少毫秒
      remaind_ms = time_event.when - unix_ts_now()

      # 如果事件已经到达，那么remaind_ms的值可能为负，设置为0
      if remaind_ms < 0:
          remaind_ms = 0

      # 根据remaind_ms的值，创建timeval结构
      timeval = create_timeval_with_ms(remaind_ms)

      # 阻塞并等待文件事件产生，最大阻塞时间由传入的timeval结构决定
      # 如果remaind_ms的值为0，那么aeApiPoll调用马上返回，不阻塞
      aeApiPool(timeval)

      # 处理已经产生的文件事件
      processFileEvents()

      # 处理所有已达到的时间事件
      processTimeEvents()
   ```
   * redis服务器的主函数
   ```
   def main():
       # 初始化服务器
       init_server()

       # 一直处理事件，知道服务器关闭为止
       while server_is_not_shutdown():
           aeProcessEvents()

       # 服务器关闭，执行清空操作
       clean_server()

   ```

***Chapter 13: 客户端***

   * redisServer中有一个 clients列表

13.1 客户端属性
   * 分为两类
      1. 通用属性，这些属性很少与特定的功能相关，无论客户端执行什么工作，他们都要用到这些属性
      1. 另一类是和特定功能相关的属性，比如操作数据库是需要用到的db属性和dict-id属性，执行事物时用到的mstate属性，以及WATCH命令用到的watched_keys属性
   * 套接字属性
      1. int fd; // 套接字
         * 伪客户端（faked client）的值为-1
         * 普通客户端，值为大于-1的整数
      1. robj* name; // 名字
      1. int flags;// 标志
         * 记录了客户端的角色（role），以及客户端目前所处的状态
      1. sds querybufer; // 输入缓冲区   
         * 缓存客户端发来的命令
      1. robj ** argv; // 参数列表
      1. int argc; // 参数长度
      1. struct redisCommand *cmd; // 命令的实现函数
         * redis中存在一个命令字典
         * 根据argv[0] 查询
      1. char buf[REDIS_REPLY_CHUNK_BYTES];
      1. int bufpos;
      1. list *reply; // 可变大小缓冲区
      1. int authenticated; // 身份验证
      1. 时间相关
         * time_t ctime;
         * time_t lastinteraction;
         * time_t obuf_soft_limit-reached_time;//  记录了输出换从去第一次到达软性限制的时间

13.2 客户端的创建与关闭
   * 创建普通客户端
      * 置于clients列表
   * 关闭普通客户端
      * 客户端进程退出、client kill、服务端timeout设置、客户端发送的命令请求的大小超过输入缓冲区大小（1GB）、要发给客户端的命令回复的大小超过了输出缓冲区的限制大小
      * 服务器使用两种模式来限制客户端输出缓冲区的大小
         * 硬性限制
         * 软性限制：obuf_soft_limit_reached_time
   * lua脚本的伪客户端
      * redisClient *lua_client; // 生命周期中一直存在
   * AOF文件的为客户端
      * 服务器在载入AOF文件时，会创建一个为客户端，用后关闭


***Chapter 14：服务器***

14.1 命令请求的执行过程 ： SET KEY VALUE
   * 过程：
      1. 客户端想服务器发送命令请求 SET KEY VALUE
      1. 服务器接收到并处理客户端发来的命令请求，在数据库中进行设置，并产生命令回复OK
      1. 服务器将命令回复OK发送给客户端
      1. 客户端接收到服务器返回来的命令回复OK，并将这个回复打印给用户看
   * 发送命令请求：客户端将命令转换成协议
   * 读取命令请求
      * 从socket读取命令，保存到保存客户端的输入缓冲区里面
      * 对缓冲区内容进行分析，提取参数
      * 调用命令执行器，执行客户端指定的命令
   * 命令执行器：查找命令实现
      * 根据argv[0] 到命令表中查找指定的命令，并且保存到客户端的cmd属性中
      * redisCommand结构的主要特性
      |属性名|类型|作用|
      ---------------
      |name|char *|命令的名字， 如set|
      |proc|redisCommandProd *|执行命令的实现函数的函数指针|
      |arity|int|命令的参数个数，如果是负数(-N)，则标识个数>=N|
      |sflags|char *|记录了命令的属性：读、写、是否允许|
      |flags|int|碎玉sflags分析得出的二进制标识|
      |calls|long long|服务器执行该命令的总次数|
      |milliseconds|long long|服务器执行该命令所耗费的时长|
   * 命令执行器：执行预备操作
      * 检查客户端状态的cmd指针是否指向NULL，如果为空，这返回
      * 根据cmd属性指向的redisCommand结构的arity属性，检查命令行参数的个数，不符合则返回
      * 检查客户端是否通过了身份验证
      * 如果服务器打开了maxmemory功能，则在执行命令前，先检查服务器内存占用情况，并在有需要时进行内存回收