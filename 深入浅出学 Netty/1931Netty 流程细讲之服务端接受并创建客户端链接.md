# 19/31Netty 流程细讲之服务端接受并创建客户端链接

### 引言

在上篇文章中我们分析了服务端的启动流程。在服务端启动成功后，就可以开始监听端口，并且处理客户端的接入请求了。

服务端的 ServerChannel 是注册在 EventLoop 线程上的，而对客户端的接入处理也是从这个地方开始。因此，在这里我们需要首先分析下在 NioEventLoop。

### NioEventLoop 的 run 方法

源码篇第一讲，我们曾分析过 NioEventLoop，它是一个特殊化的 SingThreadEventExecutor，因为其 run 方法的实现并不是单纯的从队列中取出任务执行，还包含了在 Selector 对象上执行等待的流程。对于在端口上监听客户端接入请求的服务端程序而言，显然其监听等待也是依靠了在 Selector 上的等待。那下面我们就看下其 run 方法的实现，具体如下：

```java
protected void run()
{
    for(;;)
    {
        try
        {
            try
            {
                switch(selectStrategy.calculateStrategy(selectNowSupplier, hasTasks()))
                {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO
                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));
                        if(wakenUp.get())
                        {
                            selector.wakeup();
                        }
                    default:
                }
            }
            catch(IOException e)
            {
                rebuildSelector0();
                handleLoopException(e);
                continue;
            }
            {
                //代码①：处理就绪Key和队列任务
            }
        }
        catch(Throwable t)
        {
            handleLoopException(t);
        }
        {
            //代码②：处理EventLoop关闭情况
        }
    }
}
```

#### **等待策略选择**

首先来关注下其选择等待部分的代码。首先是通过代码

```
selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())
```

来确定接下来的选择等待策略。SelectStrategy 是一个接口，其唯一实现就是 DefaultSelectStrategy。而方法 calculateStrategy 的内容体也很简单，如下：

```java
public int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception
{
    return hasTasks ? selectSupplier.get() : SelectStrategy.SELECT;
}
```

如果队列中没有任务要执行，则后续需要在 Selector 上执行等待，也就是使用 SELECT 策略。否则的话，就是执行 selectSupplier.get 方法来获取后续策略，其方法体如下：

```java
int selectNow() throws IOException
{
    try
    {
        return selector.selectNow();
    }
    finally
    {
        if(wakenUp.get())
        {
            selector.wakeup();
        }
    }
}
```

SelectStrategy 接口定义的策略属性均为负数值，非负数的返回就可以让后续的处理进入代码②的部分。结合代码，这个选择策略可以简单归纳为：在没有队列任务的情况下，执行阻塞等待就绪事件；在有队列任务的情况下，执行无阻塞的就绪选择，并且返回就绪事件数，以便后续代码进入任务和就绪事件处理环节。

#### **阻塞等待**

如果等待策略选择的结果是阻塞等待，则进入代码 `select(wakenUp.getAndSet(false))` 中。在进入方法体之前，我们先解释下属性 wakenUp。

这个属性是一个 AtomicBoolean 对象，让各个线程通过CAS操作来争夺唤醒 EventLoop 持有的 Selector 的权利，因为 Selector.wakeUp 是一个昂贵的操作。由于 NioEventLoop 在外部添加任务时其 EventLoop 线程可能仍然阻塞在 Selector.select 操作上，所以需要通过 Selector.wakeUp 进行唤醒，而为了避免频繁的执行这个操作，所以通过 wakenUp 属性进行CAS保护。而该属性在 EventLoop 准备执行阻塞 select 操作之前，都会被设置为 false 的基本状态。

说明完 wakenUp 属性，我们再来看看 select 方法的具体内容，如下：

