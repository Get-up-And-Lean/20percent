# 基于 Netty 与 CompletableFuture 模拟 RPC 同步与纯异步调用

### 前言

Netty 是一个异步非阻塞、基于事件驱动的网络编程框架，其使用范围越来越广，比如 Apache RocketMQ、Apache Dubbo 底层网络通信都是使用的 Netty；而 CompletableFuture 则是 JDK 8 中新增的类，其的出现用来解决传统 Future 的缺陷；那么当 Netty 结合 CompletableFuture 时候会产生什么火花？本 Chat 就使用 Netty 与 CompletableFuture 的结合来模拟 RPC（远程过程调用）同步与纯异步调用。

本 Chat 内容如下：

- 基于 Netty 实现 RPC 服务端
- 基于 Netty 实现 RPC 客户端
- 基于 Netty 与 CompletableFuture 实现 RPC 同步调用
- 基于 Netty 与 CompletableFuture 实现 RPC 纯异步调用
- 基于 RxJava 适配 CompletableFuture 实现 Reactive 风格异步调用

### RPC 协议帧定义

大家都知道在客户端与服务端进行网络通信时候，客户端会通过 socket 把需要发送的内容进行序列化为二进制流后发送出去，然后二进制流通过网络流向服务器端，服务端接受到该请求后会解析该请求包，然后反序列化后对请求进行处理。这看似是一个很简单的过程，但是细细想来却会发现没有那么简单。如下图是客户端与服务端交互流程：

