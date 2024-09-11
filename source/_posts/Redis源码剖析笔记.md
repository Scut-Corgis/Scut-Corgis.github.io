---
title: Redis源码剖析笔记
date: 2023-08-01 22:30:00
categories:
    - 协程
---

Redis源码剖析笔记

# 前言

**笔记来源**

1. Redis设计与实现
2. Redis核心技术与实战   -极客时间
3. **Redis源码剖析与实战**   -极客时间
4. **Redis5.0.8源码**
4. https://blog.csdn.net/gqtcgq?type=blog我看过的把源码每个模块执行流程写的最清楚的博主

强烈建议把上面的学习步骤完整走一步，理解一定会更深。当然，如果没有也没有关系，但至少要学过《Redis设计与实现》，redis版本很老但是也给了一个笼统的印象，是一本不错的redis入门读物。

阅读前请下载**Redis5.0.8源码**。

> https://redis.io/download/#redis

那么就进入Redis源码的世界吧。

> Redis源码版本5.0.8   多线程部分为6.0.12

**其他**

**小林coding**的Redis文章（*比较好的补充*）

(1) 缓存雪崩、击穿、穿透

(9) 为了拿捏Redis数据结构，我画了40张图（完整版）

(12) Redis数据类型与应用场景

(15）告别Redis!

# 全景图



![](image-20220311171244891.png)

![](image-20220311171303636.png)

单线程： 网络I/O和键值对读写是单线程

## Reactor核心框架

​	`ae.c `

```c
/* 初始化EventLoop */
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    
	...
    /* 这里发现events最初就分配了空间，setsize大小默认为10000多 */
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
    ...
    /* 所有的事件都初始化为空 */
    eventLoop->events[i].mask = AE_NONE;
    return eventLoop;
}

/* 将对应的fd注册到EPOLL中，mask为监测的事件类型，proc为回调函数 */
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    ...
	/* 取出需要注册到EPOLL的事件对应的预先分配的位置 */
    aeFileEvent *fe = &eventLoop->events[fd];
	/* 这一步其实就是注册到EPOLL里 */
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    /* 设置对应的事件回调 */
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;
	...
}
/* 时间事件的注册 */
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc)
{
    long long id = eventLoop->timeEventNextId++;
    aeTimeEvent *te;

    te = zmalloc(sizeof(*te));
    if (te == NULL) return AE_ERR;
    te->id = id;
    // 时间事件距离现在多远
    aeAddMillisecondsToNow(milliseconds,&te->when_sec,&te->when_ms);
    te->timeProc = proc;
    te->finalizerProc = finalizerProc;
    te->clientData = clientData;
    te->prev = NULL;
    // 插入到链表中
    te->next = eventLoop->timeEventHead;
    if (te->next)
        te->next->prev = te;
    eventLoop->timeEventHead = te;
    return id;
}

/* Process time events */
/* 在这个函数中处理完事件后会将周期事件重新放入链表，具体来说看事件处理函数的返回值是否为-1，
不为-1 则返回值作为下次触发时机*/
static int processTimeEvents(aeEventLoop *eventLoop) {
    ...
    aeGetTime(&now_sec, &now_ms);
    if (now_sec > te->when_sec ||
        (now_sec == te->when_sec && now_ms >= te->when_ms))
    {
        int retval;

        id = te->id;
        retval = te->timeProc(eventLoop, id, te->clientData);
        processed++;
        // 返回值不为AE_NOMORE -1，则重新放入链表
        if (retval != AE_NOMORE) {
            aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
        } else {
            //否则标记为删除的
            te->id = AE_DELETED_EVENT_ID;
        }
    } 
    ...
}
```

一个发现：Redis没有使用timerfd，无法像muduo那样可以不设置epoll的超时时间，因此Redis在使用epoll_wait()之前会计算最早超时的时间事件距离现在的时间，然后用这个时间当作epoll的超时时间

> 具体函数为ae.c的 `int aeProcessEvents(aeEventLoop *eventLoop, int flags)`




```c
/* 发现Redis的EPOLL为水平触发 */
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee = {0}; /* avoid valgrind warning */
    /* If the fd was already monitored for some event, we need a MOD
     * operation. Otherwise we need an ADD operation. */
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd;
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}
```

## 网络信息回调

`networking.c`

```c
/* 监听套接字注册回调 */
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int max = MAX_ACCEPTS_PER_CALL; //一次最多接受1000个连接
    while(max--) {
        // cfd为连接套接字
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        // 调用回调函数
        acceptCommonHandler(cfd,0,cip);
    }
}
```

```c
#define MAX_ACCEPTS_PER_CALL 1000
static void acceptCommonHandler(int fd, int flags, char *ip) {
    client *c;
    /* 创建客户端对象 */
    if ((c = createClient(fd)) == NULL) {
        close(fd); /* May be already closed, just ignore errors */
        return;
    }
    /* 不能超过客户端连接数上限 */
    if (listLength(server.clients) > server.maxclients) {
        char *err = "-ERR max number of clients reached\r\n";
        freeClient(c);
        return;
    }
	//全局连接数+1
    server.stat_numconnections++;
    c->flags |= flags;
}
```

```c
client *createClient(int fd) {
    client *c = zmalloc(sizeof(client));

    if (fd != -1) {
        anetNonBlock(NULL,fd); //非阻塞
        anetEnableTcpNoDelay(NULL,fd); //关闭Negal算法
        if (server.tcpkeepalive)	//是否开启Tcp心跳探测
            anetKeepAlive(NULL,fd,server.tcpkeepalive);
        if (aeCreateFileEvent(server.el,fd,AE_READABLE,
            readQueryFromClient, c) == AE_ERR)	//注册读回调
        {
            close(fd);
            zfree(c);
            return NULL;
        }
    }
	... // 省略了client内部结构字段初始化代码
    if (fd != -1) linkClient(c);
    initClientMultiState(c);
    return c;
}
```

```c
// 插入至server的链表中
void linkClient(client *c) {
    listAddNodeTail(server.clients,c);
    c->client_list_node = listLast(server.clients);
    // 将客户端ID插入到前缀树中
    uint64_t id = htonu64(c->id);
    raxInsert(server.clients_index,(unsigned char*)&id,sizeof(id),c,NULL);
}
```

```c
/* 收到客户端信息时的回调 */
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    client *c = (client*) privdata; //注意把client指针放到了epoll_event的用户数据里了
    int nread, readlen;
    size_t qblen;
    readlen = PROTO_IOBUF_LEN; //16KB
	...
	/* 每次输入缓冲区扩容16KB */
    qblen = sdslen(c->querybuf);
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    nread = read(fd, c->querybuf+qblen, readlen); //读套接字
    
    if (nread == -1) {
        if (errno == EAGAIN) {
            return;
        } else {
            serverLog(LL_VERBOSE, "Reading from client: %s",strerror(errno));
            freeClient(c);
            return;
        }
    } else if (nread == 0) {
        serverLog(LL_VERBOSE, "Client closed connection");
        freeClient(c);
        return;
    } else if (c->flags & CLIENT_MASTER) {
        /* Append the query buffer to the pending (not applied) buffer
         * of the master. We'll use this buffer later in order to have a
         * copy of the string applied by the last command executed. */
        c->pending_querybuf = sdscatlen(c->pending_querybuf,
                                        c->querybuf+qblen,nread);
    }

    sdsIncrLen(c->querybuf,nread);
    // 更新client结构信息
    c->lastinteraction = server.unixtime;
    if (c->flags & CLIENT_MASTER) c->read_reploff += nread;
    // redis的统计信息
    server.stat_net_input_bytes += nread;
    // 请求长度太长，超了，直接释放客户端对象
    if (sdslen(c->querybuf) > server.client_max_querybuf_len) {
        sds ci = catClientInfoString(sdsempty(),c), bytes = sdsempty();
        bytes = sdscatrepr(bytes,c->querybuf,64);
        serverLog(LL_WARNING,"Closing client that reached max query buffer length: %s (qbuf initial bytes: %s)", ci, bytes);
        sdsfree(ci);
        sdsfree(bytes);
        freeClient(c);
        return;
    }
    /* 进一步处理输入缓冲区数据 */
    processInputBufferAndReplicate(c);
}
```

## 通用命令执行流程

```c
/* 收到客户端信息时调用回调 */
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
	...
    processInputBufferAndReplicate(c);
}
/* 处理inputBuffer,按照是否为主从复制的主节点走不同的逻辑 */
void processInputBufferAndReplicate(client *c) {
    if (!(c->flags & CLIENT_MASTER)) {
        processInputBuffer(c);
    } else {
        size_t prev_offset = c->reploff;
        processInputBuffer(c);
        size_t applied = c->reploff - prev_offset;
        if (applied) {
            replicationFeedSlavesFromMasterStream(server.slaves,
                    c->pending_querybuf, applied);
            sdsrange(c->pending_querybuf,applied,-1);
        }
    }
}
/* 命令解析,实际就是将解析后的命令名和参数依次放入 client->argv[]中 */
void processInputBuffer(client *c) {
    server.current_client = c;
		...
        /* Multibulk processing could see a <= 0 length. */
        if (c->argc == 0) {
            resetClient(c);
        } else {
            /* Only reset the client when the command was executed. */
            if (processCommand(c) == C_OK) {
				...
            }
			...
        }
    }
	...
}
/* 对命令进行处理，比如参数不对返回错误等等 */
int processCommand(client *c) {
     ...
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        queueMultiCommand(c); //如果时MULTI命令，将命令入队
        addReply(c,shared.queued);
    } else {
        call(c,CMD_CALL_FULL); //真正执行命令的位置
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnKeys();
    }
    return C_OK;
}
/* 执行命令 */
void call(client *c, int flags) {
	...
    struct redisCommand *real_cmd = c->cmd;
	...
    start = server.ustime;
    c->cmd->proc(c); //对应命令回调函数调用
    duration = ustime()-start;
}
/* 命令执行函数的回复 */
// redisClient结构中有两种客户端输出缓存，一种是静态大小的数组(buf)，一种是动态大小的列表(reply)。追加回复信息时，首先尝试将信息追加到数组buf中，如果其空间不足，则将信息在追加到reply中。
void addReply(client *c, robj *obj) {
    // 有的客户端无需回复，这一步进行分类讨论
    // 有些客户端（比如Lua客户端）需要追加新数据，但无需注册socket描述符上的可写事件；有些客户端（普通客户端）需要追加数据，并注册socket描述符上的可写事件；
    if (prepareClientToWrite(c) != REDIS_OK) return;
    if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK)
        _addReplyStringToList(c,obj->ptr,sdslen(obj->ptr));
}
```

```c
/* This function is called just before entering the event loop, in the hope
 * we can just write the replies to the client output buffer without any
 * need to use a syscall in order to install the writable event handler,
 * get it called, and so forth. */
// 即在eventloop执行前执行，会遍历输出缓冲区有需要发送客户端，对其发送，发送不完才注册可写事件，以减少系统调用
// 而在muduo中，send会直接发，不会先放缓冲区
int handleClientsWithPendingWrites(void) {
    listIter li;
    listNode *ln;
    int processed = listLength(server.clients_pending_write);
	// 迭代器的next为头节点
    listRewind(server.clients_pending_write,&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;
        listDelNode(server.clients_pending_write,ln);
        
		...

        /* Try to write buffers to the client socket. */
        if (writeToClient(c->fd,c,0) == C_ERR) continue;

        /* If after the synchronous writes above we still have data to
         * output to the client, we need to install the writable handler. */
        if (clientHasPendingReplies(c)) {
            int ae_flags = AE_WRITABLE;
			...
             /* 创建可写事件 */
            if (aeCreateFileEvent(server.el, c->fd, ae_flags,
                sendReplyToClient, c) == AE_ERR)
            {
                    freeClientAsync(c);
            }
        }
    }
    return processed;
}
```

# 1 底层数据类型

**redisObject**

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;	/* 对象的引用计数 */
    void *ptr;
} robj;
```



## 1.1 SDS

> 小于44字节，嵌入式字符串，object对象结构和SDS结构连在一起，占用连续的内存空间，减少内存碎片。

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;
    uint8_t alloc;
    unsigned char flags; //SDS类型，即sdshdr8
    char buf[]
}
```

## 1.2 哈希表

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```



```c
typedef struct dictht {
    dictEntry **table; //二维数组
    unsigned long size; //Hash表大小
    unsigned long sizemask;
    unsigned long used;	//存放的节点个数
} dictht;
```

```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
	} v;
    struct dictEntry *next;
} dictEntry;
```

**rehash时机：**

```c
static int _dictExpandIfNeeded(dict *d)
{
	...
    /* 初始化为大小DICT_HT_INITIAL_SIZE，大小为4 */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);
	/* 可以重哈希的时机 */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2); //每次只会两倍大小
    }
    return DICT_OK;
}
/* 源码注释说的很清楚何时是can_resize，就是防止父进程产生不必要的copy-on-write
 * 即没有RDB子进程，并且也没有AOF子进程 */
/* Using dictEnableResize() / dictDisableResize() we make possible to
 * enable/disable resizing of the hash table as needed. This is very important
 * for Redis, as we use copy-on-write and don't want to move too much memory
 * around when there is a child performing saving operations.
 *
 * Note that even when dict_can_resize is set to 0, not all resizes are
 * prevented: a hash table is still allowed to grow if the ratio between
 * the number of elements and the buckets > dict_force_resize_ratio. */
static int dict_can_resize = 1;
static unsigned int dict_force_resize_ratio = 5;
```



1. 负载因子大于等于1并且可以扩容时。

   > 启用扩容功能的条件是：当前没有 RDB 子进程，并且也没有 AOF 子进程。

2. 负载因子大于等于5。

> 扩容至两倍大小，而C++是小于当前两倍的最大质数

渐进式哈希：

在每次增删查改的操作时，迁移一个桶，迁移的桶index++，直到迁移完所有的桶。原表找不到数据就去新表找。

**在时间事件中会用给每一个数据库一毫秒时间重哈希**,both 数据库表和expire表，当然也不有用AOF或RDB子进程

```c
/* databasesCron(void) */
...  
if (server.activerehashing) {
    for (j = 0; j < dbs_per_call; j++) {
        int work_done = incrementallyRehash(rehash_db);
        if (work_done) {
            /* If the function did some work, stop here, we'll do
                     * more at the next cron loop. */
            break;
        } else {
            /* If this db didn't need rehash, we'll try the next one. */
            rehash_db++;
            rehash_db %= server.dbnum;
        }
    }
}
...
/* Our hash table implementation performs rehashing incrementally while
 * we write/read from the hash table. Still if the server is idle, the hash
 * table will use two tables for a long time. So we try to use 1 millisecond
 * of CPU time at every call of this function to perform some rehahsing.
 *
 * The function returns 1 if some rehashing was performed, otherwise 0
 * is returned. */
int incrementallyRehash(int dbid) {
    /* Keys dictionary */
    if (dictIsRehashing(server.db[dbid].dict)) {
        dictRehashMilliseconds(server.db[dbid].dict,1);
        return 1; /* already used our millisecond for this loop... */
    }
    /* Expires */
    if (dictIsRehashing(server.db[dbid].expires)) {
        dictRehashMilliseconds(server.db[dbid].expires,1);
        return 1; /* already used our millisecond for this loop... */
    }
    return 0;
}
```



反向重哈希以节省内存： 在时间事件中会检查每个数据库的哈希表，若桶数过多，则为节省内存会缩小桶数

```c
/* If the percentage of used slots in the HT reaches HASHTABLE_MIN_FILL
 * we resize the hash table to save memory */
void tryResizeHashTables(int dbid) {
    if (htNeedsResize(server.db[dbid].dict))
        dictResize(server.db[dbid].dict);
    if (htNeedsResize(server.db[dbid].expires))
        dictResize(server.db[dbid].expires);
}
int htNeedsResize(dict *dict) {
    long long size, used;

    size = dictSlots(dict);
    used = dictSize(dict);
    return (size > DICT_HT_INITIAL_SIZE &&
            (used*100/size < HASHTABLE_MIN_FILL));
}
```



## 1.3 压缩列表

压缩链表：前三个字段：链表总长度、表尾偏移量、节点个数

​					数据entry前三个字段： prev_len  encoding content

> prevlen 1字节编码和5字节编码两种方法，<254时采用5字节
>
> encoding 字段不仅有编码类型信息，还有具体数据长度信息

![](image-20220317202631805.png)

```c
/* 编码encoding字段， *p指向encoding第一个字节, rawlen为数据长度 */
unsigned int zipStoreEntryEncoding(unsigned char *p, unsigned char encoding, unsigned int rawlen) {
    unsigned char len = 1, buf[5];
	// 如果是字符串，根据字符串的长度选择不同的编码方式
    if (ZIP_IS_STR(encoding)) {
        if (rawlen <= 0x3f) {
            if (!p) return len;
            buf[0] = ZIP_STR_06B | rawlen;
        } else if (rawlen <= 0x3fff) {
            len += 1;
            if (!p) return len;
            buf[0] = ZIP_STR_14B | ((rawlen >> 8) & 0x3f);
            buf[1] = rawlen & 0xff;
        } else {
            len += 4;
            if (!p) return len;
            buf[0] = ZIP_STR_32B;
            buf[1] = (rawlen >> 24) & 0xff;
            buf[2] = (rawlen >> 16) & 0xff;
            buf[3] = (rawlen >> 8) & 0xff;
            buf[4] = rawlen & 0xff;
        }
    } else {
        /* 默认为int类型整数. */
        if (!p) return len;
        buf[0] = encoding;
    }

    /* Store this length at p. */
    memcpy(p,buf,len);
    return len;
}
```



## 1.4 整数集合

```c
typedef struct intset {
    uint32_t encoding; //编码方式即底层int16 int32 int64
    uint32_t length;	//元素个数
    int8_t contents[];
} intset;
```

## 1.5 双向链表

一个头节点，一个尾节点，无环

注意头节点和尾节点在list为空时是指向null，没有虚拟节点

```c
list *listCreate(void)
{
    struct list *list;

    if ((list = zmalloc(sizeof(*list))) == NULL)
        return NULL;
    list->head = list->tail = NULL;
    list->len = 0;
    list->dup = NULL;
    list->free = NULL;
    list->match = NULL;
    return list;
}
```



## 1.6 quicklist

一个quicklist就是一个链表，而链表中的每个元素又是一个ziplist压缩列表

![](image-20220317220554800.png)

插入操作： 提供对应的quicklistNode和在ziplist中的偏移量

插入时会检查：插入新的数据后，单个ziplist是否不超过8KB，或者单个ziplist的元素个数是否满足要求，只要一个条件满足便可以插入新元素。否则会新建一个节点。

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

/* quicklist is a 40 byte struct (on 64-bit systems) describing a quicklist.
 * 'count' is the number of total entries.
 * 'len' is the number of quicklist nodes.
 * 'compress' is: -1 if compression disabled, otherwise it's the number
 *                of quicklistNodes to leave uncompressed at ends of quicklist.
 * 'fill' is the user-requested (or default) fill factor. */
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : 16;              /* fill factor for individual nodes */
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;

typedef struct quicklistIter {
    const quicklist *quicklist;
    quicklistNode *current;
    unsigned char *zi;
    long offset; /* offset in current ziplist */
    int direction;
} quicklistIter;

typedef struct quicklistEntry {
    const quicklist *quicklist;
    quicklistNode *node;
    unsigned char *zi;  //指向压缩链表对应的节点
    unsigned char *value;
    long long longval;
    unsigned int sz;
    int offset;
} quicklistEntry;
// quicklist插入API 是传对应的位置信息quicklistEntry，和实际插入的value
REDIS_STATIC void _quicklistInsert(quicklist *quicklist, quicklistEntry *entry,
                                   void *value, const size_t sz, int after)
```



## 1.7 跳表

**跳表的节点插入**

插入新节点本质上就是一个**维护各层指针和跨度**的过程。先找到在每层的插入位置，并保存在update数组中，同时将头节点到该位置的跨度累加，保存在rank数组中。最后计算随机高度，在每层插入节点。

1. 从最高层开始，延着后继指针找到每一层应该在的位置，累计跨度和为rank，用update数组保存每一层的前驱节点
2. update数组完全生成后，相当于就找到了要插入节点的每一层的前节点
3. 新节点的高度会随机生成，以0.25的概率。假如生成了一个比当前最高level还高的高度，那么就是直接将头节点的forward指针放入update数组，并更新跳表数据结构的level
4. 然后根据之前的update数组维护的每一层的前驱节点，根据新节点的rank排名和原来的跨度，更新前驱节点的forward指针和跨度，以及新插入节点的forward指针和跨度

```c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned int span;//代表该节点在每层到下一个节点所跨越的节点长度
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```



```c
zskiplistNode *zslCreateNode(int level, double score, robj *obj) {
    zskiplistNode *zn = zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->obj = obj;
    return zn;
}

zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
/* Returns a random level for the new skiplist node we are going to create.
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. */
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}

zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    redisAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                compareStringObjects(x->level[i].forward->obj,obj) < 0))) {
            rank[i] += x->level[i].span;//累加本层从头节点到插入位置节点的跨度综合
            x = x->level[i].forward;
        }
        update[i] = x;//得到每层的插入位置节点
    }
    level = zslRandomLevel();
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    x = zslCreateNode(level,score,obj);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);//update[i]->level[i].span - 0层和i层的update[i]之间的距离
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;//新增一个节点在后面，所以跨度加一
    }

    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {//如果新节点的层数小于表的level，将updata[i]->level[i]的span++
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}

int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* 不仅分数相等，字符串也应该相等 */
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}
```

## 1.8 listpack

Redis5.0引入，用于更换压缩链表。

**解决了连锁更新的问题，但又能实现从后向前遍历。**

![](image-20220317225732624.png)

![](image-20220317225744425.png)

**从头向尾遍历：**

和压缩链表基本相同。

因为编码类型`entry-encoding`隐式含有长度信息，比如整数编码自然知道是多少字节的整数，字符串编码的话会带有字符串长度的信息。

**从尾向头遍历：**

首先通过整个listpack总长度，跳到最后的结束符位置，然后往前一个字节一个字节的读，直到读到entry-len的结束标定，然后将最后一个entry的长度计算出来，就可以继续往前偏移这么多长度实现往前一个元素遍历。

核心在于entry最后的entry-len的设计，它是编码类型+数据的总长度。

![](image-20220317230231626.png)

entry-len从头往前一个字节一个字节读的时候，它每个字节的第一个bit如果为1表示，表示entry-len还没有结束，左边的字节仍然还是entry-len的内容。

当读到了某个字节的第一个bit为0，就表示这是最后一个字节了。最后将这些字节每后7个bit位拼接起来，就组成了entry-len。

# 2 AOF和RDB

## 2.1 AOF

&emsp;&emsp;在Redis中，执行命令**后**，会将AOF日志写入内存中全局变量中的aof_buf中，此时不会刷盘，在整个eventloop执行完后，在下一次进入eventLoop之前，会调用beforeSleep回调，该回调会将根据AOF刷盘策略将aof_buf刷盘。

&emsp;&emsp;这样其实是crash-safe的，因为redis的独特设计是一个eventloop中不会直接把客户端命令执行后的reply立马发送回去，而是放到输出缓冲区中，在下一个eventLoop中才发送出去，因此客户端收到回复时，aof已经刷盘了。

&emsp;&emsp;这样最大的好处就是集中处理，不仅使整个框架逻辑简化了，而且刷盘一次刷的更多，减少了刷盘次数。

&emsp;&emsp;所以AOF也称为**写后日志**。



**AOF文件写流程 aof.c**

```c
/* del命令写AOF的调用案例 */
if (server.aof_state != AOF_OFF)
    feedAppendOnlyFile(server.delCommand,db->id,argv,2);
```

```c
/* 根据命令的不同，追加不同的AOF记录 */
void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
    sds buf = sdsempty(); //实际的追加位置
    robj *tmpargv[3];

    /* 若对应数据库不同，增加一条切换数据库命令 */
    if (dictid != server.aof_selected_db) {
        char seldb[64];

        snprintf(seldb,sizeof(seldb),"%d",dictid);
        buf = sdscatprintf(buf,"*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
            (unsigned long)strlen(seldb),seldb);
        server.aof_selected_db = dictid;
    }
    // expire相关命令的处理
    if (cmd->proc == expireCommand || cmd->proc == pexpireCommand ||
        cmd->proc == expireatCommand) {
        /* Translate EXPIRE/PEXPIRE/EXPIREAT into PEXPIREAT */
        buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);
    } else if (cmd->proc == setexCommand || cmd->proc == psetexCommand) {
        /* Translate SETEX/PSETEX to SET and PEXPIREAT */
        tmpargv[0] = createStringObject("SET",3);
        tmpargv[1] = argv[1];
        tmpargv[2] = argv[3];
        buf = catAppendOnlyGenericCommand(buf,3,tmpargv);
        decrRefCount(tmpargv[0]);
        buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);
    } else if (cmd->proc == setCommand && argc > 3) { //set命令的处理
        int i;
        robj *exarg = NULL, *pxarg = NULL;
        /* Translate SET [EX seconds][PX milliseconds] to SET and PEXPIREAT */
        buf = catAppendOnlyGenericCommand(buf,3,argv);
        for (i = 3; i < argc; i ++) {
            if (!strcasecmp(argv[i]->ptr, "ex")) exarg = argv[i+1];
            if (!strcasecmp(argv[i]->ptr, "px")) pxarg = argv[i+1];
        }
        serverAssert(!(exarg && pxarg));
        if (exarg)
            buf = catAppendOnlyExpireAtCommand(buf,server.expireCommand,argv[1],
                                               exarg);
        if (pxarg)
            buf = catAppendOnlyExpireAtCommand(buf,server.pexpireCommand,argv[1],
                                               pxarg);
    } else {
        /* All the other commands don't need translation or need the
         * same translation already operated in the command vector
         * for the replication itself. */
        buf = catAppendOnlyGenericCommand(buf,argc,argv);
    }

    /* 追加到AOF buffer中，在下一次进入eventLoop时会flush到磁盘，然后才会在eventloop中发送给客		 * 户端回复，确保回复前写了AOF */
    if (server.aof_state == AOF_ON)
        server.aof_buf = sdscatlen(server.aof_buf,buf,sdslen(buf));

    /* If a background append only file rewriting is in progress we want to
     * accumulate the differences between the child DB and the current one
     * in a buffer, so that when the child process will do its work we
     * can append the differences to the new append only file. */
    if (server.aof_child_pid != -1)
        aofRewriteBufferAppend((unsigned char*)buf,sdslen(buf));

    sdsfree(buf);
}
```

