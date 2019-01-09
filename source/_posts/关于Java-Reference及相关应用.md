---
title: 关于Java Reference及相关应用
date: 2018-05-19 10:33:39
tags: [Java]
toc: true
categories: [Java]
description: 关于 Java Reference及相关应用
---


## 基础知识

### Reference 

Java中一共有4种引用类型(其实还有一些其他的引用类型比如FinalReference)：强引用、软引用、弱引用、虚引用。其中强引用就是我们经常使用的


```java
/**
 * a是引用对象
 * new Object是引用a指向的堆对象
 */
Object a = new Object(); 
```

在展开讨论之前， 我们必须明确一点： ** 引用对象a的Reference类型决定了堆对象new Object()的GC时机。 而引用对象a自身的GC与其它引用对象是一致的（JVM会根据引用可达性来决定引用对象的GC时间）。 **

本段内容主要引自参考引用[5](https://github.com/farmerjohngit/myblog/issues/10)

Reference类与Java 底层的GC实现紧密配合， 先来看Reference.java中的几个字段

```java
public abstract class Reference<T> {
    //引用的对象
    private T referent;        
    //回收队列，由使用者在Reference的构造函数中指定
    volatile ReferenceQueue<? super T> queue;
    //当该引用被加入到queue中的时候，该字段被设置为queue中的下一个元素，以形成链表结构
    volatile Reference next;
    //在GC时，JVM底层会维护一个叫DiscoveredList的链表，存放的是Reference对象，discovered字段指向的就是链表中的下一个元素，由JVM设置
    transient private Reference<T> discovered;  
    //进行线程同步的锁对象
    static private class Lock { }
    private static Lock lock = new Lock();
    //等待加入queue的Reference对象，在GC时由JVM设置，会有一个java层的线程(ReferenceHandler)源源不断的从pending中提取元素加入到queue
    private static Reference<Object> pending = null;
}
```

一个Reference对象的声明周期： 

![](https://camo.githubusercontent.com/9cfa0589071350377877e482c0daadf4fd4744fb/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f31312f352f313636653163323462383532333337323f773d35333226683d36323526663d706e6726733d3331333138)


主要分为Native层和Java层两个部分。

Native层在GC时将需要被回收的Reference对象加入到DiscoveredList中（代码在referenceProcessor.cpp中process_discovered_references方法），然后将DiscoveredList的元素移动到PendingList中（代码在referenceProcessor.cpp中enqueue_discovered_ref_helper方法）,PendingList的队首就是Reference类中的pending对象。 具体代码就不分析了，有兴趣的同学可以看看这篇参考引用[6](http://www.importnew.com/21628.html)。

整个过程中， Reference对象的状态转换过程如下： 

![](https://wx3.sinaimg.cn/mw690/7c35df9bly1fyy2r1ecwkj20jl08pwh0.jpg)

看看Java层代码： 

```java
private static class ReferenceHandler extends Thread {
        ...
        public void run() {
            while (true) {
                tryHandlePending(true);
            }
        }
  } 
static boolean tryHandlePending(boolean waitForNotify) {
        Reference<Object> r;
        Cleaner c;
        try {
            synchronized (lock) {
                if (pending != null) {
                    r = pending;
                    //如果是Cleaner对象，则记录下来，下面做特殊处理
                    c = r instanceof Cleaner ? (Cleaner) r : null;
                    //指向PendingList的下一个对象
                    pending = r.discovered;
                    r.discovered = null;
                } else {
                   //如果pending为null就先等待，当有对象加入到PendingList中时，jvm会执行notify
                    if (waitForNotify) {
                        lock.wait();
                    }
                    // retry if waited
                    return waitForNotify;
                }
            }
        } 
        ...

        // 如果时CLeaner对象，则调用clean方法进行资源回收
        if (c != null) {
            c.clean();
            return true;
        }
        //将Reference加入到ReferenceQueue，开发者可以通过从ReferenceQueue中poll元素感知到对象被回收的事件。
        ReferenceQueue<? super Object> q = r.queue;
        if (q != ReferenceQueue.NULL) q.enqueue(r);
        return true;
 }
```

流程比较简单：就是源源不断的从PendingList中提取出元素，然后将其加入到ReferenceQueue中去，开发者可以通过从ReferenceQueue中poll元素感知到对象被回收的事件。

另外需要注意的是，对于Cleaner类型（继承自虚引用）的对象会有额外的处理：在其指向的对象被回收时，会调用clean方法，该方法主要是用来做对应的资源回收，在堆外内存DirectByteBuffer中就是用Cleaner进行堆外内存的回收，这也是虚引用在java中的典型应用。

重要方法： 

1. clear()

```java
    /**
     * Clears this reference object.  Invoking this method will not cause this
     * object to be enqueued.
     *
     * <p> This method is invoked only by Java code; when the garbage collector
     * clears references it does so directly, without invoking this method.
     */
    public void clear() {
        this.referent = null;
    }
```

调用此方法不会导致此对象入队。此方法仅由Java代码调用；当垃圾收集器清除引用时，它直接执行，而不调用此方法。

clear的语义就是将referent置null。

清除引用对象所引用的原对象，这样通过get()方法就不能再访问到原对象了( PhantomReference除外 )。从相应的设计思路来说，既然都进入到queue对象里面，就表示相应的对象需要被回收了，因为没有再访问原对象的必要。此方法不会由JVM调用，而JVM是直接通过字段操作清除相应的引用，其具体实现与当前方法相一致。


#### GC时机

1. SoftReference 

```java
public class SoftReference<T> extends Reference<T> {
    
    static private long clock;
    
    private long timestamp;
   
    public SoftReference(T referent) {
        super(referent);
        this.timestamp = clock;
    }
 
    public SoftReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
        this.timestamp = clock;
    }

    public T get() {
        T o = super.get();
        if (o != null && this.timestamp != clock)
            this.timestamp = clock;
        return o;
    }

}
```

软引用的实现很简单， 就多了两个字段， clock和timestamp。 clock是个静态变量， 每次GC回收都会将该字段设置成当前时间。 timestamp字段则会在每次调用get方法时将其赋值为clock(如果不相等且对象没有被回收)

那这两个字段的作用是什么呢？ 这和软引用在内存不够的时候才被回收， 又有什么关系呢？ 我们来看JVM的GC源码： 

```C++
size_t
ReferenceProcessor::process_discovered_reflist(
  DiscoveredList               refs_lists[],
  ReferencePolicy*             policy,
  bool                         clear_referent,
  BoolObjectClosure*           is_alive,
  OopClosure*                  keep_alive,
  VoidClosure*                 complete_gc,
  AbstractRefProcTaskExecutor* task_executor)
{
 ...
   //还记得上文提到过的DiscoveredList吗?refs_lists就是DiscoveredList。
   //对于DiscoveredList的处理分为几个阶段，SoftReference的处理就在第一阶段
 ...
      for (uint i = 0; i < _max_num_q; i++) {
        process_phase1(refs_lists[i], policy,
                       is_alive, keep_alive, complete_gc);
      }
 ...
}

//该阶段的主要目的就是当内存足够时，将对应的SoftReference从refs_list中移除。
void
ReferenceProcessor::process_phase1(DiscoveredList&    refs_list,
                                   ReferencePolicy*   policy,
                                   BoolObjectClosure* is_alive,
                                   OopClosure*        keep_alive,
                                   VoidClosure*       complete_gc) {
  
  DiscoveredListIterator iter(refs_list, keep_alive, is_alive);
  // Decide which softly reachable refs should be kept alive.
  while (iter.has_next()) {
    iter.load_ptrs(DEBUG_ONLY(!discovery_is_atomic() /* allow_null_referent */));
    //判断引用的对象是否存活
    bool referent_is_dead = (iter.referent() != NULL) && !iter.is_referent_alive();
    //如果引用的对象已经不存活了，则会去调用对应的ReferencePolicy判断该对象是不是要被回收
    if (referent_is_dead &&
        !policy->should_clear_reference(iter.obj(), _soft_ref_timestamp_clock)) {
      if (TraceReferenceGC) {
        gclog_or_tty->print_cr("Dropping reference (" INTPTR_FORMAT ": %s"  ") by policy",
                               (void *)iter.obj(), iter.obj()->klass()->internal_name());
      }
      // Remove Reference object from list
      iter.remove();
      // Make the Reference object active again
      iter.make_active();
      // keep the referent around
      iter.make_referent_alive();
      iter.move_to_next();
    } else {
      iter.next();
    }
  }
 ...
}
```

refs_lists中存放了本次GC发现的某种引用类型（虚引用、软引用、弱引用等）， 而process_discovered_reflist方法的作用就是将不需要回收的对象从ref_lists中移除掉， ref_lists最后剩下的元素全是需要被回收的元素， 最后将其第一个元素赋值给上文提到过的Reference.java#pending字段。 

ReferencePolicy一共有4种实现: NeverClearPolicy, AlwaysClearPolicy, LRUCurrentHeapPolicy, LRUMaxHeapPolicy。 其中NeverClearPolicy永远返回false，代表永远不回收SoftReference， 在JVM中该类没有被使用， AlwaysClearPolicy则永远返回true， 在referenceProcessor.hpp#setup方法中可以设置policy为AlwaysClearPolicy。 

LRUCurrentHeapPolicy和LRUMaxHeapPolicy的should_clear_reference方法则是完全相同：

```c++
void LRUCurrentHeapPolicy::setup() {
  _max_interval = (Universe::get_heap_free_at_last_gc() / M) * SoftRefLRUPolicyMSPerMB;
  assert(_max_interval >= 0,"Sanity check");
}

void LRUMaxHeapPolicy::setup() {
  size_t max_heap = MaxHeapSize;
  max_heap -= Universe::get_heap_used_at_last_gc();
  max_heap /= M;

  _max_interval = max_heap * SoftRefLRUPolicyMSPerMB;
  assert(_max_interval >= 0,"Sanity check");
}
```

其中SoftRefLRUPolicyMSPerMB默认为1000，前者的计算方法和上次GC后可用堆大小有关，后者计算方法和（堆大小-上次gc时堆使用大小）有关。

看到这里你就知道SoftReference到底什么时候被被回收了，它和使用的策略（默认应该是LRUCurrentHeapPolicy），堆可用大小，该SoftReference上一次调用get方法的时间都有关系。

2. WeakReference 

```java
public class WeakReference<T> extends Reference<T> {

    public WeakReference(T referent) {
        super(referent);
    }

    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }

}
```

可以看到WeakReference在Java层只是继承了Reference，没有做任何的改动。那referent字段是什么时候被置为null的呢？要搞清楚这个问题我们再看下上文提到过的process_discovered_reflist方法：

```c++
size_t
ReferenceProcessor::process_discovered_reflist(
  DiscoveredList               refs_lists[],
  ReferencePolicy*             policy,
  bool                         clear_referent,
  BoolObjectClosure*           is_alive,
  OopClosure*                  keep_alive,
  VoidClosure*                 complete_gc,
  AbstractRefProcTaskExecutor* task_executor)
{
 ...

  //Phase 1:将所有不存活但是还不能被回收的软引用从refs_lists中移除（只有refs_lists为软引用的时候，这里policy才不为null）
  if (policy != NULL) {
    if (mt_processing) {
      RefProcPhase1Task phase1(*this, refs_lists, policy, true /*marks_oops_alive*/);
      task_executor->execute(phase1);
    } else {
      for (uint i = 0; i < _max_num_q; i++) {
        process_phase1(refs_lists[i], policy,
                       is_alive, keep_alive, complete_gc);
      }
    }
  } else { // policy == NULL
    assert(refs_lists != _discoveredSoftRefs,
           "Policy must be specified for soft references.");
  }

  // Phase 2:
  // 移除所有指向对象还存活的引用
  if (mt_processing) {
    RefProcPhase2Task phase2(*this, refs_lists, !discovery_is_atomic() /*marks_oops_alive*/);
    task_executor->execute(phase2);
  } else {
    for (uint i = 0; i < _max_num_q; i++) {
      process_phase2(refs_lists[i], is_alive, keep_alive, complete_gc);
    }
  }

  // Phase 3:
  // 根据clear_referent的值决定是否将不存活对象回收
  if (mt_processing) {
    RefProcPhase3Task phase3(*this, refs_lists, clear_referent, true /*marks_oops_alive*/);
    task_executor->execute(phase3);
  } else {
    for (uint i = 0; i < _max_num_q; i++) {
      process_phase3(refs_lists[i], clear_referent,
                     is_alive, keep_alive, complete_gc);
    }
  }

  return total_list_count;
}

void
ReferenceProcessor::process_phase3(DiscoveredList&    refs_list,
                                   bool               clear_referent,
                                   BoolObjectClosure* is_alive,
                                   OopClosure*        keep_alive,
                                   VoidClosure*       complete_gc) {
  ResourceMark rm;
  DiscoveredListIterator iter(refs_list, keep_alive, is_alive);
  while (iter.has_next()) {
    iter.update_discovered();
    iter.load_ptrs(DEBUG_ONLY(false /* allow_null_referent */));
    if (clear_referent) {
      // NULL out referent pointer
      //将Reference的referent字段置为null，之后会被GC回收
      iter.clear_referent();
    } else {
      // keep the referent around
      //标记引用的对象为存活，该对象在这次GC将不会被回收
      iter.make_referent_alive();
    }
    ...
  }
    ...
}
```

不管是弱引用还是其他引用类型，将字段referent置null的操作都发生在process_phase3中，而具体行为是由clear_referent的值决定的。而clear_referent的值则和引用类型相关。

```c++
ReferenceProcessorStats ReferenceProcessor::process_discovered_references(
  BoolObjectClosure*           is_alive,
  OopClosure*                  keep_alive,
  VoidClosure*                 complete_gc,
  AbstractRefProcTaskExecutor* task_executor,
  GCTimer*                     gc_timer) {
  NOT_PRODUCT(verify_ok_to_handle_reflists());
    ...
  //process_discovered_reflist方法的第3个字段就是clear_referent
  // Soft references
  size_t soft_count = 0;
  {
    GCTraceTime tt("SoftReference", trace_time, false, gc_timer);
    soft_count =
      process_discovered_reflist(_discoveredSoftRefs, _current_soft_ref_policy, true,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  update_soft_ref_master_clock();

  // Weak references
  size_t weak_count = 0;
  {
    GCTraceTime tt("WeakReference", trace_time, false, gc_timer);
    weak_count =
      process_discovered_reflist(_discoveredWeakRefs, NULL, true,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  // Final references
  size_t final_count = 0;
  {
    GCTraceTime tt("FinalReference", trace_time, false, gc_timer);
    final_count =
      process_discovered_reflist(_discoveredFinalRefs, NULL, false,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  // Phantom references
  size_t phantom_count = 0;
  {
    GCTraceTime tt("PhantomReference", trace_time, false, gc_timer);
    phantom_count =
      process_discovered_reflist(_discoveredPhantomRefs, NULL, false,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }
    ...
}
```

可以看到，对于Soft references和Weak references clear_referent字段传入的都是true，这也符合我们的预期：对象不可达后，引用字段就会被置为null，然后对象就会被回收（对于软引用来说，如果内存足够的话，在Phase 1，相关的引用就会从refs_list中被移除，到Phase 3时refs_list为空集合）。

但对于Final references和 Phantom references，clear_referent字段传入的是false，也就意味着被这两种引用类型引用的对象，如果没有其他额外处理，只要Reference对象还存活，那引用的对象是不会被回收的。Final references和对象是否重写了finalize方法有关，不在本文分析范围之内，我们接下来看看Phantom references。

3. PhantomReference

```java
public class PhantomReference<T> extends Reference<T> {
 
    public T get() {
        return null;
    }
 
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }

}
```
可以看到虚引用的get方法永远返回null，我们看个demo。

```java
public static void demo() throws InterruptedException {
        Object obj = new Object();
        ReferenceQueue<Object> refQueue =new ReferenceQueue<>();
        PhantomReference<Object> phanRef =new PhantomReference<>(obj, refQueue);

        Object objg = phanRef.get();
        //这里拿到的是null
        System.out.println(objg);
        //让obj变成垃圾
        obj=null;
        System.gc();
        Thread.sleep(3000);
        //gc后会将phanRef加入到refQueue中
        Reference<? extends Object> phanRefP = refQueue.remove();
        //这里输出true
        System.out.println(phanRefP==phanRef);
    }
```

从以上代码中可以看到，虚引用能够在指向对象不可达时得到一个'通知'（其实所有继承References的类都有这个功能），需要注意的是GC完成后，phanRef.referent依然指向之前创建Object，也就是说Object对象一直没被回收！

而造成这一现象的原因在上一小节末尾已经说了：对于Final references和 Phantom references，clear_referent字段传入的时false，也就意味着被这两种引用类型引用的对象，如果没有其他额外处理，在GC中是不会被回收的。

对于虚引用来说，从refQueue.remove();得到引用对象后，可以调用clear方法强行解除引用和对象之间的关系，使得对象下次可以GC时可以被回收掉。

#### 小结

1.我们经常在网上看到软引用的介绍是：在内存不足的时候才会回收，那内存不足是怎么定义的？为什么才叫内存不足？

软引用会在内存不足时被回收，内存不足的定义和该引用对象get的时间以及当前堆可用内存大小都有关系，计算公式在上文中也已经给出。

2.网上对于虚引用的介绍是：形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。主要用来跟踪对象被垃圾回收器回收的活动。真的是这样吗？

严格的说，虚引用是会影响对象生命周期的，如果不做任何处理，只要虚引用不被回收，那其引用的对象永远不会被回收。所以一般来说，从ReferenceQueue中获得PhantomReference对象后，如果PhantomReference对象不会被回收的话（比如被其他GC ROOT可达的对象引用），需要调用clear方法解除PhantomReference和其引用对象的引用关系。 

### ReferenceQueue

引用队列，在检测到适当的可到达性更改后，垃圾回收器将已注册的引用对象添加到该队列中。

实现了一个队列的入队(enqueue)和出队(poll还有remove)操作，内部元素就是泛型的Reference，并且Queue的实现，是由Reference自身的链表结构( 单向循环链表 )所实现的。

ReferenceQueue名义上是一个队列，但实际内部并非有实际的存储结构，它的存储是依赖于内部节点之间的关系来表达。可以理解为queue是一个类似于链表的结构，这里的节点其实就是reference本身。可以理解为queue为一个链表的容器，其自己仅存储当前的head节点，而后面的节点由每个reference节点自己通过next来保持即可。

## 具体应用

### WeakHashMap 

由于JVM帮我们管理Java程序的内存，我们总是希望当一个对象不被使用时，它会被立即回收，即一个对象的逻辑生命周期要与它的实际生命周期相一致。但是有些时候，由于写程序人的疏忽，没有注意对象的生命周期，导致对象的实际生命周期要比我们期望它的生命周期要长。这种情况叫做 unintentional object retention. 下面我们来看看由实例变量HashMap 导致的内存泄露问题。WeakHashMap正是被设计用来解决unintentional object retention问题的。 

WeakHashMap的Key被设计为WeakReference， 并且内部维护了一个ReferenceQueue，来定期清理被GC回收的Key对应的value。 

Entry对应的源码如下： 

```java
/**
     * The entries in this hash table extend WeakReference, using its main ref
     * field as the key.
     */
    private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
        V value;
        final int hash;
        Entry<K,V> next;

        /**
         * Creates new entry.
         */
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            // key 被封装成WeakReference, 其引用队列对应外层的WeakHashMap维护的引用队列
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }

        @SuppressWarnings("unchecked")
        public K getKey() {
            return (K) WeakHashMap.unmaskNull(get());
        }

        public V getValue() {
            return value;
        }

        public V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            K k1 = getKey();
            Object k2 = e.getKey();
            if (k1 == k2 || (k1 != null && k1.equals(k2))) {
                V v1 = getValue();
                Object v2 = e.getValue();
                if (v1 == v2 || (v1 != null && v1.equals(v2)))
                    return true;
            }
            return false;
        }

        public int hashCode() {
            K k = getKey();
            V v = getValue();
            return Objects.hashCode(k) ^ Objects.hashCode(v);
        }

        public String toString() {
            return getKey() + "=" + getValue();
        }
    }
```

基于此， 我们以get方法为例来看WeakHashMap如何实现内存的合理使用的: 

```java
    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
     * key.equals(k))}, then this method returns {@code v}; otherwise
     * it returns {@code null}.  (There can be at most one such mapping.)
     *
     * <p>A return value of {@code null} does not <i>necessarily</i>
     * indicate that the map contains no mapping for the key; it's also
     * possible that the map explicitly maps the key to {@code null}.
     * The {@link #containsKey containsKey} operation may be used to
     * distinguish these two cases.
     *
     * @see #put(Object, Object)
     */
    public V get(Object key) {
        Object k = maskNull(key);
        int h = hash(k);
        // 注意这里的getTable方法， 每次get/put等操作都会调用getTable方法，
        // getTable方法内部会通过调用expungeStaleEntries来实现淘汰被GC回收的key对应的entry
        Entry<K,V>[] tab = getTable();
        int index = indexFor(h, tab.length);
        Entry<K,V> e = tab[index];
        while (e != null) {
            if (e.hash == h && eq(k, e.get()))
                return e.value;
            e = e.next;
        }
        return null;
    }

```
注意这里的getTable方法， 每次get/put等操作都会调用getTable方法，getTable方法内部会通过调用expungeStaleEntries来实现淘汰被GC回收的key对应的entry
我们接下来看expungeStaleEntries是如何进行帮助GC进行内存清理的： 

```java
/**
     * Expunges stale entries from the table.
     */
    private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        
                        // 邻接表中删除key对应的entry
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        // 将value的引用指向为null， 保证下一次GC时value对应的额堆对象被回收
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```

可以看出，WeakHashMap的这种特性比较适合实现类似本地、堆内缓存的存储机制——缓存的失效依赖于GC收集器的行为。 

### ThreadLocal

#### 使用场景

首先明确一下ThreadLocal的使用场景： 

> This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own, independently initialized copy of the variable. ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).
Each thread holds an implicit reference to its copy of a thread-local variable as long as the thread is alive and the ThreadLocal instance is accessible; after a thread goes away, all of its copies of thread-local instances are subject to garbage collection (unless other references to these copies exist).

大概翻译： 

> ThreadLocal 提供了线程本地的实例。它与普通变量的区别在于，每个使用该变量的线程都会初始化一个完全独立的实例副本。ThreadLocal 变量通常被private static修饰。当一个线程结束时，它所使用的所有 ThreadLocal 相对的实例副本都可被回收。

于是我们可以这样理解： **ThreadLocal适用于每个线程需要自己独立的实例且该实例需要在多个方法中被使用， 也即变量在线程间隔离活而在类间或者方法之间共享的场景。** 

#### 实现原理

简而言之， Java为了实现线程中变量在多方法中的共享， 每个线程都会维护一个ThreadLocalMap， 引用[Java进阶（七）正确理解Thread Local的原理与适用场景](http://www.jasongj.com/java/threadlocal/), 如下图所示 ： 

![](http://www.jasongj.com/img/java/threadlocal/ThreadMap.png)

进一步我们先来看Thread类的源码： 

```java
public
class Thread implements Runnable {
    ......

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
    .......
}
```

可以看到每个Thread实例中都会维护两个ThreadLocalMap成员变量， 我们这里主要考虑第一个threadLcoals。

进一步我们来看ThreadLocalMap源码： 

```java
    /**
     * ThreadLocalMap is a customized hash map suitable only for
     * maintaining thread local values. No operations are exported
     * outside of the ThreadLocal class. The class is package private to
     * allow declaration of fields in class Thread.  To help deal with
     * very large and long-lived usages, the hash table entries use
     * WeakReferences for keys. However, since reference queues are not
     * used, stale entries are guaranteed to be removed only when
     * the table starts running out of space.
     */
    static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        ......
    }
```

这里为了说明原理， 我们只截取代码片段（仔细阅读ThreadLocalMap的实现，你会发现它是闭链的Map， 也就是说多个Key碰撞时并没有使用邻接拉链）， 重点来看ThreadLocalMap中Entry的实现， Entry继承了WeakReference， 该类的实例维护某个 ThreadLocal 与具体实例的映射。与 HashMap 不同的是，ThreadLocalMap 的每个 Entry 都是一个对key 的弱引用，这一点从super(k)可看出。另外，每个 Entry 都包含了一个对value的强引用。

1. 线程对象消亡

因此我们可以这样理解类中ThreadLocal变量的引用关系： 

Thread -Strong-> ThreadLocalMap -Strong-> Entry -Weak-> ThreadLocal
                                                -Strong-> Value 

因此当Thread声明周期结束的时候， Entry也会跟着消亡， 从而 ThreadLcoal变量以及value对象都会消亡。 

2. 线程使用的类对象消亡

接下来我们从另一个角度来看， 如果声明ThreadLocal成员变量的类对象消亡了， 但是对应的Thread生命周期并没有结束， 也会存在内存泄漏的情况。 

对于已经不再被使用且已被回收的 ThreadLocal 对象，它在每个线程内对应的实例由于被线程的 ThreadLocalMap 的 Entry 强引用，无法被回收，可能会造成内存泄漏。

针对该问题，ThreadLocalMap 的 set 方法中，通过 replaceStaleEntry 方法将所有键为 null 的 Entry 的值设置为 null，从而使得该值可被回收。另外，会在 rehash 方法中通过 expungeStaleEntry 方法将键和值为 null 的 Entry 设置为 null 从而使得该 Entry 可被回收。通过这种方式，ThreadLocal 可防止内存泄漏。

```java
private void set(ThreadLocal<?> key, Object value) {
  Entry[] tab = table;
  int len = tab.length;
  int i = key.threadLocalHashCode & (len-1);

  for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();
    if (k == key) {
      e.value = value;
      return;
    }
    if (k == null) {
      replaceStaleEntry(key, value, i);
      return;
    }
  }
  tab[i] = new Entry(key, value);
  int sz = ++size;
  if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
}
```

#### 案例

对于 Java Web 应用而言，Session 保存了很多信息。很多时候需要通过 Session 获取信息，有些时候又需要修改 Session 的信息。一方面，需要保证每个线程有自己单独的 Session 实例。另一方面，由于很多地方都需要操作 Session，存在多方法共享 Session 的需求。如果不使用 ThreadLocal，可以在每个线程内构建一个 Session实例，并将该实例在多个方法间传递，如下所示。

```java
public class SessionHandler {

  @Data
  public static class Session {
    private String id;
    private String user;
    private String status;
  }

  public Session createSession() {
    return new Session();
  }

  public String getUser(Session session) {
    return session.getUser();
  }

  public String getStatus(Session session) {
    return session.getStatus();
  }

  public void setStatus(Session session, String status) {
    session.setStatus(status);
  }

  public static void main(String[] args) {
    new Thread(() -> {
      SessionHandler handler = new SessionHandler();
      Session session = handler.createSession();
      handler.getStatus(session);
      handler.getUser(session);
      handler.setStatus(session, "close");
      handler.getStatus(session);
    }).start();
  }
}
```

该方法是可以实现需求的。但是每个需要使用 Session 的地方，都需要显式传递 Session 对象，方法间耦合度较高。

这里使用 ThreadLocal 重新实现该功能如下所示。

```java
public class SessionHandler {

  public static ThreadLocal<Session> session = new ThreadLocal<Session>();

  @Data
  public static class Session {
    private String id;
    private String user;
    private String status;
  }

  public void createSession() {
    session.set(new Session());
  }

  public String getUser() {
    return session.get().getUser();
  }

  public String getStatus() {
    return session.get().getStatus();
  }

  public void setStatus(String status) {
    session.get().setStatus(status);
  }

  public static void main(String[] args) {
    new Thread(() -> {
      SessionHandler handler = new SessionHandler();
      handler.getStatus();
      handler.getUser();
      handler.setStatus("close");
      handler.getStatus();
    }).start();
  }
}
```

使用 ThreadLocal 改造后的代码，不再需要在各个方法间传递 Session 对象，并且也非常轻松的保证了每个线程拥有自己独立的实例。

如果单看其中某一点，替代方法很多。比如可通过在线程内创建局部变量可实现每个线程有自己的实例，使用静态变量可实现变量在方法间的共享。但如果要同时满足变量在线程间的隔离与方法间的共享，ThreadLocal再合适不过。

#### 小结

- ThreadLocalMap 的 Entry 对 ThreadLocal 的引用为弱引用，避免了 ThreadLocal 对象无法被回收的问题。 
- ThreadLocalMap 的 set 方法通过调用 replaceStaleEntry 方法回收键为 null 的 Entry 对象的值（即为具体实例）以及 Entry 对象本身从而防止内存泄漏。 
- ThreadLocal 适用于变量在线程间隔离且在方法间共享的场景 

### DirectByteBuffer

更多详细关于DirectByteBuffer的相关内容参见:[堆外内存之 DirectByteBuffer 详解](http://www.importnew.com/26334.html)

DirectByteBuffer是PhantomReference类的典型应用， DirectByteBuffer的一个简单Demo如下： 

```java
import sun.nio.ch.DirectBuffer;

import java.nio.ByteBuffer;
import java.util.concurrent.TimeUnit;

public class DirectBufferDemo {
    public static void main(String[] args) throws InterruptedException {

        while (true) {
            ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024 * 1024 * 128);

            TimeUnit.SECONDS.sleep(10);

            // 手动释放DirectBufer
            ((DirectBuffer) directBuffer).cleaner().clean();

            TimeUnit.SECONDS.sleep(10);

            System.out.println("OK");
        }
    }
}
```

我们来看DirectByteBuffer源码， 会发现一个关键属性Cleaner： 

```java

    private final Cleaner cleaner;

    public Cleaner cleaner() { return cleaner; }
```

进一步查看Cleaner源码， 

```java
public class Cleaner extends PhantomReference<Object> {
    private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue();
    private static Cleaner first = null;
    private Cleaner next = null;
    private Cleaner prev = null;
    private final Runnable thunk;

    ...
}
```

Cleaner继承了PhantomReference， 内部维护了一个双线链表， 当调用Cleaner.clean()方法后， cleaner会出发thunk属性的run方法调用， 接下来我们在回到DirectByteBuffer类: 

```java
 private static class Deallocator
        implements Runnable
    {

        private static Unsafe unsafe = Unsafe.getUnsafe();

        private long address;
        private long size;
        private int capacity;

        private Deallocator(long address, long size, int capacity) {
            assert (address != 0);
            this.address = address;
            this.size = size;
            this.capacity = capacity;
        }

        public void run() {
            if (address == 0) {
                // Paranoia
                return;
            }
            unsafe.freeMemory(address);
            address = 0;
            Bits.unreserveMemory(size, capacity);
        }

    }
```
可以看到， DirectByteBuffer所对应的thunk属性中的run方法实现了DirectByteBuffer的实际对外内存释放。 

#### 使用对外内存的原因：

因为full gc 意味着彻底回收，彻底回收时，垃圾收集器会对所有分配的堆内内存进行完整的扫描，这意味着一个重要的事实——这样一次垃圾收集对Java应用造成的影响，跟堆的大小是成正比的。过大的堆会影响Java应用的性能。如果使用堆外内存的话，堆外内存是直接受操作系统管理( 而不是虚拟机 )。这样做的结果就是能保持一个较小的堆内内存，以减少垃圾收集对应用的影响。

在某些场景下可以提升程序I/O操纵的性能。少去了将数据从堆内内存拷贝到堆外内存的步骤。 

#### 什么情况下使用堆外内存

堆外内存适用于生命周期中等或较长的对象。( 如果是生命周期较短的对象，在YGC的时候就被回收了，就不存在大内存且生命周期较长的对象在FGC对应用造成的性能影响 )。
直接的文件拷贝操作，或者I/O操作。直接使用堆外内存就能少去内存从用户内存拷贝到系统内存的操作，因为I/O操作是系统内核内存和设备间的通信，而不是通过程序直接和外设通信的。
同时，还可以使用 池+堆外内存 的组合方式，来对生命周期较短，但涉及到I/O操作的对象进行堆外内存的再使用。( Netty中就使用了该方式 )

#### 堆外内存 VS 内存池

内存池：主要用于两类对象：①生命周期较短，且结构简单的对象，在内存池中重复利用这些对象能增加CPU缓存的命中率，从而提高性能；②加载含有大量重复对象的大片数据，此时使用内存池能减少垃圾回收的时间。
堆外内存：它和内存池一样，也能缩短垃圾回收时间，但是它适用的对象和内存池完全相反。内存池往往适用于生命期较短的可变对象，而生命期中等或较长的对象，正是堆外内存要解决的。

#### 堆外内存的特点

- 对于大内存有良好的伸缩性
- 对垃圾回收停顿的改善可以明显感觉到
- 在进程间可以共享，减少虚拟机间的复制

#### 对外内存回收问题

- 堆外内存回收问题，以及堆外内存的泄漏问题。这个在上面的源码解析已经提到了
- 堆外内存的数据结构问题：堆外内存最大的问题就是你的数据结构变得不那么直观，如果数据结构比较复杂，就要对它进行串行化（serialization），而串行化本身也会影响性能。另一个问题是由于你可以使用更大的内存，你可能开始担心虚拟内存（即硬盘）的速度对你的影响了。

## 参考引用 

- [1] [Reference、ReferenceQueue 详解](http://www.importnew.com/26250.html)
- [2] [堆外内存之 DirectByteBuffer 详解](http://www.importnew.com/26334.html)
- [3] [Understanding Weak References](https://web.archive.org/web/20061130103858/http://weblogs.java.net/blog/enicholas/archive/2006/05/understanding_w.html)
- [4] [深入理解java中的Soft references && Weak references && Phantom reference](https://blog.csdn.net/xlinsist/article/details/57089288)
- [5] [Java引用类型原理剖析](https://github.com/farmerjohngit/myblog/issues/10)
- [6] [gc过程中reference对象的处理](http://www.importnew.com/21628.html)
- [7] [What's the difference between SoftReference and WeakReference in Java?](https://stackoverflow.com/questions/299659/whats-the-difference-between-softreference-and-weakreference-in-java)
- [8] [Java进阶（七）正确理解Thread Local的原理与适用场景](http://www.jasongj.com/java/threadlocal/)
