Kafka 权威指南

***Chapter1. 初识Kafka***
   * 发布与订阅消息系统
      * 耦合->解耦
   * Kafka登场
      * Record & Batch
      * Pattern
      * Topic & Patition
      * Producer & Consumer
      * Borker & Cluster
      * Multi-cluster
         * Kafka的消息复制机制只能在单个集群中进行，不能在多个集群中进行
   * 为什么选择Kafka
      * 多个Producer
      * 多个Consumer
      * 基于磁盘的数据存储
         * 可以重新消费，可以消费旧的数据
      * 伸缩性
      * 高性能
   * 数据生态系统
      * 使用场景
         * 活动跟踪
         * 传递消息
         * 度量指标和日志记录
         * 提交日志
         * 流处理
   * 故事起源
      * Linkedin ： XML， JSON
      * 开源
      * 命名
         * 作家Kafka


***Chapter 2: 安装Kafka***
   * 要是先行
      * OS、Java
      * Zookeeper
         * clientPort：2181
         * peerPort：2888
         * leaderPort：3888
      * kafka Broker
   * Broker配置
      * 常规配置
         * broker.id 唯一标志符
         * port 9092 端口
         * zookeeper.connect： localhost:2181/path
         * log.dirs : broker数据
         * num.recovery.threads.per.data.dir: 线程池来处理日志文件
         * auto.create.topics.enable：自动创建topic
      * topic的默认配置
         * num.partitions: 新创建topic的分区数
            * 吞吐量是？每个分区最大的吞吐量？每个broker包含的分区数、可用的磁盘空间和网络带宽？
            * 单个broker对于分区个数是有限制的，因为分区数越多，占用的内存就愈大，完成leader的选举所需要的时间也越长
         * log.retention.ms : 根据时间来决定数据被保留多久，
            * log.retention.hour, log.retention.minutes: 取三者的最小值
            * 检测磁盘文件的最后修改时间，特例：管理工具移动的数据
         * log.retention.bytes
            * 如果log.retention.ms 和log.retention.bytes 同时指定，只要有一个条件满足就执行清理
         * log.segment.bytes: 单个segment文件超过该配置时，就会开启一个新的文件
            * 如果配置的太小，就会频繁的关闭和分配新文件，从而降低磁盘写入的整体效率
         * log.segment.ms
            * 指定多长时间之后日志片段会被关闭，与log.segment.bytes共同作用
         * message.max.bytes: 单个消息的最大大小，指的是压缩后的大小
   * 硬件的选择
      * 磁盘吞吐量
         * 机械硬盘（HDD）VS固态硬盘（SSD）
         * SATA
      * 磁盘的容量
      * 内存
         * kafka本身不需要太大的内存
         * 页面缓存，不建议和其他程序混用。
      * 网络
         * 多个消费者，造成流入流出不均衡。
         * 还有集群复制、镜像也会占用网络流量
      * CPU
         * 解压缩小号CPU，不是主要因素
   * 云端的Kafka
   * Kafka集群
      * 需要多少个broker
         * 需要多少的磁盘空间来保留数据，一个broker有多少空间可用，还有replication
         * 集群处理请求的能力，这通常与网络接口处理客户端流量的能力有关。还有集群复制要考虑。
      * broker配置
         * 相同的zookeeper.connect
         * unique broker.id
      * 操作系统调优
         * 虚拟内存
            * Kafka大量使用系统页面缓存，内存页和磁盘之间的交换（内存交换）对Kafka各方面的性能都有重大的影响。
            * 将vm.swapiness设置小一点，比如1，优先考虑减小页面缓存，而不是进行内存交换
            * vm.dirty_background.ration 设置为小于10的值
            * vm.dirty_ration设置为大于20的值
         * 磁盘
            * 文件系统，使用XFS，而不是EXT4
         * 网络
            * socket读写缓冲区
               * net.core.wmem_default, net.core.rmem_default
            * tcp socket
               * net.ipv4.tcp_wmen, net.ipv4.tcp_rmem
   * 生产环境的注意事项
      * 垃圾回收选项
         * 默认是CMS，建议改为G1
      * 数据中心布局：
         * 同一个机房、同一个机架、公用一个电源？
      * 共享zookeeper
         * 0.9之前consumer也是通过zk存储信息，0.9之后使用broker来维护这些消息（topic）

