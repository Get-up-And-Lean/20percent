# 18/31Netty 流程细讲之 ServerBootStrap 启动

## 引言

在 Netty 的引导程序中，启动一个服务端应用是一个十分简单的事情。配置链接类对象，配置子类初始化 `ChannelHandler`，再调用 `bind` 方法绑定端口号，一个服务端应用就启动完毕了，接着只需要等待客户端发送连接请求，程序就能自动为我们完成客户端接入。看着是很简单的过程，Netty 在背后却是做了相当多的工作，本文就以 `ServerBootStrap` 启动的时序动作为分析入手点，剖析在引导程序启动中，涉及到的具体代码内容。

## 总体时序流程

使用 `ServerBootStrap` 的 `bind` 方法执行对端口的监听，这里面涉及到好几个步骤，简单而言包括有：

- 按照给定的 `Channel` 类，实例化一个对象。
- 将实例化的 `Channel` 对象注册到 `EventLoop` 上。
- 设置 `Channel` 的绑定监听地址，并且更新 `Channel` 对象的 `SelectionKey` 的事件关注集。

整体的时序如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20190828140939.jpg)

可以看到，一个完整的初始化动作内容很多，并且很多都是委托给其他类完成，下面按照三个大步骤进行区分，来逐一分析。

## 初始化 Channel 对象

`ServerBootStrap` 使用 `bind` 方法绑定端口，其内部首先是执行 `Channel` 初始化方法，首先来看下方法的跳转流程，如下

```java
public ChannelFuture bind(int inetPort) {return bind(new InetSocketAddress(inetPort));
    }
public ChannelFuture bind(SocketAddress localAddress) {validate();
        return doBind(ObjectUtil.checkNotNull(localAddress, "localAddress"));
    }
private ChannelFuture doBind(final SocketAddress localAddress) {final ChannelFuture regFuture = initAndRegister();
        //... 省略其他代码
    }
```

最终代码执行到 `initAndRegister` 方法，该方法内部完成了 2 个职责：1）初始化 `Channel` 对象；2）将 `Channel` 对象注册到 `EventLoop` 上。`initAndRegister` 方法如下

```java
final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {//。。。省略代码，异常处理逻辑：channel 不为 null 则关闭；返回一个代表失败的 future 对象}
        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {if (channel.isRegistered()) {channel.close();
            } else {channel.unsafe().closeForcibly();}
        }
        return regFuture;
    }
```

初始化工作由方法 `AbstractBootstrap#init` 方法完成，该方法是一个抽象方法，在 `ServerBootStrap` 中的实现如下

```java
void init(Channel channel) throws Exception {//。。。省略相关代码，主要是设置属性和配置项到 Channel 中，分别通过 channel.attr(key).attr(attr) 和 channel.config().setOption 完成
        ChannelPipeline p = channel.pipeline();
        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        //。。。省略相关代码，主要是获取设置的客户端通道的配置对象和属性对象，局部变量名分为是 currentChildOptions 和 currentChildAttrs
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) throws Exception {final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {pipeline.addLast(handler);
                }
                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });}
        });}
```

唯一重要的部分就是向 `Channel` 的 `pipeline` 中添加一个匿名的 `ChannelHandler`，即 `ChannelInitializer` 的匿名子类。这边先解释下 `ChannelInitializer` 这个类的原理。

### ChannelInitializer

`ChannelInitializer` 是一个继承了 `ChannelInboundHandlerAdapter` 的抽象类。其主要重写了 2 个方法：`channelRegistered` 和 `handlerAdded`。两个方法的重写思路基本一致，且通过互斥手段避免其方法内容的重复执行；在大多数情况下，只有 `handlerAdded` 方法中的内容可以成功执行，因此下面我们以重写的 `handlerAdded` 进行分析，其重写内容如下

