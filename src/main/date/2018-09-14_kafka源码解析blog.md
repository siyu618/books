
服务熔断、降级、限流、异步RPC -- HyStrix
   * https://blog.csdn.net/chunlongyu/article/details/53259014

分布式架构－－基本思想汇总
   * https://blog.csdn.net/chunlongyu/article/details/52525604   


Kafka源码深度解析－系列1 －消息队列的策略与语义
   * https://blog.csdn.net/chunlongyu/article/details/52538311
   * 关键概念：
      * topic: 逻辑的数据队列
      * broker：Kafka 集群的节点
      * partition：topic 分成多个 partition，用来提升并发
      * replica/leader/follower： acks
   * 消息队列的各种语义
      * Producer 的策略
         * acks：0（不等服务器返回ack），1（leader 确认消息存下来后再返回）， all（leader 和当前 ISR 中所有的 replica 都确认消息存下来，再返回）
         * 同步发送 VS. 异步发送
      * Consumer 的策略
         * Push VS. Pull : Long pulling
         * 消费的 confirm：offset
      * Broker 的策略
         * 消费顺序的问题
         * 消息的刷盘机制：page cache、fsync 存盘
      * 消息的不重不漏
         * 完美的消息队列，不漏不重
            * 消息不会重复存储（解决 > 1 ）：代价很大，一个思路每个消息一个primeKey，在broker端去重
            * 消息不会重复消费（解决 > 1 ）：需要消息的 confirm
            * 消息不会丢失存储（解决 < 1 ）：replica
            * 消息不会丢失消费（解决 < 1 ）：需要 confirm
         * exactly once：真正做到不重不漏，exactly once，是很困难的，需要 broker、producer、consumer 和 业务方 的配合。
         * kafka 保证不漏，就是 at least once

