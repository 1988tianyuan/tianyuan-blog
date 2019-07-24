title: Reactor-Kafka（1）
author: 天渊
tags:
  - Kafka
  - reactor
categories:
  - 基础知识
date: 2019-02-02 18:10:00
---
reactor-kafka项目是遵循响应式流（Reactive Streams）规范的Kafka client，有Producer实现和Consumer实现。
<!-- more -->

### 首先讲讲Reactive Streams规范

在传统的编程范式中，我们一般通过迭代器（Iterator）模式来遍历一个序列。这种遍历方式是由调用者来控制节奏的，采用的是拉的方式。每次由调用者通过 next()方法来获取序列中的下一个值。

响应式流（Reactive Streams）规范则是推的方式，即常见的发布者-订阅者模式。当发布者有新的数据产生时，这些数据会被推送到订阅者来进行处理。在反应式流上可以添加各种不同的操作来对数据进行处理，形成数据处理链。这个以声明式的方式添加的处理链只在订阅者进行订阅操作时才会真正执行。

响应式流规范体现到Jdk中即为Java 8的Stream Api和Java 9的Flow Api，再结合Java 8的Lambda函数式编程模型，形成了独特的Reactive响应式异步编程模型，目前最重要的Reactive实现项目即为Pivatol维护的`project-reactor`，诸多项目基于`project-reactor`对原有项目进行了遵循响应式规范的重构，包括`WebFlux`和`Reactor-mongodb`，以及这里要介绍的`Reactor-Kafka`。

#### Flux和Mono

Flux和Mono是project-reactor中最重要的两个基本概念，可以把他们理解为发布订阅模型中的发布者，他们均实现了`org.reactivestreams.Publisher`接口，Mono表示的是包含 0 到 1 个发布者的异步序列，Flux 表示的是包含 0 到 N 个发布者的异步序列。

在Flux和Mono发布订阅序列中可以包含三种不同类型的消息通知：正常的包含元素的消息、序列结束的消息和序列出错的消息，分别包含`onNext()`, `onComplete()`和 `onError()`三个回调；当发布者产生需要消费的元素时，用户使用Stream流处理的方式对序列上的元素进行处理，比如`map()`或者`filter()`等等；最终调用`onSubscribe()`完成订阅。

如下图所示，通过Flux（或者Mono）将整个数据库调用链以响应式异步序列的方式贯通起来，再由reactive的web服务器（通常是reactive-netty）发布给用户：

![1548985987329](/blog/images/1548985987329.png)

整个过程均为非阻塞，吞吐量能得到极大提升。

### Reactor-Kafka

关于`Reactor-Kafka`，官方文档描述如下：

