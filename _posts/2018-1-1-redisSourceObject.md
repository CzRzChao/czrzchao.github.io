---
title: redis源码解读(八):redis对象
date: 2018-1-1 23:11:30
categories: 
- redis
tags: 
- redis源码解析
- c
---

近来在研读redis3.2.9的源码，虽然网上已有许多redis的源码解读文章，但大都不成系统，且纸上学来终觉浅，遂有该系列博文。部分知识点参照了黄建宏的《Redis设计与实现》。

# redisObject
在自定义的基础数据结构的基础上，*redis* 通过 *redisObject* 封装整合成了对外暴露的5中数据结构。

## 定义
首先看看 *redisObject* 的定义：  

```c
typedef struct redisObject {    // redis对象
    unsigned type:4;    // 类型,4bit
    unsigned encoding:4;    // 编码,4bit
    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */ // 24bit
    int refcount;   // 引用计数
    void *ptr;  // 指向各种基础类型的指针
} robj;
```
其中 *type* 用于标识 *string*、*hash*、*list*、*set*、*zset* 五种数据类型、*encoding* 用于标识底层数据结构。通过这两个字段的组合，同一种数据类型也有多种实现方式，一个完整的映射关系如下表：

|   类型 type  |       编码 encode        |             描述            |
|-------------| ------------------------ | ---------------------------|
| OBJ\_STRING | OBJ\_ENCODING\_INT       | 使用整数实现的字符串对象       |
| OBJ\_STRING | OBJ\_ENCODING\_EMBSTR    | 使用embstr编码实现的字符串对象 |
| OBJ\_STRING | OBJ\_ENCODING\_RAW       | 使用sds实现的字符串对象       |
| OBJ\_LIST   | OBJ\_ENCODING\_QUICKLIST | 使用quicklist实现的列表对象   |
| OBJ\_HASH   | OBJ\_ENCODING\_ZIPLIST   | 使用压缩表实现的hash对象      |
| OBJ\_HASH   | OBJ\_ENCODING\_HT        | 使用字典实现的hash对象        |
| OBJ\_SET    | OBJ\_ENCODING\_INSET     | 使用整数集合实现的集合对象     |
| OBJ\_SET    | OBJ\_ENCODING\_HT        | 使用字典实现的集合对象        |
| OBJ\_ZSET   | OBJ\_ENCODING\_ZIPLIST   | 使用压缩列表实现的有序集合对象 |
| OBJ\_ZSET   | OBJ\_ENCODING\_SKIPLIST  | 使用跳跃表实现的有序集合对象  |

*lru* 用于保存对象的LRU时钟或LFU的频次  
*refcount* 为对象的引用计数，redisObject都是通过简单的引用计数法进行垃圾回收  
*ptr* 保存了指向各种底层数据实例的指针

## 对象创建

```c
// 创建对象
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution), or
     * alternatively the LFU counter. */
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) { // 如果是采用LFU策略
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {    // lru记录具体的lru时钟
        o->lru = LRU_CLOCK();
    }
    return o;
}
```
基础的创建对象函数很简单，申请一个object的空间，记录type和具体数据的指针，并将引用计数置1。针对不同的数据类型 *redis* 又封装了不同的函数  

### string

string有3种编码方式，分别是`OBJ_ENCODING_INT`、`OBJ_ENCODING_EMBSTR`和`OBJ_ENCODING_RAW`。  
当一个字符串能被转换为long时，将会采用`OBJ_ENCODING_INT`

```c
if (len <= 20 && string2l(s,len,&value)) {  // 小于20位切能被转换为long
    if ((server.maxmemory == 0 ||
        !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS)) &&
        value >= 0 &&
        value < OBJ_SHARED_INTEGERS)
    {   // 使用shared数据，节省内存
        decrRefCount(o);
        incrRefCount(shared.integers[value]);
        return shared.integers[value];
    } else {    // 使用int
        if (o->encoding == OBJ_ENCODING_RAW) sdsfree(o->ptr);
        o->encoding = OBJ_ENCODING_INT;
        o->ptr = (void*) value;
        return o;
    }
}
```
其中shared是server的共享数据，主要是保存一些常用数据，用户在使用这部分数据时不用新申请内存直接用shared中的object即可。后续会细说  

而当字符串长度小于44时，会采用`OBJ_ENCODING_EMBSTR`否则就会采用`OBJ_ENCODING_RAW`  

```c
/* Create a string object with EMBSTR encoding if it is smaller than
 * OBJ_ENCODING_EMBSTR_SIZE_LIMIT, otherwise the RAW encoding is
 * used.
 *
 * The current limit of 39 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```
