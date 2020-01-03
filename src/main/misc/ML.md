# 机器学习算法

### 1. K-Means 聚类算法：无监督学习
* [K-means 聚类算法](https://www.jianshu.com/p/23a7166675a3)
   * K 的选择
      1. Elbow point 拐点法，枚举 K，找出拐点
      2. DBI（Davies-Bouldin Index）
* [kmeans算法理解及代码实现](https://www.cnblogs.com/lliuye/p/9144312.html)
   * 二分 K-Means 算法
* [K-Means 聚类算法](https://www.jianshu.com/p/4f032dccdcef)
   * 算法流程
      1. 确定一个 K， 希望得到的集合数
      2. 从数据集中随机选择 K 个数据作为质心
      3. 对数据集中的每个点，计算其与每个质点的距离，离那个近就划归为哪个集合
      4. 把所有数据归好之后，一共可得 K 个集合
      5. 计算集合的质心，如果新质心和原来的质心小于某一特定的阈值，则认为达到期望的结果，算法终止。
      6. 如果新质心和原质心距离变化较大，则需要迭代。
   * 优点
      1. 原理简单，容易实现，收敛速度快
      2. 当结果簇是密集的，而簇与簇之间却别明显的时候，效果较好
      3. 主要调参仅仅是簇的数目 K
   * 缺点
      1. K 需要预先给定，K 不好估计
      2. 对初始质心的选择敏感，对结果影响大
      3. 对噪音和异常点比较敏感。用来检测异常值。
      4. 可能局部最优，而不是全局最优。
   * 细节问题
      1. K 的选定，多尝试，取最小平方误差的 K
      2. 初始的 K 个质心如何选择。做执行，看哪个结果合理就用哪个
      3. 关于离群值？聚类是需要隔离离群值，离群值单独研究。 
      4. 单位要一致。
      5. 标准化
* [一步步教你轻松学K-means聚类算法](https://www.cnblogs.com/baiboy/p/pybnc6.html)
   * 概念
      * 簇：所有数据的点集合，簇中的对象是相似的。
      * 质心：粗总所有点的的中心（计算所有点的均值而来）
      * SSE：Sum Of Sqared Error（误差平方和），被用来评估模型的好坏，SSE 越小，表示越接近它们的质心，聚类效果越好。
   *  二分 K-Means 聚类算法
      * 伪代码
      
      ```
      首先将所有的店看成一个簇
         当簇树木小于 K 时
      对每一个簇
         计算总误差
         在给定的簇上面进行 KMeans 聚类（k=2）
         计算将给出一份为 2 之后的总误差
      选择使误差最小的那个簇进行划分操作
      ```

      
### 2. k-NearestNeighbor（kNN）分类算法
* [kNN算法：K最近邻(kNN，k-NearestNeighbor)分类算法
](https://www.cnblogs.com/jyroy/p/9427977.html)
   * 属于懒学习（lazy-learning）
* [一步步教你轻松学KNN模型算法](https://www.cnblogs.com/baiboy/p/pybnc2.html)

### 3. 决策树算法
* [一步步教你轻松学决策树算法](https://www.cnblogs.com/baiboy/p/pybnc3.html)
   * 工作原理
  
      ```
def createBranch():    
   检测数据集中所有数据的是不是只有一个分类标签
   if so，直接返回标签
   else
      寻找划分数据集的最好特征（划分之后信息熵最小，也就是信息增益最大）
      划分数据集
      创建分知节点
      for  每个划分的集合
         调用函数 createBranch(创建分支函数)，并增加返回结果到分支节点中
      return 分支节点
 
      ```       
 
### 4. 朴素贝叶斯
* [一步步教你轻松学朴素贝叶斯模型理论篇1](https://www.cnblogs.com/baiboy/p/pybnc4-1.html)
   * 有监督学习
   * 假定样本每个特征与其他特征都不相关。
   * 工作原理
    
      ```
       提取所有文章中的词条，并进行去重
       获取文档的所有类别
       计算每个类别中的文章数目
       foreach 训练文章：
          foreach 类别：
             如果词条出现在文档中 --> 增加该词条的计数值 （for 循环增加或者 矩阵相加）
             增加素有词条的计数值（比类别下词条总数）
       foreach 类别：
          foreach 词条：
             将该词条的树木除以总词条树木得到词条条件概率（P(词条|类别)）
       返回该文档属于每个类别的概率（P（类别|文档的所有词条））
                 
      ```
      
* [一步步教你轻松学朴素贝叶斯模型实现篇2 ](https://www.cnblogs.com/baiboy/p/pybnc4-2.html)
   * 词集模型（set-of-words model）：考虑文章中词出现与否
   * 词包模型（bag-of-words model） ：考虑文档中词频  
* [一步步教你轻松学朴素贝叶斯模型算法Sklearn深度篇3](https://www.cnblogs.com/baiboy/p/pybnc4-1.html)
      
### 5. 逻辑回归
* [一步步教你轻松学逻辑回归模型算法](https://www.cnblogs.com/baiboy/p/pybnc5.html)
   * 用于解决二分类（0 or 1）问题   

### 6. 关联规则 Aprior 算法 （数据挖掘算法）
* [一步步教你轻松学关联规则Apriori算法](https://www.cnblogs.com/baiboy/p/pybnc7.html)    
* [Apriori算法的原理和流程](https://blog.csdn.net/huihuisd/article/details/86489810)
   * 关联规则的表示：A => B [support = x%; confidence = y%]
   * 支持度：所有事务中，符合该关联的事务数占比
   * 置信度
   * 项集：项集就是项的集合  
   * 频繁项集：给支持度一个阈值
   * 置信度计算公式

### SVM 算法
* [一步步教你轻松学支持向量机SVM算法之理论篇1](https://www.cnblogs.com/baiboy/p/pybnc8.html)
   * SVM（Support Vector Machine）支持向量机
   * 是一种监督学习，属于分类器范畴
   * 支持向量机分类
   * SVM 学习方法的选择
      * 在训练数据现行可分时，通过硬间隔学习一个线性分类器，即现行可分支持向量机，又称为硬间隔支持向量机
      * 在训练数据近似线性可分时，通过软间隔最大化学习一个线性分类器，即线性可分支持向量机，又称为软间隔支持向量机
      * 当训练数据线性不可分时， 通过核函数及软间隔最大化学习一个非线性支持向量机。
* [一步步教你轻松学支持向量机SVM算法之案例篇2](https://www.cnblogs.com/baiboy/p/pybnc9.html)
   * 寻找最大间隔
   * SMO 高效优化算法
      * 序列最小优化（Sequential Minimal Optimization）算法
   * 核函数（kernel）      

### PCA 降维
* [一步步教你轻松学主成分分析PCA降维算法](https://www.cnblogs.com/baiboy/p/pybnc10.html)
   * 主成分分析（Principal Component Analysis，PCA）
      * 找出一个最重要的特征，然后进行分析
   * PCA 思想
      1. 去除平均值
      2. 计算协方差矩阵
      3. 计算协方差矩阵的特征值和特征向量
      4. 将特征值排序
      5. 保留前 N 个最大的特征值对应的特征向量
      6. 将数据转换大上面得到的 N 个特征向量构建的新空间中（实现了特征压缩）

### 奇异值分解SVD降维算法
* [一步步教你轻松学奇异值分解SVD降维算法](https://www.cnblogs.com/baiboy/p/pybnc11.html)
   *       


### Embedding
* [10分钟AI - 万物皆可Embedding（嵌入）](https://baijiahao.baidu.com/s?id=1623647807544402492)
* [基于Embedding的推荐系统召回策略](https://www.ctolib.com/amp/topics-138378.html)
* [推荐系统第3课：召回算法、推荐系统中的embedding及online matching serving](https://blog.csdn.net/ibelieve8013/article/details/89374942)
* [几种推荐搜索场景下的用户Embedding策略](https://blog.csdn.net/guoyuhaoaaa/article/details/84539971)
* 
