# redis事件分析
本文会结合源码(所阅读的代码版本v5.0，6.0支持多线程了，等我先把单线程版本摸透了再研究 -_-|| )，分析redis的在整个事件循环中主要做了什么（包含但不仅限于，本文所述忽略集群、主从复制、sentinel... 等多项特性的实现并没有分析）

以及详细地描述redis收到了一个来自client的命令后是如何处理的。

本文的所有分析均基于假设读者的redis服务部署在支持epoll的操作系统中，redis通过条件编译决定使用何种IO多路复用模型，源码如下：
```
/* Include the best multiplexing layer supported by this system.
 * The following should be ordered by performances, descending. */
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

## 1. 事件循环
进入时间循环的主函数aeMain中，可以看到有 beforeSleep 和 aeProcessEvents 这两个函数，afterSleep 在 aeProcessEvents 被调用
```
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

### 1.1 (beforeSleep)
* 淘汰过期的key
* unblock client
* 处理client中的写入buffer
  * 判断buffer是否能解析成一条完整的命令
  * 将buffer解析成命令，写入client成员变量argv中
  * 将解析完成的命令加入待执行队列中, 写入client成员变量mstate.commands的尾部
* 释放 modules 锁 (moduleReleaseGIL), redis框架对数据已经完成读写，允许用户自行实现的模块进行操作

### 1.2 事件处理(aeProcessEvents)
* 根据定时器中最近任务与当前时间的差，求出休眠时间间隔
* 根据上述求得的时间间隔，传入内核api epoll_wait中，若无网络IO事件唤醒进程，则会休眠指定时间
  * 关于该api的详细描述epoll_wait() api manual page: http://www.man7.org/linux/man-pages/man2/epoll_wait.2.html
  * 结合IO多路复用的休眠实现如下：
```
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    if (retval > 0) {
        int j;

        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    return numevents;
}
```
* afterSleep: 加 modules 锁(moduleAcquireGIL), 主要是担心用户自己实现的模块与redis框架代码并发读写数据导致崩溃
* 处理网络IO读事件
  * 接受连接，处理Tcp连接(acceptTcpHandler)、处理Unix domain socket (acceptUnixHandler)
  * 读来自客户端的查询(readQueryFromClient)
* 处理网络IO写事件(sendReplyToClient)
* 处理定时事件(aeTimeEvent)
  * 定时器的设计为双向链表
```
/* Time event structure */
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc;
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *prev;
    struct aeTimeEvent *next;
} aeTimeEvent;
```
  * 每次执行该类事件时，遍历整个链表，当发现节点中存在预期执行事件小于当时事件的任务时立即执行。代码如下：
```
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
te = te->next;
```
  * 题外话：腾讯面试题有一条为设计一个日调用量过亿的定时器，你会用到哪些数据结构？请务必不要使用该答案，因为效率不高。之所以redis使用该数据结构作为定时器，是因为redis在运行过程中仅会创建少量的定时任务，是应用场景和数据体量决定的，根据木桶原理，因为无法成为性能瓶颈，所以也没有优化的必要。但我们要知道，当数据体量巨大的时候,每次为o(n)的执行效率仍很容易成为瓶颈。

## 2. 如何处理来自client的命令
本章节内容基于读者已经知悉redis事件循环中的基本工作

假设收到了一条来自客户端 `get hahaha` 的命令，redis-server会作出以下处理:
* 由网络IO触发读事件回调函数 `readQueryFromClient(eventLoop,fd,fe->clientData,mask);`
  * 将从socket fd读到的数据放到 client->querybuf 的末尾
  * 对client->querybuf进行解析 `processInputBufferAndReplicate(c);`
  * 对解析成的命令进行处理`processCommand(c)`
  * 调用`call(c,CMD_CALL_FULL);`执行命令。
  * 执行回调函数`getCommand(c)`, 实现逻辑如下：
```
int getGenericCommand(client *c) {
    robj *o;

    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.nullbulk)) == NULL)
        return C_OK;

    if (o->type != OBJ_STRING) {
        addReply(c,shared.wrongtypeerr);
        return C_ERR;
    } else {
        addReplyBulk(c,o);
        return C_OK;
    }
}

void getCommand(client *c) {
    getGenericCommand(c);
}
```

  * 将命令的执行结果放到c->buf的末尾, 实现如下：
