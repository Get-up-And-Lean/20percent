# 28/31Netty 如何实现对内存泄漏的监控

## 前言

在前面的四个章节中，我们介绍了 Netty 中用于内存管理的内存池算法。Netty4 开始默认都使用内存池用于分配内存空间。而承载池化内存空间的一般都是`PooledByteBuf`对象。使用池化`ByteBuf`可以提高性能，但是使用完毕需要注意是否需要进行手工释放。在需要手动释放的场合没有释放，就会因为申请的内存空间没有归还给内存池造成内存泄漏，最终应用程序OOM。

在复杂的应用程序中找到没有释放的`PooledByteBuf`是一个比较困难的事情，在没有工具辅助的情况下只能白盒检查所有的代码，效率无疑十分低下。

好在 Netty 也考虑到了这种情况，在 Netty 中设计了专门的泄漏检测接口用于对需要手动释放的资源对象进行监控。

## JDK的弱引用和引用队列

在分析Netty的泄露监控功能之前，先来复习下其中会用到的JDK知识：引用。

在java中存在4中引用类型，分别是强引用，软引用，弱引用，虚引用。

**强引用**

强引用，是我们写程序最经常使用的方式。比如将一个值赋给一个变量，那这个对象就被该变量强引用了。除非将将`null`设置给该变量，否则因为该变量一直引用着对象，java的内存回收不会回收该对象。就算是内存不足异常发生也不会。

**软引用**

软引用所引用的对象会在java内存不足的时候，被gc回收。如果gc发生的时候，java的内存还充足则不会回收这个对象 使用的方式如下

```java
SoftReference ref = new SoftReference(new Date());
Date tmp = ref.get(); //如果对象没有被回收，则这个get操作会返回初始化的值。如果被回收了之后，则返回null
```

**弱引用**

弱引用则比软引用更差一些。只要是gc发生的时候，弱引用的对象都会被回收。使用方式上和软引用类似，如下

```java
WeakReference re = new WeakReference(new Date());
re.get();
```

**虚引用**

虚引用和前面的软引用、弱引用不同，它并不影响对象的生命周期。在java中用`java.lang.ref.PhantomReference`类表示。如果一个对象与虚引用关联，则跟没有引用与之关联一样，在任何时候都可能被垃圾回收器回收。

除了强引用之外，其余的引用都有一个引用队列可以与之配合。当java清理调用不必要的引用后，会将这个引用本身（不是引用指向的对象）添加到队列之中。代码如下

```java
ReferenceQueue<Date> queue = new ReferenceQueue<>();
WeakReference<Date> re = new WeakReference<Date>(new Date(), queue);
Reference<? extends Date> moved = queue.poll();
```

软引用是在JVM内存不足的时候才会执行回收，较为适合的使用场景是做JVM实例内缓存。

引用队列的一个适用场景：**与弱引用或虚引用配合，监控一个对象是否被GC回收**。

## Netty的实现思路

针对需要手动关闭的资源对象，Netty设计了一个接口`ResourceLeakTracker`来实现对资源对象的追踪。该接口定义了三个方法，分别如下

```java
void record();//记录当前的调用信息，实际上相当于记录了追踪对象当前的调用轨迹
void record(Object hint);//记录当前的调用信息，并且额外记录了提示信息，也就是hint
boolean close(T trackedObject);//停止追踪trackedObject对象
```

被追踪的资源对象，在手动释放自身的时候都需要调用方法`close`来改变`ResourceLeakTracker`的状态。而在`close`之前，可以通过`record`方法来记录该被追踪的资源对象在代码的什么地方被调用过。

该接口只有唯一一个实现`DefaultResourceLeak`，该实现继承了`WeakReference`。每一个`DefaultResourceLeak`会与一个需要监控的资源对象关联，同时关联着一个引用队列。

当资源对象被GC回收后，与之关联的`DefaultResourceLeak`就会进入引用队列。通过检查引用队列中的`DefaultResourceLeak`实例的状态（`close`方法的调用会导致状态变更），就能确定在资源对象被GC前，是否执行了手动关闭的相关方法，从而判断是否存在泄漏可能。

## 代码实现

### DefaultResourceLeak

首先来看下追踪对象`DefaultResourceLeak`的实现，先查看其拥有的属性，如下

