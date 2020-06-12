# 1/20第01课 线程安全和锁 Synchronized 概念

### 进程与线程的概念

- 在传统的操作系统中，程序并不能独立运行，作为资源分配和独立运行的基本单位都是进程。

> 在未配置 OS 的系统中，程序的执行方式是顺序执行，即必须在一个程序执行完后，才允许另一个程序执行；在多道程序环境下，则允许多个程序并发执行。程序的这两种执行方式间有着显著的不同。也正是程序并发执行时的这种特征，才导致了在操作系统中引入进程的概念。
>
> 自从在 20 世纪 60 年代人们提出了进程的概念后，在 OS 中一直都是以进程作为能拥有资源和独立运行的基本单位的。直到 20 世纪 80 年代中期，人们又提出了比进程更小的能独立运行的基本单位——线程（Threads），试图用它来提高系统内程序并发执行的程度，从而可进一步提高系统的吞吐量。特别是在进入 20 世纪 90 年代后，多处理机系统得到迅速发展，线程能比进程更好地提高程序的并行执行程度，充分地发挥多处理机的优越性，因而在近几年所推出的多处理机 OS 中也都引入了线程，以改善 OS 的性能。
>
> —–以上摘自《计算机操作系统（第三版）》汤小丹等编著。

- 下图是来自某知乎用户的解释：

