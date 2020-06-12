# 2/20第02课 可重入锁与 Synchronized 的其他特性

上一节中基本介绍了进程和线程的区别、实现多线程的两种方式、线程安全的概念以及如何使用 Synchronized 实现线程安全。下边介绍一下关于 Synchronized 的其他基本特性。

### Synchronized 锁重入

1、关键字 Synchronized 拥有锁重入的功能，也就是在使用 Synchronized 的时候，当一个线程得到一个对象的锁后，在该锁里执行代码的时候可以再次请求该对象的锁时可以再次得到该对象的锁。

2、也就是说，当线程请求一个由其它线程持有的对象锁时，该线程会阻塞，而当线程请求由自己持有的对象锁时，如果该锁是重入锁，请求就会成功，否则阻塞。

3、一个简单的例子就是：在一个 Synchronized 修饰的方法，或代码块的内部调用本类的其他 Synchronized 修饰的方法或代码块时，永远可以得到锁，示例代码 A 如下：

```
public class SyncDubbo {

    public synchronized void method1() {
        System.out.println("method1-----");
        method2();
    }

    public synchronized void method2() {
        System.out.println("method2-----");
        method3();
    }

    public synchronized void method3() {
        System.out.println("method3-----");
    }

    public static void main(String[] args) {
        final SyncDubbo syncDubbo = new SyncDubbo();
        new Thread(new Runnable() {
            @Override
            public void run() {
                syncDubbo.method1();
            }
        }).start();
    }
}
```

执行结果：

```
method1-----
method2-----
method3-----
```

示例代码 A 向我们演示了，如何在一个已经被 Synchronized 关键字修饰过的方法再去调用对象中其他被 Synchronized 修饰的方法。

