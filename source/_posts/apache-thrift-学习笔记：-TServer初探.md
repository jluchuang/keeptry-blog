---
title: apache thrift 学习笔记： TServer初探
date: 2018-12-13 14:33:16
tags: [并发, thrift]
categories: [并发]
toc: true
description: 这篇文章会结合Thrift源码（Apache thrift 0.11.0）对Thrift Server端 Java 源码进行简要梳理学习。 Thrift Java Server端主要基于JDK原生的NIO实现， NIO的底层主要基于不同的操作系统调用底层的Selector/epoll/kqueue等， 相关准备知识请参见之前的总结。 这篇文章重点引用了： https://www.cnblogs.com/zaizhoumo/p/8184923.html， 感谢原创作者的整理。
---

这篇文章会结合Thrift源码（Apache thrift 0.11.0）对Thrift Server端 Java 源码进行简要梳理学习。 Thrift Java Server端主要基于JDK原生的NIO实现， NIO的底层主要基于不同的操作系统调用底层的Selector/epoll/kqueue等， 相关准备知识请参见之前的总结。 

- [poll、epoll小结](http://www.keeptry.cn/2018/12/10/poll%E3%80%81epoll%E5%B0%8F%E7%BB%93/)
- [Java NIO: Selector 浅析](http://www.keeptry.cn/2018/12/11/Java-NIO-Selector-%E6%B5%85%E6%9E%90/)

这篇文章大量的参考引用了

- [RPC-Thrift（一）](https://www.cnblogs.com/zaizhoumo/p/8184923.html)

感谢作者的原创整理和分享。 

## Thrift TServer类结构

![](https://wx3.sinaimg.cn/mw690/7c35df9bly1fy54j041wzj20ld0en0uc.jpg)

Thrift 提供了多种TServer的实现， 不同的TServer使用了不同的模型， 适用的情况也有所不同。

- TSimpleServer: 阻塞IO单线程Server， 主要用于测试； 
- TThreadPoolServer: 阻塞IO多线程Server， 多线程使用Java并发包中的线程池ThreadPoolExecutor； 
- AbstractNonblockingServer: 抽象类，为非阻塞I/O Server类提供共同的方法和类； 
- TNonblockingServer: 多路复用I/O单线程Server，依赖于TFramedTransport；
- THsHaServer: 半同步/半异步Server，多线程处理业务逻辑调用，同样依赖于TFramedTransport；
- TThreadedSelectorServer：半同步/半异步Server，依赖于TFramedTransport；

实际应用场景中， 基于AbstractNonblockingServer的服务场景较多， 接下来我们重点梳理相关Tserver服务模型。 

## AbstractNonblockingServer

AbstractNonblockingServer类是非阻塞I/O TServer的父类，提供了公用的方法和类。先通过源码了解它的实现机制。 启动服务的大致流程为： 

1. startThreads() 
2. startListening() 
3. setServing(true) 
4. waitForShutdown()

具体内容依赖于AbstractNonblockingServer子类的具体实现。基于Java NIO（多路复用I/O模型）实现。

```java
/**
 * Provides common methods and classes used by nonblocking TServer
 * implementations.
 */
public abstract class AbstractNonblockingServer extends TServer {
  protected final Logger LOGGER = LoggerFactory.getLogger(getClass().getName());

  public static abstract class AbstractNonblockingServerArgs<T extends AbstractNonblockingServerArgs<T>> extends AbstractServerArgs<T> {
    public long maxReadBufferBytes = 256 * 1024 * 1024;

    public AbstractNonblockingServerArgs(TNonblockingServerTransport transport) {
      super(transport);
      transportFactory(new TFramedTransport.Factory());
    }
  }

  /**
   * The maximum amount of memory we will allocate to client IO buffers at a
   * time. Without this limit, the server will gladly allocate client buffers
   * right into an out of memory exception, rather than waiting.
   */
  final long MAX_READ_BUFFER_BYTES;

  /**
   * How many bytes are currently allocated to read buffers.
   */
  final AtomicLong readBufferBytesAllocated = new AtomicLong(0);

  public AbstractNonblockingServer(AbstractNonblockingServerArgs args) {
    super(args);
    MAX_READ_BUFFER_BYTES = args.maxReadBufferBytes;
  }

  /**
   * Begin accepting connections and processing invocations.
   * 服务启动的主流程
   */
  public void serve() {
    // start any IO threads
    // 开启IO接受线程， TNonblockingServer（THsHaServer）/TThreadedSelectorServer对应实现是不同的
    // 具体的IO处理过程都交给具体实现的selector线程
    if (!startThreads()) {
      return;
    }

    // start listening, or exit
    if (!startListening()) {
      return;
    }

    // 通过全局变量标记服务正在运行
    setServing(true);

    // this will block while we serve
    // 等待IO线程结束
    waitForShutdown();

    // 标记服务结束
    setServing(false);

    // do a little cleanup
    stopListening();
  }

  /**
   * Starts any threads required for serving.
   *
   * @return true if everything went ok, false if threads could not be started.
   */
  protected abstract boolean startThreads();

  /**
   * A method that will block until when threads handling the serving have been
   * shut down.
   */
  protected abstract void waitForShutdown();

  /**
   * Have the server transport start accepting connections.
   *
   * @return true if we started listening successfully, false if something went
   *         wrong.
   */
  protected boolean startListening() {
    try {
      serverTransport_.listen();
      return true;
    } catch (TTransportException ttx) {
      LOGGER.error("Failed to start listening on server socket!", ttx);
      return false;
    }
  }

  /**
   * Stop listening for connections.
   */
  protected void stopListening() {
    serverTransport_.close();
  }

  /**
   * Perform an invocation. This method could behave several different ways -
   * invoke immediately inline, queue for separate execution, etc.
   *
   * @return true if invocation was successfully requested, which is not a
   *         guarantee that invocation has completed. False if the request
   *         failed.
   */
  protected abstract boolean requestInvoke(FrameBuffer frameBuffer);

  ......
}
```

AbstractNonblockingServer定义了Server端服务的整体框架， 具体的每次服务请求的处理过程需要依赖两个核心类： AbstractSelectThread和FrameBuffer， 接下来我们梳理一下相关类的工作细节。

### AbstractSelectThread 

```java 
  /**
   * An abstract thread that handles selecting on a set of transports and
   * {@link FrameBuffer FrameBuffers} associated with selected keys
   * corresponding to requests.
   * 处理Server端IO事件的虚类， 对于每个IO链接会关联一个具体的FrameBuffer来处理IO数据请求
   * FrameBuffer： 主要对NIO ByteBuffer进行封装
   * SelectThread 的实现类主要对ServerSocketChannel进行封装
   */
  protected abstract class AbstractSelectThread extends Thread {
    protected Selector selector;

    // List of FrameBuffers that want to change their selection interests.
    protected final Set<FrameBuffer> selectInterestChanges = new HashSet<FrameBuffer>();

    public AbstractSelectThread() throws IOException {
      this.selector = SelectorProvider.provider().openSelector();
    }

    /**
     * If the selector is blocked, wake it up.
     */
    public void wakeupSelector() {
      selector.wakeup();
    }

    /**
     * Add FrameBuffer to the list of select interest changes and wake up the
     * selector if it's blocked. When the select() call exits, it'll give the
     * FrameBuffer a chance to change its interests.
     */
    public void requestSelectInterestChange(FrameBuffer frameBuffer) {
      synchronized (selectInterestChanges) {
        selectInterestChanges.add(frameBuffer);
      }
      // wakeup the selector, if it's currently blocked.
      selector.wakeup();
    }

    /**
     * Check to see if there are any FrameBuffers that have switched their
     * interest type from read to write or vice versa.
     */
    protected void processInterestChanges() {
      synchronized (selectInterestChanges) {
        for (FrameBuffer fb : selectInterestChanges) {
          fb.changeSelectInterests();
        }
        selectInterestChanges.clear();
      }
    }

    /**
     * Do the work required to read from a readable client. If the frame is
     * fully read, then invoke the method call.
     */
    protected void handleRead(SelectionKey key) {
      FrameBuffer buffer = (FrameBuffer) key.attachment();
      if (!buffer.read()) {
        cleanupSelectionKey(key);
        return;
      }

      // if the buffer's frame read is complete, invoke the method.
      if (buffer.isFrameFullyRead()) {
        if (!requestInvoke(buffer)) {
          cleanupSelectionKey(key);
        }
      }
    }

    /**
     * Let a writable client get written, if there's data to be written.
     */
    protected void handleWrite(SelectionKey key) {
      FrameBuffer buffer = (FrameBuffer) key.attachment();
      if (!buffer.write()) {
        cleanupSelectionKey(key);
      }
    }

    /**
     * Do connection-close cleanup on a given SelectionKey.
     */
    protected void cleanupSelectionKey(SelectionKey key) {
      // remove the records from the two maps
      FrameBuffer buffer = (FrameBuffer) key.attachment();
      if (buffer != null) {
        // close the buffer
        buffer.close();
      }
      // cancel the selection key
      key.cancel();
    }
  } // SelectThread
```

### FrameBuffer

## 参考引用

- [1] [RPC-Thrift（一）](https://www.cnblogs.com/zaizhoumo/p/8184923.html)
- [2] [Thrift源码系列----5.AbstractNonblockingServer源码](https://blog.csdn.net/chen7253886/article/details/53024848)
- [3] [Class AbstractNonblockingServer](https://people.apache.org/~thejas/thrift-0.9/javadoc/org/apache/thrift/server/AbstractNonblockingServer.html)
