# 9/20第09课 多线程异步调用之 Future 模式

### 线程计数器回顾

在《两种常用的线程计数器》 这一篇中，我们使用线程计数器的方式实现了：在主线程中设置一个阻塞点，当等待计数的线程执行完之后再执行阻塞点之后的代码。看段代码回顾一下：

```
public class SummonDragonDemo {

    private static final int THREAD_COUNT_NUM = 7;
    private static CountDownLatch countDownLatch = new CountDownLatch(THREAD_COUNT_NUM);

    public static void main(String[] args) throws InterruptedException {

        for (int i = 1; i <= THREAD_COUNT_NUM; i++) {
            int index = i;
            new Thread(() -> {
                try {
                    System.out.println("第" + index + "颗龙珠已收集到！");
                    //模拟收集第i个龙珠,随机模拟不同的寻找时间
                    Thread.sleep(new Random().nextInt(3000));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                //每收集到一颗龙珠,需要等待的颗数减1
                countDownLatch.countDown();
            }).start();
        }
        //等待检查，即上述7个线程执行完毕之后，执行await后边的代码
        countDownLatch.await();
        System.out.println("集齐七颗龙珠！召唤神龙！");
    }
}
```

这里简单的回顾了一下 CountDownLatch，这是因为 CountDownLatch 也实现了类似异步调用的过程，只不过具体的任务由线程去执行，但是会阻塞在主线程的`countDownLatch.await();` 处，（这里要讲的 Future 同样也会阻塞，只是阻塞在了真正数据获取的位置，后边会讲到）。

### 什么是异步调用

当我们调用一个函数的时候，如果这个函数的执行过程是很耗时的，我们就必须要等待，但是我们有时候并不急着要这个函数返回的结果。因此，我们可以让被调者立即返回，让他在后台慢慢的处理这个请求。对于调用者来说，则可以先处理一些其他事情，在真正需要数据的时候再去尝试获得需要的数据（这个真正需要数据的位置也就是上文提到的阻塞点）。这也是 Future 模式的核心思想：异步调用。

到了这里，你可能会想 CountDownLatch 不是也可以实现类似的功能的吗？也是可以让耗时的任务通过子线程的方式去执行，然后设置一个阻塞点等待返回的结果，情况貌似是这样的！但有时发现 CountDownLatch 只知道子线程的完成情况是不够的，如果在子线程完成后获取其计算的结果，那 CountDownLatch 就有些捉襟见衬了，所以 JDK 提供的 Future 类，不仅可以在子线程完成后收集其结果，还可以设定子线程的超时时间，避免主任务一直等待。

看到这里，似乎恍然大悟了！CountDownLatch 无法很好的洞察子线程执行的结果，使用 Future 就可以完成这一操作，那么 Future 何方神圣！下边我们就细细聊一下。

### Future 模式

虽然，Future 模式不会立即返回你需要的数据，但是，他会返回一个契约 ，以后在使用到数据的时候就可以通过这个契约获取到需要的数据。

