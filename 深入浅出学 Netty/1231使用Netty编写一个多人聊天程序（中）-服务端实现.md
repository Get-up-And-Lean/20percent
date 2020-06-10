# 12/31使用Netty编写一个多人聊天程序（中）-服务端实现

## 前言

前文中我们进行了需求澄清，协议制订，服务端设计，本文将在这些的基础上实现完整的服务端功能。

## 编解码器实现

消息的收发基础是编解码器。上文对协议的制订，最外围的结构是报文头加报文体的形式。针对这个结构，实现报文分割，我们可以直接接触Netty提供的内嵌支持`LengthFieldBasedFrameDecoder`进行报文体的长度确定和分割。

报文体中是具体的消息，我们需要根据消息的不同类型来进行具体的区分，这部分就需要自行实现解码器了，自定义解码器的类名制订为`handler.CommandDecoder`。解码器的核心思路读取第一个字节的协议类型，而后根据不同的协议类型，按照协议读取出对应的字段数据，将这些字段数据组装`Command`对象，并且向后续的处理器进行传递。整体的代码设计如下

```
public class CommandDecoder extends ChannelInboundHandlerAdapter
{
    private static final Charset CHARSET = Charset.forName("utf8");

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
    {
        ByteBuf     buf         = (ByteBuf) msg;
        byte        b           = buf.readByte();
        CommandType commandType = CommandType.value(b);
        Command     command;
        switch (commandType)
        {
            case REGISTER:
            {
                String clientName = readString(buf);
                command = new RegisterCommand();
                ((RegisterCommand) command).setRegisterName(clientName);
                break;
            }
            case LOGIN:
            {
                //...省略相似逻辑代码
            }
            case SEND_TO_CLIENT:
            {
                //...省略相似逻辑代码
            }
            case CREATE_GROUP:
            {
               //...省略相似逻辑代码
            }
            case JOIN_GROUP:
            {
               //...省略相似逻辑代码
            }
            case SEND_TO_GROUP:
            {
               //...省略相似逻辑代码
            }
            case HEART_BEAT:
            {
                //...省略相似逻辑代码
            }
            default:
                throw new IllegalStateException("Unexpected value: " + commandType);
        }
        buf.release();
        //将二进制协议解析完毕，组装成具体的Command类，传递给后续的处理器
        ctx.fireChannelRead(command);
    }

    private String readString(ByteBuf buf)
    {
        int    length  = buf.readInt();
        byte[] content = new byte[length];
        buf.readBytes(content);
        return new String(content, CHARSET);
    }
}
```

在类的功能设计中，遵循“单一职责”这一设计原则，每一个类仅仅只完成自己负责的部分。通过这种方式，有利于降低类之间的耦合，在出错的时候也方便于定位问题区域。`CommandDecoder`的设计就遵循了这一原则，其只是关注于完成将二进制数据转换为后续处理器能处理的命令对象。

这里有一个地方需要注意，在将二进制数据转换为`Command`对象后，`ByteBuf`对象需要进行手动释放。Netty中采用内存池的方式管理分配的内存，其分配的内存的载体就是`ByteBuf`对象。而`ByteBuf`在使用完毕后，必须进行释放，才能将其承载的内存空间归还给缓存池。如果不释放的话，实际上就造成了内存泄漏，最终会导致没有内存可用。因此，`CommandDecoder`在将`command`对象传递给下一个`handler`之前，执行`buf.release()`进行释放。将命令对象传递给下一个处理器，通过调用方法`io.netty.channel.ChannelHandlerContext#fireChannelRead`来完成。

## 命令处理器

命令对象被解析出来后，接着就是对命令对象的处理。通讯协议的设计，第一个字节用于表达消息类型，因此存在着很大的扩展性。对于命令处理器来说，不能使用硬编码的方式进行，否则无论后续是修改已经存在的消息格式，还是添加新的消息格式，都需要反复的修改代码，不方便也带来隐患。 在面向对象中，有一个原则叫做“开闭原则”，简单来说，就是代码需要对扩展开放，对修改封闭。显然，如果命令处理器采用硬编码解析命令的话，就违反了这一原则。 在这里，考虑到所有的命令都可以通过命令类型来进行区分，命令处理器实际上可以设计成为一个分发路由的模式。也就是命令处理器本身提供一个统一的接口，每一个具体的命令解析都继承这个接口，实现为一个针对具体命令的命令解析器。而命令处理器在获取到命令之后，根据命令类型，找到对应的命令解析器，将命令分发给其处理，命令处理器本身不执行具体的业务逻辑。