Kafka源码深度解析－序列2 －Producer －Metadata的数据结构与读取、更新策略
   * https://blog.csdn.net/chunlongyu/article/details/52622422
   * 多线程异步发送模型
      * 基本思路：发送的时候，KafkaProducer 将消息放入本地消息队列 RecordAccumulator，然后一个后台的线程 Sender 不断循环，将消息发送给 Kafka 集群。
         * 前提条件：Producer 和 Sender 都需要获取 Metadata。所谓 Metadata，就是 Topic/Partition 与 broker 的映射关系：每个 topic 的 partition ，得知其 broker 列表是啥，leader 是谁，follower 是谁。
      * 两个数据流
         * Metadata 流：Sender 从集群获取信息。KafkaProducer 从 Metadata 获取数据，然后放入队列。
         * Producer 将消息放入 RecordAccumulator，Sender 从 RecordAccumulator 读取数据，发送给 Kafka 集群。  
   * Metadata 的线程安全性
      * Metadata 的方法都是 synchronized
   * Metadata 的数据结构
      * cluster + 状态
   * Producer 读取 Metadata： waitOnMetadata
      * 如果缓存中有，直接返回；同时需要保证当前记录的 partition 为 null，或者在合法的范围内。
      * 否则一直等待获取到数据，期间可能抛出异常。
         * metadata.awaitUpdate(version, remainingWatiTimeMs)
            * 直到当前版本高于已知的版本
            * wait() ; // 等待 sender 的 notify
   * Sender 的创建
      * KafkaProducer 中的一个线程
   * Sender 的 poll()：Sender.run(long now)
      * 处理 事务，如果是事务的话。
      * sendProduceData(long now)
         * recordAccumulator.ready(): 
            * 获取可以发送的节点列表
            * 不可发送的变为可以发送的时间
            * 不知道 leader 节点的 topic 列表
         * 更新 不知道 leader 节点 的 metadata
         * 删除没有准备好的节点
         * 从 accumulator 中获取 可以发送的 batches
         * 如果需要保证发送顺序：
            * 将这些 batch 的 topicPartition mute
         * 获取 在 inFlightBatches 中过期的 batch：expiredInFlightBatches
         * 获取 accumulator 中过期的 batches： expiredBatches
         * 将 batch：expiredInFlightBatches 加入到 expiredBatches
         * update metrics
         * sendProduceRequests(batches, now)
            * sendProduceRequest(new, id, acks, reqeustTimeoutMs, List<ProducerBatch)
               * 按照 topicPartition 分好，并且对于 record 做必要的转换 magic number
               * client.send(clientRequest, now)
                  * doSend(request, isInternalRequest=false, now)
                     * doSend(clientRequest, isInternalRequest, now, builder.build(version)
                        * 构建 send 对象
                        * 构建 InFlightRequest，放入 inFlightRequests 中
                        * selector.send(send)
                           * channel.setSend(send)
      * client.poll(poolTimeout, now)
         * 如果 abortedSends 非空，则处理之并返回
            * handleAbortedSends()
            * completeResponses()
               * 调用回调函数
         * 更新 metadata 数据
         * selector.poll()：处理实际的 IO 事件
            * poll from 有缓存数据的 channel
               * 处理 read、write（completedSends）
            * poll from socket 有数据的 channel
            * poll from 连接事件
            * 关闭老的连接 maybeCloseOldestConnection(endSelect)
            * addToCompletedReceives()
               * 处理 stagedReceives
         * handleCompletedSends
            * this.inFlightRequests.completeLastSent(send.destination())
               * 取出 first
            * 构建 response
         * handleCompletedReceives
            * 处理 metadataResponse
            * 处理 apiVersionResponse
            * 处理 produceRequest 的 response
         * handleDisconnections
            * 处理失效的链接
         * handleConnections
            * 处理新的链接
         * handleInitiateApiVersionRequests
            * 处理新的 apiversion 请求
         * handleTimeoutRequests
         * handleResponses
            * 调用回调函数
   * Metadata 的两种更新机制
      * 周期性更新：每隔一段时间更新一次
      * 失效检测，强制更新；
      * 每次 poll 的时候都会检查这两种机制，hit 了就触发更新
   * Metadata 的实现检测
      * 条件1：initConnect 的时候
      * 条件2：poll 里面 IO 的时候，连接断掉了
      * 条件3：有请求超时
      * 条件4：发消息的时候，有 partition 的 leader 没有找到
      * 条件5：返回 response 和请求对不上的时候            
   * Metadata 其他的更新策略
      * 更新请求 MetadataReqeust 是 NIO 异步发送的，在 poll 返回，处理 MetaDataResponse 的时候才真正更新 metadata
      * 更新的时候，是从 metadata 中保存的所有 Node 或者说是 broker 中选择负载最小的，向其发送 MetadataReqeust 请求，获取新的 Cluster 对象。


Kafka源码深度解析－序列3 －Producer －Java NIO
   * https://blog.csdn.net/chunlongyu/article/details/52636762
   * epoll
      * LT : 水平触发（条件触发），读缓冲区只要非空就一直触发读事件；写缓冲区不满就一直触发写事件
         * 缺省模式，需要避免“写的死循环”的问题，解法为写完就取消写事件
         * 适用于NIO + BIO
      * ET：边缘触发（状态触发），读缓冲区的状态从空转为非空的时候触发一次，写缓冲区由满转为非满的时候触发一次
         * 需要避免“short read”事件，一定要把缓冲区读取完。
         * 仅仅适用于NIO

Kafka源码深度解析－序列4 －Producer －network层核心原理
   * https://blog.csdn.net/chunlongyu/article/details/52651960

Kafka源码深度解析－序列5 －Producer －RecordAccumulator队列分析
   * https://blog.csdn.net/chunlongyu/article/details/52704213

Kafka源码深度解析－序列6 －Consumer －消费策略分析https://blog.csdn.net/chunlongyu/article/details/52791874 
   * https://blog.csdn.net/chunlongyu/article/details/52663090   
   * comsumer group 两种模式
      * 负载均衡模式：多个consumer从属于一个group，一个topic的partition均匀的分配到各个consumer上
      * pub/sub模式：多个consumer从属于不同的group，
   * partition自动分配vs手动指定
      * 两者是互斥的
   * 消费确认 consume offset vs. commited offset
      * 当前拉去到的信息的offset：consume offset
      * 处理完毕，发送ack之后确定的commited offset
      * 在异步模式下 commited offset小于consume offset
      * 关键点：加入consumer挂了重启，那它将从commited offset位置开始重新消费，而不是consume offset位置。这也就意味着可能存在重复消费
   * 3种ack策略
      * 自动周期性ack ： "enable.auto.commit", "true" + "auto.commit.interval.ms", "1000"
   * exactly once：自己保存offset
      * 通过事物保证消费和保存offset的原子
      * 需要的准备工作
         * 禁用自动ack： "enable.auto.commit", "false"
         * 将每次消费的ComsumerRecord 存下来
         * 下次重启还是从记录下来的offset开始消费，seek(topic_patition, long)

Kafka源码深度解析－序列7 －Consumer －coordinator协议与heartbeat实现原理
   * https://blog.csdn.net/chunlongyu/article/details/52791874
   * 单线程的Consumer
      * KafkaProducer 是线程安全的，同时其内部有一个Sender，开了一个后台线程，不断从队列取消息进行发送
      * 是一个纯粹的单线程，所有事情都是在其poll()中实现的，coordinator、rebalance、heartbeat
   * Coordinator
      * 0.9开始去除了zk的依赖，为了避免“herd”（羊群效应）和“split brain”（脑裂）
   * 在一个group中，一个partition只能被一个consumer消费
      * 保证一个partition中的时序
   * coordiantor协议/partition分配问题
      * 分为三个步骤
         1. GCR（GroupCoordinatorRequest），发给任意的一个Coordinator，返回Coordinator
            * kafka集群为每个consume group选择一个broker作为其Coordinator
         2. JGR（JoinGroupRequest），发给Coordinator，返回身份信息leader or follower
         3. SGR (SyncGroupRequest)，发请求给Coordinator（leader 进行分配必将结果带给Coordinator），Coordinator返回分配结果给followers
            * partition的分配策略和分配结果是由client决定的
               * 没有在Coordinator做这个事情，是从灵活性角度考虑，如果让server分配，一旦需要新的策略就需要重新部署server集群
    * rebalance机制
       * Rebalance的条件：
          1. 有新的consumer加入
          2. 旧的的consumer挂掉
          3. Coordinator挂掉了
          4. topic的partition增加了
          5. consumer调用unsubscribe(),取消订阅了
       * 当consumer检测到需要进行Rebalance，所有的consumer就需要走上面的流程，进行步骤二 + 步骤三
    * heartbeat的实现
       * consumer通过这个知道需要进行Rebalance
       * 每个consumer定期往Coordinator发送heartbeat消息，一旦Coordinator返回ILLEAL_GENERATION，就说明之前的group无效，需要Rebalance
       * HeartBeatRequest 是放在delay queue中的
       * rebalance 检测
          * 将rejoinNeeded设置为true
    * failover
       * consumer和Coordinator都有可能挂掉，需要双方互相检测
       * consumer认为Coordinator挂掉，从步骤1开始，重新dicover Coordinator，然后join group + sync group
       * Coordinator认为consumer挂掉，通知其他剩下的consumer，然后进行joinGroup + sync group

Kafka源码深度解析－序列8 －Consumer －Fetcher实现原理与offset确认机制
   * https://blog.csdn.net/chunlongyu/article/details/52796639
   * offset 初始化 - 手动指定 vs. 自动指定
      * 手动：seek（topicPattition，offset）
      * 自动：poll之前请求向Coordinator请求offset
   * fetcher核心流程
      * 步骤1：fetcher.initFetchers(cluster)
         * 将所有属于同一个节点的topic-partition放在一起，生成一个fetchRequest
         * 
      * 步骤2：poll
      * 步骤3：fetcher.fetchRecords
   * 手动消费确认与自动消费确认
      * 手动：KafkaConsumer.commitSync Vs commitAsync（OffsetCommitCallback）
      * 自动：周期性的提交，DelayedQueue + delayedTask

Kafka源码深度解析－序列9 －Consumer －SubscriptionState内部结构分析
   * https://blog.csdn.net/chunlongyu/article/details/52806408
   * 两种订阅策略
      * 手动：assign
      * 自动：subscribe
      
      ```java
      public class SubscriptionState {
        //该consumer订阅的所有topics
        private final Set<String> subscription;
        //该consumer所属的group中，所有consumer订阅的topic。该字段只对consumer leader有用
        private final Set<String> groupSubscription;
        //策略1：consumer 手动指定partition, 该字段不为空
        //策略2：consumer leader自动分配，该字段为空
        private final Set<TopicPartition> userAssignment;
        //partition分配好之后，该字段记录每个partition的消费状态(策略1和策略2，都需要这个字段）
        private final Map<TopicPartition, TopicPartitionState> assignment;
      }
 
      //SubscriptionState中的字段
 
      private final Map<TopicPartition, TopicPartitionState> assignment;
      //TopicPartitionState内部结构
        private static class TopicPartitionState {
            private Long position;  //字段1：记录当前要消费的offset, 在fetchRecords中更新
            private OffsetAndMetadata committed; //字段2：记录已经commit过的offset， 在commit中更新
        }
   
        public class OffsetAndMetadata implements Serializable {
            private final long offset;
            private final String metadata; //额外字段，可以不用。比如客户端可以记录哪个client, 什么时间点做的这个commit
         }
      ```

Kafka源码深度解析－序列10 －Server入门－Zookeeper与集群管理原理
   * https://blog.csdn.net/chunlongyu/article/details/52872281
   * broker的生与死
      * zk中/brokers/ids/xxx
   * Controller
      * 为了减小zk的压力，同时降低分布式系统的复杂性，kafka引入了中央控制器Controller
      * 利用zk选举出Controller，然后Controller控制所有的broker
      * Controller监听zk上节点的变化
   * topic与partition的增加/删除
      * 管理端将增加/删除命令发给zk，Controller监听zk获取更新消息，Controller在分组发送给相关的broker
   * 101 Tech ZkClient
      * kafka、dobbo都是使用这个client的，较为轻量级
      * 三个接口
         * IZkStateListener，IZkDataListener， IZkChildListener


Kafka源码深度解析－序列11 －Server核心组件之1－KafkaController选举过程/Failover与Resignation
   * https://blog.csdn.net/chunlongyu/article/details/52933947
   * 在sever的启动函数中，可以看到以下几大核心组件
      1. socketServer + KafkaApis前者接受所有网络请求， 后者处理请求
      2. KafkaController负责Controller选举
      3. ConsumerCoordinator，用于负责consume group的负载均衡
      4. ReplicaManager机器的管理
      5. KafkaSchedule
   * 选举的基本原理
      * 在zk中创建/controller临时节点，其data用来记录当前的controller的brokerid [“version”=1，“broker“=brokerId， ”timestamp“=timestamp]
      * /controller_epoch用来记录当前的轮次
   * KafkaController与ZookeeperLeaderElector
      * 后者是前者的一个成员
      * 选举交互过程
         1. KafkaController和ZookeeperLeaderElector内部各有一个Listener，一个监听session重连，一个监听、controller变化
         2. 当session重连或者/controller节点被删除，则调用elect()函数，发起重新选举。在重新选举之前，先判断自己是否是就得Controller，如果是则先调用onRegistration退位
      * 两个关键回调
         * 新官上任 + 旧官退位

Kafka源码深度解析－序列12 －Server核心组件之2－ReplicaManager核心数据结构与Replica同步原理
   * https://blog.csdn.net/chunlongyu/article/details/52938947
   * ReplicaManger
``` java
   class ReplicaManager(val config: KafkaConfig,
                     metrics: Metrics,
                     time: Time,
                     jTime: JTime,
                     val zkUtils: ZkUtils,
                     scheduler: Scheduler,
                     val logManager: LogManager,
                     val isShuttingDown: AtomicBoolean,
                     threadNamePrefix: Option[String] = None) extends Logging with KafkaMetricsGroup {
  //核心变量：存储该节点上所有的Partition
  private val allPartitions = new Pool[(String, Int), Partition]

  ///然后对于每1个Partition，其内部又存储了其所有的replica，也就是ISR:
  class Partition(val topic: String,
                val partitionId: Int,
                time: Time,
                replicaManager: ReplicaManager) extends Logging with KafkaMetricsGroup {
  private val localBrokerId = replicaManager.config.brokerId
  private val logManager = replicaManager.logManager
  private val zkUtils = replicaManager.zkUtils
  private val assignedReplicaMap = new Pool[Int, Replica]
  //核心变量：这个Partition的leader
  @volatile var leaderReplicaIdOpt: Option[Int] = None

  //核心变量：isr，也即除了leader以外，其它所有的活着的follower集合
  @volatile var inSyncReplicas: Set[Replica] = Set.empty[Replica]
  class Replica(val brokerId: Int,
              val partition: Partition,
              time: Time = SystemTime,
              initialHighWatermarkValue: Long = 0L,
              val log: Option[Log] = None) extends Logging {

  。。。
  //核心变量：该Replica当前从leader那fetch消息的最近offset，简称为loe
  @volatile private[this] var logEndOffsetMetadata: LogOffsetMetadata = LogOffsetMetadata.UnknownOffsetMetadata
```

   * replica同步原理
      *  t0p1: b2, b3, b5（对于该partition，b2作为leader); 
      * b2的socketServer收到producer的producerRequest请求，把请求交个ReplicaManager处理，ReplicaManager调用自己的appendMessage函数，将消息存到本地日志
      * ReplicaManager生成一个DelayedProducerRequest对象，放入DelayedProducerPugator中，等待follower来把该请求pull到自己的服务器上
      * 2个followers会跟consumer一样，发送FetchRequest请求到socketServer，ReplicaManager调用自己的fetchMessage函数返回日志，同时更新2个follower的LOE（LogEndOffset），病且判断DelayedProducer是否可以complete。如果可以则发送ProduceRepose
  * 关键点
     1. 每个DelayedProduce内部办函一个ProduceResponseCallback函数。当complete之后，该callback被调用，也就处理了ProduceRequest请求
     2. leader处理ProduceRequest请求和follower同步日志，这两个事情是并行的。leader不会等待两个follower同步该消息，再处理下一个。
     * 每一个ProduceRequest对于一个该请求写入日志是的requestOffset。判断该消息是否同步完成，只要每个replica的LOE>=reqeustOffset就可以了，并不需要完全相等

