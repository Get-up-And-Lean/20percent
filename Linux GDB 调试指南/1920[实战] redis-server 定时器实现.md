# 19/20[实战] redis-server 定时器实现

### 定时器逻辑

一个网络通信模块是离不开定时器，前面介绍了在事件处理函数中如何去除最早到期的定时器对象，这里我们接着这个问题继续讨论。在 aeProcessEvents 函数（ 位于文件 ae.c 中 ）的结尾处有这样一段代码：

```
/* Check time events */
if (flags & AE_TIME_EVENTS)
    processed += processTimeEvents(eventLoop);
```

如果存在定时器事件，则调用 processTimeEvents 函数（ 位于文件 ae.c 中 ）进行处理。

```
/* Process time events */
static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;
    aeTimeEvent *te, *prev;
    long long maxId;
    time_t now = time(NULL);

    /* If the system clock is moved to the future, and then set back to the
     * right value, time events may be delayed in a random way. Often this
     * means that scheduled operations will not be performed soon enough.
     *
     * Here we try to detect system clock skews, and force all the time
     * events to be processed ASAP when this happens: the idea is that
     * processing events earlier is less dangerous than delaying them
     * indefinitely, and practice suggests it is. */
    if (now < eventLoop->lastTime) {
        te = eventLoop->timeEventHead;
        while(te) {
            te->when_sec = 0;
            te = te->next;
        }
    }
    eventLoop->lastTime = now;

    prev = NULL;
    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    while(te) {
        long now_sec, now_ms;
        long long id;

        /* Remove events scheduled for deletion. */
        if (te->id == AE_DELETED_EVENT_ID) {
            aeTimeEvent *next = te->next;
            if (prev == NULL)
                eventLoop->timeEventHead = te->next;
            else
                prev->next = te->next;
            if (te->finalizerProc)
                te->finalizerProc(eventLoop, te->clientData);
            zfree(te);
            te = next;
            continue;
        }

        /* Make sure we don't process time events created by time events in
         * this iteration. Note that this check is currently useless: we always
         * add new timers on the head, however if we change the implementation
         * detail, this check may be useful again: we keep it here for future
         * defense. */
        if (te->id > maxId) {
            te = te->next;
            continue;
        }
        aeGetTime(&now_sec, &now_ms);
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;

            id = te->id;
            retval = te->timeProc(eventLoop, id, te->clientData);
            processed++;
            if (retval != AE_NOMORE) {
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        prev = te;
        te = te->next;
    }
    return processed;
}
```

这段代码核心逻辑就是通过 eventLoop→timeEventHead 中记录的定时器对象链表遍历每个定时器对象的时间，然后与当前时间比较，如果定时器已经到期，则调用定时器对象设置的回调函数 timeProc 进行处理。

这段代码没有什么特别需要注意的地方，但是代码中作者考虑到了一种特殊场景，就是假设有人将当前的计算机时间调到了未来某个时刻，然后再调回来，这样就会出现 now（ 当前时间 ）小于 eventLoop→lastTime（ 记录在 aeEventLoop 中的上一次时间）。出现这种情况怎么办呢？Redis 的作者遍历该定时器对象链表，将这个链表中的所有定时器对象的时间设置成 0 。这样，这些定时器就会立即得到处理了。这也就是我在代码注释中说的：

```
force all the time events to be processed ASAP
```

> ASAP 是英文 As Soon As Possible（尽快）的缩写。

那么 redis-server 中到底哪些地方使用了定时器呢？我们可以在 Redis 源码中搜索创建定时器的函数 aeCreateTimeEvent ，在 initServer 函数中有这么一行（ 位于 server.c 文件中 ）：

```
if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        serverPanic("Can't create event loop timers.");
        exit(1);
 }
```

上述代码我们在前面课程也提到过，原来定时器的用途是用于 redis 的 Cron 任务。这个任务具体做些什么工作，就不是本课程的内容了，有兴趣的话可以阅读 serverCron 函数源码（ 位于 server.c 中 ）。

### aftersleep 钩子

通常情形下，在一个 EventLoop 中除了有定时器、IO Multiplexing 和 IO 事件处理逻辑外，可以根据需求自定义一些函数，这类函数我们称之为“钩子函数”。钩子函数可以位于 Loop 的任何位置，前面我们介绍的 beforesleep 函数就是在事件处理之前自定义的钩子函数（ 位于定时器时间检测逻辑之前 ）。

在 redis-server 中，在 IO Multiplexing 调用与 IO 事件处理逻辑之间也有一个自定义的钩子函数叫 aftersleep 。

```
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    //无关代码省略...
    numevents = aeApiPoll(eventLoop, tvp);

    /* After sleep callback. */
    if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
        eventLoop->aftersleep(eventLoop);

    for (j = 0; j < numevents; j++) {
        //无关代码省略...
    }    
}
```

这个函数在 main 函数中设置：

```
int main(int argc, char **argv) {
    //无关代码省略...
    aeSetBeforeSleepProc(server.el,beforeSleep);
    aeSetAfterSleepProc(server.el,afterSleep);

     return 0;
}
```

由于 afterSleep 函数的具体作用与网络通信无关，这里不再赘述。

### 小结

通过前面的讲解，我们用一张图来概括一下 redis-server 端的网络通信模型。

![img](https://images.gitbook.cn/c3f5c9e0-f20a-11e8-a886-5157ca7834b5)

如上图所示，这就是典型的利用 one loop one thread 思想实现的 reactor 网络通信模型，也是目前最主流的网络通信架构。而且由于 redis-server 的网络通信中所有的客户端 fd 和侦听 fd 都集中在一个 EventLoop 中，所以通常也说 Redis 的网络通信模型是单线程的。