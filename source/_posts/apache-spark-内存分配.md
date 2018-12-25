---
title: 'apache spark: 内存分配'
date: 2018-12-24 15:04:23
tags: [spark]
categories: [spark]
toc: true
mathjax: true
description: 简要总结Spark 在Yarn集群环境下的内存管理和分配， 这里先在原理上做简要梳理， 进一步切入源码层面给出Spark 内存管理类图逻辑， 继而给给出一些常用的Spark内存方案。 
---

简要总结Spark 在Yarn集群环境下的内存管理和分配， 这里先在原理上做简要梳理， 进一步切入源码层面给出Spark 内存管理类图逻辑， 继而给给出一些常用的Spark内存方案。 

## Spark on Yarn内存划分

Spark的整体任务计算模型如图： 

![](http://spark.apache.org/docs/2.4.0/img/cluster-overview.png)

具体到Yarn deploy的情况， Spark 的Executor会被部署到Yarn的Worker Node上， 每个Executor对应一个JVM， 可以同时运行多个Task。 具体请参看引用[2](http://spark.apache.org/docs/2.4.0/cluster-overview.html)。 

### 堆内和堆外内存规划

这里我们更关心Spark on Yarn运行时的每个Work Node上的具体内存划分情况。 Executor的内存管理建立在JVM的内存管理上， Spark对JVM空间（Heap + offHeap）进行了更详细的分配，以充分利用内存。 同时， Spark引入了off-heap(TungSten)模式， 使之可以直接在工作节点的系统中开辟堆外空间，进一步优化了内存的使用（可以理解为是独立于JVM托管的Heap之外利用c-style的malloc从os分配到的memory。由于不再由JVM托管，通过高效的内存管理，可以避免JVM object overhead和Garbage collection的开销）。

运行时Executor中的Task同时可以使用JVM和off-heap两种模式的内存。 

- **JVM OnHeap内存:** 大小由”--executor-memory”(即 spark.executor.memory)参数指定。Executor中运行的并发任务共享JVM堆内内存。 
- **JVM OffHeap内存:** 大小由“spark.yarn.executor.memoryOverhead”参数指定，主要用于JVM自身，字符串, NIO Buffer等开销。
- **Off-heap模式:** 默认情况下Off-heap模式的内存并不启用，可以通过“spark.memory.offHeap.enabled”参数开启，并由spark.memory.offHeap.size指定堆外内存的大小（占用的空间划归JVM OffHeap内存）。

由于目前工作场景及水平所限， 这里主要总结JVM模式的Executor内存管理。 以下出现有Off-heap均为JVM中区别于Heap的内存。 

### Executor内存分配

#### 可用内存总量

Yarn集群管理模式中， Spark 以Executor Container的形式在Node Manager中运行， 其可使用的内存上限由“yarn.scheduler.maximum-allocation-mb” 指定， 以下简称：MonitorMemory。

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fyhx3e5lcoj20q50c7mzc.jpg)

- Heap: 由“spark.executor.memory” 指定, 以下称为ExecutorMemory
- off-heap: 由 “spark.yarn.executor.memoryOverhead” 指定， 以下称为MemoryOverhead

由此： 

```
ExecutorMemory + MemoryOverhead <= MonitorMemory
```

若提交时，指定的ExecutorMemory与MemoryOverhead之和大于MonitorMemory， 则会导致Executor申请失败； 若运行过程中， 实际使用内存超过上限阈值， Executor进程会被Yarn终止掉(kill)。

#### Heap 

Sparkd对Heap内存的管理是逻辑上的划分管理（限制各有逻辑区内存量及记录使用状态）， 对象实例真正占用内存的管理（申请和释放）都由JVM来完成。 

“spark.executor.memory”指定的内存为JVM最大分配的堆内存（“-xmx”），Spark为了更高效的使用这部分内存，对这部分内存进行了细分，下图（备注：此图源于互联网）对基于spark 2 （1.6）对堆内存分配比例进行了描述：

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fyhxnsif41j21hm0u0qha.jpg)

其中：