> [Reactor Kafka](https://projectreactor.io/docs/kafka/release/api/index.html) is a reactive API for Kafka based on Reactor and the Kafka Producer/Consumer API. Reactor Kafka API enables messages to be published to Kafka and consumed from Kafka using functional APIs with non-blocking back-pressure and very low overheads. This enables applications using Reactor to use Kafka as a message bus or streaming platform and integrate with other systems to provide an end-to-end reactive pipeline.

> Reactor-Kafka是基于Reactor和kafka Producer/Consumer API开发的kafka响应式API客户端，可以通过非阻塞的函数式API来发布消息到kafka并消费kafka的消息，性能消耗很低。这套API能够让那些使用Reactor标准库（即Flux和Mono）的应用程序像message bus或者streaming platform一样使用kafka，并且和其他系统集成，提供端到端的响应式管道。

也就是说这套Reactor-Kafka的API能够让使用者很轻易地将Kafka与其他使用Reactor api的系统集成起来，形成一套完整的响应式流处理管路。

#### 加入java依赖

maven：

```xml
<dependency>
    <groupId>io.projectreactor.kafka</groupId>
    <artifactId>reactor-kafka</artifactId>
    <version>1.1.0.RELEASE</version>
</dependency>
```

gradle:

```groovy
dependencies {
    compile "io.projectreactor.kafka:reactor-kafka:1.1.0.RELEASE"
}
```

#### reactor consumer api

Reactor-Kafka的consumer的核心api是`reactor.kafka.receiver.KafkaReceiver`

**属性配置**：

```java
Map<String, Object> consumerProps = new HashMap<>();
consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
```

对基本的bootstrapServers和groupId以及serializer等基本属性进行配置，其他的consumer属性参考Kafka官方文档；创建一个`ReceiverOptions`对象对属性进行封装：

```java
ReceiverOptions<String, String> receiverOptions = ReceiverOptions.<String, String>create(consumerProps())
    .subscription(Collections.singleton("resource-v1-TestProduct-TestType"));
```

`.subscription`方法可以订阅多个topic

**KafkaReceiver**：

`KafkaReceiver`是reactor-kafka的consumer核心api：

```java
Flux<ReceiverRecord<String, String>> recordFlux = KafkaReceiver.create(receiverOptions()).receive();
```

`FLux`对象用于多个发布者，也就是说在`KafkaReceiver`中对应多个topic进行消费，其中key和value均设置为`String`类型：

```java
recordFlux
	.log()	//打印日志
    .doOnNext(r -> r.receiverOffset().acknowledge()) //将当前record标记为已处理
    .map(ReceiverRecord::value) //将元素由ReceiverRecord对象替换为其value
    .doOnNext(r -> handler.saveResource(r)) //处理获得的record的value
    .doOnError(e -> log.warn("消费出错", e)) //处理过程中出错则打印日志
    .subscribe(); //启动订阅
```

在该调用链上设置多个回调函数，对发布的消息进行顺序消费；

该方法是非阻塞的，调用完后立即返回，不会阻塞用户线程，而是由reactor-kafka启动异步的EventLoop进行消息的获取并由worker线程调用用户设置的回调函数进行消费；

如果`KafkaReceiver`接收到某个topic的一条消息则会顺序地调用回调函数进行处理；

需要注意的是调用链最后都要调用`subscribe()`方法启动订阅，否则整个调用链并不会生效，并且一个`Flux`或者`Mono`对象只能订阅一次，如果多次订阅的话会报错：

```java
java.lang.IllegalStateException: Multiple subscribers are not supported for KafkaReceiver flux
```

如果不想马上结束整个流处理过程的话，可以不用立即调用`subscribe()`方法，而是将`Flux`对象作为一个载体传下去，这也就是官网提到的`use Kafka as a message bus or streaming platform and integrate with other systems to provide an end-to-end reactive pipeline`的意义所在，比如，我想将消费得到的String存入mongodb，则可以当前`Flux`对象传给`reactor-mongodb`，再由`reactor-mongodb`返回一个`Flux`对象，形成完整的Stream链：

```java
// 从kafka消费得到String数据
Flux<String> receivedStrFlux = KafkaReceiver
    .create(receiverOptions())
    .receive()
    .log()
    .doOnNext(r -> r.receiverOffset().acknowledge())
    .map(ReceiverRecord::value)
    .doOnError(e -> log.warn("消费出错", e));
// 将String经过一系列转换得到一个包含Resource对象的Flux
Flux<Resource> resourceFlux = receivedStrFlux
	.map(this::handleResourceString)
    .flatMap(Flux::fromIterable)
    .map(resourceData -> {
        Resource resource = new Resource();
        BeanUtils.copyProperties(resourceData, resource);
        return resource;
    });
// 将包含Resource对象的Flux通过reactor-mongodb进行存储
// 得到另一个Flux<Resource>
Flux<Resource> resourceSavedFlux = resourceRepository.saveAll(resourceFlux);
// 对这个Flux<Resource>进行订阅
resourceSavedFlux
	.doOnError(e -> log.warn("出错啦！", e))
    .doOnComplete(() -> log.info("都存完啦！"))
    .log()
    .subscribe();
```

以上就是一个完整的reactor-kafka+reactor-mongodb的异步响应式流处理链

#### reactor consumer api其他配置

除了基本的配置，reactor consumer api还可以进行一些进一步的设置

单独订阅序号为0的partition：

```java
receiverOptions = receiverOptions.assignment(Collections.singleton(new TopicPartition(topic, 0)));
```

当调用`.doOnNext(r -> r.receiverOffset().acknowledge())`时，该record的offset并不会立即提交，而是加入一个等待提交队列进行周期性自动提交，当然也可以调用`r.receiverOffset().commit()`方法手动提交该offset，`commit()`后依然返回的是一个`Mono`对象：

```java
.doOnNext(r -> r.receiverOffset().commit().doOnSuccess(aVoid -> log.info("offset提交成功！")).subscribe())
```

`KafkaReceiver`接收kafka信息除了使用`receive()`消费最新的record，还可以手动指定`offset`进行消费：

```java
ReceiverOptions.<String, String>create(consumerProps())	.addAssignListener(receiverPartitions -> receiverPartitions.forEach(ReceiverPartition::seekToBeginning))		.subscription(Collections.singleton("resource-v1-TestProduct-TestType"));
```

使用`seekToBeginning`在每次初始化后从头开始消费，或者直接指定offset：

```java
ReceiverOptions.<String, String>create(consumerProps())	.addAssignListener(receiverPartitions -> receiverPartitions.forEach(r -> r.seek(140))).subscription(Collections.singleton("resource-v1-TestProduct-TestType"));
```

从offset=140的位置开始消费

#### reactor-kafka-consumer的生命周期

每个`KafkaReceiver`实例的生命周期都跟对应的`Flux`相关，`Flux`结束消费则相应的`KafkaReceiver`就会被关闭.