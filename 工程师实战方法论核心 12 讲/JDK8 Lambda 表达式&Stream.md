# 

# JDK8 Lambda 表达式&Stream

### 前言

本文我们来探讨 JDK8 提供的一些新特性，包含：Lambda 表达式和 Stream。首先我们看下这些特性可以给我们带来哪些变化：

- 可以让我们编写出简单、干净、易读的代码——尤其是对于集合的操作；
- 可以让我们简单的使用并行计算提高性能；
- 可以让我们开发出简单不易出错的并发代码；
- 可以让我们更好的对问题进行建模。

### Lambda 表达式

#### 第一个Lambda 表达式

使用 Lambda 表达式前我们创建线程并启动的代码如下：

```java
        Thread htread2 = new Thread(new Runnable() {

            @Override
            public void run() {
                System.out.println("hello" + Thread.currentThread().getName());
            }
        });
        htread2.start();
```

现在使用 Lambda 表达式代码如下:

```java
        Thread thread = new Thread(() -> System.out.println("hello" + Thread.currentThread().getName()));
        thread.start();
```

上面代码的 run 方法内只有一行代码，如果 run 方法体内有多行的话加`{}`括号即可：

```java
        Thread thread = new Thread(() -> {
            System.out.println("hello" + Thread.currentThread().getName());
            System.out.println("hello" + Thread.currentThread().getName());

        });
        thread.start();
```

#### 函数式接口

函数式接口是指只定义了一个抽象方法的接口，比如 Java 中已有的一些接口，就是函数式接口：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/f68e354c4ed31ee36ba8ff8a171f9669.png) 图片出处：《Java 8 函数式编程》

现在我们知道了什么是函数式接口，那么其有什么作用那？Lambda 表达式允许你直接以内联的形式为函数式接口的抽象方法提供实现，并把整个表达式作为函数式接口的实例。

#### 函数描述符

函数描述符：用来描述 Lambda 和函数式接口的签名的符号。

() -> Void 代表 了参数列表为空，且返回 Void 的函数。这正是 Runnable 接口所代表的。举另一个例子，(Apple, Apple) -> int 代表接受两个 Apple 作为参数且返回 int 的函数。

#### 类型推断

```java
Map<String, Integer> oldWordCounts = new HashMap<String, Integer>();（1）

Map<String, Integer> diamondWordCounts = new HashMap<>();（2）
```

如上（1）明确指定了泛型参数类型，则不用进行类型推断，而（2）则没有指定泛型参数，则编译器可以根据变量 diamondWordCounts 的类型，自动推断出 HashMap 的泛型。

```java
void useHashmap(Map<String, String> values){} （3）

useHashmap(new HashMap<>());（4）
```

同样（4）根据代码（3）的函数签名，可以推断出代码（4）传递的 HashMap 的泛型参数。

```java
Predicate<Integer> atLeast5 = x -> x > 5;(5)

System.out.println(atLeast5.test(6));(6)
System.out.println(atLeast5.test(4));(7)
```

运行上面代码会输出 true false;

其中 Predicate 是函数式接口，其定义：

```java
public interface Predicate<T> {
        boolean test(T t);
        ...
}
```

代码 （5）定义了一个函数，其中 x>5 是 Lambda 表达式的主体（也就是 Predicate 中 test 方法的内联），代码（5）函数描述符就是有一个参数，并且返回值为 boolean，加上左侧 Predicate 的泛型参数为 Integer，可以推断出函数描述符的参数类型就是 Integer。

```java
        BinaryOperator<Long> addLongs = (x, y) -> x + y;
        System.out.println("addlong:" + addLongs.apply(4L, 5L));
public interface BinaryOperator<T>{
 T apply(T t, T u);
}
```

类型推断系统相当智能，但若信息不够，类型推断系统也无能为力:

```java
BinaryOperator add = (x, y) -> x + y;
```

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/ebf5c79c3a311f058cc76993ebba9808.png)

这是因为不指定泛型参数时候，默认是 Object 类型，而`+`符号不能进行两个 Object 类型的运算。

### Stream

#### 从外部迭代到内部迭代

如下代码是外部迭代方式：

```java
        int countNum = 0;
        for (Integer temp : list) {
            if(temp > 4) {
                System.out.println(temp);
                countNum++;
            }
        }
```

外部迭代的特点与示意图：

- 每次迭代集合类时，都需要写很多样板代码；
- 循环改造成并行方式运行也很麻烦,需要修改每个循环；
- 上述循环无法清楚表达想要传达的意思。

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/9dc31284b57ab1501ae60ed7c91e394c.png) 图片出处：《Java 8 函数式编程》

