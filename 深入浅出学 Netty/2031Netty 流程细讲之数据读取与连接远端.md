# 20/31Netty 流程细讲之数据读取与连接远端

### 引言

前文我们介绍了 Netty 如何让 NioServerSocketChannel 服务端通道接受并且初始化一个客户端链接。对于一个客户端链接而言，主要的处理任务有两类：处理就绪事件和处理队列任务。其中就绪事件又可以再细分为读就绪，写就绪，连接就绪和接受就绪。接受就绪我们在上文已经分析过了，本文主要来分析读就绪和连接就绪事件。

### 数据读取

当 NioServerSocketChannel 初始化完成一个链接后，Netty 就可以在该链接上执行读取操作。不过在分析读取操作之前，我们先来复习下客户端链接初始化的一些细节。

#### **客户端链接初始化细节**

前文我们介绍了客户端链接在方法 `ServerBootstrapAcceptor#channelRead` 初始化的步骤，分别是：添加 ChannelInitializer 对象，设置配置项和属性，注册到 EventLoop 上。还记得前文提到过，Channel 注册到 EventLoop 上，最终会委托到一个方法 `AbstractChannel.AbstractUnsafe#register0`，该方法在将通道绑定到 EventLoop 对应的 Selector 上后，会执行一段触发 channelActive 事件的代码，如下：

```java
if(isActive())
{
    if(firstRegistration)
    {
        pipeline.fireChannelActive();
    }
    else if(config().isAutoRead())
    {
        beginRead();
    }
}
```

客户端链接是 NioSocketChannel 对象，其 isActive 方法实现在链接仍然打开的状态下返回 true，显然 firstRegistration 在客户端链接刚被服务端通道接受的情况下也是 true。因此，这里就会触发 channelActive 事件。

而对 channelActive 事件的响应，前文我们曾经介绍过，在 pipeline 的首节点中会进行，其代码如下：

```java
public void channelActive(ChannelHandlerContext ctx)
{
    ctx.fireChannelActive();
    readIfIsAutoRead();
}
private void readIfIsAutoRead()
{
    if(channel.config().isAutoRead())
    {
        channel.read();
    }
}
```

NioServerSocketChannel 和 NioSocketChannel 都会执行到这段代码（不过前者是在绑定端口时触发而后者是在注册到 EventLoop 就触发了）。这里的 read 和在 NioServerSocketChannel 一样，触发了 pipeline 的 read 事件，最终委托到方法 `AbstractChannel.AbstractUnsafe#beginRead`。来看下其代码：

```java
protected void doBeginRead() throws Exception
{
    final SelectionKey selectionKey = this.selectionKey;
    if(!selectionKey.isValid())
    {
        return;
    }
    readPending = true;
    final int interestOps = selectionKey.interestOps();
    if((interestOps & readInterestOp) == 0)
    {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

对于 NioSocketChannel 而言，其 readInterestOp 属性的值是 `SelectionKey#OP_READ`。也就是在这里，NioSocketChannel 在 Selector 上的就绪事件关注增加了对数据可读就绪的关注。

从这个调用中，也可以总结出 Channel.read 这个调用的思路。该调用触发通道上的 read 事件，并且最终传递到 HeadContext 也就是首节点，经过一系列委托调用，最终触发到 `AbstractNioChannel#doBeginRead`。而这个方法的作用就是更新通道注册的就绪事件，增加上 readInterestOp 属性代表的事件。对于 ServerChannel，这个值是 Accept；对于 SocketChannel，这个值是 Read。使用这个方法注册新的读取（可读 or 接入）事件后，就可以在下次进入 Selector.select 时就可以执行真正的读取动作了。

#### **读取数据并处理**

当 NioSocketChannel 注册到 EventLoop 上后，并且就绪事件关注修改为可读就绪事件关注后，EventLoop 线程就可以在其 run 方法中处理数据读取了。在章节五中我们已经分析过 run 方法的流程，这里不再赘述。我们直接分析处理可读就绪事件的方法 processSelectedKeys，前文分析过这个方法的内部细节，这里我们直接关注可读就绪的部分，也即是：

```java
if((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0)
{
    unsafe.read();
}
```

与之前不同，这里的 unsafe 实例是 NioByteUnsafe，下面来看下其 read 方法的实现，如下：

