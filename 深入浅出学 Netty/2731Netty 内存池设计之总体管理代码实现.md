# 27/31Netty 内存池设计之总体管理代码实现

### 引言

前文我们介绍了 Netty 中内存池的总体管理相关算法与思路。涉及到的内容包含有：小内存分配、内存块管理、线程缓存。

本文我们将进一步分析其代码实现，帮助读者彻底地掌握 Netty 中内存管理的实现方式。

### 初始化与内存分配器

#### **内存池初始化**

内存池 PollArena 是内存分配的主要切入点。我们先来看下其属性。PoolArena 的属性主要分为三类。

**1. 不可变的实例属性**，用于存储一些基本信息，如下：

- 内存页大小 pageSize。
- 内存页大小的 log2 值 pageShifts。
- 内存块大小 chunkSize。
- 触发内存页等分的子页大小溢出掩码 subpageOverflowMask。
- 存储不同的微小等分大小（从 16 至 512）的子页数组的长度 numTinySubpagePools。该变量为不可变静态类变量，值为 32（ 512/16 ）。
- 存储不同的小等分大小（从 512 至 pageSize）的子页数组的长度 numSmallSubpagePools。

**2. 内存分配相关属性**，如下：

- 用于存储不同等分大小的微小等分级别的子页的数组，tinySubpagePools。该数组上的元素实例仅用于充当链表头结点和通过头结点加锁进而保证链表操作的安全性场景。
- 用于存储不同等分大小的小等分级别的子页的数组，smallSubpagePools。与 tinySubpagePools 作用类似，只不过其管理的大小不同。
- 6 个不同使用率区间的 PoolChunkList 对象。

**3. 用于存储统计信息的属性**，如下：

- 记录微小内存申请与释放次数的 allocationsTiny 和 deallocationsTiny。
- 记录小内存申请与释放次数的 allocationsSmall 和 deallocationsSmall。
- 记录普通内存申请与释放次数的 allocationsNormal 和 deallocationsNormal。
- 记录大内存申请与释放次数的 allocationsHuge 和 deallocationsHuge
- 记录当前使用中的大内存分配消耗的字节数 activeBytesHuge。

比较重要的是第二类属性，第一类属性也是为了第二类属性来服务的。而第三类属性仅仅是用于监控信息展示。下面来看下初始化构造方法，如下。

```java
protected PoolArena(PooledByteBufAllocator parent, int pageSize, int maxOrder, int pageShifts, int chunkSize, int cacheAlignment)
{
    this.parent = parent;
    this.pageSize = pageSize;
    this.maxOrder = maxOrder;
    this.pageShifts = pageShifts;
    this.chunkSize = chunkSize;
    directMemoryCacheAlignment = cacheAlignment;
    directMemoryCacheAlignmentMask = cacheAlignment - 1;
    subpageOverflowMask = ~(pageSize - 1);
    tinySubpagePools = newSubpagePoolArray(numTinySubpagePools);//以入参为数组长度创建 PoolSubPage 数组
    for(int i = 0; i < tinySubpagePools.length; i++)
    {
        tinySubpagePools[i] = newSubpagePoolHead(pageSize);//创建特殊 PoolSubPage 实例，该实例仅用作头结点
    }
    numSmallSubpagePools = pageShifts - 9;
    smallSubpagePools = newSubpagePoolArray(numSmallSubpagePools);
    for(int i = 0; i < smallSubpagePools.length; i++)
    {
        smallSubpagePools[i] = newSubpagePoolHead(pageSize);
    }
    q100 = new PoolChunkList < T > (this, null, 100, Integer.MAX_VALUE, chunkSize);
    q075 = new PoolChunkList < T > (this, q100, 75, 100, chunkSize);
    q050 = new PoolChunkList < T > (this, q075, 50, 100, chunkSize);
    q025 = new PoolChunkList < T > (this, q050, 25, 75, chunkSize);
    q000 = new PoolChunkList < T > (this, q025, 1, 50, chunkSize);
    qInit = new PoolChunkList < T > (this, q000, Integer.MIN_VALUE, 25, chunkSize);
    q100.prevList(q075);
    q075.prevList(q050);
    q050.prevList(q025);
    q025.prevList(q000);
    q000.prevList(null);
    qInit.prevList(qInit);
    List < PoolChunkListMetric > metrics = new ArrayList < PoolChunkListMetric > (6);
    metrics.add(qInit);
    metrics.add(q000);
    metrics.add(q025);
    metrics.add(q050);
    metrics.add(q075);
    metrics.add(q100);
    chunkListMetrics = Collections.unmodifiableList(metrics);
}
```

构造方法按照作用，划分了三个阶段：

- 以构造方法的入参设置不可变的实例属性，诸如 pageSize、maxOrder 等。
- 按照计算的微小等分大小个数创建 tinySubpagePools 数组，按照计算的小等分大小个数创建 smallSubpagePools 数组。并且初始化一个特殊的 PoolSubPage 对象作为数组中每一个元素的值。该 PoolSubPage 不指向某个内存页，仅仅用作链表的头结点。
- 按照使用率划分创建 6 个不同的 PoolChunkList 并且将其连接起来形成一个链表。

内存池类 PoolArena 是一个泛型抽象类，其子类有两个：在堆上进行内存分配的 HeapArena，在直接内存上进行分配的 DirectArena。在堆上进行内存操作实际上就是针对 byte[] 对象，在给定的偏移量和区间范围内对其进行操作。

在直接内存上进行操作则相对复杂一些，因为 JDK 本身并没有提供供开发者了解的 API 支持。但是查看 DirectByteBuffer 对象，却能找到一个用于读写直接内存的类 Unsafe。它基于一个起始位置和给定的偏移量，就可以对特定内存位置的数据进行读写。申请一块直接内存也是通过 Unsafe 的 API 完成，其会返回申请的直接内存的起始位置，这是一个 long 整型。在这个起始位置上使用 Unsafe 提供的 API 就可以完成读写。

具体的读写操作读者可以参看 JDK 中的 DirectByteBuffer。

#### **子页初始化**

前文已经分析过子页的内在逻辑和对内存空间的管理方式，这里我们看下类的数据结构。PoolSubPage 有如下属性：

```java
final PoolChunk < T > chunk;//该子页对应的内存页归属的内存块对象
private final int memoryMapIdx;//该子页对应的内存页在内存块中的二叉树节点坐标
private final int runOffset;//该子页对应的内存页的偏移量
private final int pageSize;//子页大小
private final long[] bitmap;//用于管理子页中每一等分的使用信息
PoolSubpage < T > prev; //用于连接链表中的前向节点
PoolSubpage < T > next; //用于连接链表中的后继节点
boolean doNotDestroy; //该子页是否处于使用中；
int elemSize;//该子页的等分大小；或者说等分后，每一个区间的大小
private int maxNumElems; //该子页等分的区间个数
private int bitmapLength;//管理该子页等分信息的位图的有效长度
private int nextAvail;//下一个可用区间的下标
private int numAvail;//当前可用区间个数
```

