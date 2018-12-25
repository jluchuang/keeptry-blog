---
title: apache thrift 学习笔记： ThreadedSelectorServer
date: 2018-12-17 16:59:38
tags: [并发, thrift]
categories: [并发]
toc: true
description: TThreadedSelectorServer是非阻塞服务AbstractNonblockingServer的另一种实现，也是TServer的最高级形式。虽然THsHaServer对业务逻辑调用采用了线程池的方式，但是所有的数据读取和写入操作还都在单线程中完成，当需要在Client和Server之间传输大量数据时，THsHaServer就会面临性能问题。TThreadedSelectorServer将数据读取和写入操作也进行了多线程化。
---

TThreadedSelectorServer是非阻塞服务AbstractNonblockingServer的另一种实现，也是TServer的最高级形式。虽然THsHaServer对业务逻辑调用采用了线程池的方式，但是所有的数据读取和写入操作还都在单线程中完成，当需要在Client和Server之间传输大量数据时，THsHaServer就会面临性能问题。TThreadedSelectorServer将数据读取和写入操作也进行了多线程化。

## 执行流程

THsHaServer实现了IO事件的处理与数据计算过程Processor调用之间的异步处理， 在此基础上， TThreadedSelectorServer进一步实现了IO事件Accept与READ/WTITE的分离。 通过进一步增加负责READ/WRITE的线程数来增加数据的IO吞吐。 具体的执行流程如下图： 

