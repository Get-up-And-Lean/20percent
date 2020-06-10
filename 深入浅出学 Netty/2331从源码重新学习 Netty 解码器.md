# 23/31从源码重新学习 Netty 解码器

## 引言

在入门篇的学习中，我们曾经阐述过工作于 TCP 协议的应用出现的粘包和拆包现象。当时我们介绍了 Netty 用于解决拆包粘包问题的三种常见的解码器实现：

- 处理定长协议的 `FixedLengthFrameDecoder`。
- 处理固定消息分隔符的 `DelimiterBasedFrameDecoder`。
- 处理报文头 + 报文体模式协议的 `LengthFieldBasedFrameDecoder`。

今天这篇文章，我们不再介绍更多编解码器的使用，而是从这些解码器的背后实现入手，弄明白他们的处理机制，以及其在 Netty 整个读写流程中所处的地位。

## 解码

将 socket 上读取到的二进制数据转换为后端业务可以理解的消息实体，这个过程我们称之为解码，执行这个任务的组件，我们称之为解码器。而在 Netty 当中，有非常多内建的解码支持，涵盖了各种市面上的协议。每一种协议的解码需求，都有一个特定的解码器来承担，解码器都位于 `io.netty.handler.codec` 包路径下。而所有的解码器都继承了 `ByteToMessageDecoder`，该类是解码器的顶级父类，定义了这个解码过程中的模板动作。首先来看下该类的类视图，如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191222214311.png)

### 重写的 channelRead

从继承关系再配合该类的定义，可以猜到其重写了 `channelRead` 方法用于完成解码任务。来看其 `channelRead` 方法的实现，如下

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception

{if(msg instanceof ByteBuf)

    {

        // 代码①

        CodecOutputList out = CodecOutputList.newInstance();

        try

        {ByteBuf data = (ByteBuf) msg;

            first = cumulation == null;

            // 代码②

            if(first)

            {cumulation = data;}

            else

            {

                // 代码③

                cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);}

            // 代码④

            callDecode(ctx, cumulation, out);

        }

        catch(DecoderException e)

        {throw e;}

        catch(Exception e)

        {throw new DecoderException(e);

        }

        finally

        {

            // 代码⑤

            if(cumulation != null && !cumulation.isReadable())

            {

                numReads = 0;

                cumulation.release();

                cumulation = null;

            }

            // 代码⑥

            else if(++numReads >= discardAfterReads)

            {

                numReads = 0;

                discardSomeReadBytes();}

            int size = out.size();

            // 代码⑦

            firedChannelRead |= out.insertSinceRecycled();

            fireChannelRead(ctx, out, size);

            out.recycle();}

    }

    else

    {ctx.fireChannelRead(msg);

    }

}
```

首先来看 ** 代码①**。`CodecOutputList` 继承于 `ArrayList`，是一个轻量级的包装类，内部多添加了一些属性。其起到的作用就是存放解码后的消息对象，以及支持以线程变量的形式被缓存起来。所以方法 `CodecOutputList.newInstance()` 并不是新建了实例，而是从线程变量中获取了一个实例。在整个方法调用结束后，`finally` 代码块中，也会调用 `CodecOutputList#recycle` 方法回收至线程变量中。

接着来看 ** 代码②**。`ByteToMessageDecoder` 对于二进制数拆包粘包的处理思路是：首先尽可能的读取二进制数据，对于这些读取到的数据执行解码操作，解码生成的消息对象传递到后端的处理器中处理；而不能完整解码为一个消息对象的不完整报文的二进制数据，则驻留在 `ByteToMessageDecoder` 内部。Netty 的设计上，一个 `ChannelHandler` 只会运行在一个绑定的线程上，数据并没有竞争，不存在并发可能引发的数据安全问题。因此可以将不完整报文的二进制数据保留在自身的存储中，也就是 `cumulation`，其类型为 `ByteBuf`。而每次从通道中读取到新的数据，首先需要将这些数据与之前留存的 `cumulation` 进行合并，合并后才能进行解码处理。

