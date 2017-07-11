**Chapter 1 并发编程的挑战**

*1.1 Context Switch(上下文切换)*
   * 无锁并发
      * 多线程竞争锁时，会引起上下文切换，可以用一些办法来避免使用锁，入江数据的ID按照Hash算法取模分段，不同的线程处理不同的段。
   * CAS算法
      * compare and swap
   * 使用最少线程
      * 避免创建不需要的线程，比如任务很少，但是创建了很多线程来处理，这样会照成大量线程都处于等待状态。
   * 协程
      * 在单线程里实现多任务的调度，并且的单线程里维护多个任务之间的切换

*1.2 避免死锁的常见方法*
   * 避免一个线程同时获取多个锁
   * 避免一个线程在锁内同时占用多个资源
   * 尝试使用定时锁，lock.tryLock
   * 对于数据库锁，枷锁和解锁必须在一个数据库连接里面，否则会出现解锁失败的情况

*1.3* 资源限制的挑战
   * 资源限制：比如服务器带宽只有2M/s,某个资源下载速度是1M/s，则多线程的上线也是1M/s
   * 资源限制引发的问题
      * 并行为真的并行
   * 如何解决资源限制的问题
      * 硬件限制：集群并行
      * 软件限制：资源池
   * 在资源限制情况下进行并发编程
      * 找到瓶颈


**Chapter 2 Java并发机制的底层实现原理**

*2.1* volatile 的应用
   * cpu术语
      * 内存屏障（memory barrier）：一组处理器指令，用户实现对内存操作的顺序限制
      * 缓冲行（cache line）：cup高速缓存中可以分配的最小存储单位。cpu填写缓存行时会加载整个缓存行，现在CP需要执行几百次CPU指令
      * 原子操作（atomic operations）：不可终端的一个或一系列操作
      * 缓存行填充（cache line fill）：当处理器识别到从内存中读取操作数是可缓存的，处理器读取整个缓存行到适当的缓存（L1,L2,L3或所有）
      * 缓存命中（cache hit）：如果进行高速缓存行填充操作的内存为止仍是下次处理器访问的地址是，处理器从缓存中读取操作数，而不是从内存中读取
      * 写命中（write hit）：当处理器将操作数写回到一个内存缓存的区域时，它会首先检查这个缓存的内存地址是否在缓存行中，如果存在一个有效的缓存行，则处理器将这个操作数写回到到缓存，而不是写回到内存，这个操作被称为写命中
      * 写缺失（write misses the cache）：一个有效的缓存行被写入到不存在的内存区域
   * Lock前缀指令
      * 将当前处理缓存行的数据写回到系统内存
      * 这个协会内存的操作会使在其他CPU里缓存了该内存地址的数据无效
   * 缓存一致性协议
      * MESI（modified，Exclusive，Share，Invalid）http://blog.csdn.net/muxiqingyang/article/details/6615199
   * volatile的两条实现原则
      * 1）Lock 前缀指令会引起处理器缓存协会到内存
      * 2）一个处理器的缓存回写到内存会导致其他处理器的缓存无效
   * volatile的使用优化
      * LinkedTransferQueue
         * 追加字节优化性能
            * L1,L2,L3 高速缓存是64字节宽，不支持部分填充缓存行
            * 避免head和tail放在一个缓存行里面
         * 不适用的场景
            * 缓存行非64字节的
            * 共享变量不会被频繁地写

*2.2* synchronized 的实现原理与应用
   * 重量级锁
      * jdk1.6为了优化性能引入了偏向锁和轻量级锁，以及所的存储结构和升级过程
   * 实现同步的基础：java中每个对象都可以作为锁
      * 对于普通的同步方法，锁是当前实例对象
      * 对于静态同步方法，锁是当前类的class对象
      * 对于同步方法快，所示synchronized括号里面的对象
   * jvm基于进入和退出Monitor对象来实现方法的同步和代码块的同步
      * 方法块同步是通过monitorenter和monitorexit指令实现的
      * synchronized用的锁是存在java对象头里的
   * java对象头
      * 如果对象是数组类型，VM使用3个字宽（word）来存储对象头，否则用两个
         * mark word：32/64 bit，存储对象的hashCode或锁信息
            * 其中存有gc年龄
         * class metadata address：32/64 bit，存储对象类型数据的指针
         * array length：32/32 bit，数组的长度，如果对象是数组
   * 锁的升级与对比
      * 锁类别
         * 无锁状态，偏向锁状态，轻量级锁状态，重量级锁状态
            * 锁可以升级却不能降级，是处于提高获获取锁和释放锁的效率
      * 偏向锁
         * 当线程访问同步块并获取锁，会在队形头和战阵中的所记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块是不需要进行CAS操作来加锁和解锁，只需要简单测试一下对象头的Mark Word中是否存储着会想当前线程的偏向锁。
         * 偏向锁，使用一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。
         * 关闭偏向锁
            * jdk1.6 1.7中默认开启
               * -XX:BiasedLockingStartupDelay=0:关闭延时
               * -XX:UseBiasedLocking=false，关闭偏向锁，默认进入轻量级锁状态
         * 优点
            * 加解锁不需要额外的消耗，与非同步方法比，仅存在ns的差距
         * 缺点
            * 如果存在竞争，会带来额外的撤销的消耗
         * 适用场景
            * 只有一个线程访问的同步块场景
      * 轻量级锁
         * 加锁：执行同步块之前，VM在当前线程的战阵中创建用于存储所记录的空间，并将对象头中的Mark Word复制到所记录中（Displaced Mark Word）
             * 没抢到，就自旋
         * 解锁：CAS将Displaced Mark Word替换回到对象头
            * 成功则表示没有竞争的发生
            * 失败则表示存在竞争，则锁会膨胀成重量级锁，并自己阻塞。
         * 优点
            * 竞争的线程不会阻塞，提高了响应速度
         * 缺点
            * 如果始终得不到所竞争的线程，自旋会消耗CPU
         * 适用场景
            * 最求相应时间，同步块执行的速度非常快
      * 重量级锁
         * 优点
            * 线程竞争不用自旋，不会消耗CPU
         * 缺点
            * 线程阻塞，响应时间缓慢
         * 适用场景
            * 最求吞吐量，同步块执行时间较长

