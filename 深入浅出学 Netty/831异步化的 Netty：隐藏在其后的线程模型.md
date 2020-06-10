# 8/31异步化的 Netty：隐藏在其后的线程模型

## 异步化的 Netty

Netty 在官网首页有这么一句话介绍自己

> Netty is *an asynchronous event-driven network application framework* for rapid development of maintainable high performance protocol servers & clients.

异步的特性甚至还摆在事件驱动之前，可见其重要性。Netty 的异步操作在代码中随处可见，几个比较重要的地方返回都是`ChannelFuture`接口。先来重温下在什么地方会遇到异步接口。

第一处，也是最为常见，在服务端引导程序绑定监听端口的地方，代码如下

```java
ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap.group(boss, worker).channel(NioServerSocketChannel.class);
ChannelFuture sync = serverBootstrap.bind(2323).sync();
```

`bind`方法返回的`ChannelFuture`对象有两种使用方式：

- 第一种，在允许阻塞的上下文中，可以直接使用`sync`或者`await`方法等待异步任务完成。
- 第二种，当前上下文不能阻塞的情况，可以调用`ChannelFuture`的`addListener`方法注册一个回调函数。该回调函数会被异步任务被完成后触发。

第二处使用返回异步任务的地方则是紧随监听端口绑定成功之后，为了不让main方法退出，需要去等待服务端程序的关闭，代码如下

```java
ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap.group(boss, worker).channel(NioServerSocketChannel.class);
ChannelFuture sync = serverBootstrap.bind(2323).sync();
sync.channel().closeFuture().sync();
```

通过`sync.channel()`的调用获得了绑定监听端口成功的服务端通道。而后通过`closeFuture`方法获得了该服务端通道的关闭异步任务。只有在服务端通道关闭后，该异步任务才会完成。通常而言，服务端通道关闭就意味着整个网络服务应用的下线。因此在这里等待通道的关闭实质就是等待整体应用的结束。

这里的等待是有着实质的重要作用的，一般而言，我们在初始化`ServerBootstrap`都会传入工作线程池，也就是`EventLoopGroup`对象。这些线程池在服务端通道关闭后，其内部的任务队列可能还剩余一些任务没有完成。此时为了数据的正确性考虑，不能强制关闭整个程序，否则就可能造成数据不一致或其他异常。因此需要在`EventLoopGroup`上执行优雅关闭，也就是调用`shutdownGracefully`方法。该方法会首先切换`EventLoopGroup`到关闭状态从而拒绝新的任务的加入，然后在任务队列的任务都处理完成后，停止线程的运行。从而确保整体应用是在正常有序的状态下退出的。

一般而言，在服务端的代码中我们的写法都是：

```java
public static void main(String[] args)
    {
        EventLoopGroup  boss            = new NioEventLoopGroup(1);
        EventLoopGroup  worker          = new NioEventLoopGroup();
        try
        {

            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(boss, worker);
            serverBootstrap.channel(NioServerSocketChannel.class);
            ChannelFuture bind = serverBootstrap.bind(2356);
            bind.sync();
            Channel serverChannel = bind.channel();
            serverChannel.closeFuture().sync();
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
```

如果没有`serverChannel.closeFuture().sync();`就会直接结束`main`方法，然后执行`finally`中的内容，这会导致运行中的应用中断。根据上文的介绍，除了使用`sync`等待，还可以添加监听器，在监听器中进行线程池的优雅关闭。不过相对来说，`sync`等待这种写法会比较常见和简洁一些。

第三处则是在数据写出的地方，先看实例代码

```java
public static void main(String[] args)
    {
        EventLoopGroup      boss   = new NioEventLoopGroup(1);
        EventLoopGroup      worker = new NioEventLoopGroup();
        final AtomicInteger count  = new AtomicInteger();
        try
        {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(boss, worker);
            serverBootstrap.channel(NioServerSocketChannel.class);
            serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>()
            {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception
                {
                    ch.pipeline().addLast(new ChannelInboundHandlerAdapter()
                    {
                        @Override
                        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
                        {
                            ChannelFuture future = ctx.write(msg);
                            future.addListener(new ChannelFutureListener()
                            {
                                @Override
                                public void operationComplete(ChannelFuture future) throws Exception
                                {
                                    //消息数量统计
                                    count.incrementAndGet();
                                }
                            });
                        }
                    });
                }
            });
            ChannelFuture bind = serverBootstrap.bind(2356);
            bind.sync();
            Channel serverChannel = bind.channel();
            serverChannel.closeFuture().sync();
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
```

