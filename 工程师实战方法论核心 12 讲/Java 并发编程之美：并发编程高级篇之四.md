# Java 并发编程之美：并发编程高级篇之四

### 一、前言

Java 并发编程实践中的话：编写正确的程序并不容易，而编写正常的并发程序就更难了。相比于顺序执行的情况，多线程的线程安全问题是微妙而且出乎意料的，因为在没有进行适当同步的情况下多线程中各个操作的顺序是不可预期的。

并发编程相比 Java 中其他知识点学习起来门槛相对较高，学习起来比较费劲，从而导致很多人望而却步；而无论是职场面试和高并发高流量的系统的实现却都还离不开并发编程，从而导致能够真正掌握并发编程的人才成为市场比较迫切需求的。

本 Chat 作为 Java 并发编程之美系列的高级篇之四，图形结合讲解 JDK 中线程安全的并发队列实现原理，内容如下：（建议先阅读 [并发编程高级篇之三 - 锁](http://gitbook.cn/gitchat/activity/5ac85e1b2a04fd6c837137a2) ）

- JDK 中基于链表的非阻塞无界队列 ConcurrentLinkedQueue 原理剖析，ConcurrentLinkedQueue 内部是如何使用 CAS 非阻塞算法来保证多线程下入队出队操作的线程安全？
- JDK 中基于链表的阻塞队列 LinkedBlockingQueue 原理剖析，LinkedBlockingQueue 内部是如何使用两个独占锁 ReentrantLock 以及对应的条件变量保证多线程先入队出队操作的线程安全？为什么不使用一把锁，使用两把为何能提高并发度？
- JDK 中基于数组的阻塞队列 ArrayBlockingQueue 原理剖析，ArrayBlockingQueue 内部如何基于一把独占锁以及对应的两个条件变量实现出入队操作的线程安全？
- JDK 中无界优先级队列 PriorityBlockingQueue 原理剖析，PriorityBlockingQueue 内部使用堆算法保证每次出队都是优先级最高的元素，元素入队时候是如何建堆的，元素出队后如何调整堆的平衡的？
- 浅谈上面各种队列的比较，以及给出部分队列在开源框架中使用样例。

### 二、ConcurrentLinkedQueue 原理探究

ConcurrentLinkedQueue 是线程安全的无界非阻塞队列，底层数据结构使用单向链表实现，入队和出队操作使用 CAS 来实现线程安全。

#### 2.1 ConcurrentLinkedQueue 类图结构

先简单介绍下 ConcurrentLinkedQueue 的类图结构如下图：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/26b3d73c402600495cd8e3dfde8ed6fe.png)

- 如上类图 ConcurrentLinkedQueue 内部的队列是使用单向链表方式实现，其中两个 volatile 类型的 Node 节点分别用来存放队列的首尾节点。从下面无参构造函数可知默认头尾节点都是指向 item 为 null 的哨兵节点。

```
public ConcurrentLinkedQueue() {
   head = tail = new Node<E>(null);
}
```

- Node 节点内部则维护一个 volatile 修饰的变量 item 用来存放节点的值，next 用来存放链表的下一个节点，从而链接为一个单向无界链表。

首先一个图来概况该队列构成，读者可以读完本节后在回头体会这个图： ![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/7621024c726c57326fe104f865647fa6.png)

#### 2.2 ConcurrentLinkedQueue 原理介绍

本节主要介绍 ConcurrentLinkedQueue 的几个主要的方法的实现原理

##### 2.2.1 offer 操作

offer 操作是在队列末尾添加一个元素，如果传递的参数是 null 则抛出 NPE 异常，否者由于 ConcurrentLinkedQueue 是无界队列该方法一直会返回 true。另外由于使用 CAS 无阻塞算法，该方法不会阻塞调用线程，下面具体看看实现原理。

```
public boolean offer(E e) {
    //（1）e为null则抛出空指针异常
    checkNotNull(e);

   //（2）构造Node节点
    final Node<E> newNode = new Node<E>(e);

    //（3）从尾节点进行插入
    for (Node<E> t = tail, p = t;;) {

        Node<E> q = p.next;

        //（4）如果q==null说明p是尾节点，则执行插入
        if (q == null) {

            //（5）使用CAS设置p节点的next节点
            if (p.casNext(null, newNode)) {
                //（6）cas成功，则说明新增节点已经被放入链表，然后设置当前尾节点
                if (p != t)
                    casTail(t, newNode);  // Failure is OK.
                return true;
            }
        }
        else if (p == q)//(7）
            //多线程操作时候，由于poll操作移除元素后有可能会把head变为自引用，然后head的next变为新head，所以这里需要
            //重新找新的head，因为新的head后面的节点才是正常的节点。
            p = (t != (t = tail)) ? t : head;
        else
            //（8） 寻找尾节点
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

上节类图结构时候谈到构造队列时候参构造函数创建了一个 item 为 null 的哨兵节点，并且 head 和 tail 都是指向这个节点，下面通过图形结合来讲解下 offer 操作的代码实现。

- 首先看下当一个线程调用 offer（item）时候情况：首先代码（1）对传参判断空检查，如果为 null 则抛出 NPE 异常，然后代码（2）则使用 item 作为构造函数参数创建了一个新的节点，代码（3）从队列尾部节点开始循环，意图是从队列尾部添加元素。

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/d61e8ead7f4dc464f09b74659c74d634.png)

上图是执行代码（4）时候队列的情况，这时候节点 p,t,head,tail 同时指向了 item 为 null 的哨兵节点，由于哨兵节点的 next 节点为 null, 所以这里 q 指向也是 null。

代码（4）发现`q==null`则执行代码（5）通过 CAS 原子操作判断 p 节点的 next 节点是否为 null，如果为 null 则使用节点 newNode 替换 p 的 next 节点，然后执行代码（6）由于`p==t`所以没有设置尾部节点，然后退出 offer 方法，这时候队列的状态图如下：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/57faa211f7607da2287472a513eb2c7c.png)

- 上面讲解的是一个线程调用 offer 方法的情况，如果多个线程同时调用，就会存在多个线程同时执行到代码（5），假设线程 A 调用 offer（item1), 线程 B 调用 offer(item2), 线程 A 和 B 同时执行到 p.casNext(null, newNode)。而 CAS 的比较并设置操作是原子性的，假设线程 A 先执行了比较设置操作则发现当前 p 的 next 节点确实是 null 则会原子性更新 next 节点为 newNode，这时候线程 B 也会判断 p 的 next 节点是否为 null，结果发现不是 null（因为线程 A 已经设置了 p 的 next 为 newNode）则会跳到步骤（3），然后执行到步骤（4）时候队列分布图为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/017f2f8c5c923c79dbe14e8d5a078460.png)

根据这个状态图可知线程 B 会去执行代码（8），然后 q 赋值给了 p，这时候队列状态图为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/9276d3ba8af2ececc7582e08573d033e.png)

然后线程 B 再次跳转到代码（3）执行，当执行到代码（4）时候队列状态图为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/aa112a8082be1c8a9733264130186cd1.png)

由于这时候 q==null, 所以线程 B 会执行步骤（5），通过 CAS 操作判断当前 p 的 next 节点是否是 null，不是则再次循环后尝试，是则使用 newNode 替换，假设 CAS 成功了，那么执行步骤（6）由于 p!=t 所以设置 tail 节点为 newNode，然后退出 offer 方法。这时候队列分布图为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/c80654106179a24f3565939b9f660fcf.png)

- 分析到现在，offer 代码的执行路径现在就差步骤（7）还没走过，其实这个要在执行 poll 操作后才会出现，这里先看下执行 poll 操作后可能会存在的的一种情况如下图：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/2dd55c8692cca12fdcb7d0fc5fa3d10f.png)

下面分析下当队列处于这种状态时候调用 offer 添加元素代码执行到步骤（4）时候的状态图：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/5877319882ed1cfd85f37b813f70475e.png)

由于 q 节点不为空并且`p==q`所以执行步骤（7），由于`t==tail`所以 p 被赋值为了 head，然后进入循环，循环后执行到代码（4）时候队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/1928ba0bf8ac8581c4ad3aadb0f53032.png)

由于`q==null`, 所以执行步骤（5）进行 CAS 操作，如果当前没有其他线程执行 offer 操作，则 CAS 操作会成功，p 的 next 节点被设置为新增节点，然后执行步骤（6），由于`p!=t`所以设置新节点为队列为节点，现在队列状态如下：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/27edc3381499e89efe00a9068db4ef20.png)

这里自引用的节点会被垃圾回收掉。

总结：可见 offer 操作里面关键步骤是代码（5）通过原子 CAS 操作来进行控制同时只有一个线程可以追加元素到队列末尾，进行 cas 竞争失败的线程则会通过循环一次次尝试进行 cas 操作，直到 cas 成功才会返回，也就是通过使用无限循环里面不断进行 CAS 尝试方式来替代阻塞算法挂起调用线程，相比阻塞算法这是使用 CPU 资源换取阻塞所带来的开销。

##### 2.2.2 poll 操作

poll 操作是在队列头部获取并且移除一个元素，如果队列为空则返回 null，下面看看实现原理。

```
public E poll() {
    //(1) goto标记
    restartFromHead:

    //（2）无限循环
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {

            //（3）保存当前节点值
            E item = p.item;

            //（4）当前节点有值则cas变为null
            if (item != null && p.casItem(item, null)) {
                //（5）cas成功标志当前节点以及从链表中移除
                if (p != h) 
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            //（6）当前队列为空则返回null
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
            //（7）自引用了，则重新找新的队列头节点
            else if (p == q)
                continue restartFromHead;
            else//(8）
                p = q;
        }
    }
 }
    final void updateHead(Node<E> h, Node<E> p) {
        if (h != p && casHead(h, p))
            h.lazySetNext(h);
    }
