## 基本知识

### 数据单位

bit：位（二进制的一个“0”或一个“1”叫一位。）

byte： 1字节=8bit（它是一个8位的二进制数，是一个很具体的存储空间）

char： 字符

ASCII码：ASCII码中，一个英文字母（不分大小写）占一个字节的空间，一个中文汉字占两个字节的空间。GB2312 是对 ASCII 的中文扩展。

Unicode字符集：常用的是用**两个字节**（16位）表示一个字符（如果要用到非常偏僻的字符，就需要4个字节）。

UTF-8编码规则：一种变长的编码方式：它可以使用1~4个字节表示一个符号，根据不同的符号而变化字节长度。当字符在ASCII码的范围时，就用一个字节表示，保留了ASCII字符一个字节的编码做为它的一部分，如此一来UTF-8编码也可以是为视为一种对ASCII码的拓展。值得注意的是unicode编码中一个中文字符占2个字节，而UTF-8一个中文字符占3个字节。从unicode到uft-8并不是直接的对应，而是要过一些算法和规则来转换。

**UTF-8**就是每次8个位传输数据，而**UTF-16**就是每次16个位。UTF-8就是在互联网上使用最广的一种unicode的实现方式，这是为传输而设计的编码。

<u>在计算机内存中，统一使用Unicode编码，当需要保存到硬盘或者需要传输的时候，就转换为UTF-8编码。</u>

### TCP通信与socket

首先，对于 TCP 通信来说，每个 TCP Socket 在内核中都有一个**发送缓冲区**和一个**接收缓冲区**，TCP 的全双工的工作模式及 TCP 的滑动窗口便依赖于这两个独立的 Buffer 及此 Buffer 的填充状态。

接收缓冲区把数据缓存入内核，若应用进程一直没有调用 Socket 的 read 方法进行读取的话，则此数据会一直被缓存在接收缓冲区内。不管进程是否读取 Socket，对端发来的数据都会经由内核接收并且缓存到 Socket 的内核接收缓冲区中。read 所做的工作，就是把内核接收缓冲区中的数据复制到应用层用户的 Buffer 里面，仅此而已。

进程调用 Socket 的 send 发送数据的时候，最简单的情况（也是一般情况）是将数据从应用层用户的 Buffer 里复制到 Socket 的内核发送缓冲区中，然后 send 便会在上层返回。换句话说，send 返回时，数据不一定会被发送到对端（和 write写文件有点类似），send 仅仅是把应用层 Buffer 的数据复制到 Socket 的内核发送 Buffer 中。而对于 UDP 通信来说，每个 UDP Socket 都有一个接收缓冲区，而没有发送缓冲区，从概念上来说就是只要有数据就发，不管对方是否可以正确接收，所以不缓冲，不需要发送缓冲区。

其次，我们来说说 TCP/IP 的滑动窗口和流量控制机制，前面我们提到，Socket 的接收缓冲区被 TCP 和 UDP 用来缓存网络上收到的数据，一直保存到应用进程读走为止。对于 TCP 来说，如果应用进程一直没有读取，则 Buffer 满了之后，发生的动作是：通知对端 TCP 协议中的窗口关闭，保证 TCP 套接口接收缓冲区不会溢出，保证了 TCP 是可靠传输的，这个便是滑动窗口的实现。因为对方不允许发出超过通告窗口大小的数据，所以如果对方无视窗口大小而发出了超过窗口大小的数据，则接收方 TCP 将丢弃它，这就是 TCP 的流量控制原理。而对于 UDP 来说，当接收方的 Socket 接收缓冲区满时，新来的数据报无法进入接收缓冲区，此数据报就会被丢弃，UDP 是没有流量控制的，快的发送者可以很容易地淹没慢的接收者，导致接收方的 UDP丢弃数据报。

## Java 的 I/O 类库的基本架构

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

## BIO介绍

### 1.分类

#### 按照流的流向分，可以分为输入流和输出流。

