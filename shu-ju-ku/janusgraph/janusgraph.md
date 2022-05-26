# janusgraph

### 1. janusgraph底层存储结构

1.图存储方式一般两种：邻接列表|邻接矩阵

​ janusgraph以有向邻接列表进行图数据存储

该一个顶点的邻接列表包含该节点对应的属性和关联的边

2.图切割

Vertex Cut 根据点切割，每个边只存储一次，只要是节点对应的边便会多一份该节点的存储。

Edge Cut 以节点为中心，边会存两次，源节点的邻接列表存储一次，目标节点的邻接列表也会存储一次

如a--edgeA-->b

按边切割为： a:propertys;edgeA

​ b:propertys;edgeA

默认边切割

3.BigTable模型

janusgraph将图形的邻接列表的表示存储在支持bigtable数据模型的存储后端。 ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9RdDdYNlFTT2RsRm5jT2Y5bEwySTl2VGRYdVRic1RibU5la3VDcU1SaWF4V3pKaHlBOUFFVVNlcE1pY0pkOTJmM0xqemRFbUx2dnVYV083aWJOcThzRXVzdy82NDA?x-oss-process=image/format,png)

* sorted by key：标识存储后端存储的数据时按照key的大小进行排序存储的
*   sorted by column：这是JanusGraph对Bigtable数据模型有一个额外要求，存储`edge(边)`的单元格必须按column排序，并且列范围指定的单元格子集必须是有效可检索的；

    九大列族:

    e edgestore f edgestore\_lock g graphindex h graphindex\_lock i janusgraoh\_ids l system\_properties m system\_properties\_lock s systemlog t txlog

### 2. 存储逻辑结构：

1.整体结构

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9RdDdYNlFTT2RsRm5jT2Y5bEwySTl2VGRYdVRic1RibUpZSXpRZmliTDVGWGh2VWpkOFlpYjZpY0hpYUxPWVBDUzdYQ1MzM0E4REtjeEFHTEhLZFFodlQzQVEvNjQw?x-oss-process=image/format,png)

在hbase中节点的ID作为HBase的rowkey，节点上的每一个属性和一条边，作为该rowkey行的一个个独立cell。即独立的key-column-value结构。

三种排序方式：

sort by id:根据vertex id在存储后端顺序存储

sort by type:

sort by key:

2.Vertex id的结构

Vertex id的组成结构在源码类IDManager中

/\* janusGraphElement id bit format

\*\[ 0 | count |partition | ID padding (if any)]

\*/

这是janusgraph在生成所有id时统一的格式包含vertext id\edge id\property id。

在对vertext id进行序列化存储时，位置为\[partition | 0 | count | ID padding (if any)]

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9RdDdYNlFTT2RsRm5jT2Y5bEwySTl2VGRYdVRic1RibUFDRWliaE9YMXk3c3hKSGc5TGd1amhST3FlUEoybWJ1ZmljM3lUV1FYd3ZXaWM0Rk1odGdFZ1lOUS82NDA?x-oss-process=image/format,png)

1.Vertext ID 包含一个子节、8位、64个bit

2.最高位5个bit是partition id。当Storage Backend是Hbase时，JanusGraph会根据partition数量，自动计算并配置各个HBase Region的split key，从而各个partition均匀映射到HBase的多个Region中。然后通过均匀分配partition id最终实现数据均匀打散到Storage Backend的多台机器中

3.中间的count部分是流水号，其中最高位默认的0，count最大值为2的(64-5-1-3)=55次幂大小；

4.最后几个bit是ID padding，表示Vertex的类型。最常用的普通Vertex，其值为‘000’.

**vertex id是如何保证全局唯一性的呢？**

主要是基于`数据库 + 号段`模式进行分布式id的生成；

体现在图中就是`partition id + count` 来保证分布式全局唯一性； 针对不同的`partition`都有自己的0-2的55次幂的范围的id；每次要生成vertex id时，首先获取一个partition，获取对应partition对应的一组还未使用的id，用来做count；

janusgraph在底层存储中存储了对应的partition使用了多少id，从而保证了再生成新的分布式vertex id时，不会重复生成

3.edge和property的结构

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9RdDdYNlFTT2RsRm5jT2Y5bEwySTl2VGRYdVRic1RibU1lTU1pYWliR1E2bGliR0VEYWx2ZFJFaE5FaWJ2QUUzNXRaalRoZHpWMXBFYURqOVNRa25yUEY4MHcvNjQw?x-oss-process=image/format,png)

