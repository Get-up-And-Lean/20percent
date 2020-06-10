# 7/31深入剖析 Netty 的核心组件

## 前言

从上一个章节的学习，对使用 Netty 进行网络应用开发已经有了一个初步的感性的印象。但是仅仅依靠这种简单的介绍就上手 Netty 还是过于勉强了。所以本文会剖析 Netty 中的重点组件。通过对这些组件的理解和掌握，从底层熟悉和理解 Netty。

## 核心组件分析

### ByteBuf

在 JDK 的 NIO 中，我们学习到了其原生的数据承载组件`ByteBuffer`。`ByteBuffer`的体验着实不太好，读写状态的区别，还有`flip`这种乍看下不直观的操作。

Netty 设计了自己的数据存储组件`ByteBuf`。和`ByteBuffer`一样，`ByteBuf`也是代表了一段连续的二进制数据空间。同样的，`ByteBuf`也按照数据存储的位置区分为：数据存储在堆上的`HeapByteBuf`和数据存储在直接内存的`DirectByteBuf`。

`ByteBuf`的设计目标是简化用户的使用，所以不像`ByteBuffer`那样还有读写状态的区分，`ByteBuf`当中只有 2 个指针：读指针和写指针。读指针指向的位置，意味着可以从这个位置开始读取；写指针指向的位置，意味着可以将数据写入指向位置。

用图表来表达更为形象，刚申请一个`ByteBuf`的时候其状态如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191008164942.png)

此时读写指针均指向位置 0。此时没有数据可以读取，但是可以写入。

写入 4 个字节后，状态如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191008165032.png)

写指针随着数据的写入增加。此时读写之间存在着一片区域，这片区域就可以读取的内容区域。

接着读取 2 个字节后，状态如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191008165436.png)

读取数据时，读指针对应的增加读取的字节数。在读指针之前的数据则属于“不可读“的范畴。加了引号是因为读写指针可以在外部被修改，这也是将数据重复读取的原理。

从上面的三个示例来看，`ByteBuf`的使用十分简单，没有什么读写状态更加不需要翻转。`ByteBuf`带来的便利还不止如此，`ByteBuf`还具备自动扩容的能力。在Netty中申请一个`ByteBuf`都会指定一个初始容量，但是在写入的时候，如果剩余容量不足，则会自动扩容。扩容规则为：

1. 当写入后新的容量小于 512，则选择一个小于 512 但是大于容量且为 16 的倍数的值作为新容量。
2. 当写入后新的容量大于 512，则选择一个大于容量且为 2 的次方幂的值作为新容量。

这个扩容规则可以不用关心，其容量确定本质上是因为其采用的内存管理办法。我们只需要知道其可以自动扩容满足我们的写入要求即可。同时还有一点需要明确，扩容并不是我们想的直观的在原有的后续区域上继续增加新的写入区域，而是重新申请了一个满足写入大小要求的区域，将原先的数据复制到了新的区域。只不过在外部的用户无法感知到罢了。因此，为了减少因为扩容带来的数据复制引起的性能损耗，建议在初始化的时候选择一个相对合适的大小。

但是`ByteBuf`比上面说的要复杂的许多，我们来看看其相关的部分类图

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191008180351.png)

这里面只是一部分，`ByteBuf`是一个很庞大的继承体系。初略分的话大致是两个维度：

- 数据存储在堆和数据存储在直接内存的区别
- `ByteBuf`持有的内存区域是一次性的依靠 JVM 进行 GC，还是池化的内存依靠 Netty 自行管理的区别

在 Netty3 的年代，由于池化内存只是刚刚开发的功能，处于观望状态，所以官方的建议是推荐使用非池化的`ByteBuf`。

不过到了 Netty4，经过了验证和内存泄漏追踪功能的加入，池化内存也就成为了首选，也是官方所推荐的。使用池化内存可以有效的降低 JVM 的 GC 压力，平稳系统的 GC 毛刺。在高并发的场景下，性能表现更加稳定。而不是像 Nettty3 那样因为频繁的申请内存和 GC 回收，造成 GC 的 CPU 占用成折线式的不停抖动。

考虑到堆外内存在进行 Socket 读取和写入的时候可以减少一次内核态和用户态之间的数据拷贝，一般而言都是推荐使用堆外内存。

