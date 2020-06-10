# 14/31使用 Netty 完成一个轻量级 HTTP 文件下载器

### 引言

在网络开发中，不可避免地会与许多标准通讯协议打交道。在这方面，Netty 为开发者提供了很大的便利。Netty 自身除了提供编解码机制外，还提供了大量现成协议的编解码支持，这部分支持的内容可以在包 netty-codec 中找到。当然也包括我们今天要讲述，对 HTTP 的支持。

有了对 HTTP 编解码协议的支持，我们完全可以使用 Netty 来开发一款 Web 容器或者符合需求的 HTTP 容器。今天，我们就以文件的下载作为场景，介绍下 Netty 对 HTTP 的支持。本文中使用的相关代码存储于：

> https://gitee.com/eric_ds/learnNetty/tree/master/http

### HTTP 协议

在使用 Netty 完成 HTTP 服务器功能之前，首先需要对 HTTP 协议有个相对全面的了解。HTTP 协议是工作在应用层的一个文本协议，其工作模型是请求-响应模型。客户端发出 HTTP 请求，服务端回复 HTTP 响应，一来一回，一次完整的 HTTP 交互才结束。

这里我们将基本介绍下 HTTP 协议中对传输内容的格式规范要求，方便于后文对协议解析部分的代码理解。完整的 HTTP 协议中包含的内容十分的多，超出了本文的范畴，这里不再一一阐述。感兴趣的读者可以在 RFC2616 文档进行查阅。

HTTP 协议对传输的报文格式定义如下：

