---
title: apache thrift 学习笔记： TNonblockingServer && THsHaServer
date: 2018-12-17 15:00:26
tags: [并发, thrift]
categories: [并发]
toc: true
description: 在上一篇对于Thrift Server端AbstractNonblockingServer类及其内部聚合的相关核心类的简要梳理之后， 这里对具体的TNonblockingServer和THsHaServer的实现模式进行梳理总结。 
---

在上一篇对于Thrift Server端AbstractNonblockingServer类及其内部聚合的相关核心类的简要梳理之后， 这里对具体的TNonblockingServer和THsHaServer的实现模式进行梳理总结。

## TNonblockingServer

TNonblockingServer是非阻塞AbstractNonblockingServer的一种实现， 采用单线程处理IO事件， 同时对于Ready的IO事件数据也在同一个线程中串行计算。 与阻塞TSimpleServer相比， TNonblockingServer基于Java NIO Selector机制实现了单个线程对于多个IO事件的管理（IO多路复用）， 可以把TNonblockingServer看成是一个经典的Java IO多路复用的实例。 

```java
/**
 * A nonblocking TServer implementation. This allows for fairness amongst all
 * connected clients in terms of invocations.
 *
 * This server is inherently single-threaded. If you want a limited thread pool
 * coupled with invocation-fairness, see THsHaServer.
 *
 * To use this server, you MUST use a TFramedTransport at the outermost
 * transport, otherwise this server will be unable to determine when a whole
 * method call has been read off the wire. Clients must also use TFramedTransport.
 * TNonblockingServer使用单线程管理多个IO事件并串行同步调用Processor类
 * 具体的IO事件处理主要委托给内部类SelectAcceptThread
 */
public class TNonblockingServer extends AbstractNonblockingServer {

  public static class Args extends AbstractNonblockingServerArgs<Args> {
    public Args(TNonblockingServerTransport transport) {
      super(transport);
    }
  }

  private SelectAcceptThread selectAcceptThread_;

  public TNonblockingServer(AbstractNonblockingServerArgs args) {
    super(args);
  }


  /**
   * Start the selector thread to deal with accepts and client messages.
   *
   * @return true if everything went ok, false if we couldn't start for some
   * reason.
   */
  @Override
  protected boolean startThreads() {
    // start the selector
    try {
      selectAcceptThread_ = new SelectAcceptThread((TNonblockingServerTransport)serverTransport_);
      selectAcceptThread_.start();
      return true;
    } catch (IOException e) {
      LOGGER.error("Failed to start selector thread!", e);
      return false;
    }
  }

  @Override
  protected void waitForShutdown() {
    joinSelector();
  }

  /**
   * Block until the selector thread exits.
   */
  protected void joinSelector() {
    // wait until the selector thread exits
    try {
      selectAcceptThread_.join();
    } catch (InterruptedException e) {
      // for now, just silently ignore. technically this means we'll have less of
      // a graceful shutdown as a result.
    }
  }

  /**
   * Stop serving and shut everything down.
   */
  @Override
  public void stop() {
    stopped_ = true;
    if (selectAcceptThread_ != null) {
      selectAcceptThread_.wakeupSelector();
    }
  }

  /**
   * Perform an invocation. This method could behave several different ways
   * - invoke immediately inline, queue for separate execution, etc.
   */
  @Override
  protected boolean requestInvoke(FrameBuffer frameBuffer) {
    frameBuffer.invoke();
    return true;
  }

  public boolean isStopped() {
    return selectAcceptThread_.isStopped();
  }
}

```

TNonblockingServer的主题逻辑很简单吗。需要注意的是， 对于Processor的调用，与SelectThread在同一个线程内。 

### SelectAcceptThread

核心内部类SelectAcceptThread继承自AbstractSelectThread， 是典型的Java IO多路复用实现。 