```java
private volatile Record head;//Record对象形成一个后进先出的链表，head属性指向最后一次调用record方法存储的调用记录
private volatile int droppedRecords;//记录被丢弃的record对象个数
private final Set < DefaultResourceLeak <? >> allLeaks; //存储所有追踪对象实例
private final int trackedHash;//被追踪的资源对象的hash值，该值用于在close方法调用时进行比对，避免不正确的执行close
```

再来看下该类的类图，如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20200105213907.png)

可以看到，一方面实现了接口，一方面继承了弱引用，和前面的思路就对的上。再来看看其构造方法，如下

```java
DefaultResourceLeak(Object referent, ReferenceQueue < Object > refQueue, Set < DefaultResourceLeak <? >> allLeaks)
{
    super(referent, refQueue);
    trackedHash = System.identityHashCode(referent);
    allLeaks.add(this);
    headUpdater.set(this, new Record(Record.BOTTOM));
    this.allLeaks = allLeaks;
}
```

上文说接口的时候提到方法`close`，为了避免错误的调用这个方法，因此这个方法的入参是资源对象，该入参的资源对象和追踪对象存储的进行对比，以确保关闭方法是针对一开始追踪的资源对象。但是在构造方法中，我们注意到，没有使用一个变量存储一开始要追踪的资源对象，而是使用了`System.identityHashCode`获取了`hashcode`来作为`close`方法比对的依据。如果这里存储了`referent`，就会导致资源对象无法被GC。因为存在着一个强引用链，如下

> io.netty.buffer.AbstractByteBuf#leakDetector --> io.netty.util.ResourceLeakDetector#allLeaks
>
> -->io.netty.util.ResourceLeakDetector.DefaultResourceLeak --> 资源对象

因此这里只能通过别的方式保留资源对象特征，也就是`hashcode`值。

在`DefaultResourceLeak`的属性中，我们看到一个类`ResourceLeakDetector.Record`。每一次调用`DefaultResourceLeak#record()`方法时，就会生成一个`Record`实例，该实例记录了当前的堆栈信息。我们来看下`Record`的定义，如下

```java
private static final class Record extends Throwable
{
    private static final long serialVersionUID = 6065153674892850720 L;
    private static final Record BOTTOM = new Record();
    private final String hintString;
    private final Record next;
    private final int pos;
}
```

可以看到，`Record`对象继承了`Throwable`，而`Throwable`的默认构造方法中会调用方法`Throwable#fillInStackTrace()`，该方法会通过本地方法调用，将当前的线程堆栈信息生成并填充到`Throwable`对象中，从而保存了该对象被生成时候的堆栈信息。

`Record`继承了`Throwable`并且自身通过`next`形成了一个链表，这实际上暗示了这个链表就是用于存储`DefaultResourceLeak`在各个不同的调用处采集到的调用堆栈信息。

从思路到构造方法，我们可以梳理出`DefaultResourceLeak`的使用方式。当资源对象生成的时候，创建一个`DefaultResourceLeak`并且指向该资源对象。同时资源对象可以指持有`DefaultResourceLeak`对象。资源对象在各个不同地方的代码使用的时候，调用`DefaultResourceLeak`的`record`方法来记录资源对象的使用轨迹。最终在资源对象被GC回收时，在引用队列中查看`DefaultResourceLeak`是否被调用过`close`方法，如果调用过，则说明资源对象已经被释放了；如果没有的话，则意味着资源对象是直接被GC回收的，在回收之前不曾手动释放过，存在内存泄漏的隐患。

### 分配监控对象

看过了`DefaultResourceLeak`的构造方法信息后，也了解了其使用方式，下面我们来看下其具体的生成时机。我们知道在进行内存分配的时候，我们依赖的入口是内存分配器，`ByteBufAllocator#buffer()`类方法。只有池化的分配器，采用了内存池，才需要内存追踪。我们来看其实现，如下

```java
protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity)
{
    PoolThreadCache cache = threadCache.get();
    PoolArena < byte[] > heapArena = cache.heapArena;
    final ByteBuf buf;
    if(heapArena != null)
    {
        buf = heapArena.allocate(cache, initialCapacity, maxCapacity);
    }
    else
    {
        buf = PlatformDependent.hasUnsafe() ? new UnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) : new UnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
    }
    return toLeakAwareBuffer(buf);
}
```

