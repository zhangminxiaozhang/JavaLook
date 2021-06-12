### 一、消费组

`KafkaConsumer`订阅主题并且接收消息，但是单个消费者消费的速度可能跟不上生产的速度，需要多个消费者参与主题的消费。

<div align="left">  <img src="https://i.bmp.ovh/imgs/2021/06/fd97cdf4c3620988.png" width="400"/> <img src="https://i.bmp.ovh/imgs/2021/06/a4a18772012d1c0b.png" width="400"/> </div><br>

每个分区所产生的消息能够被每个消费者群组中的消费者消费，如果向消费者群组中增加更多的消费者，那么多余的消费者将会闲置。Kafka 一个很重要的特性就是，只需写入一次消息，可以支持任意多的应用读取这个消息。换句话说，每个应用都可以读到全量的消息。为了使得每个应用都能读到全量消息，应用需要有不同的消费组。

<div align="left">  <img src="https://i.bmp.ovh/imgs/2021/06/39212a42bcba1735.png" width="400"/> <img src="https://i.bmp.ovh/imgs/2021/06/d53f01d186c6b595.png" width="350"/> </div><br>

- 一个消费者群组消费一个主题中的消息，这种消费模式又称为`点对点`的消费方式，点对点的消费方式又被称为消息队列
- 一个主题中的消息被多个消费者群组共同消费，这种消费模式又称为`发布-订阅`模式

### 二、消费者重平衡

**什么时候会发生冲平衡**：

- 当Topic发生变化时，比如新的Partition加入Topic，分区和消费者会被重新分配；
- 当消费者群组发生变化时，比如某个消费者因意外因素导致下线，它离开群组之后，原本由它消费的Partition会被交由群组内其它消费者消费。这是因为消费者会在轮询消息时或者提交偏移量时发送心跳，如果消费者停止发送心跳的时间足够长，会话过期，群组协调器认为它已经死亡，因而会触发分区再均衡。

<div align="center">  <img src="https://i.bmp.ovh/imgs/2021/06/43b29c43b5dc5d30.png" width="600"/> </div><br>

**重平衡的影响：**

- 为消费者群组带来了`高可用性` 和 `伸缩性`，我们可以放心的添加消费者或移除消费者；
- 在重平衡期间，消费者无法读取消息，造成整个消费者组在重平衡的期间都不可用。另外，当分区被重新分配给另一个消费者时，消息当前的读取状态会丢失，它有可能还需要去刷新缓存，在它重新恢复状态之前会拖慢应用程序。

### 三、创建一个消费者

和生产者类似，`group.id` 这个属性不是必须的，它指定了 KafkaConsumer 是属于哪个消费者群组。创建不属于任何一个群组的消费者也是可以的

```java
Properties properties = new Properties();       
properties.put("bootstrap.server","192.168.1.9:9092");     properties.put("key.serializer","org.apache.kafka.common.serialization.StringSerializer");   properties.put("value.serializer","org.apache.kafka.common.serialization.StringSerializer");
KafkaConsumer<String,String> consumer = new KafkaConsumer<>(properties);
```

### 四、订阅主题

```java
consumer.subscribe(Collections.singletonList("customerTopic"));
```

参数传入的是一个正则表达式，正则表达式可以匹配多个主题，如果有人创建了新的主题，并且主题的名字与正则表达式相匹配，那么会立即触发一次**重平衡**，消费者就可以读取新的主题。

下述是订阅test开头的所有主题

```java
consumer.subscribe("test.*");
```

### 五、轮询请求数据

```java
try {  
  while (true) {  
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(100));  
    for (ConsumerRecord<String, String> record : records) {   
      int updateCount = 1;    
      if (map.containsKey(record.value())) {    
      updateCount = (int) map.get(record.value() + 1);}
      map.put(record.value(), updateCount);}}}
finally {  
  consumer.close();}

```

通过轮询的方式向 Kafka 请求数据，其中poll()中的时间是一个超时时间，如果在规定的时间内没有返回数据，认为这个consumer挂了，触发重平衡，将这个分区交给消费组里面的其他消费者去操作。poll()返回一个记录列表。每条记录都包含了记录所属主题的信息，记录所在分区的信息、记录在分区中的偏移量，以及记录的键值对。我们一般会遍历这个列表，逐条处理每条记录。

### 六、特殊偏移

消费者在每次调用`poll()` 方法进行定时轮询的时候，会返回由生产者写入 Kafka 但是还没有被消费者消费的记录，因此我们可以追踪到哪些记录是被群组里的哪个消费者读取的。通过消费者向`_consumer_offset` 的特殊主题中发送消息，这个主题会保存每次所发送消息中的分区偏移量，主要作用就是消费者触发重平衡后记录偏移使用的，消费者每次向这个主题发送消息，正常情况下不触发重平衡，触发重平衡后，消费者停止工作，每个消费者可能会分到对应的分区，这个主题就是让消费者能够继续处理消息所设置的。

- 自动提交：如果 `enable.auto.commit` 被设置为true，每过`auto.commit.interval.ms`，消费者会自动把从 poll() 方法轮询到的最大偏移量提交上去
- 提交当前偏移量：把 `auto.commit.offset` 设置为 false，可以让应用程序决定何时提交偏移量，使用 `commitSync()` 提交偏移量。这个 API 会提交由 poll() 方法返回的最新偏移量，提交成功后马上返回，如果提交失败就抛出异常
- 异步提交：异步提交 `commitAsync()` 与同步提交 `commitSync()` 最大的区别在于异步提交不会进行重试，同步提交会一致进行重试



https://cloud.tencent.com/developer/article/1547380