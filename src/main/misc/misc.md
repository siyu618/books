### HTTP VS. HTTPS
* [HTTP和HTTPS协议，看一篇就够了](https://blog.csdn.net/xiaoming100001/article/details/81109617)
* [HTTPS原理和CA证书申请（满满的干货）](https://blog.51cto.com/11883699/2160032)

### 探索推荐引擎内部的秘密
* [1. 推荐引擎初探](https://www.ibm.com/developerworks/cn/web/1103_zhaoct_recommstudy1/index.html)
* [2. 深入推荐引擎相关算法 - 协同过滤](https://www.ibm.com/developerworks/cn/web/1103_zhaoct_recommstudy2/index.html?ca=drs-)
* [3. 深入推荐引擎相关算法 - 聚类](https://www.ibm.com/developerworks/cn/web/1103_zhaoct_recommstudy3/index.html?ca=drs-)



### 协同过滤
* [[机器学习]推荐系统之协同过滤算法](https://www.cnblogs.com/baihuaxiu/p/6617389.html)
* [协同过滤算法(collaborative filtering)](https://blog.csdn.net/hlang8160/article/details/81433356)
* [协同过滤推荐算法总结](https://www.cnblogs.com/pinard/p/6349233.html)


### Serverless
* [如何理解 Serverless](https://infoq.cn/article/2017/10/how-to-understand-serverless)
   * serverless 的定义
      * 1. 早期用来描述那些大部分或者完全依赖第三方（云端）应用或服务来管理服务端逻辑和状态的应用。BaaS
      * 2. 应用的一部分逻辑仍然由开发者完成，但是不像传统架构那样运行在一个无状态的计算容器中，而是由事件驱动、短期执行（甚至只调用一次）、完全由第三方管理。 FaaS
   * serverless 解决的问题
      * 1. 降低硬件基础设施的部署和维护成本。
      * 2. 降低应用扩展（scaling）成本。


### Service Mesh
* [初识 Service Mesh](https://www.jianshu.com/p/e23e3e74538e)
* [Service Mesh深度解析](https://time.geekbang.org/article/2360)
   * 定义
      * 抽象：基础设施层
      * 功能：实现请求的可靠传递
      * 部署：轻量级网络代理
      * 关键：对应用程序透明
   * 演进
      * 1. 微服务之前：TCP/IP 出现之前，程序员需要考虑网络通讯细节问题；TCP/IP 出现之后将这些变成了 OS 网络层的一部分。
      * 2. 微服务时代：第一代微服务架构，开发人员需要考虑非功能性代码：服务注册、服务发现、负载均衡、熔断限流等。
         * 为了简化开发，开始使用类库，比如 Netflix OSS 套件。
         * 存在的痛点
            * 1. 内容多，门槛高
               * 业务永远只看结果
            * 2. 服务治理功能不全
            * 3. 跨语言问题
            * 4. 生计问题
         * 思考：
            * 1. 根源：最艰巨的挑战，与服务本身无关，而是服务之间的通信。
            * 2. 目标：所有的努力，都是为了保证将请求发送到正确的地方。
            * 3. 本质：无论过程如何，请求从不更改。
            * 4. 普适：适用于所有的微服务。
      * 3. 技术栈下移：使用代理解决问题，比如 Nignx、Haproxy、Apache 等反向代理
         * 功能过于简陋
      * 4. Sidecar 的出现
         * 为特定的基础设施而设计，无法通用。
      * 5. 通用型 Service Mesh 的出现
         * Linkerd、Envoy、nignmesh
      * 6. Istio
         * 集大成者，集中式控制面板。
* [什么是 Service Mesh？](https://time.geekbang.org/article/2355)
   * Service Mesh：是一个基础设施层，用于处理服务期间通信。
   * Service Mesh：回路断路器、负载均衡、延迟感知、最终一致性服务发现、重试和超时。


#### [Kubernetes](http://docs.kubernetes.org.cn/)
* K8s 特点
   * 可移植
   * 可扩展
   * 自动化
* Kubernetes 分层架构
   * 核心层：对外提供 API 构建高层的应用，对内提供插件式应用执行环境
   * 应用层：部署（无状态应用、有状态应用、批处理任务、集群应用等）和路由（服务发现、DNS 解析等）
   * 管理层：系统度量（如基础设施、容器和网络的度量），自动化（如自动扩展、都昂太 Provision 等）以及策略（RABC、Quota、PSP、NetworkPolic 等）
   * 接口层：kubectl 命令行工具、客户端 SDK 以及集群联邦
   * 生态系统：在接口层之上的庞大容器管理调度的生态系统，可以分为两个范畴   
      * 1. Kubernetes 外部：日志、监控、配置管理、CI、CD、Workflow、Faas、OTS 应用等
      * 2. Kubernetes 内部：CRI、CNI、CVI、镜像仓库等
* Kubernetes 核心的两个设计理念：
   * 容错性
   * 可扩展性