```java
public final void read()
{
     final ChannelConfig config = config();
     //代码①
     if(shouldBreakReadReady(config))
     {
         clearReadPending();
         return;
     }
     final ChannelPipeline pipeline = pipeline();
     final ByteBufAllocator allocator = config.getAllocator();
     final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
     //代码②
     allocHandle.reset(config);
     ByteBuf byteBuf = null;
     boolean close = false;
     try
     {
         do {
             //代码③
             byteBuf = allocHandle.allocate(allocator);
             //代码④
             allocHandle.lastBytesRead(doReadBytes(byteBuf));
             if(allocHandle.lastBytesRead() <= 0)
             {
                 byteBuf.release();
                 byteBuf = null;
                 close = allocHandle.lastBytesRead() < 0;
                 if(close)
                 {
                     readPending = false;
                 }
                 break;
             }
             allocHandle.incMessagesRead(1);
             readPending = false;
             pipeline.fireChannelRead(byteBuf);
             byteBuf = null;
             //代码⑤
         } while (allocHandle.continueReading());
         allocHandle.readComplete();
         pipeline.fireChannelReadComplete();
         if(close)
         {
             closeOnRead(pipeline);
         }
     }
     catch(Throwable t)
     {
         handleReadException(pipeline, byteBuf, t, close, allocHandle);
     }
     finally
     {
         //代码⑥
         if(!readPending && !config.isAutoRead())
         {
             removeReadOp();
         }
     }
}
```

代码虽然有点长，但是整体的脉络较为清晰，其流程可以概括如下：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191214130143.png)

首先我们说一下 readPending 这个属性，这个属性的命令会让人以为这个这个属性用于表达当前通道是否正在读取。但这个属性实际表达的含义是当前通道在注册读取后**是否未曾读取到数据**。该属性在调用方法 Channel.read 或者 ChannelHandlerContext.read 时被设置为 true，当这个通道成功读取到一个消息（比如 NioServerSocketChannel 类型的通道）或成功读取到字节消息亦或 EOF（比如 NioSocketChannel 类型的通道）后就被修改回 false。默认配置下，一个链接是被配置为自动读取模式，也就是只要通道上有数据就会触发 channelRead 事件，所以这个属性大多数时候并不起到什么作用。但是在需要应用逻辑来控制是否读取数据的场合，也就是通道的 autoRead 模式被关闭的情况下，读取数据是依靠应用程序主动调用 Channel.read 或 ChannelHandlerContext.read 的场合，该属性就可以用于决定取消读就绪事件关注的取消时机。这个可以在后文的代码分析中看到。

接着我们来分析下代码，首先是代码①。shouldBreakReadReady 方法用于判断是否中断读取，其内容倒也简单，如果底层 Socket 已经关闭输入流或链接终止（可能是经过 Netty 关闭，也可能是外部导致的关闭），且通道本身不支持半关闭或 Netty 之前已经确认过该通道输入流被关闭（通过标识位留存判断）；则当前读取中断，方法直接返回。该方法主要是避免后续无谓的读取操作导致的内存分配行为。

接着来看方法②。该方法重置了读取逻辑处理器的统计数据。其重置的统计数据主要是 2 个：总计读取消息数和总计读取字节数。在这里，RecvByteBufAllocator 这个接口的实现是 AdaptiveRecvByteBufAllocator。

接着来看代码③。对于读取而言，申请的 ByteBuf 的大小对于性能表现很重要。申请的 ByteBuf 太小，要完整读取 Socket 缓冲区中的数据就需要多次的读取操作，会降低系统运行效能；申请的 ByteBuf 太大，读取数据是方便了，但是对于同时服务大量客户端的服务端应用而言，每一个客户端申请的 ByteBuf 越大，总和之后，对于整体系统的内存和 GC 压力都会增大。因此 Netty 中制定了不同的策略来实现对 ByteBuf 的大小申请机制。这个会在后文开单张进行具体的说明，这边大家先了解这个情况。

接着我们来看代码④。通过方法 doReadBytes 进行实际的字节读取，其方法如下：

```java
protected int doReadBytes(ByteBuf byteBuf) throws Exception
{
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.attemptedBytesRead(byteBuf.writableBytes());
    return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());
}
```

首先通过方法 allocHandle.attemptedBytesRead 记录本次尝试读取的字节数，也就是 byteBuf 可以容纳的写入大小。而后通过方法 `ByteBuf#writeBytes(ScatteringByteChannel, int)` 完成字节的写入，该方法是 ByteBuf 写入方法之一，方法实现会尝试从入参的 Channel 最大读取第二入参长度的字节内容，并且在容量不足时自动扩容。字节读取完毕后便向后触发管道 channelRead 事件。

接着来看代码⑤，方法 `RecvByteBufAllocator.Handle#continueReading` 决定了是否继续从通道中读取数据，这里的实现是：

```
DefaultMaxMessagesRecvByteBufAllocator.MaxMessageHandle#continueReading()
```

其实现如下：

