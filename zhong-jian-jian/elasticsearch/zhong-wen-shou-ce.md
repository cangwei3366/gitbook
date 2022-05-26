# 中文手册

### 基础

Elasticsearch是一个开源的搜索引擎，建立在全文搜索引擎Apache Lucene基础之上。通过隐藏Lucene的复杂性，取而代之的提供一套简单一致的RESTFUL API.

下面官网对其的形容：

* 一个分布式的实时文档存储，_每个字段_ 可以被索引与搜索
* 一个分布式实时分析搜索引擎
* 能胜任上百个服务节点的扩展，并支持 PB 级别的结构化或者非结构化数据

全文搜索

```json
GET /logindex/users/_search
{
    "query" : {
        "match" : {
            "about" : "click protal and search"
        }
    }
}
```

短语匹配

```json
GET /logindex/users/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "refesh step"
        }
    }
}
```

高亮匹配

```json
GET /logindex/users/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "refesh step"
        }
    },
    "highlight":{
        "fields":{
            "about":{}
        }
    }
}
```

分析聚合（aggregations）

类似SQL中的group by但更强大

```json
组合查询
GET /logindex/users/_search
{
  "query": {
    "match": {
      "last_name": "cc"
    }
  },
  "aggs": {
    "all_interests": {
      "terms": {
        "field": "interests"
      }
    }
  }
}
分级汇总
GET /logindex/users/_search
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" },
            "aggs" : {
                "avg_age" : {
                    "avg" : { "field" : "age" }
                }
            }
        }
    }
}
```

#### 集群原理

Elasticsearch的主旨是随时可用和按需扩容，天生分布式的设计使得用户无需关注对多节点进行管理维护，其自身会通过管理多节点来提高扩容性和可用性。

集群状态

```
GET /_cluster/health
```

green 所有主分片和副本分片都正常运行

yellow 所有的主分片都正常运行，但不是所有的副本分片都正常运行

red 有主分片没能正常运行。有主分片没能正常运行

#### 索引

索引实际上是指向一个或者多个物理分片的逻辑命名空间。一个分片就是一个Lucene实例

注意：理想情况下，一个主分片最大能够存储 Integer.MAX\_VALUE - 128 个文档。但是实际最大值还需要参考使用场景：包括硬件，文档大小和复杂度，索引和查询文档的方式以及期望的响应时长。

索引默认情况下会被分配5个主分片，具体可通过参数设置。（每个主分片拥有n个副分片）

```
PUT /blogs
{
   "settings" : {
      "number_of_shards" : 3,
      "number_of_replicas" : 1
   }
}
```

#### 故障转移

当你在同一台机器上启动了第二个节点时，只要它和第一个节点有同样的 `cluster.name` 配置，它就会自动发现集群并加入到其中。 但是在不同机器上启动节点的时候，为了加入到同一集群，你需要配置一个可连接到的单播主机列表。

```yaml
discovery.zen.ping.unicast.hosts: ["host1", "host2:port"]	
```

### 数据的读写

使用自定义ID

```
PUT /{index}/{type}/{id}
{
  "field": "value",
  ...
}
```

自动生成ID

自动生成的 ID 是 URL-safe、 基于 Base64 编码且长度为20个字符的 GUID 字符串。 这些 GUID 字符串由可修改的 FlakeID 模式生成，这种模式允许多个节点并行生成唯一 ID ，且互相之间的冲突概率几乎为零。

创建全新文档

```
PUT /logindex/blogs/123/_create
{ ... }
```

如果创建新文档的请求成功执行，Elasticsearch 会返回元数据和一个 `201 Created` 的 HTTP 响应码。

另一方面，如果具有相同的 `_index` 、 `_type` 和 `_id` 的文档已经存在，Elasticsearch 将会返回 `409 Conflict` 响应码以及错误信息。

Elasticsearch删除文档不会立即将文档从磁盘中删除，只是将文档标记为已删除状态。随着不断的索引更多的数据，Elasticsearch 将会在后台清理标记为已删除的文档。

#### 并发控制

悲观并发控制

这种方法被关系型数据库广泛使用，它假定有变更冲突可能发生，因此阻塞访问资源以防止冲突。 一个典型的例子是读取一行数据之前先将其锁住，确保只有放置锁的线程能够对这行数据进行修改。

乐观并发控制

Elasticsearch 中使用的这种方法假定冲突是不可能发生的，并且不会阻塞正在尝试的操作。 然而，如果源数据在读写当中被修改，更新将会失败。应用程序接下来将决定该如何解决冲突。 例如，可以重试更新、使用新的数据、或者将相关情况报告给用户。

利用version号来确保应用中相互冲突的变更不会导致数据丢失

```
PUT /website/blog/1?version=1 
{
  "title": "My first blog entry",
  "text":  "Starting to get the hang of this..."
}
```

在我们索引中的文档只有现在的 `_version` 为 `1` 时，本次更新才能成功。否则返回`409 Confilct`错误。

