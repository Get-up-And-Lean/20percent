# 15/31Netty 线程模型详解：从演进的角度看源码设计

### 前言

从这篇文章开始，开始进入源码分析阶段。在之前的入门篇和实战篇中，通过代码的编写和各种特性的讲解，让我们对 Netty 有了充分的感性认知。那么接下来，通过对代码的详细走读，来细细查看哪些在感性直觉背后的实现原理。

### Netty 的线程模型

Netty 的线程模型其实没有什么太特别的地方，属于比较自然而然的设计。

首先，基于 Selector 可以同时监控多个链接的特性，很容易想到将所有通道都注册到一个 selector 对象，然后死循环获取通道上的就绪事件并且进行处理。这就是最朴素的设计，也就是作为的**单线程模型**。这种模式，比较适合客户端，用于服务端的话，因为线程数太少，无法有效的利用多核 CPU 的处理能力。

单线程无法有效的利用 CPU，而且在一个 Selector 上管理太多的链接，效率也会下降。很自然的想到利用多线程提升效率。如何使用多线程按照不同的方向扩展又有所区分。单线程模式中的点在于两个：

- 单个 Selector 关注了太多链接，导致一次取出的集合可能很大，遍历耗时太多；
- 所有的链接事件都在一个线程中处理，处理完才能处理下一个链接，导致在后面的链接长时间的等待。

根据这两点，对应的也有两种扩展模式。

**模式一**

针对单个 Selector 的问题，创建一组 Selector 对象，将链接的就绪检查平均的分配到每一个 Selector 对象上。并且为每一个 Selector 对象，绑定一个线程，线程自身执行一个死循环的就绪检查。每当有一个新的链接对象时，使用轮训或者其他策略从 Selector 组中选择一个 Selector，将链接注册上去。可以形象地用图来看：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191019181153.png)

**模式二**

解决单线程处理链接就绪事件的法子很容易想到就是线程池。这种就成了派发模式。主线程检查通道的就绪事件，发现就绪事件后，将就绪事件的处理逻辑包装为一个任务，提交到线程池中处理。提交到线程池的处理速度是十分快的。提交完毕后，主线程继续执行 select 方法，监控通道的就绪事件。可以形象地用图来看：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191019181310.png)

对于模式一，一般称之为**多主模式**。而模式二，因为在就绪事件的处理阶段引入了线程池，常称之为**多线程模式**。

对于模式一而言，还会有细化的演变。使用一组 Selector 来无差别的服务服务端链接和客户端链接显得职责有些不清晰。因此会将 Selector 分为两组：

- 第一组，只服务于服务端链接，其绑定的线程处理客户端接入就绪事件。如果应用程序只有一个服务端监听链接，那么该组的大小为 1。
- 第二组，只服务于客户端链接，其绑定的线程处理客户端读写就绪事件。该组的大小有几种思路，比如 CPU 内核数 +1，比如 CPU 内核数的 2 倍。

这种演进模式由于具备了明显的职责区分，常称之为**主从模式**。大多数公开的材料、博客、Netty 官网以及《Netty 实战》等介绍的 Netty 写法，都是使用主从模式。主从模式可以表达为：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191019181336.png)

实际当中，这几个模式并不会互相排斥，比较常见的有将主从模式和多线程模式结合在一起使用，此时这种模式就是**多线程版本主从模式**。

Netty 对线程模式的支持主要体现在 EventLoopGroup 的配置支持上。通过配置不同个数的 EventLoopGroup 以及在不同地方配置 EventLoopGroup，Netty 可以实现从单线程模式变化为多主模型，再演化为主从模式，最后终极的就是多线程版主从模型。

可以看到，整个模型的演进其实是很自然的事情，并不如一些文章中说的特别精巧或者特意的设计。是一种职责梳理后，根据需要扩展的点，很清晰，很容易就能想到的设计方案。

### Netty 线程模型涉及的类

