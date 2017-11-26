---
title: redis源码解读(七):基础数据结构之skiplist
date: 2017-11-26 20:11:30
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
6. [**quicklist**：快速列表（双向链表和压缩表二合一的复杂数据结构）](/redis/2017/11/25/redisSourceQuicklist#quicklist)
7. [**skiplist**：跳跃链表](#skiplist)


# skiplist