```java
  public boolean continueReading()
  {
      return continueReading(defaultMaybeMoreSupplier);
  }
  public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier)
  {
      return config.isAutoRead() && (!respectMaybeMoreData || maybeMoreDataSupplier.get()) && totalMessages < maxMessagePerRead && totalBytesRead > 0;
  }
```

这个方法在上个章节已经解释过，不过当时 maybeMoreDataSupplier 这个实现没有实际的意义，但是在 NioSocketChannel 读取数据时就需要具体判断了，其代码如下：

```java
private final UncheckedBooleanSupplier defaultMaybeMoreSupplier = new UncheckedBooleanSupplier()
{
    @Override
    public boolean get()
    {
        return attemptedBytesRead == lastBytesRead;
    }
}
```

显然，如果预期读取字节数 attemptedBytesRead 与最后一次切实读取的字节数 lastBytesRead 相等，则较大可能性还需要继续读取。

如果无需继续读取，则离开循环，触发管道 ChannelReadComplete 事件，并且结束本次的可读就绪 Key 的全部处理流程。

最后，我们来看下代码⑥。前文提到过 readPending 属性的作用，如果通道的读取采用手动的模式，也就是需要由业务代码来触发，则每次处理完毕可读就绪 Key，就需要移除这个可读就绪的事件关注。直到下一次再被手动触发。这里 readPending 就用于判断当前是否要移除可读就绪的事件关注。因为有可能上面代码在触发管道的 channelRead 或者 channelReadComplete 事件时业务代码又注册了 read 操作，那么这里就不需要移除可读就绪事件的关注了。

### 连接远端

Netty 也提供了客户端引导程序，方便业务快速构建起稳定健壮的客户端应用。在客户端能够完成业务数据的收发之前，首先需要完成的就是和远端应用建立连接，也就是常说的 connect。而这个过程是伴随着 NioEventLoop 对连接就绪事件的处理。不过这里我们先从引导程序开始看起。

BootStrap 是用于客户端的引导程序，这个在之前的例子中我们也曾经介绍过，其提供了 connect 方法及其不同入参的变种用于完成和远端地址的连接。我们来具体看下其代码，如下。

```java
public ChannelFuture connect(SocketAddress remoteAddress)
{
    ObjectUtil.checkNotNull(remoteAddress, "remoteAddress");
    validate();
    return doResolveAndConnect(remoteAddress, config.localAddress());
}
private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress)
{
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if(regFuture.isDone())
    {
        if(!regFuture.isSuccess())
        {
            return regFuture;
        }
        return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
    }
    else
    {
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        //省略代码：在promise上添加监听器，在监听器的完成方法中调用doResolveAndConnect0方法。
        return promise;
    }
}
```

doResolveAndConnect 方法的套路也是 Netty 中反复使用的，在前文已经介绍过，这里不再赘述。我们直接看其符合连接的核心方法 doResolveAndConnect0，其代码如下：

```java
private ChannelFuture doResolveAndConnect0(final Channel channel, SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise)
{
    try
    {
        final EventLoop eventLoop = channel.eventLoop();
        //代码①
        final AddressResolver < SocketAddress > resolver = this.resolver.getResolver(eventLoop);
        if(!resolver.isSupported(remoteAddress) || resolver.isResolved(remoteAddress))
        {
            //代码②
            doConnect(remoteAddress, localAddress, promise);
            return promise;
        }
        final Future < SocketAddress > resolveFuture = resolver.resolve(remoteAddress);
        if(resolveFuture.isDone())
        {
             //省略代码，解析无异常情况下，执行 doConnect 方法
            return promise;
        }
        resolveFuture.addListener(new FutureListener < SocketAddress > ()
        {
            @Override
            public void operationComplete(Future < SocketAddress > future) throws Exception
            {

                //省略代码，以 if 一样的处理逻辑
            }
        });
    }
    catch(Throwable cause)
    {
        promise.tryFailure(cause);
    }
    return promise;
}
```

代码虽长，但是简化之后我们可以看到只有两个核心内容：

- 对远端地址进行解析。
- 如果不能解析则执行连接操作；如果可以解析，则在解析成功后执行连接操作。

#### 地址解析

Netty 这边对地址解析做了一层封装，在连接远端地址之前，首先需要将主机名解析为 IP 地址。而这个功能就交个接口 AddressResolver 完成。考虑到 Netty 整体异步化的设计，AddressResolver 需要和一个 EventLoop 搭配使用，而实际上其绑定的 EventLoop 实例都是当前 Channel 绑定的 EventLoop 实例。