```java
private void select(boolean oldWakenUp) throws IOException
{
    Selector selector = this.selector;
    try
    {
        int selectCnt = 0;
        long currentTimeNanos = System.nanoTime();
        //代码①
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
        for(;;)
        {
            //代码②
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000 L) / 1000000 L;
            if(timeoutMillis <= 0)
            {
                if(selectCnt == 0)
                {
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }
            //代码③
            if(hasTasks() && wakenUp.compareAndSet(false, true))
            {
                selector.selectNow();
                selectCnt = 1;
                break;
            }
            int selectedKeys = selector.select(timeoutMillis);
            selectCnt++;
            //代码④
            if(selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks())
            {
                break;
            }
            //代码⑤
            if(Thread.interrupted())
            {
                if(logger.isDebugEnabled())
                {
                    logger.debug("Selector.select() returned prematurely because " + "Thread.currentThread().interrupt() was called. Use " + "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                }
                selectCnt = 1;
                break;
            }
            long time = System.nanoTime();
            //代码⑥
            if(time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos)
            {
                selectCnt = 1;
            }
            //代码⑦
            else if(SELECTOR_AUTO_REBUILD_THRESHOLD > 0 && selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD)
            {
                selector = selectRebuildSelector(selectCnt);
                selectCnt = 1;
                break;
            }
            currentTimeNanos = time;
        }
        if(selectCnt > MIN_PREMATURE_SELECTOR_RETURNS)
        {
            if(logger.isDebugEnabled())
            {
                logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.", selectCnt - 1, selector);
            }
        }
    }
    catch(CancelledKeyException e)
    {
        //。。。省略代码，日志记录异常
    }
}
```

整体代码比较长，我们分段来分析，首先来看代码①。通过 delayNanos 方法计算后面 Selector.select 可以阻塞等待的时间。其计算逻辑也很简单，如果存在定时任务，则取定时任务的触发时间与当前时间的间隔；如果不存在，则返回默认间隔， NIOEventLoop 这边是 1s。

接着来看代码②，其逻辑也很简单，如果随着时间流逝，当前时间已经超出了代码①中计算的截止时间，则退出当前循环，也就是意味着离开 select 方法。

接着来看代码③，马上就要执行 `Selector.select(timeout)` 进行阻塞等待，在这之前最后判断一次任务队列是否有了新添加的任务。如果有的话，在属性 wakenUp 执行CAS竞争，竞争成功的话，执行非阻塞 Selector.selectNow 获取可能的就绪事件并返回。代码③的作用是减少不必要的阻塞等待，在有任务的情况下。而如果竞争失败，则按照正常流程走向阻塞等待，只不过另外的线程会执行唤醒，实际上也不会阻塞多少时间。

接着来看代码④，此时已经从选择器的阻塞等待中返回，需要确认返回原因。可能的原因包括：

- 有选择 Key 进入就绪状态。
- 用户线程唤醒了选择器，这种情况意味着外部添加了直接任务或者定时任务。

虽然代码④的判断条件比较多，但是实际应对的情况也只有这两种，而将 wakenUp.get()、hasTasks()、hasScheduledTasks() 只是考虑了任务添加的不同步骤。

接着来看代码⑤，Netty 自身的 EventLoop 不会中断线程，这种调用必然是用户代码完成的，对于这种情况，Netty 也视为离开 Select 方法的条件。

接着来看代码⑥，此时已经超过当初计算的阻塞截止时间，等待下一次循环通过代码②离开 Select 方法。

接着来看代码⑦，这里的代码意味着 Selector 已经触发了JDK的底层Bug，这个问题后续我们开个单篇来具体说明。

总结的来说，阻塞等待的流程就是先计算等待的截止时间，并且在这个时间通过 for 循环执行 `Selector.select(timeout)` 阻塞等待。而在等待前，等待返回后都检查是否存在退出条件：存在就绪 Key，用户添加任务等。满足退出条件就退出循环。

#### **处理就绪 Key**

NioEventLoop 的 run 方法在从 select 方法返回后，就进入处理就绪 Key 和队列任务的环节，也就是在 run 方法中的代码①部分，其代码如下：

```java
cancelledKeys = 0;
needsToSelectAgain = false;
final int ioRatio = this.ioRatio;
if(ioRatio == 100)
{
    try
    {
        processSelectedKeys();
    }
    finally
    {
        runAllTasks();
    }
}
else
{
    final long ioStartTime = System.nanoTime();
    try
    {
        processSelectedKeys();
    }
    finally
    {
        final long ioTime = System.nanoTime() - ioStartTime;
        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
    }
}
```

if 分支和 else 分支的处理逻辑基本相似，区别在于是否配置了 ioRatio 这个参数。这个参数用于调节处理就绪 Key 和处理队列任务的时间比例，其含义是 IO 处理也就是就绪 Key 的处理时间占总时间的比例。但是即使配置为 100，队列任务也是需要处理的，因此当配置为 100 时就走 if 分支，完全限制时间了。