根据注释可以看出，主要是因为使用了jemalloc，想将embstr类型的字符串限定在64byte。那么这个44是怎么来的呢，为何注释写的又是39？  
先关注44，`object`结构体需要占用16byte，当字符串小于44时sds会采用`sdshdr8`保存字符串，`sdshdr8`结构体需要3byte，因此44+16+3=63，最后再加上sds字符串末尾的`\0`，就是64byte了。  
而39则是由于历史原因，之前在sds解读中提及到了3.2和3.0的sds结构体做了较大的变动。在3.0版本`sdshdr`需要8个字节，因此embstr只能保存39个字符。而在版本升级后，并没有将注释变更，可以作为良好编程习惯的反面例子了:)~  

embstr字符串和raw字符串的不同点在于：embstr的sds空间和object的存储空间是同时申请的，是连续的  

```c
robj *createEmbeddedStringObject(const char *ptr, size_t len) {     // 创建embstr
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);   // 同时申请obj和sds的内存
    struct sdshdr8 *sh = (void*)(o+1);

    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }

    sh->len = len;
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;
    if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}
```
这样做将原本的一个string对象的两次内存申请优化到了一次，并且在释放的时候也只需要一个free。由于embstr的所有数据都保存在连续的内存中，可以更好的利用缓存带来的优势。

对于embstr，redis没有提供任何修改的函数。例如当一个embstr被执行`APPEND`命令时，会被先转换为raw字符串，再进行拼接。