*2.3* 原子操作的实现原理
   * 术语
      * CAS
      * cache line
      * CPU pipeline：X86指令分解为5~6个步骤，使用CPU中5~6个不同功能的电路单元主城的指令处理流水线来完成。
      * Memory order violation：假共享（多个CPU指令通过时修改同一个缓存行的不同部分）引起的CPU清空流水线
   * 处理器如何实现原子操作
      * 1）通过总线加锁保证原子性
         * lock#信号
         * 导致其他CPU不能访问内存
      * 2）通过缓存锁来保证原子性
         * 缓存一致性
         * 处理器不适用缓存锁定的场景
            * 当操作的数据不能缓存在处理器内部、或操作的数据跨多个缓存行，这个是会使用总线锁
            * 有些处理器不支持缓存锁定
   * java如何实现原子操作
      * java中通过锁和循环CAS来实现原子操作
      * CAS
         * 利用处理器的CMPXCHG指令
         * 问题
            * ABA：添加版本号
            * 循环时间长开销大：pause指令
            * 只能保证一个共享变量的原子操作
      * 使用锁机制实现原子操作
         * 除了偏向锁，很多锁都用了循环CAS

**Chapter 3 Java内存模型**

*3.1* Java内存模型的基础

3.1.1 并发编程模型的两个基本问题
   * 线程之间如何进行通信
      * 命令式编程中：共享内存和消息传递
   * 线程之间如何进行同步
      * 是指程序中用户控制不同线程之间操作发生相对顺序的机制

3.1.2 Java内存模型的抽象结构
   * 所有实域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享
   * 局部变量、方法定义参数和异常处理器参数不会在线程之间共享
   * java线程之间的通行由java内存模型控制（Java Memory Model，JMM）
      * 线程之间的共享变量存储在主存（Main Memory）
      * 每个线程都已一份似有的本地内存（Local Memory），存储了主存内容的副本

3.1.3 从源代码到指令序列的重排序
   * 为了提升性能，编译器和处理器或做指令重拍
      * 编译器优化的重排：不改变单线程语义
      * 指令级并行重排：Instruction-Level—Parallelism，ILP
      * 内存系统的重排序
   * JMM是语言级的内存重排序

3.1.4 并发编程模型的分类
   * 现代处理器使用些缓冲区临时保存向内存写入的数据
      * 写缓冲区，尽自己可见数据
   * 现代处理器允许写-读 进行重排序
   * 为保证内存可见性，java编译器在生成指令序列的时候会插入内存屏障来静止特定类型的处理器重排序
      * LoadLoad Barriers
         * Load1 LoadLoad Load2： 确保load1数据的装载优先于load2及所有后续装载指令的装载
      * StoreStore Barriers
         * Store1 StoreStore Store2：确保store1的数据对其他处理器可见（刷新到内存）现有store2及所有后续的存储指令的存储
      * LoadStore Barriers
         * Load1 LoadStore Store2：确保load1数据装载优先于Store2及所有后续的存储指令刷新到到内存
      * StoreLoad Barriers
         * Store1 StoreLoad Load2：确保store1数据对其他处理器变得可见（刷新到内存）贤宇Load2及所有后续装载指令的装载
         * 同时具有其他三个指令的效果

3.1.5 happens-before
   * JDK5， JSR-133
   * 程序监视规则：一个吃现成中的每个操作，happens-before于该线程中的任意后续操作。
   * 监视器锁规则：对一个锁的解锁，happens-before与随后的对于这个锁的加锁
   * volatile变量规则：对于一个volatile域的鞋，happens-before与任意后续对这个volatile域的读
   * 传递性
   * happens-before规则仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前。

*3.2* 重排序
   * 数据依赖
   * as-if-serial
      * 不管怎么重排，程序结果不能变
   * 重排序对多线程的影响
      * 重排序破话了多线程语义

*3.3* 顺序一致性
   * 理论参考模型
      * 数据竞争与顺序一致性模型
         * 数据竞争的定义
            * 一个线程写变量A，另一个线程读A，而且写和读之间没有通过同步来排序
   * 顺序一致性模型
      * 一个线程中国的所有操作必须按照程序的顺序来执行
      * 不管是否使用同步，所有线程都只能看到一个单一的操作执行顺序。每个操作都必须原子执行且立马对所有线程可见
   * 同步程序的顺序一致性效果
   * 未同步程序的执行特性
      * JMM只提供最小安全性
         * JVM堆上分配对象是：首先对内存空间进行清零，然后才会在上面分配对象
      * JMM不保证单线程内的操作会按照程序的顺序执行（优化重排）
      * JMM不保证所有线程看到一致性的执行孙旭
      * JMM不保证对64位的long和double变量的写操作具有原子性
         * 与处理器工作机制先关
            * 总线事务

*3.4* volatile内存语义
   * volatile的特性
      * 可见性
      * 原子性
   * volatile写-读建立的happens-before关系
   * volatile写-读的内存语义
      * 当写volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存
      * 当读一个volatile变量是，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。
   * volatile实现的内存语义
      * OP2为volatile写时，不管OP1为何，都不能重排
         * 保证了volatile写之前的操作不会被编译器重排到volatile写之后
      * OP1为volatile读时，不管第二操作为什么，都不能重排序
         * 确保volatile度之后的操作不会被编译器重排序到volatile读之前
      * OP1为volatile写，OP2为volatile读时，不能重排序
      * 通过插入内存屏障来十进制特定类型的处理器重排序
         * 插入各类内存屏障，保守策略，有些会被优化掉
   * JSR-133 为什么会增强volatile的内存语义
      * 就模型中允许volatile变量与普通变量之间的重排序

*3.5* 锁的内存语义
   * 锁的释放 - 获取建立的happens-before关系
   * 锁的释放和获取的内存语义
      * 线程获取锁时，JMM会将该线程对应的本地内存置为无效。从而使得被简史其保护的临界区代码必须从主内存中去取共享变量。
   * 锁的内存语义的实现
      * CAS（compareAndSwap）具有volatile读和volatile写的内存语义
      * 程序会根据当前处理器的类型来决定是否为cmpxchg质量添加lock前缀
         * 如若在多处理器上运行：就位cmpxchg指令添加lock前缀（lock cmpxchg）
         * 不然，则不用添加：但处理器什会维护但处理器内的顺序一致性
      * lock前缀（interl手册）
         * 确保对内存的读-改-写操作原子执行
            * 奔腾及之前采用锁总线的方式，这使得其他处理器无法通过总线访问内存，开销大
            * p6等采用缓存锁定（cache locking）来保证指令执行的原子性
         * 禁止该指令与至前和之后的读写指令重排
         * 把写缓冲去中的所有数据刷新到内存中
      * 语义
         * 利用volatile变形的写-读所具有的内存语义
         * 利用CAS所附带的volatile读和volatile写的内存语义
   * concurrent包的实现
      * java线程之间四中通信方式
         * A线程写volatile变量，随后B线程读这个volatile变量
         * A线程写volatile变量，随后B线程用CAS跟新这个volatile变量
         * A线程用CAS更新一个volatile变量，随后B线程用CAS更新这个volatile变量
         * A线程用CAS根性一个volatile变量，随后B线程读这个volatile变量
      * 通用化的模式
         * 申明共享变量为volatile
         * 使用CAS的原子条件更新来实现线程之间的同步
         * 配合以volatile的读/写和CAS具有的volatile读和写的内存语义来实现线程之间的通信