```

同理本节也通过图形结合的方式来讲解代码执行逻辑：

- poll 操作是从队头获取元素，所以代码（2）内层循环是从 head 节点开始迭代，代码（3）获取当前队头的节点，当队列一开始为空时候队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/a7cbfe45c011fe50a50f4235e4203502.png)

由于 head 节点指向的为 item 为 null 的哨兵节点，所以会执行到代码（6），假设这个过程中没有线程调用 offer 方法，则此时 q 等于 null 如下图：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/f1958ebf9703883c6b432b3b064fcf6f.png)

所以执行 updateHead 方法，由于 h 等于 p 所以没有设置头结点，poll 方法直接返回 null。

- 假设执行到代码（6）时候已经有其它线程调用了 offer 方法成功添加一个元素到队列，这时候 q 指向的是新增元素的节点，这时候队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/6153fd7e8a8e82abeb506634830bc6b2.png)

所以代码（6）判断结果为 false，然后会转向代码（7）执行，而此时 p 不等于 q，所以转向代码（8）执行，执行结果是 p 指向了节点 q，此时队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/64939428eebbf40ebc489e84127f0d37.png)

然后程序转向代码（3）执行，p 现在指向的元素值不为 null，则执行`p.casItem(item, null)` 通过 CAS 操作尝试设置 p 的 item 值为 null，如果此时没有其它线程进行 poll 操作，CAS 成功则执行代码（5）由于此时 p!=h 所以设置头结点为 p，poll 然后返回被从队列移除的节点值 item。此时队列状态为:

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/2dd55c8692cca12fdcb7d0fc5fa3d10f.png)

这个状态就是讲解 offer 操作时候，offer 代码的执行路径（7）执行的前提状态。

- 假如现在一个线程调用了 poll 操作，则在执行代码（4) 时候队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/0abe43eba806d1b96d7543efaeb070a5.png)

可知这时候执行代码（6）返回 null.

- 现在 poll 的代码还有个分支（7）没有执行过，那么什么时候会执行那？下面来看看，假设线程 A 执行 poll 操作时候当前队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/185e3231f3d51cb5118d85922058a06f.png)

那么执行`p.casItem(item, null)` 通过 CAS 操作尝试设置 p 的 item 值为 null。

假设 CAS 设置成功则标示该节点从队列中移除了，此时队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/420427cb5140c14c8a4a06204441194b.png)

然后由于 p!=h, 所以会执行 updateHead 方法，假如线程 A 执行 updateHead 前另外一个线程 B 开始 poll 操作这时候线程 B 的 p 指向 head 节点，但是还没有执行到代码（6）这时候队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/5865fefec8f461d964fce9b5d4baae4a.png)

然后线程 A 执行 updateHead 操作，执行完毕后线程 A 退出，这时候队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/f08672df0a7c44f15e5ea695cf8a6ba5.png)

然后线程 B 继续执行代码（6）`q=p.next`由于该节点是自引用节点所以`p==q`所以会执行代码（7）跳到外层循环 restartFromHead，重新获取当前队列队头 head, 现在状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/39f71f385a40cb8985410619c61e7c19.png)

总结：poll 方法移除一个元素时候只是简单的使用 CAS 操作把当前节点的 item 值设置 null，然后通过重新设置头结点让该元素从队列里面摘除，被摘除的节点就成了孤立节点，这个节点会被在垃圾回收的时候会回收掉。另外执行分支中如果发现头节点被修改了要跳到外层循环重新获取新的头节点。

##### 2.2.3 peek 操作

peek 操作是获取队列头部一个元素（只不获取不移除），如果队列为空则返回 null，下面看看实现原理。

```
public E peek() {
   //(1)
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            //(2)
            E item = p.item;
            //(3)
            if (item != null || (q = p.next) == null) {
                updateHead(h, p);
                return item;
            }
            //(4)
            else if (p == q)
                continue restartFromHead;
            else
            //(5)
                p = q;
        }
    }
}
```

代码结构与 poll 操作类似，不同在于步骤（3）的使用只是少了 castItem 操作，其实这很正常，因为 peek 只是获取队列头元素值并不清空其值，根据前面我们知道第一次执行 offer 后 head 指向的是哨兵节点（也就是 item 为 null 的节点），那么第一次 peek 时候代码（3）中会发现 item==null, 然后会执行 q = p.next, 这时候 q 节点指向的才是队列里面第一个真正的元素或者如果队列为 null 则 q 指向 null。

- 当队列为空时候这时候队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/dcfc69e86b38952bdd88731c69e04d28.png)

这时候执行 updateHead 由于 h 节点等于 p 节点所以不进行任何操作，然后 peek 操作会返回 null。

- 当队列至少有一个元素时候（这里假设只有一个）这时候队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/f5e7668b503a33a2791b61086a682461.png)

这时候执行代码（5）这时候 p 指向了 q 节点，然后执行代码（3）这时候队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/dd4b6e56c6ae1b63b21b20e43e789c3e.png)

执行代码（3）发现 item 不为 null，则执行 updateHead 方法，由于 h!=p, 所以设置头结点，设置后队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/e883d0f30c64deafb2e3949ce3eb2e1d.png)

也就是剔除了哨兵节点。

总结：peek 操作代码与 poll 操作类似只是前者只获取队列头元素但是并不从队列里面删除，而后者获取后需要从队列里面删除，另外在第一次调用 peek 操作时候，会删除哨兵节点，并让队列的 head 节点指向队列里面第一个元素或者 null。

##### 2.2.4 size 操作

获取当前队列元素个数，在并发环境下不是很有用，因为 CAS 没有加锁所以从调用 size 函数到返回结果期间有可能增删元素，导致统计的元素个数不精确。

```
public int size() {
    int count = 0;
    for (Node<E> p = first(); p != null; p = succ(p))
        if (p.item != null)
            // 最大返回Integer.MAX_VALUE
            if (++count == Integer.MAX_VALUE)
                break;
    return count;
}

