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

# 定义
在自定义的基础数据结构的基础上，*redis* 通过 *redisObject* 封装整合成了对外暴露的5中数据结构。
首先看看 *redisObject* 的定义：  

```c
#define LRU_BITS 24
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

*lru* 用于保存对象的LRU时钟  
*refcount* 为对象的引用计数，redisObject都是通过简单的引用计数法进行垃圾回收  
*ptr* 保存了指向各种底层数据实例的指针

# 对象创建

```c
robj *createObject(int type, void *ptr) {   // 创建一个对象
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution). */
    o->lru = LRU_CLOCK();   // LRU时钟
    return o;
}
```
基础的创建对象函数很简单，申请一个object的空间，记录type和具体数据的指针，并将引用计数置1。针对不同的数据类型 *redis* 又封装了不同的函数  

## string

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

## list

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
由于只有一种编码，也就没有编码转换等繁琐的操作。相关的api也都是对quicklist的简单封装，就不对其源码进行解读了。

> list相关api文档可见：[redis文档](http://www.redis.cn/commands.html#list)

具体api源码在`t_list.c`中

## hash

hash对象默认的数据编码为压缩表`OBJ_ENCODING_ZIPLIST`  

```c
robj *createHashObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_HASH, zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}
```
### hset
以`hset`命令为例，探究一波hash对象的一些编码策略和存储规则

```c
void hsetCommand(client *c) {
    int update;
    robj *o;

    if ((o = hashTypeLookupWriteOrCreate(c,c->argv[1])) == NULL) return;
    hashTypeTryConversion(o,c->argv,2,3);   // 尝试转换hash编码类型
    hashTypeTryObjectEncoding(o,&c->argv[2], &c->argv[3]);
    update = hashTypeSet(o,c->argv[2],c->argv[3]);  // update 0为新增 1位更新
    addReply(c, update ? shared.czero : shared.cone);   // 通知客户端
    signalModifiedKey(c->db,c->argv[1]);    // 通知数据变更 用于事务
    notifyKeyspaceEvent(NOTIFY_HASH,"hset",c->argv[1],c->db->id);   // 推送变更订阅消息
    server.dirty++;
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
int hashTypeSet(robj *o, robj *field, robj *value) {    // 设置一个hash值，如果存在就进行更新
    int update = 0;

    if (o->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *zl, *fptr, *vptr;

        field = getDecodedObject(field);
        value = getDecodedObject(value);

        zl = o->ptr;
        fptr = ziplistIndex(zl, ZIPLIST_HEAD);
        if (fptr != NULL) {
            fptr = ziplistFind(fptr, field->ptr, sdslen(field->ptr), 1);    // 查找是否存在对应hash field
            if (fptr != NULL) {
                /* Grab pointer to the value (fptr points to the field) */
                vptr = ziplistNext(zl, fptr);    // value存储在field后
                serverAssert(vptr != NULL);
                update = 1;

                /* Delete value */
                zl = ziplistDelete(zl, &vptr);  // 删除存在的值

                /* Insert new value */
                zl = ziplistInsert(zl, vptr, value->ptr, sdslen(value->ptr));   // 插入新值
            }
        }

        if (!update) {  // 新增 field和value连续存储 filed在前 value在后 插入ziplist的尾部
            /* Push new field/value pair onto the tail of the ziplist */
            zl = ziplistPush(zl, field->ptr, sdslen(field->ptr), ZIPLIST_TAIL);
            zl = ziplistPush(zl, value->ptr, sdslen(value->ptr), ZIPLIST_TAIL);
        }
        o->ptr = zl;
        decrRefCount(field);
        decrRefCount(value);

        /* Check if the ziplist needs to be converted to a hash table */
        if (hashTypeLength(o) > server.hash_max_ziplist_entries)    // 新增是校验zip保存的对象是否超过512个,超过需要转化dict
            hashTypeConvert(o, OBJ_ENCODING_HT);
    } else if (o->encoding == OBJ_ENCODING_HT) {
        if (dictReplace(o->ptr, field, value)) { /* Insert */
            incrRefCount(field);
        } else { /* Update */
            update = 1;
        }
        incrRefCount(value);
    } else {
        serverPanic("Unknown hash encoding");
    }
    return update;
}
```

### 小结一波
* 当hash对象的所有键值对都小于64byte且hash对象的键值对数量少于512时会采用ziplist编码，否则会采用ht字典编码
* 当采用ziplist编码时，键值对按照先后顺序存储，每个键值对中field和value连续存储
* 64和512都是redis默认的限制，可以通过配置文件中的 hash-max-ziplist-entries 和 hash-max-ziplist-value 对hash对象进行配置

> hash相关api文档可见：[redis文档](http://www.redis.cn/commands.html#hash)

具体api源码在`t_hash.c`中

## set
set的底层编码可以是iniset或hashtable，以sadd命令为例对set的编码规则进行解析  

### sadd

```c
void saddCommand(client *c) {   // sadd
    robj *set;
    int j, added = 0;

    set = lookupKeyWrite(c->db,c->argv[1]); // 从DB中查找对应key
    if (set == NULL) {
        set = setTypeCreate(c->argv[2]);    // 将要添加的第一个元素作为判断依据
        dbAdd(c->db,c->argv[1],set);
    } else {
        if (set->type != OBJ_SET) {
            addReply(c,shared.wrongtypeerr);
            return;
        }
    }

    for (j = 2; j < c->argc; j++) {
        c->argv[j] = tryObjectEncoding(c->argv[j]); // 尝试对值进行压缩编码
        if (setTypeAdd(set,c->argv[j])) added++;
    }
    if (added) {
        signalModifiedKey(c->db,c->argv[1]);     // 事务 数据变更通知
        notifyKeyspaceEvent(NOTIFY_SET,"sadd",c->argv[1],c->db->id);    // 下发变更订阅消息
    }
    server.dirty += added;  // 增加执行命令数量
    addReplyLongLong(c,added);  // 返回结果给client
}
```
其中`setTypeCreate`是根据添加数据的类型，选择创建不同的set对象

```c
robj *setTypeCreate(robj *value) {  // 根据添加的数据类型创建set对象
    if (isObjectRepresentableAsLongLong(value,NULL) == C_OK)    // 如果是整数
        return createIntsetObject();
    return createSetObject();
}

robj *createSetObject(void) { // 创建hashtable的set对象
    dict *d = dictCreate(&setDictType,NULL);    // 创建字典对象 将setDictType作为type函数
    robj *o = createObject(OBJ_SET,d);
    o->encoding = OBJ_ENCODING_HT;
    return o;
}

robj *createIntsetObject(void) {    // 创建intset的set对象
    intset *is = intsetNew();   // 创建intset
    robj *o = createObject(OBJ_SET,is);
    o->encoding = OBJ_ENCODING_INTSET;
    return o;
}
```
`setTypeAdd`和hash对象的`hashTypeSet`类似，都是在插入数据的时候根据数据type或数据数量判断是否需要转换为hashTable

```c
int setTypeAdd(robj *subject, robj *value) {    // 往set对象中添加数据
    long long llval;
    if (subject->encoding == OBJ_ENCODING_HT) { // hashTable 对dict的简单封装
        if (dictAdd(subject->ptr,value,NULL) == DICT_OK) {
            incrRefCount(value);
            return 1;
        }
    } else if (subject->encoding == OBJ_ENCODING_INTSET) {
        if (isObjectRepresentableAsLongLong(value,&llval) == C_OK) {     // 待插入的是数字
            uint8_t success = 0;
            subject->ptr = intsetAdd(subject->ptr,llval,&success);  // 添加数据
            if (success) {
                /* Convert to regular set when the intset contains
                 * too many entries. */
                if (intsetLen(subject->ptr) > server.set_max_intset_entries)    // 个数大于set_max_intset_entries时转换为hash，默认为512
                    setTypeConvert(subject,OBJ_ENCODING_HT);    // 转换为hashTable
                return 1;
            }
        } else {    // 如果插入的不是int，进行转换
            /* Failed to get integer from object, convert to regular set. */
            setTypeConvert(subject,OBJ_ENCODING_HT);

            /* The set *was* an intset and this value is not integer
             * encodable, so dictAdd should always work. */
            serverAssertWithInfo(NULL,value,
                                dictAdd(subject->ptr,value,NULL) == DICT_OK);   // 添加数据
            incrRefCount(value);
            return 1;
        }
    } else {
        serverPanic("Unknown set encoding");
    }
    return 0;
}
```

### 小结一波
* 当set对象的所有元素都是数字且元素数量少于512时会采用intset编码，否则会采用ht字典编码
* 512是redis默认的限制，可以通过配置文件中的 set-max-intset-entries 对set对象自定义进行配置

> set相关api文档可见：[redis文档](http://www.redis.cn/commands.html#set)

具体api源码在`t_set.c`中

## zset
zset的有ziplist和skiplist两种编码方式。其中skiplist的编码方式是用skiplist和hashTable对数据进行存储  
以zadd为例，解析zset的编码规则

### zadd

```c
void zaddGenericCommand(client *c, int flags) { // 通用的添加函数
    // 省略各种初始化和对附加参数的处理
    scores = zmalloc(sizeof(double)*elements);  // 将分值保存在double中
    for (j = 0; j < elements; j++) {
        if (getDoubleFromObjectOrReply(c,c->argv[scoreidx+j*2],&scores[j],NULL)
            != C_OK) goto cleanup;
    }

    zobj = lookupKeyWrite(c->db,key);
    if (zobj == NULL) { // zset不存在新建
        if (xx) goto reply_to_client; /* No key + XX option: nothing to do. */
        if (server.zset_max_ziplist_entries == 0 ||
            server.zset_max_ziplist_value < sdslen(c->argv[scoreidx+1]->ptr))
        {   // 长度超过zset_max_ziplist_value 默认为64 或者zset_max_ziplist_entries被设置为0时创建跳跃表zset
            zobj = createZsetObject();
        } else {    // 创建压缩表zset
            zobj = createZsetZiplistObject();
        }
        dbAdd(c->db,key,zobj);
    } else {
        if (zobj->type != OBJ_ZSET) {
            addReply(c,shared.wrongtypeerr);
            goto cleanup;
        }
    }

    for (j = 0; j < elements; j++) {
        score = scores[j];
        if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {   // 压缩表
            unsigned char *eptr;

            ele = c->argv[scoreidx+1+j*2];
            if ((eptr = zzlFind(zobj->ptr,ele,&curscore)) != NULL) {    // 存在
                if (nx) continue;
                if (incr) {
                    score += curscore;
                    if (isnan(score)) {
                        addReplyError(c,nanerr);
                        goto cleanup;  // 出错直接跳到结尾清理内存空间
                    }
                }

                if (score != curscore) { // 分值变化，直接删了重新插入
                    zobj->ptr = zzlDelete(zobj->ptr,eptr);
                    zobj->ptr = zzlInsert(zobj->ptr,ele,score);
                    server.dirty++;
                    updated++;
                }
                processed++;
            } else if (!xx) {   // 新增
                zobj->ptr = zzlInsert(zobj->ptr,ele,score);
                if (zzlLength(zobj->ptr) > server.zset_max_ziplist_entries) // 如果长度超过zset_max_ziplist_entries 进行转换 默认64字节
                    zsetConvert(zobj,OBJ_ENCODING_SKIPLIST);
                if (sdslen(ele->ptr) > server.zset_max_ziplist_entries)   // 如果字符串长度超过zset_max_ziplist_entries 进行转换 默认128个
                    zsetConvert(zobj,OBJ_ENCODING_SKIPLIST);
                server.dirty++;
                added++;
                processed++;
            }
        } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {   // 跳跃表
            zset *zs = zobj->ptr;
            zskiplistNode *znode;
            dictEntry *de;

            ele = c->argv[scoreidx+1+j*2] = tryObjectEncoding(c->argv[scoreidx+1+j*2]);  // 尝试对值进行压缩编码
            de = dictFind(zs->dict,ele);
            if (de != NULL) {
                if (nx) continue;
                curobj = dictGetKey(de);
                curscore = *(double*)dictGetVal(de);

                if (incr) {
                    score += curscore;
                    if (isnan(score)) {
                        addReplyError(c,nanerr);
                        goto cleanup;	// 出错直接跳到结尾清理内存空间
                    }
                }

                if (score != curscore) {
                    serverAssertWithInfo(c,curobj,zslDelete(zs->zsl,curscore,curobj));
                    znode = zslInsert(zs->zsl,score,curobj);    // 新增一个跳跃表节点
                    incrRefCount(curobj); /* Re-inserted in skiplist. */    // 将原来的跳跃表节点引用计数减一
                    dictGetVal(de) = &znode->score; /* Update score ptr. */ // 将字典指向新的节点
                    server.dirty++;
                    updated++;
                }
                processed++;
            } else if (!xx) {
                znode = zslInsert(zs->zsl,score,ele);   // 添加到跳跃表中
                incrRefCount(ele); // 引用计数加一
                serverAssertWithInfo(c,NULL,dictAdd(zs->dict,ele,&znode->score) == DICT_OK);    // 插入一个元素到hashTable中 具体数据为key 分值为value
                incrRefCount(ele); // 引用计数加一
                server.dirty++;
                added++;
                processed++;
            }
        } else {
            serverPanic("Unknown sorted set encoding");
        }
    }
    // 省略一些数据清理和通知操作
}
```
`createZsetZiplistObject`和其他对象创建函数相同，将一个ziplist指针赋值给object的ptr属性。而`createZsetObject`略有不同，创建了一个`zset`结构体，持有一个skiplist指针和一个hashTable指针 

```c
typedef struct zset {   // zset结构体
    dict *dict;
    zskiplist *zsl;
} zset;

robj *createZsetObject(void) {  // 创建一个skiplist编码的zset
    zset *zs = zmalloc(sizeof(*zs));    // 申请内存
    robj *o;
    zs->dict = dictCreate(&zsetDictType,NULL);  // 创建一个hashTable字典
    zs->zsl = zslCreate();  // 创建一个skiplist跳跃表
    o = createObject(OBJ_ZSET,zs);
    o->encoding = OBJ_ENCODING_SKIPLIST;
    return o;
}
```
skiplist是有序存储的数据结构，可以通过skiplist可以很简单的完成范围操作。但是如果需要获取确定数据的分值，例如`ZSCORE`命令，如果只用skiplist结构存储数据，时间复杂度为`O(logN)`。而这种场景下，hash的时间复杂度为`O(1)`。通过利用两种数据结构存储数据，是的zset在执行两种类型操作的时候效率都不会太低。  
虽然zset用两种数据结构持有数据，但在实际存储的时候只会存储一份数据，hashTable和skiplist共享元素的分值和数据。

### 小结一波
* 当zset对象的所有元素都数据长度小于64byte且元素数量少于128时会采用ziplist编码，否则会采用skiplist编码
* 128和64都是redis默认的限制，可以通过配置文件中的 zset-max-ziplist-entries 和 zset-max-ziplist-value 两个配置项对zset对象自定义进行配置
* 当编码为编码为skiplist时，object中持有的实际数据结构为zset。而zset持有一个skiplist和一个hashTable指针。通过skiplist和hashTable两种数据结构可以同时高效满足zset的范围需求和精确操作需求
* hashTable和skiplist的元素数据为共享的，并不会保存两份数据。hashTable的key为元素数据，value为元素的分值。

> zset相关api文档可见：[redis文档](http://www.redis.cn/commands.html#sorted_set)

具体api源码在`t_zset.c`中

# 对象销毁
*redis* 采用了简单的引用计数，通过`redisObject`结构体中的`refcount`对对象的引用进行计数，当引用计数为0时就将对象销毁。  

 * 当创建一个对象时，引用计数会被初始化为1  
 * 当对象被重复使用时，会对引用计数进行加一
 * 当对象不再被使用时，会对引用计数进行减一
 * 引用计数为0时，对象所占的内存你会被释放

```c
void decrRefCount(robj *o) {    // 引用计数减一
    if (o->refcount <= 0) serverPanic("decrRefCount against refcount <= 0");
    if (o->refcount == 1) { // 当引用计数为1的时候直接释放
        switch(o->type) {
        case OBJ_STRING: freeStringObject(o); break;
        case OBJ_LIST: freeListObject(o); break;
        case OBJ_SET: freeSetObject(o); break;
        case OBJ_ZSET: freeZsetObject(o); break;
        case OBJ_HASH: freeHashObject(o); break;
        default: serverPanic("Unknown object type"); break;
        }
        zfree(o);
    } else {
        o->refcount--;
    }
}
```

*redis* 的引用计数十分简单，没有PHP等语言引用计数的复杂染色机制。主要是因为所有对象都是由 *redis* 自己创建和维护的，不会出现复杂的循环引用场景。

# 共享对象

在 *redis* 中有一种特殊的对象，在server初始化的时候创建很多常用的数据，用于全局共享。这部分数据不会被销毁，主要用于server的各种运行标识和用户数据存储。从而起到节省内存目的，比满大街的破铜烂铁不知道高到哪里去。  
例如在string对象中，在创建一个数字时，会判断是否在`shared.integers`的范围中，如果命中就不进行对象创建，直接使用对应的共享对象，并将引用计数加一

```c
if ((server.maxmemory == 0 ||
        !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS)) &&
        value >= 0 &&
        value < OBJ_SHARED_INTEGERS)
    {   // 使用shared数据，节省内存
        decrRefCount(o);  // 销毁之前创建的字符串对象
        incrRefCount(shared.integers[value]);  // 共享对象引用计数加一
        return shared.integers[value];  // 返回共享对象
    }