1. Reserved Memory保留内存，系统默认值为300，一般无需改动，不用关心此部分内存。 但如果Executor分配的内存小于 1.5 * 300 = 450M时，Executor将无法执行。
2. Storage Memory 存储内存， 用于存放广播数据及RDD缓存数据。由上图可知，Spark 2+中，初始状态下，Storage及Execution Memory均约占系统总内存的30%（1 * 0.6 * 0.5 = 0.3）。在UnifiedMemory管理中，这两部分内存可以相互借用，为了方便描述,我们使用storageRegionSize来表示“spark.storage.storageFraction”。当计算内存不足时，可以改造storageRegionSize中未使用部分，且StorageMemory需要存储内存时也不可被抢占； 若实际StorageMemory使用量超过storageRegionSize，那么当计算内存不足时，可以改造(StorageMemory – storageRegionSize)部分，而storageRegionSize部分不可被抢占。

#### Java Off-heap (Memory Overhead)

Executor 中，另一块内存为由“spark.yarn.executor.memoryOverhead”指定的Java Off-heap内存，此部分内存主要是创建Java Object时的额外开销，Native方法调用，线程栈， NIO Buffer等开销（Driect Buffer）。此部分为用户代码及Spark 不可操作的内存，不足时可通过调整参数解决, 无需过多关注。


### 任务内存管理

Executor中任务以线程的方式执行，各线程共享JVM的资源，任务之间的内存资源没有强隔离（任务没有专用的Heap区域）。因此，可能会出现这样的情况：先到达的任务可能占用较大的内存，而后到的任务因得不到足够的内存而挂起。
在Spark任务内存管理中，使用HashMap存储任务与其消耗内存的映射关系。每个任务可占用的内存大小为潜在可使用计算内存的1/2n – 1/n , 当剩余内存为小于1/2n时，任务将被挂起，直至有其他任务释放执行内存，而满足内存下限1/2n，任务被唤醒，其中n为当前Executor中活跃的任务数。
任务执行过程中，如果需要更多的内存，则会进行申请，如果，存在空闲内存，则自动扩容成功，否则，将抛出OutOffMemroyError。

## Spark Heap 内存管理类逻辑

spark-core里对Spark内存管理描述如下:

```scala
/**
 * This package implements Spark's memory management system. This system consists of two main
 * components, a JVM-wide memory manager and a per-task manager:
 *
 *  - [[org.apache.spark.memory.MemoryManager]] manages Spark's overall memory usage within a JVM.
 *    This component implements the policies for dividing the available memory across tasks and for
 *    allocating memory between storage (memory used caching and data transfer) and execution
 *    (memory used by computations, such as shuffles, joins, sorts, and aggregations).
 *  - [[org.apache.spark.memory.TaskMemoryManager]] manages the memory allocated by individual
 *    tasks. Tasks interact with TaskMemoryManager and never directly interact with the JVM-wide
 *    MemoryManager.
 *
 * Internally, each of these components have additional abstractions for memory bookkeeping:
 *
 *  - [[org.apache.spark.memory.MemoryConsumer]]s are clients of the TaskMemoryManager and
 *    correspond to individual operators and data structures within a task. The TaskMemoryManager
 *    receives memory allocation requests from MemoryConsumers and issues callbacks to consumers
 *    in order to trigger spilling when running low on memory.
 *  - [[org.apache.spark.memory.MemoryPool]]s are a bookkeeping abstraction used by the
 *    MemoryManager to track the division of memory between storage and execution.
 *
 * Diagrammatically:
 *
 * {{{
 *       +-------------+
 *       | MemConsumer |----+                                   +------------------------+
 *       +-------------+    |    +-------------------+          |     MemoryManager      |
 *                          +--->| TaskMemoryManager |----+     |                        |
 *       +-------------+    |    +-------------------+    |     |  +------------------+  |
 *       | MemConsumer |----+                             |     |  |  StorageMemPool  |  |
 *       +-------------+         +-------------------+    |     |  +------------------+  |
 *                               | TaskMemoryManager |----+     |                        |
 *                               +-------------------+    |     |  +------------------+  |
 *                                                        +---->|  |OnHeapExecMemPool |  |
 *                                        *               |     |  +------------------+  |
 *                                        *               |     |                        |
 *       +-------------+                  *               |     |  +------------------+  |
 *       | MemConsumer |----+                             |     |  |OffHeapExecMemPool|  |
 *       +-------------+    |    +-------------------+    |     |  +------------------+  |
 *                          +--->| TaskMemoryManager |----+     |                        |
 *                               +-------------------+          +------------------------+
 * }}}
 *
 *
 * There are two implementations of [[org.apache.spark.memory.MemoryManager]] which vary in how
 * they handle the sizing of their memory pools:
 *
 *  - [[org.apache.spark.memory.UnifiedMemoryManager]], the default in Spark 1.6+, enforces soft
 *    boundaries between storage and execution memory, allowing requests for memory in one region
 *    to be fulfilled by borrowing memory from the other.
 *  - [[org.apache.spark.memory.StaticMemoryManager]] enforces hard boundaries between storage
 *    and execution memory by statically partitioning Spark's memory and preventing storage and
 *    execution from borrowing memory from each other. This mode is retained only for legacy
 *    compatibility purposes.
 */
```

