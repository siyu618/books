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





