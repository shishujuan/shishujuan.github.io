---
layout:     post
title:      "Redis 主流程分析"
subtitle:   ""
date:       2018-08-11 10:54:00 +0800
author:     "ssj"
categories: 数据库
catalog: true
comments: true
tags:
    - Redis
---

> 网上分析Redis源码的文章挺多，如黄健宏的《Redis设计与实现》就很详尽的分析了redis源码，很赞。前不久看到Paul Smith的较早年份的大作《Redis：under the hood》，受益匪浅，如此从整体上对redis原理有个大的把控，不过多纠结于细节，甚好。这里我用的版本是redis2.4.18版本，跟Paul Smith的版本有所不同，不过主体流程没有太多变化，这篇文章基本是对 《Redis：under the hood》的翻译和注解。

# 1 启动
让我们从 src/redis.c 中的main() 函数开始。

## 全局服务器状态初始化
首先， `initServerConfig()` 函数被调用。这部分主要是初始化 `server` 变量，它是一个 `struct redisServer`类型，用于保存redis服务器全局状态。

```
// redis.h:388
struct redisServer {
    pthread_t mainthread;
    int arch_bits;
    int port;
    char *bindaddr;
    char *unixsocket;
    mode_t unixsocketperm;
    int ipfd;
    int sofd;
    redisDb *db; 
    list *clients; // 客户端列表
    dict *commands; // 命令字典，key为命令名如get，值为redisCommand类型。
    unsigned lruclock:22;        /* clock incrementing every minute, for LRU */
    unsigned lruclock_padding:10;
    ...
}

// redis.c:71
struct redisServer server;
```
redisServer结构体有很多成员，主要可以分为下面几种类型：

- 全局的服务器状态
- 统计信息
- 配置信息
- 复制信息
- 排序参数
- 虚拟内存配置，状态，I/O线程以及统计
- zip结构体(默认ziplist，zipmap大小等)
- event loop helpers
- pub/sub

`initServerConfig()`的作用是设置redis server的默认配置。

- 如设置redis的run_id，设置端口为6379。
- 设置db名为`dump.rdb`，pid文件目录为 `/var/run/redis.pid`，关闭aof。
- 关闭daemon模式，过期数据淘汰策略为LRU。
- 设置保存策略 `saveparams` 为 1小时有1个值变化就保存，5分钟有100个值变化保存以及1分钟有1万个值变化保存。
- 设置 `double R_PosInf = 1.0/0.0`，好吧，用python的时候除0.0会抛异常，C里面不会，这里会变成无穷大。同理，还可以设置无穷小等。
- 创建`commandTableDictType`类型的字典commands，然后调用 `populateCommandTable()`函数来初始化redis命令集。redis的字典 `dict` 的实现采用的也是经典的哈希表实现，冲突的键通过哈希表串联。
- 设置慢日志记录条件，默认是`REDIS_SLOWLOG_LOG_SLOWER_THAN`即命令执行时间超过10000毫秒(10秒)才记录慢日志。慢日志记录条数默认为128。

## 设置redis命令表
上一节提到redis的命令存储在server.commands这个字典中，其中key为命令名字，value为 `redisCommand` 结构体，其定义如下：

```
// redis.c:73
struct redisCommand readonlyCommandTable[] = {
    {"get",getCommand,2,0,NULL,1,1,1},
    {"set",setCommand,3,REDIS_CMD_DENYOOM,NULL,0,0,0},
    {"setnx",setnxCommand,3,REDIS_CMD_DENYOOM,NULL,0,0,0},
    {"setex",setexCommand,4,REDIS_CMD_DENYOOM,NULL,0,0,0},
    {"append",appendCommand,3,REDIS_CMD_DENYOOM,NULL,1,1,1},
    {"strlen",strlenCommand,2,0,NULL,1,1,1},
    {"del",delCommand,-2,0,NULL,0,0,0},
    ...
}

// redis.c:561
typedef void redisCommandProc(redisClient *c);

// redis.c:563
struct redisCommand {
    char *name;  // 命令名，如get
    redisCommandProc *proc; // 命令对应函数，如getCommand
    int arity; // 参数个数，如2
    int flags; // 标记，如set命令为REDIS_CMD_DENYOOM，表示在内存不够时不再处理set命令。
    redisVmPreloadProc *vm_preload_proc; 
    int vm_firstkey; /* The first argument that's a key (0 = no keys) */
    int vm_lastkey;  /* THe last argument that's a key */
    int vm_keystep;  /* The step between first and last key */
};

```
其中 readonlyCommandTable数组就是命令集合，redisCommand各字段分别是命名名，命令函数，参数个数，oom标记以及vm相关参数。