*3.6* final域的内存语义
   * final域的重排序规则
      * 在构造函数中对一个final域的写入，域水偶吧这个被构造对象的引用复制给一个引用变量，者两个操作之间不能重排序
      * 初次读取一个不包含final域的对象的引用，与随后初次读这个final域，这个两个操作之间不能重排序
   * 写final域的重排序规则
      * JMM禁止编译器把final域的鞋重排序到构造函数之外
      * 编译器会在final域的写之后，构造函数return之前，插入一个storestore屏障。该屏障禁止处理器吧final域的写重新排序到构造函数之外
      * 规则保证：在丢向引用为任意线程可见之前，对象的final域已经被正确初始化了，而普通与不具有这个保障
   * 读final域的重排序规则
      * 在一个线程中，初次读队形引用于初次读该队形包含的final域，JMM禁止处理器重排序这两个操作（这个规则仅针对处理器）
         * 编译器会在final域的读操作之前插入一个loadload屏障
   * final域为引用类型
      * 在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
   * 为什么final引用不能从构造函数内"溢出"
   * final语义在处理器中实现
      * x86中没有插入任何屏障。
   * JSR-133  为什么要争抢final语义
      * 旧模型中，程序可能会看到final域的值会变化

*3.7* happens-before
   * JMM的设计
      * 考虑因素
         * 程序员希望对内存模型的使用：易于理解，予以编程，希望强内存模型
         * 编译器和处理器对内存模型的实现：希望内存模型对他们的束缚越少越好，这样他们就可以做尽可能多的优化来提高性能，希望实现一个弱内存模型。
      * happens-before禁止重排
         * 会改变程序执行结果的重排序
            * JMM要求编译器和处理器必须禁止这种重排序
         * 不会改变程序执行结果的重排序
            * JMM对编译器和处理器不做要求，JMM允许这种重排
      * 特点
         * JMM向程序员提供的happens-before规则能满足程序员的需求。
            * JMM的happens-before规则不但简单易懂，而且也向程序员提供了足够强的内存可见想保证。
         * JMM对编译器和处理器的束缚已经尽可能少
            * 只要不改变程序运行结果，编译器和处理器怎么优化都行
               * 单线程程序和正确同步的多线程程序
   * happens-before的定义
      * JSR-133 定义如下
         * 如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。
            * 这是JMM对程序员的承诺
         * 两个操作之间存在happens-before关系，并不意味着java平台的具体实现必须按照happens-before制定的规则来执行。
            * 如果重排序之后的执行结果，与按happens-before关系来执行的结果一直，那么这种重排序并不非法。
            * 这是JMM对编译器和处理器重排序的约束原则
      * as-if-serial VS. happens-before
         * as-if-serial保证了单线程内程序的执行结果不被改变，happens-before关系保证正确同步的多线程程序的执行结果不被改变。
         * as-if-serial给编写单线程程序的程序员创造了一个幻境：单线程程序是按照程序的顺序来执行的。happens-before规则则给了正确同步的程序一样的幻觉。
         * 都是为了在不改变程序执行结果的前提下，尽可能的提高程序执行的并行度。
   * happens-before规则：JSR-133
      * 程序顺序规则：一个线程中到的每个操作，happens-before于该线程中任意后续操作
      * 监视器锁规则：对一个锁的解锁，happens-before于随后的对这个锁的加锁
      * volatile变量规则：对一个volatile域的写，happens-before与任意后续对这个volatile域的读
      * 传递性：A happens-before B, B happens-before ，那么A happens-before C
      * star() 规则：如果线程A执行操作ThreadB.start，那么A线城的ThreadB.start操作happens-before于线程B中的任意操作
      * join() 规则：如果线程A执行操作ThreadB.join，那么线程B中的任意操作happens-before于线程B中的任意操作

*3.8* 双重检查锁定与延迟初始化
   * 双重检查锁定的由来
      * 没有使用volatile的双重锁，仍然存在问题
   * 问题的根源
      * 对象的初始化：存在着指令重排
         * 1.memory=allocate() // 分配对象的内存空间
         * 2.ctrolInstance(memory) // 初始化对象
         * 3.instance=memory //设置instance纸箱刚分配的内存地址
      * 如何解决这个问题
         * 不允许2和3重排
         * 允许2和3重排，但是不允许其他线程"看到"这个重排
   * 基于volatile的解决方案
      * 禁止了2和3之间的重排
      * volatile instance = new Instance()
   * 基于类初始化的解决方案
      * 在首次发生任一如下情况是，一个雷火接口类型T将被立即初始化
         * T是一个类，而且一个T类型的实例将被创建  new T()
         * T是一个类，且T中声明的一个静态方法被调用 T.staticMethod()
         * T中声明的一个静态字段被赋值
         * T中声明的一个静态字段被使用，而且这个字段不是常量字段。
         * T是一个定级类（Top Level Class），而且一个断言语句嵌套在T内部被执行
      * 每个类或者接口C，都有一个唯一的初始化锁与之对应。
      * 初始化类或者接口的五个阶段
         * 初始时，class尚未初始化时，state被标记为state=noInitialization
         * 1. 通过在class对象上同步（即获取class对象的同步锁），来控制类或接口的初始化。这个获取锁的线程会一直等待，直到当前线程能够获取到这个初始化锁。
            * 第一个成功获取的线程会将state=initializing
         * 2. 线程A执行类的初始化，同时线程B在初始化锁对应的condition上等待
         * 3. 线程A设置state=initialized，然后唤醒在condition中等待的所有线程
         * 4. 线程B（总共获取两次锁）结束初始化处理。
         * 5. 线程C执行类的初始化处理（C只会获取一次锁）

*3.9* Java内存模型综述
   * 处理器的内存模型
      * 顺序一致性是一个理论参考模型
      * 根据对不同类型的读/写操作组合的执行顺序的放松，可以将长剑的处理器的内存模型划分为如下几类
         * Total Store Ordering ： 放松程序中写-读操作 （TSO）
         * 在上面的技术上，继续放松写-写操作，产生 Partial Store Order内存模型（PSO）
         * 继续放松读-写和写读-读，产生了Relax Memory Order（RMO）和Power PC内存模型
      * 各种内存模型之间的关系
         * 常见的四种处理器内存模型比常用的3中语言内存模型要弱，且都比顺序一致性模型要弱
         * 同一种语言，越是最求性能，内存模型设计的会越弱
      * JMM的内存模型可见性保证
         * 单线程程序：不会出现内存可见性问题
         * 正确同步的多线程程序：
         * 未同步/未正确同步的多线程程序：JMM为其提供了最小安全性保障，线程执行时读取到的值，要么是之前某个线程写入的值，要么是默认值（0、null、false）
      * JSR-133 对就内存模型的朽败
         * 主要有两个
            * 增强volatile的内存语义
            * 增强final的内存语义



