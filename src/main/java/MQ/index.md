### MQ
1. [Kafka](http://kafka.apache.org/)
   * [detail](Kafka.md)
2. [RocketMQ](http://rocketmq.apache.org/)
   * [detail](RocketMQ.md)
3. [Pulsar](http://pulsar.apache.org)
   * [detail](Pulsar.md)

### Q & A
1. 这几个的各有什么特点
   * 对于 Kafka 和 RocketMQ 两者都不能保证 exactly once
   * Kafka 通过 Replication 来保证数据的冗余，而 RocketMQ 通过 Slave 来保证数据的冗余

|条目| Kafka | RocketMQ | Plusar |
|---|---|---|---|
|架构|单层架构，Broker 也负责存储|单层架构，Broker 也负责存储|存储和服务分离，Broker 负责提供服务，BookKeeper 提供存储能力|
|数据冗余| Replication (ISR) | Slave (只能从属于一个Master) |BookKeeper 高可用存储|
|Consumer的消费|每个 partition 只能被一个 consumer 消费，只能消费 leader|每个 partition 只能被一个 consumer 消费，也可以消费 Slave|允许 consumer 数量超过 partition 数量|
|使用Zookeeper|Yes（减少了对 Zookeeper 的依赖）|No，使用 MetaServer|Yes，local Zookeeper & global Zookeeper|
|支持服务端过滤（减少数据传输）|No|Yes|No|
|支持 exactly once|No，需要业务端保证|No，需要业务端保证|No，需要业务端保证|
|支持事务|No|Yes|No|
|订阅模式|集群消费|集群消费，广播消费|Exclusive，Shared，Failover 三种模式|

2. [开发者说：消息队列 Kafka 和 RocketMQ 之我见](https://mp.weixin.qq.com/s/zeVuoNxRsWzM8otxFEdSVw)
   * 三种消息协议
      * JMS（Java Message Service）
      * AMQP（Advanced Message Queuing Protocol）
      * MQTT（Message Queuing Telemetry Transport）
   * push/pull 
   * 三种消息级别
      1. 至多一次：处理日志，允许丢失
      2. 至少一次：ACK 确认机制，消费者根据业务逻辑自己实现去重或幂等。
      3. 恰好一次：
         * 四次交互
         
 ```
 producer        client
     ------1. msg----------->  (write id)    ### 发送消息     
     <-----2. send back REC--                ### 写id，告知已收到
     ------3. rel ----------->               ### 知道你收到，可以被消费了
     <-----4. comp----------->  (删除id)      ### 删除id，我知道了
```
   * 数据可靠性
       * 主从
   * 服务可用性保障：复制与 failover
   * 消息队列的高级特性
      * 顺序消息
      * 事务消息
      * 消息回放
   * 影响单机性能的因素
      * 硬件层面：
         * 硬盘：NVMe > 传统 SSD > SAS > SATA
         * Raid 卡：
      * 系统层面
         * Raid 级别
         * Linux I/O 调度算法： Noop、Deadline、CFG
         * 文件系统的 block size
         * SWAP 的使用，建议关闭
   * 应用层面：
      * 文件读写的方式：顺序读写的速度原告与随机读写。
      * 缓存策略。