可以看出， Spark 通过MemoryManager（1.6之后版本默认使用UnifiedMemoryManager）， 对JVM底层进行封装， 具体的Task无法直接操作Worker Node
底层的JVM内存。 

### 类图

这里简单梳理了一下Spark运行时内存管理逻辑类图。 

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fyhx3lmp89j213n0ncdio.jpg)

具体的Spark Heap管理逻辑会在后面整体阅读完源码之后再梳理。 

## 内存调整方案

Executor中可同时运行的任务数由Executor分配的CPU的核数N 和每个任务需要的CPU核心数C决定。其中:

$$ N = spark.executor.cores $$
$$ C = spark.task.cpus $$

Executor的最大任务并行度可表示为 $TP = N / C$.  其中,C值与应用类型有关，大部分应用使用默认值1即可，因此，影响Executor中最大任务并行度的主要因素是$N$.
依据Task的内存使用特征，前文所述的Executor内存模型可以简单抽象为下图所示模型：

![](https://wx3.sinaimg.cn/mw690/7c35df9bly1fyhzw88dquj20wl0iqjsy.jpg)

其中，Executor 向yarn申请的总内存可表示为： $$ M = M_1 + M_2 $$

### 错误类型及调整方案

#### Executor OOM类错误 （错误代码137， 143等）

该类错误一般是由于Heap（M2）已达上限，Task需要更多的内存，而又得不到足够的内存而导致。因此，解决方案要从增加每个Task的内存使用量，满足任务需求 或 降低单个Task的内存消耗量，从而使现有内存可以满足任务运行需求两个角度出发。因此：

#### **增加单个Task的内存使用量**

- 增加最大Heap值， 即 上图中M2 的值，使每个Task可使用内存增加。
- 降低Executor的可用Core的数量 N , 使Executor中同时运行的任务数减少，在总资源不变的情况下，使每个Task获得的内存相对增加。

#### **降低单个Task的内存使用量**

降低单个Task的内存消耗量可从配制方式和调整应用逻辑两个层面进行优化： 

- 配置方式

减少每个Task处理的数据量，可降低Task的内存开销，在Spark中，每个partition对应一个处理任务Task, 因此，在数据总量一定的前提下，可以通过增加partition数量的方式来减少每个Task处理的数据量,从而降低Task的内存开销。针对不同的Spark应用类型，存在不同的partition调整参数如下：

$$ P = spark.default.parallism (非SQL应用) $$
$$ P = spark.sql.shuffle.partition (SQL 应用) $$

通过增加P的值，可在一定程度上使Task现有内存满足任务运行

> 当调整一个参数不能解决问题时，上述方案应进行协同调整, 若应用shuffle阶段 spill严重，则可以通过调整“spark.shuffle.spill.numElementsForceSpillThreshold”的值，来限制spill使用的内存大小 ，比如设置（2000000），该值太大不足以解决OOM问题，若太小，则spill会太频繁，影响集群性能，因此，要依据负载类型进行合理伸缩（此处，可设法引入动态伸缩机制，待后续处理）。

- 调整应用逻辑

Executor OOM 一般发生Shuffle阶段，该阶段需求计算内存较大，且应用逻辑对内存需求有较大影响，下面举例就行说明：

1. groupByKey转换为reduceByKey

一般情况下， groupByKey能实现的功能使用reduceByKey均可实现， reduceByKey存在Map端合并， 可以有效减少传输带宽占用及reduce端内存消耗。 

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fyis1u4shmj20o10dpq9g.jpg)

2. data skew预处理

