title: Yarn任务调度机制探析
author: 天渊
tags:
  - yarn
  - hadoop
categories:
  - 大数据
date: 2019-06-04 20:54:00
---
yarn作为hadoop任务调度组件，具有良好的可扩展性、高可用性以及其独特的“多租户”特性，它为不同的作业场景提供了几种特性各异的任务调度机制：`FIFO调度器`，`Capacity调度器`，`Fair调度器`
<!-- more -->

### FIFO调度器

FIFO调度器采用一个先进先出队列对提交的任务执行顺序进行调度，按照提交的顺序首先为第一个任务的请求分配资源，第一个应用的请求被满足后，再依次为队列中下一个应用分配资源

FIFO调度器的好处是实现简单，无需任务额外配置，但是缺点也很明显，不适合那种大型任务和小型任务穿插执行的共享集群，因为大型任务可能会独占集群中的全部计算资源，并且任务执行时间会很长，yarn短时间为无法为其他任务分配资源，因此只能阻塞在队列中等待大型任务执行完成释放资源：

![1559481447073](\blog\images\1559481447073.png)

如图，横轴表示集群资源利用情况，当job1执行时，由于集群资源有限，job2必须等待job1执行完成后才能执行

### Capacity调度器

容量调度器可以为不同体量的任务提供一个或多个专用的等待队列，保证小型任务一旦提交就可以分配资源启动执行，因此解决了FIFO调度无法兼顾大型任务和小型任务的问题，大型任务的执行不会造成小型任务的长时间等待

不过容量调度器也有自己的缺点，由于yarn要专门为小型任务预留一部分集群资源，分配给大型任务的资源就会相应减少，执行时间也就变长了：

![1559482380158](\blog\images\1559482380158.png)

如图，queue B配置为小型任务服务，分配的资源较少，queue A配置为大型任务服务，分配的集群资源更多，保证大型任务和小型任务能够在集群中共存而不会相互阻塞

配置容量调度器时，可以根据实际需要配置多个队列，每个队列分配不同数额的集群资源，不过如果某个队列的任务在执行过程中分配的集群资源不够用，为了不让该任务等待其他队列释放资源，需要为队列设置`maximun-capacity`，能够在资源不够用时进行动态扩容，如果集群中有空闲资源，则会为这个队列分配更多的资源，这种方式称为`队列弹性`，扩容后的资源总量保证不超过`maximun-capacity`即可

提交map-reduce任务时，通过指定`mapreduce.job.<queue-name>`来指定当前任务分配给哪一个队列

### Fair调度器

即公平调度器，目的是为所有运行的任务公平分配集群资源，在容量调度器的基础上进行了改进，能够在不同任务之间**动态**地调度集群资源：

![1559568139040](\blog\images\1559568139040.png)

如图，在公平调度器模式下，与容量调度类似，根据实际需要分配多个队列用于执行任务：

1. job1率先提交，当前集群中没有其他任务共享资源，因为job1独享集群中queue A和queue B的全部资源

2. job1执行过程中，job2提交，此时job1享有queue A为其分配的资源，而job2享有queue B为其分配的资源，job1和job2共享集群资源

3. job2独享queue B的资源时，job3同样提交到queue B中执行，此时job2和job3共享queue B的资源

4. 待job2执行完成后，job3独享queue B的资源

可以看出，相比于容量调度器，公平调度模式下几乎不会出现饥饿情况（即有任务长期无法分配到集群资源而长时间处于阻塞状态），在满足一定的分配权重和调度策略的情况下，每个任务都能分享到一定数量的集群资源

hadoop默认使用容量调度器，如果要在yarn中启用公平调度器，需要在`yarn-site.xml`作以下配置

```xml
<property>
  <name>yarn.resourcemanager.scheduler.class</name>
  <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
</property>
```

#### Fair调度器的队列放置策略

与容量调度器相似，公平调度器也可以执行某个任务提交到特定的队列中执行，也可以执行任务放置到以任务提交的用户名为队列名的队列下进行执行

在yarn-site.xml中配置`yarn.scheduler.fair.allocation.file`执行队列分配文件，在队列分配文件中制定任务的队列分配策略(来自[hadoop官网yarn-FairScheduler文档](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/FairScheduler.html))：

```xml
<?xml version="1.0"?>
<allocations>
    <queue name="sample_queue">
        <minResources>10000 mb,0vcores</minResources>
        <maxResources>90000 mb,0vcores</maxResources>
        <maxRunningApps>50</maxRunningApps>
        <maxAMShare>0.1</maxAMShare>
        <weight>2.0</weight>
        <schedulingPolicy>fair</schedulingPolicy>
        <queue name="sample_sub_queue">
            <aclSubmitApps>charlie</aclSubmitApps>
            <minResources>5000 mb,0vcores</minResources>
        </queue>
        <queue name="sample_reservable_queue">
            <reservation></reservation>
        </queue>
    </queue>

    <queueMaxAMShareDefault>0.5</queueMaxAMShareDefault>
    <queueMaxResourcesDefault>40000 mb,0vcores</queueMaxResourcesDefault>

    <queue name="secondary_group_queue" type="parent">
        <weight>3.0</weight>
        <maxChildResources>4096 mb,4vcores</maxChildResources>
    </queue>

    <user name="sample_user">
        <maxRunningApps>30</maxRunningApps>
    </user>
    <userMaxAppsDefault>5</userMaxAppsDefault>

    <queuePlacementPolicy>
        <rule name="specified" />
        <rule name="primaryGroup" create="false" />
        <rule name="nestedUserQueue">
            <rule name="secondaryGroupExistingQueue" create="false" />
        </rule>
        <rule name="default" queue="sample_queue"/>
    </queuePlacementPolicy>
</allocations>
```

