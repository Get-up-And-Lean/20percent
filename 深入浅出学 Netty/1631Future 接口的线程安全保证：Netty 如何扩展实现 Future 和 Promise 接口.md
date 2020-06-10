# 16/31Future 接口的线程安全保证：Netty 如何扩展实现 Future 和 Promise 接口

### 前言

Netty 的大部分用户接口都是异步化的，返回的都是一个 ChannelFuture 对象。该接口是 Netty 对 JDK 中的 Future 接口扩展而来。和开发者相关比较大的变化是允许添加一个 GenericFutureListener 监听器，以便在异步任务完成时触发回调任务。

接口的定义比较简单，不过如何保证并发的安全性则是一个值得思考的问题。假定在任务完成的瞬间，addListener 方法被调用，回调方法是否一定被触发？下面带着问题来看源码

### 类层次

首先让我们来看下类层次图。

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191021091127.png)

虽然大部分用户接口代码返回都是 ChannelFuture，但是实际上真正生效的是接口 ChannelPromise。从 Promise 接口继承的能力，使得该接口允许设置成功或者失败标识。下面在源码走读中具体来分析。

### 源码走读

#### **Future**

Netty 自定义的 Future 接口，继承自 JDK 的 Future 接口，不过实际当中使用到的都是自己定义的方法。方法大致上分为两类：

- 获取结果和进行结果等待的，诸如 sync，await 等
- 添加任务回调方法的，诸如 addListener、addListeners

#### **AbstractFuture**

该抽象实现主要是提供了 2 个方法的实现，分别是 `AbstractFuture#get()` 和 `AbstractFuture#get(long, java.util.concurrent.TimeUnit)`。以 get 的源码为例进行分析：

```java
public V get() throws InterruptedException, ExecutionException {await();
        Throwable cause = cause();
        if (cause == null) {return getNow();
        }
        if (cause instanceof CancellationException) {throw (CancellationException) cause;
        }
        throw new ExecutionException(cause);
    }
```

可以看到，get 方法只需要组合其他方法的逻辑即可得到结果，其中最为重要的就是 `await` 方法。这个方法则由子类去实现。

这也是许多工具类方法的经典设计技巧。通过对部分原子方法的组合，来获得新的功能。而这部分新的，公共的功能，则可以在抽象类中实现，而具体的原子功能，则推迟到子类中完成。这样不同的子类有不同的实现策略，就可以组合出不同的上层功能特性。

#### **Promise**

Future 接口对于结果内容本身是不可写的。为此，而 Promise 接口则提供了对于结果对象的写方法，提供了诸如 setSuccess、setFailure、trySuccess 等方法。从方法名就可以看出，这个接口关注的点就是在于对结果的写入。

#### **DefaultPromise**

层次较高的一个默认实现，该实现完整提供了 Future 和 Promise 的接口功能。首先来看下该类的几个重要属性。

```java
private volatile Object result;// 异步任务结果
private final EventExecutor executor;// 产生该 Promise 对象的 EventExecutor。
private Object listeners;// 异步任务监听器对象或者对象列表的存储属性
private short waiters;// 该异步任务上的等待线程数
private boolean notifyingListeners;//EventExecutor 是否正在唤醒任务监听器
```

这些属性的具体作用和生效机制需要结合后文的方法分析来分析。

**sync 方法**

sync 方法是很常用的了，用于等待任务的完成。其内部实现，实际上是委托了另外一个等待方法。代码如下：

```java
public Promise<V> sync() throws InterruptedException {await();
        rethrowIfFailed();
        return this;
    }
```

可以看到，具体的等待，是委托给了 await 方法。

**await 方法**

这个方法，顾名思义，就是执行任务等待，或者说让线程等待任务直到完成。方法体的实现遵循两个大步骤：

- 死锁可能检查
- 使用 synchronized 关键字执行线程等待

下面来从代码的角度看下：

```java
public Promise<V> await() throws InterruptedException {if (isDone()) {return this;}
        if (Thread.interrupted()) {throw new InterruptedException(toString());
        }
        checkDeadLock();
        synchronized (this) {while (!isDone()) {incWaiters();
                try {wait();
                } finally {decWaiters();
                }
            }
        }
        return this;
    }
```

死锁检查 checkDeadLock 的实现如下：

```java
protected void checkDeadLock() {EventExecutor e = executor();
        if (e != null && e.inEventLoop()) {throw new BlockingOperationException(toString());
        }
    }
```

思想也很简单。因为 Promise 实例是在 EventExecutor 被生成出来，而具体的任务也是在这个线程上执行。前文有说过，在 Netty 的线程模型中，一个 EventExecutor 是死循环执行自己队列中的任务的。因此此时在这个线程上执行等待，那么其对应的任务永远没有机会获得线程执行了。