**1. 根据就绪事件不同分发处理逻辑**

那么我们首先来看 processSelectedKeys 方法，也就是就绪 Key 的处理。该处理方法的代码如下：

```java
private void processSelectedKeys()
{
    if(selectedKeys != null)
    {
        processSelectedKeysOptimized();
    }
    else
    {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

Netty 通过反射，对 JDK 提供的内部类 sun.nio.ch.SelectorImpl 进行了优化。优化的效果是获取就绪Key集合时不需要通过 Selector.selectedKeys 方法来得到，而是直接在其 SelectedSelectionKeySet 类内部的数组进行遍历。数组遍历的效率比迭代器要高，不过这个优化不影响其遍历的本质，因此这里我们以非特殊优化的版本 processSelectedKeysPlain 方法作为分析对象。其代码如下：

```java
private void processSelectedKeysPlain(Set < SelectionKey > selectedKeys)
{
    //代码①
    if(selectedKeys.isEmpty())
    {
        return;
    }
    Iterator < SelectionKey > i = selectedKeys.iterator();
    for(;;)
    {
        final SelectionKey k = i.next();
        final Object a = k.attachment();
        //代码②
        i.remove();
        if(a instanceof AbstractNioChannel)
        {
            //代码③
            processSelectedKey(k, (AbstractNioChannel) a);
        }
        else
        {
            @SuppressWarnings("unchecked")
            NioTask < SelectableChannel > task = (NioTask < SelectableChannel > ) a;
            processSelectedKey(k, task);
        }
        if(!i.hasNext())
        {
            break;
        }
        if(needsToSelectAgain)
        {
            selectAgain();
            selectedKeys = selector.selectedKeys();
            if(selectedKeys.isEmpty())
            {
                break;
            }
            else
            {
                i = selectedKeys.iterator();
            }
        }
    }
}
```

先来看下代码①，EventLoop 线程从 Selector.select 阻塞中被唤醒可能是由于添加了用户任务导致，因此这里需要检测就绪事件集合是否为空，避免创建一个空的迭代器。

接着来看代码②，在前文我们介绍原生 NIO 接口操作时也曾介绍过。从就绪事件集合中获取元素后要删除是基本操作，否则下次获取就绪事件集合时，该元素仍然还会在集合内。

一般情况下，我们都是使用 EventLoop 的接口来注册 Netty 的 Channel，但是 NioEventLoop 也提供了接口允许注册 SelectableChannel 对象和与之关联的 NioTask 对象。当 SelectableChannel 有就绪时间发生时，NioTask 中对应的方法会被触发。不过这个方法很少使用，大多数情况下我们都还是在 Netty 提供的 API 层面执行操作。

所以这里我们来看下代码③，也就是对就绪 Key 的处理，该方法的具体代码如下：

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch)
{
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    //代码①
    if(!k.isValid())
    {
        final EventLoop eventLoop;
        try
        {
            eventLoop = ch.eventLoop();
        }
        catch(Throwable ignored){return;}
        if(eventLoop != this || eventLoop == null)
        {
            return;
        }
        unsafe.close(unsafe.voidPromise());
        return;
    }
    try
    {
        int readyOps = k.readyOps();
        if((readyOps & SelectionKey.OP_CONNECT) != 0)
        {
            //代码②：省略代码，用于处理连接就绪事件
        }
        if((readyOps & SelectionKey.OP_WRITE) != 0)
        {
            //代码③：省略代码，用于处理写出就绪事件
        }
        if((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0)
        {
            //代码④：省略代码，用于处理读取就绪或接入就绪事件
        }
    }
    catch(CancelledKeyException ignored)
    {
        unsafe.close(unsafe.voidPromise());
    }
}
```