当`ByteBuf`实例生成并且初始化完毕，持有了内存区域后，在返回给开发者使用之前，尝试为其生成一个代理，用于资源泄漏追踪，也就是方法`toLeakAwareBuffer`，代码如下

```java
protected static ByteBuf toLeakAwareBuffer(ByteBuf buf)
{
    ResourceLeakTracker < ByteBuf > leak;
    switch(ResourceLeakDetector.getLevel())
    {
        case SIMPLE:
            leak = AbstractByteBuf.leakDetector.track(buf);
            if(leak != null)
            {
                buf = new SimpleLeakAwareByteBuf(buf, leak);
            }
            break;
        case ADVANCED:
        case PARANOID:
            leak = AbstractByteBuf.leakDetector.track(buf);
            if(leak != null)
            {
                buf = new AdvancedLeakAwareByteBuf(buf, leak);
            }
            break;
        default:
            break;
    }
    return buf;
}
```

根据不同的监控级别生成不同的监控等级对象。Netty对监控分为4个等级：

1. `ResourceLeakDetector.Level#DISABLED`：关闭，这种模式下不进行泄露监控。
2. `ResourceLeakDetector.Level#SIMPLE`：简单，这种模式下以1/128（默认）的概率抽取ByteBuf进行泄露监控。这种监控并不会追踪其调用信息，只是追踪了生成`ByteBuf`时候的堆栈信息。
3. `ResourceLeakDetector.Level#ADVANCED`：增强，抽取概率与简单模式相同；每一次对`ByteBuf`的操作都会记录下堆栈信息，对性能的损耗比较大。
4. `ResourceLeakDetector.Level#PARANOID`：偏执，抽取概率为100%；每一次对`ByteBuf`的操作都会记录下堆栈堆栈信息，对性能的损耗最大。

一般而言，在项目的初期使用简单模式进行监控，如果没有问题一段时间后就可以关闭。否则升级到增强或者偏执模式尝试确认泄露位置。

不同的抽取概率主要是通过方法`ResourceLeakDetector#track`实现的。在 Netty 中，这个实例对象是一个类实例，由属性`AbstractByteBuf#leakDetector`定义，Netty 中也提供了工厂方法来我们来自定义这个类，但是实际中不会用到，这边就不展开这部分代码了。

我们来关注下其`track`实现，如下

```java
public final ResourceLeakTracker < T > track(T obj)
{
    return track0(obj);
}
private DefaultResourceLeak track0(T obj)
{
    Level level = ResourceLeakDetector.level;
    if(level == Level.DISABLED)
    {
        return null;
    }
    if(level.ordinal() < Level.PARANOID.ordinal())//代码①
    {
        if((PlatformDependent.threadLocalRandom().nextInt(samplingInterval)) == 0)
        {
            reportLeak();
            return new DefaultResourceLeak(obj, refQueue, allLeaks);
        }
        return null;
    }
    reportLeak();//代码②
    return new DefaultResourceLeak(obj, refQueue, allLeaks);//代码③
}
```

首先来看**代码①**。如果定义的监控等级为`Simple`或者`Advance`，则按照概率抽取对象进行监控。判断`(PlatformDependent.threadLocalRandom().nextInt(samplingInterval)) == 0`是通过线程内随机数来决定本次是否监控特定对象。`samplingInterval`默认情况下取值为128，这就意味着随机情况下 1 / 128 的概率会对当前对象进行监控。

接着来看**代码②**。如果监控等级为`PARANOID`，则必然对当前对象进行监控了。`reportLeak`方法用于在日志中输出错误信息，也就是内存泄漏的追踪信息。不过这里我们先暂时略过，后文再看。

最后看**代码③**。最终返回了一个`DefaultResourceLeak`对象用于完成对资源对象的追踪。上文介绍了，`DefaultResourceLeak`继承了`WeakReference`，这里这里传递了弱引用指向的对象`obj`和引用队列`refQueue`。至于`allLeaks`，这是一个全局并发安全的`Set`集合，所有的`DefaultResourceLeak`都要加入到这个并发集合中，方便后续控制。

