---
title: 'apache thrift 学习笔记: thrift client小结'
date: 2018-12-25 16:37:36
tags: [并发, thrift]
categories: [并发]
toc: true
description: 在梳理了Thrift Server端实现逻辑的基础上， 这里总结一下Thrift Client端代码逻辑。 Thrift Client 有同步和异步两种， 两种客户端分别对应各自的使用场景。 
---

在梳理了Thrift Server端实现逻辑的基础上， 这里总结一下Thrift Client端代码逻辑。 Thrift Client 有同步和异步两种， 两种客户端分别对应各自的使用场景。 

## 同步客户端

### 类图

参引 [RPC-Thrift（一）](http://www.cnblogs.com/zaizhoumo/p/8184923.html) 中的例子。 Thrift client端的类图如下： 

![](https://wx4.sinaimg.cn/mw690/7c35df9bly1fyj27xpr2pj207807j74e.jpg) 

- TServiceClient：用于以同步方式与TService进行通信；
- Iface接口和Client类都是通过Thrift文件自动生成的代码。

### TServiceClient

TService 对底层TTransport和TProtocol进行封装， 通过TTransport与Server端保持一个长连接， 同一时刻只能处理一个请求， 请求的处理过程是阻塞的， 客户端必须等待服务端结果返回才能进行后续的逻辑处理。 

```java
/**
 * A TServiceClient is used to communicate with a TService implementation
 * across protocols and transports.
 */
public abstract class TServiceClient {
  public TServiceClient(TProtocol prot) {
    this(prot, prot);
  }

  public TServiceClient(TProtocol iprot, TProtocol oprot) {
    iprot_ = iprot;
    oprot_ = oprot;
  }

  protected TProtocol iprot_;
  protected TProtocol oprot_;

  /**
  * 这里需要特别注意这个序列号， 每个序列号对应于一次请求
  * 这个序列号是线程非安全的， 也就是说如果服务层有多个线程同事调用client
  * 1. 要么每个线程对应一个client线程
  * 2. 要么加锁等待上一次调用返回
  */
  protected int seqid_;

  /**
   * Get the TProtocol being used as the input (read) protocol.
   * @return the TProtocol being used as the input (read) protocol.
   */
  public TProtocol getInputProtocol() {
    return this.iprot_;
  }

  /**
   * Get the TProtocol being used as the output (write) protocol.
   * @return the TProtocol being used as the output (write) protocol.
   */
  public TProtocol getOutputProtocol() {
    return this.oprot_;
  }

  protected void sendBase(String methodName, TBase<?,?> args) throws TException {
    sendBase(methodName, args, TMessageType.CALL);
  }

  protected void sendBaseOneway(String methodName, TBase<?,?> args) throws TException {
    sendBase(methodName, args, TMessageType.ONEWAY);
  }

  private void sendBase(String methodName, TBase<?,?> args, byte type) throws TException {
    oprot_.writeMessageBegin(new TMessage(methodName, type, ++seqid_));
    args.write(oprot_);
    oprot_.writeMessageEnd();
    oprot_.getTransport().flush();
  }

  protected void receiveBase(TBase<?,?> result, String methodName) throws TException {
    TMessage msg = iprot_.readMessageBegin();
    if (msg.type == TMessageType.EXCEPTION) {
      TApplicationException x = new TApplicationException();
      x.read(iprot_);
      iprot_.readMessageEnd();
      throw x;
    }
    if (msg.seqid != seqid_) {
      throw new TApplicationException(TApplicationException.BAD_SEQUENCE_ID,
          String.format("%s failed: out of sequence response: expected %d but got %d", methodName, seqid_, msg.seqid));
    }
    result.read(iprot_);
    iprot_.readMessageEnd();
  }
}

```

可以看到，TServiceClient不关注具体的业务逻辑， 只是将数据发送和接收逻辑抽象出来。 

### 同步Client实现

这里我简单写了一个Thrift定义， 通过Thrift 0.11.0自动生成了同步Client端源码。 

- 接口定义

定义一个接口实现简单的两个数的加减操作。 

```thrift
namespace java com.su.thrift

exception CatchableException {
    1: i32 code;
    2: string msg;
}

enum OperateType{
    SUM,
    MINUS
}

struct SuStruct{
    1: i64 result;
    2: string operateType;
}

service SuService {
    SuStruct simpleCalculate(1:i64 a, 2:i64 b, 3:OperateType op) throws (CatchableException ce);
}
```

- 自动生成的Iface接口

```java 
  public interface Iface {

    public SuStruct simpleCalculate(long a, long b, OperateType op) throws CatchableException, org.apache.thrift.TException;

  }
```

- TServiceClient子类实现

自动生成的同步Client类， 继承了TServiceClient， 并实现了业务逻辑接口， 其只要作用是对接口中方法的参数进行封装， 进一步调用父类的send以及receive方法， 实现rpc调用。 

```java
  public static class Client extends org.apache.thrift.TServiceClient implements Iface {
    public static class Factory implements org.apache.thrift.TServiceClientFactory<Client> {
      public Factory() {}
      public Client getClient(org.apache.thrift.protocol.TProtocol prot) {
        return new Client(prot);
      }
      public Client getClient(org.apache.thrift.protocol.TProtocol iprot, org.apache.thrift.protocol.TProtocol oprot) {
        return new Client(iprot, oprot);
      }
    }

    public Client(org.apache.thrift.protocol.TProtocol prot)
    {
      super(prot, prot);
    }

    public Client(org.apache.thrift.protocol.TProtocol iprot, org.apache.thrift.protocol.TProtocol oprot) {
      super(iprot, oprot);
    }

    public SuStruct simpleCalculate(long a, long b, OperateType op) throws CatchableException, org.apache.thrift.TException
    {
      send_simpleCalculate(a, b, op);
      return recv_simpleCalculate();
    }

    public void send_simpleCalculate(long a, long b, OperateType op) throws org.apache.thrift.TException
    {
      simpleCalculate_args args = new simpleCalculate_args();
      args.setA(a);
      args.setB(b);
      args.setOp(op);
      sendBase("simpleCalculate", args);
    }

    public SuStruct recv_simpleCalculate() throws CatchableException, org.apache.thrift.TException
    {
      simpleCalculate_result result = new simpleCalculate_result();
      receiveBase(result, "simpleCalculate");
      if (result.isSetSuccess()) {
        return result.success;
      }
      if (result.ce != null) {
        throw result.ce;
      }
      throw new org.apache.thrift.TApplicationException(org.apache.thrift.TApplicationException.MISSING_RESULT, "simpleCalculate failed: unknown result");
    }

  }
```

## 异步客户端

### 类图

异步客户端需要使用TNonblockingSocket，通过AsyncMethodCallback接收服务端的回调。

先来看AsyncClient调用过程： 

```java
public class HaHsAsyncClient {
    private static final Logger logger = LoggerFactory.getLogger(HaHsAsyncClient.class);

    public static void main(String[] args)
            throws IOException, TException, InterruptedException {
        TProtocolFactory protocolFactory = new TBinaryProtocol.Factory();
        TAsyncClientManager clientManager = new TAsyncClientManager();
        TNonblockingSocket transport = new TNonblockingSocket("localhost", 9001);
        SuService.AsyncClient asyncClient = new SuService.AsyncClient(protocolFactory, clientManager, transport);

        long left = 1L;
        long right = 2L;

        // 注意这里的CountDownLatch的使用， 每个client同一时刻只能处理一个请求， 这些请求共享同一个socket链接。 
        CountDownLatch countDownLatch = new CountDownLatch(1);
        asyncClient.simpleCalculate(left, right, OperateType.MINUS,
                new HaHsAsyncResultHandler(left, right, OperateType.MINUS, countDownLatch));
        countDownLatch.await(2000, TimeUnit.MILLISECONDS);

        CountDownLatch countDownLatch1 = new CountDownLatch(1);
        asyncClient.simpleCalculate(left, right, OperateType.SUM,
                new HaHsAsyncResultHandler(left, right, OperateType.SUM, countDownLatch1));

        countDownLatch1.await(2000, TimeUnit.MILLISECONDS);
        transport.close();
    }

    public static class HaHsAsyncResultHandler implements AsyncMethodCallback<SuStruct> {

        private CountDownLatch countDownLatch;
        private long leftOperator;
        private long rightOperator;
        private OperateType operateType;

        public HaHsAsyncResultHandler(long leftOperator,
                                      long rightOperator,
                                      OperateType operateType,
                                      CountDownLatch countDownLatch) {
            this.leftOperator = leftOperator;
            this.rightOperator = rightOperator;
            this.operateType = operateType;
            this.countDownLatch = countDownLatch;
        }

        @Override
        public void onComplete(SuStruct response) {
            System.out.println(response);
            this.countDownLatch.countDown();
        }

        @Override
        public void onError(Exception exception) {
            logger.error("Error in call simple calculate with left operator {} {} with right operator {} .",
                    this.leftOperator, this.rightOperator, this.operateType, exception);
            this.countDownLatch.countDown();
        }
    }
}
```

![](https://wx2.sinaimg.cn/mw690/7c35df9bly1fyj27vk8pgj20j50ejgn2.jpg)

- TAsyncClient：异步客户端抽象类，通过Thrift文件生成的AsyncClient需继承该类；
- TAsyncClientManager：异步客户端管理类，包含一个selector线程，用于转换方法调用对象；
- TAsyncMethodCall：封装了异步方法调用，Thrift文件定义的所有方法都会在AsyncClient中生成对应的继承于TAsyncMethodCall的内部类（如sayHello_call）；
- AsyncMethodCallback：接收服务端回调的接口，用户需要定义实现该接口的类。

### TAsyncClient

TAsyncClient是非线程安全的， 方法调用通过__currentMethod进行区分。 

```java
public abstract class TAsyncClient {
  protected final TProtocolFactory ___protocolFactory;
  protected final TNonblockingTransport ___transport;
  protected final TAsyncClientManager ___manager;
  // 这里需要注意的一点， AsyncClient同一时刻还是只能处理一次远程请求调用
  // ___currentMethod != null 则不能处理新的调用请求
  protected TAsyncMethodCall ___currentMethod;
  private Exception ___error;
  private long ___timeout;

  public TAsyncClient(TProtocolFactory protocolFactory, TAsyncClientManager manager, TNonblockingTransport transport) {
    this(protocolFactory, manager, transport, 0);
  }

  public TAsyncClient(TProtocolFactory protocolFactory, TAsyncClientManager manager, TNonblockingTransport transport, long timeout) {
    this.___protocolFactory = protocolFactory;
    this.___manager = manager;
    this.___transport = transport;
    this.___timeout = timeout;
  }

  public TProtocolFactory getProtocolFactory() {
    return ___protocolFactory;
  }

  public long getTimeout() {
    return ___timeout;
  }

  public boolean hasTimeout() {
    return ___timeout > 0;
  }

  public void setTimeout(long timeout) {
    this.___timeout = timeout;
  }

  /**
   * Is the client in an error state?
   * @return If client in an error state?
   */
  public boolean hasError() {
    return ___error != null;
  }

  /**
   * Get the client's error - returns null if no error
   * @return Get the client's error. <p> returns null if no error
   */
  public Exception getError() {
    return ___error;
  }

  protected void checkReady() {
    // Ensure we are not currently executing a method
    if (___currentMethod != null) {
      throw new IllegalStateException("Client is currently executing another method: " + ___currentMethod.getClass().getName());
    }

    // Ensure we're not in an error state
    if (___error != null) {
      throw new IllegalStateException("Client has an error!", ___error);
    }
  }

  /**
   * Called by delegate method when finished
   */
  protected void onComplete() {
    ___currentMethod = null;
  }

  /**
   * Called by delegate method on error
   */
  protected void onError(Exception exception) {
    ___transport.close();
    ___currentMethod = null;
    ___error = exception;
  }
}
```

### AsyncClient 

AsyncClient继承了TAsyncClient, 并且实现了AsyncIface接口, 主要实现了对业务方法的参数封装和结果获取， 初始化具体的AsyncMethodCall类， 调用TAsyncClientManager， 通过底层的NIO实现异步的远程方法调用。 

```java
  public static class AsyncClient extends org.apache.thrift.async.TAsyncClient implements AsyncIface {
    public static class Factory implements org.apache.thrift.async.TAsyncClientFactory<AsyncClient> {
      private org.apache.thrift.async.TAsyncClientManager clientManager;
      private org.apache.thrift.protocol.TProtocolFactory protocolFactory;
      public Factory(org.apache.thrift.async.TAsyncClientManager clientManager, org.apache.thrift.protocol.TProtocolFactory protocolFactory) {
        this.clientManager = clientManager;
        this.protocolFactory = protocolFactory;
      }
      public AsyncClient getAsyncClient(org.apache.thrift.transport.TNonblockingTransport transport) {
        return new AsyncClient(protocolFactory, clientManager, transport);
      }
    }

    public AsyncClient(org.apache.thrift.protocol.TProtocolFactory protocolFactory, org.apache.thrift.async.TAsyncClientManager clientManager, org.apache.thrift.transport.TNonblockingTransport transport) {
      super(protocolFactory, clientManager, transport);
    }

    public void simpleCalculate(long a, long b, OperateType op, org.apache.thrift.async.AsyncMethodCallback<SuStruct> resultHandler) throws org.apache.thrift.TException {
      checkReady();
      simpleCalculate_call method_call = new simpleCalculate_call(a, b, op, resultHandler, this, ___protocolFactory, ___transport);
      this.___currentMethod = method_call;
      ___manager.call(method_call);
    }

    public static class simpleCalculate_call extends org.apache.thrift.async.TAsyncMethodCall<SuStruct> {
      private long a;
      private long b;
      private OperateType op;
      public simpleCalculate_call(long a, long b, OperateType op, org.apache.thrift.async.AsyncMethodCallback<SuStruct> resultHandler, org.apache.thrift.async.TAsyncClient client, org.apache.thrift.protocol.TProtocolFactory protocolFactory, org.apache.thrift.transport.TNonblockingTransport transport) throws org.apache.thrift.TException {
        super(client, protocolFactory, transport, resultHandler, false);
        this.a = a;
        this.b = b;
        this.op = op;
      }

      public void write_args(org.apache.thrift.protocol.TProtocol prot) throws org.apache.thrift.TException {
        prot.writeMessageBegin(new org.apache.thrift.protocol.TMessage("simpleCalculate", org.apache.thrift.protocol.TMessageType.CALL, 0));
        simpleCalculate_args args = new simpleCalculate_args();
        args.setA(a);
        args.setB(b);
        args.setOp(op);
        args.write(prot);
        prot.writeMessageEnd();
      }

      public SuStruct getResult() throws CatchableException, org.apache.thrift.TException {
        if (getState() != org.apache.thrift.async.TAsyncMethodCall.State.RESPONSE_READ) {
          throw new java.lang.IllegalStateException("Method call not finished!");
        }
        org.apache.thrift.transport.TMemoryInputTransport memoryTransport = new org.apache.thrift.transport.TMemoryInputTransport(getFrameBuffer().array());
        org.apache.thrift.protocol.TProtocol prot = client.getProtocolFactory().getProtocol(memoryTransport);
        return (new Client(prot)).recv_simpleCalculate();
      }
    }

  }
```

### TAsyncClientManager

TAsyncClientManager是异步客户端管理类，它为维护了一个待处理的方法调用队列pendingCalls，并通过SelectThread线程监听selector事件，当有就绪事件时进行方法调用的处理。

```java 
public class TAsyncClientManager {
  private static final Logger LOGGER = LoggerFactory.getLogger(TAsyncClientManager.class.getName());
  private final SelectThread selectThread;
  //TAsyncMethodCall待处理队列
  private final ConcurrentLinkedQueue<TAsyncMethodCall> pendingCalls = new ConcurrentLinkedQueue<TAsyncMethodCall>();
  //初始化TAsyncClientManager，新建selectThread线程并启动
  public TAsyncClientManager() throws IOException {
    this.selectThread = new SelectThread();
    selectThread.start();
  }
  //方法调用
  public void call(TAsyncMethodCall method) throws TException {
    if (!isRunning()) {
      throw new TException("SelectThread is not running");
    }
    method.prepareMethodCall();//做方法调用前的准备
    pendingCalls.add(method);//加入待处理队列
    selectThread.getSelector().wakeup();//唤醒selector，很重要，因为首次执行方法调用时select Thread还阻塞在selector.select()上
  }
  public void stop() {
    selectThread.finish();
  }
  public boolean isRunning() {
    return selectThread.isAlive();
  }
  //SelectThread线程类，处理方法调用的核心
  private class SelectThread extends Thread {
    private final Selector selector;
    private volatile boolean running;
    private final TreeSet<TAsyncMethodCall> timeoutWatchSet = new TreeSet<TAsyncMethodCall>(new TAsyncMethodCallTimeoutComparator());

    public SelectThread() throws IOException {
      this.selector = SelectorProvider.provider().openSelector();
      this.running = true;
      this.setName("TAsyncClientManager#SelectorThread " + this.getId());
      setDaemon(true);//非守护线程
    }
    public Selector getSelector() {
      return selector;
    }
    public void finish() {
      running = false;
      selector.wakeup();
    }
    public void run() {
      while (running) {
        try {
          try {
            
            if (timeoutWatchSet.size() == 0) {
              //如果超时TAsyncMethodCall监控集合为空，直接无限期阻塞监听select()事件。TAsyncClientManager刚初始化时是空的
              selector.select();
            } else {
              //如果超时TAsyncMethodCall监控集合不为空，则计算Set中第一个元素的超时时间戳是否到期
              long nextTimeout = timeoutWatchSet.first().getTimeoutTimestamp();
              long selectTime = nextTimeout - System.currentTimeMillis();
              if (selectTime > 0) {
                //还没有到期，超时监听select()事件，超过selectTime自动唤醒selector
                selector.select(selectTime);
              } else {
                //已经到期，立刻监听select()事件，不会阻塞selector
                selector.selectNow();
              }
            }
          } catch (IOException e) {
            LOGGER.error("Caught IOException in TAsyncClientManager!", e);
          }
          //监听到就绪事件或者selector被唤醒会执行到此处
          transitionMethods();//处理就绪keys
          timeoutMethods();//超时方法调用处理
          startPendingMethods();//处理pending的方法调用
        } catch (Exception exception) {
          LOGGER.error("Ignoring uncaught exception in SelectThread", exception);
        }
      }
    }
    //监听到就绪事件或者selector被唤醒，如果有就绪的SelectionKey就调用methodCall.transition(key);
    private void transitionMethods() {
      try {
        Iterator<SelectionKey> keys = selector.selectedKeys().iterator();
        while (keys.hasNext()) {
          SelectionKey key = keys.next();
          keys.remove();
          if (!key.isValid()) {
            //跳过无效key，方法调用出现异常或key被取消等会导致无效key
            continue;
          }
          TAsyncMethodCall methodCall = (TAsyncMethodCall)key.attachment();
          //调用methodCall的transition方法，执行相关的动作并将methodCall的状态转换为下一个状态
          methodCall.transition(key);
          //如果完成或发生错误，从timeoutWatchSet删除该methodCall
          if (methodCall.isFinished() || methodCall.getClient().hasError()) {
            timeoutWatchSet.remove(methodCall);
          }
        }
      } catch (ClosedSelectorException e) {
        LOGGER.error("Caught ClosedSelectorException in TAsyncClientManager!", e);
      }
    }
    //超时方法调用处理
    private void timeoutMethods() {
      Iterator<TAsyncMethodCall> iterator = timeoutWatchSet.iterator();
      long currentTime = System.currentTimeMillis();
      while (iterator.hasNext()) {
        TAsyncMethodCall methodCall = iterator.next();
        if (currentTime >= methodCall.getTimeoutTimestamp()) {
          //如果超时，从timeoutWatchSet中删除并调用onError()方法
          iterator.remove();
          methodCall.onError(new TimeoutException("Operation " + methodCall.getClass() + " timed out after " + (currentTime - methodCall.getStartTime()) + " ms."));
        } else {
          //如果没有超时，说明之后的TAsyncMethodCall也不会超时，跳出循环，因为越早进入timeoutWatchSet的TAsyncMethodCall越先超时。
          break;
        }
      }
    }
    //开始等待的方法调用，循环处理pendingCalls中的methodCall
    private void startPendingMethods() {
      TAsyncMethodCall methodCall;
      while ((methodCall = pendingCalls.poll()) != null) {
        // Catch registration errors. method will catch transition errors and cleanup.
        try {
          //向selector注册并设置初次状态
          methodCall.start(selector);
          //如果客户端指定了超时时间且transition成功，将methodCall加入到timeoutWatchSet
          TAsyncClient client = methodCall.getClient();
          if (client.hasTimeout() && !client.hasError()) {
            timeoutWatchSet.add(methodCall);
          }
        } catch (Exception exception) {
          //异常处理
          LOGGER.warn("Caught exception in TAsyncClientManager!", exception);
          methodCall.onError(exception);
        }
      }
    }
  }
  //TreeSet用的比较器，判断是否是同一个TAsyncMethodCall实例
  private static class TAsyncMethodCallTimeoutComparator implements Comparator<TAsyncMethodCall> {
    public int compare(TAsyncMethodCall left, TAsyncMethodCall right) {
      if (left.getTimeoutTimestamp() == right.getTimeoutTimestamp()) {
        return (int)(left.getSequenceId() - right.getSequenceId());
      } else {
        return (int)(left.getTimeoutTimestamp() - right.getTimeoutTimestamp());
      }
    }
  }

```

### TAsyncMethodCall

TAsyncMethodCall实现了对方法调用的封装。一次方法调用过程就是一个TAsyncMethodCall实例的生命周期。TAsyncMethodCall实例在整个生命周期内有以下状态，正常情况下的状态状态过程为：CONNECTING -> WRITING_REQUEST_SIZE -> WRITING_REQUEST_BODY -> READING_RESPONSE_SIZE -> READING_RESPONSE_BODY -> RESPONSE_READ，如果任何一个过程中发生了异常则直接转换为ERROR状态。

```java
public static enum State {
    CONNECTING,//连接状态
    WRITING_REQUEST_SIZE,//写请求size
    WRITING_REQUEST_BODY,//写请求体
    READING_RESPONSE_SIZE,//读响应size
    READING_RESPONSE_BODY,//读响应体
    RESPONSE_READ,//读响应完成
    ERROR;//异常状态
  }
```

TAsyncMethodCall的源码分析如下：

```java
public abstract class TAsyncMethodCall<T> {
  private static final int INITIAL_MEMORY_BUFFER_SIZE = 128;
  private static AtomicLong sequenceIdCounter = new AtomicLong(0);//序列号计数器private State state = null;//状态在start()方法中初始化
  protected final TNonblockingTransport transport;
  private final TProtocolFactory protocolFactory;
  protected final TAsyncClient client;
  private final AsyncMethodCallback<T> callback;//回调实例
  private final boolean isOneway;
  private long sequenceId;//序列号
  
  private ByteBuffer sizeBuffer;//Java NIO概念，frameSize buffer
  private final byte[] sizeBufferArray = new byte[4];//4字节的消息Size字节数组
  private ByteBuffer frameBuffer;//Java NIO概念，frame buffer

  private long startTime = System.currentTimeMillis();

  protected TAsyncMethodCall(TAsyncClient client, TProtocolFactory protocolFactory, TNonblockingTransport transport, AsyncMethodCallback<T> callback, boolean isOneway) {
    this.transport = transport;
    this.callback = callback;
    this.protocolFactory = protocolFactory;
    this.client = client;
    this.isOneway = isOneway;
    this.sequenceId = TAsyncMethodCall.sequenceIdCounter.getAndIncrement();
  }
  protected State getState() {
    return state;
  }
  protected boolean isFinished() {
    return state == State.RESPONSE_READ;
  }
  protected long getStartTime() {
    return startTime;
  }
  protected long getSequenceId() {
    return sequenceId;
  }
  public TAsyncClient getClient() {
    return client;
  }
  public boolean hasTimeout() {
    return client.hasTimeout();
  }
  public long getTimeoutTimestamp() {
    return client.getTimeout() + startTime;
  }
  //将请求写入protocol，由子类实现
  protected abstract void write_args(TProtocol protocol) throws TException;
  //方法调用前的准备处理，初始化frameBuffer和sizeBuffer
  protected void prepareMethodCall() throws TException {
    //TMemoryBuffer内存缓存传输类，继承了TTransport
    TMemoryBuffer memoryBuffer = new TMemoryBuffer(INITIAL_MEMORY_BUFFER_SIZE);
    TProtocol protocol = protocolFactory.getProtocol(memoryBuffer);
    write_args(protocol);//将请求写入protocol

    int length = memoryBuffer.length();
    frameBuffer = ByteBuffer.wrap(memoryBuffer.getArray(), 0, length);

    TFramedTransport.encodeFrameSize(length, sizeBufferArray);
    sizeBuffer = ByteBuffer.wrap(sizeBufferArray);
  }
  //向selector注册并设置开始状态，可能是连接状态或写状态
  void start(Selector sel) throws IOException {
    SelectionKey key;
    if (transport.isOpen()) {
      state = State.WRITING_REQUEST_SIZE;
      key = transport.registerSelector(sel, SelectionKey.OP_WRITE);
    } else {
      state = State.CONNECTING;
      key = transport.registerSelector(sel, SelectionKey.OP_CONNECT);
      //如果是非阻塞连接初始化会立即成功，转换为写状态并修改感兴趣事件
      if (transport.startConnect()) {
        registerForFirstWrite(key);
      }
    }
    key.attach(this);//将本methodCall附加在key上
  }
  void registerForFirstWrite(SelectionKey key) throws IOException {
    state = State.WRITING_REQUEST_SIZE;
    key.interestOps(SelectionKey.OP_WRITE);
  }
  protected ByteBuffer getFrameBuffer() {
    return frameBuffer;
  }
  //转换为下一个状态，根据不同的状态做不同的处理。该方法只会在selector thread中被调用，不用担心并发
  protected void transition(SelectionKey key) {
    // 确保key是有效的
    if (!key.isValid()) {
      key.cancel();
      Exception e = new TTransportException("Selection key not valid!");
      onError(e);
      return;
    }
    try {
      switch (state) {
        case CONNECTING:
          doConnecting(key);//建连接
          break;
        case WRITING_REQUEST_SIZE:
          doWritingRequestSize();//写请求size
          break;
        case WRITING_REQUEST_BODY:
          doWritingRequestBody(key);//写请求体
          break;
        case READING_RESPONSE_SIZE:
          doReadingResponseSize();//读响应size
          break;
        case READING_RESPONSE_BODY:
          doReadingResponseBody(key);//读响应体
          break;
        default: // RESPONSE_READ, ERROR, or bug
          throw new IllegalStateException("Method call in state " + state
              + " but selector called transition method. Seems like a bug...");
      }
    } catch (Exception e) {
      key.cancel();
      key.attach(null);
      onError(e);
    }
  }
  //出现异常时的处理
  protected void onError(Exception e) {
    client.onError(e);//置Client异常信息
    callback.onError(e);//回调异常方法
    state = State.ERROR;//置当前对象为ERROR状态
  }
  //读响应消息体
  private void doReadingResponseBody(SelectionKey key) throws IOException {
    if (transport.read(frameBuffer) < 0) {
      throw new IOException("Read call frame failed");
    }
    if (frameBuffer.remaining() == 0) {
      cleanUpAndFireCallback(key);
    }
  }
  //方法调用完成的处理
  private void cleanUpAndFireCallback(SelectionKey key) {
    state = State.RESPONSE_READ;//状态转换为读取response完成
    key.interestOps(0);//清空感兴趣事件
    key.attach(null);//清理key的附加信息
    client.onComplete();//将client的___currentMethod置为null
    callback.onComplete((T)this);//回调onComplete方法
  }
  //读响应size，同样可能需要多多次直到把sizeBuffer读满
  private void doReadingResponseSize() throws IOException {
    if (transport.read(sizeBuffer) < 0) {
      throw new IOException("Read call frame size failed");
    }
    if (sizeBuffer.remaining() == 0) {
      state = State.READING_RESPONSE_BODY;
      //读取FrameSize完成，为frameBuffer分配FrameSize大小的空间用于读取响应体
      frameBuffer = ByteBuffer.allocate(TFramedTransport.decodeFrameSize(sizeBufferArray));
    }
  }
  //写请求体
  private void doWritingRequestBody(SelectionKey key) throws IOException {
    if (transport.write(frameBuffer) < 0) {
      throw new IOException("Write call frame failed");
    }
    if (frameBuffer.remaining() == 0) {
      if (isOneway) {
        //如果是单向RPC，此时方法调用已经结束,清理key并进行回调
        cleanUpAndFireCallback(key);
      } else {
        //非单向RPC，状态转换为READING_RESPONSE_SIZE
        state = State.READING_RESPONSE_SIZE;
        //重置sizeBuffer，准备读取frame size
        sizeBuffer.rewind();
        key.interestOps(SelectionKey.OP_READ);//修改感兴趣事件
      }
    }
  }
  //写请求size到transport，可能会写多次直到sizeBuffer.remaining() == 0才转换状态
  private void doWritingRequestSize() throws IOException {
    if (transport.write(sizeBuffer) < 0) {
      throw new IOException("Write call frame size failed");
    }
    if (sizeBuffer.remaining() == 0) {
      state = State.WRITING_REQUEST_BODY;
    }
  }
  //建立连接
  private void doConnecting(SelectionKey key) throws IOException {
    if (!key.isConnectable() || !transport.finishConnect()) {
      throw new IOException("not connectable or finishConnect returned false after we got an OP_CONNECT");
    }
    registerForFirstWrite(key);
  }
}
```

## Client与Processor的方法识别与参数传递

进一步阅读Thrift自动生成的Service源码我们会发现， 为了玩完成每次远程调用： 

1. 服务端（Processor）和客户端（Client）必须为每个方法调用封装具体的Args、Result类结构， 
2. Processor端按照方法名维护具体的方法调用map： methodName -> ProcessFunction
3. 每次调用过程， 客户端按照具体的TProtocol规定的编码方式，将封装好的FrameSize + methodName + Args通过NIO TCP发送给服务端
4. 服务端，进行解码之后完成方法调用， 返回结果给客户端
5. 除非手动关闭连接，或者调用过程中遇到任何错误， Client与Server之间的链接是一直保持的


## 小结

需要注意的是，一个AsyncClient实例只能同时处理一个方法调用，必须等待前一个方法调用完成后才能使用该AsyncClient实例调用其他方法，疑问：和同步客户端相比有什么优势？不用等返回结果，可以干其他的活？又能干什么活呢？如果客户端使用了连接池（也是AsyncClient实例池，一个AsyncClient实例对应一个连接），该线程不用等待前一个连接进行方法调用的返回结果，就可以去线程池获取一个可用的连接，使用新的连接进行方法调用，而原来的连接在收到返回结果后，状态变为可用，返回给连接池。这样相对于同步客户端单个线程串行发送请求的情况，异步客户端单个线程进行发送请求的效率会大大提高，需要的线程数变小，但是可能需要的连接数会增大，单个请求的响应时间会变长。在线程数是性能瓶颈，或对请求的响应时间要求不高的情况下，使用异步客户端比较合适。

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fyk8c2kieuj20g00mv769.jpg)

## 参考引用

- [1] [apache thrift @ github](https://github.com/apache/thrift)
- [2] [RPC-Thrift（一）](http://www.cnblogs.com/zaizhoumo/p/8184923.html)
- [3] [RPC-Thrift（四）](https://www.cnblogs.com/zaizhoumo/p/8260455.html)