两点相结合，可以总结出在实践中我们推荐使用的`ByteBuf`的具体类型，也就是`PoolDirectByteBuf`。不过在编写代码的时候我们并不会直接实例化这个类。而是通过`io.netty.buffer.ByteBufAllocator`这个接口的`buffer`方法来获得具体的实例。可以通过`io.netty.buffer.ByteBufAllocator#DEFAULT`属性来获得系统默认的分配器。初始化的Netty会进行判断，如果当前是 Android，则使用非池化的分配器；其余情况使用池化的分配器。对于服务器应用而言，池化分配器肯定是不二选择。

#### CompositeByteBuf

Netty 的官网介绍自己的`ByteBuf`是一个具备零拷贝能力的富`ByteBuffer`实现。`ByteBuf`也的确提供了`ByteBuffer`更多的功能，但是零拷贝本身而言，对于`DirectByteBuf`和`DirectByteBuffer`而言底层都是相同的，都是使用了堆外内存本身的零拷贝的特性。不过 Netty 还有额外提供了自己实现的零拷贝特性的`ByteBuf`，它是一个虚拟组合式的`ByteBuf`视图，也就是这个章节的主角：`CompositeByteBuf`。

在应用程序的编写过程中，部分场景下存在着需要将多个`ByteBuf`合并的需求。此时简单的使用`io.netty.buffer.ByteBuf#writeBytes(io.netty.buffer.ByteBuf)`接口可以完成需求，不过就是需要额外的内存拷贝。

针对这种需要聚合多个`ByteBuf`的使用场景，Netty设计了`CompositeByteBuf`类。这个类代表着一个虚拟的`ByteBuf`，其内部是由多个`ByteBuf`实例组成的数组。每一个`ByteBuf`都代表着虚拟Buffer中的某一段数据。

举个例子，如下便是由三个`ByteBuf`实例组成的`CompositeByteBuf`。

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20190928003738.png)

可以看到，三个不同的`ByteBuf`实例分别映射了虚拟Buffer不同区域的部分。`CompositeByteBuf`通过聚合的方式，对外提供了一个整体的Buffer的效果。

这种虚拟视图在某种情况下特别的好用。比如说Http协议的实现上都是按照协议头和内容体进行区分，而协议头和内容体往往会采用不同的`ByteBuf`进行存放，因此其解析方式不同的原因。而当我们需要组装一个完整的Http报文的时候，如果将代表协议头和报文体的`ByteBuf`实例一起写到一个新的`ByteBuf`自然是可以满足需求，不过也带来了数据拷贝的消耗。此时使用`CompositeByteBuf`作为一个虚拟视图聚合2个`ByteBuf`，既能避免内存拷贝，又可以在对用户表现上呈现出一个完整单一的`ByteBuf`的效果。提升了开发效率和性能。

不过需要注意，`CompositeByteBuf`聚合了多个`ByteBuf`，其在数据的读写实现上都较单一的`ByteBuf`要复杂。特别是如果数据读写跨越了多个`ByteBuf`实际的承载时，由于多次的操作，性能会有一些影响。因此一般而言，累积动作很多时候仍然使用`ByteBuf`直接写入另外一个`ByteBuf`。比如Netty编解码的默认父类等。

### Channel

Netty 也实现了自己对于通道的抽象，以便在接口的层面上添加更多能力，同时也与 NIO 的通道区分开。对于 Channel 而言，我们通常不会接触到其接口，而是一般接触到它实际使用的实现类，比较常用的实现类主要有：

- **io.netty.channel.socket.nio.NioSocketChannel**：这个实现类一般是在网络编程中，引导程序帮助我们实例化的，而且实例化的时候传递给我们也是接口`io.netty.channel.socket.SocketChannel`，并不会让我们感知到这个具体的实现。这个类代表着一个具体的 TCP 通道。
- **io.netty.channel.socket.nio.NioServerSocketChannel**：这个实现类提供的是 TCP 协议下服务端监听 Socket 通道的能力。这个实现类一般直接将类对象传递给引导程序用于启动一个基于 TCP 协议的服务端。
- **io.netty.channel.epoll.EpollServerSocketChannel**：一般而言，Java 的服务端应用都是部署在 Linux 服务器上。在 Linux 环境上，Netty 提供了自己实现的，更为高效的基于 Epoll 实现方式的 IO 复用实现。在引导程序中将`NioServerSocketChannel`替换为`EpollServerSocketChannel`可以得到更高的性能。

