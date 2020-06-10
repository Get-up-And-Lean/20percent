# 3/31Java的服务端编程进化史：从 BIO 到 NIO，最后走向 AIO

## 前言

Java 语言诞生之初就提供了`Socket`套接字相关 API，用于支撑网络编程需求。早期的`Socket`接口是同步阻塞式IO，性能并不是很高，在服务端编程这块一直处于劣势。直到 JDK1.4 发布，伴随而来的是新的 NIO 包以及其提供了 IO 复用模型下的API，极大提高了网络IO效率，很多服务器开始采用这种模型，处理能力也有了极大的提升。随着 JDK1.7 的发布，JDK 也提供了对异步IO的支持。理论上来说，异步IO 是效率最高的。但是由于 Java 主要都是服务端程序，大部分都运行在Linux 系统上，而 Linux 对 AIO 的支持较晚。因此现在采用异步 IO 的且有较大影响力的程序还不多。

## Java的服务端编年史

### 鸿蒙时代：BIO

伴随着Java的发布，带来的是`Socket`套接字API。这套API实现是的同步阻塞IO模型。下面首先来看个示例，如何使用这套API完成一个echo服务端程序

```java
public class MainDemo
{
    public static void main(String[] args) throws IOException, ExecutionException, InterruptedException
    {
        ServerSocket serverSocket = new ServerSocket();
        serverSocket.bind(new InetSocketAddress("0.0.0.0", 8080),50);
        Socket socket;
        while ((socket = serverSocket.accept()) != null)
        {
            InputStream inputStream = socket.getInputStream();
            byte[]      data        = new byte[16];
            inputStream.read(data);
            OutputStream outputStream = socket.getOutputStream();
            outputStream.write(data);
            socket.close();
        }
    }
}
```

这个程序假定网络环境良好，每次客户端发送的数据均为16字节，且未发生TCP粘包/拆包情况。

代码十分简单，首先是创建了一个服务端的`ServerSocket`实例，将这个实例绑定到一个监听地址和端口上。这里监听地址使用`.0.0.0.0`意味着可以监听本机网卡的所有IP地址。第二个参数50意味着服务端最多可以存储50个TCP三次握手进行中以及完成了三次握手还没有被取走处理的客户端链接。

监听地址和端口绑定完成后，调用方法`java.net.ServerSocket#accept`，线程阻塞直到有一个客户端链接完成TCP三次握手成功创建并且被取走。在代码中的表现就是`accept`方法返回，并且返回一个`Socket`客户端实例。

接着在`while`循环体中就是读取客户端的数据，并且原样发送回客户端。完成之后将客户端关闭，等待下一个客户端的连接。

代码很简单，问题也很突出。首先是客户端链接建立成功，获得实例后，服务端线程就阻塞在客户端的数据读取上，之后再次阻塞在数据的写出上。两个操作都成功后才能再服务下一个客户端。

一次只能服务一个客户端，且如果这个客户端发送数据较慢还会导致长时间的等待，这样就很可能造成其他尝试连接到服务端的客户端链接等待超时。造成这些问题的原因就在于服务端是单线程的，一次只能处理一个客户请求。那么很容易想到使用多线程来加速程序。服务端主线程只负责接入客户端链接，对客户端链接的数据处理交给其他线程去完成。依照这个想法，我们将上面的代码修改如下

```java
public class MainDemo
{
    public static void main(String[] args) throws IOException, ExecutionException, InterruptedException
    {
        ServerSocket serverSocket = new ServerSocket();
        serverSocket.bind(new InetSocketAddress("0.0.0.0", 8080), 50);
        Socket socket;
        while ((socket = serverSocket.accept()) != null)
        {
            final Socket finalSocket = socket;
            new Thread(new Runnable()
            {
                @Override
                public void run()
                {
                    try
                    {
                        InputStream inputStream = finalSocket.getInputStream();
                        byte[]      data        = new byte[16];
                        inputStream.read(data);
                        OutputStream outputStream = finalSocket.getOutputStream();
                        outputStream.write(data);
                        finalSocket.close();
                    }
                    catch (Exception e)
                    {
                        ;
                    }
                }
            }).start();
        }
    }
}
```

