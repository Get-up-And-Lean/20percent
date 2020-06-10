# 25/31Netty 内存池设计之内存申请代码实现

### 引言

前文我们通过引导，设问的方式一步步思考和总结出了一套管理连续内存的申请与分配的方法。同时也分析了相关算法思想下，Netty 中算法的变种表现形式。今天，我们接着上文，来为大家梳理下 Netty 中针对这段算法的代码实现方式。

### 基础属性

用于实现内存分配算法的类是 io.netty.buffer.PoolChunk。这是一个被 final 修饰的具备泛型的类。使用泛型的主要原因在于，用于分配的内存有两种形式：堆内内存也就是 byte[] 和直接内存 DirectByteBuffer。

这里有一个基础概念先明确下，进行内存空间的分配的基本是内存块，一个内存块在 Netty 中默认是 64M 大小。内存池则是有多个内存块构成。PoolChunk 代表的概念是内存块。接着我们来看下 PoolChunk 的重要属性，如下：

```java
final class PoolChunk < T > implements PoolChunkMetric
{
    final T memory;//连续内存空间的底层载体，可能是 byte[]，也可能是 DirectByteBuffer
    final boolean unpooled; //该内存块是否属于内存池的一部分
    final int offset; //当 memory 是 DirectByteBuffer 时，offer 代表着其内存空间地址的起始位置
    private final byte[] memoryMap;//存储管理内存区域的二叉树的节点的值；完全二叉树可以使用数组表现。
    private final byte[] depthMap;//存储管理内存区域的二叉树的节点的初始值
    private final int pageSize;//内存块是由连续的内存页构成，pageSize 是内存页的大小
    private final int pageShifts;//pageSize = 1 << pageShifts
    private final int maxOrder; //二叉树的最大深度
    private final int chunkSize;//内存块的大小
    private final int log2ChunkSize;//log2ChunkSize=log2(chunkSize)
    private final byte unusable;//unusable=maxOrder+1，当一个节点的值等于 unusable 时，意味着该节点不可分配了
    private int freeBytes;//当前还剩余可分配大小
}
```

上述的属性中，我们提到了一个之前未曾提到的概念，内存页。实际上，这是一个虚拟的概念。一个内存块由连续的 2maxOrder 个内存页构成。也就是说给定 pageSize 和 maxOrder，chunkSize 就被确定了。

二叉树管理的内存块可以形象的表达为：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191226230805.png)

页的大小是 pageSize，页的个数是 2maxOrder，而 chunkSize 等于 pageSize*2maxOrder。为了方便于计算，pageSize 必须是 2 的次方幂。默认情况下，取值为 8k，maxOrder 默认取值为 11，也就是默认情况下，一个 Chunk 中有 2048 个内存页。

树的节点的深度取值从根节点到叶子节点依次为 0 到 maxOrder。显然，根节点管理的内存大小是 chunkSize，叶子节点管理的内存大小是 chunkSize/2maxOrder。据此，可以得到节点管理内存大小的计算公式为 chunkSize/2h，这里 h 是节点的深度。位移计算比除法运算更快，且 chunkSize 也是 2 的次方幂，因此可以将这里的除法计算转化为位移运算即为 `chunkSize >> h`。而 chunkSize 可以看成是 `1 << log2ChunkSize`，因此节点管理的内存大小可以计算为 `1 << (log2ChunkSize-h)`，这个公式也就是 Netty 中计算节点管理内存大小的公式。

到这里，我们已经梳理完 pageSize、chunkSize、log2ChunkSize、maxOrder 几个属性之间的数学关系了。Netty 在这内存申请和释放的相关方法中，大量的使用了位运算，因此这几个属性之间的数学关系和运算转换是一个必须要理解的基础前提。

### 构造方法

看过了属性的解释，接着我们来看下 PoolChunk 的构造方法，如下：