子页 PoolSubPage 有两个构造方法，分别对应不同的场景。当这个子页是应用在 PoolArena 子页数组的元素时，此时作为头结点，并不持有内存空间，其作用是在对头结点所在的链表进行操作时，提供加锁对象，供 synchronized 关键字使用。

构造方法为：

```java
PoolSubpage(int pageSize)
{
    chunk = null;
    memoryMapIdx = -1;
    runOffset = -1;
    elemSize = -1;
    this.pageSize = pageSize;
    bitmap = null;
}
```

而当这个子页是用于进行内存分配时，在初始化的时候就需要给出更多的信息，包含有内存页信息等，代码如下：

```java
PoolSubpage(PoolSubpage < T > head, PoolChunk < T > chunk, int memoryMapIdx, int runOffset, int pageSize, int elemSize)
{
    this.chunk = chunk;
    this.memoryMapIdx = memoryMapIdx;
    this.runOffset = runOffset;
    this.pageSize = pageSize;
    bitmap = new long[pageSize >>> 10]; // pageSize/16/64
    init(head, elemSize);
}
void init(PoolSubpage < T > head, int elemSize)
{
    doNotDestroy = true;
    this.elemSize = elemSize;
    if(elemSize != 0)
    {
        maxNumElems = numAvail = pageSize/elemSize;
        nextAvail = 0;
        bitmapLength = maxNumElems >>> 6;
        if((maxNumElems & 63) != 0)
        {
            bitmapLength++;
        }
        for(int i = 0; i < bitmapLength; i++)
        {
            bitmap[i] = 0;
        }
    }
    addToPool(head);
}
```

首先是对几个重要属性的赋值，而后通过 init 方法，将内存页按照给定的等分大小进行等分。init 方法通过 elemSize 等分大小，计算出子页中的等分个数，也就是 maxNumElems，而后再根据这个值，对用于管理每一个等分的使用信息的位图数组 bitmap 进行初始化值，也就是将每一个等分所对应的数组元素设置为 0。

根据 maxNumElems，可以计算得到位图实际有效的元素长度，也就是 bitmapLength。

位图数组 bitmap 不需要每次都创建，只需要在 init 方法中进行初始化即可。而 bitmap 的最大长度自然就是 pageSize/16/64，因为在等分大小为 16 时，等分个数最多，对应的比特位也是最多的，此时位图数组长度最长。其他的等分大小都不会超过它，有效比特位数也会更少，这也就是 bitmapLength 属性的作用了，标识出有效比特位数对应到位图数组中的长度。

#### **线程缓存初始化**

看过了内存池 PoolArena 的初始化，接着我们看看线程缓存 PoolThreadCache 的构成。前文有分析过，线程缓存是一个存储于线程变量（FastThreadLocal，后文细讲，其作用与 ThreadLocal 相同）的数据结构。该数据结构由两类属性构成。

\1. 本线程当前持有的堆内存池 HeapArena，直接内存池 DirectArena。在某个线程中运行的程序需要在内存池上进行内存分配时，则获取当前线程的 PoolThreadCache 对象，然后获取这两个属性执行其分配算法。不过需要明确，这两个内存池对象并非某一个线程独占，可能会被多个线程共享，也就是说，多个 PoolThreadCache 中的 heapArena 属性或 directArena 属性可能指向的是同一个堆内存池或直接内存池。

\2. 本线程内用于存储不同规范大小的内存空间缓存 MemoryRegionCache。PoolThreadCache 对象共有三种不同规范大小（对应内存池中的规范大小划分），2 个不同内存类型（堆和直接内存）合计共 6 个 MemoryRegionCache[] 对象，分别如下：

```
private final MemoryRegionCache<byte[]>[] tinySubPageHeapCaches;
private final MemoryRegionCache<byte[]>[] smallSubPageHeapCaches;
private final MemoryRegionCache<ByteBuffer>[] tinySubPageDirectCaches;
private final MemoryRegionCache<ByteBuffer>[] smallSubPageDirectCaches;
private final MemoryRegionCache<byte[]>[] normalHeapCaches;
private final MemoryRegionCache<ByteBuffer>[] normalDirectCaches;
```

MemoryRegionCache 本质上是一个队列，其元素 Entry 存储着三个重要信息：缓存起来的内存空间所归属的 PoolChunk，缓存起来的内存空间的坐标信息（long 型变量，携带子页位图坐标和节点坐标信息），缓存起来的供复用的内存空间对应的 ByteBuffer 对象。

看过属性之后我们来看初始化方法，如下：

```java
PoolThreadCache(PoolArena < byte[] > heapArena, PoolArena < ByteBuffer > directArena, int tinyCacheSize, int smallCacheSize, int normalCacheSize, int maxCachedBufferCapacity, int freeSweepAllocationThreshold)
{
    checkPositiveOrZero(maxCachedBufferCapacity, "maxCachedBufferCapacity");
    this.freeSweepAllocationThreshold = freeSweepAllocationThreshold;
    this.heapArena = heapArena;
    this.directArena = directArena;
    if(directArena != null)
    {
        tinySubPageDirectCaches = createSubPageCaches(tinyCacheSize, PoolArena.numTinySubpagePools, SizeClass.Tiny);
        smallSubPageDirectCaches = createSubPageCaches(smallCacheSize, directArena.numSmallSubpagePools, SizeClass.Small);
        numShiftsNormalDirect = log2(directArena.pageSize);
        normalDirectCaches = createNormalCaches(normalCacheSize, maxCachedBufferCapacity, directArena);
        directArena.numThreadCaches.getAndIncrement();
    }
    else
    {
        //省略代码，不设置直接内存相关属性
    }
    if(heapArena != null)
    {
        tinySubPageHeapCaches = createSubPageCaches(tinyCacheSize, PoolArena.numTinySubpagePools, SizeClass.Tiny);
        smallSubPageHeapCaches = createSubPageCaches(smallCacheSize, heapArena.numSmallSubpagePools, SizeClass.Small);
        numShiftsNormalHeap = log2(heapArena.pageSize);
        normalHeapCaches = createNormalCaches(normalCacheSize, maxCachedBufferCapacity, heapArena);
        heapArena.numThreadCaches.getAndIncrement();
    }
    else
    {
        //省略代码，不设置堆内存相关属性
    }
    //省略代码，参数 freeSweepAllocationThreshold 是否有效且为正数检查
}
```

初始化代码内容虽然长，但是可以分为清晰的两段：赋值三个基本属性有：freeSweepAllocationThreshold、heapArena、directArena；根据 directArena 和 heapArena 是否为空设置对应的属性。

directArena 和 heapArena 自不用说，前文已经介绍过了。freeSweepAllocationThreshold 是一个新加入的属性，其作用在于当在 PoolThreadCache 执行了 freeSweepAllocationThreshold 次内存分配后，检查所有的 MemoryRegionCache 对象，执行 trim（整理）操作，将在链表中但是未曾参与过内存申请的空间归还给内存池。

接下来的两个 `if else` 结构相似，逻辑相似，只不过是按照内存类型的不同对不同的属性进行赋值。这里以 directArena 为例说明。

