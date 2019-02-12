---
title: 'Java NIO: Selector 浅析'
date: 2018-12-11 16:38:04
tags: [并发, Java]
categories:
  - [Java] 
  - [并发]
toc: true
description: 在Java中，Selector这个类是select/epoll/poll/kqueue的外包类， 在不同的平台上， 底层的实现可能有所不同， 但其基本原理是一样的。 这里结合JDK源码来简要分析 Java NIO中 Selector的实现原理。 
---

在Java中，Selector这个类是select/epoll/poll/kqueue的外包类， 在不同的平台上， 底层的实现可能有所不同， 但其基本原理是一样的。 这里结合JDK源码来简要分析 Java NIO中 Selector的实现原理。 

## 准备知识

### NIO ByteBuffer 

本质上， 缓冲区就是一个数组， 所有缓冲区都具有四个属性来提供关于其包含的数组的信息。 

```java
public abstract class Buffer {

    /**
     * The characteristics of Spliterators that traverse and split elements
     * maintained in Buffers.
     */
    static final int SPLITERATOR_CHARACTERISTICS =
        Spliterator.SIZED | Spliterator.SUBSIZED | Spliterator.ORDERED;

    // Invariants: mark <= position <= limit <= capacity
    private int mark = -1;
    private int position = 0;
    private int limit;
    private int capacity;

    ....
}
```

1. 容量（Capacity） 缓冲区能够容纳的数据元素最大数量。 容量在缓冲区创建时被设定， 并且永远不能被改变。 
2. 上界（Limit） 缓冲区中数据的总数， 代表了当前缓冲区中一共有多少数据。 
3. 位置（Position）下一个要被读或写的元素位置。 Position会自动由相应的get()和put()函数更新。 
4. 标记（Mark）一个备忘位置。用于记录上一次读写的位置。

以字节缓冲区为例， ByteBuffer是一个抽象类， 不能直接通过new语句来创建， 只能通过一个static方法allocate来创建： 

```java 
ByteBuffer byteBuffer = ByteBuffer.allocate(256);
```

以上的语句可以创建一个大小为256字节的ByteBuffer，此时，mark = -1, pos = 0, limit = 256, capacity = 256。capacity在初始化的时候确定了，运行时就不会再变化了，而另外三个变量是随着程序的执行而不断变化的。

Java的缓冲区最常用的两种状态: 

- 写数据： 一个新创建的bytebuffer， 它可以写的总数就是capacity。 
- 读数据： 如果写入了一些数据之后， 想从头开始读的话， 这时候的limit就应该是当前buytebuffer中数据的总长度。 下面这个图比较直观的说明了这个问题： 