**Chapter 4 Java 并发编程基础**

*4.1 线程简介*
   * 什么是线程
      * light weight process
   * 为什么要使用多线程
      * 更多的处理器核心
      * 更快的相应时间
      * 更好的编程模型
   * 线程优先级
      * priority [1，10]
   * 线程的状态
      * NEW：初始状态，还没调用start()
      * RUNNABLE：运行状态，包含就绪和运行，统称运行中
      * BLOCKED：阻塞状态，表示线程阻塞与锁
      * WAITING：等待状态，表示线程进入等待状态，进入该状态表示当前线程需要等待其他线程做出一些特定动作（通知或中断）
      * TIME_WAITING：超时等待状态，其可以再指定时间内返回
      * TERMINATED：终止状态
   * Daemon线程
      * 程序退出时，Daemon线程中finally块不一定会执行

*4.2 启动和终止线程*
   * 构造线程
      * parent thread为子线程设置好daemon，priority等
   * 启动
      * start()，线程最好设置名字，这样好debug问题，jstack
   * 理解中断
      * 声明抛出InterruptedException的方法，这些方法在抛出异常之前，jvm会线程程序中的中断标识清除
   * 过期的suspend，resume，stop
      * 过期的主要原因：
         * suspend掉用户，线程不会释放已经占有的资源（比如锁），容易引发死锁
         * stop，终结线程是，也不是释放资源
   * 安全地终止线程
      * 除了interrupt，还可以用boolean变量来控制是否需要停止任务并终止该线程

