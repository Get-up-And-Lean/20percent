# 17/31从 pipeline 的源码看责任链模式设计

## 前言

前文中，多次有提到 pipeline 这个组件。该组件也是 Netty 中比较核心的一个设计。任何一个通道对象在初始化的时候都会初始化一个 pipeline 对象。pipeline 负责存储添加到通道上的 `ChannelHandler` 对象。其本身是一个责任链的设计，前文也多次提到了。那下面就来逐步对其进行代码分析。

## 类层次图

首先来看下类层次图

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20190823154919.png)

从 `ChannelPipeline` 继承的接口来看，管道同时对出站事件和入站事件进行传递。其也有唯一的实现类 `DefaultChannelPipeline`。

## 源码阅读

首先来一张 IO 事件的处理顺序图，就比较好看的出来 pipeline 的工作顺序了，如下

![img](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20190823153817.png)

这张图传递了很多有用的信息，包括有：

- 入站事件从是 socket 通道产生的数据，沿着管道向后传递。
- 出战事件从是用户侧产生的 IO 请求，沿着管道向前传递。
- 入站事件可以通过 `ChannelHandlerContext.fireXX` 方法向后传递。
- 出战事件可以通过 `ChannelHandlerContext.xxx` 方法向前传递。

### 添加监听器

下面来看下接口的实现类 `DefaultChannelPipeline`。关于这个类，最先用的方法就是类似 `addLast`、`addFirst` 之类的添加监听器的方法。从名字上看，就知道这些方法的作用都是相同的，只是添加监听器时放入的位置不同。下面以 addLast 来分析下，代码如下

```java
public final ChannelPipeline addLast(ChannelHandler handler) {return addLast(null, handler);
    }
public final ChannelPipeline addLast(String name, ChannelHandler handler) {return addLast(null, name, handler);
    }
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {checkMultiplicity(handler);
            newCtx = newContext(group, filterName(name, handler), handler);
            addLast0(newCtx);
            if (!registered) {newCtx.setAddPending();
                callHandlerCallbackLater(newCtx, true);
                return this;
            }
            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {callHandlerAddedInEventLoop(newCtx, executor);
                return this;
            }
        }
        callHandlerAdded0(newCtx);
        return this;
    }
```

方法进行了两次委托，可以看到，在添加 `ChannelHandler` 时，可以赋予这个 `Handler` 名字和其绑定的 `EventExecutor`。`name` 可以为 null，因为 Netty 会自动使用类名为其自动生成，`EventExecutorGroup` 可以为 null，因此此时 `Handler` 直接绑定于通道本身的 `EventExecutor` 线程。下面来逐步看方法 `DefaultChannelPipeline#addLast(EventExecutorGroup, java.lang.String, ChannelHandler)`。

##### 共享检查

首先是方法 `checkMultiplicity`，代码如下

```java
private static void checkMultiplicity(ChannelHandler handler) {if (handler instanceof ChannelHandlerAdapter) {ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
            if (!h.isSharable() && h.added) {
                throw new ChannelPipelineException(h.getClass().getName() +
                        "is not a @Sharable handler, so can't be added or removed multiple times.");}
            h.added = true;
        }
    }
```

该方法用于判断即将要添加的监听器是否已经在别处添加过以及是否是可共享的。前文介绍 Netty 的线程模型的时候说过，`ChannelHandler` 是运行在其绑定的线程上，也就是 `EventExecutor` 上。因为单线程运行的原因，`ChannelHandler` 不需要考虑线程安全的问题。但是如果一个 `ChannelHandler` 被添加到了不同的管道中，就可能在多个线程中被并发运行。此时用户实现 `ChannelHandler` 时就需要考虑并发安全问题了。

如果用户实现的 `ChannelHandler` 是并发安全的，则可以被安全的添加到多个管道中，否则在添加的时候 Netty 会抛出异常来提醒我们。标记一个 `ChannelHandler` 是并发安全是通过在其类上是否标记了 `io.netty.channel.ChannelHandler.Sharable` 注解来判断。

##### 初始化 ChannelHandlerContext

如果要添加的 `ChannelHandler` 没问题，就通过 `newContext` 方法为该 `ChannelHandler` 初始化一个 `ChannelHandlerContext` 对象。

###### 属性

我们常提到 `pipeline` 是责任链模式，但是其内部直接存储的并不是 `ChannelHandler` 实例，而是包裹一个 `ChannelHandler` 的 `ChannelHandlerContext` 对象。这个接口，Netty 内部使用的实现类是 `DefaultChannelHandlerContext`。首先来看下其几个重要属性

