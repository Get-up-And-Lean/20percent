# CompletableFuture教程

## TOC

[TOC]

## [使用CompletableFuture](https://www.liaoxuefeng.com/wiki/1252599548343744/1306581182447650)

使用`Future`获得异步执行结果时，要么调用阻塞方法`get()`，要么轮询看`isDone()`是否为`true`，这两种方法都不是很好，因为主线程也会被迫等待。

从Java 8开始引入了`CompletableFuture`，它针对`Future`做了改进，可以传入回调对象，当异步任务完成或者发生异常时，自动调用回调对象的回调方法。

我们以获取诊断结果为例，看看如何使用`CompletableFuture`：

```java
public class CompletableFutureTest2 {
        public static void main(String[] args) throws Exception {
            // 创建异步执行任务:
            CompletableFuture<String> cf = CompletableFuture.supplyAsync(CompletableFutureTest2::diagnosis);
            // 如果执行成功:
            cf.thenAccept((result) -> {
                System.out.println("诊断结果: " + result);
            });
            // 如果执行异常:
            cf.exceptionally((e) -> {
                e.printStackTrace();
                return null;
            });
            // 主线程不要立刻结束，否则CompletableFuture默认使用的线程池会立刻关闭:
            Thread.sleep(2000);
        }

        static String diagnosis() {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
            }
            if (Math.random() < 0.3) {
                throw new RuntimeException("fetch price failed!");
            }
            return "体温过高";
        }
}
```

创建一个`CompletableFuture`是通过`CompletableFuture.supplyAsync()`实现的，它需要一个实现了`Supplier`接口的对象：

```java
public interface Supplier<T> {
    T get();
}
```

这里我们用lambda语法简化了一下，直接传入`CompletableFutureTest2::diagnosis`，因为`CompletableFutureTest2::diagnosis`静态方法的签名符合`Supplier`接口的定义（除了方法名外）。

紧接着，`CompletableFuture`已经被提交给默认的线程池执行了，我们需要定义的是`CompletableFuture`完成时和异常时需要回调的实例。完成时，`CompletableFuture`会调用`Consumer`对象：

```java
public interface Consumer<T> {
    void accept(T t);
}
```

异常时，`CompletableFuture`会调用`Function`对象：

```java
public interface Function<T, R> {
    R apply(T t);
}
```

这里我们都用lambda语法简化了代码。

可见`CompletableFuture`的优点是：

- 异步任务结束时，会自动回调某个对象的方法；
- 异步任务出错时，会自动回调某个对象的方法；
- 主线程设置好回调后，不再关心异步任务的执行。

如果只是实现了异步回调机制，我们还看不出`CompletableFuture`相比`Future`的优势。`CompletableFuture`更强大的功能是，多个`CompletableFuture`可以串行执行，例如，定义两个`CompletableFuture`，第一个`CompletableFuture`根据温度查询诊断信息，第二个`CompletableFuture`根据诊断信息查询出措施，这两个`CompletableFuture`实现串行操作如下：

```java
public class CompletableFutureTest3 {
    public static void main(String[] args) throws Exception {
        // 第一个任务:
        CompletableFuture<String> cfQuery = CompletableFuture.supplyAsync(() -> diagnosis(38.0));
        // cfQuery成功后继续执行下一个任务:
        CompletableFuture<String> cfFetch = cfQuery.thenApplyAsync((result) -> fetchMeasures(result));
        // cfFetch成功后打印结果:
        cfFetch.thenAccept((result) -> {
            System.out.println("result: " + result);
        });
        // 主线程不要立刻结束，否则CompletableFuture默认使用的线程池会立刻关闭:
        Thread.sleep(2000);
    }

    static String diagnosis(Double temperature) {
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
        }
        if(temperature> 36.5 && temperature<37.5){
            return "正常";
        }else if(temperature < 36.5){
            return "体温过低";
        }else{
            return "体温过高";
        }
    }

    static String fetchMeasures(String result) {
        System.out.println("result is "+ result);
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
        }
        return "对应的措施";
    }
}
```

除了串行执行外，多个`CompletableFuture`还可以并行执行。例如，我们考虑这样的场景：

同时从多个诊断系统查询查询诊断结果，只要任意一个返回结果，就进一步查询措施，查询措施的同时从不同的诊断系统查询，只要任意一个返回结果，就完成操作：

