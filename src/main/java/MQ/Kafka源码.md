
## 1. Kafka Server
### 1. Kafka Server 
1. Kafka shutdown
   * isStartingUp：正在启动，则抛出异常
   * 如果 shutdownLatch>0 && isShuttingDown.compareAndSet(false, true) 则进行 shutdown
      * brokerState 标记为 BrokerShuttingDown
      * 关闭 dynamicConfigManager
      * socketServer 停止处理请求
      * 关闭 requestHandlerPool
      * 关闭 kafkaScheduler
      * 关闭 kafkaApis
      * 关闭 authorizer
      * 关闭 adminManager
      * 关闭 transactionCoordinator
      * 关闭 groupCoordinator
      * 关闭 tokenManager
      * 关闭 ReplicaManager
      * 关闭 zkClient
      * 关闭 quotaManagers
      * 关闭 socketServer ：
      * 关闭 metrics
      * 关闭 brokerTopicStats
      * brokerState 标记为 NotRunning
      * startupComplete.set(false)
      * isShuttingDhown.set(false)
      * AppInfoParser.unregisterAppInfo(jmxPrefix, brokerId, metrics)
      * shutdownLatch.countDown()
2. Kafka startup
   * isShuttingDown : 正在关闭，则抛出异常
   * startupComplete : 已经启动，则退出
   * isStartingUp.compareAndSet(false, true) 为 false 表示已经在启动，则退出
   * 启动
      * brokerState 标记为 Starting
      * setup Zookeeper： initZKClient
         * chroot，创建 zkClient，创建zk中顶级路径
      * get or create ClusterId ： getOrGenerateClusterId
      * generate brokerId：getBrokerIdAndOfflineDirs
      * initialize dynamic broker configs ： config.dynamicConfig.initialize(zkClient)
      * 初始化 scheduler：new KafkaScheduler(config.backgroundThreads).startup()
      * 初始化 metrics
      * 初始化 logManager：logManager.startup()
         * flush logs && log clean up && log retention && check point
      * 初始化 socketServer：socketServer.startup(startupProcessors = false)
         * 启动，但是不开始处理请求，在整个 kafka server 启动最后开始启动处理
      * 初始化 replicaManager
         * start ISR expiration thread
      * 注册 broker 到 zk path && save checkpoint
      * 初始化 kafkaController
         * 通过 zk 选举
      * 初始化 groupCoordinator
         * 清理过期信息
      * 初始化 transactionCoordinator
      * 初始化 KafkaApis
         * KafkaController、socketServer.requestChannel、ReplicaManager、groupCoordinator、transactionCoordinator、fetchManger
      * 初始化 requestHandlerPool
         * 每个 requestHandler，从 request 队列中取请求并处理 apis.handle(request)
      * socketServer 开始处理请求：socketServer.startProcessors() 
         * configureNewConnections(): register new connections
         * processNewsResponse(): register any new response for writing
         * selector.poll(): 处理所有的 IO 事件，结束建立连接、结束关闭连接、初始化新的发送、处理 发送和接受 数据
            * 建立连接
            * 读。将请求加入 requestQueue
            * 写。pollSelectionKeys(): 发送数据如果有，并添加到 completedSends
         * processCompletedReceives(): 
            * selector.mute(connectionId)
         * processCompletedSends(): 
            * 从 inflightResponse 中删除
            * 回调 completion callback
            * unmute channel
         * processDisconnected():
      * brokerState.newState(RunnningAsBroker)
      * startupComplete.set(true)
      * isStartingUp.set(false)


### 2. socketServer.startup(startupProcessors = false)
* 获取 对象锁，创建 acceptor 和 processors
   * acceptor： 负责处理外部链接，并未每个链接分配一个 processor，并加入到 processor 的 newConnections 队列
      * nioSelector
      * serverChannel
      * processors：轮转分配处理 acceptor 的连接
      