![在这里插入图片描述](https://images.gitbook.cn/935bcba0-6c33-11ea-96fb-af457604317b)

如上图首先在客户端发送数据时，实际是把数据写入到了 TCP 发送缓存里面的，如果发送的包的大小比 TCP 发送缓存的容量大，那么这个数据包就会被分成多个包，通过 socket 多次发送到服务端。而服务端获取数据是从接受缓存里面获取的，假设服务端第一次从接受缓存里面获取的数据是整个包的一部分，这时候就产生了半包现象，半包不是说只收到了全包的一半，是说收到了全包的一部分。

服务器读取到半包数据后，会对读取的二进制流进行解析，一般的会把二进制流反序列化为对象，这里服务器由于只读取了客户端序列化对象后的一部分，所以反序列会报错。

同理如果发送的数据包大小比 TCP 发送缓存容量小，并且假设 TCP 缓存可以存放多个包，那么客户端和服务端的一次通信就可能传递了多个包，这时候服务端从接受缓存就可能一下读取了多个包，这时候就出现了粘包现象，由于服务端从接受缓存获取的二进制流是多个对象转换来的，所以在后续的反序列化时候肯定也会出错。

其实出现粘包和半包的原因是 TCP 层不知道上层业务的包的概念，它只是简单的传递流，所以需要上层应用层协议来识别读取的数据是不是一个完整的包。本节我们使用帧分隔符方法来解决半包粘包问题，为简化设计这里我们定义应用层协议帧格式为文本格式，如下图：

![在这里插入图片描述](https://images.gitbook.cn/75727700-6c2f-11ea-8b2e-a93d3912f08f)

如上图帧格式的第一部分为消息体，也就是业务需要传递的内容；第二部分为:号；第三部分为请求 id；这里使用：把消息体与请求 id 分开，是为了让服务端可以方便的提取出来这两部分内容，需要注意消息体内不能含有：号；第四部分|标识一个协议帧的结束，因为本文将要使用 Netty 的 DelimiterBasedFrameDecoder（分隔符界定帧边界）来解决半包粘包问题。对比 Dubbo 的协议帧，本文的协议帧的定义就是一个玩物，不过不要紧，我们只是为了演示使用。

### 基于 Netty 实现 RPC 服务端

首先我们基于 Netty 开发一个简单的 Demo 用来模拟 RpcServer，也就是服务提供方程序，RpcServer 的代码如下：

```Java
public final class RpcServer {
    public static void main(String[] args) throws Exception {
        // 0.配置创建两级线程池
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);// boss
        EventLoopGroup workerGroup = new NioEventLoopGroup();// worker
        // 1.创建业务处理 hander
        NettyServerHandler servrHandler = new NettyServerHandler();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class).option(ChannelOption.SO_BACKLOG, 100)
                    .handler(new LoggingHandler(LogLevel.INFO)).childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline p = ch.pipeline();
                            // 1.1 设置帧分隔符解码器
                            ByteBuf delimiter = Unpooled.copiedBuffer("|".getBytes());
                            p.addLast(new DelimiterBasedFrameDecoder(1000, delimiter));
                            // 1.2 设置消息内容自动转换为 String 的解码器到管线
                            p.addLast(new StringDecoder());
                            // 1.3 设置字符串消息自动进行编码的编码器到管线
                            p.addLast(new StringEncoder());
                            // 1.4 添加业务 hander 到管线
                            p.addLast(servrHandler);
                        }
                    });

            // 2.启动服务，并且在 12800 端口监听
            ChannelFuture f = b.bind(12800).sync();

            // 3. 等待服务监听套接字关闭
            f.channel().closeFuture().sync();
        } finally {
            // 4.优雅关闭两级线程池，以便释放线程
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

如上代码是一个典型的 NettyServer 启动程序，首先代码 0 创建了 NettyServer 的 boss 与 worker 线程池，然后代码 1 创建了业务 NettyServerHandler 这个我们后面具体讲解。

- 代码 1.1 添加 DelimiterBasedFrameDecoder 解码器到链接 channel 的管道以便使用|分隔符来确定一个协议帧的边界（避免半包粘包问题）。
- 代码 1.2 添加字符串解码器，这个在服务端链接 channel 接受到客户端发来的消息后自动把消息内容转换为字符串。
- 代码 1.3 设置字符串编码器，这个是在服务端链接 channel 向客户端写入数据时候，对数据进行编码使用。
- 代码 1.3 添加业务 handler 到管线。

代码 2 启动服务，并且在端口 12800 监听客户端发来的链接；代码 3 同步等待服务监听套接字关闭；代码 4 优雅关闭两级线程池，以便释放线程。

这里我们主要看下业务 handler 的实现，服务端在接受客户端消息，消息内容经过代码 1.1,1.2 的 hanlder 处理后，流转到 NettyServerHandler 的就是一个完整的协议帧的字符串了。NettyServerHandler 代码如下：

```Java
@Sharable
public class NettyServerHandler extends ChannelInboundHandlerAdapter {

    //5. 根据消息内容和请求 id，拼接消息帧
    public String generatorFrame(String msg, String reqId) {
        return msg + ":" + reqId + "|";
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {

        //6.处理请求
        try {
                System.out.println(msg);
                // 1.获取消息体，并且解析出请求 id
                String str = (String) msg;
                String reqId = str.split(":")[1];

                // 2.拼接结果，请求 id,协议帧分隔符(模拟服务端执行服务产生结果)
                String resp =  generatorFrame("im jiaduo ", reqId);

                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                // 3.写回结果
                ctx.channel().writeAndFlush(Unpooled.copiedBuffer(resp.getBytes()));
            } catch (Exception e) {
                e.printStackTrace();
            }
    }

   ...

}
```

如上代码，这里 @Sharable 注解是让服务端所有接受的链接对应的 channel 复用同一个 NettyServerHandler 的实例，这里可以使用@Sharable 方式是因为 NettyServerHandler 内的处理是无状态的，不会存在线程安全问题。

当数据流程到 NettyServerHandler 时候，会调用其 channelRead 方法进行处理，这里 msg 已经是一个完整的本文的协议帧了。

异步任务内代码 6.1 收下获取消息体的内容，然后根据协议格式，从中截取出请求 id,然后调用代码 6.2 拼接返回给客户端的协议帧，需要注意这里需要把请求 id 带回去；然后休眠 2s 模拟服务端任务处理，最后代码 6.3 把拼接好的协议帧写回客户端。

### 基于 Netty 实现 RPC 客户端

下面我们基于 Netty 开发一个简单的 Demo 用来模拟 RpcClient，也就是服务消费方程序，RpcClient 的代码如下：

```Java
public class RpcClient {
    // 连接通道
    private volatile Channel channel;
    // 请求 id 生成器
    private static final AtomicLong INVOKE_ID = new AtomicLong(0);
    // 启动器
    private Bootstrap b;

    public RpcClient() {
        // 1. 配置客户端.
        EventLoopGroup group = new NioEventLoopGroup();
        NettyClientHandler clientHandler = new NettyClientHandler();
        try {
            b = new Bootstrap();
            b.group(group).channel(NioSocketChannel.class).option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline p = ch.pipeline();
                            // 1.1 设置帧分隔符解码器
                            ByteBuf delimiter = Unpooled.copiedBuffer("|".getBytes());
                            p.addLast(new DelimiterBasedFrameDecoder(1000, delimiter));
                            // 1.2 设置消息内容自动转换为 String 的解码器到管线
                            p.addLast(new StringDecoder());
                            // 1.3 设置字符串消息自动进行编码的编码器到管线
                            p.addLast(new StringEncoder());
                            // 1.4 添加业务 Handler 到管线
                            p.addLast(clientHandler);

                        }
                    });
            // 2.发起链接请求，并同步等待链接完成
            ChannelFuture f = b.connect("127.0.0.1", 12800).sync();
            if (f.isDone() && f.isSuccess()) {
                this.channel = f.channel();
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void sendMsg(String msg) {
        channel.writeAndFlush(msg);
    }

    public void close() {

        if (null != b) {
            b.group().shutdownGracefully();
        }
        if (null != channel) {
            channel.close();
        }
    }

    // 根据消息内容和请求 id，拼接消息帧
    private String generatorFrame(String msg, String reqId) {
        return msg + ":" + reqId + "|";
    }

    public CompletableFuture rpcAsyncCall(String msg) {
        // 1. 创建 future
        CompletableFuture<String> future = new CompletableFuture<>();

        // 2.创建消息 id
        String reqId = INVOKE_ID.getAndIncrement() + "";

        // 3.根据消息，请求 id 创建协议帧
        msg = generatorFrame(msg, reqId);

        // 4.nio 异步发起网络请求，马上返回
        this.sendMsg(msg);

        // 5.保存 future 对象
        FutureMapUtil.put(reqId, future);

        return future;
    }

    public String rpcSyncCall(String msg) throws InterruptedException, ExecutionException {
        // 1. 创建 future
        CompletableFuture<String> future = new CompletableFuture<>();

        // 2.创建消息 id
        String reqId = INVOKE_ID.getAndIncrement()  + "";

        // 3.消息体后追加消息 id 和帧分隔符
        msg = generatorFrame(msg, reqId);

        // 4.nio 异步发起网络请求，马上返回
        this.sendMsg(msg);

        // 5.保存 future
        FutureMapUtil.put(reqId, future);

        // 6.同步等待结果
        String result = future.get();
        return result;
    }
}
```

如上代码 RpcClient 的构造函数创建了一个 NettyClient，这个与 NettyServer 类似，不在讲述，需要注意的是这里注册了业务的 NettyClientHandler 处理器到链接 channel 的管线里面，并且在与服务端完成 TCP 三次握手后把对应的 channel 对象保存了下来。

**rpcSyncCall 方法：**

下面我们看 rpcSyncCall 方法，该方法我们意在模拟同步远程调用。

- 其中代码 1 创建了一个 CompletableFuture 对象；
- 代码 2 使用原子变量生成一个请求 id；
- 代码 3 则把业务传递的 msg 消息体和请求 id 组成协议帧；
- 代码 4 则调用 sendMsg 方法通过保存的 channel 对象把协议帧异步发送出去，该方法是非阻塞的，会马上返回，所以不会阻塞业务线程；
- 代码 5 把代码 1 创建的 future 对象保存到 FutureMapUtil 中管理的并发缓存，其中 key 为请求 id，value 为创建的 future、FutureMapUtil 代码如下，可知就是管理并发缓存的一个工具类：

```Java
public class FutureMapUtil {
    // <请求 id，对应的 future>
    private static final ConcurrentHashMap<String, CompletableFuture> futureMap = new ConcurrentHashMap<String, CompletableFuture>();

    public static void put(String id, CompletableFuture future) {
        futureMap.put(id, future);
    }

    public static CompletableFuture remove(String id) {
        return futureMap.remove(id);
    }
}
```

然后代码 6 调用 future 的 get()方法，同步等待 future 的 complete()方法设置结果完成，调用 get()方法会阻塞业务线程直到 future 的结果被设置了。

**rpcAsyncCall 方法：**

下面我们看 rpcAsyncCall 异步调用，其代码实现与同步的 rpcSyncCall 类似，只不过其没有同步等待 future 有结果值，而是直接返回了 future 给调用方，然后就直接返回了，该方法不会阻塞业务线程。

到这里我们讲解了业务调用时候发起远程调用，下面我们看服务端写回结果到客户端后，客户端如何把接入写回对应的 future 的，这我们需要看注册的 NettyClientHandler，其代码如下：

```Java
@Sharable
public class NettyClientHandler extends ChannelInboundHandlerAdapter {
   ...
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 1.根据请求 id，获取对应 future
        CompletableFuture future = FutureMapUtil.remove(((String) msg).split(":")[1]);
        // 2.如果存在，则设置 future 结果
        if (null != future) {
            future.complete(((String) msg).split(":")[0]);
        }
    }
...
}
```

如上代码，当 NettyClientHandler 的 channelRead 方法被调用时候，其中 msg 已经是一个完整的本文的协议帧了（因为 DelimiterBasedFrameDecoder 与 StringDecoder 已经做过解析）。

异步任务内代码 1 首先根据协议帧格式，从消息 msg 内获取到请求 id，然后从 FutureMapUtil 管理的缓存内获取请求 id 对应的 future 对象，并移除；代码 2 如果存在，则从协议帧内获取服务端写回的数据，并调用 future 的 complete 方法把结果设置到 future，这时候由于调用 future 的 get()方法而被阻塞的线程就返回结果了。

### 测试同步与异步调用

上面我们讲解完毕了 RpcClient 与 RpcServer 的实现，下面我们从两个例子看如何使用，首先我们看 TestModelAsyncRpc 的代码：

```Java
public class TestModelAsyncRpc {

    private static final RpcClient rpcClient = new RpcClient();

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        // 1.同步调用
        System.out.println(rpcClient.rpcSyncCall("who are you"));

        // 2.发起远程调用异步，并注册回调，马上返回
        CompletableFuture<String> future = rpcClient.rpcAsyncCall("who are you");
        future.whenComplete((v, t) -> {
            if (t != null) {
                t.printStackTrace();
            } else {
                System.out.println(v);
            }

        });

        System.out.println("---async rpc call over");
    }
}
```

如上 main 函数内首先创建了一个 rpcClient 对象，然后代码 1 同步调用了其 rpcSyncCall 方法，由于是同步调用，所以在服务端执行返回结果前当前调用线程会被阻塞，直到服务端把结果写回客户端，并且客户端把结果写回到对应的 future 对象后才会返回。

代码 2 调用了异步方法 rpcAsyncCall，其不会阻塞业务调用线程，而是马上返回一个 CompletableFuture 对象，然后我们在其上设置了一个回调函数，意在等 future 对象的结果被设置后进行回调，这个实现了真正意义上的异步。

然后我们在看一个使用实例，演示如何基于 CompletableFuture 的能力，并发发起多次调用，然后对返回的多个 CompletableFuture 进行运算，这我们看 TestModelAsyncRpc2 类：

```Java
public class TestModelAsyncRpc2 {

    private static final RpcClient rpcClient = new RpcClient();

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        // 1.发起远程调用异步，马上返回
        CompletableFuture<String> future1 = rpcClient.rpcAsyncCall("who are you");
        // 2.发起远程调用异步，马上返回
        CompletableFuture<String> future2 = rpcClient.rpcAsyncCall("who are you");

        // 3.等两个请求都返回结果时候，使用结果做些事情
        CompletableFuture<String> future = future1.thenCombine(future2, (u, v) -> {

            return u + v;
        });

        // 4.等待最终结果
        future.whenComplete((v, t) -> {
            if (t != null) {
                t.printStackTrace();
            } else {
                System.out.println(v);
            }

        });
        System.out.println("---async rpc call over---");
        // rpcClient.close();

    }

}
```

如上代码 1 首先发起一次远程调用，该调用马上返回 future1；然后代码 2 又发起一次远程调用，该调用也马上返回 future2 对象；代码 3 则基于 CompletableFuture 的能力，意在让 future1 和 fuuture2 都有结果后在基于两者的结果做一件事情（这里是拼接两者结果返回），并返回一个获取回调结果的新的 future。

代码 4 基于新的 future，等其结果产生后，执行新的回调函数，进行结果打印或者异常打印。

### 基于 RxJava 适配 CompletableFuture 实现 Reactive 风格异步调用

最后我们看如何把异步调用改造为 Reactive 编程风格，这里我们基于 RxJava 让异步调用返回结果为 Flowable，其实我们只需要把返回的 CompletableFuture 转换为 Flowable 即可，我们可以在 RpcClient 里面新增一个方法如下：

```Java
    // 异步转反应式
    public Flowable<String> rpcAsyncCallFlowable(String msg) {
        // 1.1 使用 defer 操作，当订阅时候在执行 rpc 调用
        return Flowable.defer(() -> {
            // 1.2 创建含有一个元素的流
            final ReplayProcessor<String> flowable = ReplayProcessor.createWithSize(1);
            // 1.3 具体执行 RPC 调用
            CompletableFuture<String> future = rpcAsyncCall(msg);
            // 1.4 等 rpc 结果返回后设置结果到流对象
            future.whenComplete((v, t) -> {
                if (t != null) {// 1.4.1 结果异常则发射错误信息
                    flowable.onError(t);
                } else {// 1.4.2 结果 OK，则发射出 rpc 返回结果
                    flowable.onNext(v);
                    // 1.4.3 结束流
                    flowable.onComplete();
                }
            });
            return flowable;
        });
    }
