# 3/20第03课 线程本地 ThreadLocal 的介绍与使用

### ThreadLocal 概述

我们通过上两篇的学习，我们已经知道了变量值的共享可以使用`public static`变量的形式，所有的线程都使用同一个被`public static`修饰的变量。

那么如果我们想实现每一个线程都有自己的共享变量该如何解决呢？JDK 提供的 ThreadLocal 正是为了解决这样的问题的。

ThreadLocal 主要解决的就是每个线程绑定自己的值，可以将 ThreadLocal 类比喻成全局存放数据的盒子，盒子中可以存储每个线程的私有变量。

先举个例子：

```
public class ThreadLocalDemo {

    public static ThreadLocal<List<String>> threadLocal = new ThreadLocal<>();

    public void setThreadLocal(List<String> values) {
        threadLocal.set(values);
    }

    public void getThreadLocal() {
        System.out.println(Thread.currentThread().getName());
        threadLocal.get().forEach(name -> System.out.println(name));
    }

    public static void main(String[] args) throws InterruptedException {

        final ThreadLocalDemo threadLocal = new ThreadLocalDemo();
        new Thread(() -> {
            List<String> params = new ArrayList<>(3);
            params.add("张三");
            params.add("李四");
            params.add("王五");
            threadLocal.setThreadLocal(params);
            threadLocal.getThreadLocal();
        }).start();

        new Thread(() -> {
            try {
                Thread.sleep(1000);
                List<String> params = new ArrayList<>(2);
                params.add("Chinese");
                params.add("English");
                threadLocal.setThreadLocal(params);
                threadLocal.getThreadLocal();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

运行结果：

```
Thread-0
张三
李四
王五
Thread-1
Chinese
English
```

可以，看出虽然多个线程对同一个变量进行访问，但是由于`threadLocal`变量由`ThreadLocal` 修饰，则不同的线程访问的就是该线程设置的值，这里也就体现出来`ThreadLocal`的作用。

当使用`ThreadLocal`维护变量时，`ThreadLocal`为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。

> [《Java 多线程编程核心技术》](https://gitbook.cn/gitchat/column/5a24fb14e3a13b7fc5933a44?utm_source=xlgsd001)

### ThreadLocal 与 Synchronized 同步机制的比较

在同步机制中，通过对象的锁机制保证同一时间只有一个线程访问变量。这时该变量是多个线程共享的，使用同步机制要求程序慎密地分析什么时候对变量进行读写，什么时候需要锁定某个对象，什么时候释放对象锁等繁杂的问题，程序设计和编写难度相对较大。

ThreadLocal 是线程局部变量，是一种多线程间并发访问变量的解决方案。和 Synchronized 等加锁的方式不同，ThreadLocal 完全不提供锁，而使用以空间换时间的方式，为每个线程提供变量的独立副本，以保证线程的安全。

### 如何实现一个简单的 ThreadLocal

```
public class SimpleThreadLocal<T> {

    /**
     * Key为线程对象，Value为传入的值对象
     */
    private static Map<Thread, T> valueMap = Collections.synchronizedMap(new HashMap<Thread, T>());

    /**
     * 设值
     * @param value Map键值对的value
     */
    public void set(T value) {
        valueMap.put(Thread.currentThread(), value);
    }

    /**
     * 取值
     * @return
     */
    public T get() {
        Thread currentThread = Thread.currentThread();
        //返回当前线程对应的变量
        T t = valueMap.get(currentThread);
        //如果当前线程在Map中不存在，则将当前线程存储到Map中
        if (t == null && !valueMap.containsKey(currentThread)) {
            t = initialValue();
            valueMap.put(currentThread, t);
        }
        return t;
    }

    public void remove() {
        valueMap.remove(Thread.currentThread());
    }

    public T initialValue() {
        return null;
    }

