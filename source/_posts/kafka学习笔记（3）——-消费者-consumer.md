title: kafka学习笔记（3）—— 消费者 consumer
author: 天渊
tags:
  - Kafka
  - 大数据
categories: []
date: 2019-03-18 13:21:00
---
kafka-consumer作为kafka消息队列的消费者端，通过接收topic的消息进行处理然后输出到下游数据源（数据库，文件系统，或者是另外的消息队列）<!--more-->

kafka-consumer最大的特点是，遵循发布/订阅模型，多个consumer共同订阅同一个topic，通过offset对消息的消费位置进行标记：

![upload successful](\blog\images\pasted-7.png)

**consumer的offset保存在哪里？**：老版本的kafka，offset保存在zookeeper中，但由于consumer的offset数据太过庞大，不适合存放在zookeeper，因此新版本的kafka单独维护了一个保存offset的topic：`__consumer_offsets`，以group-id，topic以及partition做为组合Key对每个consumer的offset进行检索

### consumer消费数据

consumer以consumer-group（消费者组）为单位向kafka订阅topic并消费数据：

![upload successful](\blog\images\pasted-8.png)

kafka consumer抓取数据基本流程图：

![upload successful](\blog\images\pasted-9.png)

几点特性：

- **partition主从热备**：消息的读/写工作全部交给leader partition来完成，slave partition主动与leader同步，通过LEO和leader epoch等信息同步数据，leader通过ISR列表保存当前存活的slave状态
- **可靠性**：只有当ISR列表中所有副本都成功收到的消息才能提供给consumer，可以提供给consumer的这部分消息由`high water`来进行控制 (设置`replica.lag.time.max.ms`参数可保证副本同步消息的时间上限)
- **零拷贝**：kafka通过零拷贝的方式将数据返回给consumer

#### consumer的java api

使用kafka-consumer高级api进行消费操作：

1. 创建单个consumer进行消费：

   ```java
   public class ConsumerGroupTest {
       public static void main(String[] args) {
           Map<String, Object> consumerConfig = new HashMap<>();
           consumerConfig.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "10.106.151.187:9092");
           consumerConfig.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
           consumerConfig.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
           consumerConfig.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group-1");
           runConsumer(consumerConfig);
       }
       private static void runConsumer(Map<String, Object> consumerConfig) {
           Consumer<String, String> consumer = new KafkaConsumer<>(consumerConfig);
           consumer.subscribe(Collections.singleton("test_multi_par_topic_1"));
           try {
               while (true) {
                   // poll拉取消息，最长等待时间10秒
                   ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(10));
                   records.forEach(record -> {
                       printRecord(record);
                       print("当前consumer实例：", consumer.toString());
                       System.out.println();
                   });
               }
           } finally {
               consumer.close();
           }
       }
       private static void printRecord(ConsumerRecord<String, String> record) {
           System.out.println("当前record：====> "
                              + " topic:" + record.topic()
                              + " partition:" + record.partition()
                              + " offset:" + record.offset()
                              + " value:" + record.value()
                             );
       }
       private static void print(String name, Object value) {
           System.out.println("KafkaConsumer ====> " + name + " is " + "{ " + value + " }");
       }
   }
   ```

   单个consumer实例可以订阅多个topic，也可以通过正则表达式订阅topic：

   ```java
   // 订阅多个topic
   consumer.subscribe(Arrays.asList("test-topic-1", "test-topic-2"));
   // 通过正则表达式订阅多个topic
   consumer.subscribe(Pattern.compile("test.*"));
   ```

