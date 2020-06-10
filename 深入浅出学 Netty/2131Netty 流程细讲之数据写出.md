# 21/31Netty 流程细讲之数据写出

## 引言

在前文中我们介绍了连接就绪，读就绪，接受就绪三种事件流程的处理代码。这篇文章我们来介绍下最后一种流程，数据写出。当用户业务需要将数据写出时一般会调用 `channel.write` 或者是 `channel.writeAndFlush`。相关的 API 在入门篇的例子中也有过演示。本文，我们将来分析这两个 API 背后的工作原理。

## write 和 flush 的区别

在 Netty 的设计中，写出侧存在一个写出缓存。当我们调用 `channel.write` 或者 `channelHandlerContext.write` 时，其实只是将需要写出的内容放入到写出缓存的队列中。而只有调用 `channel.flush` 或者 `channelHandlerContext.flush` 才会让 Netty 将写出缓存中的数据真正写出到 Socket 的缓冲区从而通过 tcp 来发送。当然，Netty 也提供了方便的整合方法 `writeAndFlush` 用于将数据写出并且刷出到 socket 缓冲区。下面我们来分别分析下两个不同的方法。

## 数据写出 write

在 `NioSocketChannel` 的 `write` 方法实际上委托给 `pipeline` 来处理，是一个出站事件。经过管道中层层处理器对数据的转换，最终到达管道的首节点的 `write` 方法。而首节点的 write 方法再次将写出动作委托给 `unsafe.write`。在 `NioSocketChannel` 中，unsafe 实例是 `NioSocketChannelUnsafe`，其 `write` 方法继承自 `AbstractUnsafe`，其代码如下

```java
public final void write(Object msg, ChannelPromise promise)
{assertEventLoop();
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    // 代码①
    if(outboundBuffer == null)
    {safeSetFailure(promise, newClosedChannelException(initialCloseCause));
        ReferenceCountUtil.release(msg);
        return;
    }
    int size;
    try
    {
        // 代码②
        msg = filterOutboundMessage(msg);
        // 代码③
        size = pipeline.estimatorHandle().size(msg);
        if(size < 0)
        {size = 0;}
    }
    catch(Throwable t)
    {safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }
    // 代码④
    outboundBuffer.addMessage(msg, size, promise);
}
```

整个写出基本分为三个步骤：1）检查；2）检查与转换；3）放入写出缓存。

先看 ** 代码①**。首先确认写出缓存 `outboundBuffer` 是否为空。`outboundBuffer` 只有在通道关闭或者关闭写出方向连接时才会被设置为 null。因此后续的写入就直接返回失败。这里需要注意的是，如果放弃写出，需要通过方法 `ReferenceCountUtil.release` 来执行资源的释放，因为 msg 可能是一个 `ByteBuf` 或者其他可以释放的资源，不执行释放的话，会造成内存泄漏问题。

接着看 ** 代码②**。`filterOutboundMessage` 方法用于被子类重写，用于完成对 `msg` 对象的转换，以及检查要写出的对象是否是当前通道支持的类型。比如 `NioServerSocketChannel` 对该方法的重写就是直接抛出异常，因为 `NioServerSocketChannel` 只用于接受客户端接入请求，无法写出数据。对于 `NioSocketChannel` 而言，对这个方法的重写如下

```java
protected final Object filterOutboundMessage(Object msg)
{if(msg instanceof ByteBuf)
    {ByteBuf buf = (ByteBuf) msg;
        if(buf.isDirect())
        {return msg;}
        return newDirectBuffer(buf);
    }
    if(msg instanceof FileRegion)
    {return msg;}
    throw new UnsupportedOperationException("unsupported message type:" + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);}
```

如果 `ByteBuf` 不是堆外的，则尝试转换为堆外内存，也就是 `newDirectBuffer` 方法。如果这个方法是尽最大努力转换，如果当前使用的 `ByteBuf` 分配器没有使用内存池，而当前线程也没有可以使用的 `DirectByteBuf`，则不进行转换。因为直接分配一个堆外的 `ByteBuf` 消耗较大。

