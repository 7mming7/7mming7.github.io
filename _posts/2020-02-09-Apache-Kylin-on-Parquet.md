# Apache Kylin On Parquet 构建原理
### 目录
- TOC
{:toc}
## 前言
本文主要介绍Kylin on parquet全新的构建引擎如何工作的。构建cube从N多步骤缩减为两步，主要分为构建资源探测以及构建索引两个步骤。本文中Kylin on parquet构建引擎使用Build Engine V3指代。

## 资源探测|Resource Detect
对于一个构建任务如何合理的分配资源，既不能太小导致cube构建失败，又不能太大导致资源的浪费。为了更加合理的使用集群资源，以及增强构建cube的稳定性。下面以SSB的模型作为示例说明：
![](https://github.com/shuiqing301/shuiqing301.github.io/blob/master/img/posts/20200209/SSB Model.png?raw=true)
Build Engine V3在使用集群资源进行任务构建之前，会提交一个spark local的任务，估算下本次构建可能需要的计算量以及需要的资源，该步骤不会占用集群资源。资源探测步骤在web界面的表现形式如下图所示:
![](https://github.com/shuiqing301/shuiqing301.github.io/blob/master/img/posts/20200209/Resource Detect.png?raw=true){:width="600px"}
资源探测步骤中，会估算Spark plan中底层File scan对应的文件的大小。

**主要获取的信息：**
1. 记录Layout上的spark的执行计划关联的文件
   使用spark生成cube的平表的DataFrame，然后从该平表DataFrame的执行计划查找叶子节点上的**FileSourceScanExec**和**HiveTableScanExec**，将这两种Exec对应文件路径保存到构建任务的共享目录(/job_tmp/share)，例如下图中spark执行计划上的10个FileSourceScanExec的Path，会记录到segment下的resource_paths.json文件
![](https://github.com/shuiqing301/shuiqing301.github.io/blob/master/img/posts/20200209/SSB_leafs.png?raw=true)
![](https://github.com/shuiqing301/shuiqing301.github.io/blob/master/img/posts/20200209/Resource Detect Paths.png?raw=true)

2. 记录Layout的spark Rdd的分区数量，为了后续自动调参时设置**spark.executors.instances**
   把 spark 执行计划 toRDD 转成底层的 RDD 依赖链， 使用宽度遍历获取所有叶子RDD， 以及叶子RDD 的分区数目当做task 数目，并将task数目写入文件cubing_detect_items.json，传递到构建的job，spark 自动调参的时候获取 所有 segment 所有 layout 里面最大的 task 数目当做依据，配置里面加入一个因子kylin.engine.task-core-factor，用来设置 task 数目和 core 的比例，最后算出 需要启动的 instance 数目，和 content size 算出来的取最大值

## 构建索引|Load Data To Index
构建索引步骤会利用集群资源，构建相应的cube存成parquet文件
![](https://github.com/shuiqing301/shuiqing301.github.io/blob/master/img/posts/20200209/Load data in index1.png?raw=true){:width="600px"}

### 自动调整构建参数
Build Engine V3为了使构建cube的任务更加合理以及健壮，引入了构建任务自动调整参数机制，该机制的实现主要依赖cube构建的第一步资源探测的信息。
主要调整的参数如下：

**Driver端参数**
* spark.driver.memory

**Executor端参数**
* spark.executor.instances
* spark.executor.cores
* spark.executor.memory
* spark.executor.memoryOverhead
* spark.sql.shuffle.partitions

下图是Executor端进行的参数调整信息：
![](https://github.com/shuiqing301/shuiqing301.github.io/blob/master/img/posts/20200209/Auto set spark conf.png?raw=true)

### 构建Snapshot
*并发构建：*
    Build Engine V3并发构建snapshot，通过参数 **kylin.snapshot.parallel-build-enabled**(默认开启) 控制是否开启并发构建，以及参数 **kylin.snapshot.parallel-build-timeout-seconds**(默认1小时) 进行构建超时控制

### 创建平表DataFrame
将Cube上定义的fact表和lookup表构建成一个spark的DataFrame，作为后续将指标和维度聚合到parquet文件上的基础。
主要有以下步骤：

1. 构建全局字典
2. 添加表上的编码列
3. 将模型上的fact表和lookup表根据join key构建成 Spark 的 DataFrame
4. 持久化平表

从一条sql去分析构建是如何支持的
```
select
  d_year,
  s_nation,
  p_category,
  count (distinct lo_linenumber) dis_lo_linenumber,
  sum(lo_revenue) - sum(lo_supplycost) as profit
from P_lineorder
left join dates on lo_orderdate = d_datekey
left join customer on lo_custkey = c_custkey
left join supplier on lo_suppkey = s_suppkey
left join part on lo_partkey = p_partkey
where
  c_region = 'AMERICA'
  and s_region = 'AMERICA'
  and (
    d_year = 1997
    or d_year = 1998
  )
  and (
    p_mfgr = 'MFGR#1'
    or p_mfgr = 'MFGR#2'
  )
group by
  d_year,
  s_nation,
  p_category
order by
  d_year,
  s_nation,
  p_category;
```

分析sql构建SSB的模型：
![](https://github.com/shuiqing301/shuiqing301.github.io/blob/master/img/posts/20200209/SSB_MODEL.png?raw=true)

平表sql如下：


### 指标和维度聚合到Parquet

### Cuboid Spanning Tree 构建优化

### Layout 并发构建

### 构建异常后自动重试

## 运维

## 总结
### Build Engine V3优点
### Build Engine V3缺点