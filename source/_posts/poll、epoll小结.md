---
title: 'poll、epoll小结 '
date: 2018-12-10 09:46:49
tags: [并发]
categories: [并发]
toc: true
description: poll和epoll是linux高并发服务的基础，linux 系统下jdk的NIO底层也是基于epoll, 这里简单对从原理上对poll和epoll做简单总结， 并针对针对这两种IO模式给出简单的demo示例。 
---

poll和epoll是linux高并发服务的基础，linux 系统下jdk的NIO底层也是基于epoll, 这里简单对从原理上对poll和epoll做简单总结， 并针对针对这两种IO模式给出简单的demo示例。 

## 准备知识

### 用户空间与内核空间

现在操作系统都是采用虚拟存储器， 那么对于32为操作系统而言， 它的寻址空间（虚拟存储空间）为4G（2的32次方）。操作系统的核心是内核， 独立于普通的应用程序， 可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核（kernel）， 保证内核的安全， 操作系统将虚拟空间划分为两部分， 一部分为内核空间， 一部分为用户空间， 针对linux操作系统而言， 将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF）， 供内核使用， 称为内核空间， 而将较低的3G字节（从虚拟地址0x00000000到0xBFFFFFFF）， 供各个进程使用， 称为用户空间。

### 缓存IO

缓存I/O又被成为标准I/O， 大多数文件系统默认的I/O操作都是缓存I/O。 在Linux的缓存I/O机制中， 操作系统会将I/O的数据缓存在文件系统的页缓存（page cache）中， 也就是说， 数据会先被拷贝到操作系统内核的缓冲区中， 然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。 

**缓存I/O的缺点： **数据在传输过程中需要在应用程序地址空间和内核进行多次数据拷贝操作， 这些数据拷贝操作所带来的CPU以及内存开销是非常大的。 

### I/O模型

对于一次I/O访问(以read为例)， 数据会先拷贝到操作系统的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序地址空间。 所以说， 当一个read操作发生时， 它会经历两个阶段： 

1. 等待数据准备（Waiting for the data to be ready）
2. 将数据从内核拷贝到进程中(Copying the data from the kernel to the process)

正是因为这两个阶段， Linux系统产生了一下五种网络模式的方案： 

- 阻塞I/O（Blocking IO）
- 非阻塞I/O (Nonblocking IO)
- I/O多路复用 （IO multiplexing）
- 信号驱动I/O (Signal driven IO)
- 异步 I/O (Asynchronous IO)

#### 阻塞I/O 

在Linux中， 默认情况下所有的Socket都是blocking， 一个典型的读操作流程大概是这样：

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fy2lqw2nwgj20fc0970u0.jpg)

当用户进程调用了recvfrom这个系统调用， kernel就开始了IO的第一个阶段：准备数据（对于网络IO来说， 很多时候数据在一开始还没到达。比如，还没有收到一个完整的UDP包。这个时候Kernel就要等待足够的数据到来）。这个过程需要等待，也就是说数据拷贝到操作系统内核的缓冲区中是需要一个过程的。 而在用户进程这边， 整个进程会被阻塞（当然是进程自己选择阻塞）。当kernel一直等到数据准备好了， 它就会将数据从kernel中拷贝到用户内存， 然后kernel返回结果用户进程才解除block的状态， 重新运行起来。 

*Notice: blocking IO 的特点就是在IO执行的两个阶段都被block。 *

#### 非阻塞I/O

Linux 下， 可以通过设置socket使其变为non-blocking. 当对一个non-blocking socket 执行读操作时，流程是这个样子： 

![](https://wx4.sinaimg.cn/mw690/7c35df9bly1fy2lqw3913j20gr0990ur.jpg)

当用户进程发出read操作时， 如果kernel中的数据还没有准备好， 那么它并不会block用户进程， 而是立刻返回一个error。 从用户进程角度讲， 它发起一个read操作后， 并不需要等待， 而是马上就得到了一个结果。 用户进程判断结果是一个error时， 它就知道数据还没准备好， 于是它可以再次发送read操作。 一旦kernel中的数据准备好了， 并且又要再次收到了用户进程的system call时， 那么它马上就将数据拷贝到了内存， 然后返回。 

*所以， nonblocking IO的特点是用户进程需要不断的主动询问kernel数据好了没有。 *

#### I/O 多路复用（IO multiplexing）

IO multiplexing就是我们说的select， poll， epoll， 有些地方也称这种IO方式为event driven IO。 select/epoll好处就在于单个process 可以同时处理多个网络连接的I/O。 它的基本原理就是select, poll, epoll 这个function会不断的轮询所负责的所有socket， 当某个socket有数据到达了， 就通知用户进程。 

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fy2m6seic0j20gx092gnf.jpg)