一般而言，我们会在三个地方和 Channel 的不同实现打交道。

1. **情况一**：引导程序初始化完毕后，获得引导程序实际创建的 Channel 对象，等待其关闭；通过这个等待可以实现优雅退出以及在服务端关闭后执行一些业务逻辑。
2. **情况二**：在客户端链接通过`io.netty.channel.ChannelInitializer#initChannel(C)`方法被创建后，获得通道的管道对象`pipeline`，在其中添加处理器。
3. **情况三**：持有通道对象，通过通道对象来将数据写出。

下面逐个来看下具体的代码使用处。

**情况一**

首先来看下示例程序

```java
public class HelloWorld
{
    public static void main(String[] args)
    {
        NioEventLoopGroup eventLoopGroup = new NioEventLoopGroup();
        try
        {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(eventLoopGroup);
            serverBootstrap.channel(NioServerSocketChannel.class);
            serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>()
            {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception
                {
                }
            });
            /**代码一*/
            ChannelFuture bind = serverBootstrap.bind(2356);
            bind.sync();
            Channel serverChannel = bind.channel();
            serverChannel.closeFuture().sync();
            /**代码一*/
        }
        catch (InterruptedException e)
        {
            e.printStackTrace();
        }
        finally
        {
            eventLoopGroup.shutdownGracefully();
        }
    }
}
```

忽略掉一开始的那些，直接关注**代码一**这个部分。首先是引导程序开始为通道绑定一个被本地地址和监听端口。这一步会返回一个`ChannelFuture`对象，这个是Netty增强后适合`Channel`的`Future`接口。在绑定成功之前我们也不需要去处理什么业务，因此这里直接调用`sync`方法等待绑定成功。

之后服务端通道建立，监听端口绑定成功就可以开始接收客户端链接。为了能实现优雅关闭（尽可能让EventLoopGroup中的线程执行完毕退出），需要等待服务端程序自己退出后才能执行。因此这里通过`bind`这个`ChannelFuture`获得服务端通道对象，并且调用其`closeFuture`方法得到一个和退出动作关联的`future`，执行`sync`方法等待就可以在这行代码处等待通道的关闭，也就是等待服务端自己的退出。

**情况二**

一般而言我们初始化一个可以实际工作的客户端链接，都是通过`io.netty.channel.ChannelInitializer#initChannel`方法来实现的。该方法会在客户端链接被建立的时候被调用。并且入参就是建立后的客户端链接。

此时我们就可以在这个通道上调用`pipeline`方法获得与通道关联的管道，进而在其上添加处理器来完成业务逻辑处理。查看下示例代码

```java
protected void initChannel(SocketChannel ch) throws Exception
{
    ch.pipeline().addLast(new DecodeHandler());
    ch.pipeline().addLast(new BusinessHandler());
}
```

此时的这个`SocketChannel`的实现如果在 TCP 下就是`NioSocketChannel`、

**情况三**

Netty在两个实现类都提供了写出方法的具体实现，分别是

- **io.netty.channel.AbstractChannel#write(java.lang.Object)**
- **io.netty.channel.AbstractChannelHandlerContext#write(java.lang.Object)**

通道接口的写出方法实现就是实现一来提供的。其与实现二的区别在于：实现一的写出数据需要从管道的写出入口开始，经历一遍所有的处理器进行加工；实现二的写出数据则从当前的处理器开始，向下一个写出处理器传递直到最终被写出。

### pipeline

管道是 Netty 中的一个很重要的概念。其本身是一个责任链的实现，用来按照顺序来存储处理器。有一张图可以很形象说明管道的具体作用，如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20190823153817.png)

在管道中的处理器按照实现的接口功能区分为入站处理器和出站处理器两大类。

从通道读取的数据会传入管道，并且按照在管道中添加的顺序将入站数据按序在入站处理器之间传递。而出站处理器的处理顺序则是反过来的，当有数据需要写出的时候，按照在管道中添加的顺序的反序，数据被出站处理器中被传递，也就是从管道的末尾传递到头部，然后被写出到socket通道中。

