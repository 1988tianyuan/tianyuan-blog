title: kafka学习笔记（1）—— 基础知识
author: 天渊
tags:
  - Kafka
  - 大数据
categories:
  - 基础知识
date: 2019-02-13 16:31:00
---
Kafka是由LinkedIn开发并开源的分布式发布/订阅模式的消息队列系统，因其分布式及高吞吐率而被广泛使用，目前在大数据处理领域占有很重要的地位，能够很方便地与Hadoop, Spark, Storm, Flink和Flume等大数据处理工具进行集成
<!--more-->

### Kafka主要特点

Kafka有三个主要的作用：`传统意义上的消息系统`，`分布式存储`，`流处理工具`

**Kafka最重要的作用就是企业级消息队列服务**：

- 以时间复杂度为O(1)的方式提供消息持久化能力，即使对TB级以上数据也能保证常数时间复杂度的访问性能
- Server间的topic分区（partition）消费，保证了高吞吐率，同时保证每个分区内的消息顺序传输
- 发布/订阅模式中，单个topic支持多consumer，并支持consumer-group自动负载均衡

- 不同于RabbitMQ等传统消息队列，kafka不会删除历史数据
- 支持数据备份（repliaction），通过master-slave方式保证数据一致性以及高可用

**Kafka-stream**：

Kafka Stream是Apache Kafka从0.10版本引入的一个新Feature，它提供了对存储于Kafka内的数据进行流式处理和分析的功能。

### Kafka架构

Kafka涉及到的一些专用名词

- **Broker**
  　　Kafka集群包含一个或多个服务器，这种服务器被称为broker
- **Topic**
  　　每条发布到Kafka集群的消息都有一个类别，这个类别被称为Topic。（物理上不同Topic的消息分开存储，逻辑上一个Topic的消息虽然保存于一个或多个broker上但用户只需指定消息的Topic即可生产或消费数据而不必关心数据存于何处）
- **Partition**
  　　分片，每个Topic包含一个或多个Partition.
- **repliaction**
  　　复制集，每个partition包含一个或多个repliaction，其中有一个master多个slave
- **Producer**
  　　消息发布者，负责发布消息到Kafka broker
- **Consumer**
  　　消息消费者，通过拉取的方式向Kafka broker读取消息
- **Consumer Group**
  　　每个Consumer属于一个特定的Consumer Group（可为每个Consumer指定group name，若不指定group name则属于默认的group）。

kafka集群拓扑：

![](/blog/images/kafka.png)
- Kafka集群中包含若干Producer，使用push模式将消息**批量**发布到broker
- 包含若干broker，支持水平扩展，broker数量越多，集群吞吐率越高，新建的topic-partition及其复制集均分到各个broker上
- 由Consumer Group管理consumer实例，单个consumer实例可以订阅多个topic，若同一个Consumer Group中的多个consumer实例订阅了同一个topic，则由kafka集群自动进行负载均衡，统一协调分配partition进行消费
- kafka强依赖Zookeeper集群，通过Zookeeper管理集群配置，保存元数据，选举partition leader，以及在Consumer Group发生变化时进行rebalance

### Kafka与常用MQ对比

- RabbitMQ：支持多种协议栈（AMQP，XMPP, SMTP等），功能更加强大（推拉消费，延迟消费，优先消费等），安全性和可靠性要优于kafka，相比较之下Kafka设计更加简单，吞吐量更高，更加适用于大规模日志数据处理
- RocketMQ：阿里开源的消息队列，最早思路来源于kafka，具有高吞吐量、高可用性、适合大规模分布式系统应用的特点，相比于kafka在可靠性和稳定性方面均有提升，而且支持事务消息
- ZeroMQ：号称最快的消息队列系统，具有超高吞吐量，但作用场景有限，大部分情况下作为数据流传输模块嵌入到各个中间件中

### Kafka文件存储方式

Kafka以顺序I/O的方式将消息存入磁盘进行持久化，保证了足够的刷盘速度:

- Kafka存储的每条消息数据称为`Message`，每条`Message`数据包含四个属性：`offset`，`MessageSize`，`data`，`timestamp`时间戳即时间戳类型

- Kafka的消息队列在逻辑上是由`partition`的方式存在的，每个`partition`的`repliaction`在物理上由多个`segment`分段组成，每个`segment`数据文件以该段中最小的 offset 命名，文件扩展名为`.log`，查找指定 offset 的 Message 的时候，使用二分查找定位到该 Message 在哪个 `segment` 数据文件；写入消息时直接将消息添加到最新`segment`文件的末尾

- 每个`segment`分段都有自己的索引文件，扩展名为`.index`，索引文件采用稀疏索引的方式建立索引，每隔一定字节的数据建立一条索引，这样避免了索引文件占用过多的空间，从而可以将索引文件保留在内存中

![upload successful](\blog\images\pasted-0.png)

  `log.dirs=/home1/irteam/apps/kafka/data/kafka/kafka-logs`

  在该文件夹中，每个topic的partition都有自己单独的文件夹进行存放，比如`resource-v1-APIGateway-Api`这个topic编号为0的partition放如下位置：

  `/home1/irteam/apps/kafka/data/kafka/kafka-logs/resource-v1-APIGateway-Api-0/`
  