IO的输入输出是根据数据进/出**内存**来判断的：进入内存为输入，出内存为输出。对于如图15.1所示的数据流向，数据从内存到硬盘，通常称为输出流；

 对于如图15.2所示的数据流向，数据从[服务器](https://www.baidu.com/s?wd=%E6%9C%8D%E5%8A%A1%E5%99%A8&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)通过网络流向客户端，在这种情况下,Server端的内存负责将数据输出到网络里，因此Server端的程序使用输出流；Client端的内存负责从网络中读取数据，因此Client端的程序应该使用输入流。

   ![è¿æ¯å¾çæè¿°](https://img-blog.csdn.net/20160505173519730)

> 注：java的输入流主要是InputStream和Reader作为基类，而输出流则是主要由outputStream和[Writer](https://www.baidu.com/s?wd=Writer&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)作为基类。它们都是一些抽象基类，无法直接创建实例。

#### 按照操作单元划分，可以划分为字节流和字符流。

字节流操作的单元是数据单元是8位的字节，字符流操作的是数据单元为16位的字符。

> 字节流主要是由InputStream和outPutStream作为基类，而字符流则主要有Reader和Writer作为基类。

#### 按照流的角色划分为节点流和处理流。

 可以从/向一个特定的IO设备（如磁盘，网络）读/写数据的流，称为节点流。节点流也被称为低级流。图15.3显示了节点流的示意图。 
    从图15.3中可以看出，当使用节点流进行输入和输出时，程序直接连接到实际的数据源，和实际的输入/输出节点连接。 
处理流则用于对一个已存在的流进行连接和封装，通过封装后的流来实现数据的读/写功能。处理流也被称为高级流。图15.4显示了处理流的示意图。

![è¿éåå¾çæè¿°](https://img-blog.csdn.net/20160505135650158)

从图15.4可以看出，当使用处理流进行输入/输出时，程序并不会直接连接到实际的数据源，没有和实际的输入和输出节点连接。使用处理流的一个明显的好处是，只要使用相同的处理流，程序就可以采用完全相同的输入/输出代码来访问不同的数据源，随着处理流所包装的节点流的变化，程序实际所访问的数据源也相应的发生变化。

处理流可以“嫁接”在任何已存在的流的基础之上，这就允许Java应用程序采用相同的代码，透明的方式来访问不同的输入和输出设备的数据流。图15.7显示了处理流的模型。

![è¿éåå¾çæè¿°](https://img-blog.csdn.net/20160505170600458)

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

#### Io体系的基类（InputStream/Reader，OutputStream/Writer）

### 3.传统BIO通信模式

BIO通信（一请求一应答）模型图如下

![ä¼ ç"BIOéä¿¡æ¨¡åå¾](https://camo.githubusercontent.com/5ef6de9824ae82bb0c403522a647953d1193a362/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f322e706e67)

采用 **BIO 通信模型** 的服务端，通常由一个独立的 Acceptor 线程负责监听客户端的连接。我们一般通过在`while(true)` 循环中服务端会调用 `accept()` 方法等待接收客户端的连接的方式监听请求，请求一旦接收到一个连接请求，就可以建立通信套接字在这个通信套接字上进行读写操作，此时不能再接收其他客户端连接请求，只能等待同当前连接的客户端的操作执行完成， 不过可以通过多线程来支持多个客户端的连接，如上图所示。

如果要让 **BIO 通信模型** 能够同时处理多个客户端请求，就必须使用多线程（主要原因是`socket.accept()`、`socket.read()`、`socket.write()` 涉及的三个主要函数都是同步阻塞的），也就是说它在接收到客户端连接请求之后为每个客户端创建一个新的线程进行链路处理，处理完成之后，通过输出流返回应答给客户端，线程销毁。这就是典型的 **一请求一应答通信模型** 。我们可以设想一下如果这个连接不做任何事情的话就会造成不必要的线程开销，不过可以通过 **线程池机制** 改善，线程池还可以让线程的创建和回收成本相对较低。使用`FixedThreadPool` 可以有效的控制了线程的最大数量，保证了系统有限的资源的控制，实现了N(客户端请求数量):M(处理客户端请求的线程数量)的伪异步I/O模型（N 可以远远大于 M），下面一节"伪异步 BIO"中会详细介绍到。

**当客户端并发访问量增加后这种模型会出现什么问题？**

在 Java 虚拟机中，线程是宝贵的资源，线程的创建和销毁成本很高，除此之外，线程的切换成本也是很高的。尤其在 Linux 这样的操作系统中，线程本质上就是一个进程，创建和销毁线程都是重量级的系统函数。如果并发访问量增加会导致线程数急剧膨胀可能会导致线程堆栈溢出、创建新线程失败等问题，最终导致进程宕机或者僵死，不能对外提供服务。

## 伪异步 IO

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

## NIO介绍

我们使用InputStream从输入流中读取数据时，如果没有读取到有效的数据，程序将在此处阻塞该线程的执行。其实传统的输入里和输出流都是阻塞式的进行输入和输出。 不仅如此，传统的输入流、输出流都是通过字节的移动来处理的（即使我们不直接处理字节流，但底层实现还是依赖于字节处理），也就是说，面向流的输入和输出一次只能处理一个字节，因此面向流的输入和输出系统效率通常不高。 
    从JDk1.4开始，java提供了一系列改进的输入和输出处理的新功能，这些功能被统称为新IO(NIO)。新增了许多用于处理输入和输出的类，这些类都被放在java.nio包及其子包下，并且对原io的很多类都以NIO为基础进行了改写。新增了满足NIO的功能。 
    **NIO采用了内存映射对象的方式来处理输入和输出，NIO将文件或者文件的一块区域映射到内存中，这样就可以像访问内存一样来访问文件了**。通过这种方式来进行输入/输出比传统的输入和输出要快的多。

### NIO工作机制

原理：NIO由原来的阻塞读写（占用线程）变成了**单线程轮询事件**，找到可以进行读写的网络描述符进行读写。

![img](https://pic1.zhimg.com/80/v2-fc1f1b4368c19a13e064f3ea202ae1a0_hd.jpg)

Acceptor 负责接收客户端 Socket 发起的新建连接请求，并把该 Socket 绑定到一个 Reactor 线程上，于是这个Socket 随后的读写事件都交给此 Reactor 线程来处理。

Reactor 线程读取数据后，交给用户程序中的具体 Handler 实现类来完成特定的业务逻辑处理。为了不影响 Reactor 线程，我们通常使用一个单独的线程池来异步执行 Handler 的接口方法。

但实际上，我们的服务器是多核心的，而且需要高速并发处理大量的客户端连接，**单线程的 Reactor 模型就满足不了需求了，因此我们需要多线程的 Reactor**。一般原则是 Reactor（线程）的数量与 CPU 核心数（逻辑CPU）保持一致，即每个 CPU 执行一个 Reactor 线程，而客户端的 Socket 连接则随机均分到这些 Reactor 线程上去处理，如果有 8000 个连接，而 CPU 核心数为 8，则平均每个 CPU 核心承担 1000 个连接。

Channel

Selectors

Buffer 

## Linux的五种IO模型

### JavaIO与操作系统IO的联系

- 在Java中，主要有三种IO模型，分别是**阻塞IO（BIO）、非阻塞IO（NIO）和 异步IO（AIO）**。

- 在Linux(UNIX)操作系统中，共有五种IO模型，分别是：**阻塞IO模型**、**非阻塞IO模型**、**IO复用模型**、**信号驱动IO模型**以及**异步IO模型**。

**Java中提供的IO有关的API，在文件处理的时候，其实依赖操作系统层面的IO操作实现的。**比如在Linux 2.6以后，Java中NIO和AIO都是通过epoll来实现的，而在Windows上，AIO是通过IOCP来实现的。

可以把Java中的BIO、NIO和AIO理解为是Java语言对操作系统的各种IO模型的封装。程序员在使用这些API的时候，不需要关心操作系统层面的知识，也不需要根据不同操作系统编写不同的代码。只需要使用Java的API就可以了。

一次IO过程：文件从硬盘中拷贝到用户空间中，中间过渡的空间映射成内核空间。

### **阻塞IO模型**

阻塞 I/O 是最简单的 I/O 模型，一般表现为进程或线程等待某个条件，如果条件不满足，则一直等下去。条件满足，则进行下一步操作。

![img](https://user-gold-cdn.xitu.io/2018/9/9/165bde9e5723181b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

应用进程通过系统调用 `recvfrom` 接收数据，但由于内核还未准备好数据报，应用进程就会阻塞住，直到内核准备好数据报，`recvfrom` 完成数据报复制工作，应用进程才能结束阻塞状态。

这种钓鱼方式相对来说比较简单，对于钓鱼的人来说，不需要什么特制的鱼竿，拿一根够长的木棍就可以悠闲的开始钓鱼了（实现简单）。缺点就是比较耗费时间，比较适合那种对鱼的需求量小的情况（并发低，时效性要求低）。

### **非阻塞IO模型** 

我们钓鱼的时候，在等待鱼儿咬钩的过程中，我们可以做点别的事情，比如玩一把王者荣耀、看一集《延禧攻略》等等。但是，我们要时不时的去看一下鱼竿，一旦发现有鱼儿上钩了，就把鱼钓上来。

映射到Linux操作系统中，这就是非阻塞的IO模型。应用进程与内核交互，目的未达到之前，不再一味的等着，而是直接返回。然后通过轮询的方式，不停的去问内核数据准备有没有准备好。如果某一次轮询发现数据已经准备好了，那就把数据拷贝到用户空间中。

![img](https://user-gold-cdn.xitu.io/2018/9/9/165bde9e699d0713?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

应用进程通过 `recvfrom` 调用不停的去和内核交互，直到内核准备好数据。如果没有准备好，内核会返回`error`，应用进程在得到`error`后，过一段时间再发送`recvfrom`请求。在两次发送请求的时间段，进程可以先做别的事情。

### **信号驱动IO模型** 

我们钓鱼的时候，为了避免自己一遍一遍的去查看鱼竿，我们可以给鱼竿安装一个报警器。当有鱼儿咬钩的时候立刻报警。然后我们再收到报警后，去把鱼钓起来。

映射到Linux操作系统中，这就是信号驱动IO。应用进程在读取文件时通知内核，如果某个 socket 的某个事件发生时，请向我发一个信号。在收到信号后，信号对应的处理函数会进行后续处理。

![img](https://user-gold-cdn.xitu.io/2018/9/9/165bde9e6c6552e2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

应用进程预先向内核注册一个信号处理函数，然后用户进程返回，并且不阻塞，当内核数据准备就绪时会发送一个信号给进程，用户进程便在信号处理函数中开始把数据拷贝的用户空间中。

### **IO复用模型** 

我们钓鱼的时候，为了保证可以最短的时间钓到最多的鱼，我们同一时间摆放多个鱼竿，同时钓鱼。然后哪个鱼竿有鱼儿咬钩了，我们就把哪个鱼竿上面的鱼钓起来。

映射到Linux操作系统中，这就是IO复用模型。多个进程的IO可以注册到同一个管道上，这个管道会统一和内核进行交互。当管道中的某一个请求需要的数据准备好之后，进程再把对应的数据拷贝到用户空间中。

![img](https://user-gold-cdn.xitu.io/2018/9/9/165bde9e6e6da275?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

IO多路转接是多了一个`select`函数，多个进程的IO可以注册到同一个`select`上，当用户进程调用该`select`，`select`会监听所有注册好的IO，如果所有被监听的IO需要的数据都没有准备好时，`select`调用进程会阻塞。当任意一个IO所需的数据准备好之后，`select`调用就会返回，然后进程在通过`recvfrom`来进行数据拷贝。

**这里的IO复用模型，并没有向内核注册信号处理函数，所以，他并不是非阻塞的。**进程在发出`select`后，要等到`select`监听的所有IO操作中至少有一个需要的数据准备好，才会有返回，并且也需要再次发送请求去进行文件的拷贝。

### 同步IO模型

我们说阻塞IO模型、非阻塞IO模型、IO复用模型和信号驱动IO模型都是同步的IO模型。原因是因为，无论以上那种模型，真正的数据拷贝过程，都是同步进行的。

**信号驱动难道不是异步的么？** 信号驱动，内核是在数据准备好之后通知进程，然后进程再通过`recvfrom`操作进行数据拷贝。我们可以认为**数据准备阶段是异步的**，但是，**数据拷贝操作是同步的**。所以，整个IO过程也不能认为是异步的。

### **异步IO模型** 

我们钓鱼的时候，采用一种高科技钓鱼竿，即全自动钓鱼竿。可以自动感应鱼上钩，自动收竿，更厉害的可以自动把鱼放进鱼篓里。然后，通知我们鱼已经钓到了，他就继续去钓下一条鱼去了。

映射到Linux操作系统中，这就是异步IO模型。应用进程把IO请求传给内核后，完全由内核去操作文件拷贝。内核完成相关操作后，会发信号告诉应用进程本次IO已经完成。

![img](https://user-gold-cdn.xitu.io/2018/9/9/165bde9e70ade864?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

用户进程发起`aio_read`操作之后，给内核传递描述符、缓冲区指针、缓冲区大小等，告诉内核当整个操作完成时，如何通知进程，然后就立刻去做其他事情了。当内核收到`aio_read`后，会立刻返回，然后内核开始等待数据准备，数据准备好以后，直接把数据拷贝到用户控件，然后再通知进程本次IO已经完成。

### **5种IO模型对比**

![emma_1](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2016/77752ed5.jpg)

## 参考

[美团-javaNIO浅析](https://tech.meituan.com/2016/11/04/nio.html)

[深入分析 Java I/O 的工作机制](https://www.ibm.com/developerworks/cn/java/j-lo-javaio/index.html>)

[javaIO体系](https://blog.csdn.net/nightcurtis/article/details/51324105)

[跟着闪电侠学netty](https://www.jianshu.com/p/a4e03835921a)

[JavaGuide](https://github.com/Snailclimb/JavaGuide/blob/master/Java/BIO%2CNIO%2CAIO%20summary.md#12-%E4%BC%AA%E5%BC%82%E6%AD%A5-io)

[javaIO一本难念的经](https://zhuanlan.zhihu.com/p/29675083)