```c
/* 最终实际追加到Buffer中调用的函数,也是AOF文件格式 */
sds catAppendOnlyGenericCommand(sds dst, int argc, robj **argv) {
    char buf[32];
    int len, j;
    robj *o;

    buf[0] = '*';
    len = 1+ll2string(buf+1,sizeof(buf)-1,argc);
    buf[len++] = '\r';
    buf[len++] = '\n';
    dst = sdscatlen(dst,buf,len);

    for (j = 0; j < argc; j++) {
        o = getDecodedObject(argv[j]);
        buf[0] = '$';
        len = 1+ll2string(buf+1,sizeof(buf)-1,sdslen(o->ptr));
        buf[len++] = '\r';
        buf[len++] = '\n';
        dst = sdscatlen(dst,buf,len);
        dst = sdscatlen(dst,o->ptr,sdslen(o->ptr));
        dst = sdscatlen(dst,"\r\n",2);
        decrRefCount(o);
    }
    return dst;
}
```

**优点：**

* 不会阻塞当前写操作
* 不会记录错误的命令，保证日志里的命令都是正确的

**写回时机：**

1. always 同步写回
2. everysec 每秒
3. No 交给操作系统

**AOF重写：**

调度时机：

* bgrewriteaof命令被执行
* 主从复制完成RDB文件解析与加载（无论是否成功）
* AOF重写被设置为待调度执行。于`ServerCron`中若没有RDB子进程会执行
* AOF启用，同时AOF文件的大小比例超过阈值，以及AOF文件大小绝对值超出阈值, 也在`ServerCron`

fork产生子进程 --> 主进程不仅继续写前一个AOF文件，还会写入AOF重写缓冲区 -->当子进程重写完毕生成新的AOF文件后将重写缓冲区的文件加上去

> 主进程修改数据会触发写时复制

```c
/* This is how rewriting of the append only file in background works:
 *
 * 1) The user calls BGREWRITEAOF
 * 2) Redis calls this function, that forks():
 *    2a) the child rewrite the append only file in a temp file.
 *    2b) the parent accumulates differences in server.aof_rewrite_buf.
 * 3) When the child finished '2a' exists.
 * 4) The parent will trap the exit code, if it's OK, will append the
 *    data accumulated into server.aof_rewrite_buf into the temp file, and
 *    finally will rename(2) the temp file in the actual file name.
 *    The the new file is reopened as the new append only file. Profit!
 */
int rewriteAppendOnlyFileBackground(void) {
    pid_t childpid;
    long long start;
    if (aofCreatePipes() != C_OK) return C_ERR; //创建三管道
    openChildInfoPipe();
    start = ustime();
    if ((childpid = fork()) == 0) {
        char tmpfile[256];
        /* Child */
        closeListeningSockets(0); //关闭套接字
		...
        if (rewriteAppendOnlyFile(tmpfile) == C_OK) { //写AOF临时文件
			...
            //AOF重写完，子进程会往管道里写表示完毕。父进程是通过子进程退出码来检测成功或失败
            sendChildInfo(CHILD_INFO_TYPE_AOF);
            exitFromChild(0);
        } else {
            exitFromChild(1);
        }
    } else {
        /* Parent */
        server.stat_fork_time = ustime()-start;
        server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
        latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);
		...
        server.aof_rewrite_scheduled = 0;
        server.aof_rewrite_time_start = time(NULL);
        server.aof_child_pid = childpid;
        updateDictResizePolicy(); //关闭Rehash
		...
        return C_OK;
    }
}
```

**实际管道沟通过程**

![](image-20220330234836169.png)

```c
/* Create the pipes used for parent - child process IPC during rewrite.
 * We have a data pipe used to send AOF incremental diffs to the child,
 * and two other pipes used by the children to signal it finished with
 * the rewrite so no more data should be written, and another for the
 * parent to acknowledge it understood this new condition. */
int aofCreatePipes(void) {
    int fds[6] = {-1, -1, -1, -1, -1, -1};
    int j;

    if (pipe(fds) == -1) goto error; /* parent -> children data. */
    if (pipe(fds+2) == -1) goto error; /* children -> parent ack. */
    if (pipe(fds+4) == -1) goto error; /* parent -> children ack. */
    /* Parent -> children data is non blocking. */
    if (anetNonBlock(NULL,fds[0]) != ANET_OK) goto error;
    if (anetNonBlock(NULL,fds[1]) != ANET_OK) goto error;
    if (aeCreateFileEvent(server.el, fds[2], AE_READABLE, aofChildPipeReadable, NULL) == AE_ERR) goto error;	//创建对应文件描述符的回调事件，注册到EPOLL中

    server.aof_pipe_write_data_to_child = fds[1];
    server.aof_pipe_read_data_from_parent = fds[0];
    server.aof_pipe_write_ack_to_parent = fds[3];
    server.aof_pipe_read_ack_from_child = fds[2];
    server.aof_pipe_write_ack_to_child = fds[5];
    server.aof_pipe_read_ack_from_parent = fds[4];
    server.aof_stop_sending_diff = 0;
    return C_OK;
	...
}
```

实际上，当 AOF 重写子进程在执行时，主进程还会继续接收和处理客户端写请求。这些写 操作会被主进程正常写入 AOF 日志文件，这个过程是由 `feedAppendOnlyFile` 函数（在 aof.c 文件中）来完成。

`feedAppendOnlyFile` 函数在执行的最后一步，会判断当前是否有 AOF 重写子进程在运 行。如果有的话，它就会调用 `aofRewriteBufferAppend` 函数（在 aof.c 文件中），如 下所示：

```c
if (server.aof_child_pid != -1) 
    aofRewriteBufferAppend((unsigned char*)buf,sdslen(buf));
```

`aofRewriteBufferAppend` 函数的作用是将参数 buf，追加写到全局变量 server 的 `aof_rewrite_buf_blocks` 这个列表中。

这里，你需要注意的是，**参数 buf 是一个字节数组**，`feedAppendOnlyFile` 函数会将主进 程收到的命令操作写入到 buf 中。而 `aof_rewrite_buf_blocks` 列表中的每个元素是 **aofrwblock 结构体类型**，这个结构体中包括了一个字节数组，大小是 AOF_RW_BUF_BLOCK_SIZE，默认值是 10MB。此外，aofrwblock 结构体还记录了字节 数组已经使用的空间和剩余可用的空间。

```c
typedef struct aofrwblock {
    unsigned long used, free; //buf数组已用空间和剩余可用空间 
    char buf[AOF_RW_BUF_BLOCK_SIZE]; //宏定义AOF_RW_BUF_BLOCK_SIZE默认为10MB
} aofrwblock;
```

这样一来，aofrwblock 结构体就相当于是一个 10MB 的数据块，记录了 AOF 重写期间主 进程收到的命令，而 aof_rewrite_buf_blocks 列表负责将这些数据块连接起来。当 aofRewriteBufferAppend 函数执行时，它会从 aof_rewrite_buf_blocks 列表中取出一个 aofrwblock 类型的数据块，用来记录命令操作。

当然，如果当前数据块中的空间不够保存参数 buf 中记录的命令操作，那么 aofRewriteBufferAppend 函数就会再分配一个 aofrwblock 数据块。

好了，当 aofRewriteBufferAppend 函数将命令操作记录到 aof_rewrite_buf_blocks 列 表中之后，它还会**检查 aof_pipe_write_data_to_child 管道描述符上是否注册了写事 件**，这个管道描述符就对应了我刚才给你介绍的 fds[1]。

如果没有注册写事件，那么 aofRewriteBufferAppend 函数就会调用 **aeCreateFileEvent 函数**，注册一个写事件，这个写事件会**监听 aof_pipe_write_data_to_child 这个管道描述 符**，也就是主进程和重写子进程间的操作命令传输管道。

当这个管道可以写入数据时，写事件对应的回调函数 aofChildWriteDiffData（在 aof.c 文 件中）就会被调用执行。

```c
void aofChildWriteDiffData(aeEventLoop *el, int fd, void *privdata, int mask) {
	...
    while(1) {
        // 逐个从列表中取出数据块
        ln = listFirst(server.aof_rewrite_buf_blocks);
        block = ln ? ln->value : NULL;
        if (block->used > 0) {
            // 调用write将数据块写入主进程和重写子进程间的数据管道
            nwritten = write(server.aof_pipe_write_data_to_child,
                             block->buf,block->used);
            if (nwritten <= 0) return;
            // 清除缓冲区，释放内存空间
            memmove(block->buf,block->buf+nwritten,block->used-nwritten);
            block->used -= nwritten;
            block->free += nwritten;
        }
        //删除数据块节点
        if (block->used == 0) listDelNode(server.aof_rewrite_buf_blocks,ln);
    }
}
```

![](image-20220330234916980.png)

> 优点：
>
> 1. 命令传过去多少，父进程就会删除对应缓冲区多少空间，减轻内存压力，因为发给子进程，子进程会写到文件中去，而不是遗留在内存中。
> 2. 将一次性全部追加到子进程的临时文件操作分散到了多次操作中，让最后子进程退出时，父进程合并操作变得小了很多，极大的减少了父进程的阻塞时间

重写子进程是由aofReadDiffFromParent函数（在 aof.c 文件中）来完成的。这个函数会 使用一个 64KB 大小的缓冲区，然后调用 read 函数，读取父进程和重写子进程间的操作命 令传输管道中的数据。以下代码也展示了 aofReadDiffFromParent 函数的基本执行流 程

```c
ssize_t aofReadDiffFromParent(void) {
    char buf[65536]; /* Default pipe buffer size on most Linux systems. */
    ssize_t nread, total = 0;

    while ((nread =
            read(server.aof_pipe_read_data_from_parent,buf,sizeof(buf))) > 0) {
        // 放入clild side的缓冲区中
        server.aof_child_diff = sdscatlen(server.aof_child_diff,buf,nread);
        total += nread;
    }
    return total;
}
```

Redis 源码在实现 AOF 重写过程中，其实会多次让重写子进程向 主进程读取新收到的操作命令，这也是为了让重写日志尽可能多地记录最新的操作，提供 更加完整的操作记录。

其实，重写子进程在执行 rewriteAppendOnlyFile 函数时，这个函数在完成日志重写，以 及多次向父进程读取操作命令后，就会调用 write 函数，向 aof_pipe_write_ack_to_parent 描述符对应的管道中**写入“！”**，这就是重写子进程向主 进程发送 ACK 信号，让主进程停止发送收到的新写操作。这个过程如下所示：

```c
int rewriteAppendOnlyFile(char *filename) { 
    ... 
    if (write(server.aof_pipe_write_ack_to_parent,"!",1) != 1) goto werr; 
    ...
}
```

一旦重写子进程向主进程发送 ACK 信息的管道中有了数据，aof_pipe_read_ack_from_child 管道描述符上注册的读事件就会被触发，也就是说，这个 管道中有数据可以读取了。那么，aof_pipe_read_ack_from_child 管道描述符上，注册的 回调函数 **aofChildPipeReadable**（在 aof.c 文件中）就会执行。

这个函数会判断从 aof_pipe_read_ack_from_child 管道描述符读取的数据是否是“！”， 如果是的话，那它就会调用 write 函数，往 aof_pipe_write_ack_to_child 管道描述符上写 入“！”，表示主进程已经收到重写子进程发送的 ACK 信息，同时它会给重写子进程回复 一个 ACK 信息。这个过程如下所示：

```c
void aofChildPipeReadable(aeEventLoop *el, int fd, void *privdata, int mask) { 
    ... 
    if (read(fd,&byte,1) == 1 && byte == '!') {
        ... 
        if (write(server.aof_pipe_write_ack_to_child,"!",1) != 1) { ...}
} 
...
}
```

最后，重写子进程执行的 rewriteAppendOnlyFile 函数，会调用 **syncRead 函数**，从 aof_pipe_read_ack_from_parent 管道描述符上，读取主进程发送给它的 ACK 信息，并最终再读一次管道数据，防止中间的AOF数据遗漏。

```c
int rewriteAppendOnlyFile(char *filename) {
    ...
    if (syncRead(server.aof_pipe_read_ack_from_parent,&byte,1,5000) != 1 ||
        byte != '!') goto werr;
    /* Read the final diff if any. */
    aofReadDiffFromParent();
    ...
}
```

![](image-20220330165701309.png)

最终子进程向信息管道写这次AOF或RDB的操作总信息(比如写入字节数)，然后退出，父进程在时间事件中捕获子进程退出码，就可以知道是否正常退出了。

```c
int rewriteAppendOnlyFileBackground(void) {
    ...
     /* child */
    if (rewriteAppendOnlyFile(tmpfile) == C_OK) {
        sendChildInfo(CHILD_INFO_TYPE_AOF);
        exitFromChild(0);
    } else {
        exitFromChild(1);
    }
}
```

```c
   /* 时间事件会定期检查是否有RDB和AOF子进程退出 */ 
	if (server.rdb_child_pid != -1 || server.aof_child_pid != -1 ||
        ldbPendingChildren())
    {
        int statloc;
        pid_t pid;

        if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) { //wait3检查子进程退出
            int exitcode = WEXITSTATUS(statloc);
            int bysignal = 0;

            if (WIFSIGNALED(statloc)) bysignal = WTERMSIG(statloc);
			/* 根据不同的事件和退出码，执行响应的回调 */
            if (pid == -1) {
				...//错误处理
            } else if (pid == server.rdb_child_pid) {
                backgroundSaveDoneHandler(exitcode,bysignal);
                if (!bysignal && exitcode == 0) receiveChildInfo();
            } else if (pid == server.aof_child_pid) {
                backgroundRewriteDoneHandler(exitcode,bysignal);
                if (!bysignal && exitcode == 0) receiveChildInfo();
            } 
            updateDictResizePolicy(); //打开rehash开关
            closeChildInfoPipe();	//关闭对应信息管道
        }
```



## **2.2 RDB**

同样采取了fork的方式生成RDB快照。

看源码只是少了创建三管道这个过程，逻辑比AOF重写简单。



# 3 主从复制

## 3.1 概述

> 从库实际上是状态机模型，采用阻塞式（同步）访问主库(发起相应命令)，比如从库会发送自己的IP、端口、支持的协议类型、复制偏移量等给主库。主库在进入BGSAVE开始创建RDB之后也会为从库维护复制状态。

> 《极客时间-Redis源码剖析与实战-21讲》

**第一次同步**

1. 主生成全量RDB快照发送给从库，并记录RDB之后的命令与 replication buffer中
2. 将replication buffer发送给从库

主-从-从模式减少主库压力，就从库也可以有从库

**replication buffer 和 replication_backlog_buffer的区别**.

1. replication buffer用于全量复制时，记录RDB文件生成传输以及被客户端执行的时间范围内产生的新的命令，主库通过创建一个连向从库的client实例将buffer发送过去，每个从库都独有一个

> ​		buffer实际是c->reply链表, 这样发过去多少就可以清理多少链表block，清理内存。换句话说，主节点执行的命令都会复制一份放入所有slave的c->reply链表中，在输出回调中发送出去。因为只要不断线，一直都是采用这个缓冲区。如果断线了，主节点会释放套接字，清理掉client结构，所以就会丢失reply，需要采用下面的循环缓冲区进行部分重同步了。

1. replication_backlog_buffer用于断线重连时的场景，从库断线后，重连上时会发送自己的复制偏移量给主库，若还在缓冲区内，主库则将这部分命令发送过去。此缓冲区各从库共享。

**网络断开连接后**

	1. 主库中存在一个环形缓冲区，记录了主库写偏移量和从库读偏移量，两偏移量之差就是从库落后的日志部分
	2. 从库连上连接后，会将当前自己读偏移发送给主库，主库将中间部分发送给从库
	3. 若环形缓冲区溢出，则全量复制，为了避免全量复制，可以适当调大环形缓冲区

在redis中，主库负责写，从库可以处理读事件。

**redis读从库读过期数据：**

3.2版本前会返回过期数据，3.2版本后会返回空值。

因为从库可能落后主库，所以可能读到旧数据。



**源码部分**

在Redis源码中，表示Redis服务器的全局结构体struct redisServer  server中，与主从复制相关的，从节点属性如下：

```sql
server.masterhost：记录主节点的ip地址；

server.masterport：记录主节点的端口号；

server.repl_transfer_s：socket描述符，用于主从复制过程中，从节点与主节点之间的TCP通信，包括主从节点间的握手通信、接收RDB数据，以及后续的命令传播过程；

server.repl_transfer_fd：文件描述符，用于从节点将收到的RDB数据写到本地临时文件；

server.repl_transfer_tmpfile：从节点上，用于记录RDB数据的临时文件名；

server.repl_state：记录主从复制过程中，从节点的状态。

/* server.master是个client* 结构 */
server.master：当从节点接受完主节点发来的RDB数据之后，进入命令传播过程。从节点就将主节点当成一个客户端看待。server.master就是redisClient结构的主节点客户端，从节点接收该server.master发来的命令，像处理普通客户端的命令请求一样进行处理，从而实现了从节点和主节点之间的同步。

server.master->reploff：从节点记录的复制偏移量，每次收到主节点发来的命令时，就会将命令长度增加到该复制偏移量上，以保持和主节点复制偏移量的一致。

server.master->replrunid：从节点记录的主节点运行ID。

server.cached_master：主从节点复制过程中（具体应该是命令传播过程中），如果从节点与主节点之间连接断掉了，会调用freeClient(server.master)，关闭与主节点客户端的连接。为了后续重连时能够进行部分重同步，在freeClient中，会调用replicationCacheMaster函数，将server.master保存到server.cached_master。该redisClient结构中记录了主节点的运行ID，以及复制偏移。当后续与主节点的连接又重新建立起来的时候，使用这些信息进行部分重同步，也就是发送"PSYNC  <runid>  <offset>"命令。

server.repl_master_runid和server.repl_master_initial_offset：从节点发送"PSYNC  <runid> <offset>"命令后，如果主节点不支持部分重同步，则会回复信息为"+FULLRESYNC <runid>  <offset>"，表示要进行完全重同步，其中<runid>表示主节点的运行ID，记录到server.repl_master_runid中，<offset>表示主节点的初始复制偏移，记录到server.repl_master_initial_offset中。
```

## 3.2 从节点部分

### **一. TCP建链**

&emsp;&emsp;在Redis源码中，使用server.repl_state记录从节点的状态。在Redis初始化时，该状态为REDIS_REPL_NONE。

&emsp;&emsp;当从节点收到客户端用户发来的”SLAVEOF” 命令时，或者在读取配置文件，发现了”slaveof”配置选项，就会将server.repl_state置为REDIS_REPL_CONNECT状态。该状态表示下一步需要向主节点发起TCP建链。

&emsp;&emsp;在定时执行的函数serverCron中，会调用replicationCron函数检查主从复制的状态。该函数中，一旦发现当前的server.repl_state为REDIS_REPL_CONNECT，则会调用函数connectWithMaster，向主节点发起TCP建链请求，其代码如下：

```c
int connectWithMaster(void) {
    int fd;

    fd = anetTcpNonBlockBestEffortBindConnect(NULL,
        server.masterhost,server.masterport,NET_FIRST_BIND_ADDR);
    if (aeCreateFileEvent(server.el,fd,AE_READABLE|AE_WRITABLE,syncWithMaster,NULL) == AE_ERR) {
        close(fd);
        return C_ERR;
    }
    server.repl_transfer_lastio = server.unixtime;
    server.repl_transfer_s = fd;
    server.repl_state = REPL_STATE_CONNECTING;
    return C_OK;
}
```

&emsp;&emsp;server.masterhost和server.masterport分别记录了主节点的IP地址和端口号。它们要么是在slaveof选项中配置，要么是”SLAVEOF”命令中的参数。

&emsp;&emsp;首先调用`anetTcpNonBlockBestEffortBindConnect`，向主节点发起connect建链请求；该函数创建socket描述符，将该描述符设置为非阻塞，必要情况下会绑定本地地址，然后connect向主节点发起TCP建链请求。该函数返回建链的socket描述符fd；

&emsp;&emsp;然后注册socket描述符fd上的可读和可写事件，事件回调函数都为`syncWithMaster`，该函数用于处理主从节点间的握手过程；      

&emsp;&emsp;然后将socket描述符记录到server.repl_transfer_s中。**置主从复制状态server.repl_state为REDIS_REPL_CONNECTING**，表示从节点正在向主节点建链；

### **二. 复制握手**

&emsp;&emsp;当主从节点间的TCP建链成功之后，就会触发socket描述符server.repl_transfer_s上的可写事件，从而调用函数`syncWithMaster`。该函数处理从节点与主节点间的握手过程。也就是从节点在向主节点发送TCP建链请求，到接收RDB数据之前的过程。

```c
// 完整的状态转移过程 non blocking connect
void syncWithMaster(aeEventLoop *el, int fd, void *privdata, int mask) {
    char tmpfile[256], *err = NULL;
    int dfd = -1, maxtries = 5;
    int sockerr = 0, psync_result;
    socklen_t errlen = sizeof(sockerr);

    /* If this event fired after the user turned the instance into a master
     * with SLAVEOF NO ONE we must just return ASAP. */
    if (server.repl_state == REPL_STATE_NONE) {
        close(fd);
        return;
    }

    /* Check for errors in the socket: after a non blocking connect() we
     * may find that the socket is in error state. */
    if (getsockopt(fd, SOL_SOCKET, SO_ERROR, &sockerr, &errlen) == -1)
        sockerr = errno;
    if (sockerr) {
        serverLog(LL_WARNING,"Error condition on socket for SYNC: %s",
            strerror(sockerr));
        goto error;
    }

    /* Send a PING to check the master is able to reply without errors. */
    if (server.repl_state == REPL_STATE_CONNECTING) {
        serverLog(LL_NOTICE,"Non blocking connect for SYNC fired the event.");
        /* Delete the writable event so that the readable event remains
         * registered and we can wait for the PONG reply. */
        aeDeleteFileEvent(server.el,fd,AE_WRITABLE);
        server.repl_state = REPL_STATE_RECEIVE_PONG;
        /* Send the PING, don't check for errors at all, we have the timeout
         * that will take care about this. */
        err = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"PING",NULL);
        if (err) goto write_error;
        return;
    }

    /* Receive the PONG command. */
    if (server.repl_state == REPL_STATE_RECEIVE_PONG) {
        err = sendSynchronousCommand(SYNC_CMD_READ,fd,NULL);

        /* We accept only two replies as valid, a positive +PONG reply
         * (we just check for "+") or an authentication error.
         * Note that older versions of Redis replied with "operation not
         * permitted" instead of using a proper error code, so we test
         * both. */
        if (err[0] != '+' &&
            strncmp(err,"-NOAUTH",7) != 0 &&
            strncmp(err,"-ERR operation not permitted",28) != 0)
        {
            serverLog(LL_WARNING,"Error reply to PING from master: '%s'",err);
            sdsfree(err);
            goto error;
        } else {
            serverLog(LL_NOTICE,
                "Master replied to PING, replication can continue...");
        }
        sdsfree(err);
        server.repl_state = REPL_STATE_SEND_AUTH;
    }

    /* AUTH with the master if required. */
    if (server.repl_state == REPL_STATE_SEND_AUTH) {
        if (server.masterauth) {
            err = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"AUTH",server.masterauth,NULL);
            if (err) goto write_error;
            server.repl_state = REPL_STATE_RECEIVE_AUTH;
            return;
        } else {
            server.repl_state = REPL_STATE_SEND_PORT;
        }
    }

    /* Receive AUTH reply. */
    if (server.repl_state == REPL_STATE_RECEIVE_AUTH) {
        err = sendSynchronousCommand(SYNC_CMD_READ,fd,NULL);
        if (err[0] == '-') {
            serverLog(LL_WARNING,"Unable to AUTH to MASTER: %s",err);
            sdsfree(err);
            goto error;
        }
        sdsfree(err);
        server.repl_state = REPL_STATE_SEND_PORT;
    }

    /* Set the slave port, so that Master's INFO command can list the
     * slave listening port correctly. */
    if (server.repl_state == REPL_STATE_SEND_PORT) {
        sds port = sdsfromlonglong(server.slave_announce_port ?
            server.slave_announce_port : server.port);
        err = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"REPLCONF",
                "listening-port",port, NULL);
        sdsfree(port);
        if (err) goto write_error;
        sdsfree(err);
        server.repl_state = REPL_STATE_RECEIVE_PORT;
        return;
    }

    /* Receive REPLCONF listening-port reply. */
    if (server.repl_state == REPL_STATE_RECEIVE_PORT) {
        err = sendSynchronousCommand(SYNC_CMD_READ,fd,NULL);
        /* Ignore the error if any, not all the Redis versions support
         * REPLCONF listening-port. */
        if (err[0] == '-') {
            serverLog(LL_NOTICE,"(Non critical) Master does not understand "
                                "REPLCONF listening-port: %s", err);
        }
        sdsfree(err);
        server.repl_state = REPL_STATE_SEND_IP;
    }

    /* Skip REPLCONF ip-address if there is no slave-announce-ip option set. */
    if (server.repl_state == REPL_STATE_SEND_IP &&
        server.slave_announce_ip == NULL)
    {
            server.repl_state = REPL_STATE_SEND_CAPA;
    }

    /* Set the slave ip, so that Master's INFO command can list the
     * slave IP address port correctly in case of port forwarding or NAT. */
    if (server.repl_state == REPL_STATE_SEND_IP) {
        err = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"REPLCONF",
                "ip-address",server.slave_announce_ip, NULL);
        if (err) goto write_error;
        sdsfree(err);
        server.repl_state = REPL_STATE_RECEIVE_IP;
        return;
    }

    /* Receive REPLCONF ip-address reply. */
    if (server.repl_state == REPL_STATE_RECEIVE_IP) {
        err = sendSynchronousCommand(SYNC_CMD_READ,fd,NULL);
        /* Ignore the error if any, not all the Redis versions support
         * REPLCONF listening-port. */
        if (err[0] == '-') {
            serverLog(LL_NOTICE,"(Non critical) Master does not understand "
                                "REPLCONF ip-address: %s", err);
        }
        sdsfree(err);
        server.repl_state = REPL_STATE_SEND_CAPA;
    }

    /* Inform the master of our (slave) capabilities.
     *
     * EOF: supports EOF-style RDB transfer for diskless replication.
     * PSYNC2: supports PSYNC v2, so understands +CONTINUE <new repl ID>.
     *
     * The master will ignore capabilities it does not understand. */
    if (server.repl_state == REPL_STATE_SEND_CAPA) {
        err = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"REPLCONF",
                "capa","eof","capa","psync2",NULL);
        if (err) goto write_error;
        sdsfree(err);
        server.repl_state = REPL_STATE_RECEIVE_CAPA;
        return;
    }

    /* Receive CAPA reply. */
    if (server.repl_state == REPL_STATE_RECEIVE_CAPA) {
        err = sendSynchronousCommand(SYNC_CMD_READ,fd,NULL);
        /* Ignore the error if any, not all the Redis versions support
         * REPLCONF capa. */
        if (err[0] == '-') {
            serverLog(LL_NOTICE,"(Non critical) Master does not understand "
                                  "REPLCONF capa: %s", err);
        }
        sdsfree(err);
        server.repl_state = REPL_STATE_SEND_PSYNC;
    }

    /* Try a partial resynchonization. If we don't have a cached master
     * slaveTryPartialResynchronization() will at least try to use PSYNC
     * to start a full resynchronization so that we get the master run id
     * and the global offset, to try a partial resync at the next
     * reconnection attempt. */
    if (server.repl_state == REPL_STATE_SEND_PSYNC) {
        if (slaveTryPartialResynchronization(fd,0) == PSYNC_WRITE_ERROR) {
            err = sdsnew("Write error sending the PSYNC command.");
            goto write_error;
        }
        server.repl_state = REPL_STATE_RECEIVE_PSYNC;
        return;
    }

    /* If reached this point, we should be in REPL_STATE_RECEIVE_PSYNC. */
    if (server.repl_state != REPL_STATE_RECEIVE_PSYNC) {
        serverLog(LL_WARNING,"syncWithMaster(): state machine error, "
                             "state should be RECEIVE_PSYNC but is %d",
                             server.repl_state);
        goto error;
    }
	// 解析是部分重同步 还是 完全重同步，若为部分重同步，实际上master就是当前从节点的一个客户端，什么都不用处理
    psync_result = slaveTryPartialResynchronization(fd,1);

	// 部分重同步的情况
    if (psync_result == PSYNC_CONTINUE) {
        serverLog(LL_NOTICE, "MASTER <-> REPLICA sync: Master accepted a Partial Resynchronization.");
        return;
    }

    /* PSYNC failed or is not supported: we want our slaves to resync with us
     * as well, if we have any sub-slaves. The master may transfer us an
     * entirely different data set and we have no way to incrementally feed
     * our slaves after that. */
    disconnectSlaves(); /* Force our slaves to resync with us as well. */
    freeReplicationBacklog(); /* Don't allow our chained slaves to PSYNC. */

    /* Fall back to SYNC if needed. Otherwise psync_result == PSYNC_FULLRESYNC
     * and the server.master_replid and master_initial_offset are
     * already populated. */
    if (psync_result == PSYNC_NOT_SUPPORTED) {
        serverLog(LL_NOTICE,"Retrying with SYNC...");
        if (syncWrite(fd,"SYNC\r\n",6,server.repl_syncio_timeout*1000) == -1) {
            serverLog(LL_WARNING,"I/O error writing to MASTER: %s",
                strerror(errno));
            goto error;
        }
    }

    /* Prepare a suitable temp file for bulk transfer */
    while(maxtries--) {
        snprintf(tmpfile,256,
            "temp-%d.%ld.rdb",(int)server.unixtime,(long int)getpid());
        dfd = open(tmpfile,O_CREAT|O_WRONLY|O_EXCL,0644);
        if (dfd != -1) break;
        sleep(1);
    }
    if (dfd == -1) {
        serverLog(LL_WARNING,"Opening the temp file needed for MASTER <-> REPLICA synchronization: %s",strerror(errno));
        goto error;
    }

    /* 完全重同步回调 */
    if (aeCreateFileEvent(server.el,fd, AE_READABLE,readSyncBulkPayload,NULL)
            == AE_ERR)
    {
        serverLog(LL_WARNING,
            "Can't create readable event for SYNC: %s (fd=%d)",
            strerror(errno),fd);
        goto error;
    }
	//最后，置复制状态为REDIS_REPL_TRANSFER，表示开始接收主节点的RDB数据。然后执行下列操作后返回
    server.repl_state = REPL_STATE_TRANSFER;
    server.repl_transfer_size = -1;
    server.repl_transfer_read = 0;
    server.repl_transfer_last_fsync_off = 0;
    server.repl_transfer_fd = dfd;
    server.repl_transfer_lastio = server.unixtime;
    server.repl_transfer_tmpfile = zstrdup(tmpfile);
    return;

error:
    aeDeleteFileEvent(server.el,fd,AE_READABLE|AE_WRITABLE);
    if (dfd != -1) close(dfd);
    close(fd);
    server.repl_transfer_s = -1;
    server.repl_state = REPL_STATE_CONNECT;
    return;

write_error: /* Handle sendSynchronousCommand(SYNC_CMD_WRITE) errors. */
    serverLog(LL_WARNING,"Sending command to master in replication handshake: %s", err);
    sdsfree(err);
    goto error;
}


```