2. 创建消费者组订阅包含多个partition的topic：

   创建一个叫做“test-group-1”的消费组，订阅“test_multi_par_topic_1” topic， 该topic包含三个partition：

   ```java
   public static void main(String[] args) {
       Map<String, Object> consumerConfig = new HashMap<>();
       consumerConfig.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "10.106.151.187:9092");
       consumerConfig.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
       consumerConfig.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
       consumerConfig.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group-1");
       runConsumerManualCommit(consumerConfig);
   }
   
   private static void runConsumer(Map<String, Object> consumerConfig) {
       Consumer<String, String> consumer = new KafkaConsumer<>(consumerConfig);
       consumer.subscribe(Collections.singleton("test_multi_par_topic_1"));
       try {
           while (true) {
               // poll拉取消息，最长等待时间10秒
               ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(10));
               records.forEach(record -> {
                   printRecord(record);
                   print("当前consumer实例：", consumer.toString());
                   System.out.println();
               });
           }
       } finally {
           consumer.close();
       }
   }
   ```
   
   首先只启动一个实例，可以看到，当前consumer实例消费了全部的三个partition的数据：
   
![upload successful](\blog\images\pasted-10.png)

    
   向该组加入一个consumer实例，可以看到，之前`@48322fe3`这个consumer实例只分到了partition2，而新加入的`@365ae362`这个实例分到了0和1：：
   
![upload successful](\blog\images\pasted-11.png) 

![upload successful](\blog\images\pasted-12.png)

继续加consumer实例进来，最早的元老实例1，分到了partition2：
 
![upload successful](\blog\images\pasted-13.png)
	
之后加入的实例2分到了partition0：

![upload successful](\blog\images\pasted-14.png)

最新加入的实例3 `@345dd14f`，分到了parition1：

![upload successful](\blog\images\pasted-15.png)
	
接下来将元老实例1`@48322fe3`停掉，观察实例2和实例3 （注意观察实例1的最后消费offset），实例1最后消费情况，消费完offset=85就挂掉了：
   
![upload successful](\blog\images\pasted-16.png)
   
观察实例2可以发现，实例2除了继续消费partition0，还认领了之前实例3消费的partition1的任务
  
![upload successful](\blog\images\pasted-17.png)

观察实例3，消费partition1直到offset=91的位置，发生了rebalance，之后partition1的任务分派给了上面实例2，自己分到了因实例1挂掉而无人认领的partition2，并且从实例1最后消费位置即offset=86开始消费
   
   
![upload successful](\blog\images\pasted-18.png)

### consumer-group

单个consumer-group即可以只消费一个topic的消息，也可以同时消费多个topic的消息

#### consumer-group订阅topic

单个consumer-group在订阅某个topic时，如果不特殊指定订阅某个partition，kafka将启用`Coordinator`对consumer-group内的consumer实例进行负载均衡，有以下特点：

1. 组内的一个consumer实例消费全部的partition：

	![upload successful](\blog\images\pasted-19.png)

2. 组内继续加入consumer实例，一同消费多个partition，消费过程与另外的consumer完全隔离不受影响：

	![upload successful](\blog\images\pasted-20.png)

3. 组内继续加入consumer，每个consumer实例单独消费一个partition，此时consumer数目与partition数目一致：

	![upload successful](\blog\images\pasted-21.png)

4. 继续加入consumer，数目超过partition数目，此时多出来的consumer将分不到partition进行消费：

	![upload successful](\blog\images\pasted-22.png)

如上可以看出，当单个consumer-group订阅topic进行消费时，kafka的Coordinator保证topic的数据能够被均匀分派到组内的各个消费者上，并且topic中的**单个partition只能被单个consumer-group中的其中一个consumer消费**

不过，如果有其他consumer-group的consumer参与消费这个topic，将与之前的consumer-group隔离，Coordinator将单独为这个新的组进行负载均衡：

   ![upload successful](\blog\images\pasted-23.png)
    
#### consumer分区再平衡

- 每当一个consumer-group有新成员加入，并且和老成员一起消费同一个topic的时候，Coordinator都将进行分区再平衡（`rebalance`），为了平衡消费能力，老成员消费的partition有可能会被分派给新加入的consumer
- 当有consumer实例挂掉时，之前分派给他的partition将会重新分派给剩余还活着的consumer（conusmer实例通过发送心跳让kafka集群知道他还活着）
- 订阅的topic新加入partition也会触发rebalance
- 通过正则表达式订阅某个类型的topic，当新加入该类型的topic时，也会发生rebalance

