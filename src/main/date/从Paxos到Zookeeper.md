**Chapter 1:分布式架构 **

1.1 从集中式到分布式
   * 集中式
       * 部署简单
   * 分布式
       * 分布性，对等性，并发性，缺乏全局时钟，故障总是会发生
   * 分布式的各种问题
       * 通信异常
       * 网络分区
       * 三态：成功、失败、超时
      
1.2 从ACID到CAP/BASE
   * ACID
      * Atomicity：全部执行或者全部为执行
      * Consistency：一致性
      * Isolation：隔离性，事务之间是相互隔离的
         * 未授权读取
         * 授权读取
         * 可重复读取
         * 串行化
      * Durability：一旦提交，即是永久
   * CAP
      * 分布式系统不可能同时满足一致性（Consistency）、可用性（Availability）和分区容错（Partion tolerance），最多只能同时满足两个
      * C：多个副本之间保持一致性
      * A：请求在有限时间内返回结果
      * P：遇到网络分区故障时，仍然你需要对外提供满足一致性和可用性的服务，除非是整个网络环境都发生了变化
      * 放弃CAP
         * 放弃P：为了避免系统出现分区容错性问题，简单做法是将所有的数据都放在一个分布式节点上。放弃P的同时，也就放弃了系统的可扩展性
         * 放弃A：放弃可用性是一旦系统遇到网络分区或者其他故障时，那么受到影响的服务需要等待一定的时间，因此在等待恢复的时间里，服务不可用。
         * 放弃C：是指放弃强一致性，从而保留系统的最终一致性
      * 结论
         * 分区容错是一个基本需求
         * 系统架构师需要经历花在如何根据业务特点在C和A之间寻求平衡。
   * BASE
      * 无法做到强一致性，单每个应用都可以根据自身的业务特点，采取适当的方式来时系统达到最终一致性
      * 基本可用（Basically Available）
         * 系统仍然可用
            * 响应时间上的损失
            * 功能上的损失
      * 弱状态/软状态（Soft state）
         * 允许系统当中的数据存在中间状态，并认为该中间状态的存在不会影响系统的整体可用性
      * 最终一致性（Eventually consistent）
         * 强调的是系统中所有的数据副本，在经过一段时间的同步后，最终能够达到一个一致的状态
         * 五种主要变种
            * 因果一致性
            * 读己之所写
            * 会话一致性
            * 单调读一致性
            * 单调写一致性
      * 面向大型高可用可扩展的分布式系统 
      


**Chapter 2:一致性协议**

经典的一致性

2.1 2PC & 3PC

