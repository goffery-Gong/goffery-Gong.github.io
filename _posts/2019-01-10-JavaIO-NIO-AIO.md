---
layout:     post
title:      JavaIO-NIO-AIO
subtitle:   
date:       2019-03-06
author:     Gong
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - 网络编程
---
在通信编程中，涉及到三个层次的内容：

- 语言层面的IO
- 操作系统层面IO
- IO操作在计算机网络的实现

在下文，我们首先介绍关于IO的基础知识（数据单位；Socket与TCP/UDP；Socket与操作系统）；之后我们从宏观角度介绍在操作系统层面的五种IO模型和java语言层面的IO架构（主要是传统的IO包）；然后分别介绍操作系统层面的五种IO模型中（BIO，NIO，IO复用）的具体原理。

## 一、基本知识

### 1.数据单位

bit：位（二进制的一个“0”或一个“1”叫一位。）

byte： 1字节=8bit（它是一个8位的二进制数，是一个很具体的存储空间）

char： 字符

ASCII码：ASCII码中，一个英文字母（不分大小写）占一个字节的空间，一个中文汉字占两个字节的空间。GB2312 是对 ASCII 的中文扩展。

Unicode字符集：常用的是用**两个字节**（16位）表示一个字符（如果要用到非常偏僻的字符，就需要4个字节）。

UTF-8编码规则：一种变长的编码方式：它可以使用1~4个字节表示一个符号，根据不同的符号而变化字节长度。当字符在ASCII码的范围时，就用一个字节表示，保留了ASCII字符一个字节的编码做为它的一部分，如此一来UTF-8编码也可以是为视为一种对ASCII码的拓展。值得注意的是unicode编码中一个中文字符占2个字节，而UTF-8一个中文字符占3个字节。从unicode到uft-8并不是直接的对应，而是要过一些算法和规则来转换。

**UTF-8**就是每次8个位传输数据，而**UTF-16**就是每次16个位。UTF-8就是在互联网上使用最广的一种unicode的实现方式，这是为传输而设计的编码。

<u>在计算机内存中，统一使用Unicode编码，当需要保存到硬盘或者需要传输的时候，就转换为UTF-8编码。</u>

### 2.Socket

创建socket的语句时，操作系统会创建一个由文件系统管理的socket对象（如下图）。这个socket对象包含了发送缓冲区、接收缓冲区、等待队列等成员。等待队列是个非常重要的结构，它指向所有需要等待该socket事件的进程。

### 3.Socket与TCP通信

#### TCP与UDP的socket缓冲区

首先，对于 TCP 通信来说，每个 TCP Socket 在内核中都有一个**发送缓冲区**和一个**接收缓冲区**，TCP 的全双工的工作模式及 TCP 的滑动窗口便依赖于这两个独立的 Buffer 及此 Buffer 的填充状态。

接收缓冲区把数据缓存入内核，若应用进程一直没有调用 Socket 的 read 方法进行读取的话，则此数据会一直被缓存在接收缓冲区内。不管进程是否读取 Socket，对端发来的数据都会经由内核接收并且缓存到 Socket 的内核接收缓冲区中。read 所做的工作，就是把内核接收缓冲区中的数据复制到应用层用户的 Buffer 里面，仅此而已。

进程调用 Socket 的 send 发送数据的时候，最简单的情况（也是一般情况）是将数据从应用层用户的 Buffer 里复制到 Socket 的内核发送缓冲区中，然后 send 便会在上层返回。换句话说，send 返回时，数据不一定会被发送到对端（和 write写文件有点类似），send 仅仅是把应用层 Buffer 的数据复制到 Socket 的内核发送 Buffer 中。

**而对于 UDP 通信来说，每个 UDP Socket 都有一个接收缓冲区，而没有发送缓冲区**，从概念上来说就是只要有数据就发，不管对方是否可以正确接收，所以不缓冲，不需要发送缓冲区。

#### 流量控制

我们来说说 TCP/IP 的滑动窗口和流量控制机制，前面我们提到，Socket 的接收缓冲区被 TCP 和 UDP 用来缓存网络上收到的数据，一直保存到应用进程读走为止。

- 对于 TCP 来说，如果应用进程一直没有读取，则 Buffer 满了之后，发生的动作是：通知对端 TCP 协议中的窗口关闭，保证 TCP 套接口接收缓冲区不会溢出，保证了 TCP 是可靠传输的，这个便是滑动窗口的实现。因为对方不允许发出超过通告窗口大小的数据，所以如果对方无视窗口大小而发出了超过窗口大小的数据，则接收方 TCP 将丢弃它，这就是 TCP 的流量控制原理。
- 对于 UDP 来说，当接收方的 Socket 接收缓冲区满时，新来的数据报无法进入接收缓冲区，此数据报就会被丢弃，UDP 是没有流量控制的，快的发送者可以很容易地淹没慢的接收者，导致接收方的 UDP丢弃数据报。

### 4.Socket与Linux操作系统

在Linux世界，“一切皆文件”，操作系统把网络读写作为IO操作，就像读写文件那样，对外提供出来的编程接口就是Socket。所以，socket（套接字）是通信的基石，是支持TCP/IP协议网络通信的基本操作单元。socket实质上提供了进程通信的端点。进程通信之前，双方首先必须各自创建一个端点，否则是没有办法建立联系并相互通信的。一个完整的socket有一个本地唯一的socket号，这是由操作系统分配的。