```java
volatile AbstractChannelHandlerContext next;// 指向前一个 ChannelHandlerContext 的指针
volatile AbstractChannelHandlerContext prev;// 指向后一个 ChannelHandlerContext 的指针
final String name; //ChannelHandler 的名字
volatile int handlerState //ChannelHandler 当前的状态，有：初始化，添加到管道中，添加到管道完成，移除出管道
final int executionMask; // 该 ChannelHandler 具体实现了哪些事件方法的掩码标识
final EventExecutor executor; // 该 ChannelHandler 绑定的执行器
```

从指针属性就可以看到 `Pipeline` 的责任链模式就是依靠 `DefaultChannelHandlerContext` 的两个指针形成的双向链表。这也是一种较为常用的模式，将需要给用户实现的部分开放给用户，而将数据结构等固化的内容以包装类的形式实现提供。

`name` 属性这个就不必说了，或者是用户传入的，如果用户不传入，则是 Netty 采用类名 +“#0”为其生成。需要明确的是，在一个管道中，名字是不能重复的。

`handlerState` 主要是四个状态的变化，解释下：

- ** 添加中 **：如果一个 `ChannelHandler` 被添加到管道上，但是还未触发 `handlerAdded` 方法时，处于该状态。由于添加动作可能遭遇通道绑定到具体的线程，因此在通道没有绑定到具体线程前，该方法都不会触发，此时就处于添加中的状态
- ** 添加完成 **：`ChannelHandler` 添加到管道上，并且 `handlerAdded` 方法被触发了就处于这个状态。
- ** 移除完成 **：`ChannelHandler` 被移除出管道，并且 `handlerRemoved` 方法触发完成后就处于这个状态。
- ** 初始化 **：`handlerAdded` 或 `handlerRemoved` 未被执行过，就处于这个状态。

`executionMask` 用于标识 `ChannelHandler` 具体实现了哪些事件方法。入站事件方法在接口 `ChannelInboundHandler` 中定义，出站事件方法在接口 `ChannelOutboundHandler` 中定义。Netty 中定义了 17 个整型数字，这些数字均是 1 左移不同的位数得到，因此可以方便的进行并和取反操作。将 1 左移不同的位数表示不同的状态，再通过并操作，一个整型变量最多可以表达 32 个不同的状态集合。

那 Netty 是如何判断一个 `ChannelHandler` 是否实现了对应的事件方法，很简单，取决于两个条件是否同时成立：

- 该 `ChannelHandler` 是否实现了 `ChannelInboundHandler` 或者 `ChannelOutboundHandler` 接口
- 实现类对应的接口方法上，没有标记 `io.netty.channel.ChannelHandlerMask.Skip` 注解

同时成立就会给 `executionMask`”并“上对应事件的数字标识，后续就可以通过 `executionMask` 判断出该 `ChannelHandler` 实现了具体的事件方法，也就是可以传递对应的事件给予其 `ChannelHandler`。

###### 主要方法

关于 `ChannelHandlerContext` 有三类方法值得关注，首先是 `AbstractChannelHandlerContext#executor`, 其内容为

```java
public EventExecutor executor() {if (executor == null) {return channel().eventLoop();} else {return executor;}
    }
```

可以看出，如果 `ChannelHandler` 没有绑定的 `EventExecutor`，则会直接使用通道绑定 `EventExecutor`。

然后是 fireXX 方法。这一类方法是用来向后传递入站事件的，其所有的实现都遵循一个共同的套路，以下以 `fireChannelRead` 来分析，源码如下

```java
public ChannelHandlerContext fireChannelRead(final Object msg) {invokeChannelRead(findContextInbound(MASK_CHANNEL_READ), msg);
        return this;
    }
```

既然是向后传递，第一是要找到会处理这个事件的 `ChannelHandler`，也就是方法 `findContextInbound` 的作用，其代码如下

```java
private AbstractChannelHandlerContext findContextInbound(int mask) {
        AbstractChannelHandlerContext ctx = this;
        do {ctx = ctx.next;} while ((ctx.executionMask & mask) == 0);
        return ctx;
    }
```

核心判断就是 `(ctx.executionMask & mask) == 0`。前文介绍过 `executionMask` ，上面包含了这个 `ChannelHandler` 实现的方法的信息，通过与操作，不为 0 的情况就意味着对应位置的标识处于开状态，也就是实现了这个方法。

能处理事件的 `ChannelHandler` 被选择出来后，就是传递事件了，也就是方法 `invokeChannelRead`, 代码如下