```c
int slaveTryPartialResynchronization(int fd, int read_reply) {
    char *psync_replid;
    char psync_offset[32];
    sds reply;
	/* PSYNC 从发送给主的逻辑部分 */
    /* 检测自己是否有cached_master,有的话就尝试部分重同步 */
    if (!read_reply) {
        server.master_initial_offset = -1;

        if (server.cached_master) {
            psync_replid = server.cached_master->replid;
            snprintf(psync_offset,sizeof(psync_offset),"%lld", server.cached_master->reploff+1);
            serverLog(LL_NOTICE,"Trying a partial resynchronization (request %s:%s).", psync_replid, psync_offset);
        } else {
            serverLog(LL_NOTICE,"Partial resynchronization not possible (no cached master)");
            psync_replid = "?";
            memcpy(psync_offset,"-1",3);
        }

        /* 发送PSYNC命令 */
        reply = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"PSYNC",psync_replid,psync_offset,NULL);
        if (reply != NULL) {
            serverLog(LL_WARNING,"Unable to send PSYNC to master: %s",reply);
            sdsfree(reply);
            aeDeleteFileEvent(server.el,fd,AE_READABLE);
            return PSYNC_WRITE_ERROR;
        }
        return PSYNC_WAIT_REPLY;
    }

    /* PSYNC 从接收主的逻辑部分 */ */
    reply = sendSynchronousCommand(SYNC_CMD_READ,fd,NULL);
    if (sdslen(reply) == 0) {
        /* The master may send empty newlines after it receives PSYNC
         * and before to reply, just to keep the connection alive. */
        sdsfree(reply);
        return PSYNC_WAIT_REPLY;
    }
	
    aeDeleteFileEvent(server.el,fd,AE_READABLE);
	/* 检测是否为完全重同步 */
    if (!strncmp(reply,"+FULLRESYNC",11)) {
        char *replid = NULL, *offset = NULL;

        /* FULL RESYNC, parse the reply in order to extract the run id
         * and the replication offset. */
        replid = strchr(reply,' ');
        if (replid) {
            replid++;
            offset = strchr(replid,' ');
            if (offset) offset++;
        }
        if (!replid || !offset || (offset-replid-1) != CONFIG_RUN_ID_SIZE) {
            serverLog(LL_WARNING,
                "Master replied with wrong +FULLRESYNC syntax.");
            /* This is an unexpected condition, actually the +FULLRESYNC
             * reply means that the master supports PSYNC, but the reply
             * format seems wrong. To stay safe we blank the master
             * replid to make sure next PSYNCs will fail. */
            memset(server.master_replid,0,CONFIG_RUN_ID_SIZE+1);
        } else {
            memcpy(server.master_replid, replid, offset-replid-1);
            server.master_replid[CONFIG_RUN_ID_SIZE] = '\0';
            server.master_initial_offset = strtoll(offset,NULL,10);
            serverLog(LL_NOTICE,"Full resync from master: %s:%lld",
                server.master_replid,
                server.master_initial_offset);
        }
        /* 直接丢弃之前的cached_master部分，已经过时了 */
        replicationDiscardCachedMaster();
        sdsfree(reply);
        return PSYNC_FULLRESYNC;
    }
	// 
    if (!strncmp(reply,"+CONTINUE",9)) {
        /* Partial resync was accepted. */
        serverLog(LL_NOTICE,
            "Successful partial resynchronization with master.");

        /* Check the new replication ID advertised by the master. If it
         * changed, we need to set the new ID as primary ID, and set or
         * secondary ID as the old master ID up to the current offset, so
         * that our sub-slaves will be able to PSYNC with us after a
         * disconnection. */
        char *start = reply+10;
        char *end = reply+9;
        while(end[0] != '\r' && end[0] != '\n' && end[0] != '\0') end++;
        if (end-start == CONFIG_RUN_ID_SIZE) {
            char new[CONFIG_RUN_ID_SIZE+1];
            memcpy(new,start,CONFIG_RUN_ID_SIZE);
            new[CONFIG_RUN_ID_SIZE] = '\0';

            if (strcmp(new,server.cached_master->replid)) {
                /* Master ID changed. */
                serverLog(LL_WARNING,"Master replication ID changed to %s",new);

                /* Set the old ID as our ID2, up to the current offset+1. */
                memcpy(server.replid2,server.cached_master->replid,
                    sizeof(server.replid2));
                server.second_replid_offset = server.master_repl_offset+1;

                /* Update the cached master ID and our own primary ID to the
                 * new one. */
                memcpy(server.replid,new,sizeof(server.replid));
                memcpy(server.cached_master->replid,new,sizeof(server.replid));

                /* Disconnect all the sub-slaves: they need to be notified. */
                disconnectSlaves();
            }
        }

        /* Setup the replication to continue. */
        sdsfree(reply);
        /* 将cached_master 抬升为master，因为可以部分重同步了 */
        replicationResurrectCachedMaster(fd);

        /* If this instance was restarted and we read the metadata to
         * PSYNC from the persistence file, our replication backlog could
         * be still not initialized. Create it. */
        if (server.repl_backlog == NULL) createReplicationBacklog();
        return PSYNC_CONTINUE;
    }

    sdsfree(reply);
    replicationDiscardCachedMaster();
    return PSYNC_NOT_SUPPORTED;
}
```

​		

&emsp;&emsp;函数中如果发生了错误，则错误处理的方式是：删除socket描述符上注册的可读和可写事件，然后关闭描述符，置状态server.repl_state为REDIS_REPL_CONNECT，等待下次调用replicationCron时重连主节点；

&emsp;&emsp;首先检查当前主从复制状态server.repl_state是否为REDIS_REPL_NONE，若是，则说明握手过程期间，从节点收到了客户端执行的"slave  no  one"命令，因此直接关闭socket描述符，然后返回；

&emsp;&emsp;然后调用getsockopt，检查当前socket描述符的错误，若出错，则执行错误处理流程；


&emsp;&emsp;如果当前的复制状态为REDIS_REPL_CONNECTING，则说明是从节点connect主节点成功后，触发了描述符的可写事件，从而调用的该回调函数。这种情况下，先删除描述符上的可写事件，然后将状态设置为REDIS_REPL_RECEIVE_PONG，向主节点发送"PING"命令，然后返回；


&emsp;&emsp;如果当前的复制状态为REDIS_REPL_RECEIVE_PONG，则说明从节点收到了主节点对于"PING"命令的回复，触发了描述符的可读事件，从而调用的该回调函数。这种情况下，首先读取主节点的回复信息，正常情况下，主节点的回复只能有三种情况："+PONG"，"-NOAUTH"和"-ERR operation not permitted"（老版本的redis主节点），如果收到的回复不是以上的三种，则直接进入错误处理代码流程。否则，将复制状态置为REDIS_REPL_SEND_AUTH（不返回）；

&emsp;&emsp;当前的复制状态为REDIS_REPL_SEND_AUTH，如果配置了"masterauth"选项，则向主节点发送"AUTH"命令，后跟"masterauth"选项的值，然后将状态置为REDIS_REPL_RECEIVE_AUTH，然后返回；

&emsp;&emsp;如果从节点没有配置"masterauth"选项，则将复制状态置为REDIS_REPL_SEND_PORT（不返回）；


&emsp;&emsp;如果当前的复制状态为REDIS_REPL_RECEIVE_AUTH，说明从节点收到了主节点对于"AUTH"命令的回复，触发了描述符的可读事件，从而调用的该回调函数。这种情况下，首先读取主节点的回复，如果回复信息的首字节为"-"，说明认证失败，直接进入错误处理流程；否则，将状态置为REDIS_REPL_SEND_PORT（不返回）；

&emsp;&emsp;如果当前复制状态为REDIS_REPL_SEND_PORT，则向主节点发送"REPLCONF listening-port  <port>"命令，告知主节点本身的端口号，然后将复制状态置为REDIS_REPL_RECEIVE_PORT后返回；

&emsp;&emsp;如果当前的复制状态为REDIS_REPL_RECEIVE_PORT，说明从节点收到了主节点对于"REPLCONF listening-port"命令的回复，触发了描述符的可读事件，从而调用的该回调函数。这种情况下，首先读取主节点的回复，如果回复信息的首字节为"-"，说明主节点不认识该命令，这不是致命错误，只是记录日志而已；然后将复制状态设置为REDIS_REPL_SEND_CAPA（不返回）；

&emsp;&emsp;如果当前的复制状态为REDIS_REPL_SEND_CAPA，则向主节点发送"REPLCONF capa  eof"命令，告知主节点本身的"能力"，然后将复制状态置为REDIS_REPL_RECEIVE_CAPA后返回；


&emsp;&emsp;如果当前的复制状态为REDIS_REPL_RECEIVE_CAPA，说明从节点收到了主节点对于"REPLCONF capa eof"命令的回复，触发了描述符的可读事件，从而调用的该回调函数。这种情况下，首先读取主节点的回复，如果回复信息的首字节为"-"，说明主节点不认识该命令，这不是致命错误，只是记录日志，然后将复制状态设置为REDIS_REPL_SEND_PSYNC（不返回）；

&emsp;&emsp;如果复制状态为REDIS_REPL_SEND_PSYNC，则调用slaveTryPartialResynchronization函数，向主节点发送"PSYNC  <psync_runid>  <psync_offset>"命令。

&emsp;&emsp;在该函数中，如果从节点缓存了主节点，说明该从节点之前与主节点的连接断掉了，现在是重新连接，因此尝试进行部分重同步。置psync_runid为保存的主节点ID，置psync_offset为保存的主节点复制偏移加1；如果从节点没有缓存主节点，说明需要进行完全重同步，则置psync_runid为"?"，置psync_offset为"-1"；

&emsp;&emsp;发送命令成功后函数返回，将复制状态置为REDIS_REPL_RECEIVE_PSYNC后返回；


&emsp;&emsp;接下来的代码处理握手过程的最后一个状态REDIS_REPL_RECEIVE_PSYNC，走到这里，复制状态只能是REDIS_REPL_RECEIVE_PSYNC，如果不是则进入错误处理流程；

&emsp;&emsp;调用slaveTryPartialResynchronization读取主节点对于"PSYNC"命令的回复：

&emsp;&emsp;如果回复信息以"+CONTINUE"开头，说明主节点可以进行部分重同步，这种情况下，设置复制状态为REDIS_REPL_CONNECTED，后续将主节点当成一个客户端，接收该主节点客户端发来的命令请求，像处理普通客户端一样处理即可。因此函数slaveTryPartialResynchronization返回PSYNC_CONTINUE后，该函数直接返回即可；

&emsp;&emsp;如果回复信息以"+FULLRESYNC"开头，说明主节点虽然认识"PSYNC"命令，但是从节点发送的复制偏移psync_offset已经不在主节点的积压队列中了，因此需要进行完全重同步。解析出回复信息中的主节点ID，保存在server.repl_master_runid中；解析出主节点复制偏移初始值，保存在server.repl_master_initial_offset中；然后函数slaveTryPartialResynchronization返回PSYNC_FULLRESYNC；

&emsp;&emsp;如果回复信息不属于以上的情况，说明主节点不认识"PSYNC"命令，这种情况下，函数slaveTryPartialResynchronization返回PSYNC_NOT_SUPPORTED；


&emsp;&emsp;不管函数slaveTryPartialResynchronization返回PSYNC_FULLRESYNC，还是返回PSYNC_NOT_SUPPORTED，都表示接下来要进行完全重同步过程：

&emsp;&emsp;首先断开当前实例与所有从节点的连接，因为接下来要进行完全重同步，本实例会接收主节点发来的完全不同的数据，因此此举可以让该实例的从节点重新进行复制同步过程（从而也接收这些数据）；

&emsp;&emsp;然后调用freeReplicationBacklog，释放本实例的积压队列server.repl_backlog；

&emsp;&emsp;如果slaveTryPartialResynchronization函数返回的是PSYNC_NOT_SUPPORTED，说明这是老版本的主节点，不支持"PSYNC"命令，因此向主节点发送"SYNC"命令（主节点收到该命令后，直接发送RDB数据）；

&emsp;&emsp;接下来，就是为接收主节点发送来的RDB数据做准备：

&emsp;&emsp;首先创建保存RDB数据的临时文件"temp-<unixtime>.<pid>.rdb"，该文件的描述符记录到server.repl_transfer_fd中；

&emsp;&emsp;然后，注册socket描述符server.repl_transfer_s上的可读事件，事件回调函数为readSyncBulkPayload；

**从节点的复制状态转换**

![](image-20220401225925804.png)

&emsp;&emsp;在这些状态中，REDIS_REPL_CONNECT状态是从节点的初始状态，在状态转移过程中，出现了任何错误，都会关闭socket描述符，然后将状态置为REDIS_REPL_CONNECT，等待下次调用定时函数replicationCron时，重新连接主节点。

&emsp;&emsp;从REDIS_REPL_RECEIVE_PONG状态到REDIS_REPL_RECEIVE_PSYNC状态之间，是主从节点间的握手过程。

&emsp;&emsp;REDIS_REPL_RECEIVE_PSYNC状态之后，如果主节点支持部分重同步，则从节点进入状态REDIS_REPL_CONNECTED，后续从节点将主节点当成客户端server.master，从节点接收客户端server.master发来的命令，像处理普通客户端的命令请求一样进行处理，从而实现了从节点和主节点之间的同步；

&emsp;&emsp;如果主节点不支持部分重同步，则需要进行完全重同步，从节点进入REDIS_REPL_TRANSFER状态，开始接收主节点发来的RDB数据。一旦从节点接收到完整的RDB数据，则加载该RDB数据，加载完成之后，从节点进入REDIS_REPL_CONNECTED状态，将主节点当成客户端server.master，接收客户端server.master发来的命令，实现了从节点和主节点之间的同步；

### **三. 接收RDB数据**

&emsp;&emsp;正常情况下，完全重同步需要主节点将其中的数据转储到RDB文件中，然后将该文件发送给从节点。如果硬盘IO效率较差，则这种操作对于主节点的性能会造成会影响。

&emsp;&emsp;从2.8.18版本开始，Redis引入了“无硬盘复制”选项，开启该选项时，Redis在与从节点进行复制初始化时将不会将快照内容存储到硬盘上，而是直接通过网络发送给从节点，避免了硬盘的性能瓶颈。不过该功能还在试验阶段，可以在配置文件中使用"repl-diskless-sync"选项来配置开启该功能。


&emsp;&emsp;有硬盘复制的RDB数据和无硬盘复制的RDB数据，它们的格式是不一样的。有硬盘复制的RDB数据，主节点将数据保存到RDB文件后，将文件内容加上"$<len>/r/n"的头部后，发送给从节点。无硬盘复制的RDB数据，主节点直接将数据发送给从节点，而不再先保存到本地文件中，这种格式的RDB数据以"$EOF:<XXX>\r\n"开头，以"<XXX>"结尾。开头和结尾中的<XXX>内容相同，都是40字节长的，由"0123456789abcdef"中的字符组成的随机字符串。

&emsp;&emsp;在syncWithMaster函数中，握手过程结束后，需要进行完全重同步时，从节点注册了socket描述符server.repl_transfer_s上的可读事件，事件回调函数为readSyncBulkPayload。从节点调用该函数接收主节点发来的RDB数据。

```c
/* Asynchronously read the SYNC payload we receive from a master */
#define REPL_MAX_WRITTEN_BEFORE_FSYNC (1024*1024*8) /* 8 MB */
void readSyncBulkPayload(aeEventLoop *el, int fd, void *privdata, int mask) {
    char buf[4096];
    ssize_t nread, readlen, nwritten;
    off_t left;
    UNUSED(el);
    UNUSED(privdata);
    UNUSED(mask);

    /* Static vars used to hold the EOF mark, and the last bytes received
     * form the server: when they match, we reached the end of the transfer. */
    static char eofmark[CONFIG_RUN_ID_SIZE];
    static char lastbytes[CONFIG_RUN_ID_SIZE];
    static int usemark = 0;

    /* If repl_transfer_size == -1 we still have to read the bulk length
     * from the master reply. */
    if (server.repl_transfer_size == -1) {
        if (syncReadLine(fd,buf,1024,server.repl_syncio_timeout*1000) == -1) {
            serverLog(LL_WARNING,
                "I/O error reading bulk count from MASTER: %s",
                strerror(errno));
            goto error;
        }

        if (buf[0] == '-') {
            serverLog(LL_WARNING,
                "MASTER aborted replication with an error: %s",
                buf+1);
            goto error;
        } else if (buf[0] == '\0') {
            /* At this stage just a newline works as a PING in order to take
             * the connection live. So we refresh our last interaction
             * timestamp. */
            server.repl_transfer_lastio = server.unixtime;
            return;
        } else if (buf[0] != '$') {
            serverLog(LL_WARNING,"Bad protocol from MASTER, the first byte is not '$' (we received '%s'), are you sure the host and port are right?", buf);
            goto error;
        }

        /* There are two possible forms for the bulk payload. One is the
         * usual $<count> bulk format. The other is used for diskless transfers
         * when the master does not know beforehand the size of the file to
         * transfer. In the latter case, the following format is used:
         *
         * $EOF:<40 bytes delimiter>
         *
         * At the end of the file the announced delimiter is transmitted. The
         * delimiter is long and random enough that the probability of a
         * collision with the actual file content can be ignored. */
        if (strncmp(buf+1,"EOF:",4) == 0 && strlen(buf+5) >= CONFIG_RUN_ID_SIZE) {
            usemark = 1;
            memcpy(eofmark,buf+5,CONFIG_RUN_ID_SIZE);
            memset(lastbytes,0,CONFIG_RUN_ID_SIZE);
            /* Set any repl_transfer_size to avoid entering this code path
             * at the next call. */
            server.repl_transfer_size = 0;
            serverLog(LL_NOTICE,
                "MASTER <-> REPLICA sync: receiving streamed RDB from master");
        } else {
            usemark = 0;
            server.repl_transfer_size = strtol(buf+1,NULL,10);
            serverLog(LL_NOTICE,
                "MASTER <-> REPLICA sync: receiving %lld bytes from master",
                (long long) server.repl_transfer_size);
        }
        return;
    }

    /* Read bulk data */
    if (usemark) {
        readlen = sizeof(buf);
    } else {
        left = server.repl_transfer_size - server.repl_transfer_read;
        readlen = (left < (signed)sizeof(buf)) ? left : (signed)sizeof(buf);
    }

    nread = read(fd,buf,readlen);
    if (nread <= 0) {
        serverLog(LL_WARNING,"I/O error trying to sync with MASTER: %s",
            (nread == -1) ? strerror(errno) : "connection lost");
        cancelReplicationHandshake();
        return;
    }
    server.stat_net_input_bytes += nread;

    /* When a mark is used, we want to detect EOF asap in order to avoid
     * writing the EOF mark into the file... */
    int eof_reached = 0;

    if (usemark) {
        /* Update the last bytes array, and check if it matches our delimiter.*/
        if (nread >= CONFIG_RUN_ID_SIZE) {
            memcpy(lastbytes,buf+nread-CONFIG_RUN_ID_SIZE,CONFIG_RUN_ID_SIZE);
        } else {
            int rem = CONFIG_RUN_ID_SIZE-nread;
            memmove(lastbytes,lastbytes+nread,rem);
            memcpy(lastbytes+rem,buf,nread);
        }
        if (memcmp(lastbytes,eofmark,CONFIG_RUN_ID_SIZE) == 0) eof_reached = 1;
    }

    server.repl_transfer_lastio = server.unixtime;
    if ((nwritten = write(server.repl_transfer_fd,buf,nread)) != nread) {
        serverLog(LL_WARNING,"Write error or short write writing to the DB dump file needed for MASTER <-> REPLICA synchronization: %s", 
            (nwritten == -1) ? strerror(errno) : "short write");
        goto error;
    }
    server.repl_transfer_read += nread;

    /* Delete the last 40 bytes from the file if we reached EOF. */
    if (usemark && eof_reached) {
        if (ftruncate(server.repl_transfer_fd,
            server.repl_transfer_read - CONFIG_RUN_ID_SIZE) == -1)
        {
            serverLog(LL_WARNING,"Error truncating the RDB file received from the master for SYNC: %s", strerror(errno));
            goto error;
        }
    }

    /* Sync data on disk from time to time, otherwise at the end of the transfer
     * we may suffer a big delay as the memory buffers are copied into the
     * actual disk. */
    if (server.repl_transfer_read >=
        server.repl_transfer_last_fsync_off + REPL_MAX_WRITTEN_BEFORE_FSYNC)
    {
        off_t sync_size = server.repl_transfer_read -
                          server.repl_transfer_last_fsync_off;
        rdb_fsync_range(server.repl_transfer_fd,
            server.repl_transfer_last_fsync_off, sync_size);
        server.repl_transfer_last_fsync_off += sync_size;
    }

    /* Check if the transfer is now complete */
    if (!usemark) {
        if (server.repl_transfer_read == server.repl_transfer_size)
            eof_reached = 1;
    }
	// 读到了RDB结尾了，进行最后的加载工作
    if (eof_reached) {
        int aof_is_enabled = server.aof_state != AOF_OFF;

        /* Ensure background save doesn't overwrite synced data */
        if (server.rdb_child_pid != -1) {
            serverLog(LL_NOTICE,
                "Replica is about to load the RDB file received from the "
                "master, but there is a pending RDB child running. "
                "Killing process %ld and removing its temp file to avoid "
                "any race",
                    (long) server.rdb_child_pid);
            kill(server.rdb_child_pid,SIGUSR1);
            rdbRemoveTempFile(server.rdb_child_pid);
        }

        if (rename(server.repl_transfer_tmpfile,server.rdb_filename) == -1) {
            serverLog(LL_WARNING,"Failed trying to rename the temp DB into dump.rdb in MASTER <-> REPLICA synchronization: %s", strerror(errno));
            cancelReplicationHandshake();
            return;
        }
        serverLog(LL_NOTICE, "MASTER <-> REPLICA sync: Flushing old data");
        /* We need to stop any AOFRW fork before flusing and parsing
         * RDB, otherwise we'll create a copy-on-write disaster. */
        if(aof_is_enabled) stopAppendOnly();
        signalFlushedDb(-1);
        emptyDb(
            -1,
            server.repl_slave_lazy_flush ? EMPTYDB_ASYNC : EMPTYDB_NO_FLAGS,
            replicationEmptyDbCallback);
        /* Before loading the DB into memory we need to delete the readable
         * handler, otherwise it will get called recursively since
         * rdbLoad() will call the event loop to process events from time to
         * time for non blocking loading. */
        aeDeleteFileEvent(server.el,server.repl_transfer_s,AE_READABLE);
        serverLog(LL_NOTICE, "MASTER <-> REPLICA sync: Loading DB in memory");
        rdbSaveInfo rsi = RDB_SAVE_INFO_INIT;
        if (rdbLoad(server.rdb_filename,&rsi) != C_OK) {
            serverLog(LL_WARNING,"Failed trying to load the MASTER synchronization DB from disk");
            cancelReplicationHandshake();
            /* Re-enable the AOF if we disabled it earlier, in order to restore
             * the original configuration. */
            if (aof_is_enabled) restartAOFAfterSYNC();
            return;
        }
        /* Final setup of the connected slave <- master link */
        zfree(server.repl_transfer_tmpfile);
        close(server.repl_transfer_fd);
        replicationCreateMasterClient(server.repl_transfer_s,rsi.repl_stream_db);
        server.repl_state = REPL_STATE_CONNECTED;
        server.repl_down_since = 0;
        /* After a full resynchroniziation we use the replication ID and
         * offset of the master. The secondary ID / offset are cleared since
         * we are starting a new history. */
        memcpy(server.replid,server.master->replid,sizeof(server.replid));
        server.master_repl_offset = server.master->reploff;
        clearReplicationId2();
        /* Let's create the replication backlog if needed. Slaves need to
         * accumulate the backlog regardless of the fact they have sub-slaves
         * or not, in order to behave correctly if they are promoted to
         * masters after a failover. */
        if (server.repl_backlog == NULL) createReplicationBacklog();

        serverLog(LL_NOTICE, "MASTER <-> REPLICA sync: Finished with success");
        /* Restart the AOF subsystem now that we finished the sync. This
         * will trigger an AOF rewrite, and when done will start appending
         * to the new file. */
        if (aof_is_enabled) restartAOFAfterSYNC();
    }
    return;

error:
    cancelReplicationHandshake();
    return;
}
```