*4.3 线程间通信*
   * volatile和synchronized关键字
      * 同步块使用monitorenter和monitorexit指令
      * 同步方法依靠方法修饰符上的ACC_SYNCHRONIZED来完成
      * 任意一个对象都有自己的监视器
         * 同步队列
   * 等待/通知机制
      * 生产者和消费者
         * 消费者自旋
            * 问题：难以保证及时性
            * 问题：难以降低开销
      * 相关方法
         * notify
         * notifyAll
         * wait
         * wait(long)
         * wait(long, int)
         * 上述方法需要在获取锁之后调用 [[why](https://stackoverflow.com/questions/2779484/why-must-wait-always-be-in-synchronized-block])]
            * 需要放入对象的wait pool
               * 需要同步语义，不然会导致并发上的错误。
   * 等待/通知的经典范式
      * 等待方原则
         * 获取对象的锁
         * 如果条件不满足，则调用对象的wait方法，被通知仍然要检查条件
         * 条件满足则执行对应的逻辑
         ```java
         synchronized(obj) {
         while( condition not ready) {
            obj.wait()
         }
         stuff()
      * 通知方原则
          * 获取对象的锁
          * 改变条件
          * 通知所有等待在对象上的线程
          ```java
          synchronized(obj) {
             change condition : make condition ready
             obj.notify()
          }
   * 管道输入/输出流
      * PipedOutputStream, PipedInputStream
          * 字节流
      * PipedReader, PipedWriter
          * 字符
   * Thread.join()的使用
   * ThreadLocal的使用

*4.4 线程应用实例*
   * 等待超时模式
      ```java
      public synchronized Object get(long mills) throws InterruptedException {
        long future = System.currentTimeMills() + mills;
        long remaining = mills;
        while ((reuslt == null) && remaining > 0){
             wait(remaining);
             remaining = future - System.currentTimeMills();
        }
      }
   * 简单数据库连接池实例
   * 线程池技术及其示例
   * 基于线程池技术的简单Web服务器

**Chapter 5 Java中的锁**

*5.1 Lock接口*
   * 较之于synchronized
      * 缺少了隐式获取与释放的便捷性
      * 增加了所获取与释放的可操作性、可中断的获取所以及超时获取锁等多种synchronized关键字锁不具备的特性
         * 更加灵活
   * 主要新增特性
      * 尝试非阻塞地获取锁
      * 能被终端地获取锁
      * 超时获取锁
   * api
      * lock()：阻塞获取锁
      * lockInterruptibly()：相应中断
      * tryLock(): 立即返回，成功或失败获取锁
      * tryLock(long time，TimeUnit unit): 超时获取锁，超时时间内获取锁、当前线程被中断、超时时间结束
      * unlock()：释放锁
      * Condition newCondition(): 调用conditon.wait()前提是获取锁, 调用之后释放锁

*5.2 队列同步器*       Doug Lea
   * AbstractQueuedSynchronized（AQS）
      * int 变量表示同步状态
      * FIFO队列完成资源获取线程的排队工作
      * 通过继承来使用
      * 推荐定义为自定义同步组件的静态内部类
      * 同步器与锁的关系
         * 锁是面向使用者的，定义了使用者与锁交互的接口
         * 同步器是面向锁的实现者
   * 队列同步器的接口与实例
      * 需要使用的接口
         * getState
         * setState
         * compareAndSetState(int except, int update)
      * 同步器可重写的方法
         * tryAccquire(int arg)： 独占是获取同步状态
         * tryRelease(int arg)： 独占式释放同步状态
         * tryAccquireShared(int arg)：共享式获取同步状态
         * tryReleaseShared(int arg)：共享式释放同步状态
         * isHeldExclusively()：当前线程是否在独占模式下被线程占用
      * 同步器提供的模板方法
         * 独占式获取和释放同步状态
         * 共享式获取和释放同步状态
         * 查询同步队列中的等待线程状态
   * 队列同步器的实现分析
      * 同步队列
         * Node：为同步队列和等待队列公用的数据结构
            * waitStatus：CANNCELLED、SIGNAL、CONDITION、PROPAGATE、INITIAL
            * Node prev
            * Node next
            * Node nextWaiter：等待队列中的后继结点
            * Thread thread： 获取同步状态的线程
         * compareAndSetTail(Node except, Node update)
         * 遵循FIFO，首节点是获取同步状态成功的节点，首节点线程在释放同步状态时，会唤醒后继节点，后继节点将会在获取同步状态成功时将自己设置为首节点
            * 设置为head，不用CAS来保证，原因是已经获取了锁
      * 独占式同步状态的获取与释放
         * 获取
            * 使用CAS使得入队串行化
            * accquireQueued：死循环尝试获取同步状态
               * 只有前驱节点是头节点才能获取同步状态
                  * 头节点获取同步状态，释放之后，唤醒后继
                  * 维护FIFO
             ```java
             public final accquire(int arg) {
                 if (!tryAccquire(arg)
                    && accuqireQueued(add(Witer(Node.EXCLUSIVE), arg)))
                    sefinterrupt();
             }

         * 释放
             ```java
             public final boolean release(int arg) {
                if (tryRelease(arg)) {
                    Node h = head;
                    if (h != null && h.waitStatus != 0) {
                        unparkSuccessor(h);
                    }
                    return true;
                return false;
                }
             }
      * 共享式同步状态的获取与释放
         * accqurieShared(int arg)
         * releaseShared(int arg)
         * 支持多个线程同时访问并发组件
            * 如Semaphore
      * 独占式超时获取同步状态
         * doAccqureNanos(int arg, long nanosTimeout)

*5.3 重入锁*
   * synchronized
      * 隐式重入：获取锁的线程可以再次获取锁
   * 实现重入
      * 重进入：在人以县城获取到锁会后能够再次获取该所而不会被锁阻塞，该特性解决的问题
         * 1. 线程再次获取锁：锁需要识别获取锁的线程是否是当前占据的线程，如是则成功
         * 2. 锁的最终释放：lock n times，unlock n times
   * 公平与非公获取锁的区别
      * 公平性与否是针对获取所而言的，如果一个锁是公平的，则锁的获取应该遵循FIFO
      * 对于获取锁而言：公平的在获取锁的时候需要检测是否有前驱节点
      * 非公平锁会导致线程有机会连续获取锁
         * 减少了上下文切换，吞吐量更大

*5.4 读写锁*
   * ReentrantReadWriteLock 特性
      * 公平性选择：支持非公平（默认）到的锁获取方式，吞吐量还是非公平优先
      * 重入性：支持重入
         * 以读写线程为例：度线程在获取读锁之后，能够再次获取读锁。
         * 写线程获取写锁之后还能再次获取写锁，同时也可以获取读锁
      * 锁降级：遵循获取写锁、获取读锁再释放写锁的次序，写锁能够降级为读锁
   * 读写锁的接口与示例
      * getReadLockCount：返回当前读锁被获取的次数
      * getReadHoldCount：返回当前线程获取读锁的次数
      * isWriteLocked： 判断写锁是否被获取
      * getWriteHoldCount：返回当前写锁被获取的次数
   * 读写锁的实现分析
      * 读写状态的设计
         * 按位切换使用state：高16位表示读，低16位表示写
         * 推论：S不等于0时，当写状态（S&0X0000FFFF）等于0，则读状态（S>>>16）大于0，即读锁已经被获取
      * 写锁的释放与获取
         * 支持重入的排他锁
         * 如果当前线程已经获取了写锁，则增加写状态
         * 如果当前线程在获取写锁时，读锁已经被获取（读状态不为0）或者该线程不是已经获取写锁的线程，则进入等待状态
      * 读锁的获取与释放
         * 支持重入
         * 在无写锁时，读锁总能成功获取
         * 如果已经获取读锁，则增加读状态
      * 锁降级：写锁降级为读锁
         * 获取写锁，获取读锁，释放写锁
         * 读锁的目的是保证数据可见性

*5.5 LockSupport工具*
   * park： 阻塞当前线程
   * parkNanos(long nanos)：阻塞当前线程，最长不操作nanos纳秒
   * parkUntil(long deadline)：阻塞当前线程，知道deadline时间
   * unpark(Thread thread)

*5.6 Condition 接口*
   * object monitor methods VS. Condition

      |对比项|Object Monitor Methods| Condition|
      |---|---|---|
      |前置条件|获取对象锁|调用Lock.lock获取锁，调用Lock.newCondition获取Condition对象|
      |调用方式|object.wait()|condition.wait()|
      |等待队列的个数|1个|n个|
      |当前线程释放锁并进入等待状态|支持|支持|
      |当前线程释放锁并进入等待状态，相应中断|不支持|支持|
      |当前线程释放锁并接入超时等待状态|支持|支持|
      |当前线程释放锁并进入等待状态到某个时间|不支持|支持|
      |唤醒等待队列中的一个线程|支持|支持|
      |唤醒等待队列中的所有线程|支持|支持|
   * Condition 的实现分析
      * ConditionObject为AbstractQueuedSynchronized的内部类
      * 等待队列
         * FIFO队列，队列节点类型为AQS.Node
         * 含首尾节点的单项链表
         * 使用之前需获取锁，故而操作不用CAS
         * diff
            * Object监视器模型上：一个对象有一个同步队列和等待队列。
            * Lock拥有一个同步队列和多个等待队列
      * 等待
         * await：将线程从同步队列首节点移动到等待队列
      * 通知
         * signal：将线程从等待队列移动到同步队列


** Chapter 6 java并发容器和框架**

*6.1 ConcurrentHashMap的实现原理与使用*
   * 为什么要使用ConcurrentHashMap
      * 并发编程中使用HashMap可能会导致程序死循环，而线程安全的HashTable又效率低
      * 线程不安全的HashMap
         * entry列表形成唤醒结构
      * 效率低下的HashTable
         * 使用Synchronized来同步
         * 所有操作使用同一把锁
      * ConcurrentHashMap的锁分段技术可有效替身并发访问率
         * 使用锁分段技术
         * 数据一段一段的存储，每个段一把锁
   * ConcurrentHashMap的结构(JDK1.7)
      * concurrentHashMap->Segment[] (可重入锁)->HashEntry[] (存储数据, 链式)
      * concurrentHashMap的初始化
         * 初始化segment数组（默认16）
         * 初始化segmentShift和segmentMask
         * 初始化每个segment
      * 定位segment
         * 对元素的hashCode再hash
      * ConcurrentHashMap的操作
         * get
            ```java
            public V get(Object key) {
               int hash = hash(key.hashCode());
               return segmentFro(hash).get(key, hash);
            }
         * put：扩容，如何扩容
         * size：volatile，modCount
   * ConcurrentHashMap（JDK1.8）
      * 取消segments字段，直接使用transient volatile HashEntry<K,V>[] table保存数据，采用table元素作为锁，从而实现对每一行数据加锁，进一步减少并发冲突的概率
      * 将原先的table数组 + 单向链表的数据结构，变为table数组+单向链表+红黑树数据结构
         * 单向链表长度为8，则使用红黑树存储

*6.2 ConcurrentLinkedQueue*
   * 非阻塞队列：使用CAS实现
   * 结构：head + tail
   * 入队：将入队节点添加到队列的尾部
      * 定位尾节点
      * 使用CAS算法将入队节点设置为尾节点的next节点
      * 其中tail节点不一定是最后一个节点，有可能是倒数第二个节点
      * HOPS
   * 出队
      * HOPS + CAS

*6.3 Java中的阻塞队列*
   * 什么是阻塞队列
      * 阻塞插入阻塞移除

      |方法/处理方式|抛出异常|返回特殊值|一直阻塞|超时退出|
      |---|---|---|---|---|
      |插入方法|add(e)|offer(e)|put(e)|offer(e, time, unit)|
      |移除方法|remove()|pool()|take()|poll(time,unit)|
      |检查方法|element()|peek()|不可用|不可用|
   * Java里面的阻塞队列
      * ArrayBlockingQueue
         * 用数组实现的有界阻塞队列，遵循FIFO
         * 默认情况下不保证线程公平的访问队列
            * 保证公平则会降低吞吐量
      * LinkedBlockingQueue
         * 用链表实现的有界阻塞队列
            * FIFO
            * 默认最大长度为Integer.MAX_VALUE
      * PriorityBlockingQueue
         * 支持优先级的无界阻塞队列
      * DelayQueue
         * 使用PriorityQueue实现
         * 只有延时期满是才能从队列中提取元素
         * 使用场景：
            * 缓存系统设计：使用DelayQueue保存缓存元素的有效期
            * 定时任务调度
         * 实现delay接口
         * 实现延时阻塞队列
      * SynchronousQueue
         * 不存储元素的阻塞队列
         * 支持公平访问
         * 适合传递性场景
      * LinkedTransferQueue
         * 由链表结构组成的无界则色TransferQueue队列
      * LinkedBlockingDeque
   * 阻塞队列的实现原理
      * 使用通知模式实现
         * unsafe.park

*6.4 Fork/Join框架*
   * 什么是Fork/Join框架
      * JDK 1.7提供的用于并行执行任务的框架
         * 有点像分治
         * 本地的MapReduce
   * 工作窃取算法
      * 某个线程从其他队列里窃取任务来执行。
         * 每个线程负责一个任务队列（双端队列）
         * 一个线程完成其任务队列之后，在其他队列中尾部获取任务来执行
      * 优点
         * 充分利用线程进行并行计算，减少了线程间的竞争
      * 缺点
         * 某些情况下存在竞争
         * 该算法消耗了更多的线程资源，对个线程和多个双端队列
   * Fork/Join框架的设计
      * 分割任务：大人物分割成小任务
      * 执行任务合并结果
         * 分割的子任务分别放在双端队列里，然后几个线程从双端队列中获取任务执行
         * 子任务执行完的结果都统一放在一个队列里
         * 启动一个线程从队列中取数据，合并这些数据
      * ForkJoinTask
         * RecursiveAction：用于没有返回结果的任务
         * RecursiveTask：用于有返回结果的任务
      * ForkJoinPool
         * ForkJoinTask需要通过ForkJoinPool来执行
   * 使用Fork/Join框架
   * Fork/Join框架的异常处理
      * ForkJoinTask提供了isCompletedAbnormally()方法
      * task.getException()
   * Fork/Join框架的实现原理
      * ForkJoinPool 由ForkJoinTask素组和ForkJoinWorkerThread数组组成
      * ForkJoinTask的fork方法实现原理
         * 调用fork时，会挑用ForkJoinWorkerThread的pushTask方法异步地执行这个任务
            * 如何做到异步的？？？
      * ForkJoinTask的join方法实现原理
         * doJoin

**Chapter 7 Java中的13个原子操作类**

*7.1 原子更新基本类型类*
   * AtomicBoolean
   * AtomicInteger
   * AtomicLong
      * addAndGet
      * compareAndSet
      * getAndIncrement
      * lazySet
      * getAndSet
      ```java
      public final int getAndIncrement() {
         for (;;) {
             int current = get();
             int next = current + 1;
             if (compareAndSet(current, next)) {
                 return current;
              }
         }
      }
      public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, vaueOffset, expcet, update);
      }
    ```
*7.2 原子更新数组*
   * AtomicIntegerArray
   * AtomicLongArray
   * AtomicReferenceArray
   * AtomicIntegerArray
      * int addAndGet(int i, int delta)：以原子方式将输入值与数组中索引i的元素相加
      * boolean compareAndSet(int i , int except, int update)

*7.3 原子更新引用类型*
   * AtomicReference：原子更新引用类型
   * AtomicReferenceFieldUpdater：原子更新引用类型里的字段
   * AtomicMarkableReference：原子更新带有标记为的引用类型

*7.4 原子更新字段类*
   * AtomicIntegerFieldUpdater
   * AtomicLongFieldUpdater
   * AtomicStampedReference

**Chapter 8 Java中的工具类**

*8.1 等待做线程文成的CountDownLatch*
   * CountDownLatch允许一个或多个线程等待其他线程完成操作
      * 不可重新初始化

*8.2 同步屏障CyclicBarrier*
   * 让一组线程达到一个屏障（同步点）是被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才能继续运行。
      * CyclicBarrier(n);
      * CyclicBarrier(n, Runnable action); 优先执行action
   * 应用场景
      * 应用于多线程计算数据，最后合并计算结果的场景。
   * CyclicBarrier VS. CountDownLatch
      * CountDownLatch 只能使用一次，CyclicBarrier可重置
      * CyclicBarrier 提供
         * getNumberWaiting
         * isBroken

*8.3 控制并发线程数的Semaphore*
   * 应用场景
      * 用作流量控制
      * acquire + release
   * 其他方法
      * int availablePermits()
      * int getQueueLength()
      * boolean hasQueuedThreads()
      * Collection getQueuedThreads()

*8.4 线程间交换数据*
   * 用于线程间协作的工具类
   * 可以用户遗传算法
   * 可用户校对工作
   * exchange(V x, long timeout, TimeUnit unit)


**Chapter 9 Java中的线程池**

* 合理使用线程池的好处
   * 降低资源消耗
   * 提高响应速度
   * 提高线程的可管理性

*9.1 线程池的实现原理*
   * 线程池处理流程
      * 1.线程池判断核心线程池里的线程时候都在执行任务。若不是则创建一个新的工作线程来执行任务，若是则进入下一个流程
      * 2.线程池判断工作队列是否已满。若未满，则将新提交的任务存储在工作队列里面；若满，则进入下一个流程
      * 3.线程池判断线程池的线程是否都处于工作状态。若否，则创建一个新的线程来执行任务；若是，则交个饱和策略来处理这个任务
   * ThreadPoolExecutor.execute的执行逻辑
      * 1.如果当前运行的线程少于corePoolSize，则创建新线程来执行任务（执行这一步需要获取全局锁，因为要更新workers）
      * 2.如果运行线程数等于或多于corePoolSiz，则将任务加入到BlockingQueue
      * 3.如果无法将任务加入BlockingQueue（队列已满），则创建新线程来处理（这一步也要获取全局锁）
      * 4如果创建新线程将使得当前运行的超出maximumPoolSize，任务将被拒绝，并调用RejectedExecutionHandler.rejectedExecution()来执行
   * 工作线程
      * 线城池创建线程是，会将线程封装成工作线程Worker，worker在执行完成第一个任务（如果有的话），会循环获取工作队列里面的任务来执行。

*9.2 线程池的使用*
   * 创建
      ```java
      new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, milliseconds, runnableTaskQueue, handler);
      ```
      * corePoolSize：线程池基本大小
         * prestartAllCoreThreads()方法会提前创建并启动所有的基本线程
      * runnableTaskQueue：用于保存等待执行的任务的阻塞队列
         * ArrayBlockingQueue：有界阻数据塞队列，FIFO
         * LinkedBlockingQueue：有界阻链表塞队列，FIFO，吞吐量高于ArrayBlockingQueue
            * 在Executors.newFixedThreadPool中使用
         * SynchronousQueue：不存储元素的阻塞队列，吞吐量通常高于LinkedBlockingQueue
            * 在Executors.newCachedThreadPool中使用
         * PriorityBlockingQueue：具有优先级的无界阻塞队列
      * maximumPoolSize：线程池允许创建的最大线程数。如果队列满，且已创建的线程数少于对打线程数，则线程池会再创建新的线程执行任务。
         * 如果使用无界队列，这个参数就没有效果了。
      * ThreadFactory：创建线程的工厂，一般创建有名的线程，使用guava
         ```java
         new ThreadFactoryBuilder.setNameFormat("XXX-task-%d").build();
        ```
      * RejectedExecutionHandler( 饱和策略)：当队列和线程池都满的时候，说明线程池处于饱和状态，这时候对于新任务的策略
         * AbortPolicy：直接跑出异常，默认的策略
         * CallerRunsPolicy：只用提交者所在的线程来运行任务
         * DiscardOldestPolicy：丢弃队列里最旧的任务，并执行当前任务
         * DiscardPolicy：不处理，丢弃掉
         * 亦可以自定义来实现RejectedExecutionHandler接口。
      * keepAliveTime：线程活动保存时间，线程池的工作线程空闲后，保持存货的时间。
         * 任务多，且每个任务执行时间短，可以调大时间，提高线程的利用率
      * TimeUnit：keepAliveTime的单位
   * 向线程池提交任务
      * execute：提交不需要返回值的任务
      * submit：提交有返回值的任务
   * 关闭线程池
      * shutdown
         * 遍历工作线程，然后调用interrupt
         * 设置线程池状态为SHUTDOWN
      * shutdownNow
         * 遍历工作线程，然后调用interrupt
         * 设置线程池状态为STOP，尝试停止所有正在执行或暂停任务的线程，并返回执行任务的列表
      * isShutdown
      * isTerminated
   * 合理地配置线程池
      * 任务的性质：CPU密集型，IO密集型还是混合型任务
         * CPU密集型：尽量少的线程，Ncpu + 1
         * IO密集型，尽量多的线程，Ncpu * 2
         * 混合型：分多个线程池
      * 任务的优先级：高、中和低
      * 任务的执行时间：长、中和短
      * 任务的依赖性：是否依赖其他系统资源，如数据库连接
      * 建议使用有界队列
         * **能增加系统的稳定性和预警能力**
   * 线程池监控
      * taskCount：线程池需要执行的任务数量
      * completedTaskCount：线程池在运行过程中已经完成的任务数量，小于或等于taskCount
      * largestPoolSize：线程池里曾经创建过的最大线程数量
      * getPoolSize：线城池的线程数量
      * getActiveCount：获取活动的线程数量
      * 通过继承线程池定义来自定义线程池
         * beforeExecute
         * afterExecute
         * terminated

**Chapter 10 Executor框架**

*10.1 Executor框架简介*
   * Executor框架的两级调度模型
      * HotSpot VM线程模型中，java线程被一一映射为本地OS线程
      * 上层：java多线程将应用费结尾若干个任务，然后使用用户级的调度器将任务映射为固定数量的线程
      * 底层：OS内核将这些线程映射到硬件处理器上。
   * Executor框架的结构和成员
      * Executor的框架结构
         * 任务
            * 包括被执行任务需要实现的接口：Runnable接口或Callable接口
         * 任务的执行
            * 包括任务执行机制的核心接口Executor，以及继承自Executor的ExecutorService接口。
            * Executor框架有两个关键类实现了ExecutorService接口（ThreadPoolExecutor和ScheduledThreadPoolExecutor）
         * 异步计算的结果
            * 包括接口Future和实现Future接口的FutureTask类
         * 接口
            * Executor：完成任务的提交和任务的执行的分离
            * ThreadPoolExecutor：线程池的核心实现类，用来提交被执行的任务
            * ScheduledThreadPoolExecutor：一个实现类，可以再给定的延迟后运行命令，或者定期执行
               * 比Timer更加强大，灵活
            * Future， FutureTask：代表异步计算的结果
            * Runnable和Callable接口的实现类，都可以被ThreadPoolExecutor和ScheduledThreadPoolExecutor执行
         * 过程
            * 主线程首先要创建实现Runnable或者Callable接口的任务对象
               * 封装为Executors.Callable对象
            * FutureTask.get()
      * Executor框架的成员
         * ThreadPoolExecutor
            * FixedThreadPool：适用于为了满足资源管理的需求，而需要限制当前线程数量的应用场景。
               * 适用于负载比较高的服务器
            * SingleThreadExecutor
               * 适用于需要保证顺序地执行各个任务，并且在任意时间点不会有多个线程是活动的的场景。
            * CachedThreadPool
               * 是大小无界的线程池，适用于执行很多的短期异步任务的小程序，或者是负载较轻的服务器
         * ScheduledThreadPoolExecutor
            * ScheduledThreadPoolExecutor：包含多个线程
            * SingleThreadScheduledExecutor：只包含一个线程
               * 适用于需要单个后台线程执行周期任务，同时需要保证顺序地执行各个人物的应用场景
         * Future接口
         * Runnable接口和Callable接口
            * Runnable不会返回结果
            * Callable会返回结果

*10.2 ThreadPoolExecutor详解
   * 参数
      * corePool：核心池大小
      * maximumPool：最大线程池大小
      * BlockingQueue：用来暂时保存任务的工作队列
      * RejectedExecutionHandler：当ThreadPoolExecutor已经关闭或者已经饱和是（线程池满且工作队列已满），execute方法要调用的handler
      * 三种类型的ThreadPoolExecutor
         * FixedThreadPool
         * SingleThreadPool
         * CachedThreadPool
   * FixedThreadPool
      * 可重用固定线程数的线程池
         ```java
         public static ExecutorService newFixedThreadPool(int nThreads) {
            return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLSECONDS, new LinkedBlockingQueue<Runnable>)
         }
         ```
         * 核心线程数和最大线程数相同
         * keepLive为0，以为这多余的空闲线程会被立即停止
         * 使用了无界队列作为任务队列
            * 1. 当线程池中线程数量达到corePoolSize，新任务将在无界队列中等大，因此线程池中线程数量不会操作corePoolSize
            * 2. 由于1，使用无界队列时，maximumPoolSize将是一个无效的参数
            * 3. 由于1&2，keepAliveTime也无效了
            * 4. 运行中的FixedThreadPool不会拒绝任务（无界队列）
         * 步骤
            * 如果当前线程数少于corePoolSize，则创建新线程来执行任务
            * 在线程池预热之后（当前线程数等于corePoolSize），将任务加入到LinkedBlockingQueue
   * SingleThrScheduledThreadPoolExecutoreadExecutor
      * 使用单个线程
         ```java
         public static ExecutorService newSingleThreadExecutor() {
            return new FinalizableDelegatedExecutorService(1,1,0,TimeUnit.MILLISECONDS, new LinkedBlockingQueue());
         }
         ```
   * CachedThreadPool
      * 是一个根据需要创建新线程的线城池
         ```java
         public static ExecutorService newCachedThreadPool() {
            return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<Runnable>());
         }
         ```
      * 核心池为0，最大为Integer.MAX_VALUE
      * keepAliveTime为60s，意味着CachedThreadPool中的空闲线程等待新任务的最长时间为60s，超过则会被终止
      * 步骤
         * 1. 主线程执行SynchronousQueue.offer(Runnable task)，如果当前在线程池中有空前线程在执行SynchronousQueue.poll(keepAliveTime, TiimeUnit.NANOSECONDS)
            * 则配对成功，主线程将任务交给空闲线程执行
            * 否则执行2
         * 2. 当初始maximumPool为空，或者maximumPool中没有空闲线程是，将没有线程执行SynchronousQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)
            * 这种情况下步骤1将失败，此时CachedThreadPool会创建一个新的线程执行任务
         * 3. 步骤2中创建的线程将任务执行完后，会执SynchronousQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)。这个poll操作会让空闲线程最多活keepAliveTime
      * 长时间空闲的CachedThreadPool不适用任何资源