看上去问题被解决了，一切都挺美好的。服务端`ServerSocket`在客户端链接建立后，将其转发给新线程处理，自己则继续等待下一个客户端。这样客户端就能快速的接入了。但是这里存在着一个隐患。由于线程的创建和开销都消耗很大，如果短时间大量客户端链接涌入，则会一下子创建很多线程。这会对系统造成巨大的压力。而如果客户端的数据处理速度再慢一些，涌入的客户端比处理完毕的客户端多，最终会导致系统OOM而宕机。

为了解决这个问题，我们再次修改下程序，不再新建线程，而是使用线程池，修改后的代码如下

```java
public class MainDemo
{
    public static void main(String[] args) throws IOException, ExecutionException, InterruptedException
    {
        ExecutorService executorService = Executors.newFixedThreadPool(4);
        ServerSocket    serverSocket    = new ServerSocket();
        serverSocket.bind(new InetSocketAddress("0.0.0.0", 8080), 50);
        Socket socket;
        while ((socket = serverSocket.accept()) != null)
        {
            final Socket finalSocket = socket;
            executorService.submit(new Runnable()
            {
                @Override
                public void run()
                {
                    try
                    {
                        InputStream inputStream = finalSocket.getInputStream();
                        byte[]      data        = new byte[16];
                        inputStream.read(data);
                        OutputStream outputStream = finalSocket.getOutputStream();
                        outputStream.write(data);
                        finalSocket.close();
                    }
                    catch (Exception e)
                    {
                        ;
                    }
                }
            });
        }
    }
}
```

通过使用线程池，避免了线程数的无限制增长。到这里，阻塞式IO模型或者说Blocking IO模式下的服务端编程模式就确定下来了。可以简单总结为：

1. 服务端主线程负责接收客户端连接，并且生成的客户端链接投递到线程池中
2. 线程池中的线程负责执行对客户端链接的数据读写业务。

BIO 时代，比较有名的产物就是 Tomcat 了。其底层的连接模型就是上面我们介绍的这种模式。网页请求是比较适合这种模式来进行应用的，因为一个网页打开之后，TCP 连接就结束了，这使得客户端的数据操作在线程中执行的时间较为短暂就能释放线程资源供后续的客户端链接使用。但是即使这种比较适合的场景，Tomcat 的处理能力也并不是很强。在实际应用时，往往是前端放一个负载均衡的 Nginx，后端可以同时挂好几个 Tomcat 来分散处理需求。

### 走上舞台：NIO

虽然`Socket`模型提供了网络编程能力，但是性能实在比较差。即使在相对适合 BIO 应用的场景中，表现的也不尽如人意。更多时候，Java 系的服务端应用仅仅只是一种覆盖而已，在高性能网络编程领域都是 C 语言的地盘。特别早期有一个比较有名的`C10K`问题，就是单机连接数过 1w。在 Java 服务器上，几乎是一个极限的问题，即使是在硬件很好的商用服务器上。

而如果连接持续的时间较长的话，BIO 这种模型完全无法支撑。从网络模型上来说，解决`C10K`最好的办法就是 IO 多路复用。等待了 2 个版本后，Java 终于在 1.4 的发布中为我们带来了 NIO 的支撑。先抛开概念，首先来看下示例代码，仍然以上述的 echo 服务器为例，在 NIO 中可以如下实现：

```java
public class MainDemo
{
    public static void main(String[] args) throws IOException, ExecutionException, InterruptedException
    {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        Selector            selector            = Selector.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true)
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
                if (key.isAcceptable())//代码①
                {
                    //这里的channel和上文的serverSocketChannel是相同对象
                    ServerSocketChannel channel       = (ServerSocketChannel) key.channel();
                    SocketChannel       clientChannel = channel.accept();
                    clientChannel.configureBlocking(false);
                    clientChannel.register(selector, SelectionKey.OP_READ);
                }
                else if (key.isReadable())//代码②
                {
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
    }
}
```