```java
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        //Netty 中的默认实现保证了当 handlerAdded 方法被调用时，通道已经注册到了 EventLoop 上。
        if (ctx.channel().isRegistered()) {if (initChannel(ctx)) {removeState(ctx);
            }
        }
    }
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {if (initMap.add(ctx)) { // Guard against re-entrance.
            try {initChannel((C) ctx.channel());} catch (Throwable cause) {exceptionCaught(ctx, cause);
            } finally {ChannelPipeline pipeline = ctx.pipeline();
                if (pipeline.context(this) != null) {pipeline.remove(this);
                }
            }
            return true;
        }
        return false;
    }
```

`handlerAdded` 主要的作用就是触发 `ChannelInitializer#initChannel(ChannelHandlerContext)` 方法，首先是通过一个防御性的编程，避免初始化动作并发执行，也就是 `initMap.add(ctx)` 这个方法调用的作用，`initMap` 是一个并发安全的 `Set`，集合中的元素为具体的 `ChannelHandlerContext` 对象。

接着调用抽象方法 `ChannelInitializer#initChannel(C)`。该抽象方法由子类重写，在入门篇的示例中我们介绍过，一般这个重写的方法就是用来向 `pipeline` 中添加合适的处理器用于后续的业务处理。

当抽象的 `initChannel` 方法之后完毕后，将自身从 `pipeline` 中移除。由此可以看出，这个 `ChannelInitializer` 是一个一次性消耗品，当通道被注册到 `EventLoop` 上后，通过其子类重写的 `initChannel` 抽象方法完成向 `pipeline` 添加合适的处理器后，就从 `pipeline` 中删除，不再参与后续的流程，也很符合其“初始化器”的身份定位。

最终，`ChannelInitializer#initChannel(ChannelHandlerContext)` 成功完成对应通道的初始化工作后，从 `initMap` 删除自身的 `ChannelHandlerContext` 对象，防止集合无限变大。因为 `initMap` 的作用是防止并发执行，因此执行成功后，可以安全删除其中的元素。

最后，还有一点需要解释的是，`ChannelInitializer` 是一个共享的处理器，从其实现也能看出，其不持有不安全的实例属性，因此可以安全的并发应用于多个通道。在实际当中，我们常常通过 `ServerBootStrap#handler(ChannelHandler)` 传入一个 `ChannelInitializer` 对象，该对象就是被并发的应用在每一个接入进来的客户端 `Channel` 上的。

### 通过 ChannelInitializer 向 pipeline 添加 ServerBootstrapAcceptor

书接上文，在初始化 `Channel` 的环节，我们向通道添加了一个 `ChannelInitializer` 的实现子类，其目的在于当 `Channel` 注册到 `EventLoop` 上时，向 `pipeline` 添加 `ServerBootstrapAcceptor`，而这才是真正用于处理客户端接入请求业务逻辑的处理器。`ServerBootstrapAcceptor` 也是一个入站处理器，但是其具体的作用，我们这边先按下不表，待到后文用到时再细说。添加完成之后，`Channel` 的初始化工作即告完成。

## 注册 Channel 对象到 EventLoop 上

从 `initAndRegister` 方法可知，当初始化完成后，接下来就是将 `Channel` 对象注册到 `EventLoopGroup` 上。该职责由 `NioEventLoopGroup` 继承的父类方法 `MultithreadEventLoopGroup#register(Channel)` 完成。该方法内部通过 `next` 调用从 `NioEventLoop` 数组中选择一个实例，执行对应的 `register` 方法，该方法最终委托到了 `SingleThreadEventLoop#register(Channel)` 这个具体的实现上。其实现如下

```java
public ChannelFuture register(Channel channel)
{return(register( new DefaultChannelPromise( channel, this) ));}
public ChannelFuture register(final ChannelPromise promise)
{ObjectUtil.checkNotNull( promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return(promise);
}
```

Netty 将一些它认为比较高阶的 API 调用都封装在自定义的 `Unsafe` 类中，以提醒开发者不要直接使用这些方法；这些方法都是 Netty 内部自行使用的。`Unsafe` 的实例由抽象方法 `AbstractChannel#newUnsafe` 提供，这里的 `Channel` 类型是 `NioServerSocketChannel`，对这个抽象方法的实现是来自其父类方法 `AbstractNioMessageChannel#newUnsafe`, 得到的实例对象是 `AbstractNioMessageChannel.NioMessageUnsafe`，其类图如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191209144245.png)

