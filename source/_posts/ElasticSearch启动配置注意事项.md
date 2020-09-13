title: ElasticSearch启动配置注意事项
author: 天渊
tags:
  - elasticsearch
categories: []
date: 2019-09-05 11:12:00
---
配置es集群有两个非常重要的选项，这两个选项关系到es集群节点之间互相发现和master选举：<br>
<!--more-->
#### discovery.seed_hosts:
启动es进程后，该进程默认情况下会监听9200和9300两个端口，9200端口作为与外界通信的rest api接口，9300作为es集群节点间进行通信的接口。

如果不对`discovery.seed_hosts`这个选项作配置，es会检测本机9300~9305这个区间内的端口地址，并将其加入到同一个集群内。如果需要手动配置其他集群节点，则需要把所有节点可用的ip地址（或者ip:port）加入到`discovery.seed_hosts`中，该选项默认是`["127.0.0.1", "[::1]"]`即只检测本机的9300~9305端口。

例如将本地局域网的三台服务器加入到集群通信中，配置如下：<br>

```yml
discovery.seed_hosts: ["192.168.0.1:9300", "192.168.0.2:9300", "192.168.0.3"]
```
注意，`192.168.0.3`这台服务器没有指定端口，则会直接检测9300~9305端口区间

#### cluster.initial_master_nodes:
只配置`discovery.seed_hosts`还不行，还需要配置`cluster.initial_master_nodes`选项，将检测到的节点加入到选举流程中。这个过程还关系到`node.name`这个选项，只有在每个节点的es配置文件中指定唯一的`node.name`，才能和`cluster.initial_master_nodes`配合工作 （如果不手动指定`node.name`，则默认是节点hostName）：

上述三个节点的`node.name`均已指定：

```yml
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
```

关于es集群启动过程的细节问题参阅：<br>
[官网文档：启动es集群](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-bootstrap-cluster.html)<br>
[elasticsearch初学终极教程: 把Elastic Search在本地跑起来](https://kalasearch.cn/blog/chapter2-run-elastic-search-locally/)