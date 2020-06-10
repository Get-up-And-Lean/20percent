# 22/31Netty 流程细讲之精彩拾遗

### 引言

在前面的文章中，我们详细的学习了 Netty 处理流程中的方方面面。不过仍然有一些部分细节处的知识点为了避免影响主流程的学习而延后，这篇文章，我们就来好好分析一下 Netty 当中这些精彩的细节处理。

### JDK 空轮训 Bug

在 JDK 的 NIO 接口中有一个流传很广，影响很恶劣的 Bug，被称为 `NIO 100% CPU`，也被称为 NIO 空轮训。来看下如下代码：

```java
while(true)
{
    selector.select();
    Set < SelectionKey > keys = selector.selectedKeys();
    //省略代码：处理 keys 的内容
}
```

正常情况下，如果没有关注事件发生，selector.select() 是阻塞的，不返回的。但是当 Bug 触发时，没有关注事件发生，select 操作也会返回。由于没有事件发生，导致对处理 keys 的代码也不会运行到，此时就会在 while 循环中反复地执行 select，每次都不阻塞直接返回。导致 CPU 100%。

Netty 在一个 EventLoop 线程中也是使用类似的循环方式在 selector 上执行 select 等待，来看下代码：

```java
    private void select(boolean oldWakenUp) throws IOException
    {
        Selector selector = this.selector;
        try
        {
            int selectCnt = 0;
            long currentTimeNanos = System.nanoTime();
            long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
            for(;;)
            {
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000 L) / 1000000 L;
                //省略代码：判断当前是否已经超时以及是否存在队列，存在此种情况，则跳出循环
                int selectedKeys = selector.select(timeoutMillis);
                selectCnt++;
                //省略代码：selectedKeys 不为 0 或者出现队列有了新任务，则离开循环
                else if(SELECTOR_AUTO_REBUILD_THRESHOLD > 0 && selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD)
                {
                    selector = selectRebuildSelector(selectCnt);
                    selectCnt = 1;
                    break;
                }
                currentTimeNanos = time;
            }
            if(selectCnt > MIN_PREMATURE_SELECTOR_RETURNS)
            {
                //省略代码：输出 debug 日志
            }
        }
        catch(CancelledKeyException e)
        {
            //省略代码：输出 debug 日志
        }
    }
```

当 Bug 出现时，selector.select 方法不再阻塞，没有就绪事件也会返回，此时就会在 for 循环体内部快速的循环。而 Netty 使用一个局部变量 selectCnt 来计算已经执行过的 selector.select 次数。如果次数超过了阀值 SELECTOR_AUTO_REBUILD_THRESHOLD，默认值为 512，Netty 会抛弃当前的 Selector，重新建立一个。其重建 Selector 的方法为 NioEventLoop#selectRebuildSelector，整体的实现思路也不复杂，就不贴代码了，来看整体实现的流程图，如下：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191219135427.png)

重建 selector 对象完毕后，将选择计数 selectCnt 复位为 1，重新开始选择流程。

从 Netty 的实际应用中，通过这种方式，对于线上出现 CPU100% 的情况得到有效的解决。目前这个思路也被 Jetty、UnderTow 等其他网络 IO 框架所采用。

### 自适应大小 ByteBuf 分配算法

在源码篇《Netty 流程细讲之数据读取与连接远端》我们提到，在 EventLoop 线程进行数据读取时，申请的 ByteBuf 容量不够的话，在 socket 缓冲区中的数据就需要多次读取才能读取完毕，降低了程序性能；申请的 ByteBuf 容量太大的话，虽然减少了读取次数，但是浪费了内存空间，对应用的整体内存压力又上去了。找到一个合适的平衡点并不是容易的事情。

在 Netty 中，Netty 采用了策略模式来处理这个问题，其将 ByteBuf 的分配工作交给接口 RecvByteBufAllocator.Handle 来处理。首先来看下这个接口定义的几个方法。

- `allocate(ByteBufAllocator alloc)`：根据内建的策略，使用分配器分配一个合适大小的`ByteBuf`。
- `lastBytesRead(int bytes)`：记录本次读取到的字节数，统计数据内部应用于分配策略。
- `continueReading`：是否应该继续下一轮从通道读取数据。

这个 RecvByteBufAllocator.Handle 对象实例是由 `RecvByteBufAllocator#newHandle` 方法生成，也就是具体采用什么策略，实际上由 RecvByteBufAllocator 接口实现类决定。让我们来看下这个接口的相关类图，如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191219145833.png)

其区别和联系如下：

