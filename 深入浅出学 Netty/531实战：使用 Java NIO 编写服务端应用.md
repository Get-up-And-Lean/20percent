# 5/31实战：使用 Java NIO 编写服务端应用

## 需求分析

本文将实战如何使用Java NIO编写一个趋向于实际的echo应用。首先是明确这个echo服务器的需求，总结来说有以下几条：

- 服务器原样返回客户端发送的信息。
- 客户端发送的信息以'\r'作为一个消息的结尾，一个消息的最大长度不超过128。
- 客户端可能会一次发送多个消息，服务端需要按照收到的消息的顺序依次回复，不能乱序。
- 客户端可以在任意时刻关闭通道。
- 服务端不能主动关闭通道。

## 代码实战

很少有程序是一蹴而就的，一般都是在满足需求，再反复修改细节得到最终的成品。在这里，我们先以《Java的服务端编程进化史：从BIO到NIO，最后走向AIO》一文中NIO的代码作为基础蓝本进行改造。为了方便区分改造区域，我们将基础蓝本中处理客户端的单独剥离，成为一个独立的类，最后得到的基础代码如下

```java
public class MainDemo
{
    static class ClientProcessor implements Runnable
    {
        private Selector selector;

        public ClientProcessor(Selector selector)
        {
            this.selector = selector;
        }

        @Override
        public void run()
        {
            while (true)
            {
                try
                {
                    selector.select();
                    Set<SelectionKey>      selectionKeys = selector.selectedKeys();
                    Iterator<SelectionKey> iterator      = selectionKeys.iterator();
                    while (iterator.hasNext())
                    {
                        SelectionKey key = iterator.next();
                        if (key.isValid() == false)
                        {
                            continue;
                        }
                        if (key.isReadable())
                        {//代码①
                            ByteBuffer    buffer        = ByteBuffer.wrap(new byte[16]);
                            SocketChannel clientChannel = (SocketChannel) key.channel();
                            int           read          = clientChannel.read(buffer);
                            if (read == -1)//关闭分支
                            {
                                //通道连接关闭，可以取消这个注册键，后续不在触发。
                                key.cancel();
                                clientChannel.close();
                            }
                            else//读写分支
                            {
                                buffer.flip();
                                clientChannel.write(buffer);
                            }
                        }
                        iterator.remove();
                    }
                }
                catch (IOException e)
                {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) throws IOException, ExecutionException, InterruptedException
    {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        final Selector[] selectors = new Selector[Runtime.getRuntime().availableProcessors()];
        for (int i = 0; i < selectors.length; i++)
        {
            final Selector selector = Selector.open();
            selectors[i] = selector;
            new Thread(new ClientProcessor(selector)).start();
        }
        AtomicInteger id       = new AtomicInteger();
        Selector      selector = Selector.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true)
        {
            selector.select();
            Set<SelectionKey>      selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator      = selectionKeys.iterator();
            while (iterator.hasNext())
            {
                iterator.next();
                SocketChannel socketChannel = serverSocketChannel.accept();
                socketChannel.register(selectors[id.getAndIncrement() % selectors.length], SelectionKey.OP_READ);
                iterator.remove();
            }
        }
    }
}
```

### 第一次修改

针对需求2：客户端发送的信息以'\r'作为一个消息的结尾，一个消息的最大长度不超过128。这意味着我们不能以定长的形式处理消息了。而且此时我们需要考虑TCP拆包和粘包的可能。

> TCP是面向流的协议，其本身无法知道上层协议一个数据包的边界。因此在接受数据的时候，可能会因为一个消息过大而分次填充到Socket缓存区，此时应用读取到数据，感觉就是数据包被拆开了。
>
> 而如果TCP在收到数据时，将多个数据包的数据一起读取了一并填充到socket缓存区，此时应用读取到数据，感觉就是多个数据包粘合了。

结合需求2和需求3，我们需要改造的是数据的读取部分内容。主要变动有：