一个管道对象与一个通道对象是绑定的且不可分割。在管道中添加和删除处理器是使用同步来保护的。而且又因为 Netty 在一个通道上执行的操作都是串行的，因此在运行期改变管道中的处理器分布是并发安全的。这种操作并不常见，一般用于在一个监听接口上可以实现两种不同的解析协议时，在编解码器中根据当前识别的头部信息来动态的调整后续的处理器流程。

对于管道，没有更多其他的内容，最为重要的就是知道它是一个责任链的处理模式，并且出站数据的处理顺序和入站数据的处理顺序相反即可。

### ChannelHandler

`ChannelHandler`可以说是和用户打交道最多和最直接的接口了。为了实现业务逻辑，我们需要在管道`pipeline`中添加我们的业务实现。不过`ChannelHandler`是顶层接口，我们一般不会直接使用，而是去实现以下的两个接口：

- **io.netty.channel.ChannelInboundHandler**：入站处理器接口，用于处理来自通道上的数据读取业务。这个接口定义了诸多的方法，一般而言我们用其适配类，避免实现所有的接口方法，只需要实现我们感兴趣的方法即可。这个适配类是`io.netty.channel.ChannelInboundHandlerAdapter`。
- **io.netty.channel.ChannelOutboundHandler**：出站处理器接口，用于将数据进行处理并且最终写出到通道上。与入站处理器一样，这个接口也提供了适配器类，即`io.netty.channel.ChannelOutboundHandlerAdapter`。

有一个与`ChannelHandler`紧密相连的接口，它就是`io.netty.channel.ChannelHandlerContext`。在处理器插入到管道中的时候，管道会为插入的处理器生成一个与之关联的`ChannelHandlerContext`对象。这个`ChannelHandlerContext`会持有着这个对应的处理器。实际上，管道当中存储的是`ChannelHandlerContext`对象，这些对象内部有指向前后`ChannelHandlerContext`的指针，以指针指向的方实现了一个双向链表。这个双向链表就对应了管道在数据处理上的两种方向：入站数据和出站数据。

一般而言我们都是在处理完入站数据后需要写出的时候比较常使用这个接口，比如下方代码：

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
{
    ctx.write(msg);
}
```

类似我们之前实现的 echo 服务器。在收到数据后想要写出，此时就是使用`ChannelHandlerContext`接口。上个章节说管道的时候提到`ChannelHandlerContext`上执行写出和在通道上直接执行写出的区别。此时在这边直接执行写出，则会从当前处理器开始，将数据沿着出站方向，传递给下一个出站处理器。而不会完整的从管道的出站入口开始，执行所有的出站处理器。

### EventLoop

`EventLoop`接口名很符合其实际承担的职责，在一个循环（`Loop`）中，不断的处理事件（`Event`）。在Netty的网络应用中，这些事件被区分为 2 类：

1. Netty 框架自己产生的IO事件
2. 用户业务产生的一些写动作和自定义事件处理。

`EventLoop`接口最经常和我们打交道的就是`NioEventLoopGroup`这个实现。在服务端引导程序`ServerBootStrap`需要填入 2 个`EventLoopGroup`实例作为入参。

也就是在代码

```java
EventLoopGroup  boss            = new NioEventLoopGroup(1);
EventLoopGroup  worker          = new NioEventLoopGroup();
ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap.group(boss, worker);
```

`ServerBootStrap`会将服务端通道绑定到 Boss 的`EventLoopGroup`上，而将创建的客户端链接绑定到`worker`组上。因为服务端通道只有一个，一个通道只能绑定到一个线程上，并且产生处理 IO 事件在该线程身上循环。所以 Boss 组的线程大小设置为 1 即可。而 worker 组使用默认参数即可，Netty 对默认的设置是当前CPU核数的 2 倍。

虽然在业务代码中不会直接与`EventLoop`打交道，但是`EventLoop`所代表的线程模型是 Netty 的核心设计，搞明白这个，对于理解 Netty 的运行原理，有很大的帮助，这个在下一个章节会专门进行展开。

## 总结与思考

本篇文章对 Netty 中开发者最经常打交道的五个组件：ByteBuf，Channel，pipeline，ChannelHandler、EventLoop 做了详细的说明和使用讲解。掌握了这五个组件，就可以开始使用Netty编写网络应用了。不过 Netty 带给我们的除了框架上的简化，也在于其异步化的编程模式。其异步编程模式与其背后的线程模型息息相关。

下一篇文章，我们就会带读者详细的展开对 Netty 线程模型的详细分析。