Kafka源码深度解析－序列13 －Server核心组件之2(续)－ TimingWheel本质与DelayedOperationPurgatory核心结构
   * https://blog.csdn.net/chunlongyu/article/details/52971748
   * ReplicaManager内部的2个成员变量
```java
class ReplicaManager(val config: KafkaConfig,
                     metrics: Metrics,
                     time: Time,
                     jTime: JTime,
                     val zkUtils: ZkUtils,
                     scheduler: Scheduler,
                     val logManager: LogManager,
                     val isShuttingDown: AtomicBoolean,
                     threadNamePrefix: Option[String] = None) extends Logging with KafkaMetricsGroup {

  //关键组件：每来1个ProduceReuqest，写入本地日志之后。就会生成一个DelayedProduce对象，放入delayedProducePurgatory中。
  // 之后这个delayedProduce对象，要么在处理FetchRequest的时候，被complete()；要么在purgatory内部被超时.
  val delayedProducePurgatory = new DelayedOperationPurgatory[DelayedProduce](
    purgatoryName = "Produce", config.brokerId, config.producerPurgatoryPurgeIntervalRequests)

  val delayedFetchPurgatory = new DelayedOperationPurgatory[DelayedFetch](
    purgatoryName = "Fetch", config.brokerId, config.fetchPurgatoryPurgeIntervalRequests)
```
   * DelayedProducePurgatory核心结构
      * DelayedProducePurgatory 两个核心部件：watches的map，一个是Timer
      * DelayedProduce 对应的有两个：一个是delayedOperation，同时它也是一个TimerTask
      * 每当处理一个ProduceRequest，就会生产等一个DelayedProducer对象，被加入到Watcher中，同时其也是一个TimeTask，加入到Timer中。
      * 最后这个DelayedProduce可能被接下来的Fetch满足，也可能在Timer中超时，给客户端返回超错误。 如果是前者，就需要在timer中调用Task.cancel，把该任务删除。
   * Timer的实现，TimingWheel
      * DelayedQUeue的时间复杂度是O(lg(n))，同时不支持随机删除。
      * TimingWheel的，O(1)，支持Task的碎甲删除
      * 实现方式：
         * 调用者不断调用time.add函数添加新的Task，
         * Timer不是内部线程驱动，而是有一个外部的线程ExpiredOperationReaper，不断的调用time.advanceClock()函数，来驱动整个Timer
         * 总结：一个有两个外部线程，一个驱动Timer，一个executor专门用来执行过期的task。整个两个线程都是DelayedOperationPurgatory的内部变量
   * Timer的内部结构
      * Timer是最外城雷，表示一个定时器。其内部一个TimingWheel对象，TimingWheel是层次结构的，每个TimingWheel可能有parentTimingWheel（这个原理就类似于生活中的水表，不同表盘有不同的刻度
      * TimingWheel是一个时间刻度盘，每个刻度上有一个TimerTask的双向链表，称之为一个bucket，同一个bucket中的所有task的过期时间相同。因此每个bucket有一个过期时间的字段
      * 所有的TimingWheel公用了一个DelayedQueue，这个DelayQueue存储了所有的bucket，而不是TimeTask。