    public static void main(String[] args) {

        SimpleThreadLocal<List<String>> threadLocal = new SimpleThreadLocal<>();

        new Thread(() -> {
            List<String> params = new ArrayList<>(3);
            params.add("张三");
            params.add("李四");
            params.add("王五");
            threadLocal.set(params);
            System.out.println(Thread.currentThread().getName());
            threadLocal.get().forEach(param -> System.out.println(param));
        }).start();

        new Thread(() -> {
            try {
                Thread.sleep(1000);
                List<String> params = new ArrayList<>(2);
                params.add("Chinese");
                params.add("English");
                threadLocal.set(params);
                System.out.println(Thread.currentThread().getName());
                threadLocal.get().forEach(param -> System.out.println(param));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
} 
```

运行结果：

![enter image description here](https://images.gitbook.cn/10ce7eb0-f37d-11e8-938c-d1ce1f759181)

虽然上面的代码清单中的这个 ThreadLocal 实现版本显得比较简单粗糙，但其目的主要在于呈现 JDK 中所提供的 ThreadLocal 类在实现上的思路。

关于如何设计 ThreadLocal 的思路以及其原理会在后文中详细介绍，这里只做一个简单的预热。

### ThreadLocal 的应用

#### MyBatis 的使用

![enter image description here](http://images.gitbook.cn/cfae1d50-d0d7-11e7-a17f-8b69bfba7264)

SqlSessionManager 类部分代码如下：

```
private ThreadLocal<SqlSession> localSqlSession = new ThreadLocal<SqlSession>();

@Override
public Connection getConnection() {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot get connection.  No managed session is started.");
    }
    return sqlSession.getConnection();
}

@Override
public void commit() {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot commit.  No managed session is started.");
    }
    sqlSession.commit();
}

@Override
public void rollback() {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot rollback.  No managed session is started.");
    }
    sqlSession.rollback();
}
```

![enter image description here](http://images.gitbook.cn/30f9a570-d0d8-11e7-a17f-8b69bfba7264)

从上图可能看出，在 MyBatis 中，SqlSessionManager 类不但实现了 SqlSession 接口，同时也实现了 SqlSessionFactory 接口。而我们平时使用到的最多的就是 DefaultSqlSession，实现了 SqlSession，SqlSession 接口的实现如下：

![enter image description here](http://images.gitbook.cn/71f4a020-d0d8-11e7-a17f-8b69bfba7264)

SqlSessionManager 的作用如下：

![enter image description here](http://images.gitbook.cn/0d274ed0-d0d9-11e7-a17f-8b69bfba7264)

1. SqlSessionFactoryBuilder 负责接收 mybatis-config.xml 的输入流，创建 DefaultSqlSessionFactory 实例。
2. DefaultSqlSessionFactory 实现了 SqlSessionFactory 接口。
3. SqlSessionManager 实现了 SqlSessionFactory 接口，又封装了 DefaultSqlSessionFactory。

拿出 SqlSessionManager 的一个方法 getConnection 解释一下：

```
private ThreadLocal<SqlSession> localSqlSession = new ThreadLocal<SqlSession>();

@Override
public Connection getConnection() {
    final SqlSession sqlSession = localSqlSession.get();
    if (sqlSession == null) {
        throw new SqlSessionException("Error:  Cannot get connection.  No managed session is started.");
    }
    return sqlSession.getConnection();
}
```

可以看出 localSqlSession 是一个 ThreadLocal 变量，是每一个线程私有的，当有一个线程请求获取 Connection 的时候，会首先获取当前线程 ThreadLocal 中的 SqlSession，然后由 SqlSession 获取 Connection 对象，一个 ThreadLocal 的简单使用。

#### 数据库主从复制时读写分离的 ThreadLocal 使用

其实，在 MyBatis 中对 ThreadLocal 的使用主要体现在数据库连接这块，我们不仅联想到我们在实现主从复制读写分离的时候，我们是否也是用到了 ThreadLocal，先看示例：

![enter image description here](http://images.gitbook.cn/d2a4c0b0-d0da-11e7-a17f-8b69bfba7264)

上述简单的实现了数据源的 Handler 类 DataSourceHandler，在下边的类中会实现读写数据库的切换：

![enter image description here](http://images.gitbook.cn/07c55e80-d0db-11e7-a17f-8b69bfba7264)

![enter image description here](http://images.gitbook.cn/22cf0fa0-d0db-11e7-a17f-8b69bfba7264)

![enter image description here](http://images.gitbook.cn/b49183a0-d0db-11e7-a17f-8b69bfba7264)

根据 AOP 切面编程获取方法类型，根据方法的类型判断是读库还是写库，如果是读库的话就为当前线程设置访问读库的数据库信息，详细数据库主从复制读写分离的 AOP 实现案例，可以参考代码：

https://gitee.com/xuliugen/aop-choose-db-demo

![enter image description here](http://images.gitbook.cn/ed259d50-d0db-11e7-a17f-8b69bfba7264)