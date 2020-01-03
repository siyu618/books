# Flink

### [第一章-Flink介绍-《Fink原理、实战与性能优化》读书笔记](https://www.cnblogs.com/fengtingxin/p/11128213.html)
* Flink 的几大优点
   * 1. 同时支持高吞吐、低延迟、高性能
   * 2. 支持事件时间（event time）概念
   * 3. 支持有状态计算
   * 4. 支持高度灵活的窗口（windows）操作
   * 5. 基于轻量级分布式快照（snapshot）实现的容错
   * 6. 基于 JVM 实现独立的内存管理
   * 7. Save Points
* Flink 的基本架构
   * 组件栈
      * 1. API & Libraries 层
      * 2. Runtime 核心层
      * 3. 物理部署层
   * 基本架构图
      * 1. Flink Client：
      * 2. JobManager：调度和资源管理
      * 3. TaskManager

### [Flink 基本工作原理](https://blog.csdn.net/sxiaobei/article/details/80861070)
* JobClient
      * 负责接收程序、解析程序计划、优化程序执行计划，之后提交到 JobManager
      * Operator
         * 1. Source Operator：数据源
         * 2. Transformation Operator：负责转换数据，map、flatMap、reduce
         * 3. Sink Operator：数据落地、存储
* JobManager：
   * 进程，负责申请资源，协调以及控制整个 job 的执行
   * 为了保证高可用，JobManager 一般采用主从模式
   * 采用 Check Point 进行容错：check point 其实是 stream 以及 executor（TaskManager 中的 slot）的快照
* TaskManager
   * 进程，接收并执行 JobManager 发送的 task
   * 多个 task 运行于一个 JVM 中：可以共享一个 TCP 链接，共享节点间的心跳信息，减少网络传输
   * Solt：每个 slot 有自己独立的内存   
      * 同一个 Job 中，同一个 group 中不同 operator 的 task 可以共享一个 slot
      * Flink 是按照拓扑顺序从 source 依次调度到 Sink 的

### [Flink架构及其工作原理](https://www.cnblogs.com/code2one/p/10123112.html)
* Components of a Flink Setup
   * JobManager：接受 application，包含 StreamGraph（DAG），JobGraph（Logoical dataflow graph，已经优化过的，如 task chain）和 Jar，将 JobGraph 转化为 ExecutionGraph（physical dataflow graph， 并行化），包含可以并发执行的 tasks。其他工作类似 Spark driver，如向 RM 申请资源、schedule tasks、保存作业的元数据，如 checkpoints。
      * 如今的 JobManager 可以分为 JobMaster 和 ResourceManager（和下面的不同），分别负责任务和资源， 在Session 模式下就会有多个 JobMaster。
   * Resource Manager：一般是 Yarn，当 TM 有空闲的 slot 就会告诉 JM，没有足够的 slot 就会启动新的 TM，kill 掉长时间空间的 TM。
   * TaskManager：类似于 Spark 的 executor，会跑多个线程的 task，数据缓存于交换。
   * Dispatcher（Application master）：提供 REST接口来接收 client 的 application 提交，它负责启动 JM 和提交 application，同时运行 Web UI。
* 工作模式
   * Session 模式：预先启动好 AM 和 TM，每提交一个 JOB 就启动一个 JOB Manager 向 FLINK 的 RM 申请资源。适合国模小，运行时间短点的作业。
   * Job 模式：每一个 job 都重新启动一个 Flink 集群，完成后结束 Flink，且只有一个 Job Manager。资源需要按需申请，适合大作业。
* Application Deployment
   * Framewrok 模式：Flink 作业为 JAR，被提交到 Dispatcher or JM or YARN
   * Library 模式：Flink 作业为 application-specific container image，如 DockerImage，适合微服务。
* Task Excution
   * 作业调度：在流计算中预先启动好节点；在批计算中，每当某个阶段完成才启动下一个节点。
   * 资源管理：slot 作为基本单位，有大小位置。JM 有 SlotPool，向 Flink RM 申请 slot，Flink RM 发现自己的 SlotManager 中已经没有足够的 Slot，就会向集群 RM 申请，后者返回可用的 TM 的 IP，让 Flink RM 去启动，TM 启动后向 Flink RM 注册，后者向 TM 请求 slot，JM 向 TM 请求提供相应的 SLOT。JM 用完之后会释放相应的 slot，TM 会把释放的 slot 告诉 FlinkRM。
   * 任务可以是相同 operator（data parallelism），不同 operator（task parallelism），甚至不同的 application（job parallelism）。TM 提供一定数量的 slot 来控制并行的任务数。