来关注其 `register` 方法，该方法由父类提供，其定位是 `AbstractUnsafe#register`，代码如下

```java
public final void register(EventLoop eventLoop, final ChannelPromise promise)
{
     //。。。省略代码，参数有效性检查
     AbstractChannel.this.eventLoop = eventLoop;
     if(eventLoop.inEventLoop())
     {register0(promise);
     }
     else
     {
         try
         {eventLoop.execute(new Runnable()
             {
                 @Override
                 public void run()
                 {register0(promise);
                 }
             });}
         catch(Throwable t)
         {//。。。省略代码，异常情况下，关闭通道以及设置 promise 中的结果。}
     }
}
```

有关 `Channel` 的操作，基本上都需要在 `EventLoop` 上执行，因此这里 `register` 的实现也是标准的 `EventLoop` 模式：当前线程是 `EventLoop` 线程则直接执行，否则包装为 `Runnable` 对象投递到 `EventLoop` 线程中等待调度执行。来看下实际的注册方法 `register0`，如下

```java
private void register0(ChannelPromise promise)
{
     try
     {
         //。。。省略代码，检查通道是否未关闭
         boolean firstRegistration = neverRegistered;
         doRegister();
         neverRegistered = false;
         registered = true;
         pipeline.invokeHandlerAddedIfNeeded();
         safeSetSuccess(promise);
         pipeline.fireChannelRegistered();
         if(isActive())
         {if(firstRegistration)
             {pipeline.fireChannelActive();
             }
             else if(config().isAutoRead())
             {beginRead();
             }
         }
     }
     catch(Throwable t)
     {//。。。省略代码，关闭通道以及和设置 promise 对象}
}
```

概括而言，`register0` 完成了四件事：

- 执行注册方法 `doRegister`。
- 在注册之前添加到 `Channel` 的 `handler` ，触发其 `handlerAdded` 事件。
- 设置 `promise` 结果为成功，触发在 `pipeline` 中的 `handler` 的 `channelRegister` 事件。
- 通道处于激活（Socket 已经打开且通道不曾关闭），并且首次注册到 `EventLoop` 时，触发 `pipeline` 中的 `channelActive` 事件。

主要来看下 `doRegister` 方法，在 `NioServerSocketChannel` 中，该方法使用其父类方法 `AbstractNioChannel#doRegister`, 代码如下

```java
protected void doRegister() throws Exception
{
    boolean selected = false;
    for(;;)
    {
        try
        {selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        }
        catch(CancelledKeyException e)
        {//。。。省略代码，异常情况下通过 java.nio.channels.Selector#selectNow() 避免无效 key 被 Selector 缓存
        }
    }
}
```

`doRegister` 内部的实现很简单，获取 Java NIO 中的 `SelectableChannel` 对象，也就是 `NioServerSocketChannel` 包装下，底层真正的通道对象，将其注册到一个选择器上，选择器由方法 `NioEventLoop#unwrappedSelector` 提供，该选择器是该 `EventLoop` 独占持有。将通道注册到选择器时，并未注册关注事件，也就是第二个参数 `0` 的含义。也即是说，在这里，只是单纯的将 `Channel` 注册到 `EventLoop`，该通道本身目前不会对任何事件有反应，需要等待后续的更新，在这里只是一个单纯的注册动作。

## 执行地址绑定

在绑定方法 `ServerBootStrap#doBind` 的内容中，我们首先介绍了负责初始化和注册的 `initAndRegister` 方法，接下来，我们看下 `doBind` 方法当中还剩余什么，完整的方法内容如下

