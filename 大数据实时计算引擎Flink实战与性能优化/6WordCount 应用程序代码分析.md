# 6WordCount 应用程序代码分析

我们已经将 WordCount 程序代码写好了并且也在 IDEA 中和 Flink UI 上运行了 Job，并且程序运行的结果都是正常的。

那么我们来分析一下这个 WordCount 程序代码：

1、创建好 StreamExecutionEnvironment（流程序的运行环境）

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
```

2、给流程序的运行环境设置全局的配置（从参数 args 获取）

```java
env.getConfig().setGlobalJobParameters(ParameterTool.fromArgs(args));
```

3、构建数据源，WORDS 是个字符串数组

```java
env.fromElements(WORDS)
```

4、将字符串进行分隔然后收集，组装后的数据格式是 (word、1)，1 代表 word 出现的次数为 1

```java
flatMap(new FlatMapFunction<String, Tuple2<String, Integer>>() {
    @Override
    public void flatMap(String value, Collector<Tuple2<String, Integer>> out) throws Exception {
        String[] splits = value.toLowerCase().split("\\W+");

        for (String split : splits) {
            if (split.length() > 0) {
                out.collect(new Tuple2<>(split, 1));
            }
        }
    }
})
```

5、根据 word 关键字进行分组（0 代表对第一个字段分组，也就是对 word 进行分组）

```java
keyBy(0)
```

6、对单个 word 进行计数操作

```java
reduce(new ReduceFunction<Tuple2<String, Integer>>() {
    @Override
    public Tuple2<String, Integer> reduce(Tuple2<String, Integer> value1, Tuple2<String, Integer> value2) throws Exception {
        return new Tuple2<>(value1.f0, value1.f1 + value2.f1);
    }
})
```

7、打印所有的数据流，格式是 (word，count)，count 代表 word 出现的次数

```java
print()
```

8、开始执行 Job

```java
env.execute("zhisheng —— word count streaming demo");
```

### 小结与反思

本节给大家介绍了 Maven 创建 Flink Job、IDEA 中创建 Flink 项目（详细描述了里面要注意的事情）、编写 WordCount 程序、IDEA 运行程序、在 Flink UI 运行程序、对 WordCount 程序每个步骤进行分析。

通过本小节，你接触了第一个 Flink 应用程序，也开启了 Flink 实战之旅。你有自己运行本节的代码去测试吗？动手测试的过程中有遇到什么问题吗？

本节涉及的代码地址：https://github.com/zhisheng17/flink-learning/tree/master/flink-learning-examples/src/main/java/com/zhisheng/examples/streaming/wordcount