&emsp;&emsp;server.repl_transfer_size的值表示要读取的RDB数据的总长度（仅对有硬盘复制的RDB数据而言）。如果当前其值为-1，说明本次是第一次接收RDB数据。因此，首先调用syncReadLine，读取主节点发来的第一行数据("\r\n"之前的内容)到buf中，读取的超时时间为5s，如果在5s之内还读不到"\n"，则syncReadLine返回-1，因此调用函数replicationAbortSyncTransfer，终止本次复制过程，然后返回；

&emsp;&emsp;然后解析读取到的内容，如果符合无硬盘复制的RDB数据格式，则将40字节的随机串记录到静态变量eofmark中，并且置usemark为1，置server.repl_transfer_size为0，然后返回；

&emsp;&emsp;如果不符合无硬盘复制的RDB数据格式，则认为是有硬盘复制的RDB数据，从buf中解析得到RDB数据的长度，记录到server.repl_transfer_size中，并且置usemark为0后返回；


&emsp;&emsp;后续可读事件触发，再次调用该函数时，server.repl_transfer_size已不再是-1，开始接收真正的RDB数据了。usemark为0，表示是有硬盘复制的RDB数据，为1，表示是无硬盘复制的的RDB数据；


&emsp;&emsp;接下来调用read，读取RDB数据内容到buf中。read返回值为nread，如果nread小于等于0，要么说明发生了错误，要么说明主节点终止了链接，无论哪种情况，都是调用函数replicationAbortSyncTransfer，终止本次复制过程，然后返回； 

&emsp;&emsp;如果nread大于0，则先将其增加到server.stat_net_input_bytes中；

&emsp;&emsp;如果是无硬盘复制的RDB数据，则每次read之后，都判断是否接收到了末尾40字节的随机串：如果nread大于等于40，则将buf中后40个字节复制到lastbytes中；否则，将buf复制到lastbytes中的尾部。然后比对lastbytes和eofmark，如果相同，说明已经接收到了末尾，置eof_reached为1；


&emsp;&emsp;然后，将buf写入到描述符server.repl_transfer_fd中，也就是从节点保存RDB数据的临时文件中；

&emsp;&emsp;然后将nread增加到server.repl_transfer_read中，该属性记录了当前已读到的RDB数据的长度；

&emsp;&emsp;如果是无硬盘复制的RDB数据，并且已经读到了末尾，则将临时文件中末尾的40字节的随机串删除；

&emsp;&emsp;每当读取了8M的数据后，都执行一次sync操作，保证临时文件内容确实写到了硬盘; 如果是有硬盘复制的RDB数据，且server.repl_transfer_read等于server.repl_transfer_size，则说明已经接收到所有数据，置eof_reached为1；


&emsp;&emsp;如果所有的RDB数据已经接收完了，则首先将保存RDB数据的临时文件改名为配置的RDB文件名server.rdb_filename；然后调用signalFlushedDb，使得本实例的所有客户端感知到接下来要清空数据库了。然后就是调用emptyDb，清空所有数据，回调函数是replicationEmptyDbCallback，每当处理了字典哈希表中65535个bucket之后，就调用一次该函数，向主节点发送一个"\n"，以向主节点证明本实例还活着；

&emsp;&emsp;然后删除server.repl_transfer_s上的可读事件，这是因为在调用rdbLoad加载RDB数据时，每次调用rioRead都会调用processEventsWhileBlocked处理当前已触发的事件，如果不删除该可读事件的话，就会递归进入的本函数中（因此，从节点在加载RDB数据时，是不能处理主节点发来的其他数据的）；         

接下来就是调用rdbLoad加载RDB数据；


​        

&emsp;&emsp;加载完RDB数据之后，就已经完成了完全重同步过程。接下来，从节点会将主节点当成客户端，像处理普通客户端那样，接收主节点发来的命令，执行命令以保证主从一致性。

&emsp;&emsp;因此，首先关闭RDB临时文件描述符server.repl_transfer_fd，然后就使用socket描述符server.repl_transfer_s创建redisClient结构server.master，因此后续还是使用该描述符接收主节点客户端发来的命令；

&emsp;&emsp;将标记REDIS_MASTER记录到客户端标志中，以表明该客户端是主节点；

&emsp;&emsp;将复制状态置为REDIS_REPL_CONNECTED，表示主从节点已完成握手和接收RDB数据的过程；

&emsp;&emsp;主节点之前的发送"PSYNC"命令回复为"+FULLRESYNC"时，附带的初始复制偏移记录到了server.repl_master_initial_offset中，将其保存到server.master->reploff；附带的主节点ID记录到了server.repl_master_runid中，将其保存到server.master->replrunid中；如果server.repl_master_initial_offset为-1，说明主节点不认识"PSYNC"命令，因此将REDIS_PRE_PSYNC记录到客户端标志位中；

&emsp;&emsp;完成以上的操作之后，如果本实例开启了AOF功能，则首先调用stopAppendOnly，然后循环10次，调用startAppendOnly开始进行AOF转储，直到startAppendOnly返回REDIS_OK。如果startAppendOnly失败次数超过10次，则直接exit退出！！！

### 四. 命令传播

&emsp;&emsp;当复制状态变为REDIS_REPL_CONNECTED后，表示进入了命令传播阶段。后续从节点将主节点当成一个客户端，接收该主节点客户端发来的命令请求，像处理普通客户端一样处理即可。

&emsp;&emsp;在读取客户端命令的函数readQueryFromClient中，一旦从节点读到了追节点发来的同步命令，会将命令长度增加到从节点的复制偏移量server.master. reploff中：

```c
    if (nread) {
        sdsIncrLen(c->querybuf,nread);
        c->lastinteraction = server.unixtime;
        if (c->flags & REDIS_MASTER) c->reploff += nread;
        server.stat_net_input_bytes += nread;
    } 
```

&emsp;&emsp;这样，从节点的复制偏移量server.master. reploff就能与主节点保持一致了。

&emsp;&emsp;与普通客户端不同的是，主节点客户端发来的命令请求无需回复，因此，在函数prepareClientToWrite中，有下面的语句：

```c
int prepareClientToWrite(redisClient *c) {
    ...
    /* Masters don't receive replies, unless REDIS_MASTER_FORCE_REPLY flag
     * is set. */
    if ((c->flags & REDIS_MASTER) &&
        !(c->flags & REDIS_MASTER_FORCE_REPLY)) return REDIS_ERR;
    ...
}   
```

&emsp;&emsp;每次向客户端输出缓存追加新数据之前，都要调用函数prepareClientToWrite函数。如果该函数返回REDIS_ERR，表示无需向输出缓存追加新数据。

&emsp;&emsp;客户端标志中如果设置了REDIS_MASTER标记，就表示该客户端是主节点客户端server.master，并且在没有设置REDIS_MASTER_FORCE_REPLY标记的情况下，该函数返回REDIS_ERR，表示无需向输出缓存追加新数据。

## 3.3 主节点部分

### 一. 完全重同步

#### 1. 从节点建链和握手

&emsp;&emsp;从节点在向主节点发起TCP建链，以及复制握手过程中，主节点一直把从节点当成一个普通的客户端处理。也就是说，不为从节点保存状态，只是收到从节点发来的命令进而处理并回复罢了。

&emsp;&emsp;从节点在握手过程中第一个发来的命令是”PING”，主节点调用redis.c中的pingCommand函数处理，只是回复字符串”+PONG”即可。

&emsp;&emsp;接下来从节点向主节点发送”AUTHxxx”命令进行认证，主节点调用redis.c中的authCommand函数进行处理，该函数的代码如下：

```c
void authCommand(redisClient *c) {
    if (!server.requirepass) {
        addReplyError(c,"Client sent AUTH, but no password is set");
    } else if (!time_independent_strcmp(c->argv[1]->ptr, server.requirepass)) {
      c->authenticated = 1;
      addReply(c,shared.ok);
    } else {
      c->authenticated = 0;
      addReplyError(c,"invalid password");
    }
}
```

&emsp;&emsp;server.requirepass根据配置文件中"requirepass"的选项进行设置，保存了Redis实例的密码。如果该值为NULL，说明本Redis实例不需要密码。这种情况下，如果从节点发来”AUTH xxx”命令，则回复给从节点错误信息："Client sent AUTH, but no password is set"。

&emsp;&emsp;接下来，对从节点发来的密码和server.requirepass进行比对，如果匹配成功，则回复给客户端”+OK”，否则，回复给客户端错误信息："invalid password"。

&emsp;&emsp;从节点接下来发送"REPLCONF listening-port  <port>"和"REPLCONF capa  eof"命令，告知主节点自己的监听端口和“能力”。主节点通过replication.c中的replconfCommand函数处理这些命令，代码如下：

```c
void replconfCommand(redisClient *c) {
    int j;
 
    if ((c->argc % 2) == 0) {
        /* Number of arguments must be odd to make sure that every
         * option has a corresponding value. */
        addReply(c,shared.syntaxerr);
        return;
    }
 
    /* Process every option-value pair. */
    for (j = 1; j < c->argc; j+=2) {
        if (!strcasecmp(c->argv[j]->ptr,"listening-port")) {
            long port;
 
            if ((getLongFromObjectOrReply(c,c->argv[j+1],
                    &port,NULL) != REDIS_OK))
                return;
            c->slave_listening_port = port;
        } else if (!strcasecmp(c->argv[j]->ptr,"capa")) {
            /* Ignore capabilities not understood by this master. */
            if (!strcasecmp(c->argv[j+1]->ptr,"eof"))
                c->slave_capa |= SLAVE_CAPA_EOF;
        } 
	   ...
    }
    addReply(c,shared.ok);
}
```

&emsp;&emsp;“REPLCONF”命令的格式为"REPLCONF  <option>  <value> <option>  <value>  ..."。因此，如果命令参数是偶数，说明命令格式错误，回复给从节点客户端错误信息："-ERR syntax error"；

&emsp;&emsp;如果从节点发来的是"REPLCONF listening-port  <port>"命令，则从中取出<port>信息，保存在客户端的slave_listening_port属性中，记录从节点客户端的监听端口，主节点使用从节点的IP地址和监听端口，作为从节点的身份标识；

&emsp;&emsp;如果从节点发来的是"REPLCONF capa  eof"命令，则将从节点客户端的能力属性slave_capa增加SLAVE_CAPA_EOF标记，表示该从节点支持无硬盘复制。目前为止，仅有这一种能力标记。

#### 2. 完全重同步时，从节点状态转换

&emsp;&emsp;接下来，从节点会向主节点发送”SYNC”或”PSYNC”命令，请求进行完全重同步或者部分重同步。

&emsp;&emsp;主节点收到这些命令之后，如果是需要进行完全重同步，则开始在后台进行RDB数据转储（将数据保存在本地文件或者直接发给从节点）。同时，在前台接着接收客户端发来的命令请求。为了使从节点能与主节点的状态保持一致，主节点需要将这些命令请求缓存起来，以便在从节点收到主节点RDB数据并加载完成之后，将这些累积的命令流发送给从节点。

&emsp;&emsp;从收到从节点的”SYNC”或”PSYNC”命令开始，主节点开始为该从节点保存状态。从此时起，站在主节点的角度，从节点的状态会发生转换。

&emsp;&emsp;主节点为从节点保存的状态记录在客户端结构redisClient中的replstate属性中。从主节点的角度看，从节点需要经历的状态分别是：**REDIS_REPL_WAIT_BGSAVE_START**、**REDIS_REPL_WAIT_BGSAVE_END**、**REDIS_REPL_SEND_BULK**和**REDIS_REPL_ONLINE**。



&emsp;&emsp;当主节点收到从节点发来的”SYNC”或”PSYNC”命令，并且需要完全重同步时，将从节点的状态置为**REDIS_REPL_WAIT_BGSAVE_START**，表示该从节点等待主节点后台RDB数据转储的开始；



&emsp;&emsp;接下来，当主节点**开始在后台进行RDB数据转储时**，将从节点的状态置为REDIS_REPL_WAIT_BGSAVE_END，表示该从节点等待主节点后台RDB数据转储的完成；

&emsp;&emsp;主节点在后台进行RDB数据的转储的时候，依然可以接收客户端发来的命令请求，为了能使从节点与主节点保持一致，主节点需要将客户端发来的命令请求，保存到从节点客户端的输出缓存中，这就是所谓的为从节点累积命令流。当从节点的复制状态变为REDIS_REPL_ONLINE时，就**可以将这些累积的命令流发送**给从节点了。

&emsp;&emsp;如果主节点在进行后台RDB数据转储时，使用的是有硬盘复制的方式（将RDB数据保存在本地文件），则RDB数据转储完成时，将从节点的状态置为REDIS_REPL_SEND_BULK，表示**接下来要将本地的RDB文件发送给客户端**了；当所有的RDB数据发送完成后，将从节点的状态置为REDIS_REPL_ONLINE，表示可以向从节点发送累积的命令流了。

&emsp;&emsp;如果主节点在进行后台RDB数据转储时，使用的是无硬盘复制的方式（将RDB数据直接通过网络发送给从节点），则RDB数据发送完成之后，收到从节点发来的第一个"REPLCONF  ACK  <offset>"后，就将从节点的状态置为REDIS_REPL_ONLINE，表示可以向从节点发送累积的命令流了。

&emsp;&emsp;无硬盘复制的RDB数据转储，之所以要等到收到第一个"REPLCONF  ACK  <offset>"后，才能将从节点的状态置为REDIS_REPL_ONLINE。个人理解是因为无硬盘复制的RDB数据，不同于有硬盘复制的RDB数据，它没有长度标记，从节点每次从socket读取的数据量都是固定的(4k)。下面是从节点读取RDB数据时调用的readSyncBulkPayload函数中，每次read之前，计算要读取多少字节的代码，usemark为1表示无硬盘复制：

```c
    /* Read bulk data */
    if (usemark) {
        readlen = sizeof(buf);
    } else {
        left = server.repl_transfer_size - server.repl_transfer_read;
        readlen = (left < (signed)sizeof(buf)) ? left : (signed)sizeof(buf);
    }
```

&emsp;&emsp;因此，主节点在通过socket发送完RDB数据之后，如果接着就使用该socket发送累积的命令流，则从节点读取数据时，最后读到的数据中，有可能一部分是RDB数据，剩下的部分是累积的命令流。而此时从节点接下来就要加载RDB数据，无法处理这部分累积的命令流，只能丢掉，这就造成了主从数据库状态不一致了。

&emsp;&emsp;因此，等到从节点发来第一个"REPLCONF  ACK <offset>"消息之后，此时能保证从节点已经加载完RDB数据，可以接收累积的命令流了。因此，这时才可以将从节点的复制状态置为REDIS_REPL_ONLINE。

&emsp;&emsp;有硬盘复制的RDB数据，因为数据头中包含了数据长度，因此从节点知道总共需要读取多少RDB数据。因此，有硬盘复制的RDB数据转储，在发送完RDB数据之后，就可以立即将从节点复制状态置为REDIS_REPL_ONLINE。

&emsp;&emsp;根据以上的描述，总结从节点的状态转换图如下：

![](image-20220402120557723.png)

#### 3. SYNC或PSYNC命令的处理

&emsp;&emsp;主节点收到从节点发来的”SYNC”或”PSYNC”命令后，如果需要为该从节点进行完全重同步，将从节点的复制状态置为REDIS_REPL_WAIT_BGSAVE_START。开始在后台进行RDB数据转储时，则将复制状态置为REDIS_REPL_WAIT_BGSAVE_END。

&emsp;&emsp;这里有一个问题，考虑这样一种情形：当主节点收到从节点A的”SYNC”或”PSYNC”命令后，要为该从节点进行完全重同步时，在将A的复制状态变为REDIS_REPL_WAIT_BGSAVE_END时刻起，主节点在前台接收客户端的命令请求，将该命令情求保存到A的输出缓存中，并在后台进行有硬盘复制的RDB数据转储。

&emsp;&emsp;在后台进行有硬盘复制的RDB数据转储尚未完成时，如果又有新的从节点B发来了”SYNC”或”PSYNC”命令，同样需要完全重同步。此时主节点后台正在进行RDB数据转储，而且已经为A缓存了命令流。那么从节点B完全可以**重用这份RDB数据**，而无需再执行一次RDB转储了。而且将A中的**输出缓存复制**到B的输出缓存中，就能保证B的数据库状态也能与主节点一致了。因此，直接将B的复制状态直接置为REDIS_REPL_WAIT_BGSAVE_END，等到后台RDB数据转储完成时，直接将该转储文件同时发送给从节点A和B即可。

> 输出缓存复制就已经同步了，之后每次命令执行都会追加到每个slave的reply链表中的

&emsp;&emsp;但是如果此刻主节点进行的是无硬盘复制的RDB数据转储，这意味着主节点是直接将RDB数据通过socket发送给从节点A的，从节点B也就无法重用RDB数据了，因此需要再次执行一次BGSAVE操作。

> 这里实际上什么都不用做，它处于 c->replstate = REDIS_REPL_WAIT_BGSAVE_START状态，最后RDB生成时会检察所有处于这个状态的slave，为他们共同生成一份bgsave

下面就是主节点收到”SYNC”或”PSYNC”命令的处理函数syncCommand的代码：

```c
void syncCommand(client *c) {
    /* ignore SYNC if already slave or in monitor mode */
    if (c->flags & CLIENT_SLAVE) return;

    /* Refuse SYNC requests if we are a slave but the link with our master
     * is not ok... */
    if (server.masterhost && server.repl_state != REPL_STATE_CONNECTED) {
        addReplySds(c,sdsnew("-NOMASTERLINK Can't SYNC while not connected with my master\r\n"));
        return;
    }

    /* SYNC can't be issued when the server has pending data to send to
     * the client about already issued commands. We need a fresh reply
     * buffer registering the differences between the BGSAVE and the current
     * dataset, so that we can copy to other slaves if needed. */
    if (clientHasPendingReplies(c)) {
        addReplyError(c,"SYNC and PSYNC are invalid with pending output");
        return;
    }

    serverLog(LL_NOTICE,"Replica %s asks for synchronization",
        replicationGetSlaveName(c));

    /* Try a partial resynchronization if this is a PSYNC command.
     * If it fails, we continue with usual full resynchronization, however
     * when this happens masterTryPartialResynchronization() already
     * replied with:
     *
     * +FULLRESYNC <replid> <offset>
     *
     * So the slave knows the new replid and offset to try a PSYNC later
     * if the connection with the master is lost. */
    if (!strcasecmp(c->argv[0]->ptr,"psync")) {
        if (masterTryPartialResynchronization(c) == C_OK) {
            server.stat_sync_partial_ok++;
            return; /* No full resync needed, return. */
        } else {
            char *master_replid = c->argv[1]->ptr;

            /* Increment stats for failed PSYNCs, but only if the
             * replid is not "?", as this is used by slaves to force a full
             * resync on purpose when they are not albe to partially
             * resync. */
            if (master_replid[0] != '?') server.stat_sync_partial_err++;
        }
    } else {
        /* If a slave uses SYNC, we are dealing with an old implementation
         * of the replication protocol (like redis-cli --slave). Flag the client
         * so that we don't expect to receive REPLCONF ACK feedbacks. */
        c->flags |= CLIENT_PRE_PSYNC;
    }

    /* Full resynchronization. */
    server.stat_sync_full++;

    /* Setup the slave as one waiting for BGSAVE to start. The following code
     * paths will change the state if we handle the slave differently. */
    c->replstate = SLAVE_STATE_WAIT_BGSAVE_START; //这里注意，改变了状态为start
    if (server.repl_disable_tcp_nodelay)
        anetDisableTcpNoDelay(NULL, c->fd); /* 大块发数据需要打开negal算法 */
    c->repldbfd = -1;
    c->flags |= CLIENT_SLAVE;
    listAddNodeTail(server.slaves,c);

    /* Create the replication backlog if needed. */
    if (listLength(server.slaves) == 1 && server.repl_backlog == NULL) {
        /* When we create the backlog from scratch, we always use a new
         * replication ID and clear the ID2, since there is no valid
         * past history. */
        changeReplicationId();
        clearReplicationId2();
        createReplicationBacklog();
    }

    /* CASE 1: BGSAVE is in progress, with disk target. */
    if (server.rdb_child_pid != -1 &&
        server.rdb_child_type == RDB_CHILD_TYPE_DISK)
    {
        /* Ok a background save is in progress. Let's check if it is a good
         * one for replication, i.e. if there is another slave that is
         * registering differences since the server forked to save. */
        client *slave;
        listNode *ln;
        listIter li;

        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            slave = ln->value;
            if (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_END) break;
        }
        /* To attach this slave, we check that it has at least all the
         * capabilities of the slave that triggered the current BGSAVE. */
        if (ln && ((c->slave_capa & slave->slave_capa) == slave->slave_capa)) {
            /* Perfect, the server is already registering differences for
             * another slave. Set the right state, and copy the buffer. */
            copyClientOutputBuffer(c,slave);
            //这个偏移量psync_initial_offset实际上就是第一个执行bgsave的复制偏移量，为了保持一致，直接设置为相同
            replicationSetupSlaveForFullResync(c,slave->psync_initial_offset);
            serverLog(LL_NOTICE,"Waiting for end of BGSAVE for SYNC");
        } else {
            /* No way, we need to wait for the next BGSAVE in order to
             * register differences. */
            serverLog(LL_NOTICE,"Can't attach the replica to the current BGSAVE. Waiting for next BGSAVE for SYNC");
        }

    /* CASE 2: BGSAVE is in progress, with socket target. */
    } else if (server.rdb_child_pid != -1 &&
               server.rdb_child_type == RDB_CHILD_TYPE_SOCKET)
    {
        /* There is an RDB child process but it is writing directly to
         * children sockets. We need to wait for the next BGSAVE
         * in order to synchronize. */
        serverLog(LL_NOTICE,"Current BGSAVE has socket target. Waiting for next BGSAVE for SYNC");

    /* CASE 3: There is no BGSAVE is progress. */
    } else {
        if (server.repl_diskless_sync && (c->slave_capa & SLAVE_CAPA_EOF)) {
            /* Diskless replication RDB child is created inside
             * replicationCron() since we want to delay its start a
             * few seconds to wait for more slaves to arrive. */
            if (server.repl_diskless_sync_delay)
                serverLog(LL_NOTICE,"Delay next BGSAVE for diskless SYNC");
        } else {
            /* Target is disk (or the slave is not capable of supporting
             * diskless replication) and we don't have a BGSAVE in progress,
             * let's start one. */
            if (server.aof_child_pid == -1) {
                startBgsaveForReplication(c->slave_capa);
            } else {
                serverLog(LL_NOTICE,
                    "No BGSAVE in progress, but an AOF rewrite is active. "
                    "BGSAVE for replication delayed");
            }
        }
    }
    return;
}
```

&emsp;&emsp;在函数中，如果当前的客户端标志位中已经有REDIS_SLAVE标记了，则直接返回；

&emsp;&emsp;如果当前Redis实例是其他主节点的从节点，并且该从节点的复制状态不是REDIS_REPL_CONNECTED，说明当前的从节点实例，还没有到接收并加载完其主节点发来的RDB数据的步骤，这种情况下，该从节点实例是不能为其下游从节点进行同步的，因此向其客户端回复错误信息，然后返回；




&emsp;&emsp;如果当前的客户端输出缓存中已经有数据了，说明在SYNC(PSYNC)命令之前的命令交互中，该Redis实例尚有回复信息还没有完全发送给该从节点客户端，这种情况下，向该从节点客户端回复错误信息，然后返回；

&emsp;&emsp;这是因为主节点接下来需要为该从节点进行后台RDB数据转储了，同时需要将前台接收到的其他客户端命令请求缓存到该从节点客户端的输出缓存中，这就需要一个完全清空的输出缓存，才能为该从节点保存从执行BGSAVE开始的命令流。因此，如果从节点客户端的输出缓存中尚有数据，直接回复错误信息。

&emsp;&emsp;在主节点收到从节点发来的SYNC(PSYNC)命令之前，主从节点之间的交互信息都是比较短的，因此，在网络正常的情况下，从节点客户端中的输出缓存应该是很容易就发送给该从节点，并清空的。




&emsp;&emsp;接下来开始处理PSYNC或者SYNC命令：

&emsp;&emsp;如果用户发来的是"PSYNC"命令，则首先调用`masterTryPartialResynchronization`尝试进行部分重同步，如果成功，则直接返回即可。

&emsp;&emsp;如果不能为该从节点执行部分重同步，则接下来需要进行完全重同步了。首先如果用户发来的"SYNC"命令，则将REDIS_PRE_PSYNC标记增加到客户端标记中，表示该从节点客户端是老版本的Redis实例；接下来就准备进行完全重同步了，先增加server.stat_sync_full的值；


​        

&emsp;&emsp;首先将该从节点客户端的复制状态置为REDIS_REPL_WAIT_BGSAVE_START，表示该从节点需要主节点进行BGSAVE；

&emsp;&emsp;如果server.repl_disable_tcp_nodelay选项为真，则取消与从节点通信的socket描述符的TCP_NODELAY选项；  

&emsp;&emsp;将REDIS_SLAVE标记记录到从节点客户端的标志位中，以标识该客户端为从节点客户端；

&emsp;&emsp;将该从节点客户端添加到列表server.slaves中；


​        

**接下来开始分情况处理：**

* 情况1：如果当前已有子进程正在后台将RDB转储到本地文件，则轮训列表server.slaves，找到一个复制状态为REDIS_REPL_WAIT_BGSAVE_END的从节点客户端。

