# 谈谈 Java NIO

在 JDK1.4 之后，为了提高 Java IO 的效率，Java 提供了一套 New IO (NIO)，之所以称之为 New，原因在于它相对于之前的 IO 类库是新增的。此外，旧的 IO 类库提供的 IO 方法是阻塞的，New IO 类库则让 Java 可支持非阻塞 IO，所以，更多的人喜欢称之为非阻塞 IO（Non-blocking IO）。

NIO 应用非常广泛，是 Java 进阶的必学知识，此外，在 Java 相关岗位的面试中也是“常客”，对于准备深入学习 Java 的读者，了解 NIO 确有必要。

本场 Chat，我将分享以下内容：

1. IO 与 NIO 有何不同？
2. NIO 核心对象 Buffer 详解；
3. NIO 核心对象 Channel 详解；
4. NIO 核心对象 Selector 详解；
5. Reactor 模式介绍。

### 1. IO 和 NIO 相关的预备知识

#### 1.1 IO 的含义

讲 NIO 之前，我们先来看一下 IO。

Java IO 即 Java 输入输出。在开发应用软件时，很多时候都需要和各种输入输出相关的媒介打交道。与媒介进行 IO 操作的过程十分复杂，需要考虑众多因素，比如：进行 IO 操作媒介的类型（文件、控制台、网络）、通信方式（顺序、随机、二进制、按字符、按字、按行等等）。

Java 类库提供了相应的类来解决这些难题，这些类就位于 java.io 包中， 在整个 java.io 包中最重要的就是 5 个类和一个接口。5 个类指的是 File、OutputStream、InputStream、Writer、Reader；一个接口指的是 Serializable。

由于老的 Java IO 标准类提供 IO 操作（如 read()，write()）都是同步阻塞的，因此，IO 通常也被称为阻塞 IO（即 BIO，Blocking I/O）。

#### 1.2 NIO 含义

在 JDK1.4 之后，为了提高 Java IO 的效率，Java 又提供了一套 New IO（NIO），原因在于它相对于之前的 IO 类库是新增的。此外，旧的 IO 类库提供的 IO 方法是阻塞的，New IO 类库则让 Java 可支持非阻塞 IO，所以，更多的人喜欢称之为非阻塞 IO（Non-blocking IO）。

#### 1.3 四种 IO 模型

**同步阻塞 IO：**

在此种方式下，用户进程在发起一个 IO 操作以后，必须等待 IO 操作的完成，只有当真正完成了 IO 操作以后，用户进程才能运行。 Java 传统的 IO 模型属于此种方式！

**同步非阻塞 IO：**

在此种方式下，用户进程发起一个 IO 操作以后 便可返回做其它事情，但是用户进程需要时不时的询问 IO 操作是否就绪，这就要求用户进程不停的去询问，从而引入不必要的 CPU 资源浪费。其中目前 Java 的 NIO 就属于同步非阻塞 IO 。

**异步阻塞 IO：**

此种方式下是指应用发起一个 IO 操作以后，不等待内核 IO 操作的完成，等内核完成 IO 操作以后会通知应用程序，这其实就是同步和异步最关键的区别，同步必须等待或者主动的去询问 IO 是否完成，那么为什么说是阻塞的呢？因为此时是通过 select 系统调用来完成的，而 select 函数本身的实现方式是阻塞的，而采用 select 函数有个好处就是它可以同时监听多个文件句柄，从而提高系统的并发性！

**异步非阻塞 IO：**

在此种模式下，用户进程只需要发起一个 IO 操作然后立即返回，等 IO 操作真正的完成以后，应用程序会得到 IO 操作完成的通知，此时用户进程只需要对数据进行处理就好了，不需要进行实际的 IO 读写操作，因为 真正的 IO 读取或者写入操作已经由 内核完成了。目前 Java 中还没有支持此种 IO 模型。

#### 1.4 小结

所有的系统 I/O 都分为两个阶段：等待就绪和操作。举例来说，读函数，分为等待系统可读和真正的读；同理，写函数分为等待网卡可以写和真正的写。Java IO 的各种流是阻塞的。这意味着当线程调用 write() 或 read() 时，线程会被阻塞，直到有一些数据可用于读取或数据被完全写入。

需要说明的是等待就绪引起的 “阻塞” 是不使用 CPU 的，是在 “空等”；而真正的读写操作引起的“阻塞” 是使用 CPU 的，是真正在”干活”，而且这个过程非常快，属于 memory copy，带宽通常在 1GB/s 级别以上，可以理解为基本不耗时。因此，所谓 “阻塞” 主要是指等待就绪的过程。

**以socket.read()为例子：**

传统的阻塞 IO(BIO) 里面 socket.read()，如果接收缓冲区里没有数据，函数会一直阻塞，直到收到数据，返回读到的数据。

而对于非阻塞 IO(NIO)，如果接收缓冲区没有数据，则直接返回 0，而不会阻塞；如果接收缓冲区有数据，就把数据从网卡读到内存，并且返回给用户。

说得接地气一点，BIO 里用户最关心 “我要读”，NIO 里用户最关心” 我可以读了”。NIO 一个重要的特点是：socket 主要的读、写、注册和接收函数，在等待就绪阶段都是非阻塞的，真正的 I/O 操作是同步阻塞的（消耗 CPU 但性能非常高）。

### 2. NIO 核心对象 Buffer 详解

> 为什么说 NIO 是基于缓冲区的 IO 方式呢？因为，当一个链接建立完成后，IO 的数据未必会马上到达，为了当数据到达时能够正确完成 IO 操作，在 BIO（阻塞 IO）中，等待 IO 的线程必须被阻塞，以全天候地执行 IO 操作。为了解决这种 IO 方式低效的问题，引入了缓冲区的概念，当数据到达时，可以预先被写入缓冲区，再由缓冲区交给线程，因此线程无需阻塞地等待 IO。

