# 4/31深入剖析 Java NIO 组件

## 前言

在前文中提到，Java 在 1.4 版本的时候提供了 NIO，实现了对 IO 的支持。在刚发布的时候，为了区别于原先的 IO 包，NIO的含义被定义`New I/O`。但是这么多年过去了，新事务也早就稀松平常了，再叫`New I/O`会觉得名不副实。现在大家一般都将NIO定义为`NonBlocking I/O`。

NIO 相比于 BIO 而言，在概念上就复杂了许多，并且也引入了很多新的组件，需要熟悉使用这些组件才能使用 NIO 进行开发。但是掌握好这些概念后，在看 NIO 编写的程序和框架就比较容易了。

## 组件介绍

### ByteBuffer

抽象类`java.nio.Buffer`定义了一个连续的，有限的元素空间，可用于数据的读写。在 JDK 中其具体的实现类包括有8个，涵盖了除了布尔类型的 7 个基本数据类型。在这其中我们接触最多的是`ByteBuffer`，其代表着一块连续的二进制数据，有点类似于数组。`ByteBuffer`是唯一可以用于通道进行数据交换的类。首先来看下`ByteBuffer`的类图

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191005095232.png)

`ByteBuffer`有两个具体的实现类，`DirectByteBuffer`和`HeapByteBuffer`，最大的区别就是两个实现类其内部二进制数据存储在不同的空间。后面再来细说这二者的区别，现在首先说一下`ByteBuffer`的使用方式。

为了方便理解，可以将`ByteBuffer`看成是一个字节数组。但是除了存储区域外，ByteBuffer 还有额外的三个属性：

1. **capacity**：`ByteBuffer`的容量，在ByteBuffer初始化后就不会改变
2. **position**：`ByteBuffer`的位置，或者说读写开始的下标。
3. **limit**：`ByteBuffer`的`position`的终点，或者说`position`的增长不能超出`limit`。

三者的关系可以用下图来表达：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191005100709.png)

灰色区域代表着不可读写，黄色区域是可以读写的。从定义和图可以看出这三个数值存在如下关系

> 0<=position<=limit<=capacity

执行任意的读写操作的时候都会将 Position 增加对应的长度。比如一个`int`是 4 个字节构成，则写入到`ByteBuffer`中时就会从`position`位置开始写入，并且写入完成后`position`的值会增加 4，形象的说的就是向右移动 4 个字节；读取也是同理，如果从`ByteBuffer`中读取一个`int`，也是从`position`位置开始，读取 4 个字节拼装成`int`，并且`position`向右移动 4 位，或者说`position`的值增加 4。

在`ByteBuffer`初始化后，`position`的值为 0，`limit`的值和`capacity`值相同，都等于整个`ByteBuffer`的容量。

