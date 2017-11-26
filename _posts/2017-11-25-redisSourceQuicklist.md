---
title: redis源码解读(六):基础数据结构之quicklist
date: 2017-11-25 23:11:30
categories: 
- redis
tags: 
- redis源码解析
---

近来在研读redis3.2.9的源码，虽然网上已有许多redis的源码解读文章，但大都不成系统，且纸上学来终觉浅，遂有该系列博文。部分知识点参照了黄建宏老师的《Redis设计与实现》。

# 前言
本文探究的数据结构并不是 *redis* 对外暴露的5种数据结构，而是*redis*内部使用的基础数据结构，这些基础的数据结构 *redis* 不仅和 *redisObj* 一起构成了对外暴露的5种数据结构，还被运用于 *redis* 内部的各种存储和逻辑交互，支撑起了 *redis* 的运行。  
*redis* 的基础数据结构主要有以下7种：  

1. [**SDS(simple dynamic string)**：简单动态字符串](/redis/2017/11/14/redisSourceSds#sds)
2. [**ADList(A generic doubly linked list)**：双向链表](/redis/2017/11/16/redisSourceAdlist#adlist)
3. [**dict(Hash Tables)**：字典](/redis/2017/11/18/redisSourceDict#dict)
4. [**intset**：整数结合](/redis/2017/11/19/redisSourceIntset#intset)
5. [**ziplist**：压缩表](/redis/2017/11/24/redisSourceZiplist#ziplist)
6. [**quicklist**：快速列表（双向链表和压缩表二合一的复杂数据结构）](#quicklist)
7. [**skiplist**：跳跃链表](/redis/2017/11/26/redisSourceSkiplist#skiplist)


# quicklist

`quicklist`是一个3.2版本之后新增的基础数据结构，是 *redis* 自定义的一种复杂数据结构，将`ziplist`和`adlist`结合到了一个数据结构中。主要是作为`list`的基础数据结构。  
在3.2之前，`list`是根据元素数量的多少采用`ziplist`或者`adlist`作为基础数据结构，3.2之后统一改用`quicklist`，从数据结构的角度来说`quicklist`结合了两种数据结构的优缺点，复杂但是实用：

* 链表在插入，删除节点的时间复杂度很低；但是内存利用率低，且由于内存不连续容易产生内存碎片
* 压缩表内存连续，存储效率高；但是插入和删除的成本太高，需要频繁的进行数据搬移、释放或申请内存

而`quicklist`通过将每个压缩表用双向链表的方式连接起来，来寻求一种收益最大化。

## 定义
首先是`quicklist`的节点`quicklistNode`：

```c
typedef struct quicklistNode {
    struct quicklistNode *prev; // 前一个节点
    struct quicklistNode *next; // 后一个节点
    unsigned char *zl;  // ziplist
    unsigned int sz;             // ziplist的内存大小
    unsigned int count : 16;     // zpilist中数据项的个数
    unsigned int encoding : 2;   // 1为ziplist 2是LZF压缩存储方式
    unsigned int container : 2;  
    unsigned int recompress : 1;   // 压缩标志, 为1 是压缩
    unsigned int attempted_compress : 1; // 节点是否能够被压缩,只用在测试
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```
`quicklistNode`实际上就是对`ziplist`的进一步封装，其中包括：

* 指向前后压缩表节点的两个指针
* `zl`：`ziplist`指针
* `sz`：`ziplist`的内存占用大小
* `count`：`ziplist`内部数据的个数
* `encoding`：`ziplist`编码方式，1为默认方式，2为LZF数据压缩方式
* `recompress`：是否压缩，1表示压缩  
这里从变量count开始，都采用了位域的方式进行数据的内存声明，使得6个`unsigned int`变量只用到了一个`unsigned int`的内存大小。
> C语言支持位域的方式对结构体中的数据进行声明，也就是可以指定一个类型占用几位：  
1) 如果相邻位域字段的类型相同，且其位宽之和小于类型的sizeof大小，则后面的字段将紧邻前一个字段存储，直到不能容纳为止；  
2) 如果相邻位域字段的类型相同，但其位宽之和大于类型的sizeof大小，则后面的字段将从新的存储单元开始，其偏移量为其类型大小的整数倍；  
3) 如果相邻的位域字段的类型不同，则各编译器的具体实现有差异，VC6采取不压缩方式，Dev-C++采取压缩方式；  
4) 如果位域字段之间穿插着非位域字段，则不进行压缩；  
5) 整个结构体的总大小为最宽基本类型成员大小的整数倍。  

`sizeof(quicklistNode); // output:32`，通过位域的声明方式，`quicklistNode`可以节省24个字节。

通过`quicklist`将`quicklistNode`连接起来就是一个完整的数据结构了

```c
typedef struct quicklist {
    quicklistNode *head;    // 头结点
    quicklistNode *tail;    // 尾节点
    unsigned long count;    // 所有数据的数量
    unsigned int len;       // quicklist节点数量
    int fill : 16;          // 单个ziplist的大小限制
    unsigned int compress : 16;   // 压缩深度
} quicklist;
```
由于`quicklist`结构包含了压缩表和链表，那么每个`quicklistNode`的大小就是一个需要仔细考量的点。如果单个`quicklistNode`存储的数据太多，就会影响插入效率；但是如果单个`quicklistNode`太小，就会变得跟链表一样造成空间浪费。  
`quicklist`通过`fill`对单个`quicklistNode`的大小进行限制：`fill`可以被赋值为正整数或负整数，当`fill`为负数时：

* -1：单个节点最多存储4kb
* -2：单个节点最多存储8kb
* -3：单个节点最多存储16kb
* -4：单个节点最多存储32kb
* -5：单个节点最多存储64kb
* 为正数时，表示单个节点最大允许的元素个数，最大为32768个

```c
#define FILL_MAX (1 << 15)  // 32768
void quicklistSetFill(quicklist *quicklist, int fill) { // set ziplist的单个节点最大存储数据量
    if (fill > FILL_MAX) {  // 个数
        fill = FILL_MAX;
    } else if (fill < -5) { // 内存大小
        fill = -5;
    }
    quicklist->fill = fill;
}
```
在 *redis* 内部使用中，默认的最大单节点数据量设置是-2，也就是8kb

## push
`quicklist`只能在头尾插入节点，以在头部插入节点为例：

```c
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {   // 在头部插入数据
    quicklistNode *orig_head = quicklist->head;

    if (likely(_quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {  // 判断是否能够被插入到头节点中
        quicklist->head->zl = ziplistPush(quicklist->head->zl, value, sz, ZIPLIST_HEAD);  // 调用ziplist的api在头部插入数据
        quicklistNodeUpdateSz(quicklist->head); // 更新节点的sz
    } else {    // 需要新增节点
        quicklistNode *node = quicklistCreateNode();    // 新建节点
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);  // 新建一个ziplist并插入一个节点

        quicklistNodeUpdateSz(node);    // 更新节点的sz
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);   // 将新节点插入到头节点之前
    }
    quicklist->count++; // count自增
    quicklist->head->count++;
    return (orig_head != quicklist->head);  // 返回0为用已有节点 返回1为新建节点
}
```
`quicklist`的主要操作基本都是复用`ziplist`的api，其中`likely`是针对条件语句的优化，告知编译器这种情况很可能出现，让编译器针对这种条件进行优化；与之对应的还有`unlikely`。由于绝大部分时候都不需要新增节点，因此用`likely`做了优化   
在`_quicklistNodeAllowInsert`函数中，针对单个节点的内存大小做了校验

```c
REDIS_STATIC int _quicklistNodeAllowInsert(const quicklistNode *node,
                                           const int fill, const size_t sz) {   // 判断当前node是否还能插入数据
    if (unlikely(!node))
        return 0;

    int ziplist_overhead;
    /* size of previous offset */
    if (sz < 254)   // 小于254时后一个节点的pre只有1字节,否则为5字节
        ziplist_overhead = 1;
    else
        ziplist_overhead = 5;

    /* size of forward offset */
    if (sz < 64)    // 小于64字节当前节点的encoding为1
        ziplist_overhead += 1;
    else if (likely(sz < 16384))    // 小于16384 encoding为2字节
        ziplist_overhead += 2;
    else    // encoding为5字节
        ziplist_overhead += 5;

    /* new_sz overestimates if 'sz' encodes to an integer type */
    unsigned int new_sz = node->sz + sz + ziplist_overhead; // 忽略了连锁更新的情况
    if (likely(_quicklistNodeSizeMeetsOptimizationRequirement(new_sz, fill)))   // // 校验fill为负数是否超过单存储限制
        return 1;
    else if (!sizeMeetsSafetyLimit(new_sz)) // 校验单个节点是否超过8kb，主要防止fill为正数时单个节点内存过大
        return 0;
    else if ((int)node->count < fill)   // fill为正数是否超过存储限制
        return 1;
    else
        return 0;
}
```
同样，因为默认的fill为-2，所以针对为负数并且不会超过单个节点存储限制的条件做了`likely`优化；除此之外在计算的时候还忽略了`ziplist`可能发生的连锁更新；以及fill为正数时单个节点不能超过8kb

## 节点压缩
由于`list`这个结构大部分时候只会用到头尾的数据，因此 *redis* 利用lzf算法对节点中间的元素进行压缩，以达到节省内存空间的效果。压缩节点的结构体和具体函数如下：

```c
typedef struct quicklistLZF {  // lzf结构体
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;

REDIS_STATIC int __quicklistCompressNode(quicklistNode *node) { // 压缩节点
#ifdef REDIS_TEST
    node->attempted_compress = 1;
#endif

    /* Don't bother compressing small values */
    if (node->sz < MIN_COMPRESS_BYTES)  // 小于48字节不进行压缩
        return 0;

    quicklistLZF *lzf = zmalloc(sizeof(*lzf) + node->sz);

    /* Cancel if compression fails or doesn't compress small enough */
    if (((lzf->sz = lzf_compress(node->zl, node->sz, lzf->compressed,
                                 node->sz)) == 0) ||
        lzf->sz + MIN_COMPRESS_IMPROVE >= node->sz) {   // 如果压缩失败或压缩后节省的空间不到8字节放弃压缩
        /* lzf_compress aborts/rejects compression if value not compressable. */
        zfree(lzf);
        return 0;
    }
    lzf = zrealloc(lzf, sizeof(*lzf) + lzf->sz);    // 重新分配内存
    zfree(node->zl);    // 释放原有节点
    node->zl = (unsigned char *)lzf;    // 将压缩节点赋值给node
    node->encoding = QUICKLIST_NODE_ENCODING_LZF;   // 记录编码
    node->recompress = 0;
    return 1;
}
```
`quicklist`结构体中还有一个`compress`变量是用来控制压缩的深度，例如`compress`为1表示出了头尾节点其他全部都进行lzf压缩。  
具体的lzf压缩算法这里就不做探究了，感兴趣可以自行google或百度  


## 总结一波
`quicklist`除了常用的增删改查外还提供了merge、将`ziplist`转换为`quicklist`等api，这里就不详解了，可以具体查看`quicklist.h`和`quicklist.c`两个文件。

1. `quicklist`是 *redis* 在`ziplist`和`adlist`两种数据结构的基础上融合而成的一个实用的复杂数据结构
2. `quicklist`在3.2之后取代`adlist`和`ziplist`作为`list`的基础数据类型
3. `quicklist`的大部分api都是直接复用`ziplist`
4. `quicklist`的单个节点最大存储默认为8kb
5. `quicklist`提供了基于lzf算法的压缩api，通过将不常用的中间节点数据压缩达到节省内存的目的


