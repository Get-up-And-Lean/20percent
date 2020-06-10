# Java 异步编程实战（上篇）前言

异步编程是可以让程序并行运行的一种手段，其可以让程序中的一个工作单元与主应用程序线程分开独立运行，并且等工作单元运行结束后通知主应用程序线程它的运行结果或者失败原因。使用它有许多好处，例如改进的应用程序性能和减少用户等待时间等。

比如线程 A 要做从数据库 I 和数据库 II 查询一条记录，并且把两者结果拼接起来作为前端展示使用，如线程 A 是同步调用两次查询，则整个过程耗时时间为访问数据库 I 的耗时加上访问数据库 II 的耗时，如下图：

![在这里插入图片描述](https://images.gitbook.cn/32154580-bd7c-11e9-a60a-1f6c39d7f411)

如果为异步调用则可以在线程 A 内开启一个异步运行单元来从数据库 I 获取数据，然后线程 A 本身来从数据库 II 获取数据，并且等两者结果都返回后，在拼接两者结果，这时候整个过程耗时为 max(线程 A 从数据库 II 获取数据耗时，异步运行单元从数据库 I 获取数据耗时），如下图：

![在这里插入图片描述](https://images.gitbook.cn/6bb88540-bd7c-11e9-ad77-7946be4ae2e1)

可见整个过程耗时有显著缩短，对于用户来说页面响应时间会更短，对用户体验会更好，其中异步单元一般是线程池中的线程。

其实上面对异步编程的定义有点问题，其定义异步编程是可以让程序并行运行的一种手段，这里并行应该改为并发，因为并发与并行是有本质区别的，并发是指同一个时间段内多个任务同时都在执行，并且都没有执行结束，而并行是说在单位时间内多个任务同时在执行，并发任务强调在一个时间段内同时执行，而一个时间段有多个单位时间累积而成，所以说并发的多个任务在单位时间内不一定同时在执行。

在单个 cpu 的时代多个任务同时运行都是并发，这是因为 cpu 同时只能执行一个任务，单个 cpu 时代多任务是共享一个 cpu 的，当一个任务占用 cpu 运行时候，其它任务就会被挂起，当占用 cpu 的任务时间片用完后，会把 cpu 让给其它任务来使用，所以在单 cpu 时代多线程编程是没有意义的，并且线程间频繁的上下文切换还会带来开销。

如下图单个 cpu 上运行两个线程，可知线程 A 和 B 是轮流使用 cpu 进行任务处理的，也就是同时 CPU 只在执行一个线程上面的任务，当前线程 A 的时间片用完后会进行线程上下文切换，也就是保存当前线程的执行线程，然后切换线程 B 占用 cpu 运行任务。

![在这里插入图片描述](https://images.gitbook.cn/42ad9fa0-bd7c-11e9-a03f-dde2205d398c)

如下图双 cpu 时候，线程 A 和线程各自在自己的 CPU 上执行任务，实现了真正的并行运行。

![在这里插入图片描述](https://images.gitbook.cn/47825930-bd7c-11e9-a03f-dde2205d398c)

而在多线程编程实践中线程的个数往往多于 CPU 的个数，所以平时都是称多线程并发编程而不是多线程并行编程。

### 使用 Thread&Runnable 实现异步编程

在 Java 中最简单的是创建一个 Thread 来实现异步编程，比如在同步编程下我们在一个线程中要做两件事情代码大概是如下面所示：

```java
public class SyncExample {

    public static void doSomethingA() {

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("--- doSomethingA---");
    }

    public static void doSomethingB() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("--- doSomethingB---");

    }

    public static void main(String[] args) {

        long start = System.currentTimeMillis();
        // 1.执行任务 A
        doSomethingA();

        // 2.执行任务 B
        doSomethingB();

        System.out.println(System.currentTimeMillis() - start);

    }
}
```

如上代码 main 线程内首先执行了 doSomethingA 方法，然后执行了 doSomethingB 方法，那么整个过程耗时为 4s 时间，如果开启一个线程来异步执行任务 doSomethingA，main 函数所在线程执行 doSomethingB 则可以大大缩短整个任务处理耗时，上面 main 函数代码可以修改为如下：

```java
    public static void main(String[] args) throws InterruptedException {

        long start = System.currentTimeMillis();

        // 1.开启异步单元执行任务 A
        Thread thread = new Thread(() -> {
            try {
                doSomethingA();

            } catch (Exception e) {
                e.printStackTrace();
            }
        }, "threadA");
        thread.start();

        // 2.执行任务 B
        doSomethingB();

        // 3.同步等待线程 A 运行结束
        thread.join();
        System.out.println(System.currentTimeMillis() - start);
    }
```

如上代码 1 我们在 main 函数所在线程内开启了一个线程 A 用来异步执行 doSomethingA 任务，这时候线程 A 与 main 线程并发运行，也就是任务 doSomethingA 与任务 doSomethingB 并发运行，代码 3 则等 main 线程运行完 doSomethingB 任务后同步等待线程 A 运行完毕，运行上面代码，打印结果可知这时候整个过程耗时 2s 左右,可知使用异步编程可以大大缩短任务运行时间。但是上述代码存在两个问题：

- 每当执行异步任务时候直接创建了一个 Thread 来执行异步任务，这在生产实践中是不建议使用的，这是因为线程创建与销毁是有开销的，并且没有限制线程的个数，如果使用不当可能会把系统线程用尽，从而造成错误。在生产环境中一般是创建一个线程池，然后使用线程池中的线程来执行异步任务，线程池中的线程是可以被复用的，这可以大大减少线程创建与销毁开销。
- 上面使用 Thread 执行的异步任务并没有返回值，如果我们想异步执行一个任务，并且需要在任务执行完毕后获取任务执行结果，则上面这个方式是满足不了的，这时候就需要 JDK 中的 Future 了。

### FutureTask 实现异步编程

FutureTask 代表了一个可被取消的异步计算任务，该类提供了 Future 接口的实现，比如提供了开启和取消任务、查询任务是否完成、获取计算结果的接口。

FutureTask 任务的结果只有当任务完成后才能获取，并且只能通过 get 系列方法获取，当结果还没出来时候，线程调用 get 系列方法会被阻塞；另外一旦任务呗执行完成，任务不能被重启，除非运行时候使用了 runAndReset 方法；FutureTask 中的任务可以是 Callable 类型，也可以是 Runnable 类型（因为 FutureTask 实现了 Runnable 接口），FutureTask 类型的任务可以被提交到线程池执行。

我们修改上节的例子如下：

```Java
public class AsyncFutureExample {

    public static String doSomethingA() {

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("--- doSomethingA---");

        return "TaskAResult";
    }

    public static String doSomethingB() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("--- doSomethingB---");
        return "TaskBResult";

    }

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        long start = System.currentTimeMillis();

        // 1.创建 future 任务
        FutureTask<String> futureTask = new FutureTask<String>(() -> {
            String result = null;
            try {
                result = doSomethingA();

            } catch (Exception e) {
                e.printStackTrace();
            }
            return result;
        });

        // 2.开启异步单元执行任务 A
        Thread thread = new Thread(futureTask, "threadA");
        thread.start();

        // 3.执行任务 B
        String taskBResult = doSomethingB();

        // 4.同步等待线程 A 运行结束
        String taskAResult = futureTask.get();

        //5.打印两个任务执行结果
        System.out.println(taskAResult + " " + taskBResult); 
        System.out.println(System.currentTimeMillis() - start);

    }
}
```

- 如上代码 doSomethingA 和 doSomethingB 方法都是有返回值的任务，main 函数内代码 1 创建了一个异步任务 futureTask，其内部执行任务 doSomethingA。
- 代码 2 则创建了一个线程，并且以 futureTask 为执行任务，并且启动；代码 3 使用 main 线程执行任务 doSomethingB，这时候任务 doSomethingB 和 doSomethingA 是并发运行的，等 main 函数运行 doSomethingB 完毕后，执行代码 4 同步等待 doSomethingA 任务完成，然后代码 5 打印两个任务的执行结果。
- 如上可知使用 FutureTask 可以获取到异步任务的结果。

使用线程池运行方式代码如下：

```Java
    // 0 自定义线程池
    private final static int AVALIABLE_PROCESSORS = Runtime.getRuntime().availableProcessors();
    private final static ThreadPoolExecutor POOL_EXECUTOR = new ThreadPoolExecutor(AVALIABLE_PROCESSORS,
            AVALIABLE_PROCESSORS * 2, 1, TimeUnit.MINUTES, new LinkedBlockingQueue<>(5),
            new ThreadPoolExecutor.CallerRunsPolicy());

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        long start = System.currentTimeMillis();

        // 1.创建 future 任务
        FutureTask<String> futureTask = new FutureTask<String>(() -> {
            String result = null;
            try {
                result = doSomethingA();

            } catch (Exception e) {
                e.printStackTrace();
            }
            return result;
        });

        // 2.开启异步单元执行任务 A
        POOL_EXECUTOR.execute(futureTask);

        // 3.执行任务 B
        String taskBResult = doSomethingB();

        // 4.同步等待线程 A 运行结束
        String taskAResult = futureTask.get();
        // 5.打印两个任务执行结果
        System.out.println(taskAResult + " " + taskBResult);
        System.out.println(System.currentTimeMillis() - start);
    }
```

如上可知代码 0 创建了一个线程池，代码 2 添加异步任务到线程池，这里我们是调用了线程池的 execute 方法把 futureTask 提交到线程池的。

FutureTask 虽然提供了用来检查任务是否执行完成、等待任务执行结果、获取任务执行结果的方法，但是这些特色并不足以让我们写出简洁的并发代码。比如它并不能清楚的表达出多个 FutureTask 之间的关系，另外为了从 Future 获取结果，我们必须调用 get()方法，而该方法还是会在任务执行完毕前阻塞调用线程的，这明显不是我们想要的。

### 使用 CompletableFuture 实现异步编程

为了克服 FutureTask 的局限性，以及满足我们对异步编程的需要，JDK8 中提供了 CompletableFuture，CompletableFuture 的功能强大之一是其可以让两个或者多个 CompletableFuture 进行运算来产生结果，下面我们来看其提供的一个基于 thenCompose 实现当一个 CompletableFuture 执行完毕后，执行另外一个 CompletableFuture：

```Java
public class TestTwoCompletableFuture {
    // 1.异步任务，返回 future
    public static CompletableFuture<String> doSomethingOne(String encodedCompanyId) {
        // 1.1 创建异步任务
        return CompletableFuture.supplyAsync(new Supplier<String>() {

            @Override
            public String get() {

                // 1.1.1 休眠 1s，模拟任务计算
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                // 1.1.2 解密，并返回结果
                String id = encodedCompanyId;
                return id;
            }
        });
    }

    // 2.开启异步任务，返回 future
    public static CompletableFuture<String> doSomethingTwo(String companyId) {
        return CompletableFuture.supplyAsync(new Supplier<String>() {

            @Override
            public String get() {

                // 2.1,休眠 3s，模拟计算
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }

                // 2.2 查询公司信息，转换为 str，并返回
                String str = companyId + ":alibaba";
                return str;
            }
        });
    }

    public static void main(String[] args) throws InterruptedException, ExecutionException {
        // I，等 doSomethingOne 执行完毕后，接着执行 doSomethingTwo
        CompletableFuture result = doSomethingOne("123").thenCompose(id -> doSomethingTwo(id));
        System.out.println(result.get());
    }
}
```

如上 main 函数中首先调用了方法 doSomethingOne("123")开启了一个异步任务，并返回了对应的 CompletableFuture 对象，我们取名为 future1,然后在 future1 的基础上调用了 thenCompose 方法，企图让 future1 执行完毕后，激活使用其结果作为 doSomethingTwo(String companyId)方法的参数创建的异步任务返回的 CompletableFuture 任务。

### JDK8-Stream&CompletableFuture

JDK8 中提供了流式对数据进行处理的功能，它的出现允许我们以声明式方式对数据集合进行处理

比如下面代码，我们从 person 列表中需要过滤出年龄大于 10 岁的人，并且收集对应的 name 字段到 list,然后统一打印处理，在使用非 Strem 的情况下，我们会使用下面代码来实现：

```Java
public static List<Person> makeList() {
        List<Person> personList = new ArrayList<Person>();
        Person p1 = new Person();
        p1.setAge(10);
        p1.setName("zlx");
        personList.add(p1);

        p1 = new Person();
        p1.setAge(12);
        p1.setName("jiaduo");
        personList.add(p1);

        p1 = new Person();
        p1.setAge(5);
        p1.setName("ruoran");
        personList.add(p1);
        return personList;
    }

        public static void noStream(List<Person> personList) {

        List<String> nameList = new ArrayList<>();

        for (Person person : personList) {
            if (person.age >= 10) {
                nameList.add(person.getName());
            }
        }

        for(String name: nameList) {
            System.out.println(name);
        }

    }

    public static void main(String[] args) {

        List<Person> personList = makeList();

        noStream(personList);

    }
```

如上代码可知 noStream 方法内是典型的过程式编码，我们写了个 for 循环一个个判断当前 person 对象中的 age 字段值是否大于等于 10，如果是则把当前对象的 name 字段放到手动创建的 nameList 列表里面。然后在开启新的 for 循环逐个遍历 nameList 中的 name 字段。

下面我们使用 stream 方式来修改上面代码：

```Java
    public static void useStream(List<Person> personList) {

        List<String> nameList = personList.stream().filter(person -> person.getAge() >= 10)//1.过滤大于等于 10 的
                                                   .map(person -> person.getName())//2.使用 map 映射元素
                                                   .collect(Collectors.toList());//3.收集映射后元素

        nameList.stream().forEach(name -> System.out.println(name));
    }
```

如上代码我们首先从 personList 获取到流对象，然后在其上进行了 filter 运算，过滤出年龄大于等于 10 的 person,然后运用 map 方法映射 person 对象到 name 字段，然后使用 collect 方法收集所有的 name 字段为 nameList，然后从 nameList 上获取流并调用 forEach 进行打印。上面代码就是声明式编程，其可读性很强，代码直接可以说明想要什么（从代码就可以知道我们要过滤出年龄大于等于 10 的人，并且把满足条件的 person 的 name 字段收集起来，然后打印）。

下面我们看当 Stream 与 CompletableFuture 结合起来会产生什么火花，首先我们来看一个需求，这个需求是消费端对服务提供方集群中的某个服务进行广播调用（轮询调用同一个服务的不同提供者的机器）

```Java
    // 1.生成 ip 列表
        List<String> ipList = new ArrayList<String>();
        for (int i = 1; i <= 10; ++i) {
            ipList.add("192.168.0." + i);
        }

        // 2.并发调用
        long start = System.currentTimeMillis();
        List<CompletableFuture<String>> futureList = ipList.stream()
                .map(ip -> CompletableFuture.supplyAsync(() -> rpcCall(ip, ip)))//同步转换为异步
                .collect(Collectors.toList());//收集结果

       //3.等待所有异步任务执行完毕
        List<String> resultList = futureList.stream()
                                                               .map(future -> future.join())//同步等待结果
                                                               .collect(Collectors.toList());//对结果进行收集

        // 4.输出
        resultList.stream().forEach(r -> System.out.println(r));

        System.out.println("cost:" + (System.currentTimeMillis() - start));
```

- 如上代码 2，从 ipList 获取了 stream,然后通过 map 操作符把 ip 转换为了远程调用。
- 需要注意的是这里通过使用 CompletableFuture.supplyAsync 方法把 rpc 的同步调用转换为了异步，也就是把同步调用结果转换为了 CompletableFuture 对象，所以操作符 map 返回的是一个 CompletableFuture，然后 collect 操作把所有的 CompletableFuture 对象收集为 list 后返回。
- 需要注意的是这里多个 rpc 调用时并发的执行的，而不是顺序执行，因为 CompletableFuture.supplyAsync 方法把 rpc 的同步调用转换为了异步。
- 代码 3 从 futureList 获取流，然后使用 map 操作符把 future 对象转换为 future 的执行结果，这里是使用 future 的 join 方法来阻塞获取每个异步任务执行完毕，然后返回执行结果，最后使用 collect 操作把所有的结果收集到 resultList
- 代码 4 则从 resultList 获取流，然后打印结果。
- 运行上面代码会输出耗时大概为 2s，这可以证明上面 10 个 rpc 调用时并发的运行的，而不是串行执行。

### Reactive 异步编程

#### reactive 编程概述

> 反应式编程是一种涉及数据流和变化传播的异步编程范例。这意味着可以通过所采用的编程语言轻松地表达静态（例如阵列）或动态（例如事件发射器）数据流。

作为反应式编程方向的第一步，Microsoft 在.NET 生态系统中创建了 Reactive Extensions（Rx）库。然后 RxJava 在 JVM 上实现了响应式编程。随着时间的推移，通过 Reactive Streams 工作出现了 Java 的标准化，这一规范定义了 JVM 上的反应库的一组接口和交互规则。它的接口已经在父类 Flow 下集成到 Java 9 中。

另外 Java 8 还引入了 Stream，它旨在有效地处理数据流（包括原始类型），这些数据流可以在没有延迟或很少延迟的情况下访问。它是基于拉的，只能使用一次，缺少与时间相关的操作，并且可以执行并行计算，但无法指定要使用的线程池。但是它还没有设计用于处理延迟操作，例如 I / O 操作。其所不支持的特性就是 Reactor 或 RxJava 等 Reactive API 的用武之地。

Reactor 或 Rxjava 等反应性 API 也提供 Java 8 Stream 等运算符，但它们更适用于任何流序列（不仅仅是集合），并允许定义一个转换操作的管道，该管道将应用于通过它的数据，这要归功于方便的流畅 API 和使用 lambdas。它们旨在处理同步或异步操作，并允许您缓冲，合并，连接或对数据应用各种转换。

首先考虑一下，为什么需要这样的异步反应式编程库？现代应用程序可以支持大量并发用户，即使现代硬件的功能不断提高，现代软件的性能仍然是一个关键问题。

人们可以通过两种方式来提高系统的能力：

- 并行化：使用更多线程和更多硬件资源。
- 在现有资源的使用方式上寻求更高的效率。

通常，Java 开发人员使用阻塞代码编写程序。这种做法很好，直到出现性能瓶颈，此时需要引入额外的线程。但是，资源利用率的这种扩展会很快引入争用和并发问题。

更糟糕的是，会导致浪费资源。一旦程序涉及一些延迟（特别是 I / O，例如数据库请求或网络调用），资源就会被浪费，因为线程（或许多线程）现在处于空闲状态，等待数据。

所以并行化方法不是灵丹妙药，获得硬件的全部功能是必要的。

第二种方法，寻求现有资源的更高的使用率，可以解决资源浪费问题。通过编写异步，非阻塞代码，您可以使用相同的底层资源将执行切换到另一个活动任务，然后在异步处理完成后返回到当前线程进行继续处理。

但是如何在 JVM 上生成异步代码？Java 提供了两种异步编程模型：

- CallBacks：异步方法没有返回值，但需要额外的回调参数（lambda 或匿名类），在结果可用时调用它们。
- Futures：异步方法立即返回 Future 。异步线程计算任务结果，该值不会立即可用，并且可以轮询对象，直到该值可用。

但是上面两种方法都有局限性。首先多个 callback 难以组合在一起，很快导致代码难以阅读以及难以维护（称为“Callback Hell”）。

本文通过 Spring5 官网的一个例子来体验下使用 reactive 编程带来的好处，考虑下面一个例子，在用户的 UI 上展示用户喜欢的 top 5 的商品的详细信息，如果不存在的话则调用推荐服务获取 5 个；这个功能的实现需要三个服务接口支持：

- 一个是根据用户 id 获取用户喜欢的 top5 的商品的 ID 的接口（userService.getFavorites）
- 第二个是根据商品 ID 获取商品详情信息接口（favoriteService.getDetails）
- 第三个是一个根据大数据来推算用户喜爱的商品详情的服务（suggestionService.getSuggestions）

如果基于 callback 方式实现功能，可能的代码如下：

```Java
userService.getFavorites(userId, new Callback<List<String>>() { //1
  public void onSuccess(List<String> list) { //2
    if (list.isEmpty()) { //3
      suggestionService.getSuggestions(new Callback<List<Favorite>>() {//4
        public void onSuccess(List<Favorite> list) { 
          UiUtils.submitOnUiThread(() -> { //5
            list.stream()
                .limit(5)
                .forEach(uiList::show); //6
            });
        }

        public void onError(Throwable error) { //7
          UiUtils.errorPopup(error);
        }
      });
    } else {
      list.stream() //8
          .limit(5)
          .forEach(favId -> favoriteService.getDetails(favId, //9
            new Callback<Favorite>() {
              public void onSuccess(Favorite details) {//10
                UiUtils.submitOnUiThread(() -> uiList.show(details));
              }

              public void onError(Throwable error) {//11
                UiUtils.errorPopup(error);
              }
            }
          ));
    }
  }

  public void onError(Throwable error) {
    UiUtils.errorPopup(error);
  }
});
```

- 这三个服务接口都是基于 callback 的，意味着调用这三个方法不会被阻塞调用线程，而是会及时返回，当具体请求结果回来后会异步调用注册的 callback 函数（一般是在一个公共线程池例如 forkjoin 或者用户自定义线程池内执行 callback），如果结果正常则会调用 callback 的 onSuccess 方法，如果结果异常则会调用 onError 方法。
- 代码 1 中我们调用了 userService.getFavorites 接口来获取用户 userId 的推荐商品 id 列表，如果获取结果正常则会调用代码 2，如果失败则会调用代码 7，通知用户 UI 错误信息。
- 如果正常则会执行代码 3 判断推荐商品 id 列表是否为空，如果是的话则执行代码 4 调用推荐服务（suggestionService.getSuggestions），如果获取推荐商品详情失败则执行代码 7callback 的 OnError 把错误信息显示到用户 UI，否则如果成功则执行代码 5 切换线程到 UI 线程，在获取的商品详情列表上施加 jdk8 stream 运算使用 limit 获取 5 个元素然后显示到 UI 上（这个过程是 UI 线程来做）。
- 代码 3 如果判断用户推荐商品 id 列表不为空则执行代码 8，在商品 id 列表上使用 JDK9 stream 获取流，然后使用 limit 获取 5 个元素，然后执行代码 9 调用 favoriteService.getDetails 服务获取具体商品的详情，这里多个 id 获取详情是并发进行的（因为 favoriteService.getDetails 是异步的），当获取到详情成功后会执行代码 10 在 UI 线程上绘制出商品详情信息，如果失败则执行代码 11 显示错误。

如上代码基于 callback 的实现代码可读性比较差，并且会出现代码冗余，下面看看基于 Spring5 中的 reactor 库模式实现上面功能,基于 reactor 的改造需要让上面三个接口返回值都修改为 Flux（其是可以包含多个元素的流对象）在 Flux 上可以施加不同的操作类，代码如下：

```Java
userService.getFavorites(userId) //1
           .flatMap(favoriteService::getDetails) //2
           .switchIfEmpty(suggestionService.getSuggestions()) //3
           .take(5) //4
           .publishOn(UiUtils.uiThreadScheduler()) //5
           .subscribe(uiList::show, UiUtils::errorPopup); //6
```

- 代码 1 调用 getFavorites 服务获取 userId 对应的商品列表，该方法会马上返回一个流对象，然后代码 2 在流上施加 flatMap 运算把每个商品 id 转换为商品 Id 对应的商品详情信息（通过调用服务 favoriteService::getDetails），然后把所有商品详情信息组成新的流返回。
- 代码 3 判断如果返回的流中没有元素则调用推荐服务 suggestionService.getSuggestions()服务获取推荐的商品详情列表，代码 4 则从代码 2 或者代码 3 返回的流中获取 5 个元素（5 个商品详细信息），然后执行代码 5，publishOn 把当前线程切换到 UI 调度器来执行，走到这里时候其实流还没有流动，代码 6 则通过 subscribe 方法激活整个流处理链，然后在 UI 线程上绘制商品详情列表或者显示错误。

如上代码可知基于 reactor 编写的代码逻辑属于声明式编程，比较通俗易懂，代码量也比较少，并且不含有重复的代码。

future 相比 callback 要好一些，但尽管 CompletableFuture 在 Java 8 上进行了改进，但它们仍然表现不佳。一起编排多个 future 是可行但是不容易的，它们不支持延迟计算（比如 rxjava 中的 defer 操作）和高级错误处理。

考虑另外一个例子：首先我们获取一个 id 列表，然后根据 id 分别获取对应的 name 和统计数据，然后组合每个 id 对应的 name 和统计数据为一个新的数据，最后输出所有组合对的值，下面我们使用 CompletableFuture 来实现这个功能，以便保证整个过程是异步的，并且每个 id 对应的处理是并发的：

```Java
CompletableFuture<List<String>> ids = ifhIds(); //1

CompletableFuture<List<String>> result = ids.thenComposeAsync(l -> { //2
  Stream<CompletableFuture<String>> zip =
      l.stream().map(i -> { //3
        CompletableFuture<String> nameTask = ifhName(i); //3.1
        CompletableFuture<Integer> statTask = ifhStat(i); //3.2

        return nameTask.thenCombineAsync(statTask, (name, stat) -> "Name " + name + " has stats " + stat); //3.3
      });
  List<CompletableFuture<String>> combinationList = zip.collect(Collectors.toList()); //4
  CompletableFuture<String>[] combinationArray = combinationList.toArray(new CompletableFuture[combinationList.size()]);//5

  CompletableFuture<Void> allDone = CompletableFuture.allOf(combinationArray); //6
  return allDone.thenApply(v -> combinationList.stream()//7
      .map(CompletableFuture::join) 
      .collect(Collectors.toList()));
});

List<String> results = result.join(); //8
```

- 如上代码 1 我们调用 ifhIds 方法异步返回了一个 CompletableFuture 对象，其内部保存了 id 列表

- 代码 2 调用 ids 的 thenComposeAsync 方法返回一个新的 CompletableFuture 对象，新 CompletableFuture 对象的数据是代码 2 中的 lambda 表达式执行结果，表达式内代码 3 获取 id 列表的流对象，然后使用 map 操作把 id 元素转换为 name 与统计信息拼接的字符串，这里是通过代码 3.1 根据 id 获取 name 对应的 CompletableFuture 对象，代码 3.2 获取统计信息对应的 CompletableFuture，然后使用代码 3.3 把两个 CompletableFuture 对象进行合并做到的。

- 代码 3 会返回一个流对象，其中元素是所有 id 对应的 name 与统计信息组合后的结果，然后代码 4 把流中元素收集保存到了 combinationList 列表里面。代码 5 把列表转换为了数组，这是因为代码 2 的 allOf 操作符的参数必须为数组。

- 代码 6 把 combinationList 列表中的所有 CompletableFuture 对象转换为了一个 allDone（等所有 CompletableFuture 对象的任务执行完毕），到这里我们调用 allDone 的 get（）方法就可以等待所有异步处理执行完毕，但是我们目的是想获取到所有异步任务的执行结果，所以代码 7 在 allDone 上施加了 thenApply 运算，意在等所有任务处理完毕后调用所有 CompletableFuture 的 join 方法获取每个任务的执行结果，然后收集为列表后返回一个新的 CompletableFuture 对象，然后代码 8 在新的 CompletableFuture 上调用 join 方法获取所有执行结果列表。

  Reactor 本身提供了更多的开箱即用的操作符，使用 Reactor 来实现上面功能代码如下:

```Java
Flux<String> ids = ifhrIds(); //1

Flux<String> combinations =
    ids.flatMap(id -> { //2
      Mono<String> nameTask = ifhrName(id); //2.1
      Mono<Integer> statTask = ifhrStat(id); //2.2

      return nameTask.zipWith(statTask, //2.3
          (name, stat) -> "Name " + name + " has stats " + stat);
    });

Mono<List<String>> result = combinations.collectList(); //3

List<String> results = result.block(); //4
```

- 如上代码 1 我们调用 ifhIds 方法异步返回了一个 Flux 对象，其内部保存了 id 列表

- 代码 2 调用 ids 的 flatMap 方法对其中元素进行转换，代码 2.1 根据 id 获取 name 信息（返回流对象 Mono），代码 2.2 根据 id 获取统计信息(返回流对象 Mono)，代码 3 结合两个流为新的流元素。

- 代码 3 调用新流的 collectList 方法把所有的流对象转换为列表，然后返回一个新的 Mono 流对象。

- 代码 4 则调用新的 Mono 流对象的 block 方法阻塞获取所有执行结果。

  如上代码使用 reactor 方式编写的代码相比使用 CompletableFuture 实现相同功能来说，更简洁，更通俗易懂。

  Callback 和 Future 的这些弊病是相似的，并且是响应式编程与发布者 - 订阅者对的地址。

诸如 Reactor 或者 rxjava 之类的反应库旨在解决 JVM 上“经典”异步方法的这些缺点，同时还关注一些其他方面：

- 可组合性和可读性
- 数据作为一个用丰富的运算符词汇表操纵的流程
- 在您订阅之前没有任何事情发生
- 背压或消费者向生产者发出信号表明排放率过高的能力
- 高级但高价值的抽象，与并发无关

#### rxjava 异步编程

RxJava 是 Reactive Extensions 的 Java VM 实现：RxJava 是一个库，用于通过使用可观察序列来编写异步和基于事件的程序。

它扩展了观察者模式以支持数据/事件序列，并添加了允许您以声明方式组合序列的运算符，同时抽象出对低级线程、同步、线程安全和并发数据结构等问题的关注，RxJava 试图做的非常轻量级，它仅仅作为单个 JAR 实现，仅关注 Observable 抽象和相关的高阶运算函数。

首先我们使用 rxjava 来修改 jdk8stream 中过滤 Person 对象打印名称的例子：

```Java
    public static void main(String[] args) {

        //1.创建 person 列表
        List<Person> personList = makeList();

        //2.执行过滤与输出
        Flowable.fromArray(personList.toArray(new Person[0]))//2.1 转换列表为 Flowable 流对象
            .filter(person->person.getAge()>=10)//2.2 过滤
            .map(person->person.getName())//2.3.映射转换
            .subscribe(System.out::println);//2.4 订阅输出
    }
```

如上代码 2.1 首先转换 personList 列表为流对象，然后执行代码 2.2 对符合条件的 person 进行过滤，然后 2.3 转换 person 对象为 name，代码 2.4 输出过滤后的 person 的 name 字段。可知上述操作与 stream 相比更加简洁。

另外与 Stream 类似，这里如果只执行代码 2.2 与代码 2.3 则什么都不会执行，数据流不会进行流程，而必须执行代码 2.4subscribe 时候(相当于执行了 JDK8 Stream 中的终端操作符时候)数据流才会流转到不同操作符进行处理。

在 rxjava 中每个操作符返回的都是一个添加了新功能更的流对象，其实上面的代码 2 等价于：

```java
        Flowable<Person> source = Flowable.fromArray(personList.toArray(new Person[0]));
        Flowable<Person> filterSource = source.filter(person->person.getAge()>=10);
        Flowable<String> nameSource = filterSource.map(person->person.getName());
        nameSource.subscribe(System.out::println);
```

下面我们先看看如何使用 rxjava 来同步顺序调用：

```Java
public class AsyncRpcCall4 {

    public static String rpcCall(String ip, String param) {

        System.out.println(Thread.currentThread().getName() + " " +ip + " rpcCall:" + param);
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }

        return param;

    }

    public static void main(String[] args) {

        // 1.生成 ip 列表
        List<String> ipList = new ArrayList<String>();
        for (int i = 1; i <= 10; ++i) {
            ipList.add("192.168.0." + i);
        }

        // 2.顺序调用
        long start = System.currentTimeMillis();
        Flowable.fromArray(ipList.toArray(new String[0]))
           .map(v -> rpcCall(v, v))
           .subscribe(System.out::println);

        // 3.打印耗时
        System.out.println("cost:" + (System.currentTimeMillis() - start));
    }
}
```

- 如上代码 2 中使用 Flowable.fromArray 方法把 ipList 列表元素转换为了 Flowable 流对象，这里 Flowable 流对象里面的元素就是 ip 地址
- 代码 2.1 使用 map 操作符把流中的每个 ip 地址转换为 rpcCall 调用的结果后返回一个新的 Flowable 流对象（新的 Flowable 流里面元素为调用 rpcCall 的结果），
- 然后代码 2.2 订阅新的 Flowable 流，并设置回调函数，当接受到元素后打印元素内容。

运行上面代码会发现耗时为 20s 左右，这是因为上述代码每次调用 rpcCall 方法都是同步顺序进行的，并且调用线程都是 main 函数所在线程。

下面我们先使用 observeOn 方法来让 rpcCall 的执行由 main 函数所在线程切换到 IO 线程，以便让 main 函数所在线程及时释放出来:

```Java
    public static void main(String[] args) {

        // 1.生成 ip 列表
        List<String> ipList = new ArrayList<String>();
        for (int i = 1; i <= 10; ++i) {
            ipList.add("192.168.0." + i);
        }

        // 2.顺序调用
        long start = System.currentTimeMillis();
        Flowable.fromArray(ipList.toArray(new String[0]))
           .observeOn(Schedulers.io())//2.1 切换到 IO 线程执行
           .map(v -> rpcCall(v, v))//2.2 映射结果
           .subscribe(System.out::println);//2.3 订阅

        // 3.打印耗时
        System.out.println("cost:" + (System.currentTimeMillis() - start));
    }
```

- 如上代码 2.1 使用 observeOn 让 rpcCall 的执行由 main 函数所在线程，切换到了 IO 线程。
- 运行上面代码可知不等 10 次 rpc 调用全部执行完毕，main 函数就退出了，这是因为 IO 线程是 deamon 线程（可以参考我的：Java 并发编程之美 一书），而 JVM 退出的条件是当前没有用户线程存在，而现在唯一的用户线程（main 函数所在线程已经退出了），所以 jvm 就退出了；所以我们需要在 main 函数所在线程挂起：

```Java
    public static void main(String[] args) throws InterruptedException {

        // 1.生成 ip 列表
        List<String> ipList = new ArrayList<String>();
        for (int i = 1; i <= 10; ++i) {
            ipList.add("192.168.0." + i);
        }

        // 2.顺序调用
        Flowable.fromArray(ipList.toArray(new String[0])).observeOn(Schedulers.io())// 2.1 切换到 IO 线程执行
                .map(v -> rpcCall(v, v))// 2.2 映射结果
                .subscribe(System.out::println);// 2.3 订阅

        //3.
        System.out.println("main execute over and wait");
        Thread.currentThread().join();// 挂起 main 函数所在线程
    }
```

如上代码 3 我们挂起了 main 函数所在线程，上面代码运行时候 main 函数所在线程会马上从代码 2 返回，然后执行代码 3 输出打印，然后挂起自己；然后具体的 10 次 rpc 调用时在 IO 线程内执行的，到这里我们释放了 main 函数所在线程来执行 rpc 调用，但是 IO 线程内 10 个 rpc 调用还是顺序执行的，在讲解如何使用 flatmap 操作符，让 10 个 rpc 调用顺序调用转换为异步并发调用前，我们先看看另外一个操作符 subscribeOn 是如何当发射元素的线程执行比较耗时的操作时候切换为异步执行，首先我们看下下面代码：

```Java
    public static void main(String[] args) throws InterruptedException {

        //1.
        long start = System.currentTimeMillis();
        Flowable.fromCallable(() -> {//1.1
            Thread.sleep(1000); // 1.2 模拟耗时的操作
            return "Done";
        }).observeOn(Schedulers.single())//1.3
          .subscribe(System.out::println, Throwable::printStackTrace);//1.4

        //2.
        System.out.println("cost:" + (System.currentTimeMillis() - start));

        //3.
        Thread.sleep(2000); // 等待流结束
    }
```

如上代码这里代码 1.3 使用 observeOn 方法让接受元素和处理元素的逻辑从 main 函数所在线程切换为其他线程，意图希望之星完毕代码 1.1 后 main 函数所在线程会马上返回，但是实际运行上述代码后会发现代码 2 输出耗时 1s 左右。

这是因为虽然我们让接受元素的逻辑异步化了，但是发射元素的逻辑还是同步调用的，这里代码 1.1 和代码 1.2 中的休眠 1s 然后返回"Done"的操作其实还是 main 函数所在线程来处理的，所以我们还需要让发射元素的逻辑异步化，而 subscribeOn 就是做这个事情的，修改上面代码如下：

```Java
    public static void main(String[] args) throws InterruptedException {

        //1.
        long start = System.currentTimeMillis();
        Flowable.fromCallable(() -> {//1.1
            Thread.sleep(1000); // 1.2 模拟耗时的操作
            return "Done";
        }).subscribeOn(Schedulers.io())//1.3
          .observeOn(Schedulers.single())//1.4
          .subscribe(System.out::println, Throwable::printStackTrace);//1.5

        //2.
        System.out.println("cost:" + (System.currentTimeMillis() - start));

        //3.
        Thread.sleep(2000); // 等待流结束
    }
```

如上代码 1.3 使用 subscribeOn 方法让发射元素的逻辑从 main 函数所在线程切换到了 IO 线程，在运行上面代码会发现代码 2 耗时远远小于 1s 这是因为代码 1 中的流中的元素发射与接收操作全部都异步化了。

这里总结下，默认情况下被观察对象与其上施加的操作符链的运行以及把运行结果通知给观察者对象使用的是调用 subscribe 方法所在的线程，SubscribeOn 操作符可以通过设置 Scheduler 来改变这个行为，让上面操作切换到其他线程来执行；ObserveOn 操作符则可以指定一个不同的 Scheduler 让被观察者对象使用其他线程来把结果通知给观察者对象。

### Spring 框架提供的异步执行能力

Spring Framework 分别使用 TaskExecutor 和 TaskScheduler 接口提供异步执行和任务调度的抽象。 Spring 还具有支持线程池或在应用程序服务器环境中委托给 CommonJ 的接口的实现。最终，在公共接口背后使用这些实现抽象出了 Java SE 5，Java SE 6 和 Java EE 环境之间的差异。本节我们着重讲解@Async 如何实现异步处理。

可以在方法上添加@Async 注释，以便异步调用该方法。换句话说，调用者将在调用时立即返回，并且该方法的实际执行将发生在 Spring TaskExecutor 中。

```Java
    @Async
    public void dosomthingAsync() {

        System.out.println("--dosomthingAsync begin---");
        // 模拟异步处理
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("--dosomthingAsync end---");
    }
```

如上代码在方法 dosomthingAsync 上添加了@Async 的注解，所以当我们调用 dosomthingAsync 方法时候，该方法会马上返回。

使用@Async 可以有返回值，因为它们将在运行时由调用者以“正常”方式调用，而不是由容器管理的调度任务调用。例如，以下是@Async 注解的合法应用程序：

```Java
    @Component
public class AsyncTask {
...
    @Async
    public CompletableFuture<String> dosomthingAsyncFuture() {

        System.out.println("--dosomthingAsync begin---");
        CompletableFuture<String> future = new CompletableFuture<String>();

        // 模拟异步处理
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        future.complete("ok");
        System.out.println("--dosomthingAsync end---");

        return future;
    }
}
```

如上代码调用该方法后，该方法会马上返回一个 CompletableFuture 对象，如果你一直持有这个 CompletableFuture 对象，那么等 dosomthingAsyncFuture 内业务处理异步处理完毕后，就可以从 dosomthingAsyncFuture 的 get()方法获取到执行结果。那么 Spring 框架是如何做到我们调用 dosomthingAsyncFuture 时候会马上返回一个 CompletableFuture 那？其实其对该类进行了代理，经过代理后的上面的方法类似于：

```Java
public class AsynTaskProxy {

    public AsyncTask getAsyncTask() {
        return asyncTask;
    }

    public void setAsyncTask(AsyncTask asyncTask) {
        this.asyncTask = asyncTask;
    }

    private AsyncTask asyncTask;

    private TaskExecutor executor = new SimpleAsyncTaskExecutor();

    public CompletableFuture<String> dosomthingAsyncFuture() {

        return CompletableFuture.supplyAsync(new Supplier<String>() {

            @Override
            public String get() {
                try {
                    return asyncTask.dosomthingAsyncFuture().get();
                } catch (Throwable e) {
                    throw new CompletionException(e);
                }
            }
        });
    }
}
```

Spring 会对 AsyncTask 类使用类似的 AsynTaskProxy 进行代理，并且会把 AsynTask 的实例注入到 AsynTaskProxy 内部，当我们调用 AsynTask 的 dosomthingAsyncFuture 方法时候，实际调用的是 AsynTaskProxy 的 dosomthingAsyncFuture 方法，后者则使用 CompletableFuture.supplyAsync 开启了一个异步任务（其马上返回一个 CompletableFuture 对象），并且使用默认的 SimpleAsyncTaskExecutor 线程池做为异步处理线程，然后异步任务内在具体调用了 AsyncTask 实例的 dosomthingAsyncFuture 方法，并且在返回的 future 上获取执行结果。

另外默认是使用 Cglib 进行的代理，具体拦截器是 AsyncExecutionInterceptor，感兴趣的童鞋可以自己去看下其 invoke 方法代码。

需要注意的是该注解默认是不会解析的，需要加上@EnableAsync 来启动。

### Servlet3.0 的异步处理

Web 应用程序中异步性的最基本动机是处理需要更长时间才能完成的请求。可能是一个缓慢的数据库查询，对外部 REST API 的调用，或其他一些 I / O 绑定操作。这种较长的请求可能会快速耗尽 Servlet 容器线程池并影响可伸缩性。

在某些情况下，您可以在后台作业完成处理时立即返回客户端。例如，发送电子邮件，启动数据库作业，以及其他代表可以使用 Spring 的@Async 支持。

在其他需要异步结果的情况下，我们需要将处理与 Servlet 容器线程分离，否则我们将耗尽其线程池。 Servlet 3 提供了这样的支持，其中 Servlet 可以指示在退出 Servlet 容器线程后响应应该保持打开状态。

为此，Servlet 3 Web 应用程序可以调用 request.startAsync（）并使用返回的 AsyncContext 继续写入来自其他单独线程的响应。

在 Servlet3.0 规范前，Servlet 容器对 Servlet 的处理都是每个请求对应一个线程这种 1：1 的模式进行处理的,如下图（本文 Servlet 容器固定使用 tomcat）： ![在这里插入图片描述](https://images.gitbook.cn/7ab671e0-bf40-11e9-892f-eb7f9c44f94b)

上图中每当用户发起一个请求的时候，tomcat 容器中就会分配一个线程来运行具体的 servlet,这种模式下当 serlvet 内执行比较耗时的操作，比如访问了数据库、同步调用了远程 rpc、或者进行了比较耗时的计算时候当前分配给 servlet 执行任务的内部线程会一直被该 servlet 持有，并不能及时释放掉后供其他请求使用，而 tomcat 内的线程池内线程是有限的，当线程池内线程用尽后就不能再对新来的请求进行及时处理了，所以这大大限制了服务器能提供的并发请求数量。

为了解决上面问题，在 Servlet3.0 规范中引入了异步处理请求的能力，处理线程可以及时返回容器并执行其他任务，一个典型的序列异步处理的事件是： 1.请求被接收然后从 Servlet 容器(例如 tomcat)中获取一个线程来执行，请求被流转到 Filter 链进行处理，然后查找具体的 serlvet 进行处理。 3.servlet 具体处理请求参数或者请求内容来决定请求的性质 4.servlet 内开启异步线程（可以是 tomcat 容器中的其他线程也可以是业务自己创建的线程）对请求进行具体处理，这可能会发起一个远程 rpc 调用或者发起一个数据库请求；开启异步线程后，当前 servlet 就返回了，并且不对请求方产生响应结果。 5.异步线程对请求处理完毕后，会通过持有的 AsyncContext 对象把结果写回请求方。

![在这里插入图片描述](https://images.gitbook.cn/84d2a400-bf40-11e9-b6fb-e7e4831475f3)

从上述流程可知具体处理请求响应的逻辑已经不在是 Servlet 调用线程来做了，Servlet 内开启异步处理后马上就释放了 tomcat 容器线程,具体对请求进行处理与响应的是业务线程池中线程。

下面我们看在 SpringBoot 中新增一个 Servlet，如何设置其为异步处理，首先我们看一个同步处理的代码：

```Java
//1.标识为 Servlet
@WebServlet(urlPatterns = "/test")
public class MyServlet extends HttpServlet {
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        System.out.println("---begin serlvet----");
        try { 
            // 2.执行业务逻辑
            Thread.sleep(3000);

            // 3.设置响应结果
            resp.setContentType("text/html");
            PrintWriter out = resp.getWriter();
            out.println("<html>");
            out.println("<head>");
            out.println("<title>Hello World</title>");
            out.println("</head>");
            out.println("<body>");
            out.println("<h1>welcome this is my servlet1!!!</h1>");
            out.println("</body>");
            out.println("</html>");

        } catch (Exception e) {
            System.out.println(e.getLocalizedMessage());
        } finally {
        }
        // 4.运行结束，即将释放容器线程
        System.out.println("---end serlvet----");
    }
}
```

如上代码是一个典型的 Servlet，当我们访问 http://127.0.0.1:8080/test 的时候，tomcat 容器会接受到该请求，然后从容器线程池中获取到一个线程来激活容器中的 Filter 链，然后把请求路由到 MyServlet，然后 MyServlet 的 Service 方法会被调用，方法内线程休眠 3s 用来模拟 MyServlet 中的耗时操作，然后代码 3 把响应结果设置到响应对象，然后该 MyServlet 就退出了；由于 MyServlet 内是同步执行，所以从 Filter 链的执行到 MyServlet 的 service 内代码执行都是使用的同一个 tomcat 容器内的线程。下面我们改造上面代码为异步处理：

```Java
//1.开启异步支持
@WebServlet(urlPatterns = "/test", asyncSupported = true)
public class MyServlet extends HttpServlet {
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        // 2.开启异步，获取异步上下文
        System.out.println("---begin serlvet----");
        final AsyncContext asyncContext = req.startAsync();

        // 3.提交异步任务
        asyncContext.start(new Runnable() {

            @Override
            public void run() {
                try {
                    // 3.1 执行业务逻辑
                    System.out.println("---async res begin----");
                    Thread.sleep(3000);

                    // 3.2 设置响应结果
                    resp.setContentType("text/html");
                    PrintWriter out = asyncContext.getResponse().getWriter();
                    out.println("<html>");
                    out.println("<head>");
                    out.println("<title>Hello World</title>");
                    out.println("</head>");
                    out.println("<body>");
                    out.println("<h1>welcome this is my servlet1!!!</h1>");
                    out.println("</body>");
                    out.println("</html>");
                    System.out.println("---async res end----");

                } catch (Exception e) {
                    System.out.println(e.getLocalizedMessage());
                } finally {
                    // 3.3 异步完成通知
                    asyncContext.complete();
                }
            }
        });

        // 4.运行结束，即将释放容器线程
        System.out.println("---end serlvet----");
    }
}
```

- 如上代码 1，这里使用注解@WebServlet 来标识 MyServlet 是一个 Servlet，然后其中 asyncSupported 为 true 代表要异步执行，然后框架就会知道该 Servlet 要启动异步处理功能。
- MyServlet 的 service 方法中代码 2 调用 HttpServletRequest 的 startAsync()方法开启异步调用，该方法返回一个 AsyncContext，其中保存了请求与响应相关的上下文信息。
- 代码 3 调用 AsyncContext 的 start 方法并传递一个任务，该方法会马上返回，然后代码 4 打印后，当前 Servlet 就退出了，然后其调用线程（容器线程）也就是释放了。
- 代码 3 提交异步任务后，异步任务模式还是由容器中的其他线程来具体执行，这里异步任务中代码 3.1 休眠 3s 是为了模拟耗时操作，然后代码 3.2 从 asyncContext 中获取响应对象，并把响应结果写入到响应对象；代码 3.3 则调用 asyncContext.complete()标识异步任务执行完毕。

上面代码的异步执行虽然及时释放了调用 Servlet 执行的容器线程，但是异步处理还是使用了容器中的其他线程，其实我们可以使用自己的线程池来进行任务的异步处理，上面代码修改如下：

```Java
//1.开启异步支持
@WebServlet(urlPatterns = "/test", asyncSupported = true)
public class MyServlet extends HttpServlet {

    // 0 自定义线程池
    private final static int AVALIABLE_PROCESSORS = Runtime.getRuntime().availableProcessors();
    private final static ThreadPoolExecutor POOL_EXECUTOR = new ThreadPoolExecutor(AVALIABLE_PROCESSORS,
            AVALIABLE_PROCESSORS * 2, 1, TimeUnit.MINUTES, new LinkedBlockingQueue<>(5),
            new ThreadPoolExecutor.CallerRunsPolicy());

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        // 2.开启异步，获取异步上下文
        System.out.println("---begin serlvet----");
        final AsyncContext asyncContext = req.startAsync();

        // 3.提交异步任务
        POOL_EXECUTOR.execute(new Runnable() {

            @Override
            public void run() {
                try {
                    // 3.1 执行业务逻辑
                    System.out.println("---async res begin----");
                    Thread.sleep(3000);

                    // 3.2 设置响应结果
                    resp.setContentType("text/html");
                    PrintWriter out = asyncContext.getResponse().getWriter();
                    out.println("<html>");
                    out.println("<head>");
                    out.println("<title>Hello World</title>");
                    out.println("</head>");
                    out.println("<body>");
                    out.println("<h1>welcome this is my servlet1!!!</h1>");
                    out.println("</body>");
                    out.println("</html>");
                    System.out.println("---async res end----");

                } catch (Exception e) {
                    System.out.println(e.getLocalizedMessage());
                } finally {
                    // 3.3 异步完成通知
                    asyncContext.complete();
                }
            }
        });

        // 4.运行结束，即将释放容器线程
        System.out.println("---end serlvet----");
    }
}
```

如上代码 0 我们创建了自己的 JVM 内全局的线程池，然后代码 3 我们把异步任务提交到了我们的线程池来执行，这时候整个处理流程是：tomcat 容器收到请求后，从容器中获取一个线程来执行 Filter 链，然后把请求同步转发到 MyServlet 的 service 方法来执行，然后代码 3 把具体请求处理的逻辑异步切换到我们业务线程池来执行，然后 MyServlet 就返回了，然后释放了容器线程。

### Servlet3.1 提供的非阻塞 IO

虽然 Servlet3.0 规范让 Servlet 的执行变为了异步，但是其 IO 还是阻塞式的，IO 阻塞是说在 Servlet 处理请求时候从 ServletInputStream 中读取请求体时候是阻塞的，而我们想要的是当数据已经就绪时候通知我们去读取就可以了，因为这可以避免占用我们自己的线程来进行阻塞读取,下面我们通过代码直观看看什么是阻塞 IO：

```Java
@WebServlet(urlPatterns = "/testSyncReadBody", asyncSupported = true)
public class MyServletSyncReadBody extends HttpServlet {

    // 1 自定义线程池
    private final static int AVALIABLE_PROCESSORS = Runtime.getRuntime().availableProcessors();
    private final static ThreadPoolExecutor POOL_EXECUTOR = new ThreadPoolExecutor(AVALIABLE_PROCESSORS,
            AVALIABLE_PROCESSORS * 2, 1, TimeUnit.MINUTES, new LinkedBlockingQueue<>(5),
            new ThreadPoolExecutor.CallerRunsPolicy());

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        // 2.开启异步，获取异步上下文
        System.out.println("---begin serlvet----");
        final AsyncContext asyncContext = req.startAsync();

        // 3.提交异步任务
        POOL_EXECUTOR.execute(new Runnable() {

            @Override
            public void run() {
                try {
                    System.out.println("---async res begin----");
                    // 3.1 读取请求 body
                    long start = System.currentTimeMillis();
                    final ServletInputStream inputStream = asyncContext.getRequest().getInputStream();
                    try {
                        byte buffer[] = new byte[1 * 1024];
                        int readBytes = 0;
                        int total = 0;

                        while ((readBytes = inputStream.read(buffer)) > 0) {
                            total += readBytes;
                        }

                        long cost = System.currentTimeMillis() - start;
                        System.out
                                .println(Thread.currentThread().getName() + " Read: " + total + " bytes,costs:" + cost);

                    } catch (IOException ex) {
                        System.out.println(ex.getLocalizedMessage());
                    }

                    // 3.2 执行业务逻辑
                    Thread.sleep(3000);

                    // 3.3 设置响应结果
                    resp.setContentType("text/html");
                    PrintWriter out = asyncContext.getResponse().getWriter();
                    out.println("<html>");
                    out.println("<head>");
                    out.println("<title>Hello World</title>");
                    out.println("</head>");
                    out.println("<body>");
                    out.println("<h1>welcome this is my servlet1!!!</h1>");
                    out.println("</body>");
                    out.println("</html>");
                    System.out.println("---async res end----");

                } catch (Exception e) {
                    System.out.println(e.getLocalizedMessage());
                } finally {
                    // 3.3 异步完成通知
                    asyncContext.complete();
                }
            }
        });

        // 4.运行结束，即将释放容器线程
        System.out.println("---end serlvet----");
    }
}
```

- 如上代码中 3.1 从 ServletInputStream 中读取 http 请求 body 的内容（需要注意的是 http header 的内容不再 ServletInputStream 中），其中使用循环来读取内如，并且统计读取数据的数量。
- 而 ServletInputStream 中并非一开始就有数据，所以当我们的业务线程池 POOL*EXECUTOR 中的线程调用 inputStream.read 方法时候是会被阻塞的,等内核接受到请求方发来的数据后，该方法才会返回，而这之前 POOL*EXECUTOR 中的线程会一直被阻塞，这就是我们所说的阻塞 IO，阻塞 IO 时候还是会耗掉宝贵的线程。

下面借助图来进一步解释下：![img](https://images.gitbook.cn/4c498170-bf2d-11e9-b6fb-e7e4831475f3)

- 如上图容器接受到请求后会从容器线程池获取一个线程来执行具体 servlet 的 service 方法，service 方法内调用 startAsync 把请求处理切换到了业务线程池内线程，业务线程内如果调用了 ServletInputStream 的 read 方法读取 http 的请求 body 内容则业务线程会阻塞方式读取 IO 数据，及时数据还没就绪。
- 这里问题是当数据还没就绪就分配了一个业务线程来阻塞等待数据就绪，这就有点浪费资源了，下面我们看看 Servlet3.1 如果让数据就绪时候才分配业务线程来进数据读取，一般做到需要时候才分配。

在 Servlet3.1 规范中提供了非阻塞 IO：Web 容器中的非阻塞请求处理有助于增加 Web 容器可同时处理请求的连接数量。servlet 容器的非阻塞 IO 允许开发人员在数据可用时读取数据或在数据可写时写数据。非阻塞 IO 对在 Servlet 和 Filter 中的异步请求处理有效。否则，当调用 ServletInputStream.setReadListener 或 ServletOutputStream.setWriteListener 方法时将抛出 IllegalStateException。基于内核的能力，servlet3.1 运行我们在 ServletInputStream 上通过函数 setReadListener 注册一个监听器，该监听器当内核发现有数据时候才会进行回调处理函数，上面代码注册监听后，代码如下：

```Java
@WebServlet(urlPatterns = "/testaSyncReadBody", asyncSupported = true)
public class MyServletaSyncReadBody extends HttpServlet {

    // 1 自定义线程池
    private final static int AVALIABLE_PROCESSORS = Runtime.getRuntime().availableProcessors();
    private final static ThreadPoolExecutor POOL_EXECUTOR = new ThreadPoolExecutor(AVALIABLE_PROCESSORS,
            AVALIABLE_PROCESSORS * 2, 1, TimeUnit.MINUTES, new LinkedBlockingQueue<>(5),
            new ThreadPoolExecutor.CallerRunsPolicy());

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        // 2.开启异步，获取异步上下文
        System.out.println("---begin serlvet----");
        final AsyncContext asyncContext = req.startAsync();

        // 3.设置数据就绪监听器
        final ServletInputStream inputStream = req.getInputStream();
        inputStream.setReadListener(new ReadListener() {

            @Override
            public void onError(Throwable throwable) {
                System.out.println("onError:" + throwable.getLocalizedMessage());
            }

            /**
             * 当数据就绪时候，通知我们来读取
             */
            @Override
            public void onDataAvailable() throws IOException {
                try {
                    // 3.1 读取请求 body
                    long start = System.currentTimeMillis();
                    final ServletInputStream inputStream = asyncContext.getRequest().getInputStream();
                    try {
                        byte buffer[] = new byte[1 * 1024];
                        int readBytes = 0;
                        while (inputStream.isReady() && !inputStream.isFinished()) {
                            readBytes += inputStream.read(buffer);

                        }

                        System.out.println(Thread.currentThread().getName() + " Read: " + readBytes);

                    } catch (IOException ex) {
                        System.out.println(ex.getLocalizedMessage());
                    }

                } catch (Exception e) {
                    System.out.println(e.getLocalizedMessage());
                } finally {
                }
            }

            /**
             * 当请求 body 的数据全部被读取完毕后，通知我们进行业务处理
             */
            @Override
            public void onAllDataRead() throws IOException {

                // 3.2 提交异步任务
                POOL_EXECUTOR.execute(new Runnable() {

                    @Override
                    public void run() {
                        try {

                            System.out.println("---async res begin----");
                            // 3.2.1 执行业务逻辑
                            Thread.sleep(3000);

                            // 3.2.2 设置响应结果
                            resp.setContentType("text/html");
                            PrintWriter out = asyncContext.getResponse().getWriter();
                            out.println("<html>");
                            out.println("<head>");
                            out.println("<title>Hello World</title>");
                            out.println("</head>");
                            out.println("<body>");
                            out.println("<h1>welcome this is my servlet1!!!</h1>");
                            out.println("</body>");
                            out.println("</html>");
                            System.out.println("---async res end----");

                        } catch (Exception e) {
                            System.out.println(e.getLocalizedMessage());
                        } finally {
                            // 3.2.3 异步完成通知
                            asyncContext.complete();
                        }
                    }
                });
            }
        });

        // 4.运行结束，即将释放容器线程
        System.out.println("---end serlvet----");
    }
}
```

- 如代码 3 设置了一个 ReadListener 到了 ServletInputStream 流，这样当内核发现有数据已经就绪时候，就会回调其 onDataAvailable 方法，该方法内就可以马上读取到数据，这里代码 3.1 通过 inputStream.isReady()发现数据已经 ready 后，就可以从中读取数据了，需要注意的是这里 onDataAvailable 的执行是容器线程来执行的，只是在数据已经就绪时候才调用容器线程来读取数据。
- 另外当请求 body 的数据全部读取完毕后会调用 onAllDataRead 方法，该方法默认也是容器线程来执行，这里我们使用代码 3.2 切换到业务线程池来执行了。

下面我们结合图来具体说明 Servlet3.1 中 ReadListener 如何高效利用线程的：![在这里插入图片描述](https://images.gitbook.cn/59522520-bf2d-11e9-b3a4-c1edb7e0fcb6)

- 如上图容器接受到请求后会从容器线程池获取一个线程来执行具体 servlet 的 service 方法，service 方法内调用 startAsync 把请求处理切换到了业务线程池内线程，然后通过 setReadListener 注册了一个 ReadListener 到 ServletInputStream，然后就释放了容器线程。
- 当内核发现 TCP 接受缓存有数据时候，会回调注册的 ReadListener 的 onDataAvailable 方法，这时候使用的是容器线程，然后我们可以选择在 onDataAvailable 方法内是否开启异步线程来对就绪数据进行读取，以便及时释放容器线程。
- 当发现 http 的 body 内容已经被读取完毕后，onAllDataRead 方法会被调用，然后 onAllDataRead 方法内我们使用业务线程池对请求进行处理，并把结果写回请求方。
- 这里可知无论是容器线程也好，业务线程也好，这里不是出现阻塞 IO 的情况，当线程被分配来进行处理时候，当前数据已经是就绪的，可以马上进行读取的，这不会造成线程的阻塞。

Servlet3.1 不仅增加了可以非阻塞读取请求 body 的 ReadListener，还增加了可以避免阻塞写的 WriteListener 接口，在 ServletOutputStream 上可以通过 setWriteListener 进行设置，当一个 WriteListener 注册到 ServletOutputStream 时，当可以写数据时 onWritePossible()方法将被容器首次调用，这里我们不在展开讨论。

### 最后

[《Java 并发编程之美》](https://item.jd.com/12450812.html)一书已经出版，对并发编程感兴趣的童鞋可以去购买学习下。