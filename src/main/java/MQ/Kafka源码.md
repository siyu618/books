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
      * isShuttingDown.set(false)
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
   * processDisconnected()

### 4. requestHandler pool ： requestHandler
1. 死循环 从 requestChannel 的 requestQueue 中获取 request
2. 两种请求
   * ShutdownRequest：等待 shutdownComplete.countDown
   * RequestChannelRequest：
      * 调用 apis.handle(reqeust)

### 5. kafkaController.startup()
* zkClient 注册 StateChangeHandler
* eventManager.put(Startup)
   * zkClient.registerZNodeChangeHandlerAndCheckExistence(controllerChangeHandler)
      * handleCreation: eventManager.put(ControllerChange)
         * 如果之前是 leader ，而现在不是 leader，则退位：onControllerResignation
            * zkClient 取消监听一些事件
            * kafkaScheduler.shutdown
            * etc
      * handleDeletion: eventManager.put(Reelect)
         * 如果之前是 leader 而现在不是了，则退位 onControllerResignation
         * 选举 elect
            * 如果其他 broker 已经成为 Controller 则退出
            * 尝试创建临时节点
               * 成功则上位：onControllerFailover
                  * 1. 注册 controller epoch 和 变更监听器
                  * 2. 增加 epoch
                  * 3. 初始化 controller 的 context ： 当前 topics、活跃的 brokers、所有 partition 的 leader
                  * 4. 启动 controller 的 channel manager
                  * 5. 启动 replica 状态机
                  * 6. 启动 分区 状态机
               * 失败：
                  * 创建节点失败，说明有其他 broker 上位，打日志
                  * 上位失败则 退位：triggerControllerMove
                     * onControllerResignation
                     * deleteController: 删除节点
      * handleDataChange: eventManager.put(ControllerChange)


## 2. Kafka APIS
### 1. handleProduceRequest: ApiKeys.PRODUCE
1. 首先判定是否是 事务 且 是授权失败，则返回
2. 其次判定是否是 幂等 且 授权失败，则返回
3. 过滤出 未授权（unauthorizedTopicResponse） 和 不存在的（nonExistingTopicResponse） 和 可以正常处理的数据（authorizedRequestInfo）
4. 定义 callback ：sendResponseCallback
   * 构建 response 对象，并加入 对应的 processor 的 responseQueue 中，并唤醒 selector
5. 调用 replicaManager.appendRecords(timeout, requiredAcks, internalTopicsAllowed, isFromClient=true, responseCallback = sendResponseCallback, recordConversionStatsCallback = processingStatsCallback)
   * 如果 acks 不合法（0，-1，1），构建 response 并调用回调函数
   * 否则
      1. 写入本地 appendToLocalLog(), 得到 localProduceResults
         * 拒绝不可写入的内部 topic，返回异常信息。
         * 获取 partition 信息，并判定是本地是否是 leader
         * 将记录写入 leader ： partition.appendRecordsToLeader(records, isFromClient = true, requiredAcks)
            * 写入本地 leaderReplica.log.get.appendAsLeader(records, leaderEpoch, isFromClient = true)
               * append 到 active segment，并分配 offset：log.append(records, isFormClient, assignOffsets = true, leaderEpoch)
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
            * 更新 highWatermark（所有 replica 中的 LEO 的最小值）
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

### 3. handleFindCoordinatorRequest: ApiKeys.FIND_COORDINATOR
1. 如果授权失败，直接返回错误
2. 获取 metadata
   * 根据 groupid 获取内部 topic 对应的 partition =  groupCoordinator.partitionFor(findCoordinatorRequest.coordinatorKey) 
      * groupManager.partitionFor(group)
         * Utils.abs(groupId.hashCode) % groupMetadataTopicPartitionCount
   * 获取内部 topic （"__consumed_offsets"）的 metadata ： getOrCreateInternalTopic
   * partition 对应的 leader 就是该 group 的 coordinator

### 4. handleJoinGroupRequest: ApiKeys.JOIN_GROUP
1. 授权失败直接返回错误
2. 由 groupCoordinator 来处理 join-group 请求： groupCoordinator.handleJoinGroup
   * 异常情况直接返回错误
   * 如果 group 不在 cache 中就加入到 cache 中： groupManager.addGroup()
   * doJoinGroup
      * 根据 group.currentState 处理
         * Dead：返回错误
         * PreparingRebalance：
            * memberId 不存在则 addMemberAndRebalance
               * group.add ： 第一个加入的 client 被设置为 leader
               * maybePrepareRebalance
                  * 如果有成员在异步等待，取消它们并同时它们 rejoin：resetAndPropagateAssignmentError
            * 否则 updateMemberAndRebalance
               * updateMember：
               * maybePrepareRebalance
         * CompletingRebalance               
            * memberId 不存在则 addMemberAndRebalance
            * 否则 
               * 如果是 leader ：
               * 否则
                  * 如果 protocol 匹配
                     * 返回结果，对于 leader 需要返回 group 的 currentMemberMetadata，对于 follower 返回空。 
                  * updateMemberAndRebalance
         * Empty | Stable
            * memberId 不存在则 addMemberAndRebalance
            * 否则 
               * 如果是 leader
                  * updateMemberAndRebalance
               * 否则即 follower
                  * 返回结果
           
