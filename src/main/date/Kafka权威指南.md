Kafka 权威指南

***Chapter1. 初识Kafka***
   * 发布与订阅消息系统
      * 耦合->解耦
   * Kafka登场
      * Record & Batch
      * Pattern
      * Topic & Partition
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
            1. 找到Coordinator（发送给任意一个borker， 找到该consumerGroup对应的Coordinator）
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

***Chapter 5： 深入Kafka***
   * 集群成员关系
      * zk /brokers/ids/{01,2,5,9}
   * 控制器
      *  zk 节点/controller存放当前的epoch，使用epoch 来防止脑裂
   * 复制
      * leader replica：所有的读写请求都会来着
      * follower replica：从leader处复制记录，与leader保持一致，leader崩溃之后，其中之一会被选为leader
      * ISR （in sync replica）
         * 只有在ISR中的replica才会被选为leader
      * auto.leader.reblance.enable: 默认为true
   * 处理请求
      * 消息头
         * Request type: API key
         * Request version: broker可以处理不同版本的消息
         * Correlation ID: 用于标志请求消息，同时也会出现在响应消息和错误日志里面
      * broker模型
         * 一个acceptor线程
         * M个processor线程，放入请求队列，并从响应队列消费结果
         * N个IO线程，消费请求队列，并将结果放入响应队列
      * 请求类型
         * 生产请求
            * 生产者发送的请求，包含了客户端要写入borker的消息
            * 请求验证
               * 发送数据的用户是否有topic写入的权限
               * 请求里包含的acks的值是否有效（0，1，all）
               * 如果acks=all，是否有足够多的同步副本保证信息已经安全写入
            * linux系统上，消息会被写到文件系统磁盘缓存里面，依赖复制功能来保证持久性
            * 当消息给写入分区的首领之后，broker开始检查acks配置参数，如果acks为0或者1，那么broker立即返回响应；如果acks被设为all，那么请求会被保存在一个“炼狱”缓冲区中，知道leader发现所有的follower replica都发复制了该消息
         * 获取请求
            * 在消费者和follower replica需要从broker读取消息是发送的请求
            * 客户端可以指定broker最多可以从一个分区里返回多少数据。
            * 客户端也可以指定最少返回多少数据，此时有最大等待时间控制
            * zero copy技术，kafka直接把消息从文件（或者更确切地说是linux文件系统缓存）里发送到网络通道。
            * leader replica上的数据如果未被同步到follow replica上，则不能被consumer消费
               * replica.lag.time.max.ms
               * 消费者只能看到已经复制到ISR的消息
         * 元数据请求
            * 客户端从任意broker获取该数据，并缓存（metadata.max.age.ms）
      * 物理存储
         * 分区分配
            * 考虑是否多机架
            * 采用轮询的方式分配
         * 文件管理
            * segment
            * active segment（current segment）
         * 文件格式
            * 除了key，value，offset，消息里面还包含了消息的大小、校验和、消息格式版本号、、压缩算法（snappy、GZip、LZ4）和时间戳（可以是producer发送消息的时间，也可以是叨叨broker的时间，可配置）。
         * 索引
         * 清理
            * kafaka通过改变topic的保留策略来满足这些使用场景。早于保留时间的旧事件会被清楚，为每个键保留最新的值，从而达到清理的效果。
               * 只有当producer生成的时间中包含键值对的时候，为这些topic设置compact策略才有意义。如果为null就会失败。
         * 清理工作的原理
            * 每个日志片段可以分为以下两个部分
               1. 干净的部分：这些消息之前被清理过吗，每个键只有一个对应的值
               2. 污浊的部分：这些消息是在上一次清理之后写入的。 
            * 清理功能通过 log.cleaner.enabled参数来配置
            * 为污浊的部分建立map{hash_16(key):offset(8)}，保留最新的offset
            * 从干净的部分开始扫描，不在map中的直接写入，扫完之后将map中的写入
            * 交换替换片段和原有的片段。
         * 被删除的事件
            * 墓碑消息
         * 何时会清理
            * compact策略不会对当前的片段进行清理
            * 50%的磁盘使用的时候会清理，降低清理的频率