上一节梳理了 Netty 的线程模型设计和演进思路。这一章节来梳理下线程模型中会涉及到的类。Netty 的线程模型基本上可以从接口 EventLoop 和其实现类的实现细节上看出来。先看下总览的类层次图：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20190911164031.png)

有几个比较明显的意图可以直接从继承关系中看出。继承了 ExecutorService 是为了实现方法 submit 方便进行任务的提交；继承了 ScheduleExecutorService 是为了提交定时任务； EventLoop 继承了 EventLoopGroup 是为了方便后者将任务转发给具体的 EventLoop 线程去执行。

学习 Netty 的线程模型，需要比较深入的了解几个类，分别是：

1. NioEventLoopGroup：负责管理 NioEventLoop。
2. NioEventLoop：实际的线程本身，负责执行具体的 IO 任务和用户任务。

#### **NioEventLoopGroup**

照例仍然是先看下类图，如下：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20190912145627.png)

从类图可以看到，EventLoopGroup 是继承于 EventExecutorGroup，那么先来看下 EventExecutorGroup 这个接口。

**EventExecutorGroup**

先看下接口代码：

```java
public interface EventExecutorGroup extends ScheduledExecutorService, Iterable<EventExecutor> {
    boolean isShuttingDown();
    Future<?> shutdownGracefully();
    Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit);
    Future<?> terminationFuture();
    @Override
    @Deprecated
    void shutdown();
    @Override
    @Deprecated
    List<Runnable> shutdownNow();
    EventExecutor next();
    //以下省略继承的接口方法
}
```

从接口定义上来看，EventExecutorGroup 只新增了 2 类方法：

1. 用于关闭 group 以及获取关闭信息的相关方法。
2. 获取下一个可以执行任务的 EventExecutor 实例。

next 方法实际上提供了一种隐喻和暗示，当向 group 提交任务时，实际上内部可以通过 next 方法取得一个 EventExecutor 实例来真正的执行一个任务。这个暗示会在后文的抽象类实现中被印证。

**AbstractEventExecutorGroup**

该抽象实现覆盖了大部分与执行 runnable 或者 callable 的方法，几乎所有的抽象实现都是通过 next 方法获取一个 EventExecutor 实例，然后将方法的执行委托该给实例的同签名方法。来看下代码会直观，如下：

```java
@Override
public <T> Future<T> submit(Runnable task, T result) {
    return next().submit(task, result);
}
@Override
public <T> Future<T> submit(Callable<T> task) {
    return next().submit(task);
}
```

这个实现方式也印证了上节提出的猜测。

**MultithreadEventExecutorGroup**

该类顾名思义，就是通过管理多个线程进而对外提供服务的 EventExecutorGroup，关注下其构造方法。

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        //省略代码，与错误检查相关。。。
        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }
        children = new EventExecutor[nThreads];
        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                //在这里可以看到，根据给定的线程数，构造待分配使用的 EventExecutor 实例。后续的 next 方法分配从这个数组中选择。
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                //省略代码，与异常输出相关。。。
            } finally {
                //省略的代码，与 EventExecutorGroup 关闭相关。。。
            }
        }
        //省略的代码。。。
    }
```

从构造方法就可以看出，这个类就是在构造方法中事先初始化和储备了一组 EventExecutor 供后续进行分配，也就是供 next 方法进行分配。

比较重要的 newChild 方法则留给了子类去实现。这里额外提一下入参的 executor 对象。如果入参没有赋值，则使用默认实现 ThreadPerTaskExecutor，代码如下：

```java
public final class ThreadPerTaskExecutor implements Executor {
    private final ThreadFactory threadFactory;

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        this.threadFactory = threadFactory;
    }

    @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
}
```

看上去，每次调用 execute 方法都会产生一个新的线程。但实际上，任何一个 EventExecutor 实现内部**只会调用一次该方法**。

**DefaultEventExecutorGroup**

网络上大多数材料，博客和官网在介绍 Netty 的时候，都是使用主从的线程模式，所以这个类基本都见不到。这个类最大的用处是在当 Netty 处于多线程主从模式时，用于承担 ChannelHandler 运行的线程池的角色。常见的用法一般是：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191019115021.png)

前文在讲 Netty 的线程模型的时候曾经提到过这种用法。由于这个线程池中线程不需要涉及 Selector 的操作，因此使用 NIOEventLoopGroup 比较浪费，使用 DefaultEventExecutorGroup 就刚刚好。该类的主要作用就是用来提供方法 newChild 的实现。代码如下：

```java
protected EventExecutor newChild(Executor executor, Object... args) throws Exception {
        return new DefaultEventExecutor(this, executor, (Integer) args[0], (RejectedExecutionHandler) args[1]);
    }
