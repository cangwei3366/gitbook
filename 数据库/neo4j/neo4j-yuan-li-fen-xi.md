# Neo4j原理分析

## 底层原理

### 1.存储模型

The node records contain only a pointer to their first property and their first relationship (in what is oftentermed the \_relationship chain). From here, we can follow the (doubly) linked-list of relationships until we find the one we’re interested in, the LIKES relationship from Node 1 to Node 2 in this case. Once we’ve found the relationship record of interest, we can simply read its properties if there are any via the same singly-linked list structure as node properties, or we can examine the node records that it relates via its start node and end node IDs. These IDs, multiplied by the node record size, of course give the immediate offset of both nodes in the node store file.

上面的英文摘自 《Graph Databases》 (作者：IanRobinson) 一书，描述了 neo4j 的存储模型。

Node和Relationship 的 Property 是用一个 Key-Value 的双向列表来保存的；

Node 的 Relatsionship 是用一个双向列表来保存的，通过关系，可以方便的找到关系的 from-to Node节点的ID

这些id乘以节点记录的大小，当然给出了节点存储中两个节点的直接偏移量地址。

Node 节点保存第1个属性和第1个关系ID。

通过上述存储模型，从一个Node-A开始，可以方便的遍历以该Node-A为起点的图。

示例1&#x20;

在这个例子中，A~~E表示Node 的编号，R1~~R7 表示 `Relationship` 编号，P1\~P10 表示`Property` 的编号。

* Node 的存储示例图如下,每个`Node` 保存了第1个`Property` 和 第1个`Relationship`：&#x20;
* 关系的存储示意图如下：  从示意图可以看出，从 Node-B 开始，可以通过关系的 next 指针，遍历Node-B 的所有关系，然后可以到达与其有关系的第1层Nodes,在通过遍历第1层Nodes的关系，可以达到第2层Node。

### 2.物理存储

Neo4j数据库文件可持久存储，以实现长期持久性。默认情况下，Neo4j数据目录中位于data / databases / graph.db（v3.x +）中的与数据相关的文件。下面将为您提供以neostore。\*开头的文件类型以及它们存储的数据的概念：

* nodestore \*存储图形中与节点相关的数据
* 关系\*存储与在图形中创建和定义的关系有关的数据
* property \*存储数据库中的键/值属性
* label \*存储图形中与索引相关的标签数据

由于Neo4j是一个无模式的数据库，因此我们使用固定的记录长度来保留数据并遵循这些文件中的偏移量来了解如何获取数据来回答查询。下表说明了Neo4j用于存储的Java对象类型的固定大小：

```
储存档案| 记录大小| 内容
------------------------------------------------- -------------------------------------------------- ------------------------- 
neostore.nodestore.db | 15 B | 节点
neostore.relationshipstore.db | 34 B | 关系
neostore.propertystore.db | 41 B | 节点和关系的属性
neostore.propertystore.db.strings | 128 B | 字符串属性的值
neostore.propertystore.db.arrays | 128 B | 数组属性的值
索引属性| 1/3 * AVG（X）| 每个索引条目大约是平均属性值大小的1/3
```

有关属性的一些注意事项：

* 该属性记录的有效载荷为32bytes，分为四个8B块。每个字段可以保存一个键或一个值，或者既可以包含一个键又可以包含一个值。
* 密钥和类型占用3.5个字节（密钥4位，类型24位）
* 布尔，字节，短，整数，字符，浮点数适合于同一块的其余字节
* 一小块长块也装在同一块里
* 大的long或double分别存储在单独的块中（意味着该属性使用了两个块）
* 对字符串存储区或数组存储区的引用与键位于同一块中
* 如果短字符串或短数组适合剩余的块（包括键块的剩余字节），则将其存储在同一记录中
* 不能容纳在8B块中的长字符串/数组将具有指向字符串/数组存储区（128B）中的记录的指针
* ......其他类型参与其中！

磁盘上存储的数据是固定大小记录的所有链接列表。属性存储为属性记录的链接列表，每个属性记录都包含一个键和值并指向下一个属性。每个节点和关系都引用其第一个属性记录。节点还引用其关系链中的第一个关系。每个关系都引用其开始和结束节点。它还分别引用了起始节点和结束节点的上一个和下一个关系记录。