通过外部系统使用版本控制

如果你的主数据库已经有了版本号 — 或一个能作为版本号的字段值比如 `timestamp` — 那么你就可以在 Elasticsearch 中通过增加 `version_type=external` 到查询字符串的方式重用这些相同的版本号， 版本号必须是大于零的整数， 且小于 `9.2E+18` — 一个 Java 中 `long` 类型的正值。

外部版本号的处理方式和我们之前讨论的内部版本号的处理方式有些不同， Elasticsearch 不是检查当前 `_version` 和请求中指定的版本号是否相同， 而是检查当前 `_version` 是否 _小于_ 指定的版本号。 如果请求成功，外部的版本号作为文档的新 `_version` 进行存储。

```
PUT /website/blog/2?version=10&version_type=external
{
  "title": "My first external blog entry",
  "text":  "This is a piece of cake..."
}
```

#### 更新

部分更新

```
POST /website/blog/1/_update
{
   "doc" : {
      "tags" : [ "testing" ],
      "views": 0
   }
}
```

脚本更新

```
POST /website/blog/1/_update
{
   "script" : "ctx._source.views+=1"
}

POST /website/blog/1/_update
{
   "script" : "ctx._source.tags+=new_tag",
   "params" : {
      "new_tag" : "search"
   } 
}

POST /website/blog/1/_update
{
   "script" : "ctx.op = ctx._source.views == count ? 'delete' : 'none'",
    "params" : {
        "count": 1
    }
}
```

更新的文档可能不存在

指定如果文档不存在则先创建

```
POST /website/pageviews/1/_update
{
   "script" : "ctx._source.views+=1",
   "upsert": {
       "views": 1
   }
}
```

更新和冲突

指定重试次数

```
POST /website/pageviews/1/_update?retry_on_conflict=5 
{
   "script" : "ctx._source.views+=1",
   "upsert": {
       "views": 0
   }
}
```

#### 批量请求

```
GET /_mget
{
   "docs" : [
      {
         "_index" : "website",
         "_type" :  "blog",
         "_id" :    2
      },
      {
         "_index" : "website",
         "_type" :  "pageviews",
         "_id" :    1,
         "_source": "views"
      }
   ]
}
#如果检索的数据都在相同的_index中，则可以预指定
GET /website/blog/_mget
{
   "docs" : [
      { "_id" : 2 },
      { "_type" : "pageviews", "_id" :   1 }
   ]
}
#如果检索的数据在相同的_index和_type中，则可以预指定
GET /website/blog/_mget
{
   "ids" : [ "2", "1" ]
}
```

返回结果顺序与请求顺序一致，且可以通过`found`字段判断是否存在

#### 批量操作

减少与服务请求发送与响应的消耗时间，如下所示:

```json
{ action: { metadata }}\n
{ request body        }\n
{ action: { metadata }}\n
{ request body        }\n
...
#每行必须以\n结尾，这些行不能包含未转义的换行符
例如：
POST /_bulk
{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }} 
{ "create": { "_index": "website", "_type": "blog", "_id": "123" }}
{ "title":    "My first blog post" }
{ "index":  { "_index": "website", "_type": "blog" }}
{ "title":    "My second blog post" }
{ "update": { "_index": "website", "_type": "blog", "_id": "123", "_retry_on_conflict" : 3} }
{ "doc" : {"title" : "My updated blog post"} }



```

且如果`_index`和`_type`相同,则无需重复定义，例如：

```
POST /website/_bulk
{ "index": { "_type": "log" }}
{ "event": "User logged in" }
```

批量大小如何设置？

一个好的办法是开始时将 1,000 到 5,000 个文档作为一个批次, 如果你的文档非常大，那么就减少批量的文档个数。密切关注你的批量请求的物理大小往往非常有用，一千个 1KB 的文档是完全不同于一千个 1MB 文档所占的物理大小。 一个好的批量大小在开始处理后所占用的物理大小约为 5-15 MB。

### 分布式文档存储

路由策略

```
shard = hash(routing) % number_of_primary_shards
```

`routing` 是一个可变值，默认是文档的 `_id` ，也可以设置成一个自定义的值。 `routing` 通过 hash 函数生成一个数字，然后这个数字再除以 `number_of_primary_shards` （主分片的数量）后得到 **余数** 。这个分布在 `0` 到 `number_of_primary_shards-1` 之间的余数，就是我们所寻求的文档所在分片的位置。

这就解释了为什么我们要在创建索引的时候就确定好主分片的数量 并且永远不会改变这个数量：因为如果数量变化了，那么所有之前路由的值都会无效，文档也再也找不到了。

所有的文档 API（ `get` 、 `index` 、 `delete` 、 `bulk` 、 `update` 以及 `mget` ）都接受一个叫做 `routing` 的路由参数 ，通过这个参数我们可以自定义文档到分片的映射。一个自定义的路由参数可以用来确保所有相关的文档——例如所有属于同一个用户的文档——都被存储到同一个分片中。