```

关于 DefaultEventExecutor，在后文介绍，这边先留着。

**EventLoopGroup**

EventExecutorGroup 如果不是使用多线程模式，也不会用到，较为少见。EventLoopGroup 因为需要初始化的时候传入，就相对熟悉很多了。

EventLoopGroup 可以认为是特殊化的 EventExecutorGroup。提供了额外的接口用来给 io.netty.channel.Channel 进行注册。从额外提供的注册方法可以看出，EventLoopGroup 主要就是在和 Channel 在打交道。来看下接口方法。

```java
public interface EventLoopGroup extends EventExecutorGroup {
    EventLoop next();
    ChannelFuture register(Channel channel);
    ChannelFuture register(ChannelPromise promise);
    @Deprecated
    ChannelFuture register(Channel channel, ChannelPromise promise);
}
```

主要的功能就是提供了注册接口，用于给通道注册，让通道可以绑定到特定的 Selector 和对应的线程上。同时也重写了 next 方法，将返回值类型变更。整体变更到 EventLoop 体系中。

**MultithreadEventLoopGroup**

继承了 MultithreadEventExecutorGroup，重点就是提供了接口新增的三个 register 方法的实现。其原理也是通过 next 方法获取 EventLoopGroup 实例，委托其执行同签名 register 方法。

**DefaultEventLoopGroup**

这个类的作用和 DefaultEventExecutorGroup 一样，都是为了提供 newChild 方法的具体实现。具体如下：

```java
@Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        return new DefaultEventLoop(this, executor);
    }
```

可以看到和 DefaultEventExecutorGroup 不同的地方就在于返回的类不同。

**方法 next 的具体实现**

上文说到，xxGroup 的大部分方法都是通过 next 方法选择了一个 EventExecutor 实例或者 EventLoop 实例去执行同签名方法。

next 方法的功能思想我们在看 MultithreadEventExecutorGroup 有介绍过。大致是就是从一组实例中选择一个。而该方法的实现，目前在 Netty 中被标准为不稳定。其实现委托了 EventExecutorChooserFactory 工厂方法，生成一个选择器实例，也就是 EventExecutorChooser 接口的实现类。工厂类在 Netty 中存在一个默认实现 DefaultEventExecutorChooserFactory，该工厂方法生成选择器对象有两种可能：

- 传入的 Eventexecutor 为 2 的次方幂个数时，取模有特殊优化，具体的实现类为 PowerOfTwoEventExecutorChooser；
- 其余情况，采用轮训的方式，以普通取模方式得到结果，具体的实现类为 GenericEventExecutorChooser。

无论哪种，都是轮训。只不过是在细节处的代码优化罢了。理解思路最为重要。

**综述**

EventLoopGroup 的实现思路和 JDK 中对 ExecuteService 不同。JDK 中的实现思路是提供一个存储任务的队列，外部调用者将任务放入队列，而 ExecuteService 内部管理着一些线程，这些线程会在这个队列上争抢任务，争抢成功的线程则执行任务，执行完毕后继续在队列上争抢。

而 EventLoopGroup 没有统一存储任务的队列，因此实际上是将任务直接投递到具体的某一个 EventLoop 对象中让其执行。结合《Netty 的线程模型》的分析，EventLoop 本身应该是一个死循环执行的线程，其内部应该具备一个队列用于存储任务，EventLoop 会在死循环的过程中从队列中获取任务以执行。因此 EventLoopGroup 只是承担了一个管理 EventLoop 和关闭的作用，不承担调度职责。因此其实现也是较为简单。

#### **NioEventLoop**

先看下类图，如下：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20190826090817.png)

先来看下类图中的基础接口 EventExecutor。

**EventExecutor**

为了方便 EventExecutorGroup 将方法委托给 EventExecutor，EventExecutor 继承了 EventExecutorGroup 接口。而其本身并没有新增和处理任务相关的方法，只是新增了生成 Future 实例的方法。

**AbstractEventExecutor**

从源码上而言，该类仅仅只是提供了一些基础的方法，并且为个别特定方法提供一个默认实现，这些方法都是与生成 Promise 实例或者 Future 实例相关的。大致上都是类似这种：

```java
public <V> ProgressivePromise<V> newProgressivePromise() {
        return new DefaultProgressivePromise<V>(this);
    }

    @Override
    public <V> Future<V> newSucceededFuture(V result) {
        return new SucceededFuture<V>(this, result);
    }
