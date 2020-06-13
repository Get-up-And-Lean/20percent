# 14/20Windows 系统调试 Linux 程序——VisualGDB

VisualGDB 是一款 Visual Studio 插件，安装以后可以在 Windows 系统上利用 Visual Studio 强大的调试功能调试 Linux 程序。可能有读者会说，最新版的 Visual Studio 2015 或者 2017 不是自带了调试 Linux 程序的功能吗，为什么还要再装一款插件舍近而求远呢？很遗憾的是，经我测试 Visual Studio 2015 或者 2017 自带的调试程序，发现其功能很鸡肋，调试一些简单的 Linux 小程序还可以，调试复杂的或者多个源文件的 Linux 程序就力不从心了。

VisualGDB 是一款功能强大的商业软件，[点击这里详见官方网站](https://visualgdb.com/)。VisualGDB 本质上是利用 SSH 协议连接到远程 Linux 机器上，然后利用 Visual Studio 产生相应的 GDB 命令通过远程机器上的 gdbserver 传递给 GDB 调试器，其代码阅读功能建立在 samba 文件服务器之上。

利用这个工具远程调试 Linux 程序的方法有如下两种。

### 利用 VisualGDB 调试已经运行的程序

如果一个 Linux 程序已经运行，可以使用 VisualGDB 的远程 attach 功能。为了演示方便，我们将 Linux 机器上的 redis-server 运行起来：

```
[root@localhost src]# ./redis-server 
```

安装好 VisualGDB 插件以后，在 Visual Studio 的“Tools”菜单选择“Linux Source Cache Manager”菜单项，弹出如下对话框：

![img](https://images.gitbook.cn/b1d99410-f1f8-11e8-a886-5157ca7834b5)

单击 Add 按钮，配置成我们需要调试的 Linux 程序所在的机器地址、用户名和密码。

![img](https://images.gitbook.cn/d87eeb60-f1f8-11e8-9fa2-2b61cbf641db)

然后，在“Debug”菜单选择“Attach to Process…”菜单项，弹出 Attach To Process 对话框，Transport 类型选 VisualGDB，Qualifier 选择刚才我们配置的 Linux 主机信息。如果连接没有问题，会在下面的进程列表中弹出远程主机的进程列表，选择刚才启动的 redis-server，然后单击 Attach 按钮。

![img](https://images.gitbook.cn/f7341850-f1f8-11e8-a886-5157ca7834b5)

这样我们就可以在 Visual Studio 中调试这个 Linux 进程了。

![img](https://images.gitbook.cn/1d77a090-f1f9-11e8-a886-5157ca7834b5)

### 利用 VisualGDB 从头调试程序

更多的时候，我们需要从一个程序启动处，即 main() 函数处开始调试程序。在 Visual Studio 的“DEBUG”菜单选择“Quick Debug With GDB”菜单项，在弹出的对话框中配置好 Linux 程序所在的地址和目录：

![img](https://images.gitbook.cn/3bf65430-f1f9-11e8-9fa2-2b61cbf641db)

单击 Debug 按钮，就可以启动调试了。

![img](https://images.gitbook.cn/57b858d0-f1f9-11e8-b37f-7bcfd20d5d3a)

程序会自动停在 main() 函数处，这样我们就能利用强大的 Visual Studio 对 redis-server 进行调试。当然也可以在 VisualGDB 提供的 GDB Session 窗口直接输入 GDB 的原始命令进行调试。

![img](https://images.gitbook.cn/731b2670-f1f9-11e8-a886-5157ca7834b5)

没有工具是完美的，VisualGDB 也存在一些缺点，用这款工具调试 Linux 程序时可能会存在卡顿、延迟等现象。

卡顿和延迟的原因一般是由于调试远程 linux 机器上的程序时，网络不稳定导致的。如果读者调试程序所在的机器是本机虚拟机或局域网内的机器，一般不存在这个问题。

在笔者实际的使用过程中，VisualGDB 也存在一些缺点：

- VisualGDB 是一款商业软件，需要购买。当然互联网或许会有一些共享版本，有兴趣的读者可以找来学习一下。
- 由于 VisualGDB 是忠实地把用户的图形化操作通过网络最终转换为远程 linux 机器上的命令得到结果再图形化显示出来。GDB 本身对于一些代码的暂停处会存在定位不准确的问题，VisualGDB 也仍然存在这个问题。

### 扩展阅读

关于 GDB 的调试知识除了 GDB 自带的 Help 手册，国外还有一本权威的书籍 *Debugging with GDB：The The gnu Source-Level Debugger* 系统地介绍了 GDB 调试的方方面面，有兴趣的可以找来阅读一下，这里也给出书的下载链接，[请点击这里查看](https://pan.baidu.com/s/1J_JFpzrwRa-u684CZEmWKQ)。

![img](https://images.gitbook.cn/669156d0-fc59-11e8-bc5b-ef0caf885561)

GDB 调试对于 Linux C++ 开发以及阅读众多开源 C/C++ 项目是如此的重要，希望读者务必掌握它。掌握它的方法也很简单，找点程序尤其是多线程程序，实际调试一下，看看程序运行中的各种中间状态，很快就能熟悉绝大多数命令了。

关于 GDB 本身的知识，就这么多了。从下一课开始，我们将通过 GDB 正式调试 redis-server 和 redis-cli 的源码来分析其网络通信模块结构和实现思路。