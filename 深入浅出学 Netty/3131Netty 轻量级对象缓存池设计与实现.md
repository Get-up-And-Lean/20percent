# 31/31Netty 轻量级对象缓存池设计与实现

## 引言

在 Java 的世界中，对象的生命周期基本都是创建，使用，GC 回收。但是有些对象由于其创建的频度高，创建代价昂贵，且具备重复利用的可能性，因此往往希望将这类对象缓存起来，供后续的重复利用。这就出现了一种专门的设施结构：对象缓存池，专注于提供对象的复用，减少对象创建的开销。

在前文的很多分析中，特别是一些高频率使用的对象，比如`PooledByteBuf`等，我们都能见到一个类的身影，就是`Recycle`。这个类是Netty 设计的轻量级对象缓存池实现。因为像`PooledByteBuf`这种对象的创建是非常频繁的，每一次数据的读取，业务流程的处理都需要承载在这个对象上。但是一旦业务处理完毕就直接被丢弃了。如果反复的创建，对系统的GC压力来说也是很大的。使用对象缓存池，可以减少频繁重复创建的开销，提升一定的性能。

## 思路推导

像前文一样，我们先不分析代码，从设计的角度来推导对象池可能的实现方式。

对象池简单说就是将对象实例缓存起来供后续分配使用来避免瞬时大量小对象反复生成和消亡造成分配和GC压力。在设计时可以简单的做如下推导：

**首先考虑只有单线程的情况**，此时简单的构建一个 Queue 容器，对象回收时放入 Queue ，分配时从 Queue 取出，如果 Queue 中已经没有对象实例了，则创建一个新的实例。这种方式非常容易就构建了一个单线程的对象池实现。额外需要考虑的细节就是为 Queue 设置一个大小的上限，避免池化的对象实例过多而导致消耗太多的内存。

**接着考虑多线程的情况**，分配时与单线程情况相同，从当前线程的 Queue 容器中取出一个对象或者创建一个对象实例分配即可。但回收时却遇到了麻烦，对象实例被回收时的线程和分配时的线程可能一致，那处理方式与单线程相同。如果不一致，此时存在不同的策略选择，大致来说有三种：

\+ 策略一: 将对象回收至分配线程的 Queue 容器中。 + 策略二: 将对象回收至本线程的 Queue 容器，当成本线程的对象使用 + 策略三: 将对象暂存于本线程，在后续合适时机归还至分配线程。

每一种策略均各自的优缺点。

对于**策略一**，由于存在多个线程并发的归还对象实例到借出线程，因此对于借出线程而言，其存储对象实例的 Queue 容器必须是MPSC类型，多个归还线程，一个借出线程。但是采用 MPSC 队列的形式，伴随着对象的借出和归还，队列内部的节点也是在不断的生成和消亡，这样就变相的降低了对象缓存池的效果。

对于**策略二**，如果线程之间呈现明显的借出和回收分工，则会导致对象缓存池失去作用。因为借出线程总是无法回收到对象，因此只能不断的新建对象实例，；而回收线程因为很少分配对象，导致回收的对象超出上限被抛弃，从而无法有效的复用。

对于**策略三**，由于对象暂存于本线程，可以避开在回收时的并发竞争。并且因为在后续仍然会归还到借出线程，也避免了策略二可能导致的缓存失效情况。在 Netty 当中，采用的就是策略三模式。

## 算法设计

选定了实现思路之后我们就可以开始做算法上的设计。根据上面选定的策略三，首先需要构建一个数据结构用于存储当前线程分配和回收的对象实例。为了避免在出列和入列上结构本身的开销，我们采用数组的方式来存储对象。内部通过一个指针来实现类似堆栈的压栈和弹栈功能。我们将这个结构定义为 **Stack** ，每一个`Stack`实例都单属于一个特定的线程，**Stack通过线程变量的方式存储，避免全局竞争**。每一个从`Stack`被申请的对象实例都属于这个特定的`Stack`实例，最终必然要回收到该`Stack`实例之中。