```

**AbstractScheduledEventExecutor**

该抽象实现提供了对计划类方法的支持。对于计划的支持主要依靠两点：

1. 定义计划任务类 io.netty.util.concurrent.ScheduledFutureTask，用于封装包含计划时间的任务。
2. 使用优先级队列存储计划任务类，排序规则按照任务的计划时间执行排序。不过 Netty 没有使用 JDK 自带的优先级队列，而是采用自定义的实现，不过其采用的数据结构仍然是小顶堆，算法实现也和 JDK 自带的相同。故而不展开。

ScheduledFutureTask 主要是封装了和计划任务相关的一些方法，诸如获取时间，获取在优先级队列中的下标等。本身最重要的存储数据就是该任务的截止时间和周期时间（如果是周期性任务）。

概括而言，计划任务采用的优先级队列来实现。周期性任务则是每一次取出任务后，都再次计算截止时间并且再次放入优先级队列。

**SingleThreadEventExecutor**

单线程实现 EventExecutor 功能的抽象基类。其核心要点如下：

1. 其本身具备未启动、启动、关闭中、关闭，终止 5 个状态。
2. 内部使用 Queue 接口存储 Runnable 对象。
3. 使用 execute 方法首先将 runnable 对象放入队列中。并且如果 Executor 如果还没有启动，则通过 CAS 方式原子的将状态切换到启动，只有 CAS 成功了才能执行后续的步骤。CAS 成功后通过其持有的 java.util.concurrent.Executor 对象执行 execute 方法，执行一个匿名 runnable 对象，该 runnable 对象的内容就是执行 SingleThreadEventExecutor 类的 run 方法（该方法需要子类实现）。而 Executor 对象默认情况下是 ThreadPerTaskExecutor。也就是该次 execute 方法的执行时创建了一个线程来执行这个匿名的 runnable 对象。
4. 提供了 takeTask 方法用于从队列中提取数据。该实现要求 Queue 接口的实现具备 BlockingQueue 接口。该方法会尝试从任务队列和计划队列中同时提取数据，并且计划队列中的任务优先级更高一些。

核心要点基本上阐述了该类的作用和实现的思路。有了思路之后，再继续看代码就显得十分清晰了。来看下最核心的代码 execute 方法，如下：

```java
public void execute(Runnable task) {
           //错误检查，忽略相关代码
        boolean inEventLoop = inEventLoop();
        addTask(task);
        if (!inEventLoop) {
            startThread();
            if (isShutdown()) {
                //忽略代码，与 EventExecutor 关闭状态添加任务处理相关
            }
        }
        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
```

这里面首先是将任务添加到该线程独占的待处理队列中。比较核心的代码就一句 `taskQueue.offer(task)`。这个 taskQueue 就是核心要点 2 提到的 Queue 接口。

当从 EventExecutor 线程外部添加任务时，也就是判断 `if (!inEventLoop)` 为真时，则会尝试启动线程，也就是执行方法 startThread。来看代码：

```java
private void startThread() {
        if (state == ST_NOT_STARTED) {
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                boolean success = false;
                try {
                    doStartThread();
                    success = true;
                } finally {
                    if (!success) {
                        STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                    }
                }
            }
        }
    }
