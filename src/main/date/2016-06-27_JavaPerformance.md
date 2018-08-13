java performance

Chapter 1 : 策略、方法和方法论
0. 三个境界
   * 花似雾中看（I don’t know what I don’t know）
   * 悠然见南山（I know what I don’t know）
   * 一览纵山小（I already know what I need to know）
1. 性能问题现状
   * 软件开发四个阶段：分析、设计、编码和测试
   * 应用预期的吞吐是多少
   * 请求响应之间的延时预期是多少
   * 应用支持多少并发任务或并发用户
   * 当应用并发用户或是并发任务数达到最大时，可接受的吞吐量和延时是多少
   * 最差情况下延时是多少
   * 要使垃圾收集引入的延迟在可容忍范围之内，垃圾收集的频率应该是多少
2. 性能分析两种方法：自顶向下和自底向上
   * 自顶向下：监控操作系统、JVM、Java Container以及应用的新能测量统计指标
   * 自底向上：在不同平台（指底层的CPU架构或CPU数量不同）上进行性能调优
      * 指令路径长度、cache命中率


Chapter 2: 操作系统系能监控
1. 定义
   * 性能监控：非侵入式收集或查看查看应用运行性能数据的活动
   * 性能分析：侵入式，会影响吞吐量或响应性
   * 性能调优： 
2. cpu使用率
   * 分为：用户态cpu使用率和系统太cpu使用率（执行操作系统调用），尽可能降低系统态cpu使用率
       * 对于计算密集型应用：监控时钟指令数（Instruncitons Per Clock， IPC）或每指令时钟周期（Cycles Per Instruction, CPI）。存在浪费时候周期的问题，比如等待数据。 
   * 监控CPU使用频率windows ：  Task Manager，Windows typeperf
   * 监控CPS使用频率linux：GNOME System Monitor，vmstat
   * CPU 调度程序运行队列： Java API Runtime.availableProcessors() 返回虚拟处理器的个数，即系统硬件线程的个数
      * vmstat
3. 内存使用率
      * vmstat监控swap的si,so
  4.监控锁竞争：pidstat -w
  5.网络I/O使用率
  6.磁盘使用率
     * iostat

Chapter 3： JVM 概览
1. HotSpot VM 的基本架构：VM 运行时、JIT编译器（Client/Server）和垃圾收集器（Serial，throughput，CMS和G1）
   * 32位JVM，内存地址限制4G，同时受限于底层的os
   * 64位JVM，内存更大，同时也带来了新能损失，JVM内部对象表示（普通对象指针，Ordinary Object Pointers，或oops）导致CPU高速缓存行（CPU Cache Line）中可用的oops变少。 较之于32位的JVM缓存效率降低8%~15%。压缩指针（Compressed oops， -XX:+UseCompressedOops）。 同时存在寄存器卸载的情况。 