## 解析配置文件并更新配置

接下来会判断启动参数个数，如果参数个数为2：
- 第二个参数是`-v/--version` 则显示版本信息，若是`--help`显示帮助信息。如果是其他，则标识是配置文件，则解析配置文件并更新server的配置。
- 如果超过2个参数，则会判断是否是测试内存的命令，如果是，则测试内存，否则显示帮助信息。

解析配置文件是`loadServerConfig()`函数完成的，通过fgets一行行读取`redis配置文件`并更新服务器的配置。在这个函数里面可以看到若干的`if else`语句，一些所谓的编程书籍不提倡这样，包括goto使用等，然而大师级的程序员并不在意这些细节，所谓编程无定法，境界高就是可以为所欲为的。

从代码中可以发现，redis配置也可以不指定配置文件，而是在标准输入指定，运行`src/redis-server -`，然后在命令行输入配置即可。

## 开启daemon
如果配置了以守护进程运行，则会调用`daemonize()`函数通过`fork()`创建子进程然后子进程调用`setsid()`创建一个新的会话，并将标准输入输出，错误输出冲形象到`/dev/null`。

有些书上会写运行守护进程要fork()两次，其实通过setsid()创建了新的会话的话，没有必要fork()两次，redis就是这么做的。

如果以守护进程运行，后面还需调用`createPidFile()`创建pid文件，默认路径是`/var/run/redis.pid`。

## 初始化服务器
在上面步骤完成后，就调用`initServer()`函数来初始化服务器了。

### 信号设置
先是信号设置。忽略SIGHUP, SIGPIPE信号，然后通过`setupSignalHandlers()`设置TERM信号等处理函数为 sigtermHandler()等。

### 成员初始化
接着是初始化server成员变量，如客户端链表clients，slave链表slaves，这些列表都是双向链表`struct list`。

### 创建事件循环对象
接着是调用`aeCreateEventLoop()`创建`Event Loop` 并赋值给 `server.el`变量。它的类型是 `aeEventLoop`，定义如下：

```
// ae.h:89
typedef struct aeEventLoop { 
    int maxfd;
    long long timeEventNextId;
    aeFileEvent events[AE_SETSIZE]; /* Registered events */
    aeFiredEvent fired[AE_SETSIZE]; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
} aeEventLoop;

/* File event structure */
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE) */
    aeFileProc *rfileProc;
    aeFileProc *wfileProc;
    void *clientData;
} aeFileEvent;

/* Time event structure */
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc;
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *next;
} aeTimeEvent;

/* A fired event */
typedef struct aeFiredEvent {
    int fd; 
    int mask;
} aeFiredEvent;
```
`aeEventLoop` 包括时间事件链表头 `timeEventHead` ，文件事件数组 `aeFileEvent`数组，以及待处理的文件事件数组 `aeFiredEvent`(AE_SETSIZE为10240)。

可以看到文件事件结构体aeFileEvent中字段mask是标记，表示事件类型读/写。而rfileProc则是读取事件处理函数，而wfileProc是写事件处理函数。而待处理的文件事件 `aeFiredEvent` 则只包含了需要处理的文件描述符fd和它的读写标记mask。

