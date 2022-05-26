# Redis

学习一门技术，我想首先我们要清楚的知道，我们为什么选择它，它有哪些功能，优点，它的这些功能底层如何工作以及实现。报着这种想法，我们才能更清楚深刻的了解这门技术。

## 概念与基础

Redis是一款内存高速缓存数据库。Redis全称为：**Remote Dictionary Server**（远程数据服务），使用C语言编写，Redis是一个key-value存储系统（键值存储系统），支持丰富的数据类型，如：String、list、set、zset、hash等。可用于缓存，事件发布或订阅，高速队列等场景。支持网络，提供字符串，哈希，列表，队列，集合结构直接存取，基于内存，可持久化。

### **为什么选择使用Redis？**

**读写性能优异**

* Redis能读的速度是110000次/s,写的速度是81000次/s 。

**数据类型丰富**

* Redis支持二进制案例的 Strings, Lists, Hashes, Sets 及 Ordered Sets 数据类型操作。

**原子性**

* Redis的所有操作都是原子性的，同时Redis还支持对几个操作全并后的原子性执行。

**丰富的特性**

* Redis支持 publish/subscribe, 通知, key 过期等特性。

**持久化**

* Redis支持RDB, AOF等持久化方式

**发布订阅**

* Redis支持发布/订阅模式

**分布式**

* Redis Cluster

### **使用场景**

#### 热点数据缓存

redis基于内存，读写性能优异，而且redis内部是支持事务的，在使用时候能有效保证数据一致性。

作为缓存使用时，一般有两种方式保存数据：

* 读取前，先去读Redis，如果没有数据，读取数据库，将数据拉入Redis。
* 插入数据时，同时写入Redis。

方案一：实施起来简单，但是有两个需要注意的地方：

* 避免缓存击穿。（数据库没有就需要命中的数据，导致Redis一直没有数据，而一直命中数据库。）
* 数据的实时性相对会差一点。

方案二：数据实时性强，但是开发时不便于统一处理。

当然，两种方式根据实际情况来适用。如：方案一适用于对于数据实时性要求不是特别高的场景。方案二适用于字典表、数据量不大的数据存储

#### 限时业务的运用

redis中可以使用expire命令设置一个键的生存时间，到时间后redis会删除它。利用这一特性可以运用在限时的优惠活动信息、手机验证码等业务场景。

#### 计数器相关

redis由于incrby命令可以实现原子性的递增，所以可以运用于高并发的秒杀活动、分布式序列号的生成、具体业务还体现在比如限制一个手机号发多少条短信、一个接口一分钟限制多少请求、一个接口一天限制调用多少次等等。

#### 分布式锁

这个主要利用redis的setnx命令进行，setnx："set if not exists"就是如果不存在则成功设置缓存同时返回1，否则返回0 ，这个特性在俞你奔远方的后台中有所运用，因为我们服务器是集群的，定时任务可能在两台机器上都会运行，所以在定时任务中首先 通过setnx设置一个lock， 如果成功设置则执行，如果没有成功设置，则表明该定时任务已执行。 当然结合具体业务，我们可以给这个lock加一个过期时间，比如说30分钟执行一次的定时任务，那么这个过期时间设置为小于30分钟的一个时间就可以，这个与定时任务的周期以及定时任务执行消耗时间相关。

在分布式锁的场景中，主要用在比如秒杀系统等。

#### 延时操作

比如在订单生产后我们占用了库存，10分钟后去检验用户是够真正购买，如果没有购买将该单据设置无效，同时还原库存。 由于redis自2.8.0之后版本提供Keyspace Notifications功能，允许客户订阅Pub/Sub频道，以便以某种方式接收影响Redis数据集的事件。 所以我们对于上面的需求就可以用以下解决方案，我们在订单生产时，设置一个key，同时设置10分钟后过期， 我们在后台实现一个监听器，监听key的实效，监听到key失效时将后续逻辑加上。

当然我们也可以利用rabbitmq、activemq等消息中间件的延迟队列服务实现该需求。

#### 排行榜问题

关系型数据库在排行榜方面查询速度普遍偏慢，所以可以借助redis的SortedSet进行热点数据的排序。

比如点赞排行榜，做一个SortedSet, 然后以用户的openid作为上面的username, 以用户的点赞数作为上面的score, 然后针对每个用户做一个hash, 通过zrangebyscore就可以按照点赞数获取排行榜，然后再根据username获取用户的hash信息，这个当时在实际运用中性能体验也蛮不错的。

#### 简单队列

由于Redis有list push和list pop这样的命令，所以能够很方便的执行队列操作。



```
title: Redis - 数据结构与对象
date: 2021-04-09 17:46:32
permalink: /pages/a54ec8/
categories:
  - 数据库
  - nosql 数据库
  - Redis
tags:
  - 
```

## 数据结构与对象

首先对redis来说，所有的key（键）都是字符串。我们在谈基础数据结构时，讨论的是存储值的数据类型，主要包括常见的5种数据类型，分别是：String、List、Set、Zset、Hash。

#### 基础数据类型

**String字符串**

String类型是二进制安全的，意思是 redis 的 string 可以包含任何数据。如数字，字符串，jpg图片或者序列化的对象。

| 命令     | 简述          | 使用                |
| ------ | ----------- | ----------------- |
| GET    | 获取存储在给定键中的值 | GET name          |
| SET    | 设置存储在给定键中的值 | SET name value    |
| DEL    | 删除存储在给定键中的值 | DEL name          |
| INCR   | 将键存储的值加1    | INCR key          |
| DECR   | 将键存储的值减1    | DECR key          |
| INCRBY | 将键存储的值加上整数  | INCRBY key amount |
| DECRBY | 将键存储的值减去整数  | DECRBY key amount |

* 实战场景
  * **缓存**： 经典使用场景，把常用信息，字符串，图片或者视频等信息放到redis中，redis作为缓存层，mysql做持久化层，降低mysql的读写压力。
  * **计数器**：redis是单线程模型，一个命令执行完才会执行下一个，同时数据可以一步落地到其他的数据源。
  * **session**：常见方案spring session + redis实现session共享，

**List列表**

Redis中的List其实就是链表（Redis用双端链表实现List）

使用List结构，我们可以轻松地实现最新消息排队功能（比如新浪微博的TimeLine）。List的另一个应用就是消息

队列，可以利用List的 PUSH 操作，将任务存放在List中，然后工作线程再用 POP 操作将任务取出进行执行。