这种设计的好处在于，后续需要扩展或者修改对应的命令格式时，只需要新增或者修改对应的命令解析器，而命令处理器本身不需要修改。下面来看具体的代码实现

```
public class CommandHandler extends ChannelInboundHandlerAdapter
{
    private volatile EnumMap<CommandType, CommandProcessor> processors;

    public CommandHandler(EnumMap<CommandType, CommandProcessor> processors)
    {
        this.processors = processors;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
    {
        Command command = (Command) msg;
        processors.get(command.type()).process(command, ctx);
    }

    public void setProcessors(EnumMap<CommandType, CommandProcessor> processors)
    {
        this.processors = processors;
    }

    static interface CommandProcessor
    {
        void process(Command command, ChannelHandlerContext ctx);
    }
}
```

通过允许调用`set`方法，甚至可以实现在运行期更改命令解析器，从而实现动态消息协议处理变更或者消息新增的功能。考虑到允许运行期修改，解析器映射`processors`要使用`volatile`关键字进行修饰，确保修改后可见。

此外，服务端收到的每一个命令都需要发送给客户端其对应的响应，这部分响应的处理，也是在各自命令的命令解析器中完成。

### 注册命令解析器

服务端对注册命令的解析包含两个方面的作用：

- 持久化客户端信息，包含客户端标识和客户端ID。
- 当前客户端状态变更为在线，修改对应的路由表信息，由于此时是新建的客户端，因此只需要更新单聊中的路由信息。

### 登录命令解析器

服务端对登录命令的解析包含有：

- 该客户端的状态变更为在线，更新对应的路由表信息，包含单聊和群聊路由信息。

### 单聊消息命令解析器

服务端对单聊消息命令的解析包含有：

- 通过路由表寻找到目的客户端的`SocketChanel`对象，按照消息格式，发送数据给对应的客户端

### 创建群聊命令解析器

服务端对创建群聊命令的解析有：

- 持久化群聊信息，包含群聊标识和群聊ID。
- 将当前的客户端加入到该群聊中。
- 新增一个群聊路由，并且其中包含当前客户端的`SocketChannel`对象。

### 加入群聊命令解析器

服务端对加入群聊命令的解析有：

- 将当前客户端加入到群聊中
- 获取该群聊路由，在其中加入当前客户端的`SocketChannel`对象。

### 发送群聊消息命令解析器

服务端对发送群聊消息命令的解析有：

- 通过路由表寻找目的群聊，组装发送的群聊消息对象，遍历群聊路由，对除了发送者外的客户端的`SocketChannel`发送群聊消息。

### 心跳命令解析器

服务端对心跳命令的解析有：

- 更新该客户端的在线状态超时计时器，确认客户端当前的在线状态。

## 客户端在线状态维持-心跳检测

消息的接收发送都依赖于路由表，因此对路由表的及时更新就显得很重要。而路由表的维持中，重要的一点就在于客户端的上线状态变更。上线自不用说，客户端发送注册命令或者登录命令就说明了客户端上线。而客户端可以随时关闭链接下线，链接的关闭可能是通过关闭发送tcp消息来关闭，也可能是因为网络中断。

通过判断一定时间内是否有收到客户端消息来判断客户端在线是一个简单有效的方法。对于服务端来说，在内部为每一个客户端维持一个计时器，在超时时间内如果没有收到客户端发送的消息，则判定为客户端离线，在服务端这一侧关闭客户端的`SocketChannel`对象。对于服务端来说，除了业务需要发送的登录，消息等，没有功能需求需要发送数据时，则可以通过发送心跳消息来通知服务端自己仍然在线。为此，客户端也需要自己维持一个超时计时器，在超时时间内，如果没有发送过消息，则主动发送一个心跳消息。

需要注意的是，为了避免服务端误判，服务端的超时时间需要比客户端的超时时间要长。