为了简化程序，我们仍然假设网络良好，未曾发生过 TCP 拆包粘包情况，来分析下这个程序。

先看第一段代码

```java
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
serverSocketChannel.configureBlocking(false);
Selector            selector            = Selector.open();
serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
```

首先是创建服务端的 Socket 通道对象，也就是`java.nio.channels.ServerSocketChannel`，以及创建了一个选择器实例`java.nio.channels.Selector`。选择器是 Java 实现 IO 复用的核心组件。将服务端通道对象注册到选择器上，并且传入该通道关注的选择事件。服务端通道关注的是客户端通道的接入事件，也就是`accept`。到这里，准备工作就全部完成了。

之后的代码是一段 while 循环体，因为服务端是长时间处理链接，所以死循环是一个天然选择。

接下来是一个阻塞等待的过程，代码为

```java
selector.select();
Set<SelectionKey>      selectionKeys = selector.selectedKeys();
Iterator<SelectionKey> iterator      = selectionKeys.iterator();
```

线程阻塞在`java.nio.channels.Selector#select()`调用上，等待有感兴趣的事件发生。由于一开始只注册了服务端通道的`accept`关注事件，因此此时能触发的只有客户端的接入。当有客户端接入后，`select`方法就从阻塞中返回。此时调用`java.nio.channels.Selector#selectedKeys`方法获的从`select`调用后产生的选择键合集。

这里对选择键做一个说明。

> 选择键是一个标识，用于代表一个通道注册到了一个选择器上。因此选择器会包含三个重要属性：
>
> - 选择器对象
> - 通道对象
> - 通道的关注事件标识

遍历合集，取出每一个选择键。判断选择键关注的事件类型来决定不同的处理策略。来仔细看下 while 循环中的代码。

```java
Iterator<SelectionKey> iterator      = selectionKeys.iterator();
            while (iterator.hasNext())
            {
                SelectionKey key = iterator.next();
                if (key.isValid() == false)
                {
                    continue;
                }
                if (key.isAcceptable())//代码①
                {
                    //这里的channel和上文的serverSocketChannel是相同对象
                    ServerSocketChannel channel       = (ServerSocketChannel) key.channel();
                    SocketChannel       clientChannel = channel.accept();
                    clientChannel.configureBlocking(false);
                    clientChannel.register(selector, SelectionKey.OP_READ);
                }
                else if (key.isReadable())//代码②
                {
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
```

比如在代码① 处，这个选择键关注的是`accept`事件，这就意味着事件触发时，有客户端接入了。代码为

```java
if (key.isAcceptable())//代码①
                {
                    //这里的channel和上文的serverSocketChannel是相同对象
                    ServerSocketChannel channel       = (ServerSocketChannel) key.channel();
                    SocketChannel       clientChannel = channel.accept();
                    clientChannel.configureBlocking(false);
                    clientChannel.register(selector, SelectionKey.OP_READ);
                }
```

此时调用`java.nio.channels.ServerSocketChannel#accept`方法就可以获得接入的客户端通道实例。新生成的客户端通道实例也一样注册到选择器上，并且选择事件为socket可读。也就是说当 Socket 上的读取缓存区存在数据的时候选择键就会被触发。客户端通道完成注册后，这段处理逻辑就结束了。

而如果选择键表明其关注的是读取就绪事件，则可以从通道中数据读取，代码为

```java
else if (key.isReadable())//代码②
                {
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
```

在代码②处，这个选择键关注的读取就绪事件。这意味着前面注册在这个选择器上的客户端通道对应的 Socket 缓存区上存在数据，可以读取了。新建`ByteBuffer`，从通道上读取数据。然后再写入通道，完成 echo 功能。为了简化程序，这里假设写入可以一次性将`ByteBuffer`中的数据写出到通道的 Socket 缓存区中。

从上面的代码可以看到，通过`while`循环，仅仅使用一个选择器就可以处理多个通道上的任务。并且由于在获得选择键后，剩余的操作都可以较快的完成（从缓存区中读取数据和写入相比于内核等待数据的时间来说是固定可预测的），因此一个选择器就可以处理大量的通道事件，不会因为一个通道上的数据处理而大幅度延迟其他通道。