通过`Stack`结构，对于借出线程而言，就能支撑其对象的申请和回收。而要支撑在其他线程暂存借出对象，需要另外的数据结构。在应用运行的过程中，对于特定的一个线程而言，可能会有多个属于不同`Stack`实例的对象要被暂存。显然这些对象应该按照其从属的`Stack`实例区分开来，为此设计一个`Map<Stack,Queue>`的结构。使用`Stack`作为映射键来进行区分，为每一个`Stack`都生成一个 Queue 结构来存储暂存于本线程的从属于其他线程`Stack`的对象。当需要暂存其他线程的对象实例时，通过该对象关联的`Stack`，找到或者创建属于其的 Queue 结构，将对象实例放入这个 Queue 结构中。这里的 Queue 结构，我们定义为 **WeakOrderQueue**，这个名称是因为其只保证元素有序，但是并不保证及时的可见性，只保证一个最终可见。同样的，这个 Map 结构一样也是通过线程变量的方式来实现并发安全。

通过`Stack`结构和`Map`结构，我们已经实现了当前线程回收和分配对象，其他线程暂存回收对象的效果。那么还差一个功能就是在合适的时机将回收对象归还至其从属的分配线程。对于这个功能，我们可以在`Stack`的内部增加一个`WeakOrderQueue`列表，每当其他线程生成一个与`Stack`关联的`WeakOrderQueue`，就将实例添加到`Stack`的`WeakOrderQueue`列表中。当`Stack`内部持有的对象数组没有元素供分配时，则遍历`WeakOrderQueue`列表，将`WeakOrderQueue`中的数据转移到数组中。此时数组中就有了元素，就可以执行对象的分配了。

经过上面的分析，我们将整个数据结构描述为下图所示

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20200206212753.png)

有了上面这张图，我们可以将`Stack`描述如下

\+ 持有一个Element数组用于存储和分配回收对象。 + 持有WeakOrderQueue列表用于在element数组没有数据时从其他线程的WeakOrderQueue中转移数据到Element数组。

`WeakOrderQueue`是一个 SPSC 操作类型的结构，暂存线程放入对象，分配线程取出对象。在 Netty 的设计中，`WeakOrderQueue`的设计既不是循环数组也不是 SPSC 队列。而是通过一个固定大小的片段数组来存储对象，每一个片段数组通过指针进行连接。内部保留`Head`、`Tail`两个指针。`Head`指针供`Stack`使用，标识当前正在读取的片段数组，`Tail`指针供`WeakOrderQueue`使用，标识最后一个数据可以写入的片段数组。

WeakOrderQueue为了完成SPSC操作（回收线程放入对象，原分配线程取出对象），其内部设计并非循环数组也不是SPSC队列，而是采用数组队列的方式。简单来说就是通过固定大小的片段数组存储对象，每一个片段数组通过Next指针链接。内部保留Head，Tail两个指针。Head指针供Stack使用，标识当前正在读取片段数组，Tail指针供WeakOrderQueue使用，标识最后一个数据可以写入的片段数组。

## 代码分析

有了上面的算法设计之后，再来看代码的实现就容易的多。首先来看就是对象缓存池的整体入口，类`io.netty.util.Recycler`。该类提供了`get`方法用于从对象缓存池中获取对象实例。

### Recycler

作为对象缓存池的入口，其本身并不承载任何数据结构，只是提供了一些基本的属性配置，如下

```java
private final int maxCapacityPerThread; // Stack 结构内数组的最大长度，也就是一个线程内部最多可以缓存的对象实例个数
private final int maxSharedCapacityFactor; //Stack 在其他所有暂存线程中缓存的对象实例总数与 maxCapacityPerThread 的比的反值。
private final int ratioMask; // 在暂存线程暂存对象时，对于从未回收过的对象，Netty 设计按照一定比例来抛弃未回收过的对象，避免暂存线程中的实例数量增长太快。1-(1/ratioMask)即为抛弃的比例。
private final int maxDelayedQueuesPerThread; //一个线程最多为多少Stack暂存对象实例
```

