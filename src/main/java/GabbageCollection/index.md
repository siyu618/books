
# GC

* [图解 Java 垃圾回收机制，写得非常好！](https://mp.weixin.qq.com/s/jjGsr5tNHYXPnbxXIzMfWA)
* [CMS垃圾回收机制](https://www.cnblogs.com/Leo_wl/p/5393300.html)
* [JVM 与 Linux 的内存关系详解](https://www.tuicool.com/articles/y2eQ3a)
   * 由于 NIO 的 DirectByteBuffer 需要在 GC 的后期被回收，因此连续申请 DirectByteBuffer 的程序，通常需要调用 System.gc()，避免长时间不发生 FullGC 导致引用在 old 区的 DirectByteBuffer 内存泄漏 ？？？ 

### zero GC
* [一文读懂Java 11的ZGC为何如此高效](https://mp.weixin.qq.com/s/nAjPKSj6rqB_eaqWtoJsgw)
   * 着色指针 
   * 读屏障
* [Java程序员的荣光，听R大论JDK11的ZGC](https://mp.weixin.qq.com/s/8igjQv2HdXiA5vt_YncRTQ)
   * CMS 是全量的 GC
   * G1 是增量式的 GC
   
   