接着来看 ** 代码③**。上面我们讲到，每次从通道中读取到数据，需要将读取到的数据合并到之前的累积部分，也就是合并到 `cumulation` 中。将新读取到的数据添加到之前的累积数据上，Netty 设计了接口 `ByteToMessageDecoder.Cumulator` 来完成这个功能，并且提供了两个内置实现：

- 将新读取到的 `ByteBuf` 的内容，添加到 `cumulation` 属性代表的 `ByteBuf` 上，并且释放新的 `ByteBuf` 对象。
- `cumulation` 属性代表的是一个 `CompositeByteBuf`，新读取到的 `ByteBuf` 的对象直接进入到 `CompositeByteBuf` 的内容体中，没有数据拷贝的过程。

虽然方案二在添加数据的时候没有内存复制，但是 `CompositeByteBuf` 实现很复杂，对于后面的解码处理可能反而带来更大的性能损失，因此 Netty 默认采用的是第一种方案。

接着来看 ** 代码④**。该方法内部会调用 `ByteToMessageDecoder#decode` 方法来将 `cumulation` 的内容解码成特定的消息对象。这个方法是一个抽象方法，交给具体的子类去实现。而 `ByteToMessageDecoder` 只是完成对一个整体流程的控制。比如 `decode` 方法解码后会将解码的结果对象放入到 `CodecOutputList` 对象，而 `callDecode` 本身则会将解码完毕后，通过 `ChannelhandlerContext.fireChannelRead` 来将消息向后方的 handler 处理。

接着来看 ** 代码⑤** 和 ** 代码⑥**。其实 ** 代码⑤** 和 ** 代码⑥** 关注的点都是相同的，就是在完成解码后，对于剩余不完整的部分如何处理的问题。如果不存在剩余的二进制数据，也就是 ** 代码⑤** 的情况，将 `cumulation` 代表的实例释放后即可；如果是 ** 代码⑥** 的情况，`cumulation` 还剩余一部分不完整报文的数据且读取次数已经超过阀值，此时可以根据情况，对 `ByteBuf` 实例进行压缩，通过方法 `ByteBuf#discardSomeReadBytes` 完成。这个方法的效果类似于 `java.nio.ByteBuffer#compact`，都是将真正的可读数据区域移动到空间的起始处，避免 writeIndex 无谓增大，导致扩容发生。不同之处在于 `discardSomeReadBytes` 的实现上，会考虑 `ByteBuf` 内部的数据情况，决定是否压缩，以及压缩多少。

### 重写的 channelReadComplete

当数据完毕后，通道上触发 `channelReadComplete` 方法，`ByteToMessageHandler` 对该事件的处理重写如下

```java
public void channelReadComplete(ChannelHandlerContext ctx) throws Exception

{

     numReads = 0;

     discardSomeReadBytes();

     if(!firedChannelRead && !ctx.channel().config().isAutoRead())

     {ctx.read();

     }

     firedChannelRead = false;

     ctx.fireChannelReadComplete();}
```

重点来关注 `firedChannelRead` 属性，在 `channelRead` 方法中，如果有解码出消息实体，该属性被设置为 `true`。显然，如果没有消息被解码出来，并且通道本身并不是自动注册读取，意味着本次的读取并没有能真正读取到一个有效的消息，此时就需要继续发起一个读取请求，也就是 `ctx.read`。

### 对拆包粘包的解决

从 `ByteToMessageHandler` 的实现来看，其主要处理的是二进制数据的累积问题，但是从模板方法的分析来看，其并不涉及到具体的拆包和粘包的问题解决上。实际上也是如此，粘包拆包的解决，必然涉及到二进制数据的解码，而 `ByteToMessageHandler` 并不执行实际的解码，解码是留给子类去实现。我们以 `LengthFieldBasedFrameDecoder` 来分析其实现的方式，其 `decode` 抽象方法实现如下

