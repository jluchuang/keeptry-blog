---
title: 浅谈Java GC —— 从Tracing GC 说起
date: 2018-01-04 20:59:10
tags: [Java, Garbage Collection]
toc: true
categories: [Java, Garbage Collection]
description: 看过《深入理解Java虚拟机：JVM 高级特性与最佳实践》（周志明）的同学，对于GC Roots的概念应该不陌生。JVM中对内存进行回收时，需要判断对象是否仍在使用中，可以通过GC Roots Tracing辨别。这篇文章主要是我对以下几个问题的理解和资料整理：1.Tracing GC的基本思路；2. 那些对象可以作为GC Roots？ 3. 具体到Java的分带GC中， GC Roots是如何选取的？这里很多地方都是引自知乎大神RednaxelaFX的一些回答。  
---

## 摘要
看过《深入理解Java虚拟机：JVM 高级特性与最佳实践》（周志明）的同学，对于GC Roots的概念应该不陌生。JVM中对内存进行回收时，需要判断对象是否仍在使用中，可以通过GC Roots Tracing辨别。这篇文章主要是我对以下几个问题的理解和资料整理：1.Tracing GC的基本思路；2. 那些对象可以作为GC Roots？ 3. 具体到Java的分带GC中， GC Roots是如何选取的？这里很多地方都是引自知乎大神RednaxelaFX的一些回答。 

## Tracing GC基本思路