&emsp;&emsp;如果找到了一个这样的从节点客户端A，并且A的能力是大于当前从节点的。那么主节点为从节点A，在后台开始进行RDB数据转储时，同时会将前台收到的命令流缓存到从节点A的输出缓存中。因此当前发来SYNC(PSYNC)命令的从节点完全可以重用这份RDB数据，以及从节点A中缓存的命令流，而无需再执行一次RDB转储。等到本次BGSAVE完成之后，只需要将RDB文件发送给A以及当前从节点即可。

&emsp;&emsp;因此，找到这样的从节点A后，只要复制A的输出缓存中的内容到当前从节点的输出缓存中，然后调用replicationSetupSlaveForFullResync，将该从节点客户端的复制状态置为REDIS_REPL_WAIT_BGSAVE_END，然后向其发送"+FULLRESYNC"回复即可；

&emsp;&emsp;如果找不到这样的从节点客户端，则主节点需要在当前的BGSAVE操作完成之后，重新执行一次BGSAVE操作；


​        

* 情况2：如果当前有子进程在后台进行RDB转储，但是是直接将RDB数据通过socket直接发送给了从节点。这种情况下，当前的从节点无法重用RDB数据，必须在当前的BGSAVE操作完成之后，重新执行一次BGSAVE操作；



* 情况3：如果当前没有子进程在进行RDB转储，并且当前的从节点客户端可以接受无硬盘复制的RDB数据。这种情况下，先暂时不进行BGSAVE，而是在定时函数replicationCron中在执行，这样可以等到更多的从节点，以减少执行BGSAVE的次数；



* 情况4：如果当前没有子进程在进行RDB转储，并且当前的从节点客户端只能接受有硬盘复制的RDB数据，则调用startBgsaveForReplication开始进行BGSAVE操作；



&emsp;&emsp;最后，如果当前的列表server.slaves长度为1，并且server.repl_backlog为NULL，说明当前从节点客户端是该主节点实例的第一个从节点，因此调用createReplicationBacklog创建积压队列；

#### 4.开始BGSAVE操作

&emsp;&emsp;由函数startBgsaveForReplication执行BGSAVE操作。在开始执行BGSAVE操作时，需要向从节点发送"+FULLRESYNC <runid>  <offset>"信息，从节点收到该信息后，会保存主节点的运行ID，以及复制偏移的初始值，以便后续断链时可以进行部分重同步。

&emsp;&emsp;startBgsaveForReplication的代码如下：

```c
int startBgsaveForReplication(int mincapa) {
    int retval;
    int socket_target = server.repl_diskless_sync && (mincapa & SLAVE_CAPA_EOF);
    listIter li;
    listNode *ln;

    serverLog(LL_NOTICE,"Starting BGSAVE for SYNC with target: %s",
        socket_target ? "replicas sockets" : "disk");

    rdbSaveInfo rsi, *rsiptr;
    rsiptr = rdbPopulateSaveInfo(&rsi);
    /* Only do rdbSave* when rsiptr is not NULL,
     * otherwise slave will miss repl-stream-db. */
    if (rsiptr) {
        if (socket_target)
            retval = rdbSaveToSlavesSockets(rsiptr);
        else
            retval = rdbSaveBackground(server.rdb_filename,rsiptr);
    } else {
        serverLog(LL_WARNING,"BGSAVE for replication: replication information not available, can't generate the RDB file right now. Try later.");
        retval = C_ERR;
    }

    /* If we failed to BGSAVE, remove the slaves waiting for a full
     * resynchorinization from the list of salves, inform them with
     * an error about what happened, close the connection ASAP. */
    if (retval == C_ERR) {
        serverLog(LL_WARNING,"BGSAVE for replication failed");
        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            client *slave = ln->value;

            if (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_START) {
                slave->replstate = REPL_STATE_NONE;
                slave->flags &= ~CLIENT_SLAVE;
                listDelNode(server.slaves,ln);
                addReplyError(slave,
                    "BGSAVE failed, replication can't continue");
                slave->flags |= CLIENT_CLOSE_AFTER_REPLY;
            }
        }
        return retval;
    }

    /* If the target is socket, rdbSaveToSlavesSockets() already setup
     * the salves for a full resync. Otherwise for disk target do it now.*/
    if (!socket_target) {
        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            client *slave = ln->value;

            if (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_START) {
                    replicationSetupSlaveForFullResync(slave,
                            getPsyncInitialOffset()); //注意这里设置了slave复制偏移量为当前主节点的复制偏移量
            }
        }
    }

    /* Flush the script cache, since we need that slave differences are
     * accumulated without requiring slaves to match our cached scripts. */
    if (retval == C_OK) replicationScriptCacheFlush();
    return retval;
}
```

&emsp;&emsp;参数mincapa，表示从节点的"能力"，也就是是否能接受无硬盘复制的RDB数据。如果选项server.repl_diskless_sync为真，并且参数mincapa中包含SLAVE_CAPA_EOF标记，说明可以为该从节点直接发送无硬盘复制的RDB数据，因此调用rdbSaveToSlavesSockets，直接在后台将RDB数据通过socket发送给所有状态为REDIS_REPL_WAIT_BGSAVE_START的从节点；

&emsp;&emsp;否则，调用rdbSaveBackground，在后台将RDB数据转储到本地文件；


​        

&emsp;&emsp;如果rdbSaveToSlavesSockets或者rdbSaveBackground返回失败，说明创建后台子进程失败。需要断开所有处于REDIS_REPL_WAIT_BGSAVE_START状态的从节点的连接；

&emsp;&emsp;轮训列表server.slaves，找到所有处于状态REDIS_REPL_WAIT_BGSAVE_START的从节点。首先删除该从节点客户端标志位中的REDIS_SLAVE标记，然后将其从server.slaves中删除；回复从节点错误信息，然后增加REDIS_CLOSE_AFTER_REPLY标记到客户端标志位中，也就是回复完错误消息后，立即关闭与该从节点的连接；最后返回；




&emsp;&emsp;如果当前进行的是有硬盘复制的RDB转储，则轮训列表server.slaves，找到其中处于状态REDIS_REPL_WAIT_BGSAVE_START的从节点，调用replicationSetupSlaveForFullResync函数将其状态置为REDIS_REPL_WAIT_BGSAVE_END，并且发送"+FULLRESYNC<runid> <offset>"回复；

&emsp;&emsp;因为无硬盘复制的RDB数据转储，已经在rdbSaveToSlavesSockets中进行过该过程了，所以这里只处理有硬盘复制的情况。

最后，调用replicationScriptCacheFlush。

> **无硬盘复制情况**
>
> rdbSaveToSlavesSockets函数里会直接遍历所有处于slave->replstate == SLAVE_STATE_WAIT_BGSAVE_START状态的slave，为他们创建replicationSetupSlaveForFullResync()函数修改状态，然后创建RDB子进程，为所有的刚刚修改状态的slave进行rdbSaveRioWithEOFMark()即发送RDB网络流而不创建RDB文件



&emsp;&emsp;下面是函数replicationSetupSlaveForFullResync的代码：

```c
int replicationSetupSlaveForFullResync(redisClient *slave, long long offset) {
    char buf[128];
    int buflen;
 
    slave->psync_initial_offset = offset;
    slave->replstate = REDIS_REPL_WAIT_BGSAVE_END;
    /* We are going to accumulate the incremental changes for this
     * slave as well. Set slaveseldb to -1 in order to force to re-emit
     * a SLEECT statement in the replication stream. */
    server.slaveseldb = -1;
 
    /* Don't send this reply to slaves that approached us with
     * the old SYNC command. */
    if (!(slave->flags & REDIS_PRE_PSYNC)) {
        buflen = snprintf(buf,sizeof(buf),"+FULLRESYNC %s %lld\r\n",
                          server.runid,offset);
        if (write(slave->fd,buf,buflen) != buflen) {
            freeClientAsync(slave);
            return REDIS_ERR;
        }
    }
    return REDIS_OK;
}
```

&emsp;&emsp;在从节点发来"PSYNC"或"SYNC"命令后，为从节点进行完全重同步时，立即调用该函数，更改从节点客户端的复制状态为REDIS_REPL_WAIT_BGSAVE_END；

&emsp;&emsp;首先设置从节点客户端的psync_initial_offset属性为参数offset。一般情况下，参数offset是由getPsyncInitialOffset函数得到，该函数返回主节点上当前的的复制偏移量。但是如果从节点客户端B是在主节点进行RDB转储时新连接到主节点的，并且它找到了一个可以复用RDB数据和输出缓存的从节点A，则需要使用A->psync_initial_offset为参数调用本函数。也就是说，B还需要复用A的初始复制偏移量；

&emsp;&emsp;然后设置从节点客户端复制状态为REDIS_REPL_WAIT_BGSAVE_END，表示等待RDB转储完成；从复制状态变为REDIS_REPL_WAIT_BGSAVE_END的时刻起，主节点就开始在该从节点客户端的输出缓存中，为从节点累积命令流了。因此，设置server.slaveseldb为-1，这样可以在开始累积命令流时，强制增加一条"SELECT"命令到客户端输出缓存中，以免第一条命令没有选择数据库；

&emsp;&emsp;如果该从节点发送的是PSYNC命令，则直接回复给从节点信息："+FULLRESYNC <runid> <offset>"，注意这里是直接调用的write发送的信息，而没有用到输出缓存。这是因为输出缓存此时只能用于缓存命令流。从节点收到该信息后，会保存主节点的运行ID，以及复制偏移的初始值，以便后续断链时可以进行部分重同步。

#### 5.为从节点累计命令流

&emsp;&emsp;从主节点在为从节点执行BGSAVE操作的时刻起，准确的说是从节点的复制状态变为REDIS_REPL_WAIT_BGSAVE_END的时刻起，主节点就需要将收到的客户端命令请求，缓存一份到从节点的输出缓存中，也就是为从节点累积命令流。等到从节点状态变为REDIS_REPL_ONLINE时，就可以将累积的命令流发送给从节点了，从而保证了从节点的数据库状态能够与主节点保持一致。

&emsp;&emsp;前面提到过，主节点收到”SYNC”或”PSYNC”命令后，调用syncCommand时处理时，就需要保证从节点的输出缓存是空的，而且即使是需要回复从节点"+FULLRESYNC"时，也是调用write，将信息直接发送给从节点客户端，而没有使用客户端的输出缓存。这就是因为要使用客户端的输出缓存来为从节点累积命令流。




&emsp;&emsp;当主节点收到客户端发来的命令请求后，会调用call函数执行相应的命令处理函数。在call函数的最后，有下面的语句：

```c
    /* Propagate the command into the AOF and replication link */
    if (flags & REDIS_CALL_PROPAGATE) {
        int flags = REDIS_PROPAGATE_NONE;
 
        if (c->flags & REDIS_FORCE_REPL) flags |= REDIS_PROPAGATE_REPL;
        if (c->flags & REDIS_FORCE_AOF) flags |= REDIS_PROPAGATE_AOF;
        if (dirty)
            flags |= (REDIS_PROPAGATE_REPL | REDIS_PROPAGATE_AOF);
        if (flags != REDIS_PROPAGATE_NONE)
            propagate(c->cmd,c->db->id,c->argv,c->argc,flags);
    }
```

&emsp;&emsp;上面的语句中，dirty表示在执行命令处理函数时，数据库状态是否发生了变化。只要dirty不为0，就会为flags增加REDIS_PROPAGATE_REPL和REDIS_PROPAGATE_AOF标记。从而调用propagate，该函数会调用replicationFeedSlaves将该命令传播给从节点。

&emsp;&emsp;replicationFeedSlaves函数的代码如下：

```c
/* Propagate write commands to slaves, and populate the replication backlog
 * as well. This function is used if the instance is a master: we use
 * the commands received by our clients in order to create the replication
 * stream. Instead if the instance is a slave and has sub-slaves attached,
 * we use replicationFeedSlavesFromMaster() */
void replicationFeedSlaves(list *slaves, int dictid, robj **argv, int argc) {
    listNode *ln;
    listIter li;
    int j, len;
    char llstr[LONG_STR_SIZE];

    /* If the instance is not a top level master, return ASAP: we'll just proxy
     * the stream of data we receive from our master instead, in order to
     * propagate *identical* replication stream. In this way this slave can
     * advertise the same replication ID as the master (since it shares the
     * master replication history and has the same backlog and offsets). */
    if (server.masterhost != NULL) return;

    /* If there aren't slaves, and there is no backlog buffer to populate,
     * we can return ASAP. */
    if (server.repl_backlog == NULL && listLength(slaves) == 0) return;

    /* We can't have slaves attached and no backlog. */
    serverAssert(!(listLength(slaves) != 0 && server.repl_backlog == NULL));

    /* Send SELECT command to every slave if needed. */
    if (server.slaveseldb != dictid) {
        robj *selectcmd;

        /* For a few DBs we have pre-computed SELECT command. */
        if (dictid >= 0 && dictid < PROTO_SHARED_SELECT_CMDS) {
            selectcmd = shared.select[dictid];
        } else {
            int dictid_len;

            dictid_len = ll2string(llstr,sizeof(llstr),dictid);
            selectcmd = createObject(OBJ_STRING,
                sdscatprintf(sdsempty(),
                "*2\r\n$6\r\nSELECT\r\n$%d\r\n%s\r\n",
                dictid_len, llstr));
        }

        /* Add the SELECT command into the backlog. */
        if (server.repl_backlog) feedReplicationBacklogWithObject(selectcmd);

        /* Send it to slaves. */
        listRewind(slaves,&li);
        while((ln = listNext(&li))) {
            client *slave = ln->value;
            if (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_START) continue;
            addReply(slave,selectcmd);
        }

        if (dictid < 0 || dictid >= PROTO_SHARED_SELECT_CMDS)
            decrRefCount(selectcmd);
    }
    server.slaveseldb = dictid;

    /* Write the command to the replication backlog if any. */
    if (server.repl_backlog) {
        char aux[LONG_STR_SIZE+3];

        /* Add the multi bulk reply length. */
        aux[0] = '*';
        len = ll2string(aux+1,sizeof(aux)-1,argc);
        aux[len+1] = '\r';
        aux[len+2] = '\n';
        feedReplicationBacklog(aux,len+3);

        for (j = 0; j < argc; j++) {
            long objlen = stringObjectLen(argv[j]);

            /* We need to feed the buffer with the object as a bulk reply
             * not just as a plain string, so create the $..CRLF payload len
             * and add the final CRLF */
            aux[0] = '$';
            len = ll2string(aux+1,sizeof(aux)-1,objlen);
            aux[len+1] = '\r';
            aux[len+2] = '\n';
            feedReplicationBacklog(aux,len+3);
            feedReplicationBacklogWithObject(argv[j]);
            feedReplicationBacklog(aux+len+1,2);
        }
    }

    /* Write the command to every slave. */
    listRewind(slaves,&li);
    while((ln = listNext(&li))) {
        client *slave = ln->value;

        /* Don't feed slaves that are still waiting for BGSAVE to start */
        if (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_START) continue;

        /* Feed slaves that are waiting for the initial SYNC (so these commands
         * are queued in the output buffer until the initial SYNC completes),
         * or are already in sync with the master. */

        /* Add the multi bulk length. */
        addReplyMultiBulkLen(slave,argc);

        /* Finally any additional argument that was not stored inside the
         * static buffer if any (from j to argc). */
        for (j = 0; j < argc; j++)
            addReplyBulk(slave,argv[j]);
    }
}
```

&emsp;&emsp;该函数用于主节点将收到的客户端命令请求，缓存到积压队列以及所有状态不是REDIS_REPL_WAIT_BGSAVE_START的从节点的输出缓存中。也就是说，**当从节点的状态变为REDIS_REPL_WAIT_BGSAVE_END的那一刻起，主节点就一直会为从节点缓存命令流**。

&emsp;&emsp;这里要注意的是：如果当前命令的数据库id不等于server.slaveseldb的话，就需要向积压队列和所有状态不是REDIS_REPL_WAIT_BGSAVE_START的从节点输出缓存中添加一条"SELECT"命令。这也就是为什么在函数replicationSetupSlaveForFullResync中，将server.slaveseldb置为-1原因了。这样保证第一次调用本函数时，强制增加一条"SELECT"命令到积压队列和从节点输出缓存中。




&emsp;&emsp;这里在向从节点的输出缓存中追加命令流时，调用的是addReply类的函数。这些函数用于将信息添加到客户端的输出缓存中，这些函数首先都会调用prepareClientToWrite函数，注册socket描述符上的可写事件，然后将回复信息写入到客户端输出缓存中。

&emsp;&emsp;但是在从节点的复制状态变为REDIS_REPL_ONLINE之前，是不能将命令流发送给从节点的。因此，需要在prepareClientToWrite函数中进行特殊处理。在该函数中，有下面的代码：

```c
    /* Only install the handler if not already installed and, in case of
     * slaves, if the client can actually receive writes. */
    if (c->bufpos == 0 && listLength(c->reply) == 0 &&
        (c->replstate == REDIS_REPL_NONE ||
         (c->replstate == REDIS_REPL_ONLINE && !c->repl_put_online_on_ack)))
    {
        /* Try to install the write handler. */
        if (aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,
                sendReplyToClient, c) == AE_ERR)
        {
            freeClientAsync(c);
            return REDIS_ERR;
        }
    }
```

&emsp;&emsp;上面的代码保证了，当从节点客户端的复制状态尚未真正的变为REDIS_REPL_ONLINE时，是不会注册socket描述符上的可写事件的。

> c->repl_put_online_on_ack标志就是表示是否注册文件描述符可写事件，为0表示注册，否则不注册；实际意义就是表示真正的REDIS_REPL_ONLINE状态

还需要注意的是，在写事件的回调函数sendReplyToClient中，有下面的代码：

```c
    if (c->bufpos == 0 && listLength(c->reply) == 0) {
        c->sentlen = 0;
        aeDeleteFileEvent(server.el,c->fd,AE_WRITABLE);
 
        ...
    }
```

因此，当输出缓存中的内容全部发给客户端之后，就会删除socket描述符上的可写事件。这就保证了在主节点收到SYNC或PSYNC命令后，从节点的输出缓存为空时，该从节点的socket描述符上是没有注册可写事件的。

#### 6.BGSAVE操作完成

&emsp;&emsp;当主节点在后台执行BGSAVE的子进程结束之后，主节点父进程wait到该子进程的退出状态后，会调用updateSlavesWaitingBgsave进行BGSAVE的收尾工作。

&emsp;&emsp;前面在”SYNC或PSYNC命令的处理”一节中提到过，如果主节点为从节点在后台进行RDB数据转储时，如果有新的从节点的SYNC或PSYNC命令到来。则在该新从节点无法复用当前正在转储的RDB数据的情况下，主节点需要在当前BGSAVE操作之后，重新进行一次BGSAVE操作。这就是在updateSlavesWaitingBgsave函数中进行的。

&emsp;&emsp;updateSlavesWaitingBgsave函数的代码如下：

```c
void updateSlavesWaitingBgsave(int bgsaveerr, int type) {
    listNode *ln;
    int startbgsave = 0;
    int mincapa = -1;
    listIter li;
 
    listRewind(server.slaves,&li);
    while((ln = listNext(&li))) {
        redisClient *slave = ln->value;
 
        if (slave->replstate == REDIS_REPL_WAIT_BGSAVE_START) {
            startbgsave = 1;
            mincapa = (mincapa == -1) ? slave->slave_capa :
                                        (mincapa & slave->slave_capa);
        } else if (slave->replstate == REDIS_REPL_WAIT_BGSAVE_END) {
            struct redis_stat buf;
 
            /* If this was an RDB on disk save, we have to prepare to send
             * the RDB from disk to the slave socket. Otherwise if this was
             * already an RDB -> Slaves socket transfer, used in the case of
             * diskless replication, our work is trivial, we can just put
             * the slave online. */
            if (type == REDIS_RDB_CHILD_TYPE_SOCKET) {
                redisLog(REDIS_NOTICE,
                    "Streamed RDB transfer with slave %s succeeded (socket). Waiting for REPLCONF ACK from slave to enable streaming",
                        replicationGetSlaveName(slave));
                /* Note: we wait for a REPLCONF ACK message from slave in
                 * order to really put it online (install the write handler
                 * so that the accumulated data can be transfered). However
                 * we change the replication state ASAP, since our slave
                 * is technically online now. */
                slave->replstate = REDIS_REPL_ONLINE;
                slave->repl_put_online_on_ack = 1;
                slave->repl_ack_time = server.unixtime; /* Timeout otherwise. */
            } else {
                if (bgsaveerr != REDIS_OK) {
                    freeClient(slave);
                    redisLog(REDIS_WARNING,"SYNC failed. BGSAVE child returned an error");
                    continue;
                }
                if ((slave->repldbfd = open(server.rdb_filename,O_RDONLY)) == -1 ||
                    redis_fstat(slave->repldbfd,&buf) == -1) {
                    freeClient(slave);
                    redisLog(REDIS_WARNING,"SYNC failed. Can't open/stat DB after BGSAVE: %s", strerror(errno));
                    continue;
                }
                slave->repldboff = 0;
                slave->repldbsize = buf.st_size;
                slave->replstate = REDIS_REPL_SEND_BULK;
                slave->replpreamble = sdscatprintf(sdsempty(),"$%lld\r\n",
                    (unsigned long long) slave->repldbsize);
 
                aeDeleteFileEvent(server.el,slave->fd,AE_WRITABLE);
                // 在这里注册新的回调发送RDB文件
                if (aeCreateFileEvent(server.el, slave->fd, AE_WRITABLE, sendBulkToSlave, slave) == AE_ERR) {
                    freeClient(slave);
                    continue;
                }
            }
        }
    }
    if (startbgsave) startBgsaveForReplication(mincapa);
}
```

​		参数bgsaveerr表示后台子进程的退出状态；type如果为REDIS_RDB_CHILD_TYPE_DISK，表示是有硬盘复制的RDB数据；如果为REDIS_RDB_CHILD_TYPE_SOCKET，表示是无硬盘复制的RDB数据；

 

​		在函数中，轮训列表server.slaves，针对其中的每一个从节点客户端。如果有从节点客户端当前的复制状态为REDIS_REPL_WAIT_BGSAVE_START，说明该从节点是在后台子进程进行RDB数据转储期间，连接到主节点上的。并且没有合适的其他从节点可以进行复用。这种情况下，需要重新进行RDB数据转储或发送，因此置startbgsave为1，并且置mincapa为，状态为REDIS_REPL_WAIT_BGSAVE_START的所有从节点的"能力"的最小值；



​		如果从节点客户端当前的状态为REDIS_REPL_WAIT_BGSAVE_END，说明该从节点正在等待RDB数据处理完成（等待RDB转储到文件完成或者等待RDB数据发送完成）。

​		如果type为REDIS_RDB_CHILD_TYPE_SOCKET，说明无硬盘复制的RDB数据已发送给该从节点客户端，因此，置该从节点客户端的复制状态为REDIS_REPL_ONLINE，然后置从节点客户端中的repl_put_online_on_ack属性为1，表示在收到该从节点第一个"replconf ack <offset>"命令之后，才真正的调用putSlaveOnline将该从节点置为REDIS_REPL_ONLINE状态，并且开始发送缓存的命令流；这样处理的目的，已经在之前的”完全重同步时，从节点状态转换”一节中解释过了，不再赘述。



​		如果type为REDIS_RDB_CHILD_TYPE_DISK，说明RDB数据已转储到文件，接下来需要把该文件发送给所有从节点客户端。

​		如果bgsaveerr为REDIS_ERR，则直接调用freeClient释放该从节点客户端（无硬盘复制的RDB数据发送，已经在函数backgroundSaveDoneHandlerSocket中处理过这种情况了，因此无需在本函数中处理）；

​		如果bgsaveerr为REDIS_OK，打开RDB文件，描述符记录到slave->repldbfd中；置slave->repldboff为0；置slave->repldbsize为RDB文件大小；置从节点客户端的复制状态为REDIS_REPL_SEND_BULK；

​		置slave->replpreamble为需要发送给从节点客户端的RDB文件的长度信息。从节点通过该信息判断要读取多少字节的RDB数据，这也是为什么有硬盘复制的RDB数据，不需要等待从节点第一个"replconf ack <offset>"命令，而可以直接在发送完RDB数据之后，直接调用putSlaveOnline将该从节点置为REDIS_REPL_ONLINE状态；

​		然后重新注册从节点客户端的socket描述符上的可写事件，事件回调函数为sendBulkToSlave；



​		轮训完所有从节点客户端之后，如果startbgsave为1，则使用mincapa调用函数startBgsaveForReplication，重新开始一次RDB数据处理过程。



**有硬盘复制的RDB数据**

​		有硬盘复制的RDB数据，接下来需要把RDB文件发送给所有从节点。这是通过从节点socket描述符上的可写事件的回调函数sendBulkToSlave实现的。在该函数中，需要用到从节点客户端的下列属性：

```sql
slave->repldbfd，表示打开的RDB文件描述符；

slave->repldbsize，表示RDB文件的大小；

slave->repldboff，表示已经向从节点发送的RDB数据的字节数；

slave->replpreamble，表示需要发送给从节点客户端的RDB文件的长度信息；
```

sendBulkToSlave函数的代码如下：

```c
void sendBulkToSlave(aeEventLoop *el, int fd, void *privdata, int mask) {
    redisClient *slave = privdata;
    REDIS_NOTUSED(el);
    REDIS_NOTUSED(mask);
    char buf[REDIS_IOBUF_LEN];
    ssize_t nwritten, buflen;
 
    /* Before sending the RDB file, we send the preamble as configured by the
     * replication process. Currently the preamble is just the bulk count of
     * the file in the form "$<length>\r\n". */
    if (slave->replpreamble) {
        nwritten = write(fd,slave->replpreamble,sdslen(slave->replpreamble));
        if (nwritten == -1) {
            redisLog(REDIS_VERBOSE,"Write error sending RDB preamble to slave: %s",
                strerror(errno));
            freeClient(slave);
            return;
        }
        server.stat_net_output_bytes += nwritten;
        sdsrange(slave->replpreamble,nwritten,-1);
        if (sdslen(slave->replpreamble) == 0) {
            sdsfree(slave->replpreamble);
            slave->replpreamble = NULL;
            /* fall through sending data. */
        } else {
            return;
        }
    }
 
    /* If the preamble was already transfered, send the RDB bulk data. */
    lseek(slave->repldbfd,slave->repldboff,SEEK_SET);
    buflen = read(slave->repldbfd,buf,REDIS_IOBUF_LEN);
    if (buflen <= 0) {
        redisLog(REDIS_WARNING,"Read error sending DB to slave: %s",
            (buflen == 0) ? "premature EOF" : strerror(errno));
        freeClient(slave);
        return;
    }
    if ((nwritten = write(fd,buf,buflen)) == -1) {
        if (errno != EAGAIN) {
            redisLog(REDIS_WARNING,"Write error sending DB to slave: %s",
                strerror(errno));
            freeClient(slave);
        }
        return;
    }
    slave->repldboff += nwritten;
    server.stat_net_output_bytes += nwritten;
    if (slave->repldboff == slave->repldbsize) {
        close(slave->repldbfd);
        slave->repldbfd = -1;
        aeDeleteFileEvent(server.el,slave->fd,AE_WRITABLE);
        putSlaveOnline(slave);
    }
}
```