为了保证事务处理的ACID特性，就需要引入一个称谓协调者（Coordinator）的组件来统一调度所有分布式节点的执行逻辑，这些被调度的分布式节点被称谓参与者（Participant）。
   * 2PC ： Two-Phase Commit
      * 目前绝大部分数据库采用两阶段提交来完成分布式事务处理，利用该协议能方便完成所有分布式事务的协调，统一决定事务的提交或回滚，从而能够有效地保证分布式数据库一致性。
      * 将事务分解为两个阶段来处理
      * 阶段1：提交事务请求 （投票阶段）
         1. 事务查询：协调者向所有的参与者发送事务内容，询问是否可以执行事务提交操作，并开始等待参与者的相应。
         1. 执行事务：各参与节点执行事务操作，并将Undo和Redo信息记入事务的日志中。
         1. 各参与者向协调者反馈事务询问的响应
            * 如果可以成功执行返回YES，否吃返回NO
      * 阶段2：执行事务提交 （Commit）
         1. 执行事务提交：如果所有参与者返回的是都是YES，则会执行事务
            * 发送提交请求：协调者向所有参与者节点发出Commit请求
            * 事务提交：参与者受到commit请求，会正式执行事务提交操作，并在完成提交之后释放在整个事务执行期间占用的事务资源
            * 反馈事务提交结果：向协调者反馈ACK消息
            * 完成事务：done
         1. 中断事务：任一参与者想协调者反馈了NO
            * 发送回滚请求：协调者发送Rollback请求
            * 事务回滚：参与者利用Undo信息来执行事务的回滚操作
            * 返回事务回滚的结果：向协调者发送ACK
            * 中断事务：协调者接收到所有参与者反馈的ACK消息后，完成事务中断
       * 优缺点
          * 优点：原理简单，实现方便
          * 缺点：同步阻塞、单点问题（协调者）、脑裂、太过保守（没有容错机制）
   * 3PC：Three-Phase Commit
      * canCommit，preCommit，do Commit
      * canCommit
         * 事务询问：协调者向参与者发送包含事务内容的canCommit请求
         * 各参与者向协调者反馈事务询问的相应，YES or NO
      * PreCommit
         * 执行事务的预提交：协调者获得了所有的YES
            1. 发送预提交请求：协调者向参与者发送preCommit请求，并进入prepared状态
            1. 事务预提交：参与者接收preCommit请求，会执行事务操作，并将Undo和Redo信息记录到事务日志中
            1. 各参与者向协调者反馈事务执行的响应：如果参与者成功执行了事务操作，则会返回ACK，同时等待最终的指令：提交（commit）或终止（abort）
         * 中断事务：协调者或者至少一个NO
            1. 发送中断请求：发送abort请求
            1. 中断事务：收到abort请求，或者等待协调者超时，参与者都会中断事务
      * doCommit
         * 执行提交
            1. 发送提交请求
            1. 事务提交
            1. 反馈事务提交结果
            1. 完成事务
         * 中断事务
            1. 发送中断请求
            1. 事务回滚
            1. 反馈事务回滚结果
            1. 中断事务
      * 优缺点
         * 相对于2PC，降低了参与者的阻塞范围，并能再出现单点故障后继续达成一致
         * 缺点：出现网络分区后，出现数据不一致的情况 