*10.3 ScheduledThreadPoolExecutor详解*
   * ScheduledThreadPoolExecutor 的运行机制
      * DelayQueue是一个无界队列，故而线程池数量只有corePoolSize个
      * 执行
         * 1. 当调用ScheduledThreadPoolExecutor的scheduleAtFixRate或者scheduleWithFixedDelay方法时，会想DelayQueue中添加一个西线了RunnaleScheduleFuture接口的ScheduledFutureTask
         * 2. 线程池中的线程从DelayQueue中获取ScheduledFutureTask，然后执行
      * 对ThreadPoolExecutor做了如下的修改
         * 使用DelayQueue作为任务队列
         * 获取任务的方式不同
         * 执行周期任务后，增加了额外的处理
   * ScheduledThreadPoolExecutor的实现
      * ScheduledFutureTask
          * time：Long，任务将要被执行的具体时间
          * sequenceNumber：Long，表示这个任务呗加到ScheduledThreadPoolExecutor中的序号
          * period：Long，表示任务执行的间隔周期
      * DelayQueue封装了一个PriorityQueue，会对其中的ScheduledFutureTask排序：time， sequenceNumber
      * 执行步骤
         * 1. 线程从DelayQueue中获取已经到期的ScheduledFutureTask（DelayQueue.take）。
            * 到期任务是指ScheduledFutureTask的time大于等于当前的时间
         * 2. 线程执行这个ScheduledFutureTask
         * 3. 线程修改ScheduledFutureTask的tie变量为下次将要被执行的时间
         * 4. 线程将修改time之后的ScheduledFutureTask放回到DelayQueue中（DelayQueue.add）
      * take的步骤
         * 获取lock
         * 获取周期任务
            * 如果PriorityQueue为空，当前线程到Condition中等待，否则
            * 如果队列的头元素的time比当前时间打，到Condition中等待到time时间，否则
            * 获取队列头元素；如果队列非空，则唤醒Condition当中等待的所有线程
         * 释放所
      * add的步骤
         * 获取lock
         * 添加任务
            * 加入队列
            * 如果之前队列空，或者新加的成为小于之前的头元素，则唤醒所有登台的线程
         * 释放锁