- 首先是按照微小等分的不同规范大小创建 MemoryRegionCache[] 对象，即 tinySubPageDirectCaches；
- 然后是按照小等分的不同规范大小创建 MemoryRegionCache[] 对象，即 smallSubPageDirectCaches；
- 最后是按照需要缓存的普通大小的上限创建 MemoryRegionCache[]，即 normalDirectCaches。

首先来看方法 createSubPageCaches，如下：

```java
private static < T > MemoryRegionCache < T > [] createSubPageCaches(int cacheSize, int numCaches, SizeClass sizeClass)
{
    if(cacheSize > 0 && numCaches > 0)
    {
        MemoryRegionCache < T > [] cache = new MemoryRegionCache[numCaches];
        for(int i = 0; i < cache.length; i++)
        {
            cache[i] = new SubPageMemoryRegionCache < T > (cacheSize, sizeClass);
        }
        return cache;
    }
    else
    {
        return null;
    }
}
```

cacheSize 意味着 MemoryRegionCache 中队列的长度，也就是其最大缓存的内存空间个数。numCaches 是创建的 MemoryRegionCache[] 的长度，也就意味着存在 numCaches 个规范大小需要首先从 PoolThreadcache 中分配。

对于微小、小、普通三种大小的 cacheSize 默认情况下由以下三个类变量决定：

```
PooledByteBufAllocator#DEFAULT_TINY_CACHE_SIZE
PooledByteBufAllocator#DEFAULT_SMALL_CACHE_SIZE
PooledByteBufAllocator#DEFAULT_NORMAL_CACHE_SIZE
```

这三个变量的取值也可以被环境参数所改变，具体可以参看对应的定义代码。

numCaches 实际上就对应需要缓存处理的规范化大小的个数。这意味着对 tinySubPageDirectCaches 而言，数组长度也就是规范化大小的个数和 PoolArena.numTinySubpagePools 相等，是一个定值。对 smallSubPageDirectCaches 而言，数组长度和 directArena.numSmallSubpagePools 相等。也就是说能够在内存池中子页数组中寻找到的分配大小，都可以在 PoolThreadCache 找到对应的缓存空间，进而在其中的链表里寻找可以使用的复用内存空间。

而对于 normalDirectCaches，情况就会变得复杂一些。来看方法 createNormalCaches，如下：

```java
private static < T > MemoryRegionCache < T > [] createNormalCaches(int cacheSize, int maxCachedBufferCapacity, PoolArena < T > area)
{
    if(cacheSize > 0 && maxCachedBufferCapacity > 0)
    {
        int max = Math.min(area.chunkSize, maxCachedBufferCapacity);
        int arraySize = Math.max(1, log2(max/area.pageSize) + 1);
        MemoryRegionCache < T > [] cache = new MemoryRegionCache[arraySize];
        for(int i = 0; i < cache.length; i++)
        {
            cache[i] = new NormalMemoryRegionCache < T > (cacheSize);
        }
        return cache;
    }
    else
    {
        return null;
    }
}
```

可以看到，数组的长度是由最大缓存内存空间大小 maxCachedBufferCapacity 决定的。normalDirectCaches 数组的长度，是从 pageSize 到 maxCachedBufferCapacity（包含），按照 2 倍递增的规范化大小的个数。默认情况下，maxCachedBufferCapacity 的取值由类属性决定：

```
PooledByteBufAllocator#DEFAULT_MAX_CACHED_BUFFER_CAPACITY
```

默认为 32k，pageSize 默认为 8k，这意味着 normalDirectCaches 默认情况下长度为 3。

需要注意的是，如果 PoolThreadCache 在构造方法中传入的 tinyCacheSize、smallCacheSize，normalCacheSize 为 0，则意味着实际上关闭了线程缓存的功能。此时 PoolThreadCache 的作用仅仅就是提供本线程进行内存分配需要使用的堆内存池 heapArena 和直接内存池 directArena 对象。

#### **内存分配器**

前文有分析过在内存块上的内存分配算法，但是内存分配的入口并不是内存块，而是内存分配器，也就是接口 ByteBufAllocator。不同的接口实现有着不同的分配策略，先来看下接口的类图，如下

之前已经分析过了在内存块 PoolChunk 中，如何进行内存分配的。但是内存分配的入口并不是内存块，而是内存分配器，也就是接口 ByteBufAllocator。不同的接口实现有着不同的分配策略，先来看下接口的类图，如下：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191231232405.png)

从实现类的类名就可以很清楚的看到分配策略有两种：池化就是指的是从内存池中进行分配，非池化就是每次分配时都创建新的内存空间。内存空间也完全依靠 GC 来实现回收，也就是早期的传统做法。

这里我们主要关注池化的分配器 PooledByteBufAllocator，该类定义了一系列内存池实现需要用的默认参数，主要的有：

- 内存页大小 DEFAULT_PAGE_SIZE。
- 内存池二叉树最大深度 DEFAULT_MAX_ORDER。
- 线程缓存中微小内存缓存个数 DEFAULT_TINY_CACHE_SIZE。
- 线程缓存中小内存缓存个数 DEFAULT_SMALL_CACHE_SIZE。
- 线程缓存中普通内存缓存个数 DEFAULT_NORMAL_CACHE_SIZE。
- 线程缓存中普通内存最大的缓存内存大小 DEFAULT_MAX_CACHED_BUFFER_CAPACITY。
- 线程缓存中触发缓存整理的分配阀值 DEFAULT_CACHE_TRIM_INTERVAL
- 线程缓存中周期触发缓存整理的时间间隔 DEFAULT_CACHE_TRIM_INTERVAL_MILLIS，为 0 时意味着关闭该功能，默认为 0。
- 在 PoolChunk 默认缓存的供复用 ByteBuffer 对象个数 DEFAULT_MAX_CACHED_BYTEBUFFERS_PER_CHUNK。

除此之外，最为重要的就是定义了类型为 `PoolArena<byte[]>[]` 的堆内存池数组 heapArneas 属性，以及类型为 `PoolArena<ByteBuffer>[]` 的直接内存数组 directArenas 数组。这两个数组的长度默认都是 2 倍的 cpu 核数。与 NioEventLoopGroup 的默认大小相同。

这样做的目的就是为了在默认情况下，让一个 NioEventLoop 线程拥有一个实际上没有竞争的 PoolArena 对象，以减少在该对象上的同步开销。具体而言，就是线程缓存中使用的 heapArena 和 directArena 是从这两个数组中分配的，当有新的线程产生时，就会有新的线程缓存 PoolThreadCache 对象被创建，该对象会从 heapArenas 数组和 directArenas 数组中寻找分配给 PoolThreadCache 次数最少的 PoolArena。这种分配方式，可以让 PoolArena 尽可能均匀的分配到 PoolThreadCache 上。

### 内存申请

#### **获取本线程持有的 PoolArena 执行分配**

介绍过了内存池，线程缓存和内存分配器。下面我们从内存分配的入口，ByteBufAllocator 来看下内存分配的完整流程。来看看其对 buffer 方法的实现，如下：

