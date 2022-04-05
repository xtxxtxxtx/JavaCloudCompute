
Storm、Spark Streaming和Flink是现在主流的分布式实时处理框架，Spark Streaming在之前的文章中已经有了介绍。现在本人正在学习研究Flink，因此对Flink做一些简单的介绍。
今天的这篇文章主要对Flink做一些简单的介绍。
## 1、Flink的技术特点
**流处理特性：**

 1. 高吞吐、低延迟、高性能的流处理
 2. 支持带有事件时间的窗口操作
 3. 支持有状态计算的Exactly-once语义
 4. 支持高度灵活的窗口操作，支持基于time、count、session以及data-driven的窗口操作
 5. 支持具有Backpressure功能的持续流模型
 6. 支持基于轻量级分布式快照实现的容错
 7. 一个运行时间支持Batch onStreaming处理和Streaming处理
 8. Flink在JVM内部实现了自己的内存管理
 9. 支持迭代计算
 10. 支持程序自动优化：避免特定情况下Shuffle、排序等昂贵操作，中间结果有必要进行缓存


**API支持** 
  1.  对Streaming数据类应用提供DataStreamAPI
  2. 对批处理类应用，提供DataSetAPI
## 2、Flink生态圈和基本架构
Flink生态圈
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191003174116865.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
Flink基本架构
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191003174200469.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
## 3、Flink基本组件
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191003175322147.png)
**主要进程：**
1、**JobManager**是Flink系统的协调者，负责接收Flink Job，调度组成Job的多个Task的执行。同时JobManager还负责收集Job的状态信息，并管理Flink集群中从节点TaskManager。
2、**TaskManager**也是一个Actor，实际负责执行计算的Worker，在其上执行Flink Job的一组Task。每个TaskManager负责管理其所在节点上的资源信息，如内存、网络在启动时候将资源的状态向JobManager汇报。
3、**Client**：当用户提交一个Flink程序时首先创建一个Client首先对提交的Flink程序进行预处理并提交到集群中因此Client需要从用户提交的Flink程序配置中获取JobManager地址并建立连接，将Job交给JobManager。Client会将用户提交的Flink程序组装一个JobGraph是以JobGraph形式提交的。一个JobGraph是一个Flink Dataflow是由多个JobVertex组成的DAG。其中一个JobGraph包含 了一个Flink程序的如下信息：JobID、Job名称、配置信息、一组JobVertex等。
## 4、Flink和其他实时计算引擎对比
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191003192008135.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
Flink和Storm压力对比
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191003192041496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
那么在开发中应该如何选择实时计算框架呢：
1、需要关注流数据是否需要进行状态管理
2、At-least-once或者Exectly-once消息投递模式是否有特殊要求
3、对于小型独立的项目，并且需要低延迟的场景，建议使用Storm
4、如果项目已经使用了Spark并且秒级别的实时处理可以满足需求建议使用Spark Streaming
5、要求消息投递语义为Exactly Once的场景，数据量较大要求高吞吐低延迟的场景，需要进行状态管理或窗口统计的场景，建议使用Flink
6、成本考虑
