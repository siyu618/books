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




















