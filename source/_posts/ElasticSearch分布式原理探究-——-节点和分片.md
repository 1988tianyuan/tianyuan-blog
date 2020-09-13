title: ElasticSearch分布式原理探究 —— 节点和分片
author: 天渊
tags:
  - elasticsearch
categories: []
date: 2019-09-09 17:21:00
---
ElasticSearch `节点`和`分片`原理浅析
<!--more-->

### Node（节点）

一个运行中的 Elasticsearch 实例称为一个` 节点`，而集群是由一个或者多个拥有相同 `cluster.name` 配置的节点组成

Elasticsearch 集群的节点类型：

`Master eligible`：可用于选举master的节点，这种节点可参加master选主流程；通过`node.master`配置项开启，每个节点默认都是开启的

`Master`：master节点，也就是集群主节点，只有主节点才能修改集群状态信息（集群节点信息，所有索引信息，分片路由信息）

`Data Node`：数据节点，用于保存分片数据，集群可以随时增加数据节点用于动态扩容；通过`node.data`配置项开启，每个节点默认都是开启的

`Coordinating Node`：协调节点，接收client的请求并将请求分发到合适的节点，将各个节点返回的数据进行汇总并返回给client；集群中每个节点都具有协调节点的职责

`Client Node`：客户端节点，如果`node.master`和`node.data`都设置为false时，该节点就是个只拥有响应和转发功能的协调节点，专门用于响应client请求

`Hot & Warm Node`：冷热节点，通过对不同硬件配置的节点设置冷热节点

开发环境中一个node可以有多个职责，生产环境下还是推荐单个node具有各自的职责，比如master node就不要负责data node数据保存和修改的职责

![](http://img.mantian.site/201909051623_337.png)

### 分片（Primary Shard & Replica Shard）

Elasticsearch 为每个index配置了多个分片（`shard`），用以提高集群读写吞吐量，将index数据分布到多个node上，客户端通过`routing`参数（默认是文档id）来决定当前文档的请求路由到哪个`shard`上：

```
shard = hash(routing) % number_of_primary_shards
```

一个分片是一个运行的Lucence实例，并且分片数在创建索引后即确定，不允许修改

`副本（replica）`：为了解决index数据高可用的问题，可以为每个shard配备多个冗余备份，即副本`replica`，`replica`数目可以动态调整；副本数越多，写数据的性能会受影响，但会提高读数据的吞吐量；每个sharding都有一个主分片，其他都为`replica`

![](http://img.mantian.site/201909051653_788.png)

如上，该集群有一个index，这个index被分为了两个shard，即shard 0和shard 1，每个shard又有两个replica，其中shard 0的主分片就是P0，副本就是两个R0，同理sharding 1的主分片是P1，副本是两个R1

>**新建index如何分配shard和replica？**
>
>Elasticsearch 在各个data node上分布shard和replica时，会尽量保证单个index的主shard均衡分布不同的node，并且单个shard的主shard和replica分布于不同的node上
>
>每当集群有新加入data node，集群都会重新分配各个shard

**默认配置**：7.0版本中，新增index时默认shard数为1，每个shard默认有1个replica

#### shard规划的原则

- 分片数不宜设置过小：过小会导致后续无法有效地通过增加node进行横向扩容，并且单个shard数据量过多也会导致数据重新分派耗时太多
- 分片数也不宜设置过大：过大的话，也会影响搜索结果汇总时相关性打分的准确性，并且如果单个节点分片过多的话也会导致资源浪费，无法在多个节点间有效地平衡请求负载

#### 文档写操作

对文档的写操作包括`新增`,`索引（修改）`和`删除`等请求，写操作必须作用在主分片上，主分片数据更新完成后才能被复制到各个replica上：

![](http://img.mantian.site/201909051735_122.png)

1. 客户端发送某个写请求到node1，node1此时就是该请求的协调节点
2. node1通过id（或者routing字段）判断该请求对应shard0，由于shard0的主分片位于node3，因此将请求转发给node3
3. node3收到请求，修改主分片P0，修改成功后并行地将该请求转发给两个replica：R0
4. 两个replica数据修改成功后向node3报告修改成功，最后node3再向协调节点node1报告请求成功
5. node1向客户端反馈请求成功

可以看出，如果一个shard的replica越多，写操作的性能也就越低，因为它要保证所有replica都成功更新数据后才会告诉客户端当前请求成功了

此外还有两个影响写操作的配置项，通过降低安全性为代价提高写入性能：

- 修改`consistency`模式：`consistency`一致性，即至少需要多少个shard副本处于活跃状态才能允许写操作，默认`quorum`模式，即需要保证多于半数的副本能够成功写入数据：

  ```
  int( (primary + number_of_replicas) / 2 ) + 1
  ```

  例如shard有1个主shard和2个replica，此时仅需要主shard和其中1个replica处于活跃状态，协调节点就可以开始写操作，否则反馈客户端当前请求失败

  此外`consistency`还可以配置为`ALL`或者`ONE`，即所有副本都处于active状态或者仅主节点active

- 修改`timeout`：即等待副本恢复活跃状态的超时时间，如果设置为100ms，协调节点只会等待100ms，如果等待100ms后活跃副本数目仍未达到规定数量，则直接进行写操作

#### 文档读操作

当向es执行读操作获取某个文档时，协调节点会向保存了该文档的任意一个副本分片发起请求：

![](http://img.mantian.site/201909091005_698.png)

如图，此时node1是协调节点：

1. 客户端向node1发起读请求，node1通过`_id`或者`routing`字段判断所请求的文档在shard1上
2. 协调节点node1每次读请求都会通过轮询的方式向一个shard的所有副本发送请求，实现负载均衡，本次请求发往node2的副本R0
3. node2将文档返回给node1，最后由node1返回给客户端

#### 多文档更新

可以使用`mget`请求批量获取多个文档，也可以使用`bulk`请求批量插入或修改多个文档

##### mget

`mget`请求与普通的`get`读请求原理上是一致的，区别在于`mget`需要为多个分片构建多文档请求，然后再将这些请求并行转发到各个节点上，最后将结果汇总返回给客户端

##### bulk

`bulk`请求即批量更新请求，一次性修改多个文档

![](http://img.mantian.site/201909091115_343.png)

步骤如下：

1. node1作为协调节点收到客户端请求
2. node1构建针对各个分片的多个批量请求，并行地向shard1的主分片P1（保管于当前协调节点node1）和shard0的主分片P0（node3）发起批量请求
3. 各个shard的主分片更新并同步完成后，向协调节点报告更新结果，最后协调节点node1将结果收集整理并返还给客户端