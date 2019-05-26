Spark的任务调度就是如何组织任务去处理RDD中每个分区的数据，根据RDD的依赖关系构建DAG，基于DAG划分Stage，将每个Stage中的任务发到指定节点运行。基于Spark的任务调度原理，我们可以合理规划资源利用，做到尽可能用最少的资源高效地完成任务计算。

### 分布式运行框架
Spark可以部署在多种资源管理平台，例如Yarn、Mesos等，Spark本身也实现了一个简易的资源管理机制，称之为Standalone模式。由于工作中接触较多的是Saprk on Yarn，不做特别说明，以下所述均表示Spark on Yarn。Spark部署在Yarn上有两种运行模式，分别为client和cluster模式，它们的区别仅仅在于Spark Driver是运行在Client端还是ApplicationMater端。如下图所示为Spark部署在Yarn上，以cluster模式运行的分布式计算框架。
![](https://blog-10039692.file.myqcloud.com/1492761845751_7698_1492761845936.jpg)