```java
PoolChunk(PoolArena < T > arena, T memory, int pageSize, int maxOrder, int pageShifts, int chunkSize, int offset)
{
     unpooled = false;
     this.arena = arena;
     this.memory = memory;
     this.pageSize = pageSize;
     this.pageShifts = pageShifts;
     this.maxOrder = maxOrder;
     this.chunkSize = chunkSize;
     this.offset = offset;
     unusable = (byte)(maxOrder + 1);
     log2ChunkSize = log2(chunkSize);
     subpageOverflowMask = ~(pageSize - 1);//与小内存管理相关，先忽略
     freeBytes = chunkSize;
     maxSubpageAllocs = 1 << maxOrder;//与小内存管理相关，先忽略
     p.
     memoryMap = new byte[maxSubpageAllocs << 1];
     depthMap = new byte[memoryMap.length];
     int memoryMapIndex = 1;
     for(int d = 0; d <= maxOrder; ++d)
     {
         int depth = 1 << d;
         for(int p = 0; p < depth; ++p)
         {
             memoryMap[memoryMapIndex] = (byte) d;
             depthMap[memoryMapIndex] = (byte) d;
             memoryMapIndex++;
         }
     }
     subpages = newSubpageArray(maxSubpageAllocs);//与小内存管理相关，先忽略
     cachedNioBuffers = new ArrayDeque < ByteBuffer > (8);
}
```

大部分属性的含义都在《基础属性》章节说明过，这边我们来说明下 memoryMap 和 depthMap 这两个数组。memoryMap 用于存储二叉树每个节点的当前值，depthMap 用于存储二叉树每个节点的初始值。两个数组的长度是相同的，都是 `1<<(maxOrder+1)`，因为内存页的数量，也就是二叉树叶子节点的数量是 `1<<maxOrder`。这意味着二叉树总节点个数为 `(1<<maxOrder)*2-1`。用数组可以表达这样的完全二叉树，其长度应该为 `1<<(maxOrder+1)`，其中下标 0 的元素不使用。

代码：

```
int memoryMapIndex = 1;
for(int d = 0; d <= maxOrder; ++d)
{
int depth = 1 << d;
for(int p = 0; p < depth; ++p)
{
   memoryMap[memoryMapIndex] = (byte) d;
   depthMap[memoryMapIndex] = (byte) d;
   memoryMapIndex++;
}
}
```

通过两个 for 循环来为二叉树每一个深度的节点来赋值，每一个节点的初始值都等于该节点所处的深度。

### 申请空间

在 PoolChunk 上申请空间的代码如下：

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
        handle = allocateSubpage(normCapacity);//当申请的大小小于 pageSize 时走小内存申请模式，在这里暂时忽略
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

reqCapacity 是实际申请的大小，normCapacity 是对大小进行规范化后的值，所谓规范化就是寻找到比 reqCapacity 大的最小的 2 的次方幂的值，因为内存块中的数据都是按照 2 的次方幂的大小增长的。如果规范化大小比 pageSize 要小，则走小内存申请模式。在这里，我们先不分析小内存的申请模式，留待后续的专门章节分析。

如果规范化大小大于等于 pageSize，则使用 allocateRun 方法进行内存分配。其代码如下：

```java
private long allocateRun(int normCapacity)
{
    int d = maxOrder - (log2(normCapacity) - pageShifts);
    int id = allocateNode(d);
    if(id < 0)
    {
        return id;
    }
    freeBytes -= runLength(id);
    return id;
}
```

#### **计算分配节点所在深度**

根据前文中对算法的介绍，在内存块中进行空间申请，首先应该定位需要申请的大小对应的节点所在的深度。显然节点深度，内存页大小和规范大小存在如下关系：

```
pageSize<<(maxOrder-d)=normCapacity
```

考虑到 `pageSize=1<<pageShifts`，因此上述关系可以转换为：

```
1<<pageShifts+maxOrder-d=normCapacity
```

对等式两边进行 log2 操作，即可得到：

```
pageShifts+maxOrder-d = log2(normCapacity)

d=maxOrder - (log2(normCapacity) - pageShifts)
```

#### **从根节点出发至目标深度寻找可分配节点**

在明确了规范大小所在的节点的深度后，就是从根节点出发寻找分配节点，也就是方法 allocateNode，其代码如下：

