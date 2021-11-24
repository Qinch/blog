title: redis4.0_发布/订阅
category: redis 
date: 2017-12-12
tags: [redis4.0,pub/sub,sourcecode]
toc: false
comments: false
---

#### 1. channel的订阅与退订

- UBSCRIBE, UNSUBSCRIBE and PUBLISH implement the Publish/Subscribe messaging paradigm where  senders (publishers) are not programmed to send their messages to specific receivers (subscribers). Rather, published messages are characterized into channels, without knowledge of what (if any) subscribers there may be. Subscribers express interest in one or more channels, and only receive messages that are of interest, without knowledge of what (if any) publishers there are.

- Messages sent by other clients to these channels will be pushed by Redis to all the subscribed clients.

- A client subscribed to one or more channels should not issue commands, although it can subscribe and unsubscribe to and from other channels. The replies to subscription and unsubscription operations are sent in the form of messages, so that the client can just read a coherent stream of messages where the first element indicates the type of message. The commands that are allowed in the context of a subscribed client are SUBSCRIBE, PSUBSCRIBE, UNSUBSCRIBE, PUNSUBSCRIBE, PING and QUIT.
 - Please note that redis-cli will not accept any commands once in subscribed mode and can only quit the mode with Ctrl-C.

- channel订阅实现:在客户端结构体的pubsub_channels(为dict)添加key为订阅的channel的entry,在server结构体的pubsub_chanels（dict）添加key为channel的entry，该entry为所有订阅该channel的list

<!--more-->
#### 2.Pattern-matching订阅与退订
 
- The Redis Pub/Sub implementation supports pattern matching. Clients may subscribe to glob-style patterns in order to receive all the messages sent to channel names matching a given pattern.

- pattern订阅实现：在客户端结构体的pubsub_patterns(为list)添加node,该node的value为订阅的pattern(robj)，在server结构体的pubsub_patterns（为list）添加pubsubPattern Node,该node的client为订阅该pattern的客户端指针,该pattern为订阅pattern的robj.

```
//src/server.h
typedef struct pubsubPattern {
     client *client;                                                
     robj *pattern;                                                   
} pubsubPattern;                                                     
```
#### 3.源码分析
- 订阅channel代码:
```bash
/* Subscribe a client to a channel. Returns 1 if the operation succeeded, or
 * 0 if the client was already subscribed to that channel. */
int pubsubSubscribeChannel(client *c, robj *channel) {
    dictEntry *de;
    list *clients = NULL;
    int retval = 0;

    /* Add the channel to the client -> channels hash table */
	//将订阅的channel添加到client结构体的pubsub_channels字典
	//key为channel
    if (dictAdd(c->pubsub_channels,channel,NULL) == DICT_OK) {
        retval = 1;
        incrRefCount(channel);
        /* Add the client to the channel -> list of clients hash table */
		//将channel添加到server的pubsub_channels字典
		//首先在dict中查找是否该channel存在，如果存在，将c添加到entry的value链表中
		//如果不存在，在字典中添加entry,该entry的key为robj，value为client的链表
        de = dictFind(server.pubsub_channels,channel);
        if (de == NULL) {
			//den=NULL表示该channel在dict不存在
            clients = listCreate();
            dictAdd(server.pubsub_channels,channel,clients);
			//robj引用+1
            incrRefCount(channel);
        } else {
			//该channel在dict中已经存在
            clients = dictGetVal(de);
        }
        listAddNodeTail(clients,c);
    }
    /* Notify the client */
    addReply(c,shared.mbulkhdr[3]);
    addReply(c,shared.subscribebulk);
    addReplyBulk(c,channel);
    addReplyLongLong(c,clientSubscriptionsCount(c));
    return retval;
}

```

- 订阅pattern代码:

```bash
/* Subscribe a client to a pattern. Returns 1 if the operation succeeded, or 0 if the client was already subscribed to that pattern. */
//订阅一个pattern
int pubsubSubscribePattern(client *c, robj *pattern) {
    int retval = 0;

	//首先查找链表中是否存在该pattern
    if (listSearchKey(c->pubsub_patterns,pattern) == NULL) {
        retval = 1;
        pubsubPattern *pat;
		//如果不存在，则添加到client的pubsub_patterns链表结尾
        listAddNodeTail(c->pubsub_patterns,pattern);
		//增加引用计数
        incrRefCount(pattern);

		//将该pattern添加到server的pubsub_patterns链表 
        pat = zmalloc(sizeof(*pat));
        pat->pattern = getDecodedObject(pattern);
        pat->client = c;
        listAddNodeTail(server.pubsub_patterns,pat);
    }
    /* Notify the client */
    addReply(c,shared.mbulkhdr[3]);
    addReply(c,shared.psubscribebulk);
    addReplyBulk(c,pattern);
    addReplyLongLong(c,clientSubscriptionsCount(c));
    return retval;
}
```

#### 4.参考文献

《[redis.io](https://redis.io/topics/pubsub)》

《[redis_reading](https://github.com/Qinch/redis_reading/tree/read/)》