| 命令     | 简述                                                              | 使用              |
| ------ | --------------------------------------------------------------- | --------------- |
| RPUSH  | 将给定值推入到列表右端                                                     | RPUSH key value |
| LPUSH  | 将给定值推入到列表左端                                                     | LPUSH key value |
| RPOP   | 从列表的右端弹出一个值，并返回被弹出的值                                            | RPOP key        |
| LPOP   | 从列表的左端弹出一个值，并返回被弹出的值                                            | LPOP key        |
| LRANGE | 获取列表在给定范围上的所有值                                                  | LRANGE key 0 -1 |
| LINDEX | 通过索引获取列表中的元素。你也可以使用负数下标，以 -1 表示列表的最后一个元素， -2 表示列表的倒数第二个元素，以此类推。 | LINEX key index |

* 使用列表的技巧
  * lpush+lpop=Stack(栈)
  * lpush+rpop=Queue（队列）
  * lpush+ltrim=Capped Collection（有限集合）
  * lpush+brpop=Message Queue（消息队列）

**Set集合**

Redis 的 Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据

Redis 中集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。

* **命令使用**

| 命令        | 简述                        | 使用                   |
| --------- | ------------------------- | -------------------- |
| SADD      | 向集合添加一个或多个成员              | SADD key value       |
| SCARD     | 获取集合的成员数                  | SCARD key            |
| SMEMBER   | 返回集合中的所有成员                | SMEMBER key member   |
| SISMEMBER | 判断 member 元素是否是集合 key 的成员 | SISMEMBER key member |

* 实战场景
  * **标签**（tag）,给用户添加标签，或者用户给消息添加标签，这样有同一标签或者类似标签的可以给推荐关注的事或者关注的人。
  * **点赞，或点踩，收藏等**，可以放到set中实现

**Hash散列**

Redis hash 是一个 string 类型的 field（字段） 和 value（值） 的映射表，hash 特别适合用于存储对象。

* **命令使用**

| 命令      | 简述                   | 使用                            |
| ------- | -------------------- | ----------------------------- |
| HSET    | 添加键值对                | HSET hash-key sub-key1 value1 |
| HGET    | 获取指定散列键的值            | HGET hash-key key1            |
| HGETALL | 获取散列中包含的所有键值对        | HGETALL hash-key              |
| HDEL    | 如果给定键存在于散列中，那么就移除这个键 | HDEL hash-key sub-key1        |

* 实战场景
  * **缓存**： 能直观，相比string更节省空间，的维护缓存信息，如用户信息，视频信息等。

**Zset有序集合**

Redis 有序集合和集合一样也是 string 类型元素的集合,且不允许重复的成员。不同的是每个元素都会关联一个 double 类型的分数。redis 正是通过分数来为集合中的成员进行从小到大的排序。

* **命令使用**

| 命令     | 简述                           | 使用                             |
| ------ | ---------------------------- | ------------------------------ |
| ZADD   | 将一个带有给定分值的成员添加到哦有序集合里面       | ZADD zset-key 178 member1      |
| ZRANGE | 根据元素在有序集合中所处的位置，从有序集合中获取多个元素 | ZRANGE zset-key 0-1 withccores |
| ZREM   | 如果给定元素成员存在于有序集合中，那么就移除这个元素   | ZREM zset-key member1          |

* 实战场景
  * **排行榜**：有序集合经典使用场景。例如小说视频等网站需要对用户上传的小说视频做排行榜，榜单可以按照用户关注数，更新时间，字数等打分，做排行。

### 特殊数据类型

**HyperLoglogs（基数统计）**

* **什么是基数？**

举个例子，A = {1, 2, 3, 4, 5}， B = {3, 5, 6, 7, 9}；那么基数（不重复的元素）= 1, 2, 4, 6, 7, 9； （允许容错，即可以接受一定误差）

* **HyperLogLogs 基数统计用来解决什么问题**？

这个结构可以非常省内存的去统计各种计数，比如注册 IP 数、每日访问 IP 数、页面实时UV、在线用户数，共同好友数等。

* **它的优势体现在哪**？

一个大型的网站，每天 IP 比如有 100 万，粗算一个 IP 消耗 15 字节，那么 100 万个 IP 就是 15M。而 HyperLogLog 在 Redis 中每个键占用的内容都是 12K，理论存储近似接近 2^64 个值，不管存储的内容是什么，它一个基于基数估算的算法，只能比较准确的估算出基数，可以使用少量固定的内存去存储并识别集合中的唯一元素。而且这个估算的基数并不一定准确，是一个带有 0.81% 标准错误的近似值（对于可以接受一定容错的业务场景，比如IP数统计，UV等，是可以忽略不计的）。

```
127.0.0.1:6379> pfadd key1 a b c d e f g h i    # 创建第一组元素(integer) 1127.0.0.1:6379> pfcount key1                    # 统计元素的基数数量(integer) 9127.0.0.1:6379> pfadd key2 c j k l m e g a      # 创建第二组元素(integer) 1127.0.0.1:6379> pfcount key2(integer) 8127.0.0.1:6379> pfmerge key3 key1 key2          # 合并两组：key1 key2 -> key3 并集OK127.0.0.1:6379> pfcount key3(integer) 13
```

**Bitmap（位存储）**

Bitmap 即位图数据结构，都是操作二进制位来进行记录，只有0 和 1 两个状态。

* **用来解决什么问题**？

比如：统计用户信息，活跃，不活跃！ 登录，未登录！ 打卡，不打卡！ **两个状态的，都可以使用 Bitmaps**！

如果存储一年的打卡状态需要多少内存呢？ 365 天 = 365 bit 1字节 = 8bit 46 个字节左右！

* **相关命令使用**

使用bitmap 来记录 周一到周日的打卡！ 周一：1 周二：0 周三：0 周四：1 ......

```
127.0.0.1:6379> setbit sign 0 1(integer) 0127.0.0.1:6379> setbit sign 1 1(integer) 0127.0.0.1:6379> setbit sign 2 0(integer) 0127.0.0.1:6379> getbit sign 3(integer) 1127.0.0.1:6379> getbit sign 5(integer) 0127.0.0.1:6379> bitcount sign # 统计这周的打卡记录，就可以看到是否有全勤！(integer) 3
```

**geospatial (地理位置)**

Redis 的 Geo 在 Redis 3.2 版本就推出了! 这个功能可以推算地理位置的信息: 两地之间的距离, 方圆几里的人

**规则**