```

只有在线程未启动的情况下，才能执行真正的启动动作，`if (state == ST_NOT_STARTED)` 就是为了保证这一点。为了避免并发冲突，使用 CAS 的方式进行抢占式更新。只有更新成功，也就是 `if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED))` 为真的情况下，才能尝试启动线程。也就是执行方法 doStartThread。该方法有点复杂，但是省略掉和启动无关的代码后就很清晰了，如下：

```java
private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
                //省略代码，内容为设置线程对象属性，设置启动时间属性
                try {
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    //省略代码，内容为在线程关闭时执行的相关清理动作
                }
            }
        });
    }
```

单纯的执行当前对象本身的 run 方法。考虑到线程并不是一次性资源，可以合理的推测出 run 方法的方法体，必然是一个循环。再考虑到 EventExecutor 有一个任务队列。基本上就可以确定，外部或内部线程将任务投递到队列中，然后线程本身死循环从队列中取出任务进行执行。

**DefaultEventExecutor**

通过类层次图和上面的分析可知，该类需要提供一个 run 方法的实现。代码很简单，如下：

```java
protected void run() {
        for (;;) {
            Runnable task = takeTask();
            if (task != null) {
                task.run();
                updateLastExecutionTime();
            }
            if (confirmShutdown()) {
                break;
            }
        }
    }
```

可以看到，思路也很简单，就是在循环中获取任务并且执行。而如果 EventExecutor 的状态变更为关闭时，则退出循环。

SingleThreadEventExecutor 中的 execute 方法只会执行一次 Executor 的 execute 方法也能看出，这里存在一种暗示，也就是子类的 run 方法都需要是通过循环来不断地执行任务，否则只执行一次就会导致线程退出，后续任务就无法执行了。

看到这里，可以对 EventExecutor 的实现做一个总结了。简单概括包含几点：

1. EventExecutorGroup 内部管理着一组 EventExecutor 对象，每一个 EventExecutor 都实现了 runnable 接口，也实际上会被分配一个线程用于执行。
2. EventExecutor 的抽象实现类持有一个 Queue 用于存储任务，该对象需要是接口 BlockingQueue 的实现。
3. EventExecutor 的默认实现类对 run 方法的实现就是一个死循环不断的从 Queue 中获取任务进行执行。
4. 当调用 EventExecutorGroup 方法想要提交任务执行时，EventExecutorGroup 内部使用轮训机制选择一个 EventExecutor 实例，将任务投递到该实例的 Queue 中。

**SingleThreadEventLoop**

主要是提供了关于注册 Channel 的一些相关方法的实现。

相对比 SingleThreadEventExecutor，SingleThreadEventLoop 多实现了接口中关于注册的部分，下面就来看下注册方法：

```
io.netty.channel.SingleThreadEventLoop#register(io.netty.channel.Channel)
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

register 方法则是：

```
io.netty.channel.AbstractChannel.AbstractUnsafe#register
```

如下：

```java
public final void register(EventLoop eventLoop, final ChannelPromise promise)
    {
        //入参检查，包括参数非空检查，是否已经注册过检查，以及判断EventLoop是否是合适当前Channel的对象，省略相关代码
        //...被省略的代码
        //将EventLoop设置到channel的属性上
        AbstractChannel.this.eventLoop = eventLoop;
        if (eventLoop.inEventLoop())
        {
            register0(promise);
        }
        else
        {
            try
            {
                eventLoop.execute(new Runnable()
                {
                    @Override
                    public void run()
                    {
                        register0(promise);
                    }
                });
            }
            catch (Throwable t)
            {
                //异常代码，省略
            }
        }
    }
```

