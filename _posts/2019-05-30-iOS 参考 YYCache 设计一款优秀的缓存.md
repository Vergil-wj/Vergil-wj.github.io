---
layout:     post
title:      参考 YYCache 设计一款优秀的缓存
subtitle:   缓存
date:       2019-05-30
author:     Vergil
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - iOS
    - 缓存
---

[YYCache](https://github.com/ibireme/YYCache) 是国内开发者 ibireme 开源的一个线程安全的高性能键值缓存组件。此项目虽然已经不再维护，但其影响力还是很大的，实属国内精品，还是值得我们学习一下的。

## 设计一个优秀的缓存

从 YYCache 源码中，我们得出设计一个优秀的缓存可以从以下几点入手：

- 内存缓存和磁盘缓存
- 线程安全
- 缓存控制
- 缓存替换策略
- 缓存命中率
- 性能

### 内存缓存和磁盘缓存

![](https://upload-images.jianshu.io/upload_images/1776587-32f42af2bbcbc7e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

YYCache 是由内存缓存 YYMemoryCache 与磁盘缓存 YYDiskCache 相互配合组成的，内存缓存提供容量小但高速的存取功能，磁盘缓存提供大容量但低速的持久化存储。这样的设计支持用户在缓存不同对象时都能够有很好的体验。

在 YYCache 中使用接口访问缓存对象时，会先去尝试从内存缓存 YYMemoryCache 中访问，如果访问不到（没有使用该 key 缓存过对象或者该对象已经从容量有限的 YYMemoryCache 中淘汰掉）才会去从 YYDiskCache 访问，如果访问到（表示之前确实使用该 key 缓存过对象，该对象已经从容量有限的 YYMemoryCache 中淘汰掉成立）会先在 YYMemoryCache 中更新一次该缓存对象的访问信息之后才返回给接口。

### 线程安全
如果说 YYCache 这个类是一个纯逻辑层的缓存类（指 YYCache 的接口实现全部是调用其他类完成），那么 YYMemoryCache 与 YYDiskCache 还是做了一些事情的（并没有 YYCache 当甩手掌柜那么轻松），其中最显而易见的就是 YYMemoryCache 与 YYDiskCache 为 YYCache 保证了线程安全。

YYMemoryCache 使用了 pthread\_mutex 线程锁来确保线程安全，而 YYDiskCache 则选择了更适合它的 dispatch\_semaphore。

![](https://upload-images.jianshu.io/upload_images/1776587-7710b98765c7159d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为什么内存缓存使用互斥锁（pthread_mutex）？

>ibireme: 苹果员工说 libobjc 里 spinlock 是用了一些私有方法 (mach_thread_switch)，贡献出了高线程的优先来避免优先级反转的问题，但是我翻了下 libdispatch 的源码倒是没发现相关逻辑，也可能是我忽略了什么。在我的一些测试中，OSSpinLock 和 dispatch_semaphore 都不会产生特别明显的死锁，所以我也无法确定用 dispatch_semaphore 代替 OSSpinLock 是否正确。能够肯定的是，用 pthread_mutex 是安全的。

为什么磁盘缓存使用的是信号量（dispatch_semaphore）？

>dispatch\_semaphore 是信号量，但当信号总量设为 1 时也可以当作锁来。在没有等待情况出现时，它的性能比 pthread_mutex 还要高，但一旦有等待情况出现时，性能就会下降许多。相对于 OSSpinLock 来说，它的优势在于等待时不会消耗 CPU 资源。对磁盘缓存来说，它比较合适。

### 缓存控制
YYCache 提供了三种控制维度，分别是：cost、count、age。这已经满足了绝大多数开发者的需求，我们在自己设计缓存时也可以根据自己的使用环境提供合适的控制方式。

### 缓存替换策略
YYCache 使用了双向链表（_YYLinkedMapNode 与 _YYLinkedMap）实现了 LRU(least-recently-used) 策略，旨在提高 YYCache 的缓存命中率。

LRU 是缓存替换策略（Cache replacement policies）的一种，还有很多缓存替换策略诸如:

- First In First Out (FIFO)
- Last In First Out (LIFO)
- Time aware Least Recently Used (TLRU)
- Most Recently Used (MRU)
- Pseudo-LRU (PLRU)
- Random Replacement (RR)
- Segmented LRU (SLRU)
- Least-Frequently Used (LFU)
- Least Frequent Recently Used (LFRU)
- LFU with Dynamic Aging (LFUDA)
- Low Inter-reference Recency Set (LIRS)
- Adaptive Replacement Cache (ARC)
- Clock with Adaptive Replacement (CAR)
- Multi Queue (MQ) caching algorithm|Multi Queue (MQ)
- Pannier: Container-based caching algorithm for compound objects

LRU(least-recently-used) 翻译过来是“最近最少使用”，顾名思义这种缓存替换策略是基于用户最近最少访问过的缓存对象而建立。

我认为 LRU 缓存替换策略的核心思想在于：LRU 认为用户最新使用（访问）过的缓存对象为高频缓存对象，即用户很可能还会再次使用（访问）该缓存对象；而反之，用户很久之前使用（访问）过的缓存对象（期间一直没有再次访问）为低频缓存对象，即用户很可能不会再去使用（访问）该缓存对象，通常在资源不足时会先去释放低频缓存对象。

### 缓存命中率

- 缓存命中 = 用户要访问的缓存对象在高速缓存中，我们直接在高速缓存中通过 Hash 将其找到并返回给用户。

- 缓存命中率 = 用户要访问的缓存对象在高速缓存中被我们访问到的概率。

- 缓存丢失 = 由于高速缓存数量有限（占据内存等原因），所以用户要访问的缓存对象很有可能被我们从有限的高速缓存中淘汰掉了，我们可能会将其存储于低速的磁盘缓存中（如果磁盘缓存还有资源的话），那么就要从磁盘缓存中获取该缓存对象以返回给用户，这种情况我理解为（高速）缓存未命中，即缓存丢失（并不是真的被我们丢掉了，但肯定是被我们从高速缓存淘汰掉了）。

YYCache 作者通过 \_YYLinkedMapNode 与 \_YYLinkedMap 双向链表实现 LRU 缓存替换策略的思路。

### 性能
其实性能这个东西是隐而不见的，又是到处可见的。它从我们最开始设计一个缓存架构时就被带入，一直到我们具体的实现细节中慢慢成形，最后成为了我们设计出来的缓存优秀与否的决定性因素。

YYCache 中对于性能提升的实现细节：

- 异步释放缓存对象
	> Note: 对象的销毁虽然消耗资源不多，但累积起来也是不容忽视的。通常当容器类持有大量对象时，其销毁时的资源消耗就非常明显。同样的，如果对象可以放到后台线程去释放，那就挪到后台线程去。这里有个小 Tip：把对象捕获到 block 中，然后扔到后台队列去随便发送个消息以避免编译器警告，就可以让对象在后台线程销毁了。

- 锁的选择
- 使用 NSMapTable 单例管理的 YYDiskCache
- YYKVStorage 中的 _dbStmtCache
- 甚至使用 CoreFoundation 来换取微乎其微的性能提升


参考资料：

[https://github.com/ibireme/YYCache](https://github.com/ibireme/YYCache)

[https://lision.me/yycache/](https://lision.me/yycache/)