![img](https://images.gitbook.cn/21479860-99d7-11ea-8c90-bb96514abba4)

主要是分为四个部分：起始行、零个或者多个头域、标记头域结束的回车换行，以及可能存在的内容体构成。其中起始行有两种情况，分别是：请求行和状态行。

#### **起始行**

**1. 请求行**

请求行出现在客户端发出的 HTTP 请求中，表明这是一个 HTTP 请求。请求行的格式如下：

![img](https://images.gitbook.cn/5cc871c0-99d7-11ea-85b2-5731fc7f7bbd)

HTTP 方法就是我们常提到的 HTTP 动词，早期常见的有 GET、POST 两种，RFC 文档中也要求实现 HEAD 等动词，至于 PUT、DELETE、PATCH 服务器可以根据需要提供实现或者不实现。

请求 URL 就是我们常提到的 URL 地址。通过 HTTP 协议，HTTP 方案的 URL 格式定义如下：

```
http_URL = "http:" "//" host [ ":" port ] [ abs_path [ "?" query ]]
```

当访问的端口是 80 时，`:port` 可以直接省略。`abs_path` 用于对所需资源进行定位。`"?" query` 不是必须的。其目的在于传递查询参数。

HTTP 版本号用于标识当前客户端要求本次通讯采用的 HTTP 协议，目前常见的有 `HTTP/1.0` 或 `HTTP/1.1`。1.1 和 1.0 的主要区别在于，1.1 允许一个 TCP 连接上发送和接收多个 HTTP 请求响应，而 1.0 当中一次 HTTP 交互完成后就会断开 TCP 连接，对于如今比较复杂和内容丰富的页面，显然 1.0 的这种处理方式是比较低效的，因此大部分的客户端和服务端对 1.1 版本早就提供了支持。

请求行的最后以连续的换行和回车作为结束标记。

**2. 状态行**

状态行出现在服务端发出的 HTTP 响应中，用于表明这是一个 HTTP 响应，状态行的格式如下：

![img](https://images.gitbook.cn/b9863460-99d7-11ea-b09b-519edd322eee)

HTTP 版本号用于指示该响应内容使用的版本号标识。

状态码用于指示该响应的结果情况。状态码为三位整数，第一位用于表明响应的类型，后两位无定义。第一位数字有 5 种定义：

- 1XX：用于报告，表示接受到请求，继续处理流程。
- 2XX：用于成功响应。表示接受到请求，并且被正确的处理。
- 3XX：重发，为了能够正确处理，需要重新发送请求。
- 4XX：客户端出错，请求内容包含错误讯息。
- 5XX：服务端出错，服务端无法理解或者处理客户端的请求。

在协议文档中定义了一些常见的状态码，比如 200 代表成功，404 代表不存在等等。状态码可以自行定义，只需要避开已经公共存在的状态码定义即可。原因短语是对状态码的文本解释，通常情况下无需关注。因为对状态码的识别已经足够进行条件控制了。

响应码的最后也是通过连续的回车和换行结束。

#### **头域**

HTTP 协议中定义的头域内容非常多，大体而言头域可以分为三种：通用头域、请求/响应头域、实体头域。

请求头域用于客户端向服务端发送本次请求的一些附加信息。虽然说是附加信息，但是这些信息中往往包含了很重要的数据内容。比如有：

- host：请求的主机位置
- Accept-Charset：客户端能够接受的字符集

响应头域用于服务端向客户端发送本次响应的一些附加信息，一般是对响应的内容起到元数据说明的作用。

通用头域则是一些通用的附加元数据信息描述，实体头域则是定义了所传输实体的一些附加信息。比较常见的有：

- Content-Length：传输内容体长度
- Last-Modified：资源的上一次修改时间

#### **内容体**

内容体则负责承载请求或者响应中需要传输的实体信息。常见的比如向服务端提交一个表单，内容体则承载了表单信息；如果是点开一个网址，服务端响应信息中，内容体就是一段 HTML 文本。其具体的信息较多，这里不展开，感兴趣的读者可以查阅 RFC2616 文档。

### 需求澄清

用 Netty 实现 HTTP 文件下载，大体上分为两个步骤：

1. 浏览器访问特定资源 URL，获取当前可下载资源列表
2. 浏览器访问特定资源 URL，下载资源文件

对于需求 1 而言，我们将磁盘上的某个文件夹映射为网络资源地址，只需要将对应的资源操作，转化为对磁盘文件夹的展示即可。

对于需求 2 而言，资源 URL 已经直接定位到了具体的文件，则通过读取磁盘上的数据内容，使用 HTTP 协议将数据通过连接发送回客户端。

### 第一个版本

Netty 内建提供了对 HTTP 协议的支持，其具体的支持包路径在 io.netty.handler.codec.http.* 下。Netty 将 HTTP 协议中的各个部分抽象为 HttpObject 接口，来看下类图，如下

![img](https://images.gitbook.cn/1ffe8120-99d8-11ea-9d43-4b614dacd168)

HttpObject 是一个标识接口，本身没有提供方法，只是表明了其子接口的类别归属。所有的 HTTP 对象接口都继承了 HttpObject。

HttpRequest 用于存储 HTTP 协议中，请求行和头域的内容。如果客户端的请求中带有内容体，则这部分数据存储在 HttpContent 对象中。与之同理，HttpResponse 存储着状态行和头域的相关内容，响应的内容体存储在 HttpContent 之中。

对于 HTTP 请求的解析，Netty 提供了解码类 HttpRequestDecoder。该解码器可以将读取到的 ByteBuf 转换为 HttpRequest 对象和 HttpContent 对象。解码器是边解码，边向后传递解码到的内容。这种设计模式有助于减少可能的堆积内存数量。比如如果大文件上传的场景，在一个解码流程中完整解码内容体，势必要在内存中存储所有收到的数据（Netty 内建的 ByteToMessageDecoder 采用此种实现），这可能导致程序消耗巨大的内存。边解码，边传递解码对象，则可以让开发者更容易处理此类场景，避免在完整解码前的巨大内存堆积。

如果请求消息带有消息体，HttpRequestDecoder 会将整个请求解码为三种对象：

- 包含请求行和头域内容的 HttpRequest
- 部分解码中的部分内容体的 HttpContent
- 代表内容体解码完毕的 LastHttpContent

表单提交这种场景不消耗多少内存，但是也会产生 HttpContent。如果开发者不想处理 HttpRequest 和 HttpContent 分隔的这种情况，可以使用聚合处理器 HttpObjectAggregator。该处理器可以将管道中前向的解码器产生的 HttpObject 聚合在一起，并且在收到 LastHttpContent 对象后，将已经聚合的 HttpObject 聚合成为一个完整的 FullHttpRequest 对象，并且向管道后面的处理器进行传递。那么此时开发者就只需要处理一个已经完全解码完毕的 HTTP 请求。

开发者要针对 HTTP 请求发出响应，需要构建出 HttpResponse 和 HttpContent 分别用于存储状态行、头域和内容体相关数据，并且最终编码为 HTTP 协议数据。Netty 针对 HTTP 编码，也为开发者提供了现成的编码器即 HttpResponseEncoder。有了这些，就可以完成第一个版本，用于实现“返回资源列表”这个功能。主代码可以撰写如下：

```
public class HttpServer
{
    public void start() throws InterruptedException
    {
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(new NioEventLoopGroup(1), new NioEventLoopGroup());
        bootstrap.channel(NioServerSocketChannel.class);
        bootstrap.childHandler(new ChannelInitializer<SocketChannel>()
        {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception
            {
                ChannelPipeline pipeline = ch.pipeline();
                pipeline.addLast(new HttpRequestDecoder());
                pipeline.addLast(new HttpResponseEncoder());
                pipeline.addLast(new HttpObjectAggregator(1024));
                pipeline.addLast(new ListFileHandler(new File("z:/")));
            }
        });
        ChannelFuture bind = bootstrap.bind(80);
        bind.sync();
        bind.channel().closeFuture().sync();
    }

    public static void main(String[] args) throws InterruptedException
    {
        new HttpServer().start();
    }
}
```

这是服务端程序，所以对于连接管道上的处理器，首先是 HTTP 请求解码器，而后是 HTTP 响应编码器，再次是用于聚合 HTTP 对象的聚合处理器，最后则是解析请求内容，处理业务数据，返回影响的资源展示处理器，即 ListFileHandler。其代码如下：

```
public class ListFileHandler extends ChannelInboundHandlerAdapter
{
    private              File         dir;
    private              ObjectMapper mapper  = new ObjectMapper();
    private static final Charset      CHARSET = Charset.forName("utf8");

    public ListFileHandler(File dir) //使用了一个外部定义的文件夹对象，该文件夹下存储着所有可以展示的资源文件
    {
        this.dir = dir;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
    {
        FullHttpRequest fullHttpRequest = (FullHttpRequest) msg;
        String          uri             = fullHttpRequest.uri();
        uri = URLDecoder.decode(uri, "utf8");//对请求地址进行 url 解码，解决请求路径中出现的中文转码问题
        if (uri.equals("/"))//访问资源根路径，则将资源目录下的内容获取后，以 json 格式返回
        {
            String[] list   = dir.list();
            String   s      = mapper.writeValueAsString(list);
            ByteBuf  buffer = ctx.alloc().buffer();
            buffer.writeBytes(s.getBytes(CHARSET));
            DefaultFullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK, buffer);
            response.headers().add(HttpHeaderNames.CONTENT_TYPE, HttpHeaderValues.APPLICATION_JSON + ";" + HttpHeaderValues.CHARSET + "=utf8");
            response.headers().add(HttpHeaderNames.CONTENT_LENGTH, buffer.readableBytes());
            ctx.writeAndFlush(response);
            System.out.println("文件夹内容输出完毕");
        }
        else
        {
            ctx.fireChannelRead(msg);
        }
    }
}
```

从 ListFileHandler 中我们可以看到，构建返回响应，可以直接将内容体的 ByteBuf 生成后，以 `DefaultFullHttpResponse(HttpVersion, HttpResponseStatus, ByteBuf)` 构造方法创建响应对象，填充合适的头域数据后，通过 Channel 进行发送。

运行该程序，我们查看下用作测试的 Z 盘目录，内容如下：

![img](https://images.gitbook.cn/d3585a70-99d8-11ea-b902-bf483bc12b74)

访问资源的根路径就可以进行资源查看，因此我们在浏览器地址栏输入 localhost，显示效果如下：

![img](https://images.gitbook.cn/e79a3490-99d8-11ea-a84f-f7d3d4dae1cc)

第一个需求实现完毕。接下来是第二个需求，对文件的下载。按照资源列表的展示结果，我们在地址栏输入 `localhost/ideaIU-2019.3.exe` 就可以开启下载窗口进行下载。为此，我们需要实现 DownLoadHandler，用于处理资源下载请求。显然我们需要构建一个 HttpResponse 对象，用于表明当前的资源返回是一个二进制数据流；并且构建足够的 HttpContent 用于容纳需要传递的文件内容对象。按照这个思路，我们可以将 DownLoadHandler 实现如下：

```
public class DownLoadHandler extends ChannelInboundHandlerAdapter
{
    private File dir;

    public DownLoadHandler(File dir)
    {
        this.dir = dir;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
    {
        FullHttpRequest fullHttpRequest = (FullHttpRequest) msg;
        String          uri             = fullHttpRequest.uri();
        uri = URLDecoder.decode(uri, "utf8");
        File dest = dir;
        for (final String each : uri.split("/"))
        {
            File[] files = dest.listFiles(new FileFilter()
            {
                @Override
                public boolean accept(File pathname)
                {
                    return pathname.getName().equals(each);
                }
            });
            if (files.length == 1)
            {
                dest = files[0];
            }
        }
        FileInputStream inputStream = new FileInputStream(dest);
        ByteBuf         buffer      = ctx.alloc().buffer();
        buffer.writeBytes(inputStream, inputStream.available());
        inputStream.close();
        HttpResponse response = new DefaultHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK);
        response.headers().add(HttpHeaderNames.CONTENT_TYPE, HttpHeaderValues.APPLICATION_OCTET_STREAM);
        response.headers().add(HttpHeaderNames.CONTENT_LENGTH,inputStream.available());
        ctx.write(response);
        HttpContent chunk = new DefaultHttpContent(buffer);
        ctx.channel().write(chunk);
        ctx.channel().writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);
    }
}
```

该 ChannelHandler 被添加在 FileListHandler 之后。

在正确解析资源地址后，首先构建一个 HttpResponse 对象，设置好二进制数据传输的头域，将该对象写入通道中。然后通过 FileInputStream，将资源内容读取到一个 ByteBuf 中，将这个 ByteBuf 包装为一个 HttpContent 对象再次写入通道中。最后，向通道发送一个 EMPTY_LAST_CONTENT 表明 HTTP 内容体已经结束。HttpResponseEncoder 会识别到这个特定的对象，将编码状态设置为初始状态，并且在有需要的时候编码内容体结束符发送给客户端以完整结束整个 HTTP 响应。不过在 Content-Length 头域起作用的场合，就不存在额外的编码结束符内容了。

下面，我们在地址上输入 `localhost/ideaIU-2019.3.exe`，观察到浏览器首先打开了一个下载询问窗口，如下所示：

![img](https://images.gitbook.cn/33e7be80-99d9-11ea-af90-cbb5738a0aa8)

点击下载后，文件下载开始，并且在本地很快就下载完毕。

### 第二个版本

第一个版本虽然实现了功能，但是在实现的细节上却有待商榷。因为第一个版本的实现中，将本地的资源数据一次性全部读取到 ByteBuf 中，这样会造成很大的内存占用开销。如果下载并发一高，都可能导致 OOM 的发生。显然，我们可以通过将资源内容的读取改造为分批的方式，来降低内存的占用。也就是说，在传输完一部分数据后，再读取文件的下一段内容到 ByteBuf 中，在写入到通道中发送。按照这个思路，我们将 DownloadHandler 改写如下：

```
public class DownLoadHandler2 extends ChannelInboundHandlerAdapter
{
    private       File dir;
    private final int  maxContentLength = 1024 * 1024;

    public DownLoadHandler2(File dir)
    {
        this.dir = dir;
    }

    @Override
    public void channelRead(final ChannelHandlerContext ctx, Object msg) throws Exception
    {
        /*省略相同的代码部分*/
        ByteBuf               buffer      = ctx.alloc().buffer();
        int                   need        = Math.min(maxContentLength, inputStream.available());
        buffer.writeBytes(inputStream, need);
        ctx.channel().writeAndFlush(buffer).addListener(new GenericFutureListener<Future<? super Void>>()
        {
            @Override
            public void operationComplete(Future<? super Void> future) throws Exception
            {
                int need = Math.min(maxContentLength, inputStream.available());
                if (need != 0)
                {
                    ByteBuf buffer = ctx.alloc().buffer();
                    buffer.writeBytes(inputStream, need);
                    Thread.sleep(1000);
                    ctx.channel().writeAndFlush(buffer).addListener(this);
                }
                else
                {
                    inputStream.close();
                    ctx.channel().writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);
                }
            }
        });
    }
}
```

可以看到，我们通过 ChannelFuture 提供的通知功能，当一段内容的 ByteBuf 写入到通道后，继续读取下一个部分的内容，然后再写入。持续这个流程直到数据全部写入完毕。修改后的代码，我们在浏览器上输入资源信息，首先仍然是下载确认框，点击确认后，文件却不是一瞬间下载完毕的。而是如下图所示：

![img](https://images.gitbook.cn/5bdb67c0-99d9-11ea-85b2-5731fc7f7bbd)

我们人为地制造了一秒的停顿，因此在客户端的下载速度上，就被限制为上限 1M/S。经过一段时间的下载，进度向前前进了一部分，如下所示：

![img](https://images.gitbook.cn/702ea700-99d9-11ea-95eb-7d99e394b1c2)

可以看到这种分段的策略确实起到了效果。

针对这种大量数据的传输情况，有时候无法提前知道资源的内容大小，也就是说无法在头域定义 Content-Length 内容，HTTP 定义了一种头域用于分块传输数据，即 `Transfer-Encoding:Chunked`。这种情况下，客户端会不断的累积 chunk 中的内容，并且在最终收到 Chunk 结束标识符后理解到内容体传输完毕。而这种协议的变换，在 Netty 中，已经通过 HttpResponseEncoder 封装完毕，在业务处理代码中不需其他修改，只需要改动一行头域内容即可，如下：

```
response.headers().add(HttpHeaderNames.TRANSFER_ENCODING, HttpHeaderValues.CHUNKED);
```

HttpResponseEncoder 会自动的发送符合协议要求的 Chunk 编码数据，并且在收到上游发送的 EMPTY_LAST_CONTENT 对象时，编码输出 Chunk 结束符到连接中，用于高速客户端当前内容体已经传输完毕。

### 第三个版本

第二个版本已经能够降低传输文件所需要的内存开销，但是在效率上仍然显得差一些。因为其传输的本质是首先在 CPU 的帮助下将数据从磁盘读取到内核态缓存区中，再复制到用户态内存中（没有使用堆外内存的情况），而后写入 Socket 通道，通 Socket 通道传输数据又需要从用户态数据复制到内核态缓存区，而后才能通过 TCP 连接发出。操作系统为了效率，提供了零拷贝的能力，也就是直接从磁盘到 TCP 通道之间的数据传输，不再需要用户态内存的中转。Java 也在 API 层面提供了支持，即方法 `java.nio.channels.FileChannel#transferTo`。

这个方法可以将文件通道和一个可写通道连接起来进行直接的数据传输而不需要经过用户态内存的中转。效率是最高的。Netty 也提供了类似机制的支持，也就是 FileRegion 接口。该对象用于定义对一个文件的一段区域的读取，其方法 `io.netty.channel.FileRegion#transferTo` 一样也提供了将数据传输到另外一个可写通道的能力。

Netty 内建的写出处理器能够识别 FileRegion，提供更为高效的数据传输能力。

我们在 DownLoadHandler 中引入 FileRegion，就可以避免数据在磁盘，用户态内存，内核态缓冲之间的中转。代码修改如下：

```
public class DownLoadHandler2 extends ChannelInboundHandlerAdapter
{
    private       File dir;
    private final int  maxContentLength = 1024 * 1024;

    public DownLoadHandler2(File dir)
    {
        this.dir = dir;
    }

    @Override
    public void channelRead(final ChannelHandlerContext ctx, Object msg) throws Exception
    {
        /*省略相同的代码部分*/
        FileRegion fileRegion = new DefaultFileRegion(dest,0,fileSize);
        ctx.channel().writeAndFlush(fileRegion);
        ctx.channel().writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);
    }
}
```

可以看到，代码变得更加简洁，只需要构造一个 FileRegion 对象，不需要考虑分段传输等繁琐内容，Netty 内部会自动帮我们协调完成。而且由于是零拷贝直接传输，不仅降低了内存占用，更提升了吞吐量。可以说是三个方案中性能最优秀的一个。

### 思考与总结

本篇我们我们通过 Netty 构建了一个文件下载服务器，给读者阐述了 Netty 中对 HTTP 协议的多种支持情况。

从例子中可以感受到，使用 Netty 构建一个 HTTP 服务器是很方便和简单的，因为 Netty 将其中最麻烦的编解码部分已经提供了完全的支持。

开发者只需要在 Netty 提供的 HttpObject 即相关接口上进行业务代码开发即可。并且针对于文件传输这种场景，Netty 也支持了零拷贝的传输功能，使得在这种场景下的性能表现更加优秀。

Netty 中提供了大量的流行协议的编解码支持，并且还在不断的增加中。通过这些内建的编解码支持，开发者可以很容易的在其上构建业务逻辑。整个实战篇我们可以看到，使用 Netty 开发业务逻辑，是非常容易的。

而网络 IO 的很多细节，性能方面的考虑 Netty 都已经帮我们做到很好的支持程度了。但是仅仅只是掌握了 Netty 的使用方式，对其内部如果不了解，在出现问题的时候，则很难定位和解决。从下篇文章开始，我们将进入专栏的第三部分，源码阶段篇。

在这个篇章我们，我们将分析 Netty 中的重要组件和重点流程，使得开发者能够知其然并知其所以然。