如上队列分配文件，分配了多个队列，每个队列还可在其内部指定多个叶子队列

最外层隐藏的最顶级队列是root队列，这里面配置的所有队列都是root队列的叶子队列，如果完全没有指定队列分配文件，则所有任务都会默认提交到root队列中执行

队列中可以指定最小和最大分配资源数以及最大可运行的任务数，权重weight值（为同层级队列指定资源分配比例），指定权限用户（aclSubmitApps，拥有这个权限的用户可以提交和杀死这个队列中的任务，需要注意root队列的acl是`*`，即每个用户都有权限），调度策略（schedulingPolicy，在一个队列中一共有三种调度策略），最重要的是队列放置策略`queuePlacementPolicy`：

```xml
<queuePlacementPolicy>
    <rule name="specified" create="true"/>
    <rule name="user" create="false" />
    <rule name="primaryGroup" create="false" />
    <rule name="nestedUserQueue">
        <rule name="secondaryGroupExistingQueue" create="false" />
    </rule>
    <rule name="default" queue="sample_queue"/>
</queuePlacementPolicy>
```

这是一个规则列表：

- 首先如果specified为true，当前提交任务若指定了队列名则放置于指定队列执行，否则进行下一级判断
- 如果user为true，则寻找以当前用户名为队列名的队列，若未创建则创建该队列；在这里user为false，就跳过这条判断
- 如果primaryGroup为true，则寻找以当前用户的主group名命名的队列
- nestedUserQueue：嵌套用户队列，与primaryGroup不同之处在于，user是应用到root队列的，而nestedUserQueue则会为任务寻找type为parent的队列，在这个队列下再去寻找有没有以任务用户名命名的队列
- secondaryGroupExistingQueue：寻找以用户的Secondary group名字命名的队列；上面的配置中，yarn会在parent队列下寻找符合要求的嵌套队列
- 最终如果没有找到匹配以上规则的队列，则执行提交到default队列，这里即为sample_queue队列

##### Fair调度器的三种调度策略

不同于FIFO调度器和容量调度器固定的调度策略，对于所有提交到某个队列中的任务，公平调度器为这个队列提供了三种调度策略，即：默认的`fair调度策略`，传统模式的`FIFO调度策略`，还有一种是`Dominant Resource Fairness(drf)策略`

##### 抢占

公平调度器中，当一个新的任务提交到一个队列，而该队列并没有空闲资源分配给它，这时该任务需要等待其他任务完成一部分container计算然后释放资源给新任务，以达到公平运行的目的

为了使作业从提交到执行所需的时间可控，可以设置抢占模式，当等待时间超过一定阈值时即启动抢占，强迫队列中其他任务立刻让出一部分资源给新任务，达到强行公平运行的目的

yarn-site.xml中将`yarn.scheduler.fair.preemption`设置为true即可打开抢占模式，并至少配置以下两个参数中的一个：

- 最小资源抢占超时时间`minSharePreemptionTimeout`：若指定等待时间内未获得承诺的最小共享资源则会启动抢占
- 公平资源抢占超时时间`fairSharePreemptionTimeout`：若指定等待时间内未获得承诺的公平共享资源则会启动抢占；承诺的公平共享资源由公平资源抢占阈值`fairSharePreemptionThreshold`和队列公平资源分配值的乘积决定，例如，当前队列一共提交了2个job，job1独占了队列资源，job2的公平资源理应为当前队列的0.5倍资源，若`fairSharePreemptionThreshold`为0.8，则承诺给这个任务的队列资源为0.4；该阈值默认是0.5

以上两个参数均可以设置root队列级别的默认值：`defaultFairSharePreemptionThreshold `，`defaultMinSharePreemptionTimeout `

##### 延迟调度

yarn的资源管理器为任务分配节点的原则是基于任务所需数据先本地后远程，本地如果有资源就优先分配本地节点，如果本地没有资源再寻找远程节点

不过有些时候稍微等待一些时间，待本地节点释放后就可以直接在本地启动任务了，不需要再寻找远程节点，这种行为称为`延迟调度`，容量调度器和公平调度器都支持这种方式

- 对于容量调度器，设置`yarn.scheduler.capacity.node-locality-delay`开启本地延迟，该值为正整数，表示等待本地资源释放期间最多错过多少个远程资源释放的机会，比如设置为3，则表示最多等待3次远程资源释放的信息后，如果本地节点的资源仍然没释放，就直接寻找远程节点的资源，不再等本地了
- 对于公平调度器，实现稍有不同，是将`yarn.scheduler.fair.locality.threshold.node`设置某个值，比如0.5，表示等待集群中最多半数节点给过资源释放信息后，再考虑远程节点，否则在这之前都将等待本地节点释放

##### 主导资源公平性

对于容量调度或公平调度，都是基于“资源”这一概念进行策略的，资源为内存或者cpu资源的抽象，两种调度模式都是基于某种资源的分配进行调度（内存或者cpu）

不过如果某些任务对于内存或者cpu的依赖各异，这时候分配起来就比较复杂了，往往需要`Dominant Resource Fairness(drf)策略`进行支持

**Dominant Resource Fairness(drf)，主导资源公平策略**：首先观察任务的主导资源（Dominant Resource）是内存还是cpu，选出主导资源，然后根据任务之间主导资源的占比来分配资源

例如：

- job1所需内存资源占集群总内存3%，所需cpu资源占集群总cpu1%，因此job1的主导资源是内存，占比3%
- job2所需内存资源占2%，所需cpu资源占6%，job2的主导资源是cpu，占比6%
- 因此job1和job2申请资源比例为`3% : 6%`，也就是1：2，job2分配的container数量为job1的两倍