# 20/20[实战] Redis 通信协议

### 探究 redis-cli 端的网络通信模型

我们接着探究一下 Redis 源码自带的客户端 redis-cli 的网络通信模块。

使用 GDB 运行 redis-cli 以后，原来打算按 Ctrl + C 快捷键让 GDB 中断查看一下 redis-cli 跑起来有几个线程，但是实验之后发现，这样并不能让 GDB 中断下来，反而会导致 redis-cli 这个进程退出。

![img](https://images.gitbook.cn/b01ba050-f20c-11e8-a886-5157ca7834b5)

换个思路：直接运行 redis-cli ，然后使用“linux pstack 进程 id”来查看下 redis-cli 的线程数量。

```
[root@localhost ~]# ps -ef | grep redis-cli
root     35454 12877  0 14:51 pts/1    00:00:00 ./redis-cli
root     35468 33548  0 14:51 pts/5    00:00:00 grep --color=auto redis-cli
[root@localhost ~]# pstack 35454
#0  0x00007ff39e13a6e0 in __read_nocancel () from /lib64/libpthread.so.0
#1  0x000000000041e1f1 in linenoiseEdit (stdin_fd=0, stdout_fd=1, buf=0x7ffc2c9aa980 "", buflen=4096, prompt=0x839618 <config+184> "127.0.0.1:6379> ") at linenoise.c:800
#2  0x000000000041e806 in linenoiseRaw (buf=0x7ffc2c9aa980 "", buflen=4096, prompt=0x839618 <config+184> "127.0.0.1:6379> ") at linenoise.c:991
#3  0x000000000041ea1c in linenoise (prompt=0x839618 <config+184> "127.0.0.1:6379> ") at linenoise.c:1059
#4  0x000000000040f92e in repl () at redis-cli.c:1404
#5  0x00000000004135d5 in main (argc=0, argv=0x7ffc2c9abb10) at redis-cli.c:2975
```

通过上面的输出，发现 redis-cli 只有一个主线程，既然只有一个主线程，那么可以断定 redis-cli 中的发给 redis-server 的命令肯定都是同步的，这里同步的意思是发送命令后会一直等待服务器应答或者应答超时。

在 redis-cli 的 main 函数（位于文件 redis-cli.c 中）有这样一段代码：

```
/* Start interactive mode when no command is provided */
if (argc == 0 && !config.eval) {
    /* Ignore SIGPIPE in interactive mode to force a reconnect */
    signal(SIGPIPE, SIG_IGN);

    /* Note that in repl mode we don't abort on connection error.
    * A new attempt will be performed for every command send. */
    cliConnect(0);
    repl();
}
```

其中，cliConnect(0) 调用代码（位于 redis-cli.c 文件中）如下：

```
static int cliConnect(int force) {
    if (context == NULL || force) {
        if (context != NULL) {
            redisFree(context);
        }

        if (config.hostsocket == NULL) {
            context = redisConnect(config.hostip,config.hostport);
        } else {
            context = redisConnectUnix(config.hostsocket);
        }

        if (context->err) {
            fprintf(stderr,"Could not connect to Redis at ");
            if (config.hostsocket == NULL)
                fprintf(stderr,"%s:%d: %s\n",config.hostip,config.hostport,context->errstr);
            else
                fprintf(stderr,"%s: %s\n",config.hostsocket,context->errstr);
            redisFree(context);
            context = NULL;
            return REDIS_ERR;
        }

        /* Set aggressive KEEP_ALIVE socket option in the Redis context socket
         * in order to prevent timeouts caused by the execution of long
         * commands. At the same time this improves the detection of real
         * errors. */
        anetKeepAlive(NULL, context->fd, REDIS_CLI_KEEPALIVE_INTERVAL);

        /* Do AUTH and select the right DB. */
        if (cliAuth() != REDIS_OK)
            return REDIS_ERR;
        if (cliSelect() != REDIS_OK)
            return REDIS_ERR;
    }
    return REDIS_OK;
}
```

这个函数做的工作可以分为三步：

第一步，context = redisConnect(config.hostip,config.hostport);

第二步，cliAuth()

第三步，cliSelect()

先来看第一步 redisConnect 函数，这个函数实际又调用 _redisContextConnectTcp 函数，后者又调用 _redisContextConnectTcp 函数。_redisContextConnectTcp 函数是实际连接 redis-server 的地方，先调用 API getaddrinfo 解析传入进来的 IP 地址和端口号（我这里是 127.0.0.1 和 6379），然后创建 socket ，并将 socket 设置成非阻塞模式，接着调用 API connect 函数，由于 socket 是非阻塞模式，connect 函数会立即返回 −1 。

接着调用 redisContextWaitReady 函数，该函数中调用 API poll 检测连接的 socket 是否可写（ POLLOUT ），如果可写则表示连接 redis-server 成功。由于 _redisContextConnectTcp 代码较多，我们去掉一些无关代码，整理出关键逻辑的伪码如下（位于 net.c 文件中）：

```
static int _redisContextConnectTcp(redisContext *c, const char *addr, int port,
                                   const struct timeval *timeout,
                                   const char *source_addr) {
    //省略部分无关代码...    

    rv = getaddrinfo(c->tcp.host,_port,&hints,&servinfo)) != 0

    s = socket(p->ai_family,p->ai_socktype,p->ai_protocol)) == -1

    redisSetBlocking(c,0) != REDIS_OK

    connect(s,p->ai_addr,p->ai_addrlen)

    redisContextWaitReady(c,timeout_msec) != REDIS_OK

    return rv;  // Need to return REDIS_OK if alright
}
```

redisContextWaitReady 函数的代码（ 位于 net.c 文件中 ）如下：

```
static int redisContextWaitReady(redisContext *c, long msec) {
    struct pollfd   wfd[1];

    wfd[0].fd     = c->fd;
    wfd[0].events = POLLOUT;

    if (errno == EINPROGRESS) {
        int res;

        if ((res = poll(wfd, 1, msec)) == -1) {
            __redisSetErrorFromErrno(c, REDIS_ERR_IO, "poll(2)");
            redisContextCloseFd(c);
            return REDIS_ERR;
        } else if (res == 0) {
            errno = ETIMEDOUT;
            __redisSetErrorFromErrno(c,REDIS_ERR_IO,NULL);
            redisContextCloseFd(c);
            return REDIS_ERR;
        }

        if (redisCheckSocketError(c) != REDIS_OK)
            return REDIS_ERR;

        return REDIS_OK;
    }

    __redisSetErrorFromErrno(c,REDIS_ERR_IO,NULL);
    redisContextCloseFd(c);
    return REDIS_ERR;
}
```

使用 **b redisContextWaitReady** 增加一个断点，然后使用 **run** 命令重新运行下 **redis-cli**，程序会停在我们设置的断点出，然后使用 **bt** 命令得到当前调用堆栈：

```
(gdb) b redisContextWaitReady
Breakpoint 1 at 0x41bd82: file net.c, line 207.
(gdb) r
Starting program: /root/redis-4.0.11/src/redis-cli 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Breakpoint 1, redisContextWaitReady (c=0x83c050, msec=-1) at net.c:207
207        wfd[0].fd     = c->fd;
(gdb) bt
#0  redisContextWaitReady (c=0x83c050, msec=-1) at net.c:207
#1  0x000000000041c586 in _redisContextConnectTcp (c=0x83c050, addr=0x83c011 "127.0.0.1", port=6379, timeout=0x0, 
    source_addr=0x0) at net.c:391
#2  0x000000000041c6ac in redisContextConnectTcp (c=0x83c050, addr=0x83c011 "127.0.0.1", port=6379, timeout=0x0) at net.c:420
#3  0x0000000000416b53 in redisConnect (ip=0x83c011 "127.0.0.1", port=6379) at hiredis.c:682
#4  0x000000000040ce36 in cliConnect (force=0) at redis-cli.c:611
#5  0x00000000004135d0 in main (argc=0, argv=0x7fffffffe500) at redis-cli.c:2974
```

连接 redis-server 成功以后，会接着调用上文中提到的 cliAuth 函数和 cliSelect 函数，这两个函数分别根据是否配置了 config.auth 和 config.dbnum 来给 redis-server 发送相关命令。由于我们这里没配置，因此这两个函数实际什么也不做。

```
583     static int cliSelect(void) {
(gdb) n
585         if (config.dbnum == 0) return REDIS_OK;
(gdb) p config.dbnum
$11 = 0
```

接着调用 repl() 函数，在这个函数中是一个 while 循环，不断从命令行中获取用户输入：

```
//位于 redis-cli.c 文件中
static void repl(void) {
    //...省略无关代码...
    while((line = linenoise(context ? config.prompt : "not connected> ")) != NULL) {
        if (line[0] != '\0') {
            argv = cliSplitArgs(line,&argc);
            if (history) linenoiseHistoryAdd(line);
            if (historyfile) linenoiseHistorySave(historyfile);

            if (argv == NULL) {
                printf("Invalid argument(s)\n");
                linenoiseFree(line);
                continue;
            } else if (argc > 0) {
                if (strcasecmp(argv[0],"quit") == 0 ||
                    strcasecmp(argv[0],"exit") == 0)
                {
                    exit(0);
                } else if (argv[0][0] == ':') {
                    cliSetPreferences(argv,argc,1);
                    continue;
                } else if (strcasecmp(argv[0],"restart") == 0) {
                    if (config.eval) {
                        config.eval_ldb = 1;
                        config.output = OUTPUT_RAW;
                        return; /* Return to evalMode to restart the session. */
                    } else {
                        printf("Use 'restart' only in Lua debugging mode.");
                    }
                } else if (argc == 3 && !strcasecmp(argv[0],"connect")) {
                    sdsfree(config.hostip);
                    config.hostip = sdsnew(argv[1]);
                    config.hostport = atoi(argv[2]);
                    cliRefreshPrompt();
                    cliConnect(1);
                } else if (argc == 1 && !strcasecmp(argv[0],"clear")) {
                    linenoiseClearScreen();
                } else {
                    long long start_time = mstime(), elapsed;
                    int repeat, skipargs = 0;
                    char *endptr;

                    repeat = strtol(argv[0], &endptr, 10);
                    if (argc > 1 && *endptr == '\0' && repeat) {
                        skipargs = 1;
                    } else {
                        repeat = 1;
                    }

                    issueCommandRepeat(argc-skipargs, argv+skipargs, repeat);

                    /* If our debugging session ended, show the EVAL final
                     * reply. */
                    if (config.eval_ldb_end) {
                        config.eval_ldb_end = 0;
                        cliReadReply(0);
                        printf("\n(Lua debugging session ended%s)\n\n",
                            config.eval_ldb_sync ? "" :
                            " -- dataset changes rolled back");
                    }

                    elapsed = mstime()-start_time;
                    if (elapsed >= 500 &&
                        config.output == OUTPUT_STANDARD)
                    {
                        printf("(%.2fs)\n",(double)elapsed/1000);
                    }
                }
            }
            /* Free the argument vector */
            sdsfreesplitres(argv,argc);
        }
        /* linenoise() returns malloc-ed lines like readline() */
        linenoiseFree(line);
    }
    exit(0);
}
```

得到用户输入的一行命令后，先保存到历史记录中（以便下一次按键盘上的上下箭头键再次输入），然后校验命令的合法性，如果是本地命令（不需要发送给服务器的命令，如 quit 、exit）则直接执行，如果是远端命令则调用 issueCommandRepeat() 函数发送给服务器端：

```
//位于文件 redis-cli.c 中
static int issueCommandRepeat(int argc, char **argv, long repeat) {
    while (1) {
        config.cluster_reissue_command = 0;
        if (cliSendCommand(argc,argv,repeat) != REDIS_OK) {
            cliConnect(1);

            /* If we still cannot send the command print error.
             * We'll try to reconnect the next time. */
            if (cliSendCommand(argc,argv,repeat) != REDIS_OK) {
                cliPrintContextError();
                return REDIS_ERR;
            }
         }
         /* Issue the command again if we got redirected in cluster mode */
         if (config.cluster_mode && config.cluster_reissue_command) {
            cliConnect(1);
         } else {
             break;
        }
    }
    return REDIS_OK;
}
```

实际发送命令的函数是 cliSendCommand，在 cliSendCommand 函数中又调用 cliReadReply 函数，后者又调用 redisGetReply 函数，在 redisGetReply 函数中又调用 redisBufferWrite 函数，在 redisBufferWrite 函数中最终调用系统 API write 将我们输入的命令发出去：

```
//位于 hiredis.c 文件中
int redisBufferWrite(redisContext *c, int *done) {
    int nwritten;

    /* Return early when the context has seen an error. */
    if (c->err)
        return REDIS_ERR;

    if (sdslen(c->obuf) > 0) {
        nwritten = write(c->fd,c->obuf,sdslen(c->obuf));
        if (nwritten == -1) {
            if ((errno == EAGAIN && !(c->flags & REDIS_BLOCK)) || (errno == EINTR)) {
                /* Try again later */
            } else {
                __redisSetError(c,REDIS_ERR_IO,NULL);
                return REDIS_ERR;
            }
        } else if (nwritten > 0) {
            if (nwritten == (signed)sdslen(c->obuf)) {
                sdsfree(c->obuf);
                c->obuf = sdsempty();
            } else {
                sdsrange(c->obuf,nwritten,-1);
            }
        }
    }
    if (done != NULL) *done = (sdslen(c->obuf) == 0);
    return REDIS_OK;
}
```

### 分析输入 set hello world 指令后的执行流

使用 **b redisBufferWrite** 增加一个断点，然后使用 **run** 命令将 **redis-cli** 重新运行起来，接着在 **redis-cli** 中输入 **set hello world** （**hello** 是 key， **world** 是 value）这一个简单的指令后，使用 **bt** 命令查看调用堆栈如下：

```
(gdb) b redisBufferWrite
Breakpoint 2 at 0x417020: file hiredis.c, line 835.
(gdb) r
Starting program: /root/redis-4.0.11/src/redis-cli 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Breakpoint 2, redisBufferWrite (c=0x83c050, done=0x7fffffffe1cc) at hiredis.c:835
835        if (c->err)
(gdb) c
Continuing.
127.0.0.1:6379> set hello world

Breakpoint 2, redisBufferWrite (c=0x83c050, done=0x7fffffffe27c) at hiredis.c:835
835        if (c->err)
(gdb) bt
#0  redisBufferWrite (c=0x83c050, done=0x7fffffffe27c) at hiredis.c:835
#1  0x0000000000417246 in redisGetReply (c=0x83c050, reply=0x7fffffffe2a8) at hiredis.c:882
#2  0x000000000040d9a4 in cliReadReply (output_raw_strings=0) at redis-cli.c:851
#3  0x000000000040e16c in cliSendCommand (argc=3, argv=0x8650c0, repeat=0) at redis-cli.c:1011
#4  0x000000000040f153 in issueCommandRepeat (argc=3, argv=0x8650c0, repeat=1) at redis-cli.c:1288
#5  0x000000000040f869 in repl () at redis-cli.c:1469
#6  0x00000000004135d5 in main (argc=0, argv=0x7fffffffe500) at redis-cli.c:2975
```

当然，待发送的数据需要存储在一个全局静态变量 context 中，这是一个结构体，定义在 hiredis.h 文件中。

```
/* Context for a connection to Redis */
typedef struct redisContext {
    int err; /* Error flags, 0 when there is no error */
    char errstr[128]; /* String representation of error when applicable */
    int fd;
    int flags;
    char *obuf; /* Write buffer */
    redisReader *reader; /* Protocol reader */

    enum redisConnectionType connection_type;
    struct timeval *timeout;

    struct {
        char *host;
        char *source_addr;
        int port;
    } tcp;

    struct {
        char *path;
    } unix_sock;

} redisContext;
```

其中字段 obuf 指向的是一个 sds 类型的对象，这个对象用来存储当前需要发送的命令。这也同时解决了命令一次发不完需要暂时缓存下来的问题。

在 redisGetReply 函数中发完数据后立马调用 redisBufferRead 去收取服务器的应答。

```
int redisGetReply(redisContext *c, void **reply) {
    int wdone = 0;
    void *aux = NULL;

    /* Try to read pending replies */
    if (redisGetReplyFromReader(c,&aux) == REDIS_ERR)
        return REDIS_ERR;

    /* For the blocking context, flush output buffer and read reply */
    if (aux == NULL && c->flags & REDIS_BLOCK) {
        /* Write until done */
        do {
            if (redisBufferWrite(c,&wdone) == REDIS_ERR)
                return REDIS_ERR;
        } while (!wdone);

        /* Read until there is a reply */
        do {
            if (redisBufferRead(c) == REDIS_ERR)
                return REDIS_ERR;
            if (redisGetReplyFromReader(c,&aux) == REDIS_ERR)
                return REDIS_ERR;
        } while (aux == NULL);
    }

    /* Set reply object */
    if (reply != NULL) *reply = aux;
    return REDIS_OK;
}
```

拿到应答后就可以解析并显示在终端了。

总结起来，redis-cli 是一个实实在在的网络同步通信方式，只不过通信的 socket 仍然设置成非阻塞模式，这样有如下三个好处：

（1）使用 connect 连接服务器时，connect 函数不会阻塞，可以立即返回，之后调用 poll 检测 socket 是否可写来判断是否连接成功。

（2）在发数据时，如果因为对端 tcp 窗口太小发不出去，write 函数也会立即返回不会阻塞，此时可以将未发送的数据暂存，下次继续发送。

（3）在收数据时，如果当前没有数据可读，则 read 函数也不会阻塞，程序可以立即返回，继续响应用户的输入。

### Redis 的通信协议格式

Redis 客户端与服务器通信使用的是纯文本协议，以 \r\n 来作为协议或者命令或参数之间的分隔符。

我们接着通过 redis-cli 给 redis-server 发送 set hello world 命令。

```
127.0.0.1:6379> set hello world
```

此时服务器端收到的数据格式如下：

```
*3\r\n3\r\nset\r\n5\r\nhello\r\n$5\r\nworld\r\n
```

其中第一个 3 是 redis 命令的标志信息，标志以星号（ * ）开始，数字 3 是请求类型，不同的命令数字可能不一样，接着 \r\n 分割，后面就是统一的格式：

```
A指令字符长度\r\n指令A\r\nB指令或key字符长度\r\nB指令\r\nC内容长度\r\nC内容\r\n
```

不同的指令长度不一样，携带的 key 和 value 也不一样，服务器端会根据命令的不同来进一步解析。

### 小结

至此，我们将 Redis 的服务端和客户端的网络通信模块分析完了，Redis 的通信模型是非常常见的网络通信模型。Redis 也是目前业界使用最多的内存数据库，它不仅开源，而且源码量也不大，其中用到的数据结构（ 字符串、链表、集合等 ）都有自己的高效实现，是学习数据结构知识非常好的材料。想成为一名合格的服务器端开发人员，应该去学习它、用好它。

------

好了，关于 GDB 调试的课程到这里基本就结束了，但是 GDB 更多的调试技巧等着亲爱的读者你去发现。如果对课程内容有任何意见，或者想与我做进一步交流，欢迎通过以下公众号或读者群与我取得联系。学习、研究和分享高性能服务器开发技术，我们是认真的。

![img](https://images.gitbook.cn/56cc10c0-fdf1-11e8-a53e-b108479b030d)