```java
public ByteBuf buffer()
{
    if(directByDefault)//当创建 ByteBuf 时，是否优先使用直接内存。默认为 true。
    {
        return directBuffer();
    }
    return heapBuffer();
}
public ByteBuf directBuffer()
{
    return directBuffer(DEFAULT_INITIAL_CAPACITY, DEFAULT_MAX_CAPACITY);
}
public ByteBuf directBuffer(int initialCapacity, int maxCapacity)
{
    if(initialCapacity == 0 && maxCapacity == 0)
    {
        return emptyBuf;
    }
    validate(initialCapacity, maxCapacity);
    return newDirectBuffer(initialCapacity, maxCapacity);
}
```

buffer 方法中会根据是否优先直接内存属性，来判断需要创建的是堆内存 ByteBuf 还是直接内存 ByteBuf。之前我们曾经提到过，对于直接内存的操作，需要 SUN 内部的 API 才能进行，因此 Netty 会判断是否存在这一 API 的可使用情况。只有在 API 能够使用时，才会优先使用直接内存进行内存分配。

使用直接内存的好处也很明显，在 Socket 通道上进行数据操作时，如果是堆内存，则系统的 API 在内部会自行申请一个同等大小的直接内存空间，并且进行数据复制，而后使用这个直接内存进行真正的数据操作。这意味着使用堆内存会带来两个开销，一个是数据拷贝的开销，一个是一次性的直接内存申请创建开销。为此，在能够使用直接内存的场合，优先使用直接内存，有利于系统的整体性能表现。

在这里我们选择默认情况，也就是直接内存 ByteBuf 进行分析。方法随后委托给了无参 directBuffer 方法，该方法主要通过默认属性给出了有参 directBuffer 所需要的 ByteBuf 容量的初始大小和上限大小。前文我们提到过，ByteBuf 是可以在使用中按照需要自动扩容的，但是扩容也需要给出一个上限，避免无限制的膨胀。之后委托到了真正的创建 ByteBuf 实例的方法 newDirectByteBuffer。其内容如下：

```java
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity)
{
    PoolThreadCache cache = threadCache.get();
    PoolArena < ByteBuffer > directArena = cache.directArena;
    final ByteBuf buf;
    if(directArena != null)
    {
        buf = directArena.allocate(cache, initialCapacity, maxCapacity);
    }
    else
    {
        buf = PlatformDependent.hasUnsafe() ? UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) : new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
    }
    return toLeakAwareBuffer(buf);
}
```

threadCache 是一个 FastThreadLocal 类型的线程变量，只是用于存储本线程的 PoolThreadCache 对象。调用方法 cache.directArena 获取本线程持有的 directArena 对象，而后在其上调用 allocate 方法进行内存分配。

#### **在内存池上执行内存分配**

```
PoolArena#allocate(PoolThreadCache, int, int)
```

方法的功能是获取 ByteBuf 对象，并且分配内存空间对其进行初始化，具体代码如下：

```java
PooledByteBuf < T > allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity)
{
    PooledByteBuf < T > buf = newByteBuf(maxCapacity);
    allocate(cache, buf, reqCapacity);
    return buf;
}
```

newByteBuf 方法返回一个可用的 PooledByteBuf 实例。PooledByteBuf 本身持有一些元数据信息，只是为内存空间提供了一个壳，因此每次都创建一个新的实例没有太大必要。为了提高程序性能，Netty 将 PooledByteBuf 实例纳入对象池管理，对象池可以将对象实例暂存起来进行复用，减少了一些频繁使用的对象的创建开销。关于对象池的原理和实现，我们会在后文开单张进行分析。

有了 PooledByteBuf 实例后，就是分配空间并且将空间初始化给 PooledByteBuf 了，来看 allocate 方法，如下：

```java
private void allocate(PoolThreadCache cache, PooledByteBuf < T > buf, final int reqCapacity)
{
    final int normCapacity = normalizeCapacity(reqCapacity);//代码①
    if(isTinyOrSmall(normCapacity))
    {
        //省略代码：微小或者小内存分配处理逻辑
    }
    if(normCapacity <= chunkSize)
    {
       //省略代码：普通内存分配处理逻辑
    }
    else
    {
        allocateHuge(buf, reqCapacity);
    }
}
```

首先来看**代码①**。方法 normalizeCapacity 是对申请大小 reqCapacity 进行规范化，其方法内部实现就是纯粹数学计算，这里不贴出来了。规范化的结果就是如果 reqCapacity 小于 512，则返回一个最接近且大于它的 16 的倍数；如果 reqCapacity 大于 512，则返回一个最接近且大于它的 2 的次方幂值。

有了规范化大小后，针对三种大小规格：微小或者小内存申请，普通内存请求，大内存请求分别有不同的处理处理。

**1. 微小或者小内存申请**

这部分逻辑的代码如下：

```java
int tableIdx;
PoolSubpage < T > [] table;
boolean tiny = isTiny(normCapacity);
if(tiny)
{
    if(cache.allocateTiny(this, buf, reqCapacity, normCapacity))
    {
        return;
    }
    tableIdx = tinyIdx(normCapacity);//代码①
    table = tinySubpagePools;
}
else
{
    if(cache.allocateSmall(this, buf, reqCapacity, normCapacity))
    {
        return;
    }
    tableIdx = smallIdx(normCapacity);
    table = smallSubpagePools;
}
final PoolSubpage < T > head = table[tableIdx];
synchronized(head)//代码②
{
    final PoolSubpage < T > s = head.next;
    if(s != head)
    {
        assert s.doNotDestroy && s.elemSize == normCapacity;
        long handle = s.allocate();
        assert handle >= 0;
        s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity);//代码③
        incTinySmallAllocation(tiny);
        return;
    }
}
synchronized(this)
{
    allocateNormal(buf, reqCapacity, normCapacity);//代码④
}
incTinySmallAllocation(tiny);
```

首先是区分规范大小是属于微小内存申请还是小内存申请，不过两者的处理逻辑是相同的，只是选择的数据结构有差别罢了。

**获取规范大小对应的子页数组的下标**

以微小内存申请为例，首先是检查线程缓存中是否有对应大小的内存空间可以复用，如果有的话，直接从线程缓存中分配。线程缓存的分配我们后续单独说明，这里先略过。如果线程缓存中没有可以分配的复用空间，来看**代码①**。通过 tinyIdx 方法确定规范大小对应在 tinySubpagePools 数组中的下标。该数组元素是子页链表的头结点，子页链表中的元素除开头结点，均持有了相同大小的内存空间，其内存空间大小正是规范大小。

小内存申请和微小内存也是相同的逻辑思路，只不过子页数组换成了 smallSubpagePools。

**在子页上执行内存分配**

接着来看**代码②**。在操作子页链表之间，首先对头结点加锁，因为后续的内存分配动作是不能并发的。子页链表中存储的 PoolSubPage 都是有可分配空间的。一个子页如果全部空间都被分配后，会将自身从链表中删除。深入子页的分配，我们来看方法 `PoolSubpage#allocate`，如下：

```java
long allocate()
{
    if(elemSize == 0)
    {
        return toHandle(0);
    }
    if(numAvail == 0 || !doNotDestroy)
    {
        return -1;
    }
    final int bitmapIdx = getNextAvail();
    int q = bitmapIdx >>> 6;
    int r = bitmapIdx & 63;
    assert(bitmap[q] >>> r & 1) == 0;
    bitmap[q] |= 1 L << r;
    if(--numAvail == 0)
    {
        removeFromPool();
    }
    return toHandle(bitmapIdx);
}
```

