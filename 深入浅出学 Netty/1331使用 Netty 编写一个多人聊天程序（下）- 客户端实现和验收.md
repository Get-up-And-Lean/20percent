# 13/31使用 Netty 编写一个多人聊天程序（下）- 客户端实现和验收

## 引言

上篇文章中，详细阐述了服务端核心功能的设计思路和实现逻辑。与服务端相比起来，客户端的实现就比较简单了。Netty 为开发者尽最大可能保证了编程模型的一致性。服务端和客户端的区别仅仅只是在于`Channel`具体实现类和`BootStrap`引导类的区别。

客户端的代码托管于：https://gitee.com/eric_ds/learnNetty/tree/master/client

## 命令编码

和服务端一样，客户端需要发送消息，首要考虑的问题也是对命令对象`Command`的二进制编码问题。每个命令对象本身内部最清晰自身结构，因此这里采用和服务端编码`Receive`对象一样的思路，在`Command`接口上增加`writeToBuf`方法，将编码的部分职责下放到具体的`Command`对象中，因此，命令编码的handler可以编写为如下形式。

```
public class CommandEncoder extends ChannelOutboundHandlerAdapter
{
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception
    {
        if (msg instanceof Command)
        {
            Command command = (Command) msg;
            ByteBuf buffer  = ctx.alloc().buffer();
            buffer.writerIndex(4);
            command.writeToBuf(buffer);
            int writerIndex = buffer.writerIndex();
            //消息协议前四个字节是整型变量，需要计算报文体长度
            buffer.writerIndex(0).writeInt(writerIndex - 4).writerIndex(writerIndex);
            ctx.write(buffer, promise);
        }
        else
        {
            throw new IllegalArgumentException();
        }
    }
}
```

## 响应解码

客户端的响应解码的职能和服务端对命令的解码职能很接近。在服务端的处理方式中，采用了一个命令解码器，在这个类中根据协议类型字段而对后续的字节进行解析处理构建`Command`对象。这种写法的缺点主要在于需要硬编码所有的命令类型，在新增或者修改命令对象的时候不太方便。

参考命令编码的思路，我们可以将响应解码的具体内容放在具体的`Receive`对象中进行处理。也就是为`Receive`接口新增方法`readFromBuf`。在这个基础上，还需要解决的问题就是如何根据协议头的消息类型构建正确的`Receive`实现类。解决办法倒也简单，可以创建一个`EnumMap`,放入`ReceiveType`和对应`Receive`实现类的class对象，通过反射来构建。而这`EnumMap`可以在初始化的时候被传入。这样，后续消息格式变更或者消息新增，可以简单添加元素到`EnumMap`中实现，解码器则无需更改。

按照上面的思路，解码器的代码可以写为

```
public class ReceiveDecoder extends ChannelInboundHandlerAdapter
{
    private EnumMap<ReceiveType, Class<? extends Receive>> enumMap;

    public ReceiveDecoder(EnumMap<ReceiveType, Class<? extends Receive>> enumMap)
    {
        this.enumMap = enumMap;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
    {
        ByteBuf                  buffer      = (ByteBuf) msg;
        byte                     b           = buffer.readByte();
        ReceiveType              receiveType = ReceiveType.value(b);
        Class<? extends Receive> aClass      = enumMap.get(receiveType);
        Receive                  receive     = aClass.newInstance();
        receive.readFromBuf(buffer);
        ctx.fireChannelRead(receive);
    }
}
```

除了响应解码器外，在响应解码之前，首先是进行报文的拆包处理。这个拆包的思路和服务端中对命令的报文拆包思路是相同的，都是使用`LengthFieldBasedFrameDecoder`，从报文头读取长度，进而确定报文体的内容。

## 注册和登录示例

在完成命令编码和响应解码的基础上，我们就可以先完成一个注册和登录的小例子了。

首先来看下服务端的引导应用，如下：