//获取第一个队列元素（哨兵元素不算），没有则为null
Node<E> first() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            boolean hasItem = (p.item != null);
            if (hasItem || (q = p.next) == null) {
                updateHead(h, p);
                return hasItem ? p : null;
            }
            else if (p == q)
                continue restartFromHead;
            else
                p = q;
        }
    }
}

//获取当前节点的next元素，如果是自引入节点则返回真正头节点
final Node<E> succ(Node<E> p) {
    Node<E> next = p.next;
    return (p == next) ? head : next;
}
```

##### 2.2.5 remove 操作

如果队列里面存在该元素则删除给元素，如果存在多个则删除第一个，并返回 true，否者返回 false

```
public boolean remove(Object o) {

    //查找元素为空，直接返回false
    if (o == null) return false;
    Node<E> pred = null;
    for (Node<E> p = first(); p != null; p = succ(p)) {
        E item = p.item;

        //相等则使用cas值null,同时一个线程成功，失败的线程循环查找队列中其它元素是否有匹配的。
        if (item != null &&
            o.equals(item) &&
            p.casItem(item, null)) {

            //获取next元素
            Node<E> next = succ(p);

            //如果有前驱节点，并且next不为空则链接前驱节点到next,
            if (pred != null && next != null)
                pred.casNext(p, next);
            return true;
        }
        pred = p;
    }
    return false;
}
```

注：ConcurrentLinkedQueue 底层使用单向链表数据结构来保存队列元素，每个元素被包装为了一个 Node 节点，队列是靠头尾节点来维护的，创建队列时候头尾节点指向一个 item 为 null 的哨兵节点，第一次 peek 或者 first 时候会把 head 指向第一个真正的队列元素。由于使用非阻塞 CAS 算法，没有加锁，所以获取 size 的时候有可能进行了 offer，poll 或者 remove 操作，导致获取的元素个数不精确，所以在并发情况下 size 函数不是很有用。

### 三、LinkedBlockingQueue 原理探究

前面介绍了使用 CAS 算法实现的非阻塞队列 ConcurrentLinkedQueue，下面就来介绍下使用独占锁实现的阻塞队列 LinkedBlockingQueue 的实现

#### 3.1 LinkedBlockingQueue 类图结构

同理首先看下 LinkedBlockingQueue 的类图结构

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/de4f2699dd15bfbdcba3ef4b197a56eb.png)

如上类图可知 LinkedBlockingQueue 也是使用单向链表实现，也有两个 Node 分别用来存放首尾节点，并且里面有个初始值为 0 的原子变量 count 用来记录队列元素个数。另外里面有两个 ReentrantLock 的实例，分别用来控制元素入队和出队的原子性，其中 takeLock 用来控制同时只有一个线程可以从队列获取元素，其它线程必须等待，putLock 控制同时只能有一个线程可以获取锁去添加元素，其它线程必须等待。另外 notEmpty 和 notFull 是信号量，内部分别有一个条件队列用来存放进队和出队时候被阻塞的线程，其实这个是个生产者 - 消费者模型。如下是独占锁创建代码：

```
    /** 执行take, poll等操作时候需要获取该锁 */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** 当队列为空时候执行出队操作（比如take）的线程会被放入这个条件队列进行等待 */
    private final Condition notEmpty = takeLock.newCondition();

    /** 执行put, offer等操作时候需要获取该锁*/
    private final ReentrantLock putLock = new ReentrantLock();

    /**当队列满时候执行进队操作（比如put)的线程会被放入这个条件队列进行等待 */
    private final Condition notFull = putLock.newCondition();

 /** 当前队列元素个数 */
    private final AtomicInteger count = new AtomicInteger(0);
```

如下是 LinkedBlockingQueue 无参构造函数代码：

```
public static final int   MAX_VALUE = 0x7fffffff;

public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}

  public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    //初始化首尾节点,指向哨兵节点
    last = head = new Node<E>(null);
}
```

从代码可知默认队列容量为 0x7fffffff; 用户也可以自己指定容量，所以一定程度上 LinkedBlockingQueue 可以说是有界阻塞队列。

首先使用一个图来概况该队列，读者在读完本节后在回头体会下:

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/683b1c553cb71bdf84e6c3ef5a8b8d71.png)

#### 3.2 LinkedBlockingQueue 原理介绍

##### 3.2.1 offer 操作

向队列尾部插入一个元素，如果队列有空闲容量则插入成功后返回 true，如果队列已满则丢弃当前元素然后返回 false，如果 e 元素为 null 则抛出 NullPointerException 异常，另外该方法是非阻塞的。

```
    public boolean offer(E e) {

        //（1）空元素抛空指针异常
        if (e == null) throw new NullPointerException();

        //(2) 如果当前队列满了则丢弃将要放入的元素，然后返回false
        final AtomicInteger count = this.count;
        if (count.get() == capacity)
            return false;

        //(3) 构造新节点，获取putLock独占锁
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            //(4)如果队列不满则进队列，并递增元素计数
            if (count.get() < capacity) {
                enqueue(node);
                c = count.getAndIncrement();
                //(5)
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            //(6)释放锁
            putLock.unlock();
        }
        //(7)
        if (c == 0)
            signalNotEmpty();
        //(8)
        return c >= 0;
    }

