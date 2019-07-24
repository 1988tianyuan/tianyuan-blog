title: kafka学习笔记（2）—— 生产者 producer
author: 天渊
tags:
  - Kafka
  - 大数据
categories:
  - 基础知识
date: 2019-03-18 12:43:00
---
kafka作为大数据日志收集系统，能够接收来自多端的生产者数据，下图是消息经由`kafka-producer-api`向kafka集群发送消息的基本过程：
<!--more-->

![upload successful](\blog\images\pasted-3.png)

下面用kafka-producer的java api来进行说明


### kafka-producer java api

#### KafkaProducer基本操作

项目中引入以下依赖：

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.1.1</version>
</dependency>
```

最新版本的kafka-producer-api取消了同步发送消息的模式，全部默认采用异步发送消息，使用异步线程从发送队列中批量发送消息，然后返回一个`Future`对象

首先创建producer并进行配置，以下三个配置项是必选配置，其他配置都是可选配置：

```java
Map<String, Object> producerConfig = new HashMap<>();
producerConfig.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "10.106.151.187:9092");
producerConfig.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
producerConfig.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
Producer<String, String> producer = new KafkaProducer<>(producerConfig);
```

构建record并发送，发送成功后返回topic和partition的元数据：

```java
ProducerRecord<String, String> record = new ProducerRecord<>("test-topic-1", "it's a msg");
Future<RecordMetadata> future = producer.send(record);
RecordMetadata recordMetadata = future.get();
System.out.println("offset:" + recordMetadata.offset());
System.out.println("partition id: " + recordMetadata.partition());
System.out.println("topic: " + recordMetadata.topic());
```

也可以设置回调函数，对Future进行消费：

```java
CountDownLatch latch = new CountDownLatch(1);
producer.send(record, (recordMetadata, e) -> {
    System.out.println("offset:" + recordMetadata.offset());
    System.out.println("partition id: " + recordMetadata.partition());
    System.out.println("topic: " + recordMetadata.topic());
    if (e != null) {
        e.printStackTrace();
    }
    latch.countDown();
});
latch.await();
```

#### KafkaProducer配置解析

除了`bootstrap.servers`, `key.serializer`,`value.serializer` 这三个配置项是必选配置，其他配置都是可选的：

```properties
# 指定目标分区有多少个副本成功收到消息时，producer才会收到消息发送成功的响应
# 1：leader节点成功收到消息后即认为消息发送成功，一般采用这个
# all：所有replica节点都成功收到消息后才认为消息发送成功
# 0：producer无需等待任何发送成功的响应，消息发送完毕后即返回
acks=1
# 生产者缓冲区大小，消息发送到broker前可以在producer内存中进行缓冲，如果待发送的消息大小超过该值，后续发送请求则会阻塞
buffer.memory=33554432
# 和上述配置协同工作，当缓冲区不足时后续请求能够阻塞的最大时间，超过该值仍然阻塞则会抛异常
max.block.ms=60000
# 消息压缩方式
# snappy：cpu消耗低，性能好； gzip：cpu消耗高，压缩比较snappy更高
compression.type=snappy
# 消息发送失败后的重试次数（部分错误像“消息太大”之类的错误默认不重试直接报错）
retries=3
# 消息发送失败后每次重试的间隔时间
retry.backoff.ms=100
# 消息发送的单个batch大小，把多个消息合并为一个请求可以提高网络利用率，提高吞吐量，但也某种程度造成了消息延迟
batch.size=16384
# 同样服务于消息batch，producer将等待直到消息填满一个batch或者达到linger时间后直接进行发送该批次消息
linger.ms=5
# producer允许的未返回响应的最大请求个数，如果为1，则producer在未收到当前请求的响应前不会发送后续请求
# 调高该值可以提高吞吐量，前提是对消息发送顺序没要求（如果开启retry的话有可能打乱消息发布的顺序）
max.in.flight.requests.per.connection=5
# producer单次发送的最大容量，保证这个值不超过broker的message.max.bytes属性即可
max.request.size=1048576
# 发布消息的请求响应超时时间，超出该值要么重试要么报错
request.timeout.ms=30000
```

关于kafka-producer配置的一些问题如下：

1. 如何保证消息发布顺序：如果同时配置了`retries`和`max.in.flight.requests.per.connection`，当后者大于1时，有可能造成消息发布乱序（比如，消息1发布失败，消息2发布成功，紧接着重试发送消息1并成功），所以官方建议如果配置了大于0的`retries`，`max.in.flight.requests.per.connection`最好设置为1

2. 幂等消息：为了避免消息重复发布，支持单个producer对于同一个`Topic,Partition`的`Exactly Once`语义，kafka在`0.11.0.0`后引入了对幂等消息的支持，通过以下配置进行开启：

   ```properties
   # 开启幂等producer
   enable.idempotence=true
   # 如果开启幂等producer，必须对以下配置进行如下的设置
   acks = all
   retries = （大于0）
   max.inflight.requests.per.connection = （小于等于5）
   ```

3. 事务支持：kafka同时在`0.11.0.0`版本引入了事务支持，支持跨partition幂等发布消息，保证了跨分区发布消息的原子性，通过以下配置进行开启：

   ```properties
   # 设置一个字符串表示事务id
   # 开启事务后，enable.idempotence默认设置为true
   transactional.id=my_tx_id_1
   ```


#### Kafka-producer序列化器

Kafka消息队列没有规定具体的消息传输协议和消息格式，队列中统一传输二进制数据流，因此需要用户根据自己消息的协议和格式选取合适的序列化器，或者自定义序列化器

Kafka默认提供的常用序列化器有有以下几种，基本上只能用于基本数据类型和字节数组或者ByteBuffer对象的序列化：

```properties
StringDeserializer
IntegerSerializer
ByteArraySerializer
ByteBufferSerializer
DoubleSerializer
UUIDSerializer
```

用户自定义序列化器直接实现`Serializer`接口即可，如下实现一个简单的Json格式的序列化器：

```java
public class SimpleJsonSerializer implements Serializer {
	private final Gson gson = new Gson();
	@Override
	public void configure(Map configs, boolean isKey) {
	}
	@Override
	public byte[] serialize(String topic, Object data) {
		String json = gson.toJson(data);
		return json.getBytes(Charset.defaultCharset());
	}
	@Override
	public void close() {
	}
}
```

然后在producer中进行配置，即可将java对象序列化为json字节数组了：

```java
Map<String, Object> producerConfig = new HashMap<>();
producerConfig.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "10.106.151.187:9092");
producerConfig.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
producerConfig.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, SimpleJsonSerializer.class);
producerConfig.put(ProducerConfig.ACKS_CONFIG, "1");
Producer<String, Person> producer = new KafkaProducer<>(producerConfig);