这个例子中我们实现简单的消息发出的总数的功能。可以注意到，我们将计数的增加放在了任务的监听器之中实现。

这是因为执行`io.netty.channel.ChannelOutboundInvoker#write(java.lang.Object)`方法，该方法是一个异步方法，直接返回了`ChannelFuture`实例，当方法返回的时候，消息可能还没有写入到Socket发送缓冲区。如果在方法返回的时候就进行累加，累加的结果就和实际情况存在偏差了。

而在异步任务的监听器中进行累加，当方法`operationComplete`被调用时，数据已经被写入socket发送缓存区。此时进行计数累加的结果就是真正的消息发出的总数了（不考虑 TCP 通道中断的情况下）。

异步的好处显而易见，不让线程阻塞在 IO 操作上，可以尽可能的利用CPU 资源。不过异步并不是“免费午餐”，支持异步实现需要背后高效合理的线程模式设计。这也是下文要分析的内容。

## 从《Scalable IO in Java》看线程模型

在操作系统支持 IO 多路复用能力后，针对这种能力，衍生了专门使用其的编程模型，也就是`Reactor pattern`。网络上的翻译都是反应堆模式，但是觉得一点都不达意，也没有找到好的翻译，因此下文就直接称呼为 reactor 模式。

在 Java1.4 支持 NIO 后，并发界的大佬 Doug Lea 发了一个ppt，《Scalable IO In Java》。在其中阐述了使用如何将reactor 模式应用在 NIO 的编程上。一口吃不成胖子，一步步来看下线程模型是如何变化的。

早期的时候，只有 BIO 模式，也就是一个线程服务一个客户端的模型。使用图来表达的话，就类似于

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191011141158.png)

一个服务端线程阻塞在 ServerSocket 的`accept`方法，一旦方法返回，有客户端链接建立，则创建一个 handler 处理这个连接的数据读取，解码，业务计算，编码，响应数据发送。通常而言，一个 handler 运行在一个独立的线程中。

简单粗暴好理解，唯一的问题就是这种模式扩展性很差，随着客户端数量的增多，创建的线程也越来越多，而线程的创建消耗内存资源，线程的调度和上下文保存更是消耗许多 CPU 资源的。一旦线程创建的太多了，甚至会有个拐点，处理效率断崖式下跌。

这种模型在 JDK1.4 之前是唯一的选择。在 JDK 提供了 NIO 之后，情况有了彻底的改观。Reactor 模式也开始登场。首先来看下，基础reactor 模式，如下图

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191011180011.png)

在之前的文章我们介绍过，基于 IO 复用能力，一个`Selector`可以监控数以千计的客户端连接。基础 Reactor 模式也是如此，使用一个多路同步监控器来监控多个连接上的 IO 事件。这些 IO 事件可以包括连接的接口和建立（accept），连接可读（read*ready）,连接可写（write*ready）。所以这个多路同步监控器可以监控服务端通道以及在接受客户端后创建的客户端通道。

当多路同步监控器监控到 IO 事件发生时，则会将事件传递给派发器。而派发器则会将事件传递给合适的事件处理器执行处理，也就是handler，具体仍然是处理读取，解码，计算，编码，发送等逻辑。

基础 Reactor 模式中，多路同步监控器，派发器，事件处理器全部运行在同一个线程中，这个线程称之为 Reactor 线程。只不过由于 IO 多路复用的能力，所以一个线程也可以支撑数以千计的连接。这个模式当中，多路同步监控器这个角色由 NIO 中的`selector`来承担，而派发器和事件处理器则是用户自行实现的。