private void enqueue(Node<E> node) {   
 last = last.next = node;
}
```

- 步骤（2）判断如果当前队列已满则丢弃当前元素并返回 false
- 步骤（3）获取到 putLock 锁，当前线程获取到该锁后，则其它调用 put 和 offer 的线程将会被阻塞（阻塞的线程被放到 putLock 锁的 AQS 阻塞队列）。
- 步骤（4）这里有重新判断了下当前队列是否满了，这是因为在执行代码（2）和获取到 putLock 锁期间可能其它线程通过 put 或者 offer 方法向队列里面添加了新元素。重新判断队列确实不满则新元素入队，并递增计数器。
- 步骤（5）判断如果新元素入队后队列还有空闲空间，则唤醒 notFull 的条件队列里面因为调用了 notFull 的 await 操作（比如执行 put 方法而队列满了的时候）而被阻塞的一个线程，因为队列现在有空闲所以这里可以提前唤醒一个入队线程。
- 代码（6) 则释放获取的 putLock 锁，这里要注意锁的释放一定要在 finally 里面做，因为即使 try 块抛异常了，finally 也是会被执行到的。另外释放锁后其它因为调用 put 和 offer 而被阻塞的线程将会有一个获取到该锁。
- 代码（7）c==0 说明在执行代码（6）释放锁时候队列里面至少有一个元素，队列里面有元素则执行 signalNotEmpty，下面看看 signalNotEmpty 的代码：

```
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }
```

可知作用是激活 notEmpty 的条件队列中因为调用 notEmpty 的 await 方法（比如调用 take 方法并且队列为空的时候）而被阻塞的一个线程，这里也说明了调用条件变量的方法前要首先获取对应的锁。

综上可知 offer 方法中通过使用 putLock 锁保证了在队尾新增元素的原子性和队列元素个数的比较和递增操作的原子性。

##### 3.2.2 put 操作

向队列尾部插入一个元素，如果队列有空闲则插入后直接返回 true，如果队列已满则阻塞当前线程直到队列有空闲插入成功后返回 true，如果在阻塞的时候被其它线程设置了中断标志，则被阻塞线程会抛出 InterruptedException 异常而返回，另外如果 e 元素为 null 则抛出 NullPointerException 异常。

put 操作的代码结构与 offer 操作类似，代码如下：

```
public void put(E e) throws InterruptedException {
        //（1）空元素抛空指针异常
        if (e == null) throw new NullPointerException();
        //(2) 构建新节点，并获取独占锁putLock
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            //(3)如果队列满则等待
            while (count.get() == capacity) {
                notFull.await();
            }
            //（4）进队列并递增计数
            enqueue(node);
            c = count.getAndIncrement();
            //(5)
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            //(6)
            putLock.unlock();
        }
        //(7)
        if (c == 0)
            signalNotEmpty();
    }
```

- 代码（2）中使用 putLock.lockInterruptibly() 获取独占锁，相比 offer 方法中这个获取独占锁方法意味着可以被中断，具体说是当前线程在获取锁的过程中，如果被其它线程设置了中断标志则当前线程会抛出 InterruptedException 异常，所以 put 操作在获取锁过程中是可被中断的。
- 代码（3）如果当前队列已满，则调用 notFull 的 await() 把当前线程放入 notFull 的条件队列，当前线程被阻塞挂起并释放获取到的 putLock 锁，由于 putLock 锁被释放了，所以现在其它线程就有机会获取到 putLock 锁了。
- 另外考虑下代码（3）判断队列是否为空为何使用 while 循环而不是 if 语句那？其实是考虑到当前线程被虚假唤醒的问题，也就是其它线程没有调用 notFull 的 singal 方法时候 notFull.await() 在某种情况下会自动返回。如果使用 if 语句那么虚假唤醒后会执行代码（4）元素入队，并且递增计数器，而这时候队列已经是满了的，导致队列元素个数大于了队列设置的容量，导致程序出错。而使用 while 循环假如 notFull.await() 被虚假唤醒了，那么循环在检查一下当前队列是否是满的，如果是则再次进行等待。

##### 3.2.3 poll 操作

从队列头部获取并移除一个元素，如果队列为空则返回 null，该方法是不阻塞的。

```
    public E poll() {
        //(1)队列为空则返回null
        final AtomicInteger count = this.count;
        if (count.get() == 0)
            return null;
        //(2)获取独占锁
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            //(3)队列不空则出队并递减计数
            if (count.get() > 0) {//3.1
                x = dequeue();//3.2
                c = count.getAndDecrement();//3.3
                //(4)
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            //(5)
            takeLock.unlock();
        }
        //(6)
        if (c == capacity)
            signalNotFull();
        //(7)返回
        return x;
    }
    private E dequeue() {
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }
```

- 代码 (1) 如果当前队列为空，则直接返回 null
- 代码（2）获取独占锁 takeLock，当前线程获取该锁后，其它线程在调用 poll 或者 take 方法会被阻塞挂起
- 代码 (3) 如果当前队列不为空则进行出队操作，然后递减计数器。
- 代码（4）如果 c>1 则说明当前线程移除掉队列里面的一个元素后队列不为空（c 是删除元素前队列元素个数），那么这时候就可以激活因为调用 poll 或者 take 方法而被阻塞到 notEmpty 的条件队列里面的一个线程。
- 代码（6）说明当前线程移除队头元素前当前队列是满的，移除队头元素后队列当前至少有一个空闲位置，那么这时候就可以调用 signalNotFull 激活因为调用 put 或者 offer 而被阻塞放到 notFull 的条件队列里的一个线程，signalNotFull 的代码如下：

```
      private void signalNotFull() {
          final ReentrantLock putLock = this.putLock;
          putLock.lock();
          try {
              notFull.signal();
          } finally {
              putLock.unlock();
          }
      }
```

poll 代码逻辑比较简单，值得注意的是获取元素时候只操作了队列的头节点。

##### 3.2.4 peek 操作

获取队列头部元素但是不从队列里面移除，如果队列为空则返回 null，该方法是不阻塞的。

```
     public E peek() {
        //(1)
        if (count.get() == 0)
            return null;
        //(2)
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            Node<E> first = head.next;
            //(3)
            if (first == null)
                return null;
            else
            //(4)
                return first.item;
        } finally {
           //(5)
            takeLock.unlock();
        }
    }
```

peek 操作代码也比较简单，这里需要注意的是代码（3）这里还是需要判断下 first 是否为 null 的，不能直接执行代码（4）。正常情况下执行到代码（2）说明队列不为空，但是代码（1）和（2）不是原子性操作，也就是在执行点（1）判断队列不空后，在代码（2）获取到锁前有可能其它线程执行了 poll 或者 take 操作导致队列变为了空，然后当前线程获取锁后，直接执行 first.item 会抛出空指针异常。

##### 3.2.5 take 操作

获取当前队列头部元素并从队列里面移除，如果队列为空则阻塞调用线程。如果队列为空则阻塞当前线程直到队列不为空然后返回元素，如果在阻塞的时候被其它线程设置了中断标志，则被阻塞线程会抛出 InterruptedException 异常而返回。

```
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        //(1)获取锁
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            //(2)当前队列为空则阻塞挂起
            while (count.get() == 0) {
                notEmpty.await();
            }
            //(3)出队并递减计数
            x = dequeue();
            c = count.getAndDecrement();
            //(4)
            if (c > 1)
                notEmpty.signal();
        } finally {
           //(5)
            takeLock.unlock();
        }
        //(6)
        if (c == capacity)
            signalNotFull();
        //(7)
        return x;
    }
