# Java NIO 框架 Netty 之美：源码剖析之一

### 一、前言

Netty 是一个可以快速开发网络应用程序的 NIO 框架，它大大简化了 TCP 或者 UDP 服务器的网络编程。Netty 的简易和快速开发并不意味着由它开发的程序将失去可维护性或者存在性能问题，它的设计参考了许多协议的实现，比如 FTP、SMTP、HTTP 和各种二进制和基于文本的传统协议，因此 Netty 成功的实现了兼顾快速开发，性能，稳定性，灵活性为一体，不需要为了考虑一方面原因而妥协其他方面。Netty 的应用还是比较广泛的，比如阿里巴巴开源的 Dubbo 和 Sofa-Bolt 框架底层网络通讯都是基于 Netty 来实现的。

本 Chat 作为 Netty 系列的源码剖析篇，主要包含下面内容：

- Netty Server 启动源码剖析，您将能学到服务端如何进行初始化，何时接受客户端请求，何时注册接受 socket 并注册到对应的 EventLoop 管理的 selector 等
- Netty Client 启动源码剖析，您将能学到客户端如何进行初始化，何时创建的 DefaultChannelPipeline 等
- Netty 零拷贝技术内幕

> 本 chat 使用 Netty 版本：netty-all-4.1.13.Final

### 二、Netty Server 启动源码剖析

Netty 服务端启动类是 ServerBootstrap，下面我们首先看下其启动的核心时序图：