这个类主要是提供了一个`get`方法用于获取对象实例，代码如下

```java
public final T get()
{
    if(maxCapacityPerThread == 0)
    {
        return newObject((Handle < T > ) NOOP_HANDLE);
    }
    Stack < T > stack = threadLocal.get();//代码①
    DefaultHandle < T > handle = stack.pop(); //代码②
    if(handle == null)
    {
        handle = stack.newHandle();//代码③
        handle.value = newObject(handle);
    }
    return(T) handle.value;//代码④
}
```

首先是**代码①**。通过线程变量来获取当前线程对应的`Stack`实例，`pop`方法通过弹栈给出一个被缓存的对象实例。如果当前对象池内已经没有被缓存的实例时，则创建新的对象实例并且返回给调用者，也就是**代码③**和**代码④**。

这里首先来关注下接口`Recycler.Handle`，代码如下

```java
public interface Handle < T >
{
    void recycle(T object);
}
```

从**代码③**方法`Recycler.Stack#newHandle`和`Handle`的定义就可以看出，`Handle`和被缓存的对象实例是一个绑定关系，用于为缓存的对象实例提供在对象换存池中的元数据信息存储这样的作用。下面我们来详细分析下。

### DefaultHandle

`Handle`接口在 Netty 中有唯一实现，就是`Recycler.DefaultHandle`，首先来看下它的属性，如下

```java
private int lastRecycledId; //用于判断同一对象是否错误的回收
private int recycleId; // 用于判断同一对象是否错误的回收
boolean hasBeenRecycled; //该对象是否被回收过
private Stack <? > stack; //创建该对象的Stack，这也是该对象最终要被回收到的Stack。
private Object value; //可被缓存的对象实例
```

来看看接口方法`recycle`的实现，如下

```java
public void recycle(Object object)
{
    if(object != value)
    {
        throw new IllegalArgumentException("object does not belong to handle");
    }
    Stack <? > stack = this.stack;
    if(lastRecycledId != recycleId || stack == null)
    {
        throw new IllegalStateException("recycled already");
    }
    stack.push(this);
}
```

这个方法在缓存对象需要回收的时候被调用。由此可见，对`Recycler`的使用，在缓存对象内部，也是需要持有`handle`实例的引用的。这也是方法`Recycler#newObject`的内容。该方法是一个抽象方法，供具体的对象缓存池实现，开发者在方法的实现中，应该将给定的入参`Handle`实例注入到需要缓存的对象中，以确保当对象需要被回收时，可以手动的来调用其`recycle`方式实现缓存对象的回收。

这个方法除了`stack.push(this)`都是在执行正确性校验。关于`lastRecycledId`和`recycleId`如何实现正确性校验，后面单独开小节说明，这边先略过。

### Stack

无论是获取对象，还是归还对象，最终都是通过`Stack`来实现的。在分析功能之前，首先来看下`Stack`的一些重要属性，如下

```java
final WeakReference < Thread > threadRef;//该Stack关联的线程实例。
final AtomicInteger availableSharedCapacity;//该Stack当前剩余其他线程可缓存对象实例个数
final int maxDelayedQueues; // 一个线程最多可以同时为几个Stack暂存对象实例。
private final int maxCapacity; //该Stack缓存对象数组的最大容量
private final int ratioMask; // 作用等同于Recycler.ratioMask
private DefaultHandle <? > [] elements; //该Stack缓存对象数组
private int size; //数组中非空元素的个数
private int handleRecycleCount = -1; // Stack关联的线程处理的回收请求数量
private WeakOrderQueue cursor, prev; //指向当前处理的WeakOrderQueue和下一个将要处理的WeakOrderQueu
private volatile WeakOrderQueue head; //指向Stack持有的WeakOrderQueue链表的头结点，使用volatile修饰是方便进行并发修改。
```