如果 `msg` 不是 `ByteBuf` 或者 `FileRegion` 则抛出不支持异常。

这边来细说下为何要将 `ByteBuf` 转化为堆外的模式。`ByteBuf` 本质上就是一个 `byte[]` 数组对象，如果一个堆内的 `byte[]` 要通过 `socket` 写出时，就需要先将数据拷贝到堆外，之后才能用堆外的数据执行写出。这是因为，通过 `socket` 的 api 写出 `byte[]` 内容，本质上是将 `byte[]` 在内存中的地址传递给了系统的 `socket` 接口。但是由于 JVM 的 GC 是会挪动堆内数据的位置的，这就可能导致系统在读取数据时因为 JVM 的 GC 导致原先给定的地址读取到非法数据。而堆外内存则不受 GC 控制，因此通过 `socket` 执行写出的场合，如果传递的是堆内的 `byte[]` 都需要拷贝堆外执行。而 JVM 执行的拷贝是自行创建一个 `DirectByteBuffer` 实例来进行数据拷贝，而创建 `DirectByteBuffer` 是消耗较大的操作，如果 Netty 层面可以直接转化，则可以提升效率。因此在 `filterOutboundMessage` 方法内会尝试执行转换。

接着来看 ** 代码③**。Netty 中使用 `MessageSizeEstimator.Handle#size` 接口来估计需要写出的对象的大小。该接口的实现有两个：

- 默认实现：`DefaultMessageSizeEstimator.HandleImpl`，如果对象是 `ByteBuf` 或者 `ByteBufHolder` 则返回对象大小，如果是 `FileRegion` 返回 0（`FileRegion` 是在写出时直接从磁盘读取数据，对象内部没有持有 `byte[]` 等内容），其余情况返回默认的估计值。
- 专用于 Http2 协议的帧大小估计的实现：`FlowControlledFrameSizeEstimator$1`，`FlowControlledFrameSizeEstimator` 提供的匿名内部类实现。

代码③计算得到的消息大小，会在将对象放入写出缓存时用于对当前写出缓存消耗总内存的计算。

接着来看 ** 代码④**。将消息添加到写出缓存中。当这个消息代表的数据被真正写出到 socket 缓存区后，与该消息关联的 `promise` 对象会被设置写出结果，用于通知在其上的监听器。

到这里，写出任务就完成了，实际的数据写出则依托于写出缓存内部的实现逻辑。

## 数据刷出 flush

数据的刷出依靠方法 `channel.flush`，该方法触发了管道的 `flush` 事件，这个事件最终会传递到首节点，首节点的 `flush` 方法再次委托给 `unsafe` 的 `flush` 方式，其具体实现如下

```java
public final void flush()
{assertEventLoop();
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if(outboundBuffer == null)
    {return;}
    outboundBuffer.addFlush();
    flush0();}
protected final void flush0()
{if(!isFlushPending())
    {super.flush0();
    }
}
private boolean isFlushPending()
{SelectionKey selectionKey = selectionKey();
    return selectionKey.isValid()&& (selectionKey.interestOps() & SelectionKey.OP_WRITE)!= 0;}
```

方法 `outboundBuffer.addFlush()` 用于添加一个刷出标记到写出缓存中，具体的作用后文在分析。方法 `isFlushPending` 用于检查当前通道上是否已经有写出就绪等待了。如果当前通道上已经注册了可写就绪事件关注，则意味着之前已经有数据在等待刷出（后文在写出缓存流程中详细分析）。

如果 `isFlushPending` 返回 `false`，意味着当前没有数据正在刷出，则触发父类的 `flush0` 方法进行数据刷出，`flush0` 方法内部也是再次委托了 `NioSocketChannel#doWrite`，但是 `flush0` 本身也做了一些模板方法处理，比如在开始前检查通道是否仍然激活，如果不的话则作废写出缓存中的所有数据；比如在 `dowrite` 出现异常时关闭通道并且作废写出缓存中的数据。由此可以看出，调用 `dowrite` 方法时，Netty 真正的启动了数据的刷出。为了更好的理解的刷出的过程，这里我们先暂停下，来看下写出缓存 `ChannelOutboundBuffer` 的数据结构。

