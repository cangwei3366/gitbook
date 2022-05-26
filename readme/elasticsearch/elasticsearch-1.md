# ElasticSearch

### 一、基础分布式架构

1. 对复杂分布式机制的透明隐藏特性，比如分片、副本，负载均衡等
2. 支持垂直扩容和水平扩容
3. 增加或减少节点的数据 rebalance，保持负载均衡
4.  master节点，管理集群的元数据，索引的创建与删除，节点的增加和移除，默认选出一台节点，作为master

    节点
5. 节点平等的架构，每个节点都能接受所有请求，自动请求路由，响应收集

### 二、shard\&replica机制梳理

（1）index包含多个shard

（2）每个shard都是一个最小工作单元，承载部分数据，lucene实例，完整的建立索引和处理请求的能力

（3）增减节点时，shard会自动在nodes中负载均衡

（4）primary shard和replica shard，每个document肯定只存在于某一个primary shard以及其对应的replica

shard中，不可能存在于多个primary shard

（5）replica shard是primary shard的副本，负责容错，以及承担读请求负载

（6）primary shard的数量在创建索引的时候就固定了，replica shard的数量可以随时修改

（7）primary shard的默认数量是5，replica默认是1，默认有10个shard，5个primary shard，5个replica shard

（8）primary shard不能和自己的replica shard放在同一个节点上（否则节点宕机，primary shard和副本都丢

失，起不到容错的作用），但是可以和其他primary shard的replica shard放在同一个节点上

PUT /test\_index { "settings" : { "number\_of\_shards" : 3, "number\_of\_replicas" : 1 } }

（9）两个node下3个primary 在一个node 3个replic在一个node，单node下，无法分配replic

### 三、选举、容错、数据恢复

master选举，自动选举另外一个node成为新的master，承担起master责任

新的master，将丢失掉的primary shard 的某个replica shard提升为primary shard，此时cluster status会变为

yellow，因为primary shard全变为active了。但是少了一个replic shard。

重启故障的node，新master会将缺失的副本都copy一份到该node，而且该node会使用之前已有的shard数据，

只是同步一个宕机之后发生过的修改。

手动生成id，自动生成id

GUID算法

post /index/type { }

### 四、并发冲突问题

并发环境下，可能会得到脏数据。

1.悲观锁并发控制方案

在读取记录时，同时在这一行记录加锁，常见于关系型数据库中。

上锁之后，就只有一个线程可以操作这一条数据，包括行级锁，表级锁，读锁，写锁。

2.线程B操作时，当前数据的版本号与es中数据的版本号比对，如果相同，则进行操作，如果不同，则去es中读取

最新的数据版本，再进行操作。

优缺点：

悲观锁优点：方便、直接加锁，对应用程序透明

缺点：降低并发能力

乐观锁优点：并发能力高，不用加锁，

缺点：麻烦 ，每次更新时需要对比版本号

3.乐观锁并发控制

（1）多线程环境，请求乱序，可能先修改的后到，后修改的先到。

PUT /test\_index/test\_type/7?version=1 { "test\_field": "test client 1" } es中的数据的版本号，跟客户端中的数据的版本号是相同的，才能修改

(2)es提供了一个feature，就是说，可以不用它提供的内部\_version版本号来进行并发控制，可以基于你自己维护的\_

一个版本号来进行并发控制。举个列子，加入你的数据在mysql里也有一份，然后你的应用系统本身就维护了一

个版本号，无论是什么自己生成的，程序控制的。

?version=1

?version=1\&version\_type=external

version\_type=external，唯一的区别在于，\_version，只有当你提供的version与es中的\_version一模一样的时候，才

可以进行修改，只要不一样，就报错；当version\_type=external的时候，只有当你提供的version比es中的\_version\_

大的时候，才能完成修改

es，\_version=1，?version=1，才能更新成功 es，\_version=1，?version>1\&version\_type=external，才能成功，比如说?version=2\&version\_type=external

### 五、Parial update更新

普通--创建文档&替换文档