#### 分配对象实例

我们首先来分析其对获取对象的实现。也就是方法`Recycler.Stack#pop`。该方法返回的是一个`DefaultHandle`对象，上个小节我们知道，每一个`DefaultHandle`对象内部都关联着，也即是返回了`DefaultHandle`对象，就相当于返回了一个被缓存的可复用对象实例了。来看具体方法，如下

```java
DefaultHandle < T > pop()
{
    int size = this.size;
    if(size == 0)
    {
        if(!scavenge())
        {
            return null;
        }
        size = this.size;
    }
    size--;
    DefaultHandle ret = elements[size];
    elements[size] = null;
    if(ret.lastRecycledId != ret.recycleId)
    {
        throw new IllegalStateException("recycled multiple times");
    }
    ret.recycleId = 0;
    ret.lastRecycledId = 0;
    this.size = size;
    return ret;
}
```

首先让我们先忽略`if`代码块的内容，整个方法看起来就很简单明然。`size`属性是`elements`数组内有效元素的个数，也可以理解为使用数组作为栈，`size-1`的值就是栈顶元素的下标。在`if`代码块后面的部分就是一个很明了的弹栈操作了。

而如果当前数组有效元素为0，也就是`size`为0时，执行方法`scavenge`，从其他线程的为该`Stack`暂存的`WeakOrderQueue`中获取缓存对象填充到数组中，如果获取成功，则执行弹栈操作，否则就直接返回null。

从`Recycler.get`方法可知，如果无法从`Stack`中分配对象，则首先通过`Recycler.Stack#newHandle`获取一个`DefaultHandle`实例，并且通过方法`Recycler#newObject`将该实例赋值给被新建出来可缓存的用户对象。

接着我们分析下方法`scavenge`的具体实现，这个方法的目的是为了从其他线程的`WeakOrderQueue`中转移`Handle`实例到`Stack`的`elements`数组中。其代码如下

```java
boolean scavenge()
{
    if(scavengeSome())
    {
        return true;
    }
    prev = null;
    cursor = head;
    return false;
}
```

首先是通过方法`scavengeSome`从当前指向`WeakOrderQueue`尝试转移`Handle`实例到`Stack`中。该方法中会重复逐个`WeakOrderQueue`尝试，直到转移成功或者到达`WeakOrderQueue`队列的末尾。如果到达队列末尾都没有成功找到`Handle`实例并且转移，则将`cursor`指针设置为队列的头节点，也就是`head`的值，好让下一次转移从队列头开始。

下面我们来深入下方法`scavengeSome`，代码如下

```java
boolean scavengeSome()
{
    WeakOrderQueue prev;
    WeakOrderQueue cursor = this.cursor;
    if(cursor == null)
    {
        prev = null;
        cursor = head;
        if(cursor == null)
        {
            return false;
        }
    }
    else
    {
        prev = this.prev;
    }
    boolean success = false;
    do {
        if(cursor.transfer(this))
        {
            success = true;
            break;
        }
        WeakOrderQueue next = cursor.next;
        if(cursor.owner.get() == null)
        {
            if(cursor.hasFinalData())
            {
                for(;;)
                {
                    if(cursor.transfer(this))
                    {
                        success = true;
                    }
                    else
                    {
                        break;
                    }
                }
            }
            if(prev != null)
            {
                prev.setNext(next);
            }
        }
        else
        {
            prev = cursor;
        }
        cursor = next;
    } while (cursor != null && !success);
    this.prev = prev;
    this.cursor = cursor;
    return success;
}
```