我们再回到方法`toLeakAwareBuffer`上，当`DefaultResourceLeak`生成后，根据当前的监控级别生成不同的`PooledByteBuf`代理对象，其都实现了`ByteBuf`接口，因此在业务应用层面是不会被感知到的。

`SimpleLeakAwareByteBuf`和`AdvancedLeakAwareByteBuf`的作用是一致，区别在于前者只记录了`ByteBuf`创建时的堆栈信息，后者在每一次`ByteBuf`读写的时候都会记录堆栈信息，并且还可以通过`touch`方法主动记录堆栈信息。

### 记录堆栈信息

`SimpleLeakAwareByteBuf`本身不会在运行中记录更多的堆栈信息，因此其持有的`ResourceLeakTracker`对象内部只有一开始对象新建时的堆栈信息。而`AdvancedLeakAwareByteBuf`会记录每一次`ByteBuf`读写时的堆栈信息，我们可以随便抽取一个方法来看，如下

```java
public ByteBuf writeByte(int value)
{
    recordLeakNonRefCountingOperation(leak);
    return super.writeByte(value);
}
static void recordLeakNonRefCountingOperation(ResourceLeakTracker < ByteBuf > leak)
{
    if(!ACQUIRE_AND_RELEASE_ONLY)
    {
        leak.record();
    }
}
```

在每一个操作之前都需要通过方法`recordLeakNonRefCountingOperation`来尝试记录当前操作的堆栈信息。类变量`ACQUIRE_AND_RELEASE_ONLY`由环境变量赋值，默认情况下为`false`，其意义为是否只是在申请和释放`ByteBuf`的时候才记录堆栈信息。

记录堆栈信息，依靠的是方法`ResourceLeakTracker#record()`，其代码如下

```java
public void record()
{
    record0(null);
}
private void record0(Object hint)
{
    if(TARGET_RECORDS > 0)
    {
        Record oldHead;
        Record prevHead;
        Record newHead;
        boolean dropped;
        do {
            if((prevHead = oldHead = headUpdater.get(this)) == null)
            {
                return;
            }
            final int numElements = oldHead.pos + 1;
            if(numElements >= TARGET_RECORDS)
            {
                final int backOffFactor = Math.min(numElements - TARGET_RECORDS, 30);
                if(dropped = PlatformDependent.threadLocalRandom().nextInt(1 << backOffFactor) != 0)
                {
                    prevHead = oldHead.next;
                }
            }
            else
            {
                dropped = false;
            }
            newHead = hint != null ? new Record(prevHead, hint) : new Record(prevHead);
        } while (!headUpdater.compareAndSet(this, oldHead, newHead));//代码⑤
        if(dropped)
        {
            droppedRecordsUpdater.incrementAndGet(this);
        }
    }
}
```

代码相对也比较简单，通过CAS操作，将新的`Record`对象设置为头结点，也就是**代码⑤**。头结点指向的`Record`记着当前链表的长度。一旦长度超过额定阀值，也就是`TARGET_RECORDS`（默认值为 4 ），就开始按照一定的概率抛弃当前的头结点。也就是判断`dropped = PlatformDependent.threadLocalRandom().nextInt(1 << backOffFactor) != 0`处理的内容。链表超度超过阈值越大，则backOffFactor越大。整个表达式给出的含义就是抛弃头结点的概率是（1-1/2min(n-target_record,30)）。也就是链表长度越长，越容易抛弃头结点，这使得链表的长度不会膨胀到太大的程度。

### 输出内存泄漏报告

上文提到一个方法`ResourceLeakDetector#track0`用于对资源对象进行追踪。如果确定会对对象进行追踪时，会调用方法`reportLeak`尝试打印内存泄漏信息。该方法实现如下

```java
private void reportLeak()
{
    if(!logger.isErrorEnabled())
    {
        clearRefQueue();
        return;
    }
    for(;;)
    {
        @SuppressWarnings("unchecked")
        DefaultResourceLeak ref = (DefaultResourceLeak) refQueue.poll();
        if(ref == null)
        {
            break;
        }
        if(!ref.dispose())//代码①
        {
            continue;
        }
        String records = ref.toString();
        if(reportedLeaks.putIfAbsent(records, Boolean.TRUE) == null)//代码②
        {
            if(records.isEmpty())
            {
                reportUntracedLeak(resourceType);
            }
            else
            {
                reportTracedLeak(resourceType, records);
            }
        }
    }
}
boolean DefaultResourceLeak#dispose()
{
    clear();
    return allLeaks.remove(this);
}
```