```java
/**
   * The thread that will be doing all the selecting, managing new connections
   * and those that still need to be read.
   */
  protected class SelectAcceptThread extends AbstractSelectThread {

    // The server transport on which new client transports will be accepted
    private final TNonblockingServerTransport serverTransport;

    /**
     * Set up the thread that will handle the non-blocking accepts, reads, and
     * writes.
     */
    public SelectAcceptThread(final TNonblockingServerTransport serverTransport)
    throws IOException {
      this.serverTransport = serverTransport;
      serverTransport.registerSelector(selector);
    }

    public boolean isStopped() {
      return stopped_;
    }

    /**
     * The work loop. Handles both selecting (all IO operations) and managing
     * the selection preferences of all existing connections.
     * 
     */
    public void run() {
      try {
        if (eventHandler_ != null) {
          eventHandler_.preServe();
        }

        // 主循环， 服务的主体逻辑
        while (!stopped_) {
          select();
          processInterestChanges();
        }
        for (SelectionKey selectionKey : selector.keys()) {
          cleanupSelectionKey(selectionKey);
        }
      } catch (Throwable t) {
        LOGGER.error("run() exiting due to uncaught error", t);
      } finally {
        try {
          selector.close();
        } catch (IOException e) {
          LOGGER.error("Got an IOException while closing selector!", e);
        }
        stopped_ = true;
      }
    }

    /**
     * Select and process IO events appropriately:
     * If there are connections to be accepted, accept them.
     * If there are existing connections with data waiting to be read, read it,
     * buffering until a whole frame has been read.
     * If there are any pending responses, buffer them until their target client
     * is available, and then send the data.
     * 一次Select调用的处理， 对于一次Select返回的就绪IO进行串行处理（IO多路复用）
     */
    private void select() {
      try {
        // wait for io events.
        selector.select();

        // process the io events we received
        Iterator<SelectionKey> selectedKeys = selector.selectedKeys().iterator();
        while (!stopped_ && selectedKeys.hasNext()) {
          SelectionKey key = selectedKeys.next();
          selectedKeys.remove();

          // skip if not valid
          if (!key.isValid()) {
            cleanupSelectionKey(key);
            continue;
          }

          // if the key is marked Accept, then it has to be the server
          // transport.
          if (key.isAcceptable()) {
            handleAccept();
          } else if (key.isReadable()) {
            // deal with reads
            handleRead(key);
          } else if (key.isWritable()) {
            // deal with writes
            handleWrite(key);
          } else {
            LOGGER.warn("Unexpected state in select! " + key.interestOps());
          }
        }
      } catch (IOException e) {
        LOGGER.warn("Got an IOException while selecting!", e);
      }
    }

    protected FrameBuffer createFrameBuffer(final TNonblockingTransport trans,
        final SelectionKey selectionKey,
        final AbstractSelectThread selectThread) {
        return processorFactory_.isAsyncProcessor() ?
                  new AsyncFrameBuffer(trans, selectionKey, selectThread) :
                  new FrameBuffer(trans, selectionKey, selectThread);
    }

    /**
     * Accept a new connection.
     */
    private void handleAccept() throws IOException {
      SelectionKey clientKey = null;
      TNonblockingTransport client = null;
      try {
        // accept the connection
        client = (TNonblockingTransport)serverTransport.accept();
        clientKey = client.registerSelector(selector, SelectionKey.OP_READ);

        // add this key to the map
          FrameBuffer frameBuffer = createFrameBuffer(client, clientKey, SelectAcceptThread.this);

          clientKey.attach(frameBuffer);
      } catch (TTransportException tte) {
        // something went wrong accepting.
        LOGGER.warn("Exception trying to accept!", tte);
        tte.printStackTrace();
        if (clientKey != null) cleanupSelectionKey(clientKey);
        if (client != null) client.close();
      }
    }
  } // SelectAcceptThread
```

### 执行流程

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fy9tm4jbbrj20gt0u2jsm.jpg)

## THsHaServer

THsHaServer继承自TNonblockingServer， 二者的根本区别在于， TNonblockingServer只是简单的IO多路复用， 但是THsHaServer在IO多路复用的基础上进行改进， 对于每个Ready的READ事件数据计算（主要是）， 将其处理逻辑委托给一个线程池。 因此可以理解为： 

- 同步的调用Select获取Ready的IO事件（由SelectAcceptThread来完成）； 
- 异步的对读取数据进行处理（调用线程池work线程来进行数据异步计算），增加了数据计算的并行处理能力。

具体代码： 

