---
title: 浅谈Java GC —— Garbage-First(G1)
date: 2018-01-07 19:26:53
tags: [Java, Garbage Collection]
categories: [Java, Garbage Collection]
toc: true
description: G1垃圾回收器（Garbage-First Collector）最早在JDK 7中被引入， 在JDK 9中，G1已经被设置为默认的垃圾回收器。最近的一个商品信息存储服务，单机存储百万量级的商品信息，同时需要支持高并发的商品信息读/写，服务的GC策略也由原来的CMS切换到G1。这篇文章在上一篇“浅淡GC——从Tracing GC说起”的基础上，进一步总结一下自己对G1的理解。主要从以下几个方面来整理说明：1. G1 GC的设计目标和使用场景；2.G1 GC的几种基本执行场景； 3. 为满足需求G1 GC中使用的一些关键技术； 4. 一些调优方法。
---

## 背景
G1垃圾回收器（Garbage-First Collector）最早在JDK 7中被引入， 在JDK 9中，G1已经被设置为默认的垃圾回收器。最近的一个商品信息存储服务，单机存储百万量级的商品信息，同时需要支持高并发的商品信息读/写，服务的GC策略也由原来的CMS切换到G1。这篇文章在上一篇[浅淡GC——从Tracing GC说起](http://www.keeptry.cn/2018/01/04/%E6%B5%85%E8%B0%88Java-GC-%E2%80%94%E2%80%94-%E4%BB%8ETracing-GC-%E8%AF%B4%E8%B5%B7/)的基础上，进一步总结一下自己对G1的理解。主要从以下几个方面来整理说明：1. G1 GC的设计目标和使用场景；2.G1 GC的几种基本执行场景； 3. 为满足需求G1 GC中使用的一些关键技术； 4. 一些调优方法。

## G1 GC的设计目标和使用场景
参考[官网](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html), 对G1的描述：
> The Garbage-First (G1) collector is a server-style garbage collector, targeted for multi-processor machines with large memories. It meets garbage collection (GC) pause time goals with a high probability, while achieving high throughput.

总结有一下几点：
- 面向Server端的垃圾回收器，主要针对多在处理器机器以及需要占用大量内存的应用程序； 
- 与CMS垃圾回收相似，G1的垃圾回收线程可以和引用程序线程并行； 
- G1的垃圾回收过程是MARK-COMPACT的，相比于CMS，可以避免内存碎片产生； 
- 为应用程序提供更可控（可靠）的暂停时间（G1新增了STP时间的预测模型）； 
- 保证（不牺牲）应用程序的吞吐性能； 
- 不需要更多的JAVA堆空间（这里我理解是设计者期望：G1在的内存回收速度，会一直满足应用程序的内存allocate速度）

需要注意的点：
- G1并不是一款实时的垃圾回收器，G1引入的缓存暂停预测模型只能在大多数情况下保证满足设定的暂停时间。 
- G1的执行过程也包含concurrent（runs along with application threads, e.g., refinement, marking, cleanup）和parallel（multi-threaded, e.g., stop the world）的两种阶段，同时G1的Full GC也是单线程的，但是如果有很好的调优，可以避免Full GC。 

推荐的应用场景：
> The first focus of G1 is to provide a solution for users running applications that require large heaps with limited GC latency. This means heap sizes of around 6GB or larger, and stable and predictable pause time below 0.5 seconds.

我经历的项目场景： 海量数据缓存服务，单机内存（heap）old region占用量20G以上, 存储item信息频繁更新（包括增删改查）。

## G1 GC的几种基本执行场景
在说明G1的的基本场景之前， 应该清楚G1对Java Heap的划分： 
![](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/images/slide9.png)
从图中可以得出以下结论：G1是在不连续的heap分区上运行的， 相同属于相同分代（Eden， Survivor， Old）的分区也不必须连续；这样做的好处在于：
*虽然G1在对每个分区进行清理的时候仍然需要应用程序线程（application threads）暂停， 但是G1可以更关注与垃圾对象占用率最高的分区，从而在有限的可控时间内，为应用程序清理出更多可用的内存空间——这也是G1的“灵感来源”。*
G1对于大对象的分配做了优化， 即新增了Humongous regions，对于大对象的分配，直接进入老年代。 

相比于官网给出的G1执行过程，我觉得《Java Performance: The Defintive Guide》里总结的更好。所以把G1的执行过程按照以下四中执行过程说明： 
- A young collection
- 一个后台的，并发循环（A background, concurrent cycle）
- Mixed Collection（A mixed collection）
- 如果必要的话，Full GC（If necessary, a full GC）

### *A young GC*
Young GC的触发时机： eden regions被占满。
**整个Young GC的过程是Stop the word的。**
Young GC的结果： 所有的young regions都会被收集，young gc结束后，有一些region会直接升级为老年代， 同事至少有一个region被升级为Survivor。

### *A background concurrent cycle*
我理解这个过程的主要目的：就是标记出一需要在Mixed GC中被处理的Old Regions集合（Garbage 占用率最高的region）。 
最终的执行结果：标记处满足GC条件（Garbage 比率高的）Old Region set。 
这个过程是中可以多次穿插Young GC。 主要包括以下几个阶段：

| 操作                            | 描述   |
| -----------------------------  | -----  |
|初始标记（initial mark，STW）     |它标记了从GC Root开始直接可达的对象。|
|并发标记（Concurrent Marking）     | 这个阶段从GC Root开始对heap中的对象标记，标记线程与应用程序线程并行执行，并且收集各个Region的存活对象信息。|
|最终标记（Remark，STW）           |标记那些在并发标记阶段发生变化的对象，将被回收。|
|清除垃圾（Cleanup）               | 清除空Region（没有存活对象的），加入到free list。|

### *Mix Collections* 
之所以被称为Mixed GC因为这个过程中，包含了对Eden Regions和上面被标记出的Old Regions的回收操作。 在完成第2步的标记过程之后，通常后面会伴随着一系列的Mixed GC，每次Mixed GC回收之前标记处的待回收Old Regions的一个子集（为了满足GC pause time的可控），直到所有被标记的Old Region都被回收完毕。 然后G1将会开始下一周期的 back ground cycle。 

### *Full GC*
与CMS相同， G1的Full GC也是单线程的对整个Heap进行清理。 这个过程很慢，对于应用程序是不可接受的，应该尽量避免。 

## 一些关键技术

### STAB
全称是Snapshot-At-The-Beginning，由字面理解，是GC开始时活着的对象的一个快照。它是通过Root Tracing得到的，作用是维持并发GC的正确性。
那么它是怎么维持并发GC的正确性的呢？根据三色标记算法，我们知道对象存在三种状态：
- 白：对象没有被标记到，标记阶段结束后，会被当做垃圾回收掉。
- 灰：对象被标记了，但是它的field还没有被标记或标记完。
- 黑：对象被标记了，且它的所有field也被标记完了。

![](http://idiotsky.me/images/gc-1.gif)

由于并发阶段的存在，Mutator和Garbage Collector线程同时对对象进行修改，就会出现白对象漏标的情况，这种情况发生的前提是：
- Mutator赋予一个黑对象该白对象的引用。
- Mutator删除了所有从灰对象到该白对象的直接或者间接引用。

对于第一个条件，在并发标记阶段，如果该白对象是new出来的，并没有被灰对象持有，那么它会不会被漏标呢？Region中有两个top-at-mark-start（TAMS）指针，分别为prevTAMS和nextTAMS。在TAMS以上的对象是新分配的，这是一种隐式的标记。对于在GC时已经存在的白对象，如果它是活着的，它必然会被另一个对象引用，即条件二中的灰对象。如果灰对象到白对象的直接引用或者间接引用被替换了，或者删除了，白对象就会被漏标，从而导致被回收掉，这是非常严重的错误，所以SATB破坏了第二个条件。也就是说，一个对象的引用被替换时，可以通过write barrier 将旧引用记录下来。

### RSets && CSets
> **Remembered Sets or RSets**: track object references into a given region. There is one RSet per region in the heap. The RSet enables the parallel and independent collection of a region. The overall footprint impact of RSets is less than 5%.
**Collection Sets or CSets**: the set of regions that will be collected in a GC. All live data in a CSet is evacuated (copied/moved) during a GC. Sets of regions can be Eden, survivor, and/or old generation. CSets have a less than 1% impact on the size of the JVM.

Remembered Set，是辅助GC过程的一种结构，典型的空间换时间工具，和Card Table有些类似。还有一种数据结构也是辅助GC的：Collection Set（CSet），它记录了GC要收集的Region集合，集合里的Region可以是任意年代的。在GC的时候，对于old->young和old->old的跨代对象引用，只要扫描对应的CSet中的RSet即可。
逻辑上说每个Region都有一个RSet，RSet记录了其他Region中的对象引用本Region中对象的关系，属于points-into结构（谁引用了我的对象）。而Card Table则是一种points-out（我引用了谁的对象）的结构，每个Card 覆盖一定范围的Heap（一般为512Bytes）。G1的RSet是在Card Table的基础上实现的：每个Region会记录下别的Region有指向自己的指针，并标记这些指针分别在哪些Card的范围内。 这个RSet其实是一个Hash Table，Key是别的Region的起始地址，Value是一个集合，里面的元素是Card Table的Index。
![](https://tech.meituan.com/img/g1/remembered_sets.jpg)
上图中有三个Region，每个Region被分成了多个Card，在不同Region中的Card会相互引用，Region1中的Card中的对象引用了Region2中的Card中的对象，蓝色实线表示的就是points-out的关系，而在Region2的RSet中，记录了Region1的Card，即红色虚线表示的关系，这就是points-into。
而维系RSet中的引用关系靠post-write barrier和Concurrent refinement threads来维护，操作伪代码如下（出处）：
```
void oop_field_store(oop* field, oop new_value) {
  pre_write_barrier(field);             // pre-write barrier: for maintaining SATB invariant
  *field = new_value;                   // the actual store
  post_write_barrier(field, new_value); // post-write barrier: for tracking cross-region reference
}
```
post-write barrier记录了跨Region的引用更新，更新日志缓冲区则记录了那些包含更新引用的Cards。一旦缓冲区满了，Post-write barrier就停止服务了，会由Concurrent refinement threads处理这些缓冲区日志。
**RSet究竟是怎么辅助GC的呢？在做YGC的时候，只需要选定young generation region的RSet作为根集，这些RSet记录了old->young的跨代引用，避免了扫描整个old generation。 而mixed gc的时候，old generation中记录了old->old的RSet，young->old的引用由扫描全部young generation region得到，这样也不用扫描全部old generation region。所以RSet的引入大大减少了GC的工作量。**

### Pause Prediction Model 
Pause Prediction Model 即停顿预测模型。它在G1中的作用是：

> G1 uses a pause prediction model to meet a user-defined pause time target and selects the number of regions to collect based on the specified pause time target.

G1 GC是一个响应时间优先的GC算法，它与CMS最大的不同是，用户可以设定整个GC过程的期望停顿时间，参数-XX:MaxGCPauseMillis指定一个G1收集过程目标停顿时间，默认值200ms，不过它不是硬性条件，只是期望值。那么G1怎么满足用户的期望呢？就需要这个停顿预测模型了。G1根据这个模型统计计算出来的历史数据来预测本次收集需要选择的Region数量，从而尽量满足用户设定的目标停顿时间。
停顿预测模型是以衰减标准偏差为理论基础实现的：
``` Cpp
//  share/vm/gc_implementation/g1/g1CollectorPolicy.hpp
double get_new_prediction(TruncatedSeq* seq) {
    return MAX2(seq->davg() + sigma() * seq->dsd(),
                seq->davg() * confidence_factor(seq->num()));
}
```
在这个预测计算公式中：davg表示衰减均值，sigma()返回一个系数，表示信赖度，dsd表示衰减标准偏差，confidence_factor表示可信度相关系数。而方法的参数TruncateSeq，顾名思义，是一个截断的序列，它只跟踪了序列中的最新的n个元素。



## 一些调优方法

## 参考引用

- 《Java Performance: The Definitive Guide》(Scott Oaks)
- [Getting Started with the G1 Garbage Collector](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html)
- [Java Hotspot G1 GC的一些关键技术](https://tech.meituan.com/g1.html)
- [G1: One Garbage Collector To Rule Them All](https://www.infoq.com/articles/G1-One-Garbage-Collector-To-Rule-Them-All)
- [Case for Defaulting to G1 Garbage Collector in Java 9](https://www.infoq.com/articles/Make-G1-Default-Garbage-Collector-in-Java-9)
- [Tips for Tuning the Garbage First Garbage Collector](https://www.infoq.com/articles/tuning-tips-G1-GC)
- [一张图了解三色标记法](http://idiotsky.me/2017/08/16/gc-three-color/)
