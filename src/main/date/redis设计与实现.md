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