```java
private ChannelFuture doBind(final SocketAddress localAddress)
{final ChannelFuture regFuture = initAndRegister();
     final Channel channel = regFuture.channel();
     if(regFuture.cause() != null)
     {return regFuture;}
     if(regFuture.isDone())
     {ChannelPromise promise = channel.newPromise();
         doBind0(regFuture, channel, localAddress, promise);
         return promise;
     }
     else
     {final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
         regFuture.addListener(new ChannelFutureListener()
         {
             @Override
             public void operationComplete(ChannelFuture future) throws Exception
             {Throwable cause = future.cause();
                 if(cause != null)
                 {promise.setFailure(cause);
                 }
                 else
                 {promise.registered();
                     doBind0(regFuture, channel, localAddress, promise);
                 }
             }
         });
         return promise;
     }
}
```

由于上文的注册动作实际上是包装了一个 `Runnable` 对象投递到 `EventLoop` 的队列中完成的，所以在 `initAndRegister` 方法返回后，代表注册任务的 `regFuture` 可能完成也可能未完成。因此下文需要根据不同的情况具体区分，但是本质上都是需要等待注册完成后，才能执行绑定动作。所以在这里，我们直接分析实际绑定动作的方法 `doBind0`，其代码如下

```java
  private static void doBind0(final ChannelFuture regFuture, final Channel channel, final SocketAddress localAddress, final ChannelPromise promise)
  {channel.eventLoop().execute(new Runnable()
      {
          @Override
          public void run()
          {if(regFuture.isSuccess())
              {channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
              }
              else
              {promise.setFailure(regFuture.cause());
              }
          }
      });}
```

`doBind0` 也是创建了一个任务投递到 `EventLoop` 中执行，具体的绑定，是委托给方法 `AbstractChannel#bind(java.net.SocketAddress, ChannelPromise)` 执行，而这个方法继续委托给方法 `DefaultChannelPipeline#bind(java.net.SocketAddress, ChannelPromise)`。

前文曾经介绍过，`bind` 是一个出站事件，是从 `pipeline` 的尾部开始向前传递。一般而言，业务代码很少处理除了 `write` 之外的事件，那么 `bind` 这个事件最终会传递到头结点，最终处理这个事件的类是 `io.netty.channel.DefaultChannelPipeline.HeadContext`。其 `bind` 方法的实现如下

```java
public void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)
{unsafe.bind(localAddress, promise);
}
```

一如既往，仍然是将操作委托给了 `Unsafe` 类。该 `bind` 接口实际由 `AbstractChannel.AbstractUnsafe#bind`，代码如下

```java
public final void bind(final SocketAddress localAddress, final ChannelPromise promise)
{
    //。。。省略代码，参数校验
    boolean wasActive = isActive();
    try
    {doBind(localAddress);
    }
    catch(Throwable t)
    {safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }
    if(!wasActive && isActive())
    {invokeLater(new Runnable()
        {
            @Override
            public void run()
            {pipeline.fireChannelActive();
            }
        });}
    safeSetSuccess(promise);
}
```

最核心的就是这个 `doBind`，该方法由 `Channel` 提供，这里具体的实现是 `NioServerSocketChannel#doBind`，其代码如下

```java
protected void doBind(SocketAddress localAddress) throws Exception
{if(PlatformDependent.javaVersion() >= 7)
    {javaChannel().bind(localAddress, config.getBacklog());
    }
    else
    {javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```

最后仍然是委托了”底层“，也就是 Java NIO 的绑定 API。

## 绑定端口成功后，触发 ChannelActive 事件

绑定成功后，让我们回到 `Unsafe` 的 `bind` 方法中，接下里的步骤就是因为通道从未激活状态更新到激活状态，从而触发了 `channelActive` 事件, 并且设置地址绑定的 `promise` 结果为真。这里，我们需要看下 `channelActive` 事件的传播，这个事件是一个入站事件，首先是被 `HeadContext` 也就是首节点捕获，其实现代码如下

```java
public void channelActive(ChannelHandlerContext ctx)
{ctx.fireChannelActive();
     readIfIsAutoRead();}
private void readIfIsAutoRead()
{if(channel.config().isAutoRead())
     {channel.read();
     }
}
```

