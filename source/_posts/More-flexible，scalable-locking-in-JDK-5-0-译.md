---
title: 'More flexible，scalable locking in JDK 5.0 [译]'
date: 2017-03-27 00:00:00
tags: [Java, juc]
toc: true
category: Java
description: 多线程和并发并不是什么新的东西， Java语言的一个创新点之一是：它首次将跨平台的线程模型和正式的内存模型纳入到语言规范的主流编程语言。Java的核心类库中包括用于创建，启动以及处理线程的Thread类，同事语言提供了在线程之间并发通信以及兼容并发约束的结构——synchronized和volatile。虽然这简化了平台无关的并发类开发，但是并不意味着可以更简单的编写并发类。
---


多线程和并发并不是什么新的东西， Java语言的一个创新点之一是：它首次将跨平台的线程模型和正式的内存模型纳入到语言规范的主流编程语言。Java的核心类库中包括用于创建，启动以及处理线程的Thread类，同事语言提供了在线程之间并发通信以及兼容并发约束的结构synchronized和volatile。虽然这简化了平台无关的并发类开发，但是并不意味着可以更简单的编写并发类。
## synchronized简述（A quick review of synchronized)
将一段代码声明为同步代码段将会产生两个结果，通常被称为原子性（atomicity）和可见性（visibility）。
- 原子性的含义是在同一时刻对同一管程对象竞争的线程只有一个可以进入并执行对应的同步代码段，这就防止了在对共享状态进行更新的时候出现冲突的情况。
- 可见性是更难被感知的，可见性的实现与内存的缓存策略以及编译器优化存在很大的关系。通常，线程对于变量的缓存是没有任何限制的， 每个线程对于变量的更改也并不保证被其它线程立刻感知到（不论线程采用寄存器，处理器指定缓存，还是使用指令冲排序或者其他编译器优化策略），但是如果开发者使用了synchronization，如下面的代码段所示，Java 运行时将确保将确保在线程在同步代码段内对共享变量的更新将会立即被下一个进入同一管程对象的同步代码段的线程感知。对于volatile变量，也存在类似的规则。（关于Java内存的更多信息，请参阅相关主题）。

{% codeblock [java] [synchronized] %}
synchronized (lockObject) {
    // Update object state
}
{% endcodeblock %}

因此同步能够在不需要竞争条件或者破坏数据（提供同步边界的正确位置）的同时可靠的实现对共享变量的更新， 并且保证其他线程将会这些共享变量的最新状态。Java通过定义一个清晰的跨平台内存模型（在JDK 5.0中进行了修改以修复初始定义中的某些错误），可以通过遵循以下这条规则来创建一次写入，无处不在的并发类：
每当你将对一个接下来可能被其他线程读取的变量进行写入，或者读取一个可能被其他线程更改过的变量的话，你必须使用同步。
更好的一点是，在最近的JVM中同步的竞争成本（线程尝试获取一个其他线程持有的锁）是很适中的（在之前的JVM中这一点做的并不是很好，从而使得开发者认为不论竞争与否同步都会导致很大的性能开销）。

## 同步的改进（Improving on synchronized）
当然synchronized并不是完美的， JSR 166团队花费了大量时间开发了java.util.concurent.lock框架来改善synchronized的不足。synchronized存在着一些功能上的限制：
- 线程在试图获取锁的时候是没有办法被打断的（interrupt）
- 线程在试图获取锁的时候可能陷入无限等待的情况（饥饿，starving）
- 同步的使用还必须要保证线程对于锁的获取和释放在同一个栈（函数栈）空间中，这在大多数时候都是正确的（并且可以很好地处理线程执行过程中发生异常的情况），但是对于少数的非块结构的同步就会是一个很大的挑战。

### ReentrantLock类
j.u.c中的Lock类是一个定义lock的抽象类，允许开发者以Java类的方式实现锁，而不仅仅是作为语言的功能。这就为多版本的lock实现提供了空间，每种不同的实现可能有不同的调度算法，性能特点或者锁语义。ReentrantLock类实现了lock接口，在保证与synchronized有相同语意的同时还增加了锁轮询（lock polling）， 获取锁等待超时（time lock waits）以及获取锁可中断（interruptable lock waits）等多个额外的功能。另外，在锁竞争压力很大的情况下，ReentrantLock具有更好的性能（换一种说法，当大量线程同时竞争访问共享资源的时候，JVM将会花费更多的时间在执行线程而非调度线程）

对于一个锁的可重入（reentrant）的含义是什么？简单的说就是有一个所得获取计数器（acquisition count）与锁关联。如果一个已经持有特定锁资源的线程再次获取相同的锁，对应的锁获取计数器就会加1，同是在锁释放阶段需要释放与所获取相同的次数来实现真正的锁资源释放。这就与synchronized的语意很相似；如果一个线程再次进入他已经持有的管程对象限制的同步代码段， 那么这个线程将会被允许执行，线程对于管程对象（lock）的持有权在只有在线程退出其首次进入的同步代码段之后才会真正释放。
为了实现与synchronized相同的语意采用ReentrantLock实现同步的代码段必须在finnally字段进行对锁的释放操作。否则，如果线程在同步代码段内出现异常，那么对应的同步锁可能永远也不会被释放，这是很重要的一点。代码如下：

{% codeblock [java] [ReentrantLock] %}
Lock lock = new ReentrantLock(); 

lock.lock(); 
try{
    // Update object state
}
finally {
    lock.unlock(); 
}
{% endcodeblock %}

## 性能对比（Comparing the scalability of ReentrantLock and synchronization）
后面的就省略了，作者主要对比了一下ReentrantLock和synchronized在吞吐量上的性能。最后的结论是在unfair情况下，ReentrantLock的性能要好很多，而且竞争压力越大（并发度越高），ReentrantLock的优势约明显。

