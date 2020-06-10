# 2/31Unix 的五种 IO 模型解析

## Unix 的五种 IO 模型解析

## 前言

操作系统为了保护自身的稳定，会将内存空间划分为内核空间和用户空间。当我们需要通过 TCP 将数据发送出去时，在应用程序中实际上执行了将数据从用户空间拷贝至内核空间，再由内核进行实际的发送动作；而从 TCP 读取数据时则反过来，等待内核将数据准备好，再从内核空间拷贝至用户空间，应用数据才能处理。

针对在两个阶段上不同的操作，Unix 定义了 5 种 IO 模型，分别是：

- 阻塞式 IO
- 非阻塞式 IO
- IO 复用
- 信号驱动 IO
- 异步 IO

下面来逐一介绍。

## 模型介绍

### 阻塞式IO

阻塞式IO 是最流行的 IO 模型了，在客户端上特别常见，因为其编写难度最低，也最好理解。其模型如下图所示：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191017101806.png)

该图中，我们将`recvfrom`看成是一个系统调用，该调用会从用户空间切换到内核空间，直到功能完成后再切换回来。

从图中可以看到，调用返回成功或者发生错误之前，应用程序都在阻塞在方法的调用上。当方法调用成功返回后，应用程序才能开始处理数据。

这种模型在 Java 中是最古老的也最常见的 `Socket` API 了。举个例子如下

```java
public static void main(String[] args) throws IOException
    {
        Socket socket = new Socket();
        socket.connect(InetSocketAddress.createUnresolved("192.168.31.80", 4591));
        InputStream inputStream = socket.getInputStream();
        byte[]      content     = new byte[128];
        int         bytesOfRead = inputStream.read(content);
    }
```

上述示例代码中首先创建一个客户端`socket`实例，并且尝试连接一个远端的服务器地址。在连接成功后则获取输入流，并且尝试读取数据。在输入流上的read调用会阻塞直到有数据被读取成功或者连接发生了异常。`read`的调用就会经历上述将程序阻塞，然后内核等待数据准备后，将数据从内核空间复制到用户空间，也就是入参传递进来的二进制数组中。需要注意，实际读取的字节数可能小于数组的长度，方法的返回值正是实际读取的字节数。

`Socket`系列的 API 在 JDK1.0 的时候就存在了，是最古老的网络编程 API，那个时候也支持 阻塞IO 模型。

### 非阻塞式IO

允许将一个套接字设置为非阻塞。当设置为非阻塞时，是在通知内核：如果一个操作需要将当前的调用线程阻塞住才能完成时，不采用阻塞的方式，而是返回一个错误信息。其模型如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191017101850.png)

可以看到，在内核没有数据时，尝试对数据的读取不会导致线程阻塞，而是快速的返回一个错误。直到内核中收到数据时，尝试读取，就会将数据从内核复制到用户空间，进行操作。

可以看到，在非阻塞模式下，要感知是否有数据可以读取，需要不断的轮训，这么做往往会耗费大量的 CPU。所以这种模式不是很常见。

Java在1.4版本中提供新的 NIO 包，其中的`SocketChannel`提供了对非阻塞 IO 的支持。示例代码如下

```java
public static void main(String[] args) throws IOException
    {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);
        socketChannel.connect(InetSocketAddress.createUnresolved("192.168.31.80", 4591));
        ByteBuffer buffer = ByteBuffer.allocate(128);
        while (socketChannel.read(buffer) == 0)
        {
            ;
        }
    }
```

一个`SocketChannel`实例就类似从前的一个`Socket`对象。

首先是通过`SocketChannel.open()`调用新建了一个`SocketChannel`实例，默认情况下，新建的socket实例都是阻塞模式的，通过`java.nio.channels.spi.AbstractSelectableChannel#configureBlocking`调用将其设置为非阻塞模式，然后连接远程服务端。