ByteBuffer`的主要 API 包括有：

- 读写相关类：诸如`getInt`，`putInt`，`get`，`put`等；8个基本类型的读写方法，针对字节数组的读写方法；针对`ByteBuffer`的读写方法。
- 指针读写类：诸如设置`postion`和读取`position`方法；设置和读取`limit`的方法。
- 其他类型的Buffer转化类：前文提到过，`Buffer`下面有7个基本类型的各自的`Buffer`实现，这些基本类型都可以表达为不同位数的字节，因此`ByteBuffer`具备向其他类型的`Buffer`进行转化的能力。
- 其他类型：压缩方法`compack`，翻转方法`flip`，复制方法`duplicate`，切片方法`slice`等等。

**读写相关类**

这部分方法简单明了，从方法名上就能看到具体的含义。需要注意的是，`ByteBuffer`提供了在当前位置的读写方法，以及基于绝对位置的读写方法。比如以下的两个方法

- java.nio.ByteBuffer#putInt(int value)
- java.nio.ByteBuffer#putInt(int index, int value)

方法一在当前的`position`位置写入一个`int`，也就是写入 4 个字节，写完成后，`position`的值增加4；方法二则在`index`的位置写入一个`int`，写入完成对`position`不影响。

读取也是同理的，存在着基于当前位置的读取方法和基于绝对位置的读取方法。比如：

- java.nio.ByteBuffer#getInt()
- java.nio.ByteBuffer#getInt(int index)

第一个方法从当前`position`位置开始，读取 4 个字节拼装成`int`，并且在读取结束后，`position`的值增加 4；第二个方法则从`index`位置开始，读取 4 个字节拼装成`int`，读取结束后`position`的值不会变化。

**指针读写类**

`ByteBuffer`提供了对`position`和`limit`的设置方法。`ByteBuffer`中大多数时候的用户操作都是围绕在这两个属性的操纵上。理解了`position`和`limit`的含义，使用这些 API 就不存在问题了。

**其他类型的Buffer转化类**

`ByteBuffer`可以转化为其他类型的各种 Buffer 实现，比如`CharBuffer`，`IntBuffer`等等。因为这些数据都可以由多个字节结合在一起获得。需要注意的是，这些非`byte`类型的`Buffer`本质上都是`ByteBuffer`的视图，也就是说从`ByteBuffer`处调用`asXxBuffer`获得的其他类型`Buffer`对象与`ByteBuffer`对象本身使用同一个底层存储。这意味着在其他类型的`Buffer`进行的数据读写，一样也会反映到`ByteBuffer`上。两个对象的指针是独立的，但是底层存储共享同一个。

**其他类型**

**duplicate**

首先来看方法`duplicate`。该方法可以从原先的`ByteBuffer`处复制一个新的`ByteBuffer`。新的`ByteBuffer`和原先的`ByteBuffer`共享相同的底层存储，但是双方的`limit`和`position`指针是独立。此外，在创建新的`ByteBuffer`时，其position和`limit`和原先的`ByteBuffer`相同。复制的效果可以用图来表达，如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191005124419.png)

**slice**

另一个也具备复制能力，但是功能不同的方法是`slice`。从方法”切片“就可以看出这个方法的作用就是将原本的ByteBuffer分割出来一个片段，实际也的确如此。`slice`方法会创建一个新的`ByteBuffer`，新的`ByteBuffer`与原先的`ByteBuffer`共享相同的底层存储，两者的指针独立。但是新的`ByteBuffer`的position为0，而`limit`和`capacity`则等于原先`ByteBuffer`的剩余字节（原先的`ByteBuffer`的`position`到`limit`之间的数据）。切片的效果用图来表达就是

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191005125011.png)

**flip**

`flip`方法的翻译叫做翻转，有点不太好理解，这个名字主要从其作用来说的。假定有一个初始化的`ByteBuffer`对象，其写入了一个int。此时它的`position`等于 4，`limit`等于`capacity`，可以继续写入，但是读取的话没有意义，因为还没有数据。此时调用`flip`方法将从写入状态转化为读取状态，具体的变化是：

- 将`limit`设置为`position`的值
- 将`position`的值设置为 0

经过翻转之后的`position`到`limit`之间的区域，就是原先写入的内容，也就可以进行读取了。`flip`的效果可以参看下图

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191005130301.png)

**compact**

最后一个要介绍的压缩方法`compact`。随着读写的进行，`ByteBuffer`中的有效区域已经逐渐移动到`ByteBuffer`存储的后半段，这样就会导致实际的可写入位置不足。`compact`方法就是用于解决这个问题，他可以将`ByteBuffer`中的有效区域的内容移动到其存储的开头位置；这样，后面就可以空出更多的空间进行写入了。具体来说，`compact`方法的调用会到达如下效果：

- 将`position`到`limit`之间的n个字节拷贝到ByteBuffer的开头位置。拷贝完成后，`position`的值被设置为n，`limit`的值被设置为`capacity`。

整个效果用图来表达的话就是：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191005131144.png)

`ByteBuffer`还有一个比较重要的点在于，它有 2 个不同的实现类，分别是：

- 数据存储在堆中的`HeapByteBuffer`。`HeapByteBuffer`的数据存储于堆中，就类似在应用程序中通过`new`的方式申请一个二进制数组一样。其申请和释放都是在堆上进行，速度较快，但是会带来比较大的GC压力。而且在通过通道发送数据时，`HeapByteBuffer`中的数据要从堆拷贝到堆外才能写入socket缓存区，有一次数据复制的开销。
- 数据存储在堆外，或者说直接内存的`DirectByteBuffer`。`DirectByteBuffer`的数据存储于直接内存，这部分内存不受GC的控制。其申请和释放在直接内存上进行，速度比较慢。但是对 GC 的压力较小。其通过 Socket 通道发送数据时，可以直接写入 Socket 发送缓存区而不需要进行复制。

### 通道

通道具备像流一样可以读写的特性，但是流最大的不同在于流是单向的，或者用于读取数据，或者用于写出数据。而通道则是双向的，即可以在上面进行数据的读取，也可以进行数据的写出。并且通道使用的数据读写的载体也不是流所使用的字节数组，而是`ByteBuffer`。来看下通道的整体类图，如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191005230048.png)

我们重点关注的主要是有两个：

- **ServerSocketChannel**：服务端 Socket 通道，主要用于客户端链接的接入，通过调用方法`accept`来返回接入的客户端通道对象
- **SocketChannel**：客户端通道，用于行使在一个 Socket 上的读写能力。

除了和 Socket 相关的通道外，NIO 还带来了用于文件读写的文件通道`java.nio.channels.FileChannel`，感兴趣的读者可以自行搜索相关资料。

对于客户端 Socket 通道而言，主要使用以下两类方法：

- **注册选择器**：将socket通道注册到选择器上，才能享受到 IO 多路复用的好处。否则的话，通道和流的差异就不大了。
- **读写ByteBuffer**：以`ByteBuffer`作为数据载体的数据读写方法。需要注意的是，Socket 通道是非阻塞的，因此无论读取和写入都不需要担心长时间阻塞在数据准备阶段。

对于服务端通道而言，主要使用两类方法：

- **注册选择器**：这点上和客户端通道是相同的
- **accept 接受客户端**：通过这个方法来接受新的接入的客户端链接

Socket 通道对象本身是比较简单的，只要看成是原先 Socket 流的双向版本去理解即可。

### 选择器

选择器`java.nio.channels.Selector`是整个 NIO 的核心，选择器实现了 IO 多路复用的能力。使得单一线程有办法同时对多个socket通道实现监控并且及时发现需要处理的 IO 事件。选择器的使用比较简单，主要是三个步骤：

1. 通道将自身注册到一个选择器上
2. 选择器执行`select`方法阻塞等待事件发生，或者`selectNow`等待事件发生并且在无事件时快速返回
3. 选择器执行`selectedKeys`方法获得`select`或`selectNow`方法返回期间发生的IO事件。

为了理解选择器，首先需要介绍一个基本概念，选择键，其类名为`java.nio.channels.SelectionKey`。选择键是一个标识，代表着一个通道注册到一个选择器上的这种关系。一个选择键对象包含以下三个重要信息：

1. 选择键绑定的通道
2. 选择键绑定的选择器
3. 通道注册到选择器上时，关注的 IO 事件。

先介绍下 IO 事件。在 NIO 中，IO 事件一共有 4 种，分别：

1. **OP_ACCEPT**：出现可接入的客户端事件。该事件主要被服务端通道使用
2. **OP_CONNECT**：出现通道连接成功事件。该事件主要是用于客户端尝试连接服务端的情况使用
3. **OP_WRITE**：出现通道可写事件。这个事件指的是通道的 Socket 缓存区有空间可以写入了，并不是意味着数据会被发送出去（数据发送还需要等待 Tcp 将数据从 Socket 写出缓存区拿出再从内核发送）。该事件主要是用于客户端链接写出数据使用。
4. **OP_READ**：出现通道可读事件。这个事件指的是通道的 Socket 缓存区有数据可以供应用程序读取了。该事件主要是用于客户端连接读取数据使用。

4 种事件使用 4 个 int 数字表达，并且每一个事件的值都是 1 左移不同的位数得到的。因此一个通道可以通过并操作合并两个事件的值，从而在注册选择器的时候同时关注两个事件，比如代码：

```java
SelectionKey  selectionKey  = socketChannel.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE)
```

从代码可以看到，选择键是在通道注册到选择器时被生成的。其本身包含了通道对象，选择器对象，以及关注的 IO 事件信息。看如下代码

```java
SelectableChannel channel       = selectionKey.channel();
Selector          selector1     = selectionKey.selector();
int               readyOps   = selectionKey.readyOps();
if ((readyOps & SelectionKey.OP_WRITE) == SelectionKey.OP_WRITE)
{
    System.out.println("就绪事件包含写事件");
}
```

一个选择键的关注事件是在通道注册的时候初始化的，但是也可以在运行期进行更改，更改方法为`java.nio.channels.SelectionKey#interestOps(int ops)`。运行期的更改会在选择器下一次执行`select`或者`selectNow`的时候生效。