```java
/**
 * An extension of the TNonblockingServer to a Half-Sync/Half-Async server.
 * Like TNonblockingServer, it relies on the use of TFramedTransport.
 */
public class THsHaServer extends TNonblockingServer {

  public static class Args extends AbstractNonblockingServerArgs<Args> {
    public int minWorkerThreads = 5;
    public int maxWorkerThreads = Integer.MAX_VALUE;
    private int stopTimeoutVal = 60;
    private TimeUnit stopTimeoutUnit = TimeUnit.SECONDS;
    // 负责调用Processor进行数据处理的的Work线程池
    private ExecutorService executorService = null;

    public Args(TNonblockingServerTransport transport) {
      super(transport);
    }


    /**
     * Sets the min and max threads.
     *
     * @deprecated use {@link #minWorkerThreads(int)} and {@link #maxWorkerThreads(int)}  instead.
     */
    @Deprecated
    public Args workerThreads(int n) {
      minWorkerThreads = n;
      maxWorkerThreads = n;
      return this;
    }

    /**
     * @return what the min threads was set to.
     * @deprecated use {@link #getMinWorkerThreads()} and {@link #getMaxWorkerThreads()} instead.
     */
    @Deprecated
    public int getWorkerThreads() {
      return minWorkerThreads;
    }

    public Args minWorkerThreads(int n) {
      minWorkerThreads = n;
      return this;
    }

    public Args maxWorkerThreads(int n) {
      maxWorkerThreads = n;
      return this;
    }

    public int getMinWorkerThreads() {
      return minWorkerThreads;
    }

    public int getMaxWorkerThreads() {
      return maxWorkerThreads;
    }

    public int getStopTimeoutVal() {
      return stopTimeoutVal;
    }

    public Args stopTimeoutVal(int stopTimeoutVal) {
      this.stopTimeoutVal = stopTimeoutVal;
      return this;
    }

    public TimeUnit getStopTimeoutUnit() {
      return stopTimeoutUnit;
    }

    public Args stopTimeoutUnit(TimeUnit stopTimeoutUnit) {
      this.stopTimeoutUnit = stopTimeoutUnit;
      return this;
    }

    public ExecutorService getExecutorService() {
      return executorService;
    }

    public Args executorService(ExecutorService executorService) {
      this.executorService = executorService;
      return this;
    }
  }


  // This wraps all the functionality of queueing and thread pool management
  // for the passing of Invocations from the Selector to workers.
  private final ExecutorService invoker;

  private final Args args;

  /**
   * Create the server with the specified Args configuration
   */
  public THsHaServer(Args args) {
    super(args);

    // 默认worker线程池基于Java current包实现
    invoker = args.executorService == null ? createInvokerPool(args) : args.executorService;
    this.args = args;
  }

  /**
   * {@inheritDoc}
   */
  @Override
  protected void waitForShutdown() {
    joinSelector();
    gracefullyShutdownInvokerPool();
  }

  /**
   * Helper to create an invoker pool
   */
  protected static ExecutorService createInvokerPool(Args options) {
    int minWorkerThreads = options.minWorkerThreads;
    int maxWorkerThreads = options.maxWorkerThreads;
    int stopTimeoutVal = options.stopTimeoutVal;
    TimeUnit stopTimeoutUnit = options.stopTimeoutUnit;

    LinkedBlockingQueue<Runnable> queue = new LinkedBlockingQueue<Runnable>();
    ExecutorService invoker = new ThreadPoolExecutor(minWorkerThreads,
      maxWorkerThreads, stopTimeoutVal, stopTimeoutUnit, queue);

    return invoker;
  }

  protected ExecutorService getInvoker() {
    return invoker;
  }

  protected void gracefullyShutdownInvokerPool() {
    // try to gracefully shut down the executor service
    invoker.shutdown();

    // Loop until awaitTermination finally does return without a interrupted
    // exception. If we don't do this, then we'll shut down prematurely. We want
    // to let the executorService clear it's task queue, closing client sockets
    // appropriately.
    long timeoutMS = args.stopTimeoutUnit.toMillis(args.stopTimeoutVal);
    long now = System.currentTimeMillis();
    while (timeoutMS >= 0) {
      try {
        invoker.awaitTermination(timeoutMS, TimeUnit.MILLISECONDS);
        break;
      } catch (InterruptedException ix) {
        long newnow = System.currentTimeMillis();
        timeoutMS -= (newnow - now);
        now = newnow;
      }
    }
  }

  /**
   * We override the standard invoke method here to queue the invocation for
   * invoker service instead of immediately invoking. The thread pool takes care
   * of the rest.
   */
  @Override
  protected boolean requestInvoke(FrameBuffer frameBuffer) {
    try {
      Runnable invocation = getRunnable(frameBuffer);
      invoker.execute(invocation);
      return true;
    } catch (RejectedExecutionException rx) {
      LOGGER.warn("ExecutorService rejected execution!", rx);
      return false;
    }
  }

  protected Runnable getRunnable(FrameBuffer frameBuffer){
    return new Invocation(frameBuffer);
  }
}
```

### 执行流程

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fy9u4d7c0bj20lj0u20ue.jpg)

## 参考引用

- [1] [TNonblockingServer.java](https://github.com/apache/thrift/blob/master/lib/java/src/org/apache/thrift/server/TNonblockingServer.java)
- [2] [THsHaServer.java](https://github.com/apache/thrift/blob/master/lib/java/src/org/apache/thrift/server/THsHaServer.java)
- [3] [RPC-Thrift（一）](https://www.cnblogs.com/zaizhoumo/p/8184923.html)