![](https://wx2.sinaimg.cn/mw690/7c35df9bly1fy9w95bg8lj20j60au0t9.jpg)

由上图可以看到：

1. 单个AcceptThread线程负责处理Client的新建连接；
2. 多个SelectorThread线程负责处理数据读取和写入操作；
3. 单个负载均衡器SelectorThreadLoadBalancer负责将AcceptThread线程建立的新连接分配给哪个SelectorThread线程处理；
4. ExecutorService线程池负责业务逻辑的异步调用。

## 参数

```java 
  public static class Args extends AbstractNonblockingServerArgs<Args> {

    /** 
     The number of threads for selecting on already-accepted connections 
     默认负责处理IO事件的线程数为2
    */
    public int selectorThreads = 2;
    /**
     * The size of the executor service (if none is specified) that will handle
     * invocations. This may be set to 0, in which case invocations will be
     * handled directly on the selector threads (as is in TNonblockingServer)
     * 调用Processor处理业务逻辑的线程数， 默认为5
     */
    private int workerThreads = 5;
    /** Time to wait for server to stop gracefully */
    private int stopTimeoutVal = 60;
    private TimeUnit stopTimeoutUnit = TimeUnit.SECONDS;
    /** The ExecutorService for handling dispatched requests */
    private ExecutorService executorService = null;
    /**
     * The size of the blocking queue per selector thread for passing accepted
     * connections to the selector thread
     */
    private int acceptQueueSizePerThread = 4;

    /**
     * Determines the strategy for handling new accepted connections.
     */
    public static enum AcceptPolicy {
      /**
       * Require accepted connection registration to be handled by the executor.
       * If the worker pool is saturated, further accepts will be closed
       * immediately. Slightly increases latency due to an extra scheduling.
       */
      FAIR_ACCEPT,
      /**
       * Handle the accepts as fast as possible, disregarding the status of the
       * executor service.
       */
      FAST_ACCEPT
    }

    private AcceptPolicy acceptPolicy = AcceptPolicy.FAST_ACCEPT;

    .......
  }
```

这里我理解的两种工作模式： 

- FAIR_ACCEPT， 是ACCEPT新的链接与已有的链接之间的关系是公平的， 即大家都要排队进行处理， 这会导致两个问题： 
 1. 如果worker线程池饱和了（线程池排队队列满了），新的链接请求会被立即close； 
 2. 频繁的线程切换， 会增加链接ACCEPT延时； 
- FAST_ACCEPT, ACCEPT请求会由于其他IO事件， 即使现在Worker的负载已经很忙了;

## 类主体

```java
public class TThreadedSelectorServer extends AbstractNonblockingServer {
  private static final Logger LOGGER = LoggerFactory.getLogger(TThreadedSelectorServer.class.getName());

  ... args

  // The thread handling all accepts
  private AcceptThread acceptThread;

  // Threads handling events on client transports
  private final Set<SelectorThread> selectorThreads = new HashSet<SelectorThread>();

  // This wraps all the functionality of queueing and thread pool management
  // for the passing of Invocations from the selector thread(s) to the workers
  // (if any).
  private final ExecutorService invoker;

  private final Args args;

  /**
   * Create the server with the specified Args configuration
   */
  public TThreadedSelectorServer(Args args) {
    super(args);
    args.validate();
    invoker = args.executorService == null ? createDefaultExecutor(args) : args.executorService;
    this.args = args;
  }

  /**
   * Start the accept and selector threads running to deal with clients.
   *
   * @return true if everything went ok, false if we couldn't start for some
   *         reason.
   */
  @Override
  protected boolean startThreads() {
    try {
      for (int i = 0; i < args.selectorThreads; ++i) {
        selectorThreads.add(new SelectorThread(args.acceptQueueSizePerThread));
      }
      acceptThread = new AcceptThread((TNonblockingServerTransport) serverTransport_,
        createSelectorThreadLoadBalancer(selectorThreads));
      for (SelectorThread thread : selectorThreads) {
        thread.start();
      }
      acceptThread.start();
      return true;
    } catch (IOException e) {
      LOGGER.error("Failed to start threads!", e);
      return false;
    }
  }

  /**
   * Joins the accept and selector threads and shuts down the executor service.
   */
  @Override
  protected void waitForShutdown() {
    try {
      joinThreads();
    } catch (InterruptedException e) {
      // Non-graceful shutdown occurred
      LOGGER.error("Interrupted while joining threads!", e);
    }
    gracefullyShutdownInvokerPool();
  }

  protected void joinThreads() throws InterruptedException {
    // wait until the io threads exit
    acceptThread.join();
    for (SelectorThread thread : selectorThreads) {
      thread.join();
    }
  }

  /**
   * Stop serving and shut everything down.
   */
  @Override
  public void stop() {
    stopped_ = true;

    // Stop queuing connect attempts asap
    stopListening();

    if (acceptThread != null) {
      acceptThread.wakeupSelector();
    }
    if (selectorThreads != null) {
      for (SelectorThread thread : selectorThreads) {
        if (thread != null)
          thread.wakeupSelector();
      }
    }
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
   * invoker service instead of immediately invoking. If there is no thread
   * pool, handle the invocation inline on this thread
   */
  @Override
  protected boolean requestInvoke(FrameBuffer frameBuffer) {
    Runnable invocation = getRunnable(frameBuffer);
    if (invoker != null) {
      try {
        invoker.execute(invocation);
        return true;
      } catch (RejectedExecutionException rx) {
        LOGGER.warn("ExecutorService rejected execution!", rx);
        return false;
      }
    } else {
      // Invoke on the caller's thread
      invocation.run();
      return true;
    }
  }

  protected Runnable getRunnable(FrameBuffer frameBuffer) {
    return new Invocation(frameBuffer);
  }

  /**
   * Helper to create the invoker if one is not specified
   */
  protected static ExecutorService createDefaultExecutor(Args options) {
    return (options.workerThreads > 0) ? Executors.newFixedThreadPool(options.workerThreads) : null;
  }

  private static BlockingQueue<TNonblockingTransport> createDefaultAcceptQueue(int queueSize) {
    if (queueSize == 0) {
      // Unbounded queue
      return new LinkedBlockingQueue<TNonblockingTransport>();
    }
    return new ArrayBlockingQueue<TNonblockingTransport>(queueSize);
  }

  ... SelectorThread and AcceptThread
  /**
   * Creates a SelectorThreadLoadBalancer to be used by the accept thread for
   * assigning newly accepted connections across the threads.
   */
  protected SelectorThreadLoadBalancer createSelectorThreadLoadBalancer(Collection<? extends SelectorThread> threads) {
    return new SelectorThreadLoadBalancer(threads);
  }

  /**
   * A round robin load balancer for choosing selector threads for new
   * connections.
   */
  protected static class SelectorThreadLoadBalancer {
    private final Collection<? extends SelectorThread> threads;
    private Iterator<? extends SelectorThread> nextThreadIterator;

    public <T extends SelectorThread> SelectorThreadLoadBalancer(Collection<T> threads) {
      if (threads.isEmpty()) {
        throw new IllegalArgumentException("At least one selector thread is required");
      }
      this.threads = Collections.unmodifiableList(new ArrayList<T>(threads));
      nextThreadIterator = this.threads.iterator();
    }

    public SelectorThread nextThread() {
      // Choose a selector thread (round robin)
      if (!nextThreadIterator.hasNext()) {
        nextThreadIterator = threads.iterator();
      }
      return nextThreadIterator.next();
    }
  }
}
```

可以看到， 类主体很简单， 与THsHaServer的requestInvoke实现机制相同， 只是通过增加了一个HashSet来维护SelectThread， 每个SelectThread来处理多个IO链接。 

## AcceptThread

```java
/**
   * The thread that selects on the server transport (listen socket) and accepts
   * new connections to hand off to the IO selector threads
   */
  protected class AcceptThread extends Thread {

    // The listen socket to accept on
    private final TNonblockingServerTransport serverTransport;
    private final Selector acceptSelector;

    private final SelectorThreadLoadBalancer threadChooser;

    /**
     * Set up the AcceptThead
     *
     * @throws IOException
     */
    public AcceptThread(TNonblockingServerTransport serverTransport,
        SelectorThreadLoadBalancer threadChooser) throws IOException {
      this.serverTransport = serverTransport;
      this.threadChooser = threadChooser;
      this.acceptSelector = SelectorProvider.provider().openSelector();
      this.serverTransport.registerSelector(acceptSelector);
    }

    /**
     * The work loop. Selects on the server transport and accepts. If there was
     * a server transport that had blocking accepts, and returned on blocking
     * client transports, that should be used instead
     */
    public void run() {
      try {
        if (eventHandler_ != null) {
          eventHandler_.preServe();
        }

        while (!stopped_) {
          select();
        }
      } catch (Throwable t) {
        LOGGER.error("run() on AcceptThread exiting due to uncaught error", t);
      } finally {
        try {
          acceptSelector.close();
        } catch (IOException e) {
          LOGGER.error("Got an IOException while closing accept selector!", e);
        }
        // This will wake up the selector threads
        TThreadedSelectorServer.this.stop();
      }
    }

    /**
     * If the selector is blocked, wake it up.
     */
    public void wakeupSelector() {
      acceptSelector.wakeup();
    }

    /**
     * Select and process IO events appropriately: If there are connections to
     * be accepted, accept them.
     */
    private void select() {
      try {
        // wait for connect events.
        acceptSelector.select();

        // process the io events we received
        Iterator<SelectionKey> selectedKeys = acceptSelector.selectedKeys().iterator();
        while (!stopped_ && selectedKeys.hasNext()) {
          SelectionKey key = selectedKeys.next();
          selectedKeys.remove();

          // skip if not valid
          if (!key.isValid()) {
            continue;
          }

          if (key.isAcceptable()) {
            handleAccept();
          } else {
            LOGGER.warn("Unexpected state in select! " + key.interestOps());
          }
        }
      } catch (IOException e) {
        LOGGER.warn("Got an IOException while selecting!", e);
      }
    }

    /**
     * Accept a new connection.
     */
    private void handleAccept() {
      final TNonblockingTransport client = doAccept();
      if (client != null) {
        // Pass this connection to a selector thread
        final SelectorThread targetThread = threadChooser.nextThread();

        if (args.acceptPolicy == Args.AcceptPolicy.FAST_ACCEPT || invoker == null) {
          doAddAccept(targetThread, client);
        } else {
          // FAIR_ACCEPT
          try {
            invoker.submit(new Runnable() {
              public void run() {
                doAddAccept(targetThread, client);
              }
            });
          } catch (RejectedExecutionException rx) {
            LOGGER.warn("ExecutorService rejected accept registration!", rx);
            // close immediately
            client.close();
          }
        }
      }
    }

    private TNonblockingTransport doAccept() {
      try {
        return (TNonblockingTransport) serverTransport.accept();
      } catch (TTransportException tte) {
        // something went wrong accepting.
        LOGGER.warn("Exception trying to accept!", tte);
        return null;
      }
    }

    private void doAddAccept(SelectorThread thread, TNonblockingTransport client) {
      if (!thread.addAcceptedConnection(client)) {
        client.close();
      }
    }
  } // AcceptThread
```

## SelectorThread

```java
/**
   * The SelectorThread(s) will be doing all the selecting on accepted active
   * connections.
   */
  protected class SelectorThread extends AbstractSelectThread {

    // Accepted connections added by the accept thread.
    private final BlockingQueue<TNonblockingTransport> acceptedQueue;
    private int SELECTOR_AUTO_REBUILD_THRESHOLD = 512;
    private long MONITOR_PERIOD = 1000L;
    private int jvmBug = 0;

    /**
     * Set up the SelectorThread with an unbounded queue for incoming accepts.
     *
     * @throws IOException
     *           if a selector cannot be created
     */
    public SelectorThread() throws IOException {
      this(new LinkedBlockingQueue<TNonblockingTransport>());
    }

    /**
     * Set up the SelectorThread with an bounded queue for incoming accepts.
     *
     * @throws IOException
     *           if a selector cannot be created
     */
    public SelectorThread(int maxPendingAccepts) throws IOException {
      this(createDefaultAcceptQueue(maxPendingAccepts));
    }

    /**
     * Set up the SelectorThread with a specified queue for connections.
     *
     * @param acceptedQueue
     *          The BlockingQueue implementation for holding incoming accepted
     *          connections.
     * @throws IOException
     *           if a selector cannot be created.
     */
    public SelectorThread(BlockingQueue<TNonblockingTransport> acceptedQueue) throws IOException {
      this.acceptedQueue = acceptedQueue;
    }

    /**
     * Hands off an accepted connection to be handled by this thread. This
     * method will block if the queue for new connections is at capacity.
     *
     * @param accepted
     *          The connection that has been accepted.
     * @return true if the connection has been successfully added.
     */
    public boolean addAcceptedConnection(TNonblockingTransport accepted) {
      try {
        acceptedQueue.put(accepted);
      } catch (InterruptedException e) {
        LOGGER.warn("Interrupted while adding accepted connection!", e);
        return false;
      }
      selector.wakeup();
      return true;
    }

    /**
     * The work loop. Handles selecting (read/write IO), dispatching, and
     * managing the selection preferences of all existing connections.
     */
    public void run() {
      try {
        while (!stopped_) {
          select();
          processAcceptedConnections();
          processInterestChanges();
        }
        for (SelectionKey selectionKey : selector.keys()) {
          cleanupSelectionKey(selectionKey);
        }
      } catch (Throwable t) {
        LOGGER.error("run() on SelectorThread exiting due to uncaught error", t);
      } finally {
        try {
          selector.close();
        } catch (IOException e) {
          LOGGER.error("Got an IOException while closing selector!", e);
        }
        // This will wake up the accept thread and the other selector threads
        TThreadedSelectorServer.this.stop();
      }
    }

    /**
     * Select and process IO events appropriately: If there are existing
     * connections with data waiting to be read, read it, buffering until a
     * whole frame has been read. If there are any pending responses, buffer
     * them until their target client is available, and then send the data.
     */
    private void select() {
      try {

        doSelect();

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

          if (key.isReadable()) {
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

    /**
     * Do select and judge epoll bug happen.
     * See : https://issues.apache.org/jira/browse/THRIFT-4251
     */
    private void doSelect() throws IOException {
      long beforeSelect = System.currentTimeMillis();
      int selectedNums = selector.select();
      long afterSelect = System.currentTimeMillis();

      if (selectedNums == 0) {
        jvmBug++;
      } else {
        jvmBug = 0;
      }

      long selectedTime = afterSelect - beforeSelect;
      if (selectedTime >= MONITOR_PERIOD) {
        jvmBug = 0;
      } else if (jvmBug > SELECTOR_AUTO_REBUILD_THRESHOLD) {
        LOGGER.warn("In {} ms happen {} times jvm bug; rebuilding selector.", MONITOR_PERIOD, jvmBug);
        rebuildSelector();
        selector.selectNow();
        jvmBug = 0;
      }

    }

    /**
     * Replaces the current Selector of this SelectorThread with newly created Selector to work
     * around the infamous epoll 100% CPU bug.
     */
    private synchronized void rebuildSelector() {
      final Selector oldSelector = selector;
      if (oldSelector == null) {
        return;
      }
      Selector newSelector = null;
      try {
        newSelector = Selector.open();
        LOGGER.warn("Created new Selector.");
      } catch (IOException e) {
        LOGGER.error("Create new Selector error.", e);
      }

      for (SelectionKey key : oldSelector.selectedKeys()) {
        if (!key.isValid() && key.readyOps() == 0)
          continue;
        SelectableChannel channel = key.channel();
        Object attachment = key.attachment();

        try {
          if (attachment == null) {
            channel.register(newSelector, key.readyOps());
          } else {
            channel.register(newSelector, key.readyOps(), attachment);
          }
        } catch (ClosedChannelException e) {
          LOGGER.error("Register new selector key error.", e);
        }

      }

      selector = newSelector;
      try {
        oldSelector.close();
      } catch (IOException e) {
        LOGGER.error("Close old selector error.", e);
      }
      LOGGER.warn("Replace new selector success.");
    }

    private void processAcceptedConnections() {
      // Register accepted connections
      while (!stopped_) {
        TNonblockingTransport accepted = acceptedQueue.poll();
        if (accepted == null) {
          break;
        }
        registerAccepted(accepted);
      }
    }

    protected FrameBuffer createFrameBuffer(final TNonblockingTransport trans,
        final SelectionKey selectionKey,
        final AbstractSelectThread selectThread) {
        return processorFactory_.isAsyncProcessor() ?
                  new AsyncFrameBuffer(trans, selectionKey, selectThread) :
                  new FrameBuffer(trans, selectionKey, selectThread);
    }

    private void registerAccepted(TNonblockingTransport accepted) {
      SelectionKey clientKey = null;
      try {
        clientKey = accepted.registerSelector(selector, SelectionKey.OP_READ);

        FrameBuffer frameBuffer = createFrameBuffer(accepted, clientKey, SelectorThread.this);

        clientKey.attach(frameBuffer);
      } catch (IOException e) {
        LOGGER.warn("Failed to register accepted connection to selector!", e);
        if (clientKey != null) {
          cleanupSelectionKey(clientKey);
        }
        accepted.close();
      }
    }
  } // SelectorThread
```

## 小结

|  | 是否阻塞IO | ACCEPT处理 | IO处理 | 业务逻辑调用 | 特点 | 适用情况 |
|:-----------------------:|:----------:|:----------:|:------:|:------------:|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|:-------------------------------------------------------------------------:|
| TSimpleServer | 阻塞 | -- | -- | -- | 单线程处理所有操作，同一时间只能处理一个客户端连接， 当前客户端断开连接后才能接收下一个连接 | 测试使用，不能在生产环境使用 |
| TThreadPoolServer | 阻塞 | 单线程 | 线程池 | 线程池 | 有一个专用的线程用来接受连接，一旦接受了一个连接， 它就会被放入ThreadPoolExecutor中的一个worker线程里处理， worker线程被绑定到特定的客户端连接上，直到它关闭。 一旦连接关闭，该worker线程就又回到了线程池中。 如果客户端数量超过了线程池中的最大线程数，在有一个worker线程可用之前， 请求将被一直阻塞在那里。 | 性能较高，适合并发Client连接数不是太高的情况 |
| TNonblockingServer | 非阻塞 | 单线程 | 单线程 | 单线程 | 采用非阻塞的I/O可以单线程监控多个连接， 所有处理是被调用select()方法的同一个线程顺序处理的 | 适用于业务处理简单，客户端连接较少的情况， 不适合高并发场景，单线程效率低 |
| THsHaServer | 非阻塞 | 单线程 | 单线程 | 线程池 | 半同步半异步，使用一个单独的线程来处理接收连接和网络I/O，一个独立的worker线程池来处理消息。 只要有空闲的worker线程，消息就会被立即处理，因此多条消息能被并行处理。 | 适用于网络I/O不是太繁忙、对业务逻辑调用要求较高的场景 |
| TThreadedSelectorServer | 非阻塞 | 单线程 | 多线程 | 线程池 | 半同步半异步Server。用多个线程来处理网络I/O，用线程池来进行业务逻辑调用的处理。 当网络I/O是瓶颈的时候，TThreadedSelectorServer比THsHaServer的表现要好。 | 适用于网络I/O繁忙、对业务逻辑调用要求较高的、高并发场景 |

一般情况下，生产环境中使用会在TThreadPoolServer和TThreadedSelectorServer中选一个。TThreadPoolServer优势是处理速度快、响应时间短，缺点是在高并发情况下占用系统资源较高；TThreadedSelectorServer优势是支持高并发，劣势是处理速度没有TThreadPoolServer高，但在大多数情况下能也满足业务需要。



## 参考引用

- [1] [TThreadedSelectorServer.java](https://github.com/apache/thrift/blob/master/lib/java/src/org/apache/thrift/server/TThreadedSelectorServer.java)
- [2] [RPC-Thrift（一）](https://www.cnblogs.com/zaizhoumo/p/8184923.html)