因为选择器的这种特性，一个选择器上可以注册数以千计的通道，极大的提高了单机的连接数。就像上面的示例程序一样，将所有的通道注册上，就可以在一个选择器上处理成百上千的客户端链接。

不过上面的代码也不是完全没问题。可以看到，整个服务端程序是单线程的，这样就无法有效的利用多核 CPU 的性能。从这个角度出发，多线程是最容易想到的一种思路。在 NIO 中应用多线程，主要有三种思路：

1. 思路一：只有一个选择器，把选择键的处理动作以任务的形式投放到线程池中处理。
2. 思路二：多个选择器，每一个选择器搭配一个线程，并且在自身的线程中死循环等待事件发生以及处理事件。
3. 思路三：思路二的变种，区分 2 组选择器，一组选择器专门用于服务端通道，用于客户端链接的接入；一组选择器专门用于客户端通道，用于客户端通道的数据读写。

Java的选择器在底层实现上是 Epoll（大部分服务端java都运行在 Linux 上，这里以 Linux 为例）。Epoll 支持无上限的客户端链接，且扫描性能不随着连接数增大而降低。从这个角度来说，一个选择器和多个选择器差异不大。

但是生成选择键后，分发这个动作必然伴随着一系列的包装对象，以及线程池投递会新增投递队列的节点等，这些开销在思路二中是不存在的，从这个角度出发，思路二是更好的选择。

因为服务端通道只关注客户端链接接入事件，与客户端的读写在职责上有明显的不同。因此思路三将多个选择器区分为两组，一组专用于服务端通道，一组专用于客户端通道。实际上，服务端仅仅只需要一个选择器就足够了，因此服务于服务端通道的选择器组的长度一般都是 1。

根据思路三，我们可以对程序进行以下修改：

- 初始化一组选择器对象，并且初始化对应数量的线程，以死循环的方式阻塞在选择器的`select`调用上。线程的 run 方法为使用选择键进行业务处理。
- 服务端通道通过`accept`方法创建链接对象时，通过轮训的方式，选择一个选择器对象，将通道注册到其上。

针对修改点一，我们初始化一组选择器并且实现线程的 run 方法，具体如下：

```java
public Selector[] initWorkerSelectors() throws IOException
    {
        final Selector[] selectors = new Selector[Runtime.getRuntime().availableProcessors()];
        for (int i = 0; i < selectors.length; i++)
        {
            final Selector selector = Selector.open();
            selectors[i] = selector;
            new Thread(new Runnable()
            {
                @Override
                public void run()
                {
                    while (true)
                    {
                        try
                        {
                            processWithSelector(selector);
                        }
                        catch (IOException e)
                        {
                            e.printStackTrace();
                        }
                    }
                }
            }).start();
        }
        return selectors;
    }

    private void processWithSelector(Selector selector) throws IOException
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
            {
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
```

针对修改点二，我们修改原本处理新建链接的方式，将其注册到专门用于处理链接的`Selector`上，使用的选择策略是轮训，修改后如下

```java
public void start() throws IOException
    {
        Selector            selector            = Selector.open();
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        AtomicInteger id        = new AtomicInteger();
        Selector[]    selectors = initWorkerSelectors();
        while (true)
        {
            selector.select();
            Set<SelectionKey>      selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator      = selectionKeys.iterator();
            while (iterator.hasNext())
            {
                iterator.next();
                SocketChannel socketChannel = serverSocketChannel.accept();
                //通过轮训，选择一个选择器进行注册，后续所有的操作也都在对应的线程上执行
                socketChannel.register(selectors[id.getAndIncrement() % selectors.length], SelectionKey.OP_READ);
                iterator.remove();
            }
        }
    }
```

结合两段代码可以看出，在刚开始时，新建了一个选择器用于给服务端通道注册，而主线程就在循环中处理服务端通道的接入就绪事件。并且在`while`循环之前，通过方法`initWorkerSelectors`创建了一组`Selector`对象并且为每一个`Selector`对象都绑定了一个线程，在线程自身也是在`while`循环中处理器绑定的`Selector`的就绪事件。