- DefaultMaxBytesRecvByteBufAllocator：限定一个读取循环（可能执行多次读取操作）读取字节上限 maxBytesPerRead 和每次读取操作读取字节上限 individualReadMax。分配 ByteBuf 大小时，取 individualReadMax 和剩余可读字节数（maxBytesPerRead减去已经读取的总数）的较小值分配。这种策略下，一轮读取的上限是被固定的。
- DefaultMaxMessagesRecvByteBufAllocator：限定一个读取循环中最大读取的消息数。每执行一次读取操作，读取消息数 +1。这种策略下，实际相当于限定了一个读取循环中最多有几个读取操作。
- FixedRecvByteBufAllocator：继承于 DefaultMaxMessagesRecvByteBufAllocator，使用固定大小来执行每一次的ByteBuf分配。
- AdaptiveRecvByteBufAllocator：继承于 DefaultMaxMessagesRecvByteBufAllocator，通过自适应方式来计算最合适的 ByteBuf 分配大小。

默认情况下，Netty 使用 AdaptiveRecvByteBufAllocator 作为在 NioSocketChannel 上的分配器。它也是我们这个章节的主角，自适应大小的 ByteBuf 分配算法。下面就来分析其实现原理。

首先，AdaptiveRecvByteBufAllocator 内部有一个排序好的 int[] 对象，其取值如下：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191219153927.png)

AdaptiveRecvByteBufAllocator 用于分配 ByteBuf 的大小取值就在这个数组中选择。可以看到这个数组有 2 个分段，这个也是考虑到如果对方发送的数据比较小时，递增的幅度就小一些；如果发送的数据更大了，使用 16 作为递增单位则递增效率很低，此时就可以使用 2 倍递增。

接着来看看 AdaptiveRecvByteBufAllocator.HandleImpl 构造方法，如下：

```java
HandleImpl(int minIndex, int maxIndex, int initial)
{
    this.minIndex = minIndex;
    this.maxIndex = maxIndex;
    index = getSizeTableIndex(initial);
    nextReceiveBufferSize = SIZE_TABLE[index];
}
```

方法 getSizeTableIndex 从数组中选择一个数字，该数字大于且最接近于入参数字。返回该数字在数组中的下标。

在 AdaptiveRecvByteBufAllocator 中会定义三个数字：

- minIndex：最小允许的 ByteBuf 分配大小在数组中下标。默认情况下，最小允许的分配大小是 64 字节。
- maxIndex：最大允许的 ByteBuf 分配大小在数组中下标。默认情况下，最大允许的分配大小是 65536 字节。
- initial：首次分配 ByteBuf 的大小。默认情况下为 1024 字节。

AdaptiveRecvByteBufAllocator 会使用这三个参数来初始化 AdaptiveRecvByteBufAllocator.HandleImpl（以下简称 HandleImpl）。从参数命名基本可以猜出实现思路：

> 划定一个大小范围，可以分配的 ByteBuf 大小只能在这个范围内变化。而具体的数字变化，就是通过 index 参数的变化从数组中获取对应的值。而 index 只能在 minIndex 和 maxIndex 之间移动。

在 EventLoop 线程中读取数据的时候，首先会通过 allocate 方法来分配一个 ByteBuf，下面来看下对应的方法：

```java
public ByteBuf allocate(ByteBufAllocator alloc)
{
    return alloc.ioBuffer(guess());
}
public int guess()
{
    return nextReceiveBufferSize;
}
```

可以看到，对 ByteBuf 的分配是依靠属性 nextReceiveBufferSize，不过这个方法中没有对这个属性计算的逻辑。

从通道读取数据的方法是 `NioSocketChannel#doReadBytes`，其代码如下：

```java
protected int doReadBytes(ByteBuf byteBuf) throws Exception
{
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.attemptedBytesRead(byteBuf.writableBytes());
    return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());
}
```

可以看到，在真正读取之前，会通过方法 attemptedBytesRead 记录本次尝试读取的字节数。

从通道读取完毕数据后，会通过方法 lastBytesRead 记录本次读取的字节数，其实现如下：

```java
public void lastBytesRead(int bytes)
{
    if(bytes == attemptedBytesRead())
    {
        record(bytes);
    }
    super.lastBytesRead(bytes);
}
```

如果本次读取到的字节数和预期读取的字节数相等，意味着本次分配的大小可能小了，可能存在更多的数据要读取，因此可以考虑扩大下次分配 ByteBuf 的大小。来看下 record 方法，如下：

```java
private void record(int actualReadBytes)
{
    if(actualReadBytes <= SIZE_TABLE[max(0, index - INDEX_DECREMENT - 1)])
    {
        if(decreaseNow)
        {
            index = max(index - INDEX_DECREMENT, minIndex);
            nextReceiveBufferSize = SIZE_TABLE[index];
            decreaseNow = false;
        }
        else
        {
            decreaseNow = true;
        }
    }
    else if(actualReadBytes >= nextReceiveBufferSize)
    {
        index = min(index + INDEX_INCREMENT, maxIndex);
        nextReceiveBufferSize = SIZE_TABLE[index];
        decreaseNow = false;
    }
}
```