#### 2.1 edge的结构

**Edge的Column组成部分：**

* label id：边类型代表的id，在创建图schema的时候janusgraph自动生成的label id，不同于边生成的唯一全局id
* direction：图的方向，out：0、in：1
* sort key：可以指定边的属性为sort key，可多个；在同种类型的edge下，针对于sort key进行排序存储，提升针对于指定sort key的检索速度；
*
  * 该key中使用的关系类型必须是属性非唯一键或非唯一单向边标签；
  * 存储的为配置属性的value值，可多个（只存property value是因为，已经在schema的配置中保存有当前Sort key对应的属性key了，所以没有必要再存一份）
* adjacent vertex id：target节点的节点id，其实存储的是目标节点id和源节点id的差值，这也可以减少存储空间的使用
* edge id：边的全局唯一id

**Edge的value组成部分：**

* signature key：边的签名key
*
  * 该key中使用的关系类型必须是属性非唯一键或非唯一单向边标签；
  * 存储压缩后的配置属性的value值，可多个（只存property value是因为，已经在schema的配置中保存有当前signature key对应的属性key了，所以没有必要再存一份）
  * 主要作用提升edge的属性的检索速度，将常用检索的属性设置为signature key，提升查找速度
* other properties：边的其他属性
*
  * 注意！不包含配置的sort key和signature key属性值，因为他们已经在对应的位置存储过了，不需要多次存储！
  * 此处的属性，要插入属性key label id和属性value来标识是什么属性，属性值是什么；
  * 此处的property的序列化结构不同于下述所说的vertex节点的property结构，edge中`other properties`这部分存储的属性只包含：proeprty key label id + property value；不包含`property全局唯一id`！

基于上述的edge逻辑结构，JanusGraph是如何构造邻接列表的 或者 是如何获取源节点的邻接节点的？

从上述的`整体结构`部分中，我们可以知道，vertexId行后跟着当前的节点关联的所有的edge；

而在上述的`edge`的逻辑结构中，有一个`adjacent vertex id`字段，通过这个字段就可以获取到target节点的vertex id，就相当于指向了target节点，整理一下：

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9RdDdYNlFTT2RsRm5jT2Y5bEwySTl2VGRYdVRic1RibUs4MjIzUGljdkFiRWNDSmFscWptSnNFSXJpYTBteXRYa3U3aWNUdGZ0UjRtaE9LQkhOdVdQajhiUS82NDA?x-oss-process=image/format,png)

如上图，通过上述的条件，就可以构造一个VertexA指向VertexB 和 VertexC的邻接链表；

其实，JanusGraph可以理解为构造的是`双向邻接列表`， 依据上图，我们知道vertexA 和 vertexB 和 vertexC存在边关系；当我们构造vertexB的邻接列表时，会包含指向vertexA的节点，只是说在edge对应的逻辑结构中边的方向不同：

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9RdDdYNlFTT2RsRm5jT2Y5bEwySTl2VGRYdVRic1RibTNWU3F2RU83bVlEUnRXUEZtc00wdDRJaWFkcUcwV01QekQ4WktVQVFWalRTaHptekRrSG5nWVEvNjQw?x-oss-process=image/format,png)

总结：JanusGraph通过vertex id行中包含所有关联的edge，edge逻辑结构中包含指向target节点的数据来组成双向邻接列表的结构；

#### 2.2 property的结构

property的存储结构十分的简单，只包含`key id`、`property id`和`value`三部分：

* key id：属性label对应的id，有创建schema时JanusGraph创建；不同于属性的唯一id
* property id：属性的唯一id，唯一代表某一个属性
* value：属性值

注意：属性的类型包含`SINGLE`、`LIST`和`SET`三种Cardinality；当属性被设置为`LIST`类型时，因为`LIST`允许当前的节点存在多个相同的属性kv对，仅通过`key id`也就是属性的label id是无法将相同的属性label区分出来的

所以在这种情况下，JanusGraph的`property`的存储结构有所变化， `property id`也将会被存储在`column`中，如下图：

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9RdDdYNlFTT2RsRm5jT2Y5bEwySTl2VGRYdVRic1RibWpMY3k3ekFYQlM1NVc2Z3dFcXpSSGJhT3M5Um9YdW9YUWNyaDcyenl2YjJrSHFwTGJ0MUJMdy82NDA?x-oss-process=image/format,png)
