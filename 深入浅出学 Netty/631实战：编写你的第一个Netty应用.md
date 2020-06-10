# 6/31实战：编写你的第一个Netty应用

## 前言

从[《实战：使用 Java NIO 编写服务端应用》](https://gitbook.cn/gitchat/column/5daeb1e3669f843a1a4af134/topic/5daff451bae3b42c1fa7d61e)可以看到，使用 Java 原生的 NIO 接口编写一个服务端应用程序相对比较复杂，并且需要考虑许多的细节问题。而上文举的例子本身无论是在协议制定还是功能需求上都是比较简单的。而且这个例子还是一个功能定制化的应用。如果需要完成具备可扩展性的 IO 框架性质的程序，方便应用在其上进行二次开发，无疑更加困难。线程模型的设计，供二次开发的应用程序接口的设计，内存的管理，并发的控制等等一系列难度都在摆在面前。

让情况更糟糕的是，Java 的 NIO 实现存在 Bug。选择器`Selector`的`select`方法在特定的情况会在没有 IO 事件发生的情况返回，且该bug触发后，`select`就会一直在没有 IO 事件的情况返回而不是阻塞等待。通常的编程模型中，都是将`Selector`的`select`放置在一个 while 循环中。所以一旦这个 Bug 被触发，就会导致程序快速的轮训，进而CPU100%耗尽系统资源。表现在应用程序外部就是系统仍然在运行，但是无法对客户端请求作出响应了。

由于基于原生NIO接口进行开发的难度比较大，一般公司也不会选择这种方式。幸好，在 NIO 领域，有一个事实上的网络 IO 框架标准，它就是Netty。它对原生的 NIO 接口进行封装，简化了网络应用的开发，提升了开发效率。而且它也解决了原生NIO 的 CPU 100% 这个 Bug。目前，大部分公司在 Java 上构建的网络应用和许多涉及到网络调用的框架，比如 RPC 框架，其网络层都是使用 Netty 来构建的。

## Netty是什么

关于Netty是什么，能提供什么样的能力，再没有比官网的描述更加清晰的了。来看下Netty对自己的定位

> Netty是一个基于NIO的网络客户端框架，支持快速简单的开发诸如协议服务器客户端这样的网络应用。它极大的提升了开发一个TCP或UDP的socket服务器的编程效率，简化了编程工作。
>
> 快速和简便，并不意味着开发出来的程序会面临可扩展性或者性能问题。Netty 吸取了很多协议的开发经验，诸如 FTP，SMTP，HTTP 以及各种二进制协议以及文本协议等并且被精心的设计。正因如此，Netty 成功的达成了简便开发、高性能、良好稳定性和扩展性的目标而不是妥协。

Netty 的自我描述基本上已经涵盖了 Netty 的功能范围和作用，简单而言就是：

- 能够完成基于TCP和UDP的socket开发
- 开发简便，高性能、良好的稳定性和扩展性

文字的描述略显抽象，我们来看下Netty的架构设计，如图

![img](https://netty.io/images/components.png)

从总体上被分为三个部分：

- 核心层：核心层支撑着整个 Netty，基于Netty的应用程序开发都不可避免的要与核心层打交道。细分之下，核心层主要有3个大的模块：
- Netty 自定义的数据读写模块ByteBuf。提供了比NIO原生ByteBuffer更容易理解的接口和更强大的功能。
- 全局统一的抽象通信接口。将TCP和UDP都是通道的方式抽象，在应用层面提供了统一的处理接口。业务处理都只需要实现`ChannelHandler`接口。该接口用于处理通道上的数据，且不需要区分底层通道的不同。
- 可扩展的事件模型。Netty 是一个事件驱动的模型，它将一个通道当中会发生的动作都抽象成为事件。而业务处理仅需要针对自己感兴趣的事件进行处理即可。
- 传输模型：支持不同类型的传输通道抽象，诸如有 HTTP，TCP&UDP，以及为了方便测试的 In-VM 实例内通道。
- 协议支持：有了稳定高效的内核层，就可以开始应用程序开发了。网络应用开发无法绕开对各种协议的实现。而Netty非常贴心的已经提供了市面上大部分常见协议的支持。因此在编解码这块几乎不需要任何投入就能得到工业级的编解码支持。

## 编写第一个Netty应用

Netty 的目的是让网络应用开发在简单高效的基础上提供更强大的性能和更好的稳定性。要使用 Netty 进行网络应用开发，首先应该要了解 Netty 为我们提供了哪些 API 以及开发组件。为了不显得那么抽象，我们先将之前的echo的例子改造为使用Netty 来开发。为了进一步降低入门的难度，我们也像 NIO 一开始那样限定要求，具体来说：

- 客户端消息定长为 16 字节
- TCP 不发生拆包粘包
- 客户端一次只发送一条消息

基于此，我们将 echo 简化版服务器改造为使用 Netty 作为服务器框架进行编写，修改后的代码如下：

```java
public class MainDemo
{
    public static void main(String[] args)
    {
        EventLoopGroup boss   = new NioEventLoopGroup();
        EventLoopGroup worker = new NioEventLoopGroup();
        try
        {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(boss, worker)//代码一：设置整个服务端执行所使用的线程池
                    .channel(NioServerSocketChannel.class)//代码二：设置服务端使用的通道类型，作为服务端程序，必然选择ServerSocketChannel下面的各种实现类。如果是基于JDK的NIO的话，则是选择NioServerSocketChannel
                    .childHandler(new ChannelInitializer<SocketChannel>()
            {
                //代码三：该方法在客户端链接成功建立时被触发，方法入参就是新建立的客户端通道对象
                @Override
                protected void initChannel(SocketChannel ch) throws Exception
                {
                    //代码四：获取socket通道的管道对象，并且向其中添加处理器
                    ch.pipeline().addLast(new ChannelInboundHandlerAdapter()
                    {
                        @Override
                        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
                        {
                            ByteBuf buf = (ByteBuf) msg;
                            if (buf.readerIndex() != 0)
                            {
                                throw new IllegalStateException();
                            }
                            if (buf.writerIndex() != 16)
                            {
                                throw new IllegalStateException();
                            }
                            ctx.writeAndFlush(buf);
                        }

                        @Override
                        public void channelInactive(ChannelHandlerContext ctx) throws Exception
                        {
                            //客户端关闭通道后，收到消息
                            System.out.println("客户端关闭通道");
                        }
                    });
                }
            });
            ChannelFuture sync = serverBootstrap.bind(2323).sync();
            sync.channel().closeFuture().sync();
        }
        catch (InterruptedException e)
        {
            e.printStackTrace();
        }
        finally
        {
            boss.shutdownGracefully();
            worker.shutdownGracefully();
        }
    }
}
```

最直观的感觉，从代码量来看，比使用原生 NIO 接口进行代码编写少了许多。而且除了服务端启动所需要设置的一些最小化启动参数，包含有线程池，服务端的通道类型；剩下的代码都和业务逻辑直接相关。即使不了解Netty的同学，在看到这个应用程序以及相关的类名，多少都能猜测出来作用。

简单，往往是接触 Netty 的同学的第一眼印象，这也的确是 Netty 努力追求的目标。

在这个示例中，不需要用户处理任何和选择器相关的代码，只需要在有数据出现的时候，也就是方法`channelRead`被调用的时候，读取其中的数据，做出合适的处理。发送数据也是一样的简单，只需要调用方法`io.netty.channel.ChannelOutboundInvoker#writeAndFlush(java.lang.Object)`将数据写出即可。不需要像 NIO 一样考虑 Socket 缓存区满了，等待下一次可写的时机等等。如果通道关闭，也不需要考虑注销选择键等，而且全程也没有选择键。

总之，Netty做的一切都是为了让网络应用开发更加简单。当然，Netty不仅能用于服务端的开发，也一样可以用于客户端的开发。下面是和这个echo服务端搭配的客户端的代码

```java
public class ClientDemo
{
    public static void main(String[] args) throws InterruptedException
    {
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(new NioEventLoopGroup());
        bootstrap.channel(NioSocketChannel.class).handler(new ChannelInboundHandlerAdapter()
        {
            @Override
            public void channelActive(ChannelHandlerContext ctx) throws Exception
            {
                ByteBuf buffer = PooledByteBufAllocator.DEFAULT.buffer(8);//代码一
                buffer.writeInt(1).writeInt(2).writeInt(3).writeInt(4);
                ctx.writeAndFlush(buffer);
            }

            @Override
            public void channelReadComplete(ChannelHandlerContext ctx) throws Exception
            {
                System.out.println("数据发送");
            }

            @Override
            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
            {
                ByteBuf buf = (ByteBuf) msg;
                System.out.println(buf.readInt());
                System.out.println(buf.readInt());
                System.out.println(buf.readInt());
                System.out.println(buf.readInt());
                ctx.close();
                System.out.println("通道关闭");
            }
        });
        ChannelFuture sync = bootstrap.connect("127.0.0.1", 2323).sync();
    }
}
```

客户端的代码也是一样的简洁明了，在连接服务端成功后，方法`channelActive`就会被调用，此时业务上申请了一个`ByteBuf`（Netty自行设计的数据承载组件）用于写入业务数据，并且向服务端发送。这里我们可以注意到一个细节，也就是*代码一*处。申请的内存大小是 8 个字节。但是却写入了 4 个 int 整型，也就是 16 字节。但是程序没有报错，没有像 NIO 的`ByteBuffer`一样抛出指针越界异常。这是因为 Netty 的`ByteBuf`具备自动扩容能力，可以在写入容量不足时自动扩展，这是 Netty 为我们提供的诸多便利之一。

Netty 本身的实现天然的为我们解决了 Socket 写出缓存区不足时，高效写出的问题。但是 TCP 的粘包拆包问题仍然是需要自己解决的。我们将上面的环境约束去掉2条，即，允许通道发生 TCP 拆包粘包以及客户端一次不止发送一条消息。

这种情况处理也简单，只需要在读取数据之前，进行下解码工作，从数据流中分割出一个消息的”帧“即可。对于以特定结束符作为消息分隔符的解码器，Netty 提供了一个直接的抽象实现``.修改的代码部分很简单，仅需要在通道中添加处理的首先添加一个分隔符解码器，代码如下

```java
ByteBuf delimiter = PooledByteBufAllocator.DEFAULT.buffer(2);
delimiter.writeChar('\r');
ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1000, false, delimiter))
```

管道中的处理器按照添加的顺序生效。管道是一个责任链设计模式的实现，后文会详细展开，这里先不赘述。当读取数据的时候，从 Socket 通道中读取的数据首先被解码器先处理。

这个解码器的实现的原理和 NIO 中我们给出的例子差不多，也是遍历通道中已经读取出来的数据，在发现分隔符时分割出一个消息的对应区域。一旦从数据流中分割出来一个消息，则会向后面的处理器传递。触发的也仍然是`channelRead`方法。

Netty的实现也已经帮我们考虑好了多个消息拆分，以及消息拆分后，通道中可能留有不足一个消息的剩余数据的情况。开发者需要做的仅仅只是传入存储消息分割符的`ByteBuf`对象。`DelimiterBasedFrameDecoder`构造方法的前两个参数：第一个参数是最大能接受的消息大小，如果超出了还有发现消息分隔符则抛出异常；第二个参数是提取到消息后，是否要将消息末尾的分割符去除。

只是经过上面一个步骤的修改，并且只是添加了三行代码的基础上，使用 Netty 开发的 echo 服务器在功能和适用性上完全媲美了我们使用原生 NIO 经过几次修改才得到最终版本。且使用原生 NIO 开发的程序还无法规避 NIO 恼人的 CPU 100% bug。不止如此，Netty 的这个版本，在性能上还远远超出了原生 NIO 开发的版本。因为其读写数据载体`ByteBuf`实现了池化，内存控制由 Netty 自行实现，也就不再有GC的参与，极大的减少了 GC 压力，降低了系统的负载。而在开发效率上，简单明了的网络抽象，事件驱动的消息机制，自动扩缩容的`ByteBuf`都极大了方便了开发者对应用程序的开发。

## 总结与思考

本文当中，我们第一次接触了 Netty，了解了 Netty 的基本概念。并且通过 Netty，很容易的就构建了一个符合工业生产需求的 echo 服务端程序。通过这个里，对 Netty 有了第一次的直观印象。不过仅仅只是一个简单的例子，还无法领略到Netty 的精髓。Netty 为广大开发者隐藏了很多网络开发中的复杂内容，也做了足够好的抽象。但是要能熟练掌握 Netty 的开发技巧，对组件有深入的了解是必不可少的。

下一篇文章，我们会对 Netty 的核心组件进行详细展开，掌握了这些，才能真正的使用 Netty 进行网络应用开发。