```
public class Server
{
    public static void main(String[] args) throws InterruptedException
    {
        DAOFactory      daoFactory      = new MemDAOFactory();
        final Router    router          = new RouterImpl(daoFactory.getGroupDAO(), daoFactory.getRelationDAO());
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.channel(NioServerSocketChannel.class);
        serverBootstrap.group(new NioEventLoopGroup(1), new NioEventLoopGroup());
        //handler方法传入的处理器工作在服务端监听链接上
        serverBootstrap.handler(new ChannelInboundHandlerAdapter()
        {
            @Override
            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
            {
                System.out.println("msg对象是一个SocketChannel：" + (msg instanceof SocketChannel));
                ctx.fireChannelRead(msg);
            }
        });
        final EnumMap<CommandType, CommandHandler.CommandProcessor> enumMap = new EnumMap<CommandType, CommandHandler.CommandProcessor>(CommandType.class);
        enumMap.put(CommandType.LOGIN, new LoginProcessor(daoFactory.getClientDAO(), router));
        enumMap.put(CommandType.REGISTER, new RegisterProcessor(daoFactory.getClientDAO(), router));
        enumMap.put(CommandType.CREATE_GROUP, new CreateGroupProcessor(daoFactory.getGroupDAO(), daoFactory.getRelationDAO(), router));
        enumMap.put(CommandType.JOIN_GROUP, new JoinGroupProcessor(daoFactory.getGroupDAO(), daoFactory.getRelationDAO(), router));
        enumMap.put(CommandType.SEND_TO_CLIENT, new SendToClientProcessor(daoFactory.getClientDAO(), router));
        enumMap.put(CommandType.SEND_TO_GROUP, new SendToGroupProcessor(router));
        enumMap.put(CommandType.HEART_BEAT, new HeartBeatProcessor());
        serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>()
        {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception
            {
                ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4));
                ch.pipeline().addLast(new CommandDecoder());
                ch.pipeline().addLast(new CommandHandler(enumMap));
                ch.pipeline().addLast(new OfflineHandler(router));
                ch.pipeline().addLast(new ClientIdleHandler());
                ch.pipeline().addLast(new MessageEncoder());
                ch.pipeline().addLast(new ReceiveEncoder());
            }
        });
        ChannelFuture future = serverBootstrap.bind(8888);
        future.sync();
        future.channel().closeFuture().sync();
    }
}
```

由于在服务端功能实现的时候，客户端在注册成功的时候就默认登录了，我们在注册成功后，先下线，而后再登录。客户端代码如下

```
public class Client
{
    public static void main(String[] args) throws InterruptedException
    {
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(new NioEventLoopGroup(1));
        bootstrap.channel(NioSocketChannel.class);
        final EnumMap<ReceiveType, Class<? extends Receive>> enumMap = new EnumMap<ReceiveType, Class<? extends Receive>>(ReceiveType.class);
        enumMap.put(ReceiveType.LOGIN, LoginReceive.class);
        enumMap.put(ReceiveType.REGISTER, RegisterReceive.class);
        bootstrap.handler(new ChannelInitializer<SocketChannel>()
        {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception
            {
                ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4));
                ch.pipeline().addLast(new ReceiveDecoder(enumMap));
                ch.pipeline().addLast(new ChannelInboundHandlerAdapter()
                {
                    @Override
                    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
                    {
                        if (msg instanceof LoginReceive)
                        {
                            String codeMsg = ((LoginReceive) msg).getCodeMsg();
                            System.out.println("登录响应:" + codeMsg);
                        }
                        else if (msg instanceof RegisterReceive)
                        {
                            String codeMsg = ((RegisterReceive) msg).getCodeMsg();
                            System.out.println("注册响应：" + codeMsg);
                        }
                        else
                        {
                            System.out.println("当前还未识别的响应的内容");
                        }
                    }
                });
                ch.pipeline().addLast(new CommandEncoder());
            }
        });
        ChannelFuture connect = bootstrap.connect("127.0.0.1", 8888);
        connect.sync();
        Channel         channel         = connect.channel();
        System.out.println("第一次发送注册请求");
        RegisterCommand registerCommand = new RegisterCommand();
        registerCommand.setRegisterName("深入浅出学Netty读者：风火");
        channel.writeAndFlush(registerCommand);
        System.out.println("第二次发送同样的注册请求");
        registerCommand.setRegisterName("深入浅出学Netty读者：风火");
        channel.writeAndFlush(registerCommand);
        channel.closeFuture().sync();
    }
}
```

我们首先运行服务端程序，然后运行客户端程序。当客户端程序运行的时候，服务端的控制台输出如下内容：

