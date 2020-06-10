# Java NIO 框架 Netty 之美： 搭建文件上传服务器

### 一、前言

Netty 是一个可以快速开发网络应用程序的 NIO 框架，它大大简化了 TCP 或者 UDP 服务器的网络编程。Netty 的简易和快速开发并不意味着由它开发的程序将失去可维护性或者存在性能问题，它的设计参考了许多协议的实现，比如 FTP、SMTP、HTTP 和各种二进制和基于文本的传统协议，因此 Netty 成功的实现了兼顾快速开发，性能，稳定性，灵活性为一体，不需要为了考虑一方面原因而妥协其他方面。Netty 的应用还是比较广泛的，比如阿里巴巴开源的 Dubbo 和 Sofa-Bolt 框架底层网络通讯都是基于 Netty 来实现的。

本 Chat 我们来使用 Netty 搭建一个文件上传服务，主要包含下面内容：

- 使用 Netty 搭建文件上传服务端，自定义协议格式，自定义解码处理器，允许一个长连接连续上传多个文件。
- 使用 Netty 搭建文件上传客户端，从本地读取文件，按照自定义协议格式通过 Netty 连接传输文件到服务器端 。

### 二、文件上传服务设计

我们知道通过网络传输数据时候只是传递二进制流，比如客户端通过网络 socket 给服务器端发送数据时候，客户端把要发送的数据对象序列化为二进制后，通过 socket 连接直接发送给了服务器，服务器则要负责解决半包和粘包问题（参考：[Java NIO 框架 Netty 之美：粘包与半包问题](https://gitbook.cn/books/5b164aafc059e77005d11d0c/index.html)），同理客户端上传一个文件时候，也是把磁盘文件转换为了二进制流通过 socket 连接发送给文件服务器，那么服务器端则也要解决半包和粘包问题，也就是服务器要能知道一个文件的边界（读取到那个字节时候才是一个完整的文件），常用的解决半包粘包的问题有三种：包定长、分隔符、自定义帧（header+body），本文简单的使用 header+body 的变体，如下图方式来解决文件边界问题：

![enter image description here](https://images.gitbook.cn/6f683630-d692-11e8-80d1-917ffa8847fa)

如上图协议首先是 4 个字节用来记录文件的长度，然后使用 128 字节用来存放文件名称，然后接下来的 fileLength 个字节是文件的具体内容。另外本文搭建的简单文件上传服务器支持同一个链接传递多个文件，这时候字节流的格式就如下图两个文件在网络流传输时候的示意图：

![enter image description here](https://images.gitbook.cn/7e14ecf0-d692-11e8-80d1-917ffa8847fa)

到这里位置我们设计了一个简单的包协议，客户端上传在链接建立完毕后只需要先向 socket 写入一个 4 字节的 int 类型的数据来代表要上传文件的大小，然后传递 128 字节的字符串来代表文件的名字，最后从磁盘读取文件流并依次写入到 socket 就可以了。如果要上传多个文件则重复上述过程。

下面我们来看看文件服务器这边需要做什么设计：首先服务器端需要写一个文件解码器，这个解码器的工作是根据上面指定的协议，解析出一个文件，然后传递解析的文件到业务 handler，业务 handler 则对文件进行落盘处理或者保存文件到其他存储。这里有一个点需要注意是服务器等整个文件在内存里面接受完毕后在一次性写入磁盘，还是接受一部分就写入磁盘，后面接受的数据在追加到磁盘文件中。本文我们简单的使用前者进行处理。

### 三、 Netty 搭建文件上传服务端

本节使用 Spring Boot+Netty 搭建了一个简单的文件服务器，本节代码可以在 https://github.com/zhailuxu/netty-upload-server 下载。

首先我们来看看服务器端启动服务类 NettyFileUploadServer：

```Java
public class NettyFileUploadServer {

    private static final int PORT = Integer.parseInt(System.getProperty("port", "8080"));


    // （1.1）创建主从Reactor线程池
    private EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    private EventLoopGroup workerGroup = new NioEventLoopGroup();

    //启动服务
    public void init() throws Exception{


        Thread thread  = new Thread(new Runnable() {

            @Override
            public void run() {
                try {
                    // 1.2创建启动类ServerBootstrap实例，用来设置客户端相关参数
                    ServerBootstrap b = new ServerBootstrap();
                    b.group(bossGroup, workerGroup)// 1.2.1设置主从线程池组
                            .channel(NioServerSocketChannel.class)// 1.2.2指定用于创建客户端NIO通道的Class对象
                            .handler(new LoggingHandler(LogLevel.INFO))// 1.2.4设置日志handler
                            .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                            .childHandler(new ChannelInitializer<SocketChannel>() {// 1.2.5设置用户自定义handler
                                @Override
                                public void initChannel(SocketChannel ch) throws Exception {
                                    ChannelPipeline p = ch.pipeline();
                                    // 1.2.5.1 文件格式解析器
                                    p.addLast(new FileDecoder());
                                    // 1.2.5.2 业务处理器
                                    p.addLast(new NettyServerHandler());
                                }
                            });

                    // 1.3 启动服务器
                    ChannelFuture f = b.bind(PORT).sync();
                    System.out.println("----Server Started----");

                    // 1.4 同步等待服务socket关闭
                    f.channel().closeFuture().sync();
                }catch(Exception e) {
                    System.out.println("----Server error----" + e.getLocalizedMessage());
                }
                finally {
                    // 1.5优雅关闭线程池组
                    bossGroup.shutdownGracefully();
                    workerGroup.shutdownGracefully();
                }               
            }
        });

        thread.start();

    }
}
```

如上代码为 Netty Server 的正规启动方式，默认监听服务端口为 8080，这里我们只需要关注在 1.2.5.1 添加了一个文件解码器和在 1.2.5.2 添加了一个业务处理器。

首先我们来看下文件解码器 FileDecoder 如何根据协议解析出一个文件的：

```Java
public class FileDecoder extends ByteToMessageDecoder {

    private static final int fileLength = 4;
    private static final int fileNameLength = 128;

    @Override
    protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        Object decoded = decode(ctx, in);
        if (decoded != null) {
            out.add(decoded);
        }
    }

    protected Object decode(@SuppressWarnings("UnusedParameters") ChannelHandlerContext ctx, ByteBuf in)
            throws Exception {
        //2.1半包
        if (in.readableBytes() < fileLength + fileNameLength) {
            return null;
        }
        //2.2读取文件大小
        in.markReaderIndex();
        int length = in.readInt();   
        //2.3发现半包
        System.out.println(in.readableBytes() + ":" + length );
        if (in.readableBytes() <length + fileNameLength) {

            in.resetReaderIndex();

            return null;
        }

        //2.4 读取 文件名+文件内容
        return in.readRetainedSlice(length + fileNameLength);
    }
```

- 如上代码 2.1，判断缓冲区可读数据大小是否小于 4+128，如果是，则说明出现了半包，则直接返回
- 如上代码 2.2，首先记录缓冲区读 index 位置，然后读取 4 个字节内容（文件长度）
- 如上代码 2.3，如果当前缓冲区可读字节小于文件长度+文件名长度，则说明出现了半包，则在返回前先重置缓冲区读 index 位置，然后返回
- 如上代码 2.4，走到这里说明缓冲区里面已经有了完整的文件，则直接返回协议里面文件名 + 文件内容到业务 handler。

下面我们来看业务 handler NettyServerHandler 的处理逻辑：

```java
public class NettyServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf file = (ByteBuf) msg;

        //3.1 读取文件名
        byte[] fileNameBuf = new byte[128];
        file.readBytes(fileNameBuf);

        String fileName = null;
        try {
            fileName = new String(fileNameBuf,"utf-8");
        } catch (UnsupportedEncodingException e1) {
            // TODO Auto-generated catch block
            e1.printStackTrace();
        }

        System.out.println("filename:" + fileName);

        //3.2保存字节流文件到磁盘
        ByteBuffer buffer = file.nioBuffer();

        FileOutputStream targetFileOutputStream = null;
        FileChannel targetFileChannel = null;
        try {
            try {
                targetFileOutputStream = new FileOutputStream(new File(fileName), false);
            } catch (FileNotFoundException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
            targetFileChannel = targetFileOutputStream.getChannel();
            targetFileChannel.write(buffer);
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        } finally {
            if (null != targetFileOutputStream) {
                try {
                    targetFileOutputStream.close();
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
            if (null != targetFileChannel) {
                try {
                    targetFileChannel.close();
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
            if (null != file) {
                file.release();
            }
        }

        ByteBuf message = null;
        String result = fileName + " save ok";
        message = Unpooled.buffer(result.length());
        message.writeBytes(result.getBytes());

        ctx.writeAndFlush(message);//.addListener(ChannelFutureListener.CLOSE);
        System.out.println("save file ok");

    }
}
```

- 如上代码 3.1，首先从文件解码器传递过来的字节流中读取 128 个字节，然后转换为 String 类型，作为文件名。
- 如上代码 3.2，然后从文件解码器传递过来的字节流剩余的部分保存到磁盘文件。

注意：本节我们搭建的文件服务器，只适合传输小的文件，如果传输文件过大并且同时上传并发量很大，由于本节方法是需要把整个文件缓存到内存的，所以可能会造成 OOM，所以本文方法不可在生产实践用运用，本文目的只是让大家通过实践来运用 Netty，从而加深理解。读者可以自己探索下如何不用内存缓存整个文件，而是接受一个部分就写入文件，后面的追加到文件（提示：文件解码器稍加改造...）。另外读者也可以考虑下如何实现断点续传的功能。

### 四、 Netty 搭建文件上传客户端

代码实例可以在：https://github.com/zhailuxu/netty-upload-client 下载。

首先我们来看看 Netty 客户端启动代码：

```Java
public final class NettyClient {

    static final String HOST = System.getProperty("host", "127.0.0.1");
    static final int PORT = Integer.parseInt(System.getProperty("port", "8080"));

    public static void main(String[] args) throws Exception {

        // 1.1 创建Reactor线程池
        EventLoopGroup group = new NioEventLoopGroup();
        try {// 1.2 创建启动类Bootstrap实例，用来设置客户端相关参数
            Bootstrap b = new Bootstrap();
            b.option(ChannelOption.ALLOCATOR, UnpooledByteBufAllocator.DEFAULT);

            b.group(group)// 1.2.1设置线程池
                    .channel(NioSocketChannel.class)// 1.2.2指定用于创建客户端NIO通道的Class对象
                    .option(ChannelOption.TCP_NODELAY, true)// 1.2.3设置客户端套接字参数
                    .handler(new ChannelInitializer<SocketChannel>() {// 1.2.4设置用户自定义handler
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline p = ch.pipeline();

                            p.addLast(new StringDecoder());
                            p.addLast(new NettyClientHandler());

                        }
                    });

            // 1.3启动链接
            ChannelFuture f = b.connect(HOST, PORT).sync();

            // 1.4 同步等待链接断开
            f.channel().closeFuture().sync();
        } finally {
            // 1.5优雅关闭线程池
            group.shutdownGracefully();
        }
    }
}
```

如上为正规的客户端启动流程，链接的服务器默认端口是 8080，这里我们这里我们关注自定义 handler NettyClientHandler，因为上传文件就是这里进行的：

```Java
public class NettyClientHandler extends ChannelInboundHandlerAdapter {

    private final byte[] request;

    private AtomicInteger atomicInteger = new AtomicInteger(0);

    /**
     * 创建一个客户端 handler.
     */
    public NettyClientHandler() {
        request = "hello server,im a client".getBytes();
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("--- client already disconnected----");

        ctx.fireChannelInactive();
    }

    public void sendFile(ChannelHandlerContext ctx ,String fileName,boolean closeConnect) {
        FileInputStream fileInputStream = null;
        int count = 0;

        try {
            //3.1 从磁盘加载文件
            File file = new File("/Users/zhuizhumengxiang/Downloads/ServerBootstrap.jpg");
            fileInputStream = new FileInputStream(file);

            byte[] buf = new byte[1024];
            ByteBuf message = null;

            //3.2  文件长度
            System.out.println("send file size:" + fileInputStream.available() + "," + file.length());
            message = Unpooled.buffer(4);
            message.writeInt((int) file.length());
            ctx.write(message);

            //3.3 文件名
            if(fileName.length()<128) {
                int appendNum = 128 - fileName.length();
                for(int i =0;i<appendNum;++i) {
                    fileName += " ";
                }
            }
            //3.4写入文件名
            System.out.println("send file size:" + fileName.length() + "," + file.length());
            message = Unpooled.buffer(128);
            message.writeBytes(fileName.getBytes("utf-8"));
            ctx.write(message);

            //3.5写入文件内容
            int len = -1;
            while (-1 != (len = fileInputStream.read(buf))) {
                System.out.println("--- client send file ----" + len);

                count += len;
                message = Unpooled.wrappedBuffer(buf, 0, len);
                ctx.writeAndFlush(message);
            }

        } catch (FileNotFoundException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        } finally {
            if (null != fileInputStream) {
                try {
                    fileInputStream.close();
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
        }
        System.out.println("--- client send file over----" + count);
        if(closeConnect) {
            ctx.channel().close();
        }
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) {

        System.out.println("--- client already connected----");

        sendFile(ctx, "hello1.jpg",false);
        sendFile(ctx, "hello22.jpg",true);

    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {

        System.out.println(atomicInteger.getAndIncrement() + "receive from server:" + msg);
    }
```

- 如上代码我们主要看 sendFile 方法，该方法内进行的文件上传。
- 首先代码 3.1 从磁盘加载要上传的文件；然后代码 3.2 获取文件大小，然后把文件大小以四个字节的形式写入到了 socket 流；然后代码 3.3 拼接文件名为 128 字节，不足则使用空格补齐。
- 代码 3.4 写入文件到 socket，代码 3.5 写入文件流到 socket。
- 可知向 socket 写入数据的格式是按照我们制定的协议来做的：首先写入 4 个字节的文件长度，然后写入 128 个字节的文件名，然后写入文件内容。

注：本节我们使用 Netty 演示了客户端如何上传文件，其实这里完全可以不用 Netty，或者也不一定要用 Java，你用 C++ 或者其他语言的 socket 通道进行上传也是可以的，当然前提是上传时候要遵守我们制定的文件协议格式。

### 五、参考

- Netty 权威指南
- [Java 网络编程基础篇](https://gitbook.cn/gitchat/activity/5ba79876ac886912c8557b18)
- [Java NIO 框架 Netty 之美：基础篇之一](https://gitbook.cn/gitchat/activity/5b01714ca0810c23901c55ac)
- [Java NIO 框架 Netty 之美：粘包与半包问题](https://gitbook.cn/gitchat/activity/5b13e6a675742e21d6d14ea4)
- [Java NIO 框架 Netty 之美：源码分析之一](https://gitbook.cn/gitchat/activity/5bb38acdc7902c10b7cf2650)

### 六、新书推荐

- [《Java 并发编程之美》已经上市](https://item.jd.com/12450812.html)