​		如果slave->replpreamble不为NULL，说明需要发送给从节点客户端RDB数据的长度信息，因此，直接调用write向从节点客户端发送slave->replpreamble中的信息。如果写入了部分数据，则将slave->replpreamble更新为未发送的数据，如果slave->replpreamble中的数据已全部发送完成，则释放slave->replpreamble，置其为NULL；否则，直接返回，下次可写事件触发时，接着向从节点发送slave->replpreamble信息；

​        

​		如果slave->replpreamble为NULL，说明已经发送完长度信息了，接下来就是要发送实际的RDB数据了。

​		首先调用lseek将文件指针定位到该文件中未发送的位置，也就是slave->repldboff的位置；然后调用read，读取RDB文件中REDIS_IOBUF_LEN个字节到buf中；

​		然后调用write，将已读取的数据发送给从节点客户端，write返回值为nwritten，将其加到slave->repldboff中。

​		如果slave->repldboff的值等于slave->repldbsize，则表示RDB文件中的所有数据都发送完成了，因此关闭打开的RDB文件描述符slave->repldbfd；删除socket描述符上的可写事件，然后调用putSlaveOnline函数，更改该从节点客户端的复制状态为REDIS_REPL_ONLINE，接下来就可以开始向该从节点客户端发送累积的命令流了（尽管此时从节点可能还在进行RDB数据的加载，而无暇处理这些累积的命令流。不过好在有TCP输入缓冲区，可以先暂存下来，如果TCP的输入缓存被填满了，则不会向主节点发送ACK，则主节点的TCP输出缓存的剩余空间就会越来越少，当减少到水位线以下时，就不会在触发可写事件了）；



**无硬盘复制的RDB数据**

​		对于无硬盘复制的RDB数据，主节点收到从节点发来的第一个"replconf ack <offset>"命令之后，才真正的调用putSlaveOnline将该从节点置为REDIS_REPL_ONLINE状态。

​		以上的过程是在replconf命令处理函数replconfCommand中处理的。之前在”从节点建链和握手”小节中，已经看过该函数的部分代码了，接下来是该函数处理"replconf ack <offset>"命令的代码：

```c
        else if (!strcasecmp(c->argv[j]->ptr,"ack")) {
            /* REPLCONF ACK is used by slave to inform the master the amount
             * of replication stream that it processed so far. It is an
             * internal only command that normal clients should never use. */
            long long offset;
 
            if (!(c->flags & REDIS_SLAVE)) return;
            if ((getLongLongFromObject(c->argv[j+1], &offset) != REDIS_OK))
                return;
            if (offset > c->repl_ack_off)
                c->repl_ack_off = offset;
            c->repl_ack_time = server.unixtime;
            /* If this was a diskless replication, we need to really put
             * the slave online when the first ACK is received (which
             * confirms slave is online and ready to get more data). */
            if (c->repl_put_online_on_ack && c->replstate == REDIS_REPL_ONLINE)
                putSlaveOnline(c);
            /* Note: this command does not reply anything! */
            return;
        }
```

​		可见，这里在客户端的repl_put_online_on_ack属性为1，并且复制状态为REDIS_REPL_ONLINE的情况下，调用putSlaveOnline函数，将该从节点的状态真正置为REDIS_REPL_ONLINE，并开始向该从节点发送累积的命令流。



**putSlaveOnline函数**

​		完全重同步的最后一步，就是调用putSlaveOnline函数，将从节点客户端的复制状态置为REDIS_REPL_ONLINE，并开始向该从节点发送累积的命令流。

```c
void putSlaveOnline(redisClient *slave) {
    slave->replstate = REDIS_REPL_ONLINE;
    slave->repl_put_online_on_ack = 0;
    slave->repl_ack_time = server.unixtime; /* Prevent false timeout. */
    if (aeCreateFileEvent(server.el, slave->fd, AE_WRITABLE,
        sendReplyToClient, slave) == AE_ERR) {
        redisLog(REDIS_WARNING,"Unable to register writable event for slave bulk transfer: %s", strerror(errno));
        freeClient(slave);
        return;
    }
    refreshGoodSlavesCount();
    redisLog(REDIS_NOTICE,"Synchronization with slave %s succeeded",
        replicationGetSlaveName(slave));
}
```

 		首先将从节点客户端的复制状态置为REDIS_REPL_ONLINE，置客户端属性slave->repl_put_online_on_ack为0。表示该从节点已完成初始同步，接下来进入命令传播阶段；

​		然后，重新注册该从节点客户端的socket描述符上的可写事件，事件回调函数为sendReplyToClient，用于向从节点发送缓存的命令流。该函数也是向普通客户端回复命令时的回调函数；

​		最后，调用refreshGoodSlavesCount，更新当前状态正常的从节点数量。

### 二. 部分重同步

**环形缓冲区的设计：**

```c
struct redisServer {
    char *repl_backlog;				//基于字符数组的循环缓冲区
	long long repl_backlog_size;	//循环缓冲区总长度
	long long repl_backlog_histlen; //循环缓冲区中当前累积的数据的长度 
    long long repl_backlog_idx;		//循环缓冲区的写指针位置
	long long repl_backlog_off; 	//循环缓冲区最早保存的数据的首字节在全局范围内的偏移 
}
/*
在不断循环覆盖写时，维护全局范围内的偏移值，通过该值即可知道从库复制偏移量是否还在循环缓冲区中
*/
```
​		server.master_repl_offset：一个全局性的计数器。该属性只有存在积压队列的情况下才会增加计数。当存在积压队列时，每次收到客户端发来的，长度为len的请求命令时，就会将server.master_repl_offset增加len。

​		该属性也就是所谓的主节点上的复制偏移量。当从节点发来PSYNC命令后，主节点回复从节点"+FULLRESYNC  <runid> <offset>"消息时，其中的offset就是取的主节点当时的server.master_repl_offset的值。这样当从节点收到该消息后，将该值保存在复制偏移量server.master->reploff中。

​		进入命令传播阶段后，每当主节点收到客户端的命令请求，则将命令的长度增加到server.master_repl_offset上，然后将命令传播给从节点，从节点收到后，也会将命令长度加到server.master->reploff上，从而保证了主节点上的复制偏移量server.master_repl_offset和从节点上的复制偏移量server.master->reploff的一致性。

​		需要注意的，server.master_repl_offset的值并不是严格的从0开始增加的。它只是一个计数器，只要能保证主从节点上的复制偏移量一致即可。比如如果它的初始值为10，发送给从节点后，从节点保存的复制偏移量初始值也为10，当新的命令来临时，主从节点上的复制偏移量都会相应增加该命令的长度，因此这并不影响主从节点上偏移量的一致性。

​		 server.repl_backlog_size：积压队列server.repl_backlog的总容量。

​		server.repl_backlog_idx：在积压队列server.repl_backlog中，每次写入新数据时的起始索引，是一个相对于server.repl_backlog的索引。当server.repl_backlog_idx 等于server.repl_backlog的长度server.repl_backlog_size时，置其值为0，表示从头开始。

​		server.repl_backlog_histlen：积压队列server.repl_backlog中，当前累积的数据量的大小。该值不会超过积压队列的总容量server.repl_backlog_size。

​		server.repl_backlog_off：在积压队列中，最早保存的命令的首字节，在全局范围内（而非积压队列内）的偏移量。在累积命令流时，下列等式恒成立：

​		`server.master_repl_offset - server.repl_backlog_off + 1 = server.repl_backlog_histlen。`

​		以积压队列为例：如果在插入”abcdefg”之前，server.master_repl_offset的初始值为2，则插入”abcdefg”之后，积压队列中当前的数据量，也就是属性server.repl_backlog_histlen的值为7。属性server.master_repl_offset的值变为9，此时命令的首字节为”a”，它在全局的偏移量就是3。满足上面的等式。

​		根据上面的等式，主节点的积压队列中累积的命令流，首字节和尾字节在全局范围内的偏移量分别是server.repl_backlog_off和server.master_repl_offset。


​		当从节点断链重连后，向主节点发送”PSYNC  <runid>  <offset>”消息，其中的<offset>表示需要接收的下一条命令首字节的偏移量。也就是server.master->reploff + 1。

​		主节点判断<offset>的值，如果该值在下面的范围内，就表示可以进行部分重同步：

[server.repl_backlog_off, server.repl_backlog_off + server.repl_backlog_histlen]。如果<offset>的值为server.repl_backlog_off+ server.repl_backlog_histlen，也就是server.master_repl_offset + 1，说明从节点断链期间，主节点没有收到过新的命令请求。

> 环形缓冲区当然也可以用list、deque等方法实现，不过不如上面这种简洁

#### 1. masterTryPartialResynchronization函数

主节点收到从节点的”PSYNC <runid> <offset>”消息后，调用函数masterTryPartialResynchronization尝试进行部分重同步。

```c
int masterTryPartialResynchronization(redisClient *c) {
    long long psync_offset, psync_len;
    char *master_runid = c->argv[1]->ptr;
    char buf[128];
    int buflen;
 
    /* Is the runid of this master the same advertised by the wannabe slave
     * via PSYNC? If runid changed this master is a different instance and
     * there is no way to continue. */
    if (strcasecmp(master_runid, server.runid)) {
        /* Run id "?" is used by slaves that want to force a full resync. */
        if (master_runid[0] != '?') {
            redisLog(REDIS_NOTICE,"Partial resynchronization not accepted: "
                "Runid mismatch (Client asked for runid '%s', my runid is '%s')",
                master_runid, server.runid);
        } else {
            redisLog(REDIS_NOTICE,"Full resync requested by slave %s",
                replicationGetSlaveName(c));
        }
        goto need_full_resync;
    }
 
    /* We still have the data our slave is asking for? */
    if (getLongLongFromObjectOrReply(c,c->argv[2],&psync_offset,NULL) !=
       REDIS_OK) goto need_full_resync;
    if (!server.repl_backlog ||
        psync_offset < server.repl_backlog_off ||
        psync_offset > (server.repl_backlog_off + server.repl_backlog_histlen))
    {
        redisLog(REDIS_NOTICE,
            "Unable to partial resync with slave %s for lack of backlog (Slave request was: %lld).", replicationGetSlaveName(c), psync_offset);
        if (psync_offset > server.master_repl_offset) {
            redisLog(REDIS_WARNING,
                "Warning: slave %s tried to PSYNC with an offset that is greater than the master replication offset.", replicationGetSlaveName(c));
        }
        goto need_full_resync;
    }
 
    /* If we reached this point, we are able to perform a partial resync:
     * 1) Set client state to make it a slave.
     * 2) Inform the client we can continue with +CONTINUE
     * 3) Send the backlog data (from the offset to the end) to the slave. */
    c->flags |= REDIS_SLAVE;
    c->replstate = REDIS_REPL_ONLINE;
    c->repl_ack_time = server.unixtime;
    c->repl_put_online_on_ack = 0;
    listAddNodeTail(server.slaves,c);
    /* We can't use the connection buffers since they are used to accumulate
     * new commands at this stage. But we are sure the socket send buffer is
     * empty so this write will never fail actually. */
    buflen = snprintf(buf,sizeof(buf),"+CONTINUE\r\n");
    if (write(c->fd,buf,buflen) != buflen) {
        freeClientAsync(c);
        return REDIS_OK;
    }
    psync_len = addReplyReplicationBacklog(c,psync_offset);
    redisLog(REDIS_NOTICE,
        "Partial resynchronization request from %s accepted. Sending %lld bytes of backlog starting from offset %lld.",
            replicationGetSlaveName(c),
            psync_len, psync_offset);
    /* Note that we don't need to set the selected DB at server.slaveseldb
     * to -1 to force the master to emit SELECT, since the slave already
     * has this state from the previous connection with the master. */
 
    refreshGoodSlavesCount();
    return REDIS_OK; /* The caller can return, no full resync needed. */
 
need_full_resync:
    /* We need a full resync for some reason... Note that we can't
     * reply to PSYNC right now if a full SYNC is needed. The reply
     * must include the master offset at the time the RDB file we transfer
     * is generated, so we need to delay the reply to that moment. */
    return REDIS_ERR;
}
```

&emsp;&emsp;该函数返回REDIS_ERR表示不能进行部分重同步；返回REDIS_OK表示可以进行部分重同步。

 

&emsp;&emsp;首先比对"PSYNC"命令参数中的运行ID和本身的ID号是否匹配，如果不匹配，则需要进行完全重同步，因此直接返回REDIS_ERR即可；

&emsp;&emsp;然后取出"PSYNC"命令参数中的从节点复制偏移到psync_offset中，该值表示从节点需要接收的下一条命令首字节的偏移量。接下来根据积压队列的状态判断是否可以进行部分重同步，判断的条件上一节中已经讲过了，不再赘述。




&emsp;&emsp;经过上面的检查后，说明可以进行部分重同步了。因此：首先将REDIS_SLAVE标记增加到客户端标志位中；然后将从节点客户端的复制状态置为REDIS_REPL_ONLINE，并且将c->repl_put_online_on_ack置为0。这点很重要，因为只有当c->replstate为REDIS_REPL_ONLINE，并且c->repl_put_online_on_ack为0时，在函数prepareClientToWrite中，才为socket描述符注册可写事件，这样才能将输出缓存中的内容发送给从节点客户端；




&emsp;&emsp;接下来，直接向客户端的socket描述符上输出"+CONTINUE\r\n"命令，这里不能用输出缓存，因为输出缓存只能用于累积命令流。之前主节点向从节点发送的信息很少，因此内核的输出缓存中应该会有空间，因此这里直接的write操作一般不会出错；




&emsp;&emsp;接下来，调用addReplyReplicationBacklog，将积压队列中psync_offset之后的数据复制到客户端输出缓存中，注意这里不需要设置server.slaveseldb为-1，因为从节点是接着上次连接进行的；

&emsp;&emsp;最后，调用refreshGoodSlavesCount，更新当前状态正常的从节点数量；

#### 2. addReplyReplicationBacklog函数

 主节点确认可以为从节点进行部分重同步时，首先就是调用addReplyReplicationBacklog函数，将积压队列中，全局偏移量为offset的字节，到尾字节之间的所有内容，追加到从节点客户端的输出缓存中。该函数的代码如下：

```c
long long addReplyReplicationBacklog(redisClient *c, long long offset) {
    long long j, skip, len;
 
    /* Compute the amount of bytes we need to discard. */
    skip = offset - server.repl_backlog_off;
 
    /* Point j to the oldest byte, that is actaully our
     * server.repl_backlog_off byte. */
    j = (server.repl_backlog_idx +
        (server.repl_backlog_size-server.repl_backlog_histlen)) %
        server.repl_backlog_size;
 
    /* Discard the amount of data to seek to the specified 'offset'. */
    j = (j + skip) % server.repl_backlog_size;
 
    /* Feed slave with data. Since it is a circular buffer we have to
     * split the reply in two parts if we are cross-boundary. */
    len = server.repl_backlog_histlen - skip;
    while(len) {
        long long thislen =
            ((server.repl_backlog_size - j) < len) ?
            (server.repl_backlog_size - j) : len;
 
        addReplySds(c,sdsnewlen(server.repl_backlog + j, thislen));
        len -= thislen;
        j = 0;
    }
    return server.repl_backlog_histlen - skip;
}
```

&emsp;&emsp;在该函数中，首先计算需要在积压队列中跳过的字节数skip，offset为从节点所需数据的首字节的全局偏移量，server.repl_backlog_off表示积压队列中最早累积的命令首字节的全局偏移量，因此skip等于offset - server.repl_backlog_off；

 

&emsp;&emsp;接下来，计算积压队列中，最早累积的命令首字节，在积压队列中的索引j，server.repl_backlog_idx-1表示积压队列中，命令尾字节在积压队列中的索引，server.repl_backlog_size表示积压队列的总容量，server.repl_backlog_histlen表示积压队列中累积的命令的大小，因此得到j的值为：(server.repl_backlog_idx+(server.repl_backlog_size-server.repl_backlog_histlen))%server.repl_backlog_size;



&emsp;&emsp;接下来，将j置为需要数据首字节相对于积压队列中的索引；然后计算总共需要复制的字节数len；然后就是将数据循环追加到从节点客户端的输出缓存中（追加之前，已经在函数syncCommand保证该输出缓存为空）；

#### 3. feedReplicationBacklog函数

```c
void feedReplicationBacklog(void *ptr, size_t len) {
    unsigned char *p = ptr;
 
    server.master_repl_offset += len;
 
    /* This is a circular buffer, so write as much data we can at every
     * iteration and rewind the "idx" index if we reach the limit. */
    while(len) {
        size_t thislen = server.repl_backlog_size - server.repl_backlog_idx;
        if (thislen > len) thislen = len;
        memcpy(server.repl_backlog+server.repl_backlog_idx,p,thislen);
        server.repl_backlog_idx += thislen;
        if (server.repl_backlog_idx == server.repl_backlog_size)
            server.repl_backlog_idx = 0;
        len -= thislen;
        p += thislen;
        server.repl_backlog_histlen += thislen;
    }
    if (server.repl_backlog_histlen > server.repl_backlog_size)
        server.repl_backlog_histlen = server.repl_backlog_size;
    /* Set the offset of the first byte we have in the backlog. */
    server.repl_backlog_off = server.master_repl_offset -
                              server.repl_backlog_histlen + 1;
}
```

&emsp;&emsp;函数中，首先将len增加到主节点复制偏移量server.master_repl_offset中；

&emsp;&emsp;然后进入循环，将ptr追加到积压队列中，在循环中：首先计算本次追加的数据量thislen。server.repl_backlog_size表示积压队列的总容量，server.repl_backlog_idx-1表示积压队列中，累积的命令尾字节在积压队列中的索引，因此thislen等于server.repl_backlog_size-server.repl_backlog_idx，表示在积压队列的尾部之前，还可以追加多少字节。如果thislen大于len，则调整其值；

&emsp;&emsp;然后将p中的thislen个字节，复制到首地址为server.repl_backlog+server.repl_backlog_idx的内存中；

&emsp;&emsp;接下来更新server.repl_backlog_idx的值，如果其值等于积压队列的总容量，表示已经到达积压队列的尾部，因此下一次添加数据时，需要重新从头部开始，因此置server.repl_backlog_idx为0；

&emsp;&emsp;然后更新len和p；

&emsp;&emsp;最后更新server.repl_backlog_histlen的值；该值表示积压队列中累积的命令总量；


​        

&emsp;&emsp;server.repl_backlog_histlen的值最大不能超过积压队列的总容量，因此将所有数据追加到积压队列后，如果其值已经大于总容量server.repl_backlog_size，则重新置其值为server.repl_backlog_size；

&emsp;&emsp;最后，更新server.repl_backlog_off的值，使其满足等式：

`server.repl_backlog_histlen=server.master_repl_offset-server.repl_backlog_off+1`

## 3.4 定时监测函数replicationCron

&emsp;&emsp;主从节点为了探测网络是连通的，每隔一段时间，都会向对方发送一定的心跳信息。

&emsp;&emsp;之前在从节点部分介绍过，从节点在接受完RDB数据之后，清空本身数据库时，以及加载RDB数据时，都会时不时的向主节点发送一个换行符”\n”（通过回调函数replicationSendNewlineToMaster实现）；而且，当从节点本身的复制状态变为REDIS_REPL_CONNECTED之后，每隔1秒钟就会向主节点发送一个"REPLCONF ACK  <offset>"命令。以上的”\n”和"REPLCONF”命令都是从节点向主节点发送的心跳消息。

&emsp;&emsp;主节点每隔一段时间，也会向从节点发送”PING”命令，以及换行符”\n”。这是主节点向从节点发送的心跳消息。




&emsp;&emsp;主从节点收到对方发来的消息后，都会更新一个时间戳。双方都会定时检查各自时间戳的最后更新时间。这样，当主从节点间长时间没有交互时，说明网络出现了问题，主从双方都可以探测到该问题，从而断开连接；

&emsp;&emsp;以上这些探测功能就是在定时执行的函数replicationCron中实现的，该函数每隔1秒钟调用一次。该函数的代码如下：

```c
void replicationCron(void) {
    static long long replication_cron_loops = 0;
 
    /* Non blocking connection timeout? */
    if (server.masterhost &&
        (server.repl_state == REDIS_REPL_CONNECTING ||
         slaveIsInHandshakeState()) &&
         (time(NULL)-server.repl_transfer_lastio) > server.repl_timeout)
    {
        redisLog(REDIS_WARNING,"Timeout connecting to the MASTER...");
        undoConnectWithMaster();
    }
 
    /* Bulk transfer I/O timeout? */
    if (server.masterhost && server.repl_state == REDIS_REPL_TRANSFER &&
        (time(NULL)-server.repl_transfer_lastio) > server.repl_timeout)
    {
        redisLog(REDIS_WARNING,"Timeout receiving bulk data from MASTER... If the problem persists try to set the 'repl-timeout' parameter in redis.conf to a larger value.");
        replicationAbortSyncTransfer();
    }
 
    /* Timed out master when we are an already connected slave? */
    if (server.masterhost && server.repl_state == REDIS_REPL_CONNECTED &&
        (time(NULL)-server.master->lastinteraction) > server.repl_timeout)
    {
        redisLog(REDIS_WARNING,"MASTER timeout: no data nor PING received...");
        freeClient(server.master);
    }
 	...
    /* 每隔1秒钟就会向主节点发送一个"REPLCONF ACK  <offset>"命令 */
    // 此命令不会得到任何回复，只是用来更新client结构的复制偏移量和repl_ack_time时间
    if (server.masterhost && server.master &&
        !(server.master->flags & REDIS_PRE_PSYNC))
        replicationSendAck();
 
    /* If we have attached slaves, PING them from time to time.
     * So slaves can implement an explicit timeout to masters, and will
     * be able to detect a link disconnection even if the TCP connection
     * will not actually go down. */
    listIter li;
    listNode *ln;
    robj *ping_argv[1];
 
    /* First, send PING according to ping_slave_period. */
    if ((replication_cron_loops % server.repl_ping_slave_period) == 0) {
        ping_argv[0] = createStringObject("PING",4);
        replicationFeedSlaves(server.slaves, server.slaveseldb,
            ping_argv, 1);
        decrRefCount(ping_argv[0]);
    }
 
    /* Second, send a newline to all the slaves in pre-synchronization
     * stage, that is, slaves waiting for the master to create the RDB file.
     * The newline will be ignored by the slave but will refresh the
     * last-io timer preventing a timeout. In this case we ignore the
     * ping period and refresh the connection once per second since certain
     * timeouts are set at a few seconds (example: PSYNC response). */
    listRewind(server.slaves,&li);
    while((ln = listNext(&li))) {
        redisClient *slave = ln->value;
 
        if (slave->replstate == REDIS_REPL_WAIT_BGSAVE_START ||
            (slave->replstate == REDIS_REPL_WAIT_BGSAVE_END &&
             server.rdb_child_type != REDIS_RDB_CHILD_TYPE_SOCKET))
        {
            if (write(slave->fd, "\n", 1) == -1) {
                /* Don't worry, it's just a ping. */
            }
        }
    }
 
    /* Disconnect timedout slaves. */
    if (listLength(server.slaves)) {
        listIter li;
        listNode *ln;
 
        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            redisClient *slave = ln->value;
 
            if (slave->replstate != REDIS_REPL_ONLINE) continue;
            if (slave->flags & REDIS_PRE_PSYNC) continue;
            if ((server.unixtime - slave->repl_ack_time) > server.repl_timeout)
            {
                redisLog(REDIS_WARNING, "Disconnecting timedout slave: %s",
                    replicationGetSlaveName(slave));
                freeClient(slave);
            }
        }
    }
 
    /* If we have no attached slaves and there is a replication backlog
     * using memory, free it after some (configured) time. */
    if (listLength(server.slaves) == 0 && server.repl_backlog_time_limit &&
        server.repl_backlog)
    {
        time_t idle = server.unixtime - server.repl_no_slaves_since;
 
        if (idle > server.repl_backlog_time_limit) {
            freeReplicationBacklog();
            redisLog(REDIS_NOTICE,
                "Replication backlog freed after %d seconds "
                "without connected slaves.",
                (int) server.repl_backlog_time_limit);
        }
    }
 
    /* If AOF is disabled and we no longer have attached slaves, we can
     * free our Replication Script Cache as there is no need to propagate
     * EVALSHA at all. */
    if (listLength(server.slaves) == 0 &&
        server.aof_state == REDIS_AOF_OFF &&
        listLength(server.repl_scriptcache_fifo) != 0)
    {
        replicationScriptCacheFlush();
    }
 
    /* If we are using diskless replication and there are slaves waiting
     * in WAIT_BGSAVE_START state, check if enough seconds elapsed and
     * start a BGSAVE.
     *
     * This code is also useful to trigger a BGSAVE if the diskless
     * replication was turned off with CONFIG SET, while there were already
     * slaves in WAIT_BGSAVE_START state. */
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1) {
        time_t idle, max_idle = 0;
        int slaves_waiting = 0;
        int mincapa = -1;
        listNode *ln;
        listIter li;
 
        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            redisClient *slave = ln->value;
            if (slave->replstate == REDIS_REPL_WAIT_BGSAVE_START) {
                idle = server.unixtime - slave->lastinteraction;
                if (idle > max_idle) max_idle = idle;
                slaves_waiting++;
                mincapa = (mincapa == -1) ? slave->slave_capa :
                                            (mincapa & slave->slave_capa);
            }
        }
 
        if (slaves_waiting && max_idle > server.repl_diskless_sync_delay) {
            /* Start a BGSAVE. Usually with socket target, or with disk target
             * if there was a recent socket -> disk config change. */
            startBgsaveForReplication(mincapa);
        }
    }
 
    /* Refresh the number of slaves with lag <= min-slaves-max-lag. */
    refreshGoodSlavesCount();
    replication_cron_loops++; /* Incremented with frequency 1 HZ. */
}
```

&emsp;&emsp;server.repl_timeout属性是用户在配置文件中配置的"repl-timeout"选项的值，表示主从复制期间最大的超时时间，默认为60秒；

​        

&emsp;&emsp;从从节点向主节点建链开始，到读取完主节点发来的RDB数据为止，也就是复制状态从REDIS_REPL_CONNECTING到REDIS_REPL_TRANSFER期间，每当从节点读取到主节点发来的信息后，都会更新server.repl_transfer_lastio属性为当时的Unix时间戳；

&emsp;&emsp;当从节点处于REDIS_REPL_CONNECTING状态或者握手状态时，并且最后一次更新server.repl_transfer_lastio的时间已经超过了最大超时时间，则调用函数undoConnectWithMaster，断开与主节点间的连接；

&emsp;&emsp;当从节点处于REDIS_REPL_TRANSFER状态（接收RDB数据），并且最后一次更新server.repl_transfer_lastio的时间已经超过了最大超时时间，则调用函数replicationAbortSyncTransfer，终止本次复制过程；