allocate 方法只会是处于 PoolArena 子页链表中的元素才会被调用，而在链表中的元素必然都有可以分配的剩余空间，因此前两个 if 可以忽略。

来看看方法 getNextAvail，如下：

```java
private int getNextAvail()
{
    int nextAvail = this.nextAvail;
    if(nextAvail >= 0)
    {
        this.nextAvail = -1;
        return nextAvail;
    }
    return findNextAvail();
}
private int findNextAvail()
{
    final long[] bitmap = this.bitmap;
    final int bitmapLength = this.bitmapLength;
    for(int i = 0; i < bitmapLength; i++)
    {
        long bits = bitmap[i];
        if(~bits != 0)
        {
            return findNextAvail0(i, bits);
        }
    }
    return -1;
}
```

nextAvail 属性的值指向下一个可用的等分空间的位图坐标。该值会在被分配的内存空间归还到子页中时设置，可以看成是一个寻找可用等分空间位图坐标的快捷方式。如果 nextAvail 大于等于 0，意味着值有效，则返回该值并且将 nextAvail 设置为 -1，标识其无效化。而如果 nextAvail 本身就是 -1，则按照常规方式寻找可用空间的位图坐标，也就是方法 findNextAvail。

一个 bits 管理着 64 个等分空间的信息，如果全部使用，则 bits 的值应该是 0XFFFFFFFFFFFFFFFF，对应的 `~bits` 的值就是 0，因此在非零的情况下，意味着可能存在可用的空间（如果是位图数组有效长度的最后一个元素，则实际管理的区间可能不足 64 个，此时 `~bits` 为 0 也不意味着存在可用空间）。方法 findNextAvail0 的实现就是按照位图的方式根据 i 和 bits 信息返回可用空间的位图下标。如果 bits 代表的位图数组有效长度的最后一个元素，则可能出现实际上没有可用空间的情况，此时返回 -1。不过上面已经分析过，在链表中的子页，必然存在可以分配的区间，因此实际上必然会返回一个非负数的位图下标。

getNextAvail 方法返回可用空间的位图下标后，通过位操作，将对应的位图下标设置为 1，标识该空间已经被使用。而后将 numAvail 属性减 1，如果此时已经为 0，则意味着子页中再无可分配的空间，则通过方法 removePool 将该子页从链表中删除。

最后通过方法 toHandle 将位图坐标和子页空间本身的二叉树节点坐标整合在一起，整合方法很简单，使用一个 long 整型表达，高 32 位是位图坐标，低 32 位是二叉树节点坐标。由于位图坐标中存在 0 这个有效值，为了和没有位图坐标这个情况区分开，因此高 32 位的值是 0x40000000 与位图坐标值并操作之后的值。

到这里，在子页上进行内存空间的分配的操作就完成了。现在再让我们回到主方法上，也就是**代码③**。方法 initBufWithSubpage 内容如下：

```java
void initBufWithSubpage(PooledByteBuf < T > buf, ByteBuffer nioBuffer, long handle, int reqCapacity)
{
    initBufWithSubpage(buf, nioBuffer, handle, bitmapIdx(handle), reqCapacity);
}
private void initBufWithSubpage(PooledByteBuf < T > buf, ByteBuffer nioBuffer, long handle, int bitmapIdx, int reqCapacity)
{
    int memoryMapIdx = memoryMapIdx(handle);
    PoolSubpage < T > subpage = subpages[subpageIdx(memoryMapIdx)];
    buf.init(this, nioBuffer, handle, runOffset(memoryMapIdx) + (bitmapIdx & 0x3FFFFFFF) * subpage.elemSize + offset, reqCapacity, subpage.elemSize, arena.parent.threadCache());
}
```

代码本身不复杂，就是通过内存地址信息 handle，计算出该内存空间所处的二叉树节点坐标，位图坐标，子页等分大小也即可用空间大小，空间起始偏移量等信息，将这些信息传递给 ByteBuf 实例，即可初始化 ByteBuf。后续 ByteBuf 就可以在这个内存空间上执行读写操作。

**创建新的叶子并且进行内存分配**

如果规范大小对应的子页数组元素对应的链表没有可以分配空间的子页，则分配一个新的叶子节点的空间并且子页化后进行内存分配，也就是**代码④**。

方法 allocateNormal 会尝试在内存池按照正常大小分配一个内存空间，而如果实际申请的规范大小小于内存页大小的话，则会将其子页化并且在其上分配最终所需的内存空间。这个方法本身在普通大小内存申请中也会用到，因此这里先不剖析其内部流程，我们直接看其最终委托给内存块 PoolChunk 的分配方法 `PoolChunk#allocate`。

```java
boolean allocate(PooledByteBuf < T > buf, int reqCapacity, int normCapacity)
{
    final long handle;
    if((normCapacity & subpageOverflowMask) != 0)
    {
        handle = allocateRun(normCapacity);
    }
    else
    {
        handle = allocateSubpage(normCapacity);
    }
    if(handle < 0)
    {
        return false;
    }
    ByteBuffer nioBuffer = cachedNioBuffers != null ? cachedNioBuffers.pollLast() : null;
    initBuf(buf, nioBuffer, handle, reqCapacity);
    return true;
}
```

考虑到此时是在申请微小或者小内存空间，因此 `normCapacity & subpageOverflowMask` 的计算结果必然为 0。分配继续委托给方法 allocateSubpage，如下：

```java
private long allocateSubpage(int normCapacity)
{
    PoolSubpage < T > head = arena.findSubpagePoolHead(normCapacity);
    int d = maxOrder;
    synchronized(head)
    {
        int id = allocateNode(d);
        if(id < 0)
        {
            return id;
        }
        final PoolSubpage < T > [] subpages = this.subpages;
        final int pageSize = this.pageSize;
        freeBytes -= pageSize;
        int subpageIdx = subpageIdx(id);
        PoolSubpage < T > subpage = subpages[subpageIdx];
        if(subpage == null)
        {
            subpage = new PoolSubpage < T > (head, this, id, runOffset(id), pageSize, normCapacity);
            subpages[subpageIdx] = subpage;
        }
        else
        {
            subpage.init(head, normCapacity);
        }
        return subpage.allocate();
    }
}
```

这个方法最终会生成一个新的子页对象并且将子页对象添加到内存池的子页链表中，因此这里首先需要用规范大小找到对应的子页链表的头结点，也就是方法 arena.findSubpagePoolHead 的作用。

找到对应子页链表的头结点并且加锁成功后就可以开始在内存块上申请一个内存页。因为子页化只能发生在内存页上，所以这里直接 `allocateNode(maxOrder)` 来申请一个内存页，allocateNode 方法在内存块的内存分配章节已经分析过，忘记了读者可以找前文复习下。