代码比较长，首先来看`do`循环体之前的代码。这部分代码主要的作用就是检查`cursor`指针是否正确的指向了可以尝试转移数据的`WeakOrderQueue`实例。`cursor` 指针的作用是为了指向需要进行数据转移的`WeakorderQueue`，而`prev`指针则是`WeakOrderQueue`链表中`cursor`指针的上一节点，其作用是在`cursor`指向的节点要剔除，用于寻找上一节点。

准备好`cursor`指针后就进入到循环体中。首先通过方法`WeakOrderQueue#transfer`来尝试转移数据到`Stack`数组中。这个方法的具体逻辑我们放在介绍`WeakOrderQueue`的章节中详细说明。

如果有数据被转移到`Stack`的数组中，则跳出`while`循环，方法返回上层调用结果。而如果`cursor`指向的`WeakOrderQueue`没有数据可以被转移，则准备尝试下一个`WeakOrderQueue`。重复这个过程直到数据转移成功或者`WeakOrderQueue`链表被遍历完。但是在尝试下一个`WeakOrderQueue`之前，通过`cursor.owner.get()`确认该`WeakOrderQueue`所关联的，也就是创建其的线程是否还存在。如果不存在的话，就意味着该`WeakOrderQueue`不会再有新的数据被添加进来。那么就应该将这个节点从整个`WeakOrderQueue`链表中摘除，这也是`prev`指针发挥作用的地方。而在摘除这个节点之前，如果其还存在数据，也就是`cursor.hasFinalData()`为真的话，就通过`for`循环反复的将剩余数据转移到`Stack`中，直到再也没有数据为止。

上面的流程就是整个分配对象的全部了。总结来说，就是`Stack`首先从持有的元素数组中以弹栈的形式获取对象，如果数组中已经没有有效元素了，则从`WeakOrderQueue`链表中转移对象实例到数组中。`Stack`会保留指向`WeakOrderQueue`链表中节点的指针`cursor`，避免每次都要从头找起，确保每一个`WeakOrderQueue`节点都能被遍历到。

#### 回收对象实例

对象实例的回收是通过接口方法`Recycler.DefaultHandle#recycle`完成，在上个章节中我们已经介绍过，其具体的实现是委托了给了方法`Recycler.Stack#push`来完成，该方法具体代码如下

```java
void push(DefaultHandle <? > item)
{
    Thread currentThread = Thread.currentThread();
    if(threadRef.get() == currentThread)
    {
        pushNow(item);
    }
    else
    {
        pushLater(item, currentThread);
    }
}
```

首先判断当前线程是否是创建该对象的`Stack`绑定的线程，如果是的话，就意味着是在本线程上操作，执行`pushNow`方法，否则则意味着是在其他线程上，是其他线程协助进行暂存，执行`pushLater`方法。首先来看比较简单的情况，就是`pushNow`方法，代码如下

```java
private void pushNow(DefaultHandle <? > item)
{
    if((item.recycleId | item.lastRecycledId) != 0)
    {
        throw new IllegalStateException("recycled already");
    }
    item.recycleId = item.lastRecycledId = OWN_THREAD_ID;//代码①
    int size = this.size;
    if(size >= maxCapacity || dropHandle(item))//代码②
    {
        return;
    }
    if(size == elements.length)
    {
        elements = Arrays.copyOf(elements, min(size << 1, maxCapacity));
    }
    elements[size] = item;
    this.size = size + 1;
}
```

代码整体很简单，不过有两个注意点。首先是**代码①**，设置回收的`DefaultHandle`的`recycleId`和`lastRecycledId` 。因为对象在被分配的时候这两个值会被清零。在回收的时候进行赋值，可以作为重复回收的一个判断依据。

接着是**代码②**，如果`Stack`内数组的有效元素个数达到了上限或者通过方法`dropHandle`判断该对象不进行回收，则方法直接结束，意味着丢弃该对象，不回收到缓存池中。`dropHandle`方法的逻辑是对于没有被回收过的对象实例，按照一定比例进行丢弃。Netty 的考虑是如果当前线程因为业务循环执行了一个`for`循环生成了大量的短暂使用的对象实例，那么此时不需要全部都回收到缓存池中，因为实际后续可能被复用的数量没有此次激增的数量多，所以按照一定比例进行丢弃。而如果是曾经被回收过的对象，则不需要再次考虑这种情况。这个比例就是由参数`Recycler.Stack#ratioMask`进行控制的。

