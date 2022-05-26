# Kafka3.0

随着kafka的版本发布，从0.1到1.x到2.x的版本当中，kafka一直强依赖外部的服务协调管理系统zk，需要通过zk来保存一些kafka的重要元数据信息，以便实现kafka架构的稳定性，可用性以及数据一致性等。然而到了kafka2.8的版本就已经试推行去zk的kafka架构了，如果去掉了zk，那么在kafka新的版本当中使用什么技术来代替了zk的位置呢，接下来我们一起来一探究竟。

## 1、kafka2当中zk的作用

在kafka2.x及之前的版本当中，一直都需要依赖于zookeeper作为协调服务，kafka集群在启动的时候，也会向zookeeper集群当中写入很多重要的元数据，我们可以一起来看一下在kafka2当中保留在zk当中的元数据有哪些

![](broken-reference)

* /admin ： 主要保存kafka当中的核心的重要信息，包括类似于已经删除的topic就会保存在这个路径下面
* /brokers ： 主要用于保存kafka集群当中的broker信息，以及没被删除的topic信息

![](broken-reference)

* /cluster : 主要用于保存kafka集群的唯一id信息，每个kafka集群都会给分配要给唯一id，以及对应的版本号

![](broken-reference)

* /config : 集群配置信息

![](broken-reference)

* /controller ： kafka集群当中的控制器信息，控制器组件（Controller），是 Apache Kafka 的核心组件。它的主要作用是在 Apache ZooKeeper 的帮助下管理和协调整个 Kafka 集群。

![](broken-reference)

* /controller\_epoch ：主要用于保存记录controller的选举的次数，每选举一次controller，该数据就加1，controller\_epoch用于记录控制器发生变更的次数，即记录当前的控制器是第几代控制器，我们也可以称之为“控制器的纪元”。controller\_epoch的初始值为1，即集群中第一个控制器的纪元为1，当控制器发生变更时，每选出一个新的控制器就将该字段值加1
* /isr\_change\_notification ： isr列表发生变更时候的通知，在kafka当中由于存在ISR列表变更的情况发生,为了保证ISR列表更新的及时性，定义了isr\_change\_notification 这个节点，主要用于通知Controller来及时将ISR列表进行变更
* /latest\_producer\_id\_block ：使用`/latest_producer_id_block`节点来保存PID块，主要用于能够保证生产者的任意写入请求都能够得到响应。

![](broken-reference)

* /log\_dir\_event\_notification ： 主要用于保存当broker当中某些LogDir出现异常时候,例如磁盘损坏,文件读写失败等异常时候,向ZK当中增加一个通知序号，controller监听到这个节点的变化之后，就会做出对应的处理操作。

![](broken-reference)

以上就是kafka在zk当中保留的所有的所有的相关的元数据信息，这些元数据信息保证了kafka集群的正常运行。

## 2、kafka3的安装配置

在kafka3的版本当中已经彻底去掉了对zk的依赖，如果没有了zk集群，那么kafka当中是如何保存元数据信息的呢，这里我们通过kafka3的集群来一探究竟。

### 1、kafka安装配置核心重要参数

**Controller 服务器**

不管是kafka2还是kafka3当中，controller控制器都是必不可少的，通过controller控制器来维护kafka集群的正常运行，例如ISR列表的变更，broker的上线或者下线，topic的创建，分区的指定等等各种操作都需要依赖于Controller，在kafka2当中，controller的选举需要通过zk来实现，我们没法控制哪些机器选举成为Controller,而在kafka3当中,我们可以通过配置文件来自己指定哪些机器成为Controller,这样做的好处就是我们可以指定一些配置比较高的机器作为Controller节点,从而保证controller节点的稳健性。

被选中的controller节点参与元数据集群的选举，每个controller节点要么是Active状态，或者就是standBy状态.

![](broken-reference)

一般都会选择奇数台服务器成为Controller,便于过半选举成功。

**Process.Roles**

使用KRaft模式来运行kafka集群的话，我们有一个配置叫做Process.Roles必须配置，这个参数有以下四个值可以进行配置

* Process.Roles = Broker, 服务器在KRaft模式中充当 Broker。
* Process.Roles = Controller, 服务器在KRaft模式下充当 Controller。
* Process.Roles = Broker,Controller，服务器在KRaft模式中同时充当 Broker 和Controller。
* 如果process.roles 没有设置。那么集群就假定是运行在ZooKeeper模式下。

如果需要从zookeeper模式转换成为KRaft模式，那么需要进行重新格式化。如果一个节点同时是Broker和Controller节点,那么就称之为组合节点。

实际工作当中，如果有条件的话，尽量还是将Broker和Controller节点进行分离部署。避免由于服务器资源不够的情况导致OOM等一系列的问题

**Quorum Voters**

通过controller.quorum.voters配置来实习哪些节点是Quorum的投票节点,所有想要成为控制器的节点,都必须放到这个配置里面。

每个Broker和每个Controller 都必须设置 `controller.quorum.voters`。需要注意的是，controller.quorum.voters 配置中提供的节点ID必须与提供给服务器的节点ID匹配。

比如在Controller1上，node.Id必须设置为1，以此类推。注意，控制器id不强制要求你从0或1开始。然而，分配节点ID的最简单和最不容易混淆的方法是给每个服务器一个数字ID，然后从0开始。

### 2、kafka3集群的环境安装搭建

#### 2.1、下载并解压安装包

bigdata01下载kafka的安装包，并进行解压

```shell
[hadoop@bigdata01 kraft]$ cd /opt/soft/
[hadoop@bigdata01 soft]$ wget http://archive.apache.org/dist/kafka/3.1.0/kafka_2.12-3.1.0.tgz
[hadoop@bigdata01 soft]$ tar -zxf kafka_2.12-3.1.0.tgz -C /opt/install/
```

修改kafka的配置文件broker.properties

```
[hadoop@bigdata01 kafka_2.12-3.1.0]$ cd /opt/install/kafka_2.12-3.1.0/config/kraft/
[hadoop@bigdata01 kraft]$ vim broker.properties
```

修改编辑内容如下

```
node.id=1
controller.quorum.voters=1@bigdata01:9093
listeners=PLAINTEXT://bigdata01:9092
advertised.listeners=PLAINTEXT://bigdata01:9092
log.dirs=/opt/install/kafka_2.12-3.1.0/kraftlogs
```

修改kafka的配置文件server.properties

```
[hadoop@bigdata01 kafka_2.12-3.1.0]$ cd /opt/install/kafka_2.12-3.1.0/config/kraft/
[hadoop@bigdata01 kraft]$ vim server.properties 
```

修改编辑内容如下：

```
node.id=1
controller.quorum.voters=1@bigdata01:9093,2@bigdata02:9093,3@bigdata03:9093
listeners=PLAINTEXT://bigdata01:9092,CONTROLLER://bigdata01:9093
advertised.listeners=PLAINTEXT://bigdata01:9092
log.dirs=/opt/install/kafka_2.12-3.1.0/topiclogs
process.roles=broker,controller
```

创建两个文件夹

```
[hadoop@bigdata01 kafka_2.12-3.1.0]$ mkdir -p /opt/install/kafka_2.12-3.1.0/kraftlogs
[hadoop@bigdata01 kafka_2.12-3.1.0]$ mkdir -p /opt/install/kafka_2.12-3.1.0/topiclogs
```

同步安装包到其他机器上面去

```
[hadoop@bigdata01 kafka_2.12-3.1.0]$ cd /opt/install/[hadoop@bigdata01 install]$ scp -r kafka_2.12-3.1.0/ bigdata02:$PWD
[hadoop@bigdata01 install]$ scp -r kafka_2.12-3.1.0/ bigdata03:$PWD
```

#### 2.2、第二台以及第三台机器修改配置文件

第二台机器bigdata02修改配置文件

```
[hadoop@bigdata02 opt]$ cd /opt/install/kafka_2.12-3.1.0/config/kraft/
[hadoop@bigdata02 kraft]$ vim broker.properties 
```

修改broker.properties内容如下

```
node.id=2
controller.quorum.voters=2@bigdata02:9093
listeners=PLAINTEXT://bigdata02:9092
advertised.listeners=PLAINTEXT://bigdata02:9092
```

修改server.properties内容如下

```
[hadoop@bigdata02 kafka_2.12-3.1.0]$ cd /opt/install/kafka_2.12-3.1.0/config/kraft/
[hadoop@bigdata02 kraft]$ vim server.properties 
```

```
node.id=2
controller.quorum.voters=1@bigdata01:9093,2@bigdata02:9093,3@bigdata03:9093
listeners=PLAINTEXT://bigdata02:9092,CONTROLLER://bigdata02:9093
advertised.listeners=PLAINTEXT://bigdata02:9092
process.roles=broker,controller
```

第三台机器bigdata03修改配置文件

```
[hadoop@bigdata03 opt]$ cd /opt/install/kafka_2.12-3.1.0/config/kraft/
[hadoop@bigdata03 kraft]$ vim broker.properties 
```

修改broker.properties内容如下

```
node.id=3
controller.quorum.voters=3@bigdata03:9093
listeners=PLAINTEXT://bigdata03:9092
advertised.listeners=PLAINTEXT://bigdata03:9092
```

修改server.properties内容如下

```
[hadoop@bigdata02 kafka_2.12-3.1.0]$ cd /opt/install/kafka_2.12-3.1.0/config/kraft/
[hadoop@bigdata02 kraft]$ vim server.properties 
```

```
node.id=3
controller.quorum.voters=1@bigdata01:9093,2@bigdata02:9093,3@bigdata03:9093
listeners=PLAINTEXT://bigdata03:9092,CONTROLLER://bigdata03:9093
advertised.listeners=PLAINTEXT://bigdata03:9092
process.roles=broker,controller
```

#### 2.3、服务器集群启动

第一台机器启动kafka服务

```
[hadoop@bigdata01 kafka_2.12-3.1.0]$  ./bin/kafka-storage.sh random-uuidYkJwr6RESgSJv-sxa1R1mA
[hadoop@bigdata01 kafka_2.12-3.1.0]$  ./bin/kafka-storage.sh format -t YkJwr6RESgSJv-sxa1R1mA -c ./config/kraft/server.propertiesFormatting /opt/install/kafka_2.12-3.1.0/topiclogs
[hadoop@bigdata01 kafka_2.12-3.1.0]$ ./bin/kafka-server-start.sh ./config/kraft/server.properties
```

![](broken-reference)

第二台机器启动kafka服务

```
[hadoop@bigdata02 kafka_2.12-3.1.0]$ ./bin/kafka-storage.sh format -t YkJwr6RESgSJv-sxa1R1mA -c ./config/kraft/server.propertiesFormatting /opt/install/kafka_2.12-3.1.0/topiclogs
[hadoop@bigdata02 kafka_2.12-3.1.0]$ ./bin/kafka-server-start.sh ./config/kraft/server.properties
```

第三台机器启动kafka服务

```
[hadoop@bigdata03 kafka_2.12-3.1.0]$ ./bin/kafka-storage.sh format -t YkJwr6RESgSJv-sxa1R1mA -c ./config/kraft/server.propertiesFormatting /opt/install/kafka_2.12-3.1.0/topiclogs
[hadoop@bigdata03 kafka_2.12-3.1.0]$ ./bin/kafka-server-start.sh ./config/kraft/server.properties
```

#### 2.4、创建kafka的topic

集群启动成功之后，就可以来创建kafka的topic了，使用以下命令来创建kafka的topic

```
./bin/kafka-topics.sh --create --topic kafka_test --partitions 3 --replication-factor 2 --bootstrap-server bigdata01:9092,bigdata02:9092,bigdata03:9092
```

![](broken-reference)

#### 2.5、任意一台机器查看kafka的topic

组成集群之后，任意一台机器就可以通过以下命令来查看到刚才创建的topic了

```
[hadoop@bigdata03 ~]$ cd /opt/install/kafka_2.12-3.1.0/
[hadoop@bigdata03 kafka_2.12-3.1.0]$ bin/kafka-topics.sh  --list --bootstrap-server bigdata01:9092,bigdata02:9092,bigdata03:9092
```

#### 2.6、消息生产与消费

使用命令行来生产以及消费kafka当中的消息

```
[hadoop@bigdata01 kafka_2.12-3.1.0]$ bin/kafka-console-producer.sh --bootstrap-server bigdata01:9092,bigdata02:9092,bigdata03:9092 --topic kafka_test
[hadoop@bigdata02 kafka_2.12-3.1.0]$ bin/kafka-console-consumer.sh --bootstrap-server bigdata01:9092,bigdata02:9092,bigdata03:9092 --topic kafka_test --from-beginning
```

## 3、Kafka当中Raft的介绍

### 3.1、kafka强依赖zk所引发的问题

前面我们已经看到了kafka3集群在没有zk集群的依赖下，也可以正常运行，那么kafka2在zk当中保存的各种重要元数据信息，在kafka3当中如何实现保存的呢？

kafka一直都是使用zk来管理集群以及所有的topic的元数据，并且使用了zk的强一致性来选举集群的controller，controller对整个集群的管理至关重要，包括分区的新增，ISR列表的维护，等等很多功能都需要靠controller来实现，然后使用zk来维护kafka的元数据也存在很多的问题以及存在性能瓶颈。

以下是kafka将元数据保存在zk当中的诸多问题。

**1、元数据存取困难**

元数据的存取过于困难，每次重新选举的controller需要把整个集群的元数据重新restore，非常的耗时且影响集群的可用性。

**2、元数据更新网络开销大**

整个元数据的更新操作也是以全量推的方式进行，网络的开销也会非常大。

**3、强耦合违背软件设计原则**

Zookeeper 对于运维来说，维护Zookeeper也需要一定的开销，并且kafka强耦合与zk也并不好，还得时刻担心zk的宕机问题，违背软件设计的高内聚，低耦合的原则。

**4、网络分区复杂度高**

Zookeeper本身并不能兼顾到broker与broker之间通信的状态，这就会导致网络分区的复杂度成几何倍数增长。

**5、zk本身不适合做消息队列**

zookeeper不适合做消息队列，因为 zookeeper有1M的消息大小限制 zookeeper的children太多会极大的影响性能 znode太大也会影响性能 znode太大会导致重启zkserver耗时10-15分钟 zookeeper仅使用内存作为存储，所以不能存储太多东西。

**6、并发访问zk问题多**

最好单线程操作zk客户端，不要并发，临界、竞态问题太多

基于以上各种问题，所以提出了脱离zk的方案，转向自助研发强一致性的元数据解决方案，也就是KIP-500。

KIP-500议案提出了在Kafka中处理元数据的更好方法。基本思想是"**Kafka on Kafka**"，将Kafka的元数据存储在Kafka本身中，无需增加额外的外部存储比如ZooKeeper等。

在KIP-500议案中，元数据将存储在Kafka分区中，而不是存储在ZooKeeper中。控制器将是该分区的所有者。Kafka本身就不会配置和管理外部元数据系统。

元数据将被视为日志。需要最新更新的Brokers只能读取日志的末尾。类似于需要最新日志条目的使用者仅需要读取日志的最后而不是整个日志的方式。Brokers还将能够在整个流程重启期间保留其元数据缓存。

去zookeeper之后的kafka新的架构

![](broken-reference)

在KIP-500中，Kafka控制器会将其元数据存储在Kafka分区中，而不是存储在ZooKeeper中。但是，由于控制器依赖于该分区，因此分区本身不能依赖控制器来进行领导者选举之类的事情。而是，管理该分区的节点必须实现自我管理的Raft仲裁。

在kafka3.0的新的版本当中，使用了新的KRaft协议，使用该协议来保证在元数据仲裁中准确的复制元数据，这个协议类似于zk当中的zab协议以及类似于Raft协议，但是KRaft协议使用的是基于事件驱动的模式，与ZAB协议和Raft协议还有点不一样

在kafka3.0之前的的版本当中，主要是借助于controller来进行leader partition的选举，而在3.0协议当中，使用了KRaft来实现自己选择leader，并最终令所有节点达成共识，这样简化了controller的选举过程，效果更加高效。

### 3.2、kafka去掉zk的优势

