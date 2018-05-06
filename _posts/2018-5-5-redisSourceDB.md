---
title: redis源码解读(九):redisDB
date: 2018-5-5 15:11:30
categories: 
- redis
tags: 
- redis源码解析
- c
---

许久没有更新redis源码解读系列，连之前写的博客都有了些许陌生感，着实难堪，先水两篇找找感觉（万幸《这就是街舞》总决赛挺好看= =）

# DB数据结构
*redis* 是一个内存数据库，除了`redisObject`还需要一个DB数据结构承载所有的数据，这个数据结构就是`redisDB`  

```c
typedef struct redisDb {    // redis的db结构体
    dict *dict;                 // 保存数据库中的键值对
    dict *expires;              // 记录所有key的过期时间
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP) */
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    struct evictionPoolEntry *eviction_pool;    /* Eviction pool of keys */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
} redisDb;
```
`dict`和`expires`是`redisDB`中最主要的两个属性，分别保存了对象数据键值对和key的过期时间，其底层数据结构都是`dict`字典。  我认为可能是由于过期时间并不是数据的固有属性，不是所有的数据都设有过期时间，虽然分开存储需要两次查找，但是却能节省内存开销。  
`blocking_keys`和`ready_keys`主要为了实现`BLPOP`等阻塞命令。  
`watched_keys`用于实现`watch`命令，记录正在被watch的一些key。  
`eviction_pool`是记录可能被lru淘汰的一些备选key。  
`id`为当前数据库的id，redis支持单个服务多数据库。  
`avg_ttl`记录了平均的ttl，并不是一个准确的平均值，是有随机抽样的key计算出来的。  

# 服务器和客户端的数据库
*redis* 有两个特殊的数据结构，`redisServer`和`redisClient`，分别保存了redis服务器和客户端的一些信息。两者都持有redisDb的元素

```c
struct redisServer {    // 服务端
    // ...
    redisDb *db;    // DB数组
    int dbnum;	   // db数量
    // ...
}

typedef struct client { // 客户端
    // ...
    redisDb *db;    // 指向当前选中的db指针
    // ...
}
```

redis服务器在启动的时候回初始化`redisServer`，其中就会初始化`dbnum`个`redisDb`。

```c
void initServer(void) { // 初始化server结构体
    // ...
    server.db = zmalloc(sizeof(redisDb)*server.dbnum);  // 初始化db 字典
    /* Create the Redis databases, and initialize other internal state. */
    for (j = 0; j < server.dbnum; j++) {    // 初始化db参数
        server.db[j].dict = dictCreate(&dbDictType,NULL);
        server.db[j].expires = dictCreate(&keyptrDictType,NULL);
        server.db[j].blocking_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].ready_keys = dictCreate(&setDictType,NULL);
        server.db[j].watched_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].eviction_pool = evictionPoolAlloc();
        server.db[j].id = j;
        server.db[j].avg_ttl = 0;
    }
    // ...
}
```
而redis客户端在连接上redis服务器的时候会实例化一个`redisClient`结构体，用于记录client的各种信息，并且默认会将db指向0号db。如果切换Db需要用`select`命令进行切换。

```c
client *createClient(int fd) {  // 创建一个客户端
    client *c = zmalloc(sizeof(client));
    // ...
    selectDb(c,0);	// 选择0号db
    // ...
}

int selectDb(client *c, int id) {  // 切换db
    if (id < 0 || id >= server.dbnum)
        return C_ERR;
    c->db = &server.db[id]; // 将客户端的db指向对应id的db
    return C_OK;
}
```

# 键值对

## 添加键值对
所有数据在redis中都是以键值对的方式存储，以简单的`set`命令为例：