```

- 代码（1）当前线程获取到独占锁，其它调用 take 或者 poll 的线程将会被阻塞挂起。
- 代码（2）如果队列为空则阻塞挂起当前线程，并把当前线程放入 notEmpty 的条件队列。
- 代码（3）进行出队操作并递减计数。
- 代码（4）如果 c>1 说明当前队列不为空，则唤醒 notEmpty 的条件队列的条件队列里面的一个因为调用 take 或者 poll 而被阻塞的线程。
- 代码（5）释放锁。
- 代码（6）如果 c == capacity 则说明当前队列至少有一个空闲位置，则激活条件变量 notFull 的条件队列里面的一个因为调用 put 或者 offer 而被阻塞的线程。

##### 3.2.6 remove 操作

删除队列里面指定元素，有则删除返回 true，没有则返回 false

```
public boolean remove(Object o) {
    if (o == null) return false;

    //（1）双重加锁
    fullyLock();
    try {

        //（2)遍历队列找则删除返回true
        for (Node<E> trail = head, p = trail.next;
             p != null;
             trail = p, p = p.next) {
             //(3)
            if (o.equals(p.item)) {
                unlink(p, trail);
                return true;
            }
        }
        //(4)找不到返回false
        return false;
    } finally {
        //(5)解锁
        fullyUnlock();
    }
}
```

- 代码（1）通过 fullyLock 获取双重锁，当前线程获取后，其它线程进行入队或者出队的操作时候就会被阻塞挂起。

```
void fullyLock() {
    putLock.lock();
    takeLock.lock();
}
```

- 代码（2）遍历队列寻找要删除的元素，找不到则直接返回 false，找到则执行 unlink 操作，unlik 操作代码如下：

```
    void unlink(Node<E> p, Node<E> trail) {
      p.item = null;
      trail.next = p.next;
      if (last == p)
          last = trail;
      如果当前队列满，删除后，也不忘记唤醒等待的线程
      if (count.getAndDecrement() == capacity)
          notFull.signal();
    }
```

可知删除元素后，如果发现当前队列有空闲空间，则唤醒 notFull 的条件队列中一个因为调 用 put 或者 offer 方法而被阻塞的线程。

- 代码（5）调用 fullyUnlock 方法使用与加锁顺序相反的顺序释放双重锁

```
void fullyUnlock() {
    takeLock.unlock();
    putLock.unlock();
}
```

总结下，由于 remove 方法在删除指定元素前加了两把锁，所以在遍历队列查找指定元素过程中是线程安全的，并且此时其它调用入队出队操作的线程全部会被阻塞，另外获取多个资源锁与释放的顺序是相反的。

##### 3.2.7 size 操作

- int size() : 获取当前队列元素个数。

```
    public int size() {
        return count.get();
    }
```

由于在操作出队入队时候操作 Count 的时候是加了锁的，所以相比 ConcurrentLinkedQueue 的 size 方法比较准确。这里考虑下为何 ConcurrentLinkedQueue 中需要遍历链表来获取 size 而不适用一个原子变量那？这是因为使用原子变量保存队列元素个数需要保证入队出队操作和操作原子变量是原子性操作，而 ConcurrentLinkedQueue 是使用 CAS 无锁算法的，所以无法做到这个。

注：LinkedBlockingQueue 内部是通过单向链表实现，使用头尾节点来进行入队和出队操作，也就是入队操作都是对尾节点进行操作，出队操作都是对头节点进行操作，而头尾节点的操作分别使用了单独的独占锁保证了原子性，所以出队和入队操作是可以同时进行的。另外头尾节点的独占锁都配备了一个条件队列，用来存放被阻塞的线程，并结合入队出队操作实现了一个生产消费模型。

### 四、ArrayBlockingQueue 原理探究

上节介绍了有界链表方式的阻塞队列 LinkedBlockingQueue，本节来研究下有界使用数组方式实现的阻塞队列 ArrayBlockingQueue 的原理

#### 4.1 ArrayBlockingQueue 类图结构

同理为了能从全局一览 ArrayBlockingQueue 的内部构造，先看下类图：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/4ef4e76cd9eaa123f7a9812831e9dc4c.png)

如图 ArrayBlockingQueue 内部有个数组 items 用来存放队列元素，putindex 变量标示入队元素下标，takeIndex 是出队下标，count 统计队列元素个数，从定义可知并没有使用 volatile 修饰，这是因为访问这些变量使用都是在锁块内，而加锁已经保证了锁块内变量的内存可见性了。

另外有个独占锁 lock 用来保证出入队操作原子性，这保证了同时只有一个线程可以进行入队出队操作，另外 notEmpty，notFull 条件变量用来进行出入队的同步。

另外由于 ArrayBlockingQueue 是有界队列，所以构造函数必须传入队列大小参数，构造函数代码如下：

```
  public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
  }

    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```

可知默认情况下使用的是 ReentrantLock 提供的非非公平独占锁进行出入队操作的加锁。

首先一个图概况该队列，读者可以读完本节后在回头体会下：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/b518219c9588f60d8958e1ecfa8f3bee.png)

#### 4.2 ArrayBlockingQueue 原理介绍

本节主要讲解下面几个主要函数的原理。

##### 4.2.1 offer 操作

向队列尾部插入一个元素，如果队列有空闲容量则插入成功后返回 true，如果队列已满则丢弃当前元素然后返回 false，如果 e 元素为 null 则抛出 NullPointerException 异常，另外该方法是不阻塞的。

```
    public boolean offer(E e) {
        //（1）e为null，则抛出NullPointerException异常
        checkNotNull(e);
        //（2）获取独占锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //（3）如果队列满则返回false
            if (count == items.length)
                return false;
            else {
                //（4）否者插入元素
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }
```

- 代码（2）获取独占锁，当前线程获取该锁后，其它入队和出队操作的线程都会被阻塞挂起后放入 lock 锁的 AQS 阻塞队列。
- 代码（3）如果队列满则直接返回 false，否者调用 enqueue 方法后返回 true，enqueue 的代码如下：

```
    private void enqueue(E x) {
        //（6）元素入队
        final Object[] items = this.items;
        items[putIndex] = x;
        //（7）计算下一个元素应该存放的下标
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        //(8)
        notEmpty.signal();
    }
```

如上代码首先把当前元素放入 items 数组，然后计算下一个元素应该存放的下标，然后递增元素个数计数器，最后激活 notEmpty 的条件队列中因为调用 poll 或者 take 操作而被阻塞的的一个线程。这里由于在操作共享变量比如 count 前加了锁，所以不存在内存不可见问题，加过锁后获取的共享变量都是从主内存获取的，而不是在 CPU 缓存或者寄存器里面的值。

- 代码（5）释放锁，释放锁后会把修改的共享变量值比如 Count 的值刷新回主内存中，这样其它线程通过加锁在次读取这些共享变量后就可以看到最新的值。

##### 4.2.2 put 操作

向队列尾部插入一个元素，如果队列有空闲则插入后直接返回 true，如果队列已满则阻塞当前线程直到队列有空闲插入成功后返回 true，如果在阻塞的时候被其它线程设置了中断标志，则被阻塞线程会抛出 InterruptedException 异常而返回，另外如果 e 元素为 null 则抛出 NullPointerException 异常。

```
public void put(E e) throws InterruptedException {
    //(1)
    checkNotNull(e);
    final ReentrantLock lock = this.lock;

    //(2)获取锁（可被中断）
    lock.lockInterruptibly();
    try {

        //(3)如果队列满，则把当前线程放入notFull管理的条件队列
        while (count == items.length)
            notFull.await();

        //(4)插入元素
        enqueue(e);
    } finally {
        //(5)
        lock.unlock();
    }
}
```

- 代码（2）在获取锁的过程中当前线程被其它线程中断了，则当前线程会抛出 InterruptedException 异常而退出。
- 代码（3）判断如果当前队列满了，则把当前线程阻塞挂起后放入到 notFull 的条件队列，注意这里也是使用了 while 而不是 if。
- 代码（4）如果队列不满则插入当前元素，此处不再累述。

##### 4.2.3 poll 操作

从队列头部获取并移除一个元素，如果队列为空则返回 null，该方法是不阻塞的。

```
public E poll() {
    //(1)获取锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //（2）当前队列为空则返回null,否者调用dequeue（）获取
        return (count == 0) ? null : dequeue();
    } finally {
        //(3)释放锁
        lock.unlock();
    }
}
```

- 代码（1）获取独占锁
- 代码（2）如果队列为空则返回 null，否者调用 dequeue() 方法，dequeue 代码如下：

```
private E dequeue() {
    final Object[] items = this.items;

    //（4）获取元素值
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    //（5）数组中值值为null;
    items[takeIndex] = null;

    //（6）队头指针计算，队列元素个数减一
   if (++takeIndex == items.length)
            takeIndex = 0;
    count--;

    //（7）发送信号激活notFull条件队列里面的一个线程
    notFull.signal();
    return x;
}
```

可知首先获取当前队头元素保存到局部变量，然后重置队头元素为 null，并重新设置队头下标，元素计数器递减，最后发送信号激活 notFull 的条件队列里面一个因为调用 put 或者 offer 而被阻塞的线程。

##### 4.2.4 take 操作

获取当前队列头部元素并从队列里面移除，如果队列为空则阻塞调用线程。如果队列为空则阻塞当前线程直到队列不为空然后返回元素，如果在阻塞的时候被其它线程设置了中断标志，则被阻塞线程会抛出 InterruptedException 异常而返回。

```
public E take() throws InterruptedException {
    //(1)获取锁
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {

        //（2）队列为空，则等待，直到队列有元素
        while (count == 0)
            notEmpty.await();
        //（3）获取队头元素
        return dequeue();
    } finally {
        //(4) 释放锁
        lock.unlock();
    }
}
```

take 操作的代码也比较简单与 poll 相比只是步骤（2）如果队列为空则把当前线程挂起后放入到 notEmpty 的条件队列，等其它线程调用 notEmpty.signal() 方法后在返回，需要注意的是这里也是使用 while 循环进行检测并等待而不是使用 if。

##### 4.2.5 peek 操作

获取队列头部元素但是不从队列里面移除，如果队列为空则返回 null，该方法是不阻塞的。

```
public E peek() {
    //(1)获取锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //（2）
        return itemAt(takeIndex);
    } finally {
       //(3)
        lock.unlock();
    }
}

 @SuppressWarnings("unchecked")