当用户进程调用了select， 那么整个进程就会被block， 而同时， kernel会”监视“所有select负责的socket， 当任何一个socket中的数据准备好了， select 就会返回。 这个时候用户再次调用read操作， 将数据从kernel拷贝到用户进程。 

*所以， I/O 多路复用的特点是通过一种机制一个进程能同时等待多个文件描述符， 而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态， select（）函数就可以返回。 *

这个图和blocking IO 的图其实并没有太大的不同， 事实上， 还更差一些。 因为这里需要使用两个system call（select 和recvfrom）， 而blocking IO只调用一个system call（recvfrom）。 但是， 用select 的优势在于它可以同时处理多个connection。 

所以，如果处理的连接数不是很高的话， 使用select/epoll 的web server 不一定比使用multi-threading + blocking IO的 web server性能好， 可能延时更大。 select/epoll 的优势并不是对于单个连接能处理得更快， 而是在于能处理更多的连接。 

在IO multiplexing Model 中， 实际中， 对于每一个socket，一般都设置成non-blocking, 但是， 如上图所示， 整个用户的process其实是一直是被block的。 只不过process是被select这个函数block， 而不是被socket IO给block。 

#### 异步 I/O（Asynchronous IO）

Linux 下的asynchronous IO 其实用的很少。 先看一下它的流程：