***Chapter 6：可靠的数据传递***
   * 系统有可能看起来是可靠的，实际上有可能不是。 
   * 可靠性保证
      * kafka可以保证分区消息的顺序。
      * 只有当消息被写入分区的所有同步副本时(但是不一定要写入磁盘)，它才被认为是“已提交的”。
          * 生产者可以选择接收不同类型的确认，比如消息在被完全提交时的确认，或者在消息被写入leader replica时的确认，或者在消息被发送到网络之后。
      * 只要有一个replica还是活跃的，那么已经提交的消息是不会丢失的。
      * 消费者只能读取已经提交的消息。
      * 构建一个可靠的系统需要作出一些权衡，这种权衡一般是指消息存储的可靠性和一致性的重要的重要程度与可用性、高吞吐量、低延迟和硬件成本的重要程度之间的权衡。
   * 复制
      * Kafka的复制机制和分区的多副本架构是Kafka可靠性的核心保证
      * 对于follower replcia来说，需要满足以下条件才被认为是同步的。
         1. 与zk之间有一个活跃的会话， 也是就是说在过去6s(可以配置)内向ZK发送过心跳
         2. 在过去10s（可以配置）从leader哪里获取过消息的
         3. 在过去10s没从leader获取过最新的消息，光从leader那里获取消息还不够，它必须几乎是零延迟的。
      * 非同步副本
         * 如果一个或多个replica在同步和非同步状态之间来回切换，说明集群内部出现了问题，通常是Java不恰当的垃圾回收
         * 非同步副本不会带来性能上的损失，但是意味着更低的有效复制系数，在发生宕机时丢失数据的风险更高
      * 滞后的同步副本会导致生产者和消费者变慢
   * broker配置
      * 复制系数：replication.factor
         * broker级别的是default.replication.factor（默认是3）
         * 在于可用性和存储硬件之间做出权衡
         * 副本的分布也很重要，在不同的broker上，在不同的机架上
      * 不完全的首领选举：unclean.leader.election
         * 只能在broker级别设置，实际上是集群范围内，默认为true
         * 如果在leader replica不可用时，一个同步副本会被选为新的leader。如果在选举过程中没有数据丢失，也就是说提交的数据同时存在于所有的同步副本上，那么这个选举就是完全的
         * 如果leader不可用时，其他的副本是不同步的
            * 如果有3个副本，其中的两个follower不可用，这个时候如果生产者继续往leader写数据，所有消息都会被确认并提交。加入此时leader 挂了，一个follower启动成功，则其成为一个不同步副本
            * 如果有3个副本，因为网络问题导致两个follower复制消息滞后，则出现两个不同步副本，如果这时候leader down， 两个follower都成为了不同步副本。
         * 如果不同步的副本不能成为新的leader，则分区在旧的leader恢复之前是不可用的。
         * 如果不同步副本可以被提升为新的leader，则存在数据丢失，数据不一致了。
         * 在银行这样要求高一致性的系统中，该设置为false
      * 最少同步副本：min.insync.replicas
         * isr中至少有几个replica 同步之后才确认提交
   * 在可靠的系统里使用生产者
      * 需要producer和broker协作才能保证数据不丢失
         * 根据可靠性需求配置强档的acks值
         * 在参数配置和代码里正确处理错误
      * 发送确认
         * acks=0，如果生产者能够通过网络把消息发送出去，那么就确认消息成功写入kafka
         * acks=1，如果leader replcia收到消息并写入分区数据（不一定刷盘）时，会响应确认或错误。如果此时正发生了leader选举，生产者会收到LeaderNotAvailableException异常，如果producer能正确处理该异常，如重发，最终能安全达到新leader。不过此场景仍然存在数据丢失的情况，比如信息写入leader但尚未同步到follower，此时leader挂了。
         * acks=all，意味着leader返回确认或错误响应之前，会等待所有同步副本（ISR）返回确认。如果和min.insync.replicas参数结合起来，就可以决定在返回确认之前至少有多少个副本嫩巩固收到消息。
      * 配置生产者的重试参数
         * 两种错误
            * producer可以自动处理的错误
               * 比如LEADER_NOT_AVAILABLE，可以重试几次，没准leader已经选举出来了
               * 重试的次数取决于系统的目标
                  * MirrorMaker会无限次重试 retries=MAX_INT
                  * 重试可能导致消息的重复，达到消息至少被保存一次
                  * 如果要做到exactly once， 需要producer + broker + consumer协作，通常每个消息有一个唯一的key，则consumer可以对其处理，还有一些系统支持迷瞪性。
            * 需要开发者手动处理的错误
               * INVALID_CONFIG，重试也没有意义
      * 额外的错误处理
         * dev需要处理的错误
            * 不可重试的broker错误，例如消息大小错误，认证错误
            * 在消息发送之前发生的错误，例如序列化错误
            * 在生产者达到重试次数上限或者在消息占用内存达到内存上限时发生的错误
   * 在可靠的系统里使用消费者
      * 在从partition读取数据的时候，消费者会获取一批事件，检测这批事件的最大偏移量，然后从这个偏移量开始读取另外的一批事件。
      * 如果一个消费者退出，另一个消费者需要知道从什么地方开始继续处理，它需要知道前一个消费者在退出前处理的最后一个偏移量是多少。
         * 如果一个消费者提交了偏移量，却未能消费完信息，则出现了消息丢失。
      * 消费者的可靠性配置
         * group.id: 同一个group信息
         * auto.offset.reset：指定了在没有偏移量或者请求的偏移量不存在的时候，消费者的处理逻辑
            * earliest
            * latest
         * enable.auto.commit: 自动提交与否
            * 自动提交的缺点，无法控制重复处理消息，
         * auto.commit.interval.ms：自动提交的时间间隔，默认5s/次，提交频繁会增加额外的开销，但也会降低重复处理消息的概率
      * 显示提交偏移量
         0. 如果希望能够更多的控制偏移量提交的时间点，那么就要考虑如何提交
            * 要么是减少重复处理消息
            * 要么是因为把消息处理逻辑放在了轮询之外
         1. 总是在处理完事件后在提交偏移量
            * 可以使用自动提交，或者在轮询结束时候进行手动提交
         2. 提交频度是性能和重复消息数量之间的权衡
            * 可以在循环里多次提交，也可以在循环外一次提交
         3. 确保对提交的偏移量心里有数
            * 在轮询中提交的可能是获取到的数据的最大的偏移量，而不是消费到的偏移量
         4. 再均衡（Rebalance）
            * 需要分区被撤销之前提交偏移量，并在分配新分区的时候清理之前的状态
         5. 消费者可能需要重试
            * 在进行轮询之后，有些消息不会被完全处理，而提交的是偏移量，而不是对消息的确认。（存在31被正确处理了，而30还没又被成功处理）
            * 第一种解决模式：在遇到可重试错误时候，提交最后一个处理成功的消息的偏移量，然后把还没有处理好的消息保存到缓冲区里（这样下一次轮询不会被覆盖掉），调用消费者的pause（）来保证其他的轮询不会反悔数据，在保持轮询的时候需要尝试重新处理（不能停止轮询）。如果成功或者重试达到上限并决定放弃，则记录并丢弃消息，然后调用消费者的resume()让消费者继续从轮询中获取新数据
            * 第二种解决模式：遇到可重试错误时，把错误写入一个独立的topic，然后继续。为这个独立的topic配置一个独立的消费者群组（dead-letter-queue），重试时需要暂停该主题。
         6. 消费者可能需要维护状态
            * 可以使用kafka stream，其提供了保存状态
            * 或者其他的流式处理平台
         7. 长时间处理
            * 如果一个消息要处理很长时间，导致轮询被暂停了，会出问题。因为轮询中有heart-beat
            * 可以用单独的线程池处理请求
         8. 仅一次传递
            * 幂等性写入：将结果放入到支持唯一键的系统里。
            * 如果消息系统知识事物，则可以使用acid来保证，client自身维护偏移+seek()
   * 验证系统可靠性
      * 三个层面的验证：配置验证、应用程序验证和生产环境的应用程序监控
      * 配置验证
         * 对broker和客户端的配置进行验证
         * 验证配置是否满足你的需求
         * 帮助理解系统的行为，了解系统的正真行为是什么，了解对kafka基本准则是否存在偏差，然后加以改进，同时了解这些准则是如何被应用到各种场景中的。
         * kafka提供了org.apache.kafka.tools.{VerifiableProducer,VerifiableConsumer}  
         * 运行测试
            * leader election：如果leader down，需要多久producer和consumer才能恢复
            * controller election：重启控制器之后系统需要多长时间能恢复
            * 依次重启：可以依次重启borker而不丢失数据不？ 
            * 不完全leader election测试：如果依次停止所有replica（确保每个副本变为不同步的），然后启动一个不同步的broker会发生什么？要怎样才能恢复正常？这样做是可以接受的么？
      * 应用程序验证
         * 集成测试
         * 故障测试：
            1. 客户端从服务器断开连接
            2. leader election
            3. 依次重启broker
            4. 依次重启consumer
            5. 依次重启producer
      * 在生产环境监控可靠性
         * kafka的java客户端JMX度量指标
            * producer的error-rate和retry-rate
            * producer日志中的剩余重试次数为0
            * consumer：consumer-lag，表明消费者处理速度与最近提交到分区里的偏移量之间还有多少差距
            * 监控数据流，确保所生成的数据被及时的消费了