心跳检测，也称之为空闲检测，Netty已经为我们提供了内置支持，也就是类`io.netty.handler.timeout.IdleStateHandler`。该处理器支持对链路上的读操作进行超时跟踪，也支持对写操作进行超时跟踪，也支持同时对两者进行超时跟踪。来看下其构造方法

```
public IdleStateHandler(
            long readerIdleTime, long writerIdleTime, long allIdleTime,
            TimeUnit unit) {
        this(false, readerIdleTime, writerIdleTime, allIdleTime, unit);
    }
public IdleStateHandler(boolean observeOutput,
            long readerIdleTime, long writerIdleTime, long allIdleTime,
            TimeUnit unit){   //...省略构造方法
}
```

`readerIdleTime`、`writerIdleTime`，`allIdleTime`分表代表读取空闲超时，写出空闲超时，读或写空闲超时。对于`allIdleTime`而言，无论读取还是写出，只要超时了都会触发这个方法。

这个类的使用也很简单，将其放在`pipeline`的任意位置均可。当对应的超时事件触发时，则会沿着pipeline，从当前位置按照入站方向进行传播。传播的方式是调用下一个入站处理器的`io.netty.channel.ChannelInboundHandler#userEventTriggered`方法。传播的事件有

```
    public static final IdleStateEvent FIRST_READER_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.READER_IDLE, true);
    public static final IdleStateEvent READER_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.READER_IDLE, false);
    public static final IdleStateEvent FIRST_WRITER_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.WRITER_IDLE, true);
    public static final IdleStateEvent WRITER_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.WRITER_IDLE, false);
    public static final IdleStateEvent FIRST_ALL_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.ALL_IDLE, true);
    public static final IdleStateEvent ALL_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.ALL_IDLE, false);
```

对于客户端而言，需要设置一个写空闲超时，可以在`pipeline`中添加`IdleStateHandler`，并且设置写空闲超时，当出现超时事件时，向服务端发送一个心跳消息。

对于服务端而言，需要设置一个读空闲超时，可以在对应连接的pipeline中添加`IdleStateHandler`，并且设置读取空闲，当出现空闲事件时，关闭当前的连接。关闭当前连接需要一个处理器能够响应`IdleStateHandler`传递的读取空闲事件。该处理的代码可以设计如下

```
public class ClientIdleHandler extends ChannelInboundHandlerAdapter
{
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception
    {
        if (evt == IdleStateEvent.READER_IDLE_STATE_EVENT || evt == IdleStateEvent.FIRST_READER_IDLE_STATE_EVENT)
        {
            ctx.channel().close();
        }
    }
}
```

而连接关闭的时候产生的客户端离线维护动作可以交给其他的处理器和路由来负责。

## 路由设计

路由功能可以说是整个服务端的核心。负责单聊，群聊路由表的维护；使用单聊和群聊路由表来路由对应的消息。

### 单聊路由表更新

首先来看单聊路由的实现。根据上文对单聊功能的设计，单聊的存储结构是一个Map结构，用客户端ID映射到其`SocketChannel`对象。出于并发安全的考虑，显然应该是一个`ConCurrentMap`对象。当客户端上线时，需要新增映射元素。来考虑一种情况，服务端当前压力很重，线程的负载比较重，有一个客户端链接成功发送登录命令，由于线程调度的原因，还没有执行到路由的`put`方法，此时客户端应用宕机并且快速重启，再次链接并且发送登录指令。在这种情况下，两个线程竞争更新单聊路由元素，可能将已经失效的`SocketChannel`元素更新到路由表中。

这种异常情况，无法单纯依靠并发控制手段，诸如锁或者内存中“JVM内单key排他任务”等方式保证数据的正确性。因为这些手段的安全性保证是在事件按照时间顺序发生的前提下，而线程调度可能导致事件发生和时间本身乱序。

解决这个问题有两个方面的思路：