## ChannelOutboundBuffer 数据结构

`ChannelOutboundBuffer` 是一个由其元素对象 `ChannelOutboundBuffer.Entry` 构成的单向链表。该链表有三个指针：

- `tailEntry`：指向链表中最后一个元素。
- `unflushedEntry`：指向链表中第一个未标记为刷出的元素。
- `flushedEntry`：指向链表中第一个标记为刷出的元素。

一开始，队列是空的，三个指针也都没有任何指向。可以通过调用方法 `ChannelOutboundBuffer#addMessage` 来添加元素。

### 添加元素

插入第一个元素，队列就成了

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191218152203.png)

每次插入元素，都会改变 `tailEntry` 的值。多次插入后队列就成了

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191218152253.png)

有了形象的图之后再来看代码就好理解了，如下

```java
public void addMessage(Object msg, int size, ChannelPromise promise)
{
    // 代码①
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if(tailEntry == null)
    {flushedEntry = null;}
    else
    {
        Entry tail = tailEntry;
        tail.next = entry;
    }
    tailEntry = entry;
    if(unflushedEntry == null)
    {unflushedEntry = entry;}
    // 代码②
    incrementPendingOutboundBytes(entry.pendingSize, false);
}
```

首先是 ** 代码①**。构建一个新的 `Entry` 实例，这边在构建的时候，传递了 2 个大小参数：`size`,`total(msg)`。两者主要是用途不同：前者是其他方法中预测的 `msg` 代表的字节长度，该长度加上 `Entry` 对象本身占据的字节长度，就成为了一个 Entry 对象消耗的内存大小；后者是私有方法 `total` 计算得到的数据大小（该大小不一定表现为字节长度），用于在数据写出的时，以写出进度的时候通知 `promise`，如果 `promise` 的类型是 `ChannelProgressivePromise`。

在 ** 代码②** 之前，都是在往队列的尾巴上添加新的元素。

接着看 ** 代码②**。`incrementPendingOutboundBytes` 方法通过 CAS 方式将新增 Entry 的对象消耗内存大小增加到队列的总体内存消耗大小中。并且在消耗大小超过给定的水平线，设置不可写标记，并且触发 `channelWritabilityChanged` 事件。

### 添加刷出标记

写出缓存通过方法 `addFlush`，设定了 `flushedEntry` 的指向，意味着在这之前添加的元素都需要被刷出。仍然接着上面的图，调用 `addFlush`，会将 `flushedEntry` 指向 `unflushedEntry`，此时队列变为

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191218160244.png)

每次添加刷出标记，都会将 `unflushedEntry` 置空。如果此时再通过 `addMessage` 添加数据，队列就会变成

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191218161811.png)

或者

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191218161825.png)

看完变化图，在来看代码，`addFlush` 代码如下

```java
public void addFlush()
{
    Entry entry = unflushedEntry;
    // 代码①
    if(entry != null)
    {if(flushedEntry == null)
        {flushedEntry = entry;}
        // 代码②
        do {
            flushed++;
            if(!entry.promise.setUncancellable())
            {int pending = entry.cancel();
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);
        unflushedEntry = null;
    }
}
```

首先来看 ** 代码①**，只有当前写出缓存队列中存在未标记为刷出的 `Entry` 时，才需要添加刷出标记，否则要么队列没数据，要么已经标记过刷出 `Entry` 了。

接着来看 ** 代码②**。从 `unflushedEntry` 指向的 `Entry` 开始，不断的遍历，累加未真正写出的需要刷出的 `Entry` 的总数。并且设置每一个 `Entry` 的 `promise` 状态为不可取消，这一步是用来保留一个取消机会给外部的。每一次添加刷出标记，必然是要刷出整个队列的内容，因此 `unflushedEntry` 指针在最后设置为空。