#### consumer rebalance的大致流程：

1. Topic/Partition的改变或者新Consumer的加入或者已有Consumer停止，将触发Coordinator注册在Zookeeper上的watch，Coordinator收到通知准备发起Rebalance操作。
2. Coordinator通过在HeartbeatResponse中返回IllegalGeneration错误码通知各个consumer发起Rebalance操作。
3. 存活的Consumer向Coordinator发送JoinGroupRequest
4. Coordinator在Zookeeper中增加Group的Generation ID并将新的Partition分配情况写入Zookeeper
5. Coordinator向存活的consumer发送JoinGroupResponse
6. 存活的consumer收到JoinGroupResponse后重新启动新一轮消费

### consumer配置

除了上述`bootstrap.servers`,`key.deserializer`,`value.deserializer`,`group.id`这几个配置项必须进行配置之外，其他的配置项均为可选项：

```properties
# 单次拉取消息的最小字节数，如果该值不为0，在consumer拉取消息时，如果可消费数据达不到该值，broker将等待一段时间(fetch.max.wait.ms)直到有足够数据或者等待超时，再返回给consumer
fetch.min.bytes=1024
# 单次拉取消息的最长等待时间
fetch.max.wait.ms=500
# 指定consumer单次poll()从分区中拉取的最大字节数，默认1MB
max.partition.fetch.bytes=1048576
# kafka集群判断consumer连接中断的最长等待时间，默认10秒，如果consumer超过该时间没有发送心跳，则会判断为死亡进而触发rebalance
session.timeout.ms=10000
# 心跳间隔时间，建议不超过session.timeout.ms的1/3
heartbeat.interval.ms=3000
# 是否开启自动提交offset，默认是开启的，如果开启，将每隔auto.commit.interval.ms的时间将消费完成但未提交offset的consumer统一向kafka集群提交一次（风险：当consumer消费完成但未提交offset就挂掉时，重启后将造成消息重复消费）
enable.auto.commit=true
# 自动统一提交offset的时间间隔
auto.commit.interval.ms=5000
# 开始消费的consumer-group（或者该consumer掉线已久，已经丢失有效的offset信息）初始化offset的策略，
# latest：默认值，默认从最新的有效offset开始消费
# earliest：从最早的有效offset开始消费
# none：如果kafka没有保存该consumer-group的offset信息则直接抛异常
auto.offset.reset=[latest, earliest, none]
# 分区分派策略类，kafka-clients自带的分区策略有Range和RoundRobin
# 必须是PartitionAssignor的实现类
partition.assignment.strategy=class org.apache.kafka.clients.consumer.RangeAssignor
# 单次poll()拉取的消息数量最大值，默认是500个
max.poll.records=500
```

### kafka consumer的offset提交策略

consumer每次消费数据完成后需要提交offset给集群以更新自己的消费状态，提交过程非常重要，直接关系到消费过程的稳定性，提交异常的话可能导致以下两种后果：

- 重复消费：一般发生在还没来得及提交offset即被中断消费的情况
  
  ![upload successful](\blog\images\pasted-24.png)

- 丢失消费：一般发生在消费过程有耗时操作的情况，如果在此期间提交了当前还没处理完成的消息的offset，会造成消费丢失：

  ![upload successful](\blog\images\pasted-25.png)

kafka提供了`手动commit`和`自动commit`两种提交策略：

1. **自动提交**：将`enable.auto.commit`设置为true并且将`auto.commit.interval.ms`设置为大于0的时间，即可开启自动提交，每次调用`consumer.poll()`方法时都会检查是否达到提交间隔时间，并将上一次拉取消息中最大的一个offset信息发送给集群中的`__consumer_offsets`这个topic

   缺点：自动提交不需要用户手动commit，因此提交过程不可控