显然，基础 Reactor 模式无法有效利用多核 CPU。由于 IO 复用和非阻塞式 IO 的存在，使得基于 Reactor 模式下，io 事件的处理不再是阻塞式，可以有效的利用 CPU。但是解码，计算和编码则无法预计。为此，可以将非 IO 动作：解码、计算、编码这三个动作从 handler 中剥离，使用单独的 Processor 处理。并且让 Processor 运行在独立的线程中，以此来提高 reactor 线程的运行效率。通常来说， processor 是运行在线程池中，doug lea 给这个起了个名字，worker thread pools。

演进后的模型如下图

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191011210453.png)

随着连接数的增多，仅仅依靠一个 Reactor 处理读写事件也会显得效率不够以及对 CPU 的利用不充分了。此时，可以将reactor线程扩充。考虑到只有一个服务端通道，且其 IO 事件只有客户端的连接事件；而客户端通道的事件主要是读事件和写事件，与服务端通道存在明显的区分。因此将 Reactor 区分为 2 类：执行服务端通道的接入类和执行客户端通道的读写类。细化来说，此时存在 2 组 reactor 线程：

- 主 Reactor 线程，只有一个，负责处理服务端通道上的 IO 事件，也就是客户端的接入。
- 子 Reactor 线程，通常多个，负责处理客户端通道上的 IO 事件，也就是客户端链接的读写就绪。

简单而言，就是主 Reactor 在收到客户端接入时，选择一个子 Reactor 线程，将客户端链接分发给它，进行后续的读写处理。而子Reactor 线程在遇到非 IO 工作时，继续分发给 Worker thread pool 处理。

使用图来表达这个模式就是

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191011214835.png)

在 Doug lea 的 PPT 中将只增加了 Worker thread pools 的模式和多线程 Reactor 模式统称为 Reactor 模式的多线程版本。但是在大部分的中文博客中将前者称之为多线程 Reactor 模式，将后者称之为主从 Reactor模式，未能查找到这种起名的来源，不过后文会沿用这种传统，将上述三种模式称之为：单线程 Reactor 模式，多线程 Reactor 模式，主从 Reactor 模式。

## Netty 的线程模型

Netty 可以通过配置，来实现不同的线程模型。而且需要改动的代码相当的少。首先来看第一种，单线程 Reactor 模式，对应的代码如下

```java
class HelloWorld
{
    public static void main(String[] args)
    {
        EventLoopGroup boss   = new NioEventLoopGroup(1);
        try
        {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(boss).channel(NioServerSocketChannel.class);
            serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>()
            {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception
                {
                     ch.pipeline().addLast(new DecoderHandler());
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
        }
    }
}
```

在`main`方法的第一行中，我们将 Boss 线程组的大小设置为 1，这意味着该`NioEventLoopGroup`中的线程只有 1 个。而后续 Netty的服务引导程序的 Group 配置中，我们只传递了该 Group。这使得在Netty 发生的所有操作都是运行在这个线程上。此时，Netty 的线程模式就是单线程 Reactor 模式。当然，这种配置方式比较少出现在实践中。

更常规的配置方式是创建两个`EventLoopGroup`，并且将之配置到`ServerBootStrap`。如下