## 真正的数据写出

在了解 `ChannelOutboundBuffer` 结构的基础上，我们在回头去看前面是 `NioSocketChannel#doWrite` 就更好理解了，下面来看代码

```java
protected void doWrite(ChannelOutboundBuffer in) throws Exception
{SocketChannel ch = javaChannel();
    int writeSpinCount = config().getWriteSpinCount();
    do {if( in.isEmpty())
        {clearOpWrite();
            return;
        }
        int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
        // 代码①
        ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
        int nioBufferCnt = in.nioBufferCount();
        // 代码②
        switch(nioBufferCnt)
        {
            case 0:
                // 代码④
                writeSpinCount -= doWrite0(in);
                break;
            case 1:
                {
                    // 代码③
                    ByteBuffer buffer = nioBuffers[0];
                    int attemptedBytes = buffer.remaining();
                    final int localWrittenBytes = ch.write(buffer);
                    if(localWrittenBytes <= 0)
                    {incompleteWrite(true);
                        return;
                    }
                    adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite); in .removeBytes(localWrittenBytes);
                    --writeSpinCount;
                    break;
                }
            default:
                {long attemptedBytes = in .nioBufferSize();
                    final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                    if(localWrittenBytes <= 0)
                    {incompleteWrite(true);
                        return;
                    }
                    adjustMaxBytesPerGatheringWrite((int) attemptedBytes, (int) localWrittenBytes, maxBytesPerGatheringWrite); in .removeBytes(localWrittenBytes);
                    --writeSpinCount;
                    break;
                }
        }
    } while (writeSpinCount > 0);
    incompleteWrite(writeSpinCount < 0);
}
```

首先来看 ** 代码①**。方法 `ChannelOutboundBuffer#nioBuffers(int, long)` 从需要写出的 Entry 开始，按照给定的检查数量和给定的数据长度逐个检查 `Entry`，如果 `entry.msg` 是一个 `ByteBuf` 则获取其相同数据的等价内部 `ByteBuffer` 对象，所有这些 `ByteBuf` 提取的内部 `ByteBuffer` 对象都放入 `ByteBuffer[]` 中，并返回这个数组对象。同时可以通过方法 `ChannelOutboundBuffer#nioBufferCount` 获取该数组中有效元素的个数（因为 `ByteBuffer[]` 对象本身会被线程变量缓存起来复用）。

如果逐个检查 `Entry` 的过程中，发现 `Entry.msg` 不是 `ByteBuf` 类型，返回的数组中则没有有效数据。

接着来看 ** 代码②**。根据 `ByteBuffer[]` 有效元素的个数，区分为三种不同的情况：

- 0：这意味着 `ChannelOutboundBuffer` 当前需要写出的数据不是 `ByteBuf`，调用通用数据写出方法 `AbstractNioByteChannel#doWrite0`。
- 1：这意味着当前只有一个 `ByteBuffer` 需要写出，直接调用原生接口进行数据写出即可。
- \>1：与只有 1 个元素的处理流程完全相同，只不过写出的时候调用数组写出进行处理。

优先来看简单情况的处理流程，也就是 ** 代码③**。

首先是方法 `SocketChannel#write(java.nio.ByteBuffer)` 来将 ByteBuffer 写出，如果本次写出的字节数为 0，意味着当前的 socket 缓冲区已经满了，此时在继续轮训等待的意义不大，则调用方法 `incompleteWrite` 注册可写就绪事件关注，等待 socket 缓冲区有写出的空间了，再继续写出。如果本次写出的字节数不等于 0，意味着有数据被写出（写出到 socket 缓冲区）了，此时调用方法 `adjustMaxBytesPerGatheringWrite` 来调整配置项：单次写出最大字节数。其调整逻辑也很简单：如果本次写出数和配置的最大写出数相等，则配置项数值增大 2 倍；反而则缩小为二分之一。通过这种动态变化的时候，可以让配置的单次最大写出数始终接近于 socket 的缓冲区大小。

