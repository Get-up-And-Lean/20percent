# 19/20第19课 进一步分析 Executor 框架

在第07课《使用线程池实现线程的复用》一节中介绍了线程池 Executor 框架的基本使用与最佳实践，这一节进一步分析 Executor 框架。

### 线程池实现原理

![enter image description here](https://images.gitbook.cn/e49c5e30-f1bf-11e7-9d5e-fb4b85bebad9)

一般线程池的处理流程如上图所示，具体细节如下：

> 1、当前线程数小于 corePoolSize，则创建新线程来执行任务；
> 2、当前线程数大于等于 corePoolSize，则将任务加入 BlockingQueue；
> 3、若队列已满，则创建新线程处理任务；
> 4、若线程数超过 maxinumPoolsize，则任务将被拒绝，并调用RejectExecutionHandler.rejectedExecution() 方法；

### JDK 对线程池的支持

JDK 提供了 Executor 框架，可以让我们有效的管理和控制我们的线程，其实质也就是一个线程池。Executor 下的接口和类继承关系如下：

![enter image description here](https://images.gitbook.cn/35007410-f1bb-11e7-9d5e-fb4b85bebad9)

> （1） Executor：一个接口，其定义了一个接收 Runnable 对象的方法 execute，其方法签名为 void execute(Runnable command)；
>
> （2）ExecutorService：是一个比 Executor 使用更广泛的子类接口，其提供了生命周期管理的方法，以及可跟踪一个或多个异步任务执行状况返回 Future 的方法；
>
> （3）AbstractExecutorService：ExecutorService 执行方法的默认实现；
>
> （4）ScheduledExecutorService：一个可定时调度任务的接口；
>
> （5）ThreadPoolExecutor：线程池，可以通过调用 Executors 以下静态工厂方法来创建线程池并返回一个 ExecutorService 对象：
>
> （6）ScheduledThreadPoolExecutor：ScheduledExecutorService 的实现，一个可定时调度任务的线程池；

### Executor 框架的两级调度模型

何为 Executor 框架的两级调度模型？简单的来说就是**我们使用的 Java 线程被一对一映射为本地操作系统线程**。Java 线程启动的时候会启动一个本地操作系统线程，当该 Java 线程终止时，这个操作系统线程也会被回收。操作系统会调度所有线程并将它们分配给可用的 CPU。

由此出现了应用程序通过 Executor 框架控制上层的调用，而下层的调度由操作系统的内核来控制，从而形成了两级的调度模型，并且下层的调度不受应用程序的控制，任务的两级调度模型如下图：

![enter image description here](https://images.gitbook.cn/900ff830-f1c0-11e7-acdb-b3a42243c716)

### Executor 框架的结构和使用流程

Executor 主要由三部分组成：任务产生部分，任务处理部分，结果获取部分。（设计模式：生产者与消费者模式）

Executor 框架的使用流程如下：

![enter image description here](https://images.gitbook.cn/5f158f00-f1c1-11e7-acdb-b3a42243c716)

1、主线程首先要创建实现 Runnable 或 Callable 接口的任务对象。工具类 Executors 可以把一个 Runnable 对象封装成为一个 Callable 对象`（Executors.callable(Runnable task)` 或者 `Executors.callable(Runnable task, Object result)）`；

2、然后可以把 Runnable 对象直接交给 ExecutorService 执行（`ExecutorService.execute(Runnable command)）`，或者也可以把 Runnable 对象或 Callable 对象交给 ExecutorService 执行`（ExecutorService.submit(Runnable task)` 或 `ExecutorService.submit(Callable t)）`。

3、如果执行 submit 方法，ExecutorService 将返回一个实现 Future 接口的对象（到目前为止的 JDK 中返回的是 FutureTask 对象）。由于 FutureTask 实现了 Runnable 接口，程序员也可以创建 FutureTask，然后直接交给 ExecutorService 执行。

4、最后主线程可以执行 FutureTask.get() 方法来等待任务执行完成。主线程也可以执行`FutureTask.cancel(boolean matInterruptIfRunning)` 来取消此任务的执行。