而时间事件则是 `aeTimeEvent` 类型，存储的包括时间事件ID，时间事件执行时间(秒 when_sec 和 毫秒 when_ms），此外还有时间事件的处理函数 timeProc 等。这是一个单向链表结构，next指向下一个时间事件，时间事件和文件事件最后都是在redis服务器的大循环中处理的。


### 服务器监听

如果指定了端口，则会启动anetTcpServer并开始监听。监听端口默认为6379，配置文件可以指定绑定的ip和端口。对应文件描述符为ipfd。如果是设置的unixsocket，则启动anetUnixServer，对应文件描述符为sofd。

这里跟我们平时写WEB服务器程序基本一致，只是稍作了封装，流程也是通用的socket()，bind()，listen()。

```
int anetTcpServer(char *err, int port, char *bindaddr)
{
    int s;       
    struct sockaddr_in sa;

    if ((s = anetCreateSocket(err,AF_INET)) == ANET_ERR)
        return ANET_ERR;

    memset(&sa,0,sizeof(sa));
    sa.sin_family = AF_INET;
    sa.sin_port = htons(port); 
    sa.sin_addr.s_addr = htonl(INADDR_ANY);
    if (bindaddr && inet_aton(bindaddr, &sa.sin_addr) == 0) {
        anetSetError(err, "invalid bind address");
        close(s);
        return ANET_ERR;
    }    
    if (anetListen(err,s,(struct sockaddr*)&sa,sizeof(sa)) == ANET_ERR)
        return ANET_ERR;
    return s;
}
```

### 数据库
`initServer()` 中还完成了数据库初始化。默认是16个db，对应类型为dbDictType，id为0-15。此外还要初始化过期键字典expires，阻塞键字典 blocking_keys，观察键字典 watched_keys等。

```
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP) */
    dict *io_keys;              /* Keys with clients waiting for VM I/O */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;
} redisDb;
```
redis的持久化分为两种方式，rdb和aof。其中rdb是使用二进制格式存储数据，包括键值类型和数据，过期时间等，saveparams里面指定存储条件，详细格式说明见 [Redis-RDB-Format](https://github.com/sripathikrishnan/redis-rdb-tools/wiki/Redis-RDB-Dump-File-Format)。aof则是按redis命令存储，重启后可以按照aof中的命令重放恢复数据。

### 注册时间事件
接着在server.el中注册时间事件。这个就是1毫秒后要开始执行的事件 `serverCron()`，主要是为了在服务器启动后马上运行它，后面该函数100毫秒执行一次。这个函数要做很多事情，主要包括：

- 缓存当前时间，因为在LRU和VM访问对象时都要记录访问时间，每次调用 `time(NULL)` 开销太大，而缓存这个时间并不会有很大影响。
- 更新LRUClock值，这个用于LRU策略。`redisServer` 有一个全局的 `lruclock`，该时钟每100ms更新一次。虽然lru用了22位，但是因为它最大为`REDIS_LRU_CLOCK_MAX((1<<21)-1)`，其实是只用到21位，精度为10秒，所以它每242天会重新开始计时(跟redis源码注释中说的略有不同，注释说22位的话wrap时间为1.5年左右，但其实最大是用了21位)。而每个`redisObject`也有一个自己的 lruclock，这样在使用内存超过maxmemory之后就可以根据全局时钟和每个redisObject的时钟进行比较，确定是否淘汰。这里有个问题是，因为LRUClock每隔242天会重置，所以可能会导致一些很久没有访问的键它的lru更大，不过这个没有太大问题，一个键这么久没有访问，说明不太活跃。
- 如果达到了条件，执行BGSAVE（根据save配置来决定）和AOF文件重写。BGSAVE和AOF重写都是在子进程中执行的，这样不会影响redis主进程继续处理请求，见`rdbSaveBackground()`。**注意，aof文件定期刷磁盘主要在beforeSleep中通过后台IO线程执行，serverCron只是在对aof刷磁盘操作推迟时做些处理。**
- 打印统计信息。如key的数目，设置了过期时间的key的数目，连接的client数目，slave数目以及内存使用情况等，统计信息每50个循环(50*100ms=5秒)打印一次。
- 还有resize 哈希表，关闭超时客户端连接，后台的AOF重写，BGSAVE（如多少秒内有多少个键发生了变化执行的保存操作）。
- 计算LRU信息并删除一部分过期的键，如果开启了vm的话还要swap一些键值到磁盘上。
- 如果是slave，还需要从master同步数据。

### 注册文件事件
那之前我们创建了一个TcpServer而且已经开始监听，需要对客户端连接事件进行注册和处理。这在Linux上面是通过 epoll 来实现的。

`aeCreateFileEvent()`函数主要是设置aeFileEvent结构体的值，包括指定该文件事件是读还是写，根据读写事件指定对应的处理函数 rfileProc和 wfileProc。这里对tcp服务器指定的函数是 `acceptTcpHandler()`。最终都是通过 `aeApiAddEvent()` 函数使用 `epoll_ctl()` 将 tcp socket的fd注册到epoll中，客户端连接的命令处理都是在 `acceptTcpHandler()`中完成，这个函数后面分析。

```
// redis.c:980
aeCreateFileEvent(server.el,server.ipfd,AE_READABLE, acceptTcpHandler,NULL);

// ae.c:88
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE) */
    aeFileProc *rfileProc;
    aeFileProc *wfileProc;
    void *clientData;
} aeFileEvent;

// ae_epoll.c:29
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {           
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee;
    /* If the fd was already monitored for some event, we need a MOD           
     * operation. Otherwise we need an ADD operation. */                       
    int op = eventLoop->events[fd].mask == AE_NONE ?                           
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;                                     

    ee.events = 0; 
    mask |= eventLoop->events[fd].mask; /* Merge old events */                 
    if (mask & AE_READABLE) ee.events |= EPOLLIN;                              
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;                             
    ee.data.u64 = 0; /* avoid valgrind warning */                              
    ee.data.fd = fd;                                                           
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;                     
    return 0;
}
```

### 其他
- 如果配置了AOF的话，接着会打开AOF文件。
- 如果是32位机器，且没有配置maxmemory的话，会将maxmemory设置为3.5G，数据淘汰策略为 `REDIS_MAXMEMORY_NO_EVICTION`，这个选项的意思是`禁止删除数据，永远不过期，只会对写操作返回一个错误`，这也是默认的数据淘汰数据。
- 如果开启了vm，则会调用`vmInit()`初始化vm模块。
- 调用`slowlogInit()`初始化slowlog。
- 调用`bioInit()`初始化后台IO线程，一共两个后台线程，其中一个用于关闭文件，一个用于定期将AOF文件刷到磁盘。redis的AOF刷新策略有ALWAYS，EVERYSEC，NO，默认是EVERYSEC。ALWAYS是同步写入，会在进入每个事件循环后通过 `aof_fsync()` (在linux下面就是fdatasync)同步到磁盘。而EVERYSEC则是通过 `aof_background_fsync()`将刷磁盘操作加入到一个作业列表中(每秒执行一次)，由bioInit创建的后台IO线程执行刷磁盘操作。设置为NO则不刷新磁盘，同步到磁盘的操作由操作系统决定。
- 调用 `srand(time(NULL)^getpid())` 初始化随机数发生器。

## 恢复数据
如果开启了AOF，则从AOF文件重放命令来恢复redis数据。否则，则是从RDB文件恢复数据。每次进行RDB持久化时，redis都是将内存中的数据库的数据全部写到文件中，不是增量的持久化。

```
if (server.appendonly) {
    if (loadAppendOnlyFile(server.appendfilename) == REDIS_OK)
        redisLog(REDIS_NOTICE,"DB loaded from append only file: %ld seconds",time(NULL)-start);
} else {
    if (rdbLoad(server.dbfilename) == REDIS_OK) {
        redisLog(REDIS_NOTICE,"DB loaded from disk: %ld seconds",
            time(NULL)-start);
    } else if (errno != ENOENT) {
        redisLog(REDIS_WARNING,"Fatal error loading the DB. Exiting.");
        exit(1);
    }    
}  
```

## 建立事件循环

接着，redis注册`beforeSleep()`函数到事件循环中，这个函数在每次进入事件循环时首先调用它，它主要做两件事：

- 对开启vm的情况下，将那些请求已经交换到磁盘的key的客户端解除阻塞并处理这些客户端请求。
- 调用 `flushAppendOnlyFile()` 将AOF文件刷到磁盘，最终调用的是 `aof_fsync()` 或者 `aof_background_fsync()`。

## 进入事件循环
redis接着正式调用 `aeMain()` 函数进入事件循环。当有时间事件或者文件事件需要处理时，会调用他们对应的处理函数进行处理。`aeProcessEvents()`封装了处理函数，时间事件通过自定义的函数处理，而文件事件则通过`epoll或者kqueue或者select`系统调用来处理，在Linux里面通常使用的是epoll。

`aeProcessEvents()` 会优先处理文件事件，其次才是处理时间事件。文件事件就是通过 `aeApiPoll()` 函数来获取事件，并将触发的事件加入到 `server.el.fired` 数组中，最终就是调用`epoll_wait()`获取事件，其中超时时间设置的是距离最近一次时间事件的时间，这样如果没有文件事件也不会太耽误时间事件执行。获取到文件事件后，会根据事件类型是读还是写调用相应的方法处理。如读事件就是调用的 `acceptTcpHandler()` 处理的。

接着处理时间事件，从 `server.el.timeEventHead` 可以拿到时间事件链表的头，遍历该链表，如果有时间事件的执行时间到了，则执行对应的函数即可。这里有个地方注意下，如果时间事件处理函数返回值不是-1，则表示该时间事件需要定期执行，需要设置该事件下一次执行时间而不是从时间事件链表移除它，如serverCron这个时间事件，就是这样的定期执行事件，100ms执行一次。


# 2 请求处理和响应
现在服务器已经启动完毕了，接下来看看redis是如何在事件循环中接收客户端请求并处理请求的，这里以 TCP 方式为例分析，unix socket的类似。

## 接收连接
redis处理客户端请求是在函数 `acceptTcpHandler()` 完成的。这个函数通过 `anetTcpAccept()` 接收客户端请求，然后调用的 `acceptCommandHandler()` 来处理客户端请求。

在acceptCommandHandler最终调用的是 `createClient(fd)`函数创建了redisClient对象，初始化该对象的变量，并将该连接fd注册到事件循环中，事件类型为 `AE_READABLE(EPOOLIN)`。该事件处理函数为 `readQueryFromClient()`，用于处理客户端连接的命令。这样，我们之前注册了`listenfd`，用于在新连接到来时接收新连接，现在将客户端连接fd注册到事件循环，完成客户端命令处理。注意这里必须通过`anetNonBlock(NULL, fd)`将客户端连接fd设置为非阻塞的。

```
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd; 
    char cip[128];
    cfd = anetTcpAccept(server.neterr, fd, cip, &cport);
    if (cfd == AE_ERR) {
        redisLog(REDIS_WARNING,"Accepting client connection: %s", server.neterr);
        return;
    }    
    redisLog(REDIS_VERBOSE,"Accepted %s:%d", cip, cport);
    acceptCommonHandler(cfd);
}

redisClient *createClient(int fd) {
    redisClient *c = zmalloc(sizeof(redisClient));
    c->bufpos = 0;

    anetNonBlock(NULL,fd);
    anetTcpNoDelay(NULL,fd);
    if (aeCreateFileEvent(server.el,fd,AE_READABLE,
        readQueryFromClient, c) == AE_ERR)
    {
        close(fd);
        zfree(c);
        return NULL;
    }
    ...
}

```

## 从客户端读取命令
当客户端发送命令时，会通过`readQueryFromClient()`处理。它每次读取最多`REDIS_IOBUF_LEN(16*1024)`16K字节到缓存数组buf中，最后将缓存的数据拷贝到 redisClient->querybuf 中，然后调用 `processInputBuffer()` 函数处理客户端命令。

`processInputBuffer()` 解析客户端的原始命令字符串并将命令参数设置到 redisClient->argv 数组中，命令参数是 `redisObject` 类型的结构体。注意命令类型有两种，我们通过 `redis-cli` 发送的命令类型为 `REDIS_REQ_MULTIBULK`，这种命令以`*`开头，符合 [redis protocol](https://redis.io/topics/protocol)，调用`processMultibulkBuffer()`函数处理。另外一种命令是 `REDIS_REQ_INLINE`，这种命令是你通过其他工具连接的时候发的，比如通过 `telnet localhost 6379`，这种命令是直接的原生字符串，没有使用 `redis protocol`封装客户端命令。

```
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    redisClient *c = (redisClient*) privdata;
    char buf[REDIS_IOBUF_LEN];
    int nread;
    
    server.current_client = c;
    nread = read(fd, buf, REDIS_IOBUF_LEN);
    ...
    processInputBuffer(c);
    server.current_client = NULL;
}

void processInputBuffer(redisClient *c) {
    while(sdslen(c->querybuf)) {
        /* Immediately abort if the client is in the middle of something. */
        if (c->flags & REDIS_BLOCKED || c->flags & REDIS_IO_WAIT) return;
        
        /* Determine request type when unknown. */
        if (!c->reqtype) {
            if (c->querybuf[0] == '*') {
                c->reqtype = REDIS_REQ_MULTIBULK;
            } else {
                c->reqtype = REDIS_REQ_INLINE;
            }
        }

        if (c->reqtype == REDIS_REQ_INLINE) {
            if (processInlineBuffer(c) != REDIS_OK) break;
        } else if (c->reqtype == REDIS_REQ_MULTIBULK) {
            if (processMultibulkBuffer(c) != REDIS_OK) break;
        } else {
            redisPanic("Unknown request type");
        }

        /* Multibulk processing could see a <= 0 length. */
        if (c->argc == 0) {
            resetClient(c);
        } else {
            /* Only reset the client when the command was executed. */
            if (processCommand(c) == REDIS_OK)
                resetClient(c);
        }
    }
}
```

在`processInputBuffer()`解析到完整命令后，便会调用 `processCommand()`函数开始处理客户端命令。先通过`lookupCommand`根据命令字符串找到命令的处理程序。在调用命令处理程序执行实际的命令前，会先执行一系列的检查：

- 如果命令不存在或者参数个数不对，返回错误。
- 如果redis配置了密码而客户端请求没有通过认证就发正式命令，返回错误。
- 比如如果当前是在一个 `PUB/SUB` 上下文中，则只允许 `subscribe/unsubscribe`等命令。
- 如果设置了最大内存maxmemory，则执行 `SET` 等设置了 `REDIS_CMD_DENYOOM`标识的命令时会返回错误。
- 如果redis服务器正在加载DB文件，则只允许`info`命令，其他命令报错。
- 如果是事务命令 `MULTI`，只要输入的不是`exec，discard，multi，watch`等命令，则将命令加入队列中以便后面批量执行。


## 执行命令
在上一节的检查OK后执行 `call()` 函数真正开始执行命令，它调用的是 `redisCommand` 的proc指向的函数，为 `redisCommandProc` 类型对象。命令执行完后，会通过 `addReply()` 函数将执行结果缓存到 redisClient 的 buf 数组中。

那这个响应数据什么时候会发送给客户端呢？这是因为在 `addReply()` 中会调用 `_installWriteEvent()`，该函数就是将客户端连接的fd加入到事件循环中，事件类型为`AE_WRITABLE(EPOLLOUT)`，然后将响应数据通过`_addReplyToBuffer()`和`_addReplyObjectToList()`写入到响应缓存`redisClient->buf和redisClient->reply`中。这里的buf和reply两个地方都是用于写响应缓存的，如果响应的总的数据长度(响应数据长度+数据本身)小于 `REDIS_REPLY_CHUNK_BYTES(7500)`字节，则用buf数组缓存数据，否则用reply链表来存储数据。

当下一个事件循环到来时，会读取到该客户端连接fd，然后通过函数 `sendReplyToClient()`从响应缓存读取数据并发送响应数据给客户端，然后移除写事件。命令执行完成后，redis会重置redisClient对象并接收后续命令。


```
void call(redisClient *c) {
    long long dirty, start = ustime(), duration;

    dirty = server.dirty;
    c->cmd->proc(c);
    dirty = server.dirty-dirty;
    duration = ustime()-start;
    slowlogPushEntryIfNeeded(c->argv,c->argc,duration);

    if (server.appendonly && dirty > 0) 
        feedAppendOnlyFile(c->cmd,c->db->id,c->argv,c->argc);
    if ((dirty > 0 || c->cmd->flags & REDIS_CMD_FORCE_REPLICATION) &&
        listLength(server.slaves))
        replicationFeedSlaves(server.slaves,c->db->id,c->argv,c->argc);
    if (listLength(server.monitors))
        replicationFeedMonitors(server.monitors,c->db->id,c->argv,c->argc);
    server.stat_numcommands++;
}
```

写操作如`SET/ZADD`等会让redis服务器变的dirty，后台IO线程执行的BGSAVE很重要，它会根据时间和修改过的key的数目来刷数据到rdb中。而如果开启了AOF还需要刷新AOF文件到磁盘。`feedAppendOnlyFile` 是将客户端命令写入到AOF文件中，所以可以通过AOF文件重放来恢复数据。当然这里会对`expire，setex`等命令做些转换，将过期时间设置为绝对时间，添加必要的`SELECT db`的命令等。另外则是直接通过 `catAppendOnlyGenericCommand()` 将命令写入到AOF文件。

如果有slave连接到该服务器，则通过 `replicationFeedSlaves()` 将命令发给slave服务器(slave同步时master会先通过BGSAVE保存一份rdb并发送给slave，后续的命令同步则由replicationFeedSlaves()来完成)。如果有客户端通过 `monitor` 命令连接到该服务器，则还要通过 `replicationFeedMonitors()` 发送命令字符串过去，并带上命令时间戳。


# 3 总结
这篇文章主要对redis的初始化流程做了分析，包括启动时配置初始化，使用epoll支持高并发，读取解析和响应命令等，流程图如下(来自 Redis：under the hood 博客)。这里还有篇 Paul Smith的大作 [More Redis internals: Tracing a GET & SET](https://pauladamsmith.com/blog/2011/03/redis_get_set.html) 用于跟踪 redis 的 `GET/SET` 命令流程的，值得一看。

![Redis启动流程](https://upload-images.jianshu.io/upload_images/286774-8237f4462e90fe9b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 参考资料
- https://pauladamsmith.com/articles/redis-under-the-hood.html (基本是对这篇文章的翻译加注解)
- https://redis.io/topics/protocol (redis协议，看了可以比较深入理解redis客户端和服务端的交互)
