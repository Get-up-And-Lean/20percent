# Java 网络编程基础篇

### 一、前言

网络通讯在系统交互中是必不可少的一部分，无论是面试还是工作中都是绕不过去的一部分，本节我们来谈谈 Java 网络编程中的一些知识，本 chat 内容如下：

- 网络通讯基础知识，剖析网络通讯的本质和需要注意的点
- 使用 Java BIO 阻塞套接字 实现简单 TCP 网络通讯
- 使用 Java NIO 非阻塞套接字实现简单非阻塞 TCP 网络通讯
- JavaIO 模型与 Java NIO 中 ByteBuffer

### 二、 网络通讯基础知识

网络通讯的本质用一句话来说是处于两个主机上的两个进程之间进行通讯，如下图：

![enter image description here](https://images.gitbook.cn/7a5953e0-cc73-11e8-a06e-69818d785e22)

如上图主机 A 和 B 上面有好多进程，比如 QQ 进程、手淘进程、微信进程、浏览器进程等等。

这里假如进程 1 为微信进程，在应用层微信肯定自己约定了自己的应用成层协议（比如约定协议包为协议头 + 消息内容）。

那么当主机 A 上的微信用户给主机 B 上的微信用户发送消息时候，发送的消息内容要首先经过程序把要发送的数据转换为自己的应用层协议格式的数据，把消息转换为应用层包是在用户程序代码里面做的。做完这些后网卡驱动程序会接着把应用层包转换为运输层的 TCP 包或者 UDP 包，在运输层会把应用层包作为数据，然后在数据包前添加协议头组成运输层包，协议头里面会包含目的地址的网络端口号。

然后运输层的包会被作为数据包的数据部分，然后在数据部分前面添加 IP 层的头部部分，头部里面会含有当前主机的 IP 和目的地址的 IP 组成 IP 层的包。

IP 层的包最后会被转换为数据链路层的数据包帧，在帧的头部会新增当前主机网卡 MAC 地址和下一跳的主机的 MAC 地址，注意这里不是目的主机的 MAC 地址，因为在源主机和目的主机之间很可能有好多路由器，这时候下一跳的 MAC 地址就是当前主机连接的路由器的 MAC 地址。

![enter image description here](https://images.gitbook.cn/d6bb13d0-cc73-11e8-a06e-69818d785e22)

最后数据链路层的数据帧会被转换会在物理层通过二进制流通过网络传递到网络上，网络流经过路由器时候路由器会首先把二进制流转换为数据链路层的数据帧，然后转换为网络层的 IP 数据包，然后读取目的地址的 IP，然后查找路由表进行路由选择，然后把 IP 数据包重新转换为数据链路层的帧。

这时候数据帧里面的目的 MAC 地址是路由选择的主机的 MAC 地址，然后把数据帧通过物理层透明的把二进制流传递到下一站，如果下一站就是目的 IP 所在主机，则网卡驱动会吧二进制流依次转换为数据链路层数据帧、IP 包、传输层 TCP 包或者 UDP 包，最后交给应用程序进程进行处理，应用程序转换包为具体数据然后进行处理。

这里需要注意的是网络层只是能确定目的主机，还记得 IP 包里记录了目的主机的 IP，但是一个主机上可能会有多个进程，那么具体把数据交给那个进程进行处理？这个就是运输层的作用，运输层包里面记录的端口号，运输层会把数据交给具体端口号的进程。即网络通讯的 socket 地址实际是 IP + 端口号。

另外整个通讯过程中传输的是二进制流，没有业务数据包的概念（比如我发送了一个业务请求包），当发送方发了多个业务包后，接受方的运输层并不知道每个包的边界，它只是把接受到的数据传输给应用层，所以应用层要自己根据约定好的协议解析二进制流为业务所需要的包，即半包粘包问题，可以参考：[Java NIO 框架 Netty 之美：粘包与半包问题](https://gitbook.cn/gitchat/activity/5b13e6a675742e21d6d14ea4)。

另外由于网络传输的都是二进制流，所以在发送方进程需要把要发送的数据（比如本文字符串）进行序列化为二进制流，而接受方应用层需要根据对应的反序列化把二进制流转换为具体的数据（比如文本字符串），即序列化与反序列化问题。

另外路由器只有网络层、数据链路层、物理层三层结构。

一次网络通讯看此很复杂，中间需要做好多协议包的转换，但是好在除了应用层协议部分是需要应用程序自己来做，其他下层都是由网卡驱动程序来完成的，这些对应用程序是透明的。

### 三、使用 Java BIO 阻塞套接字 实现简单 TCP 网络通讯

下面我们看看如何实现 Java BIO 套接字实现简单 TCP 通讯程序。

- 客户端程序

```
public class BioClient {

    public static void main(String[] args) {

        Socket socket = null;
        BufferedReader bufferedReader = null;
        PrintWriter printWriter = null;

        try {
            //1 发起链接
            socket = new Socket("127.0.0.1", 7001);
            //2 转换socket输入流
            bufferedReader = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            //3 转换socket输出流，并写入
            printWriter = new PrintWriter(socket.getOutputStream());
            printWriter.write("hello server ,im a client\n");
            printWriter.flush();
            //4 从socket读取
            String response = bufferedReader.readLine();
            System.out.println(response);


        }catch(Exception e) {
            e.printStackTrace();
        }finally {
            //5 关闭流和套接字
            if(null != bufferedReader) {
                try {
                    bufferedReader.close();
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }

            if(null != printWriter) {
                printWriter.close();
            }

            if(null != socket) {
                try {
                    socket.close();
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
        }
    }

}
```

其中代码 1 创建了一个客户端套接字，其链接的远端服务器 IP 为 127.0.0.1，远端服务监听端口为 7001。下面我们看 Socket 构造函数代码：

```
    public Socket(String host, int port)
        throws UnknownHostException, IOException
    {
        this(host != null ? new InetSocketAddress(host, port) :
             new InetSocketAddress(InetAddress.getByName(null), port),
             (SocketAddress) null, true);
    }
    private Socket(SocketAddress address, SocketAddress localAddr,
                   boolean stream) throws IOException {
       ...
        try {
            createImpl(stream);
            //绑定本地地址
            if (localAddr != null)
                bind(localAddr);
            //调用connect发起远程链接
            if (address != null)
                connect(address);
        } catch (IOException e) {
            close();
            throw e;
        }
    }
```

如上代码可知 socket 构造函数内部最终还是调用 connect 方法。

这里需要注意的是当调用 connect 方法时候，connect 方法会阻塞当前调用线程，直到完成了与服务端的 TCP 三次握手才返回，也就是说 connect 方法是阻塞的。

另外代码 2 转换 socket 输入流为缓冲流，代码 3 转换 socket 输出流，并写入内容到输出流，需要注意写入操作也是阻塞的，当写入内容到 socket 缓冲区后才返回。

代码 4 从输入流读取一行内容，该方法也是阻塞的，如果服务端没有写回内容到客户端，则这里一直阻塞。

- 服务端代码

```
package com.network.bio.BIO;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.security.AccessControlContext;

public class BioServer {

    public static void main(String arg[]) {
        ServerSocket serverSocket = null;

        try {
            //1 创建服务端监听套接字
            serverSocket = new ServerSocket(7001);

            System.out.println("server is started ");
            //2 循环监控客户端的链接
            while(true) {
                //2.1获取客户端的链接套接字
                final Socket acceptSocket  = serverSocket.accept();
                System.out.println("server is accept client:  " + acceptSocket.getRemoteSocketAddress().toString() );
                //2.2 开启线程异步处理接受套机子
                new Thread(new Runnable() {

                    @Override
                    public void run() {
                        BufferedReader bufferedReader = null;
                        PrintWriter printWriter = null;

                        try {
                            bufferedReader = new BufferedReader(new InputStreamReader(acceptSocket.getInputStream()));
                            printWriter = new PrintWriter(acceptSocket.getOutputStream());

                            String receive = bufferedReader.readLine();
                            System.out.println(receive);

                            printWriter.write("hello client ,im server");
                            printWriter.flush();
                        } catch (IOException e) {
                            // TODO Auto-generated catch block
                            e.printStackTrace();
                        }finally {
                            if(null != bufferedReader) {
                                try {
                                    bufferedReader.close();
                                } catch (IOException e) {
                                    // TODO Auto-generated catch block
                                    e.printStackTrace();
                                }
                            }

                            if(null != printWriter) {
                                printWriter.close();
                            }

                            if(null != acceptSocket) {
                                try {
                                    acceptSocket.close();
                                } catch (IOException e) {
                                    e.printStackTrace();
                                }
                            }
                        }

                    }
                }).start();
            }

        } catch (Exception e) {
            e.printStackTrace();

        }finally {
            //3.关闭服务监听套接字
            if(null != serverSocket) {
                try {
                    serverSocket.close();
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
        }
    }
}
```

其中代码 1 创建服务端监听套接字，监听端口为 7001，其构造函数代码如下：

```
 public ServerSocket(int port, int backlog, InetAddress bindAddr) throws IOException {
        setImpl();
        if (port < 0 || port > 0xFFFF)
            throw new IllegalArgumentException(
                       "Port value out of range: " + port);
        if (backlog < 1)
          backlog = 50;
        try {
           //绑定端口
            bind(new InetSocketAddress(bindAddr, port), backlog);
        } catch(SecurityException e) {
            close();
            throw e;
        } catch(IOException e) {
            close();
            throw e;
        }
    }
```

其中代码 2 循环监听客户端的链接，其中代码 2.1 是阻塞的， 当接受到一个客户端链接后，才会返回，然后会开启一个线程来处理链接套接字，然后监听线程继续阻塞直到来了新的链接。

这里服务端每接受到一个新的链接后，都会新建一个线程来进行处理，处理完毕后要销毁线程，而线程的创建和销毁是需要开销的，并且这里不对线程个数进行线程，而线程是系统很宝贵的资源，无限制的新开线程对系统性能影响很大。其实这里可以使用线程池+队列方式来提高线程的复用性。

### 四、使用 Java NIO 非阻塞套接字实现简单非阻塞 TCP 网络通讯

本节我们使用 JDK 中原生 NIO API 来创建一个简单的 TCP 客户端与服务器交互的网络程序。

#### 4.1 客户端程序

这个客户端功能是当客户端连接到服务端后，给服务器发送一个 Hello，然后从套接字里面读取服务器端返回的内容并打印，具体代码如下：

```
public class NioClient {

    // (1)创建发送和接受缓冲区
    private static ByteBuffer sendbuffer = ByteBuffer.allocate(1024);
    private static ByteBuffer receivebuffer = ByteBuffer.allocate(1024);

    public static void main(String[] args) throws IOException {
        // (2) 获取一个客户端socket通道
        SocketChannel socketChannel = SocketChannel.open();
        // （3）设置socket为非阻塞方式
        socketChannel.configureBlocking(false);
        // （4）获取一个选择器
        Selector selector = Selector.open();
        // （5）注册客户端socket到选择器
        SelectionKey selectionKey = socketChannel.register(selector, 0);
        // （6）发起连接
        boolean isConnected = socketChannel.connect(new InetSocketAddress("127.0.0.1", 7001));

        // (7)如果连接没有马上建立成功，则设置对链接完成事件感兴趣
        if (!isConnected) {
            selectionKey.interestOps(SelectionKey.OP_CONNECT);

        }

        int num = 0;
        while (true) {

            // (8) 选择已经就绪的网络IO操作，阻塞方法
            int selectCount = selector.select();
            System.out.println(num + "selectCount:" + selectCount);
            // （9）返回已经就绪的通道的事件
            Set<SelectionKey> selectionKeys = selector.selectedKeys();

            //(10)处理所有就绪事件
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            SocketChannel client;
            while (iterator.hasNext()) {
                //(10.1)获取一个事件，并从集合移除
                selectionKey = iterator.next();
                iterator.remove();
                //(10.2)获取事件类型
                int readyOps = selectionKey.readyOps();
                //(10.3)判断是否是OP_CONNECT事件
                if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                    //（10.3.1）等待客户端socket完成与服务器端的链接
                    client = (SocketChannel) selectionKey.channel();
                    if (!client.finishConnect()) {
                        throw new Error();

                    }

                    System.out.println("--- client already connected----");

                    //(10.3.2)设置要发送给服务端的数据
                    sendbuffer.clear();
                    sendbuffer.put("hello server,im a client".getBytes());
                    sendbuffer.flip();
                    //(10.3.3)写入输入。
                    client.write(sendbuffer);
                    //(10.3.4)设置感兴趣事件，读事件
                    selectionKey.interestOps(SelectionKey.OP_READ);

                //(10.4)判断是否是OP_READ事件
                } else if ((readyOps & SelectionKey.OP_READ) != 0) {
                    client = (SocketChannel) selectionKey.channel();
                    //（10.4.1）读取数据并打印
                    receivebuffer.clear();
                    int count = client.read(receivebuffer);
                    if (count > 0) {
                        String temp = new String(receivebuffer.array(), 0, count);
                        System.out.println(num++ + "receive from server:" + temp);
                    }

                }
            }
        }
    }
```

- 代码（1）分别创建了一个发送和接受 buffer，用来发送数据时候 byte 化内容和接受数据。
- 代码（2）获取一个客户端套接字通道。
- 代码（3）设置 socket 通道为非阻塞模式，默认是阻塞模式。
- 代码（4）（5）获取一个选择器，然后注册客户端套接字通道到该选择器，并且设置感兴趣的事情为 0，就是不对任何事件感兴趣。
- 代码（6）（7）调用套接字通道的 connect 方法，连接服务器（服务器套接字地址为 `127.0.0.1:7001`），由于步骤（3）设置了为非阻塞，所以步骤（6）马上会返回。代码（7）判断连接是否已经完成，如果没有，则设置选择器去监听 `OP_CONNECT` 事件，也就是指明对该事件感兴趣。 然后进入 while 循环进行事件处理，其中代码（8）选择已经就绪的网络IO事件，如果当前没有就绪的则阻塞当前线程。当有就绪事件后，会返回获取的事件个数，会执行代码（9）具体取出来具体事件列表。
- 代码（10）循环处理所有就绪事件，代码（10.1）迭代出一个事件 key，然后从集合中删除，代码（10.2）获取事件 key 感兴趣的标志，代码（10.3）则看兴趣集合里面是否有 `OP_CONNECT`，如果有则说明有 `OP_CONNECT` 事件已经就绪了，那么执行步骤（10.3.1）等待客户端与服务端完成三次握手，然后步骤（10.3.2）（10.3.3）写入 `hello server,im a client` 到服务器端。然后代码（10.3.4）设置对 `OP_READ` 事件感兴趣。
- 代码（10.4）则看如果当前事件 key 是 `OP_READ` 事件，说明服务器发来的数据已经在接受 buffer 就绪了，客户端可以去具体拿出来了，然后代码（10.4.1）从客户端套接字里面读取数据并打印。

> 注：设置套接字为非阻塞后，connect 方法会马上返回的，所以需要根据结果判断是否为链接建立 OK 了，如果没有成功，则需要设置对该套接字的 `op_connect` 事件感兴趣，在这个事件到来的时候还需要调用 finishConnect 方法来具体完成与服务器的链接，在 finishConnect 返回 true 后说明链接已经建立完成了，则这时候可以使用套接字通道发送数据到服务器，并且设置堆该套接字的 `op_read` 事件感兴趣，从而可以监听到服务端发来的数据，并进行处理。

#### 4.2 服务端程序

服务端程序代码如下：

```
public class NioServer {

    // (1) 缓冲区
    private ByteBuffer sendbuffer = ByteBuffer.allocate(1024);
    private ByteBuffer receivebuffer = ByteBuffer.allocate(1024);
    private Selector selector;

    public NioServer(int port) throws IOException {
        // (2)获取一个服务器套接字通道
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        // （3）socket为非阻塞
        serverSocketChannel.configureBlocking(false);
        // （4）获取与该通道关联的服务端套接字
        ServerSocket serverSocket = serverSocketChannel.socket();
        // （5）绑定服务端地址
        serverSocket.bind(new InetSocketAddress(port));
        // （6）获取一个选择器
        selector = Selector.open();
        // （7）注册通道到选择器，选择对OP_ACCEPT事件感兴趣
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        System.out.println("----Server Started----");

        // (8)处理事件
        int num = 0;
        while (true) {
            // (8.1)获取就绪的事件集合
            int selectKeyCount = selector.select();
            System.out.println(num++ + "selectCount:" + selectKeyCount);

            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            // (8.2)处理就绪事件
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while (iterator.hasNext()) {
                SelectionKey selectionKey = iterator.next();
                iterator.remove();
                processSelectedKey(selectionKey);
            }
        }
    }

    private void processSelectedKey(SelectionKey selectionKey) throws IOException {

        SocketChannel client = null;
        // (8.2.1)客户端完成与服务器三次握手
        if (selectionKey.isAcceptable()) {
            // (8.2.1.1)获取完成三次握手的链接套接字
            ServerSocketChannel server = (ServerSocketChannel) selectionKey.channel();
            client = server.accept();
            if (null == client) {
                return;
            }
            System.out.println("--- accepted client---");

            // （8.2.1.2）该套接字为非阻塞模式
            client.configureBlocking(false);
            // （8.2.1.3）注册该套接字到选择器，对OP_READ事件感兴趣
            client.register(selector, SelectionKey.OP_READ);

            // (8.2.2)为读取事件
        } else if (selectionKey.isReadable()) {
            // (8.2.2.1) 读取数据
            client = (SocketChannel) selectionKey.channel();
            receivebuffer.clear();
            int count = client.read(receivebuffer);
            if (count > 0) {
                String receiveContext = new String(receivebuffer.array(), 0, count);
                System.out.println("receive client info:" + receiveContext);
            }
            // (8.2.2.2)发送数据到client
            sendbuffer.clear();
            client = (SocketChannel) selectionKey.channel();
            String sendContent = "hello client ,im server";
            sendbuffer.put(sendContent.getBytes());
            sendbuffer.flip();
            client.write(sendbuffer);
            System.out.println("send info to client:" + sendContent);

        }

    }


    public static void main(String[] args) throws IOException {
        int port = 7001;
        NioServer server = new NioServer(port);
    }
}
```

- 代码（1）分别创建了一个发送和接受 buffer，用来发送数据时候 byte 化内容，和接受数据。
- 代码（2）获取一个服务端监听套接字通道。
- 代码（3）设置 socket 通道为非阻塞模式，默认是阻塞模式。
- 代码（4）获取与该通道关联的服务端套接字
- 代码（5）绑定服务端套接字监听端口为 `7001`
- 代码（6）（7） 获取一个选择器，并注册通道到选择器，选择对 `OP_ACCEPT` 事件感兴趣，到这里服务端已经开始监听客户端链接了。
- 代码（8） 具体处理事件，8.1 选择当前就绪的事件，8.2 遍历所有就绪事件，顺序调用 `processSelectedKey` 进行处理。
- 代码（8.2.1） 当前事件key对应的 `OP_ACCEPT` 事件，则执行代码 8.2.1.1 获取已经完成三次握手的链接套接字，并通过代码 8.2.1.2 设置该链接套接字为非阻塞模式，通过代码 8.2.1.3 注册该链接套接字到选择器，并设置对对 `OP_READ` 事件感兴趣。
- 代码（8.2.2） 判断如果当前事件 key 为 `OP_READ` 则通过代码 8.2.2.1 链接套接字里面获取客户端发来的数据，通过代码 8.2.2.2 发送数据到客户端。

> 注：在这个例子里面监听套接字 serverSocket 和 serverSocket 接受到的所有链接套接字都注册到了同一个选择器上，其中 `processSelectedKey` 里面 8.2.1 是用来处理 serverSocket 接受的新链接的，8.2.2 是用来处理链接套接字的读写的。

到这里服务端和客户端就搭建好了，首先启动服务器，然后运行客户端，会输入如下：

```
0selectCount:1    
--- client already connected----  
1selectCount:1
2receive from server:hello client ,im server
```

这时候服务器的输出结果为：

```
----Server Started----
0selectCount:1
--- accepted client---
1selectCount:1
receive client info:hello server,im a client
send info to client:hello client ,im server
```

简单分析下结果：

服务器端启动后，会先输出

```
----Server Started----
```

客户端启动后去链接服务器端，三次握手完毕后，服务器会获取 `op_accept` 事件，会通过 accept 获取链接套接字，所以输出了：

```
0selectCount:1
--- accepted client---
```

然后客户端接受到三次握手信息后，获取到了 `op_connect` 事件，所以输出：

```
0selectCount:1    
--- client already connected----  
```

然后发送数据到服务器端。

服务端收到数据后，选择器会选择出 `op_read` 事件，读取客户端发来的内容，并发送回执到客户端：

```
1selectCount:1
receive client info:hello server,im a client
send info to client:hello client ,im server
```

客户端收到服务器端回执后，选择器会选择出 `op_read` 事件，所以客户端会读取服务器端发来的内容，所以输出：

```
1selectCount:1
2receive from server:hello client ,im server
```

### 五、Java IO 模型与 Java NIO 中 ByteBuffer

#### 5.1 Java IO 模型

![enter image description here](https://images.gitbook.cn/23ae1010-cc75-11e8-a06e-69818d785e22)

如上图当网络应用进程向 socket 写入数据时候，首先需要在应用程序内申请一个写 buffer，然后把数据写入到写 buffer，然后应用程序的执行会用用户态切换到核心态，核心态程序把应用程序写 buffer 里面的数据拷贝到操作系统层面的写缓存里面。

当应用程序读取数据时候是需要把操作系统层面的读 buffer 里面的数据拷贝到应用程序层面的读 buffer 里面。一般情况下应用程序层面的 buffer 都是从堆空间里面申请的，这就需要在用户态和核心态之间数据传输时候进行一次数据 copy。

这是因为核心态是不能直接应用程序堆内存的，必须转换为直接内存, 我们可以看下 NIO 里面使用的 rt.jar 包里面的 IOUtil 类：

```
static int write(FileDescriptor paramFileDescriptor, ByteBuffer paramByteBuffer, long paramLong, NativeDispatcher paramNativeDispatcher)
    throws IOException
  {
    //I 如果是直接内存，则直接调用native方法写入
    if ((paramByteBuffer instanceof DirectBuffer)) {
      return writeFromNativeBuffer(paramFileDescriptor, paramByteBuffer, paramLong, paramNativeDispatcher);
    }

  // II 否者创建一个临时的直接内存缓存
    int i = paramByteBuffer.position();
    int j = paramByteBuffer.limit();
    assert (i <= j);
    int k = i <= j ? j - i : 0;
    ByteBuffer localByteBuffer = Util.getTemporaryDirectBuffer(k);
   //复制内容到临时直接内存
    try {
      localByteBuffer.put(paramByteBuffer);
      localByteBuffer.flip();

      paramByteBuffer.position(i);
      // III 调用native方法写入
      int m = writeFromNativeBuffer(paramFileDescriptor, localByteBuffer, paramLong, paramNativeDispatcher);
      if (m > 0)
      {
        paramByteBuffer.position(i + m);
      }
      return m;
    } finally {
    //IV释放临时直接内存
      Util.offerFirstTemporaryDirectBuffer(localByteBuffer);
    }
  }
```

如果用户态申请的堆外内存（直接内存）那么就会省去中间的拷贝操作，操作系统层面会直接使用用户态申请的堆外内存（直接内存）里面的数据。

#### 5.2 使用 ByteBuffer 分配堆内存与堆外内存

Java NIO 中提供了一个 ByteBuffer 用来分配发送和接受缓存用的，其分为两种模式的内存，一个是我们常用的堆内存，一个是堆外内存(直接内存)。

- 当我们调用 ByteBuffer 的 allocate 方法时候，实际分配的是堆内存，其代码如下：

```
    public static ByteBuffer allocate(int capacity) {
        if (capacity < 0)
            throw new IllegalArgumentException();
        return new HeapByteBuffer(capacity, capacity);
    }
    HeapByteBuffer(int cap, int lim) {       
        super(-1, 0, lim, cap, new byte[cap], 0);
    }
```

可知这里是 new 了一个 byte 数组，所以分配的是堆内存。

```
    ByteBuffer(int mark, int pos, int lim, int cap,  
                 byte[] hb, int offset)
    {
        super(mark, pos, lim, cap);
        this.hb = hb;
        this.offset = offset;
    }
```

可知其内部是通过 hb 这个指针来指向了分配的堆内存。

- 当调用 ByteBuffer 的 allocateDirect 方法时候，分配的就是堆外内存：

```
    public static ByteBuffer allocateDirect(int capacity) {
        return new DirectByteBuffer(capacity);
    }
    protected static final Unsafe unsafe = Bits.unsafe();

 DirectByteBuffer(int cap) {                   // package-private

        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);
        //1 使用Unsafe分配堆外内存
        long base = 0;
        try {
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
       //2 对分配的堆外内存进行初始化
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
       //3.创建堆外内存回收器
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }
```

可知堆外内存是使用 UNSAFE 类进行内存分配的。

### 六、参考

- 计算机网络第五版
- Netty 权威指南
- Java NIO

### 七、进阶篇

- [Java NIO 框架 Netty 之美：基础篇之一](https://gitbook.cn/gitchat/activity/5b01714ca0810c23901c55ac)
- [Java NIO 框架 Netty 之美：粘包与半包问题](https://gitbook.cn/gitchat/activity/5b13e6a675742e21d6d14ea4)