***Chapter 7: 构建数据管道*** 
   * 构建数据管道时需要考虑的问题
      * 及时性
      * 可靠性
         * 至少传递一次
         * 仅传递一次
      * 高吞吐量和动态吞吐量
      * 数据格式
      * 转换
         * ETL：extract-Transform-Load
            * 管道负责处理数据，
         * ELT：extract-Load-transform
            * 高保真
      * 安全性
         * 支持认证
      * 故障处理能力
      * 耦合性和灵活性
         * 临时数据管道
         * 元数据丢失
         * 末端处理
   * 如何在connect API和客户端API之间进行选择
      * 首选connect，因为其提供了一些开箱即用的特性：配置管理、偏移量处处、并行处理、错误处理，而且支持多种数据类型的REST管理API
   * Kafka Connect
      * 运行Connect
         * bin/connect-distributed.sh config/connect-distributed.properties （单机standalone 运行不起来。。。）
            * bootstrap.servers
            * group.id
            * key.converter & value.converter
      * 连接器示例：文件数据源&文件数据池 & 连接器示例：从mysql到ES
         * 从源导数据到kafka topic（由转化器转换数据），并且从该topic导出数据到dest
      * 深入理解Connect
         1. 连接器和任务
            * 连接器：
               * 决定需要多少个任务
               * 按照任务来才分数据复制
               * 从worker进程获取任务配置并将其传递西区
            * 任务
               * 负责将数据移入或移出kafka
         2. worker进程
            * 是连接器和任务的“容器”，
         3. 转化器和Connect的数据模型
         4. 偏移量管理
   * Connect之外的选择
      * 用于其他数据存储的摄入框架
         * Flume，Logstash，Fluentd
      * 给予图形界面的ETL工具
      * 流式处理框架

