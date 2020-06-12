# 12/20第12课 再谈弱引用 WeakReference

### 简单回顾

在上几篇的时候，已经简单的介绍了不正当使用 ThreadLocal 造成 OOM 的原因，以及 ThreadLocal 的基本原理，下边我们首先回顾一下 ThreadLocal 的原理图以及各类之间的关系：

**1、Thread、ThreadLocal、ThreadLocalMap、Entry 之间的关系（图 A）：**

![这里写图片描述](https://img-blog.csdn.net/20171020172529956?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上图中描述了：一个 Thread 中只有一个 ThreadLocalMap，一个 ThreadLocalMap 中可以有多个 ThreadLocal 对象，其中一个 ThreadLocal 对象对应一个 ThreadLocalMap 中一个的 Entry 实体（也就是说：一个 Thread 可以依附有多个 ThreadLocal 对象）。

**2、ThreadLocal 各类引用关系（图 B）：**

在 ThreadLocal 的生命周期中，都存在这些引用。（ 实线代表强引用，虚线代表弱引用）

![这里写图片描述](https://img-blog.csdn.net/20171020200142500?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

ThreadLocal 到 Entry 对象 key 的引用断裂，而不及时的清理 Entry 对象，可能会造成 OOM 内存溢出！

### 引用的类型

我们对引用的理解也许很简单，就是：如果 reference 类型的数据中存储的数值代表的是另外一块内存的起始地址，就称这块内存代表着一个引用。但是书上说的这种方式过于狭隘，一个对象在这种定义下只有被引用或者没有被引用两种状态，对于如何描述一些“食之无味，弃之可惜”的对象就显得无能为力。我们希望能描述这样一类对象：当内存空间还足够时，则能保留在内存之中；如果内存在进行垃圾收集后还是非常紧张，则可以抛弃这些对象。很多系统的缓存功能都符合这样的应用场景。

一般的引用类型分为：**强引用**（Strong Reference）、**软引用**（Soft Reference）、**弱引用**（Weak Reference）、**虚引用**（Phantom Reference）四种，这四种引用强度依次逐渐减弱。

**1、下边是四中类型的介绍：**

（1）**强引用**：就是指在程序代码之中普遍存在的，类似“Object obj = new Object()”这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象，也就是说即使Java虚拟机内存空间不足时，GC 收集器也绝不会回收该对象，如果内存空间不够就会导致内存溢出。

（2）**软引用**：用来描述一些还有用，但并非必需的对象。对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中并进行回收，以免出现内存溢出。如果这次回收还是没有足够的内存，才会抛出内存溢出异常。在 JDK 1.2 之后，提供了 SoftReference 类来实现软引用。

软引用适合引用那些可以通过其他方式恢复的对象，例如：数据库缓存中的对象就可以从数据库中恢复，所以软引用可以用来实现缓存。等会会介绍 MyBatis 中的使用软引用实现缓存的案例。

（3）**弱引用**：也是用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。**当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象**。在 JDK 1.2 之后，提供了 WeakReference 类来实现弱引用。ThreadLocal 使用到的就有弱引用。

（4）**虚引用**：也称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是希望能在这个对象被收集器回收时收到一个系统通知。在 JDK 1.2 之后，提供了PhantomReference 类来实现虚引用。

**2、各引用类型的生命周期及作用：**

![这里写图片描述](https://img-blog.csdn.net/20171112203109203?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### ThreadLocal 中的弱引用

上述我们知道了当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。我们的 ThreadLocal 中 ThreadLocalMap 中的 Entry 类的 key 就是弱引用的，如下：

![这里写图片描述](https://img-blog.csdn.net/20171112204117334?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

而弱引用会在垃圾收集器工作的时候进行回收，也就是说，只要执行垃圾回收，这些对象就会被回收，也就是上述图 B 中的虚线连接的地方断开了，就成了一个没有 key 的 Entry，下边演示一下：

**1、演示案例简介：**

我们知道一个线程 Thread 可以有多个 ThreadLocal 变量，这些变量存放在 Thread 中的 ThreadLocalMap 变量中，那么我们下边就在主线程 main 中定义多个 ThreadLocal 变量，然后我们想办法执行几次 GC 垃圾回收，再看一下 ThreadLocalMap 中 Entry 数组的变化情况。

**2、演示代码：**

```
public class ThreadLocalWeakReferenceGCDemo {

    private static final int THREAD_LOOP_SIZE = 20;

    public static void main(String[] args) throws InterruptedException {

        try {
            //等待连接JConsole
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        for (int i = 1; i < THREAD_LOOP_SIZE; i++) {
            ThreadLocal<Map<Integer, String>> threadLocal = new ThreadLocal<>();
            Map<Integer, String> map = new HashMap<>();
            map.put(i, "我是第" + i + "个ThreadLocal数据！");
            threadLocal.set(map);
            threadLocal.get();

            System.out.println("第" + i + "次获取ThreadLocal中的数据");

            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

**3、正常执行：**

当 for 循环执行到最后一个的时候，看一下 ThreadLocalMap 的情况：

![这里写图片描述](https://img-blog.csdn.net/20171112210220014?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看到此时的 ThreadLocalMap 中有21个 ThreadLocal 变量（也就是21个 Entry），其中有3个表示 main 线程中表示的其他 ThreadLocal 变量，这是正常的执行，并没有发生 GC 收集。

**3、非正常执行：**

当 for 循环执行到中间的时候手动执行 GC 收集，然后再看一下：

通过 JConsole 工具手动执行 GC 收集：

![这里写图片描述](https://img-blog.csdn.net/20171112210912357?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

执行结果：

![这里写图片描述](https://img-blog.csdn.net/20171112210834285?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看出算上主线程中其他的 Entry 一共还有6个，也就可以证明在执行 GC 收集的时候，弱引用被回收了。

**4、你可能会问道，弱引用被回收了只是回收了 Entry 的 key 引用，但是 Entry 应该还是存在的吧？**

事情是这样的，我们的 ThreadLocal 已经帮我们把 key 为 null 的 Entry 清理了，在 ThreadLocal 的`get(),set(),remove()`的时候都会清除线程 ThreadLocalMap 里所有 key 为 null 的 value。

![这里写图片描述](https://img-blog.csdn.net/20171112211306898?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上述源码中描述了清除并重建索引的过程，源码过多，不截图显示。所以，我们最后看到的实际上是已经清除过 key 为 null 的 Entry 之后的结果。这也说明了正常情况下使用 ThreadLocal 是不会出现 OOM 内存溢出的，出现内存溢出是和弱引用没有半点关系的！

**5、上述代码虽然是手动执行的 GC，但正常情况下的 GC 也是会回收弱引用的**。

如下（注意：实验请适当调节参数，避免电脑死机），假如我们上述的代码的主函数 main 改成如下方式：

```
public static void main(String[] args) throws InterruptedException {

        ThreadLocal<Map<Integer, String>> threadLocal1 = new ThreadLocal<>();
        Map<Integer, String> map1 = new HashMap<>(1);
        map1.put(1, "我是第1个ThreadLocal数据！");
        threadLocal1.set(map1);

        ThreadLocal<Map<Integer, String>> threadLocal2 = new ThreadLocal<>();
        Map<Integer, String> map2 = new HashMap<>(1);
        map2.put(2, "我是第2个ThreadLocal数据！");
        threadLocal2.set(map2);

        for (int i = 3; i <= MAIN_THREAD_LOOP_SIZE; i++) {
            ThreadLocal<Map<Integer, String>> threadLocal = new ThreadLocal<>();
            Map<Integer, String> map = new HashMap<>(1);
            map.put(i, "我是第" + i + "个ThreadLocal数据！");
            threadLocal.set(map);
            threadLocal.get();

            if (i > 20) {
                //-Xms20m -Xmx20m -Xmn10m -XX:+PrintGCDetails -XX:SurvivorRatio=8
                //会触发GC
                byte[] allocation1, allocation2, allocation3, allocation4;
                allocation1 = new byte[2 * 1024 * 1024];
                allocation2 = new byte[2 * 1024 * 1024];
                allocation3 = new byte[2 * 1024 * 1024];
                allocation4 = new byte[4 * 1024 * 1024];
            }
        }
        System.out.println("-------" + threadLocal1.get());
        System.out.println("-------" + threadLocal2.get());
    }
```

设置 VM 参数：

![这里写图片描述](https://img-blog.csdn.net/20171113134347068?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

最后的运行结果：

![这里写图片描述](https://img-blog.csdn.net/20171113134859865?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

调试中查看 threadLocal 的数据，如下：

![这里写图片描述](https://img-blog.csdn.net/20171113134745272?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](https://img-blog.csdn.net/20171113130705611?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可见，虽然这里我们自己定义了30个 ThreadLocal 变量，但是最后确只有14个，其中还有三个是属于其他的，还有一点值得注意的是，我们的`threadLocal1`和`threadLocal2` 变量，在进行GC垃圾回收的时候，弱引用的 Key 是没有进行回收的，最后存活了下来！使得我们最后通过 get 方法可以获取到正确的数据。

**6、为什么 threadLocal1 和 threadLocal2 变量没有被回收？**

这里我们就需要重新认识一下，什么是：当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象，这里的重点是：只被弱引用关联的对象

首先举个实例：

```
public class WeakRefrenceDemo {

    public static void main(String[] args) {
        User user = new User("hello", "123");
        WeakReference<User> userWeakReference = new WeakReference<>(user);
        System.out.println(userWeakReference.get());
        //另一种方式触发GC，强制执行GC
        System.gc();
        System.runFinalization();
        System.out.println(userWeakReference.get());
    }

    public static class User {
        private String userName;
        private String userPwd;
        //省去全参构造方法和toString()方法
    }
}
```

执行结果：

![这里写图片描述](https://img-blog.csdn.net/20171113164601012?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看到，上述过程尽管 GC 执行了垃圾收集，但是弱引用还是可以访问到结果的，也就是没有被回收，这是因为除了一个弱引用 userWeakReference 指向了 User 实例对象，还有 user 指向 User 的实例对象，只有当 user 和 User 实例对象的引用断了的时候，弱引用的对象才会被真正的回收，看下图：

![这里写图片描述](https://img-blog.csdn.net/20171113171119154?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

由上图可知道，`user`和`new User()`是在不同的内存空间的，他们之间是通过引用进行关联起来的。

如果把上述主函数改成代码如下，将`user = null`，则断开了他们之间的引用关系，但是还有一个弱引用 userWeakReference 指向`new User()`：

```
public static void main(String[] args) {

        User user = new User("hello", "123");
        WeakReference<User> userWeakReference = new WeakReference<>(user);
        System.out.println(userWeakReference.get());
        user = null; //断开引用
        System.gc(); //强制执行GC
        System.runFinalization();
        System.out.println(userWeakReference.get());
    }
```

执行结果如下：

![这里写图片描述](https://img-blog.csdn.net/20171113171408249?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看到断开了 user 和 new User() 之间的引用之后，就只有弱引用了，因此，上述的那段话：都会回收掉只被弱引用关联的对象。因此该 new User() 会被回收。

因此，就出现了最开始看到的 threadLocal1、threadLocal2 都还可以访问到数据（for 循环里边的，由于作用于的问题，引用已经断开了），那我我们只有通过手动设为 null 的方式，看一下效果，代码改为如下：

```
    public static void main(String[] args) throws InterruptedException {

        ThreadLocal<Map<Integer, String>> threadLocal1 = new ThreadLocal<>();
        Map<Integer, String> map1 = new HashMap<>(1);
        map1.put(1, "我是第1个ThreadLocal数据！");
        threadLocal1.set(map1);
        threadLocal1 = null;
        System.gc(); //强制执行GC
        System.runFinalization();

        ThreadLocal<Map<Integer, String>> threadLocal2 = new ThreadLocal<>();
        Map<Integer, String> map2 = new HashMap<>(1);
        map2.put(2, "我是第2个ThreadLocal数据！");
        threadLocal2.set(map2);
        threadLocal2 = null;
        System.gc(); //强制执行GC
        System.runFinalization();

        ThreadLocal<Map<Integer, String>> threadLocal3 = new ThreadLocal<>();
        Map<Integer, String> map3 = new HashMap<>(1);
        map3.put(3, "我是第3个ThreadLocal数据！");
        threadLocal3.set(map3);

        ThreadLocal<Map<Integer, String>> threadLocal4 = new ThreadLocal<>();
        Map<Integer, String> map4 = new HashMap<>(1);
        map4.put(4, "我是第4个ThreadLocal数据！");
        threadLocal4.set(map4);
        System.out.println("-------" + threadLocal3.get());
    }
}
```

执行结果：

![这里写图片描述](https://img-blog.csdn.net/20171113172525701?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看到，是我们想要的结果，弱引用也被回收了。

另外还有一种可能是，我们得到的结果有3个，分别是2、3、4，这是有可能的，这是由于垃圾回收器是一个优先级较低的线程， 因此不一定会很快发现那些只具有弱引用的对象，即只有等到系统垃圾回收机制运行时才会被回收，但是这里我们已经看到了我们想要的结果。

**7、总结**

到了这里，你应该明白，并不是所有弱引用的对象都会在第二次 GC 回收的时候被回收，而是回收掉只被弱引用关联的对象。因此，使用弱引用的时候要注意到！希望以后在面试的时候，不要上来张口就说，弱引用在第二次执行 GC 之后就会被回收！知其然，知其所以然！

### 引用队列

在很多场景中，我们的程序需要在一个对象的可达性（GC 可达性，判断对象是否需要回收）发生变化的时候得到通知，引用队列就是用于收集这些信息的队列。

在创建 SoftReference 对象时，可以为其关联一个引用队列，当 SoftReference 所引用的对象被回收的时候，Java 虚拟机就会将该 SoftReference 对象添加到预支关联的引用队列中。

需要检查这些通知信息时，就可以从引用队列中获取这些 SoftReference 对象。

不仅仅是 SoftReference 支持使用引用队列，软引用和虚引用也可以关相应的引用队列。

![这里写图片描述](https://img-blog.csdn.net/20171113181229884?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

先看一个简单的案例：

```
public class WeakCache {

    private void printReferenceQueue(ReferenceQueue<Object> referenceQueue) {
        WeakEntry sv;
        while ((sv = (WeakEntry) referenceQueue.poll()) != null) {
            System.out.println("引用队列中元素的key：" + sv.key);
        }
    }

    private static class WeakEntry extends WeakReference<Object> {
        private Object key;

        WeakEntry(Object key, Object value, ReferenceQueue<Object> referenceQueue) {
            //调用父类的构造函数，并传入需要进行关联的引用队列
            super(value, referenceQueue);
            this.key = key;
        }
    }

    public static void main(String[] args) {
        ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();
        User user = new User("xuliugen", "123456");
        WeakCache.WeakEntry weakEntry = new WeakCache.WeakEntry("654321", user, referenceQueue);
        System.out.println("还没被回收之前的数据：" + weakEntry.get());

        user = null;
        System.gc(); //强制执行GC
        System.runFinalization();

        System.out.println("已经被回收之后的数据：" + weakEntry.get());
        new WeakCache().printReferenceQueue(referenceQueue);
    }
}
```

执行结果：

![这里写图片描述](https://img-blog.csdn.net/20171113190428489?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

ReferenceQueue 引用队列记录了 GC 收集器回收的引用，这样的话，我们就可以通过引用队列的数据来判断引用是否被回收，以及被回收之后做相应的处理，例如：如果使用弱引用做缓存则需要清除缓存，或者重新设置缓存等。

其实，上述的代码，是从 MyBatis 的源码中抽离出来的，MyBatis 在缓存的时候也提供了对弱引用和软引用的支持，MyBatis 相关的源码如下：

![这里写图片描述](https://img-blog.csdn.net/20171113192249656?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

任何一个牛逼的框架，也是一个一个知识点的使用。这篇文章的内容很多，还是希望对你有所帮助！另外，个人能力有限，难免有所疏漏，如果有任何建议和意见，欢迎在读者圈进行交流。