### 3. socketServer.startProcessors()
* processor 对象的成员
   * newConnections 队列
   * inflightResponses map
   * responseQueue
* run：死循环处理
   * configureNewConnections() ：处理新的连接
      * channel = newConnections.poll()
      * selector.register(xxx, channel)
   * processNewsResponses()：处理 responseQueue 中的数据
      * 循环 dequeue responseQueue
      * 设置每个 channel 的 send，每次只能发送一个数据
   * poll()
      * selector.poll()
         * keyswithBufferedRead: poll from channels that have buffered data
         * readyKeys：有读写 的 channel
         * immediatelyConnectedKeys ： 刚刚建立的连接
         * pollSelectionKeys(): 处理 I/O 事件
            * shuffle keys：避免 饥饿
            * 建立连接 对于 connect 事件
            * 尝试读，并加入到 stageReceives 中
            * 写 channel.write() ，如果写完就加入到 completedSends 中
            * 关闭不合法的 channel
         * 关闭最老的连接
         * addToCompletedReceives：将已经
   * processCompletedReceived()
   * processCompletedSends()
   * processDisconnnected()

### 4. requestHandler pool ： requestHandler
1. 死循环 从 requestChannel 的 requestQueue 中获取 request
2. 两种请求
   * ShutdownRequest：等待 shutdownComplete.countDown
   * RequestChannelRequest：
      * 调用 apis.handle(reqeust)


## 2. Kafka APIS
### 1. handleProduceRequest: ApiKeys.PRODUCE
1. 首先判定是否是 事务 且 是授权失败，则返回
2. 其次判定是否是 幂等 且 授权失败，则返回
3. 过滤出 未授权（unauthorizedTopicResponse） 和 不存在的（nonExistingTopicResponse） 和 可以正常处理的数据（authorizedRequestInfo）
4. 定义 callback ：sendResponseCallback
   * 构建 response 对象，并加入 对应的 processor 的 responseQueue 中，并唤醒 selector
5. 调用 replicaManager.appendRecords(timeout, requiredAcks, internalTopicsAllowed, isFromClinet=true, responseCallback = sendResponseCallback, recordConversionStatsCallback = processingStatsCallback)
   * 如果 acks 不合法（0，-1，1），构建 response 并调用回调函数
   * 否则
      1. 写入本地 appendToLocalLog(), 得到 localProduceResults
         * 拒绝不可写入的内部 topic，返回异常信息。
         * 获取 partition 信息，并判定是本地是否是 leader
         * 将记录写入 leader ： partition.appendRecordsToLeader(records, isFromClinet = true, requiredAcks)
            * 写入本地 leaderReplica.log.get.appendAsLeader(records, leaderEpoch, isFromClient = true)
               * append 到 active segment，并分配 offset：log.append(records, isFormclient, assignOffsets = true, leaderEpoch)
                  1. 获取新的 offset
                  2. 更新 epoch 缓存
                  3. 检查 记录的大小 是否超过 允许的最大的大小
                  4. 收集  producer 的 metadata
                  5. 看看是否需要 生成新的 segment
                  6. 写入 segment：segment.append()
                     * append the messages : log.append(records)
                        * 写入 channel：records.writehFullTo(channel)
                     * update the in memory max timestamp and corresponding offset
                     * append an entry to the index(if needed)
                  7. 更新 producer 状态
                  8. 更新 transaction 信息
                  9. 更新 producerStateManger.updateMapEndOffset
                  10. 更新 LEO
                  11. 更新 first unstable offset
                  12. 如果满足条件，flush 到磁盘
                     * log.flush
                     * offsetIndex.flush
                     * timeINdex.flush
                     * txnIndex.flush
            * 激活 follower 的 fetch ： replicaManager.tryCompleteDelayedFetch()
            * 更新 highWatermark（所有relica中的 LEO 的最小值）
            * 如果 highWatermark 更新了，尝试结束一些 pending 的 request。（此时不需要获取 leaderIsrUpdateLock）
               * replicaManager.tryCompleteDelayedFetch()
               * replicaManager.tryCompleteDelayedProduce()
               * replicaManager.tryCompleteDelayedDeleteRecords()
         * 更新统计信息，并返回结果（topicPartition, LogAppendResult（info））
      2. 更新统计信息 recordConversionStatsCallback
      3. 如果需要被延时处理（acks = -1 && 有数据需要append && 至少一个分区成功了），
         * 构建延时 produce，并放入delayedProducePurgatory
            * tryComplete // 可能此时已经处理成功了
            * add to watcher
      4. 否则：立即调用回调函数 responseCallback 即 sendResponseCallback