获取 AddressResolver 实例的工作则交给 AddressResolverGroup 接口的方法 getResolver 提供，AddressResolverGroup 是一个抽象类，其使用模板方法已经定义了 getResolver 方法。该方法内部通过 synchronized 与 IdentityHashMap 存储 EventLoop 和 AddressResolver 映射的方式，确保了每一个 EventLoop 只会持有一个 AddressResolver 实例，后续的获取都是获取首次生成的对象，而不会重复创建。

我们来看下 AddressResolver 的类图，如下：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191215232103.png)

从类图可以看到，主要的模板功能由 AbstractAddressResolver 抽象父类完成，类定义为：

```
AbstractAddressResolver<T extends SocketAddress> implements AddressResolver<T>
```

该类主要完成的功能有：

- 父类持有属性 executor，存储着与该地址解析器绑定的 EventLoop 实例。
- 父类持有属性 matcher，存储着一个 TypeParameterMatcher 实例，该实例是用于确认需要解析的地址对象类是否与类定义中的泛型匹配。这个是一个独立的工具类。
- 提供了接口 AddressResolver 所有方法的抽象实现，每一个实现的内部逻辑都用模板方法的思路实现，将参数检查，Promise 对象生成，入参类型是否匹配等工作都在模板方法中完成了。只剩余真正的解析动作留给子类实现。

第一点和第三点都很简单，就不在代码层面展开分析了。第二点，TypeParameterMatcher 的实现，是很有意思的。

TypeParameterMatcher 是应用在父类使用了泛型定义，获取子类具体的泛型定义的场合。比如一个类定义为 `Abstract class Fo<T>`，其子类定义为 `class Son extendsFo<String>`，通过 TypeParameterMatcher 就能获取到类 Son 对泛型的定义为 String，而后就可以通过方法 match(Object) 来判断传入的参数是否符合吻合泛型定义。

这个类的应用场景一般是在一个泛型的抽象父类有多种子类且不同泛型定义的情况下，子类自身需要判断入参是否是子类的泛型定义类的对象来使用的。如果在各个子类通过手动代码实现，则每一个子类都需要实现冗余的这部分代码。但是通过 TypeParameterMatcher 就可以将这个工作自动化，并且放入在父类实现，成为模板方法的一部分。

对于 TypeParameterMatcher 而言，其最重要的实现就是如何获取子类的泛型定义，除开一些缓存的考量，这个功能的方法主要是 `TypeParameterMatcher#find0(Object $1,Class $2,String $3)`，其代码比较长，这里不贴出，但是思路实际上很简单，介绍下三个入参：`$1` 是当前子类的对象，`$2` 是父类的类对象，`$3` 是泛型在父类类对象中的名称。通过 `$3` 确定 `$2` 中要寻找的泛型位置（因为可能存在多个泛型参数），之后通过 `$1.getClass` 获取子类的类对象，通过子类对象的父类换个 `$2` 对比，寻找到正确层次的父类，并且通过前面确定的泛型位置获取泛型参数信息，并且根据不同类型的泛型定义，最终得到在子类中，该泛型的明确定义对象类。

重新回到 AbstractAddressResolver 上，上面已经分析过了这个类的作用，而 NoopAddressResolver 是一个空实现的子类，不执行任何操作，且对任何判断都返回 true。该类实际上将解析的工作放弃，其应用在地址解析是由管道中的 handler 完成的场合。这个后文来分析其场景，现在这里来聊下主要的子类实现 InetSocketAddressResolver 提供的功能。该子类本身并不提供任何能力实现，AddressResolver 接口定义地址解析功能实际上是委托给了内部的 NameResolver 接口实例去实现的。该接口的实现子类有：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191216190348.png)

这些子类的作用分别是：

- DefaultNameResolver：默认的名称解析实现，使用 JDK 内置的名称解析能力。该实现在解析名称时会阻塞，因为其使用的 JDK 内置实现是阻塞的。
- DnsNameResolver：Netty 提供的 DNS 域名解析实现，其内部也是使用 BootStrap 构建的客户端去连接 DNS 服务器查询地址解析信息。
- RoundRobinInetAddressResolver：本身不实现解析能力，依靠入参的 NameResolver 进行解析，但是每次都解析出名称对应的全部地址，随机选择一个进行返回。
- CompositeNameResolver：本身不实现解析能力，依靠入参的 NameResolver 数组进行解析，只要任意一个解析成功就返回解析结果，如果全部失败，则返回最后一个的错误。

#### **连接目标地址**

成功完成地址的名称解析后，得到了远端名称的 IP 地址（或者本来就是 IP 地址则可以省略解析的步骤），就开始连接远端。前文提到有展示，该功能依靠方法 `io.netty.bootstrap.Bootstrap#doConnect` 实现连接，以下是其代码：