Kakfa Without ZooKeeper的优势：

**部署运维更加加单**：不需要额外依赖zk，也不用时刻关心zk是否正常，也不需要额外的去部署zk了。

**监控更加便捷**：其次，由于信息的集中，从Kafka获取监控信息，就变得轻而易举，不用再到zk里转一圈了。与各种监控软件的集成也更加便捷

**速度更快**：摆脱了zk的种种现实，系统的可拓展性大大增强，号称可支持百万partition，并且系统的节点启动与关闭的时间也大大降低

![](broken-reference)

### 3.2、kakfa3当中的Raft介绍

前面我们已经知道了在kafka3当中可以不用再依赖于zk来保存kafka当中的元数据了，转而使用Kafka Raft来实现元数据的一致性，简称**KRaft**，并且将元数据保存在kafka自己的服务器当中，大大提高了kafka的元数据管理的性能。

KRaft运行模式的Kafka集群，不会将元数据存储在 Apache ZooKeeper中。即部署新集群的时候，无需部署ZooKeeper集群，因为Kafka将元数据存储在 Controller 节点的 KRaft Quorum中。KRaft可以带来很多好处，比如可以支持更多的分区，更快速的切换Controller，也可以避免Controller缓存的元数据和Zookeeper存储的数据不一致带来的一系列问题。

在新的版本当中，控制器Controller节点我们可以自己进行指定,这样最大的好处就是我们可以自己选择一些配置比较好的机器成为Controller节点，而不像在之前的版本当中，我们无法指定哪台机器成为Controller节点，而且controller节点与broker节点可以运行在同一台机器上，并且控制器controller节点不再向broker推送更新消息,而是让Broker从这个Controller Leader节点进行拉取元数据的更新。

![](broken-reference)

### 3.3、如何查看kafka3当中的元数据信息

在kafka3当中，不再使用zk来保存元数据信息了，那么在kafka3当中如何查看元数据信息呢，我们也可以通过kafka自带的命令来进行查看元数据信息，在KRaft中，有两个命令常用命令脚本，kafka-dump-log.sh和kakfa-metadata-shell.sh需要我们来进行关注，因为我们可以通过这两个脚本来查看kafka当中保存的元数据信息。

#### 3.3.1、**Kafka-dump-log.sh** 脚本来导出元数据信息

KRaft模式下，所有的元数据信息都保存到了一个内部的topic上面，叫做@metadata，例如Broker的信息,Topic的信息等,我们都可以去到这个topic上面进行查看,我们可以通过kafka-dump-log.sh这个脚本来进行查看该topic的信息。

所以我们需要有一个工具查看当前的数据内容。

Kafka-dump-log.sh是一个之前就有的工具，用来查看Topic的的文件内容。这工具加了一个参数--cluster-metadata-decoder用来，查看元数据日志，如下所示:

```
[hadoop@bigdata01 kafka_2.12-3.1.0]$ cd /opt/install/kafka_2.12-3.1.0
[hadoop@bigdata01 kafka_2.12-3.1.0]$ bin/kafka-dump-log.sh  --cluster-metadata-decoder --skip-record-metadata  --files  /opt/install/kafka_2.12-3.1.0/topiclogs/__cluster_metadata-0/00000000000000000000.index,/opt/install/kafka_2.12-3.1.0/topiclogs/__cluster_metadata-0/00000000000000000000.log  >>/opt/metadata.txt
```

#### 3.3.2、kafka-metadata-shell.sh直接查看元数据信息

平时我们用zk的时候，习惯了用zk命令行查看数据，简单快捷。bin目录下自带了kafka-metadata-shell.sh工具，可以允许你像zk一样方便的查看数据。

使用kafka-metadata-shell.sh脚本进入kafka的元数据客户端

```
[hadoop@bigdata01 kafka_2.12-3.1.0]$ bin/kafka-metadata-shell.sh --snapshot /opt/install/kafka_2.12-3.1.0/topiclogs/__cluster_metadata-0/00000000000000000000.log
```

![](broken-reference)

![](broken-reference)

## 4、Raft算法介绍