***Chapter 3： Kafka生产者--向Kafka写入数据***0
   * 生产者概览
      * 序列化（序列化器）
      * partitioner（如果指定分区则不再计算，如果没有指定，则计算分区）
      * topic-parttion：队列，队列中以batch存放消息
      * 发送给broker
      * 成功，broker响应RecordMetaData（topic、partition、offset）
      * 失败，返回错误。Producer可以选择retry或者返回失败
   * 创建Kafka生产者
      * bootstrap.servers: 指定broker的地址清单，不需要包含所有的broker地址，其可以从一个查询其他broker的地址，建议至少两个，以防单点失败
      * key.serializer: 实现org.apache.kafka.common.serialization.Serializer接口的class
      * value.serializer: 可以与key.serializer相同
      * 发送方式：
         * 发送并忘记(fire-and-forget)
            * 不关心是否正常到达
         * 同步发送
            * send().get()
         * 异步发送
            * send(xxx, callback())
   * 发送消息到Kafka
```java
ProducerRecord<String, String> record = new PRoducerRecord<>("customerContry", "Precision Products", "France");//1
try {
	producer.send(record);//2
} catch (Exception e) {
	e.printStackTrace();//3
}
```
      * 同步发送消息
         * producer.send().get();
         * 成功：返回RecordMetadata
         * 异常：如果broekr返回不允许重发消息的异常或者已将超过了重发的次数
            * 可重试异常，可以通过重发来解决
               * 链接异常，可以通过重建链接
               * no leader异常，通过重新为分区选举leader来解决
            * 不可重试异常
               * 消息太大异常
      * 异步发送消息
         * send(xxx, Callback);
   * 生产者的配置
      * acks：指定必须要有多少个分区副本收到消息，生产者才确认消息写入是成功的
         * acks=0，不需要来自任何服务器的响应，吞吐率高
         * acks=1，只需要topic-partition的leader节点收到消息，生产者就会收到来自服务器的成功响应
            * 如果消息无法大大leader节点，生产者会收到一个错误响应，为了避免丢失数据，生产者会重发数据
            * 如果一个没有收到消息的节点成为新leader，消息还是会丢失。
            * 吞吐量取决于是同步发送还是异步发送
         * acks=all，只有当所有参与复制的阶段全部收到消息时，生产者才收到服务器的成功响应。
            * 这个模式是最安全的，延迟较高。
      * buffer.memory：用来设置生产者内存缓冲区的大小，生产产者用它缓冲要发送到broker的数据
         * 如果应用程序发送速度超过发送到服务器的速度，会导致生产空间的不足。这个实收send方法要么阻塞，要么抛出异常，取决于：block.on.buffer.full参数的设置（在0.9之后被替换成max.block.ms，表示在抛出异常前可以阻塞的时间）
      * compression.type
         * 默认情况下是不压缩的
         * 可以设置为snappy、gzip、lz4
         * 压缩可以降低网络开销（cpu消耗和压缩比）
      * batch.size
         * 该参数指定了一个批次可以使用的内存大小，按照字节数计算（而不是消息个数）
         * 该值设置太小会造成频发发送，会增加一些额外的开销
      * linger.ms
         * 该参数指定了生产者在发送批次之前等待更多消息加入批次的时间。 
         * kafka会在批次填满或者linger.ms达到上限时吧批次发送出去。
         * 设置为>0，让生产者在发送批次之前等待一会儿，使更多的消息加入到这个批次，虽然这样会增加演示，单一提升吞吐量
      * client.id 服务器用来标志消息的来源
      * max.in.flight.requests.per.connection
         * 指定了生产者在收到服务器响应之前可以发送多少个消息，值越高，占用的内存越大，不过也会提升吞吐率。
         * 将其设置为1，可以保证消息是按照发送顺序写入服务器的，及时发生了重试。
      * timeout.ms、reqeust.timeout.ms和metadata.fetch.timeout.ms
         * request.timeout.ms指定了生产者在发送数据时等待服务器返回响应的时间
         * metadata.fetch.timeout.ms指定了生产者在获取元数据（比如谁是leader）时等待服务器返回响应的时间。如果等待响应超时，那么生产者要么重试发送数据，要么返回一个错误（抛出异常或者执行回调）。
         * timeout.ms指定了broke等待同步副本返回消息确认的时间，与acks的配置相匹配——如果在指定时间内没有收到同步副本的确认，那么broker就会返回一个错误。
      * max.block.ms
         * send方法调用或者使用partitionsFor方法获取数据生产者的阻塞时间。当生产者的发送缓冲区已满，或者没有可用的元数据时，这些方法就会阻塞。在阻塞时间大大max.block.ms时，生产者就会抛出超时异常。
      * max.request.size
         * 用于控制生产者发送的请求大小。它可一直能发送的单个消息的最大值，也可以值单个请求里所有消息总的大小。
         * broker对可接收的消息最大值也有自己的限制（message.max.bytes)，所以两边的配置最好可以匹配。避免生产者发送消息时被broker拒绝。
      * receive.buffer.bytes和send.buffer.bytes
         * receive.buffer.bytes指定了tpc socket接受数据包缓冲区大小
         * send.buffer.bytesz指定了tcp socket发送数据缓冲区大小
         * 如果被设置为-1，就是用系统的默认值
         * 若果生产者或消费者与broker处于不同的数据中心，那么可以适当增大这些值，因为跨数据中心的网络一般都有比较高的延迟和较低的带宽
      * 保证顺序
         * kafka可以保证同一个分区里面的消息是有序的
         * 如果把retries设置为非零证书，通知吧max.in.flight.requests.per.connection设置为比1大的数，那么，如果第一个批次消息写入失败，而第二个批次写入成功，broekr会重试写入第一个批次。如果第一批次也写入成功，那么两个批次的顺序就反过来了。
         * 一般来说，如果某些场景要求消息是有序的，那么消息是否写入成功也是很关键的，所以不建议将retries设置为0，可以把max.in.flight.requests.per.connection设置为1，这样在生产者发送第一消息时，就不会有其他消息的发送。不过这样会严重影响生产者的吞吐量，所以只有在对消息的顺序有严格要求的情况下才能这么做。
   * 序列化器
      * 自定义序列化器
      * 使用Avro序列化器
      * 在kafka中使用Avro
         * schema注册表：Confluent Schema Registry
   * 分区
      * 如果key为null，使用RoundRobin算法分区
      * 非null，hash取模
      * 实现自定义分区策略：实现Partitioner接口
   * 旧版的生产者API
      * SyncProducer、AysncProducer