![upload successful](\blog\images\pasted-1.png)
  其中`.index`，`.log`，`.timestamp`三个文件分别对应`分段索引文件`，`分段数据`和`时间索引`
    
  **时间索引**：`.timeindex`文件用于保存当前分段中消息发布时间与offset的稀疏索引，用于定期删除消息（`log.retention.hours`参数）
    
    
### Kafka搭建集群即基本操作

1. 前期工作

   首先保证当前主机安装有jdk，推荐jdk 8 及其以上版本

   Kafka发行版自带zookeeper，无需单独安装zookeeper集群，当然也可以自己另外搭建zookeeper集群

   最新版kafka发行版下载地址：

   https://www.apache.org/dyn/closer.cgi?path=/kafka/2.1.0/kafka_2.11-2.1.0.tgz

   下载到本地并解压tar文件得到`kafka_2.11-2.1.0`文件夹 （2.11是kafka源码的scala版本，2.1.0是kafka实际发行版本）：
   
2. 初始化配置

   为了简便起见这里就只配置单点broker；进入`config`文件夹，`server.properties`是kafka集群的主配置文件，大部分配置都可以选择默认，部分配置需要注意一下：

   ```properties
   # 当前主机id，也是集群中唯一id，将保存于zookeeper中
   broker.id=0
   # 监听地址和端口，如果没有配置的话默认当前主机名；端口默认9092
   listeners = PLAINTEXT://your.host.name:9092
   # 给生产者和消费者的广播地址，如果没设置的话默认采用上面的listeners属性
   advertised.listeners=PLAINTEXT://your.host.name:9092
   # kafka数据存放路径
   log.dirs=/tmp/kafka-logs
   # topic默认的分片数，创建topic的适合可以单独指定该属性
   # 理论上，分区数量越多，吞吐量越大，但会造成更严重的资源消耗
   num.partitions=1
   # 是否允许自动创建topic（当producer或者consumer发布/订阅某个不存在的topic时）
   auto.create.topics.enable=true
   # 是否允许删除topic（删除topic后需要重启broker，不过即使这样也无法完全删除topic数据，需要进入zookeeper删除topic元数据，不过很危险，不推荐）
   delete.topic.enable=true
   # kafka采取异步刷盘的方式将内存中收到的消息序列化到硬盘上，下面两个条件任意满足一项即开启刷盘
   # 每收到10000条消息刷盘一次
   log.flush.interval.messages=10000
   # 每隔1000ms刷盘一次
   log.flush.interval.ms=1000
   # 消息删除策略，保存消息的最长时间
   log.retention.hours=168
   # 单个分区保留消息的最大容量，默认就是1G
   log.retention.bytes=1073741824
   # segment分段的最大容量，单个segment超出这个容量后将创建一个新的segment继续保存消息
   log.segment.bytes=1073741824
   # 可接收的消息最大大小（压缩后的大小）
   message.max.bytes=1048576
   # zookeeper配置
   zookeeper.connect=localhost:2181
   zookeeper.connection.timeout.ms=6000
   ```

   关于kafka broker更详细的配置策略请参考官网：https://kafka.apache.org/documentation/#brokerconfigs

3. 配置完成后，首先启动zookeeper，如果启动的是kafka默认自带的zookeeper的话，进入kafka安装目录，按照以下方式启动：

   ```shell
   > ./bin/zookeeper-server-start.sh ./config/zookeeper.properties
   ```

   再执行以下命令可以进入zookeeper，即可查看zookeeper启动状态和执行各种zookeeper相关命令，输入quit退出：

   ```shell
   > ./bin/zookeeper-shell.sh localhost:2181
   ```


4. zookeeper启动完成后，启动kafka broker （-daemon 参数指定kafka进程后台运行）

   ```shell
   > ./bin/kafka-server-start.sh -daemon ./config/server.properties
   ```

   kafka运行日志存放于`安装路径/logs/server.log`中，可以查看运行状态和报错信息

5. 启动完成后进入zookeeper控制台输入`ls /brokers/ids`命令即可查看注册的kafka broker信息


### 基本操作

再控制台中进行一些基本操作，包括创建topic，发布和消费数据

#### 创建topic

输入以下命令创建一个名为test_topic_1的topic：

```shell
> ./bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test_topic_1
```

每次执行kafka脚本都需要指定zookeeper；指定topic的`replication-factor`即复制集数量为1，分片数量`partitions`为1，名称为test_topic_1，然后通过以下命令查看当前所有topic：

```shell
> ./bin/kafka-topics.sh --list --zookeeper localhost:2181
```

执行以下命令查看某个topic的状态，包含分片信息，复制集信息，分片leader以及`Isr`：

```shell
> ./bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test_topic_1
```

需要注意的是， 创建topic时，`replication-factor`不能大于集群中broker的数量，因为每个partition的replication将会均匀分布到不同的broker上；以下是一个partition为2，replication为1的topic信息：

![upload successful](\blog\images\pasted-2.png)

（`Isr`：repliaction副本存活列表，用于leader进行复制集数据同步，这个集合中的所有节点都是存活状态，并且跟leader同步，长时间未与leader进行同步的副本将被踢出该列表）

#### 发布/订阅消息

使用以下命令向`test_topic_1`进入producer控制台，发布消息：

```shell
./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test_topic_1
```

另外再开一个session，使用以下命令进入consumer消费`test_topic_1`的消息：

```shell
./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test_topic_1
```