![enter image description here](https://images.gitbook.cn/96f0c840-d2c6-11e8-85fe-917e42ad7a85)

- 如上时序图，当调用 ServerBootstrap 的 bind 方法后，会具体执行到 initAndRegister 方法，该方法内在时序图的步骤 4 是创建一个服务端端套接字通道 NioServerSocketChannel 实例，NioServerSocketChannel 的构造函数内通过

```
    private static ServerSocketChannel newSocket(SelectorProvider provider) {
        try {

            return provider.openServerSocketChannel();
        } catch (IOException e) {
            throw new ChannelException(
                    "Failed to open a server socket.", e);
        }
    }
```

创建了一个 ServerSocketChannel 对象（就是 JavaNIO 里面的服务端套接字通道），然后设置 ServerSocketChannel 为非阻塞模式：

```
 protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
        super(parent);
        this.ch = ch;
        this.readInterestOp = readInterestOp;
        try {
            ch.configureBlocking(false);
        } catch (IOException e) {
            try {
                ch.close();
            } catch (IOException e2) {
                if (logger.isWarnEnabled()) {
                    logger.warn(
                            "Failed to close a partially initialized socket.", e2);
                }
            }

            throw new ChannelException("Failed to enter non-blocking mode.", e);
        }
    }
```

然后通过下面代码，在其父类构造函数创建了默认管线

```
    protected AbstractChannel(Channel parent) {
        this.parent = parent;
        id = newId();
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }
    protected DefaultChannelPipeline newChannelPipeline() {
        return new DefaultChannelPipeline(this);
    }
```

- 创建完毕 NioServerSocketChannel 后，然后时序图中步骤（5）对其管理的服务端套接字选项进行设置，其中套接字选项设置的 init 方法中通过下面代码添加了一个 ServerBootstrapAcceptor 到管线，这个很重要，后面会提起

```
 p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
```

- 通过上面步骤完成了 NioServerSocketChannel 的初始化，下面我们看如何注册 NioServerSocketChannel 到对应的选择器上的，由于是服务端套接字，所以会从 Boss NioEventLoopGroup 中选择一个 NioEventLoop，然后注册套接字到 NioEventLoop 管理的选择器上，我们知道一个 NioEventLoopGroup 里面会含有多个 NioEventLoop，那么具体选择那一个那？Netty 中根据 NioEventLoopGroup 管理的 NioEventLoop 的个数具体选择使用那种选择策略：

```
    public EventExecutorChooser newChooser(EventExecutor[] executors) {
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTwoEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }
```

其中 PowerOfTwoEventExecutorChooser 逻辑如下：

```
 private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[idx.getAndIncrement() & executors.length - 1];
        }
    }
```

其中 GenericEventExecutorChooser 逻辑如下：

```
 private static final class GenericEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        GenericEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[Math.abs(idx.getAndIncrement() % executors.length)];
        }
    }
```

通过选择策略会从 group 中选择一个 NioEventLoop，然后会把服务端套接字通道注册到 NioEventLoop 对应的选择器上，下面我们看 NioEventLoop 的 register 方法：

```
    public ChannelFuture register(final ChannelPromise promise) {
        ObjectUtil.checkNotNull(promise, "promise");
        promise.channel().unsafe().register(this, promise);
        return promise;
    }
```

其中这里 unsafe 为 NioMessageUnsafe，所以我们看其 register 方法，内部最后会调用 doRegister：

```
  protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
            //具体注册套接字到selector
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
               ...
            }
        }
    }
```

到这里我们分析完了服务端如何进行初始化，何时注册接受 socket 并注册到对应的 EventLoop 管理的 selector，何时创建的默认管线等，下面我们看服务端何时接受客户端请求的。

下面我们看 NioMessageUnsafe 的 read 方法：

```
public void read() {
...            
            try {
                try {
                    do {
                        //I获取一个客户端连接，并放入readbuf list
                        int localRead = doReadMessages(readBuf);
                        if (localRead == 0) {
                            break;
                        }
                        if (localRead < 0) {
                            closed = true;
                            break;
                        }

                        allocHandle.incMessagesRead(localRead);
                    } while (allocHandle.continueReading());
                } catch (Throwable t) {
                    exception = t;
                }
                //II 传递每个连接套接字对象到管线，激活管线的channelRead方法
                int size = readBuf.size();
                for (int i = 0; i < size; i ++) {
                    readPending = false;
                    pipeline.fireChannelRead(readBuf.get(i));
                }
                readBuf.clear();
                allocHandle.readComplete();
                //III 激活管线的readComplete方法
                pipeline.fireChannelReadComplete();
                //Iv 如何出现异常 激活管线的exceptionCaught方法
                if (exception != null) {
                    closed = closeOnReadError(exception);

                    pipeline.fireExceptionCaught(exception);
                }

               ...
            } finally {
                // Check if there is a readPending which was not processed yet.
               ...
            }
        }
    }
```

如上代码比较简单，下面我们来看 I 中的 doReadMessages 方法：

```
 protected int doReadMessages(List<Object> buf) throws Exception {
        SocketChannel ch = SocketUtils.accept(javaChannel());

        try {
            if (ch != null) {
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }
        } catch (Throwable t) {
            logger.warn("Failed to create a new channel from an accepted socket.", t);

            try {
                ch.close();
            } catch (Throwable t2) {
                logger.warn("Failed to close a socket.", t2);
            }
        }

        return 0;
    }
    public static SocketChannel accept(final ServerSocketChannel serverSocketChannel) throws IOException {
        try {
            return AccessController.doPrivileged(new PrivilegedExceptionAction<SocketChannel>() {
                @Override
                public SocketChannel run() throws IOException {
                    return serverSocketChannel.accept();
                }
            });
        } catch (PrivilegedActionException e) {
            throw (IOException) e.getCause();
        }
    }
```

可知是调用了 serverSocketChannel.accept() 来获取一个完成 TCP 三次握手的客户端连接套接字通道。

其中 II 传递每个连接套接字对象到管线，激活管线的 channelRead 方法，还记得在服务端套接字初始化时候，套接字选项设置的 init 方法中通过下面代码添加了一个 ServerBootstrapAcceptor 到管线，那么 ServerBootstrapAcceptor 的 channelRead 也是会被调用的，其主要作用是给完成 tcp 三次握手的套接字通道进行初始化，并注册到 worker 线程池的 NioEventLoop 管理的选择器上，如下代码：

```
  public void channelRead(ChannelHandlerContext ctx, Object msg) {
            final Channel child = (Channel) msg;

            child.pipeline().addLast(childHandler);

            setChannelOptions(child, childOptions, logger);

            for (Entry<AttributeKey<?>, Object> e: childAttrs) {
                child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }

            try {
                childGroup.register(child).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }
```

### 三、Netty Client 启动源码剖析

下面首先我们来看 Bootstrap 的启动时序图：

![enter image description here](https://images.gitbook.cn/fa0c0750-d2c6-11e8-85fe-917e42ad7a85)

- 如上时序图首先调用了 connect 方法，其中步骤（3）与服务端初始化一致，都是调用 initAndRegister 方法创建套接字通道然后进行初始化，最后注册到对应的 NioEvenLoop 管理的选择器上，不同在于 client 端创建的套接字通道为 NioSocketChannel，服务端创建的套接字通道为 NioServerSocketChannel。

其中 NioSocketChannel 的构造函数如下：

```
 public NioSocketChannel() {
        this(DEFAULT_SELECTOR_PROVIDER);
    }


    public NioSocketChannel(SelectorProvider provider) {
        this(newSocket(provider));
    }


    public NioSocketChannel(SocketChannel socket) {
        this(null, socket);
    }
 private static SocketChannel newSocket(SelectorProvider provider) {
        try {

            return provider.openSocketChannel();
        } catch (IOException e) {
            throw new ChannelException("Failed to open a socket.", e);
        }
    }
```

可知 NioSocketChannel 内部管理了一个 Java NIO 里面的 SocketChannel 对象。

时序图7 init 方法是对 NioSocketChannel 进行初始化：

```
 void init(Channel channel) throws Exception {
        ChannelPipeline p = channel.pipeline();
        p.addLast(config.handler());

        final Map<ChannelOption<?>, Object> options = options0();
        synchronized (options) {
            setChannelOptions(channel, options, logger);
        }

        final Map<AttributeKey<?>, Object> attrs = attrs0();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                channel.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }
        }
    }
```

可知是根据配置项对套接字选项 和 管线进行设置。时序图 8 是注册 NioSocketChannel 管理的套接字通道到选择器 ，这个与服务端流程一致，不在讲述。

- 代码 18、19 等套接字注册完毕后，调用 NioSocketChannel 的 connect 方法向服务端发起 TCP 三次握手请求。

无论客户端还是服务器端通道，其内部管理的套接字通道最终都注册到了其对应的 NioEventLoop 上，多个套接字通道可以注册到同一个 NioEventLoop 上，但是同一个套接字通道只能注册到一个 NioEventLoop 上。并且服务端接受套接字是注册到了 boss NioEventLoop 上的，其接受到的完成 TCP 三次握手的连接套接字则是注册到 work NioEventLoop 上的。

每个 NioEventLoop 实际上是一个线程，其内部维护了一个 selector 选择器，线程对注册到 selector 选择器上的套接字通道事件进行监控，发现有读写请求或者连接请求后就会对其进行处理，比如读取发客户端来的数据，然后把数据传递给管线，让管线里面的 handler 依次进行处理。

### 四、Netty 零拷贝技术内幕

#### 4.1 Java IO 模型

![enter image description here](https://images.gitbook.cn/523a2fb0-d2c7-11e8-85fe-917e42ad7a85)

如上图当网络应用进程向 socket 写入数据时候，首先需要在应用程序内申请一个写 buffer，然后把数据写入到写 buffer，然后应用程序的执行会用用户态切换到核心态，核心态程序把应用程序写 buffer 里面的数据拷贝到操作系统层面的写缓存里面。当应用程序读取数据时候是需要把操作系统层面的读 buffer 里面的数据拷贝到应用程序层面的读 buffer 里面。一般情况下应用程序层面的 buffer 都是从堆空间里面申请的，这就需要在用户态和核心态之间数据传输时候进行一次数据 copy。这是因为核心态是不能直接应用程序堆内存的，必须转换为直接内存。

这是因为核心态是不能直接应用程序堆内存的，必须转换为直接内存, 我们可以看下 NIO 里面使用的 rt.jar 包里面的 IOUtil 类的读写方法：

```
static int write(FileDescriptor paramFileDescriptor, ByteBuffer paramByteBuffer, long paramLong, NativeDispatcher paramNativeDispatcher)
    throws IOException
  {
    //I 如果是直接内存，则直接调用native方法写入
    if ((paramByteBuffer instanceof DirectBuffer)) {
      return writeFromNativeBuffer(paramFileDescriptor, paramByteBuffer, paramLong, paramNativeDispatcher);
    }

  // II 否者创建一个临时的直接内存缓存
    int i = paramByteBuffer.position();
    int j = paramByteBuffer.limit();
    assert (i <= j);
    int k = i <= j ? j - i : 0;
    ByteBuffer localByteBuffer = Util.getTemporaryDirectBuffer(k);
   //复制内容到临时直接内存
    try {
      localByteBuffer.put(paramByteBuffer);
      localByteBuffer.flip();

      paramByteBuffer.position(i);
      // III 调用native方法写入
      int m = writeFromNativeBuffer(paramFileDescriptor, localByteBuffer, paramLong, paramNativeDispatcher);
      if (m > 0)
      {
        paramByteBuffer.position(i + m);
      }
      return m;
    } finally {
    //IV释放临时直接内存
      Util.offerFirstTemporaryDirectBuffer(localByteBuffer);
    }
  }
static int read(FileDescriptor paramFileDescriptor, ByteBuffer paramByteBuffer, long paramLong, NativeDispatcher paramNativeDispatcher)
    throws IOException
  {
    if (paramByteBuffer.isReadOnly())
      throw new IllegalArgumentException("Read-only buffer");
    //如果是直接内存，则直接把数据读入直接内存
    if ((paramByteBuffer instanceof DirectBuffer)) {
      return readIntoNativeBuffer(paramFileDescriptor, paramByteBuffer, paramLong, paramNativeDispatcher);
    }

    //如果不是是直接内存，则先创建一个直接内存缓存，数据读入缓存
    ByteBuffer localByteBuffer = Util.getTemporaryDirectBuffer(paramByteBuffer.remaining());
    try {
      int i = readIntoNativeBuffer(paramFileDescriptor, localByteBuffer, paramLong, paramNativeDispatcher);
      localByteBuffer.flip();
  //从直接缓存把数据写入用户传递的堆内存里面
      if (i > 0)
        paramByteBuffer.put(localByteBuffer);
      return i;
    } finally {
//释放直接缓存
      Util.offerFirstTemporaryDirectBuffer(localByteBuffer);
    }
  }
```

如果用户态申请的堆外内存（直接内存）那么就会省去中间的拷贝操作，操作系统层面会直接使用用户态申请的堆外内存（直接内存）里面的数据。而 netty 就是在从 socket 进行读写时候使用的直接缓存，这省去了一次数据拷贝，所以相比之前来说 netty 做到了零拷贝。

#### 4.2 堆外内存的使用

Java NIO 中提供了一个 ByteBuffer 用来分配发送和接受缓存用的，其分为两种模式的内存，一个是我们常用的堆内存，一个是堆外内存（直接内存）。

当调用 ByteBuffer 的 allocateDirect 方法时候，分配的就是堆外内存：

```
    public static ByteBuffer allocateDirect(int capacity) {
        return new DirectByteBuffer(capacity);
    }
    protected static final Unsafe unsafe = Bits.unsafe();

 DirectByteBuffer(int cap) {                   // package-private

        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);
        //1 使用Unsafe分配堆外内存
        long base = 0;
        try {
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
       //2 对分配的堆外内存进行初始化
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
       //3.创建堆外内存回收器
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }
```

可知堆外内存是使用 UNSAFE 类进行内存分配的。

### 五、参考

- 《Netty 权威指南》

### 六、其他

- 《[Java 并发编程之美](https://item.jd.com/12450812.html)》一书已经上架，欢迎订购