首节点首先是向后传递了这个事件，然后在配置了自动读取的情况下，执行了读取。`read` 方法调用是委托了 `pipeline` 的 `read` 事件，这个事件是一个出站事件，最终这个事件仍然是首节点实现了处理方法，其代码如下

```java
public void read(ChannelHandlerContext ctx)
{unsafe.beginRead();
}
```

依然是传统做法委托给 `Unsafe`，而 `Unsafe` 的 `beginRead` 方法最终是委托到 `AbstractChannel` 的 `doBeginRead` 抽象方法，最终的实现方法是 `AbstractNioChannel#doBeginRead`, 其代码如下

```java
protected void doBeginRead() throws Exception
{// Channel.read() or ChannelHandlerContext.read() was called
     final SelectionKey selectionKey = this.selectionKey;
     if(!selectionKey.isValid())
     {return;}
     readPending = true;
     final int interestOps = selectionKey.interestOps();
     if((interestOps & readInterestOp) == 0)
     {selectionKey.interestOps(interestOps | readInterestOp);
     }
}
```

这个方法的核心就是将之前 `Channel` 注册到 `EventLoop` 上的 `Selector` 时产生的 `SelectionKey` 的关注事件集合进行更新，增加对 `readInterestOp` 事件的关注。`readInterestOp` 的值是在类初始化的构造器传入的，而 `NioServerSocketChannel` 的传入的值为 `OP_ACCEPT`，也就是其感兴趣的事件为客户端接入就绪事件。注意，这个方法是在 `EventLoop` 的线程中执行的，此时变更了 `SelectionKey` 的关注事件，在下一次循环时就可以生效了。

到这里为止，服务端应用在经历了初始化 `Channel` 对象；注册其到 `EventLoop` 上；将 `Channel` 绑定到对应的端口；触发 `ChannelActive` 事件进而增加 `Channel` 对客户端接入就绪事件的关注后，整个启动流程结束，可以开始对外提供服务了。

## 额外点

在服务端启动过程中，在 `Channel` 注册到 `EventLoop` 环节，方法 `AbstractUnsafe#register0` 有部分代码如下

```java
boolean firstRegistration = neverRegistered;
doRegister();
neverRegistered = false;
registered = true;
pipeline.invokeHandlerAddedIfNeeded();
safeSetSuccess(promise);
pipeline.fireChannelRegistered();
if(isActive())
{if(firstRegistration)
    {pipeline.fireChannelActive();
    }
    else if(config().isAutoRead())
    {beginRead();
    }
}
```

而在绑定阶段，也就是 `NioMessageUnsafe.bind` 方法中，也有类似的一段，如下

```java
public final void bind(final SocketAddress localAddress, final ChannelPromise promise)
{
    //。。。省略代码
    if(!wasActive && isActive())
    {invokeLater(new Runnable()
        {
            @Override
            public void run()
            {pipeline.fireChannelActive();
            }
        });}
    safeSetSuccess(promise);
}
```

那么这里两次尝试调用 `pipeline.fireChannelActive()` 是否会造成 `channelActive` 呢？答案是不会的。

对于 `NioServerSocketChannel` 而言，在注册 `EventLoop` 阶段，因为没有绑定地址，所以 isActive 是返回 false 的，也就不会启动 pipeline.fireChannelActive。

由于 Netty 中的 `Channel` 不是一次性消耗对象，可以反复注册，第一个方法是用于给 Channel 取消注册后再次注册的场景下使用。

## 总结与思考

本文详细的分析了 `ServerBootStrap` 的启动流程，以及内部细节的方方面面。可以看到，Netty 当中对代码的分层和封装是比较厚的，完成一个功能，常常需要委托比较多的层次。但是大体上也有规律可循，`pipeline` 中的头结点和尾结点负责实现对事件的默认处理，`Unsafe` 用于实现一些和通道相关但不适合暴露给开发者的操作，而具体操作如果和通道相关，则方法会委托到具体的 `Channel` 实现类。

本文讲述了服务端启动的流程，在启动完毕后，服务端就开始监听端口上的接入请求，下一篇文章我们就来剖析下客户端链接的接入处理流程。