```

如上代码由于 CompletableFuture 是可以设置回调函数的，所以把其转换为 Reactive 风格编程很容易。

然后我们可以使用下面代码进行测试：

```Java
public class TestModelAsyncRpcReactive {
    // 1.创建 rpc 客户端
    private static final RpcClient rpcClient = new RpcClient();

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        // 2.发起远程调用异步，并注册回调，马上返回
        Flowable<String> result = rpcClient.rpcAsyncCallFlowable("who are you");
        //3.订阅流对象
        result.subscribe(/* onNext */r -> {
            System.out.println(Thread.currentThread().getName() + ":" + r);
        }, /* onError */error -> {
            System.out.println(Thread.currentThread().getName() + "error:" + error.getLocalizedMessage());
        });

        System.out.println("---async rpc call over");
    }
}
```

如上代码我们发起 RPC 调用后马上返回了一个 Flowable 流对象，这时真正的 RPC 调用还没有发出去，等代码 3 订阅了流对象时候才真正发起 RPC 调用。

### 总结

本文基于 Netty 与 CompletableFuture 的结合来模拟 RPC（远程过程调用）同步与纯异步调用，希望大家动手实践，以便加深理解。

最后作者的[《Java 异步编程实战》](https://item.jd.com/12778422.html)一书已经出版，对异步编程感兴趣的同学可以去购买学习下。