- 检查每一个字节，确认是否是消息结束符。
- 累积聚合字节直到一个完整的消息被拆分出来。

按照上述需求改动之后的代码如下：

```java
public class NioDemo
{
        static class ClientProcessor implements Runnable
        {
            private Selector selector;

            public ClientProcessor(Selector selector)
            {
                this.selector = selector;
            }

            @Override
            public void run()
            {
                while (true)
                {
                    try
                    {
                        selector.select();
                        Set<SelectionKey> selectionKeys = selector.selectedKeys();
                        for (SelectionKey each : selectionKeys)
                        {
                            if (each.isValid() == false)
                            {
                                continue;
                            }
                            if (each.isReadable())
                            {
                                SocketChannel clientChannel = (SocketChannel) each.channel();
                                ByteBuffer    readBuffer    = (ByteBuffer) each.attachment();
                                int           read          = clientChannel.read(readBuffer);
                                if (read == -1)
                                {
                                    //通道连接关闭，可以取消这个注册键，后续不在触发。
                                    each.cancel();
                                    clientChannel.close();
                                }
                                else
                                {
                                    //翻转buffer，从写入状态切换到读取状态
                                    readBuffer.flip();
                                    int              position = readBuffer.position();
                                    int              limit    = readBuffer.limit();
                                    List<ByteBuffer> buffers  = new ArrayList<>();
                                    //新增二：按照协议从流中分割出消息
                                    /**从readBuffer确认每一个字节，发现分割符则切分出一个消息**/
                                    for (int i = position; i < limit; i++)
                                    {
                                        //读取到消息结束符
                                        if (readBuffer.get(i) == '\r')
                                        {
                                            ByteBuffer message = ByteBuffer.allocate(i - readBuffer.position()+1);
                                            readBuffer.limit(i+1);
                                            message.put(readBuffer);
                                            readBuffer.limit(limit);
                                            message.flip();
                                            buffers.add(message);
                                            readBuffer.limit(limit);
                                        }
                                    }
                                    /**从readBuffer确认每一个字节，发现分割符则切分出一个消息**/
                                    /**将所有得到的消息发送出去**/
                                    for (ByteBuffer buffer : buffers)
                                    {
                                        //新增三
                                        while (buffer.hasRemaining())
                                        {
                                            clientChannel.write(buffer);
                                        }
                                    }
                                    /**将所有得到的消息发送出去**/
                                    //新增四：压缩readBuffer，压缩完毕后进入写入状态。并且由于长度是256，压缩之后必然有足够的空间可以写入一条消息
                                    readBuffer.compact();
                                }
                            }
                        }
                        selectionKeys.clear();
                    }
                    catch (IOException e)
                    {
                        e.printStackTrace();
                    }
                }
            }
        }

        public static void main(String[] args) throws IOException, ExecutionException, InterruptedException
        {
            ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.bind(new InetSocketAddress(8899));
            final Selector[] selectors = new Selector[Runtime.getRuntime().availableProcessors()];
            for (int i = 0; i < selectors.length; i++)
            {
                final Selector selector = Selector.open();
                selectors[i] = selector;
                new Thread(new ClientProcessor(selector)).start();
            }
            AtomicInteger id       = new AtomicInteger();
            Selector      selector = Selector.open();
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            while (true)
            {
                selector.select();
                Set<SelectionKey>      selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator      = selectionKeys.iterator();
                while (iterator.hasNext())
                {
                    iterator.next();
                    SocketChannel socketChannel  = serverSocketChannel.accept();
                    socketChannel.configureBlocking(false);
                    Selector      selectForChild = selectors[id.getAndIncrement() % selectors.length];
                    //新增一：每一个选择键或者说通道持有一个独占的buffer，用于数据的累计读取
                    socketChannel.register(selectForChild, SelectionKey.OP_READ, ByteBuffer.allocate(128 * 2));
                    selectForChild.wakeup();
                    iterator.remove();
                }
            }
        }
}
```

主要的改动有四点，下面逐一来分析。