![这里写图片描述](https://img-blog.csdn.net/20171030141701571?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上图显示的是一个串行程序调用的流程，可以看出当有一个程序执行的时候比较耗时的时候，其他程序必须等待该耗时操作的结束，这样的话客户端就必须一直等待，知道返回数据才执行其他的任务处理。

![这里写图片描述](https://img-blog.csdn.net/20171030145843667?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上图展示的是 Future 模式流程图，在广义的 Future 模式中，虽然获取数据是一个耗时的操作，但是服务程序不等数据完成就立即返回客户端一个伪造的数据（就是上述说的“契约”），实现了 Future 模式的客户端并不急于对其进行处理，而是先去处理其他业务，充分利用了等待的时间，这也是 Future 模式的核心所在，在完成了其他数据无关的任务之后，最后在使用返回比较慢的 Future 数据。这样在整个调用的过程中就不会出现长时间的等待，充分利用时间，从而提高系统效率。

#### Future 主要角色

![这里写图片描述](https://img-blog.csdn.net/20171030154606597?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### Future 的核心结构图

![这里写图片描述](https://img-blog.csdn.net/20171030160102759?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上述的流程就是说：Data 为核心接口，这是客户端希望获取的数据，在 Future 模式中，这个 Data 接口有两个重要的实现，分别是：RealData 和 FutureData。RealData 就是真实的数据，FutureData 他是用来提取 RealData 真是数据的接口实现，用于立即返回得到的，他实际上是真实数据 RealData 的代理，封装了获取 RealData 的等待过程。

说了这些理论的东西，倒不如直接看代码来的直接些，请看代码！

### Future 模式的简单实现

主要包含以下5个类，对应着 Future 模式的主要角色：

![这里写图片描述](https://img-blog.csdn.net/20171030161335148?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

1、Data 接口

```
/**
* 返回数据的接口
*/
public interface Data {

    String getResult();
}
```

2、FutureData 代码

```
/**
* Future数据，构造很快，但是是一个虚拟的数据，需要装配RealData
*/
public class FutureData implements Data {

    private RealData realData = null;
    private boolean isReady = false;

    private ReentrantLock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    @Override
    public String getResult() {
        while (!isReady) {
            try {
                lock.lock();
                condition.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
        return realData.getResult();
    }

    public void setRealData(RealData realData) {
        lock.lock();
        if (isReady) {
            return;
        }
        this.realData = realData;
        isReady = true;
        condition.signal();
        lock.unlock();
    }
}
```

3、RealData 代码

```
public class RealData implements Data {

    private String result;

    public RealData(String param) {
        StringBuffer sb = new StringBuffer();
        sb.append(param);
        try {
            //模拟构造真实数据的耗时操作
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        result = sb.toString();
    }

    @Override
    public String getResult() {
        return result;
    }
}
```

4、Client 代码

```
public class Client {

    public Data request(String param) {
        //立即返回FutureData
        FutureData futureData = new FutureData();
        //开启ClientThread线程装配RealData
        new Thread(() -> {
            {
                //装配RealData
                RealData realData = new RealData(param);
                futureData.setRealData(realData);
            }
        }).start();
        return futureData;
    }
}
```

5、Main

```
/**
* 系统启动，调用Client发出请求
*/
public class Main {

    public static void main(String[] args) {
        Client client = new Client();
        Data data = client.request("Hello Future!");
        System.out.println("请求完毕！");

        try {
            //模拟处理其他业务
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("真实数据：" + data.getResult());
    }
}
```

6、执行结果：

![这里写图片描述](https://img-blog.csdn.net/20171030161842527?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### JDK 中的 Future 模式实现

上述实现了一个简单的 Future 模式的实现，因为这是一个很常用的模式，在 JDK 中也给我们提供了对应的方法和接口，先看一下实例：

```
public class RealData implements Callable<String> {

    private String result;

    public RealData(String result) {
        this.result = result;
    }

    @Override
    public String call() throws Exception {
        StringBuffer sb = new StringBuffer();
        sb.append(result);
        //模拟耗时的构造数据过程
        Thread.sleep(5000);
        return sb.toString();
    }
}
```

这里的 RealData 实现了 Callable 接口，重写了 call 方法，在 call 方法里边实现了构造真实数据耗时的操作。

```
public class FutureMain {

    public static void main(String[] args) throws ExecutionException, InterruptedException {

        FutureTask<String> futureTask = new FutureTask<>(new RealData("Hello"));

        ExecutorService executorService = Executors.newFixedThreadPool(1);
        executorService.execute(futureTask);

        System.out.println("请求完毕！");

        try {
            Thread.sleep(2000);
            System.out.println("这里经过了一个2秒的操作！");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("真实数据：" + futureTask.get());
        executorService.shutdown();
    }
}
```

执行结果：

![这里写图片描述](https://img-blog.csdn.net/20171031093508010?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上述代码，通过：FutureTask futureTask = new FutureTask<>(new RealData("Hello")); 这一行构造了一个 futureTask 对象，表示这个任务是有返回值的，返回类型为 String，下边看一下 FutureTask 的类图关系：

![这里写图片描述](https://img-blog.csdn.net/20171031094033309?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

FutureTask 实现了 RunnableFuture 接口，RunnableFuture 接口继承了 Future 和 Runnable 接口。因为 RunnableFuture 实现了 Runnable 接口，因此 FutureTask 可以提交给 Executor 进行执行，FutureTask 有两个构造方法，如下：

构造方法（一），参数为 Callable：

![这里写图片描述](https://img-blog.csdn.net/20171031094507268?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

构造方法（二），参数为 Runnable：

![这里写图片描述](https://img-blog.csdn.net/20171031094513537?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上述的第二个构造方法，传入的是 Runnable 接口的话，会通过 Executors.callable（）方法转化为 Callable，适配过程如下：

![这里写图片描述](https://img-blog.csdn.net/20171031094622105?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](https://img-blog.csdn.net/20171031103834636?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这里为什么要将 Runnable 转化为 Callable 呢？首先看一下两者之间的区别：

```
(1) Callable规定的方法是call()，Runnable规定的方法是run()；
(2) Callable的任务执行后可返回值，而Runnable的任务是不能返回值得； 
(3) call()方法可以抛出异常，run()方法不可以；
(4) 运行Callable任务可以拿到一个Future对象，Future 表示异步计算的结果。
```

最关键的是第二点，就是 Callable 具有返回值，而 Runnable 没有返回值。Callable 提供了检查计算是否完成的方法，以等待计算的完成，并获取计算的结果。

计算完成后只能使用 get 方法来获取结果，如果线程没有执行完，Future.get() 方法可能会阻塞当前线程的执行；如果线程出现异常，Future.get() 会 throws InterruptedException 或者 ExecutionException；如果线程已经取消，会抛出 CancellationException。取消由 cancel 方法来执行。isDone 确定任务是正常完成还是被取消了。

一旦计算完成，就不能再取消计算。如果为了可取消性而使用`Future`但又不提供可用的结果，则可以声明`Future<?>` 形式类型、并返回 `null` 作为底层任务的结果。