思路三基本而言就是使用 NIO 开发程序最为常见的套路了，广泛的应用在诸多基于 NIO 开发的网络 IO 框架中。比如著名的Netty。

### 走向未来：AIO

在第一篇网络IO模型介绍中，5种IO模型中只有一种是异步的，也就是异步IO。异步IO能提供更简单的编程模型和更好的效率（当然，效率取决于具体底层的实现。就目前来说Linux上的AIO实现性能没有明显的提升，以至于JDK的AIO实现在Linux上仍然是epoll）。再次间隔两个版本后，Java在1.7版本中为我们带来了异步IO的支持，也就是`java.nio.channels.AsynchronousChannel`系列。网络模型中有介绍，在异步IO下，应用程序是全程无阻塞的。需要关心的细节并不多，仍然以echo服务器为例子，我们来编写一个基于AIO（Asynchronous IO）的例子，代码如下

```java
public class MainDemo
{
    static class ClientReadHandler implements CompletionHandler<Integer, ByteBuffer>
    {
        AsynchronousSocketChannel socketChannel;

        public ClientReadHandler(AsynchronousSocketChannel socketChannel)
        {
            this.socketChannel = socketChannel;
        }

        @Override
        public void completed(Integer result, ByteBuffer buffer)
        {
            //代码①
            if (result == 16)
            {
                socketChannel.write(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>()
                {

                    @Override
                    public void completed(Integer result, ByteBuffer attachment)
                    {
                        //如果一次没有全部写完，继续写
                        if (attachment.hasRemaining())
                        {
                            socketChannel.write(attachment, attachment, this);
                        }
                        else
                        {
                            try
                            {
                                socketChannel.close();
                            }
                            catch (IOException e)
                            {
                                e.printStackTrace();
                            }
                        }
                    }

                    @Override
                    public void failed(Throwable exc, ByteBuffer attachment)
                    {
                    }
                });
            }
        }

        @Override
        public void failed(Throwable exc, ByteBuffer attachment)
        {
        }
    }

    public static void main(String[] args) throws IOException, ExecutionException, InterruptedException
    {
        final AsynchronousServerSocketChannel asynchronousServerSocketChannel = AsynchronousServerSocketChannel.open();
        asynchronousServerSocketChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>()
        {
            @Override
            public void completed(final AsynchronousSocketChannel clientChannel, Object attachment)
            {
                asynchronousServerSocketChannel.accept(null, this);
                final ByteBuffer buffer = ByteBuffer.wrap(new byte[16]);
                clientChannel.read(buffer, buffer, new ClientReadHandler(clientChannel));
            }

            @Override
            public void failed(Throwable exc, Object attachment)
            {
            }
        });
    }
}
```

首先先是新建一个异步服务端通道，在该通道上注册一个`accept`的回调函数。具体代码如下

```java
public void startServer() throws IOException
    {
        final AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open();
        server.bind(new InetSocketAddress(2333));
        server.accept(server, new AcceptHandler());
    }

class AcceptHandler implements CompletionHandler<AsynchronousSocketChannel, AsynchronousServerSocketChannel>
    {
        /**
         * 新创建的链接对象作为入参result被传入
         */
        @Override
        public void completed(AsynchronousSocketChannel result, AsynchronousServerSocketChannel server)
        {
            //先忽略对新连接的处理相关代码
            //继续等待下一个接入的链接
            server.accept(server, this);
        }

        @Override
        public void failed(Throwable exc, AsynchronousServerSocketChannel server)
        {
        }
    }
```

从创建代码来看，代码变的更加简单了。在`bind`方法完成后，服务端通道已经开始监听端口并等待连接了。此时可以调用`accept`方法并且传入一个回调方法对象，该回调方法会`completed`在链接创建成功后被调用，在一个异步线程中。

