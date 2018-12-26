Java Reactor VS Proactor

#### Q: 为什么java NIO 比 BI O性能要好？
这个问题说的有点绝对了，的确在一些场景下NIO要比BIO的性能要好不少。
对于每个请求的处理时间较短（或者说每个请求的处理较为简单）的场景 ，使用NIO性能有较为显著的提升，单机QPS也有较大的提升，举个例子，redis server 使用单线程的 reactor，单机能扛到万级别的qps。

对于单个请求处理时间长，比如一个 request 需要传输一个大的文件，在这个场景下，使用 reactor 并不能获得太多的收益，这个时候还不如使用 Connection Per request 的模式，编程的复杂度降低很多。

#### Q: 使用不同的线程模型，为什么会有较大的性能差别呢？ 
不同的模型，对于机器资源的消耗是不一样的，不同线程模型运行时消耗也是不一样的。
java 中的线程和OS的线程是 1:1 的。每个线程占用的内存，64位机器是1M，32位机器是320kb（？）。对于机器而言其内存是有限的。

1. 线程数量首先与近期可用内存资源
对于 Connection Per Thread ， 在高并发的情况下 线程的大小 * 链接数量，所以并发数显然会受限于机器可用内存大小，很难能扛到万级别的 qps （10000M = 10G，同时考虑到程序本身业务需要的内存，其实不是太现实）。
对于 Reactor 模式，线程的数量控制到和 core 的数量相关的。

2. 程序性能也受到 CS（Context Switch）和锁的使用的影响。
在 reactor 中，线程数量较少，整体的调度和切换开销小。同时因为使用了 reacotr，减少了锁的使用。


### 同步异步、阻塞非阻塞
同步异步说的是，做事情的方法：是自己来做，还是交给别人去干（其他的线程）。
阻塞非阻塞说的是，针对请求方而言的：阻塞是请求的时候要等到结果，非阻塞是说在请求的时候可以立即获得反馈。


#### Reference :
* reactor: [传说中神一样的Reactor反应器模式](https://www.cnblogs.com/crazymakercircle/p/9833847.html)
* Netty: [Netty Bootstrap（图解）|秒懂](http://www.cnblogs.com/crazymakercircle/p/9998643.html)
* epoll Bug: 
   * [JDK epoll bug](https://www.jianshu.com/p/3ec120ca46b2)
   * [空轮询bug](http://www.cnblogs.com/JAYIT/p/8241634.html)
* [Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)
* [怎样理解阻塞非阻塞与同步异步的区别](https://www.zhihu.com/question/19732473)


## TODO :
* Proactor / AIO