为什么 `bulk` API 需要有换行符的有趣格式，而不是发送包装在 JSON 数组中的请求，例如 `mget` API？

在批量请求中引用的每个文档可能属于不同的主分片， 每个文档可能被分配给集群中的任何节点。这意味着批量请求 `bulk` 中的每个 _操作_ 都需要被转发到正确节点上的正确分片。

如果单个请求被包装在 JSON 数组中，那就意味着我们需要执行以下操作：

* 将 JSON 解析为数组（包括文档数据，可以非常大）
* 查看每个请求以确定应该去哪个分片
* 为每个分片创建一个请求数组
* 将这些数组序列化为内部传输格式
* 将请求发送到每个分片

这是可行的，但需要大量的 RAM 来存储原本相同的数据的副本，并将创建更多的数据结构，Java虚拟机（JVM）将不得不花费时间进行垃圾回收。

相反，Elasticsearch可以直接读取被网络缓冲区接收的原始数据。 它使用换行符字符来识别和解析小的 `action/metadata` 行来决定哪个分片应该处理每个请求。

这些原始请求会被直接转发到正确的分片。没有冗余的数据复制，没有浪费的数据结构。整个请求尽可能在最小的内存中处理。

### 搜索

#### 多索引，多类型

```
/_search    	#在所有索引搜索所有的类型
/gb,us/_search  #在gb和us索引中搜索所有文档
/g*,u*/_search  #模糊匹配
/gb,us/user,tweet/_search #在gb和us索引中搜索user和tweet类型
/_all/user,tweet/_search  #在所有索引中搜索user和tweet类型
```

#### 在分布式系统中深度分页

理解为什么深度分页是有问题的，我们可以假设在一个有 5 个主分片的索引中搜索。 当我们请求结果的第一页（结果从 1 到 10 ），每一个分片产生前 10 的结果，并且返回给 _协调节点_ ，协调节点对 50 个结果排序得到全部结果的前 10 个。

现在假设我们请求第 1000 页—结果从 10001 到 10010 。所有都以相同的方式工作除了每个分片不得不产生前10010个结果以外。 然后协调节点对全部 50050 个结果排序最后丢弃掉这些结果中的 50040 个结果。

可以看到，在分布式系统中，对结果排序的成本随分页的深度成指数上升。这就是 web 搜索引擎对任何查询都不要返回超过 1000 个结果的原因。

#### 轻量搜索

```
#name 字段中包含 mary 或者 john
#date 值大于 2014-09-10
#_all 字段包含 aggregations 或者 geo
?q=%2Bname%3A(mary+john)+%2Bdate%3A%3E2014-09-10+%2B(aggregations+geo)
```

以简洁的表达很复杂的查询，这种精简让调试更加晦涩和困难。而且很脆弱，一些查询字符串中很小的语法错误，像 `-` ， `:` ， `/` 或者 `"` 不匹配等，将会返回错误而不是搜索结果。不推荐直接向用户暴露查询字符串搜索功能

#### 映射与分析

**分析**

包含下面的过程：

将一块文本分成适合倒排索引的独立的词条

之后，将这些词条归一化为标准格式以提高可搜索性

分析器执行以上操作。包括三个功能

字符过滤器

​ 首先，字符串按顺序通过每个字符过滤器。过滤HTML内容，或者将&转换为and

分词器

​ 接下来，字符串被分词器分为单个词条。

Token过滤器

​ 最后，词条按顺序通过每个token过滤器。这个过程可能改变词条（小写化，词干提取），删除词条（例如a ,and,the等无用词），或者增加词条（例如，同义词）。

Elasticsearch提供了开箱即用的字符过滤器、分词器和token 过滤器。 这些可以组合起来形成自定义的分析器以用于不同的目的。

内置分析器

例如：

```
"Set the shape to semi-transparent by calling set_trans(5)"
```

标准分析器（standard）