&emsp;&emsp;在读取客户端发来的消息的函数readQueryFromClient中，每次从socket描述符上读取到数据后，就会更新客户端结构中的lastinteraction属性。

&emsp;&emsp;因此，当从节点处于REDIS_REPL_CONNECTED状态时（命令传播阶段），如果最后一次更新server.master->lastinteractio的时间已经超过了最大超时时间，则调用函数freeClient，断开与主节点间的连接；

以上就是**从节点**探测网络是否连通的方法；




&emsp;&emsp;如果当前从节点的复制状态为REDIS_REPL_CONNECT，则调用connectWithMaster开始向主节点发起建链请求。从节点收到客户端发来的”SLAVEOF”命令，或从节点实例启动，从配置文件中读取到了"slaveof"选项后，就将复制状态置为REDIS_REPL_CONNECT，而在此处开始向主节点发起TCP建链；




&emsp;&emsp;如果当前从节点的server.master属性已配置好，说明该从节点已处于REDIS_REPL_CONNECTED状态，并且主节点支持PSYNC命令的情况下，调用函数replicationSendAck向主节点发送"REPLCONF ACK <offset>"消息，这就是从节点向主节点发送心跳消息；




&emsp;&emsp;主节点每隔一定时间也会向从节点发送心跳消息，以使从节点可以更新属性server.repl_transfer_lastio的值。

&emsp;&emsp;首先是每隔server.repl_ping_slave_period秒，向从节点输出缓存以及积压队列中追加"PING"命令；

&emsp;&emsp;然后就是轮训列表server.slaves，对于处于REDIS_REPL_WAIT_BGSAVE_START状态的从节点，或者处于REDIS_REPL_WAIT_BGSAVE_END状态的从节点，且当目前是无硬盘复制的RDB转储时，直接调用write向从节点发送一个换行符；




&emsp;&emsp;当主节点将从节点的复制状态置为REDIS_REPL_ONLINE后，每当收到从节点发来的换行符"\n"（从节点加载RDB数据时发送）或者"REPLCONF ACK <offset>"信息时，就会更新该从节点客户端的repl_ack_time属性。

&emsp;&emsp;因此，主节点轮训server.slaves列表，如果其中的某个从节点的repl_ack_time属性的最近一次的更新时间，已经超过了最大超时时间，则调用函数freeClient，断开与从节点间的连接；

&emsp;&emsp;以上就是**主节点**探测网络是否连通的方法；




&emsp;&emsp;在freeClient函数中，每当释放了一个从节点客户端后，都会判断列表server.slaves当前长度，如果其长度为0，说明该主节点已经没有连接的从节点了，因此就会设置属性server.repl_no_slaves_since为当时的时间戳；

&emsp;&emsp;server.repl_backlog_time_limit属性值表示当主节点没有从节点连接时，积压队列最长的存活时间，该值默认为1个小时。

&emsp;&emsp;因此，如果主节点当前已没有从节点连接，并且配置了server.repl_backlog_time_limit属性值，并且积压队列还存在的情况下，则判断属性server.repl_no_slaves_since最近一次更新时间是否已经超过配置的server.repl_backlog_time_limit属性值，若已超过，则调用freeReplicationBacklog释放积压队列；


​        

&emsp;&emsp;如果主节点当前已没有从节点连接，并且Redis实例关闭了AOF功能，并且列表server.repl_scriptcache_fifo的长度非0，则调用函数replicationScriptCacheFlush；




&emsp;&emsp;之前在函数syncCommand中介绍过，如果当前没有进行RDB数据转储，则当支持无硬盘复制的RDB数据的从节点的"PSYNC"命令到来时，并非立即启动BGSAVE操作，而是等待一段时间再开始。这是因为无硬盘复制的RDB数据无法复用，Redis通过这种方式来等待更多的从节点到来，从而减少执行BGSAVE操作的次数；

&emsp;&emsp;配置文件中"repl-diskless-sync-delay"选项的值，记录在server.repl_diskless_sync_delay中，该值就是主节点等待的最大时间。

&emsp;&emsp;因此，轮训列表server.slaves，针对其中处于REDIS_REPL_WAIT_BGSAVE_START状态的从节点，得到这些从节点的空闲时间的最大值max_idle，以及能力的最小值mincapa；

&emsp;&emsp;轮训完之后，如果max_idle大于选项server.repl_diskless_sync_delay的值，则以参数mincapa调用函数startBgsaveForReplication，开始BGSAVE操作；




&emsp;&emsp;最后，调用refreshGoodSlavesCount，更新当前状态正常的从节点数量。



# 4 哨兵

## 4.1 概述

```c
struct sentinelState {
    char myid[CONFIG_RUN_ID_SIZE+1]; /* This sentinel ID. */
    uint64_t current_epoch;         /* Current epoch. */
    dict *masters;      /* Dictionary of master sentinelRedisInstances.
                           Key is the instance name, value is the
                           sentinelRedisInstance structure pointer. */
    int tilt;           /* Are we in TILT mode? */
    int running_scripts;    /* Number of scripts in execution right now. */
    mstime_t tilt_start_time;       /* When TITL started. */
    mstime_t previous_time;         /* Last time we ran the time handler. */
    list *scripts_queue;            /* Queue of user scripts to execute. */
    char *announce_ip;  /* 向其他哨兵实例发送的IP信息 */
    int announce_port;  /* 向其他哨兵实例发送的端口号 */
	...
} sentinel;
```

```c
/* sentinel.masters的value类型 */
//sentinelRedisInstance 是一个通用的结构体，它不仅可以表示主节点，也可以表示从节点或者其他的哨兵实例
typedef struct sentinelRedisInstance { 
    /* flags 设置为 SRI_MASTER、SRI_SLAVE 或 SRI_SENTINEL 这三种宏定义（在 sentinel.c 文件中）时，就分别表示当前实例是主节点、从节点或其他哨兵 */
    int flags; //实例类型、状态的标记
    char *name;//实例名称
    char *runid;//实例ID
    
    redisAsyncContext *cc; /* Hiredis context for commands. */
    redisAsyncContext *pc; /* Hiredis context for Pub / Sub. */++
 	uint64_t config_epoch; //配置的纪元 
    sentinelAddr *addr; //实例地址信息 
    ... 
    mstime_t s_down_since_time; //主观下线的时长 
    mstime_t o_down_since_time; //客观下线的时长 
    ... 
    dict *sentinels;//监听同一个主节点的其他哨兵实例
	dict *slaves; //主节点的从节点 ...
}
```



![](image-20220330180249192.png)

```c
/* 实际初始化的函数顺序 */  //我看了好久，才看明白哭了

//server.c
		//更换端口
        initSentinelConfig();
		// 替换命令表，初始化sentinel结构
        initSentinel();
//config.c
		loadServerConfig(configfile,options);
		loadServerConfigFromString(config);
//sentinel.c
		sentinelHandleConfiguration(argv+1,argc-1); //这里就创建了主服务器的masters实例了
		/* 此时还没有创建命令连接和订阅连接,但已经放入sentinel.masters表格了 */
		createSentinelRedisInstance() 
//server.c
        sentinelIsRunning(void)
        /* 这个函数初始化时好像什么都没做 */
        sentinelGenerateInitialMonitorEvents();
//server.c
		serverCron()
//sentinel.c	
		sentinelTimer(void)
		sentinelHandleDictOfRedisInstances(dict *instances)
        /* sentinel主逻辑函数，几乎所有主逻辑都在这个函数中 */
		sentinelHandleRedisInstance(sentinelRedisInstance *ri)
        /* 最后在时间事件中建立连接 */
		sentinelReconnectInstance(sentinelRedisInstance *ri)
```

```c
void sentinelReconnectInstance(sentinelRedisInstance *ri) {
    if (ri->link->disconnected == 0) return; //已经建立连接则返回
    if (ri->addr->port == 0) return; /* port == 0 means invalid address. */
    instanceLink *link = ri->link;
    mstime_t now = mstime();
	// 重连时间间隔不能太小
    if (now - ri->link->last_reconn_time < SENTINEL_PING_PERIOD) return;
    ri->link->last_reconn_time = now;

    /* 命令连接. */
    if (link->cc == NULL) {
        link->cc = redisAsyncConnectBind(ri->addr->ip,ri->addr->port,NET_FIRST_BIND_ADDR);
        if (link->cc->err) {
            // 建立连接失败, 重置
            instanceLinkCloseConnection(link,link->cc);
        } else {
            link->pending_commands = 0;
            link->cc_conn_time = mstime();
            link->cc->data = link;
            redisAeAttach(server.el,link->cc);
            redisAsyncSetConnectCallback(link->cc,
                    sentinelLinkEstablishedCallback);
            redisAsyncSetDisconnectCallback(link->cc,
                    sentinelDisconnectCallback);
            sentinelSendAuthIfNeeded(ri,link->cc);
            sentinelSetClientName(ri,link->cc,"cmd");

            /* Send a PING ASAP when reconnecting. */
            sentinelSendPing(ri);
        }
    }
    /* Pub / Sub， 只有与非哨兵节点会建立订阅连接 */
    if ((ri->flags & (SRI_MASTER|SRI_SLAVE)) && link->pc == NULL) {
        link->pc = redisAsyncConnectBind(ri->addr->ip,ri->addr->port,NET_FIRST_BIND_ADDR);
        if (link->pc->err) {
            instanceLinkCloseConnection(link,link->pc);
        } else {
            int retval;

            link->pc_conn_time = mstime();
            link->pc->data = link;
            redisAeAttach(server.el,link->pc);
            redisAsyncSetConnectCallback(link->pc,
                    sentinelLinkEstablishedCallback);
            redisAsyncSetDisconnectCallback(link->pc,
                    sentinelDisconnectCallback);
            sentinelSendAuthIfNeeded(ri,link->pc);
            sentinelSetClientName(ri,link->pc,"pubsub");
            /* Now we subscribe to the Sentinels "Hello" channel. */
            retval = redisAsyncCommand(link->pc,
                sentinelReceiveHelloMessages, ri, "%s %s",
                sentinelInstanceMapCommand(ri,"SUBSCRIBE"),
                SENTINEL_HELLO_CHANNEL);
            if (retval != C_OK) {
                /* If we can't subscribe, the Pub/Sub connection is useless
                 * and we can simply disconnect it and try again. */
                instanceLinkCloseConnection(link,link->pc);
                return;
            }
        }
    }
    /* Clear the disconnected status only if we have both the connections
     * (or just the commands connection if this is a sentinel instance). */
    if (link->cc && (ri->flags & SRI_SENTINEL || link->pc))
        link->disconnected = 0;
}
```



> 以下部分文字说明来自《redis设计与实现》

**监控 — 选主 — 故障转移**

**初始化流程：**

	1. 初始化服务器状态，使用sentinel专用代码，启动并初始化sentinel
	2. 通过配置文件初始化sentinel状态的master属性，里面记录了监视的主服务器列表
	3. 创建连向主服务器的网络连接(命令连接、订阅连接)，订阅连接为__sentinel__:hello频道
	4. 通过INFO命令*(10s一次)*获取主服务器信息，从而得知其下属从服务器，更新主服务器对应实例结构
	5. 获取从服务器信息，创建连向从服务器的连接	
	6. 默认情况下，2s一次向hello频道所有的主从服务器发送信息，包括自己的信息和监视主服务器的信息
	> (往主服务器的命令连接发送PUBLISH命令,其他哨兵会在主服务器的订阅连接收到此信息)
	7. 对于监视同一频道的sentinel会受到其他sentinel的信息，更新主服务实例结构的sentinel字典
	8. 创建连向其他sentinel的命令连接

> 其实INFO命令得到回复的信息非常非常多。基本上服务器的所有信息都在里面。

**监控：**

	1. 每秒一次的频率向与它创建了命令连接的实例发送PING命令，若在指定实际范围内都是无效回复，则标记为主观下线
	2. 发送命令询问其他sentinel是否同意主服务器已下线，当数量达到配置指定数量时，标记其为客观下线

**选主：**

	1. 由监视这个下线主服务器的各个sentinel协商选举领头sentinel，类似**Raft**领导选举

注意：

>  **监控和选主**都是 **is-master-down** 命令,既用来**询问主观下线**，又用来拉票**，**为哨兵的核心命令。

**故障转移：**

 1. 在已下线主服务器属下的所有从服务器里面，挑选出一个从服务器，并将其转换为主服务器

    > 挑选标准：优先级最大的 > 复制偏移量最大的 > id号小的

 2. 让已下线主服务器属下的所有从服务器改为复制新的主服务器

    > slave of noone 和 slave of <id>

 3. 将已下线的主服务器设置为新主服务器的从服务器，它重新上线时，会成为新主服务器的从服务器

> 从节点也会判断主观下线，但不会客观下线，下线的从服务器不会选为主节点



# 5 集群

```c
typedef struct clusterNode {
    mstime_t ctime; /* Node object creation time. */
    char name[CLUSTER_NAMELEN]; /* Node name, hex string, sha1-size */
    int flags;      /* CLUSTER_NODE_... */
    uint64_t configEpoch; /* 这个节点看到的主从复制小集群纪元 */
    unsigned char slots[CLUSTER_SLOTS/8]; /* slots handled by this node */
    int numslots;   /* Number of slots handled by this node */
    int numslaves;  /* Number of slave nodes, if this is a master */
    struct clusterNode **slaves; /* pointers to slave nodes */
    struct clusterNode *slaveof; /* pointer to the master node. Note that it
                                    may be NULL even if the node is a slave
                                    if we don't have the master node in our
                                    tables. */
    mstime_t ping_sent;      /* Unix time we sent latest ping */
    mstime_t pong_received;  /* Unix time we received the pong */
    mstime_t fail_time;      /* Unix time when FAIL flag was set */
    mstime_t voted_time;     /* Last time we voted for a slave of this master */
    mstime_t repl_offset_time;  /* Unix time we received offset for this node */
    mstime_t orphaned_time;     /* Starting time of orphaned master condition */
    long long repl_offset;      /* Last known repl offset for this node. */
    char ip[NET_IP_STR_LEN];  /* Latest known IP address of this node */
    int port;                   /* Latest known clients port of this node */
    int cport;                  /* Latest known cluster port of this node. */
    clusterLink *link;          /* TCP/IP link with this node */
    list *fail_reports;         /* List of nodes signaling this as failing */
} clusterNode;

```

```c
typedef struct clusterState {
    clusterNode *myself;  /* This node */
    uint64_t currentEpoch;	//整个集群纪元
    int state;            /* CLUSTER_OK, CLUSTER_FAIL, ... */
    int size;             /* Num of master nodes with at least one slot */
    dict *nodes;          /* Hash table of name -> clusterNode structures */
    dict *nodes_black_list; /* Nodes we don't re-add for a few seconds. */
    clusterNode *migrating_slots_to[CLUSTER_SLOTS];
    clusterNode *importing_slots_from[CLUSTER_SLOTS];
    clusterNode *slots[CLUSTER_SLOTS];
    uint64_t slots_keys_count[CLUSTER_SLOTS];
    rax *slots_to_keys;
    /* The following fields are used to take the slave state on elections. */
    mstime_t failover_auth_time; /* Time of previous or next election. */
    int failover_auth_count;    /* Number of votes received so far. */
    int failover_auth_sent;     /* True if we already asked for votes. */
    int failover_auth_rank;     /* This slave rank for current auth request. */
    uint64_t failover_auth_epoch; /* Epoch of the current election. */
    int cant_failover_reason;   /* Why a slave is currently not able to
                                   failover. See the CANT_FAILOVER_* macros. */
    /* Manual failover state in common. */
    mstime_t mf_end;            /* Manual failover time limit (ms unixtime).
                                   It is zero if there is no MF in progress. */
    /* Manual failover state of master. */
    clusterNode *mf_slave;      /* Slave performing the manual failover. */
    /* Manual failover state of slave. */
    long long mf_master_offset; /* Master offset the slave needs to start MF
                                   or zero if stil not received. */
    int mf_can_start;           /* If non-zero signal that the manual failover
                                   can start requesting masters vote. */
    /* The followign fields are used by masters to take state on elections. */
    uint64_t lastVoteEpoch;     /* Epoch of the last vote granted. */
    int todo_before_sleep; /* Things to do in clusterBeforeSleep(). */
    /* Messages received and sent by type. */
    long long stats_bus_messages_sent[CLUSTERMSG_TYPE_COUNT];
    long long stats_bus_messages_received[CLUSTERMSG_TYPE_COUNT];
    long long stats_pfail_nodes;    /* Number of nodes in PFAIL status,
                                       excluding nodes without address. */
} clusterState;
```

> 其实每种框架，看上面提供的数据结构对应的成员就大概知道做了什么了。redis集群所有需要的数据结构都在上面。核心为**configEpoch**和currentEpoch，为整个分布式系统的逻辑时钟

**哈希槽**

用CRC16算法得出16bit数对16384取模得出对应的槽。

各redis结点互相交换槽信息以获得全部的槽信息，客户端会缓存一份。

当分片已经迁移，会返回MOVED命令，重定向槽位置，若正在迁移返回ASK。

**分片迁移**

采用`gossip协议`

集群节点每 1 秒一次，通过随机选择五个节点，然后，再 在其中挑选和当前节点最长时间没有发送 Pong 消息的节点，作为目标节点发送Ping消息

> 其实meet ping pong消息结构没有差别，消息头为发送节点信息，带有一些gossip信息，一般为1/10总节点数，但最少三个。

**Meet消息：**

客户端或者说管理员用的，把某个节点加入集群，其实就是接受meet命令的节点向对应节点建立连接，通过ping、pong加入集群

**Ping消息：**

消息头：将    **消息类型，发送消息节点的名称、IP、(发送节点的)slots 分布**  写入

> 其实有非常多的信息，复制偏移量也在里面，是主从切换的重要考虑因素

消息体：随机选多个节点的信息放入。如果选出的节点是当前节点自身、正在握手的节点、失联的节 点以及没有地址信息的节点，那么  是不会为这些节点设置 Ping 消息体 的。而对于可能有故障的节点会在消息体末尾构建独特的消息体信息。

> 源码中优先发送fail主观下线信息，以便更快主从切换

**Pong消息：**

节点收到ping消息后，会回复Pong消息，他们使用的是同一个函数，内容逻辑也是一样的，从而达到gossip协议的效果。

> 源码中用pong信息做广播效果，比如发现某个主节点客观下线通知所有人，又比如某个从节点竞选成功，向所有主节点发送pong更新集群信息。

在源码中，管理员操作集群的命令都是cluster XX XX。所以服务器都是通过clusterCommand进行处理，然后在这个处理函数中读取第二个字段来走不同的逻辑。

```c
void clusterCommand(client *c) {
    if (server.cluster_enabled == 0) {
        addReplyError(c,"This instance has cluster support disabled");
        return;
    }

    if (c->argc == 2 && !strcasecmp(c->argv[1]->ptr,"help")) {
        const char *help[] = {
"ADDSLOTS <slot> [slot ...] -- Assign slots to current node.",
"BUMPEPOCH -- Advance the cluster config epoch.",
"COUNT-failure-reports <node-id> -- Return number of failure reports for <node-id>.",
"COUNTKEYSINSLOT <slot> - Return the number of keys in <slot>.",
"DELSLOTS <slot> [slot ...] -- Delete slots information from current node.",
"FAILOVER [force|takeover] -- Promote current replica node to being a master.",
"FORGET <node-id> -- Remove a node from the cluster.",
"GETKEYSINSLOT <slot> <count> -- Return key names stored by current node in a slot.",
"FLUSHSLOTS -- Delete current node own slots information.",
"INFO - Return onformation about the cluster.",
"KEYSLOT <key> -- Return the hash slot for <key>.",
"MEET <ip> <port> [bus-port] -- Connect nodes into a working cluster.",
"MYID -- Return the node id.",
"NODES -- Return cluster configuration seen by node. Output format:",
"    <id> <ip:port> <flags> <master> <pings> <pongs> <epoch> <link> <slot> ... <slot>",
"REPLICATE <node-id> -- Configure current node as replica to <node-id>.",
"RESET [hard|soft] -- Reset current node (default: soft).",
"SET-config-epoch <epoch> - Set config epoch of current node.",
"SETSLOT <slot> (importing|migrating|stable|node <node-id>) -- Set slot state.",
"REPLICAS <node-id> -- Return <node-id> replicas.",
"SLOTS -- Return information about slots range mappings. Each range is made of:",
"    start, end, master and replicas IP addresses, ports and ids",
NULL
        };
        addReplyHelp(c, help);
    } else if (!strcasecmp(c->argv[1]->ptr,"meet") && (c->argc == 4 || c->argc == 5)) {
        /* CLUSTER MEET <ip> <port> [cport] */
        long long port, cport;

        if (getLongLongFromObject(c->argv[3], &port) != C_OK) {
            addReplyErrorFormat(c,"Invalid TCP base port specified: %s",
                                (char*)c->argv[3]->ptr);
            return;
        }
...
    
}
```

可以看到help是所有命令的说明。

# 6 缓存

## 6.1 **旁路缓存**

意味着需要在应用程序中新增缓存逻辑处理的代码，无法修改应用程序源码的应用场景，就无法使用redis做旁路缓存。

## 6.2 **只读缓存**

读操作走缓存，写操作不走缓存

读操作若缓存没有，从数据库读，再更新缓存

写操作先更新数据库，再删除缓存

## 6.3 读写缓存

即读写操作都可以走缓存，有redis宕机数据丢失风险

**缓存淘汰**

LRU  LFU  随机淘汰  按过期时间的先后删除

## **6.4 缓存一致性**

对于读写缓存而言，必须采用同步直写策略，即更新缓存时也要同时更新数据库，要不都得成功或者失败，需要引入事务机制，相对于比较繁琐。

***只读缓存：***

情况一：先删缓存再更新数据库

​				A删除缓存` ->` B读数据失败从数据库读到了旧值并更新缓存使缓冲中存放了旧值`->`A再更新数据库

``----->``造成缓存不一致

​				延迟双删： A更新完数据库后，睡眠一小段时间，再进行一次缓存删除   *（缺点是睡眠时间很难选择）*

​	

情况二：先更新数据库再删缓存 （推荐用法）

​			1. 引入重试机制：采用消息队列里面存放删除缓存的信息，保证一定能删除缓存

​			2. 订阅Mysql  binlog

> 若必须保证一致性，可在redis缓存客户端暂存并发读请求，等数据库更新完、缓存删除后，再读取数据



# 7 分布式锁

**单Redis：** 加锁即创建一个键值对，释放锁即删除这个键值对。为了防止客户端执行命令时崩溃忘记释放锁，给锁设置过期时间。并且用客户端的唯一标识作为键值对的值。

​	**SET命令**，提供了NX和EX选项

* NX，表示当操作的key不存在时，Redis会直接创建；当操作的key已经存在了，则返回NULL值，Redis对key不做任何修改。
* EX，表示设置key的过期时间

可以用以下命令加锁

```sql
SET lockKey uid EX expireTime NX
# 创建key为stockLock，value为客户端ID 1033
SET stockLock 1033 EX 30 NX
```

因为使用了NX，则别的客户端就无法修改了，实现了加锁的效果

**解锁**

通过Lua脚本。通过GET命令获取锁对应key的value，并判断value是否等于客户端自身的ID。如果是就表明客户端正拿着锁，可以进行响应操作，否则脚本直接返回

```lua
if redis.call("get",lockKey) == uid then
    return redis.call("del",lockKey)
else 
    return 0
end
```



**多Redis：**向所有的Redis结点请求锁，并最后向所有Redis结点请求释放锁，都是半数以上通过即可。`RedLock算法`

# 8 事务

## 8.1 ACID

1. 原子性
   * 命令入队时命令错误全部不执行
   * 命令执行时错误跳过命令往下执行
2. 一致性
   * 入队时不执行自然一致
   * 错误命令不执行也能一致
   * crash后清空，用RDB文件回复也能一致
3. **隔离性**
   * 命令入队时必须使用watch机制，观察操作的键是否在执行命令前被修改，若修改则整个事务不执行
   * 若执行前无并发操作，当然隔离性保证
4. 持久性
   * AOF的配置  
   * RDB模型无法保证

## 8.2 具体实现

 Redis通过**MULTl，EXEC，WATCH**，DISCARD等命令来实现事务（transaction）功能。

&emsp;&emsp;事务从MULTI命令开始，之后，该客户端发来的其他命令会被排队，客户端发来EXEC命令之后，Redis会依次执行队列中的命令。并且在执行期间，服务器不会中断事务而改去执行其他客户端的命令请求，它会将事务中的所有命令都执行完毕后，然后才去处理其他客户端的命令请求。

&emsp;&emsp;WATCH命令，使得客户端可以在EXEC命令执行之前，监视任意数量的数据库键，并在EXEC命令执行时，检查被监视的键是否被其他客户端修改了。如果是的话，Redis将拒绝执行事务，并向客户端返回代表事务执行失败的空回复。

&emsp;&emsp;DISCARD命令会清空事务队列，并退出事务模式；

### 一.常规流程