```c
o = dbUnshareStringValue(c->db,c->argv[1],o);
o->ptr = sdscatlen(o->ptr,append->ptr,sdslen(append->ptr));

robj *dbUnshareStringValue(redisDb *db, robj *key, robj *o) {   // 将字符串对象转换为raw
    serverAssert(o->type == OBJ_STRING);
    if (o->refcount != 1 || o->encoding != OBJ_ENCODING_RAW) {
        robj *decoded = getDecodedObject(o);    // 将encoding_int转换为raw
        o = createRawStringObject(decoded->ptr, sdslen(decoded->ptr));
        decrRefCount(decoded);  // 引用计数-1
        dbOverwrite(db,key,o);  // 覆盖原有key
    }
    return o;
}
```
> string相关api文档可见：[redis文档](http://www.redis.cn/commands.html#string)  

具体api源码在`t_string.c`中

### list

在3.2.9中quicklist取代了之前的linkedlist和ziplist，quicklist的原理及相关源码解析可以查看：[redis源码解读(六):基础数据结构之quicklist](/redisSourceQuicklist)  
list对象的创建就是对quicklistCreate的简单调用  

```c
robj *createQuicklistObject(void) {
    quicklist *l = quicklistCreate();
    robj *o = createObject(OBJ_LIST,l);
    o->encoding = OBJ_ENCODING_QUICKLIST;
    return o;
}
```
由于只有一种编码，也就没有编码转换等繁琐的操作了。相关的api也都是对quicklist的简单封装，就不对其源码进行解读了。

> list相关api文档可见：[redis文档](http://www.redis.cn/commands.html#list)

具体api源码在`t_list.c`中

### hash

hash对象默认的数据编码为压缩表`OBJ_ENCODING_ZIPLIST`  

```c
robj *createHashObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_HASH, zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}
```
#### hset
以`hset`命令为例，探究一波hash对象的一些编码策略和存储规则

```c
void hsetCommand(client *c) {   // hset和hmset
    int i, created = 0;
    robj *o;

    if ((c->argc % 2) == 1) {   // 参数数量判断
        addReplyError(c,"wrong number of arguments for HMSET");
        return;
    }

    if ((o = hashTypeLookupWriteOrCreate(c,c->argv[1])) == NULL) return;    // 获取hash表，不存在就创建
    hashTypeTryConversion(o,c->argv,2,c->argc-1);   // 尝试转换hash对象编码

    for (i = 2; i < c->argc; i += 2)
        created += !hashTypeSet(o,c->argv[i]->ptr,c->argv[i+1]->ptr,HASH_SET_COPY); // 添加hash数据

    /* HMSET (deprecated) and HSET return value is different. */
    char *cmdname = c->argv[0]->ptr;    // 获取操作命令
    if (cmdname[1] == 's' || cmdname[1] == 'S') {
        /* HSET */
        addReplyLongLong(c, created);   // 通知客户端
    } else {
        /* HMSET */
        addReply(c, shared.ok); // 通知客户端
    }
    signalModifiedKey(c->db,c->argv[1]);    // 通知数据变更 用于事务
    notifyKeyspaceEvent(NOTIFY_HASH,"hset",c->argv[1],c->db->id);   // 推送变更订阅消息
    server.dirty++; // service 执行命令数量加1
}
```
其中`hashTypeLookupWriteOrCreate`用于从db中查找对应数据，如果不存在就创建一个

```c
robj *hashTypeLookupWriteOrCreate(client *c, robj *key) {   //  从db中查找数据 如果不存在就创建一个
    robj *o = lookupKeyWrite(c->db,key);
    if (o == NULL) {    // 不存在则创建
        o = createHashObject();
        dbAdd(c->db,key,o);
    } else {
        if (o->type != OBJ_HASH) {      // 如果不是hash
            addReply(c,shared.wrongtypeerr);
            return NULL;
        }
    }
    return o;
}
```
有了hash对象之后，需要判断当前的编码是否满足要插入数据的需求，如果key或value长度大于64byte将会采用ht字典编码  

```c
void hashTypeTryConversion(robj *o, robj **argv, int start, int end) {  // 尝试将ziplist编码的hash转换为ht编码
    int i;

    if (o->encoding != OBJ_ENCODING_ZIPLIST) return;

    for (i = start; i <= end; i++) {
        if (sdsEncodedObject(argv[i]) &&
            sdslen(argv[i]->ptr) > server.hash_max_ziplist_value)
        {   // 如果key或value的长度大于64byte
            hashTypeConvert(o, OBJ_ENCODING_HT);    // 将ziplist编码转换为ht字典
            break;
        }
    }
}
```
在`hsetCommand`中，调用了`hashTypeSet`函数，这是真正往hash对象中添加数据的函数  

```c
int hashTypeSet(robj *o, sds field, sds value, int flags) { // 往hash对象中添加数据
    int update = 0;

    if (o->encoding == OBJ_ENCODING_ZIPLIST) {  // 压缩表编码
        unsigned char *zl, *fptr, *vptr;

        zl = o->ptr;
        fptr = ziplistIndex(zl, ZIPLIST_HEAD);
        if (fptr != NULL) {
            fptr = ziplistFind(fptr, (unsigned char*)field, sdslen(field), 1);  // 查找是否存在对应hash field
            if (fptr != NULL) {
                vptr = ziplistNext(zl, fptr);   // value存储在field后
                serverAssert(vptr != NULL);
                update = 1;

                zl = ziplistDelete(zl, &vptr);  // 删除存在的值

                zl = ziplistInsert(zl, vptr, (unsigned char*)value, // 插入新值
                        sdslen(value));
            }
        }

        if (!update) {  // 新增 field和value连续存储 filed在前 value在后 插入ziplist的尾部
            zl = ziplistPush(zl, (unsigned char*)field, sdslen(field),
                    ZIPLIST_TAIL);
            zl = ziplistPush(zl, (unsigned char*)value, sdslen(value),
                    ZIPLIST_TAIL);
        }
        o->ptr = zl;

        if (hashTypeLength(o) > server.hash_max_ziplist_entries)    // 如果hash中的元素大于512个，转换为ht字典编码
            hashTypeConvert(o, OBJ_ENCODING_HT);
    } else if (o->encoding == OBJ_ENCODING_HT) {    // ht字典编码
        dictEntry *de = dictFind(o->ptr,field); // 查找对应field
        if (de) {
            sdsfree(dictGetVal(de));
            if (flags & HASH_SET_TAKE_VALUE) {  // 直接使用传入的数据
                dictGetVal(de) = value;
                value = NULL;
            } else {
                dictGetVal(de) = sdsdup(value); // 需要新申请数据空间 复制数据
            }
            update = 1;
        } else {    // 不存在新增
            sds f,v;
            if (flags & HASH_SET_TAKE_FIELD) {
                f = field;
                field = NULL;
            } else {
                f = sdsdup(field);
            }
            if (flags & HASH_SET_TAKE_VALUE) {
                v = value;
                value = NULL;
            } else {
                v = sdsdup(value);
            }
            dictAdd(o->ptr,f,v);    // 添加到hash的字典中
        }
    } else {
        serverPanic("Unknown hash encoding");
    }

    if (flags & HASH_SET_TAKE_FIELD && field) sdsfree(field);   // 释放没有用的field和value
    if (flags & HASH_SET_TAKE_VALUE && value) sdsfree(value);
    return update;
}
```
#### 小结
* 当hash对象的所有键值对都小于64byte且hash对象的键值对数量少于512时会采用ziplist编码，否则会采用ht字典编码
* 当采用ziplist编码时，键值对按照先后顺序存储，每个键值对中field和value连续存储

> hash相关api文档可见：[redis文档](http://www.redis.cn/commands.html#hash)

具体api源码在`t_hash.c`中