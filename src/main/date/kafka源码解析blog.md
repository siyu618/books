
服务熔断、降级、限流、异步RPC -- HyStrix
   * https://blog.csdn.net/chunlongyu/article/details/53259014

分布式架构－－基本思想汇总
   * https://blog.csdn.net/chunlongyu/article/details/52525604   


Kafka源码深度解析－系列1 －消息队列的策略与语义
   * https://blog.csdn.net/chunlongyu/article/details/52538311

Kafka源码深度解析－序列2 －Producer －Metadata的数据结构与读取、更新策略
   * https://blog.csdn.net/chunlongyu/article/details/52622422

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