2.2 Paxos算法
   * 理论上讲：分布式计算机领域，试图在异步系统和不可靠的通道上来达到一致性状态是不可能的。（拜占庭将军问题）
   * 问题描述
      假设有一组可以提出提案的进程集合，都那么对于一个一致性算法来说需要保证一下几点
         1. 在这些被提出的提案中，只有一个会被选定
         1. 如果没有提案被提出，也就不会有被选定的提案
         1. 当一个提案被选定后，进程应该可以获取被选定的提案的信息
      对于一致来说，安全性需求如下
         1. 只有被提出的提案才能被选定
         1. 只有一个值被选定
         1. 如果某个进程认为提案被选定了，那么这个必须是真的被选定的那个
   * 三种参与者：Proposer， Acceptor和Learner
   * 提案的选定
      * 单一Acceptor，存在单点问题
      * 多个Acceptor，Proposer向一个Acceptor集合发送提案，同样集合中的每个Acceptor都可能会批准（Accept）该提案的时候，则认为该提案被选定了。
         * Acceptor集合为超过一半
   * 推导过程
      * P1: 一个Acceptor必须批准他接收到的第一个提案
      * P2: 如果编号为M0，Value值为V0的提案(M0,V0)被选定了，那么所有比编号M0更高且被选定的提案，其Value值也为VO
      * P2a: 如果编号为M0，Value值为V0的提案(M,V0)被选定了，那么所有比编号M0更高且被Acceptor批准的提案，其Value值必须是V0
      * P2b: 如果一个提案(M0,V0)被选定后，那么任何Proposer产生的编号更高的提案，其Value值都是V0
      * P2c: 对于任意的Mn和Vn，如果提案(Mn, Vn)被提出，那么肯定存在一个由半数以上的Acceptor组成的集合S，满足一下两个条件中的任意一个
         * S中不存在任何批准过编号小于Mn提案的Acceptor
         * 选取S中所有Acceptor批准过的编号小于Mn的提案，其中编号最大的那个其Value值是Vn
   * Proposer生成提案
      1. Proposer选择一个新的提案编号Mn，然后向某个Acceptor集合的成员发送请求，要求该集合中的Acceptor做出如下的响应
         * 向Proposer承诺，保证不再批准任何标号小于Mn的提案
         * 如果Acceptor已经批准过任何天，那么其就想Proposer反馈当前该Acceptor已经批准的编号小于Mn但为最大编号的那个提案的值
      1. 如果Poposer收到来自半数以上的Acceptor的响应结果，那么他就可以产生编号为Mn，值为Vn的提案，这里的Vn是所有响应中编号最大的提案的Value值。当然还存在另一中情况，就是半数以上的Acceptor都没有批准过任何提案，那么此时Vn值就可以有Proposer任意选择
   * Acceptor批准提案
      * Acceptor响应Proposer的请求
         * Prepare请求：Acceptor可以再任何时候响应一个Prepare请求
         * Acceptor请求：在不违背Acceptor现有承诺的前提下，可以响应Acceptor请求
         * 算法优化：如果一个Acceptor收到一个编号Mn的Prepare请求，但此时已经对编号大于Mn的Prepare请求做出了响应，则可以丢弃该请求
      * 算法陈述
         * 阶段一
            1. Proposer选择一个提案编号Mn，然后向Acceptor的某个超过半数的自己成员发送编号为Mn的Prepare请求
            1. 如果一个Acceptor受到一个编号为Mn的Prepare请求，且编号Mn大于把你Acceptor已经相应的所所有Prepare请求的编号，那么它就会将已经批准过的最大编号的提案作为相应反馈给Proposer，同时该Acceptor会承诺不会再批准任何编号小于Mn的提案
         * 阶段二
            1. 如果Proposer收到来自半数以上的Acceptor对于其发出的编号为Mn的Prepare请求的相应，那么它就会发送一个针对(Mn,Vn)提案的Accept请求给Acceptor。注意Vn的值就是接收到相应编号最大的天的值，如果响应中不好好任何提案，那么它就是任意值
            1. 如果Acceptor收到这个针对(Mn,Vn)提案的Acceptor请求，只要改Acceptor尚未对编号大于Mn的Prepare请求做出响应，它就可以通过这个提案
   * 提案的获取（Learner）
      * 方案1：Acceptor发送所有批准的提案给所有的Learner
      * 方案2：Acceptor发送所有批准的提案给主Learner
      * 方案3：Acceptor发送所有批准的提案给主Learner集合
   * 选取主Porposer保证算法的活性 
      * 防止死循环
      
*** Chapter 3: Paxos的工程实践

