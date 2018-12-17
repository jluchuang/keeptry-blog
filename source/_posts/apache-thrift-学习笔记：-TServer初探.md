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
   * FrameBuffer： 主要对NIO ByteBuffer和SocketChannel进行封装
   * SelectThread 的实现类主要对NIO Selector的封装进行封装
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

Thrift FrameBuffer类主要是对SocketChannel以及ByteBuffer进行封装， 主要的操作包括通过对已经Ready的IO事件进行处理， 包括：

- 从内核中读取Ready的数据； 
- 调用用户实现的Processor类， 完成数据的计算； 
- 将计算结果写回给内存；

这个过程中， 通过Java Selector.wakeup方法， FrameBuffer频繁通知Selector类： 当前的SocketChannel已经准备准备好Read/Write操作， 以完成对应IO Channel长连接的读写状态转换。 

Thrift为FrameBufffer定义了以下几个状态： 

```java
  /**
   * Possible states for the FrameBuffer state machine.
   */
  private enum FrameBufferState {
    // in the midst of reading the frame size off the wire
    READING_FRAME_SIZE,
    // reading the actual frame data now, but not all the way done yet
    READING_FRAME,
    // completely read the frame, so an invocation can now happen
    READ_FRAME_COMPLETE,
    // waiting to get switched to listening for write events
    AWAITING_REGISTER_WRITE,
    // started writing response data, not fully complete yet
    WRITING,
    // another thread wants this framebuffer to go back to reading
    AWAITING_REGISTER_READ,
    // we want our transport and selection key invalidated in the selector
    // thread
    AWAITING_CLOSE
  }
```

这里整理了一下正常情况下FrameBuffer状态的转换： 

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fy9q4s5qr1j20o00anaaz.jpg)

在任何状态执行时出现异常， FrameBuffer状态都会转换为AWAITING_CLOSE。 

接下来时FrameBuffer的源码：

