# 29/31Netty 如何实现更快的 ThreadLocal

### 引言

在前面关于内存分配的代码分析文章中，我们在多个场合见到了一个类 FastThreadLocal。从名字可以看出，这应该是 ThreadLocal 的高性能版本。

在 Netty 中，为了打磨性能的细节，在很多方面都做了一些改造，这篇文章我们就来分析下关于 FastThreadLocal 设计与实现。

### 原生的 ThreadLocal

在介绍 Netty 的 FastThreadLocal 之前，我们先来看下再JDK中的 ThreadLocal 是怎么样的一种解决方案，才能对比出二者的设计上差异化思考。

ThreadLocal 的设计意图是让每一个线程都持有自身独立的变量副本，这样，多个线程之间使用的数据是独立的副本，没有交互，也就不会出现多线程并发下可能导致的数据并发安全问题。

#### **ThreadLocalMap**

要了解 ThreadLocal 的原理，首先先了解一个数据结构 ThreadLocal.ThreadLocalMap。这是一个内部对象，并没有对开发者暴露其 API，其 API 的使用主要都是通过 ThreadLocal 委托的。首先来看其属性定义，如下：

```java
private Entry[] table;//ThreadLocalMap 是一个 Map 映射的结构，使用数组作为背后的元素存储
private int size = 0; //当前 Map 内元素的个数
private int threshold; //当前 table 数组扩容的阀值
static class Entry extends WeakReference < ThreadLocal <? >>
{
    Object value;
    Entry(ThreadLocal <? > k, Object v)
    {
        super(k);
        value = v;
    }
}
```

这里面比较有意思是 Entry 这个类，它继承了 WeakReference，并且使用 ThreadLocal 对象作为引用对象。这意味着当 ThreadLocal 被GC回收后，这个 Entry 就无法定位了。这个特性，后续的方法中我们会看到它的用途。

**1. 设置元素**

当我们调用 ThreadLocal.set 方法时，最终会委托给 ThreadLocalMap 的 set 方法，如下：

```java
private void set(ThreadLocal <? > key, Object value)
{
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);
    for(Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)])
    {
        ThreadLocal <? > k = e.get();
        if(k == key)
        {
            e.value = value;
            return;
        }
        if(k == null)
        {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if(!cleanSomeSlots(i, sz) && sz >= threshold) rehash();
}
```

和 HashMap 的思路类似，首先是求出 Key 的 hash 作为散列的依据，只不过和 HashMap 中自己计算不同，这边是直接使用了 `ThreadLocal#threadLocalHashCode` 属性值作为散列依据，通过 `key.threadLocalHashCode & (len - 1)` 得到散列后在 table 数组中的下标。

不过与 HashMap 不同的是，如果散列后的槽位本身上已经有数据了，ThreadLocalMap 并不是在对应的槽位上通过链表挂载相同散列值的 Key，而是按照顺序寻找下一个槽位，并且再次判断，重复这个过程直到找到 key 相同或者 key 不存在的槽位，亦或者本身不存在元素的槽位。

如果找到对应的槽位，其上还不存在 Entry，则新建一个 Entry。每当新建 Entry 也就是新增元素成功，都会首先尝试清除“陈旧”的 Entry。上文说过，Entry 继承了 WeakReference，如果其引用的 ThreadLocal 对象被 GC 了，则意味着 Key 消失，也就是意味着 Entry 内的 value 无法被访问到。此时认为这个 Entry 陈旧，可以被清除。下面来看下 cleanSomeSlots 的具体实现，如下：

```java
private boolean cleanSomeSlots(int i, int n)
{
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if(e != null && e.get() == null)
        {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ((n >>>= 1) != 0);
    return removed;
}
```

入参 i 是刚刚新 Entry 插入的位置，必然不是陈旧，从这个槽位开始，向后检查 log2n 次。检查的方式就是通过 Entry.get 来确认其弱引用指向的 ThreadLocal 是否已经被回收。如果某个槽位上的 Entry 已经陈旧，通过方法 expungeStaleEntry 来清除它，代码如下：