```c
// 生成字符串对象通用函数
void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {
    long long milliseconds = 0; /* initialized to avoid any harmness warning */

    if (expire) {   // 如果设置了过期时间
        if (getLongLongFromObjectOrReply(c, expire, &milliseconds, NULL) != C_OK)   // 校验是否为数字 不是将错误返回客户端
            return;
        if (milliseconds <= 0) {
            addReplyErrorFormat(c,"invalid expire time in %s",c->cmd->name);    // 将结果返回客户端
            return;
        }
        if (unit == UNIT_SECONDS) milliseconds *= 1000;
    }

    // 实现nx和xx的set方式
    if ((flags & OBJ_SET_NX && lookupKeyWrite(c->db,key) != NULL) ||
        (flags & OBJ_SET_XX && lookupKeyWrite(c->db,key) == NULL))
    {
        addReply(c, abort_reply ? abort_reply : shared.nullbulk);   // 将结果返回客户端
        return;
    }
    setKey(c->db,key,val);  // 设置key
    server.dirty++; // 记录写操作数量
    if (expire) setExpire(c->db,key,mstime()+milliseconds); // 设置过期时间
    notifyKeyspaceEvent(NOTIFY_STRING,"set",key,c->db->id); // 推送set变更推送
    if (expire) notifyKeyspaceEvent(NOTIFY_GENERIC,
        "expire",key,c->db->id);    // 推送过期时间推送
    addReply(c, ok_reply ? ok_reply : shared.ok);   // 将结果返回客户端
}

void setKey(redisDb *db, robj *key, robj *val) {    // 设置键值对
    if (lookupKeyWrite(db,key) == NULL) {   // 不存在
        dbAdd(db,key,val);
    } else {    // 存在覆写
        dbOverwrite(db,key,val);
    }
    incrRefCount(val);  // 引用计数加一
    removeExpire(db,key);   // 移除过期时间字典中的记录
    signalModifiedKey(db,key);  // 通知变更 用于watch命令
}

void dbAdd(redisDb *db, robj *key, robj *val) { // db添加键值对
    sds copy = sdsdup(key->ptr);    // 复制key
    int retval = dictAdd(db->dict, copy, val);  // 往字典中添加键值对

    serverAssertWithInfo(NULL,key,retval == DICT_OK);
    if (val->type == OBJ_LIST) signalListAsReady(db, key);  // 如果是list对象 判断是否有阻塞命令在监听
    if (server.cluster_enabled) slotToKeyAdd(key);  // 集群相关操作
 }

void setExpire(redisDb *db, robj *key, long long when) {    // 设置过期时间
    dictEntry *kde, *de;

    /* Reuse the sds from the main dict in the expire dict */
    kde = dictFind(db->dict,key->ptr);  // 找到对应的字典节点
    serverAssertWithInfo(NULL,key,kde != NULL);
    de = dictReplaceRaw(db->expires,dictGetKey(kde));   // 将过期时间加入expire的字典中 共用同一个key sds对象
    dictSetSignedIntegerVal(de,when);
}
```

整个db的读写操作都是在dict对象的操作上封装了一层，因此没有什么好探究的。主要是加入了很多功能性的逻辑，比如订阅通知的推送，操作数的记录，watch的支持，阻塞命令的支持等等。

需要注意，虽然将过期时间和具体的键值对分开存储，但是expires的key和dict的key是同一个的对象实例，并没用重新复制一个sds对象。  

## 过期键值对删除策略
对于设置了过期时间的键值对，*redis* 有2种删除策略：1. 惰性删除 2. 定期删除  
定期删除是`redisServer`的一种时间事件，每隔一段时间就会进行一次遍历删除。之后分析`redisServer`和事件机制的时候再细说，这里主要看惰性删除。  
所谓惰性删除就是不主动删除键值对，只有在读写数据之前，对需要操作的key进行检测，如果过期则进行删除操作。通过这样的策略，能够减轻cpu的负担，但是对内存不够友好，过期的key不会在第一时间被删除。不过 *redis* 还有定期删除和lru策略，从其他方式上减轻了内存的浪费。  
所有读写数据库的命令在执行之前都会调用`expireIfNeeded`函数，用来判断对应key是否过期

```c
int expireIfNeeded(redisDb *db, robj *key) {    // 判断key是否过期
    mstime_t when = getExpire(db,key);
    mstime_t now;

    if (when < 0) return 0; /* No expire for this key */    // 没有设置

    /* Don't expire anything while loading. It will be done later. */
    if (server.loading) return 0;

    now = server.lua_caller ? server.lua_time_start : mstime(); // lua脚本执行的时候只取lua脚本执行开始时间作为判断
    if (server.masterhost != NULL) return now > when;   // 如果是丛库不进行删除

    /* Return when this key has not expired */
    if (now <= when) return 0;  // 未过期

    /* Delete the key */
    server.stat_expiredkeys++;
    propagateExpire(db,key);    // 如果过期了 针对aof 追加一条del命令
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    return dbDelete(db,key);
}

int dbDelete(redisDb *db, robj *key) {  // 删除db中的键值对
    /* Deleting an entry from the expires dict will not free the sds of
     * the key, because it is shared with the main dictionary. */
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);    // 从过期字典中删除
    if (dictDelete(db->dict,key->ptr) == DICT_OK) { // 从字典中删除
        if (server.cluster_enabled) slotToKeyDel(key);  // 集群操作
        return 1;
    } else {
        return 0;
    }
}
```
`expireIfNeeded`的源码中有几个小细节还是值得关注一下的：  