内部迭代代码实例：

```java
list.stream().filter(a->a>4).count();
```

内部迭代特点与示意图：

- 每次迭代，需要很少的代码；
- 修改为并行运行只需要将 Stream 修改为 parallelStream.list.parallelStream().filter(a->a>4).count()；
- 可以清楚的表达需求。

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/489ff31ef5497696ead5e43a18ade5f7.png) 图片出处：《Java 8 函数式编程》

#### 实现机制

Stream 中有好多操作符，如下图，按照类型分为中间操作与终端操作：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/66f300f396c78049db9711b28bdef5a4.png)

所谓中间操作也叫惰性求值，这样的操作返回值是一个 Stream 而不是真的求值结果:

```java
        list.stream().filter(a->a>4);
```

比如上面 filter 操作符，其返回的是一个 Stream 对象，并且是没有执行对 list 元素的遍历与过滤,可以使用下面代码进行验证：

```java
        Stream<Integer> stream = list.stream().filter(a -> {
            System.out.println("-----" + a);
            return a > 4;
        });
```

执行上面代码并不会打印输出，但是当在返回的 Stream 上调用 count 操作符时候则会打印输出`stream.count()`，因为 count 为终端操作，会激活整个操作符链。

整个过程和建造者模式有共通之处，建造者模式使用一系列操作设置属性和配置，最后调 用一个 build 方法，这时对象才被真正创建。

#### 常用操作

##### **map**

映射操作，将一个元素使用函数映射为另外一个元素，如下图使用函数将原来的正方形转换为了圆形：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/a448226b238b782e2ef56e70a0502836.png) 图片出处：《Java 8 函数式编程》

```java
        List<Integer> list = new ArrayList<>();
        list.add(1);
        list.add(2);

        list.stream().map(a->a*2).forEach(System.out::println);
```

如上代码将 list 里面的每个元素的值映射为原来值乘以 2，然后输出。

##### **collect(toList())**

收集元素为列表,如下图，Stream 里面的每个元素被收集为了一列表：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/6afc6ade9f7204f854123d2a78bbf61c.png)

```java
        List<Integer> list = new ArrayList<>();
        list.add(1);
        list.add(2);
        list = list.stream().map(a->a*2).collect(Collectors.toList());
```

如上代码，在把原来元素值都乘以2后，在把所有元素收集为一个 list。

collect 里面还有好多函数，求最大值，最小值，平均值，分组等等，很强大。

##### **filter**

根据表达式过滤元素，过滤后返回一个只含过滤后的元素的 Stream：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/e95ee35f9b7cd1e01e92cfc6ba064d50.png) 图片出处：《Java 8 函数式编程》

```java
        List<Integer> list = new ArrayList<>();
        list.add(1);
        list.add(2);

        list.stream().filter(a->a>1).forEach(System.out::println);
```

如上代码只会打印出元素值大于 1 额元素。

##### **flatMap**

可以将元素替换为 Stream，并将多个 Stream 合并为一个流：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/3db228fc47bfafd4fc4dc74dcb1a3656.png) 图片出处：《Java 8 函数式编程》

```java
        List<Integer> one = new ArrayList<>();
        one.add(1);
        one.add(2);

        List<Integer> two = new ArrayList<>();
        two.add(34);
        two.add(4);

        Stream.of(one, two)// 使用两个list作为元素，组成流
                .flatMap(list -> list.stream())// 转换每个list为Stream，并合并两个流为一个流
                .forEach(System.out::println);// 遍历所有元素
```

##### **reduce**

reduce 操作可以实现从一组值中生成一个值：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/d04e810c476dd5bf041b27e4bc8529bb.png) 图片出处：《Java 8 函数式编程》

```java
        List<Integer> list = new ArrayList<>();
        list.add(1);
        list.add(2);
        list.add(3);
        list.add(4);

        int total = list.stream().reduce(0,(a,b)->a+b);
        System.out.println(total);
```

如上代码求列表里面元素和。

还有其他好多操作，limit、skip 等等。

##### **操作符整合实例**

```java
        List<Integer> list1 = new ArrayList<>();
        list1.add(1);
        list1.add(2);

        List<Integer> list2 = new ArrayList<>();
        list2.add(3);
        list2.add(4);

        Stream<Integer> list1Stream = list1.stream().filter(a->a>1).map(a->a*2); //(1)
        Stream<Integer> list2Stream = list2.stream().filter(a->a>3).map(a->a*2);//(2)

        Optional<Integer> total = Stream.of(list1Stream,list2Stream).flatMap(a->a).map(a->a+1).reduce((accl,a)->accl+a);//(3)

        System.out.println(total.get()); 
```