```java
private int allocateNode(int d)
{
    int id = 1;
    int initial = -(1 << d);
    byte val = value(id);
    if(val > d)
    { // unusable
        return -1;
    }
    while(val < d || (id & initial) == 0)
    {
        id <<= 1;
        val = value(id);
        if(val > d)
        {
            id ^= 1;
            val = value(id);
        }
    }
    byte value = value(id);
    assert value == d && (id & initial) == 1 << d: String.format("val = %d, id & initial = %d, d = %d", value, id & initial, d);
    setValue(id, unusable); // mark as unusable
    updateParentsAlloc(id);
    return id;
}
```

首先来看 initial 变量，其取值为 `-(1 << d)`，假定 d 为 11，则 initial 的二进制表示为 0x11111111111111111111100000000000。对于在深度 d 的节点，节点在 memoryMap 中的下标的取值从 `1<<d` 到 `(1<<(d+1)-1)`，这意味对于深度 d 的节点，节点下标 & initial 的值等于节点下标本身。而对于深度小于 d 的节点，节点下标 & initial 的值为 0。在排除目标节点深度不会超过 d 的情况下，这个特性可以用于判断目标节点是否处于深度 d。

value 方法就是从 memoryMap 中取值。前文提到过，memoryMap 存储的二叉树中每个节点的当前值。这个值的含义是该节点能够分配的连续空间所在的深度。也就是如果 `value>d` 就意味着该节点无法分配所需空间。

用于寻找对应节点的代码块在：

```java
while(val < d || (id & initial) == 0)
    {
        id <<= 1;
        val = value(id);
        if(val > d)
        {
            id ^= 1;
            val = value(id);
        }
    }
```

首先来看循环条件 `val < d || (id & initial) == 0`。从根节点的 `value<=d` 可以确认当前内存块或者说二叉树存在分配的节点。从根节点出发向下寻找，直到深度 d。在没有达到深度 d 之前，`id&initial` 的值为 0，考虑深度 d 之前，`val<=d` 必然成立，而 val 总是需要计算的，因此为了减少计算，循环条件将二者整合 `val < d || (id & initial) == 0`。`val<d` 必然是中间节点，但是 `val==d` 却不一定是中间节点，此时就需要 `(id & initial) == 0` 来协助是否中间节点。

循环结束后，目标节点就被寻找到了，其下标就是 id。接着将该节点的值标记为不可使用，也就是最大深度 +1，即 unusable。

按照算法，此时需要不断更新父节点的值直到根节点为止，调用方法 updateParentsAlloc，如下：

```java
private void updateParentsAlloc(int id)
{
    while(id > 1)
    {
        int parentId = id >>> 1;
        byte val1 = value(id);
        byte val2 = value(id ^ 1);
        byte val = val1 < val2 ? val1 : val2;
        setValue(parentId, val);
        id = parentId;
    }
}
```

结合前文对算法的分析，这里的代码就很清楚了。获取父节点的 id，并且对比自己和兄弟节点的值，选择较小的值设置到父节点上。重复这个过程直到根节点。

#### **使用节点下标初始化 ByteBuf**

从二叉树获得可以分配空间的节点下标。接着就是使用这个节点下标转化为其对应管辖的空间来初始化 ByteBuf。

二叉树使用数组来存放，节点的下标是一个 int 整型，但是在 allocate 方法中是使用一个 long 变量来存储这个数据。这是因为一个 long 变量被拆分为高低 2 个 32 位整型来看待，低位的 32 位整型用来存储节点的下标，而高位则用于小内存分配，这个在后文分析小内存分配的时候再讲述。

有了节点下标值 handle 之后，使用方法 initBuf 初始化一个 ByteBuf，下面是其代码：

```java
   void initBuf(PooledByteBuf < T > buf, ByteBuffer nioBuffer, long handle, int reqCapacity)
  {
      int memoryMapIdx = memoryMapIdx(handle);
      int bitmapIdx = bitmapIdx(handle);
      if(bitmapIdx == 0)
      {
          byte val = value(memoryMapIdx);
          assert val == unusable: String.valueOf(val);
          buf.init(this, nioBuffer, handle, runOffset(memoryMapIdx) + offset, reqCapacity, runLength(memoryMapIdx), arena.parent.threadCache());
      }
      else
      {
          initBufWithSubpage(buf, nioBuffer, handle, bitmapIdx, reqCapacity);
      }
  }
```

