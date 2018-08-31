## 详细的过程

### initServerConfig\(\)

initServerConfig\(\) 主要是填充 struct redisServer 这个结构体，redis 所有相关的配置都在里面。

### initServer\(\)

```c
void initServer() {
    // 创建事件循环结构体
    server.el = aeCreateEventLoop(server.maxclients+REDIS_EVENTLOOP_FDSET_INCR);

    // 分配数据集空间
    server.db = zmalloc(sizeof(redisDb)*server.dbnum);

    /* Open the TCP listening socket for the user commands. */
    // listenToPort() 中有调用 listen()
    if (server.port != 0 &&
        listenToPort(server.port,server.ipfd,&server.ipfd_count) == REDIS_ERR)
        exit(1);
    ......

    // 初始化 redis 数据集
    /* Create the Redis databases, and initialize other internal state. */
    for (j = 0; j < server.REDIS_DEFAULT_DBNUM; j++) { // 初始化多个数据库
        // 哈希表，用于存储键值对
        server.db[j].dict = dictCreate(&dbDictType,NULL);
        // 哈希表，用于存储每个键的过期时间
        server.db[j].expires = dictCreate(&keyptrDictType,NULL);
        server.db[j].blocking_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].ready_keys = dictCreate(&setDictType,NULL);
        server.db[j].watched_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].id = j;
        server.db[j].avg_ttl = 0;
    }
    ......
    // 创建接收 TCP 或者 UNIX 域套接字的事件处理
    // TCP
    /* Create an event handler for accepting new connections in TCP and Unix
     * domain sockets. */
    for (j = 0; j < server.ipfd_count; j++) {

        // acceptTcpHandler() tcp 连接接受处理函数
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                redisPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
    ......
}
```

在这里，创建了事件中心，是 redis 的网络模块，如果你有学过 linux 下的网络编程，那么知道这里一定和 select/epoll/kqueue 相关。

接着，是初始化数据中心，我们平时使用 redis 设置的键值对，就是存储在里面。这里不急着深入它是怎么做到存储我们的键值对的，接着往下看好了，因为我们主要是想把大致的脉络弄清楚。

接着，是初始化数据中心，我们平时使用 redis 设置的键值对，就是存储在里面。这里不急着深入它是怎么做到存储我们的键值对的，接着往下看好了，因为我们主要是想把大致的脉络弄清楚。

在最后一段的代码中，redis 给 listen fd 注册了回调函数 acceptTcpHandler，也就是说当新的客户端连接的时候，这个函数会被调用，详情接下来再展开 。

### aeMain\(\)

接着就开始等待请求的到来。

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {

        // 进入事件循环可能会进入睡眠状态。在睡眠之前，执行预设置
        // 的函数 aeSetBeforeSleepProc()。
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);

        // AE_ALL_EVENTS 表示处理所有的事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```

前面的两个函数都属于是初始化的工作，到这里的时候，redis 正式进入等待接收请求的状态。具体的实现，和 select/epoll/kqueue 这些 IO 多路复用的系统调用相关，而这也是网络编程的基础部分了。继续跟踪调用链：

```c

```

可以看到，aeApiPoll 即是 IO 多路复用调用的地方，当有请求到来的时候，进程会觉醒以处理到来的请求。如果你有留意到上面的定时事件处理，也就明白相应的定时处理函数是在哪里触发的了。