```java
//Timer
class Timer(taskExecutor: ExecutorService, tickMs: Long = 1, wheelSize: Int = 20, startMs: Long = System.currentTimeMillis) {
  ....
  //核心变量 TimingWheel
  private[this] val timingWheel = new TimingWheel(
    tickMs = tickMs,
    wheelSize = wheelSize,
    startMs = startMs,
    taskCounter = taskCounter,
    delayQueue
  )

  //Timer的核心函数之1：加入一个TimerTask
  def add(timerTask: TimerTask): Unit = {
    readLock.lock()
    try {
      addTimerTaskEntry(new TimerTaskEntry(timerTask))
    } finally {
      readLock.unlock()
    }
  }

  //Timer的核心函数之2：Tick，每走1次，内部判断过期的TimerTask，执行其run函数
  def advanceClock(timeoutMs: Long): Boolean = {
    var bucket = delayQueue.poll(timeoutMs, TimeUnit.MILLISECONDS)
    if (bucket != null) {
      writeLock.lock()
      try {
        while (bucket != null) {
          timingWheel.advanceClock(bucket.getExpiration())
          bucket.flush(reinsert)
          bucket = delayQueue.poll()
        }
      } finally {
        writeLock.unlock()
      }
      true
    } else {
      false
    }
  }

  //TimingWheel内部结构
private[timer] class TimingWheel(tickMs: Long, wheelSize: Int, startMs: Long, taskCounter: AtomicInteger, queue: DelayQueue[TimerTaskList]) {

  private[this] val interval = tickMs * wheelSize   //每1格的单位 ＊ 总格数（比如1格是1秒，60格，那总共也就能表达60s)

  //核心变量之1：每个刻度对应一个TimerTask的链表
  private[this] val buckets = Array.tabulate[TimerTaskList](wheelSize) { _ => new TimerTaskList(taskCounter) }

  ...
  //核心变量之2：parent TimingWheel
  @volatile private[this] var overflowWheel: TimingWheel = null

  private[timer] class TimerTaskList(taskCounter: AtomicInteger) extends Delayed {

  private[this] val root = new TimerTaskEntry(null) //链表的头节点
  root.next = root
  root.prev = root

  //每个TimerTaskEntry封装一个TimerTask对象，同时内部3个变量
private[timer] class TimerTaskEntry(val timerTask: TimerTask) {

  @volatile
  var list: TimerTaskList = null   //指向该链表自身
  var next: TimerTaskEntry = null  //后一个节点
  var prev: TimerTaskEntry = null  //前1个节点

 //因为同1个bucket(TimerTaskEntryList)里面的过期时间都相等，所以整个bucket记录了一个过期时间的字段expiration
    private[this] val expiration = new AtomicLong(-1L)

//除非该bucekt被重用，否则一个bucket只会有1个过期时间
  def setExpiration(expirationMs: Long): Boolean = {
    expiration.getAndSet(expirationMs) != expirationMs
  }
｝
```
   * Timer的三大核心功能
      * 添加：将一个TimerTask加入到Timer
      * 过期：时间到了，执行所有那些过期的TimeTask
      * 取消：时间未到，取消TImeTask，把TimerTask删除
   * TimingWheel的本质
     * DelayedQueue
     * 刻度盘的层次：currentTime