2. HotSpot VM运行时
   * 命令行选项：标准选项（Stand Option）、非标准选项（Nonstandard Option， -X）和非稳定选项（Developer Option， -XX）。
   * VM生存周期：JNI_CreateJavaVM ， DestroyJavaVM
   * VM类加载：
       * 类加载阶段：查找二进制文件，检查语法，
       * 类加载器委派：当求求类击打在其查到和加载某个类时，该类加载器可以转而请求别的类加载器来加载。类加载器之间是层级化关系，每个类加载器都可以委派给上一级类加载器。查找顺序为：启动类加载器、扩展类加载器和系统类加载器。
       * 启动类加载器：负责加载BOOTCLassPATH路径中的类。
       * 类型安全：不同的类加载器加载的同名的类是不同的类
       * HotSpot类元数据：instantKlass，klassOop
       * 内部的类加载数据：加载过程中VM维护了三张散列表。SystemDictionary 包含已经加载的类，类名/类加载器（包括初始类加载器和定义类加载器）于klassOop对象之间的映射。 PlaceholdTable包含了当前正在加载的类，他用于检查ClassCircularityError。LoaderConstraintTable用于追踪类型安全检查的约束条件。这些散列都需要检索以确保访问安全。     
   * 字节码验证 ： 类型推到和类型检查
   * 类数据共享：多个JVM进程间共享：系统jar中的部分类。主要目的是减少启动时间。
   * 解释器：基于TemplateTable中的信息在内存中生成解释器。 -XX:+UnlockDiagnosticVMOptions和-XX:+PrintIntepreter。监控程序中的重要热点（Hot Spot）。
   * 异常处理：由VM解释器、JIT编译器和其他VM组件一起协作实现。 
   * 同步：synchronized ， 使用monitor对象来保证线程运行代码之间的互斥。-XX:+USeBiasedLocking允许线程自身使用偏向锁。
   * 线程管理
      * a.线程模型：Java线程别一对一映射为本地操作系统线程。
      * b.线程创建和销毁：java.lang.Thread.start() 或者JNI将已存在的本地线程关联到HotSpotVM上。   C++ JavaThread =》OSThread=》本地线程
      * c.线程状态：新线程、线程在Java中、线程在VM中、线程阻塞
      * d.VM内部线程：VM线程、周期任务线程、垃圾收集线程、JIT编译器线程、信号分发线程
      * e.VM操作和安全点：VMThread监控VMOperationQueue的C++对象
   * C++堆管理：Arena及其子类是C/C++内存管理函数malloc/free之上的一层。会创建3个全局ChunkPool。
   * java本地接口
   * VM致命错误处理：hs_err_pid<pid>.log -XX:ErrorFile. -XX:OnError=“cmd1 args..;cmd2...
   * HotSpot VM垃圾收集器
   * 1.分带垃圾收集器
       * 弱分代假设1：大多数分配的对象存活时间很短
       * 弱分带假设2：存活时间久的对象很少引用存活时间端的对象
       * 新生代（Young Gen）：大多数新创建的对象呗分配到新生代中。MinorGC
       * 老年代（Old Gen）：新生代中长期存活的对象最后会被提升（Promote）或晋升（Tenure）到老年代。Full GC，收集时间长。 
       * 永久代（Metaspace）
       * 垃圾收集器不需要扫描整个老年代就能识别新生代中的存活对象，从而缩短MinorGC的时间。老年代以512字节为块划分成若干张卡（Card）。卡表是单字节数组，每个数组元素对应对中的一张卡，每次老年代对象中某个引用新生代的字段发生变化时，Vm须将改卡对应的卡表元素设置为相应的值，从而将该引用字段所在的卡表标记为脏。MinorGC过程中，垃圾收集器只会在张卡中扫描查找老年代到新生代的应用。 解释器和JIT编译器使用写屏障（Write Barrier）维护卡表。 
         * 新生代（Young Generation）
            * Eden ： 大多数新对象分配于此（有部分大的对象直接分配在老年代）
            * Survivor（一对）：存放至少经历一次MinorGC。
            * Minor GC过程中，Surivivor可能不足以容纳Eden和另一个Survivor中的存活对象，如果一处，多余的对象会被移到老年代（称为过早提升 Premature Promotion），可能导致提升失败（Promotion Failure）。 
            * 快速内存分配：线程本地分配缓冲区（Thread-Local allocation Buffer， TLAB），为每个线程设置各自的缓冲区，一次改善多线程分配的吞吐量。
         * 垃圾收集器
            * Serial收集器：老年代采用华东压缩标记-清除（Sliding Compacting Mark-Sweep），都会stop the world
            * Parallel收集器：吞吐量优先，与serial类似，也会stop the world
            * Mostly-Concurrent收集器：低延迟优先， CMS（Concurrent Mark-Sweep GC，CMS收集器）：a初始标记：标记外部（GC Roots）可直接到达的对象；b并发标记（预清除）：标记从这些对象可大的存活对象；c重新标记：重新遍历所有在并发标记期间有变动的对象并进行最后的标记；d并发清除。 缺点：需要更大的堆，同时存在浮动垃圾，空间碎片化（Fragmentation）。
            * Garbage-First收集器：CMS替代者
            * 应用程序对GC的影响
               * 内存分配的速率
               * 存活数据的多少
               * 老年组中的引用更新
   * HotSpot VM JIT编译器
      * 编译：是指从高级语言生成机器码的过程。 源代码==>类文件==>jar==>中间代码（Intermediate Representation，IR）==>机器码
        * 优化：简单恒等变化、常量折叠、公共表达式消除以及函数内联。还有循环的执行上。
           * 完成指令选择之后，就必须将寄存器指派给程序中的所有变量。一般情况下，存活变量的数目会操作寄存器的个数，需要通过寄存器和栈来回移动变量。将值移动到栈中成为值卸载或者寄存器卸载。本地分配器用轮询调度算法。
        * 类型继承关系分析：内联，对于覆盖的方法使用类型继承关系分析。 逆优化。
        * 编译策略：起初都是解释执行，调用次数多了会成为编译。每个方法都有：方法调用计数器和回边计数器。 超过阈值是粗发变迁移，其会进入别一个或多个编译器线程见识的队列。 通常解释器会重置计数器然后继续在解释器中执行该方法。
            * 逆优化：将那些经过入肝级内联而来的编译帧转换为等价的解释器帧的过程。 
      * client JIT编译器概览：
      * Server JIT编译器概览
      * 静态单赋值—程序依赖图：获取每次操作执行过程中的最小约束，是的可以对操作进行基金重拍和全局技术，依次减少计算。
      * 未来增强展望
   * HotSpot VM自适应调优
       * 自适应java堆调整
       * 超越自动优化

             