3.1 Chubby
   * 3.1.1 概述
      * 面向松耦合分布式系统的锁服务
      * 客户端接口类似于UNIX文件系统结构
   * 3.1.2 应用场景
      * 集中服务的Master选举
      * 元数据存储
   * 3.1.3 设计目标
      * 对上层应用程序的侵入性更小
      * 便于提供数据的发布与订阅
      * 开发人员对基于锁的接口更为熟悉
      * 更便捷的构建可靠的服务
         * Quorum机制：过半机制
      * 提供完整的、独立的分布式锁服务，而非仅仅是一个一致性协议的客户端库
      * 提供粗粒度锁服务
      * 在提供锁服务的同时提供对小文件的读写能力
      * 高可用、高可靠
      * 提供事件通知机制
   * 3.1.4 Chubby技术架构
      * 系统结构
         * 单个master，租期（Master lease），通过续租来延长租期
         * Master轮训DNS列表
      * 目录与文件
         * /ls/foo/wombat/pouch
         * 数据节点
            * 持久节点：需要显式调用接口API来删除
            * 临时节点：会在对应的客户端回话失效后被自动删除
            * 节点包括的4个单调递增的64位编号
               * 实例编号
               * 文件内容编号
               * 所编号
               * ACL编号
      * 锁与锁序列器
         * 接收消息乱序：虚拟时间和虚拟同步
         * chubby采用锁延迟和锁序列器两种策略来解决上面我们提到的由于消息延迟和重排序引起的分布式缩问题
            * 锁延迟：异常情况下，Chubby服务器会为该锁保留一定的时间， 在此期间其他客户端无法获取锁
            * 锁序列化：需要chubby的上层应用配合在代码中加入相应的修改逻辑，
      * Chubby中的事件通知机制
         * 文件内容变更
         * 节点删除
         * 子节点新增、删除
         * Master服务器转移
      * Chubby中的缓存
         * 客户端中实现了缓存
            * 通过租期机制来奥正缓存一致性
      * 会话和会话激活（KeepAlive）
         * KeepAlive  
            * master接受到客户端的KeepAlive请求
            * master阻塞该请求，并且等待该客户按的当前回去租期即将过其实，才为其续租该客户端的会话租期
               * 默认是12s，不过master会根据负载请求动态调整
            * 正在运行过程中，每个chubby客户端总是会有一个keepAlive请求阻塞在Master服务器上
            * master还利用keepalive响应来传递chubby时间通知和缓存过期通知给客户端
         * 会话超时
            * 客户端也维护着一个和master端相近的会话租期
               * 一方面，keepalive响应在网络传输过程中会话费一定的时间
               * 另一方面，Mater服务端和chubby客户端存在时钟不一致性现象
            * 客户端案遭本地的会话租期超时时间，监测到其回话租期已经过期却尚未接收到Master响应的KeepAlive响应，则它将无法确定Master服务器端是否已经终止了当前会话，我们称这个时候库换处于“危险状态”。此时chubby客户端会清空其本地缓存，并将其标记为不可用。通知，客户端还会等待一个被称作“宽限期（45s）”的时间周期。
                  * jeopardy -> safe/expired
      * Chubby Master的故障恢复
         * master故障恢复时间不记入会话的生命周期
         * 新的master选举产生后
            1 确定master周期
            1 选举产生新的Master能够立即对客户单的Master寻址请求进行响应，但是不会立即开始处理客户端会话相关的请求操作
            1 master根据本地数据库中存储的会话和锁信息，来构建服务器的内存状态
            1 此时，master已经能够处理客户端的keepalive请求了，但是依然无法处理其他会话相关的操作
            1 master 会发送一个“Master故障切换”事件给每一个会话，客户端收到之后会清空本地缓存，并警告上层应用程序可能已经丢失别的事件，之后在向master响应
            1 此时master会一直等待客户端的应答，知道每个会话都应答了这个切换事件
            1 在master接收到所有的客户端的应答之后，就能够开始处理所有的请求操作了
            1 如果客户端使用了一个在故障切换之前创建的句柄，master会重新为其创建这个句柄的内存对象，并执行调用
   * 3.1.5 Paxos协议实现
      * chubby服务端基本架构，大致分为三层
         1 最底层是容错日志系统（Fault-Tolerant Log），通过Paxos算法能够保证集群中所有机器上的日志完全一致，同时具备较好的容错性
         1 日志层之上是Key-Value类型的容错数据库（Fault-Tolerant DB），其通过下层的日志来保证一致性和容错性
         1 存储层智商就是chubby对外提供的分布式锁服务和小文件存储服务
      * Prepare->Promise->Propose->Accept

3.2 Hyperspace

      
** Chapter 4 Zookeeper与Paxos**

4.1 初识Zookeeper

4.1.1 Zookeeper介绍
   * 典型的分布式数据一致性解决方案：可以实现数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master选举、分布式锁和分布式队列等功能。
   * 保证分布式一致性
      * 顺序一致性
      * 原子性
      * 单一视图
      * 可靠性
      * 实时性
   * 设计目标
      1. 简单的数据模型
      1. 可以构建集群
      1. 顺序访问
      1. 高性能

4.1.2 Zookeeper从何而来
   * Yahoo