当通道注册到选择器上完毕后，选择器就可以监控在该通道上发生的 IO 事件。主要是通过以下两个方法

- select
- selectNow

两个方法的区别在于：`select`方法会一直阻塞直到监控的通道发生了关注的 IO 事件，而`selectNow`会快速查看监控通道，如果没有关注的 IO 事件，则马上返回。

当`select`或者`selectNow`方法返回后，可以通过方法`java.nio.channels.Selector#selectedKeys`来获取本次发生了 IO 事件的选择键。然后针对这些选择键进行处理，主要是获取选择键关联的通道和就绪事件，对就绪事件进行处理，比如在通道上进行读写。

选择器内部维持着三个集合来支撑对应的功能，分别是：

- **选择键集合**：该集合持有了所有注册到该选择器上的选择键。该集合可以通过方法`keys`方法来获得。该集合不可直接操作。其中元素的添加是通过通道注册来进行的，而元素的删除则是通过选择键的取消或者通道关闭来自动完成。
- **已就绪的键集合**：该集合包含了所有在`select`方法期间通道上产生了关注事件的对应的键的集合。该集合可以通过方法`selectedKeys`获得。该集合不可以添加，但是可以执行删除，无论是调用集合的`remove`方法或者调用其生成的迭代器的`remove`都可以。该集合元素的添加由选择器自身完成。
- **已取消的键集合**：该集合包含了所有取消的键。该集合不可直接获取。执行了`java.nio.channels.SelectionKey#cancel`和关闭了的通道对应的键会被添加到这个集合中。并且在下一次`select`的时候，前两个集合中与该集合的交集元素会被删除，而本集合则会被清空。

## 综述

`ByteBuffer`提供二进制数据承载能力，`SocketChannel`提供通道抽象和读写接口，`Selector`提供选择器能力，三者在一起完成了 IO 复用的模型。

## 总结与思考

本文详细分析了在 NIO 模式下，Java 为我们带来了三个重要的组件，分别是：ByteBuffer，Channel，Selector。并且具体讲解了每个组件的使用方式和注意事项。通过对三大组件的学习，读者已经掌握了基础的 API 使用，也可以在这个基础上尝试编写网络应用。但是可以开发出基于 NIO 的网络应用和开发出一个高质量，高稳定，高性能的网络应用之间存在着不小的差距。

下一个章节，会带领大家使用 NIO 的原生 API 开发一个网络应用程序，并且逐步解决实际中遇到的问题，来修改我们的程序。届时大家就会明白使用原生 API 开发一个网络程序，会遇到怎样的挑战。