死锁检查通过后，就开始使用 synchronized 关键字进行线程等待。等待区域的代码逻辑很简单。

```java
synchronized (this) {while (!isDone()) {incWaiters();
                try {wait();
                } finally {decWaiters();
                }
            }
        }
```

使用 synchronized 对当前对象加锁，而后在一个 while 循环中，检查任务是否完成，如果没有完成，则依靠 Object 的 wait 方法让线程进入等待。在等待的前后会分别对等待计数器进行加减。对等待计数有两方面的作用，一方面避免过多的线程在这个对象上执行等待，否则后期唤醒的效率就比较低。另外一个作用，后面任务完成后、执行唤醒时，通过检查等待计数，如果计数为 0，就不需要执行 notifyAll。

**addListener 方法**

还有一个高频率的用户方法就是对 Future，添加任务监听器。代码如下：

```java
public Promise<V> addListener(GenericFutureListener<? extends Future<? super V>> listener) {checkNotNull(listener, "listener");
        // 使用本对象为锁，确保对 listeners 属性操作的并发正确性
        synchronized (this) {addListener0(listener);
        }
        // 如果结果对象已经被设置，则直接触发监听
        if (isDone()) {notifyListeners();
        }
        return this;
    }
```

代码的逻辑很清晰，主要是两步：

- 使用 synchronized 锁定当前对象后，将监听器添加到本对象里的监听器列表中（如果当前没有监听器，则只是将入参监听器设置为对象的监听器）
- 添加监听器成功后，如果当前任务已经完成，则触发当前在该对象上的监听器。

**addListener0**

先来看下实际添加监听器的方法 addListener0，具体代码如下：

```java
private void addListener0(GenericFutureListener<? extends Future<? super V>> listener) {if (listeners == null) {listeners = listener;} else if (listeners instanceof DefaultFutureListeners) {((DefaultFutureListeners) listeners).add(listener);
        } else {listeners = new DefaultFutureListeners((GenericFutureListener<?>) listeners, listener);}
    }
```

考虑到功能覆盖的完整性，监听器必然是一个列表的模式。但是实际使用中，大多数时候用户都只是设置一个监听器对象，设置多个的情况比较少。因此 Netty 按照对象持有的监听器数量的不同采用了不同的存储策略。

如果第一个监听器被添加到异步任务中时，此时直接将监听器对象赋值给异步任务对象的 listeners 属性，也就是代码 `if (listeners == null) {listeners = listener;}` 的作用。

第二次添加时，则需要将 listeners 属性转变为一个列表属性，也就是 DefaultFutureListeners。此时需要将之前的监听器对象和本次监听器对象合并，形成一个 DefaultFutureListeners 对象，也就是代码

```
listeners = new DefaultFutureListeners((GenericFutureListener<?>) listeners, listener);
```

后续再次添加的时候，只需要将监听器对象添加到列表中即可。

**notifyListeners**

这个方法不仅仅在这个时候会被触发，更多的时候是在任务线程设置结果的时候触发。放在下一个章节中进行分析更清晰。

**setSuccess**

Promise 中有 4 个方法都是用于对结果对象的设置。

首先来看 setSuccess 的实现，其余的 setFailure 等逻辑是类似的。代码如下：

```java
public Promise<V> setSuccess(V result) {if (setSuccess0(result)) {return this;}
        throw new IllegalStateException("complete already:" + this);
    }
private boolean setSuccess0(V result) {return setValue0(result == null ? SUCCESS : result);
    }
// 设置成功的核心方法从这里开始看
private boolean setValue0(Object objResult) {
        // 通过 CAS 的方式进行防御性编程，确保设置 value 的唯一性。
        if (RESULT_UPDATER.compareAndSet(this, null, objResult) ||
            RESULT_UPDATER.compareAndSet(this, UNCANCELLABLE, objResult)) {
            // 如果设置成功，则可以进行通知唤醒工作
            if (checkNotifyWaiters()) {notifyListeners();
            }
            return true;
        }
        return false;
    }
```

首先是一个 CAS 抢占式设置结果对象，CAS 大家都很熟悉了，用来避免并发，防止多次唤醒的防御性编程手段。CAS 成功的线程就开始执行后续的流程。先来看下 checkNotifyWaiters 的具体内容，如下：

```java
private synchronized boolean checkNotifyWaiters() {if (waiters > 0) {notifyAll();
        }
        return listeners != null;
    }
```

方法用 synchronized 修饰，首先获得对象的锁，然后检查在锁上等待的线程数，这个检查主要是避免无畏的 notifyAll 调用，节省一些开销。