**代码①**用于确认追踪对象是否已经停止了追踪。如果还未停止追踪就进入了引用队列，意味着其追踪的资源对象在被GC之前并没有调用相关的手动释放方法。

前文《Netty的实现思路》中我们介绍过`ResourceLeakTracker.close`方法，当资源对象将资源释放时，需要调用该方法来停止追踪。比如说在`SimpleLeakAwareByteBuf`中，使用完毕将内存空间归还给内存池时会调用`release`方法，这个代理中其实现的逻辑如下

```java
public boolean release()
{
    if(super.release())
    {
        closeLeak();
        return true;
    }
    return false;
}
private void closeLeak()
{
    boolean closed = leak.close(trackedByteBuf);
    assert closed;
}
public boolean close(T trackedObject)
{
    try
    {
        return close();
    }
    finally
    {
        reachabilityFence0(trackedObject);
    }
}
public boolean close()
{
    if(allLeaks.remove(this))
    {
        clear();
        headUpdater.set(this, null);
        return true;
    }
    return false;
}
```

方法的层级有点深，不过逻辑还是很清晰。首先是调用父类的`release`来完成资源的释放。如果释放成功则调用`closeLeak`来关闭资源追踪。方法委托两次后最终的效果是从全局泄漏追踪对象集合`allLeaks`删除当前的泄漏追踪对象。在删除成功的基础上，将弱引用的引用清空（这个操作不会导致被引用对象的GC，并且GC线程也不会使用这个Java方法来清除引用），将`DefaultResourceLeak`的头结点设置为`null`，标记其已经处于关闭状态。

从上面可以看到，如果`PooledByteBuf`在使用完毕后调用`release`方法归还自身空间给内存池，这个操作本身会被代理到（存在代理的情况下）`DefaultResourceLeak#close()`方法上，进而关闭资源追踪。

而如果一个`PooledByteBuf`没有调用`release`而是被GC了，则意味着其持有的从内存池获取的空间没有归还，造成了实际上的内存泄漏。

在回头去看看**代码②**。当内存泄漏被检测到的情况，将`DefaultResourceLeak`内部的堆栈信息输出成字符串。同一个泄漏情况，生成的字符串是相同的，因此这里使用了一个并发的`Map`用于去重检查。避免重复的输出内存泄漏日志。

当检测级别为`Simple`时，追踪的堆栈信息不足，也就是说生成堆栈字符串是空的；此时后台输出的日志会提示将检测级别开启到`Advance`以上，来确认泄漏的对象的调用轨迹。

### 保持资源对象的引用存活

到上个小节，我们就将Netty中如何实现内存泄漏追踪的思路和实现都分析完毕了。不过这里有个细节还需要额外说明下。

上个章节曾提到的一个方法`reachabilityFence0`。

在JVM的规定中，如果一个实例对象不再被需要，则可以判定为可回收。即使该实例对象的一个具体方法正在执行过程中，也是可以的。更确切一些的说，如果一个实例对象的方法体中，不再需要读取或者写入实例对象的属性，则此时JVM可以回收该对象，即使方法还没有完成。

然而这样会导致一个问题，在`DefaultResourceLeak#close()`方法中，如果`close`方法还没有执行完毕，`trackedObject`对象实例就被 GC 回收了，就会导致`DefaultResourceLeak`对象被加入到引用队列中，从而可能在`reportLeak`方法调用中触发方法`dispose`，假设此时`close`方法才刚开始执行，则`dispose`方法可能返回`true`。程序就会判定这个对象出现了泄露，然而实际上却没有。

要解决这个问题，只需要让`close`方法执行完毕前，让对象不要回收即可。`reachabilityFence0`方法就完成了这个作用。

## 总结与思考

学习过了Netty中内存池的相关知识，也学习了Netty如何监控内存泄漏。相信读者现在对于Netty在内存空间和资源管理方面有了比较深的认识。Netty 做的所有的一切都是为了更优化的应用程序性能。

在Netty中，对性能优化的追求还远远不止于此。下一个章节，我们来看下之前文章中曾提到过的`FastThreadLocal`。