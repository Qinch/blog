title: redis4.0_object源码剖析
category: redis 
date: 2017-12-3
tags: [redis4.0,object,sourcecode]
toc: false
comments: false
---

#### 1.redisObject介绍

- Redis使用对象来表示数据库中的key和value,每次在redis中创建一个新的键值对时，至少需要创建两个对象对象:键对象和值对象。
- Redis中的每个对象都由一个redisObject结构体表示。
```bash
typedef struct redisObject {                                          
     //对象的类型
     unsigned type:4;
     //对象采用的编码方式                                             
     unsigned encoding:4;
     //对象最后一次被命令程序访问的时间                             
     unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                             * LFU data (least significant 8 bits frequency
                             * and most significant 16 bits decreas time). */
     //对象引用计数
     int refcount; 
     //指向底层数据结构的指针
     void *ptr;                                                                                                                                                        
} robj;                                                                                                                                                               
```
- redisObject结构体中type表示对象的类型，type的取值如下：

```bash
/* A redis object, that is a type able to hold a string / list / set */

/* The actual Redis Object */
#define OBJ_STRING 0
#define OBJ_LIST 1
#define OBJ_SET 2
#define OBJ_ZSET 3
#define OBJ_HASH 4
```

- redisObject结构体中encoding表示type类型的对象采用哪种数据结构作为底层实现，encoding的取值如下:

```bash
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
//sds
#define OBJ_ENCODING_RAW 0     /* Raw representation */
//"1234"存储为数字1234，即ptr=(void*)1234;
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
```

#### 2.object.c源码分析
- 创建一个redisObject对象
```bash
//创建一个redisObject对象
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
	//sds
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
	//引用计数
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution), or
     * alternatively the LFU counter. */
	//lru字段用于记录对象最后一次被命令程序访问的时间
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }
    return o;
}
```

- redisObject对象refcount字段减1.

```bash
void decrRefCount(robj *o) {
    if (o->refcount == 1) {
        switch(o->type) {
        case OBJ_STRING: freeStringObject(o); break;
        case OBJ_LIST: freeListObject(o); break;
        case OBJ_SET: freeSetObject(o); break;
        case OBJ_ZSET: freeZsetObject(o); break;
        case OBJ_HASH: freeHashObject(o); break;
        case OBJ_MODULE: freeModuleObject(o); break;
        default: serverPanic("Unknown object type"); break;
        }
        zfree(o);
    } else {
        if (o->refcount <= 0) serverPanic("decrRefCount against refcount <= 0");
        if (o->refcount != OBJ_SHARED_REFCOUNT) o->refcount--;
    }
}
```

#### 4.参考文献
《[redis_reading](https://github.com/Qinch/redis_reading/tree/read/)》