如上代码 1 和代码 2 分别返回了一个 steam 这个是延迟计算操作，当执行完代码 1 和 2 后其实还没对列表 1 和列表 2 应用过滤和映射操作，只是简单返回 Stream 对象。

代码 1 首先过滤 list 里面小于等于 1 的值，过滤后列表里面还剩下 2 这个元素，然后对映射元素值为原来的值 *，这里代码 1 获取到一个含有元素之为 4 的 Stream。同理代码 2 获取到一个值为 8 个元素的 Stream。

代码 3 首先把 list 1 Stream，list 2 Stream 作为一个流的元素，然后通过 faltmap 操作合并另个流为一个流（流里面含有4、8两个元素），然后映射流里面值为原来值加 1（之后流里面元素值为5、9），其实到这里还是没有对列表元素进行遍历，只是构建操作符链，真正激活时刻是当调用了 reduce 操作（其为终端操作）。调 reduce 后，列表 1 的元素会顺序经过 1 的过滤和映射返回一个含有元素 4 的流，代码 2 返回一个含有元素 8 的流，然后经历 3 返回结果 14。

##### **并行流**

```java
        List<Integer> list = new ArrayList<>();
        IntStream.range(1, 100).forEach(a -> list.add(a));

        list.stream().filter(a -> a > 10).forEach(System.out::println);
```

如上代码创建了一个包含 100 元素的列表，然后过滤掉小于等于 10 的元素后，进行遍历。

要把上面遍历转换为并行流只需要：

```java
list.parallelStream().filter(a -> a > 10).forEach(System.out::println);
```

但是我们测算下两者耗时，却发现并行流比顺序流更耗时，你可以增加列表元素为 10000、100000 也是一样。其实并行流并不是银弹，不是所有情况下都比串行执行快，并行流底层是使用 fork-join 框架来实现的（它默认的 线程数量就是你的 CPU 核数量，这个值是由Runtime.getRuntime().available- Processors() 得到的），其思想是递归的把任务拆分为子任务进行执行，如下图：

![image.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/597d3d6866beb0753446c01a928cdf26.png) 图片出处：《Java 8 函数式编程》

其本质是分而治之思想，并行化并不是没有代价的，并行化过程本身需要对流做递划分，把每个子流的归并操作分配到不同的线程，然后把这些操作的结果合并成一个值。但在多个 CPU 之间移动数据的代价也可能比你想的要大，所以很重要的一点是要保 CPU 内并行执行工作的时间，比在 CPU 之间传输数据的时间长。

何时适合使用并行流：

- 设 N 是要处理的元素的总数，Q 是一个元素通过流水线的大致处理成本，则 N*Q 就是这个对成本的一个粗略的定性估计。Q 值较高就意味 着使用并行流时性能好的可能性比较大。

```java
        //并行流
        list.parallelStream().filter(a -> a > 10).forEach(a->{try {
            System.out.println(a);
            Thread.sleep(100);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }});

        //串行流
        list.stream().filter(a -> a > 10).forEach(a->{try {
            System.out.println(a);
            Thread.sleep(100);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }});
```

如上代码，在每个元素处理时候，让调用线程 sleep 100ms，会发现使用并行比较顺序好很多。

- 另外使用并行流，还要考虑流背后的数据结构是否易于分解，例如 ArrayList 的拆分效果比 LinkedList 高得多，因为前者用不着遍历就进行切分，而后者则必须遍历者进行切分（因为不支持随机访问）。
- 输入数据的大小会影响并行化处理对性能的提升。将问题分解之后并行化处理，再将结 果合并会带来额外的开销。因此只有数据足够大、每个数据处理管道花费的时间足够多 时，并行化处理才有意义。

在日常代码开发中，如果你们用了 JDK8 以及以上版本的话，建议新功能使用 Lambda 表达式和 Stream，另外流式编程已经是一种趋势，比如最近比较流行的反应式编程代表 RxJava。

### 参考文献

[1]Richard Warburton.Java8函数式编程[M].人民邮电出版社:北京,2015-3:1.

[2]Raoul-Gabriel Urma , Mario Fusco , Alan Mycroft.Java 8实战[M].人民邮电出版社:北京,2016-04-25:1.

[3]JDK9 Flow ：https://www.baeldung.com/java-9-reactive-streams

[4]RxJava ：https://github.com/ReactiveX/RxJava

[5]WebFlux ：https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html