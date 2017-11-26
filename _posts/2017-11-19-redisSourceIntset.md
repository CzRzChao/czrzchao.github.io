---
title: redis源码解读(四):基础数据结构之intset
date: 2017-11-19 23:30:10
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
4. [**intset**：整数结合](#intset)
5. [**ziplist**：压缩表](/redis/2017/11/24/redisSourceZiplist#ziplist)
6. [**quicklist**：快速列表（双向链表和压缩表二合一的复杂数据结构）](/redis/2017/11/25/redisSourceQuicklist#quicklist)
7. [**skiplist**：跳跃链表](/redis/2017/11/26/redisSourceSkiplist#skiplist)

# intset
整数集合是 *redis* 对外数据结构`set`的底层实现之一，当集合元素不大于设定值并且元素都是整数时，就会用`intset`作为`set`的底层数据结构。

## 定义
`inset`结构体定义如下：

```c
typedef struct intset {
    uint32_t encoding;  // 编码方式，一个元素所需要的内存大小
    uint32_t length;    // 集合长度
    int8_t contents[];  // 集合数组
} intset;
```
* `encoding`为`inset`的编码方式，有3种编码方式，分别对应不同范围的整型：
	
	```c
	#define INTSET_ENC_INT16 (sizeof(int16_t))  // -32768~32767
	#define INTSET_ENC_INT32 (sizeof(int32_t))  // -2147483648~2147483647
	#define INTSET_ENC_INT64 (sizeof(int64_t))  // -2^63~2^63-1
	```
	`intset`的编码是由最大的一个数决定的，如果有一个数是int64，那么整个`inset`的编码都是int64。
* `length`是`inset`的整数个数
* `contents`整数数组

`intset`的内存是连续的，所有的数据增删改查操作都是在内存地址偏移的基础上进行的，并且整数的保存也是有序的，一个保存了5个int16的`intset`的内存示意图如下：
![intest](/images/intset.png)

由于`intset`是在内存上直接操作赋值，并且所存储的值都超过了一个字节，所以需要考虑大小端的问题：
>大端模式，是指数据的高字节保存在内存的低地址中，而数据的低字节保存在内存的高地址中，这样的存储模式有点儿类似于把数据当作字符串顺序处理：地址由小向大增加，而数据从高位往低位放；这和我们的阅读习惯一致。  
小端模式，是指数据的高字节保存在内存的高地址中，而数据的低字节保存在内存的低地址中，这种存储模式将地址的高低和数据位权有效地结合起来，高地址部分权值高，低地址部分权值低。

*redis* 的所有存储方式都是小端存储，在`endianconv.h`中有一段大小端的宏定义，如果当前cpu的字节序为大端就进行相应的转换：

```c
#if (BYTE_ORDER == LITTLE_ENDIAN)
#define memrev16ifbe(p)
#define memrev32ifbe(p)
#define memrev64ifbe(p)
#define intrev16ifbe(v) (v)
#define intrev32ifbe(v) (v)
#define intrev64ifbe(v) (v)
#else
#define memrev16ifbe(p) memrev16(p)
#define memrev32ifbe(p) memrev32(p)
#define memrev64ifbe(p) memrev64(p)
#define intrev16ifbe(v) intrev16(v)
#define intrev32ifbe(v) intrev32(v)
#define intrev64ifbe(v) intrev64(v)
#endif
```
在`intset`相关的源码中有很多`intrev32ifbe`之类的操作就是在进行大小端转换。大小端深入的一些知识点就不在这做详解，可以自行google或百度。

## 新增元素
这里针对`intset`的新增元素的过程做一个解析，因为这个过程涉及到了`intset`的升级、查找和插入。  
首先看新增元素的主体函数：

```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {    // 新增
    uint8_t valenc = _intsetValueEncoding(value);   // 获取对应value编码
    uint32_t pos;
    if (success) *success = 1;

    if (valenc > intrev32ifbe(is->encoding)) {  // 编码大于当前，升级新增
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);   // 升级并新增
    } else {
        if (intsetSearch(is,value,&pos)) {  // 查找是否存在，pos为小于value的最大值的pos
            if (success) *success = 0;
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1);   // 重新多申请一个空间
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);   // 如果没有找到pos是小于该数字的前一个， 将pos后数据后移一位
    }

    _intsetSet(is,pos,value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```
这个函数获取了value值对应的编码，这个编码是根据3种编码的数据范围确定的。如果待插入数据的编码大于当前`intset`的编码，就需要进行升级，这个先跳过，我们先看正常的新增流程。  
为了确保`intset`元素的唯一性，再插入之前会进行一次查找，`intsetSearch`函数定义如下：

```c
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) { // 查找
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    /* The value can never be found when the set is empty */
    if (intrev32ifbe(is->length) == 0) {    // intset空值判断
        if (pos) *pos = 0;
        return 0;
    } else {
        if (value > _intsetGet(is,intrev32ifbe(is->length)-1)) {
            if (pos) *pos = intrev32ifbe(is->length);   // 如果value大于当前intset的最大值，将pos赋值为length
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;  // 如果value小于当前intset的最小值，将pos赋值为0
            return 0;
        }
    }

    while(max >= min) { // 二分查找
        mid = ((unsigned int)min + (unsigned int)max) >> 1; // (min+max)/2
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) { // 找到对应元素
        if (pos) *pos = mid;
        return 1;
    } else {    // 没有找到
        if (pos) *pos = min;
        return 0;
    }
}
```
上述函数的作用就是利用`intset`有序的特性，通过二分法对目标value进行查找，如果找到返回1，反之返回0，pos作为引用传入函数中，会被赋值为value在intset中对应的位置。  
`intsetSearch`中多次调用的`_intsetGet`是用来获取对应pos的value值的函数：

```c
static int64_t _intsetGet(intset *is, int pos) {    // 获取值
    return _intsetGetEncoded(is,pos,intrev32ifbe(is->encoding));
}

static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc) {    // 根据encode获取对应的值
    int64_t v64;
    int32_t v32;
    int16_t v16;

    if (enc == INTSET_ENC_INT64) {
        memcpy(&v64,((int64_t*)is->contents)+pos,sizeof(v64));
        memrev64ifbe(&v64); // 大小端转换
        return v64;
    } else if (enc == INTSET_ENC_INT32) {
        memcpy(&v32,((int32_t*)is->contents)+pos,sizeof(v32));
        memrev32ifbe(&v32);
        return v32;
    } else {
        memcpy(&v16,((int16_t*)is->contents)+pos,sizeof(v16));
        memrev16ifbe(&v16);
        return v16;
    }
}
```
可以看到`intset`在获取值的时候都是通过地址偏移、内存拷贝，然后进行大小端转换处理完成的。

继续之前的新增元素流程，当查不到对应value时，会在原有内存的基础上进行realloc，多申请一个`intset->encoding`的内存。由于`intset`的内存为连续，因此插入时，比value大的元素都要向后移动一个`intset->encoding`，也就是`intsetMoveTail`函数干的活：

```c
static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {    // 将数据后移
    void *src, *dst;
    uint32_t bytes = intrev32ifbe(is->length)-from;
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        src = (int64_t*)is->contents+from;
        dst = (int64_t*)is->contents+to;
        bytes *= sizeof(int64_t);
    } else if (encoding == INTSET_ENC_INT32) {
        src = (int32_t*)is->contents+from;
        dst = (int32_t*)is->contents+to;
        bytes *= sizeof(int32_t);
    } else {
        src = (int16_t*)is->contents+from;
        dst = (int16_t*)is->contents+to;
        bytes *= sizeof(int16_t);
    }
    memmove(dst,src,bytes); // 由于移动前后地址会有重叠，因此要利用memmove进行内存拷贝 memcpy无法保障结果正确性
}
```
由于移动的操作是在原有内存地址基础上进行的，因此在这里不能用`memcpy`进行内存拷贝，需要用`memmove`。在内存重叠的情况下，`memcpy`在拷贝的过程中，可能部分地址在被拷贝之前就被新的值覆盖了，导致拷贝这部分地址时拷贝的并不是我们期望的值。依旧是老套路，感兴趣自己去google或百度吧！    
最后的`_intsetSet`和`_intsetGet`差不多，就不多讲了。

## 升级
上面只介绍了`intset`普通的新增场景，那么当插入的value大于当前`intset`的encode时就需要对`intset`进行升级，以适应更大的值：

```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) { // 升级并且添加新元素
    uint8_t curenc = intrev32ifbe(is->encoding);
    uint8_t newenc = _intsetValueEncoding(value);
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize */
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */
    while(length--) // 从尾部开始，将原有数据进行迁移
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    if (prepend)    // 小于0在集合头部
        _intsetSet(is,0,value);
    else    // 在集合尾部
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```
首先当需要对原有`intset`进行升级时，插入的元素一定是大于当前`intset`的最大值或者小于当前`intset`的最小值的，因此带插入的value一定是在首尾，只需判断其正负即可。  
升级的操作主要是将原本数据的内存地址大小进行一个统一的变更，从原`intset`的`length`+`prepend`开始，一个一个扩展迁移。  
进行完扩展迁移之后把带插入的元素插入到头或尾即可。  
一个`INTSET_ENC_INT16`->`INTSET_ENC_INT32`的升级示例如下图：
![intsetupgrade](/images/intsetupgrade.png)

## 总结一波
`intset`主要有以下特性：

1. 内存连续，数值存储有序、无重复
2. 有三种编码方式，通过升级的方式进行编码切换
3. 不支持降级
4. 小端存储

其他一些删除、随机获取value等api就不详细介绍了。老套路，源码在`intset.h`和`intset.c`中。

