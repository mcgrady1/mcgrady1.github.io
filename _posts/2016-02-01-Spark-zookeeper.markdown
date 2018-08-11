---
layout:     post
title:      "Spark和Zookeeper学习"
subtitle:   "Project Summary"
date:       2015-03-01 12:00:00
author:     "Mcgrady"
header-img: "img/about-bg.jpg"
header-mask: 0.3
catalog:    true
tags:
    - MapReduce
    - Tolerance
    - Ram vs IO
---


> Spark，Zookeeper加HDFS是我在做一个分布式项目时使用的工具，结合这三者的优势可以实现性能高，稳定性强的分布式平台，这里对之前的知识做一下总结，主要做面使用，请勿转载。


# spark与hadoop对比

Spark和hadoop都可以进行MapReduce的运算，简单的说就是如果一个问题可以划分成若干单元，每个单元的计算互不相关，单元计算结果可以在可以承受的时间内合成为总结果的计算，则该问题就可以归类为一个MapReduce运算。再说直白一点：所有分治模型都可交由hadoop解决。可以说spark是功能更全面的hadoop，支持一些诸如filter、group之类的操作，但是原本思想仍是map reduce，差别不太大。基本功能相似，但为什么Spark相对hadoop会有如此大的效率提升？下面我们首先分析这一点。

### **MapReduce**

MapReduce，通过简单的Mapper和Reducer的抽象提供一个编程模型，可以在一个由几十台上百台的PC组成的不可靠集群上并发地，分布式地处理大量的数据集，而把并发、分布式(如机器间通信)和故障恢复等计算细节隐藏起来。而Mapper和Reducer的抽象，又是各种各样的复杂数据处理都可以分解为的基本元素。这样，复杂的数据处理可以分解为由多个Job(包含一个Mapper和一个Reducer)组成的有向无环图(DAG),然后每个Mapper和Reducer放到Hadoop集群上执行，就可以得出结果。。