```java
class HelloWorld
{
    public static void main(String[] args)
    {
        EventLoopGroup boss   = new NioEventLoopGroup(1);
        EventLoopGroup worker   = new NioEventLoopGroup();
        try
        {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(boss, worker).channel(NioServerSocketChannel.class);
            serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>()
            {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception
                {
                      ch.pipeline().addLast(new DecoderHandler());
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

比第一个程序多了一个`worker`的`EventLoopGroup`。默认情况下，`NioEventLoopGroup`的线程数是内核数的 2 倍。在配置的时候也与第一个不同，同时传递了 2 个进去`serverBootstrap.group(boss, worker)`。Boss 组用于服务端通道处理客户端接入就绪事件，Worker 组用于处理客户端通道读写就绪事件。简单而言，就是 Boss 组线程监听着服务端的接入就绪事件，并且在处理成功后将接入的客户端通道分发给 Worker 组。之后worker组就监控在其上的客户端通道的读写就绪事件。

此时在客户端通道上的读写，编解码，计算都是运行在 Worker 组的线程中。为了避免并发问题，一个通道只会绑定在一个线程上。Netty 将这种方式称之为串行化设计。在这种配置模式下，串行化设计可以理解为一个通道上的所有 ChannelHandler 都运行同一个线程上，避免了上下文切换，减少了同步的损耗，同时应用整体又是并行的。实践证明，这种模式的性能是十分高效的。

每一个`NioEventLoopGroup`都管理着一定数量的`NioEventLoop`线程，而一个`NioEventLoop`都会持有一个`Selector`对象，也就是`NioEventLoop`线程实际上就是reactor线程。因此上述的这种配置模式下，Netty 此时的模式比较接近于没有使用 Worker thread Pool 的主从 reactor 模式。

当然，Netty 也提供了 Worker thread pool 模式的支持。但是这种方式比较少用，Netty 官网不能提到，社区中也没有描述。具体的代码如下

```java
class HelloWorld
{
    public static void main(String[] args)
    {
        EventLoopGroup       boss        = new NioEventLoopGroup(1);
        EventLoopGroup       worker      = new NioEventLoopGroup();
        final EventLoopGroup childWorker = new NioEventLoopGroup();

        try
        {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(boss, worker).channel(NioServerSocketChannel.class);
            serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>()
            {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception
                {
                   ch.pipeline().addLast(childWorker,new DecoderHandler());
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
            childWorker.shutdownGracefully();
        }
    }
}
```

代码主要的改变就是增加了一个`childWorker`组。并且在客户端通道的管道对象添加ChannelHandler时，选择关联一个`EventExecutorGroup`。这意味对应的`ChannelHandler`运行在关联的这个`EventExecutorGroup`的某个线程中（这个关联关系是在add类方法中被确定的）。

如果每一个处理器都被额外的`EventExecutorGroup`关联，那么一个通道上除了读写调用工作在通道关联的 Reactor 线程上，剩余的`ChannelHandler`都可以工作在自定义线程上。此种情况，就是《Scalable IO In Java》提到的 Worker thread pools 模式。更贴近于多线程 Reactor 模式。在这种模式下，串行化则有了另外一种含义，那就是：一个`Channel`上的某个具体的`ChannelHandler`总是运行在一个固定的线程中，不会被并发，所有对该`Channelhandler`的调用都是串行的。

## 综述

上面讨论了 reactor 模式及其多线程版本，以及 Netty 不同的设置对应的不同模式。在 Netty 中有一个设计原则就是避免对一个通道的并发操作，甚至于避免对一个通道上的一个具体的`Channelhandler`的并发操作。对`ChannelHandler`的调用都是串行执行的，因此用户在实现业务代码的时候就需要考虑并发安全的问题，简化了代码的处理。为了实现这个串行设计的目标，Netty 中的通道和 ChannelHandler 都被绑定到一个具体的线程上。在没有显示绑定的情况，`ChannelHandler`会被绑定到其关联的通道绑定的线程上。

理解了这一点，对于为什么 Netty 许多操作都是返回一个异步任务对象就很容易了。因为如果当前线程不是需要操作的通道或者`ChannelHandler`绑定的线程，则 Netty 都会为当前操作生成一个对象，投入到其绑定的线程的任务队列，让线程自行取出并且执行。而投入完毕的时候任务并不会马上完成，因此只能返回一个异步任务对象给调用者。而如果操作线程就是当前通道或者`ChannelHandler`绑定的线程则可以执行具体的操作而不用将操作包装为任务进行投递。但是为了接口的统一，此时也是返回一个异步任务对象。只不过这个返回的异步任务对象，在返回的时候就已经是已完成的状态了。

## 总结与思考

本文讨论了《Scalable IO In Java》中提到的几种在 NIO 使用场景下的线程模式变种，详细分析了其变化和演进的思路和修改点。并且以Netty 自身的支持为切入，分析了 Netty 的线程模型，以及 Netty 如何通过参数变化来支持不同的线程模型。对线程模型的理解，也就能理解Netty中的一些并发安全保证和异步化接口背后的原理。

关于 Netty 还有一块很重要的内容，也是其主要的 API 来源，就是事件驱动。Netty 在官网对自己的描述就是一个事件驱动的框架。下一篇文章，我们就会来详细的讲解 Netty 中的事件究竟是个怎么回事以及如何在基于事件的模型下开发 Netty 程序。