如果回收对象的线程不是分配对象的线程，则就需要通过方法`Recycler.Stack#pushLater`来将对象暂存至当前线程。等待分配对象的线程在`Stack`内可分配对象数量不足时再被转移。相关代码在下一个章节分析`WeakOrderQueue`中专门讲解。

### WeakOrderQueue

#### 暂存其他分配线程创建的对象

`Stack`主要服务于分配，而`WeakOrderQueue`则主要服务于回收。上文说过，当回收对象的线程不是对象分配的线程时，是通过方法`Recycler.Stack#pushLater`来完成对象暂存的，下面让我们看看这个方法。

```java
private void pushLater(DefaultHandle <? > item, Thread thread)
{
    Map < Stack <? > , WeakOrderQueue > delayedRecycled = DELAYED_RECYCLED.get();
    WeakOrderQueue queue = delayedRecycled.get(this);
    if(queue == null)
    {
        if(delayedRecycled.size() >= maxDelayedQueues)
        {
            delayedRecycled.put(this, WeakOrderQueue.DUMMY);
            return;
        }
        if((queue = WeakOrderQueue.allocate(this, thread)) == null)
        {
            return;
        }
        delayedRecycled.put(this, queue);
    }
    else if(queue == WeakOrderQueue.DUMMY)
    {
        return;
    }
    queue.add(item);
}
```

方法的实现逻辑很清晰，首先从线程变量中取得属于本线程的`WeakHashMap<Stack,WeakOrderQueue>`实例，再以待回收对象关联的`Stack`对象作为 key ，获取其对应的`WeakOrderQueue`实例。如果没有则新建，如果当前线程已经为足够多的`Stack`都创建了`WeakOrderQueue`，则不再新建，转而放入一个特殊的`WeakOrderQueue`实例，代表着不为某个具体的`Stack`服务。

获取`WeakOrderQueue`实例后，通过其`add`方法将元素添加到其中。来看具体的代码细节，如下

```java
void add(DefaultHandle <? > handle)
{
    handle.lastRecycledId = id;
    Link tail = this.tail;
    int writeIndex;
    if((writeIndex = tail.get()) == LINK_CAPACITY)
    {
        if(!head.reserveSpace(LINK_CAPACITY))
        {
            return;
        }
        this.tail = tail = tail.next = new Link();
        writeIndex = tail.get();
    }
    tail.elements[writeIndex] = handle;
    handle.stack = null;
    tail.lazySet(writeIndex + 1);
}
```

`WeakOrderQueue`内部不是采用链表的形式，而是采用一个数组连着一个数组的形式存储对象。每一个数组都继承了`AtomicInteger`，其值用于做元素放入的写入下标。每个数组都有读取下标和写入下标，由于读取下标只有分配线程会读取和修改，不存在并发可见性问题，因此普通变量即可。而写入下标是由暂存线程修改，分配线程读取，需要保证可见性，应该使用`volatile`关键字修饰，这里为了编码简单，让数组直接继承`AtomicInteger`，使用其值本身作为写入下标。

这段代码不难，核心就是往数组中放入元素。如果数组已经满了，则尝试新建一个数组，连接到当前数组的下一个指针上。不过在新建数组之前，首先需要通过静态方法`WeakOrderQueue.Head#reserveSpace(atomic.AtomicInteger, int)`来确定非分配线程外暂存的对象实例是否达到了上限。如果是的话，则不再新建数组，放弃回收该对象。