* Highly-Available Setup
   * 当一个 TM 挂掉而 RM 又无法找到空闲的资源时，就只能暂时降并行度，直到有空闲的资源重启 TM。
   * 当 JM挂掉就靠 ZK 来重新选举，和找到 JM 存储到远程 storage 的元数据、job Graph。重启 JM 并从最后一个完成的 checkpoint 开始。
      * JM 在执行期间会得到每个 task checkpoints 的 state 存储路径并写到远程 storage，同时在 ZK 存储路径上留下 checkpoint 指到哪里找大上面的存储路径。
* 背压
   * 数据涌入的速度大于处理速度。在 source operator 中，可以通过 Kafka 解决。在任务见的 operator 可以有如下解决方法：
      * local exchange：taks1 和 task2 在同一个工作节点，那么 buffer pool 可以直接交给下一个任务，当 task2 消费 buffer pool 中的信息减慢时，当前任务task1 填充 buffer pool 的速度也会减慢。
      * remote exchange：TM 保证每个 task 至少有一个 incoming 和 outgoing 缓冲区。当下游的 receiver 的处理速度低于上游的 sender 的发送速率，receiver 的 incoming 队列就会开始累计数据（需要空闲的 buffer 来放 从 TCP 链接中接收到的数据），当记满后就不再接收数据。上游的 sender 利用 netty 水位机制，当网络中的缓冲数据过多的时候停止发送。
* Data Transfer in Flink
   * TM 负责数据在 tasks 之间的转移，转移之前会存储到 buffer。每一个 TM 有 32Kb 的网络 buffer 用于接收和发送数据。
      * 如果 sender 和 receiver 在不同的进程，那么会通过操作系统的网络栈来通信。没对 TM 保持 permanent TCP 链接来交换数据，每个 sender 能够给所有 receiver 发送数据，反之所有的 receiver 任务能够接收所有 sender 任务的数据。TM保证每个任务都之地少有一个 incoming 和 outgoing 的 buffer，并增加额外的缓冲区来避免死锁。
      * 如果sender 和 receiver 任务在同一个 TM 进程， sender 会序列化结果数据到 buffer，如果满了就放到队列。receiver 任务通过队列得到数据并进行反序列化。这样得到的好处是解耦任务并允许在任务重使用可变对象，从而减少了对象实例化和垃圾收集。一旦数据被序列化，就能安全的修改。而缺点是计算消耗大，在一些条件下能够把 task 穿起来，避免序列化。
   * Flow control with bach pressure
      * receiver 接受满了，sender 要减慢发送。

* Event Time Processing
   * watermark: records holding a timestamp long value。递增的。
   * watermarks and event time：当收到 WM 大于所有目前拥有的 WM，就会把 event-time clock 更新为所有 WM 中最小的 WM。
   * Timestamp Assignment and Watermark Generation
      * Source Function
      * AssignerWithPeriodicWatermarks
      * AssignerWithPunctuatedWatermarks

* State Management
   * state 和特定的 operator 关联，operator 需要注册它的 state，state 有两种类型
      * 1. operator state：由同意并行任务处理的所有记录都可以访问相同的 state，而其他的 task 或 operatoe 不能访问，即一个 task 一个专属的 state。这种 state 有三种 primitives
         * Liststate：state as a list of entries
         * union list state: 同上，但是在任务失败和作业从 savepoint 重启的行为不一样
         * broadcast state：同一样一个 task 专属一个state，但 state 都是一样的。只有所有的 task 的更新一样时候，即输入数据一样（一开始广播所以一样，但数据的顺序可能不一样），对数据的处理一样，才能保证 state 一样。这种 state 只能存在内存，所以没有 rocksDB  backend。
      * 2. keyed state：相同 key 的 record 共享一个 state   
         * Value state：每个 key 一个值，这个值可以是复杂的数据结构
         * List state：每个 key 一个 list
         * Map state：每个 key 一个 map   
   * State backends：决定了 state 如何被存储、访问和维持。它的主要职责是本地 state 的管理和 checkpoint state到远程。在管理方面，可选择将 state 存储到内存还是磁盘。
      * MemeoryStateBachend
      * FsStateBachend
      * RocksDBStateBackend
      * 三个都支持异步 checkpoint，其中 RocksDBState 还支持 incremental 的 checkpoint。
   * Scaling Stateful Operators
      * Flink 会根据 input rate 调整并发度。对于 stateful operator 有以下四种方式：
         * keyed state：根据 key group 来调整。
         * list state：所有 list entries 会被收集并重新均匀分布。
         * union list state：增加并发时，广播真个 list，所以 rescaling后，所有task都有所有的 list update。
         * broadcast state