raft算法中文版本翻译介绍：[https://github.com/maemual/raft-zh\_cn/blob/master/raft-zh\_cn.md](https://github.com/maemual/raft-zh\_cn/blob/master/raft-zh\_cn.md)

著名的CAP原则又称CAP定理的提出，真正奠基了分布式系统的诞生，CAP定理指的是在一个分布式系统中，[一致性](https://baike.baidu.com/item/%E4%B8%80%E8%87%B4%E6%80%A7/9840083)（Consistency）、[可用性](https://baike.baidu.com/item/%E5%8F%AF%E7%94%A8%E6%80%A7/109628)（Availability）、[分区容错性](https://baike.baidu.com/item/%E5%88%86%E5%8C%BA%E5%AE%B9%E9%94%99%E6%80%A7/23734073)（Partition tolerance），这三个[要素](https://baike.baidu.com/item/%E8%A6%81%E7%B4%A0/5261200)最多只能同时实现两点，不可能三者兼顾。

分布式系统为了提高系统的可靠性，一般都会选择使用多副本的方式来进行实现，例如hdfs当中数据的多副本，kafka集群当中分区的多副本等，\*\*但是一旦有了多副本的话，那么久面临副本之间一致性的问题，而一致性算法就是 用于解决分布式环境下多副本的数据一致性的问题。\*\*业界最著名的一致性算法就是大名鼎鼎的Paxos，但是Paxos比较晦涩难懂，不太容易理解，所以还有一种叫做Raft的算法，更加简单容易理解的实现了一致性算法。

### 4.1、Raft协议的工作原理

#### 4.1.1、Raft协议当中的角色分布

Raft协议将分布式系统当中的角色分为Leader（领导者），Follower（跟从者）以及Candidate（候选者）

* \*\*Leader：\*\*主节点的角色，主要是接收客户端请求，并向Follower同步日志，当日志同步到过半及以上节点之后，告诉follower进行提交日志
* \*\*Follower：\*\*从节点的角色，接受并持久化Leader同步的日志，在Leader通知可以提交日志之后，进行提交保存的日志
* \*\*Candidate：\*\*Leader选举过程中的临时角色。

#### 4.1.2、Raft协议当中的底层原理

Raft协议当中会选举出Leader节点，Leader作为主节点，完全负责replicate log的管理。Leader负责接受所有客户端的请求，然后复制到Follower节点，如果leader故障，那么follower会重新选举leader，Raft协议的一致性，概括主要可以分为以下三个重要部分

* Leader选举
* 日志复制
* 安全性

其中Leader选举和日志复制是Raft协议当中最为重要的。

Raft协议要求系统当中，任意一个时刻，只有一个leader，正常工作期间，只有Leader和Follower角色，并且Raft协议采用了类似网络租期的方式来进行管理维护整个集群，Raft协议将时间分为一个个的时间段（term），也叫作任期，每一个任期都会选举一个Leader来管理维护整个集群，如果这个时间段的Leader宕机，那么这一个任期结束，继续重新选举leader。

**Raft 算法将时间划分成为任意不同长度的任期（term）**。任期用连续的数字进行表示。**每一个任期的开始都是一次选举（election），一个或多个候选人会试图成为领导人**。如果一个候选人赢得了选举，它就会在该任期的剩余时间担任领导人。在某些情况下，选票会被瓜分，有可能没有选出领导人，那么，将会开始另一个任期，并且立刻开始下一次选举。**Raft 算法保证在给定的一个任期最多只有一个领导人**。

![](broken-reference)

#### 4.1.3、Leader选举的过程

Raft使用心跳来进行触发leader选举，当服务器启动时，初始化为follower角色。leader向所有Follower发送周期性心跳，如果Follower在选举超时间内没有收到Leader的心跳，就会认为leader宕机，稍后发起leader的选举。

每个Follower都会有一个倒计时时钟，是一个随机的值，表示的是Follower等待成为Leader的时间，倒计时时钟先跑完，就会当选成为Lader，这样做得好处就是每一个节点都有机会成为Leader。

![1641795323848](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1641795323848.png?lastModify=1647874003)

具体详细过程实现描述如下：

* 增加节点本地的current term，切换到candidate状态
* 自己给自己投一票
* 给其他节点发送RequestVote RPCs，要求其他节点也投自己一票
* 等待其他节点的投票回复

整个过程中的投票过程可以用下图进行表述

![Untitled Diagram](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/Untitled%20Diagram-1641796432070.png?lastModify=1647874003)

leader节点选举的限制

* 每个节点只能投一票，投给自己或者投给别人
* 候选人所知道的日志信息，一定不能比自己的更少，即能被选举成为leader节点，一定包含了所有已经提交的日志
* 先到先得的原则

#### 4.1.4、数据一致性保证（日志复制机制）

前面通过选举机制之后，选举出来了leader节点，然后leader节点对外提供服务，所有的客户端的请求都会发送到leader节点，由leader节点来调度这些并发请求的处理顺序，保证所有节点的状态一致，\*\*leader会把请求作为日志条目（Log entries）加入到他的日志当中，然后并行的向其他服务器发起AppendEntries RPC复制日志条目。\*\*当这条请求日志被成功复制到大多数服务器上面之后，Leader将这条日志应用到它的状态机并向客户端返回执行结果。

* 客户端的每个请求都包含被复制状态机执行的指令
* leader将客户端请求作为一条心得日志添加到日志文件中，然后并行发起RPC给其他的服务器，让他们复制这条信息到自己的日志文件中保存。
* 如果这条日志被成功复制，也就是大部分的follower都保存好了执行指令日志，leader就应用这条日志到自己的状态机中，并返回给客户端。
*   如果follower宕机或者运行缓慢或者数据丢失，leader会不断地进行重试，直至所有在线的follower都成功复制了所有的日志条目。

    <img src="file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1641798087757.png?lastModify=1647874003" alt="1641798087757" data-size="original">

**状态机说明：**

要让所有节点达成一致性的状态，大部分都是基于复制状态机来实现的（Replicated state machine）

简单来说就是：**初始相同的状态 + 相同的输入过程 = 相同的结束状态**，这个其实也好理解，就类似于一对双胞胎，出生时候就长得一样，然后吃的喝的用的穿的都一样，你自然很难分辨。其中最重要的就是一定要注意中间的相同输入过程，各个不同节点要以相同且确定性的函数来处理输入，而不要引入一个不确定的值。使用replicated log来实现每个节点都顺序的写入客户端请求，然后顺序的处理客户端请求，最终就一定能够达到最终一致性。

* **日志格式说明：**

所有节点持久化保存在本地的日志，大概就是类似于这个样子，

![1641799817056](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1641799817056.png?lastModify=1647874003)

上图显示，共有八条日志数据，其中已经提交了7条，提交的日志都将通过状态机持久化到本地磁盘当中，防止宕机。

**日志复制的保证机制**

* 如果两个节点不同的日志文件当中存储着**相同的索引和任期号，那么他们所存储的命令是相同的。**（原因：leader最多在一个任期里的一个日志索引位置创建一条日志条目，日志条目所在的日志位置从来不会改变）。
* 如果不同日志中两个条目有着相同的索引和任期号，那么他们之前的所有条目都是一样的（原因：每次RPC发送附加日志时，**leader会把这条日志前面的日志下标和任期号一起发送给follower，如果follower发现和自己的日志不匹配，那么就拒绝接受这条日志，这个称之为一致性检查**）

**日志的不正常情况**

一般情况下，Leader和Followers的日志保持一致，因此 Append Entries 一致性检查通常不会失败。然而，Leader崩溃可能会导致日志不一致：**旧的Leader可能没有完全复制完日志中的所有条目**。

下图阐述了一些Followers可能和新的Leader日志不同的情况。**一个Follower可能会丢失掉Leader上的一些条目，也有可能包含一些Leader没有的条目，也有可能两者都会发生**。丢失的或者多出来的条目可能会持续多个任期。

![1641802361605](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1641802361605.png?lastModify=1647874003)

**如何保证日志的正常复制**

如果出现了上述leader宕机，导致follower与leader日志不一致的情况，那么就需要进行处理，保证follower上的日志与leader上的日志保持一致，leader通过强制follower复制它的日志来处理不一致的问题，follower与leader不一致的日志会被强制覆盖。**leader为了最大程度的保证日志的一致性，且保证日志最大量，leader会寻找follower与他日志一致的地方，然后覆盖follower之后的所有日志条目，从而实现日志数据的一致性。**

具体的操作就是：leader会从后往前不断对比，每次Append Entries失败后尝试前一个日志条目，直到成功找到每个Follower的日志一致的位置点，然后向该Follower所在位置之后的条目进行覆盖

**详细过程如下：**

* Leader维护了每个Follower节点下一次要接收的日志的索引，即nextIndex
* Leader选举成功后将所有Follower的nextIndex设置为自己的最后一个日志条目+1
* Leader将数据推送给Follower，如果Follower验证失败（nextIndex不匹配），则在下一次推送日志时缩小nextIndex，直到nextIndex验证通过

总结一下就是：**当 leader 和 follower 日志冲突的时候**，leader 将**校验 follower 最后一条日志是否和 leader 匹配**，如果不匹配，**将递减查询，直到匹配，匹配后，删除冲突的日志**。这样就实现了主从日志的一致性。

### 4.2、Raft协议算法代码实现

前面我们已经大致了解了Raft协议算法的实现原理，如果我们要自己实现一个Raft协议的算法，其实就是将我们讲到的理论知识给翻译成为代码的过程，具体的开发需要考虑的细节比较多，代码量肯定也比较大，好在有人已经实现了Raft协议的算法了，我们可以直接拿过来使用一探究竟

创建maven工程并导入jar包地址如下

```
  <dependencies>        <dependency>            <groupId>com.github.wenweihu86.raft</groupId>            <artifactId>raft-java-core</artifactId>            <version>1.8.0</version>        </dependency>        <dependency>            <groupId>com.github.wenweihu86.rpc</groupId>            <artifactId>rpc-java</artifactId>            <version>1.8.0</version>        </dependency>        <dependency>            <groupId>org.rocksdb</groupId>            <artifactId>rocksdbjni</artifactId>            <version>5.1.4</version>        </dependency>    </dependencies>    <build>        <plugins>            <plugin>                <groupId>org.apache.maven.plugins</groupId>                <artifactId>maven-compiler-plugin</artifactId>                <version>3.5.1</version>                <configuration>                    <source>1.8</source>                    <target>1.8</target>                </configuration>            </plugin>        </plugins>    </build>
```

定义Server端代码实现：

```
package cn.naixue.raft.server;import cn.naixue.raft.server.service.ExampleService;import cn.naixue.raft.server.service.impl.ExampleServiceImpl;import com.github.wenweihu86.raft.RaftOptions;import com.github.wenweihu86.raft.RaftNode;import com.github.wenweihu86.raft.proto.RaftMessage;import com.github.wenweihu86.raft.service.RaftClientService;import com.github.wenweihu86.raft.service.RaftConsensusService;import com.github.wenweihu86.raft.service.impl.RaftClientServiceImpl;import com.github.wenweihu86.raft.service.impl.RaftConsensusServiceImpl;import com.github.wenweihu86.rpc.server.RPCServer;import org.apache.commons.lang3.StringUtils;import java.util.ArrayList;import java.util.List;public class Server1 {    public static void main(String[] args) {        // parse args        // peers, format is "host:port:serverId,host2:port2:serverId2"        //localhost:16010:1,localhost:16020:2,localhost:16030:3 localhost:16010:1        String servers = "localhost:16010:1,localhost:16020:2,localhost:16030:3";        // local server        RaftMessage.Server localServer = parseServer("localhost:16010:1");        String[] splitArray = servers.split(",");        List<RaftMessage.Server> serverList = new ArrayList<>();        for (String serverString : splitArray) {            RaftMessage.Server server = parseServer(serverString);            serverList.add(server);        }        // 初始化RPCServer        RPCServer server = new RPCServer(localServer.getEndPoint().getPort());        // 设置Raft选项，比如：        // just for test snapshot        RaftOptions raftOptions = new RaftOptions();      /*  raftOptions.setSnapshotMinLogSize(10 * 1024);        raftOptions.setSnapshotPeriodSeconds(30);        raftOptions.setMaxSegmentFileSize(1024 * 1024);*/        // 应用状态机        ExampleStateMachine stateMachine = new ExampleStateMachine(raftOptions.getDataDir());        // 初始化RaftNode        RaftNode raftNode = new RaftNode(raftOptions, serverList, localServer, stateMachine);        raftNode.getLeaderId();        // 注册Raft节点之间相互调用的服务        RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);        server.registerService(raftConsensusService);        // 注册给Client调用的Raft服务        RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);        server.registerService(raftClientService);        // 注册应用自己提供的服务        ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);        server.registerService(exampleService);        // 启动RPCServer，初始化Raft节点        server.start();        raftNode.init();    }    private static RaftMessage.Server parseServer(String serverString) {        String[] splitServer = serverString.split(":");        String host = splitServer[0];        Integer port = Integer.parseInt(splitServer[1]);        Integer serverId = Integer.parseInt(splitServer[2]);        RaftMessage.EndPoint endPoint = RaftMessage.EndPoint.newBuilder()                .setHost(host).setPort(port).build();        RaftMessage.Server.Builder serverBuilder = RaftMessage.Server.newBuilder();        RaftMessage.Server server = serverBuilder.setServerId(serverId).setEndPoint(endPoint).build();        return server;    }}
```

将Server端代码定义Server2以及Server3然后运行三个Server端

定义客户端代码实现如下：

```
package cn.naixue.raft.client;import cn.naixue.raft.server.service.ExampleMessage;import cn.naixue.raft.server.service.ExampleService;import com.github.wenweihu86.rpc.client.RPCClient;import com.github.wenweihu86.rpc.client.RPCProxy;import com.google.protobuf.util.JsonFormat;public class ClientMain {    public static void main(String[] args) {        // parse args        String ipPorts = args[0];        String key = args[1];        String value = null;        if (args.length > 2) {            value = args[2];        }        // init rpc client        RPCClient rpcClient = new RPCClient(ipPorts);        ExampleService exampleService = RPCProxy.getProxy(rpcClient, ExampleService.class);        final JsonFormat.Printer printer = JsonFormat.printer().omittingInsignificantWhitespace();        // set        if (value != null) {            ExampleMessage.SetRequest setRequest = ExampleMessage.SetRequest.newBuilder()                    .setKey(key).setValue(value).build();            ExampleMessage.SetResponse setResponse = exampleService.set(setRequest);            try {                System.out.printf("set request, key=%s value=%s response=%s\n",                        key, value, printer.print(setResponse));            } catch (Exception ex) {                ex.printStackTrace();            }        } else {            // get            ExampleMessage.GetRequest getRequest = ExampleMessage.GetRequest.newBuilder().setKey(key).build();            ExampleMessage.GetResponse getResponse = exampleService.get(getRequest);            try {                String value1 = getResponse.getValue();                System.out.println(value1);                System.out.printf("get request, key=%s, response=%s\n",                        key, printer.print(getResponse));            } catch (Exception ex) {                ex.printStackTrace();            }        }        rpcClient.stop();    }}
```

先启动三个服务端，然后启动客户端，就可以将实现客户端向服务端发送消息，并且服务端会向三台机器进行保存消息了。

## 5、kafka源码阅读环境准备

### 1、kafka的版本选择

我们这里选择kafka最新版本3.1.0的版本，该版本为当前最新稳定版，kafka3.1.0的源码以及安装包下载地址为：

[http://archive.apache.org/dist/kafka/3.1.0/](http://archive.apache.org/dist/kafka/3.1.0/)

下载kafka的源码包，下载地址如下：

[http://archive.apache.org/dist/kafka/3.1.0/kafka-3.1.0-src.tgz](http://archive.apache.org/dist/kafka/3.1.0/kafka-3.1.0-src.tgz)，下载源码包之后，将源码包进行解压

### 2、安装jdk

选择jdk1.8的版本进行安装即可

### 3、安装开发工具IDEA以及scala插件

省略

### 4 安装开发工具IDEA和scala插件

* 安装idea开发工具

![1576709522729](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1576709522729.png?lastModify=1647874003)

* 安装scala插件

![1576709593493](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1576709593493.png?lastModify=1647874003)

![1576709618336](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1576709618336.png?lastModify=1647874003)

![1576709669579](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1576709669579.png?lastModify=1647874003)

### 5 安装gradle

由于Kafka的源码没有采用maven去管理，而是用的gradle，大家就把这个想成是一个类似于maven的代码管理工具即可。安装它的方式跟安装maven一样。

*   下载gradle安装包，并解压

    gradle下载地址：[https://services.gradle.org/distributions/gradle-7.3.3-all.zip](https://services.gradle.org/distributions/gradle-7.3.3-all.zip)

    解压gradle压缩包到一个没有中文没有空格的路径下

    <img src="file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644203342705.png?lastModify=1647874003" alt="1644203342705" data-size="original">
* 配置gradle环境变量

![1644203527210](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644203527210.png?lastModify=1647874003)

![1644203586532](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644203586532.png?lastModify=1647874003)

*   配置gradle本地仓库位置

    gradle的本地仓库位置默认存放在C盘当前用户下，我们也可以自己定义本地仓库的位置，只需要配置GRADLE\_USER\_HOME环境变量即可

![1644214795216](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644214795216.png?lastModify=1647874003)

* 配置gradle使用国内镜像源进行下载jar包

在gradle安装的根路径下，有一个文件夹名称叫做init.d，在该文件夹下面创建一个空文件名称叫做init.gradle

![1644214599339](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644214599339.png?lastModify=1647874003)

在init.gradle文件当中填写以下内容，进行定义gradle的国内镜像源

```
allprojects{    repositories {        def REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/groups/public/'                all { ArtifactRepository repo ->            if (repo instanceof MavenArtifactRepository) {                def url = repo.url.toString()                if (url.startsWith('https://repo1.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {                    project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."                    remove repo                }            }        }        maven { ArtifactRepository repo ->             // 判断属性是否存在，之后再设置            if(repo.metaClass.hasProperty(repo, 'allowInsecureProtocol')){                allowInsecureProtocol true            }            url REPOSITORY_URL        }    }}
```

### 6、使用gradle编译kafka源码

kafka源码已经解压好了，gradle也安装好了，剩下就是通过gradle来对kafka源码进行编译

* 使用gradle来编译kafka源码

配置好了gradle的本地源之后，就可以使用gradle来进行编译kafka源码了，进入kafka源码的根路径，然后在cmd下使用`gradle idea`命令来进行编译

![1644204981752](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644204981752.png?lastModify=1647874003)

打开powershell窗口后使用gradle idea来进行编译

![1644205047630](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644205047630.png?lastModify=1647874003)

### 7、导入编译完成之后的kafka源码

编译成功之后，就可以通过idea来导入kafka源码了

![image-20201016165356802](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20201016165356802.png?lastModify=1647874003)

![1644218218956](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644218218956.png?lastModify=1647874003)

![1644230029082](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644230029082.png?lastModify=1647874003)

> Kafka的源码量不大，结构清晰简单，我们主要阅读的源码在client包和core 包。
>
> client包里有producer和consumer的源码，server端的源码在core包里。

### 8、windows下安装单节点zookeeper

* 下载zookeeper 3.6.3的压缩包，然后在windows下进行解压

[http://archive.apache.org/dist/zookeeper/zookeeper-3.6.3/apache-zookeeper-3.6.3-bin.tar.gz](http://archive.apache.org/dist/zookeeper/zookeeper-3.6.3/apache-zookeeper-3.6.3-bin.tar.gz)

![1644299756530](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644299756530.png?lastModify=1647874003)

这里我解压到了一个没有中文没有空格的路径下

*   修改zookeeper的配置文件zoo.cfg

    进入到解压之后的zookeeper的安装根路径下，在conf路径下进行修改zookeeper的配置选项zoo.cfg

    <img src="file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644299826126.png?lastModify=1647874003" alt="1644299826126" data-size="original">
* 修改zoo.cfg配置选项

```
dataDir=D:\\soft_ware_install\\apache-zookeeper-3.6.3-bin\\datadir
```

*   修改zookeeper的环境设置zkEnv.cmd

    在zookeeper的根路径下，有个bin目录的文件夹，下面有个脚本文件叫做zkEnv.cmd，修改zkEnv.cmd，添加两行配置

    ```
    set JAVA=D:\soft_ware_install\jdk8u144\jdk\bin\javaset JAVA_HOME=D:\soft_ware_install\jdk8u144\jdk
    ```

    <img src="file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644300038563.png?lastModify=1647874003" alt="1644300038563" data-size="original">
*   启动zookeeper单节点服务

    进入zookeeper的家目录的bin文件夹下面，然后在该路径下安装shift键，点击鼠标右键，打开Powershell窗口来启动zookeeper

    启动命令

    ```
    PS D:\soft_ware_install\apache-zookeeper-3.6.3-bin\bin> .\zkServer.cmd
    ```

![1644300144521](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644300144521.png?lastModify=1647874003)

![1644300191813](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644300191813.png?lastModify=1647874003)

### 9、windows下解压scala2.13的压缩包版本

在kafka3.x当中，需要使用scala2.12或者2.13的版本，我们这里选择使用2.13的版本来为kafka的源码进行配置scala，下载scala2.13的zip压缩包，下载地址如下：

[https://downloads.lightbend.com/scala/2.13.8/scala-2.13.8.zip](https://downloads.lightbend.com/scala/2.13.8/scala-2.13.8.zip)

下载之后，直接解压到一个没有中文没有空格的路径下即可，不需要配置scala环境变量

![1644300889885](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644300889885.png?lastModify=1647874003)

### 10、为idea当中的kafa源码添加scala的sdk

scala安装成功之后，就需要为kafka源码添加scala的sdk即可

File -> Project Structure -> Platform Settings -> Global Libraries -> 把scala目录配置进去。

![1644301015931](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644301015931.png?lastModify=1647874003)

![1644301120587](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644301120587.png?lastModify=1647874003)

选中所有的module

![1644301147463](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644301147463.png?lastModify=1647874003)

### 11、修改kafka的配置文件server.properties

修改idea当中源码路径下有个config文件夹，文件夹下面有个server.properties的配置文件，我们需要修改server.properties这个配置文件

```
log.dirs=D:\\kafka_logs
```

![1644301287386](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644301287386.png?lastModify=1647874003)

### 12、启动kafka的服务端，生产者以及消费者

前面修改好了kafka的配置文件之后，接下来就可以启动kafka的服务端，启动成功之后，就可以启动kafka的生产者，往kafka服务器当中发送消息，然后启动消费者，消费kafka当中的消息

启动kafka的服务端

在idea当中的kafka源码里，有个core的模块，这个就是kafka的核心源码启动逻辑都在这里

启动路径如下：kafka-3.1.0-src -> core -> src -> main -> scala -> kafka -> Kafka

第一次启动不需要加任何参数，启动会报错，然后第二次启动，编辑main方法的参数，带上启动参数即可

![1644301724267](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644301724267.png?lastModify=1647874003)

![1644301758085](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644301758085.png?lastModify=1647874003)

启动之后没有报错，证明kafka服务端启动成功

启动kafka的生产者端

kafka的服务端启动成功之后，接下来便可以启动kafka的生产者，向kafka当中生产数据了

在这里我们使用ConsoleProducer来向kafka集群当中生产数据即可

在kafka-3.1.0-src源码下，还是找到core核心模块，然后使用ConsoleProducer来生产数据

kafka-3.1.0-src -> core -> src -> main -> scala -> kafka -> tools -> ConsoleProducer

第一次启动会报错，报错之后，第二次启动时候添加参数

```
--topic zp --bootstrap-server localhost:9092 
```

![1644302193654](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644302193654.png?lastModify=1647874003)

![1644302213779](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644302213779.png?lastModify=1647874003)

启动成功之后，向kafka单节点当中发送数据

![1644302248133](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644302248133.png?lastModify=1647874003)

启动kafka的消费者来消费数据

kafka的生产者启动成功之后，接下来便可以启动kafka的消费者，向kafka当中生产数据了

在这里我们使用ConsoleConsumer来向kafka集群当中生产数据即可

在kafka-3.1.0-src源码下，还是找到core核心模块，然后使用ConsoleConsumer来生产数据

kafka-3.1.0-src -> core -> src -> main -> scala -> kafka -> tools -> ConsoleConsumer

第一次启动会报错，报错之后，第二次启动时候添加参数

```
--topic zp --bootstrap-server localhost:9092 --from-beginning
```

![1644302326338](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644302326338.png?lastModify=1647874003)

![1644302359006](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644302359006.png?lastModify=1647874003)

启动成功之后，查看控制台接收到的kafka消息

![1644302380723](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/1644302380723.png?lastModify=1647874003)

## 6、生产者源码剖析

接下来我们从生产者源码来开始源码阅读之旅

### 1、源码阅读的两种方式

*   1、场景驱动的方式

    ```
    比如我们分析生产者代码的时候，我们就先写一个生产者的代码，根据代码一步一步去分析。KafkaProducer
    ```
*   2、画图分析

    ```
    看源码的时候边分析，边画图
    ```

> 看到写得比较好的源码的时候，我们也去分析一下，里面的一些架构的技巧，代码编程技巧。分享优秀的架构和编程设计。

### 2、消息发送设计思想

如果让你来设计一个消息的生产者，你应该考虑哪些方面，能够让消息发送成功，且效率高，且占用网络带宽更少，且效率更高？

### 3、生产者源码之Producer核心流程

生产者主要作用就是将生产的数据发送到kafka的broker服务器当中去进行保存，这里面涉及到网络的操作，本地缓冲区的操作，以及消息的压缩等

![kafka producer流程](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/kafka%20producer%E6%B5%81%E7%A8%8B.png?lastModify=1647874003)

* 1、ProducerInterceptors是一个拦截器，对发送的数据进行拦截处理
* 2、Serializer 对消息的key和value进行序列化
* 3、通过使用分区器作用在每一条消息上，实现数据分发进行入到topic不同的分区中
* 4、RecordAccumulator缓存消息，实现批量发送
* 5、Sender从RecordAccumulator获取消息
* 6、构建ClientRequest对象
* 7、将ClientRequest交给 NetWorkClient准备发送
* 8、NetWorkClient 将请求放入到KafkaChannel的缓存
* 9、发送请求到kafka集群
* 10、Sender线程接受服务端发送的响应
* 11、执行绑定的回调函数

### 4、生产者源码之Producer初始化构造方法

在kafka的源码包下有一个examples的模块，这个模块下都是各种示例代码，在该模块下的src/main/java路径下的kafka.examples包下创建一个ProduceMessages生产者代码，用于生产数据，简单代码如下：

```
package kafka.examples;import org.apache.kafka.clients.producer.KafkaProducer;import org.apache.kafka.clients.producer.ProducerConfig;import org.apache.kafka.clients.producer.ProducerRecord;import org.apache.kafka.common.serialization.StringSerializer;import java.util.Properties;public class ProduceMessages {    public static void main(String[] args) throws InterruptedException {        int messageNum = 0;        String messageStr = "";        boolean isAsync = true;        Properties props = new Properties();        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,"localhost:9092");        props.put(ProducerConfig.CLIENT_ID_CONFIG, "DemoProducer");        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());        KafkaProducer<String, String> producer = new KafkaProducer<String,String>(props);        int i =0;        while(true){            long startTime = System.currentTimeMillis();            messageNum ++;            messageStr = messageStr+ messageNum;            ProducerRecord<String, String> record = new ProducerRecord<>("zp", messageNum + "", "world"+ i);            if(isAsync){                //发送数据                producer.send(record,new DemoCallBack(startTime,messageNum,messageStr));                Thread.sleep(1000);            }else{                try{                    //发送数据                    producer.send(record);                    Thread.sleep(1000);                }catch (InterruptedException exception){                    exception.printStackTrace();                }            }        }    }}
```

#### 初始化KafkaProducer对象

```
//todo:初始化KafkaProducer对象producer = new KafkaProducer<>(props);
```

#### KafkaProducer对象重要参数初始化

```
 /**     * kafkaProducer初始化都是在这个方法里面实现的，这个方法有很多参数，也有很多重要的类     * @param config 我们传入的一些必要参数，以及各种默认参数都配置在ProducerConfig当中     * @param keySerializer  序列化     * @param valueSerializer  序列化     * @param metadata  元数据信息     * @param kafkaClient  客户端对象     * @param interceptors  拦截器     * @param time  数据发送的默认系统时间     */    @SuppressWarnings("unchecked")    KafkaProducer(ProducerConfig config,                  Serializer<K> keySerializer,                  Serializer<V> valueSerializer,                  ProducerMetadata metadata,                  KafkaClient kafkaClient,                  ProducerInterceptors<K, V> interceptors,                  Time time) {        try {            //通过传入的参数给全局变量producerConfig进行赋值            this.producerConfig = config;            //设置当前系统时间            this.time = time;            //在kafka新版本当中，生产者可以配置事务id，用于保证数据生产的事务性，如果没有提供事务id，那么生产者仅限于幂等性来保证数据的exactly  once语义            String transactionalId = config.getString(ProducerConfig.TRANSACTIONAL_ID_CONFIG);            this.clientId = config.getString(ProducerConfig.CLIENT_ID_CONFIG);            //LogContext主要是用于在没有添加groupID的时候，将消息进一步包装，每个消费者添加一个唯一标识            LogContext logContext;            if (transactionalId == null)                logContext = new LogContext(String.format("[Producer clientId=%s] ", clientId));            else                logContext = new LogContext(String.format("[Producer clientId=%s, transactionalId=%s] ", clientId, transactionalId));            log = logContext.logger(KafkaProducer.class);            log.trace("Starting the Kafka producer");            Map<String, String> metricTags = Collections.singletonMap("client-id", clientId);            //********************集群监控信息，一般在阅读源码的时候，都会忽略掉监控信息*******************************//            MetricConfig metricConfig = new MetricConfig().samples(config.getInt(ProducerConfig.METRICS_NUM_SAMPLES_CONFIG))                    .timeWindow(config.getLong(ProducerConfig.METRICS_SAMPLE_WINDOW_MS_CONFIG), TimeUnit.MILLISECONDS)                    .recordLevel(Sensor.RecordingLevel.forName(config.getString(ProducerConfig.METRICS_RECORDING_LEVEL_CONFIG)))                    .tags(metricTags);            List<MetricsReporter> reporters = config.getConfiguredInstances(ProducerConfig.METRIC_REPORTER_CLASSES_CONFIG,                    MetricsReporter.class,                    Collections.singletonMap(ProducerConfig.CLIENT_ID_CONFIG, clientId));            JmxReporter jmxReporter = new JmxReporter();            jmxReporter.configure(config.originals(Collections.singletonMap(ProducerConfig.CLIENT_ID_CONFIG, clientId)));            reporters.add(jmxReporter);            MetricsContext metricsContext = new KafkaMetricsContext(JMX_PREFIX,                    config.originalsWithPrefix(CommonClientConfigs.METRICS_CONTEXT_PREFIX));            this.metrics = new Metrics(metricConfig, reporters, time, metricsContext);            this.producerMetrics = new KafkaProducerMetrics(metrics);            //********************集群监控信息，一般在阅读源码的时候，都会忽略掉监控信息*******************************//            //开始设置消息的分区器，分区器决定了消息使用哪种方式发送到指定topic的哪个分区里面去            this.partitioner = config.getConfiguredInstance(                    ProducerConfig.PARTITIONER_CLASS_CONFIG,                    Partitioner.class,                    Collections.singletonMap(ProducerConfig.CLIENT_ID_CONFIG, clientId));            //消息发送失败时候的重试时间间隔            long retryBackoffMs = config.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG);            //***************************设置key和value的序列化器，也支持自定义序列化器*************************//            if (keySerializer == null) {                this.keySerializer = config.getConfiguredInstance(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,                                                                                         Serializer.class);                this.keySerializer.configure(config.originals(Collections.singletonMap(ProducerConfig.CLIENT_ID_CONFIG, clientId)), true);            } else {                config.ignore(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG);                this.keySerializer = keySerializer;            }            if (valueSerializer == null) {                this.valueSerializer = config.getConfiguredInstance(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,                                                                                           Serializer.class);                this.valueSerializer.configure(config.originals(Collections.singletonMap(ProducerConfig.CLIENT_ID_CONFIG, clientId)), false);            } else {                config.ignore(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG);                this.valueSerializer = valueSerializer;            }            //***************************设置key和value的序列化器，也支持自定义序列化器*************************//            //设置拦截器，拦截器给准备发送出去的消息先添加了一道拦截，可以实现对消息的过滤            List<ProducerInterceptor<K, V>> interceptorList = (List) config.getConfiguredInstances(                    ProducerConfig.INTERCEPTOR_CLASSES_CONFIG,                    ProducerInterceptor.class,                    Collections.singletonMap(ProducerConfig.CLIENT_ID_CONFIG, clientId));            if (interceptors != null)                this.interceptors = interceptors;            else                this.interceptors = new ProducerInterceptors<>(interceptorList);            //配置进群资源监听器            ClusterResourceListeners clusterResourceListeners = configureClusterResourceListeners(keySerializer,                    valueSerializer, interceptorList, reporters);            //todo: max.request.size 生产者往服务端发送消息的时候，规定一条消息最大多大？            //如果你超过了这个规定消息的大小，你的消息就不能发送过去。            //默认是1M，这个值偏小，在生产环境中，我们需要修改这个值。            //经验值是10M。但是大家也可以根据自己公司的情况来。            this.maxRequestSize = config.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG);            //todo:  指的是缓存大小            //默认值是32M，这里设置的是内存缓冲区的大小默认为32M，这个内存大小主要是用于发送数据的缓冲区大小            //生产者发送数据出去之前，都需要先对数据在本地进行缓冲，这里就是本地缓冲区的大小，默认是32M，            //如果发送速度比较快，缓冲区到服务器broker比较慢，那么就会把这个缓冲区填满，缓冲区填满了生产者线程就会阻塞            //阻塞时间由max.block.ms 这个配置选项决定，默认是60s，实际工作当中，这个缓冲区大小，一般都跟producer端配置的内存大小保持一致            this.totalMemorySize = config.getLong(ProducerConfig.BUFFER_MEMORY_CONFIG);            //设置数据的压缩方式，很明显数据发送到服务端之前，可以先在本地缓冲区进行压缩一下，减少数据体积，减少网络占用            //压缩方式的值可以是  node,gzip,snappy,lz4,zstd等各种方式，压缩是对发送给broker的一批数据进行压缩，            //用时间换空间的思想，压缩更加消耗CPU，消耗压缩时间，但是发送数据的吞吐量更大了。计算机里面到处都是各种时间换空间，空间换时间的思想。            this.compressionType = CompressionType.forName(config.getString(ProducerConfig.COMPRESSION_TYPE_CONFIG));            //这个阻塞时间，决定了生产者在发送数据之前，这个超时参数限制了等待元数据获取和缓冲区分配的总时间，用户提供的序列化程序和分区程序中的阻塞不算在这个时间里面            //说白了，我的数据发送到哪一个分区需要先拉取元数据信息，然后通过元数据当中的分区数，以及分区方法决定数据进入哪一个分区            //如果元数据拉取超时，本地缓冲区满了阻塞，都会等待这么长时间            this.maxBlockTimeMs = config.getLong(ProducerConfig.MAX_BLOCK_MS_CONFIG);            int deliveryTimeoutMs = configureDeliveryTimeout(config, log);            this.apiVersions = new ApiVersions();            this.transactionManager = configureTransactionState(config, logContext);            //这里初始化了一个核心组件叫做RecordAccumulator            this.accumulator = new RecordAccumulator(logContext,                    config.getInt(ProducerConfig.BATCH_SIZE_CONFIG),                    this.compressionType,                    lingerMs(config),                    retryBackoffMs,                    deliveryTimeoutMs,                    metrics,                    PRODUCER_METRIC_GROUP_NAME,                    time,                    apiVersions,                    transactionManager,                    new BufferPool(this.totalMemorySize, config.getInt(ProducerConfig.BATCH_SIZE_CONFIG), metrics, time, PRODUCER_METRIC_GROUP_NAME));            //构建kafka集群通信的通信地址            List<InetSocketAddress> addresses = ClientUtils.parseAndValidateAddresses(                    config.getList(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG),                    config.getString(ProducerConfig.CLIENT_DNS_LOOKUP_CONFIG));            if (metadata != null) {                this.metadata = metadata;            } else {                this.metadata = new ProducerMetadata(retryBackoffMs,                        config.getLong(ProducerConfig.METADATA_MAX_AGE_CONFIG),                        config.getLong(ProducerConfig.METADATA_MAX_IDLE_CONFIG),                        logContext,                        clusterResourceListeners,                        Time.SYSTEM);                this.metadata.bootstrap(addresses);            }            this.errors = this.metrics.sensor("errors");            //这里构建了一个核心组件Sender线程            this.sender = newSender(logContext, kafkaClient, this.metadata);            String ioThreadName = NETWORK_THREAD_PREFIX + " | " + clientId;            //构建了一个线程，然后将sender对象传入进去了            this.ioThread = new KafkaThread(ioThreadName, this.sender, true);            //线程启动了            this.ioThread.start();            config.logUnused();            AppInfoParser.registerAppInfo(JMX_PREFIX, clientId, metrics, time.milliseconds());            log.debug("Kafka producer started");        } catch (Throwable t) {            // call close methods if internal objects are already constructed this is to prevent resource leak. see KAFKA-2121            close(Duration.ofMillis(0), true);            // now propagate the exception            throw new KafkaException("Failed to construct kafka producer", t);        }    }
```

#### 核心组件之RecordAccumulator

* 初始化组件RecordAccumulator对象，用于缓存数据

```
            //这里初始化了一个核心组件叫做RecordAccumulator，这里主要是用这个对象来对消息进行本地缓存，说白了就是为了攒一批消息一起发送出去            this.accumulator = new RecordAccumulator(logContext,                    config.getInt(ProducerConfig.BATCH_SIZE_CONFIG),                    this.compressionType,                    lingerMs(config),                    retryBackoffMs,                    deliveryTimeoutMs,                    metrics,                    PRODUCER_METRIC_GROUP_NAME,                    time,                    apiVersions,                    transactionManager,                    new BufferPool(this.totalMemorySize, config.getInt(ProducerConfig.BATCH_SIZE_CONFIG), metrics, time, PRODUCER_METRIC_GROUP_NAME));
```

#### 核心组件之Sender线程

* Sender是一个线程， 继承自Runnable， KafkaThread也是一个线程，继承自Thread。KafkaThread主要是将Sender线程设置为一个后台线程。 这种编程思想将业务代码和线程代码隔离开，代码逻辑更清晰一些。

```
   //这里构建了一个核心组件Sender线程            this.sender = newSender(logContext, kafkaClient, this.metadata);            String ioThreadName = NETWORK_THREAD_PREFIX + " | " + clientId;            //构建了一个线程，然后将sender对象传入进去了            this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
```

#### 核心组件之NetworkClient

* 在上面看到调用了一个newSender的方法，其实就是在newSender方法当中初始化了Sender对象，一起来看Sender对象初始化过程
* 上面在调用`newSender`方法进行初始化`Sender`线程中，内部初始化了网络组件`NetworkClient`

```
   // visible for testing    Sender newSender(LogContext logContext, KafkaClient kafkaClient, ProducerMetadata metadata) {        // maxInflightRequests 生产者发送消息给服务端，具体到服务端某个leader Partition去接收消息，但是leader Partition有可能宕机下线了，那么生产者怎么办？？        //这里有两种选项，1、发送不过去了就不发了，客户端阻塞，不给这个partition发送消息了        //2、继续发送，如果发送maxInflightRequests  次没响应就不发了，例如maxInflightRequests取值为5，那么就是发送5次过去没响应就不发了，那么这里就存在消息乱序的问题了        // A给B发送消息，由于网络原因，B没有收到，A继续发送了五次，等到B的网络好了，那么五次消息当中消息还会按照顺序发送的方式到达B嘛？？？如果不能顺序到达，那么是不是就出现了消息乱序的问题了        //大家都说kafka是分区内部有序的，源码当中maxInflightRequests默认取值是5，说明分区内部也有可能乱序        int maxInflightRequests = configureInflightRequests(producerConfig);        //发送给服务端请求超时时间        int requestTimeoutMs = producerConfig.getInt(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG);        ChannelBuilder channelBuilder = ClientUtils.createChannelBuilder(producerConfig, time, logContext);        ProducerMetrics metricsRegistry = new ProducerMetrics(this.metrics);        Sensor throttleTimeSensor = Sender.throttleTimeSensor(metricsRegistry.senderMetrics);        //todo: 初始化了一个重要的网络管理的组件        /**         * connections.max.idle.ms: 默认值是9分钟，一个网络连接最多空闲多久，超过这个空闲时间，就关闭这个网络连接。         * 可以设置为-1 ，不回收连接，减少频繁的创建和销毁连接         *         * todo: max.in.flight.requests.per.connection：默认是5         *  发送数据的时候，其实是有多个网络连接。每个网络连接可以忍受 producer端发送给broker消息后，消息没有响应的个数。         *  因为kafka有重试机制，所以有可能会造成数据乱序，如果想要保证有序，这个值要把设置为1.         *         * reconnect.backoff.ms：socket尝试重新连接指定主机的时间间隔         * send.buffer.bytes：   socket发送数据的缓冲区的大小，默认值是128K         * receive.buffer.bytes：socket接受数据的缓冲区的大小，默认值是32K         * 注意：这里为什么发送缓冲区比接收缓冲区大，因为发送缓冲区设置大一点，可以将多批次消息合并成为一个包，进行沾包处理，提高发送效率         */        KafkaClient client = kafkaClient != null ? kafkaClient : new NetworkClient(                new Selector(producerConfig.getLong(ProducerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG),                        this.metrics, time, "producer", channelBuilder, logContext),                metadata,                clientId,                maxInflightRequests,                producerConfig.getLong(ProducerConfig.RECONNECT_BACKOFF_MS_CONFIG),                producerConfig.getLong(ProducerConfig.RECONNECT_BACKOFF_MAX_MS_CONFIG),                producerConfig.getInt(ProducerConfig.SEND_BUFFER_CONFIG),                producerConfig.getInt(ProducerConfig.RECEIVE_BUFFER_CONFIG),                requestTimeoutMs,                producerConfig.getLong(ProducerConfig.SOCKET_CONNECTION_SETUP_TIMEOUT_MS_CONFIG),                producerConfig.getLong(ProducerConfig.SOCKET_CONNECTION_SETUP_TIMEOUT_MAX_MS_CONFIG),                time,                true,                apiVersions,                throttleTimeSensor,                logContext);        //获取acks的参数，用于消息发送成功确认        /**         *  retries: 重试的次数，默认为0，表示不重试         *  acks:         *     0:         *      producer发送数据到broker后，就完了，没有返回值，不管写成功还是写失败都不管了。         *       如果是这种请求数据丢失的概率是非常之高，一般不建议设置为0         *         *   1：         *      producer发送数据到broker后，数据成功写入leader partition以后返回响应。         *      数据 -》 broker（leader partition）         *      也有丢失数据的风险         *         *  -1或者all：         *       producer发送数据到broker后，数据要写入到leader partition里面，并且数据同步到所有的         *       follower partition里面以后，才返回响应。         */        short acks = configureAcks(producerConfig, log);        return new Sender(logContext,                client,                metadata,                this.accumulator,                maxInflightRequests == 1,                producerConfig.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG),                acks,                producerConfig.getInt(ProducerConfig.RETRIES_CONFIG),                metricsRegistry.senderMetrics,                time,                requestTimeoutMs,                producerConfig.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG),                this.transactionManager,                apiVersions);    }
```

#### 生产者源码之Producer端元数据管理

前面已经看到了核心网络组件NetworkClient以及Sender线程和RecordAccumulator的初始化，接下来一起来看一下生产者如何获取到的元数据信息，在KafkaProducer初始化方法当中，有一段代码如下：

```
  //第一次启动生产者，metadata肯定为空，因为我们没有传入过元数据这个对象作为生产者的参数，进入else分支            if (metadata != null) {                this.metadata = metadata;            } else {                //第一次就是在这里开始组件元数据信息对象                this.metadata = new ProducerMetadata(retryBackoffMs,                        config.getLong(ProducerConfig.METADATA_MAX_AGE_CONFIG),                        config.getLong(ProducerConfig.METADATA_MAX_IDLE_CONFIG),                        logContext,                        clusterResourceListeners,                        Time.SYSTEM);                //元数据信息对象调用了bootstrap方法，进入查看bootstrap方法                this.metadata.bootstrap(addresses);            }
```

![image-20220208185643958](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220208185643958.png?lastModify=1647874003)

查看Metadata当中的bootstrap方法定义如下：

```
  public synchronized void bootstrap(List<InetSocketAddress> addresses) {        //第一次全量拉取所有的元数据信息，如果元数据信息都是保存在zookeeper当中，那么就需要从zk当中去获取元数据，有可能比较慢        //在kafka3.x当中，使用kraft模式，可以将元数据保存在kafka的topic当中        this.needFullUpdate = true;        //获取一次元数据，就将元数据更新的数值加1        this.updateVersion += 1;        //调用了MeatdataCache的bootstrap方法，查看bootstrap方法        this.cache = MetadataCache.bootstrap(addresses);    }
```

![image-20220208185938589](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220208185938589.png?lastModify=1647874003)

查看MetadataCache这个类的bootstrap方法的具体实现如下

```
    static MetadataCache bootstrap(List<InetSocketAddress> addresses) {        Map<Integer, Node> nodes = new HashMap<>();        int nodeId = -1;        for (InetSocketAddress address : addresses) {            nodes.put(nodeId, new Node(nodeId, address.getHostString(), address.getPort()));            nodeId--;        }        //其实就是new了一个MetaCache的对象出来，并且给了一堆的空集合，最后一个参数调用了Cluster的bootstrap这个方法，        //查看Cluster这个对象的bootstrap方法        return new MetadataCache(null, nodes, Collections.emptyList(),                Collections.emptySet(), Collections.emptySet(), Collections.emptySet(),                null, Collections.emptyMap(), Cluster.bootstrap(addresses));    }
```

![image-20220208190119763](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220208190119763.png?lastModify=1647874003)

查看Cluster这个类当中的bootstrap方法实现如下

```
 /**     * Create a "bootstrap" cluster using the given list of host/ports     * @param addresses The addresses     * @return A cluster for these hosts/ports     */    public static Cluster bootstrap(List<InetSocketAddress> addresses) {        List<Node> nodes = new ArrayList<>();        int nodeId = -1;        for (InetSocketAddress address : addresses)            nodes.add(new Node(nodeId--, address.getHostString(), address.getPort()));        //最后的最后就是创建了一个Cluster对象        return new Cluster(null, true, nodes, new ArrayList<>(0),            Collections.emptySet(), Collections.emptySet(), Collections.emptySet(), null, Collections.emptyMap());    }
```

![image-20220208190306903](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220208190306903.png?lastModify=1647874003)

查看Cluster对象的构造方法，里面初始化了很多的对象，初始化方法定义如下

```
 private Cluster(String clusterId,                    boolean isBootstrapConfigured,                    Collection<Node> nodes,                    Collection<PartitionInfo> partitions,                    Set<String> unauthorizedTopics,                    Set<String> invalidTopics,                    Set<String> internalTopics,                    Node controller,                    Map<String, Uuid> topicIds) {        this.isBootstrapConfigured = isBootstrapConfigured;        this.clusterResource = new ClusterResource(clusterId);        // make a randomized, unmodifiable copy of the nodes        List<Node> copy = new ArrayList<>(nodes);        Collections.shuffle(copy);        this.nodes = Collections.unmodifiableList(copy);        // Index the nodes for quick lookup        Map<Integer, Node> tmpNodesById = new HashMap<>();        Map<Integer, List<PartitionInfo>> tmpPartitionsByNode = new HashMap<>(nodes.size());        for (Node node : nodes) {            tmpNodesById.put(node.id(), node);            // Populate the map here to make it easy to add the partitions per node efficiently when iterating over            // the partitions            tmpPartitionsByNode.put(node.id(), new ArrayList<>());        }        this.nodesById = Collections.unmodifiableMap(tmpNodesById);        // index the partition infos by topic, topic+partition, and node        // note that this code is performance sensitive if there are a large number of partitions so we are careful        // to avoid unnecessary work        Map<TopicPartition, PartitionInfo> tmpPartitionsByTopicPartition = new HashMap<>(partitions.size());        Map<String, List<PartitionInfo>> tmpPartitionsByTopic = new HashMap<>();        for (PartitionInfo p : partitions) {            tmpPartitionsByTopicPartition.put(new TopicPartition(p.topic(), p.partition()), p);            tmpPartitionsByTopic.computeIfAbsent(p.topic(), topic -> new ArrayList<>()).add(p);            // The leader may not be known            if (p.leader() == null || p.leader().isEmpty())                continue;            // If it is known, its node information should be available            List<PartitionInfo> partitionsForNode = Objects.requireNonNull(tmpPartitionsByNode.get(p.leader().id()));            partitionsForNode.add(p);        }        // Update the values of `tmpPartitionsByNode` to contain unmodifiable lists        for (Map.Entry<Integer, List<PartitionInfo>> entry : tmpPartitionsByNode.entrySet()) {            tmpPartitionsByNode.put(entry.getKey(), Collections.unmodifiableList(entry.getValue()));        }        // Populate `tmpAvailablePartitionsByTopic` and update the values of `tmpPartitionsByTopic` to contain        // unmodifiable lists        Map<String, List<PartitionInfo>> tmpAvailablePartitionsByTopic = new HashMap<>(tmpPartitionsByTopic.size());        for (Map.Entry<String, List<PartitionInfo>> entry : tmpPartitionsByTopic.entrySet()) {            String topic = entry.getKey();            List<PartitionInfo> partitionsForTopic = Collections.unmodifiableList(entry.getValue());            tmpPartitionsByTopic.put(topic, partitionsForTopic);            // Optimise for the common case where all partitions are available            boolean foundUnavailablePartition = partitionsForTopic.stream().anyMatch(p -> p.leader() == null);            List<PartitionInfo> availablePartitionsForTopic;            if (foundUnavailablePartition) {                availablePartitionsForTopic = new ArrayList<>(partitionsForTopic.size());                for (PartitionInfo p : partitionsForTopic) {                    if (p.leader() != null)                        availablePartitionsForTopic.add(p);                }                availablePartitionsForTopic = Collections.unmodifiableList(availablePartitionsForTopic);            } else {                availablePartitionsForTopic = partitionsForTopic;            }            tmpAvailablePartitionsByTopic.put(topic, availablePartitionsForTopic);        }        this.partitionsByTopicPartition = Collections.unmodifiableMap(tmpPartitionsByTopicPartition);        this.partitionsByTopic = Collections.unmodifiableMap(tmpPartitionsByTopic);        this.availablePartitionsByTopic = Collections.unmodifiableMap(tmpAvailablePartitionsByTopic);        this.partitionsByNode = Collections.unmodifiableMap(tmpPartitionsByNode);        this.topicIds = Collections.unmodifiableMap(topicIds);        Map<Uuid, String> tmpTopicNames = new HashMap<>();        topicIds.forEach((key, value) -> tmpTopicNames.put(value, key));        this.topicNames = Collections.unmodifiableMap(tmpTopicNames);        this.unauthorizedTopics = Collections.unmodifiableSet(unauthorizedTopics);        this.invalidTopics = Collections.unmodifiableSet(invalidTopics);        this.internalTopics = Collections.unmodifiableSet(internalTopics);        this.controller = controller;    }
```

看到最后我们会发现，其实第一次来的时候，并没有获取到任何的元数据信息，如果没有获取到元数据信息，那么需要发送消息，在哪里获取的元数据信息呢？

### 5、生产者源码之Producer端元数据管理

生产者需要获取元数据信息，都是将元数据信息封装到javaBean对象当中去了，这里主要涉及到几个核心的javaBean对象

#### Metadata类的定义

```
/** * A class encapsulating some of the logic around metadata. * <p> * This class is shared by the client thread (for partitioning) and the background sender thread. * * Metadata is maintained for only a subset of topics, which can be added to over time. When we request metadata for a * topic we don't have any metadata for it will trigger a metadata update. * <p> * If topic expiry is enabled for the metadata, any topic that has not been used within the expiry interval * is removed from the metadata refresh set after an update. Consumers disable topic expiry since they explicitly * manage topics while producers rely on topic expiry to limit the refresh set. */public class Metadata implements Closeable {    private final Logger log;    private final long refreshBackoffMs;    private final long metadataExpireMs;    private int updateVersion;  // bumped on every metadata response    private int requestVersion; // bumped on every new topic addition    private long lastRefreshMs;    private long lastSuccessfulRefreshMs;    private KafkaException fatalException;    private Set<String> invalidTopics;    private Set<String> unauthorizedTopics;    private MetadataCache cache = MetadataCache.empty();    private boolean needFullUpdate;    private boolean needPartialUpdate;    private final ClusterResourceListeners clusterResourceListeners;    private boolean isClosed;    private final Map<TopicPartition, Integer> lastSeenLeaderEpochs;    /**     * Create a new Metadata instance     *     * @param refreshBackoffMs         The minimum amount of time that must expire between metadata refreshes to avoid busy     *                                 polling     * @param metadataExpireMs         The maximum amount of time that metadata can be retained without refresh     * @param logContext               Log context corresponding to the containing client     * @param clusterResourceListeners List of ClusterResourceListeners which will receive metadata updates.     */    public Metadata(long refreshBackoffMs,                    long metadataExpireMs,                    LogContext logContext,                    ClusterResourceListeners clusterResourceListeners) {        this.log = logContext.logger(Metadata.class);        this.refreshBackoffMs = refreshBackoffMs;        this.metadataExpireMs = metadataExpireMs;        this.lastRefreshMs = 0L;        this.lastSuccessfulRefreshMs = 0L;        this.requestVersion = 0;        this.updateVersion = 0;        this.needFullUpdate = false;        this.needPartialUpdate = false;        this.clusterResourceListeners = clusterResourceListeners;        this.isClosed = false;        this.lastSeenLeaderEpochs = new HashMap<>();        this.invalidTopics = Collections.emptySet();        this.unauthorizedTopics = Collections.emptySet();    }            }
```

#### MetadataCache类的定义

MetadataCache类主要用于保存获取到服务器端的元数据，其定义如下

```
public class MetadataCache {    private final String clusterId;    private final Map<Integer, Node> nodes;    private final Set<String> unauthorizedTopics;    private final Set<String> invalidTopics;    private final Set<String> internalTopics;    private final Node controller;    private final Map<TopicPartition, PartitionMetadata> metadataByPartition;    private final Map<String, Uuid> topicIds;    private Cluster clusterInstance; }
```

#### Cluster类的定义

查看Cluster类的定义，申明了很多的全局变量，都是一些常用的集合

关于 topic 的详细信息（leader 所在节点、replica 所在节点、isr 列表）都是在 `Cluster` 实例中保存的。

```
/** * An immutable representation of a subset of the nodes, topics, and partitions in the Kafka cluster. */public final class Cluster {    private final boolean isBootstrapConfigured;    private final List<Node> nodes;    private final Set<String> unauthorizedTopics;    private final Set<String> invalidTopics;    private final Set<String> internalTopics;    private final Node controller;    private final Map<TopicPartition, PartitionInfo> partitionsByTopicPartition;    private final Map<String, List<PartitionInfo>> partitionsByTopic;    private final Map<String, List<PartitionInfo>> availablePartitionsByTopic;    private final Map<Integer, List<PartitionInfo>> partitionsByNode;    private final Map<Integer, Node> nodesById;    private final ClusterResource clusterResource;    private final Map<String, Uuid> topicIds;    private final Map<Uuid, String> topicNames;}
```

### 6、生产者源码之Producer拉取元数据信息

前面我们基本上把初始化KafkaProducer的流程给大家分析了，接下来按照场景驱动的方式，开始发送数据，实际生产环境中都是通过==异步==的方式发送消息，性能比较高。下面我们就来分析异步发送的消息的代码业务逻辑。

核心代码就在producer.send()这个方法的具体实现，进入到producer.send这个方法当中

```
public class ProduceMessages {    public static void main(String[] args) {        Properties props = new Properties();        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,"localhost:9092");        props.put(ProducerConfig.CLIENT_ID_CONFIG, "DemoProducer");        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());        KafkaProducer<String, String> producer = new KafkaProducer<String,String>(props);        ProducerRecord<String, String> record = new ProducerRecord<>("zp", "hello", "world");        //发送数据        producer.send(record);        producer.close();    }}
```

![image-20220208191422079](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220208191422079.png?lastModify=1647874003)

查看KafkaProducer当中的send方法具体实现如下

```
    /**     * Asynchronously send a record to a topic. Equivalent to <code>send(record, null)</code>.     * See {@link #send(ProducerRecord, Callback)} for details.     */    @Override    public Future<RecordMetadata> send(ProducerRecord<K, V> record) {        return send(record, null);    }
```

![image-20220208191453701](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220208191453701.png?lastModify=1647874003)

继续查看send方法具体实现如下：

```
 /**        * 通过异步的方式来进行消息的方法，并且调用回调函数来通知生产者消息是否发送成功 ，这个方法一旦调用，消息写入到本地缓冲区，就会立马返回，这样可以提高消息发送的效率     * 可以并行的一起发送消息，都写入到本地缓冲区，而且不需要等待服务器的响应  。最终执行完了，这个方法的返回结果是这个数据写入到kafka服务器的元数据信息，例如这个数据的分区号，这个数据的offset对应的值等     * 发送数据还可以带上数据的时间戳，如果我们给数据指定了CREATE_TIME这个属性，那么就使用我们自己指定的时间戳的值，如果没有指定，那么就使用数据的发送出来的时间，如果启用了LOG_APPEND_TIME,     * 那么就使用kafka集群的服务器时间作为每条数据的时间戳     */    @Override    public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {        // intercept the record, which can be potentially modified; this method does not throw exceptions        ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);        return doSend(interceptedRecord, callback);    }
```

![image-20220208192948145](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220208192948145.png?lastModify=1647874003)

#### 6.1、doSend方法的具体实现

接下来我们可以一起来看到doSend方法当中有大量逻辑实现代码如下

```
 /**     * Implementation of asynchronously send a record to a topic.     */    private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {        TopicPartition tp = null;        try {            throwIfProducerClosed();            // first make sure the metadata for the topic is available            long nowMs = time.milliseconds();            ClusterAndWaitTime clusterAndWaitTime;            try {                //同步等待获取元数据信息                clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);            } catch (KafkaException e) {                if (metadata.isClosed())                    throw new KafkaException("Producer closed while send in progress", e);                throw e;            }            //等待获取元数据信息耗费的时间            nowMs += clusterAndWaitTime.waitedOnMetadataMs;            //剩余等待的时间            long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);            Cluster cluster = clusterAndWaitTime.cluster;            //对数据key进行序列化            byte[] serializedKey;            try {                serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());            } catch (ClassCastException cce) {                throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +                        " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +                        " specified in key.serializer", cce);            }            //对value进行序列化            byte[] serializedValue;            try {                serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());            } catch (ClassCastException cce) {                throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +                        " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +                        " specified in value.serializer", cce);            }            //数据进入到哪一个分区里面去            int partition = partition(record, serializedKey, serializedValue, cluster);            tp = new TopicPartition(record.topic(), partition);            //设置消息是否只读            setReadOnly(record.headers());            Header[] headers = record.headers().toArray();            int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),                    compressionType, serializedKey, serializedValue, headers);            ensureValidRecordSize(serializedSize);            long timestamp = record.timestamp() == null ? nowMs : record.timestamp();            if (log.isTraceEnabled()) {                log.trace("Attempting to append record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);            }            // producer callback will make sure to call both 'callback' and interceptor callback            //绑定回调函数            Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);            if (transactionManager != null && transactionManager.isTransactional()) {                transactionManager.failIfNotReadyForSend();            }            //将消息缓存到RecordAccumulator当中去            RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,                    serializedValue, headers, interceptCallback, remainingWaitMs, true, nowMs);            //判断消息是否应该封装到一个新的批次            if (result.abortForNewBatch) {                int prevPartition = partition;                partitioner.onNewBatch(record.topic(), cluster, prevPartition);                partition = partition(record, serializedKey, serializedValue, cluster);                tp = new TopicPartition(record.topic(), partition);                if (log.isTraceEnabled()) {                    log.trace("Retrying append due to new batch creation for topic {} partition {}. The old partition was {}", record.topic(), partition, prevPartition);                }                // producer callback will make sure to call both 'callback' and interceptor callback                interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);                result = accumulator.append(tp, timestamp, serializedKey,                    serializedValue, headers, interceptCallback, remainingWaitMs, false, nowMs);            }            if (transactionManager != null && transactionManager.isTransactional())                transactionManager.maybeAddPartitionToTransaction(tp);            //调用wakeup方法，唤醒sender线程            if (result.batchIsFull || result.newBatchCreated) {                log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);                this.sender.wakeup();            }            return result.future;            // handling exceptions and record the errors;            // for API exceptions return them in the future,            // for other exceptions throw directly        } catch (ApiException e) {            log.debug("Exception occurred during message send:", e);            if (callback != null)                callback.onCompletion(null, e);            this.errors.record();            this.interceptors.onSendError(record, tp, e);            return new FutureFailure(e);        } catch (InterruptedException e) {            this.errors.record();            this.interceptors.onSendError(record, tp, e);            throw new InterruptException(e);        } catch (KafkaException e) {            this.errors.record();            this.interceptors.onSendError(record, tp, e);            throw e;        } catch (Exception e) {            // we notify interceptor about all exceptions, since onSend is called before anything else in this method            this.interceptors.onSendError(record, tp, e);            throw e;        }    }
```

![image-20220208193800196](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220208193800196.png?lastModify=1647874003)

**6.1.1、对数据进行序列化和反序列化**

在doSend方法当中，对数据进行了序列化和反序列化操作

```
 //对数据key进行序列化            byte[] serializedKey;            try {                serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());            } catch (ClassCastException cce) {                throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +                        " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +                        " specified in key.serializer", cce);            }            //对value进行序列化            byte[] serializedValue;            try {                serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());            } catch (ClassCastException cce) {                throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +                        " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +                        " specified in value.serializer", cce);            }
```

**6.1.2、数据的分区操作**

在doSend方法当中，也对数据进入到哪一个分区当中，做了计算

```
  //数据进入到哪一个分区里面去  int partition = partition(record, serializedKey, serializedValue, cluster);
```

**6.1.3、根据元数据，封装数据topicAndPartition对象**

```
  tp = new TopicPartition(record.topic(), partition);
```

**6.1.4、判断消息发送大小是否超过最大值**

```
   int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),                    compressionType, serializedKey, serializedValue, headers);
```

**6.1.5、绑定回调函数**

```
   //绑定回调函数            Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);
```

**6.1.6、消息封装到RecordAccumulator当中去**

```
  //将消息缓存到RecordAccumulator当中去            RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,                    serializedValue, headers, interceptCallback, remainingWaitMs, true, nowMs);
```

**6.1.7、唤醒sender线程**

```
   //调用wakeup方法，唤醒sender线程            if (result.batchIsFull || result.newBatchCreated) {                log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);                this.sender.wakeup();            }
```

### 7、生产者拉取元数据过程深度剖析

同步等待获取元数据信息调用waitOnMetadata方法，查看waitOnMetaData方法的具体实现

在doSend方法当中，同步等待获取元数据信息的代码如下，这段代码就是专门去获取元数据信息的。

```
  ClusterAndWaitTime clusterAndWaitTime;            try {                //同步等待获取元数据信息                clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);            } catch (KafkaException e) {                if (metadata.isClosed())                    throw new KafkaException("Producer closed while send in progress", e);                throw e;            }
```

查看waitOnMetadata方法的具体实现如下

```
 /**     *  //todo: 连接上kafka集群，获取topic对应的元数据信息  (topic名称、分区编号，拉取元数据最大的超时时间）     * Wait for cluster metadata including partitions for the given topic to be available.     * @param topic The topic we want metadata for     * @param partition A specific partition expected to exist in metadata, or null if there's no preference     * @param nowMs The current time in ms     * @param maxWaitMs The maximum time in ms for waiting on the metadata     * @return The cluster containing topic metadata and the amount of time we waited in ms     * @throws TimeoutException if metadata could not be refreshed within {@code max.block.ms}     * @throws KafkaException for all Kafka-related exceptions, including the case where this method is called after producer close     */    private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long nowMs, long maxWaitMs) throws InterruptedException {        // add topic to metadata topic list if it is not there already and reset expiry        // add topic to metadata topic list if it is not there already and reset expiry        //todo: 我们使用的是场景驱动的方式，然后我们目前代码执行到的producer端初始化完成。        //我们知道这个cluster里面其实没有元数据，只是有我们写代码的时候设置address        Cluster cluster = metadata.fetch();        //todo: 该topic未授权        if (cluster.invalidTopics().contains(topic))            throw new InvalidTopicException(topic);        //todo: 把当前topic添加到元数据对象中        metadata.add(topic, nowMs);        //todo: 根据当前的topic从这个集群的cluster元数据信息里面查看分区的信息。        //因为我们目前是第一次执行这段代码，所以这儿肯定是没有对应的分区的信息的。        Integer partitionsCount = cluster.partitionCountForTopic(topic);        // Return cached metadata if we have it, and if the record's partition is either undefined        // or within the known partition range        //todo: 如果在元数据里面获取到了分区的信息        //我们用场景驱动的方式，我们知道如果是第一次代码进来这儿，代码是不会运行这儿。        if (partitionsCount != null && (partition == null || partition < partitionsCount))            return new ClusterAndWaitTime(cluster, 0);        //todo:如果代码执行到这儿，说明真的需要去服务端拉取元数据。        //记录当前时间        long remainingWaitMs = maxWaitMs;        long elapsed = 0;        // Issue metadata requests until we have metadata for the topic and the requested partition,        // or until maxWaitTimeMs is exceeded. This is necessary in case the metadata        // is stale and the number of partitions for this topic has increased in the meantime.        //todo: do while循环就相当于，不断在尝试等待获取元数据，如果有了集群的元数据之后，partitionsCount不为null，就表示有了集群元数据，        // 然后退出循环        do {            if (partition != null) {                log.trace("Requesting metadata update for partition {} of topic {}.", partition, topic);            } else {                log.trace("Requesting metadata update for topic {}.", topic);            }            metadata.add(topic, nowMs + elapsed);            //todo: 1)获取当前元数据的版本            //在Producer管理元数据时候，对于他来说元数据是有版本号的。            //每次成功更新元数据，都会递增这个版本号。            //todo: 2)把needUpdate 标识赋值为true            int version = metadata.requestUpdateForTopic(topic);            //todo: 唤醒sender线程，开始执行拉取元数据操作,拉取元数据的操作是由Sender线程完成的            //这里会涉及到java线程知识，并发知识，大家一定要掌握，没有掌握好的同学，大家需要补一补            sender.wakeup();            try {                //todo: 同步的等待sender线程拉取元数据                metadata.awaitUpdate(version, remainingWaitMs);            } catch (TimeoutException ex) {                // Rethrow with original maxWaitMs to prevent logging exception with remainingWaitMs                throw new TimeoutException(                        String.format("Topic %s not present in metadata after %d ms.",                                topic, maxWaitMs));            }            //todo： 获取一下集群的元数据信息。（上面可能成功更新了集群元数据信息了）            cluster = metadata.fetch();            //todo: 计算一下 拉取元数据已经花了多少时间            elapsed = time.milliseconds() - nowMs;            //todo: 如果花的时间大于 最大等待的时间，那么就报超时。            if (elapsed >= maxWaitMs) {                throw new TimeoutException(partitionsCount == null ?                        String.format("Topic %s not present in metadata after %d ms.",                                topic, maxWaitMs) :                        String.format("Partition %d of topic %s with partition count %d is not present in metadata after %d ms.",                                partition, topic, partitionsCount, maxWaitMs));            }            metadata.maybeThrowExceptionForTopic(topic);            //todo: 计算出来 还可以用的时间。            remainingWaitMs = maxWaitMs - elapsed;            //todo:获取该topic的分区数，            // 如果这个值不为null，说明前面sender线程已经获取到元数据了。            partitionsCount = cluster.partitionCountForTopic(topic);        } while (partitionsCount == null || (partition != null && partition >= partitionsCount));        //todo:  返回集群的元数据信息和获取元数据信息需要的时间        //2个参数        //第一个参数表示集群的元数据信息        //第二个参数表示拉取元数据需要的时间        return new ClusterAndWaitTime(cluster, elapsed);    }
```

![image-20220208195950720](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220208195950720.png?lastModify=1647874003)

#### 7.1、查看awaitUpdate方法的具体实现内容如下

```
 /**     * Wait for metadata update until the current version is larger than the last version we know of     */    public synchronized void awaitUpdate(final int lastVersion, final long timeoutMs) throws InterruptedException {        long currentTimeMs = time.milliseconds();        long deadlineMs = currentTimeMs + timeoutMs < 0 ? Long.MAX_VALUE : currentTimeMs + timeoutMs;        //可以查看waitObject方法的具体实现，使用ctrl + alt + B快捷键进入方法的具体实现        time.waitObject(this, () -> {            // Throw fatal exceptions, if there are any. Recoverable topic errors will be handled by the caller.            maybeThrowFatalException();            return updateVersion() > lastVersion || isClosed();        }, deadlineMs);        if (isClosed())            throw new KafkaException("Requested metadata update after close");    }
```

![image-20220208195741732](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220208195741732.png?lastModify=1647874003)

#### 7.2、查看sender线程是如何拉取元数据信息的

拉取元数据信息，其实就是在KafkaProducer这个类当中的waitOnMeatadata方法中去实现的，在这个方法当中唤醒了sender线程，sender线程就会去运行run方法，那么我们直接去查看sender线程的run方法即可

sender的主要逻辑实际在这个方法中。这个方法前面一大段逻辑都是处理开启事物的消息发送，我们为了简化代码理解，不做这部分分析，直接进入非事务的消息发送逻辑中。

直接查看sender线程当中的run方法的具体实现如下

```
 /**     * The main run loop for the sender thread     */    @Override    public void run() {        log.debug("Starting Kafka producer I/O thread.");        // main loop, runs until close is called        while (running) {            try {                runOnce();            } catch (Exception e) {                log.error("Uncaught error in kafka producer I/O thread: ", e);            }        }        log.debug("Beginning shutdown of Kafka producer I/O thread, sending remaining records.");        // okay we stopped accepting requests but there may still be        // requests in the transaction manager, accumulator or waiting for acknowledgment,        // wait until these are completed.        while (!forceClose && ((this.accumulator.hasUndrained() || this.client.inFlightRequestCount() > 0) || hasPendingTransactionalRequests())) {            try {                runOnce();            } catch (Exception e) {                log.error("Uncaught error in kafka producer I/O thread: ", e);            }        }        // Abort the transaction if any commit or abort didn't go through the transaction manager's queue        while (!forceClose && transactionManager != null && transactionManager.hasOngoingTransaction()) {            if (!transactionManager.isCompleting()) {                log.info("Aborting incomplete transaction due to shutdown");                transactionManager.beginAbort();            }            try {                runOnce();            } catch (Exception e) {                log.error("Uncaught error in kafka producer I/O thread: ", e);            }        }        if (forceClose) {            // We need to fail all the incomplete transactional requests and batches and wake up the threads waiting on            // the futures.            if (transactionManager != null) {                log.debug("Aborting incomplete transactional requests due to forced shutdown");                transactionManager.close();            }            log.debug("Aborting incomplete batches due to forced shutdown");            this.accumulator.abortIncompleteBatches();        }        try {            this.client.close();        } catch (Exception e) {            log.error("Failed to close network client", e);        }        log.debug("Shutdown of Kafka producer I/O thread has completed.");    }
```

![image-20220208200313346](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220208200313346.png?lastModify=1647874003)

#### 7.3、查看runonce方法的具体实现如下

查看runonce方法的具体实现如下

```
  /**     * Run a single iteration of sending     *     */    void runOnce() {        if (transactionManager != null) {            try {                transactionManager.maybeResolveSequences();                // do not continue sending if the transaction manager is in a failed state                if (transactionManager.hasFatalError()) {                    RuntimeException lastError = transactionManager.lastError();                    if (lastError != null)                        maybeAbortBatches(lastError);                    //通过客户端对象，调用poll方法去拉取元数据，真正的发送网络请求都是在这个方法里面，后续我们                    //专门讲网络通信这一块儿再回来查看这块儿源码                    client.poll(retryBackoffMs, time.milliseconds());                    return;                }                // Check whether we need a new producerId. If so, we will enqueue an InitProducerId                // request which will be sent below                transactionManager.bumpIdempotentEpochAndResetIdIfNeeded();                if (maybeSendAndPollTransactionalRequest()) {                    return;                }            } catch (AuthenticationException e) {                // This is already logged as error, but propagated here to perform any clean ups.                log.trace("Authentication exception while processing transactional request", e);                transactionManager.authenticationFailed(e);            }        }        long currentTimeMs = time.milliseconds();        //拉取到了元数据信息，当然就是开始发送数据了        long pollTimeout = sendProducerData(currentTimeMs);        client.poll(pollTimeout, currentTimeMs);    }
```

![image-202202082008138865](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-202202082008138865.png?lastModify=1647874003)

**7.3.1、查看runonce方法中的sendProduerData方法的具体实现**

在我们看到通过KafkaClient这个类调用poll方法拉取元数据的时候，在下面继续调用了sendProducerData方法来发送数据，sendPorducerData方法的具体实现如下

```
 private long sendProducerData(long now) {        //todo: 1、获取元数据        // 因为这里采用场景驱动的方式，由于代码第一次进来，目前还没有获取到元数据。 所以这个Cluster对象中是没有元数据的        //todo: 注意: 如果没有获取到元数据，下面的一些代码逻辑不需要看了，因为下面的代码都需要依赖于这个元数据        /**         * 现在我们的代码是第二次进来，这个时候就应该有对应的元数据了。         * todo:步骤一：         *      获取元数据         */        Cluster cluster = metadata.fetch();        // get the list of partitions with data ready to send        /***         * todo:  步骤二：         *      判断哪些partition有消息可以发送，获取到这个partition的leaderPartition对应的broker主机         *      获取当前准备发送的partitions         */        // get the list of partitions with data ready to send        //目前消息已经封装在不同分区队列中的批次中，判断哪些批次可以把数据发送出去        RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);        // if there are any partitions whose leaders are not known yet, force metadata update        /**         * todo: 步骤三：         *     如果有些partition没有leader信息，更新metadata         */        if (!result.unknownLeaderTopics.isEmpty()) {            // The set of topics with unknown leader contains topics with leader election pending as well as            // topics which may have expired. Add the topic again to metadata to ensure it is included            // and request metadata update, since there are messages to send to the topic.            for (String topic : result.unknownLeaderTopics)                this.metadata.add(topic, now);            log.debug("Requesting metadata update due to unknown leader topics from the batched records: {}",                result.unknownLeaderTopics);            this.metadata.requestUpdate();        }        // remove any nodes we aren't ready to send to        //遍历所有获取的网络节点，基于网络连接状态检测这些节点是否可用，如果不可用就剔除        Iterator<Node> iter = result.readyNodes.iterator();        long notReadyTimeout = Long.MAX_VALUE;        while (iter.hasNext()) {            Node node = iter.next();            /**             * todo: 步骤四：             *      检查与要发送数据的主机网络是否建立好，去掉那些不能发送信息的节点             */            if (!this.client.ready(node, now)) {                //如果网络没有建立好，这里返回的就是false， !false就可以进来了                //移除这些主机                iter.remove();                notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));            }        }        // create produce requests        /**         * todo: 步骤五：         *        有可能要发送的partition有很多个，这些partition的leader分区可能在同一台节点上         *        p0:leader ----> brokerId=0         *        p1:leader ----> brokerId=0         *        p2:leader ----> brokerId=1         *        p3:leader ----> brokerId=2         *   按照brokerId进行分区，同一个broker的partition为同一组         *  key    value         *  0      p0,p1         *  1      p2         *  2      p3         */        // 获取要发送的records，如果网络没有建立好，这块代码也是不会执行的        Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes, this.maxRequestSize, now);        addToInflightBatches(batches);        //保证顺序发送 也就是该参数 max.in.flight.requests.per.connection = 1        if (guaranteeMessageOrder) {            // Mute all the partitions drained            //如果网络没有建立好，batches为空，这块代码也不会执行            for (List<ProducerBatch> batchList : batches.values()) {                for (ProducerBatch batch : batchList)                    this.accumulator.mutePartition(batch.topicPartition);            }        }        /**         * TODO: 步骤六：         * todo: 放弃超时的batches         * 超时批次的处理逻辑         */        accumulator.resetNextBatchExpiryTime();        List<ProducerBatch> expiredInflightBatches = getExpiredInflightBatches(now);        List<ProducerBatch> expiredBatches = this.accumulator.expiredBatches(now);        expiredBatches.addAll(expiredInflightBatches);        // Reset the producer id if an expired batch has previously been sent to the broker. Also update the metrics        // for expired batches. see the documentation of @TransactionState.resetIdempotentProducerId to understand why        // we need to reset the producer id here.        if (!expiredBatches.isEmpty())            log.trace("Expired {} batches in accumulator", expiredBatches.size());        for (ProducerBatch expiredBatch : expiredBatches) {            String errorMessage = "Expiring " + expiredBatch.recordCount + " record(s) for " + expiredBatch.topicPartition                + ":" + (now - expiredBatch.createdMs) + " ms has passed since batch creation";            failBatch(expiredBatch, new TimeoutException(errorMessage), false);            if (transactionManager != null && expiredBatch.inRetry()) {                // This ensures that no new batches are drained until the current in flight batches are fully resolved.                transactionManager.markSequenceUnresolved(expiredBatch);            }        }        sensors.updateProduceRequestMetrics(batches);        // If we have any nodes that are ready to send + have sendable data, poll with 0 timeout so this can immediately        // loop and try sending more data. Otherwise, the timeout will be the smaller value between next batch expiry        // time, and the delay time for checking data availability. Note that the nodes may have data that isn't yet        // sendable due to lingering, backing off, etc. This specifically does not include nodes with sendable data        // that aren't ready to send since they would cause busy looping.        long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);        pollTimeout = Math.min(pollTimeout, this.accumulator.nextExpiryTimeMs() - now);        pollTimeout = Math.max(pollTimeout, 0);        if (!result.readyNodes.isEmpty()) {            log.trace("Nodes with data ready to send: {}", result.readyNodes);            // if some partitions are already ready to be sent, the select time would be 0;            // otherwise if some partition already has some data accumulated but not ready yet,            // the select time will be the time difference between now and its linger expiry time;            // otherwise the select time will be the time difference between now and the metadata expiry time;            pollTimeout = 0;        }        //todo: 将待发送的ProducerBatch封装成为ClientRequest        sendProduceRequests(batches, now);        return pollTimeout;    }
```

![image-20220209101628486](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220209101628486.png?lastModify=1647874003)

在整个sendProducerData()方法当中，主要经历了以下几个步骤

* 1、先获取元数据信息，第一次进来获取不到
* 2、根据accumulator中待发送消息对应的主题分区，检查kafka集群对应的node哪些可用，哪些不可用。得到ReadyCheckResult 结果
* 3、如果ReadyCheckResult 中的unknownLeaderTopics有值，那么则需要更新Kafka集群元数据
* 4、循环readyNodes，检查KafkaClient对该node是否符合网络IO的条件，不符合的从集合中删除。
* 5、通过accumulator.drain()方法把待发送的消息按node号进行分组，返回Map\<Integer, List\<ProducerBatch>>
* 6、把待发送的batch添加到Sender的inFlightBatches中。inFlightBatches是Map\<TopicPartition,List\<ProducerBatch>>，可见是按照主题分区来存储的。
* 7、获取所有过期的batch，循环做过期处理
* 8、计算接下来外层程序逻辑中调用NetWorkClient的poll操作时的timeout时间
* 9、调用sendProduceRequests()方法，将待发送的ProducerBatch封装成为ClientRequest，然后“发送”出去。注意这里的发送，其实只是加入发送的队列。等到NetWorkClient进行poll操作时，才发生网络IO
* 10、返回第7步中计算的poll操作timeout时间

**7.3.3.1、查看sendProduceRequests方法的具体实现**

在sendProduceRequests方法当中，执行了很多的步骤逻辑，前面的一些步骤都是在为发送数据做准备的，最后调用了sendProduceRequests方法来将数据封装到发送的队列，等待最后进行网络操作才会将数据真正的发送出去

```
   /**     * Transfer the record batches into a list of produce requests on a per-node basis     */    private void sendProduceRequests(Map<Integer, List<ProducerBatch>> collated, long now) {        //这里循环遍历每一个批次，每一个批次的数据循环出来之后，又调用了sendProduceRequest这个方法，注意这个方法跟sendProduceRequests是不一样的，这两个方法        //没有关系，不是递归        for (Map.Entry<Integer, List<ProducerBatch>> entry : collated.entrySet())            sendProduceRequest(now, entry.getKey(), acks, requestTimeoutMs, entry.getValue());    }
```

![image-20220209103732449](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220209103732449.png?lastModify=1647874003)

**7.3.3.2、查看sendProduceRequest方法的具体实现**

通过sendProduceRequests方法里面循环遍历每一个批次的数据，然后紧接着就调用了sendProduceRequest方法，这两个方法没有任何关系，不是重载，也不是递归的关系，sendProduceRequest方法的具体实现如下

```
/**     * Create a produce request from the given record batches     */    private void sendProduceRequest(long now, int destination, short acks, int timeout, List<ProducerBatch> batches) {        if (batches.isEmpty())            return;        final Map<TopicPartition, ProducerBatch> recordsByPartition = new HashMap<>(batches.size());        // find the minimum magic version used when creating the record sets        byte minUsedMagic = apiVersions.maxUsableProduceMagic();        //遍历所有的batch消息，找到最新的版本号信息        for (ProducerBatch batch : batches) {            if (batch.magic() < minUsedMagic)                minUsedMagic = batch.magic();        }        //比那里ProducerBatch集合，整理成ProducerBatch和MemoryRecords        ProduceRequestData.TopicProduceDataCollection tpd = new ProduceRequestData.TopicProduceDataCollection();        for (ProducerBatch batch : batches) {            TopicPartition tp = batch.topicPartition;            MemoryRecords records = batch.records();            // down convert if necessary to the minimum magic used. In general, there can be a delay between the time            // that the producer starts building the batch and the time that we send the request, and we may have            // chosen the message format based on out-dated metadata. In the worst case, we optimistically chose to use            // the new message format, but found that the broker didn't support it, so we need to down-convert on the            // client before sending. This is intended to handle edge cases around cluster upgrades where brokers may            // not all support the same message format version. For example, if a partition migrates from a broker            // which is supporting the new magic version to one which doesn't, then we will need to convert.            if (!records.hasMatchingMagic(minUsedMagic))                records = batch.records().downConvert(minUsedMagic, 0, time).records();            ProduceRequestData.TopicProduceData tpData = tpd.find(tp.topic());            if (tpData == null) {                tpData = new ProduceRequestData.TopicProduceData().setName(tp.topic());                tpd.add(tpData);            }            tpData.partitionData().add(new ProduceRequestData.PartitionProduceData()                    .setIndex(tp.partition())                    .setRecords(records));            recordsByPartition.put(tp, batch);        }        String transactionalId = null;        if (transactionManager != null && transactionManager.isTransactional()) {            transactionalId = transactionManager.transactionalId();        }        ProduceRequest.Builder requestBuilder = ProduceRequest.forMagic(minUsedMagic,                new ProduceRequestData()                        .setAcks(acks)                        .setTimeoutMs(timeout)                        .setTransactionalId(transactionalId)                        .setTopicData(tpd));        //构建回调函数        RequestCompletionHandler callback = response -> handleProduceResponse(response, recordsByPartition, time.milliseconds());        String nodeId = Integer.toString(destination);        //构建ClientRequest对象，这个才是真正准备发送数据的对象        ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0,                requestTimeoutMs, callback);        //调用send方法，发送数据        client.send(clientRequest, now);        log.trace("Sent produce request to {}: {}", nodeId, requestBuilder);    }
```

* 1、循环batches，MemoryRecords records = batch.records(); 获取ProducerBatch中组织好的消息内容也。按照主题分区分堆ProducerBatch和MemoryRecords得到两个以TopicPartition为Key的Map，produceRecordsByPartition和recordsByPartition
* 2、声明ProduceRequest.Builder对象，他内部有引用指向produceRecordsByPartition
* 3、生成ClientRequest对象
* 4、调用NetWorkClient的send方法。
* 在NetWorkClient的send方法中，通过Builder对象的build方法生成ProduceRequest对象。再通过request.toSend(destination, header)，得到NetworkSend对象send。生成InFlightRequest对象，保存起来。最后调用selector.send(send)；这个方法会把send请求加入队列等待随后的poll方法把它发送出去。

![image-20220209110036941](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220209110036941.png?lastModify=1647874003)

**7.4、查看runonce方法里面的 client.poll方法的具体实现**

```
  //拉取到了元数据信息，当然就是开始发送数据了        long pollTimeout = sendProducerData(currentTimeMs);        client.poll(pollTimeout, currentTimeMs);
```

在前面的runonce方法当中，有两行代码，一行代码是执行了sendProducerData这个方法，我们已经具体分析过了，其实底层就是去执行了网络操作，还有一行代码是执行了client.poll方法，接下来我们就来看一下client.poll方法的具体试下如下：

```
 /**     * Do actual reads and writes to sockets.     *     * @param timeout The maximum amount of time to wait (in ms) for responses if there are none immediately,     *                must be non-negative. The actual timeout will be the minimum of timeout, request timeout and     *                metadata timeout     * @param now The current time in milliseconds     * @return The list of responses received     */    @Override    public List<ClientResponse> poll(long timeout, long now) {        ensureActive();        if (!abortedSends.isEmpty()) {            // If there are aborted sends because of unsupported version exceptions or disconnects,            // handle them immediately without waiting for Selector#poll.            //判断如果版本不支持，或者断开连接，优先处理这些不支持的发送请求            List<ClientResponse> responses = new ArrayList<>();            handleAbortedSends(responses);            completeResponses(responses);            return responses;        }        //todo: 1、封装了一个要拉取元数据请求        long metadataTimeout = metadataUpdater.maybeUpdate(now);        try {            //todo: 2、发送请求、进行复杂的网络操作            /**             * 但是我们目前还没有学习到kafka的网络             * 所以这儿大家就只需要知道这儿会发送网络请求。             */            this.selector.poll(Utils.min(timeout, metadataTimeout, defaultRequestTimeoutMs));        } catch (IOException e) {            log.error("Unexpected error during I/O", e);        }        // process completed actions        long updatedNow = this.time.milliseconds();        List<ClientResponse> responses = new ArrayList<>();        //处理完成的Sender对象        handleCompletedSends(responses, updatedNow);        /**         * todo: 3、处理响应，响应里面就会有我们需要的元数据。         *         * 这个地方是我们在看生产者是如何获取元数据的时候，看的。         * 其实Kafak获取元数据的流程跟我们发送消息的流程是一模一样。         * 获取元数据 ---> 判断网络连接是否建立好 ---> 建立网络连接 ---> 发送请求（获取元数据的请求）---> 服务端发送回来响应（带了集群的元数据信息）         */        //处理完成的Send对象        handleCompletedReceives(responses, updatedNow);        //处理关闭的响应        handleDisconnections(responses, updatedNow);        //处理连接状态变化        handleConnections();        //循环处理nodesNeedingApiVersionsFetch        handleInitiateApiVersionRequests(updatedNow);        //处理超时的请求        handleTimedOutConnections(responses, updatedNow);        handleTimedOutRequests(responses, updatedNow);        //处理完成的ClientResponse        completeResponses(responses);        return responses;    }
```

![image-20220209113740645](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220209113740645.png?lastModify=1647874003)

**7.4.1、查看 metadataUpdater.maybeUpdate方法的具体实现**

在NetworkClient当中执行poll方法的时候，拉取元数据的请求都是封装在了这一行代码块当中，

```
 //todo: 1、封装了一个要拉取元数据请求        long metadataTimeout = metadataUpdater.maybeUpdate(now);
```

查看这一行代码的具体实现如下，使用ctrl + alt + B快捷键来查看该方法的具体实现在NetworkClient这个类当中的具体实现如下

```
 @Override        public long maybeUpdate(long now) {            // should we update our metadata?            long timeToNextMetadataUpdate = metadata.timeToNextUpdate(now);            long waitForMetadataFetch = hasFetchInProgress() ? defaultRequestTimeoutMs : 0;            //判断时间是否超时            long metadataTimeout = Math.max(timeToNextMetadataUpdate, waitForMetadataFetch);            if (metadataTimeout > 0) {                return metadataTimeout;            }            // Beware that the behavior of this method and the computation of timeouts for poll() are            // highly dependent on the behavior of leastLoadedNode.            Node node = leastLoadedNode(now);            if (node == null) {                log.debug("Give up sending metadata request since no node is available");                return reconnectBackoffMs;            }            //继续调用了maybeUpdate方法，查看maybeUpdate方法具体实现如下            return maybeUpdate(now, node);        }
```

![image-20220209111540128](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220209111540128.png?lastModify=1647874003)

**7.4.2、继续查看maybeUpdate方法的重载实现如下**

```
/**         * Add a metadata request to the list of sends if we can make one         */        private long maybeUpdate(long now, Node node) {            //todo:获取主机id            String nodeConnectionId = node.idString();            //todo:判断网络是否建立成功            if (canSendRequest(nodeConnectionId, now)) {                //这里来判断是全量拉取数据还是增量拉取数据                Metadata.MetadataRequestAndVersion requestAndVersion = metadata.newMetadataRequestAndVersion(now);                MetadataRequest.Builder metadataRequest = requestAndVersion.requestBuilder;                log.debug("Sending metadata request {} to node {}", metadataRequest, node);                //todo:创建了一个拉取元数据的请求，所有的元数据的获取网络请求都在这里                sendInternalMetadataRequest(metadataRequest, nodeConnectionId, now);                inProgress = new InProgressData(requestAndVersion.requestVersion, requestAndVersion.isPartialUpdate);                return defaultRequestTimeoutMs;            }            // If there's any connection establishment underway, wait until it completes. This prevents            // the client from unnecessarily connecting to additional nodes while a previous connection            // attempt has not been completed.            if (isAnyNodeConnecting()) {                // Strictly the timeout we should return here is "connect timeout", but as we don't                // have such application level configuration, using reconnect backoff instead.                return reconnectBackoffMs;            }            if (connectionStates.canConnect(nodeConnectionId, now)) {                // We don't have a connection to this node right now, make one                log.debug("Initialize connection to node {} for sending metadata request", node);                initiateConnect(node, now);                return reconnectBackoffMs;            }            // connected, but can't send more OR connecting            // In either case, we just need to wait for a network event to let us know the selected            // connection might be usable again.            return Long.MAX_VALUE;        }
```

![image-20220209112223678](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220209112223678.png?lastModify=1647874003)

**7.4.3、查看sendInternalMetadataRequest方法的具体实现**

sendInternalMetadataRequest方法具体实现内容如下

```
    // package-private for testing    void sendInternalMetadataRequest(MetadataRequest.Builder builder, String nodeConnectionId, long now) {        ClientRequest clientRequest = newClientRequest(nodeConnectionId, builder, now, true);        doSend(clientRequest, true, now);    }
```

**7.4.4、查看doSend方法的具体实现如下**

```
private void doSend(ClientRequest clientRequest, boolean isInternalRequest, long now) {        ensureActive();        String nodeId = clientRequest.destination();        if (!isInternalRequest) {            // If this request came from outside the NetworkClient, validate            // that we can send data.  If the request is internal, we trust            // that internal code has done this validation.  Validation            // will be slightly different for some internal requests (for            // example, ApiVersionsRequests can be sent prior to being in            // READY state.)            if (!canSendRequest(nodeId, now))                throw new IllegalStateException("Attempt to send a request to node " + nodeId + " which is not ready.");        }        AbstractRequest.Builder<?> builder = clientRequest.requestBuilder();        try {            NodeApiVersions versionInfo = apiVersions.get(nodeId);            short version;            // Note: if versionInfo is null, we have no server version information. This would be            // the case when sending the initial ApiVersionRequest which fetches the version            // information itself.  It is also the case when discoverBrokerVersions is set to false.            if (versionInfo == null) {                version = builder.latestAllowedVersion();                if (discoverBrokerVersions && log.isTraceEnabled())                    log.trace("No version information found when sending {} with correlation id {} to node {}. " +                            "Assuming version {}.", clientRequest.apiKey(), clientRequest.correlationId(), nodeId, version);            } else {                version = versionInfo.latestUsableVersion(clientRequest.apiKey(), builder.oldestAllowedVersion(),                        builder.latestAllowedVersion());            }            // The call to build may also throw UnsupportedVersionException, if there are essential            // fields that cannot be represented in the chosen version.            doSend(clientRequest, isInternalRequest, now, builder.build(version));        } catch (UnsupportedVersionException unsupportedVersionException) {            // If the version is not supported, skip sending the request over the wire.            // Instead, simply add it to the local queue of aborted requests.            log.debug("Version mismatch when attempting to send {} with correlation id {} to {}", builder,                    clientRequest.correlationId(), clientRequest.destination(), unsupportedVersionException);            ClientResponse clientResponse = new ClientResponse(clientRequest.makeHeader(builder.latestAllowedVersion()),                    clientRequest.callback(), clientRequest.destination(), now, now,                    false, unsupportedVersionException, null, null);            if (!isInternalRequest)                abortedSends.add(clientResponse);            else if (clientRequest.apiKey() == ApiKeys.METADATA)                metadataUpdater.handleFailedRequest(now, Optional.of(unsupportedVersionException));        }    }
```

在doSend方法里面使用了方法的重载，继续查看doSend方法的四个参数具体实现如下

```
    private void doSend(ClientRequest clientRequest, boolean isInternalRequest, long now, AbstractRequest request) {        String destination = clientRequest.destination();        RequestHeader header = clientRequest.makeHeader(request.version());        if (log.isDebugEnabled()) {            log.debug("Sending {} request with header {} and timeout {} to node {}: {}",                clientRequest.apiKey(), header, clientRequest.requestTimeoutMs(), destination, request);        }        Send send = request.toSend(header);        InFlightRequest inFlightRequest = new InFlightRequest(                clientRequest,                header,                isInternalRequest,                request,                send,                now);        this.inFlightRequests.add(inFlightRequest);        selector.send(new NetworkSend(clientRequest.destination(), send));    }
```

看到这里就又涉及到网络NIO的操作了，后面我们专门花时间来讲网络操作

**7.4.2、查看client.poll方法里面的handleCompletedReceives方法的具体实现**

上面我们已经看完了poll方法当中的执行的maybeUpdate方法，接下来看一下poll方法当中的handleCompletedReceives方法的具体实现内容如下

```
/**     * Handle any completed receives and update the response list with the responses received.     *     * @param responses The list of responses to update     * @param now The current time     */    //todo: 获取服务端的响应，然后按照响应分类处理    private void handleCompletedReceives(List<ClientResponse> responses, long now) {        //todo: 从completedReceives中遍历所有接受到的响应，completedReceives中的信息是在上一层的selector.poll中添加进去的        for (NetworkReceive receive : this.selector.completedReceives()) {            //获取响应的节点id            String source = receive.source();            //从inFlightRequests中获取缓存的request对象            InFlightRequest req = inFlightRequests.completeNext(source);            //解析响应信息，验证响应头，生成Struct对象            AbstractResponse response = parseResponse(receive.payload(), req.header);            if (throttleTimeSensor != null)                throttleTimeSensor.record(response.throttleTimeMs(), now);            //打印日志            if (log.isDebugEnabled()) {                log.debug("Received {} response from node {} for request with header {}: {}",                    req.header.apiKey(), req.destination, req.header, response);            }            // If the received response includes a throttle delay, throttle the connection.            maybeThrottle(response, req.header.apiVersion(), req.destination, now);            //todo: 如果是元数据请求的响应            if (req.isInternalRequest && response instanceof MetadataResponse)                //更新元数据信息                metadataUpdater.handleSuccessfulResponse(req.header, now, (MetadataResponse) response);            //todo: 如果是版本协调的响应            else if (req.isInternalRequest && response instanceof ApiVersionsResponse)                handleApiVersionsResponse(responses, req, now, (ApiVersionsResponse) response);            else                //todo: 其他的响应，就添加响应到本地的响应队列中                responses.add(req.completed(response, now));        }    }
```

![image-20220209114447604](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220209114447604.png?lastModify=1647874003)

**7.4.2.1、查看handleCompletedReceives方法当中的handleSuccessfulResponse方法的具体实现**

在上面的handleCompletedReceives方法当中，进行了判断，如果是元数据请求，那么就去执行了handleSuccessfulResponse这个方法，使用ctrl + alt + B快捷键查看该方法的具体实现内容如下

```
 @Override        public void handleSuccessfulResponse(RequestHeader requestHeader, long now, MetadataResponse response) {            // If any partition has leader with missing listeners, log up to ten of these partitions            // for diagnosing broker configuration issues.            // This could be a transient issue if listeners were added dynamically to brokers.            //将响应的response进行处理成为一个集合，确定哪些分区没有监听器            List<TopicPartition> missingListenerPartitions = response.topicMetadata().stream().flatMap(topicMetadata ->                topicMetadata.partitionMetadata().stream()                    .filter(partitionMetadata -> partitionMetadata.error == Errors.LISTENER_NOT_FOUND)                    .map(partitionMetadata -> new TopicPartition(topicMetadata.topic(), partitionMetadata.partition())))                .collect(Collectors.toList());            //打印日志            if (!missingListenerPartitions.isEmpty()) {                int count = missingListenerPartitions.size();                log.warn("{} partitions have leader brokers without a matching listener, including {}",                        count, missingListenerPartitions.subList(0, Math.min(10, count)));            }            // Check if any topic's metadata failed to get updated            Map<String, Errors> errors = response.errors();            if (!errors.isEmpty())                log.warn("Error while fetching metadata with correlation id {} : {}", requestHeader.correlationId(), errors);            // When talking to the startup phase of a broker, it is possible to receive an empty metadata set, which            // we should retry later.            //如果没有获取到broker，元数据拉取失败            if (response.brokers().isEmpty()) {                log.trace("Ignoring empty metadata response with correlation id {}.", requestHeader.correlationId());                this.metadata.failedUpdate(now);            } else {                //todo:元数据获取到了，真正的去更新元数据信息                this.metadata.update(inProgress.requestVersion, response, inProgress.isPartialUpdate, now);            }            inProgress = null;        }
```

![image-20220209114854940](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220209114854940.png?lastModify=1647874003)

**7.4.2.2、查看update方法的具体实现内容如下**

上面看到了在handleSuccessfulResponse方法当中执行了

```
 this.metadata.update(inProgress.requestVersion, response, inProgress.isPartialUpdate, now);
```

这样一行代码，通过update方法，具体实现了更新元数据信息。

查看metadata.update方法的具体实现如下

```
/**     * Updates the cluster metadata. If topic expiry is enabled, expiry time     * is set for topics if required and expired topics are removed from the metadata.     *     * @param requestVersion The request version corresponding to the update response, as provided by     *     {@link #newMetadataRequestAndVersion(long)}.     * @param response metadata response received from the broker     * @param isPartialUpdate whether the metadata request was for a subset of the active topics     * @param nowMs current time in milliseconds     */    public synchronized void update(int requestVersion, MetadataResponse response, boolean isPartialUpdate, long nowMs) {        Objects.requireNonNull(response, "Metadata response cannot be null");        if (isClosed())            throw new IllegalStateException("Update requested after metadata close");        this.needPartialUpdate = requestVersion < this.requestVersion;        this.lastRefreshMs = nowMs;        //拉取到了元数据信息，就将版本号加1        this.updateVersion += 1;        //判断是否需要全量更新元数据信息        if (!isPartialUpdate) {            this.needFullUpdate = false;            this.lastSuccessfulRefreshMs = nowMs;        }        //获取集群ID        String previousClusterId = cache.clusterResource().clusterId();        //todo:在这里进行全量或者增量元数据信息的更新        this.cache = handleMetadataResponse(response, isPartialUpdate, nowMs);        Cluster cluster = cache.cluster();        maybeSetMetadataError(cluster);        this.lastSeenLeaderEpochs.keySet().removeIf(tp -> !retainTopic(tp.topic(), false, nowMs));        String newClusterId = cache.clusterResource().clusterId();        if (!Objects.equals(previousClusterId, newClusterId)) {            log.info("Cluster ID: {}", newClusterId);        }        clusterResourceListeners.onUpdate(cache.clusterResource());        log.debug("Updated cluster metadata updateVersion {} to {}", this.updateVersion, this.cache);    }
```

![image-20220209115616289](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220209115616289.png?lastModify=1647874003)

**7.4.2.3、查看update方法当中调用handleMetadataResponse方法的具体试下**

在update方法当中，执行了handleMetadataResponse这个方法，该方法的具体实现内容如下：

```
 /**     * Transform a MetadataResponse into a new MetadataCache instance.     */    private MetadataCache handleMetadataResponse(MetadataResponse metadataResponse, boolean isPartialUpdate, long nowMs) {        // All encountered topics.        //将所有的topic封装到这个Set集合当中，set集合可以去重        Set<String> topics = new HashSet<>();        // Retained topics to be passed to the metadata cache.        Set<String> internalTopics = new HashSet<>();        Set<String> unauthorizedTopics = new HashSet<>();        Set<String> invalidTopics = new HashSet<>();        List<MetadataResponse.PartitionMetadata> partitions = new ArrayList<>();        Map<String, Uuid> topicIds = new HashMap<>();        Map<String, Uuid> oldTopicIds = cache.topicIds();        for (MetadataResponse.TopicMetadata metadata : metadataResponse.topicMetadata()) {            String topicName = metadata.topic();            Uuid topicId = metadata.topicId();            topics.add(topicName);            // We can only reason about topic ID changes when both IDs are valid, so keep oldId null unless the new metadata contains a topic ID            Uuid oldTopicId = null;            if (!Uuid.ZERO_UUID.equals(topicId)) {                topicIds.put(topicName, topicId);                oldTopicId = oldTopicIds.get(topicName);            } else {                topicId = null;            }            if (!retainTopic(topicName, metadata.isInternal(), nowMs))                continue;            if (metadata.isInternal())                internalTopics.add(topicName);            //判断元数据信息没有任何错误            if (metadata.error() == Errors.NONE) {                //todo：循环遍历所有分区的元数据信息                for (MetadataResponse.PartitionMetadata partitionMetadata : metadata.partitionMetadata()) {                    // Even if the partition's metadata includes an error, we need to handle                    // the update to catch new epochs                    //更新所有的元数据信息                    updateLatestMetadata(partitionMetadata, metadataResponse.hasReliableLeaderEpochs(), topicId, oldTopicId)                        .ifPresent(partitions::add);                    //如果元数据信息有错误，调用requestUpdate方法重新获取元数据信息                    if (partitionMetadata.error.exception() instanceof InvalidMetadataException) {                        log.debug("Requesting metadata update for partition {} due to error {}",                                partitionMetadata.topicPartition, partitionMetadata.error);                        requestUpdate();                    }                }            } else {                //如果元数据信息有任何的错误，调用requestUpdate方法重新获取元数据信息                if (metadata.error().exception() instanceof InvalidMetadataException) {                    log.debug("Requesting metadata update for topic {} due to error {}", topicName, metadata.error());                    requestUpdate();                }                if (metadata.error() == Errors.INVALID_TOPIC_EXCEPTION)                    invalidTopics.add(topicName);                else if (metadata.error() == Errors.TOPIC_AUTHORIZATION_FAILED)                    unauthorizedTopics.add(topicName);            }        }        Map<Integer, Node> nodes = metadataResponse.brokersById();        if (isPartialUpdate)            return this.cache.mergeWith(metadataResponse.clusterId(), nodes, partitions,                unauthorizedTopics, invalidTopics, internalTopics, metadataResponse.controller(), topicIds,                (topic, isInternal) -> !topics.contains(topic) && retainTopic(topic, isInternal, nowMs));        else            return new MetadataCache(metadataResponse.clusterId(), nodes, partitions,                unauthorizedTopics, invalidTopics, internalTopics, metadataResponse.controller(), topicIds);    }
```

![image-20220209120159719](file:///D:/cc/study/md/kafka/day02%E8%AF%BE%E4%BB%B6/kafka3%E6%96%B0%E7%89%B9%E6%80%A7%E8%A7%A3%E5%AF%86/kafka3%E6%96%B0%E7%89%B9%E6%80%A7.assets/image-20220209120159719.png?lastModify=1647874003)

**7.4.2.4、查看handleMetadataResponse 方法当中的updateLatestMetadata方法的具体实现**

在handleMetadataResponse 方法当中，判断如果获取到的元数据信息没有任何问题，那么继续调用了updateLatestMetadata方法，该方法的具体实现如下：

```
/**     * Compute the latest partition metadata to cache given ordering by leader epochs (if both     * available and reliable) and whether the topic ID changed.     */    private Optional<MetadataResponse.PartitionMetadata> updateLatestMetadata(            MetadataResponse.PartitionMetadata partitionMetadata,            boolean hasReliableLeaderEpoch,            Uuid topicId,            Uuid oldTopicId) {        //获取topic对应的partition        TopicPartition tp = partitionMetadata.topicPartition;        //判断如果leader分区有Leader Epoch的值，并且分区元数据的Leader Epoch值出现        if (hasReliableLeaderEpoch && partitionMetadata.leaderEpoch.isPresent()) {            //todo:获取新的Leader Epoch的值            int newEpoch = partitionMetadata.leaderEpoch.get();            //todo：获取当前的Leader  Epoch的值            Integer currentEpoch = lastSeenLeaderEpochs.get(tp);            if (topicId != null && !topicId.equals(oldTopicId)) {                // If the new topic ID is valid and different from the last seen topic ID, update the metadata.                // Between the time that a topic is deleted and re-created, the client may lose track of the                // corresponding topicId (i.e. `oldTopicId` will be null). In this case, when we discover the new                // topicId, we allow the corresponding leader epoch to override the last seen value.                log.info("Resetting the last seen epoch of partition {} to {} since the associated topicId changed from {} to {}",                         tp, newEpoch, oldTopicId, topicId);                lastSeenLeaderEpochs.put(tp, newEpoch);                return Optional.of(partitionMetadata);            } else if (currentEpoch == null || newEpoch >= currentEpoch) {                // If the received leader epoch is at least the same as the previous one, update the metadata                log.debug("Updating last seen epoch for partition {} from {} to epoch {} from new metadata", tp, currentEpoch, newEpoch);                lastSeenLeaderEpochs.put(tp, newEpoch);                return Optional.of(partitionMetadata);            } else {                // Otherwise ignore the new metadata and use the previously cached info                log.debug("Got metadata for an older epoch {} (current is {}) for partition {}, not updating", newEpoch, currentEpoch, tp);                return cache.partitionMetadata(tp);            }        } else {            // Handle old cluster formats as well as error responses where leader and epoch are missing            lastSeenLeaderEpochs.remove(tp);            return Optional.of(partitionMetadata.withoutLeaderEpoch());        }    }
```

最后如果metadata获取成功，那么就直接更新元数据信息，

### 8、kafka加载元数据主体流程图