```
int _addReplyToBuffer(client *c, const char *s, size_t len) {
    size_t available = sizeof(c->buf)-c->bufpos;

    if (c->flags & CLIENT_CLOSE_AFTER_REPLY) return C_OK;

    /* If there already are entries in the reply list, we cannot
    * add anything more to the static buffer. */
    if (listLength(c->reply) > 0) return C_ERR;

    /* Check that the buffer has enough space available for this string. */
    if (len > available) return C_ERR;

    memcpy(c->buf+c->bufpos,s,len);
    c->bufpos+=len;
    return C_OK;
}
```
  * 至此，由网络IO回调的整个读事件已完成
* 由网络IO触发写事件回调函数`sendReplyToClient()`, 将`c->buf`中的数据通过发回给客户端。

## 3. 为什么redis是单线程的效率依旧很高
* 基于内存的kv服务，天然具备效率优势，以hash字典存储数据，查询效率为o(1)。

* 使用 MurmurHash2 算法来计算键的哈希值，这种算法的优点在于，即使输入的键是有规律的，算法仍能给出一个很好的随机分布性，降低产生冲突的可能性，并且算法的计算速度也非常快。

* 数据结构的选型合理高效，这个太厉害了，有空重新写一篇文章来分析这个点；TODO: 写一篇新文章。

* 单线程程序，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗CPU，不用去考虑各种锁的问题，减小了用户态及内核态的频繁切换；
  
* 使用多路I/O复用模型，非阻塞IO；

* 自定义的通讯协议简单，解析效率高，判断一个命令是否完整，仅需判断数据中是否包含`\r\n`, 若存在即可解析并执行
```
newline = strchr(c->querybuf+c->qb_pos,'\r');
if (newline == NULL) {
    if (sdslen(c->querybuf)-c->qb_pos > PROTO_INLINE_MAX_SIZE) {
        addReplyError(c,"Protocol error: too big mbulk count string");
        setProtocolError("too big mbulk count string",c);
    }
    return C_ERR;
}

/* Buffer should also contain \n */
if (newline-(c->querybuf+c->qb_pos) > (ssize_t)(sdslen(c->querybuf)-c->qb_pos-2))
    return C_ERR;
```

* 代码简洁高效, 判断参数时不是比较整个字符串，而是仅仅判断首个字符。总所周知，这是个o(n)变o(1)的骚操作。示例如下，判断NX, XX时都只判断首字幕：
```
/* SET key value [NX] [XX] [EX <seconds>] [PX <milliseconds>] */
void setCommand(client *c) {
    int j;
    robj *expire = NULL;
    int unit = UNIT_SECONDS;
    int flags = OBJ_SET_NO_FLAGS;

    for (j = 3; j < c->argc; j++) {
        char *a = c->argv[j]->ptr;
        robj *next = (j == c->argc-1) ? NULL : c->argv[j+1];

        if ((a[0] == 'n' || a[0] == 'N') &&
            (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' &&
            !(flags & OBJ_SET_XX))
        {
            flags |= OBJ_SET_NX;
        } else if ((a[0] == 'x' || a[0] == 'X') &&
                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' &&
                   !(flags & OBJ_SET_NX))
        {
            flags |= OBJ_SET_XX;
        } else if ((a[0] == 'e' || a[0] == 'E') &&
                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' &&
                   !(flags & OBJ_SET_PX) && next)
        {
            flags |= OBJ_SET_EX;
            unit = UNIT_SECONDS;
            expire = next;
            j++;
        } else if ((a[0] == 'p' || a[0] == 'P') &&
                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' &&
                   !(flags & OBJ_SET_EX) && next)
        {
            flags |= OBJ_SET_PX;
            unit = UNIT_MILLISECONDS;
            expire = next;
            j++;
        } else {
            addReply(c,shared.syntaxerr);
            return;
        }
    }

    c->argv[2] = tryObjectEncoding(c->argv[2]);
    setGenericCommand(c,flags,c->argv[1],c->argv[2],expire,unit,NULL,NULL);
}
```

* 综上所述，确实有点厉害，所以人家这么快。