* Checkpoints， Savepoints and State Recovery
   * Flink's Ligheweight CheckPointing Algorithm
      * 基于分布式快照算法 Chandy-Lamport 实现。 
      * 特殊的 record 叫 Checkpoint Barrier（由 JM 产生），它带有 checkpoint id 把流进行划分。在 CB 前的 records 会被包含到 checkpoint，之后的会被包含到之后的 checkpoint。
      * 当 source task 收到这种消息，就会停止发送 records，出发 state backed 对本地的state 进行 checkpoint，并广播 checkpoint id 到下游task。当 checkpoint 完成时，state backend 唤醒 source task，后者向 JM 确定相应的 checkpoint id 已经完成任务。
      * 当下游一获得其中一个 CB 时，就会暂停处理对应 source 的数据（完成 checkpoint 之后的数据），并将这些数据存到缓冲区，直到其他相同 id 的 CB 都到齐，就会把 state 进行 checkpoint， 并广播 CB 到下游。直到所有 CB 别广播到下游，才开始处理排队在缓冲区的数据。当然其他没有发送 CB 的 source 仍会继续处理。
      * 最后，当所有 sink 回想 JM 发送 BC 确定 checkpoint 完成。
      * 优化点：
         * 当 operator 的state很大时候，复制真个 state 并发送到远程 storage 会很费时。而 rocksDB state backend 支持 asynchronous and incremental 的 checkpoints。当触发 checkpoint 时候，backedn 会快照所有本地的是state 修改（直到上一次的 checkpoint），后台线程异步发送到远程 storage。
         * 在等其余 DB 时，已经完成 checkpoint 的 source 数据需要排队。但如果使用 at-least-once 。当时当所有的 CB 到齐再 checkpoint，存储的 state 就已经包含了下一次 checkpoing 才记录的数据。
   * Recovery from Consistent Checkpoints
   * Savepoints
      * Checkpoint  + 元数据


### [Flink 轻量级异步快照 ABS 实现原理](https://blog.csdn.net/zhangjun5965/article/details/87712957)               
* 异步屏障快照（ABS，Asynchronous Barrier Snapshot）
   * 能够以低成本备份 DAG（有向无环图） 或 DCG（有向有环图）计算作业的状态。
   * 核心思想：通过屏障（barrier）消息来标记快照的时间点和对应的数据，从而将数据流和快照时间解耦以实现异步快照操作，
* 计算机作业模型
   * G = （T， E）   ； T 是作业的子任务（Flink 中叫做 operator，算子）即图的节点；E 作为 图中的边的集合。
   * 算子状态最为关键，通道内的数据作为补充；同时通道内地数据量要大得多。
   * 将作业的执行划分为多个阶段（stage），每个阶段对应一个数据流窗口以及所有相关计算结果。
* 屏障（Barrier）
   * 内部一种特殊的内部消息，用于将数据流从时间上切分为多个窗口，每个窗口对应于一系列连续快照中的一个。
   * 屏障由 JobManager 定时广播给计算任务所有的 source，其后伴随数据流一起流至下游。
   * 每个 barrier 是属于当前快照的数据与属于下一个快照的分割点。
   * 校准：
      * 1. 其中一个上游数据流接受到快照屏障 barrier n 之后，算子会暂停对该数据流的处理，直到另外的数据流也接收到 barrier n 它才会重新开始处理对应数据流。这种机制避免了两个数据流的效率不一致，导致算子的快照窗口不属于当前窗口的数据。
      * 2. 算子暂停对一个数据流的处理并不会阻塞该数据流的接受，而是将其暂时缓存起来，因为这些数据属于快照 n+1，而不是快照 n。
      * 3. 当另一个数据流的 barrier n 也抵达，算子会将所有正在处理的结果发至下游，随后做快照并发送 barrier n 到下游。
      * 4. 最后，算子重新开始对所有数据流的处理，当然是优先处理已经被缓存的数据。
      
* 送达语义
   * 木桶原理。
   * 校准会带来数毫秒的延时。

### [Stream Processing：Apache Flink快照(snapshot)原理](https://blog.csdn.net/liangyihuai/article/details/86019192)
*  快照
   * 对系统当前状态的存储
*  跟其他的快照相比，Apache Flink 的优势何在？ 
   * Spark ： 通过上游节点重新计算来恢复宕机前的数据
      * 优点：不需要存储算子当前的状态，节约空间。缺点：增加计算量和计算时间。
   * Hadoop：采用数据备份到不同的机架机器硬盘来实现 
      * 缺点：速度方面有点慢。
   * Flink ： 异步轻量级
* Flink 的快照原理
   * 系统快照
      * 1. 首先在所有的数据源注入屏障数据，然后向所有的输出节点广播
      * 2. 如果一个非数据源的节点收到屏障，就阻塞该屏障所在的数据源，也就暂时不接受这一输入流的数据，直到接收到其他输入流的所有屏障。所以，如果一个该节点只有一个输入流，不用阻塞等待。 
      * 3. 当这个节点收到了跟其他链接的输入流的所有屏障，便开始生成当前节点的一个快照。   
   * DAG VS. DCG
      * 对于 DCG，在接收到回向的屏障后，也生成当前的快照。