ProducerRecord<String, Person> record = new ProducerRecord<>("test-topic-1", Person.builder().id(0).name("liugeng").build());

```

#### kafka-producer分区器

producer发送消息时无需手动指定发送到某个partition，producer-api默认的`DefaultPartitioner`会根据消息中有无设置key来进行分区操作：

1. key不为null：计算key的hash值，然后和partition数量取模，映射到不同的partition中
2. key为null：使用Round-Robin进行轮询映射

如果需要自己实现分区器（例如需要根据key指定特定的分区，或者某个key的消息需要占用多个分区），可以实现`Partitioner`接口，以下是一个例子，将key为`last`的record映射到最后一个partition上，剩下的record通过hash映射到其他partition：

```java
public class SimpleCustomerPartitioner implements Partitioner {
	@Override
	public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
		List<PartitionInfo> partitionInfoList = cluster.availablePartitionsForTopic(topic);
		int partitionNum = partitionInfoList.size();
		if (keyBytes == null || !(key instanceof String)) {
			throw new InvalidRecordException("the key is necessary !");
		}
		if ("last".equals(key)) {
			return partitionNum - 1;
		}
		return Math.abs(Utils.murmur2(keyBytes)) % (partitionNum -1);
	}
	@Override
	public void close() {
	}
	@Override
	public void configure(Map<String, ?> configs) {
	}
}
```

分区器的`partition`方法返回指定partition的id，需要注意的是partition的id是从0开始的