从设计模式的角度看， Socket其实是一个外观模式，它把复杂的TCP/IP协议栈隐藏在Socket接口后面，对用户来说，一组简单的Socket接口就是全部。当应用程序创建一个socket时，操作系统就返回一个整数作为描述符（descriptor）来标识这个套接字。然后，应用程序以该描述符为传递参数，通过调用函数来完成某种操作（例如通过网络传送数据或接收输入的数据）。以TCP 为例，典型的Socket 使用如下： 

![](http://ww2.sinaimg.cn/large/006tNc79ly1g5omrfmeqwj30da0dnjsf.jpg)

在许多操作系统中，Socket描述符和其他I/O描述符是集成在一起的，操作系统把socket描述符实现为一个指针数组，这些指针指向内部数据结构。进一步看，操作系统为每个运行的进程维护一张单独的文件描述符表。当进程打开一个文件时，系统把一个指向此文件内部数据结构的指针写入文件描述符表，并把该表的索引值返回给调用者。

既然Socket和操作系统的IO操作相关，那么各操作系统IO实现上的差异会导致Socket编程上的些许不同。看看我Mac上的Socket.so 会发现和CentOS上的还是些不同的。 

进程进行Socket操作时，也有着多种处理方式，如阻塞式IO，非阻塞式IO，多路复用(select/poll/epoll)，AIO等等。下一节将介绍操作系统的五种IO模型。

**[ps.系统传统的IO调用过程](https://blog.csdn.net/linxdcn/article/details/72903422) **

以下是操作系统一般情况下的IO调用。在传统的文件IO操作中，我们都是调用操作系统提供的底层标准IO系统调用函数 read()、write() ，此时调用此函数的进程（在JAVA中即java进程）由当前的**用户态切换到内核态**，然后OS的内核代码负责将相应的文件数据读取到内核的IO缓冲区，然后再把数据从内核IO缓冲区拷贝到进程的私有地址空间中去，这样便完成了一次IO操作。如下图所示。

![img](https://img-blog.csdn.net/20170607224313512?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGlueGRjbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 二、操作系统的五种IO模型

### JavaIO与操作系统IO的联系

- 在Java中，主要有三种IO模型，分别是**阻塞IO（BIO）、非阻塞IO（NIO）和 异步IO（AIO）**。
- 在Linux(UNIX)操作系统中，共有五种IO模型，分别是：**阻塞IO模型**、**非阻塞IO模型**、**IO复用模型**、**信号驱动IO模型**以及**异步IO模型**。

**Java中提供的IO有关的API，在文件处理的时候，其实依赖操作系统层面的IO操作实现的。**比如在Linux 2.6以后，Java中NIO和AIO都是通过epoll来实现的，而在Windows上，AIO是通过IOCP来实现的。

可以把Java中的BIO、NIO和AIO理解为是Java语言对操作系统的各种IO模型的封装。程序员在使用这些API的时候，不需要关心操作系统层面的知识，也不需要根据不同操作系统编写不同的代码。只需要使用Java的API就可以了。

一次IO过程：文件从硬盘中拷贝到用户空间中，中间过渡的空间映射成内核空间。

### **阻塞IO模型**

阻塞 I/O 是最简单的 I/O 模型，一般表现为进程或线程等待某个条件，如果条件不满足，则一直等下去。条件满足，则进行下一步操作。

![](http://ww2.sinaimg.cn/large/006tNc79ly1g5omsl5dsrj30h909e0tb.jpg)

应用进程通过系统调用 `recvfrom` 接收数据，但由于内核还未准备好数据报，应用进程就会阻塞住，直到内核准备好数据报，`recvfrom` 完成数据报复制工作，应用进程才能结束阻塞状态。

这种钓鱼方式相对来说比较简单，对于钓鱼的人来说，不需要什么特制的鱼竿，拿一根够长的木棍就可以悠闲的开始钓鱼了（实现简单）。缺点就是比较耗费时间，比较适合那种对鱼的需求量小的情况（并发低，时效性要求低）。

### **非阻塞IO模型** 

我们钓鱼的时候，在等待鱼儿咬钩的过程中，我们可以做点别的事情，比如玩一把王者荣耀、看一集《延禧攻略》等等。但是，我们要时不时的去看一下鱼竿，一旦发现有鱼儿上钩了，就把鱼钓上来。

映射到Linux操作系统中，这就是非阻塞的IO模型。应用进程与内核交互，目的未达到之前，不再一味的等着，而是直接返回。然后通过轮询的方式，不停的去问内核数据准备有没有准备好。如果某一次轮询发现数据已经准备好了，那就把数据拷贝到用户空间中。

![](http://ww3.sinaimg.cn/large/006tNc79ly1g5omt3r1ygj30ji0aeab4.jpg)

应用进程通过 `recvfrom` 调用不停的去和内核交互，直到内核准备好数据。如果没有准备好，内核会返回`error`，应用进程在得到`error`后，过一段时间再发送`recvfrom`请求。在两次发送请求的时间段，进程可以先做别的事情。

### **信号驱动IO模型** 

我们钓鱼的时候，为了避免自己一遍一遍的去查看鱼竿，我们可以给鱼竿安装一个报警器。当有鱼儿咬钩的时候立刻报警。然后我们再收到报警后，去把鱼钓起来。

映射到Linux操作系统中，这就是信号驱动IO。应用进程在读取文件时通知内核，如果某个 socket 的某个事件发生时，请向我发一个信号。在收到信号后，信号对应的处理函数会进行后续处理。

![](http://ww3.sinaimg.cn/large/006tNc79ly1g5omtj989uj30jq0avt9l.jpg)

应用进程预先向内核注册一个信号处理函数，然后用户进程返回，并且不阻塞，当内核数据准备就绪时会发送一个信号给进程，用户进程便在信号处理函数中开始把数据拷贝的用户空间中。

### **IO复用模型** 

我们钓鱼的时候，为了保证可以最短的时间钓到最多的鱼，我们同一时间摆放多个鱼竿，同时钓鱼。然后哪个鱼竿有鱼儿咬钩了，我们就把哪个鱼竿上面的鱼钓起来。

映射到Linux操作系统中，这就是IO复用模型。多个进程的IO可以注册到同一个管道上，这个管道会统一和内核进行交互。**当管道中的某一个请求需要的数据准备好之后，进程再把对应的数据拷贝到用户空间中。**

![](http://ww1.sinaimg.cn/large/006tNc79ly1g5omultsn1j30jq0adjsc.jpg)

IO多路转接是多了一个`select`函数，多个进程的IO可以注册到同一个`select`上，当用户进程调用该`select`，`select`会监听所有注册好的IO，如果所有被监听的IO需要的数据都没有准备好时，`select`调用进程会阻塞。当任意一个IO所需的数据准备好之后，`select`调用就会返回，然后进程在通过`recvfrom`来进行数据拷贝。

**这里的IO复用模型，并没有向内核注册信号处理函数，所以，他并不是非阻塞的。**进程在发出`select`后，要等到`select`监听的所有IO操作中至少有一个需要的数据准备好，才会有返回，并且也需要再次发送请求去进行文件的拷贝。

### 同步IO模型

我们说阻塞IO模型、非阻塞IO模型、IO复用模型和信号驱动IO模型都是同步的IO模型。原因是因为，无论以上那种模型，真正的数据拷贝过程，都是同步进行的。

**信号驱动难道不是异步的么？** 信号驱动，内核是在数据准备好之后通知进程，然后进程再通过`recvfrom`操作进行数据拷贝。我们可以认为**数据准备阶段是异步的**，但是，**数据拷贝操作是同步的**。所以，整个IO过程也不能认为是异步的。

### **异步IO模型** 

我们钓鱼的时候，采用一种高科技钓鱼竿，即全自动钓鱼竿。可以自动感应鱼上钩，自动收竿，更厉害的可以自动把鱼放进鱼篓里。然后，通知我们鱼已经钓到了，他就继续去钓下一条鱼去了。

映射到Linux操作系统中，这就是异步IO模型。应用进程把IO请求传给内核后，完全由内核去操作文件拷贝。内核完成相关操作后，会发信号告诉应用进程本次IO已经完成。

![](http://ww3.sinaimg.cn/large/006tNc79ly1g5omuzoz45j30jd0bogma.jpg)

用户进程发起`aio_read`操作之后，给内核传递描述符、缓冲区指针、缓冲区大小等，告诉内核当整个操作完成时，如何通知进程，然后就立刻去做其他事情了。当内核收到`aio_read`后，会立刻返回，然后内核开始等待数据准备，数据准备好以后，直接把数据拷贝到用户控件，然后再通知进程本次IO已经完成。

### **5种IO模型对比**

![emma_1](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2016/77752ed5.jpg)

## 三、Java 的 I/O 类库的基本架构

Java 的 I/O 操作类在包 java.io 下，大概有将近 80 个类，但是这些类大概可以分成四组，分别是：

1. 基于字节操作的 I/O 接口：InputStream 和 OutputStream
2. 基于字符操作的 I/O 接口：Writer 和 Reader
3. 基于磁盘操作的 I/O 接口：File
4. 基于网络操作的 I/O 接口：Socket

**前两组主要是根据传输数据的数据格式，后两组主要是根据传输数据的方式**。虽然 Socket 类并不在 java.io 包下，但是我仍然把它们划分在一起，因为我个人认为 I/O 的核心问题要么是数据格式影响 I/O 操作，要么是传输方式影响 I/O 操作，也就是将什么样的数据写到什么地方的问题，I/O 只是人与机器或者机器与机器交互的手段，除了在它们能够完成这个交互功能外，我们关注的就是如何提高它的运行效率了，而数据格式和传输方式是影响效率最关键的因素了。我们后面的分析也是基于这两个因素来展开的。

![img](https://www.ibm.com/developerworks/cn/java/j-lo-javaio/image002.png)

![img](https://www.ibm.com/developerworks/cn/java/j-lo-javaio/image004.png)



注意两点：

- 一个是操作数据的方式是可以组合使用的（就是指节点流和处理流的问题），如这样组合使用:

  `OutputStream out = new BufferedOutputStream(new ObjectOutputStream(new FileOutputStream("fileName"))`；

- 流最终写到什么地方必须要指定，要么是写到磁盘要么是写到网络中。其实从上面的类图中我们发现，**写网络实际上也是写文件**，只不过写网络还有一步需要处理：底层操作系统再将数据传送到其它地方而不是本地磁盘。

## 四、BIO介绍

### 1.分类

#### 按照流的流向分，可以分为输入流和输出流。

IO的输入输出是根据数据进/出**内存**来判断的：进入内存为输入，出内存为输出。对于如图15.1所示的数据流向，数据从内存到硬盘，通常称为输出流；

 对于如图15.2所示的数据流向，数据从[服务器](https://www.baidu.com/s?wd=%E6%9C%8D%E5%8A%A1%E5%99%A8&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)通过网络流向客户端，在这种情况下,Server端的内存负责将数据输出到网络里，因此Server端的程序使用输出流；Client端的内存负责从网络中读取数据，因此Client端的程序应该使用输入流。

   ![](http://ww2.sinaimg.cn/large/006tNc79ly1g5omvwxmd6j30nr041jsy.jpg)

> 注：java的输入流主要是InputStream和Reader作为基类，而输出流则是主要由outputStream和[Writer](https://www.baidu.com/s?wd=Writer&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)作为基类。它们都是一些抽象基类，无法直接创建实例。

#### 按照操作单元划分，可以划分为字节流和字符流。

字节流操作的单元是数据单元是8位的字节，字符流操作的是数据单元为16位的字符。

> 字节流主要是由InputStream和outPutStream作为基类，而字符流则主要有Reader和Writer作为基类。

#### 按照流的角色划分为节点流和处理流。

 可以从/向一个特定的IO设备（如磁盘，网络）读/写数据的流，称为节点流。节点流也被称为低级流。图15.3显示了节点流的示意图。 
    从图15.3中可以看出，当使用节点流进行输入和输出时，程序直接连接到实际的数据源，和实际的输入/输出节点连接。 
处理流则用于对一个已存在的流进行连接和封装，通过封装后的流来实现数据的读/写功能。处理流也被称为高级流。图15.4显示了处理流的示意图。

![](http://ww2.sinaimg.cn/large/006tNc79ly1g5omvpzw8dj30po04h0ul.jpg)

从图15.4可以看出，当使用处理流进行输入/输出时，程序并不会直接连接到实际的数据源，没有和实际的输入和输出节点连接。使用处理流的一个明显的好处是，只要使用相同的处理流，程序就可以采用完全相同的输入/输出代码来访问不同的数据源，随着处理流所包装的节点流的变化，程序实际所访问的数据源也相应的发生变化。

处理流可以“嫁接”在任何已存在的流的基础之上，这就允许Java应用程序采用相同的代码，透明的方式来访问不同的输入和输出设备的数据流。图15.7显示了处理流的模型。

![](http://ww1.sinaimg.cn/large/006tNc79ly1g5omw4ww1aj30r109zq5q.jpg)

#### 常用的流的分类表

|   分类       | 字节输入流   | 字节输出流  | 字符输入流        | 字符输出流   |
| ----------| ------------| -----------| ---------------| ----------- |
| 抽象基类    | *InputStream*        | *OutputStream*        | *Reader*          | *Writer*           |
| 访问文件   | **FileInputStream**  | **FileOutputStream**  | **FileReader**    | **FileWriter**     |
| 访问数组   | **ByteArrayInputStream** | **ByteArrayOutputStream** | **CharArrayReader** | **CharArrayWriter** |
| 访问管道   | **PipedInputStream** | **PipedOutputStream** | **PipedReader**   | **PipedWriter**    |
| 访问字符串 |                      |                       | StringReader      | StringWriter       |
| 缓冲流     | BufferedInputStream  | BufferedOutputStream  | BufferedReader    | BufferedWriter     |
| 转换流     |                      |                       | InputStreamReader | OutputStreamWriter |
| 对象流     | ObjectInputStream    | ObjectOutputStream    |                   |                    |
| 抽象基类   | *FilterInputStream*  | *FilterOutputStream*  | *FilterReader*    | *FilterWriter*     |
| 打印流     |                      | PrintStream           | PrintWriter       |                    |
| 推回输入流 | PushbackInputStream  | PushbackReader        |                   |                    |
| 特殊流     | DataInputStream      | DataOutputStream      |                   |                    |

> *注：表中粗体字所标出的类代表节点流，必须直接与指定的物理节点关联：斜体字标出的类代表抽象基类，无法直接创建实例，其余正常的表示处理流。*

### 2.常用的BIO流的用法

见[javaIO](https://cyc2018.github.io/CS-Notes/#/notes/Java%20IO?id=%E4%B8%80%E3%80%81%E6%A6%82%E8%A7%88)

### 3.传统BIO通信模式

BIO通信（一请求一应答）模型图如下

![](http://ww4.sinaimg.cn/large/006tNc79ly1g5omweglp6j30ig0chmxl.jpg)

采用 **BIO 通信模型** 的服务端，通常由一个独立的 Acceptor 线程负责监听客户端的连接。我们一般通过在`while(true)` 循环中服务端会调用 `accept()` 方法等待接收客户端的连接的方式监听请求，请求一旦接收到一个连接请求，就可以建立通信套接字在这个通信套接字上进行读写操作，此时不能再接收其他客户端连接请求，只能等待同当前连接的客户端的操作执行完成， 不过可以通过多线程来支持多个客户端的连接，如上图所示。

如果要让 **BIO 通信模型** 能够同时处理多个客户端请求，就必须使用多线程（主要原因是`socket.accept()`、`socket.read()`、`socket.write()` 涉及的三个主要函数都是同步阻塞的），也就是说它在接收到客户端连接请求之后为每个客户端创建一个新的线程进行链路处理，处理完成之后，通过输出流返回应答给客户端，线程销毁。这就是典型的 **一请求一应答通信模型** 。我们可以设想一下如果这个连接不做任何事情的话就会造成不必要的线程开销，不过可以通过 **线程池机制** 改善，线程池还可以让线程的创建和回收成本相对较低。使用`FixedThreadPool` 可以有效的控制了线程的最大数量，保证了系统有限的资源的控制，实现了N(客户端请求数量):M(处理客户端请求的线程数量)的伪异步I/O模型（N 可以远远大于 M），下面一节"伪异步 BIO"中会详细介绍到。

**当客户端并发访问量增加后这种模型会出现什么问题？**

在 Java 虚拟机中，线程是宝贵的资源，线程的创建和销毁成本很高，除此之外，线程的切换成本也是很高的。尤其在 Linux 这样的操作系统中，线程本质上就是一个进程，创建和销毁线程都是重量级的系统函数。如果并发访问量增加会导致线程数急剧膨胀可能会导致线程堆栈溢出、创建新线程失败等问题，最终导致进程宕机或者僵死，不能对外提供服务。

## 五、伪异步 IO

为了解决同步阻塞I/O面临的一个链路需要一个线程处理的问题，后来有人对它的线程模型进行了优化一一后端通过一个线程池来处理多个客户端的请求接入，形成客户端个数M：线程池最大线程数N的比例关系，其中M可以远远大于N.通过线程池可以灵活地调配线程资源，设置线程的最大值，防止由于海量并发接入导致线程耗尽。

![ä¼ªå¼æ­¥IOæ¨¡åå¾](https://camo.githubusercontent.com/04b258a50ca7f9762f43d64e70f4489440bae4eb/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f332e706e67)

代码示例：

```java
/**
 * @Auther: Goffery Gong
 * @Date: 2019/3/17 17:17
 * @Description: ，连接上服务端8000端口之后，每隔2秒，我们向服务端写一个带有时间戳的 "hello world"
 */
public class IOClient{
    public static void main(String[] args) {
        new Thread(() -> {
            try {
                Socket socket = new Socket("127.0.0.1", 8000);
                while (true) {
                    try {
                        socket.getOutputStream().
                            write((new Date() + ": hello world").getBytes());
                        socket.getOutputStream().flush();
                        Thread.sleep(2000);
                    } catch (Exception e) {
                    }
                }
            } catch (IOException e) {
            }
        }).start();
    }
}
```

```java
public class IOServer {
    public static void main(String[] args) throws Exception {

        ServerSocket serverSocket = new ServerSocket(8000);
        ExecutorService pool = Executors.newFixedThreadPool(2);
        while (true) {
            // (1) 阻塞方法获取新的连接
            Socket socket = serverSocket.accept();
            // (2) 每一个新的连接都创建一个线程，负责读取数据
            pool.submit(new SocketHandler(socket));
        }
    }
}

class SocketHandler implements Runnable {
    Socket socket;

    public SocketHandler(Socket socket) {
        this.socket = socket;
    }

    @Override
    public void run() {
        try {
            byte[] data = new byte[1024];
            InputStream inputStream = socket.getInputStream();
            while (true) {
                int len;
                // (3) 按字节流方式读取数据
                while ((len = inputStream.read(data)) != -1) {
                    System.out.println(new String(data, 0, len));
                }
            }
        } catch (IOException e) {}
    }
}
```

## 六、NIO介绍

我们使用InputStream从输入流中读取数据时，如果没有读取到有效的数据，程序将在此处阻塞该线程的执行。其实传统的输入里和输出流都是阻塞式的进行输入和输出。 不仅如此，传统的输入流、输出流都是通过字节的移动来处理的（即使我们不直接处理字节流，但底层实现还是依赖于字节处理），也就是说，面向流的输入和输出一次只能处理一个字节，因此面向流的输入和输出系统效率通常不高。 
    从JDk1.4开始，java提供了一系列改进的输入和输出处理的新功能，这些功能被统称为新IO(NIO)。新增了许多用于处理输入和输出的类，这些类都被放在java.nio包及其子包下，并且对原io的很多类都以NIO为基础进行了改写。新增了满足NIO的功能。 
    **NIO采用了内存映射对象的方式来处理输入和输出，NIO将文件或者文件的一块区域映射到内存中，这样就可以像访问内存一样来访问文件了**。通过这种方式来进行输入/输出比传统的输入和输出要快的多。

### NIO工作机制

原理：NIO由原来的阻塞读写（占用线程）变成了**单线程轮询事件**，找到可以进行读写的网络描述符进行读写。

![img](http://ww3.sinaimg.cn/large/006tNc79ly1g5omx2sj69j30dy045gll.jpg)

Acceptor 负责接收客户端 Socket 发起的新建连接请求，并把该 Socket 绑定到一个 Reactor 线程上，于是这个Socket 随后的读写事件都交给此 Reactor 线程来处理。

Reactor 线程读取数据后，交给用户程序中的具体 Handler 实现类来完成特定的业务逻辑处理。为了不影响 Reactor 线程，我们通常使用一个单独的线程池来异步执行 Handler 的接口方法。

但实际上，我们的服务器是多核心的，而且需要高速并发处理大量的客户端连接，**单线程的 Reactor 模型就满足不了需求了，因此我们需要多线程的 Reactor**。一般原则是 Reactor（线程）的数量与 CPU 核心数（逻辑CPU）保持一致，即每个 CPU 执行一个 Reactor 线程，而客户端的 Socket 连接则随机均分到这些 Reactor 线程上去处理，如果有 8000 个连接，而 CPU 核心数为 8，则平均每个 CPU 核心承担 1000 个连接。

Channel

Selectors

Buffer 

### Java NIO

见[javaNIO](https://cyc2018.github.io/CS-Notes/#/notes/Java%20IO?id=%E4%B8%80%E3%80%81%E6%A6%82%E8%A7%88)

## 七、多路复用IO

select/poll/epoll 都是 I/O 多路复用的具体实现，select 出现的最早，之后是 poll，再是 epoll。

好文强烈推荐：[如果这篇文章说不清epoll的本质，那就过来掐死我吧](https://zhuanlan.zhihu.com/p/64138532)

### [select](https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=select)

#### select的原理流程

1. 将需要监视的socket保存到一个数组fds中，之后调用select，如果数组fds中的所有sokect都没有数据，则进程阻塞（将进程添加到所有socket下的等待队列中）

   ![img](http://ww2.sinaimg.cn/large/006tNc79ly1g5ouzt6g8vj30dg09s3yy.jpg)

2. 当socket接收到数据时，从select返回，唤醒进程（所谓唤起进程，就是将进程从所有的等待队列中移除，加入到工作队列里面）；

   ![img](http://ww3.sinaimg.cn/large/006tNc79ly1g5ov0ab3kmj30e70bdaao.jpg)

3. 遍历数组，使用FD_ISSET判断具体哪个socket收到了数据，之后处理；

   ![img](http://ww1.sinaimg.cn/large/006tNc79ly1g5ov1fzjgdj30df09rgm9.jpg)

> 补充说明： 本节只解释了select的一种情形。当程序调用select时，内核会先遍历一遍socket，如果有一个以上的socket接收缓冲区有数据，那么select直接返回，不会阻塞。这也是为什么select的返回值有可能大于1的原因之一。如果没有socket有数据，进程才会阻塞。

#### select调用伪代码

```c
int select(int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);Copy to clipboardErrorCopied
```

有三种类型的描述符类型：readset、writeset、exceptset，分别对应读、写、异常条件的描述符集合。fd_set 使用数组实现，数组大小使用 FD_SETSIZE 定义。

timeout 为超时参数，调用 select 会一直阻塞直到有描述符的事件到达或者等待的时间超过 timeout。

成功调用返回结果大于 0，出错返回结果为 -1，超时返回结果为 0。

```c
fd_set fd_in, fd_out;
struct timeval tv;

// Reset the sets
FD_ZERO( &fd_in );
FD_ZERO( &fd_out );

// Monitor sock1 for input events
FD_SET( sock1, &fd_in );

// Monitor sock2 for output events
FD_SET( sock2, &fd_out );

// Find out which socket has the largest numeric value as select requires it
int largest_sock = sock1 > sock2 ? sock1 : sock2;

// Wait up to 10 seconds
tv.tv_sec = 10;
tv.tv_usec = 0;

// Call the select，如果没有数据则阻塞
int ret = select( largest_sock + 1, &fd_in, &fd_out, NULL, &tv );

// Check if select actually succeed，调用成功则遍历数组，找出
if ( ret == -1 )
    // report error and abort
else if ( ret == 0 )
    // timeout; no event detected
else
{
    if ( FD_ISSET( sock1, &fd_in ) )
        // input event on sock1

    if ( FD_ISSET( sock2, &fd_out ) )
        // output event on sock2
}Copy to clipboardErrorCopied
```

#### select缺点

其一，每次调用select都需要将进程加入到所有监视socket的等待队列，每次唤醒都需要从每个队列中移除。这里涉及了两次遍历，而且每次都要将整个fds列表传递给内核，有一定的开销。正是因为遍历操作开销大，出于效率的考量，才会规定select的最大监视数量，默认只能监视1024个socket。

其二，进程被唤醒后，程序并不知道哪些socket收到数据，还需要遍历一次。

### [poll](https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=poll)

```c
int poll(struct pollfd *fds, unsigned int nfds, int timeout);Copy to clipboardErrorCopied
```

pollfd 使用链表实现。

```c
// The structure for two events
struct pollfd fds[2];

// Monitor sock1 for input
fds[0].fd = sock1;
fds[0].events = POLLIN;

// Monitor sock2 for output
fds[1].fd = sock2;
fds[1].events = POLLOUT;

// Wait 10 seconds
int ret = poll( &fds, 2, 10000 );
// Check if poll actually succeed
if ( ret == -1 )
    // report error and abort
else if ( ret == 0 )
    // timeout; no event detected
else
{
    // If we detect the event, zero it out so we can reuse the structure
    if ( fds[0].revents & POLLIN )
        fds[0].revents = 0;
        // input event on sock1

    if ( fds[1].revents & POLLOUT )
        fds[1].revents = 0;
        // output event on sock2
}Copy to clipboardErrorCopied
```

### [select与poll比较](https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=%e6%af%94%e8%be%83)

#### [1. 功能](https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=_1-%e5%8a%9f%e8%83%bd)

select 和 poll 的功能基本相同，不过在一些实现细节上有所不同。

- select 会修改描述符，而 poll 不会；
- ==select 的描述符类型使用数组实现，FD_SETSIZE 大小默认为 1024，因此默认只能监听 1024 个描述符。==如果要监听更多描述符的话，需要修改 FD_SETSIZE 之后重新编译；==而 poll 的描述符类型使用链表实现，没有描述符数量的限制；==
- poll 提供了更多的事件类型，并且对描述符的重复利用上比 select 高。
- 如果一个线程对某个描述符调用了 select 或者 poll，另一个线程关闭了该描述符，会导致调用结果不确定。

#### [2. 速度](https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=_2-%e9%80%9f%e5%ba%a6)

**select 和 poll 速度都比较慢**。

- **select 和 poll 每次调用都需要将全部描述符从应用进程缓冲区复制到内核缓冲区**。
- **select 和 poll 的返回结果中没有声明哪些描述符已经准备好，所以如果返回值大于 0 时，应用进程都需要使用轮询的方式来找到 I/O 完成的描述符**。

#### [3. 可移植性](https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=_3-%e5%8f%af%e7%a7%bb%e6%a4%8d%e6%80%a7)

几乎所有的系统都支持 select，但是只有比较新的系统支持 poll。

### [epoll](https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=epoll)

#### 1.epoll原理和流程

1. 进程调用`epoll_create`创建eventpoll对象（也就是程序中epfd所代表的对象）

2. 维护监视列表：创建epoll对象后，可以用`epoll_ctl`添加或删除所要监听的socket，内核会将eventpoll添加到**socket的等待队列中**；

   - 监视列表的数据结构：红黑树

3. 当socket收到数据后，中断程序会给eventpoll的“就绪列表”添加socket引用；**中断程序会操作eventpoll对象，而不是直接操作进程**（对比上面select图，加入到socket等待队列中的是进程，而epoll则是eventpoll对象）。

4. 当程序执行到`epoll_wait`时，如果rdlist已经引用了socket，那么epoll_wait直接返回，如果rdlist为空，阻塞进程

   1. 进程阻塞和唤醒进程：假设计算机中正在运行进程A和进程B，在某时刻进程A运行到了epoll_wait语句。如下图所示，内核会将进程A放入eventpoll的等待队列中，阻塞进程

      ![](http://ww1.sinaimg.cn/large/006tNc79ly1g5ovs24n3fj30er0bxq3w.jpg)

   2. 当socket接收到数据，中断程序一方面修改rdlist，另一方面唤醒eventpoll等待队列中的进程，进程A再次进入运行状态（如下图）。也因为rdlist的存在，进程A可以知道哪些socket发生了变化

      ![](http://ww2.sinaimg.cn/large/006tNc79ly1g5ovsiashrj30ev0by3zl.jpg)

#### 2. 实现细节

1. eventpoll对象中就序列表rdlist的数据结构：双向队列
2. eventpoll对象中，维护监视队列的数据结构（rbr）：红黑树。需要满足快速插入，删除，检索（避免重复）

> ps：因为操作系统要兼顾多种功能，以及由更多需要保存的数据，rdlist并非直接引用socket，而是通过epitem间接引用，红黑树的节点也是epitem对象。同样，文件系统也并非直接引用着socket。为方便理解，本文中省略了一些间接结构。

![](http://ww3.sinaimg.cn/large/006tNc79ly1g5ow80h4foj30ep0cv75b.jpg)

#### 3.epoll调用伪代码

```c
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);Copy to clipboardErrorCopied
```

epoll_ctl() 用于向内核注册新的描述符或者是改变某个文件描述符的状态。已注册的描述符在内核中会被维护在一棵红黑树上，通过回调函数内核会将 I/O 准备好的描述符加入到一个链表中管理，进程调用 epoll_wait() 便可以得到事件完成的描述符。

从上面的描述可以看出，epoll 只需要将描述符从进程缓冲区向内核缓冲区拷贝一次，并且进程不需要通过轮询来获得事件完成的描述符。

epoll 仅适用于 Linux OS。

epoll 比 select 和 poll 更加灵活而且没有描述符数量限制。

epoll 对多线程编程更有友好，一个线程调用了 epoll_wait() 另一个线程关闭了同一个描述符也不会产生像 select 和 poll 的不确定情况。

```c
// Create the epoll descriptor. Only one is needed per app, and is used to monitor all sockets.
// The function argument is ignored (it was not before, but now it is), so put your favorite number here
int pollingfd = epoll_create( 0xCAFE );

if ( pollingfd < 0 )
 // report error

// Initialize the epoll structure in case more members are added in future
struct epoll_event ev = { 0 };

// Associate the connection class instance with the event. You can associate anything
// you want, epoll does not use this information. We store a connection class pointer, pConnection1
ev.data.ptr = pConnection1;

// Monitor for input, and do not automatically rearm the descriptor after the event
ev.events = EPOLLIN | EPOLLONESHOT;
// Add the descriptor into the monitoring list. We can do it even if another thread is
// waiting in epoll_wait - the descriptor will be properly added
if ( epoll_ctl( epollfd, EPOLL_CTL_ADD, pConnection1->getSocket(), &ev ) != 0 )
    // report error

// Wait for up to 20 events (assuming we have added maybe 200 sockets before that it may happen)
struct epoll_event pevents[ 20 ];

// Wait for 10 seconds, and retrieve less than 20 epoll_event and store them into epoll_event array
int ready = epoll_wait( pollingfd, pevents, 20, 10000 );
// Check if epoll actually succeed
if ( ret == -1 )
    // report error and abort
else if ( ret == 0 )
    // timeout; no event detected
else
{
    // Check if any events detected
    for ( int i = 0; i < ret; i++ )
    {
        if ( pevents[i].events & EPOLLIN )
        {
            // Get back our connection pointer
            Connection * c = (Connection*) pevents[i].data.ptr;
            c->handleReadEvent();
         }
    }
}Copy to clipboardErrorCopied
```

### 三种模式对比

epoll在select和poll（poll和select基本一样，有少量改进）的基础引入了eventpoll作为中间层，使用了先进的数据结构，是一种高效的多路复用技术。

**epoll只有在持有很多连接，并且每个连接都不是特别活跃的时候效率才高，其他的情况不见得比select好**

==epoll只有在持有很多连接，并且每个连接都不是特别活跃的时候效率才高，其他的情况不见得比select好==

![](http://ww2.sinaimg.cn/large/006tNc79ly1g5ow9cexhhj30pv0c1tbv.jpg)

### [应用场景](https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=%e5%ba%94%e7%94%a8%e5%9c%ba%e6%99%af)

很容易产生一种错觉认为只要用 epoll 就可以了，select 和 poll 都已经过时了，其实它们都有各自的使用场景。

#### [1. select 应用场景](https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=_1-select-%e5%ba%94%e7%94%a8%e5%9c%ba%e6%99%af)

select 的 timeout 参数精度为 1ns，而 poll 和 epoll 为 1ms，因此 select 更加适用于实时性要求比较高的场景，比如核反应堆的控制。

select 可移植性更好，几乎被所有主流平台所支持。

#### [2. poll 应用场景](https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=_2-poll-%e5%ba%94%e7%94%a8%e5%9c%ba%e6%99%af)

poll 没有最大描述符数量的限制，如果平台支持并且对实时性要求不高，应该使用 poll 而不是 select。

#### [3. epoll 应用场景](https://cyc2018.github.io/CS-Notes/#/notes/Socket?id=_3-epoll-%e5%ba%94%e7%94%a8%e5%9c%ba%e6%99%af)

只需要运行在 Linux 平台上，有大量的描述符需要同时轮询，并且这些连接最好是长连接。

需要同时监控小于 1000 个描述符，就没有必要使用 epoll，因为这个应用场景下并不能体现 epoll 的优势。

需要监控的描述符状态变化多，而且都是非常短暂的，也没有必要使用 epoll。因为 epoll 中的所有描述符都存储在内核中，造成每次需要对描述符的状态改变都需要通过 epoll_ctl() 进行系统调用，频繁系统调用降低效率。并且 epoll 的描述符存储在内核，不容易调试。

### 总结

多路复用往往在提升性能方面有着重要的作用。select系统调用的功能是对多个文件描述符进行监视，当有文件描述符的文件读写操作完成以及发生异常或者超时，该调用会返回这些文件描述符。select 需要遍历所有的文件描述符，就遍历操作而言，复杂度是 O(N)。

epoll相关系统调用是在Linux 2.5 后的某个版本开始引入的。该系统调用针对传统的select/poll不足，设计上作了很大的改动。select/poll 的缺点在于: 

1. 每次调用时要重复地从用户模式读入参数，并重复地扫描文件描述符。
2. 每次在调用开始时，要把当前进程放入各个文件描述符的等待队列。在调用结束后，又把进程从各个等待队列中删除。

epoll 是把 select/poll 单个的操作拆分为 1 个 epoll*create，多个 epoll*ctrl和一个 wait。此外，操作系统内核针对 epoll 操作添加了一个文件系统，每一个或者多个要监视的文件描述符都有一个对应的inode 节点，主要信息保存在 eventpoll 结构中。而被监视的文件的重要信息则保存在 epitem 结构中，是一对多的关系。由于在执行 epoll*create 和 epoll*ctrl 时，已经把用户模式的信息保存到内核了， 所以之后即便反复地调用 epoll_wait，也不会重复地拷贝参数，不会重复扫描文件描述符，也不反复地把当前进程放入/拿出等待队列。

所以，当前主流的Server侧Socket实现大都采用了epoll的方式，例如Nginx， 在配置文件可以显式地看到 `use epoll`。





## 参考

[美团-javaNIO浅析](https://tech.meituan.com/2016/11/04/nio.html)

[深入分析 Java I/O 的工作机制](https://www.ibm.com/developerworks/cn/java/j-lo-javaio/index.html>)

[javaIO体系](https://blog.csdn.net/nightcurtis/article/details/51324105)

[跟着闪电侠学netty](https://www.jianshu.com/p/a4e03835921a)

[JavaGuide](https://github.com/Snailclimb/JavaGuide/blob/master/Java/BIO%2CNIO%2CAIO%20summary.md#12-%E4%BC%AA%E5%BC%82%E6%AD%A5-io)

[javaIO一本难念的经](https://zhuanlan.zhihu.com/p/29675083)

[老曹眼中的网络编程基础](https://mp.weixin.qq.com/s/XXMz5uAFSsPdg38bth2jAA)