内存页申请成功后，根据内存页的下标，计算出在 PoolChunk 的子页数组的下标，获得该数组元素。如果该数组元素为空，则创建 PoolSubPage 对象，并且使用内存页节点下标，申请规范化大小来初始化子页信息；如果该数组元素不为空，则可以直接复用该对象，使用规范化大小初始化子页信息。PoolChunk 中的子页数组的长度和其本身的内存页的个数是一致的，因为如果一个 PoolChunk 全部的内存页都用于子页化，此时子页数组长度最长，也就是和内存页个数相等了。PoolSubPage 本身只是一个壳，所以可以用数组存储起来，供反复使用。

在子页数组上的每一个子页都唯一的对应了一个内存页，因此在复用的时候只需要提供本次等分大小即可初始化，而该子页对应的内存空间，内存页节点下标，内存偏移量等，都是不变的。

无论是新建子页亦或者再次初始化子页，都会将该子页对象加入到内存池的对应等分大小的子页链表中。这也是在这边传递子页链表头节点的作用。

子页初始化完毕后，调用 `PoolSubpage#allocate` 分配一个微小或者小内存空间。这个方法在上文中已经分析过。

子页分配内存空间成功后，返回了携带内存空间信息的 long 变量，ByteBuf 实例使用这个内存空间坐标信息完成初始化，由于内存空间坐标信息携带了位图部分的信息，因此初始化方法最终委托到 initBufWithSubpage，这个方法在上个章节已经分析过了，这里略去。

这里有一个可以**优化的小点**，在需要对子页链表进行操作时，才需要对链表的头结点加锁。而在上面的代码中，从内存块中申请一个内存页，并且子页化化这个过程，实际上并不需要对头节点加锁。可以将加锁范围缩小到 subpage.allocate() 代码的前后。减少了加锁的范围，意味着更好的并发度。

**2. 普通大小内存申请**

讲完了微小内存或者小内存申请后，我们来看下普通大小内存的做法，该部分的代码如下：

```java
if(cache.allocateNormal(this, buf, reqCapacity, normCapacity))
{
    return;
}
synchronized(this)
{
    allocateNormal(buf, reqCapacity, normCapacity);
    ++allocationsNormal;
}
```

首先仍然是在线程缓存上尝试分配，这里依然先略过，后续专门分析。

对内存池 PoolArena 加锁，而后调用方法 allocateNormal 尝试分配一个普通大小的内存空间，来看具体的代码。

```java
private void allocateNormal(PooledByteBuf < T > buf, int reqCapacity, int normCapacity)
{
    if(q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) || q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) || q075.allocate(buf, reqCapacity, normCapacity))
    {
        return;
    }
    PoolChunk < T > c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
    boolean success = c.allocate(buf, reqCapacity, normCapacity);
    assert success;
    qInit.add(c);
}
```

可以看到，Netty 尝试在几个不同使用率区间的 PoolChunkList 内寻找 PoolChunk 进行分配尝试，如果任意成功则会将 ByteBuf 初始化完毕并返回。

如果所有的 PoolChunkList 都没有足够空间的 PoolChunk 可以分配空间，则新建一个 PoolChunk 进行分配，此时必然成功。

Netty 对 PoolChunkList 的选择顺序值得思考一番，首先是尝试使用率在 50%~75%，其次是 25%~50%，再次是 1%~25%，最后是 75%~100%。这种顺序，会尽可能的提升 PoolChunk 的使用率。至于为何不将 1%~25% 的 PoolChunkList 放在第一个分配，是考虑在这个链表中的内存块在回收了内存空间后，有较大的概率可以归零进而完全释放自身，因此放在靠后的位置进行分配。

newChunk 是根据参数创建一个新的内存块，对应的构造方法在《Netty 内存池设计之内存申请代码实现》已经分析过，这里不再赘述。

新创建的内存块在内存分配完毕后，加入到 qInit 链表，不过链表的 add 方法内部会判断使用率是否在本链表管理的区间，如果超出了，则会协助其添加到下一个链表去，代码很简单，这里就不展开。

PoolChunkList 的 allocate 方法实现也很简单，从链表头开始，遍历每一个 PoolChunk，在其上调用 allocate 方法来判断，如果成功则停止，并且判断 PoolChunk 的使用率，如果低于当前 PoolChunkList 的下线阀值，则移动到前驱的 PoolChunkList 中。

PoolChunk.allocate 方法在上个章节《创建新的叶子并且进行内存分配》分析过，不过当其在申请普通大小内存时，寻找空间阶段，会执行到方法 allocateRun，该方法在内存块的内存分配中已经分析过，这里不再赘述。

**3. 大内存申请**

如果规范大小大于内存块大小，则创建一个所需大小的临时内存块，用于保证 ByteBuf 初始化一致的 API。其本身不会被内存池所缓存，是一个一次性的消耗品，当 ByteBuf 使用完毕，调用 release 方法时，就会执行自身的 destory 方法将自身销毁，将内存归还给 JVM。

来看 allocateHuge 方法，如下：

```java
private void allocateHuge(PooledByteBuf < T > buf, int reqCapacity)
{
    PoolChunk < T > chunk = newUnpooledChunk(reqCapacity);
    activeBytesHuge.add(chunk.chunkSize());
    buf.initUnpooled(chunk, reqCapacity);
    allocationsHuge.increment();
}
```

通过 newUnpooledChunk 创建了一个 unpooled 属性为 true 的 PoolChunk 对象，并且初始化给 ByteBuf。

### 内存释放

#### **ByteBuf 释放自身**

内存释放是通过方法 `ReferenceCounted#release()` 完成，ByteBuf 接口继承了这一接口，该方法的具体提供者是 `AbstractReferenceCountedByteBuf#release()`，代码如下：

```java
public boolean release()
{
    return handleRelease(updater.release(this));
}
private boolean handleRelease(boolean result)
{
    if(result)
    {
        deallocate();
    }
    return result;
}
```

updater 是 Netty 的一个工具类 ReferenceCountUpdater 的具体子类，用于对对象进行引用计数。当调用 release 方法时，会将引用计数减 1。如果引用计数归零，则意味着可以安全释放掉对象。也就是调用到了 deallocate 方法。该方法由 PooledByteBuf 提供实现，代码如下：

```java
protected final void deallocate()
{
    if(handle >= 0)
    {
        final long handle = this.handle;
        this.handle = -1;
        memory = null;
        chunk.arena.free(chunk, tmpNioBuf, handle, maxLength, cache);
        tmpNioBuf = null;
        chunk = null;
        recycle();
    }
}
```

在使用中的 ByteBuf，其 handle 必然大于等于 0。上面的方法，就是完成了三件事：

- 将自身一些属性设置为 null 或者表示不使用的特定值，比如将 handle 设置为 -1。
- 通过 PoolArena.free 方法归还内存空间和可复用的 ByteBuffer 对象。
- 通过 recycle 方法回收自身实例，供后续复用。

recycle 涉及到 Netty 中的对象缓存池，我们后续单独说明。将自身属性设置为不可用的数据状态值后，剩余的内容就是依托内存池进行空间释放。也就是 PoolArena.free 方法的内容。

#### **归还空间至内存池**

通过 PoolArena.free 方法，将使用的内存空间归还给内存池，来看其具体方法，如下：