标准分析器是Elasticsearch默认使用的分析器。它是分析各种语言文本最常用的选择。它根据 [Unicode 联盟](http://www.unicode.org/reports/tr29/) 定义的 _单词边界_ 划分文本。删除绝大部分标点。最后，将词条小写。它会产生

```
set, the, shape, to, semi, transparent, by, calling, set_trans, 5
```

**简单分析器**（simple）

简单分析器在任何不是字母的地方分隔文本，将词条小写。它会产生

```
set, the, shape, to, semi, transparent, by, calling, set, trans
```

**空格分析器**（whitespace）

空格分析器在空格的地方划分文本。它会产生

```
Set, the, shape, to, semi-transparent, by, calling, set_trans(5)
```

**语言分析器**（english）

特定语言分析器可用于 [很多语言](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-lang-analyzer.html)。它们可以考虑指定语言的特点。例如， `英语` 分析器附带了一组英语无用词（常用单词，例如 `and` 或者 `the` ，它们对相关性没有多少影响），它们会被删除。 由于理解英语语法的规则，这个分词器可以提取英语单词的 _词干_ 。

`英语` 分词器会产生下面的词条：

```
set, shape, semi, transpar, call, set_tran, 5
```

注意看 `transparent`、 `calling` 和 `set_trans` 已经变为词根格式。

测试分析器

```
GET /_analyze
{
  "analyzer": "standard",
  "text": "Text to analyze"
}
#响应
{
   "tokens": [
      {
         "token":        "text",
         "start_offset": 0,
         "end_offset":   4,
         "type":         "<ALPHANUM>",
         "position":     1
      },
      {
         "token":        "to",
         "start_offset": 5,
         "end_offset":   7,
         "type":         "<ALPHANUM>",
         "position":     2
      },
      {
         "token":        "analyze",
         "start_offset": 8,
         "end_offset":   15,
         "type":         "<ALPHANUM>",
         "position":     3
      }
   ]
}
token 是实际存储到索引中的词条。 position 指明词条在原始文本中出现的位置。 start_offset 和 end_offset 指明字符在原始字符串中的位置。
```

**映射**

核心简单域映射

* 字符串: `string`
* 整数 : `byte`, `short`, `integer`, `long`
* 浮点数: `float`, `double`
* 布尔型: `boolean`
* 日期: `date`

当你索引一个包含新域的文档—之前未曾出现-- Elasticsearch 会使用动态映射 ，通过JSON中基本数据类型，尝试猜测域类型。

注：当未指定映射类型十，根据第一条插入数据动态映射。被映射为日期类型的字段，非日期格式数据无法插入。

```js
#获取索引gb中类型tweet的映射
GET /gb/_mapping/tweet

index：analyzed|not_analyzed|no
分别代表分析缩影、索引这个域，但索引的是精确值、不索引这个域
```

更新映射

可增加，但是无法修改，避免索引中的数据出错

核心复杂域映射

```
{ "tag": [ "search", "nosql" ]}
#数组中所有值必须是相同数据类型
内部对象映射
{
    "tweet":            "Elasticsearch is very flexible",
    "user": {
        "id":           "@johnsmith",
        "gender":       "male",
        "age":          26,
        "name": {
            "full":     "John Smith",
            "first":    "John",
            "last":     "Smith"
        }
    }
}
Lucene通过扁平化处理为
{
    "tweet":            [elasticsearch, flexible, very],
    "user.id":          [@johnsmith],
    "user.gender":      [male],
    "user.age":         [26],
    "user.name.full":   [john, smith],
    "user.name.first":  [john],
    "user.name.last":   [smith]
}
内部对象数组
{
    "followers": [
        { "age": 35, "name": "Mary White"},
        { "age": 26, "name": "Alex Jones"},
        { "age": 19, "name": "Lisa Smith"}
    ]
}
扁平化处理为
{
    "followers.age":    [19, 26, 35],
    "followers.name":   [alex, jones, lisa, smith, mary, white]
}
但是丢失了之间的相关性
```

#### 请求体查询

```
match_all #查询简单的匹配所有文档
match	  #对全文字段进行分析并查询 对精确字段进行精确匹配
multi_match #在多个字段执行相同的match
{
    "multi_match": {
        "query":    "full text search",
        "fields":   [ "title", "body" ]
    }
}
range查询
{
	“range":{
		"age":{
			"gte":20,
			"lt":30
		}
	}
}
term精确匹配
需要设置该字段的mapping为not_analyze
terms多值精确匹配
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}

exists 和 missing字段判断查询
{//判断指定字段是否有值
	”exists":{
		"field":"title"
	}
}

should
如果满足这些语句中的任意语句，将增加 _score ，否则，无任何影响。它们主要用于修正每个文档的相关性得分。
filter
必须 匹配，但它以不评分、过滤模式来进行。这些语句对评分没有贡献，只是根据过滤标准来排除或包含文档。
constant_score
非精确查找，标志无需对查询进行评分计算，所有评分默认返回1。它将一个不变的常量评分应用于所有匹配的文档。它被经常用于你只需要执行一个 filter 而没有其它查询（例如，评分查询）的情况下。
{
    "constant_score":   {
        "filter": {
            "term": { "category": "ebooks" } 
        }
    }
}
```

验证查询

```
GET /gb/tweet/_validate/query
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```

此api可以验证请求是否合法。

?explain参数可以返回关于查询不合法的信息

#### 排序与相关性

```
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : { "term" : { "user_id" : 1 }}
        }
    },
    "sort": { "date": { "order": "desc" }}
}
_score不被计算，因为它没用用于排序

多级排序
"sort":[
        { "date":   { "order": "desc" }},
        { "_score": { "order": "desc" }}
    ]
```

字符串排序

按字符串的每一个字符排序

```
"tweet": { 
    "type":     "string",
    "analyzer": "english",
    "fields": {
        "raw": { 
            "type":  "string",
            "index": "not_analyzed"
        }
    }
}
搜索排序
GET /_search
{
    "query": {
        "match": {
            "tweet": "elasticsearch"
        }
    },
    "sort": "tweet.raw"
}
multi_match多字段搜索语法
GET /_search
{
  "query": {
    "multi_match": {
      "type":     "most_fields", 
      "query":    "not happy foxes",
      "fields": [ "title", "title.english" ]
    }
  }
}
```

**相关性计算**

Elasticsearch 的相似度算法被定义为检索词频率/反向文档频率， _TF/IDF_ ，包括以下内容：

**检索词频率**

检索词在该字段出现的频率？出现频率越高，相关性也越高。 字段中出现过 5 次要比只出现过 1 次的相关性高。

**反向文档频率**

每个检索词在索引中出现的频率？频率越高，相关性越低。检索词出现在多数文档中会比出现在少数文档中的权重更低。

**字段长度准则**

字段的长度是多少？长度越长，相关性越低。 检索词出现在一个短的 title 要比同样的词出现在一个长的 content 字段权重更大。

TF = （某一类中词条出现次数）/（该类所有词条数）

IDF = log(语料文档总数/(包含该词条的文档数+1))

TF-IDF = TF \* IDF

通过explain参数可以理解匹配信息

**DOC Values**

`Elasticsearch` 中的 `Doc Values` 常被应用到以下场景：

* 对一个字段进行排序
* 对一个字段进行聚合
* 某些过滤，比如地理位置过滤
* 某些与字段相关的脚本计算

#### 分布式检索

对单个文档处理，文档的唯一性由\_index,\_type和routing values（通常默认是该文档的\_id）的组合来确定集群中哪个分片含有此文档。

搜索需要一种更加复杂的执行模型，因为我们不知道查询会命中哪些文档。且找到所有匹配的文档后，多分片中的结果必须组合成排序列表。为此，搜索被执行成一个两阶段过程，我们称之为query then fetch。

**查询阶段**

查询会广播到索引的每一个分片拷贝（主分片或副分片）。每个分片在本地执行搜索并构建一个匹配文档的优先队列。

![查询过程分布式搜索](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/elas\_0901.png)

查询阶段包含以下三个步骤:

1. 客户端发送一个 `search` 请求到 `Node 3` ， `Node 3` 会创建一个大小为 `from + size` 的空优先队列。
2. `Node 3` 将查询请求转发到索引的每个主分片或副本分片中。每个分片在本地执行查询并添加结果到大小为 `from + size` 的本地有序优先队列中。
3. 每个分片返回各自优先队列中所有文档的 ID 和排序值给协调节点，也就是 `Node 3` ，它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表。

当一个搜索请求被发送到某个节点时，这个节点就变成了**协调节点**。 这个节点的任务是广播查询请求到所有相关分片并将它们的响应整合成全局排序后的结果集合，这个结果集合会返回给客户端。

**取回阶段**

深分页

from+size分页请求：from过大时，查询阶段会对大量文档进行搜索匹配，取回阶段又过滤了前n条，指取了size条，这样会占用、浪费大量的CPU、内存和带宽。

如果确实需要从集群取回大量文档，可以通过scroll查询禁用排序使取回行为更有效率。

游标查询

游标查询允许我们 先做查询初始化，然后再批量地拉取结果。 这有点儿像传统数据库中的 _cursor_ 。

游标查询会取某个时间点的快照数据。 查询初始化之后索引上的任何变化会被它忽略。 它通过保存旧的数据文件来实现这个特性，结果就像保留初始化时的索引 _视图_ 一样。

深度分页的代价根源是结果集全局排序，如果去掉全局排序的特性的话查询结果的成本就会很低。 游标查询用字段 `_doc` 来排序。 这个指令让 Elasticsearch 仅仅从还有结果的分片返回下一批结果。

```
GET /old_index/_search?scroll=1m 
{
    "query": { "match_all": {}},
    "sort" : ["_doc"], 
    "size":  1000
}
1m:保持游标一分钟
下一次查询以上一次查询返回的scrollid作为参数
GET /_search/scroll
{
    "scroll": "1m", 
    "scroll_id" : "xxxxxxx"
}
```

**搜索选项**

偏好

偏好这个参数 `preference` 允许 用来控制由哪些分片或节点来处理搜索请求。 它接受像 `_primary`, `_primary_first`, `_local`, `_only_node:xyz`, `_prefer_node:xyz`, 和 `_shards:2,3` 这样的值, 这些值在 [search `preference`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-request-preference.html) 文档页面被详细解释。

但是最有用的值是某些随机字符串，它可以避免 _bouncing results_ 问题。

想象一下有两个文档有同样值的时间戳字段，搜索结果用 `timestamp` 字段来排序。 由于搜索请求是在所有有效的分片副本间轮询的，那就有可能发生主分片处理请求时，这两个文档是一种顺序， 而副本分片处理请求时又是另一种顺序。

这就是所谓的 _bouncing results_ 问题: 每次用户刷新页面，搜索结果表现是不同的顺序。 让同一个用户始终使用同一个分片，这样可以避免这种问题， 可以设置 `preference` 参数为一个特定的任意值比如用户会话ID来解决。

超时问题

time\_out标明分片是否返回的是部分结果

路由

routing

搜索类型

缺省的搜索类型是 `query_then_fetch` 。 在某些情况下，你可能想明确设置 `search_type` 为 `dfs_query_then_fetch` 来改善相关性精确度。搜索类型 `dfs_query_then_fetch` 有预查询阶段，这个阶段可以从所有相关分片获取词频来计算全局词频。

#### 索引管理

```js
#禁止自动创建索引
action.auto_create_index: false
#删除索引
DELETE /mysql_index
DELETE /index_one,index_two
DELETE /index_*
DELETE /_all
DELETE /*
#开启指定删除功能 防止全部删除命令或通配符删除
action.destructive_requires_name: true

主分片设置
number_of_shards
副分片设置
number_of_replicas
```

**配置分析器**

一个分析器就是在一个包里面组合了三种函数的一个包装器，三种函数按照顺序被执行：

字符过滤器：

字符过滤器 用来 `整理` 一个尚未被分词的字符串。例如，包含HTML格式的文本会被清除html\_strip。

一个分析器可能有0个或者多个字符过滤器

分词器：

一个分析器有一个唯一的分词器。

词单元过滤器：

对词元进行修改，添加或移除。例如，lowercase和stop过滤器，词干过滤器，ngram和edge\_ngram过滤器可以产生适合用于部分匹配或者自动补全的词单元。

自定义分析器

```json
模板：
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": { ... custom character filters ... },
            "tokenizer":   { ...    custom tokenizers     ... },
            "filter":      { ...   custom token filters   ... },
            "analyzer":    { ...    custom analyzers      ... }
        }
    }
}

PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": { #自定义字符过滤器
                "&_to_and": {
                    "type":       "mapping",
                    "mappings": [ "&=> and "]
            }},
            "filter": {#自定义词过滤器
                "my_stopwords": {
                    "type":       "stop",
                    "stopwords": [ "the", "a" ]
            }},
            "analyzer": {
                "my_analyzer": {
                    "type":         "custom",
                    "char_filter":  [ "html_strip", "&_to_and" ],
                    "tokenizer":    "standard",
                    "filter":       [ "lowercase", "my_stopwords" ]
            }}
}}}
#测试使用
GET /my_index/_analyze?analyzer=my_analyzer
The quick & brown fox
```

**类型与映射**

不同类型的同一字段建立不同映射会建立失败，原因是以\_type的形式，共享所有相同的映射。映射在本质上被 _扁平化_ 成一个单一的、全局的模式。

#### 分片内部原理

* 为什么搜索是 _近_ 实时的？
* 为什么文档的 CRUD (创建-读取-更新-删除) 操作是 _实时_ 的?
* Elasticsearch 是怎样保证更新被持久化在断电时也不丢失数据?
* 为什么删除文档不会立刻释放空间？
* `refresh`, `flush`, 和 `optimize` API 都做了什么, 你什么情况下应该使用他们

倒排索引被写入磁盘后是 _不可改变_ 的:它永远不会修改。 不变性有重要的价值：

* 不需要锁。如果你从来不更新索引，你就不需要担心多进程同时修改数据的问题。
* 一旦索引被读入内核的文件系统缓存，便会留在哪里，由于其不变性。只要文件系统缓存中还有足够的空间，那么大部分读请求会直接请求内存，而不会命中磁盘。这提供了很大的性能提升。
* 其它缓存(像filter缓存)，在索引的生命周期内始终有效。它们不需要在每次数据改变时被重建，因为数据不会变化。
* 写入单个大的倒排索引允许数据被压缩，减少磁盘 I/O 和 需要被缓存到内存的索引的使用量。

当然，一个不变的索引也有不好的地方。主要事实是它是不可变的! 你不能修改它。如果你需要让一个新的文档 可被搜索，你需要重建整个索引。这要么对一个索引所能包含的数据量造成了很大的限制，要么对索引可被更新的频率造成了很大的限制。

如何在保证不变性的前提下实现倒排索引的更新？解决方案:用更多的索引。

**动态更新索引**

通过增加新的补充索引来反映新近的修改，而不是直接重写整个倒排索引。每一个倒排索引都会被轮流查询到—从最早的开始—查询完后再对结果进行合并。

Elasticsearch 基于 Lucene, 这个 java 库引入了 _按段搜索_ 的概念。 每一 _段_ 本身都是一个倒排索引， 但 _索引_ 在 Lucene 中除表示所有 _段_ 的集合外， 还增加了 _提交点_ 的概念，新的文档首先被添加到内存索引缓存中，因为fsync操作代价太大，所有加入了内=磁盘文件缓存，最后写入到一个基于磁盘的段。

逐段搜索会以如下流程进行工作：

1. 新文档被收集到内存索引缓存
2. 不时地, 缓存被 _提交_ ：
   * 一个新的段—一个追加的倒排索引—被写入磁盘。
   * 一个新的包含新段名字的 _提交点_ 被写入磁盘。
   * 磁盘进行 _同步_ — 所有在文件系统缓存中等待的写入都刷新到磁盘，以确保它们被写入物理文件。
3. 新的段被开启，让它包含的文档可见以被搜索。
4. 内存缓存被清空，等待接收新的文档。

当一个查询被触发，所有已知的段按顺序被查询。词项统计会对所有段的结果进行聚合，以保证每个词和每个文档的关联都被准确计算。 这种方式可以用相对较低的成本将新文档添加到索引。

删除和更新

段是不可改变的，所以既不能从把文档从旧的段中移除，也不能修改旧的段来进行反映文档的更新。 取而代之的是，每个提交点会包含一个 `.del` 文件，文件中会列出这些被删除文档的段信息。

当一个文档被 “删除” 时，它实际上只是在 `.del` 文件中被 _标记_ 删除。一个被标记删除的文档仍然可以被查询匹配到， 但它会在最终结果被返回前从结果集中移除。

文档更新也是类似的操作方式：当一个文档被更新时，旧版本文档被标记删除，文档的新版本被索引到一个新的段中。 可能两个版本的文档都会被一个查询匹配到，但被删除的那个旧版本文档在结果集返回前就已经被移除。

refreshAPI

在 Elasticsearch 中，写入和打开一个新段的轻量的过程叫做 _refresh_ 。 默认情况下每个分片会每秒自动刷新一次。这就是为什么我们说 Elasticsearch 是 _近_ 实时搜索: 文档的变化并不是立即对搜索可见，但会在一秒之内变为可见。

这些行为可能会对新用户造成困惑: 他们索引了一个文档然后尝试搜索它，但却没有搜到。这个问题的解决办法是用 `refresh` API 执行一次手动刷新:

```json
POST /blogs/_refresh 
```

并不是所有的情况都需要每秒刷新。可能你正在使用 Elasticsearch 索引大量的日志文件， 你可能想优化索引速度而不是近实时搜索， 可以通过设置 `refresh_interval` ， 降低每个索引的刷新频率。

持久化变更

如果没有用 `fsync` 把数据从文件系统缓存刷（flush）到硬盘，我们不能保证数据在断电甚至是程序正常退出之后依然存在。为了保证 Elasticsearch 的可靠性，需要确保数据变化被持久化到磁盘。

即使通过每秒刷新（refresh）实现了近实时搜索，我们仍然需要经常进行完整提交来确保能从失败中恢复。但在两次提交之间发生变化的文档怎么办？我们也不希望丢失掉这些数据。

Elasticsearch 增加了一个 _translog_ ，或者叫事务日志，在每一次对 Elasticsearch 进行操作时均进行了日志记录。通过 translog ，整个流程看起来是下面这样：

一个文档被索引之后，就会被添加到内存缓冲区，_并且_ 追加到了 translog

刷新（refresh）使分片处于缓存被清空但是事务日志不会清空的状态，分片每秒被刷新（refresh）一次：

* 这些在内存缓冲区的文档被写入到一个新的段中，且没有进行 `fsync` 操作。
* 这个段被打开，使其可被搜索。
* 内存缓冲区被清空。

每隔一段时间—例如 translog 变得越来越大—索引被刷新（flush）；一个新的 translog 被创建，并且一个全量提交被执行。在刷新（flush）之后，段被全量提交，并且事务日志被清空

* 所有在内存缓冲区的文档都被写入一个新的段。
* 缓冲区被清空。
* 一个提交点被写入硬盘。
* 文件系统缓存通过 `fsync` 被刷新（flush）。
* 老的 translog 被删除。

translog 提供所有还没有被刷到磁盘的操作的一个持久化纪录。当 Elasticsearch 启动的时候， 它会从磁盘中使用最后一个提交点去恢复已知的段，并且会重放 translog 中所有在最后一次提交后发生的变更操作。

translog 也被用来提供实时 CRUD 。当你试着通过ID查询、更新、删除一个文档，它会在尝试从相应的段中检索之前， 首先检查 translog 任何最近的变更。这意味着它总是能够实时地获取到文档的最新版本。

这个执行一个提交并且截断 translog 的行为在 Elasticsearch 被称作一次 _flush_ 。 分片每30分钟被自动刷新（flush），或者在 translog 太大的时候也会刷新。

```json
POST /blogs/_flush  #刷新索引

POST /_flush?wait_for_ongoing #等待所有刷新完成 
```

默认 translog 是每 5 秒被 `fsync` 刷新到硬盘， 或者在每次写请求完成之后执行(e.g. index, delete, update, bulk)。这个过程在主分片和复制分片都会发生。最终， 基本上，这意味着在整个请求被 `fsync` 到主分片和复制分片的translog之前，你的客户端不会得到一个 200 OK 响应。

在每次请求后都执行一个 fsync 会带来一些性能损失，尽管实践表明这种损失相对较小（特别是bulk导入，它在一次请求中平摊了大量文档的开销）。

但是对于一些大容量的偶尔丢失几秒数据问题也并不严重的集群，使用异步的 fsync 还是比较有益的。比如，写入的数据被缓存到内存中，再每5秒执行一次 `fsync` 。

这个行为可以通过设置 `durability` 参数为 `async` 来启用：

```js
{
    "index.translog.durability": "async",
    "index.translog.sync_interval": "5s"
}
```

**段合并**

`optimize` API大可看做是 _强制合并_ API。它会将一个分片强制合并到 `max_num_segments` 参数指定大小的段数目。 这样做的意图是减少段的数量（通常减少到一个），来提升搜索性能。

`optimize` API _不应该_ 被用在一个活跃的索引————一个正积极更新的索引。后台合并流程已经可以很好地完成工作。 optimizing 会阻碍这个进程。不要干扰它！

在特定情况下，使用 `optimize` API 颇有益处。例如在日志这种用例下，每天、每周、每月的日志被存储在一个索引中。 老的索引实质上是只读的；它们也并不太可能会发生变化。

在这种情况下，使用optimize优化老的索引，将每一个分片合并为一个单独的段就很有用了；这样既可以节省资源，也可以使搜索更加快速：

```json
POST /logstash-2014-10/_optimize?max_num_segments=1
```

请注意，使用 `optimize` API 触发段合并的操作不会受到任何资源上的限制。这可能会消耗掉你节点上全部的I/O资源, 使其没有余裕来处理搜索请求，从而有可能使集群失去响应。 如果你想要对索引执行 `optimize`，你需要先使用分片分配（查看 [迁移旧索引](https://www.elastic.co/guide/cn/elasticsearch/guide/current/retiring-data.html#migrate-indices)）把索引移到一个安全的节点，再执行。

## 深入搜索

## 处理人类语言

全文搜索是一场 _查准率_ 与 _查全率_ 之间的较量—查准率即尽量返回较少的无关文档，而查全率则尽量返回较多的相关文档。

* 对于用户查询意图的识别，以下是一些可优化的地方。
* 清理变音符号，对词元进行归一化
* 提取单词词干，清除单数和复数差异以及时态差异，还原为词根
* 同义词匹配
* 拼写纠错

### 语言分析器

多语言分析器配置

```
PUT /my_index
{
  "mappings": {
    "blog": {
      "properties": {
        "title": { 
          "type": "string",
          "fields": {
            "english": { 
              "type":     "string",
              "analyzer": "english"
            }
          }
        }
      }
    }
  }
}
单独的english分析器会去除not停用词
主title字段使用standard分气息
title.english子字段使用english分析器
再进行多字段搜索语法搜索
GET /_search
{
  "query": {
    "multi_match": {
      "type":     "most_fields", 
      "query":    "not happy foxes",
      "fields": [ "title", "title.english" ]
    }
  }
}
注：not和no属于english分析器停用词列表，会反转他们后面词汇的含义
自定义分析器停用词
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_english": {
          "type": "english",
          "stem_exclusion": [ "organization", "organizations" ],#防止被缩减为词干 
          "stopwords": [ 
            "a", "an", "and", "are", "as", "at", "be", "but", "by", "for",
            "if", "in", "into", "is", "it", "of", "on", "or", "such", "that",
            "the", "their", "then", "there", "these", "they", "this", "to",
            "was", "will", "with"
          ]
        }
      }
    }
  }
}
PUT /my_index
{
    "settings": {
        "analysis": {
            "analyzer": {
                "my_html_analyzer": {
                    "tokenizer":     "standard",
                    "char_filter": [ "html_strip" ]#自定义过滤器
                }
            }
        }
    }
}
GET /my_index/_analyze?analyzer=my_html_analyzer
<p>Some d&eacute;j&agrave; vu <a href="http://somedomain.com>">website</a>
```

### 词汇识别

ICU插件使用国际化组件Unicode函数库，对处理亚洲语言特别有用的icu\_分词器，还有大量对除英语外其他语言进行正确匹配和排序所必须的分词过滤器。

```
./bin/plugin -install elasticsearch/elasticsearch-analysis-icu/$VERSION 
在每一个节点运行此命令
```

IK分词器与ICU分词器对比

### 归一化词元

现在使用最多的语汇单元过滤器是lowercase过滤器；它将每个词元转换为小写。

```js
GET /_analyze?tokenizer=standard&filters=lowercase
The QUICK Brown FOX! 
```

### 词根还原

### 停用词

### 同义词

### 拼写纠错

## 聚合

## 地理位置

## 数据建模

## 监控、管理和部署