```java
protected final void decode(ChannelHandlerContext ctx, ByteBuf in , List < Object > out) throws Exception

{Object decoded = decode(ctx, in);

     if(decoded != null)

     {out.add(decoded);

     }

}
```

其委托方法 `LengthFieldBasedFrameDecoder#decode(ChannelHandlerContext, ByteBuf)` 解码二进制数据，并且将解码得到的消息实体添加到 `List` 对象中，也就是 `ByteToMessageHandler` 的 `CodecOutputList`。

在 `ByteToMessageHandler` 的 `callDecode` 方法中会通过循环反复调用 `decode` 方法，直到当前 `cumulation` 的数据无法解码出新的消息实体。

### 综述

`ByteToMessageHandler` 是所有解码器的父类，其主要的职责是解决通道上二进制数据累积消费的问题。通过将二进制数据累积在自身内部，降低了其他 `ChannelHandler` 处理的难度。子类只要专心实现一个消息本身的解码功能即可。

## 编码

与解码相对应的就是编码了。其职责是完成从消息对象到二进制数据的转换，或者更确切一些的说，是完成消息对象到 `ByteBuf` 对象的转变。与解码相似，Netty 也提供了一个抽象父类来完成一些模板的功能，就是 `MessageToByteEncoder`。大部分的解码类都继承了这个抽象类，下面来看下其 `write` 方法的重写内容，如下

```java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception

{

    ByteBuf buf = null;

    try

    {

        // 代码①

        if(acceptOutboundMessage(msg))

        {I cast = (I) msg;

            buf = allocateBuffer(ctx, cast, preferDirect);

            try

            {

                // 代码②

                encode(ctx, cast, buf);

            }

            finally

            {ReferenceCountUtil.release(cast);

            }

            if(buf.isReadable())

            {ctx.write(buf, promise);

            }

            else

            {buf.release();

                ctx.write(Unpooled.EMPTY_BUFFER, promise);

            }

            buf = null;

        }

        else

        {ctx.write(msg, promise);

        }

    }

    catch(EncoderException e)

    {throw e;}

    catch(Throwable e)

    {throw new EncoderException(e);

    }

    finally

    {if(buf != null)

        {buf.release();

        }

    }

}
```

整个方法其实很简单，首先是 ** 代码①** 处通过方法 `acceptOutboundMessage` 判断当前传递的对象是否是本编码器可以处理的范围，依靠的就是 `TypeParameterMatcher` 的类型判断能力，这个类在之前的文章已经介绍过了。之后就是 ** 代码②** 调用方法 `encode` 完成消息对象到 `ByteBuf` 的转换。

转换成功后，如果存在可读内容，则继续调用 `write` 方法传递下去，直到到达 `pipeline` 的头结点，最终被写出到通道上。

## 总结与思考

Netty 对于通道上读取数据的累积处理放在了处理器上，因为一个 `ChannelHandler` 可以对应一个通道，不用处理并发和数据安全的一些问题；而如果在 `NioEventLoop` 实现则会相对复杂一些。对于编解码的掌握，只要通过 `ByteToMessageHandler` 和 `MessageToByteHandler` 两个抽象类即可。抽象父类完成了大部分需要关心的问题，其子类只需要实现最简单的解码和编码的内容。

到这篇文章为止，我们已经梳理了 Netty 的线程模型，异步任务设计机制，管道设计机制，读写流程，连接流程，以及一些精彩疑难点。相信读者经过这么长时间的学习，对 Netty 的内部机制已经相当的熟悉。不过 Netty 为我们带来远远不止这些。从 Netty3 升级到 Netty4 最大的变化就是带来了内存池。Netty 开始自行管理内存池，内存池功能的出现极大的减缓了在高并发高负载下应用程序的 GC 表现，提升了系统整体吞吐和稳定性。

接下来的文章中，我们将会逐一介绍 Netty 背后这些精彩的功能设计和数据结构。