![img](https://images.gitbook.cn/15746741638801)

这个输出佐证了在`ServerBootStrap`类中，`handler`方法和`childHandler`方法的区别。前者传入的处理器用于服务端建立监听的链接，后者服务于客户端接入的链接。

在来看下客户端的控制台输出

![img](https://images.gitbook.cn/15746741638816)

可以看到，消息的发送和响应的接收不是习惯的一个发送一个响应的顺序，而是两个发送，两个响应。入门篇有提到，Netty中大部分操作都是异步的。这里的写出刷新也是如此。当我们在`main`线程中调用`writeAndFlush`，实际上是将消息对象包装为一个写出任务，投递到`Channel`绑定的`NioEventLoop`线程上执行。当从`pipeline`的处理器走完一遍流程后，被添加到`Channel`关联的发送队列，并且刷写到socket上。

因此当在`main`线程中执行`writeAndFlush`方法，将消息投递到`NioEventLoop`的任务队列时，方法就已经返回了，方法返回，和数据是否写出完全无关。由于投递任务的速度很快，所以控制台连续输出了两次请求发送。

## 单聊消息示例

为了验证两个客户端单聊是否正确，我们将`Client`的代码复制一份，命令为了`Client2`。为了能正确解析群聊消息响应，我们在主程序的`EnumMap`中增加了`MsgFromClientReceive`类型的映射，对`Client`的修改内容如下

![img](https://images.gitbook.cn/15746741638828)

白色划线部分即为新增内容。

下面我们分别启动`Client1`和`Client2`。在这里我们仍然假定服务端是全新启动，其尚未存储数据，因此仍然使用注册命令，后续将会采用登录命令。`Client1`启动后，控制台输出如下

![img](https://images.gitbook.cn/15746741638839)

`Client2`的代码在注册成功之后就发送一个信息给client1。其改造后的代码如下

![img](https://images.gitbook.cn/15746741638851)

运行`Client2`，其控制台输出如下内容

![img](https://images.gitbook.cn/15746741638861)

回头查看`Client1`的控制台，已经收到了来自`Client1`消息，如下

![img](https://images.gitbook.cn/15746741638872)

## 群聊消息示例

接下来我们来验证群聊的功能。为了验证这一功能，首先先将`Client`代码复制为三份，分别命名为`Client1`、`Client2`、`Client3`。首先启动服务端，然后依次启动三个客户端，全部注册成功。控制台分别如下输出

![img](https://images.gitbook.cn/15746741638883)

![img](https://images.gitbook.cn/15746741638896)

![img](https://images.gitbook.cn/15746741638908)

接着让`client1`发送创建群聊指令，而让`client2`和`client3`都选择加入该群聊。首先我们需要将群聊相关的响应内容加入解码映射中，如下

![img](https://images.gitbook.cn/15746741638919)

而后在响应处理部分，也需要增加这几个响应的对应处理。如下

![img](https://images.gitbook.cn/15746741638931)

运行`client2`和`client3`，让其加入群聊，控制台输出如下

![img](https://images.gitbook.cn/15746741638942)

![img](https://images.gitbook.cn/15746741638952)

都成功加入群聊后，我们来看下发送消息的效果。全部断开连接，让`clien1`和`client2`执行登录指令完成登录，让`client3`执行登录和发送群聊消息命令。`client3`的主要命令如下

![img](https://images.gitbook.cn/15746741638963)

执行后的效果是：

![img](https://images.gitbook.cn/15746741638973)

可见群聊消息并没有发送给源发送者，在看`client1`和`client2`的输出内容

![img](https://images.gitbook.cn/15746741638984)![img](https://images.gitbook.cn/15746741638995)

## 心跳消息示例

服务端要能演示客户端空闲超时关闭连接，需要增加2个新的处理器：

- 空闲检测处理器，也就是`IdleStateHandler`
- 接收并处理空闲事件的业务处理器，也就是`ClientIdleHandler`

为了避免`pipeline`中其他处理器将读取事件处理后不向后传递导致错误触发空闲事件，`IdleStateHandler`应该作为pipeline中入站事件的第一个处理器，而`ClientIdleHandler`紧随其后即可，添加后的效果如下

![img](https://images.gitbook.cn/15746741639006)

读取空闲2秒就会触发空闲事件。我们让`client1`执行登录操作后空闲，看看服务端控制台的输出。

![img](https://images.gitbook.cn/15746741639017)

如果客户端不发出消息，在连续2次超时后，服务端主动终止了连接。为了避免这个问题，我们在客户端增加了心跳检测。一样也是基于超时的原理，只不过对超时事件的处理改为发送一个心跳消息。和服务端相同，客户端的`pipeline`中，增加一个空闲检测`handler`和在这之后的处理空闲事件发送心跳命令的心跳`handler`。

代码如下：

![img](https://images.gitbook.cn/15746741639027)

要能够防止服务端检测超时，这边的写空闲检测的时间要小于服务端读空闲的时间。

启动客户端，执行登录命令，查看控制台输出：

![img](https://images.gitbook.cn/15746741639039)

在有心跳命令的情况下，服务端就不会把客户端下线了。

## 思考与总结

本文分析了创建客户端所需要设计的命令解码器和响应编码器两个重点。并且给出了客户端引导启动的完整例子。随后按照协议的内部，逐步的对每一个需求功能进行验收和结果输出。

到这里，完成了使用 Netty 开发一个多人在线聊天的项目。整个项目涉及到客户端状态的并发安全设计，编解码设计，路由设计等功能需求。真实项目的开发在业务上多出很多要求，但是通信的本质大体是相似的，都可以使用到这个例子中的思想和设计模式。

下一篇文章，将会来讲解 Netty 中对协议的直接支持，并且以 HTTP 下载文件作为示例进行演示。