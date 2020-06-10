# 谈谈Netty的线程模型

### 前言

Netty 是一个异步、基于事件驱动的网络应用程序框架，其对 Java NIO 进行了封装，大大简化了 TCP 或者 UDP 服务器的网络编程。其应用还是比较广泛的，比如 Apache Dubbo 、Apache RocketMq、Zuul 2.0 服务网关、Spring WebFlux、Sofa-Bolt 底层网络通讯都是基于 Netty 来实现的，本节我们谈谈 Netty4 中的线程模型。

### Netty 的线程模型

![在这里插入图片描述](https://images.gitbook.cn/0f863400-c6e3-11e9-a81a-91f9bfe6443e)

如上图下侧为 Netty Server 端,当 NettyServer 启动时候会创建两个 NioEventLoopGroup 线程池组，其中 boss 组用来接受客户端发来的连接，worker 组则负责对完成 TCP 三次握手的连接进行处理；如上图每个 NioEventLoopGroup 里面包含了多个 NioEventLoop，每个 NioEventLoop 中包含了一个 NIO Selector、一个队列、一个线程；其中线程用来做轮询注册到 Selector 上的 Channel 的读写事件和对投递到队列里面的事件进行处理。

当 NettyServer 启动时候会注册监听套接字通道 NioServerSocketChannel 到 boss 线程池组中的某一个 NioEventLoop 管理的 Selector 上，然后其对应的线程则会负责轮询该监听套接字上的连接请求；当客户端发来一个连接请求时候，boss 线程池组中注册了监听套接字的 NioEventLoop 中的 Selector 会读取读取完成了 TCP 三次握手的请求，然后创建对应的连接套接字通道 NioSocketChannel，然后把其注册到 worker 线程池组中的某一个 NioEventLoop 中管理的一个 NIO Selector 上，然后该连接套接字通道 NioSocketChannel 上的所有读写事件都由该 NioEventLoop 管理。

当客户端发来多个连接时候，NettyServer 端则会创建多个 NioSocketChannel，而 worker 线程池组中的 NioEventLoop 是有个数限制的，所以 Netty 有一定的策略把很多 NioSocketChannel 注册到不同的 NioEventLoop 上，也就是每个 NioEventLoop 中会管理好多客户端发来的连接，然后通过循环轮询处理每个连接的读写事件。

如上图上侧部分为 Netty Client 部分，当 NettyClient 启动时候会创建一个 NioEventLoopGroup，用来发起请求并对建立 TCP 三次连接的套接字的读写事件进行处理。当调用 Bootstrap 的 connect 方法发起连接请求后内部会创建一个 NioSocketChannel 用来代表该请求，并且会把该 NioSocketChannel 注册到 NioSocketChannel 管理的某个 NioEventLoop 的 Selector 上，然后该 NioEventLoop 的读写事件都有该 NioEventLoop 负责处理。

Netty 之所以说是异步非阻塞网络框架是因为通过 NioSocketChannel 的 write 系列方法向连接里面写入数据时候是非阻塞的，马上会返回的，即使调用写入的线程是我们的业务线程，这是 Netty 通过在 ChannelPipeline 中判断调用 NioSocketChannel 的 write 的调用线程是不是其对应的 NioEventLoop 中的线程来实现的，如果发现不是则会把写入请求封装为 WriteTask 投递到其对应的 NioEventLoop 中的队列里面，然后等其对应的 NioEventLoop 中的线程轮询连接套接字的读写事件时候捎带从队列里面取出来执行；总结说就是每个 NioSocketChannel 对应的读写事件都是在其对应的 NioEventLoop 管理的单线程内执行，对同一个 NioSocketChannel 不存在并发读写，所以无需加锁处理。

另外当从 NioSocketChannel 中读取数据时候，并不是使用业务线程来阻塞等待，而是等 NioEventLoop 中的 IO 轮询线程发现 Selector 上有数据就绪时候，通过事件通知方式来通知我们业务数据已经就绪，可以来读取并处理了。

总结一句话就是使用 Netty 框架进行网络通信时候，当我们发起请求后请求会马上返回，而不会阻塞我们的业务调用线程；如果我们想要获取请求的响应结果，也不需要业务调用线程使用阻塞的方式来等待，而是当响应结果出来时候使用 IO 线程异步通知业务的方式，可知在整个请求-响应过程中业务线程不会由于阻塞等待而不能干其他事情。

下面我们讨论两个细节，第一是完成 TCP 三次握手的套接字应该注册到 worker 线程池中的哪一个 NioEventLoop 的 Selector 上，第二个是 NioEventLoop 中的线程负责监听注册到 Selector 上的所有连接的读写事件和处理队列里面的消息，那么会不会导致由于处理队列里面任务耗时太长导致来不及处理连接的读写事件？

对于第一个问题 NioEventLoop 的分配，Netty 默认使用的是 PowerOfTwoEventExecutorChooser，其代码如下：

```
    private final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
        @Override
        public EventExecutor next() {
            return children[childIndex.getAndIncrement() & children.length - 1];
        }
    }
```

可知是采用的轮询取模方式来进行分配。

对于第二个问题，Netty 默认是采用时间均分策略来避免某一方处于饥饿状态:

```
//1.记录开始处理时间
final long ioStartTime = System.nanoTime();
try {//1.1 处理连接套接字的读写事件
    processSelectedKeys();
} finally {
    // 1.2 计算连接套接字处理耗时，ioRatio 默认为 50
    final long ioTime = System.nanoTime() - ioStartTime;
    //1.3 运行队列里面任务
    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
}               
```

如上代码 1.1 处理所有注册到当前 NioEventLoop 的 Selector 上的所有连接套接字的读写事件，代码 1.2 用来统计其耗时，由于默认情况下 ioRatio 为 50，所以代码 1.3 尝试使用与代码 1.2 执行相同的时间来运行队列里面的任务。 也就是处理套接字读写事件与运行队列里面任务是使用时间片轮转方式轮询执行。

### 总结

Netty 的异步非阻塞基于事件驱动的模型大大简化了我们编写网络应用程序的成本。

更多技术分享，请关注微信公众号：技术原始积累；另外[《Java 并发编程之美》](https://item.jd.com/12450812.html)已经出版，对并发感兴趣的同学可以参考下。