![](https://wx2.sinaimg.cn/mw690/7c35df9bly1fy2m6sbyvuj20fw0903zx.jpg)

用户进程发起read操作之后， 立刻就可以开始去做其它的事。 而另一方面， 从kernel的角度， 当它受到一个asynchronous read之后， 首先它会立刻返回， 所以不会对用进程产生任何block。 然后kernel会等待数据准备完成， 然后将数据拷贝到用户内存， 当这一切完成之后， kernel会给用户进程发送一个signal， 告诉它read完成了。 

#### blocking 和 non-blocking 的区别

调用 blocking IO会一直block 住对应的进程直到操作完成， 而non-blocking IO在kernel还准备数据的情况下立刻返回。 

#### synchronous IO 和 asynchronous IO的区别

在说明synchronous IO和asynchronous IO的区别之前，需要先给出两者的定义。POSIX的定义是这样子的：

> A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes;
> An asynchronous I/O operation does not cause the requesting process to be blocked;

两者的区别就在于synchronous IO做 ”IO operation“的时候将会阻塞process。 按照这个定义， 之前所述的blocking IO, non-blocking IO, IO multiplexing 都属于 synchronous IO。 

有人会说，non-blocking IO并没有被block啊。这里有个非常“狡猾”的地方，定义中所指的”IO operation”是指真实的IO操作，就是例子中的recvfrom这个system call。non-blocking IO在执行recvfrom这个system call的时候，如果kernel的数据没有准备好，这时候不会block进程。但是，当kernel中数据准备好的时候，recvfrom会将数据从kernel拷贝到用户内存中，这个时候进程是被block了，在这段时间内，进程是被block的。

而asynchronous IO则不一样，当进程发起IO 操作之后，就直接返回再也不理睬了，直到kernel发送一个信号，告诉进程说IO完成。在这整个过程中，进程完全没有被block。

#### 各IO 模型的比较

![](https://wx4.sinaimg.cn/mw690/7c35df9bly1fy2m6sceeaj20h2093gnt.jpg)

通过上面的图片， 可以发现non-blocking IO 和 asynchronous IO 的区别还是很明显的。 在non-blocking IO 中， 虽然进程大部分时间都不会被block, 但是它仍然要求进程主动去check， 并且当数据准备完成后， 也需要进程主动再次调用recvfrom来将数据拷贝到用户内存。 而asynchronous IO则完全不同。 它就像是用户进程将整个IO操作交给了内核完成， 然后内核做完后发信号通知。 在此期间， 用户进程不需要去检查IO操作状态， 也不需要主动去拷贝数据。 

## select && poll 

select、poll、epoll都是IO多路复用的机制。 I/O 多路复用就是通过一种机制， 一个进程可以监视多个描述符， 一旦某个描述符就绪（一般是读就绪或者写就绪）， 能够通知用户应用程序进行相应读写操作。 但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

### select函数原型

```c
/* According to POSIX.1-2001, POSIX.1-2008 */
#include <sys/select.h>

/* According to earlier standards */
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int select(int nfds, fd_set *readfds, fd_set *writefds,
            fd_set *exceptfds, struct timeval *timeout);
```

select 函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，直到有描述副就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以 通过遍历fdset，来找到就绪的描述符。

select目前几乎在所有的平台上支持，其良好跨平台支持也是它的一个优点。select的一 个缺点在于单个进程能够监视的文件描述符的数量存在最大限制，在Linux上一般为1024，可以通过修改宏定义甚至重新编译内核的方式提升这一限制，但 是这样也会造成效率的降低。

### poll 函数原型

```c
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

不同于select 使用三个位图来表示三个fdset的方式， poll使用一个pollfd的指针实现。 

```c
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```

pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。 和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符。

- *从上面看，select和poll都需要在返回后，通过遍历文件描述符来获取已经就绪的socket。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降。*

### poll code example

[poll.c](https://github.com/jluchuang/concurrent-fun/blob/master/c_code/poll.c)

## epoll 

拿select模型为例，假设我们的服务器需要支持100万的并发连接，则在__FD_SETSIZE 为1024的情况下，则我们至少需要开辟1k个进程才能实现100万的并发连接。除了进程间上下文切换的时间消耗外，从内核/用户空间大量的无脑内存拷贝、数组轮询等，是系统难以承受的。因此，基于select模型的服务器程序，要达到10万级别的并发访问，是一个很难完成的任务。

由于epoll 的实现机制与select 、 poll的机制完全不同， 上面所说的select的缺点在epoll上不复存在。 
设想一下如下场景： 有100万个客户端同时与一个服务器进程保持着TCP连接。而每一时刻， 通常只有几百上千个TCP连接是活跃的(事实上大部分场景都是这种情况)。 如何实现这样的高并发？

在select/poll时代， 服务器进程每次都把这100万个连接告诉操作系统(从用户态复制句柄数据结构到内核态)， 让操作系统内核去查询这些套接字上是否有事件发生， 轮询完后， 再将句柄数据复制到用户态， 让服务器应用程序轮询处理已发生的网络事件， 这一过程资源消耗较大， 因此， select/poll一般只能处理几千的并发连接。

epoll的设计和实现与select完全不同。 epoll通过在Linux内核中申请一个简易的文件系统(文件系统一般用什么数据结构实现？B+树)。把原先的select/poll调用分成了3个部分：

1. 调用epoll_create()建立一个epoll对象(在epoll文件系统中为这个句柄对象分配资源)

2. 调用epoll_ctl向epoll对象中添加这100万个连接的套接字

3. 调用epoll_wait收集发生的事件的连接

如此一来， 要实现上面说是的场景， 只需要在进程启动时建立一个epoll对象， 然后在需要的时候向这个epoll对象中添加或者删除连接。 同时， epoll_wait的效率也非常高， 因为调用epoll_wait时， 并没有一股脑的向操作系统复制这100万个连接的句柄数据， 内核也不需要去遍历全部的连接。

### epoll 实现原理 

当某一进程调用epoll_create方法时， Linux内核会创建一个eventpoll结构体， 这个结构体中有两个成员与epoll的使用方式密切相关。 eventpoll结构体如下所示：

```c
struct eventpoll{
    ....
    /*红黑树的根节点，这颗树中存储着所有添加到epoll中的需要监控的事件*/
    struct rb_root  rbr;
    /*双链表中则存放着将要通过epoll_wait返回给用户的满足条件的事件*/
    struct list_head rdlist;
    ....
};
```

每一个epoll对象都有一个独立的eventpoll结构体， 用于存放通过epoll_ctl方法向epoll对象中添加进来的事件。 这些事件都会挂载在红黑树中， 如此， 重复添加的事件就可以通过红黑树而高效的识别出来(红黑树的插入时间效率是lgn， 其中n为树的高度)。

而所有添加到epoll中的事件都会与设备(网卡)驱动程序建立回调关系， 也就是说， 当相应的事件发生时会调用这个回调方法。 这个回调方法在内核中叫ep_poll_callback, 它会将发生的事件添加到rdlist双链表中。

在epoll中，对于每一个事件，都会建立一个epitem结构体，如下所示：

```c
struct epitem{
    struct rb_node  rbn;//红黑树节点
    struct list_head    rdllink;//双向链表节点
    struct epoll_filefd  ffd;  //事件句柄信息
    struct eventpoll *ep;    //指向其所属的eventpoll对象
    struct epoll_event event; //期待发生的事件类型
}
```

当调用epoll_wait检查是否有事件发生时， 只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可。 如果rdlist不为空， 则把发生的事件复制到用户态， 同时将事件数量返回给用户。

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1fy2vc9j2tgj20f20clq55.jpg)

**从上面讲解可知： 通过红黑树和双链表数据结构， 并结合回调机制， 造就了epoll的高效。** 

OK，讲解完了Epoll的机理，我们便能很容易掌握epoll的用法了。一句话描述就是：三步曲。

第一步：epoll_create()系统调用。此调用返回一个句柄，之后所有的使用都依靠这个句柄来标识。

第二步：epoll_ctl()系统调用。通过此调用向epoll对象中添加、删除、修改感兴趣的事件，返回0标识成功，返回-1表示失败。

第三部：epoll_wait()系统调用。通过此调用收集收集在epoll监控中已经发生的事件。

### 工作模式 

epoll对文件描述符的操作有两种模式：LT（level trigger）和ET（edge trigger）。LT模式是默认模式，LT模式与ET模式的区别如下：

- LT模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。
- ET模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

[http://www.man7.org/linux/man-pages/man7/epoll.7.html]（http://www.man7.org/linux/man-pages/man7/epoll.7.html）给出了一个用来区分两种工作模式的例子， 具体翻译如下： 

假如有这样一个例子：
1. 我们已经把一个用来从管道中读取数据的文件句柄(RFD)添加到epoll描述符
2. 这个时候从管道的另一端被写入了2KB的数据
3. 调用epoll_wait(2)，并且它会返回RFD，说明它已经准备好读取操作
4. 然后我们读取了1KB的数据
5. 调用epoll_wait(2)......

LT模式：
如果是LT模式，那么在第5步调用epoll_wait(2)之后，仍然能受到通知。

ET模式：
如果我们在第1步将RFD添加到epoll描述符的时候使用了EPOLLET标志，那么在第5步调用epoll_wait(2)之后将有可能会挂起，因为剩余的数据还存在于文件的输入缓冲区内，而且数据发出端还在等待一个针对已经发出数据的反馈信息。只有在监视的文件句柄上发生了某个事件的时候 ET 工作模式才会汇报事件。因此在第5步的时候，调用者可能会放弃等待仍在存在于文件输入缓冲区内的剩余数据。


*总结： ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。*

### epoll code example

- 参见 [高并发网络编程之epoll详解](http://www.open-open.com/lib/view/open1410403215664.html)
- [epoll.c](https://github.com/jluchuang/concurrent-fun/blob/master/c_code/epoll.c)

## 参考引用 

- [1] [http://www.man7.org/index.html](http://www.man7.org/index.html)
- [2] [Unix Network programming](https://www.amazon.com/UNIX-Network-Programming-Richard-Stevens/dp/0139498761)
- [3] [高并发网络编程之epoll详解](http://www.open-open.com/lib/view/open1410403215664.html)
- [4] [Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)
- [5] [Java NIO(5): IO多路复用](https://zhuanlan.zhihu.com/p/27419141)
- [6] [大话 Select、Poll、Epoll](https://cloud.tencent.com/developer/article/1005481)