2. **手动提交**：手动提交又分为`commitSync()`和`commitAsync()`两种方式，该模式下`enable.auto.commit`必须为false，consumer api任何情况都不会自动帮用户提交offset，一切由用户主动控制：

   `commitSync()`：同步提交，直接提交之前拉取的最新的offset，用户可自己保证在消息处理完毕后直接提交当前offset，缺点是整个提交过程是阻塞的，并且提交失败会进行重试

   `commitAsync()`：异步提交，同样也是提交之前拉取的最新的offset，改善了同步提交过程会产生阻塞的缺点，支持回调函数，提交过程不进行重试

比较好的手动提交方式是`同步和异步组合提交`：

```java
try {
    while (true) {
        // poll拉取消息，最长等待时间10秒
        try {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(10));
            records.forEach(record -> {
                printRecord(record);
                print("当前consumer实例：", consumer.toString());
                System.out.println();
            });
        } finally {
            // 消费过程未报错则执行异步提交，速度更快，不阻塞consumer线程
            consumer.commitAsync((offsets, exception) -> {
                // offsets中的`offsetAndMetadata`包含本次提交后希望处理的下一个offset
                print("本次提交后希望处理的下一个offset:", offsets);
                if (exception != null) {
                    exception.printStackTrace();
                }
            });
        }
    }
} catch (Exception e) {
    e.printStackTrace();
} finally {
    try {
        // 消费过程报错的话就尝试同步提交，防止未提交offset造成风险
        consumer.commitSync();
    } finally {
        consumer.close();
    }
}
```
### kafka consumer消费时指定offset

有些时候我们并不想按照默认情况读取最新offset位置的消息，kafka对此提供了查找特定offset的api，可以实现向后回退几个消息或者向前跳过几个消息：

```java
private static void runConsumerSeekOffset(Map<String, Object> consumerConfig, long specificOffset) {
    Consumer<String, String> consumer = new KafkaConsumer<>(consumerConfig);
    consumer.subscribe(Collections.singleton("test_multi_par_topic_1"));
    try {
        // 使用seek()方法将consumer的offset重置到指定位置
        consumer.assignment().forEach(topicPartition -> {
            consumer.seek(topicPartition, specificOffset);
        });
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(10));
            records.forEach(record -> {
                printRecord(record);
                print("当前consumer实例：", consumer.toString());
                System.out.println();
            });
        }
    } finally {
        consumer.close();
    }
}
```

### kafka consumer rebalance监听器

consumer和kafka的Coodinator交互的心跳心中包含是否需要rebalance信息，如果需要rebalance则停止拉取数据并直接提交offset，rebalance完成后consumer有可能失去对当前分配的partition的消费权

kafka提供了rebalance监听器用于让用户设置在rebalance发生后需要做的一些资源回收的操作，并且可以在监听器中手动提交当前已经处理完成的消息offset，以下是相对比较保险不会造成消费丢失的一种提交策略：

```java
private static void runConsumerRebanlanceListener(Map<String, Object> consumerConfig) {
    Consumer<String, String> consumer = new KafkaConsumer<>(consumerConfig);
    // 已处理的消息的offset暂存区
    final Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();
    consumer.subscribe(Collections.singleton("test_multi_par_topic_1"), new ConsumerRebalanceListener() {
        @Override
        public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
            System.out.println("发生rebalance！");
            System.out.println("当前分配的partitions：" + partitions);
		   // 将已经处理完成的offset进行同步提交
            consumer.commitSync(currentOffsets);
        }
        @Override
        public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
            System.out.println("rebalance完成！");
            System.out.println("rebalance后分配的partitions：" + partitions);
        }
    });
    try {
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(100);
            currentOffsets.clear();
            records.forEach(record -> {
                printRecord(record);
                print("当前consumer实例：", consumer.toString());
                // 当前处理完成，待提交的offset，存入暂存区
                currentOffsets.put(
                    new TopicPartition(record.topic(), record.partition()),
                    new OffsetAndMetadata(record.offset() + 1, null));
                System.out.println();
            });
            // 每隔批次消息处理完成后异步提交
            consumer.commitAsync(currentOffsets, null);
        }
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        try {
            // 出错后同步提交
            consumer.commitSync(currentOffsets);
        } finally {
            consumer.close();
        }
    }
}
```