- 并发竞争产生错误，是一次性事件。只要有有效的客户端链接后续与服务端产生通信行为，就可以将正确的链接更新到路由表中。也就是说，每次客户端发送数据到服务端时，都对比下路由表中的单聊路由信息。如果客户端ID到`SocketChannel`对象的映射是错误的，则修改为当前发送数据的`SocketChannel`对象。
- 上述的并发竞争导致数据错误，根源在于采用了数据替换的方式。而数据替换无法避免因为线程调度导致的时间顺序错乱问题。那么我们在更新数据的时候不采取替换的方式，而是采取增加和删除的方式就容易解决了。比如客户端登录时，通过get方式首先获取当前的映射元素，如果为空则使用`putIfAbsent`方法放入元素；如果不为空，则将路由中的`SocketChannel`和当前的`SocketChannel`形成数组，使用`java.util.concurrent.ConcurrentMap#replace(K, V, V)`方法完成类似CAS的替换。删除也是类似的思路，如果当前路由存在的元素是单一元素，则调用`java.util.concurrent.ConcurrentMap#remove`进行删除；如果存在的元素是数组元素，则获取当前元素，并且将自身`SocketChannel`从数组中删除，将这个新的对象调用`ConcurrentMap#replace(K, V, V)`进行替换，如果成功就完成删除了。通过这种方式，线程并发更新，最后的留存结果必然是真正在线的一方，因为不在线的一方最终会执行删除操作，剔除自身`SocketChannel`对象。

从两个思路的角度来看，第一个思路简单清晰，实现代码也较为简单，缺点是每一次收到客户端消息都需要检查单聊路由。而实际上，在第一次更新到正确值之后，后续的检查都是没有意义被浪费的，造成了不必要的性能损失。第二个思路比较复杂，实现代码也较多一些，好处在于一次更新即可设置到正确的值。因此在服务端实现中，我们选择思路二作为实现依据。其实现的代码如下

```
    private ConcurrentMap<String, Object> clientRouter = new ConcurrentHashMap<>();

    @Override
    public void clientOnline(String clientId, Channel channel)
    {
        Object exist = clientRouter.get(clientId);
        while (true)
        {
            if (exist == null)
            {
                if ((exist = clientRouter.putIfAbsent(clientId, channel)) == null)
                {
                    break;
                }
            }
            else
            {
                Channel[] array = null;
                if (exist instanceof Channel)
                {
                    array = new Channel[2];
                    array[0] = (Channel) exist;
                    array[1] = channel;
                }
                else
                {
                    Channel[] oldArray = (Channel[]) exist;
                    array = new Channel[oldArray.length + 1];
                    System.arraycopy(oldArray, 0, array, 1, oldArray.length);
                    array[0] = channel;
                }
                if (clientRouter.replace(clientId, exist, array))
                {
                    break;
                }
                else
                {
                    exist = clientRouter.get(clientId);
                }
            }
        }
    }

    @Override
    public void clientOffline(String clientId, Channel channel)
    {
        Object exist = clientRouter.get(clientId);
        while (true)
        {
            if (exist == channel)
            {
                if (clientRouter.remove(clientId, channel))
                {
                    break;
                }
                else
                {
                    exist = clientRouter.get(clientId);
                }
            }
            else if (exist instanceof Channel[])
            {
                Channel[] array = (Channel[]) exist;
                if (array.length==2)
                {
                    Channel newValue =array[0]==channel?array[1]:array[0];
                    if (clientRouter.replace(clientId, exist, newValue))
                    {
                        break;
                    }
                }
                else{
                    ArrayList<Channel> newArrays = new ArrayList<>();
                    for (Channel each : array)
                    {
                        if (each!=channel)
                        {
                            newArrays.add(each);
                        }
                    }
                    Channel[] channels = newArrays.toArray(new Channel[0]);
                    if (clientRouter.replace(clientId, exist, channels))
                    {
                        break;
                    }
                }
                exist= clientRouter.get(clientId);
            }
            else
            {
                throw new IllegalStateException();
            }
        }
    }
```

可以看到，无论是上线还是下线，相关的操作都是在一个循环中完成的。因为既然是类CAS竞争操作，在多线程并发竞争的情况下，必然存在失败的一方，但是在循环中，不断的尝试，最终都可以成功完成自己的任务。

### 群聊路由表更新

群聊路由表的操作包含有：

- 创建群聊
- 将客户端加入群聊
- 客户端离线时退出群聊