首先是**新增一**。

通道在注册选择器是附带上一个附件：`ByteBuffer`。这样就使得这个选择键或者说这个通道有一个独占的`ByteBuffer`用于通道数据的读取和累积。因为TCP拆包粘包可能性的存在，一次读取的数据可能分割出多个消息，也可能不足以分割出一个消息。这就有可能导致一次读取到的数据不能刚好的完全处理完毕，会剩余下一些。那么下一次通道数据的读取必须在剩余数据的基础上进行累加，而不是简单的新建一个`ByteBuffer`从通道中数据，否则就会丢失一部分数据。通过对每一个通道注册读取时，始终携带一个`ByteBuffer`对象作为附件，可以累积之前未解析完成的报文数据，从而保证数据的完整性。

然后是**新增二**。

由于客户端消息不再是定长，而是采用分隔符进行分割。因此也不能像一开始一样，新建一个固定长度的`ByteBuffer`，读取满了就算一个消息。而是必须逐一检查每一个字节，确认消息分隔符的存在。当发现一个消息分隔符时，就把`ByteBuffer`从`position`位置到分隔符之间的所有数据提取到一个新的`ByteBuffer`中，这部分数据也就是一个完整的，单一的客户端消息。

在新增二的这段代码中，需要意识到一次读取的`readBuffer`是可能存在多个消息的，因此使用for循环不断的检查判断，每次发现一个消息后，先在`readBuffer`设置这个消息的区间，主要就是设置`limit`位置。提取完毕后再将`limit`恢复到原先的位置，继续拆分下一个可能的消息。

再次是**新增三**

并不是像之前一样执行一次写入就完毕了。因此非阻塞的socket通道在写出的时候，只是将数据写出到socket的写出缓存区。如果缓存区满了，写方法会直接返回，而不会等待所有的数据全部写出。这和阻塞式通道是不同的。

因此需要通过`while`循环确认整个`buffer`的内容都被写出了，才能开始写出下一个`buffer`。

最后是**新增四**

客户端消息分拆完毕并且都发送出去后，就需要再次开始读取。在代码执行到新增四这个地方的时候`readBuffer`处于读取状态，并且有效数据区域也相对靠后了。此时通过`compact`方法将有效区域移动到头部，并且切换到写入状态，方便执行下一次通道中读取数据任务。

### 第二次修改

上述的代码能够很好的完成需求，但是在性能上有一个缺陷。在将客户端消息写出时是在当前线程循环判断`buffer`是否都被写出完毕。如果当前tcp通道速率不足或者较为拥堵，导致socket缓存区比较容易满，或者较长时间处于满的状态，那么应用程序就会一直告诉循环，因为数据始终无法写出。这部分循环消耗了cpu却又没有做任何有意义的事情。此时可以更新通道关注的IO事件为读事件和写事件。当通道可写时再执行写入，而不是无意义的自旋浪费。

按照这种思路修改，意味着一个通道上有自己独占的读取`ByteBuffer`，以及在无法及时写出时还需要关联写出的`ByteBuffer`。只将`readBuffer`作为选择键的附件就不够了，需要用一个类结构将这两者合并。最后修改的代码如下

