title: redis4.0_输入/输出缓冲区处理
category: redis 
date: 2017-12-13
tags: [redis4.0,event,network,sourcecode]
toc: false
comments: false
---

之前读了一下redis事件处理器的代码[点我](http://chinchao.xyz/2017/12/10/redis4-event/)，今天无所事事，看了下redis对输入缓冲区(querybuf)和输出缓冲区(buf/replylist)的代码，记录一下学习过程。

#### 1. 读缓冲区处理
读取fd数据,然后放入client的输入缓冲区(querybuf)的整个调用栈如下：

````bash
1211	int processMultibulkBuffer(client *c) {
(gdb) bt
#0  processMultibulkBuffer (c=c@entry=0x7ffff6f15e00) at networking.c:1211
#1  0x000000000043c736 in processInputBuffer (c=0x7ffff6f15e00)
    at networking.c:1408
#2  0x00000000004262de in aeProcessEvents (
    eventLoop=eventLoop@entry=0x7ffff6e340a0, flags=flags@entry=11) at ae.c:452
#3  0x000000000042670b in aeMain (eventLoop=0x7ffff6e340a0) at ae.c:495
#4  0x00000000004232d6 in main (argc=<optimized out>, argv=0x7fffffffe3e8)
    at server.c:3897
```

<!--more-->
#### 1.1 读事件注册

- 在redis代码src/server.c文件中void initServer(void)函数首先完成了socket创建，bind, listen操作，然后在事件处理器上注册了accept处理的handler(function: acceptTcpHandler)

```bash
/* Create an event handler for accepting new connections in TCP and Unix
 * domain sockets. */

    //fd事件
    //在事件处理器上注册handler, 处理listen fd上的可读事件(表示有新的连接到来)
    for (j = 0; j < server.ipfd_count; j++) {
	//在每个listfd上创建event
	//aeCreateFileEvent means epoll_ctl
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
```

- 其中acceptTcpHandler的实现如下:

acceptCommonHandler函数为cfd（新连接fd）创建对应的client结构体，并且为cfd注册读事件

```bash
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
    char cip[NET_IP_STR_LEN];
    UNUSED(el);
    UNUSED(mask);
    UNUSED(privdata);

    //max表示每次listenfd可读的时候，accept获取连接的最大执行次数
    while(max--) {
	//accept封装
	//cip为客户端的ip地址，cport为客户端的端口
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        if (cfd == ANET_ERR) {
            if (errno != EWOULDBLOCK)
                serverLog(LL_WARNING,
                    "Accepting client connection: %s", server.neterr);
            return;
        }
        serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);
	//给accept fd创建client结构体
        acceptCommonHandler(cfd,0,cip);
    }
}
```
- acceptCommonHandler函数中调用createClient函数创建客户端对应的结构体对象:

```bash
    //创建客户端对应的数结构体
    if ((c = createClient(fd)) == NULL) {
        serverLog(LL_WARNING,
            "Error registering fd event for the new client: %s (fd=%d)",
            strerror(errno),fd);
        close(fd); /* May be already closed, just ignore errors */
        return;
    }
```
- 其中createClient函数调用aeCreateFileEvent在事件处理器上注册该fd的读事件（function:readQueryFromClient）：

```bash
 /* passing -1 as fd it is possible to create a non connected client.
     * This is useful since all the commands needs to be executed
     * in the context of a client. When commands are executed in other
     * contexts (for instance a Lua script) we need a non connected client. */
	//fd =-1表示伪客户端
    if (fd != -1) {
		//设置为非阻塞模式
        anetNonBlock(NULL,fd);
        anetEnableTcpNoDelay(NULL,fd);
        if (server.tcpkeepalive)
            anetKeepAlive(NULL,fd,server.tcpkeepalive);
		//读取客户端发送的消息
		//注册fd读回调函数
        if (aeCreateFileEvent(server.el,fd,AE_READABLE,
            readQueryFromClient, c) == AE_ERR)
        {
            close(fd);
			//释放c
            zfree(c);
            return NULL;
        }
    }