final E itemAt(int i) {
        return (E) items[i];
}
```

peek 的实现更简单，首先获取独占锁，然后从数组 items 中获取当前队头下标的值并返回，在返回前释放了获取的锁。

##### 4.2.6 size 操作

获取当前队列元素个数。

```
public int size() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return count;
    } finally {
        lock.unlock();
    }
}
```

size 操作是简单的，获取锁后直接返回 count，并在返回前释放锁。也许你会疑问这里有没有修改 Count 的值，只是简单的获取下，为何要加锁那？其实如果 count 声明为 volatile 这里就不需要加锁了，因为 volatile 类型变量保证了内存的可见性，而 ArrayBlockingQueue 的设计中 count 并没有声明为 volatile，是因为 count 的操作都是在获取锁后进行的，而获取锁的语义之一是获取锁后访问的变量都是从主内存获取的，这保证了变量的内存可见性。

注：ArrayBlockingQueue 通过使用全局独占锁实现同时只能有一个线程进行入队或者出队操作，这个锁的粒度比较大，有点类似在方法上添加 synchronized 的意味。ArrayBlockingQueue 的 size 操作的结果是精确的，因为计算前加了全局锁。

#### 五、PriorityBlockingQueue 原理探究

PriorityBlockingQueue 是带优先级的无界阻塞队列，每次出队都返回优先级最高或者最低的元素，内部是平衡二叉树堆的实现。

#### 5.1 PriorityBlockingQueue 类图结构

下面首先通过类图来从全局了解下 PriorityBlockingQueue 的结构

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/cdd36610e54402ba69c3cab45e802ef4.png)

如图 PriorityBlockingQueue 内部有个数组 queue 用来存放队列元素，size 用来存放队列元素个数，allocationSpinLock 是个自旋锁，用 CAS 操作来保证同时只有一个线程可以扩容队列，状态为 0 或者 1，其中 0 表示当前没有在进行扩容，1 标示当前正在扩容。

如下构造函数，默认队列容量为 11，默认比较器为 null，也就是使用元素的 compareTo 方法进行比较来确定元素的优先级，这意味着队列元素必须实现了 Comparable 接口;

```
 private static final int DEFAULT_INITIAL_CAPACITY = 11;

 public PriorityBlockingQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    public PriorityBlockingQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    public PriorityBlockingQueue(int initialCapacity,
                                 Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        this.comparator = comparator;
        this.queue = new Object[initialCapacity];
    }