将要修改的数据PUT到es中进行全量替换，老的document标记为delete，然后创建一个新的document

partial update 发生在shard内部

post /index/type/id/\_update { "doc": { "要修改的少数几个field即可，不需要全量的数据" } } 通过groovy进行partial update 1.内置脚本 POST /test\_index/test\_type/id/\_update { "script":"ctx.\_source.num+=1" } 2.外部脚本 test-add-tags.groovy: ctx._source.tags+=new\_tag POST /test\_index/test\_type/id/update { "script":"{ "lang":"groovy", "file" : "test-add-tags", "params" : { "new\_tag" : "tag1" } } } 3.upsert操作 如果指定document不存在，就执行upsert的初始化操作，如果存在，就执行doch或者指定的partial update操作_ POST /test\_index/test\_type/id/\_update { "script" : "ctx.\_source.num+=1", "upsert" : { "num" : 0, "tags" : \[] } } partial update 内置乐观锁并发控制 post /index/type/id/\_update?retry\_on\_conflict=5\&version=6

es中该数据的版本号为6，而请求中的version为5，比较后发现不同，就去系统获取最新的版本进行更新操作，继

续判断版本号是否相同，如果相同则进行更新操作，如果不相同，那么继续获取最新版本号进行更新操作，持续操

作这个步骤，持续次数为retry\_on\_conflict

### 六、数据路由算法

shard = hash(routing) % number\_of\_primary\_shards

进行操作时document的\_id(手动指定or自动生成)routing=\_id，公式求余后就是primary shard的id

发送请求时可以手动指定一个routing value，如PUT /index/type/id?routing= user\_id

可以保证某一类的document一定被路由到一个shard上，方便后续进行应用级别的负载均衡，以及提升批量读取

的性能

primary shard一旦建立索引不可变，会造成数据丢失

### 七、写一致性原理以及quorum机制

(1) PUT /index/type/id?consistency=one

one: 当前写操作，只要有一个primary shard 是active活跃可用的，就可以执行

all: 要求当前写操作，必须所有的primary shard和replica shard 都是活跃的

quorum：默认值，要求所有shard中，必须满足quorum机制个可用的

(2)quorum机制

quorum=int(primary+ number\_of\_replicas)/2)+1,且number\_of\_replicas>1时才生效

如：quorum=int((3+1)/2)+1=3

所以要求6个shard中至少有3个shard是active状态的，才可以执行这个写操作

(3) 如果节点数少于quorum数量，可能导致无法执行任何写操作

当primary=1，number\_of\_replicas=1，此时就2个shard，但是可能在同一个node,导致变成单节点集群无法工作

(4) quorum 不齐全时，wait，默认一分钟，timeout,100,30s

### 八、内部查询以及请求负载均衡原理

1.客户端发送请求到任意一个node，成为coordinate node

2.coordinate node对document进行路由，请求转发到对应node此时会使用round-robin随机轮询算法，在

primary shard以及其所有replica中随机选择一个，让读请求负载均衡

3、接收请求的node返回document给coordinate node

4、coordinate node返回document给客户端

### 九、分词流程

1.character filter 预处理,过滤html标签，特殊符号处理：&->and

2.tokenizer： 单词分词

3.token filter ：时态以及大小写转换

4.同义词换行

5.常用内置分词器介绍

### 十、mapping理解

1.执行put操作时，es会自动建立索引，同时建立type以及对应的mapping

2.mapping自动定义了每个field的数据类型

3.数据类型可能是text or date or full text or exact value

4.exact value，分词时将整个值作为一个关键词建立到倒排索引；full text 会经过分词 ，normalization(时态转

换、同义词转换、大小写转换)，才会建立倒排索引

5.根据数据类型，进行搜索时的分词处理行为与存储时一致，这就是为什么存储时分词器与搜索时分词器一置的原

因。 6.可以自动or手动创建mapping

mapping，相当于index的type的元数据，每一个type都有自己的mapping，决定数据类型，建立倒排索引的行

为，以及搜索的行为。

核心数据类型： string, byte，short，integer，long float，double, boolean, date