基本的磁盘空间计算示例可能类似于：

场景1-初始状态

* 节点数：4M节点
  * 每个节点具有3个属性（总共1200万个属性）
* 关系数：200万个关系
  * 每个关系都有1个属性（总共2M个属性）

这将转换为磁盘上的以下大小：

* 节点：4.000.000x15B = 60.000.000B（60MB）
* 关系：2.000.000x34B = 68.000.000B（68MB）
* 属性：14.000.000x41B = 574.000.000B（574MB）
* 总计：**703MB**

方案2-4倍增长+增加的属性+所有属性的索引

* 节点数：1600万个节点
  * 每个节点具有5个属性（总共80M个属性）
* 关系数：800万个关系
  * 每个关系具有2个属性（总共1600万个属性）

这将转换为磁盘上的以下大小：

* 节点：16.000.000x15B = 240.000.000B（240MB）
* 关系：8.000.000x34B = 272.000.000B（272MB）
* 属性：96.000.000x41B = 3.936.000.000B（3936MB）
* 索引：4448MB \*〜33％= 1468MB
* 总计：**5916MB**

## 查询原理

### 1. 查询调优

#### 1.1 查询如何执行

Cypher执行引擎会将每个Cypher查询都转为一个执行计划。在执行查询时，执行计划将告知Neo4j执行什么样的操作。

#### 1.2查询性能分析

查看执行计划对查询进行分析时有两个Cypher语句可用：

**1.2.1 EXPLAIN**

如果只想查看查询计划，而不想运行该语句，可以在查询语句中加入EXPLAIN。此时，该语句将返回空结果，对数据库不会做出任何改变。

```
EXPLAIN MARCH(n) return count(n)
```

**1.2.2 PROFILE**

如果想运行查询语句并查看哪个运算符占了大部分的工作，可以使用PROFILE。此时，该语句将被运行，并跟踪传递了多少行数据给每个运算符，以及每个运算符与存储层交互了多少以获取必要的数据。注意，加入PROFILE的查询语句将占用更多的资源，所以除非真正在做性能分析，否则不要使用PROFILE。

```
PROFILE MARCH(n) return count(n)
```

#### 1.3查询调优举例

写一个找到'山西省'的查询语句。比较初级的做法是按如下方式写：

```
MATCH (p { name: '山西省' })
RETURN p
```

这个查询将会找到 '山西省' 节点，但是随着数据库中节点数的增加，该查询将越来越慢。可以通过使用PROFILE来找到原因。

```
PROFILE
MATCH (p { name: '山西省' })
RETURN p
```

&#x20;首先需要记住的是，查看执行计划应该从底端往上看。在这个过程中，我们注意到从最后一行开始的Rows列中的数字远高于给定的name属性为 '山西省' 的一个节点。在Operator列中我们看到AllNodeScan被使用到了，这意味着查询计划器扫描了数据库中的所有节点。

向上移动一行看Filter运算符，它将检查由AllNodeScan传入的每个节点的name属性。这看起来是一种非常低效的方式来查找 '山西省'。

解决这个问题的办法是，无论什么时候我们查询一个节点，都应该指定一个标签来帮助查询计划器缩小搜索空间的范围。对于这个查询，可简单地添加一个Person标签。

```
PROFILE
MATCH (p:Person { name: 'Tom Hanks' })
RETURN p
```

&#x20;这次最后一行Rows的值已经降低了，这里没有扫描到之前扫描到的那些节点。NodeByLabelScan运算符表明首先在数据库中做了一个针对所有标签节点的线性扫描。一旦完成后，后续将针对所有节点执行Filter运算符，依次比较每个节点的name属性。

这在某些情况下看起来还可以接受，但是如果频繁通过name属性来查询标签，针对带有标签的节点的name属性创建索引将获得更好的性能。

```
CREATE INDEX ON :quhua1(name)
```

现在再次运行该查询将运行得更快。

```
PROFILE
MATCH (p:quhua1 { name: 'Tom Hanks' })
RETURN p
```

&#x20;查询计划下降到单一的行并使用了NodeIndexSeek运算符，它通过模式索引寻找到对应的节点。

### 2.执行计划

本小节主要描述执行计划(Execution Plan)中的运算符。