来看下`AcceptHandler`，当链接创建成功后，`completed`被调用，并且传入两个入参。入参一就是新创建的链接对象，入参二则是在调用`accept`方法时传入的附件对象，在这里附件对象就是服务端通道本身。显然我们可以通过`server.accept(server, this)`的方式来不断的循环，使得服务端通道始终等待新连接的产生。

上述的代码忽略了对新建链接的处理，接下来我们补上这个部分。也是使用回调的方式来进行处理。我们要实现的是从通道上读取数据，那么我们的链接回调对象也是围绕这个来建立，具体代码如下

```java
static class ClientReadHandler implements CompletionHandler<Integer, ByteBuffer>
    {
        AsynchronousSocketChannel socketChannel;
        public ClientReadHandler(AsynchronousSocketChannel socketChannel)
        {
            this.socketChannel = socketChannel;
        }
        @Override
        public void completed(Integer result, ByteBuffer buffer)
        {
            //代码①
            if (result == 16)
            {
                //数据读取完毕，准备写出
                socketChannel.write(buffer);
            }
        }
        @Override
        public void failed(Throwable exc, ByteBuffer attachment)
        {
        }
    }
```

为了使用这个回调对象，我们需要对`AcceptHandler`的`completed`进行下修改，增加对链接的读取操作。修改后如下

```java
public void completed(AsynchronousSocketChannel result, AsynchronousServerSocketChannel server)
        {
            //先忽略对新连接的处理相关代码
            //继续等待下一个接入的链接
            ByteBuffer buffer = ByteBuffer.allocate(16);
            result.read(buffer, buffer,new ClientReadHandler(result));
            server.accept(server, this);
        }
```

解释下`read`方法。三个入参，入参一是通道数据读取的目的地，入参二是传入回调对象的附件对象，入参三就是回调对象了。`read`方法被调用后，会马上返回。而一旦内核将数据填充到 buffer 完毕，则会依靠异步线程触发`ClientReadHandler`的`completed`方法。

来说下`ClientReadHandler`的`completed`方法，也就是代码①处。该方法的第一个入参是本次读取成功的字节数，第二个入参就是附件对象了。在这里我们假设一次读取完毕了所有 16 字节的数据，那么接下来就是执行写出动作了。写出方法一样也可以接受回调对象，回调对象可以编写如下

```java
class WriteHandler implements CompletionHandler<Integer, ByteBuffer>
    {
        AsynchronousSocketChannel socketChannel;
        public WriteHandler(AsynchronousSocketChannel socketChannel)
        {
            this.socketChannel = socketChannel;
        }
        @Override
        public void completed(Integer result, ByteBuffer attachment)
        {
            //如果一次没有全部写完，继续写
            if (attachment.hasRemaining())
            {
                socketChannel.write(attachment, attachment, this);
            }
            else
            {
                try
                {
                    socketChannel.close();
                }
                catch (IOException e)
                {
                    e.printStackTrace();
                }
            }
        }
        @Override
        public void failed(Throwable exc, ByteBuffer attachment)
        {
        }
    }
```

为了使用这个回调对象，我们需要对`ClientReadHandler`的`completed`方法进行修改，让它使用这个回调对象，修改后如下

```java
public void completed(Integer result, ByteBuffer buffer)
        {
            if (result == 16)
            {
                //数据读取完毕，准备写出
                socketChannel.write(buffer, buffer, new WriteHandler(socketChannel));
            }
        }
```

解释下这个`write`方法。第一个入参是承载需要写出的数据的`ByteBuffer`，第二个入参是传递给回调对象的附件对象，第三个入参是回调对象。当通道上的数据被写出时，`WriteHandler`的`completed`被调用。

来看下`WriteHandler`的`completed`方法，在这个实现中我们处理了没能一次写出全部数据的情况，处理方法也很简单，再次执行写出并发并且注册回调函数即可。由于都是在异步线程中执行，因此程序执行很快，并且不会阻塞主线程。

经过上面的分析，我们现在来看下在 AIO 上实现了 echo 服务器的代码全貌