```java
 /**
   * Class that implements a sort of state machine around the interaction with a
   * client and an invoker. It manages reading the frame size and frame data,
   * getting it handed off as wrapped transports, and then the writing of
   * response data back to the client. In the process it manages flipping the
   * read and write bits on the selection key for its client.
   */
   public class FrameBuffer {
    private final Logger LOGGER = LoggerFactory.getLogger(getClass().getName());

    // the actual transport hooked up to the client.
    protected final TNonblockingTransport trans_;

    // the SelectionKey that corresponds to our transport
    protected final SelectionKey selectionKey_;

    // the SelectThread that owns the registration of our transport
    protected final AbstractSelectThread selectThread_;

    // where in the process of reading/writing are we?
    protected FrameBufferState state_ = FrameBufferState.READING_FRAME_SIZE;

    // the ByteBuffer we'll be using to write and read, depending on the state
    protected ByteBuffer buffer_;

    protected final TByteArrayOutputStream response_;

    // the frame that the TTransport should wrap.
    protected final TMemoryInputTransport frameTrans_;

    // the transport that should be used to connect to clients
    protected final TTransport inTrans_;

    protected final TTransport outTrans_;

    // the input protocol to use on frames
    protected final TProtocol inProt_;

    // the output protocol to use on frames
    protected final TProtocol outProt_;

    // context associated with this connection
    protected final ServerContext context_;

    public FrameBuffer(final TNonblockingTransport trans,
        final SelectionKey selectionKey,
        final AbstractSelectThread selectThread) {
      trans_ = trans;
      selectionKey_ = selectionKey;
      selectThread_ = selectThread;
      buffer_ = ByteBuffer.allocate(4);

      frameTrans_ = new TMemoryInputTransport();
      response_ = new TByteArrayOutputStream();
      inTrans_ = inputTransportFactory_.getTransport(frameTrans_);
      outTrans_ = outputTransportFactory_.getTransport(new TIOStreamTransport(response_));
      inProt_ = inputProtocolFactory_.getProtocol(inTrans_);
      outProt_ = outputProtocolFactory_.getProtocol(outTrans_);

      if (eventHandler_ != null) {
        context_ = eventHandler_.createContext(inProt_, outProt_);
      } else {
        context_  = null;
      }
    }

    /**
     * Give this FrameBuffer a chance to read. The selector loop should have
     * received a read event for this FrameBuffer.
     *
     * @return true if the connection should live on, false if it should be
     *         closed
     */
    public boolean read() {
      if (state_ == FrameBufferState.READING_FRAME_SIZE) {
        // try to read the frame size completely
        if (!internalRead()) {
          return false;
        }

        // if the frame size has been read completely, then prepare to read the
        // actual frame.
        if (buffer_.remaining() == 0) {
          // pull out the frame size as an integer.
          int frameSize = buffer_.getInt(0);
          if (frameSize <= 0) {
            LOGGER.error("Read an invalid frame size of " + frameSize
                + ". Are you using TFramedTransport on the client side?");
            return false;
          }

          // if this frame will always be too large for this server, log the
          // error and close the connection.
          if (frameSize > MAX_READ_BUFFER_BYTES) {
            LOGGER.error("Read a frame size of " + frameSize
                + ", which is bigger than the maximum allowable buffer size for ALL connections.");
            return false;
          }

          // if this frame will push us over the memory limit, then return.
          // with luck, more memory will free up the next time around.
          if (readBufferBytesAllocated.get() + frameSize > MAX_READ_BUFFER_BYTES) {
            return true;
          }

          // increment the amount of memory allocated to read buffers
          readBufferBytesAllocated.addAndGet(frameSize + 4);

          // reallocate the readbuffer as a frame-sized buffer
          buffer_ = ByteBuffer.allocate(frameSize + 4);
          buffer_.putInt(frameSize);

          state_ = FrameBufferState.READING_FRAME;
        } else {
          // this skips the check of READING_FRAME state below, since we can't
          // possibly go on to that state if there's data left to be read at
          // this one.
          return true;
        }
      }

      // it is possible to fall through from the READING_FRAME_SIZE section
      // to READING_FRAME if there's already some frame data available once
      // READING_FRAME_SIZE is complete.

      if (state_ == FrameBufferState.READING_FRAME) {
        if (!internalRead()) {
          return false;
        }

        // since we're already in the select loop here for sure, we can just
        // modify our selection key directly.
        if (buffer_.remaining() == 0) {
          // get rid of the read select interests
          selectionKey_.interestOps(0);
          state_ = FrameBufferState.READ_FRAME_COMPLETE;
        }

        return true;
      }

      // if we fall through to this point, then the state must be invalid.
      LOGGER.error("Read was called but state is invalid (" + state_ + ")");
      return false;
    }

    /**
     * Give this FrameBuffer a chance to write its output to the final client.
     */
    public boolean write() {
      if (state_ == FrameBufferState.WRITING) {
        try {
          if (trans_.write(buffer_) < 0) {
            return false;
          }
        } catch (IOException e) {
          LOGGER.warn("Got an IOException during write!", e);
          return false;
        }

        // we're done writing. now we need to switch back to reading.
        if (buffer_.remaining() == 0) {
          prepareRead();
        }
        return true;
      }

      LOGGER.error("Write was called, but state is invalid (" + state_ + ")");
      return false;
    }

    /**
     * Give this FrameBuffer a chance to set its interest to write, once data
     * has come in.
     */
    public void changeSelectInterests() {
      if (state_ == FrameBufferState.AWAITING_REGISTER_WRITE) {
        // set the OP_WRITE interest
        selectionKey_.interestOps(SelectionKey.OP_WRITE);
        state_ = FrameBufferState.WRITING;
      } else if (state_ == FrameBufferState.AWAITING_REGISTER_READ) {
        prepareRead();
      } else if (state_ == FrameBufferState.AWAITING_CLOSE) {
        close();
        selectionKey_.cancel();
      } else {
        LOGGER.error("changeSelectInterest was called, but state is invalid (" + state_ + ")");
      }
    }

    /**
     * Shut the connection down.
     */
    public void close() {
      // if we're being closed due to an error, we might have allocated a
      // buffer that we need to subtract for our memory accounting.
      if (state_ == FrameBufferState.READING_FRAME ||
          state_ == FrameBufferState.READ_FRAME_COMPLETE ||
          state_ == FrameBufferState.AWAITING_CLOSE) {
        readBufferBytesAllocated.addAndGet(-buffer_.array().length);
      }
      trans_.close();
      if (eventHandler_ != null) {
        eventHandler_.deleteContext(context_, inProt_, outProt_);
      }
    }

    /**
     * Check if this FrameBuffer has a full frame read.
     */
    public boolean isFrameFullyRead() {
      return state_ == FrameBufferState.READ_FRAME_COMPLETE;
    }

    /**
     * After the processor has processed the invocation, whatever thread is
     * managing invocations should call this method on this FrameBuffer so we
     * know it's time to start trying to write again. Also, if it turns out that
     * there actually isn't any data in the response buffer, we'll skip trying
     * to write and instead go back to reading.
     */
    public void responseReady() {
      // the read buffer is definitely no longer in use, so we will decrement
      // our read buffer count. we do this here as well as in close because
      // we'd like to free this read memory up as quickly as possible for other
      // clients.
      readBufferBytesAllocated.addAndGet(-buffer_.array().length);

      if (response_.len() == 0) {
        // go straight to reading again. this was probably an oneway method
        state_ = FrameBufferState.AWAITING_REGISTER_READ;
        buffer_ = null;
      } else {
        buffer_ = ByteBuffer.wrap(response_.get(), 0, response_.len());

        // set state that we're waiting to be switched to write. we do this
        // asynchronously through requestSelectInterestChange() because there is
        // a possibility that we're not in the main thread, and thus currently
        // blocked in select(). (this functionality is in place for the sake of
        // the HsHa server.)
        state_ = FrameBufferState.AWAITING_REGISTER_WRITE;
      }
      requestSelectInterestChange();
    }

    /**
     * Actually invoke the method signified by this FrameBuffer.
     */
    public void invoke() {
      frameTrans_.reset(buffer_.array());
      response_.reset();

      try {
        if (eventHandler_ != null) {
          eventHandler_.processContext(context_, inTrans_, outTrans_);
        }
        processorFactory_.getProcessor(inTrans_).process(inProt_, outProt_);
        responseReady();
        return;
      } catch (TException te) {
        LOGGER.warn("Exception while invoking!", te);
      } catch (Throwable t) {
        LOGGER.error("Unexpected throwable while invoking!", t);
      }
      // This will only be reached when there is a throwable.
      state_ = FrameBufferState.AWAITING_CLOSE;
      requestSelectInterestChange();
    }

    /**
     * Perform a read into buffer.
     *
     * @return true if the read succeeded, false if there was an error or the
     *         connection closed.
     */
    private boolean internalRead() {
      try {
        if (trans_.read(buffer_) < 0) {
          return false;
        }
        return true;
      } catch (IOException e) {
        LOGGER.warn("Got an IOException in internalRead!", e);
        return false;
      }
    }

    /**
     * We're done writing, so reset our interest ops and change state
     * accordingly.
     */
    private void prepareRead() {
      // we can set our interest directly without using the queue because
      // we're in the select thread.
      selectionKey_.interestOps(SelectionKey.OP_READ);
      // get ready for another go-around
      buffer_ = ByteBuffer.allocate(4);
      state_ = FrameBufferState.READING_FRAME_SIZE;
    }

    /**
     * When this FrameBuffer needs to change its select interests and execution
     * might not be in its select thread, then this method will make sure the
     * interest change gets done when the select thread wakes back up. When the
     * current thread is this FrameBuffer's select thread, then it just does the
     * interest change immediately.
     */
    protected void requestSelectInterestChange() {
      if (Thread.currentThread() == this.selectThread_) {
        changeSelectInterests();
      } else {
        this.selectThread_.requestSelectInterestChange(this);
      }
    }
  } // FrameBuffer
```