接着是调用方法 `ChannelOutboundBuffer#removeBytes` 传递已经写出的字节长度，根据已经写出的字节长度，可以判断当前有多少 `Entry` 被写出了，被写出的 `Entry` 关联的 `promise` 会被设置为 success。在这个过程中，`flushedEntry` 会逐步向后移动，指向当前还没有完全写出的 `Entry`。

看过了简单的情况，我们在来看复杂的处理情况，也就是 ** 代码④**。方法 `AbstractNioByteChannel#doWrite0` 从 `ChannelOutboundBuffer` 获取当前的 `Entry`，如果当前 `Entry` 为 null 则直接返回 0，否则委托给方法 `AbstractNioByteChannel#doWriteInternal`, 来看下其代码内容

```java
    private int doWriteInternal(ChannelOutboundBuffer in, Object msg) throws Exception
    {if (msg instanceof ByteBuf)
        {// 省略代码：将 ByteBuf 数据写出，如果实际写出了字节，则变更当前 Entry 的写出进度并返回 1；如果没有实际写出字节，则从队列中移除该 Entry 并返回 0}
        else if (msg instanceof FileRegion)
        {// 省略代码：如果 FileRegion 已经传输了全部指定的长度，则移除队列当前的 Entry 并返回 0；否则通过 JDk NIO 接口实现两个通道之间直接的数据传输，避免通过 ByteBuffer 中转，提升效率。传输完成后如果传输完毕也要移除队列的当前 Entry。并且最终返回 1}
        else
        {throw new Error();
        }
        // 如果 if 和 else if 分支中在有数据可以写，但是实际写出字节数为 0 的情况下，意味着 socket 缓冲区满，返回 Integer.MAX_VALUE 以表示。
        return WRITE_STATUS_SNDBUF_FULL;
    }
```

再看 `NioSocketChannel#doWrite`，其数据写出功能，是在一个 `while` 循环中，默认最多循环 16 次。在超出循环次数，或者直接发现 socket 缓存区已满无法写出的情况下，则会跳出循环，调用 `AbstractNioByteChannel#incompleteWrite` 去处理无法完整写出数据的情况，来看具体的代码

```java
    protected final void incompleteWrite(boolean setOpWrite)
    {if (setOpWrite)
        {setOpWrite();
        }
        else
        {clearOpWrite();
            eventLoop().execute(flushTask);
        }
    }
```

如果在 `doWrite` 中，写出循环次数耗尽而进入该方法，此时 `setOpWrite` 为 `false`。Netty 认为此种情况下，将写出动作包装为任务，投递到 `EventLoop` 线程中，等待下次调度即可。而如果是因为 socket 缓存区满导致数据无法写出，则无法预计而是可以写出，此时的处理策略就是注册可写事件关注，等待被 `selector` 触发。也就是方法 `setOpWrite` 的内容。

## 可写就绪事件处理

上文我们讲到，如果在写出过程中，发现 socket 缓冲区满了，此时会注册可写就绪事件，等待下次被 `selector` 调度。那我们接下来来看下 Netty 中对于可写就绪事件的处理，其代码在方法 `NioEventLoop#processSelectedKey(SelectionKey, AbstractNioChannel)` 的写事件处理分支，如下

```java
if((readyOps & SelectionKey.OP_WRITE) != 0)
{ch.unsafe().forceFlush();}
```

而这个 `forceFlush` 方法的实现就是直接委托给父类的 `flush0` 方法，这个方法已经在《数据刷出 flush》中介绍过了。

## 总结与思考

在 `Channel` 中，调用方法只是将数据放入了写出缓存，并不会触发真正的写出动作。而调用 `flush` 则将会之前加入到缓存中的数据都写出。而实际写出的过程也区分为两个大的步骤，首先是尝试在 `EventLoop` 线程中将所有数据都写出；而如果因为 socket 缓冲区满无法写出时，则注册可写就绪事件，等待被 `selector` 调度时再继续写出动作。