Map/Reduce框架和[分布式文件系统](http://hadoop.apache.org/docs/r1.0.4/cn/hdfs_design.html)是运行在一组相同的节点上的，也就是说，计算节点和存储节点通常在一起。Map/Reduce框架由一个单独的**master JobTracker** 和每个集群节点一个**slave TaskTracker**共同组成。master负责调度构成一个作业的所有任务，这些任务分布在不同的slave上，master监控它们的执行，重新执行已经失败的任务。而slave仅负责执行由master指派的任务。

## hadoop

#### 数据处理流程

- 从HDFS文件系统读取数据集
- 将数据集拆分成小块并分配给所有可用节点
- 针对每个节点上的数据子集进行计算
- 中间结果暂时保存在内存中，达到阈值会写到磁盘上
- 重新分配中间态结果并按照键进行分组
- 通过对每个节点计算的结果进行汇总和组合对每个键的值进行“Reducing”
- 将计算而来的最终结果重新写入 HDFS

map产生的中间结果暂时保存在内存中，该缓冲区的默认大小是100MB，可以通过参数io.sort.mb来调整其大小。当缓冲区中的数据使用率达到一定阀值后，将环形缓冲区中的部分数据写到磁盘上，生成一个临时的Linux本地数据的spill文件；然后在缓冲区的使用率再次达到阀值后，再次生成一个spill文件。直到数据处理完毕，在磁盘上会生成很多的临时文件。

由于这种方法严重依赖持久存储，每个任务需要多次执行读取和写入操作，因此速度相对较慢。但另一方面由于磁盘空间通常是服务器上最丰富的资源，这意味着MapReduce可以处理非常海量的数据集。同时也意味着相比其他类似技术，Hadoop的MapReduce通常可以在廉价硬件上运行，因为该技术并不需要将一切都存储在内存中。MapReduce具备极高的缩放潜力，生产环境中曾经出现过包含数万个节点的应用。

#### Hadoop的局限性

- 表层上只提供了Map和Reduce两个操作，处理逻辑隐藏在代码中，整体逻辑不够清晰
- 相对于Storm等流式框架，时延比较高，只适用于批数据处理，难以处理实时数据

Apache Hadoop及其MapReduce处理引擎提供了一套久经考验的批处理模型，最适合处理对时间要求不高的非常大规模数据集。

## Spark

Apache Spark是一种包含流处理能力的下一代批处理框架。特色是提供了一个集群的分布式内存抽象RDD（Resilient Distributed DataSet），即弹性分布式数据集。Spark上有RDD的两种操作，actor和traformation。transformation包括map、flatMap等操作，actor是返回结果，如Reduce、count、collect等。

#### 批处理模式

与MapReduce不同，Spark的数据处理工作全部在内存中进行，只在一开始将数据读入内存，以及将最终结果持久存储时需要与存储层交互。所有中间态的处理结果均存储在内存中。

#### Spark流结构

Spark流结构如下图所示：

![img](http://upload-images.jianshu.io/upload_images/1860293-2cf4ba79c3d9d03f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

**对比 Hadoop MapReduce 和 Spark 的 Shuffle 过程**

如果熟悉 Hadoop MapReduce 中的 shuffle 过程，可能会按照 MapReduce 的思路去想象 Spark 的 shuffle 过程。然而，它们之间有一些区别和联系。从 high-level 的角度来看，两者并没有大的差别。 都是将 mapper（Spark 里是 ShuffleMapTask）的输出进行 partition，不同的 partition 送到不同的 reducer（Spark 里 reducer 可能是下一个 stage 里的 ShuffleMapTask，也可能是 ResultTask）。Reducer 以内存作缓冲区，边 shuffle 边 aggregate 数据，等到数据 aggregate 好以后进行 reduce() （Spark 里可能是后续的一系列操作）。

从 low-level 的角度来看，两者差别不小。 Hadoop MapReduce 是 sort-based，进入 combine() 和 reduce() 的 records 必须先 sort。这样的好处在于 combine/reduce() 可以处理大规模的数据，因为其输入数据可以通过外排得到（mapper 对每段数据先做排序，reducer 的 shuffle 对排好序的每段数据做归并）。目前的 Spark 默认选择的是 hash-based，通常使用 HashMap 来对 shuffle 来的数据进行 aggregate，不会对数据进行提前排序。如果用户需要经过排序的数据，那么需要自己调用类似 sortByKey() 的操作；如果你是Spark 1.1的用户，可以将spark.shuffle.manager设置为sort，则会对数据进行排序。在Spark 1.2中，sort将作为默认的Shuffle实现。

从实现角度来看，两者也有不少差别。 Hadoop MapReduce 将处理流程划分出明显的几个阶段：map(), spill, merge, shuffle, sort, reduce() 等。每个阶段各司其职，可以按照过程式的编程思想来逐一实现每个阶段的功能。在 Spark 中，没有这样功能明确的阶段，只有不同的 stage 和一系列的 transformation()，所以 spill, merge, aggregate 等操作需要蕴含在 transformation() 中。

如果我们将 map 端划分数据、持久化数据的过程称为 shuffle write，而将 reducer 读入数据、aggregate 数据的过程称为 shuffle read。那么在 Spark 中，问题就变为怎么在 job 的逻辑或者物理执行图中加入 shuffle write 和 shuffle read 的处理逻辑？以及两个处理逻辑应该怎么高效实现？

#### 优势和局限

使用Spark而非Hadoop MapReduce的主要原因是速度。在内存计算策略和先进的DAG调度等机制的帮助下，Spark可以用更快速度处理相同的数据集。

Spark的另一个重要优势在于多样性。该产品可作为独立集群部署，或与现有Hadoop集群集成。该产品可运行批处理和流处理，运行一个集群即可处理不同类型的任务。

为流处理系统采用批处理的方法，需要对进入系统的数据进行缓冲。缓冲机制使得该技术可以处理非常大量的传入数据，提高整体吞吐率，但等待缓冲区清空也会导致延迟增高。这意味着Spark Streaming可能不适合处理对延迟有较高要求的工作负载。

Spark支持故障恢复的方式也不同，通过数据的血缘关系，再执行一遍前面的处理，Checkpoint，将数据集存储到持久存储中。

#### Spark作业调度

对RDD的操作分为transformation和action两类，真正的作业提交运行发生在action之后，调用action之后会将对原始输入数据的所有transformation操作封装成作业并向集群提交运行。这个过程大致可以如下描述：

- 由DAGScheduler对RDD之间的依赖性进行分析，通过DAG来分析各个RDD之间的转换依赖关系
- 根据DAGScheduler分析得到的RDD依赖关系将Job划分成多个stage
- 每个stage会生成一个TaskSet并提交给TaskScheduler，调度权转交给TaskScheduler，由它来负责分发task到worker执行

#### RDD依赖关系

Spark中RDD的粗粒度操作，每一次transformation都会生成一个新的RDD，这样就会建立RDD之间的前后依赖关系，在Spark中，依赖关系被定义为两种类型，分别是窄依赖和宽依赖

- 窄依赖，父RDD的分区最多只会被子RDD的一个分区使用，
- 宽依赖，父RDD的一个分区会被子RDD的多个分区使用

[![rddependency](https://wongxingjun.github.io/figures/spark-stage-division/rddDependency.jpg)](https://wongxingjun.github.io/figures/spark-stage-division/rddDependency.jpg)
图中左边都是窄依赖关系，可以看出分区是1对1的。右边为宽依赖关系，有分区是1对多。

#### stage的划分

stage的划分是Spark作业调度的关键一步，它基于DAG确定依赖关系，借此来划分stage，将依赖链断开，每个stage内部可以并行运行，整个作业按照stage顺序依次执行，最终完成整个Job。实际应用提交的Job中RDD依赖关系是十分复杂的，依据这些依赖关系来划分stage自然是十分困难的，Spark此时就利用了前文提到的依赖关系，调度器从DAG图末端出发，逆向遍历整个依赖关系链，遇到ShuffleDependency（宽依赖关系的一种叫法）就断开，遇到NarrowDependency就将其加入到当前stage。stage中task数目由stage末端的RDD分区个数来决定，RDD转换是基于分区的一种粗粒度计算，一个stage执行的结果就是这几个分区构成的RDD。
[![stageDivide](https://wongxingjun.github.io/figures/spark-stage-division/stageDivide.jpg)](https://wongxingjun.github.io/figures/spark-stage-division/stageDivide.jpg)
图中可以看出，在宽依赖关系处就会断开依赖链，划分stage，这里的stage1不需要计算，只需要计算stage2和stage3，就可以完成整个Job。

#### Task

stage 下的一个任务执行单元，一般来说，一个 rdd 有多少个 partition，就会有多少个 task，因为每一个 task 只是处理一个 partition 上的数据.

#### 总结

Spark是多样化工作负载处理任务的最佳选择。Spark批处理能力以更高内存占用为代价提供了无与伦比的速度优势。对于重视吞吐率而非延迟的工作负载，则比较适合使用Spark Streaming作为流处理解决方案

### **HDFS**

Hadoop分布式文件系统（HDFS）是一个高度容错性的系统，适合部署在廉价的机器上。HDFS能提供高吞吐量的数据访问，非常适合大规模数据集上的应用。HDFS放宽了一部分POSIX约束，来实现流式读取文件系统数据的目的。HDFS在最开始是作为Apache Nutch搜索引擎项目的基础架构而开发的。硬件错误是常态而不是偶尔的异常。因此错误检测和快速、自动的恢复是HDFS最核心的架构目标。运行在HDFS上的应用和普通的应用不同，需要流式访问它们的数据集。HDFS更多的考虑到了数据批处理，而不是用户交互处理。比之数据访问的低延迟问题，更关键的在于数据访问的高吞吐量。HDFS被调节以支持大文件存储。HDFS应用需要一个“一次写入多次读取”的文件访问模型。这一假设简化了数据一致性问题，并且使高吞吐量的数据访问成为可能。移动计算比移动数据更划算，在数据达到海量级别的时候更是如此。

​HDFS采用master/slave架构。一个HDFS集群是由一个Namenode和多个Datanodes组成。Namenode是一个中心服务器，负责管理文件系统的名字空间(namespace)以及客户端对文件的访问。集群中的Datanode一般是一个节点一个，负责管理它所在节点上的存储。

![img](https://img-blog.csdn.net/20160730172644499?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

​HDFS暴露了文件系统的名字空间，用户能够以文件的形式在上面存储数据。从内部看，一个文件其实被分成一个或多个数据块，这些块存储在一组Datanode上。Namenode执行文件系统的名字空间操作，比如打开、关闭、重命名文件或目录。它也负责确定数据块到具体Datanode节点的映射。Datanode负责处理文件系统客户端的读写请求。在Namenode的统一调度下进行数据块的创建、删除和复制。

**文件系统的名字空间**：HDFS支持传统的层次型文件组织结构。用户或者应用程序可以创建目录，然后将文件保存在这些目录里。

![img](https://img-blog.csdn.net/20160730172657891?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

**副本存放** ：副本的存放是HDFS可靠性和性能的关键。HDFS采用一种称为机架感知(rack-aware)的策略来改进数据的可靠性、可用性和网络带宽的利用率。大多数情况下，副本系数是3，HDFS的存放策略是将一个副本存放在本地机架的节点上，一个副本放在同一机架的另一个节点上，最后一个副本放在不同机架的节点上。

**安全模式** ：Namenode启动后会进入一个称为安全模式的特殊状态。处于安全模式的Namenode是不会进行数据块的复制的。当Namenode检测确认某个数据块的副本数目达到最小副本数，那么该数据块就会被认为是副本安全的；在一定百分比的数据块被Namenode检测确认是安全之后（加上一个额外的30秒等待时间），Namenode将退出安全模式状态。接下来它会确定还有哪些数据块的副本没有达到指定数目，并将这些数据块复制到其他Datanode上。Namenode上保存着HDFS的名字空间。对于任何对文件系统元数据产生修改的操作，Namenode都会使用一种称为EditLog的事务日志记录下来。整个文件系统的名字空间，包括数据块到文件的映射、文件的属性等，都存储在一个称为FsImage的文件中，这个文件也是放在Namenode所在的本地文件系统上。

- fsimage：是NameNode启动时对整个文件系统的快照。
- edits：是在NameNode启动后，对文件系统的改动序列。

Namenode在内存中保存着整个文件系统的名字空间和文件数据块映射(Blockmap)的映像。Datanode将HDFS数据以文件的形式存储在本地的文件系统中，它并不知道有关HDFS文件的信息。HDFS通讯协议都是建立在TCP/IP协议之上。HDFS的主要目标就是即使在出错的情况下也要保证数据存储的可靠性。常见的三种出错情况是：Namenode出错, Datanode出错和网络割裂(network partitions)。

​HDFS的架构支持数据均衡策略。FsImage和Editlog是HDFS的核心数据结构。如果这些文件损坏了，整个HDFS实例都将失效。因而，Namenode可以配置成支持维护多个FsImage和Editlog的副本。Namenode是HDFS集群中的单点故障(single point offailure)所在。如果Namenode机器故障，是需要手工干预的。

​Secondary NameNode的整个目的在HDFS中提供一个Checkpoint Node，通过阅读官方文档可以清晰的知道，它只是NameNode的一个助手节点。首先，它定时到NameNode去获取edits，并更新到fsimage上，一旦它有新的fsimage文件，它将其拷贝回NameNode上，NameNode在下次重启时回使用这个新的fsimage文件，从而减少重启的时间。所以，Secondary NameNode所做的是在文件系统这设置一个Checkpoint来帮助NameNode更好的工作；它不是取代NameNode，也不是NameNode的备份。　

​HDFS中的文件总是按照64M被切分成不同的块，每个块尽可能地存储于不同的Datanode中。HDFS以文件和目录的形式组织用户数据。它提供了一个命令行的接口(DFSShell)让用户与HDFS中的数据进行交互。一个HDFS集群主要由一个NameNode和很多个Datanode组成：Namenode管理文件系统的元数据，而Datanode存储了实际的数据。

## Zookeeper

#### **Master节点失效**

Spark Master的容错分为两种情况：Standalone集群模式和单点模式。

Standalone集群模式下的Master容错是通过ZooKeeper来完成的，即有多个Master，一个角色是Active，其他的角色是Standby。当处于Active的Master异常时，需要重新选择新的Master，通过ZooKeeper的ElectLeader功能实现。关于ZooKeeper的实现，这里就不展开了，感兴趣的朋友可以参考Paxos。

要使用ZooKeeper模式，你需要在conf/spark-env.sh中为`SPARK_DAEMON_JAVA_OPTS`添加一些选项，详见下表。

| 系统属性                         | 说明                                       |
| ---------------------------- | ---------------------------------------- |
| `spark.deploy.recoveryMode`  | 默认值为`NONE`。设置为`ZOOKEEPER`后，可以在Active Master异常之后重新选择一个Active Master |
| `spark.deploy.zookeeper.url` | ZooKeeper集群地址（比如`192.168.1.100:2181,192.168.1.101:2181`） |
| `spark.deploy.zookeeper.dir` | 用于恢复的ZooKeeper目录，默认值为`/spark`            |

设置`SPARK_DAEMON_JAVA_OPTS`的实际例子如下：

```
SPARK_DAEMON_JAVA_OPTS="$SPARK_DAEMON_JAVA_OPTS
    -Dspark.deploy.recoveryMode =ZOOKEEPER"
```

应用程序启动运行时，指定多个Master地址，它们之间用逗号分开，如下所示：

```
MASTER=spark://192.168.100.101:7077,spark://192.168.100.102:7077 bin/spark-shell
```

在ZooKeeper模式下，恢复期间新任务无法提交，已经运行的任务不受影响。

此外，Spark Master还支持一种更简单的单点模式下的错误恢复，即当Master进程异常时，重启Master进程并从错误中恢复。具体方法是设置`spark.deploy.recoveryMode`属性的值为`FILESYSTEM`，并为`spark.deploy.recoveryDirectory`属性设置一个本地目录，用于存储必要的信息来进行错误恢复。

#### **Slave节点失效**

Slave节点运行着Worker、执行器和Driver程序，所以我们分三种情况讨论下3个角色分别退出的容错过程。

- Worker异常停止时，会先将自己启动的执行器停止，Driver需要有相应的程序来重启Worker进程。
- 执行器异常退出时，Driver没有在规定时间内收到执行器的`StatusUpdate`，于是Driver会将注册的执行器移除，Worker收到`LaunchExecutor`指令，再次启动执行器。
- Driver异常退出时，一般要使用检查点重启Driver，重新构造上下文并重启接收器。第一步，恢复检查点记录的元数据块。第二步，未完成作业的重新形成。由于失败而没有处理完成的RDD，将使用恢复的元数据重新生成RDD，然后运行后续的Job重新计算后恢复。

## 参考资料
hadoop资料 https://winway.github.io/2017/01/11/big-data-hdfs/
hadoop基本组件 https://blog.csdn.net/Zonzereal/article/details/78095110
spark stage、job介绍  https://www.jianshu.com/p/014cb82f462a