> [《Java 多线程编程核心技术》](https://gitbook.cn/gitchat/column/5a24fb14e3a13b7fc5933a44?utm_source=xlgsd001)

4、那么，为什么要引入可重入锁这种机制？

我们上一篇文章中介绍了一个“对象一把锁，多个对象多把锁”，可重入锁的概念就是：自己可以获取自己的内部锁。

假如有一个线程 T 获得了对象 A 的锁，那么该线程 T 如果在未释放前再次请求该对象的锁时，如果没有可重入锁的机制，是不会获取到锁的，这样的话就会出现死锁的情况。

就如代码 A 体现的那样，线程 T 在执行到`method1（）`内部的时候，由于该线程已经获取了该对象 syncDubbo 的对象锁，当执行到调用`method2（）` 的时候，会再次请求该对象的对象锁，如果没有可重入锁机制的话，由于该线程 T 还未释放在刚进入`method1（）` 时获取的对象锁，当执行到调用`method2（）` 的时候，就会出现死锁。

5、那么可重入锁到底有什么用呢？

正如上述代码 A 和第4条解释的那样，最大的作用是**避免死锁**。假如有一个场景：用户名和密码保存在本地 txt 文件中，则登录验证方法和更新密码方法都应该被加 synchronized，那么当更新密码的时候需要验证密码的合法性，所以需要调用验证方法，此时是可以调用的。

6、关于可重入锁的实现原理，是一个大论题，在这里篇幅有限不再学习，有兴趣可以移步至：[cnblogs](http://www.cnblogs.com/pureEve/p/6421273.html) 进行学习。

7、可重入锁的其他特性：父子可继承性

可重入锁支持在父子类继承的环境中，示例代码如下：

```
public class SyncDubbo {

    static class Main {
        public int i = 5;
        public synchronized void operationSup() {
            i--;
            System.out.println("Main print i =" + i);
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    static class Sub extends Main {
        public synchronized void operationSub() {
            while (i > 0) {
                i--;
                System.out.println("Sub print i = " + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) {
        new Thread(new Runnable() {
            public void run() {
                Sub sub = new Sub();
                sub.operationSub();
            }
        }).start();
    }
}
```

### Synchronized 的其他特性

- 出现异常时，锁自动释放

就是说，当一个线程执行的代码出现异常的时候，其所持有的锁会自动释放，示例如下：

```
public class SyncException {

    private int i = 0;

    public synchronized void operation() {
        while (true) {
            i++;
            System.out.println(Thread.currentThread().getName() + " , i= " + i);
            if (i == 10) {
                Integer.parseInt("a");
            }
        }
    }

    public static void main(String[] args) {
        final SyncException se = new SyncException();
        new Thread(new Runnable() {
            public void run() {
                se.operation();
            }
        }, "t1").start();
    }
}
```

执行结果如下：

```
t1 , i= 2
t1 , i= 3
t1 , i= 4
t1 , i= 5
t1 , i= 6
t1 , i= 7
t1 , i= 8
t1 , i= 9
t1 , i= 10
java.lang.NumberFormatException: For input string: "a"
    at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
    //其他输出信息
```

可以看出，当执行代码报错的时候，程序不会再执行，即释放了锁。

- 将任意对象作为监视器 monitor

  public class StringLock {

  `private String lock = "lock"; public void method() {    synchronized (lock) {        try {            System.out.println("当前线程： " + Thread.currentThread().getName() + "开始");            Thread.sleep(1000);            System.out.println("当前线程： " + Thread.currentThread().getName() + "结束");        } catch (InterruptedException e) {    } } } public static void main(String[] args) {    final StringLock stringLock = new StringLock();    new Thread(new Runnable() {        public void run() {            stringLock.method();        }    }, "t1").start();new Thread(new Runnable() {    public void run() {        stringLock.method();    } }, "t2").start(); } `

  }

执行结果：

```
当前线程： t1开始
当前线程： t1结束
当前线程： t2开始
当前线程： t2结束
```

- 单利模式-双重校验锁：

普通加锁的单利模式实现：

```
public class Singleton {

    private static Singleton instance = null; //懒汉模式
    //private static Singleton instance = new Singleton(); //饿汉模式

    private Singleton() {

    }

    public static synchronized Singleton newInstance() {
        if (null == instance) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

使用上述的方式可以实现多线程的情况下获取到正确的实例对象，但是每次访问`newInstance（）`方法都会进行加锁和解锁操作，也就是说该锁可能会成为系统的瓶颈，为了解决这个问题，有人提出了“双重校验锁”的方式，示例代码如下：

```
public class DubbleSingleton {

    private static DubbleSingleton instance;

    public static DubbleSingleton getInstance(){
        if(instance == null){
            try {
                //模拟初始化对象的准备时间...
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //类上加锁，表示当前对象不可以在其他线程的时候创建
            synchronized (DubbleSingleton.class) { 
                //如果不加这一层判断的话，这样的话每一个线程会得到一个实例
                //而不是所有的线程的到的是一个实例
                if(instance == null){ 
                    instance = new DubbleSingleton();
                }
            }
        }
        return instance;
    }
}
```

但是，需要注意的是，上述的代码是错误的写法，这是因为：**指令重排优化**，可能会导致初始化单利对象和将该对象地址赋值给 instance 字段的顺序与上面 Java 代码中书写的顺序不同。

例如：线程 A 在创建单例对象时，在构造方法被调用之前，就为该对象分配了内存空间并将对象设置为默认值。此时线程 A 就可以将分配的内存地址赋值给 instance 字段了，然而该对象可能还没有完成初始化操作。线程 B 来调用 newInstance() 方法，得到的 就是未初始化完全的单例对象，这就会导致系统出现异常行为。

为了解决上述的问题，可以使用`volatile`关键字进行修饰 instance 字段。volatile 关键字在这里的含义就是**禁止指令的重排序优化**（另一个作用是提供内存可见性），从而保证 instance 字段被初始化时，单例对象已经被完全初始化。

最终代码如下：

```
public class DubbleSingleton {

    private static volatile DubbleSingleton instance;

    public static DubbleSingleton getInstance(){
        if(instance == null){
            try {
                //模拟初始化对象的准备时间...
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //类上加锁，表示当前对象不可以在其他线程的时候创建
            synchronized (DubbleSingleton.class) { 
                //如果不加这一层判断的话，这样的话每一个线程会得到一个实例
                //而不是所有的线程的到的是一个实例
                if(instance == null){ 
                    instance = new DubbleSingleton();
                }
            }
        }
        return instance;
    }
}
```

那么问题来了，为什么 volatile 关键字可以实现禁止指令的重排序优化以及什么是指令重排序优化呢？

在 Java 内存模型中我们都是围绕着**原子性、有序性和可见性**进行讨论的。为了确保线程间的原子性、有序性和可见性，Java 中使用了一些特殊的关键字申明或者是特殊的操作来告诉虚拟机，在这个地方，要注意一下，不能随意变动优化目标指令。关键字 volatile 就是其中之一。

指令重排序是 JVM 为了优化指令，提高程序运行效率，在不影响单线程程序执行结果的前提下，尽可能地提高并行度（比如：将多条指定并行执行或者是调整指令的执行顺序）。编译器、处理器也遵循这样一个目标。注意是单线程。可想而知，多线程的情况下指令重排序就会给程序员带来问题。

最重要的一个问题就是程序执行的顺序可能会被调整，另一个问题是对修改的属性无法及时的通知其他线程，已达到所有线程操作该属性的可见性。

根据编译器的优化规则，如果不使用 volatile 关键字对变量进行修饰的，那么这个变量被修改后，其他线程可能并不会被通知到，甚至在别的想爱你城中，看到变量修改顺序都会是反的。一旦使用 volatile 关键字进行修饰的话，虚拟机就会特别小心的处理这种情况。

### volatile 与 synchronized 的区别

volatile 关键字的作用就是强制从公共堆栈中取得变量的值，而不是线程私有的数据栈中取得变量的值。

![enter image description here](http://images.gitbook.cn/b1541990-be1c-11e7-b678-1dcef838e98a)

1. 关键字 volatile 是线程同步的轻量级实现，性能比 synchronized 要好，并且 volatile 只能修于变量，而 synchronized 可以修饰方法，代码块等。
2. 多线程访问 volatile 不会发生阻塞，而 synchronized 会发生阻塞。
3. 可以保证数据的可见性，但不可以保证原子性，而 synchronized 可以保证原子性，也可以间接保证可见性，因为他会将私有内存和公共内存中的数据做同步。
4. volatile 解决的是变量在多个线程之间的可见性，而 synchronized 解决的是多个线程之间访问资源的同步性。

### volatile 的使用

很不幸的是，我个人比较熟悉的 MyBatis 框架没有用到任何 volatile 相关的知识，只能拿自己目前还不是很熟悉的 Spring 简单截图，不多做解释，示意图如下：

![enter image description here](http://images.gitbook.cn/2e9e89c0-d05c-11e7-9c94-71d522a5528f)

虽然部分内容看不懂，但是它确实用到了。因此，我们只有很清楚的了解 volatile 关键字的作用，以后再看 Spring 源代码的时候我们才可以很清楚的知道它的作用，而不是云里雾里，不知所云。

有一点我们还是可以看懂的，volatile 修饰的都是属性而不是方法，Spring 使用 volatile 来保证变量在多个线程之间的可见性！

然后，我们再看一道牛客网上的笔试题：

![enter image description here](http://images.gitbook.cn/be825440-d331-11e7-a545-ad02fd69db03)

出于运行速率的考虑，Java 编译器会把经常经常访问的变量放到缓存（严格讲应该是工作内存）中，读取变量则从缓存中读。但是在多线程编程中，内存中的值和缓存中的值可能会出现不一致。volatile 用于限定变量只能从内存中读取，保证对所有线程而言，值都是一致的。**但是 volatile 不能保证原子性，也就不能保证线程安全。** 因此答案就是 A。