这里有一点需要注意的地方在于，在增加数组的写入下标，也就是代码`tail.lazySet(writeIndex + 1)`之前，首先将`handle`的`stack`属性设置为了空。这是为了避免`Stack`出现无法回收的情况下。假定`Stack`关联的线程已经终止并且被GC，原本`Stack`应该一起被GC。但是由于`handle`中存在强引用，而`handle`又被其他线程强引用着，`Stack`实例无法被GC，导致该暂存线程中的`WeakHashMap`也无法清理本应该失效的 Key 。间接的导致了内存无法及时回收的问题。所以为了避免这一情况，这在设置数组下标之前，及时的将`handle`对`Stack`的引用设置为null。

此外，在这里设置数组下标使用了`lazySet`。这是`volatile`的高级用法。由于存在多个暂存线程服务于一个分配线程，因此对于特定的某一个暂存线程而言，不需要在回收数据后马上就让分配线程可见，因为分配线程也可以从其他的`WeakOrderQueue`上得到数据。因此这里不是使用`set`而是使用`lazySet`，这个方法保证写入的值最终必然会被其他线程可见，但是不保证及时性。降低及时性的好处在于提升了写入时候的性能，提升性能本质是依靠减少了内存屏障的开销（内存屏障是一个高级话题，后续我们使用的篇章进行分析，这里大家先明白这个事实即可）。

#### 转移对象实例至Stack

上文分析`Stack`的时候曾经提到过，当`Stack`数组中没有数据时，则会尝试从`WeakOrderQueue`链表中选择一个节点，并且尝试从其上转移缓存对象数据到`Stack`的数组中。这个转移依靠的就是方法`WeakOrderQueue#transfer`，来看下代码的具体内容

```java
boolean transfer(Stack <? > dst)
{
    /*****代码块①******/
    Link head = this.head.link;
    if(head == null)
    {
        return false;
    }
    if(head.readIndex == LINK_CAPACITY)
    {
        if(head.next == null)
        {
            return false;
        }
        this.head.link = head = head.next;
        this.head.reclaimSpace(LINK_CAPACITY);
    }
    final int srcStart = head.readIndex;
    int srcEnd = head.get();
    final int srcSize = srcEnd - srcStart;
    if(srcSize == 0)
    {
        return false;
    }
    /*****代码块①******/
    /*****代码块②******/
    final int dstSize = dst.size;
    final int expectedCapacity = dstSize + srcSize;
    if(expectedCapacity > dst.elements.length)
    {
        final int actualCapacity = dst.increaseCapacity(expectedCapacity);
        srcEnd = min(srcStart + actualCapacity - dstSize, srcEnd);
    }
    /*****代码块②******/
    if(srcStart != srcEnd)
    {
        final DefaultHandle[] srcElems = head.elements;
        final DefaultHandle[] dstElems = dst.elements;
        int newDstSize = dstSize;
        /*****代码块③******/
        for(int i = srcStart; i < srcEnd; i++)
        {
            DefaultHandle element = srcElems[i];
            if(element.recycleId == 0)
            {
                element.recycleId = element.lastRecycledId;
            }
            else if(element.recycleId != element.lastRecycledId)
            {
                throw new IllegalStateException("recycled already");
            }
            srcElems[i] = null;
            if(dst.dropHandle(element))
            {
                // Drop the object.
                continue;
            }
            element.stack = dst;
            dstElems[newDstSize++] = element;
        }
        /*****代码块③******/
        /*****代码块④******/    
        if(srcEnd == LINK_CAPACITY && head.next != null)
        {
            this.head.reclaimSpace(LINK_CAPACITY);
            this.head.link = head.next;
        }
        /*****代码块④******/
        head.readIndex = srcEnd;
        if(dst.size == newDstSize)
        {
            return false;
        }
        dst.size = newDstSize;
        return true;
    }
    else
    {
        return false;
    }
}
```