Kafka源码深度解析－序列14 －Server核心组件之3－SocketServer与NIO－ 1+N+M 模型
   * https://blog.csdn.net/chunlongyu/article/details/53036414
   * 入口KafkaServer
```java
  def startup() {
    try {

        ...
        //关键组件：SocketServer
        socketServer = new SocketServer(config, metrics, kafkaMetricsTime)
        socketServer.startup()

        ...
        //关键组件：KafkaApis
        apis = new KafkaApis(socketServer.requestChannel, replicaManager, consumerCoordinator,
          kafkaController, zkUtils, config.brokerId, config, metadataCache, metrics, authorizer)

        ...
        //关键组件：KafkaRequestHandlerPool
        requestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.requestChannel, apis, config.numIoThreads)

        ...
        }

        ...
      }
    }
    catch {
      case e: Throwable =>
        fatal("Fatal error during KafkaServer startup. Prepare to shutdown", e)
        isStartingUp.set(false)
        shutdown()
        throw e
    }
  }
```
   *  1 + N + M模型
      * 1个监听线程，负责监听新的socket链接
      * N个IO线程来负责对socket进行读写，N一般等于CPU核数
      * M个worker线程，负责处理数据
   * RequestChannel
      * 1个request队列
      * N个response队列
```java
      class RequestChannel(val numProcessors: Int, val queueSize: Int) extends KafkaMetricsGroup {

  ...
  //1个request Queue
  private val requestQueue = new ArrayBlockingQueue[RequestChannel.Request](queueSize)

  //N个response Queue
  private val responseQueues = new Array[BlockingQueue[RequestChannel.Response]](numProcessors)
  for(i <- 0 until numProcessors)
    responseQueues(i) = new LinkedBlockingQueue[RequestChannel.Response]()
  ...
```
   * KafkaRequestHandlerPool的run函数