## 小结

到这里我们可以初步的对ThriftServer端NonblockingServer的实现有整体的概念了： 

- TServer 将IO事件Accept/Read/Write等状态转换委托给SelectThread， 然后主线程等待SelectThread线程结束； 
- SelectThread对Java NIO的Selector进行封装， 进行常规的Java NIO Selector多路复用机制的循环； 
- FrameBuffer对Java NIO的SocketChannel以及ByteBuffer进行封装， 每个FrameBuffer都自己维护这状态转换， 并实时通知SelectThread， 实现对客户端请求的处理； 

具体调用逻辑如下： 

![](https://wx4.sinaimg.cn/mw690/7c35df9bly1fy9rk9bqkrj20zc0q5gp6.jpg)

### 内存管理

需要还需要注意的一点是TServer对内存的管理， 每个FrameBuffer会维护一个ByteBuffer与具体的SocketChannel关联（TNonblockingTransport）， 每次读写之后Buffer都会被释放， 但是在全局会维护一个MAX_READ_BUFFER_BYTES， 每次内存申请和释放都会更新这个值。 当总得内存占用超过这个值的时候， ThriftServer端会报错。 

默认Server端内存大小为： 256M字节。 

```java 
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
```


## 参考引用

- [1] [AbstractNonblockingServer.java](https://github.com/apache/thrift/blob/master/lib/java/src/org/apache/thrift/server/AbstractNonblockingServer.java)
- [2] [RPC-Thrift（一）](https://www.cnblogs.com/zaizhoumo/p/8184923.html)
- [3] [Thrift源码系列----5.AbstractNonblockingServer源码](https://blog.csdn.net/chen7253886/article/details/53024848)
- [4] [Class AbstractNonblockingServer](https://people.apache.org/~thejas/thrift-0.9/javadoc/org/apache/thrift/server/AbstractNonblockingServer.html)