***Chapter 4：Kafka消费者，从Kafka读取数据****
   * KafkaConsumer概念
      * Consumer & ConsumerGroup
         * 分配过程
            1. 找到Coordinator（发送给任意一个borker，找到该consumerGroup对应的Coordinator）
            2. join group（向Coordinator发送join请求，返回leader节点）
            3. 同步partition分配结果（leader负责分配，其他的同步结果）
   * 创建Kafka消费者
      * bootstrap.servers
      * key.deserializer: 默认是org.apache.kafka.comon.serialization.StringDeserialization
      * value.deserializer: 默认是org.apache.kafka.comon.serialization.StringDeserialization
   * 订阅Topic
      * consumer.subscribe(Collections.singletonList("customerCountries"))
      * comsumer.subscribe("test.*")
   * poll
```java
try {
	while(true) {
	   ConsumerRecords<String, String> records = comsumer.poll(1000);
	   for (ConsumerRecord<String, String> record : records) {
	      custContryMap.put(record.value(), custContryMap.getOrDefault(record.value(), 0));
	   }
	   JSONObject json = new JSONObject(custContryMap);
	   sout(json.toString());
	}
} finally {
	consumer.close();
}
```
   * consumer的配置
      * fetch.min.bytes ：指定消费者从服务器获取记录的最小字节数
         * 可以调大，以降低broker的负载
      * fetch.max.wait.ms : 通过fetch.min.bytes告诉Kafka，等到有足够多的消息才返回给消费者，而该参数用于指定btoker的等待时间，默认500ms
      * max.patition.fetch.bytes: 服务从每个分区里返回给消费者的最大字节数
         * 与fetch.min.bytes 相关
         * 与max.message.size 相关
      * session.timeout.ms
         * 指定了消费者在被认为失望之前可以与服务器断开连接的时间，默认是3s
         * heartbeat.interval.ms指定了poll方法向Coordinator发送heartbeat的披露，一般为session.timeout.ms的三分之一
      * auto.offset.reset
         * 指定了消费者在读取一个没有偏移量的分区或者偏移量无效的时候，该如何处理
         * 默认是latest
         * 也有，earliest
      * enable.auto.commit
         * 默认为true，表示了偏移量的是否自动提交
         * 通过auto.commit.interval.ms属性来控制提交的频率
      * partition.assignment.strategy
         * 分配策略：PartitionAssignor
         * Range：可能导致不均衡
         * RoundRobin：较为均衡，至多一个之差
      * client.id: broker用来标识从客户端过来的消息，主要用于日志和metrics
      * max.poll.records: 控制单次call返回的记录数量
      * receive.buffer.bytes和send.buffer.bytes
         * socket在读写数据时用到的TCP缓冲区的大小的设置
   * 提交和偏移量
      * 自动提交：
         * 自动提交也是在轮询（poll）中进行的，
         * 存在丢失已处理的信息，进而导致重复消费的可能
            * close（）方法之前也会提交
      * 提交当前的偏移量
         * commitSync会提交有pooll返回的最新偏移量，还是存在信息重复处理的可能
      * 异步提交
         * 提交最后一个偏移量（这次poll回来的）commitAsync()
         * 重试异步提交
            * 使用单调递增的序列号来维护异步提交的顺序，每次提交或者毁掉里面提交偏移量时递增序号。	有条件的重试。
      * 同步和异步组合提交
         * 消费时候commitAsync()
         * finally中commitSync()
      * 提交特定的偏移量
         * 跟踪记录偏移量（map）
         * 定期commit（sync or Async）
   * rebalance listener
      * ConsumerRebalanceListender
         * onPartitionsRevoked:在在均衡开始之前和消费者停止读取消息之后
         * onPartitionsAssigned：在重新分区之后和消费和消费者开始读取消息之前
   * 从特定偏移量开始处理record
      * seekToBeginning
      * seekToEnd