```java
void free(PoolChunk < T > chunk, ByteBuffer nioBuffer, long handle, int normCapacity, PoolThreadCache cache)
{
    if(chunk.unpooled)
    {
        int size = chunk.chunkSize();
        destroyChunk(chunk);
        activeBytesHuge.add(-size);
        deallocationsHuge.increment();
    }
    else
    {
        SizeClass sizeClass = sizeClass(normCapacity);
        if(cache != null && cache.add(this, chunk, nioBuffer, handle, normCapacity, sizeClass))
        {
            return;
        }
        freeChunk(chunk, handle, sizeClass, nioBuffer, false);
    }
}
```

如果 PoolChunk 是一个大内存的内存块，其 unpooled 属性为 true，则直接调用 destory 方法销毁内存块，归还其内存空间给 JVM。

其余情况的话，则首先尝试将内存空间放入线程缓存中，这里暂时忽略。

如果线程缓存放不下或者线程缓存本身没有开启，则调用 freeChunk 方法来归还空间给内存块，代码如下：

```java
void freeChunk(PoolChunk < T > chunk, long handle, SizeClass sizeClass, ByteBuffer nioBuffer, boolean finalizer)
{
    final boolean destroyChunk;
    synchronized(this)
    {
        if(!finalizer)
        {
            switch(sizeClass)
            {
                case Normal:
                    ++deallocationsNormal;
                    break;
                case Small:
                    ++deallocationsSmall;
                    break;
                case Tiny:
                    ++deallocationsTiny;
                    break;
                default:
                    throw new Error();
            }
        }
        destroyChunk = !chunk.parent.free(chunk, handle, nioBuffer);
    }
    if(destroyChunk)
    {
        destroyChunk(chunk);
    }
}
```

在这个里面，不太好理解的是 `if(!finalizer)` 中的代码。这个 if 判断是用于在特殊情况下的逻辑处理。

> 线程缓存中的数据可能会一直存在，直到该线程销毁时在 finalizer 方法中去释放其中包含的内存空间。从 finalizer 方法中释放内存空间时，是调用 `PoolThreadCache.MemoryRegionCache#freeEntry`。对于 Tomcat 而言，当其关闭一个应用后就会卸载其对应的 WebAppClassLoader，而在执行上述方法的时候，对于 Normal，Small，Tiny 这三个枚举，可能需要 WebAppClassLoader 来进行载入，而此时该 Loader 已经卸载，无法执行载入工作，就会抛出异常。因此这里加入了 if 逻辑来判断。

#### **归还空间至内存块**

在统计完成后，执行真正的内存释放方法 `PoolChunkList#free`，代码如下：

```java
boolean free(PoolChunk < T > chunk, long handle, ByteBuffer nioBuffer)
{
    chunk.free(handle, nioBuffer);
    if(chunk.usage() < minUsage)
    {
        remove(chunk);
        return move0(chunk);
    }
    return true;
}
```

通过 PoolChunk.free 将空间释放归还给内存块。内存空间归还给内存块后，其使用率就下降了，如果低于当前 PoolChunkList 的下限，则从当前链表中移除，也就是 remove 方法。并且尝试放入前驱链表中，也就是 remove0 方法。当然，如果当前链表是 p000 的话，则没有前驱链表，此时返回 false，表明该 Chunk 已经不再使用。

而如果不再使用的话，则会调用 destroyChunk 方法将内存块的内存释放，归还给 JVM。如果是堆内存的话，则直接依赖于 GC 即可；如果是直接内存的话，则依赖 SUN 内部的 API。比如 Unsafe 这个类。

PoolChunk.free 在《Netty 内存池设计之内存申请代码实现》的空间归还章节说过，不过当时没有分析微小内存或小内存的归还场景，这里补充一下。首先还是先看完整的代码，如下：

```java
void free(long handle, ByteBuffer nioBuffer)
{
    int memoryMapIdx = memoryMapIdx(handle);
    int bitmapIdx = bitmapIdx(handle);
    if(bitmapIdx != 0)
    {
        PoolSubpage < T > subpage = subpages[subpageIdx(memoryMapIdx)];
        assert subpage != null && subpage.doNotDestroy;
        PoolSubpage < T > head = arena.findSubpagePoolHead(subpage.elemSize);
        synchronized(head)
        {
            if(subpage.free(head, bitmapIdx & 0x3FFFFFFF))
            {
                return;
            }
        }
    }
    freeBytes += runLength(memoryMapIdx);
    setValue(memoryMapIdx, depth(memoryMapIdx));
    updateParentsFree(memoryMapIdx);
    if(nioBuffer != null && cachedNioBuffers != null && cachedNioBuffers.size() < PooledByteBufAllocator.DEFAULT_MAX_CACHED_BYTEBUFFERS_PER_CHUNK)
    {
        cachedNioBuffers.offer(nioBuffer);
    }
}
```

如果是微小内存或小内存归还的场景，bitmapIdx 必然不等于 0。通过 handle 可以得到内存空间所在的内存页的下标，也就可以得到该内存页对应的子页在子页数组中下标，进而获取该子页对象。

任何涉及到子页上的操作，分配和归还，都首先需要对该子页的等分大小对应的在 PoolArena 中的链表的头结点加锁，以避免并发造成数据错误。

子页上的内存释放是通过方法 `PoolSubpage#free` 完成，其内部实现很简单就不罗列代码了，其主要功能有：

- 将内存空间对应的位图下标值更新为 0，表示其空间可用。
- 将 nextAvail 设置为内存空间对应的位图下标；下一次申请时直接使用 nextAvail，加快申请速度。
- 内存空间归还后，如果当前子页不在链表中，则添加到链表中；如果当前子页所有空间均没被使用，则从链表中删除并且归还空间给内存块；如果子页所在链表只有该子页一个元素时，则不删除。

PoolSubPage.free 方法的返回值表示该子页是否仍然在子页链表中。如果返回 false，说明该子页已经从内存池的子页链表中移除，此时该子页持有的空间对应的内存页可以归还给内存块，也就是执行后续代码的逻辑。否则的话，该子页还在使用中，就不需要其他的处理，直接方法返回。内存释放的整体流程也就到此为止。

### 线程缓存

在上文分析内存申请和释放的过程中，我们都看到了线程缓存 PoolThreadCache 的身影。当线程缓存开启的时候，也就是 tinyCacheSize、smallCacheSize、normalCacheSize 不为 0 的情况下，内存的申请与释放都会优先从线程缓存中进行处理。

#### **通过线程缓存进行内存分配**

在通过内存池申请内存，也就是方法中：

```
PoolArena#allocate(PoolThreadCache,PooledByteBuf<T>, int)
```

在尝试分配微小、小、普通大小三种类型的内存时，都会优先尝试从线程缓存中申请。对应的，PoolThreadCache 也分别提供了三种方法来支持不同的大小：allocateTiny、allocateSmall、allocateNormal。这三个方法的代码如下：