```java
class KafkaRequestHandlerPool(val brokerId: Int,
                              val requestChannel: RequestChannel,
                              val apis: KafkaApis,
                              numThreads: Int) extends Logging with KafkaMetricsGroup {
  ...
  val threads = new Array[Thread](numThreads)
  val runnables = new Array[KafkaRequestHandler](numThreads)
  for(i <- 0 until numThreads) {
    runnables(i) = new KafkaRequestHandler(i, brokerId, aggregateIdleMeter, numThreads, requestChannel, apis)
    threads(i) = Utils.daemonThread("kafka-request-handler-" + i, runnables(i))
    threads(i).start()
  }

class KafkaRequestHandler(id: Int,
                          brokerId: Int,
                          val aggregateIdleMeter: Meter,
                          val totalHandlerThreads: Int,
                          val requestChannel: RequestChannel,
                          apis: KafkaApis) extends Runnable with Logging {
  this.logIdent = "[Kafka Request Handler " + id + " on Broker " + brokerId + "], "

  def run() {
    while(true) {
      try {
        var req : RequestChannel.Request = null
        while (req == null) {
          // We use a single meter for aggregate idle percentage for the thread pool.
          // Since meter is calculated as total_recorded_value / time_window and
          // time_window is independent of the number of threads, each recorded idle
          // time should be discounted by # threads.
          val startSelectTime = SystemTime.nanoseconds
          req = requestChannel.receiveRequest(300) //从队列中取出request

          val idleTime = SystemTime.nanoseconds - startSelectTime
          aggregateIdleMeter.mark(idleTime / totalHandlerThreads)
        }

        if(req eq RequestChannel.AllDone) {
          debug("Kafka request handler %d on broker %d received shut down command".format(
            id, brokerId))
          return
        }
        req.requestDequeueTimeMs = SystemTime.milliseconds
        trace("Kafka request handler %d on broker %d handling request %s".format(id, brokerId, req))
        apis.handle(req)  //处理结果，同时放入response队列
      } catch {
        case e: Throwable => error("Exception when handling request", e)
      }
    }
  }
}
```
   * mute/unmute机制：消息有序性的保证
      * 在processor的run函数中，有一个核心机制：mute/unmute，该机制保证了消息会按照顺序处理，而不会乱序

Kafka源码深度解析－序列15 －Log文件结构与flush刷盘机制
   * https://blog.csdn.net/chunlongyu/article/details/53784033
   * 每个topic_partition对应于一个目录
      * log.dir
   * 文件的offset作为meissageId
   * 变长消息存储
   * flush刷盘机制
      * 将数据写入文件系统之后，数据其实是在pagecache里面，并没有刷到磁盘上，如果此时操作系统挂了，数据也就丢失了。
      * 一方面fsync系统调用强制刷盘，另一方面，操作系统后台线程，定期刷盘。
      * log.flush.interval.message
      * log.flush.interval.ms
      * log.flush.scheduler.interval.ms
   * 多线程写同一个log文件
      * lock
   * 数据文件分段 + 索引文件（稀疏索引）