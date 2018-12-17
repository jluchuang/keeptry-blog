---
title: apache thrift 学习笔记： 模块概览
date: 2018-12-13 11:31:11
tags: [并发, thrift]
categories: [并发]
toc: true
description: Thrift 是一个轻量的， 语言无关的软件栈， 通过实现实现相关与语言的代码自动生成机制， 来提供跨语言的序列化/RPC框架。 这里主要在概念上对Thrift的模块划分以及协议分层做简单的梳理总结。 
---

Thrift 是一个轻量的， 语言无关的软件栈， 通过实现实现相关与语言的代码自动生成机制， 来提供跨语言的序列化/RPC框架。 这里主要在概念上对Thrift的模块划分以及协议分层做简单的梳理总结。 

## Thrift分层架构

我们可以按照下图简要描述Apache Thrift网络协议栈，

```java 
  +-------------------------------------------+
  | Server                                    |
  | (single-threaded, event-driven etc)       |
  +-------------------------------------------+
  | Processor                                 |
  | (compiler generated)                      |
  +-------------------------------------------+
  | Protocol                                  |
  | (JSON, compact etc)                       |
  +-------------------------------------------+
  | Transport                                 |
  | (raw TCP, HTTP etc)                       |
  +-------------------------------------------+
```

按官网给出的描述

- Transport层： 提供对网络IO的简单抽象， 这一层的抽象将Thrift同具体系统底层的IO实现抽离出来； 
- Protocol层： 这一层定义了Thrift服务之间通信的数据编码方式， 只要服务端和客户端协商好了通信的Protocol，就可以通过Transport层进行通信；  
- Processor层： Processor的代码是基于用户定义的业务结构接口， 通过编译器自动生成的， 这一层定义了通过Protocol层读取出的数据的处理逻辑， 具体的应用中， 用户需要自实现对自动成Processor接口； 
- Server层： 服务层将以上的三层整合起来
1. 创建一个网络连接Transport（accept / connect）
2. 定义协议通讯的编码方式（Protocol）
3. 基于定义好的Protocol实现Processor
4. 等待数据IO的到来吗， 调用Processor对数据进行处理  

Github上这张图片对于以上四层的说明更为详细： 

![](https://wx3.sinaimg.cn/mw690/7c35df9bly1fy53e3nijjj20mp0bljru.jpg)

## 各协议栈功能简述

进一步理解Thrift是如何运行的， 我们来看网络上流传最多的一张图

![](https://wx2.sinaimg.cn/mw690/7c35df9bly1fy53k67p5lj20bs0akq3b.jpg)

- **底层I/O模块**：负责实际的数据传输，可以是Socket、文件、压缩数据流等；

- **TTransport**：定义了消息怎样在Client和Server之间进行通信的，负责以字节流的方式发送和接收消息。TTransport不同的子类负责Thrift字节流(Byte Stream)数据在不同的IO模块上的传输，如：TSocket负责Socket传输，TFileTransport负责文件传输；

- **TProtocol**：定义了消息时怎样进行序列化的，即负责结构化数据（如对象、结构体等）与字节流消息的转换，对Client侧是将结构化数据组装成字节流消息，对Server端则是从字节流消息中提取结构化数据。TProtocol不同的子类对应不同的消息格式转换，如TBinaryProtocol对应字节流。

- **TServer**：负责接收客户端请求，并将请求转发给Processor。TServer各个子类实现机制不同，性能也差距很大。

- **Processor**：负责处理客户端请求并返回响应，包括RPC请求转发、参数解析、调用用户定义的代码等。Processor的代码时Thrift根据IDL文件自动生成的，用户只需根据自动生成的接口进行业务逻辑的实现就可以，Processor是Thrift框架转入用户逻辑的关键。

- **ServiceClient**：负责客户端发送RPC请求，和Processor一样，该部分的代码也是由Thrift根据IDL文件自动生成的。


## 参考引用

- [1] [apache/thrift](https://github.com/apache/thrift)
- [2] [Thrift network stack](http://thrift.apache.org/docs/concepts)
- [3] [RPC-Thrift（一）](https://www.cnblogs.com/zaizhoumo/p/8184923.html)