***Chapter 8：跨集群数据镜像***
   * 不同部门有不同的集群，对于SLA有不同的要求，工作负载也不同
      * 为每个业务创建单独的集群， 管理多个集群本质上就是重复多次运行单独的集群
      * Kafka MirrorMaker VS. 数据库复制
   * 跨集群镜像的使用场景
      * 区域集群和中心集群
         * 每个地方有一个区域集群，然后又有一个中心集群（会镜像各个区域集群的数据）
      * 冗余（DR）
         * 数据冗余
      * 云迁移
         * 本地和每个云服务区都会有一个Kafka集群
   * 多集群架构
      * 跨数据中心通信的一些现实情况
         * 高延迟
         * 有限的带宽
         * 高成本
            * 带宽价格高
         * 架构原则
            1. 每个数据中心至少需要一个集群
            2. 每两个数据中心的数据复制要做到每个事件仅一次（除非出现错误需要重试）
            3. 如果有可能，尽量从远程数据中心读取数据，而不是向远程数据中心写入数据
      * Hub和Spoke架构
         * 一个中心，多个地域
         * 变体：一个leader， 一个follower
         * 优点：数据只会在本地的数据中心生成，而且每个数据中心只会被镜像到重要数据中心一次；易于部署、配置和监控
         * 缺点：一个数据中心无法访问另一个数据中心的数据
      * 双活架构
         * 当有两个或多个数据中心需要共享数据并且每个数据中心都可以生产和读取数据的时候，可以使用双活（Active-Active）架构
            * 优点
               * 为就近的用户服务，获得性能上的优势，而且不会因为数据的可用性问题在功能方面做出牺牲
               * 冗余和弹性：一种简单透明的实效备援方案
            * 缺点
               * 如何在进行多个位置的数据异步读取和异步更新是避免冲突
                  * 比如镜像技术：如何确保同一份数据不会永无止境的来回镜像（不同的命名空间+filter）
               * 数据一致性问题
                  * 向第一数据中心写，之后向第二个数据中心读取，可能读不到；解决方案，将用户粘在某一个数据中心
                  * 用户在一个数据中心订购A，而第二个数据中心几乎同时受到了订购B的订单，在经过镜像之后，每个数据中心都有了A和B；解决方案，在两个数据中心之间定义一致的规则，用于确定哪个才是正确的。
      * 主备架构
         * Active-Standby
         * 优点：易于实现
         * 缺点：浪费了一个集群
      * 实效备援包括的内容
         1. 数据丢失和不一致性：异步的解决方案，存在丢失数据的可能
         2. 实效备援之后的起始偏移量
            * 偏移量自动重置
            * 复制偏移量topic
            * 基于时间的实效备援：通过时间去索引寻找
            * 偏移量外部映射 ： 外部存储
         3. 失效备援之后
            * 原有主机群清理，且变为灾备集群
         4. 关于集群发现
            * zk， etcd，consul
      * 延展集群（Stretch Cluster）：跨越多个数据中心安装多个kafka集群
         * min.isr, acks=all
         * 同步复制是这种架构的最大优势
         * 至少要有3个具有高带宽和低延迟的数据中心上安装kafka和zk
   * Kafka的Mirror Maker
      * 镜像过程
         * MirrorMaker为每个消费者分配一个线程，消费者从源集群的topic和partition上读数据，然后通过公共生产者将数据发送到目标集群。
         * 默认情况下，消费者每60s通知生产者发送所有的数据到kafka，并等待kafka的确认
         * 然后通知源集群提交相应的偏移量
      * 配置
         * consumer.config
         * producer.config
         * new.consumer
         * nums.streams
         * whitelist
      * 生产环境部署MirrorMaker
         * 度量指标
      * 调优
         * tcp缓冲区大小的增加
         * 词用时间窗口自动伸缩
         * 减少tcp慢启动时间
         * max.in.flight.requests.per.connection:默认只允许一个处理中的请求
         * linger.ms和batch.size
         * 提升消费者吞吐量
            * range
            * fetch.max.bytes
            * fetch.min.bytes和fetch.max.wait
   * 其他跨越集群镜像方案
      * uber的uReplica
      * Confluent的Replica 