显然，出于并发安全的考虑，群聊路由信息的最外层，也是一个`ConCurrentMap`结构。这样在新增群聊的时候才能避免同一个群聊ID互相覆盖的潜在问题（引起问题的可能情况和单聊中的线程调度导致的问题相似）。由于群聊路由信息在内存中创建后，不需要删除（需求不包含，实际上可以通过群聊中最后一次发送过消息的时间，将较少使用的群聊从内存中删除，后续有使用时再次加载信息），本身也不需要互相覆盖之类的操作，因此群聊创建应该采用`putIfAbsent`方法来放入群聊ID和群聊路由的映射关系。

上篇设计方案中，群聊的结构体中存储的要素是该群聊下在线的客户端的`SocketChannel`。对于这个结构，也存在客户端上下线对添加和删除的并发问题。其解决思路和单聊中的并发解决思路是相同的，因此这里就不贴出赘述代码了，大家可以自行在代码仓库中查看。

群聊路由表的更新时机有：

- 群聊被创建时，创建该群聊的客户端要加入其中
- 客户端请求加入群聊时
- 客户端上线，其所有加入的群聊都需要更新该客户端信息
- 客户端离线，其所有加入的群聊都需要更新该客户端信息

### 消息发送

在正确维护路由表的基础上，消息的发送就显得十分容易了。对于单聊消息而言，通过目的客户端ID获得其`SocketChannel`对象，写入对应的消息即可。对于群聊消息而言，通过群聊ID获取群聊ID对应的群聊路由，遍历其中的`SocketChannel`，发送消息即可，在遍历的时候，注意剔除来源客户端即可。

## 发送时数据编码

在业务的处理器中，发送给客户端的消息必然是一个`message.Receive`对象。显然，我们需要通过编码器，来将对象编码为符合协议要求的二进制数据。因为通讯协议最外层格式为报文头+报文体的格式，而报文体则按照协议有不同的类型。因此这里可以设计两个编码器：

- 第一个编码器负责将对象按照协议转化为报文体的二进制数据
- 第二个解码器负责将计算报文体的二进制数据长度，以此为依据生成报文头，并且将报文头和报文体组装在一起，调用`Channel`的`writeAndFlush`方法进行数据写出。

对于第一个编码器而言，为了方便完成`Receive`对象到二进制数据的转换，考虑对`Receive`接口增加一个`message.Receive#writeToMessage(ByteBuf)`方法。这样子将`Receive`转换为报文协议二进制数据的职责下放到每一个具体的`Receive`对象中，编码器的职责就变得简单而固化：通过ByteBuf分配器申请合适大小的`ByteBuf`对象，传递给`Receive`进行序列化，而后传递给下一个编码器。其代码可以实现为

```
public class ReceiveEncoder extends ChannelOutboundHandlerAdapter
{
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception
    {
        Receive receive = (Receive) msg;
        ByteBuf buffer  = ctx.alloc().buffer();
        //空出报文头的写入空间
        buffer.writerIndex(4);
        receive.writeToMessage(buffer);
        ctx.write(buffer, promise);
    }
}
```

报文头是4个字节的整型数字，因此我们在申请`ByteBuf`后，直接空出来开头的4个字节区域，留给下一个编码器写入报文体的长度。填充报文头的编码器代码如下

```
public class MessageEncoder extends ChannelOutboundHandlerAdapter
{
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception
    {
        ByteBuf buf           = (ByteBuf) msg;
        int     oldWriteIndex = buf.writerIndex();
        int     length        = oldWriteIndex - 4;
        //报文头写入报文体的长度后，恢复正确的写入下标位置。否则在输出的时候数据会出错。
        buf.writerIndex(0).writeInt(length).writerIndex(oldWriteIndex);
    }
}
```

两个编码器的顺序是先`ReceiveEncoder处理`而后`MessageEncoder处理`，因此在`pipeline`中，`ReceiveEncoder`需要在更靠后的位置。

## 思考与总结

本篇文章在上文总体设计的基础上，对重点实现功能进行了剖析，通过这篇文章，可以掌握到对实现一个多人聊天的服务端相关功能的实现思路。在实现的过程中，通过设计思路的讲解，分析了一些经典的设计手段，诸如分发模式的使用，在类的功能设计上应用单一职责，开闭原则等；并且对功能实现中可能存在的并发安全问题做了详尽的阐述。

在下一篇文章中，我们将会完成客户端的设计，并且进行功能验收。