```java
boolean allocateTiny(PoolArena <? > area, PooledByteBuf <? > buf, int reqCapacity, int normCapacity)
{
    return allocate(cacheForTiny(area, normCapacity), buf, reqCapacity);
}
boolean allocateSmall(PoolArena <? > area, PooledByteBuf <? > buf, int reqCapacity, int normCapacity)
{
    return allocate(cacheForSmall(area, normCapacity), buf, reqCapacity);
}
boolean allocateNormal(PoolArena <? > area, PooledByteBuf <? > buf, int reqCapacity, int normCapacity)
{
    return allocate(cacheForNormal(area, normCapacity), buf, reqCapacity);
}
private boolean allocate(MemoryRegionCache <? > cache, PooledByteBuf buf, int reqCapacity)
{
    if(cache == null)
    {
        return false;
    }
    boolean allocated = cache.allocate(buf, reqCapacity);
    if(++allocations >= freeSweepAllocationThreshold)
    {
        allocations = 0;
        trim();
    }
    return allocated;
}
```

从代码上一目了然，三种不同大小的 allocate 方法都是为了寻找对应大小的内存空间链表对象，也即 MemoryRegionCache。寻找的方式很简单，只需要通过需要需要申请的规范大小类型和内存类型就可以确定对应的 MemoryRegionCache[] 对象，再通过申请的规范大小就可以计算出所需要的 MemoryRegionCache 在数组中的下标，进而得到对象。

在 MemoryRegionCache 上的分配就相对简单了，如下：

```java
public final boolean allocate(PooledByteBuf < T > buf, int reqCapacity)
{
    Entry < T > entry = queue.poll();
    if(entry == null)
    {
        return false;
    }
    initBuf(entry.chunk, entry.nioBuffer, entry.handle, buf, reqCapacity);
    entry.recycle();
    ++allocations;
    return true;
}
```

MemoryRegionCache 本身实际上是 Entry 的链表，如果链表中有可用的元素，也就是 Entry，则使用 Entry 中存储的数据——PoolChunk 对象、内存空间坐标 handle、可能存在的 ByteBuffer 实例来初始化 ByteBuf。

在 Entry 中的数据被用于初始化 ByteBuf 后，该 Entry 就不需要使用了，其本身只是一个数据的容器。因此通过 recycle 方法直接回收到对象缓存池，供后续复用。

当在线程缓存上成功分配的次数超过了阀值后，也即是判断 `if(++allocations >= freeSweepAllocationThreshold)` 生效时，则对线程缓存执行整理操作。

#### **线程缓存的整理操作**

线程缓存的整体操作依靠方法 trim 进行，具体如下：

```java
void trim()
{
    trim(tinySubPageDirectCaches);
    trim(smallSubPageDirectCaches);
    trim(normalDirectCaches);
    trim(tinySubPageHeapCaches);
    trim(smallSubPageHeapCaches);
    trim(normalHeapCaches);
}
private static void trim(MemoryRegionCache <? > [] caches)
{
    if(caches == null)
    {
        return;
    }
    for(MemoryRegionCache <? > c: caches)
    {
        trim(c);
    }
}
private static void trim(MemoryRegionCache <? > cache)
{
    if(cache == null)
    {
        return;
    }
    cache.trim();
}
public final void MemoryRegionCache#trim()
{
    int free = size - allocations;
    allocations = 0;
    if(free > 0)
    {
        free(free, false);
    }
}
```

线程缓存整理的设计思路就是遍历所有的内存空间 MemoryRegionCache 对象，执行其 trim 方法。线程缓存和内存空间的分配次数是分别统计的。这是为了明确哪一些大小的内存空间较少的从线程缓存中被分配，就可以将这部分内存空间归还给内存池。因为线程缓存中的空间都是被本线程独占，通过这种方式进行整理，就可以将没有被分配过的部分内存空间空间归还给内存池，从而提高内存池的整体利用率。

trim 的实现思路也简单，将该 MemoryRegionCache 的大小 size 减去分配次数 allocations 得到的 free 就意味着有 free 个 Entry 从未被使用过，从队列中获取 free 个 Entry，执行`MemoryRegionCache#freeEntry`，代码如下：

```java
private void freeEntry(Entry entry, boolean finalizer)
{
    PoolChunk chunk = entry.chunk;
    long handle = entry.handle;
    ByteBuffer nioBuffer = entry.nioBuffer;
    if(!finalizer)
    {
        entry.recycle();
    }
    chunk.arena.freeChunk(chunk, handle, sizeClass, nioBuffer, finalizer);
}
```

将 Entry 中存储的内存空间坐标 handle，可能缓存的 ByteBuffer 实例，一并归还给内存空间所属的 PoolChunk 对象。

如果 freeEntry 不是由 PoolThreadCache 的 finalizer 方法发起，则 Entry 对象回收供后续复用，也就是执行 recycle 方法。这个方法还有一个作用，recycle 方法会清除 Entry 对象对 PoolChunk 的强引用，使得 PoolChunk 对象可以被 GC 掉。

而如果是由 finalizer 方法发起，意味着 PoolThreadCache 对象所在的线程消亡，此时就不需要再回收 Entry 对象了，因为所有的 MemoryRegionCache 也都需要被释放掉。任由 Entry 被 GC 回收掉即可。

#### **通过线程缓存进行内存释放**

当 ByteBuf 执行 release 方法释放自身内存空间时，也是优先尝试将内存空间缓存给当前的线程缓存。在 PoolArena.free 方法中，如果释放的 PoolChunk 不是大内存的情况下，则优先尝试将内存空间添加到线程缓存中，如果线程缓存已经满了或者线程缓存功能本身没有打开，则调用 freeChunk 方法将内存空间归还到内存块。

添加到线程缓存使用的方法是 `PoolThreadCache#add`，代码如下：

```java
boolean add(PoolArena <? > area, PoolChunk chunk, ByteBuffer nioBuffer, long handle, int normCapacity, SizeClass sizeClass)
{
    MemoryRegionCache <? > cache = cache(area, normCapacity, sizeClass);
    if(cache == null)
    {
        return false;
    }
    return cache.add(chunk, nioBuffer, handle);
}
```

首先还是通过 cache 方法取得规范大小对应的 MemoryRegionCache，执行 add 方法将三个要素包装为一个 Entry。MemoryRegionCache 中的队列是固定大小的，这个大小在初始化的时候传入。如果当前队列已经满了，则无法添加。

### 总结与思考

到这为止，我们从最重要的内存块上的内存分配算法到整体的内存池的分配算法，以及小内存情况下优化子页算法都已一一梳理过。每一章算法分析之后都跟随着对应的 Netty 当中的代码实现的讲解分析。经过四个章节的学习，相信读者已经对 Netty 对内存的管理有了深入的理解和掌握。

在 Netty4 最初的版本的时候，内存池并不是默认作为开放的，也是为了避免出现内存泄漏的问题。因为内存泄漏不仅仅可能来自内存池实现的缺陷，也可能是来自应用程序错误的使用。经过几个版本的稳定后，现在内存池已经默认被使用。但是内存池的使用也对应用程序带来了一定的约束。比如需要在正确的时机对申请的内存对象进行释放，否则就会造成内存泄漏。而内存泄漏一旦发生，排查起来困难也是比较大。Netty 在这一方面也为开发者提供了对应的支持，下一个章节，我们就来学习 Netty 中如何实现对内存或者说资源的追踪检测，而这一检测功能又是如何帮助开发者发现可能的潜在的内存泄漏功能。