---
title: redis源码解读(一):redis基础数据结构
date: 2017-11-14 21:48:30
categories: 
- redis
tags: 
- redis
- 源码
---

近来在研读redis3.2.9的源码，虽然网上已有许多redis的源码解读文章，但大都不成系统，且纸上学来终觉浅，遂有该系列博文。部分知识点参照了黄建宏老师的《Redis设计与实现》。

# 前言
本文探究的数据结构并不是 *redis* 对外暴露的5种数据结构，而是*redis*内部使用的基础数据结构，这些基础的数据结构 *redis* 不仅和 *redisObj* 一起构成了对外暴露的5种数据结构，还被运用于 *redis* 内部的各种存储和逻辑交互，支撑起了 *redis* 的运行。  
*redis* 的基础数据结构主要有以下7种：  

1. [**SDS(simple dynamic string)**：简单动态字符串](#sds)
2. [**ADList(A generic doubly linked list)**：双向链表](#adlist)
3. [**dict(Hash Tables)**：字典](#dict)
4. [**intset**：整数结合](#intset)
5. [**quicklist**：快速列表，双向链表和压缩表二合一的复杂数据结构](#quicklist)
6. [**ziplist**：跳跃链表](#ziplist)
7. **zipmap**：跳跃图（*redis* 3.2.9中并没有被使用到）

***
# SDS
*redis* 没有直接使用c语言的字符串，而是自己定义了一个字符串数据结构：**SDS(simple dynamic string)**作为默认的字符串。我们设置的所有键值基本都是**SDS**。   
那么为何 *redis* 要自己定义一个字符串数据结构呢？我们看看**SDS**的定义以及他和C语言字符串的区别就清楚了。

## SDS的定义
### 3.2和3.0的差别
**sds**在 *redis* 3.2版本做了较大的变动，这里就简单顺便介绍一下两个版本的差别。在3.2版本以前**SDS**只有一种数据结构，到了3.2版本以后**SDS**根据存储的内容会选择不同的数据结构，以到达节省内存的效果！

```c
// 3.0及以前
struct sdshdr {
	// 记录buf数组中已使用字节数量
    unsigned int len;
    // 记录buf数组中未使用的字节数量
    unsigned int free;
    // 字节数组，存储字符串
    char buf[];
};

// >=3.2
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
可以看到，在3.2以后的版本，*redis* 的**SDS**分为了5种数据结构，分别应对不同长度的字符串需求，具体的类型选择如下。

```c
static inline char sdsReqType(size_t string_size) { // 获取类型
    if (string_size < 1<<5)     // 32
        return SDS_TYPE_5;
    if (string_size < 1<<8)     // 256
        return SDS_TYPE_8;
    if (string_size < 1<<16)    // 65536 64k
        return SDS_TYPE_16;
    if (string_size < 1ll<<32)  // 4294967296 4GB
        return SDS_TYPE_32;
    return SDS_TYPE_64;
}
```  
`__attribute__ ((__packed__))`是什么呢？首先就要了解编译器内存对齐的优化策略：
> struct的分配的内存是内部最大元素的整数倍  

如果内存不对齐，cpu的的内存访问速度会大大下降，至于内存对齐的原理这里就不做赘述了，有兴趣可以自行google或百度。  
而`__attribute__ ((__packed__))`这个声明就是用来告诉编译器取消内存对齐优化，按照实际的占用字节数进行对齐，效果如下：

```c
printf("%ld\n", sizeof(struct sdshdr8));  // 3
printf("%ld\n", sizeof(struct sdshdr16)); // 5
printf("%ld\n", sizeof(struct sdshdr32)); // 9
printf("%ld\n", sizeof(struct sdshdr64)); // 17
```
通过加上`__attribute__ ((__packed__))`声明，`sdshdr16`节省了1个字节，`sdshdr32`节省了3个字节，`sdshdr64`节省了7个字节。 
但是内存对齐怎么办呢，不能为了一点内存大大拖慢cpu的寻址效率啊？*redis* 通过自己在malloc等c语言内存分配函数上封装了一层**zmalloc**，将内存分配收敛，并解决了内存对齐的问题。在内存分配前有这么一段代码：

```c
if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \    // 确保内存对齐！
```
这段代码写的比较抽象，简而言之就是先判断当前要分配的`_n`个内存是否是`long`类型的整数倍，如果不是就在`_n`的基础上加上内存大小差值，从而达到了内存对齐的保证。

### SDS数据结构解读
扯了一大堆有点偏题了，回到正题，以 *redis* 3.2.9的`sdshdr8`为例来解读一下这个数据结构：

![sdshdr8](/images/sdshdr8.png)  

* `len`记录当前字节数组的长度（不包括`\0`），使得获取字符串长度的时间复杂度由*O(N)*变为了*O(1)*
* `alloc`记录了当前字节数组总共分配的内存大小（不包括`\0`）
* `flags`记录了当前字节数组的属性、用来标识到底是`sdshdr8`还是`sdshdr16`等
* `buf`保存了字符串真正的值以及末尾的一个`\0`

整个**SDS**的内存是连续的，统一开辟的。为何要统一开辟呢？因为在大多数操作中，`buf`内的字符串实体才是操作对象。如果统一开辟内存就能通过`buf`头指针进行寻址，拿到整个`struct`的指针，而且通过`buf`的头指针减一直接就能获取`flags`的值，骚操作代码如下！

```c
// flags值的定义
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4

// 通过buf获取头指针
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))

// 通过buf的-1下标拿到flags值
unsigned char flags = s[-1];
```
至于`buf`末尾的`\0`是为了复用`string.h`中部分字符串操作函数。

## 创建SDS
这块比较直白易懂，直接show源码：

```c
sds sdsnewlen(const void *init, size_t initlen) {   // 创建sds
    void *sh;
    sds s;  // 指向字符串头指针
    char type = sdsReqType(initlen);
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;  // 如果是空字符串直接使用SDS_TYPE_8，方便后续拼接
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */

    sh = s_malloc(hdrlen+initlen+1);    // 分配空间大小为 sdshdr大小+字符串长度+1
    if (!init)
        memset(sh, 0, hdrlen+initlen+1);    // 初始化内存空间
    if (sh == NULL) return NULL;
    s = (char*)sh+hdrlen;
    fp = ((unsigned char*)s)-1; // 获取flags指针
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);    // sdshdr5的前5位保存长度，后3位保存type
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);       // 获取sdshdr指针
            sh->len = initlen;      // 设置len
            sh->alloc = initlen;    // 设置alloc
            *fp = type; // 设置type
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    if (initlen && init)
        memcpy(s, init, initlen);   // 内存拷贝字字符数组赋值
    s[initlen] = '\0';  // 字符数组最后一位设为\0
    return s;
}
```
由于`sdshdr5`的只用来存储长度为32字节以下的字符数组，因此flags的5个bit就能满足长度记录，加上type所需的3bit，刚好为8bit一个字节，因此`sdshdr5`不需要单独的`len`记录长度，并且只有32个字节的存储空间，动态的变更内存余地较小，所以 *redis* 直接不存储`alloc`，当`sdshdr5`需要扩展时会直接变更成更大的**SDS**数据结构。  
除此之外，**SDS**都会多分配1个字节用来保存`'\0'`。

## SDS拼接
**SDS**的拼接函数`sdscatlen`和C语言的`strcat`有着很大的出入，直接上代码：

```c
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);  // 获取当前字符串长度

    s = sdsMakeRoomFor(s,len);  // 重点!
    if (s == NULL) return NULL;
    memcpy(s+curlen, t, len);   // 内存拷贝
    sdssetlen(s, curlen+len);   // 设置sds->len
    s[curlen+len] = '\0';       // 在buf的末尾追加一个\0
    return s;
}
```
可以看到除了调用了一下`sdsMakeRoomFor`以外，就是正常的内存拷贝和设置使用长度等等，那么就来看看`sdsMakeRoomFor`好了：

```c
sds sdsMakeRoomFor(sds s, size_t addlen) {  // 确保sds字符串在拼接时有足够的空间
    void *sh, *newsh;
    size_t avail = sdsavail(s); // 获取可用长度
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s;

    len = sdslen(s);    // 获取字符串长度 O(1)
    sh = (char*)s-sdsHdrSize(oldtype);  // 获取当前sds指针
    newlen = (len+addlen);  // 分配策略 SDS_MAX_PREALLOC=1024*1024=1M
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;    // 如果拼接后的字符串小于1M，就分配两倍的内存
    else
        newlen += SDS_MAX_PREALLOC; // 如果拼接后的字符串大于1M，就分配多分配1M的内存

    type = sdsReqType(newlen);  // 获取新字符串的sds类型

    if (type == SDS_TYPE_5) type = SDS_TYPE_8;  // 如果type为SDS_TYPE_5直接优化成SDS_TYPE_8

    hdrlen = sdsHdrSize(type);
    if (oldtype==type) {
        newsh = s_realloc(sh, hdrlen+newlen+1); // 类型没变在原有基础上realloc
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
        newsh = s_malloc(hdrlen+newlen+1);  // 类型发生变化需要重新malloc
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);  // 将老字符串拷贝到新的内存快中
        s_free(sh); // 释放老的sds内存
        s = (char*)newsh+hdrlen;
        s[-1] = type;   // 设置sds->flags
        sdssetlen(s, len);  // 设置sds->len
    }
    sdssetalloc(s, newlen); // 设置sds->alloc
    return s;
}
```
该方法是用来确保**SDS**字符串在拼接时有足够的空间，如果空间不够就会重新分配内存，但是分配内存并不是只分配当下要用的内存，而是采用了冗余的预分配内存，通过源码可以看到内存的预分配策略主要有两种：

1. 拼接后的字符串长度不超过1M，分配两倍的内存
2. 拼接够的字符串长度超过1M，多分配1M的内存   

通过这两种策略，在字符串拼接时会预分配一部分内存，下次拼接的时候就可能不再需要进行内存分配了，将原本N次字符串拼接需要N次内存重新分配的次数优化到最多需要N次，是典型的空间换时间的做法。  
当然，如果新的字符串长度超过了原有字符串类型的限定那么还会涉及到一个重新生成`sdshdr`的过程。  

还有一个细节需要注意，由于`sdshrd5`并不存储`alloc`值，因此无法获取`sdshrd5`的可用大小，如果继续采用`sdshrd5`进行存储，在之后的拼接过程中每次都还是要进行内存重分配。因此在发生拼接行为时，`sdshrd5`会被直接优化成`sdshrd8`。

## SDS惰性空间释放
在`SDS`的字符串缩短操作中，多余出来的空间并不会直接释放，而是保留这部分空间，待以后再用。以`sdstrim`为例：

```c
sds sdstrim(sds s, const char *cset) {  // sds trim操作
    char *start, *end, *sp, *ep;
    size_t len;

    sp = start = s;
    ep = end = s+sdslen(s)-1;
    while(sp <= end && strchr(cset, *sp)) sp++; // 从头遍历
    while(ep > sp && strchr(cset, *ep)) ep--;   // 从尾部遍历
    len = (sp > ep) ? 0 : ((ep-sp)+1);
    if (s != sp) memmove(s, sp, len);   // 内存拷贝
    s[len] = '\0';
    sdssetlen(s,len);   // 重新设置sds->len
    return s;
}
```
将`A. :HelloWroldA.:`进行`sdstrim(s,"Aa. :");`后，如下图所示：
![sdstrim](/images/sdstrim.png)
可以看到内存空间并没有被释放，甚至空闲的空间都没有被置空。由于**SDS**是通过`len`值标识字符串长度，因此**SDS**完全不需要受限于c语言字符串的那一套`\0`结尾的字符串形式。在后续需要拼接扩展时，这部分空间也能够再次被利用起来，降低了内存重新分配的概率。

当然，**SDS**也提供了真正的释放空间的方法，以供真正需要释放空闲内存时使用。

## 总结一波
1. *redis* 3.2之后，针对不同长度的字符串引入了不同的**SDS**数据结构，并且强制内存对齐1，将内存对齐交给统一的内存分配函数，从而达到节省内存的目的
2. **SDS**的字符串长度通过`sds->len`来控制，不受限于C语言字符串`\0`，可以存储二进制数据，并且将获取字符串长度的时间复杂度降到了*O(1)*
3. **SDS**的头和`buf`字节数组的内存是连续的，可以通过寻址方式获取**SDS**的指针以及`flags`值
4. **SDS**的拼接扩展有一个内存预分配策略，用空间减少每次拼接的内存重分配可能性
5. **SDS**的缩短并不会真正释放掉对应空闲空间
6. **SDS**分配内存都会多分配1个字节用来在`buf`的末尾追加一个`\0`，在部分场景下可以和C语言字符串保证同样的行为甚至复用部分`string.h`的函数

其他的一些细节或者操作函数这里就不做详解了，感兴趣的同学可以刷一波源码，**SDS**相关的源码在`sds.h`和`sds.c`文件中。

***
# ADList
**ADList(A generic doubly linked list)**是 *redis* 自定义的一种双向链表，广泛运用于 *redisClients* 、 *redisServer* 、发布订阅、慢查询、监视器等（注：3.0及以前还会被运用于`list`结构中，在3.2以后被`quicklist`取代）。  
链表是最常用的数据结构之一了，对于其算法和ADT就不做赘述，不了解的同学可以google、百度或者参考《数据结构：C语言版》等书籍。

## 链表和链表节点定义
每个链表节点都是用一个`listNode`表示

```c
typedef struct listNode {   // 双向节点
    struct listNode *prev;
    struct listNode *next;
    void *value;    // 空指针，可以被指向任何类型
} listNode;
```
在简单的链表定义中，只用节点结构就已经能够满足链表的需求了，但是 *redis* 通过`list`结构体持有链表，使得链表操作更加方便、规范。

```c
typedef struct list {
    listNode *head;		// 头指针
    listNode *tail;		// 尾指针
    void *(*dup)(void *ptr);    // 复制函数
    void (*free)(void *ptr);    // 节点释放函数
    int (*match)(void *ptr, void *key); // 对比函数函数
    unsigned long len;  // list长度
} list;
```
* `dup`为节点复制函数
* `free`为节点释放函数
* `match`为节点比较函数

通过这样的定义，`adlist`有了以下优点：

1. 双向：可以灵活的访问前置或者后置节点
2. `list`头指针和尾指针：可以方便的获取头尾节点或者从头尾遍历查找
3. `len`：使获取`list`由*O(N)*变为*O(1)*
4. 通过`void`实现多态：不同的实例化链表对象可以持有不同的值，其对应的3个操作函数也可以自定义，是不是有点`interface`的感觉！

## 链表迭代器
除了双向链表的定义外，*redis* 还定义了一个迭代器，用于遍历链表：

```c
typedef struct listIter {   // 列表迭代器
    listNode *next;
    int direction;  // 迭代器遍历方向
} listIter;
```
其中`direction`用于标识迭代器的遍历方向：

```c
#define AL_START_HEAD 0     // 从头遍历
#define AL_START_TAIL 1     // 从尾遍历
```
通过定义`listIter`，*redis* 在需要遍历`list`时，不需要再复制各种tmp值，只需要调用`listIter`的遍历函数。
以`listSearchKey`为例：

```c
listNode *listSearchKey(list *list, void *key)  // list查找key
{
    listIter iter;
    listNode *node;

    listRewind(list, &iter);    // 初始化迭代器
    while((node = listNext(&iter)) != NULL) {   // 迭代器遍历
        if (list->match) {  // 如果定义了match函数
            if (list->match(node->value, key)) {
                return node;
            }
        } else {    // 直接进行值比较
            if (key == node->value) {
                return node;
            }
        }
    }
    return NULL;
}
```
所有和遍历有关的行为都收敛到了`listIter`中，`list`就专注负责存储。  

`ADList`相关的定义都在`adlist.h`和`adlist.c`文件中，感兴趣的同学可以自行查看

***
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
    int iterators;  // 迭代器的迭代进度
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
* `iterators` *redis* 字典也实现了自己的迭代器，用于遍历，该变量用于记录迭代器的迭代进度  

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

### rehash
随着操作的不断执行，hash表保存的键值对会逐渐的增多或者减少，这时就会暴露一些问题。如果hash表很大，但是键值对太少，也就是hash表的负载因子(`dictht->used`/`dictht->size`)太小，就会有大量的内存浪费；如果hash表的负载因子太大，就会影响字典的查找效率。这时候就需要进行rehash将hash表的负载因子控制在一个合理的范围。



# intset




# quicklist





# ziplist