```

首先通过一个图来对该队列进行概况，读者读完本机后，可以回头在体会下：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/e80e62b7ea5f4f947d901525d07a60ce.png)

#### 5.2 原理介绍

##### 5.2.1 offer 操作

offer 操作作用是在队列插入一个元素，由于是无界队列，所以一直返回 true，如下是 offer 函数的代码：

```
public boolean offer(E e) {

    if (e == null)
        throw new NullPointerException();

    //获取独占锁
    final ReentrantLock lock = this.lock;
    lock.lock();

    int n, cap;
    Object[] array;

    //如果当前元素个数>=队列容量，则扩容(1)
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);

    try {
        Comparator<? super E> cmp = comparator;

        //默认比较器为null (2)
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            //自定义比较器 (3)
            siftUpUsingComparator(n, e, array, cmp);

        //队列元素增加1，并且激活notEmpty的条件队列里面的一个阻塞线程（9）
        size = n + 1;
        notEmpty.signal();//激活调用take（）方法被阻塞的线程
    } finally {
        //释放独占锁
        lock.unlock();
    }
    return true;
}
```

如上代码，主流程比较简单，下面主要看看如何进行扩容的和内部如何建堆的，首先看下扩容逻辑：

```
private void tryGrow(Object[] array, int oldCap) {
    lock.unlock(); //释放获取的锁
    Object[] newArray = null;

    //cas成功则扩容(4)
    if (allocationSpinLock == 0 &&
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                 0, 1)) {
        try {
            //oldGap<64则扩容新增oldcap+2,否者扩容50%，并且最大为MAX_ARRAY_SIZE
            int newCap = oldCap + ((oldCap < 64) ?
                                   (oldCap + 2) : // grow faster if small
                                   (oldCap >> 1));
            if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            allocationSpinLock = 0;
        }
    }

    //第一个线程cas成功后，第二个线程会进入这个地方，然后第二个线程让出cpu，尽量让第一个线程执行下面点获取锁，但是这得不到肯定的保证。(5)
    if (newArray == null) // back off if another thread is allocating
        Thread.yield();
    lock.lock();//(6)
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```

tryGrow 目的是扩容，这里要思考下为啥在扩容前要先释放锁，然后使用 cas 控制只有一个线程可以扩容成功。其实这里不先释放锁，也是可行的，也就是在整个扩容期间一直持有锁，但是扩容是需要花时间的，如果扩容时候还占用锁那么其它线程在这个时候是不能进行出队和入队操作的，这大大降低了并发性。所以为了提高性能，使用 CAS 控制只有一个线程可以进行扩容，并且在扩容前释放了锁，让其它线程可以进行入队出队操作。

spinlock 锁使用 CAS 控制只有一个线程可以进行扩容，CAS 失败的线程会调用 Thread.yield() 让出 cpu，目的意在让扩容线程扩容后优先调用 lock.lock 重新获取锁，但是这得不到一定的保证。有可能 yield 的线程在扩容线程扩容完成前已经退出，并执行代码（6）获取到了锁，这时候获取到的锁的线程发现 newArray 为 null 就会执行代码（1）。如果当前数组扩容还没完毕，当前线程会再次调用 tryGrow 方法，然后释放锁，这又给扩容线程获取锁提供了机会，如果这时候扩容线程还没扩容完毕，则当前线程释放锁后有调用 yield 方法出让 CPU。可知当扩容线程进行扩容期间，其他线程是原地自旋通过代码（1）检查当前扩容是否完毕，等扩容完毕后才退出代码（1）的循环。

当扩容线程扩容完毕后会重置自旋锁变量 allocationSpinLock 为 0，这里并没有使用 UNSAFE 方法的 CAS 进行设置是因为同时只可能有一个线程获取了该锁，并且 allocationSpinLock 被修饰为了 volatile。

当扩容线程扩容完毕后会执行代码 (6) 获取锁，获取锁后复制当前 queue 里面的元素到新数组。

然后看下具体建堆算法：

```
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    Comparable<? super T> key = (Comparable<? super T>) x;

    //队列元素个数>0则判断插入位置，否者直接入队(7)
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = array[parent];
        if (key.compareTo((T) e) >= 0)
            break;
        array[k] = e;
        k = parent;
    }
    array[k] = key;(8)
}
```

下面用图来解释上面算法过程，假设队列初始化容量为 2, 创建的优先级队列的泛型参数为 Integer。

- 首先调用队列的 offer(2) 方法，希望插入元素 2 到队列，插入前队列状态如下图：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/7f1c83e7b09b30b6e4763f8318680a73.png)

首先执行代码（1)，从上图变量值可知判断值为 false，所以紧接着执行代码（2），由于 k=n=size=0 所以代码（7）判断结果为 false，所以会执行代码（8）直接把元素 2 入队，最后执行代码（9）设置 size 的值加 1，这时候队列的状态如下图：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/6ee88c2475207c6e0a0bde56e75eacc3.png)

- 然后调用队列的 offer(4) 时候，首先执行代码（1)，从上图变量值可知判断为 false，所以执行代码（2），由于 k=1, 所以进入 while 循环，由于 parent=0;e=2;key=4; 默认元素比较器是使用元素的 compareTo 方法，可知 key>e 所以执行 break 退出 siftUpComparable 中的循环; 然后把元素存到数组下标为 1 的地方，最后执行代码（9）设置 size 的值加 1，这时候队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/474180f1c0dbf5469269cd6970282c9f.png)

- 然后调用队列的 offer(6) 时候，首先执行代码（1)，从上图变量值知道这时候判断值为 true, 所以调用 tryGrow 进行数组扩容, 由于 2<64 所以 newCap=2 + (2+2)=6; 然后创建新数组并拷贝，然后调用 siftUpComparable 方法，由于 k=2>0 进入 while 循环，由于 parent=0;e=2;key=6;key>e 所以 break 后退出 while 循环; 并把元素 6 放入数组下标为 2 的地方，最后设置 size 的值加 1，现在队列状态：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/ac4c5d4f068d4dbdc1b68b881f3a2896.png)

- 然后调用队列的 offer(1) 时候，首先执行代码（1)，从上图变量值知道这次判断值为 false，所以执行代码（2），由于`k=3`, 所以进入 while 循环，由于`parent=0;e=4;key=1; key<e`，所以把元素 4 复制到数组下标为 3 的地方，然后 k=0 退出 while 循环；然后把元素 1 存放到下标为 0 地方，现在状态：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/a639f1398c44205ca8319af4c14471b3.png)

这时候二叉树堆的树形图如下：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/c5f925b6f025664359316e47863951a7.png)

可知堆的根元素是 1，也就是这是一个最小堆，那么当调用这个优先级队列的 poll 方法时候，会一次返回堆里面值最小的元素。

##### 5.2.2 poll 操作

poll 操作作用是获取队列内部堆树的根节点元素，如果队列为空，则返回 null。poll 函数代码如下：

```
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();//获取独占锁
    try {
        return dequeue();
    } finally {
        lock.unlock();//释放独占锁
    }
}
```

如上代码可知在进行出队操作过程中要先加锁，这意味着，当当前线程进行出队操作时候，其它线程不能再进行入队和出队操作，但是从前面介绍 offer 函数时候知道这时候可以有其它线程进行扩容，下面主要看下具体执行出队操作的 dequeue 方法的代码：

```
private E dequeue() {

    //队列为空，则返回null
    int n = size - 1;
    if (n < 0)
        return null;
    else {

        //获取队头元素(1)
        Object[] array = queue;
        E result = (E) array[0];

        //获取队尾元素，并值null(2)
        E x = (E) array[n];
        array[n] = null;

        Comparator<? super E> cmp = comparator;
        if (cmp == null)//(3)
            siftDownComparable(0, x, array, n);
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;//（4）
        return result;
    }
}
```

如上代码，如果队列为空则直接返回 null，否者执行代码（1）获取数组第一个元素作为返回值存放到变量 Result，这里需要注意下数组里面第一个元素是优先级最小或者最大的元素，出队操作就是返回这个元素。 然后代码（2）获取队列尾部元素存放到变量 x, 并且置空尾部节点，然后执行代码（3）插入变量 x 到数组下标为 0 的位置后，重新调成堆为最大或者最小堆，然后返回。这里重要的是看如何去掉堆的根节点后，使用剩下的节点重新调整为一个最大或者最小堆，下面我们看下 siftDownComparable 的代码实现：

```
    private static <T> void siftDownComparable(int k, T x, Object[] array,
                                               int n) {
        if (n > 0) {
            Comparable<? super T> key = (Comparable<? super T>)x;
            int half = n >>> 1;           // loop while a non-leaf
            while (k < half) {
                int child = (k << 1) + 1; // assume left child is least
                Object c = array[child];（5）
                int right = child + 1;（6)
                if (right < n &&
                    ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)(7)
                    c = array[child = right];
                if (key.compareTo((T) c) <= 0)(8)
                    break;
                array[k] = c;
                k = child;
            }
            array[k] = key;(9)
        }
    }