***Chapter 9：管理kafka***
   * topic工具 kafka-topic.sh
      * --create
      * --alter
      * --delete
      * --list
      * --describe
   * 消费者群组 kafka-consumer-group.sh
      * --list
      * --describe
      * --delete
   * 动态配置变更 kafka-config.sh
      * --alter add/delete
   * 分区管理
      * 首选的首选首领
         * 自动首领再均衡
      * 修改分区副本
      * 修改复制系数
      * 转处日志片段
      * 副本验证
   * 生产和消费
   * 客户端ACL
   * 不安全的操作
      * bad idea：修改zk上的数据
      * 移动集群控制器
      * 取消分区重分配
      * 溢出待删除的topic
      * 手动删除topic


***Chapter 10：监控Kafka***
   * 度量指标基础
      * JMX（Java Management Extensions）
      * 内部度量或外部度量（可用性或延迟）
      * 应用程序健康监测
         * 使用外部进程来报告broker的运行状态（健康监测）
         * 在broker停止发送度量指标时发出报警（stale度量指标）
      * 度量指标的覆盖面
   * broker的度量指标
      * 非同部分区
         * 集群级别问题
         * 主机级别问题
      * broker度量指标
         * 活跃控制器数量
         * 请求处理器空闲率
         * topic流入字节
         * topic流出字节
         * topic流入的消息
         * 分区数量
         * leader数量
         * 离线分区
         * 请求度量指标
      * topic和partition的度量指标
      * java虚拟机监控
         * GC
         * java操作系统监控
      * OS操作系统监控
         * 系统负载
            * 平均负载：等在处理器执行的线程数
      * 日志
   * 客户端监控
      * 生产者度量指标
         * 生产者整体度量指标
         * per-broker和per-topic度量指标
      * 消费者度量指标
         * fetch manager度量
         * per-broker和per-topic度量
      * Coordinator度量指标
      * 配额
   * 延时监控
   * 端到端监控

***Chapter 11：流式样处理***
   * 




