4.1.3 Zookeeper的基本概念
   * 集群角色
      * 没有采用典型的Master/Slave      
      * 引入了Leader、Follower和Observer
         * Observer不参与Leader选举过程，也不参与“过半写成功”策略
   * 会话（Session）
      * TCP连接，默认端口2181
      * SessionTimeout
   * 数据节点（ZNode）
      * 机器节点 + 数据节点
      * ZNode tree （/foo/path）
      * 持久节点 + 临时节点
         * 顺序节点：SEQUENTIAL
   * 版本
      * version：当前ZNode版本
      * cversion：当前ZNode子节点的版本
      * aversion：当前ZNode的ACL版本
   * Watcher
      * 事件监听器
   * ACL
      * CREATE
      * READ
      * WRITE
      * DELETE
      * ADMIN

4.1.4 为什么选择Zookeeper
   * 应用广泛、开源

4.2 Zookeeper与ZAB协议

4.2.1 ZAB协议
   * Zookeeper没有完全采用Paxos算法，而是使用了一种称为Zookeeper Atomic Broadcast（ZAB）的协议作为其数据一致性的核心算法
   * ZAB的核心是定义了对于那些会改变Zoookeeper服务数据状态的事务请求的处理方式
      * 所有事务请求必须由一个全局唯一的服务器来协调处理，这样的服务器被称为Leader服务器，而余下的其他服务器则称为Follower服务器。Leader服务器负责讲一个客户端事务请求转换成一个事务Proposal（提议），并将该Proposal分发给急群众的所有Follower服务器。之后Leader服务器需要等待所有Follower服务器的反馈，一旦草果半数的Follower服务器进行了正确的反馈后，那么Leader就会再次向所有的Follower服务器分发Commit消息，要求其将前一个Proposal进行提交。

4.2.2 协议介绍
   * 两种基本模式：消息广播和崩溃恢复
   * 消息广播
      * 基于TCP协议
      * 二阶段提交
      * 每个事务有ZXID
      * Leader服务器为每个Follower分配一个队列，并根据FIFO进行发送。
      * Follower服务器在接收到这个事务proposal之后，会首先将其以事务日志的形式写入到本地磁盘，并且在成功写入后反馈给Leader服务器ACK响应
      * Leader服务器若收到超过半数的ACK（包括自己的）之后，leader会广播一个Commit消息给所有的Follower服务器以通知其进行事务提交。Leader也会完成事务提交
      * 而每一个Follower服务器在接收到Commit之后也会完成对事务的提交。
   * 崩溃恢复
      * Leader服务器崩溃，需要高效可靠的选举算法
      * Leader需要快速让Leader知道自己是Leader，同时也需要让其他机器感知新的Leader
      * 基本特性
         * ZAB协议需要确保哪些已经在Leader服务器上提交的事务最终被所有服务器都提交
         * ZAB协议需要确保丢弃哪些只在Leader服务器上被提出的事务 
         * 拥有最大事务编号的机器被选为Leader
            * 保证有所有的已经提交的提案
               * 可以省去Leader服务器检查Proposal的提交和丢弃工作的这一步操作
      * 数据同步
         * 完成Leader选举之后，正式工作之前会确认事务日志中所有的Proposal是否已经被过半的Follower提交
         * Leader服务器会为每一个Follower准备一个队列，并将那些没有被各个Follower服务器同步的事务以Proposal的形式逐步发送给各个Follower，并且在每一个Proposal消息之后再发送一个Commit消息，表示该事务已经被提交。等到Follower服务器将所有其尚未同步的事务Proposal都从Leader服务器上同步过来并成功应用到本地数据库中后，Leader服务器就会将该Follower加入到真正可用的Foloower列表中，并开始其后的流程。
      * 如何处理需要被丢弃的事务
         * 依赖于ZXID的设计
            * ZXID 64位：低32位是一个简单的单调递增的计数器，而高32位代表leader周期的epoch编号，每当一个选举产生一个新的leader，会取其最大的ZXID的epoch然后加1作为新的epoch号，并将低32位清零
            * 当拥有就的epoch的机器启动时，不能成为Leader，故而成为Follower
            * 同时leader服务器会根据自己最后被提交的Proposal来和Follower服务器的Proposal进行对比，对比的结果当然是Leader会要求Follower进行一个回退操作，会退到一个确实已经被急群众过半机器提交的最新的事务Proposal。