**和 EventLoop 相关的调用都有一个固定的套路**，如果当前线程是 EventLoop 的线程则直接执行对应的操作；如果不是，则把对应的操作封装为一个匿名的 runnable 对象，投递到 eventLoop 中被执行。上述的代码也是使用了这个套路去执行 register0 方法。下面看下 register0 的方法。

```java
private void register0(ChannelPromise promise)
    {
        try
        {
            //检查在真正执行注册之前，通道是否仍然保持打开状态。
            if (!promise.setUncancellable() || !ensureOpen(promise)) {return; }
            boolean firstRegistration = neverRegistered;
            doRegister();
            neverRegistered = false;
            registered = true;
            //先调用 handlerAdded 方法，再设置 promise。避免用户在 FutureListener 中自己调用导致的冲突可能
            pipeline.invokeHandlerAddedIfNeeded();
            safeSetSuccess(promise);
            pipeline.fireChannelRegistered();
            if (isActive())
            {
                //通道可以注册后取消注册再次被注册，只有首次注册才会触发事件
                if (firstRegistration)
                {
                    pipeline.fireChannelActive();
                }
                else if (config().isAutoRead())
                {
                    //如果再次注册，则也需要开始读取数据 
                    beginRead();
                }
            }
        }
        catch (Throwable t)
        {
            //异常处理代码，主要是设置 promise 失败，省略代码
        }
    }
```

register0 的内容可以概括为：

- 执行真正的注册动作，也就是方法 doRegister。
- 注册成功后，对应的通道就有了其绑定的线程，也就是后续该通道的逻辑处理都在这个线程上执行。如果之前已经有 ChannelHandler 添加到这个通道的管道 pipeline 上，那么首先执行这些 ChannelHandler 的 handlerAdded 方法。
- 触发管道的 channelRegistered 事件。
- 如果通道是首次注册到线程，并且处于激活状态，则触发 channelActive 事件。

其中 doRegister 方法是被具体的 Channel 类实现的。对于 NIO 来说，则是被 io.netty.channel.nio.AbstractNioChannel 提供了实现，如下：

```java
protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
                //省略代码，异常处理相关
            }
        }
    }
```

总结来说就是，在 NioChannel 上注册了一个不关心任何就绪事件的 SelectionKey。此时仅仅是完成了注册，但是由于没有关注就绪事件，不会产生效果。对关注的就绪事件集合的更新，在代码的其他地方。

**DefaultEventLoop**

该实现只是为了提供 run 方法的实现，且实现内容也是不断循环取出任务。和 DefaultEventExecutor 是相同的。

实际上在 Netty 中这个实现并不会被使用，没有什么关注意义。

**NioEventLoop**

铺垫了半天，再次说到正主了。一直到 DefaultEventLoop，实际上都是面向普通的任务，和 NIO 都没有任何关系。NioEventLoop 是专门用于处理 NIO 事件的特殊化 EventLoop。其内部实现了对注册通道的读写相关操作。由于内部逻辑较为复杂，直接理解源代码比较麻烦，需要结合场景来进行，因此对于 NioEventLoop 的源码分析将会根据场景区分为《服务端接入客户端链接》《客户端链接读取和写出数据》两个场景来分析其中的源码，在这里先不展开。

**综述**

透过对 EventLoop 接口和实现类的源码分析，该类的设计意图就十分明显，简而言之就是通过自身持有的任务队列，死循环不断的从队列中获取任务进行执行。其他的代码部分都是围绕在这个核心部分，保障其并发正确性的一些必要防御性编程手段，诸如 EventLoop 自身的状态设计，队列接口的实现者选择等。

### 总结与思考

本文我们从线程模型的演变开始，对 Netty 线程模型涉及到的相关类做了源码解析。线程模型的演变分析中，可以梳理出线程模型设计的思路。而这些思路在 Netty 相关源码中都能找到合适的映射。一个设计良好的框架，是可以清晰地回溯思路的。