1. **数据结构**

   在表示客户端的结构体redisClient中，具有一个multiState结构的mstate属性

   ```c
   typedef struct redisClient {
       ...
       multiState mstate;      /* MULTI/EXEC state */
       ...
   } redisClient;
   ```

    multiState结构其实就是一个事务队列，用于保存事务中的命令。multiState结构的定义如下：

   ```c
   /* Client MULTI/EXEC state */
   typedef struct multiCmd {
       robj **argv;
       int argc;
       struct redisCommand *cmd;
   } multiCmd;
    
   typedef struct multiState {
       multiCmd *commands;     /* Array of MULTI commands */
       int count;              /* Total number of MULTI commands */
   } multiState;
   
   ```

    multiState中的commands数组属性保存每条命令，使用count标记命令条数。

   **2.MULTI命令**

   当客户端发来MULTI命令之后，该命令的处理函数是multiCommand，代码如下：

   ```c
   
   void multiCommand(redisClient *c) {
       if (c->flags & REDIS_MULTI) {
           addReplyError(c,"MULTI calls can not be nested");
           return;
       }
       c->flags |= REDIS_MULTI;
       addReply(c,shared.ok);
   }
   ```

   向客户端的标志位中增加REDIS_MULTI标记，表示客户端进入事务模式。

   **3.事务命令排队**

   ```c
   int processCommand(redisClient *c) {
       ...
       /* Exec the command */
       if (c->flags & REDIS_MULTI &&
           c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
           c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
       {
           queueMultiCommand(c);
           addReply(c,shared.queued);
       } else {
           call(c,REDIS_CALL_FULL);
           ...
       }
       return REDIS_OK;
   }
   ```

   因此，在事务模式下，只要客户端发来的不是EXEC、DISCARD、MULTI、WATCH这几个事务命令中之一，就会调用queueMultiCommand函数，将命令排队到c->mstate中，然后回复客户端"+QUEUED"信息。

   

   queueMultiCommand函数的代码如下：

   ```c
   void queueMultiCommand(redisClient *c) {
       multiCmd *mc;
       int j;
    
       c->mstate.commands = zrealloc(c->mstate.commands,
               sizeof(multiCmd)*(c->mstate.count+1));
       mc = c->mstate.commands+c->mstate.count;
       mc->cmd = c->cmd;
       mc->argc = c->argc;
       mc->argv = zmalloc(sizeof(robj*)*c->argc);
       memcpy(mc->argv,c->argv,sizeof(robj*)*c->argc);
       for (j = 0; j < c->argc; j++)
           incrRefCount(mc->argv[j]);
       c->mstate.count++;
   }
   ```

   **4.EXEC**

   如果客户端发来了EXEC命令，则调用execCommand函数进行处理。该函数的代码如下：

   ```C
   void execCommand(redisClient *c) {
       int j;
       robj **orig_argv;
       int orig_argc;
       struct redisCommand *orig_cmd;
       int must_propagate = 0; /* Need to propagate MULTI/EXEC to AOF / slaves? */
    
       if (!(c->flags & REDIS_MULTI)) {
           addReplyError(c,"EXEC without MULTI");
           return;
       }
    
       /* Check if we need to abort the EXEC because:
        * 1) Some WATCHed key was touched.
        * 2) There was a previous error while queueing commands.
        * A failed EXEC in the first case returns a multi bulk nil object
        * (technically it is not an error but a special behavior), while
        * in the second an EXECABORT error is returned. */
       if (c->flags & (REDIS_DIRTY_CAS|REDIS_DIRTY_EXEC)) {
           addReply(c, c->flags & REDIS_DIRTY_EXEC ? shared.execaborterr :
                                                     shared.nullmultibulk);
           discardTransaction(c);
           goto handle_monitor;
       }
    
       /* Exec all the queued commands */
       unwatchAllKeys(c); /* Unwatch ASAP otherwise we'll waste CPU cycles */
       orig_argv = c->argv;
       orig_argc = c->argc;
       orig_cmd = c->cmd;
       addReplyMultiBulkLen(c,c->mstate.count);
       for (j = 0; j < c->mstate.count; j++) {
           c->argc = c->mstate.commands[j].argc;
           c->argv = c->mstate.commands[j].argv;
           c->cmd = c->mstate.commands[j].cmd;
    
           /* Propagate a MULTI request once we encounter the first write op.
            * This way we'll deliver the MULTI/..../EXEC block as a whole and
            * both the AOF and the replication link will have the same consistency
            * and atomicity guarantees. */
           if (!must_propagate && !(c->cmd->flags & REDIS_CMD_READONLY)) {
               execCommandPropagateMulti(c);
               must_propagate = 1;
           }
    
           call(c,REDIS_CALL_FULL);
    
           /* Commands may alter argc/argv, restore mstate. */
           c->mstate.commands[j].argc = c->argc;
           c->mstate.commands[j].argv = c->argv;
           c->mstate.commands[j].cmd = c->cmd;
       }
       c->argv = orig_argv;
       c->argc = orig_argc;
       c->cmd = orig_cmd;
       discardTransaction(c);
       /* Make sure the EXEC command will be propagated as well if MULTI
        * was already propagated. */
       if (must_propagate) server.dirty++;
    
   handle_monitor:
       /* Send EXEC to clients waiting data from MONITOR. We do it here
        * since the natural order of commands execution is actually:
        * MUTLI, EXEC, ... commands inside transaction ...
        * Instead EXEC is flagged as REDIS_CMD_SKIP_MONITOR in the command
        * table, and we do it here with correct ordering. */
       if (listLength(server.monitors) && !server.loading)
           replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
   }
   ```

函数中，如果该客户端当前没有处于事务模式下，则回复客户端错误信息；

如果客户端标志位中设置了REDIS_DIRTY_CAS或REDIS_DIRTY_EXEC标记，则回复客户端相应的错误信息，然后调用discardTransaction终止客户端的事务模式，最后给monitor发送消息后直接返回。REDIS_DIRTY_CAS标记表示该客户端WATCH的某个key被修改了；REDIS_DIRTY_EXEC标记表示事务中，某条命令在经过processCommand函数检查时出现了错误，比如找不到该命令，或者命令参数个数出错等。




接下来开始执行事务中的命令：

首先调用unwatchAllKeys，设置客户端c不再WATCH任何key；然后记录客户端当前的命令属性到orig_*中，以便后续恢复；

接下来，轮训c->mstate中排队的每个命令，依次执行命令。注意：在遇到第一个写操作时，需要调用execCommandPropagateMulti函数，先向从节点和AOF文件追加一个MULTI命令；c->mstate中的命令依次执行，某个命令执行失败，不影响后续命令的执行；

最后，将orig_*中记录的命令属性恢复，并调用discardTransaction终止客户端的事务模式；给monitor发送消息后直接返回；

**5.错误处理**

事务期间，可能会遇到两种类型的错误。

一种错误是语法错误，比如参数错误，命令名字出错等。这些错误是在processCommand函数中，调用命令处理函数之前检查出来的。一旦检查出错，则调用flagTransaction函数，向客户端标志位中增加REDIS_DIRTY_EXEC标记：

```c
void flagTransaction(redisClient *c) {
    if (c->flags & REDIS_MULTI)
        c->flags |= REDIS_DIRTY_EXEC;
}
```

这种情况下，该命令不会入队，在最终的EXEC命令执行函数中，直接回复客户端错误信息。

 

还有一种错误是运行错误，这种错误只有真正调用命令执行函数时才能检查出来。比如向一个字符串类型的键，执行RPUSH这样只针对列表的命令。

在EXEC命令处理函数execCommand中，针对排队的命令是依次调用call函数的。因此这种情况下，出错的命令，不会影响后续命令的执行。

### 二.WATCH命令

在表示数据库的redisDb结构中，具有watched_keys字典属性：

```c
typedef struct redisDb {
    ...
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    ...
} redisDb;
```

 该字典以数据库键为key，以列表clients为value；列表clients中记录了WATCH该数据库键的所有客户端。

通过这种结构，可以快速得到某个数据库键当前有哪些客户端在WATCH；


​        

在表示客户端的redisClient结构中，具有watched_keys列表属性

```c
typedef struct redisClient {
    ...
    multiState mstate;      /* MULTI/EXEC state */
    ...
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */
    ...
} redisClient;
```

每个列表元素是一个watchedKey结构，该结构具有key和db属性；列表记录了当前客户端WATCH了那些数据库键：

```c
typedef struct watchedKey {
    robj *key;
    redisDb *db;
} watchedKey;
```

剩下的很简单，执行watch就插入对应数据结构就行。


# 9 惊群

![](image-20220316100645068.png)

对于热点数据，一堆前端服务器向memcache缓存层发送get请求，但是此时某个请求更新了数据库的值并删除了缓存服务器里的对应值，那么这所有的前端服务器就会在得知memcache没有对应数据后，穿透到数据库层，全部向数据库发起get请求，给数据库造成巨大压力，这就是惊群。

facebook的解决方法时，在get失败后，缓存服务器会给与第一个get失败的请求一个租赁lease，接着其他的请求到达时，缓存服务器发现lease存在会告知前端服务器你们先等等，知道第一个服务器携带lease通过put更新了缓存，然后缓存服务器就可回应新的一轮请求了。



# 10 后台线程

Redis 还启动了 3 个线程来执行**关闭 fd、AOF 刷盘、惰性释放 key 的内存**等操作。

采用了生产者消费者模型，即队列+互斥锁+条件变量。

> RDB文件的创建使用了子进程，严格来说redis是多线程多进程的模型

## 10.1 后台异步任务

 `bio.h` `bio.c`

* 总的来说，创建了三个线程分别处理三种不同的任务，任务用链表的方式组织，使用生产者消费者模式，无后台任务时沉睡在条件变量上

```c
/* bio.h*/
#define BIO_CLOSE_FILE    0 /* Deferred close(2) syscall. */
#define BIO_AOF_FSYNC     1 /* Deferred AOF fsync. */
#define BIO_LAZY_FREE     2 /* Deferred objects freeing. */
#define BIO_NUM_OPS       3 /* 线程数，即任务类别数 */

/* bio.c */
static pthread_t bio_threads[BIO_NUM_OPS];
static pthread_mutex_t bio_mutex[BIO_NUM_OPS];
static pthread_cond_t bio_newjob_cond[BIO_NUM_OPS];
static pthread_cond_t bio_step_cond[BIO_NUM_OPS];
// 任务链表，每种任务都有一个链表
static list *bio_jobs[BIO_NUM_OPS];
// 每种任务的数量统计
static unsigned long long bio_pending[BIO_NUM_OPS];
// job任务结构
struct bio_job {
    time_t time; /* Time at which the job was created. */
    /* Job specific arguments pointers */
    void *arg1, *arg2, *arg3;
};

/* 后台线程初始化 */
void bioInit(void) {
    pthread_attr_t attr;
    pthread_t thread;
    int j;

    /* Initialization of state vars and objects */
    for (j = 0; j < BIO_NUM_OPS; j++) {
        pthread_mutex_init(&bio_mutex[j],NULL);
        pthread_cond_init(&bio_newjob_cond[j],NULL);
        pthread_cond_init(&bio_step_cond[j],NULL);
        bio_jobs[j] = listCreate();
        bio_pending[j] = 0;
    }

	... /* 省略了设置attr线程栈大小部分代码 */
        
    /* Ready to spawn our threads. We use the single argument the thread
     * function accepts in order to pass the job ID the thread is
     * responsible of. */
    for (j = 0; j < BIO_NUM_OPS; j++) {
        void *arg = (void*)(unsigned long) j;
        if (pthread_create(&thread,&attr,bioProcessBackgroundJobs,arg) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize Background Jobs.");
            exit(1);
        }
        bio_threads[j] = thread;
    }
}
/* 后台线程对应事件循环，生产者消费者模式 */
void *bioProcessBackgroundJobs(void *arg) {
    struct bio_job *job;
    unsigned long type = (unsigned long) arg; //取出线程执行的任务类型
    sigset_t sigset;

	...
   
    /* 关闭后台线程的SIGALRM信号, 确保只有主线程可以收到此信号 */
    pthread_mutex_lock(&bio_mutex[type]);
    sigemptyset(&sigset);
    sigaddset(&sigset, SIGALRM);
    if (pthread_sigmask(SIG_BLOCK, &sigset, NULL))
        serverLog(LL_WARNING,
            "Warning: can't mask SIGALRM in bio.c thread: %s", strerror(errno));

    while(1) {
        listNode *ln;
        … 
        //从类型为type的任务队列中获取第一个任务 
        ln = listFirst(bio_jobs[type]);
        job = ln->value;
        … 
        //判断当前处理的后台任务类型是哪一种 
        if (type == BIO_CLOSE_FILE) {
            close((long)job->arg1); //如果是关闭文件任务，那就调用close函数
        } else if (type == BIO_AOF_FSYNC) { 
            redis_fsync((long)job->arg1); //如果是AOF同步写任务，那就调用redis_fsy
        } else if (type == BIO_LAZY_FREE) { 
            //如果是惰性删除任务，那根据任务的参数分别调用不同的惰性删除函数执行 
            if (job->arg1) lazyfreeFreeObjectFromBioThread(job->arg1);
            else if (job->arg2 && job->arg3) 
                lazyfreeFreeDatabaseFromBioThread(job->arg2,job->arg3);
            else if (job->arg3) 
                lazyfreeFreeSlotsMapFromBioThread(job->arg3);
        } else { 
            serverPanic("Wrong job type in bioProcessBackgroundJobs().");
        } 
        … //任务执行完成后，调用listDelNode在任务队列中删除该任务 
        listDelNode(bio_jobs[type],ln); //将对应的等待任务个数减一。 
        bio_pending[type]--;
        …
    }
}

/* 创建后台任务API */
void bioCreateBackgroundJob(int type, void *arg1, void *arg2, void *arg3) {
    struct bio_job *job = zmalloc(sizeof(*job));

    job->time = time(NULL);
    job->arg1 = arg1;
    job->arg2 = arg2;
    job->arg3 = arg3;
    pthread_mutex_lock(&bio_mutex[type]);
    listAddNodeTail(bio_jobs[type],job); //尾插
    bio_pending[type]++;	//对应任务数加一
    pthread_cond_signal(&bio_newjob_cond[type]);
    pthread_mutex_unlock(&bio_mutex[type]);
}
```

## 10.2 异步删除与同步删除 

``lazyfree.c``

```c
int dbAsyncDelete(redisDb *db, robj *key) {
	// 直接将过期表中的键删除，安全的，因为是共享对象，只会使refCount减1
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);

	// 只将哈希表中的对应条目去除，但没有释放内存
    dictEntry *de = dictUnlink(db->dict,key->ptr);
    if (de) {
        // 去除对应条目中的val
        robj *val = dictGetVal(de);
        size_t free_effort = lazyfreeGetFreeEffort(val);

		
        if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1) {
            atomicIncr(lazyfree_objects,1);
            // 把条目中的val用后台线程异步清理，因为val可能很大
            bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
            // 然后把val置NULL，防止重复清理
            dictSetVal(db->dict,de,NULL);
        }
    }
	// 然后直接在主线程中清理key 和 哈希表对应条目entry
    if (de) {
        dictFreeUnlinkedEntry(db->dict,de);
        if (server.cluster_enabled) slotToKeyDel(key);
        return 1;
    } else {
        return 0;
    }
}
/* 所以只有val是在后台线程清理， key 和 entry还是同步清理 */
```

```c
/* 同步直接原地清理 */
int dbSyncDelete(redisDb *db, robj *key) {
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);
    if (dictDelete(db->dict,key->ptr) == DICT_OK) {
        if (server.cluster_enabled) slotToKeyDel(key);
        return 1;
    } else {
        return 0;
    }
}
```

# 11 缓存淘汰

**目的：** 为了在 Redis server 内存使用量超过上限值的时候， 筛选一些冷数据出来，把它们从 Redis server 中删除，以保证 server 的内存使用量不超出上限

1. LRU    

   > 包括 allkey 和 只淘汰 过期键, LFU同理

2. LFU

3. 按键的过期时间

4. 随机删除

5. 什么都不做

## 11.1 近似LRU

&emsp;&emsp;首先，Redis并不是每次访问到键都会调用底层的系统调用去获取unix时间以更新key的lru时间，这对性能损耗太大。Redis采用了全局LRU时钟的手法，在键值对创建时获取全局LRU时间作为其访问时间戳，并在每次访问时去获取全局LRU时钟值去更新时间戳。

**全局LRU时间**：即每次Redis定期调用时间事件对应函数`serverCron`时，0.1s一次，此函数就会更新保存在Redis里的全局LRU时钟；但在redis中lru的**默认时间精度为1s**。单位为毫秒，除以1000，精度就为1s了。

&emsp;&emsp;Redis的LRU替换策略处于对内存和性能的考虑，并没有严格采用链表的形式，将所有的键按访问时间连起来。而是采用了**固定大小的待淘汰数据集合**，每次随机选择一些key加入待淘汰的数据集合中。最后，再按照待淘汰集合中key的空闲时间长度，删除空间时间最长的key，近似实现LRU算法的效果。

```C
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; //记录LRU信息，宏定义为24
    int refcount;
    void *ptr;
}robj;
```

```c
void initServerConfig(void) {
    ...
    unsigned int lruclock = getLRUClock(); //获取全局LRU时钟值
    atomicSet(server.lruclock,lruclock);
    ...
}
```

```c
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    ...
    //缓存替换策略为LFU就将lru变量设置为LFU的计数值
    if(server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8 | LFU_INIT_VAL);
    } else {
        o->lru = LRU_CLOCK();
    }
    return o;
}
```

```c
// 查找键值对时更新对应的LRU时钟值，或者LFU计数值
robj *lookupKey(redisDb *db, robj *key, int flags) {
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);
		...
            if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
                updateLFU(val);
            } else {
                val->lru = LRU_CLOCK();
            }
		...
}
```

**实际执行阶段**：处理每个命令时检查是否需要释放内存

![](image-20220330111038711.png)



```c
//processCommand()
if (server.maxmemory && !server.lua_timedout) {
    int out_of_memory = freeMemoryIfNeededAndSafe() == C_ERR;
```

```c
//int freeMemoryIfNeeded(void) 
for (i = 0; i < server.dbnum; i++) {
    db = server.db+i;
    dict = (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) ?
        db->dict : db->expires;
    if ((keys = dictSize(dict)) != 0) {
        //开始采集信息
        evictionPoolPopulate(i, dict, db->dict, pool);
        total_keys += keys;
    }
}
```

紧接着，evictionPoolPopulate 函数会遍历待淘汰的候选键值对集合，也就是 EvictionPoolLRU 数组。在遍历过程中，它会尝试把采样的每一个键值对插入 EvictionPoolLRU 数组，这主要取决于以下两个条件之一：

1. 它能在数组中找到一个尚未插入键值对的空位； 

2. 它能在数组中找到一个空闲时间小于采样键值对空闲时间的键值对。

这两个条件有一个成立的话，evictionPoolPopulate 函数就可以把采样键值对插入EvictionPoolLRU 数组。等所有采样键值对都处理完后，evictionPoolPopulate 函数就完 成对待淘汰候选键值对集合的更新了。

```c
//int freeMemoryIfNeeded(void) 
for (k = EVPOOL_SIZE-1; k >= 0; k--) { //从数组最后一个key开始查找 
    if (pool[k].key == NULL) continue; //当前key为空值，则查找下一个key
	... //从全局哈希表或是expire哈希表中，获取当前key对应的键值对；
	//如果当前key对应的键值对不为空，选择当前key为被淘汰的key 
    if (de) {
        bestkey = dictGetKey(de);
        break;
	} else {} //否则，继续查找下个key 
}
```

最后，一旦选到了被淘汰的 key，freeMemoryIfNeeded 函数就会根据 Redis server 的 惰性删除配置，来执行同步删除或异步删除

```c
if (bestkey) {
    db = server.db+bestdbid;
    robj *keyobj = createStringObject(bestkey,sdslen(bestkey)); 						propagateExpire(db,keyobj,server.lazyfree_lazy_eviction); //如果配置了惰性删除，则进行异步删除 
    if (server.lazyfree_lazy_eviction) 
        dbAsyncDelete(db,keyobj);
	else //否则进行同步删除 
        dbSyncDelete(db,keyobj);
}
```

![](image-20220330235036793.png)

## 11.2 LFU

&emsp;&emsp;在Redis中，lfu和lru只能同时存在一种策略，当选择LFU算法时，它会将redisObject结构体中的lru变量的前16bit放UNIX时间戳，后8bit放访问次数。

&emsp;&emsp;我们知道LFU算法，指的是最近访问最少频率的键被淘汰，这个频率的概念不仅包括了访问次数，还带有时间的概念，因此redis通过在lru变量中同时保存最近访问时间和访问频率来实施这种算法。

&emsp;&emsp;首先，在键值对创建时，它会将全局LRU时钟放入前16bit，然后访问次数8bit默认初始化为5。

&emsp;&emsp;当每次更新时，即每次访问到这个键时，它会检查前一次访问的时间戳，比较前一次时间戳距离这次过了多少分钟，**在默认情况下，访问次数的衰减大小就是 等于上一次访问距离当前的分钟数**，即实现了频率减少的这种概率。接着它会用某种概率，去增加一次访问时间，具体来说就是生成一个随机数，若随机数大于某个阈值，我就给你增加一次访问次数，在你原先访问次数少的时候，增加的概率就更大，否则你原来的访问次数越大，增加的概率就越少，并且当访问次数已经到达最大的8bit数255了，就不再增加访问次数。最后再读取全局时间用于更新前16个bit位。

&emsp;&emsp;最后淘汰键值对时，跟近似LRU算法基本一致。采用了**固定大小的待淘汰数据集合**，每次随机选择一些key加入待淘汰的数据集合中。最后，再按照待淘汰集合中key的访问次数，删除访问次数最小的key，实现LFU算法的效果。

![](image-20220330120620723.png)

# 12 数据淘汰

删除操作实际上是包括了两步子操作

* 子操作一：将被淘汰的键值对从哈希表中去除，这里的哈希表既可能是设置了过期 key 的哈希表，也可能是全局哈希表。
* 子操作二：释放被淘汰键值对所占用的内存空间。

如果这两个子操作一起做，那么就是**同步删除**；如果只做了子操作一，而子操 作二由后台线程来执行，那么就是**异步删除**。

**异步删除：**

1. 将被淘汰的键从哈希表中去除，但此时还没有回收内存空间
2. 计算释放被淘汰的键值对内存空间的开销，如果开销小，直接在主I/O中同步删除
3. 否则创建一个惰性删除任务，交给后台线程完成内存释放

# 13 多线程Redis 6.0

&emsp;&emsp;和我irono不同，不是主Reactor和多个从Reactor结构。我的irono会将整个连接对象分发给从I/O线程，接着整个连接的所有事件处理包括I/O包括业务逻辑都在从I/O线程中做，主Reactor只负责监听和分发Tcp连接对象。

&emsp;&emsp;而Redis的实现很有趣，在redis6.0之前，如果遇到某个套接字可读或可写事件，它会直接对其进行处理，比如直接read或write它，当然write有些差别，是先放到输出缓冲区然后Redis会在进入eventloop之前对其遍历处理。

&emsp;&emsp;但在Redis6.0中，主线程碰到Tcp连接可读事件，并调用读回调函数时，它回调函数里修改了逻辑，它会在满足一些条件时，将这个读事件处理推迟，将这个读事件放入一个全局变量-链表中，相当于对事件循环执行过程中，会将所有读写事件实际I/O推迟，放到读和写两个待执行事件链表中。

&emsp;&emsp;然后主线程在进入下一次事件循环前，同样也会执行一个叫做beforeSleep的回调函数，此函数会执行分发操作，将刚才放到链表里的事件用round-robin的方法均匀的分发给I/O线程中，当然主线程也会分到一部分，接着所有线程开始处理分发到的I/O任务，主线程处理完后会等待I/O线程处理完毕，源码的检查方法是遍历每一个I/O线程的任务链表累计数量，当累计和为0就表示所有I/O线程都处理完了，接着主线程会对所有的处理完I/O的事件进行处理，比如执行对应的命令，所以Redis的多线程实际上只有主线程会执行命令，I/O线程只负责I/O任务或者命令的解析，实际命令执行还是在主线程中做。

&emsp;&emsp;Redis多线程也是一种多I/O线程的实现思路，即它实际分发的是客户端的I/O事件，而我分发的是连接，为什么我不采取Redis的手法呢？因为场景不同，首先Redis的命令的执行全部主线程中，因为执行命令无需加锁，所以它为了满足这个无锁执行命令的需求才采取了这样一种比较复杂的策略，而我没有这样的负担，而且很显然我的处理在分发完Tcp连接后，主Reactor和从Reactor基本就没有联系了，耦合性更低性能也更好，而且我从I/O线程也能执行对应的读写逻辑，因此可以同时处理多个Tcp连接的用户回调函数，而Redis这种设计不行，当然Redis这样设计也是因为自己的需求，毕竟一切的设计都是基于需求。

&emsp;&emsp;而且多reactor的方法是全并行的，各个eventloop只需要管理自己的连接事件或者说文件描述符，各个reactor处理自己的成员。但Redis这种本质上不是完全并行的，而是串行和并行都使用了，比如主线程需要收集所有的I/O读写事件放入对应的链表中，然后在主线程中进行round-robin分发任务，最后在所有I/O线程处理完I/O事件后也只有主线程执行命令，这段时间里其实只有主线程在工作，而从I/O线程都在睡眠，所以它并不是全并行的，性能也不及多Reactor的方案能达到全并行。

> [13 | Redis 6.0多IO线程的效率提高了吗？](file:///G:/华工总文件夹/CS总文件夹/CS资料库/202-Redis源码剖析与实战/03-事件驱动框架和执行模型模块 (7讲)/13丨Redis6.html)

![](image-20220329185321150.png)

```c
/* Initialize the data structures needed for threaded I/O. */
void initThreadedIO(void) {
	...
    /* Spawn and initialize the I/O threads. */
    for (int i = 0; i < server.io_threads_num; i++) {
        /* Things we do for all the threads including the main thread. */
        io_threads_list[i] = listCreate();
        if (i == 0) continue; /* Thread 0 is the main thread. */

        /* Things we do only for the additional threads. */
        pthread_t tid;
        pthread_mutex_init(&io_threads_mutex[i],NULL);
        io_threads_pending[i] = 0;
        /* 这个锁会让I/O线程先进入不了循环,等待主线程给任务给他 */
        //看源码会在handleClientsWithPendingWritesUsingThreads()的startThreadedIO()解锁
        pthread_mutex_lock(&io_threads_mutex[i]); /* Thread will be stopped. */
        if (pthread_create(&tid,NULL,IOThreadMain,(void*)(long)i) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize IO thread.");
            exit(1);
        }
        io_threads[i] = tid;
    }
}
```

```c
void *IOThreadMain(void *myid) {
	...
    while(1) {
        /* Wait for start */
        /* 注意到I/O线程没任务时其实是不断自旋的 */
        for (int j = 0; j < 1000000; j++) {
            if (io_threads_pending[id] != 0) break;
        }

        /* Give the main thread a chance to stop this thread. */
        /* 注意到I/O线程没任务时其实是不断自旋的 */
        if (io_threads_pending[id] == 0) {
            pthread_mutex_lock(&io_threads_mutex[id]);
            pthread_mutex_unlock(&io_threads_mutex[id]);
            continue;
        }
        ...
        listIter li;
        listNode *ln;
        //获取I/O线程要处理的客户端列表
        listRewind(io_threads_list[id],&li);
        while((ln = listNext(&li))) {
            client *c = listNodeValue(ln);
            if (io_threads_op == IO_THREADS_OP_WRITE) {
                writeToClient(c,0);
            } else if (io_threads_op == IO_THREADS_OP_READ) {
                readQueryFromClient(c->conn);
            } else {
                serverPanic("io_threads_op value is unknown");
            }
        }
        listEmpty(io_threads_list[id]);
        // 处理完后，处理的客户端数置0，则主线程察觉到I/O线程处理完毕。原子操作
        io_threads_pending[id] = 0;
		...
    }
}
```

```c
/* 读操作推迟 */
void readQueryFromClient(connection *conn) {
	...
    if (postponeClientRead(c)) return;
    ...
}

int postponeClientRead(client *c) {
    if (server.io_threads_active &&
        server.io_threads_do_reads &&
        !clientsArePaused() &&
        !ProcessingEventsWhileBlocked &&
        !(c->flags & (CLIENT_MASTER|CLIENT_SLAVE|CLIENT_PENDING_READ)))
    {
        c->flags |= CLIENT_PENDING_READ;
        // 将要处理的读操作放入全局变量 clients_pending_read 链表
        listAddNodeHead(server.clients_pending_read, c);
        return 1;
    } else {
        return 0;
    }
}
```

![](image-20220329190510026.png)

> 