代码有点啰嗦，我们用流程图的形式简化下，如下：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191219171634.png)

总结来说其思想就是如果读取的字节数大于等于预期，则扩容一次；如果读取的字节数小于预期，则判断是否小于指定缩容大小（当前下标左移缩容步长得到的数组的值），第一次小于则设置一个标志位，第二次小于则执行缩容并复位标志位。在缩容上相对谨慎，避免一次接受的数据较少，而后续接受的数据较大又需要扩容导致增加读取次数。

在执行读取的过程中，只有在本次读取与预期读取相等时才会触发这个调整方法，也就是读取过程中只可能存在扩容操作。

而在读取方法完成后，RecvByteBufAllocator.Handle#readComplete 方法会被触发，其内部会以本次读取的总的字节数作为单位调用调整方法。此时的大小很可能是一个消息的大小或者一次读取的完整大小亦或者接近于 socket 缓冲区大小。而如果上一次读取中扩容的较大，那么此时也是一个缩容的机会。

通过这种自适应的算法，一个通道上反复执行读取，可以让预先分配的 ByteBuf 尽可能的贴近于实际的使用需求。

### 属性 addTaskWakesUp 详解

在章节《Netty 线程模型详解：从演进的角度看源码设计》分析 SingleThreadEventExecutor 实现时，曾经提到其内部有一个属性 addTaskWakesUp，其生效的代码在添加任务处，如下：

```java
public void execute(Runnable task)
{
    //省略代码，将任务添加到队列中
    if(!addTaskWakesUp && wakesUpForTask(task))
    {
        wakeup(inEventLoop);
    }
}
protected void wakeup(boolean inEventLoop)
{
    if(!inEventLoop || state == ST_SHUTTING_DOWN)
    {
        taskQueue.offer(WAKEUP_TASK);
    }
}
```

这个属性在代码中的作用不太直观，不好理解。网络上甚少资料会提及这个属性，有提及的基本都分析为” 在添加任务的时候是否唤醒线程”。这个解释是错误的，与代码真实的意图直接相反。

单单看这个属性以其代码难以看出具体的用途，我们来结合其在不同实现中的不同表现来查看。

在 DefaultEventExecutor 实现中，其 run 方法如下：

```java
protected void run()
{
    for(;;)
    {
        Runnable task = takeTask();
        if(task != null)
        {
            task.run();
            updateLastExecutionTime();
        }
        if(confirmShutdown())
        {
            break;
        }
    }
}
```

其适用阻塞队列进行任务的获取，因此当向队列推送任务的时候，这个线程就会从阻塞的 takeTask 方法被唤醒返回。不需要其他的唤醒方式，因此在这个类中，addTaskWakesUp 属性为 true，在 execute 方法提交任务时，wakeup 方法不会被触发。

而在 NioEventLoop 实现中，其 run 方法是阻塞在 Selector 上，此时通过 execute 方法添加任务，EventLoop 也不会从阻塞中恢复来处理任务。因此需要执行下唤醒方法，在 NioEventLoop 中，对 wakeup 进行了重写，如下：

```java
protected void wakeup(boolean inEventLoop)
{
     if(!inEventLoop && wakenUp.compareAndSet(false, true))
     {
         selector.wakeup();
     }
}
```

在 NioEventLoop 中，addTaskWakesUp 的值为 false。

综合两个例子来看，addTaskWakesUp 的作用就很明显了，该属性意味着具体的实现类在任务被添加到队列中，线程自身会是否会自行唤醒（从阻塞中恢复）。如果该属性为 false，则需要执行 wakeup 方法（特殊情况除外，使用 wakesUpForTask 方法判断）。各个实现子类可以通过重写 wakeup 方法实现合适自身的线程唤醒方法。

### 总结与思考

本章节中我们梳理了一些在前面章节中被暂时延后的内容。在 Netty 当中这样的细节考虑还有很多的地方，当然，就不是每一处都有技术的光辉。实际上从整个框架的代码细节也可以看出，实现一个 Demo 式的项目或者框架并不是很难，但是要能稳定的在生产上应用，需要经历非常多事故或者 bug 的打磨。而这些打磨往往不是技术或者经验丰富就能避免的，这也是生产级别的框架和刚发布的框架之间的差异。

经过 5 个章节的 Netty 流程细讲，相信读者已经对 Netty 如何处理连接建立，数据读写等有了相当程度的掌握。下一个章节，我们将梳理网络中编程中一个重要的步骤，解码。通过源码的方式，来学习 Netty 如何处理编解码以及处理拆包粘包的问题。