`java.nio.channels.SocketChannel`使用`java.nio.ByteBuffer`作为数据读写的容器，这里先不细说，可以简单的将`ByteBuffer`看成是一个内部持有二进制数据的包装类。

调用方法`java.nio.channels.SocketChannel#read(java.nio.ByteBuffer)`时会将内核中已经准备好的数据复制到`ByteBuffer`中。但是如果内核中此时并没有数据（或者说socket的读取缓冲区没有数据），则方法会立刻返回，并不会阻塞住。这也就对应了上图中，在内核等待数据的阶段（socket的读取缓冲区没有数据），读取调用时会立刻返回错误的。只不过在Java中，返回的错误在上层处理为返回一个读取为0的结果。

### IO复用

IO复用指的应用程序阻塞在系统提供的两个调用`select`或`poll`上。当应用程序关注的套接字存在可读情况（也就是内核收到数据了），`select`或`poll`的调用被返回。此时应用程序可以通过`recvfrom`调用完成数据从内核空间到用户空间的复制，进而进行处理。具体的模型如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191017101939.png)

可以看到，和 阻塞式IO 相比，都需要等待，并不存在优势。而且由于需要2次系统调用，其实还稍有劣势。但是IO复用的优点在于，其`select`调用，可以同时关注多个套接字，在规模上提升了处理能力。

IO复用的模型支持一样也是在JDK1.4中的 NIO 包提供了支持。可以参看如下示例代码：

```java
public static void main(String[] args) throws IOException
    {
        /**创建2个Socket通道**/
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);
        socketChannel.connect(InetSocketAddress.createUnresolved("192.168.31.80", 4591));
        SocketChannel socketChannel2 = SocketChannel.open();
        socketChannel2.configureBlocking(false);
        socketChannel2.connect(InetSocketAddress.createUnresolved("192.168.31.80", 4591));
        /**创建2个Socket通道**/
        /**创建一个选择器，并且两个通道在这个选择器上注册了读取关注**/
        Selector selector = Selector.open();
        socketChannel.register(selector, SelectionKey.OP_READ);
        socketChannel2.register(selector, SelectionKey.OP_READ);
        /**创建一个选择器，并且两个通道在这个选择器上注册了读取关注**/
        ByteBuffer buffer = ByteBuffer.wrap(new byte[128]);
        //选择器可以同时检查所有在其上注册的通道，一旦哪个通道有关注事件发生，select调用就会返回，否则一直阻塞
        selector.select();
        Set<SelectionKey>      selectionKeys = selector.selectedKeys();
        Iterator<SelectionKey> iterator      = selectionKeys.iterator();
        while (iterator.hasNext())
        {
            SelectionKey  selectionKey = iterator.next();
            SocketChannel channel      = (SocketChannel) selectionKey.channel();
            channel.read(buffer);
            iterator.remove();
        }
    }
```

代码一开始，首先是新建了2个客户端通道，连接到服务端上。接着创建了一个选择器`Selector`。选择器就是 Java 中实现 IO 复用的关键。选择器允许通道将自身的关注事件注册到选择器上。完成注册后，应用程序调用`java.nio.channels.Selector#select()`方法，程序进入阻塞等待直到注册在选择器上的通道中发生其关注的事件，则`select`调用会即可返回。然后就可以从选择器中获取刚才被选中的键。从键中可以获取对应的通道对象，然后就可以在通道对象上执行读取动作了。

结合IO复用模型，可以看到，`select`调用的阻塞阶段，就是内核在等待数据的阶段。一旦有了数据，内核等待结束，`select`调用也就返回了。

### 信号驱动IO

与非阻塞IO类似，其在数据等待阶段并不阻塞，但是原理不同。信号驱动IO是在套接字上注册了一个信号调用方法。这个注册动作会将内核发出一个请求，在套接字的收到数据时内核会给进程发出一个`sigio`信号。该注册调用很快返回，因此应用程序可以转去处理别的任务。当内核准备好数据后，就给进程发出了信号。进程就可以通过`recvfrom`调用来读取数据。其模型如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191017102604.png)