1. 如果当前redis数据库是从库的话，将不会主动进行过期键值对淘汰，只会返回是否过期；
2. 为了保证aof文件中不冗余过期数据，在淘汰过期key的时候会主动给aof文件添加一条del命令；
3. 从`dbDelete`函数的注释中可以看到上文提到的expires和dict共用key对象，并且在删除expires的时候并不会删除key对象，在后续删除dict键值对时才会进行删除。

前两点比较好理解，第三点是如何做到的呢？明明调用的是同一个`dictDelete`函数啊= =。  
如果对第三篇《dict解读》还有印象的话，会记得dict结构体中有一个叫做`dictType`的结构体属性，这个结构体定义了dict的各种操作，通过实例化的时候注入不同`dictType`的对象，达到类似interface的效果。  
在实例化db的时候，`redisDb`的dict和expires分别注入了不同的dictType对象

```c
typedef struct dictType {   // 各种字典操作
    unsigned int (*hashFunction)(const void *key);  // 计算hash值的函数
    void *(*keyDup)(void *privdata, const void *key);   // 键复制
    void *(*valDup)(void *privdata, const void *obj);   // 值复制
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);  // 键比较
    void (*keyDestructor)(void *privdata, void *key);   // 键销毁
    void (*valDestructor)(void *privdata, void *obj);   // 值销毁
} dictType;

/* Db->dict, keys are sds strings, vals are Redis objects. */
dictType dbDictType = {
    dictSdsHash,                /* hash function */
    NULL,                       /* key dup */
    NULL,                       /* val dup */
    dictSdsKeyCompare,          /* key compare */
    dictSdsDestructor,          /* key destructor */
    dictObjectDestructor   /* val destructor */
};

/* Db->expires */
dictType keyptrDictType = {
    dictSdsHash,               /* hash function */
    NULL,                      /* key dup */
    NULL,                      /* val dup */
    dictSdsKeyCompare,         /* key compare */
    NULL,                      /* key destructor */
    NULL                       /* val destructor */
};

server.db[j].dict = dictCreate(&dbDictType,NULL);
server.db[j].expires = dictCreate(&keyptrDictType,NULL);
```
而`dictDelete`函数中会通过调用`dictType`中的`keyDestructor`和`valDestructor`的函数，对对应的键值对象进行销毁

```c
#define dictFreeVal(d, entry) \
    if ((d)->type->valDestructor) \
        (d)->type->valDestructor((d)->privdata, (entry)->v.val)

#define dictFreeKey(d, entry) \
    if ((d)->type->keyDestructor) \
        (d)->type->keyDestructor((d)->privdata, (entry)->key)

static int dictGenericDelete(dict *d, const void *key, int nofree)  // 移除一个entry
{
    unsigned int h, idx;
    dictEntry *he, *prevHe;
    int table;

    if (d->ht[0].size == 0) return DICT_ERR; /* d->ht[0].table is NULL */
    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);

    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        prevHe = NULL;
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                /* Unlink the element from the list */
                if (prevHe)
                    prevHe->next = he->next;
                else
                    d->ht[table].table[idx] = he->next;
                if (!nofree) {  // 是否调用type中的free
                    dictFreeKey(d, he);	// key的free方法
                    dictFreeVal(d, he);	// val的free方法
                }
                zfree(he);	// 释放dict的entry
                d->ht[table].used--;
                return DICT_OK;
            }
            prevHe = he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) break;
    }
    return DICT_ERR; /* not found */
}
```

由于expires的dictType中的`keyDestructor`和`valDestructor`都是NULL，因此共用的key对象并不会在移除过期时间时进行被释放，而expires的val是一个`int64_t`也不需要特殊的释放函数，直接通过zfree节点就能删除键值对了。


# 总结一波
1. `redisDb`是在字典数据结构上的封装，所有的值都是以键值对的方式存储的
2. `redisServer`持有多个数据库对象，client通过`select`修改指向数据库的指针选择不同数据库
3. `redisDb`的具体键值对和过期时间分别存储在dict和expires两个属性中，两者共用key对象
4. *redis* 通过惰性删除和定期删除两种方式清除过期的键值对
5. 通过给expires和dict属性注入不同的`dictType`，使得删除expires的时候并不会清除共用的key对象

数据结构的相关定义都在`server.h`中，其他一些db操作基本都在`db.c`中，想要深究可以自行查看源码~