```

同理下面我们结合图来模拟上面调整堆的算法过程，接着上节队列的状态继续讲解，上节队列元素序列为 1，2，6，4：

- 第一次调用队列的 poll() 方法时候，首先执行代码（1）（2），这时候变量 size =4;n=3;result=1；x=4; 这时候队列状态

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/a4cd6455c16fca078ebc438570a983d4.png)

然后执行代码（3）调整堆后队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/ea8adb5aed5d58c2b48656ac41cd5fcf.png)

- 第二次调用队列的 poll() 方法时候，首先执行代码（1）（2），这时候变量 size =3;n=2;result=2；x=6; 这时候队列状态：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/23dc42e53ea5701ddb24217d959613d0.png)

然后执行代码（3）调整堆后队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/767ab099617a490e7d4c9a5abeccc1eb.png)

- 第三次调用队列的 poll() 方法时候，首先执行代码（1）（2），这时候变量 size =2;n=1;result=4；x=6; 这时候队列状态：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/5ff0ef3d165be5105b956d504fd4d320.png)

然后执行代码（3）调整堆后队列状态为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/5ff0ef3d165be5105b956d504fd4d320.png)

- 第四次直接返回元素 6.

下面重点说说 siftDownComparable 这个调整堆的算法： 首先说下堆调整的思路，由于队列数组第 0 个元素为树根，出队时候要被移除，这时候数组就不在是最小堆了，所以需要调整堆，具体是要从被移除的树根的左右子树中找一个最小的值来当树根，左右子树又会看自己作为根节点的树的左右子树里面那个是最小值，这是一个递归，直到树叶节点结束递归，如果还不明白，没关系，下面结合图来说明下，假如当前队列内容如下：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/549c0f29679e9373c88f3550b549c186.png)

其对应的二叉堆树为：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/a147e6741f7a606767fea341b4eeada2.png)

这时候如果调用了 poll(); 那么 result=2;x=11；队列末尾的元素设置为 null 后，剩下的元素调整堆的步骤如下图：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/8782e5063ea982dfc0fe267d075adfb5.png)

如上图（1）树根的 leftChildVal = 4;rightChildVal = 6; 4<6; 所以 c=4; 然后 11>4 也就是 key>c；所以使用元素 4 覆盖树根节点的值，现在堆对应的树如图（2）。

然后树根的左子树树根的左右孩子节点中 leftChildVal = 8;rightChildVal = 10; 8<10; 所以 c=8; 然后发现 11>8 也就是 key>c；所以元素 8 作为树根左子树的根节点，现在树的形状如图（3）, 这时候判断 k<half 为 false 就会退出循环，然后把 x=11 设置到数组下标为 3 的地方，这时候堆树如图（4），至此调整堆完毕，siftDownComparable 返回 result=2，poll 方法也返回了。

##### 5.2.3 put 操作

put 操作内部调用的 offer, 由于是无界队列，所以不需要阻塞

```
public void put(E e) {
    offer(e); // never need to block
}
```

##### 5.2.4 take 操作

take 操作作用是获取队列内部堆树的根节点元素，如果队列为空则阻塞，如下代码：

```
public E take() throws InterruptedException {
    //获取锁，可被中断
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {

        //如果队列为空，则阻塞，把当前线程放入notEmpty的条件队列
        while ( (result = dequeue()) == null)
            notEmpty.await();//阻塞当前线程
    } finally {
        lock.unlock();//释放锁
    }
    return result;
}
```

如上代码，首先通过 lock.lockInterruptibly() 获取独占锁，这个方式获取的锁是对中断进行响应的。然后调用 dequeue 方法返回堆树根节点元素，如果队列为空，则返回 false，然后当前线程调用 notEmpty.await() 阻塞挂起当前线程，直到有线程调用了 offer（）方法（offer 方法内在添加元素成功后调用了 notEmpty.signal 方法会激活一个阻塞在 notEmpty 的条件队列里面的一个线程）。另外这里使用 while 而不是 if 是为了避免虚假唤醒。

##### 5.2.5 size 操作

获取队列元个数，如下代码，在返回 size 前加了锁，保证在调用 size() 方法时候不会有其它线程进行入队和出队操作，另外由于 size 变量没有被修饰为 volatie，这里加锁也保证了多线程下 size 变量的内存可见性。

```
public int size() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return size;
    } finally {
        lock.unlock();
    }
}
```

注：PriorityBlockingQueue 队列内部使用二叉树堆维护元素优先级，内部使用数组作为元素存储的数据结构，这个数组是可扩容的，当当前元素个数 >= 最大容量时候会通过算法扩容，出队时候始终保证出队的元素是堆树的根节点，而不是在队列里面停留时间最长的元素，默认元素优先级比较规则是使用元素的 compareTo 方法来做，用户可以自定义优先级的比较规则。

### 六、浅谈各种队列的对比

上面介绍的各种队列中只有 ConcurrentLinkedQueue 是使用 UNSAFE 类提供的 CAS 非阻塞算法实现的，其他几个队列内部都是使用锁来保证线程安全的。使用 CAS 算法的效率较好，那么是不是所有场景都用 ConcurrentLinkedQueue 那？

其实不然，因为 ConcurrentLinkedQueue 还是无界队列，无界队列使用不当可能造成 OOM。所以当使用 ConcurrentLinkedQueue 的时候在添加元素前应该先判断当前队列元素个数是否已经达到了设定的阈值，如果达到就做一定的处理措施，比如直接丢弃等。这里需要注意判断当前队列元素个数与阈值这个操作不是原子性的，最终会导致队列元素个数比设置的阈值大。

ConcurrentLinkedQueue 在 Tomcat 的的 NioEndPoint 中得到了应用，通过使用 ConcurrentLinkedQueue 将同步转换为异步，可以让 tomcat 同时接受更多请求，模型如下图：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/0ea7af948bf4864ca742422383d1a0d6.png)

tomcat 的 NioEndPoint 模式中 acceptor 线程负责接受用户请求，接受后把请求放入到 poll 线程对应的队列，poll 线程从队列里面获取任务后委托给 worker 线程具体处理。

LinkedBlockingQueue 和 ArrayBlockingQueue 都是有界阻塞队列，不同在于一个底层数据结构是链表，一个是数组；另外前者入队出队使用单独的锁，而后者出入队使用同一个锁，所以前者的并发度比后者高。另外创建前者时候可以不指定队列大小，默认队列元素个数为 Integer.MAX_VALUE，而后者必须要指定数组大小。所以使用 LinkedBlockingQueue 时候要记得指定队列大小。

比如比较有名的 LogBack 日志系统的异步日志打印实现中就是用了 ArrayBlockingQueue 作为缓冲队列，如下图，业务检查调用异步 log 进行写入日志时候，实际是把日志放入了 ArrayBlockingQueue 队列就返回了，而具体真正写入日志到磁盘是一个日志线程从队列里面获取任务来做的，这其实是一个多生产单消费模型：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/a3cedfae205133fc27033855e035fef2.png)

PriorityBlockingQueue 是无界阻塞队列，是一个队列元素有优先级的队列，前面的队列模式都是 FIFO 先进先出，而 PriorityBlockingQueue 而是优先级最高的元素先出队，而不管谁先进入队列的，所以 PriorityBlockingQueue 经常会用在一些任务具有优先级的场景。还比如上面说的 logback 异步日志模型，如果把日志等级分了优先级，比如 error>warn>info，那么上述模型中队列就可以使用 PriorityBlockingQueue，日志线程会先从队列里面首先获取 error 级别的日志，但是需要注意的是如果业务线程一直向队列里面写入 error 级别日志，那么可能先写入到队列的 warn 和 info 级别的日志将很久甚至永远没机会写入到磁盘。还有一点要注意 PriorityBlockingQueue 是无界限队列，要注意判断队列元素个数不要超过设置的阈值。