### 基本知识

bit：位（二进制的一个“0”或一个“1”叫一位。）

byte： 1字节=8bit（它是一个8位的二进制数，是一个很具体的存储空间）

char： 字符

ASCII码：ASCII码中，一个英文字母（不分大小写）占一个字节的空间，一个中文汉字占两个字节的空间。GB2312 是对 ASCII 的中文扩展。

Unicode字符集：常用的是用**两个字节**（16位）表示一个字符（如果要用到非常偏僻的字符，就需要4个字节）。

UTF-8编码规则：一种变长的编码方式：它可以使用1~4个字节表示一个符号，根据不同的符号而变化字节长度。当字符在ASCII码的范围时，就用一个字节表示，保留了ASCII字符一个字节的编码做为它的一部分，如此一来UTF-8编码也可以是为视为一种对ASCII码的拓展。值得注意的是unicode编码中一个中文字符占2个字节，而UTF-8一个中文字符占3个字节。从unicode到uft-8并不是直接的对应，而是要过一些算法和规则来转换。

**UTF-8**就是每次8个位传输数据，而**UTF-16**就是每次16个位。UTF-8就是在互联网上使用最广的一种unicode的实现方式，这是为传输而设计的编码。

<u>在计算机内存中，统一使用Unicode编码，当需要保存到硬盘或者需要传输的时候，就转换为UTF-8编码。</u>

### IO流分类

#### 按照流的流向分，可以分为输入流和输出流。

1. IO的输入输出是根据数据进/出**内存**来判断的：进入内存为输入，出内存为输出。对于如图15.1所示的数据流向，数据从内存到硬盘，通常称为输出流；

   对于如图15.2所示的数据流向，数据从[服务器](https://www.baidu.com/s?wd=%E6%9C%8D%E5%8A%A1%E5%99%A8&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)通过网络流向客户端，在这种情况下,Server端的内存负责将数据输出到网络里，因此Server端的程序使用输出流；Client端的内存负责从网络中读取数据，因此Client端的程序应该使用输入流。

   ![è¿æ¯å¾çæè¿°](https://img-blog.csdn.net/20160505173519730)

2. > 注：java的输入流主要是InputStream和Reader作为基类，而输出流则是主要由outputStream和[Writer](https://www.baidu.com/s?wd=Writer&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)作为基类。它们都是一些抽象基类，无法直接创建实例。

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

### 常用的io流的用法

1. #### Io体系的基类（InputStream/Reader，OutputStream/Writer）

### Java 的 I/O 类库的基本架构

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

### NIO介绍

我们使用InputStream从输入流中读取数据时，如果没有读取到有效的数据，程序将在此处阻塞该线程的执行。其实传统的输入里和输出流都是阻塞式的进行输入和输出。 不仅如此，传统的输入流、输出流都是通过字节的移动来处理的（即使我们不直接处理字节流，但底层实现还是依赖于字节处理），也就是说，面向流的输入和输出一次只能处理一个字节，因此面向流的输入和输出系统效率通常不高。 
    从JDk1.4开始，java提供了一系列改进的输入和输出处理的新功能，这些功能被统称为新IO(NIO)。新增了许多用于处理输入和输出的类，这些类都被放在java.nio包及其子包下，并且对原io的很多类都以NIO为基础进行了改写。新增了满足NIO的功能。 
    **NIO采用了内存映射对象的方式来处理输入和输出，NIO将文件或者文件的一块区域映射到内存中，这样就可以像访问内存一样来访问文件了**。通过这种方式来进行输入/输出比传统的输入和输出要快的多。

