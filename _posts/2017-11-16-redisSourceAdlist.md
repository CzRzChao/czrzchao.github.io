---
title: redis源码解读(二):基础数据结构之ADLIST
date: 2017-11-16 22:40:30
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

1. [**SDS(simple dynamic string)**：简单动态字符串](/redisSourceSds#sds)
2. [**ADList(A generic doubly linked list)**：双向链表](#adlist)
3. [**dict(Hash Tables)**：字典](/redisSourceDict#dict)
4. [**intset**：整数集合](/redisSourceIntset#intset)
5. [**ziplist**：压缩表](/redisSourceZiplist#ziplist)
6. [**quicklist**：快速列表（双向链表和压缩表二合一的复杂数据结构）](/redisSourceQuicklist#quicklist)
7. [**skiplist**：跳跃链表](/redisSourceSkiplist#skiplist)


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


## 总结一波
1. *redis* 的链表是双向无环的
2. `ADList`持有链表的头尾节点指针，链表长度以及链表操作函数
3. `listNode`的值为void，可以保存任意类型的值
4. 通过实现不同的操作函数，`ADlist`可以保存各种不同类型的数据
5. 迭代操作由专门的迭代器实现

其他一些api就不做深入解析了，`ADList`相关的定义都在`adlist.h`和`adlist.c`文件中，如果感兴趣可以自行查看