```java
public class HelloWorld
{
    public static void main(String[] args) throws IOException
    {
        new HelloWorld().startServer();
    }

    public void startServer() throws IOException
    {
        final AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open();
        server.bind(new InetSocketAddress(2333));
        server.accept(server, new AcceptHandler());
    }

    class AcceptHandler implements CompletionHandler<AsynchronousSocketChannel, AsynchronousServerSocketChannel>
    {

        /**
         * 新创建的链接对象作为入参result被传入
         *
         * @param result
         * @param server
         */
        @Override
        public void completed(AsynchronousSocketChannel result, AsynchronousServerSocketChannel server)
        {
            //先忽略对新连接的处理相关代码
            //继续等待下一个接入的链接
            ByteBuffer buffer = ByteBuffer.allocate(16);
            result.read(buffer, buffer, new ClientReadHandler(result));
            server.accept(server, this);
        }

        @Override
        public void failed(Throwable exc, AsynchronousServerSocketChannel server)
        {
        }
    }

    class ClientReadHandler implements CompletionHandler<Integer, ByteBuffer>
    {
        AsynchronousSocketChannel socketChannel;

        public ClientReadHandler(AsynchronousSocketChannel socketChannel)
        {
            this.socketChannel = socketChannel;
        }

        @Override
        public void completed(Integer result, ByteBuffer buffer)
        {
            if (result == 16)
            {
                //数据读取完毕，准备写出
                socketChannel.write(buffer, buffer, new WriteHandler(socketChannel));
            }
        }

        @Override
        public void failed(Throwable exc, ByteBuffer attachment)
        {
        }
    }

    class WriteHandler implements CompletionHandler<Integer, ByteBuffer>
    {
        AsynchronousSocketChannel socketChannel;

        public WriteHandler(AsynchronousSocketChannel socketChannel)
        {
            this.socketChannel = socketChannel;
        }

        @Override
        public void completed(Integer result, ByteBuffer attachment)
        {
            //如果一次没有全部写完，继续写
            if (attachment.hasRemaining())
            {
                socketChannel.write(attachment, attachment, this);
            }
            else
            {
                try
                {
                    socketChannel.close();
                }
                catch (IOException e)
                {
                    e.printStackTrace();
                }
            }
        }

        @Override
        public void failed(Throwable exc, ByteBuffer attachment)
        {
        }
    }
}
```

纵观整个代码，可以看到，全部流程都是异步的，都可以在回调方法中完成。并且和数据相关的操作都由系统本身完成了，业务代码不再需要处理数据读写。这种编程模型好理解，也好操作。概念上也少于 NIO 提供的。Netty 曾经有一个 5.0 预览版，但是由于 AIO 和 NIO 编程模型差异很大，提升了 Netty 的复杂度，而性能上因为底层都是相同的，没有带来显著提升，最终 5.0 被放弃，Netty 仍然只是使用 NIO。同时 Java 的 AIO 并不提供 UDP 的支持。

## 综述

随着 Java 版本的提升，对网络模型的支持也越来越丰富全面。程序性能和编码效率都在提升。不过由于在 NIO 时代出现了 Netty 这个性能强悍，功能稳定的生产级框架。在 AIO 出现后，市场上尚未出现大范围普及和接受的 AIO 框架。各大公司，各种 RPC 框架，底层基本都是 Netty，也有一些基于 NIO 自己实现网络层的比如 Undertow。AIO 则鲜少见到。

## 总结与思考

本篇文章带大家回顾了随着 JDK 的版本演进，其 IO 编程模型的变化。针对不同 JDK 的不同版本，不同的 IO 模型下，使用 Demo 具体分析了线程模型是如何反应在编码方式上的。随着 IO 模型的变化，编码的复杂度和理解的概念多寡也是一直都在变化。不同的 IO 模型各自有其适应场景，不是一个非此即彼的关系。读者可以根据自己的场景，技术情况，团队情况，选择合适于自己的编程模式。

当然，本篇专栏的重点是在 NIO 上，因此下篇文章会重点分析在 NIO 中我们所使用到的组件。帮助读者彻底掌握使用 Java NIO API 进行网络编程。