```java
public class NioDemo2
{
        static class ChannelBuffer
        {
            ByteBuffer       readBuffer;
            ByteBuffer[]     writeBuffers;
            List<ByteBuffer> list = new LinkedList<>();
        }

        static class ClientProcessor implements Runnable
        {
            private Selector selector;

            public ClientProcessor(Selector selector)
            {
                this.selector = selector;
            }

            @Override
            public void run()
            {
                while (true)
                {
                    try
                    {
                        selector.select();
                        Set<SelectionKey> selectionKeys = selector.selectedKeys();
                        for (SelectionKey each : selectionKeys)
                        {
                            if (each.isValid() == false)
                            {
                                continue;
                            }
                            if (each.isReadable())
                            {
                                SocketChannel clientChannel = (SocketChannel) each.channel();
                                ChannelBuffer channelBuffer = (ChannelBuffer) each.attachment();
                                ByteBuffer    readBuffer    = channelBuffer.readBuffer;
                                int           read          = clientChannel.read(readBuffer);
                                if (read == -1)
                                {
                                    each.cancel();
                                    clientChannel.close();
                                    continue;
                                }
                                else
                                {
                                    readBuffer.flip();
                                    int              position = readBuffer.position();
                                    int              limit    = readBuffer.limit();
                                    List<ByteBuffer> buffers  = new ArrayList<>();
                                    for (int i = position; i < limit; i++)
                                    {
                                        //读取到消息结束符
                                        if (readBuffer.get(i) == '\r')
                                        {
                                            ByteBuffer message = ByteBuffer.allocate(i - readBuffer.position()+1);
                                            readBuffer.limit(i+1);
                                            message.put(readBuffer);
                                            readBuffer.limit(limit);
                                            message.flip();
                                            buffers.add(message);
                                            readBuffer.limit(limit);
                                        }
                                    }
                                    //修改一：区分对象是否还有剩余数据未发送完成，以及在数据不可写入的时候关注通道可写事件等待下一次机会
                                    if (channelBuffer.writeBuffers == null)
                                    {
                                        //没有还未发送完全的数据，那么本次需要发送的数据可以直接进行发送
                                        ByteBuffer[] byteBuffers = buffers.toArray(new ByteBuffer[buffers.size()]);
                                        clientChannel.write(byteBuffers);
                                        boolean hasRemaining = hasRemaining(byteBuffers);
                                        if (hasRemaining)
                                        {
                                            channelBuffer.writeBuffers = byteBuffers;
                                            each.interestOps(each.interestOps() | SelectionKey.OP_WRITE);
                                        }
                                    }
                                    else
                                    {
                                        //还有尚未发送完全的数据，新产生的数据需要放入队列
                                        channelBuffer.list.addAll(buffers);
                                    }
                                    readBuffer.compact();
                                }
                            }
                            if (each.isWritable())
                            {
                                //新增二：新增在通道可写事件触发后，将之前剩余的数据写出
                                SocketChannel clientChannel = (SocketChannel) each.channel();
                                ChannelBuffer channelBuffer = (ChannelBuffer) each.attachment();
                                ByteBuffer[]  writeBuffers  = channelBuffer.writeBuffers;
                                clientChannel.write(writeBuffers);
                                boolean hasRemaining = hasRemaining(writeBuffers);
                                if (hasRemaining == false)
                                {
                                    channelBuffer.writeBuffers = null;
                                    List<ByteBuffer> list = channelBuffer.list;
                                    if (list.isEmpty() == false)
                                    {
                                        writeBuffers = list.toArray(new ByteBuffer[list.size()]);
                                        list.clear();
                                        clientChannel.write(writeBuffers);
                                        if (hasRemaining(writeBuffers))
                                        {
                                            //仍然有数据没有完全写出，保留对可写事件的关注
                                        }
                                        else
                                        {
                                            //没有数据要写出了，取消对可写事件的关注
                                            each.interestOps(each.interestOps() & ~SelectionKey.OP_WRITE);
                                        }
                                    }
                                    else
                                    {
                                        //没有数据要写出了，取消对可写事件的关注
                                        each.interestOps(each.interestOps() | ~SelectionKey.OP_WRITE);
                                    }
                                }
                                else
                                {
                                    //数据还没有写完，继续保持对可写事件的关注
                                }
                            }
                        }
                        selectionKeys.clear();
                    }
                    catch (IOException e)
                    {
                        e.printStackTrace();
                    }
                }
            }

            private boolean hasRemaining(ByteBuffer[] byteBuffers)
            {
                boolean hasRemaining = false;
                for (ByteBuffer byteBuffer : byteBuffers)
                {
                    if (byteBuffer.hasRemaining())
                    {
                        hasRemaining = true;
                        break;
                    }
                }
                return hasRemaining;
            }
        }

        public static void main(String[] args) throws IOException, ExecutionException, InterruptedException
        {
            ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.bind(new InetSocketAddress(8899));
            final Selector[] selectors = new Selector[Runtime.getRuntime().availableProcessors()];
            for (int i = 0; i < selectors.length; i++)
            {
                final Selector selector = Selector.open();
                selectors[i] = selector;
                new Thread(new ClientProcessor(selector)).start();
            }
            AtomicInteger id       = new AtomicInteger();
            Selector      selector = Selector.open();
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            while (true)
            {
                selector.select();
                Set<SelectionKey>      selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator      = selectionKeys.iterator();
                while (iterator.hasNext())
                {
                    iterator.next();
                    SocketChannel socketChannel = serverSocketChannel.accept();
                    socketChannel.configureBlocking(false);
                    Selector      selectForChild = selectors[id.getAndIncrement() % selectors.length];
                    //新增一：使用ChannelBuffer作为选择键的附件
                    ChannelBuffer channelBuffer = new ChannelBuffer();
                    channelBuffer.readBuffer = ByteBuffer.allocate(128 * 2);
                    socketChannel.register(selectForChild, SelectionKey.OP_READ, channelBuffer);
                    selectForChild.wakeup();
                    iterator.remove();
                }
            }
        }
}
```

