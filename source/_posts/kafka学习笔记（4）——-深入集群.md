title: kafka学习笔记（4）—— 深入集群
author: 天渊
tags:
  - Kafka
  - 大数据
categories:
  - 基础知识
date: 2019-03-18 13:33:00
---
## KafkaController

`KafkaController`其实就是kafka集群中其中一个broker，他是由zookeeper在多个broker选举出来的`leader broker`，肩负`partition assign`，`consumer rebalance`，`partition election`等重任 <!--more-->

![upload successful](\blog\images\pasted-26.png)

### KafkaController选举

- 新加入集群的broker向zookeeper创建临时节点`/controller`，创建成功则为KafkaController，并在当前节点写入以下信息，其他broker节点会监听`/controller`节点的变化情况

  ```json
  {“version”:1,”brokerid”:1,”timestamp”:”1512018424988”}
  ```

- 新选出的Controller(leader broker)会在zookeeper的`/controller_epoch`节点上创建递增序列，用于区别不同代的leader，并向zookeeper同步元数据，其他follower broker则会监听当前的`controller_epoch`的值，如果在和某个自称为leader的broker通信时发现他的epoch不是最新值，则会选择忽略本次通信（防止controller脑裂）

![upload successful](\blog\images\pasted-27.png)

### controller主导partition leader选举

KafkaController有一个很重要的功能就是在某个partition的leader出现不可用时，主导这个partition各副本之间的新leader选举，单个partition分为 leader副本和follow副本：

- leader副本：每隔partition都有一个leader和多个follower，所有producer和consumer的请求都得通过leader进行处理
- follower副本：follower副本不处理客户端的读写请求，唯一任务就是从首领那里同步数据，如果leader发生崩溃（leader副本所在的broker发生down机或者该broker和zookeeper同步超时），controller则会启动该partition的leader选举，并将新选举产生的leader信息同步给zookeeper

![upload successful](\blog\images\pasted-28.png)

## Partition Leader

分区leader负责消息的读写并协调各副本的数据同步

### hight water mark

kafka为了保证消息数据的高可靠性，只有已经被所有副本完全同步的消息才能被consumer消费，这个所谓的“已经被所有副本完全同步的消息”由`high water mark`来标定，只有offset小于`high water mark`的消息才对consumer可见：

![upload successful](\blog\images\pasted-29.png)

如上所示，只有offset < 3的消息才是可被消费的消息

### LEO (log end offset)

日志末端位移，即当前副本日志中下一条消息的offset，上图中Replaca 0的LEO为5，以此类推，Replica的LEO为4，Replica的LEO为3

partition的leader和followers均保留一份自己的LEO值，同时leader保有所有follower的LEO值，follower向leader同步数据时，leader会根据follower当前的LEO值判断需要同步给他的消息范围，并根据follower的LEO值更新`high water mark`

### ISR (insync replicas)

处于同步状态的partition副本列表，partition的leader保留一份，并且在zookeeper也保留一份，这份列表记录了当前有哪些副本处于有效同步状态（包含leader自己）：

- `replica.lag.time.max.ms`：超过这个时间还未与leader同步的follower将会被踢出ISR列表
- 挂掉的follower重新与leader同步，在同步进度追上leader后重新加入ISR
- `min.insync.replicas`：最小同步副本数，如果ISR当中的副本数目不足，当前partition则会变为不可用状态，拒绝任何produce和consume请求；该参数用于平衡kafka集群的可用性和一致性
- ISR与高水位的关系：ISR中副本的最低LEO即为`high water mark`
- `unclean.leader.election`：不完全的首领选举，如果设置为true，在进行leader选举时可以选举ISR列表以外的副本作为新leader，但这种情况下丢失消息的几率就比较高了

### leader_epoch

当前partition的leader分代标记，用于在某个副本崩溃重启后与当前leader同步消息时，判断当前leader_epoch是否与自己崩溃前保存的leader_epoch信息一致，并根据leader_epoch信息判断是否需要做日志truncate

（关于老版本kafka使用高水位进行truncate风险及kafka丢消息的往事，细节比较复杂，详细请参考这篇文章：[confluence cwiki: use leader epoch rather than high water mark for truncation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-101+-+Alter+Replication+Protocol+to+use+Leader+Epoch+rather+than+High+Watermark+for+Truncation) )

leader_epoch的数据结构是一个键值对：`LeaderEpoch -> StartOffset`，其中`LeaderEpoch `是一个单调递增序列号，每次进行leader选举后都会产生一个`LeaderEpoch `序列号，`StartOffset`是该次选举完成后新leader自己的LEO值

## Partition副本同步

为了保证消息一致性，kafka使用了`high water mark`，`ISR`，`LEO`和`leader_epoch`等方式来处理follower和leader间的消息同步，同步流程如下：