```java
private int expungeStaleEntry(int staleSlot)
{
    Entry[] tab = table;
    int len = tab.length;
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;
    Entry e;
    int i;
    for(i = nextIndex(staleSlot, len);
        (e = tab[i]) != null; i = nextIndex(i, len))
    {
        ThreadLocal <? > k = e.get();
        if(k == null)
        {
            e.value = null;
            tab[i] = null;
            size--;
        }
        else
        {
            int h = k.threadLocalHashCode & (len - 1);
            if(h != i)
            {
                tab[i] = null;
                while(tab[h] != null) h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

expungeStaleEntry 方法的思路可以分为两个阶段：

- 第一阶段，将槽位 staleSlot 上的 Entry 的属性 value 设置为 null，使得其指向的对象可以被 GC。并且将该槽位设置为 null，让该 Entry 对象本身也被 GC。
- 第二阶段，从槽位 staleSlot 开始向后遍历直到空槽位为止，如果遇到 Entry.get 为 null 的情况，对应的将该槽位上的数据释放；如果 Entry.get 不为 null 的情况，判断 threadLocalHashCode 散列之后的槽位是否是当前的槽位，如果不是的话，则尝试将其挪动到散列后的正确位置上，如果对应位置有 Entry，继续向后挪动直到放下为止。

expungeStaleEntry 方法会尝试在一定数量范围内（从当前陈旧的槽位向后遍历到空槽位），清除弱引用被 GC 的 Entry 元素。并且在这个过程中协助将散列位置不足够恰当的元素尝试挪动到散列最恰当的位置。

cleanSomeSlots 方法执行完毕后，如果在扫描范围内没有陈旧的槽位，再检查当前元素个数是否超过了阈值，超过的情况下就启动扩容，也即是方法 rehash。rehash 方法内部实现很简单，首先是遍历 table 数组，清除所有的陈旧 Entry，在清楚陈旧 Entry 后剩余元素个数仍然超过 3/4 的给定阈值，则将数组扩容两倍长度，并且将旧数组中的元素放入新数组即可。

**2. 获取元素**

与设置元素相对的就是获取元素，当调用 ThreadLocal.get 时，底层是委托到 ThreadLocalMap.getEntry 方法上的，其代码如下：

```java
private Entry getEntry(ThreadLocal <? > key)
{
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if(e != null && e.get() == key) return e;
    else return getEntryAfterMiss(key, i, e);
}
```

如果不能直接命中，则进入慢速路径，也就是方法 getEntryAfterMiss，代码如下：

```java
private Entry getEntryAfterMiss(ThreadLocal <? > key, int i, Entry e)
{
     Entry[] tab = table;
     int len = tab.length;
     while(e != null)
     {
         ThreadLocal <? > k = e.get();
         if(k == key) return e;
         if(k == null) expungeStaleEntry(i);
         else i = nextIndex(i, len);
         e = tab[i];
     }
     return null;
}
```

从槽位 i 开始向后寻找，如果遇到槽位上的 Entry 的 key 与入参为同一对象，则意味着查询命中；如果在查询过程中遇到陈旧的槽位，则清除该槽位。如果槽位上的 key 不是要查询的对象，则向后继续寻找槽位。

当遇到空槽位，则意味着要寻找的元素在数组中不存在。

这里面有个小细节可以注意，`if(k == null) expungeStaleEntry(i)`，当槽位上的 Entry 为陈旧数据时，通过方法 expungeStaleEntry 清除数据，而该方法会继续遍历数组找到原本可以直接散列到该槽位上的 Entry 移动过来，因此如果存在可以命中的数据的情况下，此时 tab[i] 必然不为空。因此在这个 if 条件下，指针 i 就不需要移动了。

**3. 元素的删除**

有了上面的基础，看元素删除就很好理解了，其代码如下：

```java
private void remove(ThreadLocal <? > key)
{
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);
    for(Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)])
    {
        if(e.get() == key)
        {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

简单的概括就是通过散列找到直接槽位，遍历槽位直到寻找到正确槽位上的 Entry，通过 clear 方法删除其 key 的值，并且通过 expungeStaleEntry 清除这个陈旧的 Entry，并且向后寻找一个适合该槽位的 Entry 进行挪动。

#### **Thread、ThreadLocal、ThreadLocalMap 三者关系**

在 Thread 类中有一个属性 `Thread#threadLocals`，该属性正是 ThreadLocalMap。结合 ThreadLocalMap 中以 ThreadLocal 对象为 key，这三者的关系就很明确了，用图来表达就是：

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20200108192200.png)

实现线程变量，这种变量副本有两种可能的思路：

- 使用一个并发的 Map 结构，以 Thread 为 key，将线程变量和 Thread 对象关联在一起。此时，ThreadLocal 内部应该是一个 Map 类型的结构。
- 将 Thread 内部存储一个 Map 结构的属性，该结构以 ThreadLocal 为 key，将线程变量和 ThreadLocal 关联起来，这样不同的 ThreadLocal 在不同的 Thread 中就有不同的值了。

JDK 一开始的时候使用的是方案一。不过方案一会导致多线程在 Map 结构并发竞争，所以后来就修改为了方案二。也就是上图中的这种模式。

#### **ThreadLocal**

有了上面内容的铺垫后，再看 ThreadLocal 就很好理解了，仍然从 API 层面入手。首先是元素的设置，方法是 `ThreadLocal#set`，如下：

```java
public void set(T value)
{
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if(map != null) map.set(this, value);
    else createMap(t, value);
}
ThreadLocalMap getMap(Thread t)
{
    return t.threadLocals;
}
void createMap(Thread t, T firstValue)
{
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

代码十分简单，首先是通过方法 getMap 来得到当前线程的 threadLocals 属性的值。如果存在就直接设置，否则的话就创建一个 ThreadLocalMap 并且初始化就放入需要填入的值。

其他的诸如 get 和 remove 方法这边就不展示了，有了上面的基础，相信读者理解这些方法是很容易的。

### Netty 改进的 FastThreadLocal

JDK 中的 ThreadLocalMap 采用开放地址法进行槽位寻找和选择，如果 ThreadLocal 的散列值碰撞较多，无论是查询还是设置，需要检索的槽位也会比较多。Netty针对这一点进行了改进。

Netty 针对这一个情况，进行了一个微妙的改进：

- 设计了新的 InternalThreadLocalMap 类，内部使用一个 Object[] 存储线程变量副本，并且不使用hash算法进行槽位寻找。
- 设计了新的 FastThreadLocal 类，每一个 FastThreadLocal 都被赋予一个全局单调递增的 ID，这个 id 的值可以直接在 InternalThreadLocalMap 的数组上进行寻址。

Netty 通过将hash散列后寻找修改为数组下标直接寻址的方式来提高线程变量的获取性能，按照 Netty 的测试，在频繁获取的情况下，性能会有三倍的提升。了解了原理之后，考虑到 FastThreadLocal 和 ThreadLocal 的相似性，在代码方面就很好理解，这里我们以元素设置来展开分析下，通过方法 `FastThreadLocal#set(V)`，代码如下：

```java
public final void set(V value)
{
    if(value != InternalThreadLocalMap.UNSET)
    {
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        setKnownNotUnset(threadLocalMap, value);
    }
    else
    {
        remove();
    }
}
```

深入方法 InternalThreadLocalMap.get()，如下：

```java
public static InternalThreadLocalMap get()
{
    Thread thread = Thread.currentThread();
    if(thread instanceof FastThreadLocalThread)
    {
        return fastGet((FastThreadLocalThread) thread);
    }
    else
    {
        return slowGet();
    }
}
private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread)
{
    InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
    if(threadLocalMap == null)
    {
        thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
    }
    return threadLocalMap;
}
```

可以看到，如果当前的线程是 Netty 设计的 FastThreadLocalThread 类型，则直接获取其属性 threadLocalMap（ threadLocalMap() 方法的作用就是获取该属性）。如果该属性为 null，则创建 InternalThreadLocalMap 并赋值给它。最终返回该属性指向的 InternalThreadLocalMap 对象。这一点上，思路和 JDK 在 Thread 中定义 threadLocals 属性是一样的。都是通过在线程中的属性来避免并发竞争。

而如果当前线程并不是Netty的线程对象时，则回归到 JDK 的 ThreadLocal 模式上，slowGet 的代码如下：

```java
private static InternalThreadLocalMap slowGet()
{
    ThreadLocal < InternalThreadLocalMap > slowThreadLocalMap = UnpaddedInternalThreadLocalMap.slowThreadLocalMap;
    InternalThreadLocalMap ret = slowThreadLocalMap.get();
    if(ret == null)
    {
        ret = new InternalThreadLocalMap();
        slowThreadLocalMap.set(ret);
    }
    return ret;
}
```

通过一个静态不可变的类属性 UnpaddedInternalThreadLocalMap.slowThreadLocalMap 来获取本线程的 InternalThreadLocalMap。

显然，在 FastThreadLocalThread 中，只需要通过对象的属性即可获取到 InternalThreadLocalMap，而在其他线程类型中，仍然需要通过 ThreadLocal 才能获取 InternalThreadLocalMap，在这个地方就相对的慢上了。

回过头，我们再看看 set 方法中对 setKnownNotUnset 方法的调用，其代码如下：

```java
private void setKnownNotUnset(InternalThreadLocalMap threadLocalMap, V value)
{
    if(threadLocalMap.setIndexedVariable(index, value))
    {
        addToVariablesToRemove(threadLocalMap, this);
    }
}
public boolean setIndexedVariable(int index, Object value)
{
    Object[] lookup = indexedVariables;
    if(index < lookup.length)
    {
        Object oldValue = lookup[index];
        lookup[index] = value;
        return oldValue == UNSET;
    }
    else
    {
        expandIndexedVariableTableAndSet(index, value);
        return true;
    }
}
```

index 是 FastThreadLocal 的一个不可变属性，就是上文思路中提到的全局单调递增的 id 值。从方法 setIndexedVariable 也能看到该 id 值的作用就是在 InternalThreadLocalMap 的 Object[] 数组上进行下标寻址。

如果 index 的值超过了数组的长度，则将数组扩容后，再放入元素。扩容并放入元素的方法就是 expandIndexedVariableTableAndSet，代码很简单，就不展开了。

元素成功设置完毕后，Netty 在 InternalThreadLocalMap 数组的 0 下标位置，设置了一个 IdentityHashMap 转化成的集合，用于存放该线程上所有设置过值的 FastThreadLocal 对象，每次设置值成功时，都需要将 FastThreadLocal 对象添加到这个集合中。

这个集合的作用在方法 `FastThreadLocal#removeAll`，当要清空该线程所有的线程变量时，通过遍历这个集合，可以快速的清空这些 FastThreadLocal 设置的槽位。以避免在 InternalThreadLocalMap 数组上执行遍历并清空的动作，不过这个做法，实际上带来的效益并不是很高。

这边提一下为什么要提供 removeAll 这个方法。如果在一些容器环境比如 Tomcat 这种环境去使用 Netty，如果该线程的 InternalThreadLocalMap 对象中还存在值，则这个则该这个对象值会持有该对象类的引用，而该对象类的引用会持有加载这个 WebApp 的 WebAppClassLoader（Tomcat 中每一个应用都是单独的ClassLoader去加载），而 WebAppClassLoader 会持有所有由它加载的类。这里面就存在一条强引用链。

> Thread-->InternalThreadLocalMap-->value-->value.class-->WebAppClassLoader--> all classess loadby WebAppClassLoader

这种场景下，当Tomcat 对一个应用进行 reload 操作时，本应该销毁的 WebAppClassLoader 却无法销毁，导致其加载的类占据的内存空间也无法释放。多次之后就容易 OOM 了。

### 思考与总结

Netty 对于框架中可以改进的地方永远都是不放弃的。除了 ThreadLocal 外，Netty 还提供了许多其他高效的数据结构。下一个章节，我们就来分析下Netty 中为我们提供的高效的定时器实现。