4.2.3 深入ZAB协议
   * 系统模型
      * 完整性（Integrity）
      * 前置性（Prefix）
   * 问题描述
      * 主进程周期： epoch， readny（） 
      * 事务：transactions(v,z), Z = <epoch, count>
   * 算法描述
      * 包括消息广播和崩溃恢复两个阶段，进一步可以分解为：发现（Discovery）、同步（Synchronization）和广播（Broadcast）阶段
      * 阶段1 发现： 主要就是Leader选举过程，用于多个分布式进程中选出主进程，准Leader的Follower的工作流程分别如下
         * F.1.1 Follower F将自己最后接受的事务Proposal的epoch值CEPOCH（处理过的最后一个事务的）发送给准Leader
         * L.1.1 当接收到来自过半Follower的CEPOCH消息后，准Leader L会生成NEWEPOCH消息给这些过半的Follower： e_new = 最大Epoch + 1
         * F.1.2 当Follower接收到来自Leader L的NEWEPOCH消息后，如果其检测到当前的CEPOCH值小于e_new，就将最后处理的事务epoch设置为e_new，同时向这个准Leader反馈ACK消息。 在这个反馈消息ACK-E(F, h))中包含了而当前这个Follower的epoch 以及该该Follower的历史事务Proposal集合：Hf
         * Leader选择一个Follower， 使其作为初始化集合I
      * 阶段2 同步：
         * L.2.1 Leader L会将e_new和I以 NEW_LEADER(e_new, I)消息的形式发送给所有Quroum中的Follower
         * F.2.1 当Follower接受到来自Leader L的NEW_LEADER消息后，Follower发现CEPOCH（e） 和 e_new 不同，则直接进入下一轮循环；如果相同，那么Follower就会执行事务应用操作。Follow都会接受，最后Follower会反馈给Leader，表明自己已经接受并处理所有I中的Proposal
         * L.2.2 当Leader接收到来自国安Follower针对NEWLEADER的反馈消息后，就会想所有的Follower发送Commit消息，至此，LEader完成阶段2
         * F.2.2 当Follower收到来自Leader的Commit消息后，就会依次处理并提交所有在I中未处理的事务。至此，Follower完成阶段2
      * 阶段3 广播
         * L.3.1 Leader L接收到来自客户端新的事务请求后，会生成对应的事务Proposal，并根据ZXID的顺序想所有Follower发送提案<e, <v,z>>
         * F.3.1 Follower 根骨消息接收的先后次序来处理这些来自Leader的事务Proposal，并将它们追加到h中去，之后在反馈给Leader
         * L.3.2 当Leader接收到来自过半Follower针对事务Proposal的ACK消息后，就会发送Commit消息给所有的Follower，要求它们进行事务的提交
         * F.3.2 当Follower F接收到来自Leader的Commit消息之后，就会开始提价哦事务Proposal。需要注意的是赐个FollowerF必定已经提交了先前的事务
   * 运行时分析
      * 每一个进程的可能的三种状态
         * LOOKING ： leader选举阶段
         * FOLLOWING： Follower服务器和Leader保持同步状态
         * LEADING：Lader服务器作为主进程领导状态
      * LEADER定义如下：
         * 如果一个准Leader 接受到来自过半的Follower进程针对L的NEWLEADER反馈消息，那么L就成为了周期e的Leader

4.2.4 ZAB 与PAXOS算法的联系与区别
   * 联系
      1. 两者都纯在一个类似于Leader进程的角色，由其负责协调多个Follower进程的运行
      1. Leader进程都会等待超过半数的Follower做出正确的反馈后，才会将一个提案进行提交
      1. 在ZAB协议中，每个Proposal中都包含了一个epoch值，用来代表当前Leader周期，在Paxos中，同样存在一个一个标识，叫Ballot
   * 不同
      1. paxos：读 + 写
      1. ZAB增加了同步阶段
   * 本质区别
      1. ZAB：够条件高可用主备系统
      1. PAXOS：构建一个分布式的一致性状态及系统      