*10.4 FutureTask讲解*
   * FutureTask简介
      * FutureTask的3中状态
         * 未启动：new FutureTask
         * 已启动：FutureTask.run
            * 已启动的task，
               * 调用futureTask.cancel（true）中断
               * 调用futureTask.cancel（false）不会对正在执行的线程产生影响（让正在执行的任务完成）
         * 已完成：正常结束、取消、异常
   * FutureTask的实现
      * 实现基于AQS（AbstractQueuedSynchronizer）
      * FutureTask.get
         * 调用AQS.acquireSharedInterruptibly(int arg)，
            * 这个方法首先回调ync中实现的tryAcquireShare方法来判断acquire是否成功，成功调点是state为RAN或CANCLELLED，且runner不为null
         * 如果成功则get返回。如果失败则到线程等待队列中等起其他线程执行release操作
         * 其他线程release操作，当前线程再次执行tryAcquireShared，当前线程离开线程等待队列并唤醒他的后继线程
         * 最后返回计算的结果或者抛出异常
      * FutureTask.run执行过程
         * 执行构造函数中指定的任务：Callable.call
         * 以原子方式来更新同步状态（AQS.compareAndSetState(int expect, int update)设state为执行完成状态RAN）
            * 如果操作成功，设置计算结果result的值为Callable.call，然后调用AQS.release
         * AQS.releaseShared 首先会毁掉在自雷Sync中实现的tryReleaseShared来执行release操作；唤醒等待队列中的第一个线程
         * 调用FutureTask.done

**11 Java并发编程实战**

*11.1 生产者消费者模式**
   * 使用容器来解决两者的强耦合问题
   * 单线程串行 ==> 多线程生产者消费者
   * 多生产者多消费者
   * 线程池与生产者消费者模式

*11.2 线上问题定位*
   * top
   * jstatck

*11.3 性能测试*
   * netstat -anp
   * ps -eLf

*11.4 异步线程池*
   * 落地，持久化
   * 任务的窗台：NEW，EXECUTING、RETRY、SUSPEND、TERMINER、FINISH
   * 任务池任务的隔离
      * 不同的线程池处理不同优先级的任务
   * 任务池的重试策略
      * 为不同的任务设置不同的重试策略
   * 使用任务池的注意事项
      * 任务需要无状态
   * 异步任务的属性
      * 名称、下次执行时间、一致性次数、任务类型、任务优先级和执行是的报错信心
