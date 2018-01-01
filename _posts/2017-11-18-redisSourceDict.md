---
title: redis源码解读(三):基础数据结构之dict
date: 2017-11-18 21:50:30
categories: 
- redis
tags: 
- redis源码解析
- c
---

近来在研读redis3.2.9的源码，虽然网上已有许多redis的源码解读文章，但大都不成系统，且纸上学来终觉浅，遂有该系列博文。部分知识点参照了黄建宏的《Redis设计与实现》。

# 前言
本文探究的数据结构并不是 *redis* 对外暴露的5种数据结构，而是*redis*内部使用的基础数据结构，这些基础的数据结构 *redis* 不仅和 *redisObj* 一起构成了对外暴露的5种数据结构，还被运用于 *redis* 内部的各种存储和逻辑交互，支撑起了 *redis* 的运行。  
*redis* 的基础数据结构主要有以下7种：  

1. [**SDS(simple dynamic string)**：简单动态字符串](/redis/2017/11/14/redisSourceSds#sds)
2. [**ADList(A generic doubly linked list)**：双向链表](/redis/2017/11/16/redisSourceAdlist#adlist)
3. [**dict(Hash Tables)**：字典](#dict)
4. [**intset**：整数结合](/redis/2017/11/19/redisSourceIntset#intset)
5. [**ziplist**：压缩表](/redis/2017/11/24/redisSourceZiplist#ziplist)
6. [**quicklist**：快速列表（双向链表和压缩表二合一的复杂数据结构）](/redis/2017/11/25/redisSourceQuicklist#quicklist)
7. [**skiplist**：跳跃链表](/redis/2017/11/26/redisSourceSkiplist#skiplist)


# dict
`dict`顾名思义就是字典，也就是保存一种键值对的数据结构。每个键都是唯一的，不同的键可以映射到不同的值上。在很多的高级编程语言中都实现了这一数据结构。例如`php`的`array`，`python`的字典。  
`redis`的数据库就是用字典实现的，所有增删改查操作也是构建在`dict`的操作之上。
## 字典的定义
首先是字典的节点：

```c
typedef struct dictEntry {
    void *key;  // 键
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;    // 值
    struct dictEntry *next; // 拉链法解决冲突，下一个节点
} dictEntry;
```
当存在相同hash值的节点时，采用拉链法解决冲突。  
字典节点的`val`为一个联合，可以是指针、`uint64_t`、`int64_t`或者是`double`。

有了节点后需要一个hash表将节点聚合

```c
typedef struct dictht { // hash表
    dictEntry **table;  // 节点数组
    unsigned long size; // hash表大小
    unsigned long sizemask; // hash表掩码，等于size-1，用于计算hash值
    unsigned long used; // 已有节点数量
} dictht;
```
* `table`是一个数组，每一个元素都指向一个字典节点
* `size`保存了当前hash表的大小
* `sizemask`为hash表的掩码，用于计算不同的key的hash值，且等于size-1
* `used`记录了已有节点的数量

有了这两个数据结构以后实际上就能实现一个简单的字典了，但是同`ADList`一样，*redis* 对`dictht`又封装了一层，使的字典的操作更加方便规范。并且字典的`rehash`也是基于这一层的定义实现的，如果不清楚什么是`rehash`不用急，继续看下去你就明白了：

```c
typedef struct dict {   // 字典
    dictType *type; // 各种字典操作方法
    void *privdata; // 私有数据，用于传给操作函数
    dictht ht[2];   // 两个hash表，一个用来存储当前使用的，一个用来rehash
    long rehashidx; // rehash标志位，用于判断是否在rehash和记录rehash进度
    int iterators;  // 迭代器的运行数量
} dict;
```
* `type`保存了各种字典操作方法，其结构体如下：

	```c
	typedef struct dictType {   // 各种字典操作
	    unsigned int (*hashFunction)(const void *key);  // 计算hash值的函数
	    void *(*keyDup)(void *privdata, const void *key);   // 键复制
	    void *(*valDup)(void *privdata, const void *obj);   // 值复制
	    int (*keyCompare)(void *privdata, const void *key1, const void *key2);  // 键比较
	    void (*keyDestructor)(void *privdata, void *key);   // 键销毁
	    void (*valDestructor)(void *privdata, void *obj);   // 值销毁
	} dictType;
	```
* `private`为调用操作方法时需要传入的一些私有数据，大多数情况为NULL
* `ht[2]`这是 *redis* 字典和常见一些高级编程语言中的hash表的实现最不相同的地方，下面会细说
* `rehashidx`rehash的标志位，记录rehash的进度
* `iterators` *redis* 字典也实现了自己的迭代器，用于遍历，该变量用于记录迭代器的运行数量  

## 字典的初始化及新增键值对
### 字典初始化
有了字典的定义之后，我们来梳理一下字典的初始化及插入键值对的流程。首先是初始化：

```c
dict *dictCreate(dictType *type, void *privDataPtr) {   // 创建字典
    dict *d = zmalloc(sizeof(*d));

    _dictInit(d,type,privDataPtr);
    return d;
}

int _dictInit(dict *d, dictType *type, void *privDataPtr) { // 字典初始化
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->iterators = 0;
    return DICT_OK;
}
```
上面两段代码很简单，就是字典的初始化过程，初始化之后我们就有了一个空的`dict`指针。现在需要往这个空的`dict`中插入键值对了。

### 新增键值对
*redis* 提供了`dictAdd`函数用于往字典中新增一条记录：

```c
int dictAdd(dict *d, void *key, void *val)  // 新增一个键值对
{
    dictEntry *entry = dictAddRaw(d,key);   // 增加一个entry

    if (!entry) return DICT_ERR;
    dictSetVal(d, entry, val);  // 设置值
    return DICT_OK;
}

dictEntry *dictAddRaw(dict *d, void *key)   // 新增一个entry
{
    int index;
    dictEntry *entry;
    dictht *ht;

    if (dictIsRehashing(d)) _dictRehashStep(d); // 如果在rehash执行一步rehash

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    if ((index = _dictKeyIndex(d, key)) == -1)  // 获取当前key的hash值，如果存在直接返回NULL
        return NULL;

    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];    // rehash直接插入到rehash的hash中
    entry = zmalloc(sizeof(*entry));    // 分配内存
    entry->next = ht->table[index];
    ht->table[index] = entry;       // 插入到index头部
    ht->used++; // 已用数量自增

    /* Set the hash entry fields. */
    dictSetKey(d, entry, key);  // 设置key
    return entry;
}
```
我们先不care各种`rehash`的判断和操作，`dict`的新增元素主要分为两步，先根据key新增一个`entry`，然后再将具体的值设置到这个`entry`中。上述的代码还是比计较好理解的，无非就是获取`hash`值和内存分配，设置键值。这块的重点在`_dictKeyIndex`函数中：

```c
static int _dictKeyIndex(dict *d, const void *key)  // 获取hash index 如果存在返回-1
{
    unsigned int h, idx, table;
    dictEntry *he;

    /* Expand the hash table if needed */
    if (_dictExpandIfNeeded(d) == DICT_ERR) // 在需要扩展dict
        return -1;
    /* Compute the key hash value */
    h = dictHashKey(d, key);    // 根据hash算法计算key对应的hash值
    for (table = 0; table <= 1; table++) {  // 在两个hash表中查找是否存在相同key
        idx = h & d->ht[table].sizemask;    // 通过&hash表掩码，将对应key散列到hash表中
        /* Search if this slot does not already contain the given key */
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key))
                return -1;
            he = he->next;
        }
        if (!dictIsRehashing(d)) break; // 如果没有在rehash阶段直接break
    }
    return idx;
}
```
在`_dictKeyIndex`函数中主要做了三件事：

1. `_dictExpandIfNeeded`：在需要的时候扩展`dict`
2. `dictHashKey`：计算key对应的hash值
3. 在字典中查找是否存在相同的key

后两者都没什么好说的，主要是在`_dictExpandIfNeeded`函数中：

```c
#define DICT_HT_INITIAL_SIZE     4  // hash初始大小

static int _dictExpandIfNeeded(dict *d) // 在必要时扩展字典
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK; // 如果已经在rehash直接返回

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE); // 如果是第一次新增直接扩展为4

   	// 省略rehash相关...
    return DICT_OK;
}
```
由于字典是第一次新增，因此hash表直接被扩展为初始化大小4，如果我们再插入一组键值对，字典的结构就如下图所示：

![sdshdr8](/images/dict.png)  

以上就是字典初始化后新增的函数调用流程，这里不对具体的hash算法做解读，感兴趣的同学可以自己去了解。

## rehash
随着操作的不断执行，hash表保存的键值对会逐渐的增多或者减少，这时就会暴露一些问题。如果hash表很大，但是键值对太少，也就是hash表的负载(`dictht->used`/`dictht->size`)太小，就会有大量的内存浪费；如果hash表的负载太大，就会影响字典的查找效率。这时候就需要进行rehash将hash表的负载控制在一个合理的范围。  

### 什么时候发生rehash
之前的`_dictExpandIfNeeded`的源码中省去了rehash相关部分：

```c
static int _dictExpandIfNeeded(dict *d) // 在必要时扩展字典
{
    if (dictIsRehashing(d)) return DICT_OK; // 如果已经在rehash直接返回
	
	// 省略字典初始化相关
	
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {   // 当userd大于size并且  字典处于可以rehash或者负载5时进行扩展
        return dictExpand(d, d->ht[0].used*2);  // 扩展为used*2
    }
    return DICT_OK;
}
```
`dict_can_resize`是 *redis* 的一个常量，当有**BGSAVE**或者**BGREWRITEAOF**命令在执行时，会将该常量置为0：

```c
void updateDictResizePolicy(void) { // 更新字典rehash控制符 如果有aof或者rdb的子进程 禁止rehash
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
        dictEnableResize();
    else
        dictDisableResize();
}
```
大部分操作系统都会使用写时复制来优化子进程的执行效率，而上述的两个命令都需要创建当前服务的子进程来执行。因此在有子进程的情况下，*redis* 将负载值提高到了5尽可能避免在有子进程时进行rehash，最大限度节省内存。  
另外，当负载小于0.1时会对字典size进行收缩，收缩操作并不是在dict的底层函数定义调用的，而是由具体应用时决定、对应的`htNeedsResize`函数也是在`server.c`中

```c
#define HASHTABLE_MIN_FILL        10      /* Minimal hash table fill 10% */

int htNeedsResize(dict *dict) {
    long long size, used;

    size = dictSlots(dict);
    used = dictSize(dict);
    return (size > DICT_HT_INITIAL_SIZE && (used*100/size < HASHTABLE_MIN_FILL));
}
```

### 渐进式rehash
在之前的`dict`定义中，可以看到 *redis* 的`dict`结构中有两个`dictht`。index为0的`dictht`是用来正常存放数据的，而index为1的`dictht`只有在rehash时才会用到。  
在`_dictExpandIfNeeded`函数中，当满足扩展条件时会调用`dictExpand`，这个函数就是用来变更hashTable的size：

```c
int dictExpand(dict *d, unsigned long size) // 变更hashTable size
{
    dictht n; /* the new hash table */
    unsigned long realsize = _dictNextPower(size);  // 获取size值

    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    /* Rehashing to the same table size is not useful. */
    if (realsize == d->ht[0].size) return DICT_ERR;

    // 为新的hashTable开辟空间
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys. */
    if (d->ht[0].table == NULL) {   // 初始化
        d->ht[0] = n;
        return DICT_OK;
    }

    /* Prepare a second hash table for incremental rehashing */
    d->ht[1] = n;   // 渐进式rehash 将新的hashTable赋值给ht[1]
    d->rehashidx = 0;   // 标志开始rehash
    return DICT_OK;
}
```
其中`_dictNextPower`是用来获取rehash后的hashTable的`size`大小，size的值是2的整数倍。

```c
static unsigned long _dictNextPower(unsigned long size)
{
    unsigned long i = DICT_HT_INITIAL_SIZE;

    if (size >= LONG_MAX) return LONG_MAX;
    while(1) {  // 每次乘2
        if (i >= size)
            return i;
        i *= 2;
    }
}
```

在`ht[1]`初始化后，就开始rehash了。*redis* 采用了渐进式的rehash，并不是一次性把所有的rehash操作执行完，而是一点一点把老的hashTable中的数据迁移到新的hashTable中。这和很多持有hash表的高级语言不太一样。*redis* 是一个内存数据库，需要实时提供高可用、高效率的读写服务，因此如果一次性把所有rehash操作执行完会导致短时间大量的内存操作，高可用就无法保障了。  
rehash主要有两种方式：

1. `dictRehashMilliseconds`：按照ms计时的rehash操作，是`databasesCron`中针对redis的DB进行rehash。`databasesCron`是redis的时间事件之一，每隔一段时间就会执行。

	```c
	int dictRehashMilliseconds(dict *d, int ms) {   // 时间限制的rehash
	    long long start = timeInMilliseconds();
	    int rehashes = 0;
	
	    while(dictRehash(d,100)) {	// 每次执行100个hash值
	        rehashes += 100;
	        if (timeInMilliseconds()-start > ms) break;
	    }
	    return rehashes;
	}
	```
2. `_dictRehashStep`：一步的rehash操作，该函数在`dict`的增删改查操作中都会被调用。
	
	```c
	static void _dictRehashStep(dict *d) {  // 一步的rehash
	    if (d->iterators == 0) dictRehash(d,1);
	}
	```
	例如之前列出的`dictAddRaw`代码中有一行
	
	```c
	dictEntry *dictAddRaw(dict *d, void *key)   // 新增一个entry
	{
		// 省略...
	    if (dictIsRehashing(d)) _dictRehashStep(d); // 如果在rehash执行一步rehash
	    // 省略...
		ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];    // rehash直接插入到rehash的hash中
	    // 省略...
	}
	```
	通过这样一次次的调用单步rehash，对应的`dict`的`ht[0]`上的数据会被一步一步迁移到`ht[1]`上。  

需要注意，由于rehash是渐进式的，无法避免并发的新增操作。为了保证rehash的正常进行，在rehash期间的新增操作都会被添加到`ht[1]`中。  
在`_dictRehashStep`和`dictRehashMilliseconds`函数中，都调用了同一个函数`dictRehash`，这是 *redis* 的通用rehash函数，来瞅瞅具体的rehash代码吧：

```c
int dictRehash(dict *d, int n) {    // 通用rehash函数
    int empty_visits = n*10; /* Max number of empty buckets to visit. */    // 最多读n*10个空，防止rehash长时间阻塞
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);    // 确保rehash index没有溢出
        while(d->ht[0].table[d->rehashidx] == NULL) {   // 查找非空的节点
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) { // 一次把一整个hash table节点全部rehash 顺序倒置
            unsigned int h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;    // 重新计算hash值
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */   // 判断是否完全rehash
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];    // 将rehash的hash赋值给0，1只是用来rehash的
        _dictReset(&d->ht[1]);	// 将ht[1]置空
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```
由于hash表是一个离散的数据结构，因此可能会有大量的空节点，可以想象一个巨大的hash表中只有少量几个元素，*redis* 要进行rehash，哪怕只是一步rehash也会耗费很长的时间。因此在rehash时，*redis* 最多只会遍历 *10n* 个空节点，n为需要执行的rehash步数，如果遍历了 *10n* 个节点还没找到非空节点会直接返回。  
`dict`结构体中持有两个hashTable，`ht[1]`只有在rehash是才会使用。当rehash进行完时，会将对应的hashTable赋值`ht[0]`,而`ht[1]`会被置空。

## 总结一波

1. *redis* 的字典是由两个hash表组成，第一个hash表是正常保存数据，第二个hash表仅用来rehash
2. hash表使用拉链法来解决冲突
3. 当没有子进程在运行时，hash表的负载大于1时就会进行rehash扩展hash表大小；有子进程时负载的阈值提高到了5
4. 当hash表的负载小于0.1时，hash表会进行rehash收缩hash表大小
5. *redis* 的rehash是渐进式的，每次增删改查操作都会执行一步rehash

*dictIterator* 和其他一些API的源码就不做过多的解析了，字典相关的大部分源码在`dict.h`和`dict.c`中，想继续深入了解的可以自行查看。