6. 清理 produceRequest
      
### 2. handleFetchRequest: ApiKeys.FETCH  
1. 构建 fetchContext
   * 对于来自 follower 的请求：have clusterAction on cluster resource   
   * 对于来自 consumer 的请求：have READ permission 
2. 定义 maybeConvertFetchedData 函数
   * record batch 的版本，在不同的版本中压缩不同，故而在 C/S 之间需要适配
3. 定义 processResponseCallback    
   * 如果是来自 follower 的请求，更新统计信息；
   * 如果是来自 client 的请求，也更新统计信息
   * 发送数据
4. replicaManager.fetchMessages(): 从 leader replica 读取数据，等待数据或者超时的发生
   * readFromLocalLog
      * 对每个 topicPartition 调用 read
         * 决定是否只从 leader replica 读取数据
         * 决定是否只读取 committed 的数据还是 high watermark的数据
         * 获取 logReadInfo
            * log.read（）：从 log 读取数据
               * 获取 startOffset 对应的 segmentEntry
               * 如果 （startOffset > next || segmentEntry == null || startOffset < logStartOffset)，返回异常
               * 依次从 segmentEntry.read()中读取数据，直到读到数据。
                  * 获取文件中实际的log.slice(startPosition，fetchSize)
                     * 返回 fileRecords
                     * 此时不读磁盘
            * 如果 leader throttled，返回空数据
            * 对于 FetchRequest version 3， 未满的数据直接返回空
         * 返回 logReadResult
   * 如果（请求不想等待 || 请求没有需要任何数据 || 有足够的数据来返回 || 出错了），则立即调用 responseCallback 
   * 否则，放入 delayedFetchPurgatory
      
###       
      
## Kafka Producer
### Kafka Producer
1. 线程安全，建议是用单个 producer       
   * metadata: 存储 topic 的数据
   * accumulator：存储发送的记录，RecordAccumulator
   * client：NetworkClinet
   * sender：Sender
      * kafkaClient
      * metadata
      * accumulator
   * ioThread(this.sender).start
   
2. KafkaProducer.send(ProducerRecord<K,V> record, Callback callback)
   * interceptedRecord = this.interceptors.onsend(record)
   * doSend(interceptedRecord, callback)
      * 如果 producer 关闭就退出
      * 获取 topic 对应的 metadata：waitOnMetadata（）
      * 序列化 key 和 value
      * 计算 record 对应的 partition：partition（）
         * partitioner.partition(record.topic, record.key(), serializedKey, record.value(), serializedValue, cluster)
            * keyValue == null, 调用 nextValue(topic) % numPartitions
            * keyValue != null, Utils.murmur2(keyBytes) % numPartitions
      * 估算 record 的大小，如果超出则抛出异常
      * 加入到队列：accumulator.append(tp, timestamp, serializedKey, serializedValue, headers, interceptCallback, remainingWaitMs);
      * 如果 batch 满或者有新的 batch 生成，唤醒 sender      