Data Skew是指任务间处理的数据量存在较大差异。如左图所示，key 为010的数据较多，当发生shuffle时，010所在分区存在大量数据，不仅拖慢Job执行（Job的执行时间由最后完成的任务决定）。 而且导致010对应Task内存消耗过多，可能导致OOM. 而右图，经过预处理（加盐，此处仅为举例说明问题，解决方法不限于此）可以有效减少Data Skew导致 的问题。 

![](https://wx4.sinaimg.cn/mw690/7c35df9bly1fyis1ry5ssj21c10rp1bf.jpg)

更多数据倾斜问题解决方案可以参见: [美团技术团队:Spark性能优化指南——高级篇](https://zhuanlan.zhihu.com/p/22024169)

#### **Beyond…… memory, killed by yarn.**

出现该问题原因是由于实际使用内存上限超过申请的内存上限而被Yarn终止掉了, 首先说明Yarn中Container内存监控机制：

- Container进程的内存使用量：以Container进程为根的进程树中所有进程的内存使用总量。
- Container被杀死的判断依据：进程树总内存（物理内存或虚拟内存）使用量超过向Yarn申请的内存上限值，则认为该Container使用内存超量，可以被“杀死”。

因此，对该异常的分析要从是否存在子进程两个角度出发。

1. 不存在子进程

根据Container进程杀死的条件可知，在不存在子进程时，出现killed by yarn问题是于由Executor(JVM)进程自身内存超过向Yarn申请的内存总量$M$ 所致。由于未出现[Executor OOM](#executor-oom类错误-错误代码137-143等)所述的OOM异常，因此可判定其为 $M1$ (Overhead)不足, 依据Yarn内存使用情况有如下两种方案：

- 如果，M未达到Yarn单个Container允许的上限时，可仅增加M1 ，从而增加M；如果，M达到Yarn单个Container允许的上限时，增加 $M_1$， 降低 $M_2$.
- 操作方法：在提交脚本中添加 --conf spark.yarn.executor.memoryOverhead=3072(或更大的值，比如4096等)   --conf spark.executor.memory = 10g 或 更小的值，注意二者之各要小于Container监控内存量,否则伸请资源将被yarn拒绝。
减少可用的Core的数量 $N$, 使并行任务数减少，从而减少Overhead开销
操作方法：在提交脚本中添加   --executor-cores=3  <比原来小的值>  或 --conf spark.executor.cores=3  <比原来小的值>

2. 存在子进程

Spark 应用中Container以Executor（JVM进程）的形式存在，因此根进程为Executor对应的进程, 而Spark 应用向Yarn申请的总资源$M = M_1  + M_2$ , 都是以Executor（JVM）进程（非进程树）可用资源的名义申请的。申请的资源并非一次性全量分配给JVM使用，而是先为JVM分配初始值，随后内存不足时再按比率不断进行扩容，直致达到Container监控的最大内存使用量$M$。当Executor中启动了子进程（调用shell等）时，子进程占用的内存（记为 $S$） 就被加入Container进程树，此时就会影响Executor实际可使用内存资源（Executor进程实际可使用资源为：$M - S$），然而启动JVM时设置的可用最大资源为$M$， 且JVM进程并不会感知Container中留给自己的使用量已被子进程占用，因此，当JVM使用量达到 $M - S$，还会继续开劈内存空间，这就会导致Executor进程树使用的总内存量大于$M$ 而被Yarn 杀死。


## 参考引用

- [1] [github:spark-core](https://github.com/apache/spark/tree/master/core)
- [2] [Cluster Mode Overview](http://spark.apache.org/docs/2.4.0/cluster-overview.html)
- [3] [Tuning Spark](https://spark.apache.org/docs/latest/tuning.html)
- [4] [Spark on Yarn之Executor内存管理 by barrenlake陈诚](https://www.jianshu.com/p/10e91ace3378)
- [5] [Spark Architecture](https://0x0fff.com/spark-architecture/)
- [6] [Spark Memory Management](https://0x0fff.com/spark-memory-management/)
- [7] [Deep Dive: Memory Management in Apache Spark](https://www.slideshare.net/databricks/deep-dive-memory-management-in-apache-spark?from_action=save)
- [8] [美团技术团队:Spark性能优化指南——高级篇](https://zhuanlan.zhihu.com/p/22024169)