在正式介绍 Buffer 之前，我们先来 Stream，以便更深刻的理解 Java IO 与 NIO 的不同。

#### 2.1 Stream

Java IO 是面向流的 I/O，这意味着我们需要从流中读取一个或多个字节。它使用流来在数据源/槽和 Java 程序之间传输数据。使用此方法的 I/O 操作较慢。下面来看看在 Java 程序中使用输入/输出流的数据流图 (注意：图中输入/输出均以 Java Program 为参照物)：

![enter image description here](http://images.gitbook.cn/3477e7a0-6a63-11e8-b138-6bde7fa5e463)

#### 2.2 Buffer

Buffer 是一个对象，它包含一些要写入或读出的数据。在 NIO 中，数据是放入 Buffer 对象的，而在 IO 中，数据是直接写入或者读到 Stream 对象的。应用程序不能直接对 Channel 进行读写操作，而必须通过 Buffer 来进行，即 Channel 是通过 Buffer 来读写数据的，如下示意图。

![enter image description here](http://images.gitbook.cn/3e896b60-6a63-11e8-969d-bdea77f73f2b)

在 NIO 中，所有的数据都是用 Buffer 处理的，它是 NIO 读写数据的中转池。Buffer 实质上是一个数组，通常是一个字节数据，但也可以是其他类型的数组。但一个缓冲区不仅仅是一个数组，重要的是它提供了对数据的结构化访问，而且还可以跟踪系统的读写进程。

**Buffer 读写步骤：**

使用 Buffer 读写数据一般遵循以下四个步骤：

- 写入数据到 Buffer；
- 调用 flip() 方法；
- 从 Buffer 中读取数据；
- 调用 clear() 方法或者 compact() 方法。

当向 Buffer 写入数据时，Buffer 会记录下写了多少数据。一旦要读取数据，需要通过 flip() 方法将 Buffer 从写模式切换到读模式。在读模式下，可以读取之前写入到 Buffer 的所有数据。

一旦读完了所有的数据，就需要清空缓冲区，让它可以再次被写入。有两种方式能清空缓冲区：调用 clear() 或 compact() 方法。clear() 方法会清空整个缓冲区。compact() 方法只会清除已经读过的数据。任何未读的数据都被移到缓冲区的起始处，新写入的数据将放到缓冲区未读数据的后面。

**Buffer种类：**

Buffer主要有如下几种：

> CharBuffer、DoubleBuffer、IntBuffer、LongBuffer、ByteBuffer、ShortBuffer、FloatBuffer

上述缓冲区覆盖了我们可以通过 I/O 发送的基本数据类型：

> characters，double，int，long，byte，short和float

#### 2.3 Buffer 结构

Buffer 有几个重要的属性如下，

![enter image description here](http://images.gitbook.cn/38fb3cd0-6966-11e8-8614-25103124b6ae)

结合如下结构图解释一下：

- position 记录当前读取或者写入的位置，写模式下等于当前写入的单位数据数量，从写模式切换到读模式时，置为 0，在读的过程中等于当前读取单位数据的数量；
- limit 代表最多能写入或者读取多少单位的数据，写模式下等于最大容量 capacity；从写模式切换到读模式时，等于 position，然后再将 position 置为 0，所以，读模式下，limit 表示最大可读取的数据量，这个值与实际写入的数量相等。
- capacity 表示 buffer 容量，创建时分配。

之所以介绍这一节是为了更好的解释为何写/读模式切换时需要调用 flip() 方法，通过上述解释，相信读者已经明白为何写/读模式切换需要调用 flip() 方法了。附上 flip() 方法的解释：

> Flips this buffer. The limit is set to the current position and then the position is set to zero. If the mark is defined then it is discarded.

![enter image description here](http://images.gitbook.cn/6c669990-6a63-11e8-969d-bdea77f73f2b)

#### 2.4 Buffer 的选择

通常情况下，操作系统的一次写操作分为两步：

1. 将数据从用户空间拷贝到系统空间（即从 JVM 内存拷贝到系统内存）。
2. 从系统空间往网卡写。

同理，读操作也分为两步：

1. 将数据从网卡拷贝到系统空间；
2. 将数据从系统空间拷贝到用户空间。

对于 NIO 来说，缓存的使用可以使用[DirectByteBuffer](http://gitbook.cn/gitchat/activity/5af07387585c260a21a32b97)（堆外内存，关于堆外内存，如果存在疑问请阅读我的[另一篇文章](http://gitbook.cn/gitchat/activity/5af07387585c260a21a32b97)）和 HeapByteBuffer（堆外内存）。如果使用了 DirectByteBuffer，一般来说可以减少一次系统空间到用户空间的拷贝。但Buffer创建和销毁的成本更高，更不宜维护，通常会用内存池来提高性能。如果数据量比较小的中小应用情况下，可以考虑使用 heapBuffer；反之可以用 directBuffer。

#### 2.5 Buffer 使用实例

```
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class IO_Demo
{
    public static void main(String[] args) throws Exception
    {
        String infile = "D:\\Users\\data.txt";
        String outfile = "D:\\Users\\dataO.txt";
        // 获取源文件和目标文件的输入输出流
        FileInputStream fin = new FileInputStream(infile);
        FileOutputStream fout = new FileOutputStream(outfile);
        // 获取输入输出通道
        FileChannel fileChannelIn = fin.getChannel();
        FileChannel fileChannelOut = fout.getChannel();
        // 创建缓冲区，分配1K堆内存
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        while (true)
        {
            // clear方法重设缓冲区，使它可以接受读入的数据
            buffer.clear();
            // 从输入通道中读取数据数据并写入buffer
            int r = fileChannelIn.read(buffer);
            // read方法返回读取的字节数，可能为零，如果该通道已到达流的末尾，则返回-1
            if (r == -1)
            {
                break;
            }
            // flip方法将 buffer从写模式切换到读模式
            buffer.flip();
            // 从buffer中读取数据然后写入到输出通道中
            fileChannelOut.write(buffer);
        }
        //关闭通道
        fileChannelOut.close();
        fileChannelIn.close();
        fout.close();
        fin.close();
    }
}
```

### 3. NIO 核心对象 Channel 详解

#### 3.1 简要回顾

第一节中的例子所示，当执行：`fileChannelOut.write(buffer)`，便将一个 buffer 写到了一个通道中。相较于缓冲区，通道更加抽象，因此，我在第一节详细介绍了缓冲区，并穿插了通道的内容。

引用 Java NIO 中权威的说法：通道是 I/O 传输发生时通过的入口，而缓冲区是这些数据传输的来源或目标。对于离开缓冲区的传输，需要输出的数据被置于一个缓冲区，然后写入通道。对于传回缓冲区的传输，一个通道将数据写入缓冲区中。

> 例如：
>
> 有一个服务器通道serverChannel，一个客户端通道 SocketChannel clientChannel；
>
> 服务器缓冲区：serverBuffer，客户端缓冲区：clientBuffer。
>
> - 当服务器想向客户端发送数据时，需要调用 clientChannel.write(serverBuffer)。当客户端要读时，调用 clientChannel.read(clientBuffer)
> - 当客户端想向服务器发送数据时，需要调用 serverChannel.write(clientBuffer)。当服务器要读时，调用 serverChannel.read(serverBuffer)

#### 3.2 关于 Channel

Channel 是一个对象，可以通过它读取和写入数据。可以把它看做 IO 中的流。但是它和流相比还有一些不同：

- Channel 是双向的，既可以读又可以写，而流是单向的（所谓输入/输出流）；
- Channel 可以进行异步的读写；
- 对 Channel 的读写必须通过 buffer 对象；

正如上面提到的，所有数据都通过 Buffer 对象处理，所以，输出操作时不会将字节直接写入到 Channel 中，而是将数据写入到 Buffer 中；同样，输入操作也不会从 Channel 中读取字节，而是将数据从 Channel 读入 Buffer，再从 Buffer 获取这个字节。

因为 Channel 是双向的，所以 Channel 可以比流更好地反映出底层操作系统的真实情况。特别是在 Unix 模型中，底层操作系统通常都是双向的。

在 Java NIO 中 Channel 主要有如下几种类型：

- FileChannel：从文件读取数据的
- DatagramChannel：读写 UDP 网络协议数据
- SocketChannel：读写 TCP 网络协议数据
- ServerSocketChannel：可以监听 TCP 连接

### 4. NIO 核心对象 Selector 详解

#### 4.1 关于 Selector

通道和缓冲区的机制，使得 Java NIO 实现了同步非阻塞 IO 模式，在此种方式下，用户进程发起一个 IO 操作以后便可返回做其它事情，而无需阻塞地等待 IO 事件的就绪，但是用户进程需要时不时的询问 IO 操作是否就绪，这就要求用户进程不停的去询问，从而引入不必要的 CPU 资源浪费。

鉴于此，需要有一个机制来监管这些 IO 事件，如果一个 Channel 不能读写（返回 0），我们可以把这件事记下来，然后切换到其它就绪的连接（channel）继续进行读写。在 Java NIO 中，这个工作由 selector 来完成，这就是所谓的同步。

Selector 是一个对象，它可以接受多个 Channel 注册，监听各个 Channel 上发生的事件，并且能够根据事件情况决定 Channel 读写。这样，通过一个线程可以管理多个 Channel，从而避免为每个 Channel 创建一个线程，节约了系统资源。如果你的应用打开了多个连接（Channel），但每个连接的流量都很低，使用 Selector 就会很方便。

要使用 Selector，就需要向 Selector 注册 Channel，然后调用它的 select() 方法。这个方法会一直阻塞到某个注册的通道有事件就绪，这就是所说的轮询。一旦这个方法返回，线程就可以处理这些事件。

下面这幅图展示了一个线程处理 3 个 Channel 的情况：

![enter image description here](http://images.gitbook.cn/beab0300-6a70-11e8-969d-bdea77f73f2b)

#### 4.2 Selector 使用

**1.创建 Selector 对象**

通过 Selector.open() 方法，我们可以创建一个选择器：

```
Selector selector = Selector.open();
```

**2. 将 Channel 注册到选择器中**

为了使用选择器管理 Channel，我们需要将 Channel 注册到选择器中:

```
channel.configureBlocking(false);
SelectionKey key =channel.register(selector,SelectionKey.OP_READ);
```

注意，注册的 Channel 必须设置成异步模式才可以，否则异步 IO 就无法工作，这就意味着我们不能把一个 FileChannel 注册到 Selector，因为 FileChannel 没有异步模式，但是网络编程中的 SocketChannel 是可以的。

需要注意 register() 方法的第二个参数，它是一个“interest set”，意思是注册的 Selector 对 Channel 中的哪些事件感兴趣，事件类型有四种（对应 SelectionKey 的四个常量）：

- OP_ACCEPT
- OP_CONNECT
- OP_READ
- OP_WRITE

通道触发了一个事件意思是该事件已经 Ready（就绪）。所以，某个 Channel 成功连接到另一个服务器称为 Connect Ready。一个 ServerSocketChannel 准备好接收新连接称为 Accept Ready，一个有数据可读的通道可以说是 Read Ready，等待写数据的通道可以说是 Write Ready。

如果你对多个事件感兴趣，可以通过 or 操作符来连接这些常量：

```
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE; 
```

**3. 关于 SelectionKey**

请注意对 register() 的调用的返回值是一个 SelectionKey。 SelectionKey 代表这个通道在此 Selector 上的这个注册。当某个 Selector 通知您某个传入事件时，它是通过提供对应于该事件的 SelectionKey 来进行的。SelectionKey 还可以用于取消通道的注册。SelectionKey 中包含如下属性：

- The interest set
- The ready set
- The Channel
- The Selector
- An attached object (optional)

这几个属性很好理解，interest set 代表感兴趣事件的集合；ready set 代表通道已经准备就绪的操作的集合；Channel 和 Selector：我们可以通过 SelectionKey 获得 Selector 和注册的 Channel；attached object ：可以将一个对象或者更多信息 attach 到 SelectionKey 上，这样就能方便的识别某个给定的通道。例如，可以附加与通道一起使用的 Buffer。

SelectionKey 还有几个重要的方法，用于检测 Channel 中什么事件或操作已经就绪，它们都会返回一个布尔类型：

```
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable(); 
```

**4. 通过 SelectionKeys() 遍历**

从上文我们知道，对于每一个注册到 Selector 中的 Channel 都有一个对应的 SelectionKey，那么，多个 Channel 注册到 Selector 中，必然形成一个 SelectionKey 集合，通过 SelectionKeys() 方法可以获取这个集合。因此，当 Selector 检测到有通道就绪后，我们可以通过调用 selector.selectedKeys() 方法返回的 SelectionKey 集合来遍历，进而获得就绪的 Channel，再进一步处理。实例代码如下：

```
// 获取注册到selector中的Channel对应的selectionKey集合
Set<SelectionKey> selectedKeys = selector.selectedKeys();
// 通过迭代器进行遍历，获取已经就绪的Channel，
Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
while(keyIterator.hasNext()) 
{ 
SelectionKey key = keyIterator.next();
if(key.isAcceptable()) 
{
// a connection was accepted by a ServerSocketChannel.
// 可通过Channel()方法获取就绪的Channel并进一步处理
SocketChannel channel = (SocketChannel)key.channel();
// TODO 

} 
else if (key.isConnectable()) 
{
// TODO
} 
else if (key.isReadable()) 
{
// TODO
} 
else if (key.isWritable()) 
{
// TODO
}
// 删除处理过的事件
keyIterator.remove();
}
```

**5. select() 方法检测 Selector 中是否有 Channel 就绪**

在进行遍历之前，我们至少应该知道是否已经有 Channel 就绪，否则遍历完全是徒劳。Selector 提供了 select() 方法，它会返回一个数值，代表就绪 Channel 的数量，如果没有 Channel 就绪，将一直阻塞。除了 select()，还有其它几种，如下：

- int select()： 阻塞到至少有一个通道就绪；
- int select(long timeout)：select() 一样，除了最长会阻塞 timeout 毫秒（参数），超时后返回0，表示没有通道就绪；
- int selectNow()：不会阻塞，不管什么通道就绪都立刻返回，此方法执行非阻塞的选择操作。如果自从前一次选择操作后，没有通道变成可选择的，则此方法直接返回零。

加入 select() 方法后的代码：

```
// 反复循环,等待IO
while(true)
{
// 等待某信道就绪,将一直阻塞，直到有通道就绪
selector.select();
// 获取注册到selector中的Channel对应的selectionKey集合
Set<SelectionKey> selectedKeys = selector.selectedKeys();
// 通过迭代器进行遍历，获取已经就绪的Channel，
Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
while(keyIterator.hasNext()) 
{ 
SelectionKey key = keyIterator.next();
if(key.isAcceptable()) 
{
// a connection was accepted by a ServerSocketChannel.
// 可通过Channel()方法获取就绪的Channel并进一步处理
SocketChannel channel = (SocketChannel)key.channel();
// TODO 

} 
else if (key.isConnectable()) 
{
// TODO
} 
else if (key.isReadable()) 
{
// TODO
} 
else if (key.isWritable()) 
{
// TODO
}
// 删除处理过的事件
keyIterator.remove();
}
}
```

#### 4.3 Selector 使用实例

```
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class TCPServer
{
    // 超时时间，单位毫秒
    private static final int TimeOut = 3000;
    // 本地监听端口
    private static final int ListenPort = 1978;

    public static void main(String[] args) throws IOException
    {
        // 创建选择器
        Selector selector = Selector.open();
        // 打开监听信道
        ServerSocketChannel listenerChannel = ServerSocketChannel.open();
        // 与本地端口绑定
        listenerChannel.socket().bind(new InetSocketAddress(ListenPort));
        // 设置为非阻塞模式
        listenerChannel.configureBlocking(false);
        // 将选择器绑定到监听信道,只有非阻塞信道才可以注册选择器.并在注册过程中指出该信道可以进行Accept操作
        // 一个serversocket channel准备好接收新进入的连接称为“接收就绪”
        listenerChannel.register(selector, SelectionKey.OP_ACCEPT);

        // 反复循环,等待IO
        while (true)
        {
            // 等待某信道就绪(或超时)
            int keys = selector.select(TimeOut);
            //刚启动时连续输出0，client连接后一直输出1
            if (keys == 0)
            {
                System.out.println("独自等待.");
                continue;
            }

            // 取得迭代器，遍历每一个注册的通道
            Set<SelectionKey> set = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = set.iterator();

            while (keyIterator.hasNext())
            {
                SelectionKey key = keyIterator.next();
                if(key.isAcceptable()) 
                {
                    // a connection was accepted by a ServerSocketChannel.
                    // 可通过Channel()方法获取就绪的Channel并进一步处理
                    SocketChannel channel = (SocketChannel)key.channel();
                    // TODO 

                } 
                else if (key.isConnectable()) 
                {
                    // TODO
                } 
                else if (key.isReadable()) 
                {
                    // TODO
                } 
                else if (key.isWritable()) 
                {
                    // TODO
                }
                // 删除处理过的事件
                keyIterator.remove();
            }
        }
    }
}
```

特别说明：例子中 selector 只注册了一个 Channel，注册多个 Channel 操作类似。如下：

```
for (int i=0; i<3; i++)
{
// 打开监听信道
ServerSocketChannel listenerChannel = ServerSocketChannel.open();
// 与本地端口绑定
listenerChannel.socket().bind(new InetSocketAddress(ListenPort+i));
// 设置为非阻塞模式
listenerChannel.configureBlocking(false);
// 注册到selector中
listenerChannel.register(selector, SelectionKey.OP_ACCEPT);
}
```

在上面的例子中，对于通道 IO 事件的处理并没有给出具体方法，在此，举一个更详细的例子：

```
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;

public class NIO_Learning
{
    private static final int BUF_SIZE = 256;
    private static final int TIMEOUT = 3000;

    public static void main(String args[]) throws Exception
    {
        // 打开服务端 Socket
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        // 打开 Selector
        Selector selector = Selector.open();

        // 服务端 Socket 监听8080端口, 并配置为非阻塞模式
        serverSocketChannel.socket().bind(new InetSocketAddress(8080));
        serverSocketChannel.configureBlocking(false);

        // 将 channel 注册到 selector 中.
        // 通常我们都是先注册一个 OP_ACCEPT 事件, 然后在 OP_ACCEPT 到来时, 再将这个 Channel 的 OP_READ 注册到 Selector 中.
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (true)
        {
            // 通过调用 select 方法, 阻塞地等待 channel I/O 可操作
            if (selector.select(TIMEOUT) == 0)
            {
                System.out.print("超时等待...");
                continue;
            }

            // 获取 I/O 操作就绪的 SelectionKey, 通过 SelectionKey 可以知道哪些 Channel 的哪类 I/O 操作已经就绪.
            Iterator<SelectionKey> keyIterator = selector.selectedKeys().iterator();

            while (keyIterator.hasNext())
            {
                SelectionKey key = keyIterator.next();
                // 当获取一个 SelectionKey 后, 就要将它删除, 表示我们已经对这个 IO 事件进行了处理.
                keyIterator.remove();

                if (key.isAcceptable())
                {
                    // 当 OP_ACCEPT 事件到来时, 我们就有从 ServerSocketChannel 中获取一个 SocketChannel,
                    // 代表客户端的连接
                    // 注意, 在 OP_ACCEPT 事件中, 从 key.channel() 返回的 Channel 是 ServerSocketChannel.
                    // 而在 OP_WRITE 和 OP_READ 中, 从 key.channel() 返回的是 SocketChannel.
                    SocketChannel clientChannel = ((ServerSocketChannel) key.channel()).accept();
                    clientChannel.configureBlocking(false);
                    //在 OP_ACCEPT 到来时, 再将这个 Channel 的 OP_READ 注册到 Selector 中.
                    // 注意, 这里我们如果没有设置 OP_READ 的话, 即 interest set 仍然是 OP_CONNECT 的话, 那么 select 方法会一直直接返回.
                    clientChannel.register(key.selector(), SelectionKey.OP_READ,
                            ByteBuffer.allocate(BUF_SIZE));
                }

                if (key.isReadable())
                {
                    SocketChannel clientChannel = (SocketChannel) key.channel();
                    ByteBuffer buf = (ByteBuffer) key.attachment();
                    long bytesRead = clientChannel.read(buf);
                    if (bytesRead == -1)
                    {
                        clientChannel.close();
                    }
                    else if (bytesRead > 0)
                    {
                        key.interestOps(SelectionKey.OP_READ | SelectionKey.OP_WRITE);
                        System.out.println("Get data length: " + bytesRead);
                    }
                }

                if (key.isValid() && key.isWritable())
                {
                    ByteBuffer buf = (ByteBuffer) key.attachment();
                    buf.flip();
                    SocketChannel clientChannel = (SocketChannel) key.channel();

                    clientChannel.write(buf);

                    if (!buf.hasRemaining())
                    {
                        key.interestOps(SelectionKey.OP_READ);
                    }
                    buf.compact();
                }
            }
        }
    }
}
```

#### 4.4 小结

如从上述实例所示，可以将多个 Channel 注册到同一个 Selector 对象上，实现一个线程同时监控多个 Channel 的请求状态，但有一个不容忽视的缺陷：

所有读/写请求以及对新连接请求的处理都在同一个线程中处理，无法充分利用多 CPU 的优势，同时读/写操作也会阻塞对新连接请求的处理。因此，有必要进行优化，可以引入多线程，并行处理多个读/写操作。

一种优化策略是：

将 Selector 进一步分解为 Reactor，从而将不同的感兴趣事件分开，每一个 Reactor 只负责一种感兴趣的事件。这样做的好处是：

- 分离阻塞级别，减少了轮询的时间；
- 线程无需遍历 set 以找到自己感兴趣的事件，因为得到的 set 中仅包含自己感兴趣的事件。下文将要介绍的 Reactor 模式便是这种优化思想的一种实现。

### 5. Reactor 模式介绍

【特别说明】Reactor 模式是一种设计模式，不是为 Java 量身定做的，C++ 等语言也可以实现 Reactor 模式。因此，切忌先入为主将 Reactor 的概念与 Java 对号入座。

关于 Reactor 模式，Wikipedia 上释义为：

> “The reactor design pattern is an event handling pattern for handling service requests delivered concurrently by one or more inputs. The service handler then demultiplexes the incoming requests and dispatches them synchronously to associated request handlers.”

从这个描述中，我们知道 Reactor 模式首先是事件驱动的，有一个或多个并发输入源，有一个 Service Handler，有多个 Request Handlers；这个 Service Handler 会同步的将输入的请求（Event）多路复用的分发给相应的 Request Handler。如果用图来表达：

![enter image description here](http://images.gitbook.cn/1f3477c0-6b2e-11e8-b102-2915f3681805)

从结构上，这有点类似生产者消费者模式，即有一个或多个生产者将事件放入一个 Queue 中，而一个或多个消费者主动的从这个 Queue 中 Poll 事件来处理；而 Reactor 模式则并没有 Queue 来做缓冲，每当一个 Event 输入到 Service Handler 之后，该 Service Handler 会主动的根据不同的 Event 类型将其分发给对应的 Request Handler 来处理。

#### 5.1 Reactor 模式实现

Reactor 模式中有些概念对于初学者来说会比较生涩难懂，鉴于此，我们先来看一个例子（出处：[Reactor An Object Behavioral Pattern for Demultiplexing and Dispatching Handles for Synchronous Events](http://www.dre.vanderbilt.edu/~schmidt/PDF/reactor-siemens.pdf)），该例子以 Logging Server 来分析 Reactor 模式，其实现完全遵循 Reactor 描述。Logging Server 中的 Reactor 模式实现分两个部分：Client 连接到 Logging Server 和 Client 向 Logging Server 写 Log。因而对它的描述分成这两个部分。

**Part-I：Client 连接到 Logging Server**

![enter image description here](http://images.gitbook.cn/3deb44a0-6b29-11e8-8431-a75f9cd1b0ae)

交互步骤：

1. Logging Server 注册 LoggingAcceptor 到 InitiationDispatcher。
2. Logging Server 调用 InitiationDispatcher 的 handle_events() 方法启动。
3. InitiationDispatcher 内部调用 select() 方法（Synchronous Event Demultiplexer），阻塞等待 Client 连接。
4. Client 连接到 Logging Server。
5. InitiationDisptcher 中的 select() 方法返回，并通知 LoggingAcceptor 有新的连接到来。
6. LoggingAcceptor 调用 accept 方法 accept 这个新连接。
7. LoggingAcceptor 创建新的 LoggingHandler。
8. 新的 LoggingHandler 注册到 InitiationDispatcher 中(同时也注册到 Synchonous Event Demultiplexer 中)，等待 Client 发起写 log 请求。

**Part-II：Client向Logging Server写Log** ![enter image description here](http://images.gitbook.cn/4d8e6950-6b29-11e8-84bc-a9fc89cfdb07)

交互步骤：

1. Client 发送 log 到 Logging server。
2. InitiationDispatcher 监测到相应的 Handle 中有事件发生，返回阻塞等待，根据返回的 Handle 找到 LoggingHandler，并回调 LoggingHandler 中的 handle_event() 方法。
3. LoggingHandler 中的 handle_event() 方法中读取 Handle 中的 log 信息。
4. 将接收到的 log 写入到日志文件、数据库等设备中。3.4 步骤循环直到当前日志处理完成。
5. 返回到 InitiationDispatcher 等待下一次日志写请求。

**小结**

看了上述 Reactor 模式的例子，是不是感觉和 Selector 的例子有些类似？不用怀疑，它们不只是类似，在 Java 的 NIO 中，对 Reactor 模式有无缝的支持。

#### 5.2 Java NIO 对 Reactor 的实现（单线程版）

网上有很多关于 Java NIO 对 Reactor 实现的例子，这些例子的素材基本都是出自纽约州立大学 Doug Lea 的 PPT：[Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)，在此，我同样借鉴 Doug Lea 的思想进行举例。

##### **5.2.1 服务端代码**

**1. 创建 Reactor**

仔细阅读代码，你会发现与 Selector 实例很相似。

```
import java.io.IOException;
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.util.Iterator;
import java.util.Set;

public class Reactor implements Runnable
{
    public final Selector selector;
    public final ServerSocketChannel serverSocketChannel;

    public Reactor(int port) throws IOException
    {
        // 创建选择器
        selector = Selector.open();
        // 打开监听信道
        serverSocketChannel = ServerSocketChannel.open();
        // 与本地端口绑定
        InetSocketAddress inetSocketAddress = new InetSocketAddress(InetAddress.getLocalHost(), port);
        System.out.println(InetAddress.getLocalHost());
        serverSocketChannel.socket().bind(inetSocketAddress);
        // 设置为非阻塞模式,只有非阻塞信道才可以注册选择器,否则异步IO就无法工作
        serverSocketChannel.configureBlocking(false);
        // 向selector注册该channel    
        SelectionKey selectionKey = serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        // 利用selectionKey的attache功能绑定Acceptor
        selectionKey.attach(new Acceptor(this));
    }

    @Override
    public void run()
    {
        try
        {
            while (!Thread.interrupted())
            {
                System.out.println("等待....");
                // 阻塞等待某信道就绪
                selector.select();

                // 取得已就绪事件的key集合，遍历每一个注册的通道(本文作为举例，只有一个通道)
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectionKeys.iterator();
                //Selector如果发现channel有OP_ACCEPT或READ事件发生，下列遍历就会进行。  
                while (it.hasNext())
                {  
                    SelectionKey selectionKey = it.next();
                    // 根据事件的key进行调度
                    dispatch(selectionKey);
                    // 删除处理过的事件
                    it.remove();
                }
            }
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
    }

    void dispatch(SelectionKey key)
    {
        Runnable r = (Runnable) (key.attachment());
        if (r != null)
        {
            r.run();//调度事件对应的处理流程
        }
    }

}
```

与 Selector 实例最大的不同在于，使用 selectionKey 对象的 attach() 方法，为每一个注册到 Selector 中的 Channel 都绑定了一个 Handler，当 Channel 有事件就绪时，可通过调度 Handler 进行处理。

**2. 创建 Acceptor**

Acceptor 本质上也是一个 Handler，只不过特殊一些：用于处理 Client 的连接请求，即它感兴趣的通道事件类型是 OP_ACCEPT。

```
import java.io.IOException;
import java.nio.channels.SelectionKey;
import java.nio.channels.SocketChannel;

public class Acceptor implements Runnable
{
    private Reactor reactor;

    public Acceptor(Reactor reactor)
    {
        this.reactor = reactor;
    }

    @Override
    public void run()
    {
        try
        {
            // 接受client连接请求
            SocketChannel sc = reactor.serverSocketChannel.accept();   
            System.out.println(sc.socket().getRemoteSocketAddress().toString() + " is connected.");

            if (sc != null)
            {
                sc.configureBlocking(false);
                // SocketChannel向selector注册一个OP_READ事件，然后返回该通道的key 
                SelectionKey sk = sc.register(reactor.selector, SelectionKey.OP_READ); 
                // 使一个阻塞住的selector操作立即返回
                reactor.selector.wakeup(); 
                // 通过key为新的通道绑定一个附加的TCPHandler对象
                sk.attach(new TCPHandler(sk, sc)); 
            }
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
    }
}
```

**3. 创建 Handler**

从 Acceptor 的代码中可以看出，当服务端接受客户端的连接请求后（通过 serverSocketChannel 调用 accept() 获得了一个新的通道 SocketChannel），会将新的通道注册到 Selector 并绑定一个 Handler，用于处理读写事件，本例中，这个 Handler 命名为 TCPHandler。

```
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.SocketChannel;

public class TCPHandler implements Runnable
{
    private final SelectionKey sk;
    private final SocketChannel sc;

    int state;

    public TCPHandler(SelectionKey sk, SocketChannel sc)
    {
        this.sk = sk;
        this.sc = sc;
        state = 0; // 初始状态设置为READING，client连接后，读取client请求  
    }

    @Override
    public void run()
    {
        try
        {
            if (state == 0)
                read(); // 读取客户端请求数据
            else
                send(); // 向客户端发送反馈数据

        }
        catch (IOException e)
        {
            System.out.println("[Warning!] A client has been closed.");
            closeChannel();
        }
    }

    private void closeChannel()
    {
        try
        {
            sk.cancel();
            sc.close();
        }
        catch (IOException e1)
        {
            e1.printStackTrace();
        }
    }

    private synchronized void read() throws IOException
    {
        // 创建一个读取通道数据的缓冲区
        ByteBuffer inputBuffer = ByteBuffer.allocate(1024);
        inputBuffer.clear();

        int numBytes = sc.read(inputBuffer); // 读取数据
        if (numBytes == -1)
        {
            System.out.println("[Warning!] A client has been closed.");
            closeChannel();
            return;
        }
        // 将读取的字节转换为字符串类型
        String str = new String(inputBuffer.array()); 
        if ((str != null) && !str.equals(" "))
        {
            process(str); // 进一步处理获取的数据
            System.out.println(sc.socket().getRemoteSocketAddress().toString() + " > " + str);
            // 切换状态，准备向客户端发送反馈数据
            state = 1; 
            // 通过key改变通道注册的事件类型
            sk.interestOps(SelectionKey.OP_WRITE);
            sk.selector().wakeup();
        }
    }

    private void send() throws IOException
    {
        String str = "Your message has sent to " + sc.socket().getLocalSocketAddress().toString()
                + "\r\n";
        // 创建发送数据的缓存区并写入数据
        ByteBuffer outputBuffer = ByteBuffer.allocate(1024);

        outputBuffer.put(str.getBytes());
        outputBuffer.flip();
        // 向客户端发送反馈数据
        sc.write(outputBuffer); 
        // 切换状态
        state = 0; 
        // 通过key改变通道注册的事件类型
        sk.interestOps(SelectionKey.OP_READ);
        sk.selector().wakeup();
    }

    void process(String str)
    {
        // do process(decode, logically process, encode)..
        // 略
    }

}
```

**4. 服务端主函数 Main**

```
import java.io.IOException;

public class Main
{

    public static void main(String[] args)
    {
        try
        {
            Reactor temp = new Reactor(12345);
            temp.run();
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
    }
}
```

##### **5.2.2 客户端代码**

客户端相对来说很简单，不做过多解释。

```
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.InetAddress;
import java.net.Socket;
import java.net.UnknownHostException;

public class Client
{

    public static void main(String[] args) throws UnknownHostException
    {
        // 初始化待连接服务端的地址
        String hostName = InetAddress.getLocalHost().toString();
        int port = 12345;
        try
        {
            Socket client = new Socket(InetAddress.getLocalHost(), port); 
            System.out.println("Connected to " + InetAddress.getLocalHost().toString());

            // 分别创建客户端端输入输出流和控制台输入流
            PrintWriter out = new PrintWriter(client.getOutputStream());
            BufferedReader in = new BufferedReader(new InputStreamReader(client.getInputStream()));
            BufferedReader stdIn = new BufferedReader(new InputStreamReader(System.in));
            String input;

            while ((input = stdIn.readLine()) != null)
            { 
                //打印来自服务端的反馈数据
                out.println(input); 
                out.flush(); 
                if (input.equals("exit"))
                {
                    break;
                }
                // 打印控制台输入，即客户端向服务端发送的数据
                System.out.println("server: " + in.readLine());
            }
            client.close();
            System.out.println("client stop.");
        }
        catch (UnknownHostException e)
        {
            System.err.println("Don't know about host: " + hostName);
        }
        catch (IOException e)
        {
            System.err.println("Couldn't get I/O for the socket connection");
        }

    }

}
```

**小结**

上面例子是 Java NIO 对 Reactor 模式的单线程版实现，多线程版及更多内容可参考 Doug Lea 的 PPT：[Scalable IO In Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)。

#### 5.3 Reactor 模式介绍

上面的实例，建议读者自己运行一下，切实感受一下 Reactor 模式。有了前面这么多铺垫，是时候推出 Reactor 模式了，先来看一下 Reactor 模式的模块关系图（图片出自：[Reactor An Object Behavioral Pattern for Demultiplexing and Dispatching Handles for Synchronous Events](http://www.dre.vanderbilt.edu/~schmidt/PDF/reactor-siemens.pdf)）：

![enter image description here](http://images.gitbook.cn/7391c9f0-6b9a-11e8-abbc-337f0fb12aa9)

**概念解释：**

- Handle：即操作系统中的句柄，是对资源在操作系统层面上的一种抽象，它可以是打开的文件、一个连接 (Socket)、Timer 等。由于 Reactor 模式一般使用在网络编程中，因而这里一般指 Socket Handle，即一个网络连接（Connection，即 Java NIO 中的 Channel）。这个 Channel 注册到 Synchronous Event Demultiplexer 中，以监听 Handle 中发生的事件，对 ServerSocketChannnel 可以是 CONNECT 事件，对 SocketChannel 可以是 READ、WRITE、CLOSE 事件等。
- Synchronous Event Demultiplexer：阻塞等待一系列的 Handle 中的事件到来，如果阻塞等待返回，即表示在返回的 Handle 中可以不阻塞的执行返回的事件类型。这个模块一般使用操作系统的 select 来实现。在 Java NIO 中用 Selector 来封装，当 Selector.select() 返回时，可以调用 Selector 的 selectedKeys() 方法获取 `Set<SelectionKey>`，一个 SelectionKey 表达一个有事件发生的 Channel 以及该 Channel 上的事件类型。上图的“Synchronous Event Demultiplexer ---notifies--> Handle”的流程如果是对的，那内部实现应该是 select() 方法在事件到来后会先设置 Handle 的状态，然后返回。
- Initiation Dispatcher：用于管理 Event Handler，即 EventHandler 的容器，用以注册、移除 EventHandler 等；另外，它还作为 Reactor 模式的入口调用 Synchronous Event Demultiplexer 的 select 方法以阻塞等待事件返回，当阻塞等待返回时，根据事件发生的 Handle 将其分发给对应的 Event Handler 处理，即回调 EventHandler 中的 handle_event() 方法。
- Event Handler：定义事件处理方法：handle_event()，以供 InitiationDispatcher 回调使用。
- Concrete Event Handler：事件 EventHandler 接口，实现特定事件处理逻辑。

**Reactor 模式各个模块之间的交互**

下面这幅图同样出自上面列出的文章。结合图片可以很清晰的看出各个模块交互的过程，以 main 函数为入口，流程如下：

![enter image description here](http://images.gitbook.cn/e51cd670-6b9d-11e8-b29b-970daea2f15e)

1. 初始化 InitiationDispatcher，并初始化一个 Handle 到 EventHandler 的 Map。
2. 注册 EventHandler 到 InitiationDispatcher 中，每个 EventHandler 包含对相应 Handle 的引用，从而建立 Handle 到 EventHandler 的映射（Map）。
3. 调用 InitiationDispatcher 的 handle_events() 方法以启动 Event Loop。在 Event Loop 中，调用 select() 方法（Synchronous Event Demultiplexer）阻塞等待 Event 发生。
4. 当某个或某些 Handle 的 Event 发生后，select() 方法返回，InitiationDispatcher 根据返回的 Handle 找到注册的 EventHandler，并回调该 EventHandler 的 handle_events() 方法。
5. 在 EventHandler 的 handle_events() 方法中还可以向 InitiationDispatcher 中注册新的 Eventhandler，比如对 AcceptorEventHandler 来，当有新的 client 连接时，它会产生新的 EventHandler 以处理新的连接，并注册到 InitiationDispatcher 中。

**致谢：**

本文的一些图片和文字引用了一些博客和论文，尊重原创是每一个写作者应坚守的底线，在此，将本文引用过的文章一一列出，以表敬意：

1. [Reactor An Object Behavioral Pattern for Demultiplexing and Dispatching Handles for Synchronous Events](http://www.dre.vanderbilt.edu/~schmidt/PDF/reactor-siemens.pdf)
2. Doug Lea 的 PPT [Scalable IO In Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)
3. 上善若水的博客《[Reactor 模式详解](http://www.blogjava.net/DLevin/archive/2015/09/02/427045.html)》
4. Heaven-Wang的博客《[JavaNIO 详解](https://blog.csdn.net/suifeng3051/article/details/48441629)》