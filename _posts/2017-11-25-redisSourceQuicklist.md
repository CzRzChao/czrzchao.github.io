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
在3.2之前，`list`是根据元素数量的多少采用`ziplist`或者`adlist`作为基础数据结构，具体不太清楚是为何统一改用`quicklist`，但是从数据结构的角度来说`quicklist`结合了两种数据结构的优缺点，复杂但是实用。

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
`quicklistNode`实际上就是一个压缩表，其中包括：

* 指向前后压缩表节点的两个指针
* `zl`：压缩表数据字符串指针
* `sz`：压缩表的内存大小
* `count`：压缩表内部数据的个数
* `encoding`：压缩表编码方式，1为默认方式，2为LZF数据压缩方式
* `recompress`：是否压缩，1表示压缩  
这里从变量count开始，都采用了位域的方式进行数据的内存声明，使得6个`unsigned int`变量只用到了一个`unsigned int`的大小，最大程度的节省内存。
> C语言支持位域的方式对结构体中的数据进行声明，也就是可以指定一个类型占用几位：  
1) 如果相邻位域字段的类型相同，且其位宽之和小于类型的sizeof大小，则后面的字段将紧邻前一个字段存储，直到不能容纳为止；  
2) 如果相邻位域字段的类型相同，但其位宽之和大于类型的sizeof大小，则后面的字段将从新的存储单元开始，其偏移量为其类型大小的整数倍；  
3) 如果相邻的位域字段的类型不同，则各编译器的具体实现有差异，VC6采取不压缩方式，Dev-C++采取压缩方式；  
4) 如果位域字段之间穿插着非位域字段，则不进行压缩；  
5) 整个结构体的总大小为最宽基本类型成员大小的整数倍。  

`sizeof(quicklistNode); // 32`，通过位域的声明方式，`quicklistNode`可以节省24个字节。

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

为正数时，最大为32kb：

```c
#define FILL_MAX (1 << 15)  // 32kb
void quicklistSetFill(quicklist *quicklist, int fill) { // set ziplist的单个节点最大内存
    if (fill > FILL_MAX) {
        fill = FILL_MAX;
    } else if (fill < -5) {
        fill = -5;
    }
    quicklist->fill = fill;
}
```

## quicklistNode存储结构