可以看到，代码整体复杂了许多，主要的新增点在2个地方：

- **新增一**：当向通道写出数据的时候，如果无法一次性完全写完则不再循环等待，而是更新选择键的关注事件，增加了对通道可写事件的关注。等待后续通道可写的时候继续当前未完成的写入。而如果通道当前已经有未写出完全的数据，则新分割出需要回传的消息则需要进入待发送队列，以保持回复客户端消息的顺序正确性。
- **新增二**：当通道可写事件触发后，则开始将之前未写完的数据继续写出。重复这个过程直到数据全部被写出。此时检查待发送队列中是否存在数据，如果存在的话导出为`ByteBuffer`数组，并且继续执行写出。写出的思路仍然是相同的，如果不能一次写出，则注册对通道可写事件的关注（如果已经关注了则无需修改）。如果待发送队列没有数据，之前的`ByteBuffer`数组也都写出完毕了，则可以取消对通道可写事件的关注。

## 注意点

在上面的代码中可以注意到，调用`java.nio.channels.Selector#selectedKeys`获得的已就绪的选择键集合在遍历完成后执行了`clear`方法。这是个必要的动作。

前文有分析过选择器本身会持有三个集合，其中之一就是已就绪的键集合。该集合的数据会在`java.nio.channels.Selector#select()`类方法被调用后添加以及刷新。而集合的数据只能被用户删除，选择器本身是不会选择的。

这就意味着如果对已就绪的选择键执行完操作后，如果不从已就绪的键集合中删除它，那么下次`select`方法返回的时候还会再次处理到这个键，而此时这个选择键代表的通道实际上可能并没有关注事件发生。从而可能带来程序的错误。

因此正确的用法就是在处理完就绪的选择键后，将该键从就绪集合中删除。

## 总结与思考

本文使用NIO原生的API开发了一个echo服务器，并且随着实际项目中的问题的引入，一步步的不断修改原始程序。随着修改的不断进行，这个demo的echo服务器越来越完善，也越来越接近生产项目。这个过程并不轻松，需要介入思考的点也挺多的。而这还只是一个最简单的echo服务器。如果是生产上的项目，需要考虑的东西就更多了。而如果要尽可能的考虑复用，事情会就变得更复杂了。更不要说JDK中还藏着臭名昭著的CPU 100% bug。

在项目的过程中，如果将大量的时间消耗在技术的实现细节本身上，对于业务的关注显然就会降低，不利于业务的快速开展和验证。这个时候，我们就需要求助于生产级的IO框架：Netty。下一篇文章，我们会带领大家看看，使用Netty开发一个网络IO应用，是这么多的简单和轻松。