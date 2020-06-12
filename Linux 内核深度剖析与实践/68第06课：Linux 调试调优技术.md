# 6/8第06课：Linux 调试调优技术

这一节我们一起看下工作中常常使用的调试技巧。调试是软件开发过程中必不可少的环节，对于嵌入式开发者来讲很多工作都是体现在调试上，有句话讲“程序不是写出来的，好程序是调出来的”，一点都不夸张。纵观来看调试可以分为硬件断点调试和软件断点调试。硬件调试需要 CPU 的支持，CPU 内部提供了两组寄存器用来存储设置的断点，但是这种场景决定于内部硬件设计与调试器和其他调试工具无关。本 Chat 主要讲软件层面的调试，也是开发者工作中最常用到的调试方法。 谈到软件，无疑其决定因素是操作系统，这里我们以 Linux 操作系统为例对内核态，用户态以及其调试工具和调试方法进行剖析。

### 内核态

先介绍下单步调试的方式，相信大部分工程师常用单步调试方式都会选择 GDB，GDB 是最常用的调试工具，信号是 GDB 实现断点功能的基础，用 GDB 向某个地址打断点的时候，实际上就是向该地址写入断点指令。当程序运行到这条指令后就会触发 SIGTRAP 信号，这时候 GDB 就会收到这个信号，根据程序停止在的位置查询断点表，要是发现在断点表里有此地址，则判定找到断点。下面我们先讲解下如何用 GDB 单步调试内核。

#### 内核调试的配置

用 GDB 调试内核的话首先需要内核支持 KGDB，配置如下：

![image](http://images.gitbook.cn/e3a16cd0-e6d1-11e7-9d41-e3e9a1666a2a)

配置成功后编译内核，然后修改 Uboot 启动参数以支持 KGDB 调试：

```
setenv bootargs 'console=ttyS0,115200n8 kgdboc=ttyS0,115200 kgdbwait …… nfsroot=……'
```

由 Uboot 启动，等到 KGDB 的位置时停下等待 GDB 的连接，如图：

![image](http://images.gitbook.cn/1966af10-e6d2-11e7-9cea-f9db13770de9)

然后用 GDB 调试 vmlinux（注意此时是交叉编译器的 GDB）：

```
$arm-none-linux-gnueabi-gdb vmlinux
```

顺利的话显示如下：

![image](http://images.gitbook.cn/4fbfcce0-e6d2-11e7-8a6f-6d6a9842276a)

接下来就和调试应用程序一样调试内核了。

#### Kdump 的调试方式

除了单步调试外还有一种场景是我们经常碰到的，在系统运行中 Linux 内核发生崩溃的时候。这时候可以通过 Kdump 方式收集内核崩溃前的内存，生成 vmcore 文件，通过 vmcore 文件诊断出内核崩溃的原因。其中 kexec 是实现 Kdump 机制的关键，它由两部分组成：一是内核空间的系统调用 kexec_load，负责在生产内核（production kernel 或 first kernel）启动时将捕获内核（capture kernel 或 sencond kernel）加载到指定地址。二是用户空间的工具 kexec-tools，他将捕获内核的地址传递给生产内核，从而在系统崩溃的时候能够找到捕获内核的地址并运行。没有 kexec 就没有 Kdump。先有 kexec 实现了在一个内核中可以启动另一个内核，才让 Kdump 有了用武之地。

下面附一张 Kdump 的实现原理，感兴趣的小伙伴可以进一步研究，这部分不是本 Chat 重点，下面让我们具体讲下如何使用 Kdump。

![image](http://images.gitbook.cn/8eee1de0-e6d2-11e7-a770-21a6515874de)

经常用到分析 vmcore 的工具就是 crash，掌握 crash 的技巧对于定位问题有着很重要的意义。配置 kdump 完成后当系统崩溃的时候在 /var/crash/ 当天日期/生成一个 vmcore 文件，下面就可以对 vmcore 文件进行分析。

```
crash vmlinuz-4.2.0-27-generic vmcore
进入 crash
crash>
进入 crash 环境后就可以运用 crash 命令进行调试，比如 bt、dis、struct 等。
```

### 用户态

按照上面讲解内核的顺序我们先讲解如何在用户态用 GDB 工具调试。其实在用户态有两种方式可以进入 GDB，一种是直接在命令上输入 GDB，然后再在 GDB 中用 file 命令加载要调试的程序，另外一种是直接在命令行上使用 GDB 程序名。

![image](http://images.gitbook.cn/23e30d20-e6d3-11e7-8a6f-6d6a9842276a)

进入 GDB 后，我们一起来看下是如何使用的。

- 设置断点：

![image](http://images.gitbook.cn/33d56dd0-e6d4-11e7-9cea-f9db13770de9)

- 查看断点处情况 (gdb) info b：

![image](http://images.gitbook.cn/62aa1b60-e6d4-11e7-9cea-f9db13770de9)

- 运行代码 (gdb) r：

![image](http://images.gitbook.cn/98eb4460-e6d4-11e7-a770-21a6515874de)

- 单步运行 (gdb) n：

![image](http://images.gitbook.cn/be994360-e6d4-11e7-9d41-e3e9a1666a2a)

- 程序继续运行 (gdb) c：

使程序继续往下运行，直到再次遇到断点或程序结束。

- (gdb) bt：

用来打印栈帧指针，也可以在该命令后加上要打印的栈帧指针的个数，查看程序执行到此时经过哪些函数呼叫的程序，程序调用堆栈是当前函数之前的所有已调用函数的列表。每个函数及其变量都被分配了一个帧，最近调用的函数在0号帧中。

- (gdb) frame 1 用于打印指定栈帧。
- (gdb) info reg 查看寄存器使用情况。
- (gdb) info stack 查看堆栈使用情况。
- 退出GDB (gdb) q。

### 总结

我们以 Linux 系统为例介绍了编译器 GCC，调试器 GDB 以及堆栈的调试方法和一些常用工具和基本命令，包括 run、info、bt、list 等。在面对工作中常碰到的调试技术掌握这些基本可以应付了，如果想获取更多的调试技巧请参考官方提供的 GDB 调试手册。

[参考资料1](https://www.ibm.com/developerworks/library/l-gdb/index.html?S_TACT=105AGX52&S_CMP=cn-a-l)、 [参考资料2](http://www.yolinux.com/TUTORIALS/GDB-Commands.html)、 [参考资料3](https://www.gnu.org/manual/gdb-4.17/gdb.html)。