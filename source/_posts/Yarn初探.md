title: Yarn初探
author: 天渊
tags:
  - Hadoop
  - Yarn
categories:
  - 大数据
date: 2019-05-05 23:15:00
---
Yarn (Yet Another Resource Manager) 是Hadoop 2.0引入的集群计算资源管理系统，最初是为了改善MapReduce任务调度过程，同时也可以支持其他多种分布式计算模式，Yarn不与任何一种计算框架耦合，只参与集群计算资源（CPU，内存等）的分配以及计算任务的调度

探究MapReduce任务从调用`submit()`提交任务到Yarn运行任务的整个过程是件很有意思的事

<!--more-->

### 初始化任务

初始化任务包括以下四个阶段：

1. `申请任务`：主要是向Yarn申请`jobId`
2. `保存job执行文件`：保存job配置信息，分片信息和Jar包等文件
3. `加入任务队列`：向`ResourceManager`提交任务，加入任务队列

#### 申请任务

MapReduce的Client在调用`job.submit()`后，交由`JobSubmitter`进行任务提交，调用`submitJobInternal`方法首先申请一个`jobId`:

```java
//... 略
JobID jobId = submitClient.getNewJobID();
job.setJobID(jobId);
//... 并行度切分，保存Job执行文件，提交任务等
```

其中`submitClient`是mapreduce的RPC client，有两种实现

- `LocalJobRunner`： 用于提交本地运行的任务，本地环境测试就是使用的这个client
- `YARNRunner`：用于向Yarn集群提交任务

如果配置Job时设置配置项`mapreduce.framework.name`为`yarn`，mapreduce将采用`YARNRunner`作为client进行任务提交工作，`YARNRunner`为当前Job分配一个`jobId`作为本次任务的唯一ID

#### 保存Job执行文件

Job执行文件包括job.splits, job.xml和job的Jar包等文件，mapreduce向文件系统（本地文件系统或者hdfs）申请一块区域用于存放执行文件：

```java
//... 
Path submitJobDir = new Path(jobStagingArea, jobId.toString());
//...
Path submitJobFile = JobSubmissionFiles.getJobConfPath(submitJobDir);
```

`submitJobDir`是用于保存Job执行文件的目录，`submitJobFile`即为当前Job文件的目录，格式为`.../staging/jobId`

#### 加入任务队列

使用`submitClient`向Yarn集群（在这里为`ResourceManager`节点）发起RPC请求提交任务：

```java
status = submitClient.submitJob(jobId, submitJobDir.toString(), job.getCredentials());
```

`ResourceManager`会把当前Job加入到任务执行队列中待有可执行任务的资源可用后启动该任务

### 运行Job

集群中能够运行Job的资源是有限的，队列中待执行的Job想要运行需要满足一定的条件，目前Yarn提供了三种任务调度策略：`FIFO调度`，`容量调度`，`公平调度`，日后再分析

#### 启动Container

`ResourceManager`会定期接收各个`NodeManager`发来的节点资源使用信息，某个Job满足运行条件后首先需要申请一个可以运行任务的`NodeManager`，在之上启动一个“容器”：`Container`

> 如何理解Yarn的Container？
>
> 可以理解为Yarn为Job运行而启动的一个运行环境，这个运行环境包含运行资源（程序运行所需要的数据，内存占用，还有Vcores虚拟核数，CPU占用的一个虚拟量化指标）

如果Job配置了本地限制（即任务所需的优先需要加载本地HDFS资源，或者同一机架的HDFS副本），`ResourceManager`申请容器运行的节点时会优先申请存储有所需副本的节点，如果实在找不到再基于hadoop网络拓扑模型寻找当前机架的其他节点或者其他机架的节点，使得Job运行时所需要的数据尽量为本地数据，降低对集群带宽的依赖

#### 启动MrAppMaster

启动`Container`后，client会申请在这个“容器”中启动`MrAppMaster`，这个`MrAppMaster`读取Job执行文件，获取Job的配置文件和splits等信息，然后根据这些配置文件进行接下来的任务（直接运行任务，或者申请更多的`Container`并行启动任务）

根据splits规划，如果需要申请更多节点运行并行任务，`MrAppMaster`会向`ResourceManager`申请启动更多的`Container`，然后在这些`Container`中启动mapTask（或者reduceTask），这些task进程在Yarn环境中统一称为`YarnChild`

#### 启动Task

`MrAppMaster`启动完成后，根据splits启动多个mapTask，待mapTask均完成后，再根据该job配置的reduce数目启动多个reduceTask，启动流程与mapTask完全一样，Yarn并不关心具体执行的什么任务，它只需要接收`MrAppMaster`的资源分配请求然后申请启动相应数量的`Container`即可，启动完成后任务内部的交互也不由Yarn负责，当Job完成后再向client返回任务执行结果

### Yarn的特点

Yarn作为通用性很强的分布式计算资源调度框架，能够很好地和多种计算框架如MapReduce, Spark, Storm等进行集成，计算框架专注于计算逻辑的实现，Yarn则专注于集群资源的分配和调度

对于除了MapReduce以外的其他计算框架，把上述的`MrAppMaster`替换为任何一种Master进程，把mapTask或者reduceTask替换为任何一个work进程，对于Yarn来说都没有问题，只要实现了Yarn的规范和api，都可以在Yarn上面运行