### 5. handleSyncGroupRequest: ApiKeys.SYNC_GROUP
1. 如果 memberId 不存在，返回 UNKNOWN_MEMBER_ID 错误
2. 如果 generationId 不匹配，返回 ILLEAGLE_GENERATION 错误
3. 查看 group.currentState
   * Empty | Dead : 返回 UNKNOWN_MEMBER_ID 错误
   * PreparingRebalance ： 返回 REBALANCE_IN_PROGRESS 错误
   * CompletingRebalance： 
      * 如果是 leader ，将 state 落地： groupManager.storeGroup
         * appendForGroup
            * replicaManager.appendRecords // 落盘
   * Stable：返回 metadata
      * completeAndScheduleNextHeartbeatExpiration
         * 完成此次 heartbeat，并调度下次 heartbeat            

### 6. handleOffsetFetchRequest: ApiKeys.OFFSET_FETCH
1. createResponse 
   * 如果未授权直接返回
   * 如果 header.apiVersion == 0，则从 Zookeeper 获取数据
   * 否则 从 Kafka 获取 offset
      * 如果 request 的 partitions 字段为空，groupCoordinator.handleFetchOffsets(offsetFetchRequest.groupId)
         * groupManager.getOffsets : 不能返回 陈旧 的 offset
            * 
      * 否则 groupCoordinator.handleFetchOffsets(offsetFetchRequest.groupId, Some(authorizedPartitions))

### 7. handleOffsetCommitRequest： ApiKeys.OFFSET_COMMIT
1. 授权不合法，直接返回错误
2. header.apiVersion === 0， 保存到 Zookeeper
3. header.apiVersion > 0: groupCoordinator.handleCommitOffsets
   * groupCoordinator.doCommitOffsets()
      *  groupManager.storeOffsets: 写入到 replication log，然后放入 cache
         * appendForGroup
            * replicaManager.appendRecords ;// 落地
            * putCacheCallback // 更新缓存
            

### 8. [Kafka Zero-Copy 使用分析](https://www.jianshu.com/p/d47de3d6d8ac)
* 前言
   * NIO
   * Zero Copy
   * 磁盘顺序写
   * Queue 数据结构的极致使用
* Kafka 在什么场景下是用 Zero Copy
   * 消费消息， Fetch。
   * Consumer 以及 Follower 从 leader partition 拉取数据的时候。 
   * API： java.nio.FileChannel.transferTo(long position, long count, WritableByteChannel target)
* Kafka 使用 Zero-Copy 的流程分析
   * 数据成成
      * 1. KafkaApis.handle(): case ApiKeys.FETCH => handleFetchRequests(request)      
         * replicaManager.fetchMessage()
         * val logReadResults = readFromLog()
            * val result = readFromLocalLog( // 返回  Seq\[(TopicPartition, LogReadResult)]
               * localReplica.log.read() // log 是 kafka.cluster.Log
                  * segment.read(startOffset
                     * 返回 FetchDataInfo 
                        * fetchOffsetMetadata: LogOffsetMetadata,
                        * records:Records
                           * **writeTo() ==>  bytesTransferred = tl.transferFrom(channel, position, count); ==>  fileChannel.transferTo(position, count, socketChannel)**
                        * firstEntryIncomplete:Boolean
                        * abortedTransactions
         * processResponseCallback(responsePartitionData: Seq\[(TopicPartition, FetchPartitionData)])
              * 数据转化：Seq\[(TopicPartition, FetchPartitionData)] --> partitions ： LinkedHashMap[TopicPartition, FetchResponse.PartitionData[Records]] 
              * 如果请求来自 follower
                 * 数据转换 unconvertedFetchResponse:FetchResponse\[Records] = fetchContext.updateAndGenerateResponseData(partitions)
                    * FetchResponse\[Records] 成员 LinkedHashMap<TopicPartition, PartitionData<T>> responseData 存放数据
                       * PartitionData 里的 records 存放了实际取的数据
                 * createResponse
                    * 对 unconvertedFetchResponse 里面的 PartitionData 转换
                    * 重新构建 response:FetchResponse
                 * sendResponseExemptThrottle: sendResponse
                    * 构建 responseSend = request.context.buildResponse(response)
                    * 构建 SendResponse(request, responseSend, responseAsString, onCompleteCallback)
                    * sendResponse(response)
                       * requestChannel.sendResponse(response)
                          * 获取对应的 processor，加入到其对应的 responseQueue 中 
   * 数据发送: Processor.run()
      * 注册发送：processNewResponses()     
         * Processor.selector.send(response)
            * channel.setSend(send)
               * this.send = send
               * 注册写 this.transportLayer.addInterestOps(SelectionKey.OP_WRITE);
      * 实际发送：poll()
         * pollSelectionKeys()
            * send = channel.write(); // Send 为 MultiRecordsSend
               * KafkaChannel.send(send)
                  * send.writeTo(transportLayer)
                     * 几层调用到：FileRecords 的 writeTo
                        * PlaintextTransportLayer.transferFrom(channel, position, count)
                           * fileChannel.transferTo(position, count, socketChannel); **Zero-Copy happens here**



            