memoryMapIdx 属性就是 handler 的低 32 位，这边还不涉及小内存分配，因此 bitmapIdx 必然为 0。

对于二叉树节点 memoryMapIdx 而言，该节点映射的内存区域相对于整个内存块的偏移量由方法 runOffset 计算得到，如下：

```java
private int runOffset(int id)
{
    int shift = id ^ 1 << depth(id);
    return shift * runLength(id);
}
```

`1 << depth(id)` 得到的是二叉树在深度 d 的最左侧第一个节点在数组中的下标。该值与节点下标 id 进行异或操作，得到的结果就是在深度 d 上，在节点 memoryMapIdx 之前还有多少个同深度的节点。

以树深度 5 为例，在深度 4 上的 memoryMapIdx 为 23 为例，内存块为 32k 大小，情况如下：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191229170722.png)

在 memoryMapIdx 节点之前，同深度节点个数为 `23^(1<<4)=7`。

当计算出 memoryMapIdx 节点之前同深度节点的个数时，其管理的内存空间相对于内存块起始位置的偏移量就很好计算了，就是个数 * 该深度的单位空间大小，也就是 `shift * runLength(id)`。

将 PoolChunk（内存块引用），handle（节点信息），`runOffset(memoryMapIdx) + offset`（节点管辖内存相对于内存块起始区域的偏移量），runLength(memoryMapIdx)（节点管辖的内存区域大小）作为 ByteBuf 的初始参数，即可完成初始化。

这一步完成，空间申请就结束了。ByteBuf 对象中包含了内存块的引用，自身可以使用的内存大小，自身使用的内存相对于内存块的起始位置的偏移量，有了这三者就可以实现对应内存区域的读写操作了。

### 空间归还

空间归还和空间申请是一个反向操作的步骤，在 PoolChunk 中归还空间使用的方法是 free，如下：

```java
void free(long handle, ByteBuffer nioBuffer)
{
    int memoryMapIdx = memoryMapIdx(handle);
    int bitmapIdx = bitmapIdx(handle);
    if(bitmapIdx != 0)
    {
        //省略代码，暂不涉及小内存归还
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

首先是通过 handle 计算出节点在二叉树的下标 memoryMapIdx。将该节点的值恢复为初始值，也就是 depth(memoryMapIdx)。由于自身的值的变化，因此需要将父节点的值也更新，并且循环更新直到根节点，使用方法 updateParentsFree。该方法与申请空间时更新父节点的方法 updateParentsAlloc 十分相似，但是需要考虑额外的一点就是由于是归还空间，就会存在一种申请空间时不存在的情况：两个兄弟节点的值都为初始值时，父节点的值也为初始值而不是两者之中的较小值。其代码如下：

```java
private void updateParentsFree(int id)
{
    int logChild = depth(id) + 1;
    while(id > 1)
    {
        int parentId = id >>> 1;
        byte val1 = value(id);
        byte val2 = value(id ^ 1);
        logChild -= 1;
        if(val1 == logChild && val2 == logChild)
        {
            setValue(parentId, (byte)(logChild - 1));
        }
        else
        {
            byte val = val1 < val2 ? val1 : val2;
            setValue(parentId, val);
        }
        id = parentId;
    }
}
```

如文所述，额外就是多了 `if(val1 == logChild && val2 == logChild)` 的这种情况。

### 总结与思考

本文对照着内存块申请与释放的算法，讲解了 Netty 中承担这一职责的组件 PoolChunk。通过代码分析的方式，进一步对 Netty 的内存分配算法做了讲解。

在代码分析的过程中，我们也忽略了一些流程，这些流程涉及到的小内存分配和线程缓存等，会在后面的章节中专门进行说明。下一篇文章，我们来学习下 Netty 整体内存池的分配。