![](https://pic3.zhimg.com/80/v2-07952bbba1ff33a30b213a4941b4242e_hd.png)

jdk封装了flip方法来实现两个状态之间的切换 ：

```java 
/**
     * Flips this buffer.  The limit is set to the current position and then
     * the position is set to zero.  If the mark is defined then it is
     * discarded.
     *
     * <p> After a sequence of channel-read or <i>put</i> operations, invoke
     * this method to prepare for a sequence of channel-write or relative
     * <i>get</i> operations.  For example:
     *
     * <blockquote><pre>
     * buf.put(magic);    // Prepend header
     * in.read(buf);      // Read data into rest of buffer
     * buf.flip();        // Flip buffer
     * out.write(buf);    // Write header + data to channel</pre></blockquote>
     *
     * <p> This method is often used in conjunction with the {@link
     * java.nio.ByteBuffer#compact compact} method when transferring data from
     * one place to another.  </p>
     *
     * @return  This buffer
     */
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
```

### NIO SocketChannel

NIO中通过channel封装了对数据源的操作， 通过channel可以操作数据源， 但又不必关心数据源的具体无力结构。 

这个数据源可能是多种的， 比如可以是文件， 也可以是socket, 在大多数应用中， channel 与文件描述符或者socket是一一对应的。 

在Java IO中， 基本上可以分为文件类和Stream类两大类。 Channel 也相应地分为了FileChannel和Socket Channel， 其中socket channel又分为三大类， 一个是用于端口监听的ServerSocketChannel, 第三类是用于TCP通信的SocketChannel， 第三类是用于UDP通信的DatagramChannel。 

Channel可以使用ByteBuffer进行读写， 这是它的一个方便之处。 SocketChannel本质上是对socket的一种封装。 

下面是一个SocketChannel的Server 和Client端的代码示例

**Server**

```java
public class WebServer {
    public static void main(String args[]) {
        try {
            ServerSocketChannel ssc = ServerSocketChannel.open();
            ssc.socket().bind(new InetSocketAddress("127.0.0.1", 8000));
            SocketChannel socketChannel = ssc.accept();

            ByteBuffer readBuffer = ByteBuffer.allocate(128);
            socketChannel.read(readBuffer);

            readBuffer.flip();
            while (readBuffer.hasRemaining()) {
                System.out.println((char)readBuffer.get());
            }

            socketChannel.close();
            ssc.close();
        }
        catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**Client**

```java
public class WebClient {
    public static void main(String[] args) {
        SocketChannel socketChannel = null;
        try {
            socketChannel = SocketChannel.open();
            socketChannel.connect(new InetSocketAddress("127.0.0.1", 8000));

            ByteBuffer writeBuffer = ByteBuffer.allocate(128);
            writeBuffer.put("hello world".getBytes());

            writeBuffer.flip();
            socketChannel.write(writeBuffer);
            socketChannel.close();
        } catch (IOException e) {
        }
    }
}
```

**Scatter / Gather**

Channel 提供了一种被称为 Scatter/Gather 的新功能，也称为本地矢量 I/O。Scatter/Gather 是指在多个缓冲区上实现一个简单的 I/O 操作。对于一个 write 操作而言，数据是从几个缓冲区（通常就是一个缓冲区数组）按顺序抽取（称为 gather）并使用 channel 发送出去。缓冲区本身并不需要具备这种 gather 的能力。gather 过程等效于全部缓冲区的内容被连结起来，并在发送数据前存放到一个大的缓冲区中。对于 read 操作而言，从 通道读取的数据会按顺序被散布（称为 scatter）到多个缓冲区，将每个缓冲区填满直至通道中的数据或者缓冲区的最大空间被消耗完。


大多数现代操作系统都支持本地矢量 I/O（native vectored I/O）。我们在一个通道上发起一个 Scatter/Gather 操作时，该请求会被翻译为适当的本地调用来直接填充或抽取缓冲区。这是一个很大的进步，因为减少或避免了缓冲区拷贝和系统调用。Scatter/Gather 应该使用直接的 ByteBuffers 以从本地 I/O 获取最大性能优势。

客户端Gather 的例子: 

```java
public class WebClient {
    public static void main(String[] args) {
        SocketChannel socketChannel = null;
        try {
            socketChannel = SocketChannel.open();
            socketChannel.connect(new InetSocketAddress("127.0.0.1", 8000));

            ByteBuffer writeBuffer = ByteBuffer.allocate(128);
            ByteBuffer buffer2 = ByteBuffer.allocate(16);
            writeBuffer.put("hello ".getBytes());
            buffer2.put("world".getBytes());

            writeBuffer.flip();
            buffer2.flip();
            ByteBuffer[] bufferArray = {writeBuffer, buffer2};
            socketChannel.write(bufferArray);
            socketChannel.close();
        } catch (IOException e) {
        }
    }
}
```

## Selector

### why use a selector

通过Selector， 我们可以通过一个线程来管理多个channel， 要知道线程上下文切换的代价对于操作系统来说是十分昂贵的， 而且每个Thread 都需要占用内存空间。 有了上一篇[poll、epool小结](http://www.keeptry.cn/2018/12/10/poll%E3%80%81epoll%E5%B0%8F%E7%BB%93/)的基础， 我们可以这样来理解： Channel 是Java 对于系统IO的封装， Selector是Java 对于底层select/poll/epoll/kqueue的封装。 

### 属性

- keys: 所有注册到Selector的Channel表示的SelectionKey都会存在于该集合中。 keys元素的添加会在Channel注册到Selector时发生。 
- selectedKeys： 该集合中的每个SelectionKey都是其对应的Channel在上一次操作selection期间被检查到至少有一种SelectionKey中所感兴趣的操作已经准备好被处理。 该集合是Keys的一个子集。 
- cancelledKeys： 执行了取消操作的SelectionKey会被放到该集合中， 该集合是keys的一个子集。 

*需要注意的一点是： 只有继承自SelectableChannel的子类对象才能被注册到Selector中， 这些子类对象都是可以被设置为non-blocking模式的。 *

### 使用模式

我们首先给出一个最基本的Selector使用流程： 

```java
    ServerSocketChannel serverChannel = ServerSocketChannel.open();
    serverChannel.configureBlocking(false);
    int port = 8081;          
    serverChannel.socket().bind(new InetSocketAddress(port));
    Selector selector = Selector.open();
    serverChannel.register(selector, SelectionKey.OP_ACCEPT);
    while(true){
        int n = selector.select();
        if(n > 0) {
            Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
            while (iter.hasNext()) {
                SelectionKey selectionKey = iter.next();
                ......
                iter.remove();
            }
        }
    }
```

说明： 

1. 启动一个ServerSocketChannel在8081端口进行监听； 
2. 将ServerSocketChannel注册到Selector， 等待新的连接到来； 
3. 调用Select.select方法来处理就绪的IO事件； 
4. 省略部分主要是对SelectionKey.OP_READ、SelectionKey.OP_WRITE、SelectionKey.OP_ACCEPT事件的处理； 
5. 将处理完毕的IO事件对应的SelectionKey从selectKeys集合中删除； 

### 实现原理

请参见: 参考引用[5](http://www.importnew.com/26258.html)
这里注明一句： **在Linux系统中JDK NIO使用的是 LT ，而Netty epoll使用的是 ET。**

## 总结

经过两天总结， 算是初步的对Java NIO的IO多路复用做了比较详细的总结。 这是Java Server端处理高并发网络请求的基础， 接下来准备梳理一下Apache Thrift源码， 希望能对这几个月的工作和学习有个较好的积累。

## 参考引用 

- [1] [https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html)
- [2] [nio(1)：buffer](https://zhuanlan.zhihu.com/p/27296046)
- [3] [nio(2):channel](https://zhuanlan.zhihu.com/p/27365009)
- [4] [Introduction to the Java NIO Selector](https://www.baeldung.com/java-nio-selector)
- [5] [Selector 实现原理](http://www.importnew.com/26258.html)
- [6] [Java NIO(6): Selector](https://zhuanlan.zhihu.com/p/27434028)