```

`shared.integers`的默认范围为0-9999  

```
#define OBJ_SHARED_INTEGERS 10000
for (j = 0; j < OBJ_SHARED_INTEGERS; j++) {
    shared.integers[j] =
        makeObjectShared(createObject(OBJ_STRING,(void*)(long)j));
    shared.integers[j]->encoding = OBJ_ENCODING_INT;
}
```

除了用于共享存储数据的`shared.integers`，还有很多用于redisServer运行的字符串常量  

```c
shared.crlf = createObject(OBJ_STRING,sdsnew("\r\n"));
shared.ok = createObject(OBJ_STRING,sdsnew("+OK\r\n"));
shared.err = createObject(OBJ_STRING,sdsnew("-ERR\r\n"));
// ...
```
可以自行查看`server.c`中的`createSharedObjects`函数。

# LRU
`redisObject`中的lru属性专门用来记录对象的被访问情况，lru保存了最近一次对象被正常访问的时间：  

```c
#define LRU_BITS 24
typedef struct redisObject {    // redis对象
    // ...
    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */
    // ...
} robj;
```
如果 *redis* 设定了`maxmemory`数值，且`maxmemory-policy`设置为`allkeys-lru`或`volatile-lru`时，*redis* 会根据对象中的`lru`属性对DB中的数据进行淘汰。

在每次正常访问数据时，都会更新对应数据的lru时钟  

```c
robj *lookupKey(redisDb *db, robj *key, int flags) {    // 从DB中查找对应key
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {   // 如果存在
        robj *val = dictGetVal(de);

        if (server.rdb_child_pid == -1 &&
            server.aof_child_pid == -1 &&
            !(flags & LOOKUP_NOTOUCH))
        {	// 当有rdb和aof子进程在运行时，不进行lru更新
            val->lru = LRU_CLOCK(); // 更新lru时间
        }
        return val;
    } else {
        return NULL;
    }
}
```
`object`命令比较特殊，这个命令可以查看key对应的对象的状态：引用计数、编码和lru时钟和系统时钟的时差。这个命令在访问数据的时候并不会更新lru时钟，因为其直接对DB进行查找操作，并没有通过`db.c`封装的函数进行访问。

```c
void objectCommand(client *c) { // object操作对应的函数
    robj *o;

    if (!strcasecmp(c->argv[1]->ptr,"refcount") && c->argc == 3) {  // 获取对象的引用计数
        if ((o = objectCommandLookupOrReply(c,c->argv[2],shared.nullbulk))
                == NULL) return;
        addReplyLongLong(c,o->refcount);
    } else if (!strcasecmp(c->argv[1]->ptr,"encoding") && c->argc == 3) {   // 获取对象的编码
        if ((o = objectCommandLookupOrReply(c,c->argv[2],shared.nullbulk))
                == NULL) return;
        addReplyBulkCString(c,strEncoding(o->encoding));
    } else if (!strcasecmp(c->argv[1]->ptr,"idletime") && c->argc == 3) {   // 获取对象lru和系统lru时间的差值
        if ((o = objectCommandLookupOrReply(c,c->argv[2],shared.nullbulk))
                == NULL) return;
        addReplyLongLong(c,estimateObjectIdleTime(o)/1000);
    } else {
        addReplyError(c,"Syntax error. Try OBJECT (refcount|encoding|idletime)");
    }
}

robj *objectCommandLookup(client *c, robj *key) {   // 从DB中查找对应数据对象
    dictEntry *de;
    if ((de = dictFind(c->db->dict,key->ptr)) == NULL) return NULL; // 直接查找db
    return (robj*) dictGetVal(de);
}

robj *objectCommandLookupOrReply(client *c, robj *key, robj *reply) {   // 获取object，如果不存在reply
    robj *o = objectCommandLookup(c,key);
    if (!o) addReply(c, reply);
    return o;
}
```