Tracing GC的思路就是(引自：[RednaxelaFX](https://www.zhihu.com/people/rednaxelafx/activities))：给定一个集合的引用作为根出发，通过引用关系遍历对象图，能被遍历到的（可到达的）对象就被判定为存活，其余对象（也就是没有被遍历到的）就自然被判定为死亡。
我们都看过这张图：
![](https://wx3.sinaimg.cn/mw690/7c35df9bgy1fn5nodgc27j20pv0dqwev.jpg)
我最初误以为，“Tracing GC是要找到所有能被回收的空间”， 这和上面所说的Tracing GC的基本思路是有区别的，现在看来Tracing GC是通过判断哪些对象是存活的，也就是MARK，然后才会衍生出具体的GC过程中的SWEEP或COPY以及COMPACT等回收操作。 

**注意再注意**：*Tracing GC的本质是通过找出所有活对象来把其余空间认定为“无用”，而不是找出所有死掉的对象并回收它们占用的空间。*
GC roots这组引用是tracing GC的起点。要实现语义正确的tracing GC，就必须要能完整枚举出所有的GC roots，否则就可能会漏扫描应该存活的对象，导致GC错误回收了这些被漏扫的活对象。

## 哪些对象可以作为GC Roots
注意这一节标题中“可以作为”这几个字，也就是说实际的GC实现中，GC Roots是不固定的。 下面引用[RednaxelaFX](https://www.zhihu.com/people/rednaxelafx/activities)的回答。

所谓“GC roots”，或者说tracing GC的“根集合”，就是一组必须活跃的引用。例如说，这些引用可能包括：
- 所有Java线程当前活跃的栈帧里指向GC堆里的对象的引用；换句话说，当前所有正在被调用的方法的引用类型的参数/局部变量/临时值。
- VM的一些静态数据结构里指向GC堆里的对象的引用，例如说HotSpot VM里的Universe里有很多这样的引用。
- JNI handles，包括global handles和local handles（看情况）
- 所有当前被加载的Java类（看情况）Java类的引用类型静态变量（看情况）
- Java类的运行时常量池里的引用类型常量（String或Class类型）（看情况）
- String常量池（StringTable）里的引用

注意，是一组必须活跃的引用，不是对象。

## GC带来的问题与分代GC
**STOP THE WORLD**：几乎所有的JAVA应用程序都是多线程的，同时Garbage Collector 本身也是多线程执行的。 因此对于一次GC过程的讨论主要包括两组进程的运行：Those performing application logic, and those performing GC。当负责GC的线程进行tracing 标记（tracks object references）,或者移动标记对象等操作的时候，GC线程必须确保应用程序线程（Application threads）没有正在使用这些对象。当GC线程移动一个对象的时候，往往会导致对象在内存（堆）中的地址变化，这就会导致应用程序线程再也没有办法访问到这些对象；为了防止这种现象的发生，在GC 过程中， 应用程序必须暂停执行（The pauses when all application threads are stopped are called stop-the-world pauses.）。

对传统的、基本的GC实现来说，由于它们在GC的整个工作过程中都要“stop-the-world”，如果能想办法缩短GC一次工作的时间长度就是件重要的事情。如果说收集整个GC堆耗时太长，那不如只收集其中的一部分？于是就有好几种不同的划分（partition）GC堆的方式来实现部分收集，而分代式GC就是这其中的一个思路。这个思路所基于的基本假设大家都很熟悉了：weak generational hypothesis——大部分对象的生命期很短（die young），而没有die young的对象则很可能会存活很长时间（live long）。这是对过往的很多应用行为分析之后得出的一个假设。基于这个假设，如果让新创建的对象都在young gen里创建，然后频繁收集young gen，则大部分垃圾都能在young GC中被收集掉。
由于young gen的大小配置通常只占整个GC堆的较小部分，而且较高的对象死亡率（或者说较低的对象存活率）让它非常适合使用copying算法来收集，这样就不但能降低单次GC的时间长度，还可以提高GC的工作效率。

> All GC algorithms have stop-the-world pauses during collection of the young generation.

为了进一步解决stop-the-world时间太长的问题， 同时提高GC能够应付的应用内存分配速率（allocation rate），于是就演进出了相关的并发GC算法（如CMS， G1）。

并发GC根本上要跟应用玩追赶游戏：应用一边在分配，GC一边在收集，如果GC收集的速度能跟得上应用分配的速度，那就一切都很完美；一旦GC开始跟不上了，垃圾就会渐渐堆积起来，最终到可用空间彻底耗尽的时候，应用的分配请求就只能暂时等一等了，等GC追赶上来。所以，对于一个并发GC来说，能够尽快回收出越多空间，就能够应付越高的应用内存分配速率，从而更好地保持GC以完美的并发模式工作。

## Java分代GC中的GC Roots

分代式GC是一种部分收集（partial collection）的做法。在执行部分收集时，从GC堆的非收集部分指向收集部分的引用，也必须作为GC roots的一部分。
具体到分两代的分代式GC来说，如果第0代叫做young gen，第1代叫做old gen，那么如果有minor GC / young GC只收集young gen里的垃圾，则young gen属于“收集部分”，而old gen属于“非收集部分”，那么从old gen指向young gen的引用就必须作为minor GC / young GC的GC roots的一部分。 在不考虑具体的JVM实现方式的前提下，下图简单展示了分代GC中，一次young GC的GC Roots的枚举过程：
![](https://wx2.sinaimg.cn/mw690/7c35df9bly1fn5ouhfsztj20m80a8gm3.jpg)

继续具体到HotSpot VM里的分两代式GC来说，除了old gen到young gen的引用之外，有些带有弱引用语义的结构，例如说记录所有当前被加载的类的SystemDictionary、记录字符串常量引用的StringTable等，在young GC时必须要作为strong GC roots，而在收集整堆的full GC时则不会被看作strong GC roots。
换句话说，young GC比full GC的GC roots还要更大一些。如果不能理解这个道理，那整个讨论也就无从谈起了。

## 参考引用
1. 《深入理解Java虚拟机：JVM 高级特性与最佳实践》（周志明）
2. [知乎：Java GC 为什么要分代——RednaxelaFX的回答](https://www.zhihu.com/question/53613423/answer/135743258)
3. 《Java Performance: The Definitive Guide》(scott Oaks)