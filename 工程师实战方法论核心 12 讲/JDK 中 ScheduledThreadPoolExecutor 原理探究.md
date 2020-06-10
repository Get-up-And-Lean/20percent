# JDK 中 ScheduledThreadPoolExecutor 原理探究

### 一、前言

在[之前的一篇 chat 中](https://gitbook.cn/gitchat/activity/5b7bf4db641c5e1b7c61c376)讲解过 Java 中线程池 `ThreadPoolExecutor` 原理探究，`ThreadPoolExecutor` 是 Executors 工具类里面的一部分功能，下面来介绍另外一部分功能也就是 `ScheduledThreadPoolExecutor`的实现，后者是一个可以指定一定延迟时间后或者定时进行任务调度执行的线程池。

本 Chat 内容如下：

- `ScheduledThreadPoolExecutor` 整体结构剖析。
- 单次任务执行的 Schedule 方法原理剖析。
- 周期性任务、固定延迟执行的 `ScheduleWithFixedDelay` 方法原理剖析。
- 周期性任务、固定频率执行的 `ScheduleAtFixedRate` 方法原理剖析。

### 二、类图介绍

![enter image description here](https://images.gitbook.cn/62fb0dc0-b278-11e8-b9f0-8310634143ca)

Executors 其实是个工具类，里面提供了好多静态方法，根据用户选择返回不同的线程池实例。

`ScheduledThreadPoolExecutor` 继承了 `ThreadPoolExecutor` 并实现 `ScheduledExecutorService`接口。

线程池队列是 `DelayedWorkQueue`，和 `DelayedQueue` 类似是一个延迟队列。

`ScheduledFutureTask` 是具有返回值的任务，继承自 FutureTask，FutureTask 内部有个变量 state 用来表示任务的状态，一开始状态为 NEW，所有状态为：

```
    private static final int NEW          = 0;//初始状态
    private static final int COMPLETING   = 1;//执行中状态
    private static final int NORMAL       = 2;//正常运行结束状态
    private static final int EXCEPTIONAL  = 3;//运行中异常
    private static final int CANCELLED    = 4;//任务被取消
    private static final int INTERRUPTING = 5;//任务正在被中断
    private static final int INTERRUPTED  = 6;//任务已经被中断
```

可能的任务状态转换路径：

```
    NEW -> COMPLETING -> NORMAL //初始状态->执行中->正常结束
    NEW -> COMPLETING -> EXCEPTIONAL//初始状态->执行中->执行异常
    NEW -> CANCELLED//初始状态->任务取消
    NEW -> INTERRUPTING -> INTERRUPTED//初始状态->被中断中->被中断
```

`ScheduledFutureTask` 内部还有个变量 period 用来表示任务的类型，任务类型如下：

- period=0，说明当前任务是一次性的，执行完毕后就退出了。
- period 为负数，说明当前任务为 fixed-delay 任务，是定时可重复执行任务。
- period 为整数，说明当前任务为 fixed-rate 任务，是定时可重复执行任务。

`ScheduledThreadPoolExecutor` 的一个构造函数如下，可知线程池队列是 `DelayedWorkQueue`：

```
    //使用改造后的Delayqueue.
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        //调用父类ThreadPoolExecutor的构造函数
        super(corePoolSize, Integer.MAX_VALUE, 0, TimeUnit.NANOSECONDS,
              new DelayedWorkQueue());
    }
     public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```

### 三、原理剖析

本节讲解三个重要函数：

- `schedule(Runnable command, long delay,TimeUnit unit)`
- `scheduleWithFixedDelay(Runnable command,long initialDelay,long delay,TimeUnit unit)`
- `scheduleAtFixedRate(Runnable command,long initialDelay,long period,TimeUnit unit)`

#### 3.1 `schedule(Runnable command, long delay,TimeUnit unit)`方法

该方法作用是提交一个延迟执行的任务，任务从提交时间算起延迟 unit 单位的 delay 时间后开始执行，提交的任务不是周期性任务，任务只会执行一次，代码如下：

```
public ScheduledFuture<?> schedule(Runnable command,
                                   long delay,
                                   TimeUnit unit) {
    //(1)参数校验
    if (command == null || unit == null)
        throw new NullPointerException();

    //（2）任务转换
    RunnableScheduledFuture<?> t = decorateTask(command,
        new ScheduledFutureTask<Void>(command, null,
                                      triggerTime(delay, unit)));
    //（3）添加任务到延迟队列
    delayedExecute(t);
    return t;
}
```

- 如上代码（1）参数校验，如果 command 或者 unit 为 null，抛出 NPE 异常。
- 代码（2）装饰任务，把提交的 command 任务转换为 `ScheduledFutureTask`，`ScheduledFutureTask` 是具体放入到延迟队列里面的东西，由于是延迟任务，所以 `ScheduledFutureTask` 实现了 `long getDelay(TimeUnit unit)` 和 `int compareTo(Delayed other)` 方法，triggerTime 方法转换延迟时间为绝对时间，也就是把当前时间的纳秒数加上延迟的纳秒数后的 long 型值，如下 `ScheduledFutureTask` 的构造函数如下：

```
ScheduledFutureTask(Runnable r, V result, long ns) {
  //调用父类FutureTask的构造函数
  super(r, result);
  this.time = ns;
  this.period = 0;//period为0，说明为一次性任务
  this.sequenceNumber = sequencer.getAndIncrement();
}
```

构造函数内部首先调用了父类 FutureTask 的构造函数，父类 FutureTask 的构造函数代码如下：

```
//通过适配器把runnable转换为callable
public FutureTask(Runnable runnable, V result) {
   this.callable = Executors.callable(runnable, result);
   this.state = NEW;       //设置当前任务状态为NEW
}
```

FutureTask 中任务又被转换为了 Callable 类型后，保存到了变量 this.callable 里面，并设置 FutureTask 的任务状态为 NEW。

然后 `ScheduledFutureTask` 构造函数内部设置 time 为上面说的绝对时间，需要注意这里 period 的值为 0，这说明当前任务为一次性任务,不是定时反复执行任务。

其中 `long getDelay(TimeUnit unit)` 方法代码如下，用来获取当前任务还有多少时间就过期了：

```
//元素过期算法，装饰后时间-当前时间，就是即将过期剩余时间
public long getDelay(TimeUnit unit) {
  return unit.convert(time - now(), NANOSECONDS);
}
```

##### **`compareTo(Delayed other)` 方法代码如下：**

```
public int compareTo(Delayed other) {
    if (other == this) // compare zero ONLY if same object
        return 0;
    if (other instanceof ScheduledFutureTask) {
        ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
        long diff = time - x.time;
        if (diff < 0)
            return -1;
        else if (diff > 0)
            return 1;
        else if (sequenceNumber < x.sequenceNumber)
            return -1;
        else
            return 1;
    }
    long d = (getDelay(TimeUnit.NANOSECONDS) -
              other.getDelay(TimeUnit.NANOSECONDS));
    return (d == 0) ? 0 : ((d < 0) ? -1 : 1);
}
```

compareTo 作用是加入元素到延迟队列后，内部建立或者调整堆时候会使用该元素的 compareTo 方法与队列里面其他元素进行比较，让最快要过期的元素放到队首。所以无论什么时候向队列里面添加元素，队首的的元素都是最即将过期的元素。

- 代码（3）添加任务到延迟队列，delayedExecute 的代码如下：

```
private void delayedExecute(RunnableScheduledFuture<?> task) {

    //(4)如果线程池关闭了，则执行线程池拒绝策略
    if (isShutdown())
        reject(task);
    else {
        //(5)添加任务到延迟队列
        super.getQueue().add(task);

        //（6）再次检查线程池状态
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
            //（7）确保至少一个线程在处理任务
            ensurePrestart();
    }
}
```

代码（4）首先判断当前线程池是否已经关闭了，如果已经关闭则执行线程池的拒绝策略

否者执行代码（5）添加任务到延迟队列。添加完毕后还要重新检查线程池是否被关闭了，如果已经关闭则从延迟队列里面删除刚才添加的任务，但是有可能线程池线程已经从任务队列里面移除了该任务，也就是该任务已经在执行了，所以还需要调用任务的 cancle 方法取消任务。

如果代码（6）判断结果为 false，则会执行代码（7）确保至少有一个线程在处理任务，即使核心线程数 corePoolSize 被设置为 0，ensurePrestart 代码如下：

```
void ensurePrestart() {
    int wc = workerCountOf(ctl.get());
    //增加核心线程数
    if (wc < corePoolSize)
        addWorker(null, true);
    //如果初始化corePoolSize==0，则也添加一个线程。
    else if (wc == 0)
        addWorker(null, false);
    }
```

如上代码首先首先获取线程池中线程个数，如果线程个数小于核心线程数则新增一个线程，否者如果当前线程数为 0 则新增一个线程。

通过上面代码我们分析了如何添加任务到延迟队列，下面我们看线程池里面的线程如何获取并执行任务的，从前面讲解的 `ThreadPoolExecutor` 我们知道具体执行任务的线程是 Worker 线程，Worker 线程里面调用具体任务的 run 方法进行执行，由于这里任务是 `ScheduledFutureTask`，所以我们下面看看 `ScheduledFutureTask` 的 run 方法：

```
public void run() {

    //（8）是否只执行一次
    boolean periodic = isPeriodic();

    //（9）取消任务
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
    //（10）只执行一次，调用schdule时候
    else if (!periodic)
        ScheduledFutureTask.super.run();

    //（11）定时执行
    else if (ScheduledFutureTask.super.runAndReset()) {
        //（11.1）设置time=time+period
        setNextRunTime();

        //（11.2）重新加入该任务到delay队列
        reExecutePeriodic(outerTask);
    }
}   
```

- 如上代码（8）isPeriodic 的作用是判断当前任务是一次性任务还是可重复执行的任务，isPeriodic 的代码如下：

```
        public boolean isPeriodic() {
            return period != 0;
        }
```

可知内部是通过 period 的值来判断，由于转换任务创建 ScheduledFutureTask 时候传递的 period 为 0 ，所以这里 isPeriodic 返回 false。

- 代码（9）判断当前任务是否应该被取消，canRunInCurrentRunState 的代码如下

```
    boolean canRunInCurrentRunState(boolean periodic) {
        return isRunningOrShutdown(periodic ?
                                   continueExistingPeriodicTasksAfterShutdown :
                                   executeExistingDelayedTasksAfterShutdown);
    }
```

这里传递的 periodic 为 false，所以 `isRunningOrShutdown` 的参数为 `executeExistingDelayedTasksAfterShutdown`，`executeExistingDelayedTasksAfterShutdown` 默认是 true 标示当其它线程调用了 shutdown 命令关闭了线程池后，当前任务还是要执行，否者如果为 false，标示当前任务要被取消。

- 由于 periodic 为 false，所以执行代码（10）调用父类 FutureTask 的 run 方法具体执行任务，FutureTask 的 run 方法代码如下：

```
public void run() {
        //(12)
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;

        //(13)
        try {

            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    //(13.1)
                    setException(ex);
                }
                //(13.2)
                if (ran)
                    set(result);
            }
        } finally {
            ...        
        }
    }
```

如上代码（12）如果任务状态不是 NEW 则直接返回，或者如果当前任务状态为NEW但是使用 CAS 设置当然任务的持有者为当前线程失败则直接返回。代码（13）具体调用 callable 的 call 方法执行任务，这里在调用前又判断了任务的状态是否为 NEW 是为了避免在执行代码（12）后其他线程修改了任务的状态（比如取消了该任务）。

如果任务执行成功则执行代码（13.2）修改任务状态，set 方法代码如下：

```
    protected void set(V v) {
        //如果当前任务状态为NEW，则设置为COMPLETING
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            //设置当前任务终状为NORMAL，也就是任务正常结束
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
```

如上代码首先 CAS 设置当前任务状态从 NEW 转换到 COMPLETING，这里多个线程调用时候只有一个线程会成功，成功的线程在通过 `UNSAFE.putOrderedInt` 设置任务的状态为正常结束状态，这里没有用 CAS 是因为同一个任务只可能有一个线程可以运行到这里，这里使用 `putOrderedInt` 比使用 CAS 函数或者 `putLongVolatile` 效率要高，并且这里的场景不要求其它线程马上对设置的状态值可见。

这里思考个问题，这里什么时候多个线程会同时执行 CAS 设置任务状态从态从 NEW 到 COMPLETING？其实当同一个 comand 被多次提交到线程池时候就会存在这样的情况，由于同一个任务共享一个状态值 state。

如果任务执行失败，则执行代码（13.1），setException 的代码如下，可见与 set 函数类似：

```
    protected void setException(Throwable t) {
        //如果当前任务状态为NEW，则设置为COMPLETING
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;

            //设置当前任务终态为EXCEPTIONAL，也就是任务非正常结束
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL);                             finishCompletion();
        }
    }
```

到这里代码（10）逻辑执行完毕，一次性任务也就执行完毕了，

下面会讲到如果任务是可重复执行的，则不会执行步骤（10）而是执行代码（11）。

#### 3.2 scheduleWithFixedDelay(Runnable command,long initialDelay,long delay,TimeUnit unit)

当任务执行完毕后，延迟固定间隔时间后再次运行（fixed-delay 任务）：其中 initialDelay 说明提交任务后延迟多少时间开始执行任务 command，delay 表示当任务执行完毕后延长多少时间后再次运行 command 任务，unit 是 initialDelay 和 delay 的时间单位。任务会一直重复运行直到任务运行时候抛出了异常或者取消了任务，或者关闭了线程池。`scheduleWithFixedDelay` 的代码如下：

```
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        //(14)参数校验
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();

        //（15）任务转换,注意这里是period=-delay<0
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,                                    triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        //（16）添加任务到队列
        delayedExecute(t);
        return t;
    }
```

如上代码（14）进行参数校验，校验失败则抛出异常，代码（15）转换 command 任务为 `ScheduledFutureTask`，这里需要注意的是这里传递给 `ScheduledFutureTask` 的 period 变量的值为 -delay，period < 0 这个说明该任务为可重复执行的任务。

然后代码（16）添加任务到延迟队列后返回。

任务添加到延迟队列后线程池线程会从队列里面获取任务，然后调用 `ScheduledFutureTask` 的 run 方法执行，由于这里 period<0 所以 isPeriodic 返回 true，所以执行代码（11），runAndReset 的代码如下：

```
  protected boolean runAndReset() {
        //(17)
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;

        //(18)
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {

          ...        
    }
        return ran && s == NEW;//(19)
    }
```

该代码和 FutureTask 的 run 类似，只是任务正常执行完毕后不会设置任务的状态，这样做是为了让任务成为可重复执行的任务，这里多了代码（19）如果当前任务正常执行完毕并且任务状态为 NEW 则返回 true 否者返回 false。

如果返回了 true 则执行代码（11.1）`setNextRunTime` 方法设置该任务下一次的执行时间，`setNextRunTime` 的代码如下：

```
      private void setNextRunTime() {
            long p = period;
            if (p > 0)//fixed-rate类型任务
                time += p;
            else//fixed-delay类型任务
                time = triggerTime(-p);
        }
```

如上代码这里 p < 0 说明当前任务为 `fixed-delay` 类型任务，然后设置 time 为当前时间加上 `-p` 的时间，也就是延迟 `-p` 时间后在次执行。

总结：本节介绍的 `fixed-delay` 类型的任务的执行实现原理如下，当添加一个任务到延迟队列后，等 initialDelay 时间后，任务就会过期，过期的任务就会被从队列移除，并执行，执行完毕后，会重新设置任务的延迟时间，然后在把任务放入延迟队列实现的，依次往复。需要注意的是如果一个任务在执行某一个次时候抛出了异常，那么这个任务就结束了，但是不影响其它任务的执行。

#### 3.3 scheduleAtFixedRate(Runnable command,long initialDelay,long period,TimeUnit unit)

相对起始时间点固定频率调用指定的任务（fixed-rate 任务）：当提交任务到线程池后延迟 initialDelay 个时间单位为 unit 的时间后开始执行任务 comand ，然后 `initialDelay + period` 时间点再次执行，然后在 `initialDelay + 2 * period` 时间点再次执行，依次往复，直到抛出异常或者调用了任务的 cancel 方法取消了任务在结束或者关闭了线程池。

`scheduleAtFixedRate` 的原理与 `scheduleWithFixedDelay` 类似，下面我们讲下不同点,首先调用 `scheduleAtFixedRate` 时候代码如下：

```
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                              long initialDelay,
                                              long period,
                                              TimeUnit unit) {
    ...
    //装饰任务类，注意period=period>0，不是负的
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(period));
    ...
     return t;
}
```

如上代码 `fixed-rate` 类型的任务在转换 `command` 任务为 `ScheduledFutureTask` 的时候设置的 `period=period` 不在是 `-period`。

所以当前任务执行完毕后，调用 `setNextRunTime` 设置任务下次执行的时间时候执行的是 `time += p` 而不在是 `time = triggerTime(-p);`。

总结：相对于 `fixed-delay` 任务来说，`fixed-rate` 方式执行规则为时间为 `initdelday + n*period;` 时候启动任务，但是如果当前任务还没有执行完，下一次要执行任务的时间到了，不会并发执行，下次要执行的任务会延迟执行，要等到当前任务执行完毕后在执行一个任务。

#### 四、总结

本章讲解了 `ScheduledThreadPoolExecutor` 的实现原理，如下图内部使用的 `DelayQueue` 来存放具体任务，其中任务分为三种，其中一次性执行任务执行完毕就结束了，`fixed-delay` 任务保证同一个任务多次执行之间间隔固定时间，`fixed-rate` 任务保证任务执行按照固定的频率执行，其中任务类型使用 `period` 的值来区分。