## Kafka Consumer
### Kafka Consumer
1. 非线程安全
2. KafkaConsuemr.poll(final Timer timer, final boolean includeMetadataInTimeout)
   * 获取锁// 单线程使用
   * 如果没有订阅，则抛异常
   * 拉取数据，直到超时
      * 如果 includeMetadataInTimeout 为 true，首先调用 updateAssignmentMetadataIfNeeded(Timer timer)，如果返回 false，直接返回空集合
         * coordinator.poll(timer) ： 确保知道 coordinator，并且 consumer 已经加入 group，和处理定期提交 offset
            * 处理提交回调 invokeCompletedOffsetCommitCallbacks
            * 如果是自动分配 partittion
               * pollHeartbeat(): 更新 线程的更新时间，使得 AbstractCoordinator.HeartbeatThread 线程可以继续执行 heartbeat 操作
               * 如果 coordinatorUnknown（）获取coordinator失败 同时 ensureCoordinatorReady 也失败 则返回 false
                  * ensureCoordinatorReady().lookupCoordinator() : 向最小负载的机器发送请求以获取 coordinator 节点
                  * 如果 rejoinNeededOrPending() 返回true
                     * subscriptions.hasPatternSubscription()： 因为 初始的 metadata 获取和 初始的 rebalance 存在竞争，需要保证在初始的 joining 之前获取新的 metadata。这就需要我们首先获取集群中匹配 pattern 的 topic 数据。
                        * 获取 metadata
                  * ensureActiveGroup() : 确保 group is active， joined and synced
                     * ensureCoordinatorReady：再其次确保 group ready
                     * startHeartbeatThreadIfNeeded: 启动 heartbeat 线程
                     * joinGroupIfNeeded() : 加入 group ，而不启动 heartbeat 线程
                        * rejoinNeededOrPending ： 如果需要加入或者有 pending 的请求
                           * 确保 coordinator 可用
                           * onJoinPrepare（）：提交之前的 offset，并且清理 
                              * maybeAutoCommitOffsetsSync ： 先提交 commited offsets 如果 自动提交开启的话。
                           * initiateJoinGroup(): 
                              * 禁用 heartbeat 线程
                              * sendJoinGroupRequest() : 发送 JGR & SGR
                                 * JoinGroupResponseHandler
                                    * 如果返回是 leader： onJoinLeader
                                       * performAssignment：执行分配
                                          * 设置为 leader = true
                                          * 监控所有的 topic
                                          * 执行分配：RangeAssigner、RoundRobinAssigner、StickyAssigner
                                       * sendSyncGroupRequest ：向 coordinator 返回分配结果（非空结果）
                                          * SyncGroupResponseHandler
                                             * 保存分配结果
                                    * 否则是 follower： onJoingFollower
                                       * sendSyncGroupRequest：返回分配结果（空结果）
                             * 激活 heartbeat 线程
            * 否则是手动分配则等待 metadata
               * 如果没有 ready 的 nodes，则 等待 metadata
            * maybeAutoCommitOffsetsAsync ： 异步提交 commited offset
               * doAutoCommitOffsetsAsync
         * 更新 fetch position：updateFetchPositions(Timer timer)
      * 调用并等待 updateAssignmentMetadataIfNeeded 返回true
      * pollForFetches：fetch 数据，并且更新 consumed position
         * fetcher.fetchedRecords() 如果非空，则返回
            * parseCompletedFetch（）： 从 completedFetches 中获取数据并放到 nextInLineRecords 中
               * 更新 subscriptions
            * 从 nextInlineRecords 中抽取 record 放入 map fetched 中，并更新 remaining count
            * 如此循环直到获得足量的数据 或者 没有数据可以获取
            
         * 发送请求获取数据：fetcher.sendFetches(); 请求不重发
            * prepareFetchRequests（）： 构建请求
            * client.sendRequest(), 注册回调函数 RequestFeatureListener
               * 将返回的结果加入到 completedFetches 中
         * 如果 coordinator 需要 rejoin 则直接返回空
         * return fetcher.fetchedRecords();
      * 如果拉去数据非空需要在返回数据之前，可以发送下次获取数据请求，因为已经提交了 consumed offset。
      * 返回结果：需要经过 interceptors 处理
   * 释放锁

      
      
             