首先来看下代码①部分。当一个通道从 EventLoop 取消注册，或者被关闭时，都会导致这个就绪Key本身非法。但是两种情况不同，如果是取消注册的话，通道本身仍然是打开可用的。所以如果通道当前尚未绑定 EventLoop 或者绑定的 EventLoop 与当前 EventLoop 不同，只需要单纯的忽略即可。如果是其余情况，则尝试关闭通道。不过在这里的实现实际上有点问题，假设我们设置 NioEventLoopGroup 的大小为 1，则 `if(eventLoop != this || eventLoop == null)` 不为会真，或者说通道重新注册的 EventLoop 恰好又是原来的这个，都可能导致通道被错误关闭。关于这个问题的讨论可以在 issue-5125 中找到，不过最后也没有提供修复方案。

如果就绪 Key 是合法的，就意味着可能存在就绪事件需要处理（除开 JDK Bug 导致的空轮训）。这边有四种就绪事件可以处理，本身也是被定义在 Java NIO 的接口中，分别是：有数据可读就绪事件，有空间可写就绪事件，链接完成就绪事件，链接接入就绪事件。对于服务端监听端口的通道而言，其关注的就绪事件是链接接入就绪事件，所以让我们来到代码④的部分。其余的就绪事件，我们放在后续的内容中说明。

代码④的代码体只有一行，即 unsafe.read()，由于这里的 Channel 类型是 NioServerSocketChannel，所以 unsafe 的具体类是 NioMessageUnsafe。

**2. 连接就绪事件具体处理**

上文提到，处理连接就绪事件的实际方法为 NioMessageUnsafe.read，其方法内容体如下：

```java
public void read()
{
    assert eventLoop().inEventLoop();
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    //代码①
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.reset(config);
    boolean closed = false;
    Throwable exception = null;
    try
    {
        try
        {
            do {
                //代码②
                int localRead = doReadMessages(readBuf);
                if(localRead == 0)
                {
                    break;
                }
                if(localRead < 0)
                {
                    closed = true;
                    break;
                }
                allocHandle.incMessagesRead(localRead);
                //代码③
            } while (allocHandle.continueReading());
        }
        catch(Throwable t)
        {
            exception = t;
        }
        int size = readBuf.size();
        for(int i = 0; i < size; i++)
        {
            readPending = false;
            //代码④
            pipeline.fireChannelRead(readBuf.get(i));
        }
        readBuf.clear();
        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();
        if(exception != null)
        {
            closed = closeOnReadError(exception);
            pipeline.fireExceptionCaught(exception);
        }
        if(closed)
        {
            inputShutdown = true;
            if(isOpen())
            {
                close(voidPromise());
            }
        }
    }
    finally
    {
        if(!readPending && !config.isAutoRead())
        {
            removeReadOp();
        }
    }
}
```

代码比较复杂，我们逐段来看。

**首先是代码①**，用于获取 RecvByteBufAllocator.Handle 实例，其代码如下：

```java
public RecvByteBufAllocator.Handle recvBufAllocHandle()
{
     if(recvHandle == null)
     {
         recvHandle = config().getRecvByteBufAllocator().newHandle();
     }
     return recvHandle;
}
```

这个接口在 Netty 中提供了三种实现，所以这里实现的选择主要取决于 config 对象中存储 RecvByteBufAllocator 具体是什么。而 config 对象是随着 NioServerSocketChannel 初始化而初始化的，如下：

```java
public NioServerSocketChannel(ServerSocketChannel channel)
{
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

因此我们最终得到的 RecvByteBufAllocator.Handle 实现类是 AdaptiveRecvByteBufAllocator.HandleImpl。handler 的作用，主要是用来判断读取过程中单次读取的大小以及是否还有后续数据可以读取等任务。因此在执行读取之前，首先用方法 `allocHandle.reset(config);` 将一些统计数据归零，方便执行后续的判断。

**接着看方法②**，开始真正读取数据。Netty 中将在链接上读取二进制数据和在端口监听链接上获取接入新链接都视为读取操作，不过前者读取的是二进制数据，后者读取的是 Message（这种统一带来了更复杂的编码细节，读者可以考虑是否有可取之处或者是否有其他的做法）。doReadMessages 方法的内容如下：

```java
  protected int doReadMessages(List < Object > buf) throws Exception
  {
      SocketChannel ch = SocketUtils.accept(javaChannel());
      try
      {
          if(ch != null)
          {
              buf.add(new NioSocketChannel(this, ch));
              return 1;
          }
      }
      catch(Throwable t)
      {
          //。。。省略代码，异常处理，关闭通道
      }
      return 0;
  }
