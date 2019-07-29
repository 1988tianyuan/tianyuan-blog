title: Apahce Nifi学习 —— 何为Nifi?
author: 天渊
tags:
  - nifi
categories:
  - 大数据
date: 2019-07-29 21:49:00
---
Apache Nifi 是一个开源的分布式系统数据流自动化管理工具，官网：[https://nifi.apache.org](https://nifi.apache.org/)

Nifi全称是`Niagarafiles`，尼亚加拉files，从名字可以看出这个工具就是用于对付像尼亚加拉瀑布一样的大型数据流
<!--more-->

#### 为何要有Nifi ？

随着大数据生态的发展和物流网技术的兴起，企业越发重视对系统间数据流交互的可靠性以及稳定性的管理，但系统之间的数据流管理存在以下几个挑战：

1. **系统故障**：网络故障，磁盘故障，软件崩溃，或者人为犯错
2. **下游系统数据处理瓶颈**：下游系统都有数据处理上限，上游输入的数据量不能超过下游系统处理上限
3. **异常数据**：数据流中的异常数据是不可避免的（太大，太快，损坏的，错误的数据）
4. **业务系统升级改造**：业务系统升级，数据类型变更，需要做出快速响应
5. **系统升级引起的兼容问题**：多系统之间升级不同步引起数据兼容问题，此时如果系统之间耦合性太高的话会导致整个系统不可用
6. 其他诸如对外界环境变动的响应（法律法规变更，规章制度变更等）和生产环境平滑升级等问题

为了解决以上问题，Nifi应运而生。总的来说，需要Nifi这么一种自动化管理工具来为大型分布式系统提供数据流可靠性管理，数据清洗和过滤，数据流背压控制，或者系统数据接口解耦等功能

#### Nifi的特点

- **基于web的UI**：完全可视化设计，控制和监控
- **高性能**
  - 数据丢失容错vs保证交付
  - 低延迟和高吞吐量
  - 动态优先级
  - 流可以在运行时修改
  - 背压(Back pressure)
- **数据流跟踪**
  - 从始至终的数据流追踪机制
- **高可扩展性**
  - 定制化Processor
  - 支持快速开发和有效测试
- **安全**
  - 支持SSL,SSH,HTTPS加密等
  - 多租户授权和内部授权/策略管理

#### Nifi的核心概念

| 术语               | 对应Flow Based Programming的术语 | 描述                                                         |
| ------------------ | -------------------------------- | ------------------------------------------------------------ |
| FlowFile           | Information Packet               | Nifi中的基本数据流对象，通过`attribute`来标记各个数据流对象的状态 |
| FlowFile Processor | Black Box                        | 负责数据流的清洗，过滤，提取和切割等操作，还可以实现数据路由的合并，变换，及系统间的各种协调 |
| Connection         | Bounded Buffer                   | 连接器为各个处理器提供连通管道，以队列的形式提供优先级设置（先进先出等，优先队列等），还提供背压设置和数据负载均衡设置 |
| Flow Controller    | Scheduler                        | 流控制器维护处理器之间的连接关系，管理和分配所有处理器使用的线程 |
| Process Group      | subnet                           | 处理器组，一系列Processor和Connection的集合，通过输入端口（input）接收数据，通过输出端口（output）发送数据 |

以国外某个博主画的图为例（[源地址](https://www.freecodecamp.org/news/nifi-surf-on-your-dataflow-4f3343c50aa2/)）：

![1563333234565](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563333234565.png)



#### Nifi的架构

##### 单机架构

Nifi运行在Jvm中，由`Web Server`，`Flow Controller`，`FlowFile Repository` ，`Content Repository`，`Provenance Repository`等组件组成：

![NiFi Architecture Diagram](https://nifi.apache.org/docs/nifi-docs/html/images/zero-master-node.png)

##### 集群架构

Nifi也支持集群架构模式，多节点共同协调处理，每个节点处理逻辑相同，不过各节点分配了不同的数据集。集群中由zookeeper选举出一个`Cluster Coordinator`集群节点协调者，用户在web ui上对Nifi组件的所有改动都由`Cluster Coordinator`通知给所有节点

另外集群中存在一个`Primary Node`主节点，也由zookeeper选举得到，主要用途是提供`Isolated Processor`的配置，在主节点中定义的`Isolated Processor`只在这个主节点上工作，其他节点没有这个Processor。适用场景是提供一个统一的外部数据拉取的接口，然后再通过一些分派机制（loadBalance等）将数据统一分派给剩余节点进行处理，避免多节点同时拉取外部数据发生数据竞争而产生不可预测的后果

![NiFi Cluster Architecture Diagram](https://nifi.apache.org/docs/nifi-docs/html/images/zero-master-cluster.png)



#### Nifi的基本用法（一个简单的例子）

Nifi提供了几十种Processer，用不同的处理逻辑处理不同场景不同格式的数据，具体提供的组件可以参考文档：<https://nifi.apache.org/docs/nifi-docs/>

在这里仅以我做过的一个实际项目为例来进行说明

##### 准备Nifi环境

安装过程就不再赘述了，安装完成后访问任何一个节点的8080端口都能打开web ui，Nifi后续所有操作都在web ui上进行：`http://anynode:8080/nifi/`

进入web ui，长这样，目前已经创建了多个Processor Group：

![1563333859182](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563333859182.png)

##### 创建Processor Group

Processor Group可以将一系列独立的数据处理逻辑封装在一起，与其他Group进行隔离

将上方的图标拖动到下方界面进行创建

![1563334045695](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563334045695.png)

创建完成后，Group的界面能够显示当前状态，打开关闭情况，输入输出情况和队列缓存情况，双击进入Group面板

##### 创建Processor —— kafka consumer

Processor是数据流处理的核心控件，可以接收输入，进行路由中转并发送给外部数据源，现在以一个Kafka Consumer Processor为例来进行说明，拖动上方Processor图标到面板中进行创建：

![1563334687067](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563334687067.png)

我们选用`ConsumerKafka_0_11`用来消费外部kafka集群中的数据，采用kafka 0.11版本的Consumer Api实现：

![1563334861136](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563334861136.png)

ConsumerKafka_0_11这个Processor支持从kafka消费数据作为输入流，将数据封装为FlowFile进行后续的处理：

![1563342367031](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563342367031.png)

双击Processor面板进行配置，设置面板一共包含四个tab，第一个tab用于一些基本设置：

###### 基础settings

![1563343520868](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563343520868.png)

需要重点关注`Automatically Terminate Relationships`，Relationship表示该Processor对应的输出（即Connection），如果某个Relationship未被勾选，表示无法自动处理，需要用户手动指定Connection指向下游的其他Processor；如果被勾选，那么这个Relationship是可以自动终结的，无需用户指定，Nifi就帮你处理了

###### scheduling配置

第二个tab为scheduling配置，用来指定Processor处理数据的周期和触发处理的规则，默认是基于时间周期性调度，也可以指定Cron表达式进行固定时间的调度：

![1563343869114](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563343869114.png)

###### properties配置

配置Processor的属性，每个Processor都有自己独有的属性设置，并且还可以自己指定attribute用于后续的处理，

![1563343926871](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563343926871.png)

这里面做了一些基本配置，包括kafka集群地址，订阅的topic，group-id等

最后一个comments tab作为备注使用

##### 创建下游Processor ——  EvaluateJsonPath

上游kafka consumer processor获取数据后封装为FlowFile发往下游进行处理，我这里使用的数据结构是json，因此下游处理器采用EvaluateJsonPath解析json数据，将其中的一些值提取出来包装为attribute再发往下游：

![1563344614818](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563344614818.png)

Nifi专门提供了JsonPath方式抽取Json数据

两个Processor创建完成后，需要一个Connection将他们连接起来，直接点击ConsumerKafka_0_11上面的箭头并拉到EvaluateJsonPath上就可以进行两个Processor之间的Connection的配置：

![1563344757201](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563344757201.png)

指定Relationships，表示`ConsumerKafka_0_11成功获取到Kafka数据后，将FlowFile发送给下游的EvaluateJsonPath去处理`：

![1563344893717](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563344893717.png)

##### 创建下游Processor —— RouteOnAttribute

提取了Json数据，我们想根据提取出来的数据进行路由，这时候就需要RouteOnAttribute这个Processor：

![1563345038335](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563345038335.png)

Nifi提供了Expression Language表达式来进行一些简单的编程操作

使用之前抽取的json字段action作为路由依据，以“create”开头的action则返回true，否则返回false

然后创建EvaluateJsonPath到RouteOnAttribute之间的Connection

##### 创建下游Processor —— InvokeHttp

配置好路由后，我们想将带“create”开头action的数据发送给http接口A，将不带“create”的数据发送给http接口B，那么就得创建两个InvokeHttp，进行一些基本配置包括method和url：

![1563345522963](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563345522963.png)

然后从RouteOnAttribute创建两个连接到这两个InvokeHttp处理器，两个连接分别是isCreation和unmatched（即当前数据不带“create”）：

最终我们的数据处理流构建完成了，这也是一个有向无环图，此时每个Processor都处于停止状态（只有停止状态才能修改配置）：

![1563345754157](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563345754157.png)

##### 启动Processor Group

返回Processor Group处理界面，直接启动整个Group （当然也可以每个Processor分别启动）

![1563346961196](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563346961196.png)

可以看到，整个处理流在5分钟内接收了三条数据，其中2条数据匹配了"isCreation"，发往了第一个接口A，其余两条数据发往了接口B，实现了自定义数据路由

##### 调节背压管路

如果某个Processor处理能力有限，就会造成上游数据积压。Nifi的Connection组件提供了一个队列用以缓冲下游暂时处理不了的数据，默认是10000条数据或者1GB size， 并且可以指定过期时间

关闭ConsumerKafka和EvalueteJsonPath这两个Processor，双击"success"这个Connection进行配置：

![1563347524271](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563347524271.png)

#### 查看Nifi日志

Nifi日志大体上分为**数据日志（Data Provenance）**和**事件日志（Bulletin）**，点击web ui右上角即可查看这两种日志：

![1563347976805](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563347976805.png)



##### Data Provenance

Nifi为每一条进入数据处理流的FlowFile都生成了一个唯一的uuid，可以查看每一条数据在各个节点处理的时间，处理类型（SEND，ROUTE，RECEIVE等），数据大小，以及处理的服务器节点，：

![1563348137310](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563348137310.png)

##### Bulletin

事件日志用于记录Nifi处理器处理数据遇到的一些问题，默认级别是WARN，在这里可以看到数据处理失败的一些信息以及对应的数据uuid和响应的Processer，便于结合`Data Provenance`跟踪追溯：

![1563348249531](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1563348249531.png)