```java
private static void doConnect(final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise connectPromise)
{
    final Channel channel = connectPromise.channel();
    channel.eventLoop().execute(new Runnable()
    {
        @Override
        public void run()
        {
            if(localAddress == null)
            {
                channel.connect(remoteAddress, connectPromise);
            }
            else
            {
                channel.connect(remoteAddress, localAddress, connectPromise);
            }
            connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
        }
    });
}
```

该方法就是在 EventLoop 线程中投递了一个任务来执行方法 `AbstractChannel#connect(SocketAddress, ChannelPromise)`，这个方法是触发了管道的 connect 事件，该事件默认的处理方式管道的首节点，其处理代码如下：

```java
public void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise)
{
    unsafe.connect(remoteAddress, localAddress, promise);
}
```

这里 Unsafe.connect 的实际实现如下：

```java
public final void connect(final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise)
{
    if(!promise.setUncancellable() || !ensureOpen(promise))
    {
        return;
    }
    try
    {
        if(connectPromise != null)
        {
            throw new ConnectionPendingException();
        }
        boolean wasActive = isActive();
        if(doConnect(remoteAddress, localAddress))
        {
            fulfillConnectPromise(promise, wasActive);
        }
        else
        {
            //省略代码：在 EventLoop 注册定时任务，截止时间为连接超时时间，任务内容为将 promise 设置为失败，并且异常原因设置为超时。在 connectPromise 设置监听器，监听器在连接尝试取消时触发，设置 closeFuture。
        }
    }
    catch(Throwable t)
    {
        promise.tryFailure(annotateConnectException(t, remoteAddress));
        closeIfClosed();
    }
}
```

连接的主要方法就是在 doConnect 上，其实现为：

```java
protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception
{
    if(localAddress != null)
    {
        doBind0(localAddress);
    }
    boolean success = false;
    try
    {
        boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
        if(!connected)
        {
            selectionKey().interestOps(SelectionKey.OP_CONNECT);
        }
        success = true;
        return connected;
    }
    finally
    {
        if(!success)
        {
            doClose();
        }
    }
}
```

SocketUtils.connect 方法内部就是直接调用 JDK 的 NIO 原生接口进行连接动作，由于 SocketChannel 是被设置在非阻塞模式下，该调用一般会返回 false。而返回 false 的情况下，则将连接就绪关注注册到 Selector 中，等待该事件就绪后被触发。

这里我们来看下 NioEventLoop 对连接就绪事件的处理方法，如下：

```java
if((readyOps & SelectionKey.OP_CONNECT) != 0)
{
    int ops = k.interestOps();
    ops &= ~SelectionKey.OP_CONNECT;
    k.interestOps(ops);
    unsafe.finishConnect();
}
```

首先是要取消对连接就绪事件的关注，否则 Selector 就会一直返回该事件就绪的提醒。然后是通过 finishConnect 方法完成连接的后续步骤，其内容如下：

```java
public final void finishConnect()
{
     assert eventLoop().inEventLoop();
     try
     {
         boolean wasActive = isActive();
         doFinishConnect();
         fulfillConnectPromise(connectPromise, wasActive);
     }
     catch(Throwable t)
     {
         fulfillConnectPromise(connectPromise, annotateConnectException(t, requestedRemoteAddress));
     }
     finally
     {
         if(connectTimeoutFuture != null)
         {
             connectTimeoutFuture.cancel(false);
         }
         connectPromise = null;
     }
}
```

方法 doFinishConnect() 的实现是调用 JDK NIO 的原生接口将 SocketChannel 的连接过程完成，而后通过 fulfillConnectPromise 方法设置上文代码中出现过的 connectPromise 的结果为成功，也就是通知在其上的监听器，连接成功完成。

客户端通道连接远端服务的过程，委托了很多个方法，但是总结而言其实不复杂：通过 JDK NIO 原生 connect 接口发起连接请求，并且注册 Selector 连接就绪事件的关注；等待连接就绪事件触发后，通过原生的 finishConnect 接口完成通道连接，并且设置异步 promise 为成功，用于通知等待连接成功的监听器。

### 总结与思考

本篇文章详细分析了 Netty 如何进行数据读取和远端连接。目前我们已经分析完毕客户端接受、连接远端，数据读取，可以看到 Netty 在这其中，设计了很多原生 NIO 并不具备的辅助性功能来提高代码的健壮性。不过部分细节和整个数据处理无关，看代码的时候需要有所取舍，应该重点关注主流程的内容。

下篇文章，我们将介绍最后一个流程环节：数据的写出。