```

实际的代码实现也很简单，`SocketUtils.accept(javaChannel())` 的核心就是调用 JDK NIO 的接口，通过 serverSocketChannel.accept() 来返回一个接入的通道对象，也就是 SocketChannel。并且使用 NioSocketChannel 进行封装，并且添加到 buf 中。

**接着看方法③**，方法 allocHandle.continueReading() 用于判断是否继续读取，其代码如下：

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

第二个方法中的表达罗列的有点多，我们分别列开：

- `config.isAutoRead()`
- `(!respectMaybeMoreData || maybeMoreDataSupplier.get())`
- `totalMessages < maxMessagePerRead`
- `totalBytesRead > 0`

第一个条件取决于配置，默认情况下，均为 true。

第二个条件解释下 respectMaybeMoreData。该参数字面含义”肯定（尊重）更多数据的判断“，当参数为 true 时，表达式的值就取决于后面 maybeMoreDataSupplier.get() 的结果，此为肯定（尊重）的含义。该值默认为 true。`maybeMoreDataSupplier.get()` 方法内容体为 `return attemptedBytesRead == lastBytesRead`，由于在 NioServerSocketChannel 读取的是消息，也就是新接入客户端链接对象，因此不存在字节读取，这个表达式始终为 true。

第三个条件在每次接受一个连接后，totalMessages 会加 1，而 maxMessagePerRead 默认情况下为 16（这个 16 默认数字的设置在 ChannelMetadata#defaultMaxMessagesPerRead 中）。在小于 16 时，该条件也会 true。

第四个条件，由于 NioServerSocketChannel 每次读取的是消息，因此 totalBytesRead 是不会变化的，均为 0，因此该表达式恒定为 false。

四个条件结合，对于 NioServerSocketChannel，一次读取一个消息，即一次只接受一个客户端接入的链接，随后结束循环，进入后续流程。

**接着来看代码④**，通过管道传递 channelRead 事件，还记得上篇文章中我们提到，NioServerSocketChannel 在注册到 EventLoop 时会添加一个 ServerBootstrapAcceptor 处理器到自身的管道中。而这个处理器重写了 channelRead 方法来接收这个事件，其代码如下：

```java
public void channelRead(ChannelHandlerContext ctx, Object msg)
{
    final Channel child = (Channel) msg;
    child.pipeline().addLast(childHandler);
    setChannelOptions(child, childOptions, logger);
    for(Entry < AttributeKey <? > , Object > e: childAttrs)
    {
        child.attr((AttributeKey < Object > ) e.getKey()).set(e.getValue());
    }
    try
    {
        childGroup.register(child).addListener(new ChannelFutureListener()
        {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception
            {
                if(!future.isSuccess())
                {
                    forceClose(child, future.cause());
                }
            }
        });
    }
    catch(Throwable t)
    {
        forceClose(child, t);
    }
}
```

通过代码 `child.pipeline().addLast(childHandler)` 给接受的客户端链接通道加入了处理器，这个处理器就是我们 ServerBootStrap 引导程序中写入的 ChannelInitializer 实现类，用于为通道加入用户自定义的业务处理器。可以看到整个方法就是完成了添加初始化处理器 ChannelInitializer，设置通道配置（option）和属性（attribute），而后将其注册到 EventLoop 实例上。这边的 EventLoop 实例由 childGroup 属性提供，而该属性，是由我们引导程序中配置 `ServerBootstrap#group(EventLoopGroup, EventLoopGroup)` 中第二个入决定。也就是之前入门篇我们提到过的 workerGroup。

至此，一个链接的接受处理就完成了。

#### **处理队列任务**

大多数情况下，不会在 NioServerSocketChannel 注册的 EventLoop 上处理队列任务，因为处于职责考虑，其上的 EventLoopGroup 单独为 boss_group，和用于处理读写 IO 的 worker_group 区分开。因此处理队列任务的部分，我们放在后面读取数据的内容分析。

### 总结与思考

本文从 NioEventLoop 的 run 方法出发，分析了其在 NioServerSocketChannel 处理接入连接中承担的作用，并且详细剖析了整个接入过程使用到的代码逻辑。一个客户端链接被接受后，接着就是在这个客户端链接上的读写操作了。下一篇文章我们就来介绍下在读写过程中，Netty 的处理逻辑。