![enter image description here](http://images.gitbook.cn/98097fb0-3de3-11e8-a1a7-ab51e1c649fb)

通过上述可以大致了解，线程和进程是干什么的了，那么我们下边给进程和线程总结一下概念：

**进程**（Process）

计算机中的程序关于某数据集合上的一次运行活动，是系统进行资源分配和调度的基本单位，是操作系统结构的基础。在早期面向进程设计的计算机结构中，进程是程序的基本执行实体；在当代面向线程设计的计算机结构中，进程是线程的容器。程序是指令、数据及其组织形式的描述，进程是程序的实体。

> [《Java 多线程编程核心技术》](https://gitbook.cn/gitchat/column/5a24fb14e3a13b7fc5933a44?utm_source=xlgsd001)

**线程**

有时被称为**轻量级进程**（Lightweight Process，LWP），是程序执行流的最小单元。线程是程序中一个单一的顺序控制流程。进程内一个相对独立的、可调度的执行单元，是系统独立调度和分派 CPU 的基本单位指运行中的程序的调度单位。在单个程序中同时运行多个线程完成不同的工作，称为多线程。

### 进程和线程的关系

![enter image description here](http://images.gitbook.cn/d3f501c0-3de3-11e8-b7c0-431e07ab6b5c)

### 线程和进程各自有什么区别和优劣

1. 进程是资源分配的最小单位，线程是程序执行的最小单位。
2. 进程有自己的独立地址空间，每启动一个进程，系统就会为它分配地址空间，建立数据表来维护代码段、堆栈段和数据段，这种操作非常昂贵。而线程是共享进程中的数据的，使用相同的地址空间，因此 CPU 切换一个线程的花费远比进程要小很多，同时创建一个线程的开销也比进程要小很多，线程的上下文切换的性能消耗要小于进程。
3. 线程之间的通信更方便，同一进程下的线程共享全局变量、静态变量等数据，而进程之间的通信需要以通信的方式（IPC）进行。不过如何处理好同步与互斥是编写多线程程序的难点。
4. 但是多进程程序更健壮，多线程程序只要有一个线程死掉，整个进程也死掉了，而一个进程死掉并不会对另外一个进程造成影响，因为进程有自己独立的地址空间。

### 同步与异步

对于一次方法的调用来说，同步方法调用一旦开始，就必须等待该方法的调用返回，后续的方法才可以继续执行；异步的话，方法调用一旦开始，就可以立即返回，调用者可以执行后续的方法，这里的异步方法通常会在另一个线程里真实的执行，而不会妨碍当前线程的执行。

### 并行与并发

并发和并行是两个相对容易比较混淆的概念。他都可以表示在同一时间范围内有两个或多个任务同时在执行，但其在任务调度的时候还是有区别的，首先看下图：

并发任务执行过程：

![enter image description here](http://images.gitbook.cn/33d4e430-bea9-11e7-b8d1-0f780f5f89c9)

并行任务执行过程：

![enter image description here](http://images.gitbook.cn/619fe9f0-bea9-11e7-a4c4-09983f698a15)

从上图中可以看到，两个任务在执行的时候，并发是没有时间上的重叠的，两个任务是交替执行的，由于切换的非常快，对于外界调用者来说相当于同一时刻多个任务一起执行了；而并行可以看到时间上是由重叠的，也就是说并行才是真正意义上的同一时刻可以有多个任务同时执行。

### Java 实现多线程方式

1、继承 Thread，重写 run() 方法：

```
public class MyThread extends Thread {

    @Override
    public void run() {
        while (true) {
            System.out.println(this.currentThread().getName());
        }
    }

    public static void main(String[] args) {
        MyThread thread = new MyThread();
        thread.start(); //线程启动的正确方式
    }
}
```

输出结果：

```
Thread-0
Thread-0
Thread-0
...
```

另外，要明白启动线程的是 start() 方法而不是 run() 方法，如果用 run() 方法，那么他就是一个普通的方法执行了，也就是只是会执行一次，这也是一个常见的笔试面试题哦！

2、实现 Runable 接口：

```
public class MyRunnable implements Runnable {

    @Override
    public void run() {
        System.out.println("123");
    }

    public static void main(String[] args) {
        MyRunnable myRunnable = new MyRunnable();
        Thread thread = new Thread(myRunnable, "t1");
        thread.start();
    }
}
```

这里`Thread`和`Runnable`的关系是这样的：Thread 类本身实现了 Runnable 接口，并且持有 run 方法，但 Thread 类的 run 方法主体是空的，Thread 类的 run 方法通常是由子类的 run 方法重写，详细定义如下：

Runable 接口的定义：

```
public interface Runnable {
    public abstract void run();
}
```

Thread 类的定义：

```
public class Thread implements Runnable {
}
```

### 线程安全

线程安全概念：当多个线程访问某一个类（对象或方法）时，这个类始终能表现出正确的行为，那么这个类（对象或方法）就是线程安全的。

线程安全就是多线程访问时，采用了加锁机制，当一个线程访问该类的某个数据时，进行保护，其他线程不能进行访问，直到该线程读取完，释放了锁，其他线程才可使用。这样的话就不会出现数据不一致或者数据被污染的情况。 线程不安全就是不提供数据访问保护，有可能出现多个线程先后更改数据以至于所得到的数据是脏数据。这里的加锁机制常见的如：Synchronized

### Synchronized 修饰符

1、Synchronized：可以在任意对象及方法上加锁，而加锁的这段代码称为**互斥区**或**临界区**。

2、不使用 Synchronized 实例（代码 A）：

```
public class MyThread extends Thread {

    private int count = 5;

    @Override
    public void run() {
        count--;
        System.out.println(this.currentThread().getName() + " count:" + count);
    }

    public static void main(String[] args) {
        MyThread myThread = new MyThread();
        Thread thread1 = new Thread(myThread, "thread1");
        Thread thread2 = new Thread(myThread, "thread2");
        Thread thread3 = new Thread(myThread, "thread3");
        Thread thread4 = new Thread(myThread, "thread4");
        Thread thread5 = new Thread(myThread, "thread5");
        thread1.start();
        thread2.start();
        thread3.start();
        thread4.start();
        thread5.start();
    }
}
```

输出的一种结果如下：

```
thread3 count:2
thread4 count:1
thread1 count:2
thread2 count:3
thread5 count:0
```

可以看到，上述的结果是不正确的，这是因为，多个线程同时操作`run()`方法，对 count 进行修改，进而造成错误。

3、使用 Synchronized 实例（代码 B）：

```
public class MyThread extends Thread {

    private int count = 5;

    @Override
    public synchronized void run() {
        count--;
        System.out.println(this.currentThread().getName() + " count:" + count);
    }

    public static void main(String[] args) {
        MyThread myThread = new MyThread();
        Thread thread1 = new Thread(myThread, "thread1");
        Thread thread2 = new Thread(myThread, "thread2");
        Thread thread3 = new Thread(myThread, "thread3");
        Thread thread4 = new Thread(myThread, "thread4");
        Thread thread5 = new Thread(myThread, "thread5");
        thread1.start();
        thread2.start();
        thread3.start();
        thread4.start();
        thread5.start();
    }
}
```

输出结果：

```
thread1 count:4
thread2 count:3
thread3 count:2
thread5 count:1
thread4 count:0
```

可以看出代码 A 和代码 B 的区别就是在`run()`方法上加上了 Synchronized 修饰。

说明如下：

当多个线程访问 MyThread 的 run 方法的时候，如果使用了 Synchronized 修饰，那个多线程就会以排队的方式进行处理（这里排队是按照 CPU 分配的先后顺序而定的），一个线程想要执行 Synchronized 修饰的方法里的代码，首先是尝试获得锁，如果拿到锁，执行 Synchronized 代码体的内容，如果拿不到锁的话，这个线程就会不断的尝试获得这把锁，直到拿到为止，而且多个线程同时去竞争这把锁，也就是会出现锁竞争的问题。

### 一个对象有一把锁，多个线程多个锁！

何为，一个对象一把锁，多个线程多个锁！首先看一下下边的实例代码（代码 C）：

```
public class MultiThread {

    private int num = 200;

    public synchronized void printNum(String threadName, String tag) {
        if (tag.equals("a")) {
            num = num - 100;
            System.out.println(threadName + " tag a,set num over!");
        } else {
            num = num - 200;
            System.out.println(threadName + " tag b,set num over!");
        }
        System.out.println(threadName + " tag " + tag + ", num = " + num);
    }

    public static void main(String[] args) throws InterruptedException {
        final MultiThread multiThread1 = new MultiThread();
        final MultiThread multiThread2 = new MultiThread();

        new Thread(new Runnable() {
            public void run() {
                multiThread1.printNum("thread1", "a");
            }
        }).start()

        new Thread(new Runnable() {
            public void run() {
                multiThread2.printNum("thread2", "b");
            }
        }).start();
    }
}
```

输出结果：

```
thread1 tag a,set num over!
thread1 tag a, num = 100
thread2 tag b,set num over!
thread2 tag b, num = 0
```

可以看出，有两个对象：`multiThread1`和`multiThread2`，如果多个对象使用同一把锁的话，那么上述执行的结果就应该是：`thread2 tag b, num = -100`，因此，是每一个对象拥有该对象的锁的。

关键字`synchronized`取得的锁都是对象锁，而不是把一段代码或方法当做锁，所以上述实例代码 C 中哪个线程先执行`synchronized` 关键字的方法，那个线程就持有该方法所属对象的锁，两个对象，线程获得的就是两个不同对象的不同的锁，他们互不影响的。

那么，我们在正常的场景的时候，肯定是有一种情况的就是，所有的对象会对一个变量 count 进行操作，那么如何实现哪？很简单就是加 static，我们知道，用 static 修改的方法或者变量，在该类的所有对象是具有相同的引用的，这样的话，无论实例化多少对象，调用的都是一个方法，代码如下（代码 D）：

```
public class MultiThread {

    private static int num = 200;

    public static synchronized void printNum(String threadName, String tag) {
        if (tag.equals("a")) {
            num = num - 100;
            System.out.println(threadName + " tag a,set num over!");
        } else {
            num = num - 200;
            System.out.println(threadName + " tag b,set num over!");
        }
        System.out.println(threadName + " tag " + tag + ", num = " + num);
    }

    public static void main(String[] args) throws InterruptedException {
        final MultiThread multiThread1 = new MultiThread();
        final MultiThread multiThread2 = new MultiThread();

        new Thread(new Runnable() {
            public void run() {
                multiThread1.printNum("thread1", "a");
            }
        }).start();

        new Thread(new Runnable() {
            public void run() {
                multiThread2.printNum("thread2", "b");
            }
        }).start();
    }
}
```

输出结果：

```
thread1 tag a,set num over!
thread1 tag a, num = 100
thread2 tag b,set num over!
thread2 tag b, num = -100
```

可以看出，对变量和方法都加上了 static 修饰，就可以实现我们所需要的场景。同时也说明了，对于非静态 static 修饰的方法或变量，是一个对象一把锁的。

### 对象锁的同步和异步

- 同步：Synchronized

同步的概念就是共享，我们要知道“共享”这两个字，如果不是共享的资源，就没有必要进行同步，也就是没有必要进行加锁。

同步的目的就是为了线程的安全，其实对于线程的安全，需要满足两个最基本的特性：原子性和可见性。

- 异步：Synchronized

异步的概念就是独立，相互之间不受到任何制约，两者之间没有任何关系，这里的异步可以理解为多个线程之间不会竞争共享资源。

- 示例代码：

```
    public class MyObject {

        public void method() {
            System.out.println(Thread.currentThread().getName());
        }

        public static void main(String[] args) {
            final MyObject myObject = new MyObject();

            Thread t1 = new Thread(new Runnable() {
                public void run() {
                    myObject.method();
                }
            }, "t1");

            Thread t2 = new Thread(new Runnable() {
                public void run() {
                    myObject.method();
                }
            }, "t2");

            t1.start();
            t2.start();
        }
    }
```

上述代码中`method()`就是异步的方法。一方面，他不会出现对共享变量的修改，另一方面，无需保证访问该方法的线程安全性。

### Synchronized 应用

注意：

我们通常情况下使用的大多数框架 SSM、以及常见的多线程任务调度框架、大数据框架等，其中都大量的使用了Synchronized 这个最简单的实现多线程同步的关键字，后续文章也会以我们常用的框架为基础，介绍其中使用的到的多线程技术，但不会详细的介绍其实现的原理，多会以截图的形式展示出来，这里只是让大家认识到，不是多线程技术没有用，而是很多的框架都给我们在底层屏蔽掉了这些内容。

对于一个满足于码农世界的程序员来说，熟练的使用各种框架提供给我们的服务就基本可以了，但是如果想了解框架内部的实现原理，首先，多线程的学习就是我们首要的任务，这也是本次达人课的重点。关于框架内部使用到多线程技术的更多细节问题希望读者能借此专题的学习，自己把源码下载下来研究一下。

1、MyBatis 中的使用：

![enter image description here](http://images.gitbook.cn/f27add80-d057-11e7-a349-11523d5feb7e)

MyBatis 中对于数据库连接池的处理，我们肯定都知道是一个需要保证线程安全的东西，上图中就大致展示了 MyBatis 中我们常用的两种数据源对象：PooledDataSource 和 UnpooledDataSource。根据上述显示的部分方法可以看出都是用到了简单的 Synchronized 关键字。

```
//请求数据库连接的次数
protected long requestCount = 0; 

public synchronized long getRequestCount() {
    return requestCount;
}
```

上述是获取连接中请求的数量，因为在不同时刻，请求的数量可能会发生变化，但是在对这个`requestCount`进行访问的时候，要加锁，以保证在访问这个方法的时候，不会被其他线程修改，可以看出就是我们平常简单的使用。

```
//请求数据库连接的次数
protected long requestCount = 0;
//获取连接的累积时间
protected long accumulatedRequestTime = 0; 

public synchronized long getAverageRequestTime() {
    return requestCount == 0 ? 0 : accumulatedRequestTime / requestCount;
}
```

上述代码，展示了获取请求的平均时间。

```
protected void pushConnection(PooledConnection conn) throws SQLException {

    synchronized (state) { //同步
        //从activeConnections集合中移除该PooledConnection对象
        state.activeConnections.remove(conn);
        if (conn.isValid()) { //检测PooledConnection对象是否有效
            //检测空闲连接数是否以达到上限，以及PooledConnection是否为该连接池的连接
            if (state.idleConnections.size() < poolMaximumIdleConnections && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
                state.accumulatedCheckoutTime += conn.getCheckoutTime(); //累积checkout时长
                if (!conn.getRealConnection().getAutoCommit()) { //回滚提交事务
                    conn.getRealConnection().rollback();
                }
                //为返还连接创建新的PooledConnection对象，然后添加到idleConnections集合中
                PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
                state.idleConnections.add(newConn);
                newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
                newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());
                conn.invalidate(); //将原PooledConnection对象设置为无效
                if (log.isDebugEnabled()) {
                    log.debug("Returned connection " + newConn.getRealHashCode() + " to pool.");
                }
                state.notifyAll();
            } else { //空闲连接数以达到上限或PooledConnection对象并不属于该连接池
                state.accumulatedCheckoutTime += conn.getCheckoutTime(); //累积checkout时长
                if (!conn.getRealConnection().getAutoCommit()) {
                    conn.getRealConnection().rollback();
                }
                conn.getRealConnection().close(); //关闭真正的数据库连接
                if (log.isDebugEnabled()) {
                    log.debug("Closed connection " + conn.getRealHashCode() + ".");
                }
                conn.invalidate();
            }
        } else {
            if (log.isDebugEnabled()) {
                log.debug("A bad connection (" + conn.getRealHashCode() + ") attempted to return to the pool, discarding connection.");
            }
            state.badConnectionCount++;
        }
    }
}
```

既然是数据库连接池，肯定是需要把用完的连接扔到连接池中去，上述代码就是 MyBatis 中使用完的连接重新添加到连接池的过程，其中使用 synchronized 对 state 进行了同步操作，state 表示当前数据库的一个连接的状态。

上述的代码，看不懂不要紧，不是本课程的重点，但是我们应该很清楚的认识到，多线程实实在在使用到了。同样，如果我们面试的时候遇到了让我们手写数据库连接池的时候，我们应该也要考虑到多线程的情况下如何保证数据的正确性。