整个方法有些长，我们分块来看。首先是**代码块①**，在`WeakOrderQueue`的属性中，`head`属性指向当前可以使用的`Link`，也就是当前暂存对象的数组。通过代码块①，可以明确，该`WeakOrderQueue`中是否存在可以供转移的对象实例。需要注意的是，如果`head`指向的`Link`已经被消耗完毕了，则通过`WeakOrderQueue.Head#reclaimSpace`调用来恢复暂存线程剩余可暂存对象实例数量。前文我们说过，暂存线程中总共可以暂存的对象实例数是有上限的，在申请一个数组的时候需要从这个总数中扣除，当一个数组消耗完了再恢复扣除的数量到这个总数。将扣除和恢复的大小拉宽到一个数组的长度，也是为了避免频繁的对这个总数执行并发争夺，减少性能开销。

接着来看**代码块②**。这部分代码在代码块①确认有对象可以转移的前提下，确认了可以转移的对象个数，范围就是[srcStart,srcEnd)之间。`srcEnd`在数组可以转移的个数和`Stack`可以容纳的个数中选择较小值。

确定了有数据可以转移后，就到了转移环节，也就是**代码块③**。代码内容很简单，首先是通过`recycleId`和`lastRecycledId`的比对来验证是否出现重复回收。如果没问题，仍然需要方法`Stack#dropHandle`来判断当前对象是否需要抛弃。原因也是为了避免突然的对象创建高峰导致缓存不必要数量的对象实例。如果对象可以被回收，则转移到`Stack`的数组中。此时可以将`DefaultHandle`的`stack`属性从`null`更新为目标`stack`。

转移完成后来就来到了**代码块④**。这里的作用和代码块①中的类似，如果当前使用的`Link`已经消耗完毕，则恢复暂存线程剩余可暂存对象实例数量。

#### 并发可见性

WeakOrderQueue中很多地方并没有使用Volatile修饰，不影响正确性。比如`Link`的`next`就没有，可能会导致暂存线程新建了`Link`而分配线程无法发现）。最坏的情况无非就是分配线程对一些数据不可见造成没有数据可以转移出去，但是在足够的时间下， CPU 的 Buffer 是有限的，最终分配线程必然看到这些数据。

而数组的 volatile 写下标是为了保证在指针变化时，link数组中确实有了数据。避免由于重排序造成Stack线程看到指针却提取不到对象的情况发生。

#### 重复回收检测

为了避免在代码中错误的回收对象，框架设计了一种简单的重复回收检测规则。通过以下方式进行： 可回收对象（以下简称对象）设置 2 个数字属性`recycleId`，`lastRecycleId`。`lastRecycleId`用来存储最后一次回收对象的线程ID（或`Recycler`的 ID，该 ID 存在主要是为了避免每一个`Stack`都有单独 ID，起到节省作用）。`recycleId`在元素进入Stack的数组时设置值与`lastRecycleId`相等。后续通过该相等关系是否存在判断是否重复回收。

涉及到该规则的操作主要有：

\+ 从对象池中取出的对象时判断`recycleId`和`lastRecycleId`是否相等，否则抛出异常，相等则设置两个id为0. + 对象回收至本线程时判断是否2个ID为0，否的情况抛出异常。是则设置`recycleId=lastRecycleId=Recycler ID`。 + 对象回收至其他线程时设置`lastRecycleId=回收线程ID`。 + 对象从其他线程转移至`stack`时如果`recycleId`为0则设置`recycleId=lastRecycleId`.如果`recycleId`不为0，意味着在其他线程也执行了回收动作，抛出异常。

**这种重复检测机制并无法覆盖所有的情况，仍然是需要在编码过程中避免出错的出现重复回收代码**。举个例子：对象的`Stack`是 A 线程；B ，C 线程先后回收了对象；`Stack`执行 Pop 操作时将 B 线程对象转移至自身数组中，并且成功弹出，清空`recycleId`和`lastRecycleId`。后续相同的 Pop 操作转移了线程 C 的对象。此种情况就会造成一个对象回收2次并且被弹出2次。