```
- readQueryFromClient函数实现如下：

```bash
//当fd可读的时候，回调函数
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    //每个连接的client结构体的privdata为对应client指针，即c
    client *c = (client*) privdata;
    int nread, readlen;
    size_t qblen;
    UNUSED(el);
    UNUSED(mask);

    //IO buffer 16K
    readlen = PROTO_IOBUF_LEN;
    /* If this is a multi bulk request, and we are processing a bulk reply
     * that is large enough, try to maximize the probability that the query
     * buffer contains exactly the SDS string representing the object, even
     * at the risk of requiring more read(2) calls. This way the function
     * processMultiBulkBuffer() can avoid copying buffers to create the
     * Redis Object representing the argument. */
    //multibulklen表示待从读取的参数的个数
    //bulklen表示当前参数的长度
    //客户端为redis-cli
    //如果还有待读取的bulk（参数）,并且上次读取的最后一个参数的数据不完整，
    //并且上次读取的最后一个参数大于等于PROTO_MUBULK_BIG_ARG
    if (c->reqtype == PROTO_REQ_MULTIBULK && c->multibulklen && c->bulklen != -1
        && c->bulklen >= PROTO_MBULK_BIG_ARG)
    {
	//待读取的数据长度
        int remaining = (unsigned)(c->bulklen+2)-sdslen(c->querybuf);

	//确定该次读取的数据长度
	//如果大于PROTO_IOBUF_LEN,则读取PROTO_IOBUF_LEN
	//否则读取全部剩余的数据
        if (remaining < readlen) readlen = remaining;
    }

    //qlen表示当前buf中的数据量
    qblen = sdslen(c->querybuf);
    //peak of querybuff size
    //判断是否更新峰值
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
    //makeroom for readlen
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    //读取客户端请求到querybuf缓冲区
    nread = read(fd, c->querybuf+qblen, readlen);
    if (nread == -1) {
	//表示读取缓冲区为空
        if (errno == EAGAIN) {
            return;
        } else {
            serverLog(LL_VERBOSE, "Reading from client: %s",strerror(errno));
	    //释放客户端
            freeClient(c);
            return;
        }
        serverLog(LL_VERBOSE, "Client closed connection");
        freeClient(c);
        return;
    } else if (c->flags & CLIENT_MASTER) {
        /* Append the query buffer to the pending (not applied) buffer
         * of the master. We'll use this buffer later in order to have a
         * copy of the string applied by the last command executed. */
		//将读取的数据追加到pending_querbuf末尾
        c->pending_querybuf = sdscatlen(c->pending_querybuf,
                                        c->querybuf+qblen,nread);
    }

    //增加querbuf的长度
    sdsIncrLen(c->querybuf,nread);
    //更新server与客户端最新的交互时间
    c->lastinteraction = server.unixtime;
    if (c->flags & CLIENT_MASTER) c->read_reploff += nread;
    server.stat_net_input_bytes += nread;
    //如果querybuf缓冲区长度大于client_max_querybuf_len
    if (sdslen(c->querybuf) > server.client_max_querybuf_len) {
        sds ci = catClientInfoString(sdsempty(),c), bytes = sdsempty();

        bytes = sdscatrepr(bytes,c->querybuf,64);
        serverLog(LL_WARNING,"Closing client that reached max query buffer length: %s (qbuf initial bytes: %s)", ci, bytes);
        sdsfree(ci);
        sdsfree(bytes);
        freeClient(c);
        return;
    }

    /* Time to process the buffer. If the client is a master we need to
     * compute the difference between the applied offset before and after
     * processing the buffer, to understand how much of the replication stream
     * was actually applied to the master state: this quantity, and its
     * corresponding part of the replication stream, will be propagated to
     * the sub-slaves and to the replication backlog. */
    if (!(c->flags & CLIENT_MASTER)) {
	//处理客户端输入缓冲区
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

```
- 其中processInputBuffer函数的源码如下：

```bash
/* This function is called every time, in the client structure 'c', there is
 * more query buffer to process, because we read more data from the socket
 * or because a client was blocked and later reactivated, so there could be
 * pending query buffer, already representing a full command, to process. */
void processInputBuffer(client *c) {
    server.current_client = c;
    /* Keep processing while there is something in the input buffer */
    while(sdslen(c->querybuf)) {
        /* Return if clients are paused. */
        if (!(c->flags & CLIENT_SLAVE) && clientsArePaused()) break;

        /* Immediately abort if the client is in the middle of something. */
        if (c->flags & CLIENT_BLOCKED) break;

        /* CLIENT_CLOSE_AFTER_REPLY closes the connection once the reply is
         * written to the client. Make sure to not let the reply grow after
         * this flag has been set (i.e. don't process more commands).
         *
         * The same applies for clients we want to terminate ASAP. */
        if (c->flags & (CLIENT_CLOSE_AFTER_REPLY|CLIENT_CLOSE_ASAP)) break;

        /* Determine request type when unknown. */
        if (!c->reqtype) {
	    //redis-cli客户端
            if (c->querybuf[0] == '*') {
                c->reqtype = PROTO_REQ_MULTIBULK;
            } else {
                c->reqtype = PROTO_REQ_INLINE;
            }
        }

        if (c->reqtype == PROTO_REQ_INLINE) {
            if (processInlineBuffer(c) != C_OK) break;
        } else if (c->reqtype == PROTO_REQ_MULTIBULK) {
	    //如果返回C_OK，表示从querybuf中解析出一个完成的命令
            if (processMultibulkBuffer(c) != C_OK) break;
        } else {
            serverPanic("Unknown request type");
        }

        /* Multibulk processing could see a <= 0 length. */
        if (c->argc == 0) {
            resetClient(c);
        } else {
            /* Only reset the client when the command was executed. */
	    //执行从querybuf解析出来的命令
            if (processCommand(c) == C_OK) {
                if (c->flags & CLIENT_MASTER && !(c->flags & CLIENT_MULTI)) {
                    /* Update the applied replication offset of our master. */
                    c->reploff = c->read_reploff - sdslen(c->querybuf);
                }

                /* Don't reset the client structure for clients blocked in a
                 * module blocking command, so that the reply callback will
                 * still be able to access the client argv and argc field.
                 * The client will be reset in unblockClientFromModule(). */
                if (!(c->flags & CLIENT_BLOCKED) || c->btype != BLOCKED_MODULE)
                    resetClient(c);
            }
            /* freeMemoryIfNeeded may flush slave output buffers. This may
             * result into a slave, that may be the active client, to be
             * freed. */
            if (server.current_client == NULL) break;
        }
    }
    server.current_client = NULL;
}

```
- 其中processMultibulkBuffer函数的源码如下：

```bash
/* Process the query buffer for client 'c', setting up the client argument
 * vector for command execution. Returns C_OK if after running the function
 * the client has a well-formed ready to be processed command, otherwise
 * 如果read buffer中不是一个完整的命令，则返回C_ERR
 * C_ERR if there is still to read more buffer to get the full command.
 * The function also returns C_ERR when there is a protocol error: in such a
 * case the client structure is setup to reply with the error and close
 * the connection.
 *
 * This function is called if processInputBuffer() detects that the next
 * command is in RESP format, so the first byte in the command is found
 * to be '*'. Otherwise for inline commands processInlineBuffer() is called. */
int processMultibulkBuffer(client *c) {
    char *newline = NULL;
    int pos = 0, ok;
    long long ll;

	//multibulklen=0表示现在buffer中是一个新的命令
	//此时，首先获取*num中的num字段
    if (c->multibulklen == 0) {
        /* The client should have been reset */
        serverAssertWithInfo(c,NULL,c->argc == 0);

        /* Multi bulk length cannot be read without a \r\n */
		//'\r'字符首次在querbuf出现的位置
        newline = strchr(c->querybuf,'\r');
		//如果querybuf中没有'\r'字符
		//表示不是一个完整的参数
        if (newline == NULL) {
            if (sdslen(c->querybuf) > PROTO_INLINE_MAX_SIZE) {
                addReplyError(c,"Protocol error: too big mbulk count string");
                setProtocolError("too big mbulk count string",c,0);
            }
            return C_ERR;
        }

        /* Buffer should also contain \n */
        if (newline-(c->querybuf) > ((signed)sdslen(c->querybuf)-2))
            return C_ERR;

        /* We know for sure there is a whole line since newline != NULL,
         * so go ahead and find out the multi bulk length. */
        serverAssertWithInfo(c,NULL,c->querybuf[0] == '*');
		//*num 获取num字段
        ok = string2ll(c->querybuf+1,newline-(c->querybuf+1),&ll);
        if (!ok || ll > 1024*1024) {
            addReplyError(c,"Protocol error: invalid multibulk length");
            setProtocolError("invalid mbulk count",c,pos);
            return C_ERR;
        }

		//*num\r\n
		//pos指向num之后下一个字段,即：第一个参数的字符长度字段的首地址($len)
        pos = (newline-c->querybuf)+2;
        if (ll <= 0) {
            sdsrange(c->querybuf,pos,-1);
            return C_OK;
        }

		//表示该cmd的待读取的参数个数
        c->multibulklen = ll;

        /* Setup argv array on client structure */
        if (c->argv) zfree(c->argv);
		//创建multibulklen参数对应的multibulklen个robj
        c->argv = zmalloc(sizeof(robj*)*c->multibulklen);
    }

    serverAssertWithInfo(c,NULL,c->multibulklen > 0);
	
	//mutibulklen表示命令中参数的个数，
	//根据此变量才从buffer中解析每个参数(bulk)
    while(c->multibulklen) {
        /* Read bulk length if unknown */
		//如果bulklen为-1，表示开始一个新的bulk读取
        if (c->bulklen == -1) {
            newline = strchr(c->querybuf+pos,'\r');
            if (newline == NULL) {
                if (sdslen(c->querybuf) > PROTO_INLINE_MAX_SIZE) {
                    addReplyError(c,
                        "Protocol error: too big bulk count string");
                    setProtocolError("too big bulk count string",c,0);
                    return C_ERR;
                }
                break;
            }

            /* Buffer should also contain \n */
            if (newline-(c->querybuf) > ((signed)sdslen(c->querybuf)-2))
                break;

            if (c->querybuf[pos] != '$') {
                addReplyErrorFormat(c,
                    "Protocol error: expected '$', got '%c'",
                    c->querybuf[pos]);
                setProtocolError("expected $ but got something else",c,pos);
                return C_ERR;
            }

			//获取$len中的len字段
            ok = string2ll(c->querybuf+pos+1,newline-(c->querybuf+pos+1),&ll);
            if (!ok || ll < 0 || ll > 512*1024*1024) {
                addReplyError(c,"Protocol error: invalid bulk length");
                setProtocolError("invalid bulk length",c,pos);
                return C_ERR;
            }

			//指向长度为len的字符串的首地址
            pos += newline-(c->querybuf+pos)+2;
            if (ll >= PROTO_MBULK_BIG_ARG) {
                size_t qblen;

                /* If we are going to read a large object from network
                 * try to make it likely that it will start at c->querybuf
                 * boundary so that we can optimize object creation
                 * avoiding a large copy of data. */
                sdsrange(c->querybuf,pos,-1);
                pos = 0;
                qblen = sdslen(c->querybuf);
                /* Hint the sds library about the amount of bytes this string is
                 * going to contain. */
                if (qblen < (size_t)ll+2)
                    c->querybuf = sdsMakeRoomFor(c->querybuf,ll+2-qblen);
            }
			//当前参数（bulk）的长度
            c->bulklen = ll;
        }

        /* Read bulk argument */
        if (sdslen(c->querybuf)-pos < (unsigned)(c->bulklen+2)) {
            /* Not enough data (+2 == trailing \r\n) */
            break;
        } else {
            /* Optimization: if the buffer contains JUST our bulk element
             * instead of creating a new object by *copying* the sds we
             * just use the current sds string. */
			//如果buffer中仅含有一个参数(bulk)
            if (pos == 0 &&
                c->bulklen >= PROTO_MBULK_BIG_ARG &&
                (signed) sdslen(c->querybuf) == c->bulklen+2)
            {
                c->argv[c->argc++] = createObject(OBJ_STRING,c->querybuf);
                sdsIncrLen(c->querybuf,-2); /* remove CRLF */
                /* Assume that if we saw a fat argument we'll see another one
                 * likely... */
				//重新创建一个querybuf
                c->querybuf = sdsnewlen(NULL,c->bulklen+2);
                sdsclear(c->querybuf);
                pos = 0;
            } else {
				//创建String robj
                c->argv[c->argc++] =
                    createStringObject(c->querybuf+pos,c->bulklen);
                pos += c->bulklen+2;
            }
			//表示读取了一个完整的bulk（参数）
            c->bulklen = -1;
			//待读取的bulk（参数）数量
            c->multibulklen--;
        }
    }

    /* Trim to pos */
	//将最后一个’\r‘字符的地址偏移量pos到querbuf结尾的字段截取出来，保存到querbuf
    if (pos) sdsrange(c->querybuf,pos,-1);

    /* We're done when c->multibulk == 0 */
	//表示从querbuf中读取了一个完整命令
    if (c->multibulklen == 0) return C_OK;

    /* Still not ready to process the command */
    return C_ERR;
}
```

#### 2. 写缓冲区处理
redis写缓冲区的处理过程如下：在进入epoll_wait或者select阻塞之前，会执行beforeSleep函数，该函数会调用handleClientsWithPendingWrites函数处理fd中的写缓冲区，进行循环写，如果返回-1，并且此时缓冲区中还有数据，则在该fd注册写事件;如果循环写将输出缓冲区中的数据写完，则此时删除该fd上的写事件.
- 函数调用栈如下
```bash
Thread 1 "redis-server" hit Breakpoint 1, writeToClient (fd=8,
    c=c@entry=0x7ffff6f15e00, handler_installed=handler_installed@entry=0)
    at networking.c:935
935	int writeToClient(int fd, client *c, int handler_installed) {
(gdb) bt
#0  writeToClient (fd=8, c=c@entry=0x7ffff6f15e00,
    handler_installed=handler_installed@entry=0) at networking.c:935
#1  0x00000000004391cb in handleClientsWithPendingWrites ()
    at networking.c:1060
#2  0x0000000000429b61 in beforeSleep (eventLoop=<optimized out>)
    at server.c:1234
#3  0x00000000004266fe in aeMain (eventLoop=0x7ffff6e340a0) at ae.c:494
#4  0x00000000004232d6 in main (argc=<optimized out>, argv=0x7fffffffe3e8)
    at server.c:3897
```

#### 3.参考文献
《[redis_reading](https://github.com/Qinch/redis_reading/tree/read/)》