```java
public class SaveOffsetsOnRebalance implements ConsumerRebalanceListener {
	public void onPartitionsRevokde(Collection<TopicAndPartition> partitions) {
	   commitDBTransaction();
	}
	public void onPartitionsAssigned(Collection<TopicAndPartition> partitions) {
	   for (TopicPartition partition: partitions) {
	      consumer.seek(partition, getOffsetFromDB(partition));
	   }
	}
}
consumer.subscribe(topics, new SaveOffsetsOnRebalance());
consuer.poll(0);
for (TopicPartition partition : consumer) {
	consumer.seek(partition, getOffsetFromDB(partition));
}
while (true) {
	ConsumerRecords<String, String> records = consumer.poll(100);
	for (ConsumerRecord<String, String> record : records) {
	   processRecord(record);
	   storeRecordInDB(record);
	   storeOffsetInDB(record.topic(), record.partition(), record.offset());
	}
	commitDBTransaction();
}
```      
   * 如何退出
      * 要想退出循环，需要另一个线程调用consumer.wakeup()
         * 是原有的poll抛出异常WakeupException
      * Runtime.getRuntime.addShutdownHook()

   * 反序列化器
      * Deserializer
      * Avro
   * 独立消费者——为设么以及怎样使用没有群组的消费者
      * 不需要订阅主体，自己分配partition
      * 可能需要周期性调用consumer》partitionsFor()方法来检查是否有新分区加入
   * 旧版的消费者API
 


