Neo4j将执行一个查询的任务分解为一些被称为运算符的小块。每个运算符负责整个查询中的一小部分。这些以模式(pattern)形式连接在一起的运算符被称为一个执行计划。

每个运算符用如下统计信息来注解：

Rows 运算符产生的行数。只有带有profile的查询才有。 EstimatedRows由运算符所产生的预估的行数。编译器使用这个估值来选择合适的执行计划。 DbHits每个运算符都会向Neo4j存储引擎请求像检索或者更新数据这样的工作\*\*。数据库命中次数\*\*就是DbHits。

## 多标签多索引分析

Overall Aim: • Investigate the performance improvement of using index structures that result from multi-label keys for update and query operations in the de-facto industry standard for graph databases, Neo4j • Quantify the impact of the keys, their size, and the size of their index

根据工作需要，对neo4j关于多标签以及索引结构在查询、导入和更新时的执行时间以及原理分析，以便进行性能改进。以下是分析结果：

&#x20;在多标签检索时，优先命中有索引的标签。

多标签对数据导入影响很大。

在多标签查询时，无索引会导致命中标签个数倍数的的db\_hit，所以最好有索引。

此次统计是根据接口返回参数：result\_available\_after、result\_consumed\_after。其说明如下：

result\_available\_after is the time before the first result was found and made available. There may be many more results that still need to be found and streamed from the query, but this is reasonably when a client can begin getting results streamed back.

result\_consumed\_after is the time at which all results were found and consumed by / sent to the client that made the query. The query is completely finished at this point.

As for seeing result\_available\_after go to 0, that's likely a combination of two things:

Since you just ran the query previously, the query has been cached, and so for subsequent executions there is no need to compile and plan, the plan is just retrieved from the cache and executed. This factors out compile and planning time.

If the query was on the same data, then the parts of the graph touched and retrieved should now be in the pagecache. On subsequent executions, it will be faster to traverse since for traversal it only has to hit the pagecache. So this factors out disk I/O. This is why it is ideal to have enough RAM to allow pagecache to be configured to cover as much of the graph as possible, to avoid the costs of disk I/O.

## 常用语句总结

```


数据导入
可以通过添加约束的方式来优化导入速度
CREATE CONSTRAINT ON (区划:区划) assert 区划.code is uniquewang

创建关系
MATCH (a:Test),(b:TEST2)
WHERE a.name ='TEST-NAME' AND b.name = 'TEST-NAME1'
CREATE (a)-[r:名称]->(b) 
RETURN r

创建多个节点多个关系
CREATE p = (a:Test{name:' '}) -[:rel1] -> (node) <- [:rel2] -(b:Test{name:' '})

删除所有节点
match(n) detach delete n

删除某一关系
MATCH p=()-[r:`属于`]->() DELETE p 

数据导入
可以通过添加约束的方式来优化导入速度
CREATE CONSTRAINT ON (entityName:entityName) assert entityname.property is unique
为source和target节点添加后，导入速度不论是关系还是节点都很快

关系导入
USING PERIODIC COMMIT 300 设置自动提交条数 防止内存溢出
LOAD CSV WITH HEADERS FROM "file:///zf-qh.csv" AS line  match (from:政府{id:line.from}),(to:区划{code:line.to})  merge (from)-[r:属于{}]->(to)
* WITH HEADERS 忽略csv文件的头
*MERGE 防止重复

节点导入
USING PERIODIC COMMIT 1000
LOAD CSV FROM "file:///actors.csv" AS line
MERGE (a:actors{personId:line[0],name:line[1],type:line[2]})

空值处理
LOAD CSV FROM "file:///rel民族-省市2.csv" as line match (from:区划{code:line[0]}),(to:民族{name:line[4]}) merge (from)-[r:包括{num:case when line[1] is not null then line[1] else [] end ,boynum:case when line[2] is not null then line[2] else [] end,girlnum:case when line[3] is not null then line[3] else [] end}]-(to)

修改标签和关系
match(n)-[r:拥有]->(m) create(n)-[r2:包括]->(m) set r2=r with r delete r

```

参考链接：https://sunxiang0918.cn/2015/06/27/neo4j-%E5%BA%95%E5%B1%82%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84%E5%88%86%E6%9E%90/
