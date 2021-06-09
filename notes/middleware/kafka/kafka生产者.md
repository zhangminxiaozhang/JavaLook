### 一、创建一个生产者

- 首先创建了一个 `Properties` 对象

- 使用 `StringSerializer` 序列化器序列化 key / value 键值对

- 在这里我们创建了一个新的生产者对象，并为键值设置了恰当的类型，然后把 `Properties` 对象传递给他。

```java
private Properties properties = new Properties();
properties.put("bootstrap.servers","broker1:9092,broker2:9092");
properties.put("key.serializer","org.apache.kafka.common.serialization.StringSerializer");
properties.put("value.serializer","org.apache.kafka.common.serialization.StringSerializer");
Producer<String, byte[]> producer = new KafkaProducer<String,String>(properties);
```

### 二、消息发送

- **简单消息发送**

```java
ProducerRecord<String,String> record =new ProducerRecord<String, String>("CustomerCountry","West","France");
producer.send(record);
```

这里调用的`ProducerRecord`的构造函数是

```java
public ProducerRecord(String topic, K key, V value) {}
```

发送成功后，`send()` 方法会返回一个 `Future(java.util.concurrent)` 对象，`Future` 对象的类型是 `RecordMetadata`类型，上面这段代码没有考虑返回值，所以没有生成对应的 `Future` 对象，所以没有办法知道消息是否发送成功，类似`UDP`传输

- **同步消息发送**

```java
ProducerRecord<String,String> record =new ProducerRecord<String, String>("CustomerCountry","West","France");
try{  
  RecordMetadata recordMetadata = producer.send(record).get();
}catch(Exception e){
  e.printStackTrace()；}

```

调用 `get()` 方法等待 Kafka 响应。如果服务器返回错误，`get()` 方法会抛出异常，如果没有发生错误，我们会得到 `RecordMetadata` 对象，可以用它来查看消息记录。

缺点：同步发送消息在同一时间只能有一个消息发送，这会造成许多消息无法直接发送，造成消息滞后，无法发挥效益最大化。

- **异步消息发送**

```java
ProducerRecord<String, String> producerRecord = new ProducerRecord<>("CustomerCountry", "Huston", "America");        producer.send(producerRecord,new DemoProducerCallBack());
```

其中的`DemoProducerCallBack`是继承`CallBack`接口，如果发生了异常，打印异常信息

```java
class DemoProducerCallBack implements Callback {
  public void onCompletion(RecordMetadata metadata, Exception exception) { 
    if(exception != null){     
      exception.printStackTrace();;  
    } 
  }
}
```

### 三、分区策略

Kafka 为我们提供了默认的分区策略，同时它也支持你自定义分区策略。如果要自定义分区策略的话，你需要重写生产者端的参数

```java
public interface Partitioner extends Configurable, Closeable {    
  public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);
  public void close();  
  default public void onNewBatch(String topic, Cluster cluster, int prevPartition) {};
}
```

上述中`topic`表示需要传递的主题；`key`表示消息中的键值；`keyBytes`表示分区中序列化过后的key，`byte`数组的形式传递；`value` 表示消息的 value 值；`valueBytes` 表示分区中序列化后的值数组；`cluster`表示当前集群的原数据。

##### 默认的分区策略如下：

- **顺序轮询**

消息是均匀的分配给每个 partition，即每个分区存储一次消息。

<div align="center">  <img src="https://i.bmp.ovh/imgs/2021/06/b32187fc549f004a.png" width="600"/> </div><br>

- **随机轮询**

随机的向 partition 中保存消息

<div align="center">  <img src="https://i.bmp.ovh/imgs/2021/06/d3456a3a06bf9e8c.png" width="600"/> </div><br>

实现的代码如下:  先计算出该主题总的分区数，然后随机地返回一个小于它的正整数。

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return ThreadLocalRandom.current().nextInt(partitions.size());
```

- **按照 key 进行消息保存**

同一个 Key 的所有消息都进入到相同的分区里面，由于每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略

<div align="center">  <img src="https://i.bmp.ovh/imgs/2021/06/bd0cceea5157787b.png" width="600"/> </div><br>

实现代码如下：

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return Math.abs(key.hashCode()) % partitions.size();
```

### 四、生产者的一些参数

**acks**：指定了要有多少个分区副本接收消息，生产者才认为消息是写入成功的。此参数对消息丢失的影响较大

1.如果 acks = 0，就表示生产者也不知道自己产生的消息是否被服务器接收了，它才知道它写成功了。如果发送的途中产生了错误，生产者也不知道，它也比较懵逼，因为没有返回任何消息。

2.如果 acks = 1，只要集群的 Leader 接收到消息，就会给生产者返回一条消息，告诉它写入成功。如果发送途中造成了网络异常或者 Leader 还没选举出来等其他情况导致消息写入失败，生产者会收到错误消息，这时候生产者往往会再次重发数据。因为消息的发送也分为同步和 异步，Kafka 为了保证消息的高效传输会决定是同步发送还是异步发送。如果让客户端等待服务器的响应（通过调用 `Future` 中的 `get()` 方法），显然会增加延迟，如果客户端使用回调，就会解决这个问题。

3.如果 acks = all，这种情况下是只有当所有参与复制的节点都收到消息时，生产者才会接收到一个来自服务器的消息，但是延迟较高。



**retries**：决定了生产者可以重发的消息次数，如果达到这个次数，生产者会放弃重试并返回错误。默认情况下，生产者在每次重试之间等待 100ms



**batch.size**：多个消息需要被发送到同一个分区时，生产者会把它们放在同一个批次里。该参数指定了一个批次可以使用的内存大小，当批次被填满，消息就会就绪，并不需要所有的批次都被填满就可以发送