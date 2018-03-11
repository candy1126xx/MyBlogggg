---
layout:     post                    # 使用的布局
title:      ConcurrentHashMap原理与使用               # 标题 
subtitle:   ConcurrentHashMap原理与使用              #副标题
date:       2017-05-31              # 时间
author:     candy1126xx                      # 作者
header-img: img/home-bg-o.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Java
---

## 原理
ConcurrentHashMap采用分段锁，每一把锁锁住一段数据，这样在多线程访问不同段的数据时，不会存在锁竞争。

ConcurrentHashMap内部维护一个Segment数组segments和一个Node数组table。

Segment继承自ReentrantLock，代表一把重入锁。segments就是table所用到的所有锁。一个锁管理几个Node。

Node实现了Map.Entry接口，一个Node就是一个键值对。Node中封装了hash、key、value、next，可以在相同的hash值下建立链表。其中，key和hash用final修饰，value、next用volatile修饰。table就是整张哈希表。

#### Node与Segment的对应关系
Node在table中的索引、Segment在segments中的索引都是由key的哈希值计算得出的。其中：

* hash值 >> segmentShift位，然后再& segmentMask得到在segments中的索引；
* (table的长度 - 1) & hash 得到在table中的索引。

因此，Node和Segment实现对应。

#### put()实现
1. 计算Segment得到在segments中的索引；
2. 获取Segment对象，tryLock()尝试获取锁；
4. 获取成功后，计算Node在table中的索引；
5. 获取索引位置的链表的第一个Node，开始遍历；比较每个Node的key，如果相同更新value，否则创建新的Node加入链表末端。

#### get()实现
1. 计算Node在table中的索引；
2. 获取索引位置的链表的第一个Node，开始遍历；
3. 比较每个Node的key，如果相同返回value，否则返回null。

注意到get()操作并不加锁。读操作的正确性由final和volatile保证。

#### size()等需要遍历整个table的操作
先给3次机会，不lock所有的Segment，遍历所有Segment，累加各个Segment的大小得到整个Map的大小，如果某相邻的两次计算获取的所有Segment的更新的次数（每个Segment都有一个modCount变量，这个变量在Segment中的Entry被修改时会加一，通过这个值可以得到每个Segment的更新操作的次数）是一样的，说明计算过程中没有更新操作，则直接返回这个值。如果这三次不加锁的计算过程中Map的更新次数有变化，则之后的计算先对所有的Segment加锁，再遍历所有Segment计算Map大小，最后再解锁所有Segment。

## 对比HashTable的优势
HashTable线程安全的策略实现代价太大，get/put所有相关操作都是synchronized的，这相当于给整个哈希表加了一把大锁，多线程访问时候，只要有一个线程访问或操作该对象，其他线程只能阻塞，相当于将所有的操作串行化，在高并发场景中性能就会非常差。

ConcurrentHashMap采用分段锁，在容器中有多把锁，每一把锁锁住一段数据，这样在多线程访问时不同段的数据时，就不会存在锁竞争了，这样便可以有效地提高并发效率。