这种模型的优点就是在数据包到达之前，进程不会被阻塞。而且采用通知的方式也避免了轮训带来的损耗。

这种模型在Java中并没有对应的实现。

### 异步IO

异步IO的实现一般是通过系统调用，向内核注册了一个套接字的读取动作。这个调用一般包含了：缓存区指针，缓存区大小，偏移量、操作完成时的通知方式。该注册动作是即刻返回的，并且在整个IO的等待期间，进程都不会被阻塞。当内核收到数据，并且将数据从内核空间复制到用户空间完成后，依据注册时提供的通知方式去通知进程。其模型如下：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191017102721.png)

与信号驱动 IO 相比，最大的不同在于信号驱动 IO 是内核通知应用程序可以读取数据了；而 异步IO 是内核通知应用程序数据已经读取完毕了。

Java 在 1.7 版本引入对 异步IO 的支持，可以看如下的例子：

```java
public class MainDemo
{
    public static void main(String[] args) throws IOException, ExecutionException, InterruptedException
    {
        final AsynchronousSocketChannel asynchronousSocketChannel = AsynchronousSocketChannel.open();
        Future<Void>                    connect                   = asynchronousSocketChannel.connect(InetSocketAddress.createUnresolved("192.168.31.80", 3456));
        connect.get();
        ByteBuffer buffer = ByteBuffer.wrap(new byte[128]);
        asynchronousSocketChannel.read(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>()
        {
            @Override
            public void completed(Integer result, ByteBuffer buffer)
            {
                //当读取到数据，流中止，或者读取超时到达时均会触发回调
                if (result > 0)
                {
                    //result代表着本次读取的数据，代码执行到这里意味着数据已经被放入buffer了
                    processWithBuffer(buffer);
                }
                else if (result == -1)
                {
                    //流中止，没有其他操作
                }
                else{
                    asynchronousSocketChannel.read(buffer, buffer, this);
                }
            }

            private void processWithBuffer(ByteBuffer buffer)
            {
            }

            @Override
            public void failed(Throwable exc, ByteBuffer attachment)
            {
            }
        });
    }
}
```

代码看上去和IO复用时更简单了。

首先是创建一个异步的 Socket 通道，注意，这里和 NIO 最大的区别就在于创建的是异步Socket通道，而 NIO 创建的属于同步通道。

执行`connect`方法尝试连接远程，此时方法会返回一个`future`，这意味着该接口是非阻塞的。实际上`connect`动作也是可以传入回调方法，将连接结果在回调方法中进行传递的。这里为了简化例子，就直接使用`future`了。

连接成功后开始在通道上进行读取动作。这里就是和 NIO 中最大的不同。读取的时候需要传入一个回调方法。当数据读取成功时回调方法会被调用，并且当回调方法被调用时读取的数据已经被填入了`ByteBuffer`。

主线程在调用读取方法完成后不会被阻塞，可以去执行别的任务。可以看到在整个过程都不需要用户线程参与，内核完成了所有的工作。

## 同步异步对比

在网络上经常会有同步异步的争论，实际上根据 POSIX 的定义，这两个的区别是非常清晰的。

- **同步**：同步操作导致进程阻塞，直到 IO 操作完成时。
- **异步**：异步操作不导致进程阻塞。

来看下五种 IO 模型的对比，如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191017104715.png)

可以看到，根据定义，前 4 种模型，在数据的读取阶段，全部都是阻塞的，因此是同步IO。而异步IO模型在整个IO过程中都不阻塞，因此是异步IO。

## 总结与思考

本篇文章论述了在 Unix 中的 5 种 IO 模式的联系和区别，并且给出每种 IO 模式下对应的 Java 相关的 Demo 代码。相信读者对 Java 如何对网络模型进行支持有了清晰的认识。同时，也能感受到，不同的网络模型有不同的适用场景。不同的场景下，不同的开发模型的复杂度相差很大，在代码的可读性上也带来了不同的挑战。