方法的返回结果，标识了该异步任务对象上是否存在着监听器，如果存在的话，则执行监听器唤醒，也就是 notifyListeners 方法。继续来看。

```java
private void notifyListeners() {EventExecutor executor = executor();
        if (executor.inEventLoop()) {final InternalThreadLocalMap threadLocals = InternalThreadLocalMap.get();
            final int stackDepth = threadLocals.futureListenerStackDepth();
            if (stackDepth < MAX_LISTENER_STACK_DEPTH) {threadLocals.setFutureListenerStackDepth(stackDepth + 1);
                try {notifyListenersNow();
                } finally {threadLocals.setFutureListenerStackDepth(stackDepth);
                }
                return;
            }
        }
        safeExecute(executor, new Runnable() {
            @Override
            public void run() {notifyListenersNow();
            }
        });}
```

从代码可以看出，notifyListenersNow 方法才是正主，剩下的都是都是为了防御性的保证。一开始的 if 方法是为了保证监听器的触发时是在对应的 EventExecutor 线程中被执行。紧接着使用 stackDepth 用来标记当前的唤醒调用深度，这个调用深度的含义是在监听器触发中再次触发其他的监听器。Netty 要避免过深的监听器触发。

如果当前线程不是 EventExecutor 线程，或者当前的调用深度超出了限定值，则将监听器的触发包装为一个 runnable 方法，投递到 EventExecutor 线程的队列中等待执行。

接下来来看下真正的触发监听器的方法，如下：

```java
 private void notifyListenersNow() {
        Object listeners;
        synchronized (this) { // 阶段一
            if (notifyingListeners || this.listeners == null) {return;}
            notifyingListeners = true;
            listeners = this.listeners;
            this.listeners = null;
        }
        for (;;) {if (listeners instanceof DefaultFutureListeners) { // 阶段二
                notifyListeners0((DefaultFutureListeners) listeners);} else {notifyListener0(this, (GenericFutureListener<?>) listeners);}
            synchronized (this) { // 阶段三
                if (this.listeners == null) {
                    notifyingListeners = false;
                    return;
                }
                listeners = this.listeners;
                this.listeners = null;
            }
        }
    }
```

整个代码明显的分为三个层次。

第一阶段，通过 synchronized 获取当前对象的锁。之后判断监听器触发工作是否已经有线程在执行了，也就是判断属性 notifyingListeners 是否为真。如果存在监听器并且没有其他线程执行监听器触发，则本线程获得监听器的触发执行权。获得执行权会做 2 个事情：

- 将 notifyingListeners 属性设置为 true，标识已经有线程正在处理触发监听器的任务。
- 将本对象的 listeners 属性的值赋予方法内临时变量，并且将该属性清除。

第一步确保了对监听器触发任务的唯一执行权，避免了其他线程的争夺。第二步则是获取了稳定的待触发监听器列表，避免在触发期间，这个列表再有变化。

第二阶段，就是触发监听器方法。

第三阶段，再次获取对象锁。并且检查 listeners 属性是否被更新，如果被更新，则意味着在第二阶段时，有新的监听器被添加到这个对象中。此时再次获取监听器列表，并且将属性 listeners 设置为空，重新执行阶段二。

阶段二和三会反复执行，直到在阶段三时，没有新增的监听器。此时就会将 notifyingListeners 属性设置为 false。给予其他线程执行的可能性。

**综述**

整个 DefaultPromise 的设计并不是很复杂，但是其针对线程安全做了几个重要的设计点，分别是：

- 任何针对 listeners 属性的操作，都需要先对对象加锁。确保读取和写入不会有数据不一致问题。
- 对结果对象的设置采用 CAS 方式，避免并发竞争。
- 监听器的触发只能在 EventExecutor 线程中执行，避免多线程竞争。

这里面涉及的线程安全的主要问题是，如何确保添加的监听器一定会被触发。分两点来看：

1. 如果添加时，有其他线程正在执行触发监听器，假设为线程 A，也就是 notifyingListeners 为真。那么本线程添加结束后，线程 A 必然可以发现新添加的监听器，必然会触发。
2. 如果在添加时，其他线程已经执行完毕监听器触发的任务。那么本线程也可以直接执行 notifyListeners 方法来触发监听器。

### 总结与思考

本文对 Netty 中的 ChannelFuture 接口进行了源码分析，从其实现上，可以学习到一些经典的并发编程技巧：

1. 通过 synchronized 关键字包围易变属性，通过局部变量置换出，以避免使用属性时被并发修正。
2. 通过 CAS 操作进行防御性编程，相比 synchronized 方法更能提升性能。

下篇文章我们将介绍 Netty 中的另外一个重要组件 Pipeline。