两级无法直接添加，我们一般会下载城市数据(这个网址可以查询 GEO： [http://www.jsons.cn/lngcode](http://www.jsons.cn/lngcode))！

* 有效的经度从-180度到180度。
* 有效的纬度从-85.05112878度到85.05112878度。

| 命令                | 简述                  | 使用                                                            |
| ----------------- | ------------------- | ------------------------------------------------------------- |
| geoadd            | 添加地理位置              | geoadd listname \[ip cityname]+                               |
| geopos            | 获取指定的成员的经度和纬度       | geops listname \[cityname]+                                   |
| geodist           | 计算距离                | geodist listname source target \[m km mi ft]                  |
| georadius         | 附近的人,通过半径来查询        | georadius listname ipx ipy \[半径] \[m km mi ft]                |
| georadiusbymember | 显示与指定成员一定半径范围内的其他成员 | georadiusbymember listname citiyname 1000 \[半径] \[m km mi ft] |

**底层**

geo底层的实现原理实际上就是Zset, 我们可以通过Zset命令来操作geo

#### 底层数据结构与对象机制

**对象**

Redis并没有直接使用这些数据结构来实现键值对数据库，而是基于这些数据结构创建了一个对象系统，这个系统

包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象这五种类型的对象，每种对象都用到了至少一种

我们前面所介绍的数据结构。

> 通过这五种不同类型的对象，Redis可以在执行命令之前，根据对象的类型来判断一个对象是否可以执行给定
>
> 的命令。使用对象的另一个好处是，我们可以针对不同的使用场景，为对象设置多种不同的数据结构实现，
>
> 从而优化对象在不同场景下的使用效率。

除此之外，Redis的对象系统还实现了基于引用计数技术的内存回收机制，当程序不再使用某个对象的时候，这个

对象所占用的内存就会被自动释放；另外，Redis还通过引用计数技术实现了对象共享机制，这一机制可以在适当

的条件下，通过让多个数据库键共享同一个对象来节约内存。最后，Redis的对象带有访问时间记录信息，该信息

可以用于计算数据库键的空转时长，在服务器启用了maxmemory功能的情况下，空转时长较大的那些键可能会优

先被服务器删除。

![img](https://www.pdai.tech/\_images/db/redis/db-redis-object-2-3.png)

Redis的每种对象其实都由**对象结构(redisObject)** 与 **对应编码的数据结构**组合而成，而每种对象类型对应若干编码方式，不同的编码方式所对应的底层数据结构是不同的。

**Redis 每个键都带有类型信息, 使得程序可以检查键的类型, 并为它选择合适的处理方式**.

**操作数据类型的命令除了要对键的类型进行检查之外, 还需要根据数据类型的不同编码进行多态处理**.

因此Redis构建了自己的类型系统：

* redisObject 对象.
* 基于 redisObject 对象的类型检查.
* 基于 redisObject 对象的显式多态函数.
* 对 redisObject 进行分配、共享和销毁的机制

```
/* * Redis 对象 */typedef struct redisObject {​    // 类型    unsigned type:4;​    // 编码方式    unsigned encoding:4;​    // LRU - 24位, 记录最末一次访问时间（相对于lru_clock）; 或者 LFU（最少使用的数据：8位频率，16位访问时间）    unsigned lru:LRU_BITS; // LRU_BITS: 24​    // 引用计数    int refcount;​    // 指向底层数据结构实例    void *ptr;​} robj;
```

![img](https://www.pdai.tech/\_images/db/redis/db-redis-object-1.png)

**其中type、encoding和ptr是最重要的三个属性**。

*   **type记录了对象所保存的值的类型**，它的值可能是以下常量中的一个：

    ```
    /** 对象类型*/#define OBJ_STRING 0 // 字符串#define OBJ_LIST 1 // 列表#define OBJ_SET 2 // 集合#define OBJ_ZSET 3 // 有序集#define OBJ_HASH 4 // 哈希表  
    ```

    * **encoding记录了对象所保存的值的编码**，它的值可能是以下常量中的一个：

    ```
    /** 对象编码*/#define OBJ_ENCODING_RAW 0     /* Raw representation */#define OBJ_ENCODING_INT 1     /* Encoded as integer */#define OBJ_ENCODING_HT 2      /* Encoded as hash table */#define OBJ_ENCODING_ZIPMAP 3  /* 注意：版本2.6后不再使用. */#define OBJ_ENCODING_LINKEDLIST 4 /* 注意：不再使用了，旧版本2.x中String的底层之一. */#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
    ```

    * **ptr是一个指针，指向实际保存值的数据结构**，这个数据结构由type和encoding属性决定。举个例子， 如果一个redisObject 的type 属性为`OBJ_LIST` ， encoding 属性为`OBJ_ENCODING_QUICKLIST` ，那么这个对象就是一个Redis 列表（List)，它的值保存在一个QuickList的数据结构内，而ptr 指针就指向quicklist的对象；

**空转时长**：当前时间减去键的值对象的lru时间，就是该键的空转时长。Object idletime命令可以打印出给定键的空转时长

如果服务器打开了maxmemory选项，并且服务器用于回收内存的算法为volatile-lru或者allkeys-lru，那么当服务器占用的内存数超过了maxmemory选项所设置的上限值时，空转时长较高的那部分键会优先被服务器释放，从而回收内存。

**命令的类型检查和多态**

**当执行一个处理数据类型命令的时候，redis执行以下步骤**

* 根据给定的key，在数据库字典中查找和他相对应的redisObject，如果没找到，就返回NULL；
* 检查redisObject的type属性和执行命令所需的类型是否相符，如果不相符，返回类型错误；
* 根据redisObject的encoding属性所指定的编码，选择合适的操作函数来处理底层的数据结构；
* 返回数据结构的操作结果作为命令的返回值。

![img](https://www.pdai.tech/\_images/db/redis/db-redis-object-3.png)

**对象共享**

> redis一般会把一些常见的值放到一个共享对象中，这样可使程序避免了重复分配的麻烦，也节约了一些CPU时间。

**redis预分配的值对象如下**：

* 各种命令的返回值，比如成功时返回的OK，错误时返回的ERROR，命令入队事务时返回的QUEUE，等等
* 包括0 在内，小于REDIS\_SHARED\_INTEGERS的所有整数（REDIS\_SHARED\_INTEGERS的默认值是10000）

注意：共享对象只能被字典和双向链表这类能带有指针的数据结构使用。像整数集合和压缩列表这些只能保存字符串、整数等自勉之的内存数据结构

**为什么redis不共享列表对象、哈希对象、集合对象、有序集合对象，只共享字符串对象**？

* 列表对象、哈希对象、集合对象、有序集合对象，本身可以包含字符串对象，复杂度较高。
* 如果共享对象是保存字符串对象，那么验证操作的复杂度为O(1)
* 如果共享对象是保存字符串值的字符串对象，那么验证操作的复杂度为O(N)
* 如果共享对象是包含多个值的对象，其中值本身又是字符串对象，即其它对象中嵌套了字符串对象，比如列表对象、哈希对象，那么验证操作的复杂度将会是O(N的平方)

如果对复杂度较高的对象创建共享对象，需要消耗很大的CPU，用这种消耗去换取内存空间，是不合适的

**引用计数以及对象的消毁**

redisObject中有refcount属性，是对象的引用计数，显然计数0那么就是可以回收。

* 每个redisObject结构都带有一个refcount属性，指示这个对象被引用了多少次；
* 当新创建一个对象时，它的refcount属性被设置为1；
* 当对一个对象进行共享时，redis将这个对象的refcount加一；
* 当使用完一个对象后，或者消除对一个对象的引用之后，程序将对象的refcount减一；
* 当对象的refcount降至0 时，这个RedisObject结构，以及它引用的数据结构的内存都会被释放。

**底层数据结构**

![img](https://www.pdai.tech/\_images/db/redis/db-redis-object-2-3.png)

* 简单动态字符串 - sds
* 压缩列表 - ZipList
* 快表 - QuickList
* 字典/哈希表 - Dict
* 整数集 - IntSet
* 跳表 - ZSkipList

**简单动态字符串 - sds**

> Redis 是用 C 语言写的，但是对于Redis的字符串，却不是 C 语言中的字符串（即以空字符’\0’结尾的字符数组），它是自己构建了一种名为 **简单动态字符串（simple dynamic string,SDS**）的抽象类型，并将 SDS 作为 Redis的默认字符串表示。

**SDS 定义**

> 这是一种用于存储二进制数据的一种结构, 具有动态扩容的特点. 其实现位于src/sds.h与src/sds.c中。

* **SDS的总体概览**如下图:

![img](https://www.pdai.tech/\_images/db/redis/db-redis-ds-x-3.png)

其中`sdshdr`是头部, `buf`是真实存储用户数据的地方. 另外注意, 从命名上能看出来, 这个数据结构除了能存储二进制数据, 显然是用于设计作为字符串使用的, 所以在buf中, 用户数据后总跟着一个\0. 即图中 `"数据" + "\0"`是为所谓的buf。

* 如下是**6.0源码中sds相关的结构**：

![img](https://www.pdai.tech/\_images/db/redis/db-redis-ds-x-1.png)

通过上图我们可以看到，SDS有五种不同的头部. 其中sdshdr5实际并未使用到. 所以实际上有四种不同的头部, 分别如下:

![img](https://www.pdai.tech/\_images/db/redis/db-redis-ds-x-2.png)

其中：

* `len` 保存了SDS保存字符串的长度
* `buf[]` 数组用来保存字符串的每个元素
* `alloc`分别以uint8, uint16, uint32, uint64表示整个SDS, 除过头部与末尾的\0, 剩余的字节数.
* `flags` 始终为一字节, 以低三位标示着头部的类型, 高5位未使用.

**为什么使用SDS**

> **为什么不使用C语言字符串实现，而是使用 SDS呢**？这样实现有什么好处？

* **常数复杂度获取字符串长度**

由于 len 属性的存在，我们获取 SDS 字符串的长度只需要读取 len 属性，时间复杂度为 O(1)。而对于 C 语言，获取字符串的长度通常是经过遍历计数来实现的，时间复杂度为 O(n)。通过 `strlen key` 命令可以获取 key 的字符串长度。

* **杜绝缓冲区溢出**

我们知道在 C 语言中使用 `strcat` 函数来进行两个字符串的拼接，一旦没有分配足够长度的内存空间，就会造成缓冲区溢出。而对于 SDS 数据类型，在进行字符修改的时候，**会首先根据记录的 len 属性检查内存空间是否满足需求**，如果不满足，会进行相应的空间扩展，然后在进行修改操作，所以不会出现缓冲区溢出。

* **减少修改字符串的内存重新分配次数**

C语言由于不记录字符串的长度，所以如果要修改字符串，必须要重新分配内存（先释放再申请），因为如果没有重新分配，字符串长度增大时会造成内存缓冲区溢出，字符串长度减小时会造成内存泄露。

而对于SDS，由于`len`属性和`alloc`属性的存在，对于修改字符串SDS实现了**空间预分配**和**惰性空间释放**两种策略：

1、`空间预分配`：对字符串进行空间扩展的时候，扩展的内存比实际需要的多，这样可以减少连续执行字符串增长操作所需的内存重分配次数。

2、`惰性空间释放`：对字符串进行缩短操作时，程序不立即使用内存重新分配来回收缩短后多余的字节，而是使用 `alloc` 属性将这些字节的数量记录下来，等待后续使用。（当然SDS也提供了相应的API，当我们有需要时，也可以手动释放这些未使用的空间。）

* **二进制安全**

因为C字符串以空字符作为字符串结束的标识，而对于一些二进制文件（如图片等），内容可能包括空字符串，因此C字符串无法正确存取；而所有 SDS 的API 都是以处理二进制的方式来处理 `buf` 里面的元素，并且 SDS 不是以空字符串来判断是否结束，而是以 len 属性表示的长度来判断字符串是否结束。

* **兼容部分 C 字符串函数**

虽然 SDS 是二进制安全的，但是一样遵从每个字符串都是以空字符串结尾的惯例，这样可以重用 C 语言库\`\` 中的一部分函数。

**空间预分配补进一步理解**

当执行追加操作时，比如现在给`key=‘Hello World’`的字符串后追加`‘ again!’`则这时的len=18，free由0变成了18，此时的`buf='Hello World again!\0....................'`(.表示空格)，也就是buf的内存空间是18+18+1=37个字节，其中‘\0’占1个字节redis给字符串多分配了18个字节的预分配空间，所以下次还有append追加的时候，如果预分配空间足够，就无须在进行空间分配了。在当前版本中，当新字符串的长度小于1M时，redis会分配他们所需大小一倍的空间，当大于1M的时候，就为他们额外多分配1M的空间。

思考：**这种分配策略会浪费内存资源吗**？

答：执行过APPEND 命令的字符串会带有额外的预分配空间，这些预分配空间不会被释放，除非该字符串所对应的键被删除，或者等到关闭Redis 之后，再次启动时重新载入的字符串对象将不会有预分配空间。因为执行APPEND 命令的字符串键数量通常并不多，占用内存的体积通常也不大，所以这一般并不算什么问题。另一方面，如果执行APPEND 操作的键很多，而字符串的体积又很大的话，那可能就需要修改Redis 服务器，让它定时释放一些字符串键的预分配空间，从而更有效地使用内存。

**小结**

redis的字符串表示为sds，而不是C字符串（以\0结尾的char\*）， 它是Redis 底层所使用的字符串表示，它被用在几乎所有的Redis 模块中。可以看如下对比：

![img](https://www.pdai.tech/\_images/db/redis/redis-ds-2.png)

一般来说，SDS 除了保存数据库中的字符串值以外，SDS 还可以作为缓冲区（buffer）：包括 AOF 模块中的AOF缓冲区以及客户端状态中的输入缓冲区。

**压缩列表 - ZipList**

ziplist是为了提高存储效率而设计的一种特殊编码的双向链表。它可以存储字符串或者整数，存储整数时是采用整数的二进制而不是字符串形式存储。他能在O(1)的时间复杂度下完成list两端的push和pop操作。但是因为每次操作都需要重新分配ziplist的内存，所以实际复杂度和ziplist的内存使用量相关。

整个ziplist在内存中的存储格式如下：

![img](https://www.pdai.tech/\_images/db/redis/db-redis-ds-x-6.png)

* `zlbytes`字段的类型是uint32\_t, 这个字段中存储的是整个ziplist所占用的内存的字节数
* `zltail`字段的类型是uint32\_t, 它指的是ziplist中最后一个entry的偏移量. 用于快速定位最后一个entry, 以快速完成pop等操作
* `zllen`字段的类型是uint16\_t, 它指的是整个ziplit中entry的数量. 这个值只占2bytes（16位）: 如果ziplist中entry的数目小于65535(2的16次方), 那么该字段中存储的就是实际entry的值. 若等于或超过65535, 那么该字段的值固定为65535, 但实际数量需要一个个entry的去遍历所有entry才能得到.
* `zlend`是一个终止字节, 其值为全F, 即0xff. ziplist保证任何情况下, 一个entry的首字节都不会是255

**Entry结构**

**第一种情况**：一般结构

`prevlen`：前一个entry的大小，编码方式见下文；

`encoding`：不同的情况下值不同，用于表示当前entry的类型和长度；

`entry-data`：真是用于存储entry表示的数据；

**第二种情况**：在entry中存储的是int类型时，encoding和entry-data会合并在encoding中表示，此时没有entry-data字段；

redis中，在存储数据时，会先尝试将string转换成int存储，节省空间；

此时entry结构：

* **prevlen编码**

当前一个元素长度小于254（255用于zlend）的时候，prevlen长度为1个字节，值即为前一个entry的长度，如果长度大于等于254的时候，prevlen用5个字节表示，第一字节设置为254，后面4个字节存储一个小端的无符号整型，表示前一个entry的长度；

```
<prevlen from 0 to 253> <encoding> <entry>      //长度小于254结构0xFE <4 bytes unsigned little endian prevlen> <encoding> <entry>   //长度大于等于254      
```

* **encoding编码**

encoding的长度和值根据保存的是int还是string，还有数据的长度而定；

前两位用来表示类型，当为“11”时，表示entry存储的是int类型，其他表示存储的是string；

**存储string时**：

`|00pppppp|` ：此时encoding长度为1个字节，该字节的后六位表示entry中存储的string长度，因为是6位，所以entry中存储的string长度不能超过63；

`|01pppppp|qqqqqqqq|` 此时encoding长度为两个字节；此时encoding的后14位用来存储string长度，长度不能超过16383；

`|10000000|qqqqqqqq|rrrrrrrr|ssssssss|ttttttt|` 此时encoding长度为5个字节，后面的4个字节用来表示encoding中存储的字符串长度，长度不能超过2^32 - 1;

**存储int时**：

`|11000000|` encoding为3个字节，后2个字节表示一个int16；

`|11010000|` encoding为5个字节，后4个字节表示一个int32;

`|11100000|` encoding 为9个字节，后8字节表示一个int64;

`|11110000|` encoding为4个字节，后3个字节表示一个有符号整型；

`|11111110|` encoding为2字节，后1个字节表示一个有符号整型；

`|1111xxxx|` encoding长度就只有1个字节，xxxx表示一个0 - 12的整数值；

`|11111111|` 还记得zlend么？

* **源码中数据结构支撑**

你可以看到为了操作上的简易实际还增加了几个属性

```
/* We use this function to receive information about a ziplist entry. * Note that this is not how the data is actually encoded, is just what we * get filled by a function in order to operate more easily. */typedef struct zlentry {    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/    unsigned int prevrawlen;     /* Previous entry len. */    unsigned int lensize;        /* Bytes used to encode this entry type/len.                                    For example strings have a 1, 2 or 5 bytes                                    header. Integers always use a single byte.*/    unsigned int len;            /* Bytes used to represent the actual entry.                                    For strings this is just the string length                                    while for integers it is 1, 2, 3, 4, 8 or                                    0 (for 4 bit immediate) depending on the                                    number range. */    unsigned int headersize;     /* prevrawlensize + lensize. */    unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on                                    the entry encoding. However for 4 bits                                    immediate integers this can assume a range                                    of values and must be range-checked. */    unsigned char *p;            /* Pointer to the very start of the entry, that                                    is, this points to prev-entry-len field. */} zlentry;          @pdai: 代码已经复制到剪贴板    
```

* `prevrawlensize`表示 previous\_entry\_length字段的长度
* `prevrawlen`表示 previous\_entry\_length字段存储的内容
* `lensize`表示 encoding字段的长度
* `len`表示数据内容长度
* `headersize` 表示当前元素的首部长度，即previous\_entry\_length字段长度与encoding字段长度之和
* `encoding`表示数据类型
* `p`表示当前元素首地址

#### 为什么ZipList特别省内存

> 所以只有理解上面的Entry结构，我们才会真正理解ZipList为什么是特别节省内存的数据结构。@pdai

* ziplist节省内存是相对于普通的list来说的，如果是普通的数组，那么它每个元素占用的内存是一样的且取决于最大的那个元素（很明显它是需要预留空间的）；
* 所以ziplist在设计时就很容易想到要尽量让每个元素按照实际的内容大小存储，**所以增加encoding字段**，针对不同的encoding来细化存储大小；
* 这时候还需要解决的一个问题是遍历元素时如何定位下一个元素呢？在普通数组中每个元素定长，所以不需要考虑这个问题；但是ziplist中每个data占据的内存不一样，所以为了解决遍历，需要增加记录上一个元素的length，**所以增加了prelen字段**。

**为什么我们去研究ziplist特别节省内存的数据结构**？ 在实际应用中，大量存储字符串的优化是需要你对底层的数据结构有一定的理解的，而ziplist在场景优化的时候也被考虑采用的首选。

#### ziplist的缺点

最后我们再看看它的一些缺点：

* ziplist也不预留内存空间, 并且在移除结点后, 也是立即缩容, 这代表每次写操作都会进行内存分配操作.
* 结点如果扩容, 导致结点占用的内存增长, 并且超过254字节的话, 可能会导致链式反应: 其后一个结点的entry.prevlen需要从一字节扩容至五字节. **最坏情况下, 第一个结点的扩容, 会导致整个ziplist表中的后续所有结点的entry.prevlen字段扩容**. 虽然这个内存重分配的操作依然只会发生一次, 但代码中的时间复杂度是o(N)级别, 因为链式扩容只能一步一步的计算. 但这种情况的概率十分的小, 一般情况下链式扩容能连锁反映五六次就很不幸了. 之所以说这是一个蛋疼问题, 是因为, 这样的坏场景下, 其实时间复杂度并不高: 依次计算每个entry新的空间占用, 也就是o(N), 总体占用计算出来后, 只执行一次内存重分配, 与对应的memmove操作, 就可以了

### 快表 - QuickList

> quicklist这个结构是Redis在3.2版本后新加的, 之前的版本是list(即linkedlist)， 用于String数据类型中。

它是一种以ziplist为结点的双端链表结构. 宏观上, quicklist是一个链表, 微观上, 链表中的每个结点都是一个ziplist。

#### quicklist结构

* 如下是**6.0源码中quicklist相关的结构**：

```
/* Node, quicklist, and Iterator are the only data structures used currently. */​/* quicklistNode is a 32 byte struct describing a ziplist for a quicklist. * We use bit fields keep the quicklistNode at 32 bytes. * count: 16 bits, max 65536 (max zl bytes is 65k, so max count actually < 32k). * encoding: 2 bits, RAW=1, LZF=2. * container: 2 bits, NONE=1, ZIPLIST=2. * recompress: 1 bit, bool, true if node is temporarry decompressed for usage. * attempted_compress: 1 bit, boolean, used for verifying during testing. * extra: 10 bits, free for future use; pads out the remainder of 32 bits */typedef struct quicklistNode {    struct quicklistNode *prev;    struct quicklistNode *next;    unsigned char *zl;    unsigned int sz;             /* ziplist size in bytes */    unsigned int count : 16;     /* count of items in ziplist */    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */    unsigned int recompress : 1; /* was this node previous compressed? */    unsigned int attempted_compress : 1; /* node can't compress; too small */    unsigned int extra : 10; /* more bits to steal for future usage */} quicklistNode;​/* quicklistLZF is a 4+N byte struct holding 'sz' followed by 'compressed'. * 'sz' is byte length of 'compressed' field. * 'compressed' is LZF data with total (compressed) length 'sz' * NOTE: uncompressed length is stored in quicklistNode->sz. * When quicklistNode->zl is compressed, node->zl points to a quicklistLZF */typedef struct quicklistLZF {    unsigned int sz; /* LZF size in bytes*/    char compressed[];} quicklistLZF;​/* Bookmarks are padded with realloc at the end of of the quicklist struct. * They should only be used for very big lists if thousands of nodes were the * excess memory usage is negligible, and there's a real need to iterate on them * in portions. * When not used, they don't add any memory overhead, but when used and then * deleted, some overhead remains (to avoid resonance). * The number of bookmarks used should be kept to minimum since it also adds * overhead on node deletion (searching for a bookmark to update). */typedef struct quicklistBookmark {    quicklistNode *node;    char *name;} quicklistBookmark;​​/* quicklist is a 40 byte struct (on 64-bit systems) describing a quicklist. * 'count' is the number of total entries. * 'len' is the number of quicklist nodes. * 'compress' is: -1 if compression disabled, otherwise it's the number *                of quicklistNodes to leave uncompressed at ends of quicklist. * 'fill' is the user-requested (or default) fill factor. * 'bookmakrs are an optional feature that is used by realloc this struct, *      so that they don't consume memory when not used. */typedef struct quicklist {    quicklistNode *head;    quicklistNode *tail;    unsigned long count;        /* total count of all entries in all ziplists */    unsigned long len;          /* number of quicklistNodes */    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */    unsigned int bookmark_count: QL_BM_BITS;    quicklistBookmark bookmarks[];} quicklist;​typedef struct quicklistIter {    const quicklist *quicklist;    quicklistNode *current;    unsigned char *zi;    long offset; /* offset in current ziplist */    int direction;} quicklistIter;​typedef struct quicklistEntry {    const quicklist *quicklist;    quicklistNode *node;    unsigned char *zi;    unsigned char *value;    long long longval;    unsigned int sz;    int offset;} quicklistEntry;      
```

这里定义了6个结构体:

* `quicklistNode`, 宏观上, quicklist是一个链表, 这个结构描述的就是链表中的结点. 它通过zl字段持有底层的ziplist. 简单来讲, 它描述了一个ziplist实例
* `quicklistLZF`, ziplist是一段连续的内存, 用LZ4算法压缩后, 就可以包装成一个quicklistLZF结构. 是否压缩quicklist中的每个ziplist实例是一个可配置项. 若这个配置项是开启的, 那么quicklistNode.zl字段指向的就不是一个ziplist实例, 而是一个压缩后的quicklistLZF实例
* `quicklistBookmark`, 在quicklist尾部增加的一个书签，它只有在大量节点的多余内存使用量可以忽略不计的情况且确实需要分批迭代它们，才会被使用。当不使用它们时，它们不会增加任何内存开销。
* `quicklist`. 这就是一个双链表的定义. head, tail分别指向头尾指针. len代表链表中的结点. count指的是整个quicklist中的所有ziplist中的entry的数目. fill字段影响着每个链表结点中ziplist的最大占用空间, compress影响着是否要对每个ziplist以LZ4算法进行进一步压缩以更节省内存空间.
* `quicklistIter`是一个迭代器
* `quicklistEntry`是对ziplist中的entry概念的封装. quicklist作为一个封装良好的数据结构, 不希望使用者感知到其内部的实现, 所以需要把ziplist.entry的概念重新包装一下.

#### quicklist内存布局图

quicklist的内存布局图如下所示:

#### quicklist更多额外信息

下面是有关quicklist的更多额外信息:

*   ```
    quicklist.fill
    ```

    的值影响着每个链表结点中, ziplist的长度.

    1. 当数值为负数时, 代表以字节数限制单个ziplist的最大长度. 具体为:
    2. \-1 不超过4kb
    3. \-2 不超过 8kb
    4. \-3 不超过 16kb
    5. \-4 不超过 32kb
    6. \-5 不超过 64kb
    7. 当数值为正数时, 代表以entry数目限制单个ziplist的长度. 值即为数目. 由于该字段仅占16位, 所以以entry数目限制ziplist的容量时, 最大值为2^15个
*   ```
    quicklist.compress
    ```

    的值影响着quicklistNode.zl字段指向的是原生的ziplist, 还是经过压缩包装后的quicklistLZF

    1. 0 表示不压缩, zl字段直接指向ziplist
    2. 1 表示quicklist的链表头尾结点不压缩, 其余结点的zl字段指向的是经过压缩后的quicklistLZF
    3. 2 表示quicklist的链表头两个, 与末两个结点不压缩, 其余结点的zl字段指向的是经过压缩后的quicklistLZF
    4. 以此类推, 最大值为2^16
* `quicklistNode.encoding`字段, 以指示本链表结点所持有的ziplist是否经过了压缩. 1代表未压缩, 持有的是原生的ziplist, 2代表压缩过
* `quicklistNode.container`字段指示的是每个链表结点所持有的数据类型是什么. 默认的实现是ziplist, 对应的该字段的值是2, 目前Redis没有提供其它实现. 所以实际上, 该字段的值恒为2
* `quicklistNode.recompress`字段指示的是当前结点所持有的ziplist是否经过了解压. 如果该字段为1即代表之前被解压过, 且需要在下一次操作时重新压缩.

quicklist的具体实现代码篇幅很长, 这里就不贴代码片断了, 从内存布局上也能看出来, 由于每个结点持有的ziplist是有上限长度的, 所以在与操作时要考虑的分支情况比较多。

quicklist有自己的优点, 也有缺点, 对于使用者来说, 其使用体验类似于线性数据结构, list作为最传统的双链表, 结点通过指针持有数据, 指针字段会耗费大量内存. ziplist解决了耗费内存这个问题. 但引入了新的问题: 每次写操作整个ziplist的内存都需要重分配. quicklist在两者之间做了一个平衡. 并且使用者可以通过自定义`quicklist.fill`, 根据实际业务情况, 经验主义调参.

### 字典/哈希表 - Dict

> 本质上就是哈希表, 这个在很多语言中都有，对于开发人员人员来说比较熟悉，这里就简单介绍下。

#### 数据结构

**哈希表结构定义**：

```
typedef struct dictht{    //哈希表数组    dictEntry **table;    //哈希表大小    unsigned long size;    //哈希表大小掩码，用于计算索引值    //总是等于 size-1    unsigned long sizemask;    //该哈希表已有节点的数量    unsigned long used; }dictht      
```

哈希表是由数组 table 组成，table 中每个元素都是指向 dict.h/dictEntry 结构，dictEntry 结构定义如下：

```
typedef struct dictEntry{     //键     void *key;     //值     union{          void *val;          uint64_tu64;          int64_ts64;     }v;      //指向下一个哈希表节点，形成链表     struct dictEntry *next;}dictEntry            
```

key 用来保存键，val 属性用来保存值，值可以是一个指针，也可以是uint64\_t整数，也可以是int64\_t整数。

注意这里还有一个指向下一个哈希表节点的指针，我们知道哈希表最大的问题是存在哈希冲突，如何解决哈希冲突，有开放地址法和链地址法。这里采用的便是链地址法，通过next这个指针可以将多个哈希值相同的键值对连接在一起，用来**解决哈希冲突**。

![img](https://www.pdai.tech/\_images/db/redis/db-redis-ds-x-13.png)

#### 一些要点

* **哈希算法**：Redis计算哈希值和索引值方法如下：

```
#1、使用字典设置的哈希函数，计算键 key 的哈希值hash = dict->type->hashFunction(key);#2、使用哈希表的sizemask属性和第一步得到的哈希值，计算索引值index = hash & dict->ht[x].sizemask;      
```

* **解决哈希冲突**：这个问题上面我们介绍了，方法是链地址法。通过字典里面的 \*next 指针指向下一个具有相同索引值的哈希表节点。
* **扩容和收缩**：当哈希表保存的键值对太多或者太少时，就要通过 rerehash(重新散列）来对哈希表进行相应的扩展或者收缩。具体步骤：

1、如果执行扩展操作，会基于原哈希表创建一个大小等于 ht\[0].used\*2n 的哈希表（也就是每次扩展都是根据原哈希表已使用的空间扩大一倍创建另一个哈希表）。相反如果执行的是收缩操作，每次收缩是根据已使用空间缩小一倍创建一个新的哈希表。

2、重新利用上面的哈希算法，计算索引值，然后将键值对放到新的哈希表位置上。

3、所有键值对都迁徙完毕后，释放原哈希表的内存空间。

* **触发扩容的条件**：

1、服务器目前没有执行 BGSAVE 命令或者 BGREWRITEAOF 命令，并且负载因子大于等于1。

2、服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令，并且负载因子大于等于5。

ps：负载因子 = 哈希表已保存节点数量 / 哈希表大小。

* **渐近式 rehash**

什么叫渐进式 rehash？也就是说扩容和收缩操作不是一次性、集中式完成的，而是分多次、渐进式完成的。如果保存在Redis中的键值对只有几个几十个，那么 rehash 操作可以瞬间完成，但是如果键值对有几百万，几千万甚至几亿，那么要一次性的进行 rehash，势必会造成Redis一段时间内不能进行别的操作。所以Redis采用渐进式 rehash,这样在进行渐进式rehash期间，字典的删除查找更新等操作可能会在两个哈希表上进行，第一个哈希表没有找到，就会去第二个哈希表上进行查找。但是进行 增加操作，一定是在新的哈希表上进行的。

### 整数集 - IntSet

> 整数集合（intset）是集合类型的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis 就会使用整数集合作为集合键的底层实现。

#### intset结构

首先看源码结构

```
typedef struct intset {    uint32_t encoding;    uint32_t length;    int8_t contents[];} intset;
```

`encoding` 表示编码方式，的取值有三个：INTSET\_ENC\_INT16, INTSET\_ENC\_INT32, INTSET\_ENC\_INT64

`length` 代表其中存储的整数的个数

`contents` 指向实际存储数值的连续内存区域, 就是一个数组；整数集合的每个元素都是 contents 数组的一个数组项（item），各个项在数组中按值得大小**从小到大有序排序**，且数组中不包含任何重复项。（虽然 intset 结构将 contents 属性声明为 int8\_t 类型的数组，但实际上 contents 数组并不保存任何 int8\_t 类型的值，contents 数组的真正类型取决于 encoding 属性的值）

#### 内存布局图

其内存布局如下图所示

![img](https://www.pdai.tech/\_images/db/redis/db-redis-ds-x-8.png)

我们可以看到，content数组里面每个元素的数据类型是由encoding来决定的，那么如果原来的数据类型是int16, 当我们再插入一个int32类型的数据时怎么办呢？这就是下面要说的intset的升级。

#### 整数集合的升级

当在一个int16类型的整数集合中插入一个int32类型的值，整个集合的所有元素都会转换成32类型。 整个过程有三步：

* 根据新元素的类型（比如int32），扩展整数集合底层数组的空间大小，并为新元素分配空间。
* 将底层数组现有的所有元素都转换成与新元素相同的类型， 并将类型转换后的元素放置到正确的位上， 而且在放置元素的过程中， 需要继续维持底层数组的有序性质不变。
* 最后改变encoding的值，length+1。

**那么如果我们删除掉刚加入的int32类型时，会不会做一个降级操作呢**？

不会。主要还是减少开销的权衡。

### 跳表 - ZSkipList

> 跳跃表结构在 Redis 中的运用场景只有一个，那就是作为有序列表 (Zset) 的使用。跳跃表的性能可以保证在查找，删除，添加等操作的时候在对数期望时间内完成，这个性能是可以和平衡树来相比较的，而且在实现方面比平衡树要优雅，这就是跳跃表的长处。跳跃表的缺点就是需要的存储空间比较大，属于利用空间来换取时间的数据结构。

#### 什么是跳跃表

> 跳跃表要解决什么问题呢？如果你一上来就去看它的实现，你很难理解设计的本质，所以先要看它的设计要解决什么问题。

对于于一个单链表来讲，即便链表中存储的数据是有序的，如果我们要想在其中查找某个数据，也只能从头到尾遍历链表。这样查找效率就会很低，时间复杂度会很高，是 O(n)。比如查找12，需要7次查找

![img](https://www.pdai.tech/\_images/db/redis/db-redis-ds-x-9.png)

如果我们增加如下两级索引，那么它搜索次数就变成了3次

![img](https://www.pdai.tech/\_images/db/redis/db-redis-ds-x-10.png)

#### Redis跳跃表的设计

redis跳跃表并没有在单独的类（比如skplist.c)中定义，而是其定义在server.h中, 如下:

```
/* ZSETs use a specialized version of Skiplists */typedef struct zskiplistNode {    sds ele;    double score;    struct zskiplistNode *backward;    struct zskiplistLevel {        struct zskiplistNode *forward;        unsigned int span;    } level[];} zskiplistNode;typedef struct zskiplist {    struct zskiplistNode *header, *tail;    unsigned long length;    int level;} zskiplist;           
```

其内存布局如下图:

![img](https://www.pdai.tech/\_images/db/redis/db-redis-ds-x-11.png)

**zskiplist的核心设计要点**

* **头结点**不持有任何数据, 且其level\[]的长度为32
* 每个结点
  * `ele`字段，持有数据，是sds类型
  * `score`字段, 其标示着结点的得分, 结点之间凭借得分来判断先后顺序, 跳跃表中的结点按结点的得分升序排列.
  * `backward`指针, 这是原版跳跃表中所没有的. 该指针指向结点的前一个紧邻结点.
  *   ```
      level
      ```

      字段, 用以记录所有结点(除过头结点外)；每个结点中最多持有32个zskiplistLevel结构. 实际数量在结点创建时, 按幂次定律随机生成(不超过32). 每个zskiplistLevel中有两个字段

      * `forward`字段指向比自己得分高的某个结点(不一定是紧邻的), 并且, 若当前zskiplistLevel实例在level\[]中的索引为X, 则其forward字段指向的结点, 其level\[]字段的容量至少是X+1. 这也是上图中, 为什么forward指针总是画的水平的原因.
      * `span`字段代表forward字段指向的结点, 距离当前结点的距离. 紧邻的两个结点之间的距离定义为1.

#### 为什么不用平衡树或者哈希表

* **为什么不是平衡树，先看下作者的回答**

[https://news.ycombinator.com/item?id=1171423](https://news.ycombinator.com/item?id=1171423)

```
There are a few reasons:They are not very memory intensive. It's up to you basically. Changing parameters about the probability of a node to have a given number of levels will make then less memory intensive than btrees.A sorted set is often target of many ZRANGE or ZREVRANGE operations, that is, traversing the skip list as a linked list. With this operation the cache locality of skip lists is at least as good as with other kind of balanced trees.They are simpler to implement, debug, and so forth. For instance thanks to the skip list simplicity I received a patch (already in Redis master) with augmented skip lists implementing ZRANK in O(log(N)). It required little changes to the code.About the Append Only durability & speed, I don't think it is a good idea to optimize Redis at cost of more code and more complexity for a use case that IMHO should be rare for the Redis target (fsync() at every command). Almost no one is using this feature even with ACID SQL databases, as the performance hint is big anyway.About threads: our experience shows that Redis is mostly I/O bound. I'm using threads to serve things from Virtual Memory. The long term solution to exploit all the cores, assuming your link is so fast that you can saturate a single core, is running multiple instances of Redis (no locks, almost fully scalable linearly with number of cores), and using the "Redis Cluster" solution that I plan to develop in the future.           
```

简而言之就是实现简单且达到了类似效果。

* **skiplist与平衡树、哈希表的比较**

来源于：[https://www.jianshu.com/p/8ac45fd01548](https://www.jianshu.com/p/8ac45fd01548)

skiplist和各种平衡树（如AVL、红黑树等）的元素是有序排列的，而哈希表不是有序的。因此，在哈希表上只能做单个key的查找，不适宜做范围查找。所谓范围查找，指的是查找那些大小在指定的两个值之间的所有节点。

在做范围查找的时候，平衡树比skiplist操作要复杂。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历并不容易实现。而在skiplist上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现。

平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而skiplist的插入和删除只需要修改相邻节点的指针，操作简单又快速。

从内存占用上来说，skiplist比平衡树更灵活一些。一般来说，平衡树每个节点包含2个指针（分别指向左右子树），而skiplist每个节点包含的指针数目平均为1/(1-p)，具体取决于参数p的大小。如果像Redis里的实现一样，取p=1/4，那么平均每个节点包含1.33个指针，比平衡树更有优势。

查找单个key，skiplist和平衡树的时间复杂度都为O(log n)，大体相当；而哈希表在保持较低的哈希值冲突概率的前提下，查找时间复杂度接近O(1)，性能更高一些。所以我们平常使用的各种Map或dictionary结构，大都是基于哈希表实现的。

从算法实现难度上来比较，skiplist比平衡树要简单得多。

## RDB和AOF机制详解

为了防止数据丢失以及服务重启时能够恢复数据，Redis支持数据的持久化，主要分为两种方式，分别是RDB和AOF; 当然实际场景下还会使用这两种的混合模式。

* **为什么需要持久化**？

Redis是个基于内存的数据库。那服务一旦宕机，内存中的数据将全部丢失。通常的解决方案是从后端数据库恢复这些数据，但后端数据库有性能瓶颈，如果是大数据量的恢复，1、会对数据库带来巨大的压力，2、数据库的性能不如Redis。导致程序响应慢。所以对Redis来说，实现数据的持久化，避免从后端数据库中恢复数据，是至关重要的。

## 发布订阅模式

## 事务

## 事件

## 高可用

## 性能调优

##