```java
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);// 这个调用用于对象调用轨迹追踪，和事件传递无关
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {next.invokeChannelRead(m);
        } else {executor.execute(new Runnable() {
                @Override
                public void run() {next.invokeChannelRead(m);
                }
            });}
    }
```

首先是取出下一个 `ChannelHandler` 绑定的 `EventExecutor`，通过 `executor.inEventLoop()` 判断当前 `ChannelHandler` 绑定的 `EventExecutor` 的线程是否与下一个节点的线程相同；相同就直接传递了，也就是调用 `next.invokeChannelRead(m)`；如果不同的话，则包装为 `runnable` 任务，投递到下一个节点的线程的任务队列中。

从这个处理方式再次印证了，一个 `ChannelHandler` 始终在自己绑定的线程中被执行，在不共享的情况下，其是单线程执行。这种设计，使得实现 `ChannelHandler` 的难度下降不少，不用考虑并发安全，是个很好的设计。

最后是出站事件的传递方法。这类方法都会返回一个 `Future` 对象，其内部实现也是统一套路，以下以 `write` 方法为例分析，代码如下

```java
public ChannelFuture write(Object msg) {return write(msg, newPromise());
    }
public ChannelFuture write(final Object msg, final ChannelPromise promise) {write(msg, false, promise);
        return promise;
    }
private void write(Object msg, boolean flush, ChannelPromise promise) {
        // 省略代码，主要是异常检查，以及确认 promise 是否被取消；如果取消了则不处理后续流程，直接返回
        final AbstractChannelHandlerContext next = findContextOutbound(flush ?
                (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
        final Object m = pipeline.touch(msg, next);// 记录轨迹，和事件无关。
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {// 模式一
            if (flush) {next.invokeWriteAndFlush(m, promise);
            } else {next.invokeWrite(m, promise);
            }
        } else {// 模式二
            final AbstractWriteTask task;
            if (flush) {task = WriteAndFlushTask.newInstance(next, m, promise);
            }  else {task = WriteTask.newInstance(next, m, promise);
            }
            if (!safeExecute(executor, task, promise, m)) {task.cancel();
            }
        }
    }
```

与入站相似，如果下一个节点的线程与当前节点的线程一致，则直接执行写出动作，也就模式一的代码内容。如果线程不一致，则将写出动作包装为一个任务，投递到下一个节点的任务队列中等待被调度，也即是模式二的代码内容。这些与入站事件的处理逻辑是相似的，就不赘述了。

##### 添加 CTX 到管道中

`ChannelHandlerContext` 初始化完毕，就可以添加到 pipeline 中了。添加的实现很简单，就是将前后指针指向的节点设置下即可，这个就不看代码了。

##### 触发 handlerAdded 方法

ctx 添加到管道后就应该触发对应 `ChannelHandler` 的 `handlerAdded` 方法了。但是此时有几种情况需要考虑：

- 当前通道尚未注册到一个 `EventExecutor` 上，此时要推迟 `handlerAdded` 方法的执行，直到通道注册完毕。再触发这个方法。
- 当前线程不是 ctx 绑定的线程，将 ctx 状态设置为添加中，并且将 `handlerAdded` 的调用包装为 runnable 任务，投递到其绑定的线程中执行
- 当前线程和 ctx 绑定的线程一致，将 ctx 状态设置为添加完成，触发 `handlerAdded` 方法。

第二和第三种情况都好理解，对于第一种情况，在实现上，`Pipeline` 内部将一个 ctx 包装 `PendingHandlerCallback` 对象，这个对象会形成一个单向链表。当通道注册到线程上时，会读取首节点，对象内部记录了 ctx 信息。读取到列表后，就逐步执行其 `handlerAdded` 方法。

### 传递入站事件方法

管道最主要的作用就是来传递具体的事件。在传递上，和 ctx 有异曲同工之妙。入站事件传递都是以 fireXX 命名的，以 `fireChannelRead` 来分析，下面是代码

```java
public final ChannelPipeline fireChannelRead(Object msg) {AbstractChannelHandlerContext.invokeChannelRead(head, msg);
        return this;
    }
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {next.invokeChannelRead(m);
        } else {executor.execute(new Runnable() {
                @Override
                public void run() {next.invokeChannelRead(m);
                }
            });}
    }
```

入站事件都是选择首节点开始触发，触发的思路也是老样子，判断是否和节点绑定的线程一致，一致直接执行，不一致则投递到任务队列。

ctx 的命令很有规律，fireXX 就是传递给下一个节点，invokeXX 就是调用本 `handler` 支持的方法。

出站事件传递和入站基本一致，只不过事件的开始是从尾结点开始，从尾结点向头结点传递，这里不展开代码分析了。