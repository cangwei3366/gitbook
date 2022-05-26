# 问题记录

*   \_bulk单次最大数据量处理

    一般建议1000-5000个文档，如果文档很大，可以适当减少队列，大小建议是5-15MB，默认不能超过100M，可以在es的配置文件（即$ES\_HOME下的config下的elasticsearch.yml）中
* \_reindex 实测速度大概是bulk导入数据的5-10倍

```
POST _reindex
{
  "source": {
    "index": "old_index"
  },
  "dest": {
    "index": "new_index",
    "version_type": "internal" #强制转移，若目标索引存在数据，则覆盖
  }
}
```

​ 迁移效率：数据量几十个G的时候，\_reindex太慢

1）批量大小值可能太小。需要结合堆内存、线程池调整大小；

```
批量大小取决于数据、分析和集群配置，但一个好的起点是每批处理5-15 MB。
POST _reindex
{
  "source": {
    "index": "source",
    "size": 5000
  },
  "dest": {
    "index": "dest",
    "routing": "=cat" #文档路由，如果source_index中文档无路由，则无需设置
  }
}
```

2）reindex的底层是scroll实现，借助scroll并行优化方式，提升效率；

Reindex支持Sliced Scroll以并行化重建索引过程。 这种并行化可以提高效率，并提供一种方便的方法将请求分解为更小的部分

用Scroll遍历数据那确实是接受不了，现在Scroll接口可以并发来进行数据遍历了。

每个Scroll请求，可以分成多个Slice请求，可以理解为切片，各Slice独立并行，利用Scroll重建或者遍历要快很多倍。

slicing的设定分为两种方式：手动设置分片、自动设置分片。

自动设置分片如下：

```
POST _reindex?slices=5&refresh
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

大小设置注意事项：

​ 1）slices大小的设置可以手动指定，或者设置slices设置为auto，auto的含义是：针对单索引，slices大小=分片数；针对多索引，slices=分片的最小值。 ​ 2）当slices的数量等于索引中的分片数量时，查询性能最高效。slices大小大于分片数，非但不会提升效率，反而会增加开销。 ​ 3）如果这个slices数字很大(例如500)，建议选择一个较低的数字，因为过大的slices 会影响性能。

​ 4）ES副本数设置为0。如果要进行大量批量导入，可以考虑通过设置index.number\_of\_replicas来禁用副本：0。主要原因在于：复制

文档时，将整个文档发送到副本节点，并逐字重复索引过程。 这意味着每个副本都将执行分析，索引和潜在合并过程。相反，如果您使

用零副本进行索引，然后在提取完成时启用副本，则恢复过程本质上是逐字节的网络传输。 这比复制索引过程更有效。

```
PUT /my_logs/_settings
{
	"number_of_replicas": 1
}
```

1.  增加refresh间隔 如果你的搜索结果不需要接近实时的准确性，考虑先不要急于索引刷新refresh。可以将每个索引的refresh\_interval到30s。

    如果正在进行大量数据导入，可以通过在导入期间将此值设置为-1来禁用刷新。完成后不要忘记重新启用它! 设置方法：

```
PUT /my_logs/_settings
{ 
	"refresh_interval": -1 
}
```

实践证明，比默认设置reindex速度能提升10倍+。

3）跨索引、跨集群的核心是写入数据，考虑写入优化角度提升效率。

* esindex 名称不支持大写，遇到oracle 同一个名称（区分大小写时），比如 TEST和test时在ES索引库中只能以小写命名。
* 基于es6.3.2版本测试 es index 最大支持数 目前测试2K个后kibana 索引管理界面加载崩溃，es 服务器挂掉。（预计咱们7W个idnex）如果索引中数据量较大，考虑reindex时消耗时间问题，来解决sockettimeout问题。
*   index 设置codec=best\_compression。

    field String 类型设置成keyword类型。

    两者功能作用压缩率约40%。
