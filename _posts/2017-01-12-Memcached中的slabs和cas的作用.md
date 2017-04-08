---
layout: post
title: Memcached中的slabs和CAS(Compare-And-Set)
date: 2017-01-12 00:42
categories: programming
tags: [memcached, memory, CAS]
---

#### slabs 的结构
memcached中将slab分为多类，分类的标准是slab中的基本块item的大小，相邻的slabclass中的item大小关系有一个固定的系数(1.25)决定。这些slab类以一个数组进行存储，其中每个元素是一个`slabclass_t`指针。每个slabclass中含有多个相同类型的slab, 每个slab实际上是一个page(大小为1M)，然后这个page按照item的大小被分为多个item，这些空闲的item由slots链表进行管理。

#### slabs 的初始化过程
1. 如果程序要求预先分配内存，则会将所要求的内存预先分配出来。
2. 为每一个slabclass计算item的大小，并保证item大小是内存nbytes对齐。
3. 如果要求预先分配，则为每一个slabclass分配一个slab(即1M)的内存，并根据每个slabclass中item的大小将其分割为items，由slots管理。

#### 分配 slab 的过程
1. 扩展相应slabclass的slab指针链表
2. 试图从全局的空闲pages中分配，如果没有，则会从已经分配的页面中获取（这是预分配的情况）或者直接从系统中分配内存。
3. 将分配的页按照item的大小（item结构体的大小加上数据部分的大小）把slab分割为空闲的item，交由slots管理。

#### 从 slabclass 中分配 item
1. 如果当前的slots链表中没有空闲的item，则重新分配slab。
2. 从slots中取出一个item，增加其引用计数返回。

#### 将 item 归还给 slabclass
1. 将item的flag设置为`ITEM_SLABBED`，表示其没有被取走
2. 将item中所属的class设置为０
3. 将item加入到空闲item链表slots中

#### slab 的 rebalance 过程
在实际应用过程中，有的slab-class 使用的比其他的slab-class多，使用的较少的slab-class所占用的内存将会浪费内存，即使有爬虫线程在不断的删除过期的item，但是这些回收的item是否归还给当前的slab-class，所以需要额外的平衡线程将用的少的slab-class的内存页传送到用的比较多的slab-class中。
1. `slab_rebalance_start`: 这个函数的功能主要的作用是选取需要转出内存页的slab-class的内存页，源码中选取的是slab-class中的第一个内存页，因为第一个内存页中的items最先分配出去，其中的items存在的时间一般较大。确定这个内存页的起始位置和终止位置。
2. `slab_rebalance_alloc`: 这个函数的主要作用是从指定的slab-class中分配一个item，如果这个item处于被转移的内存页中，则将其引用置为0,　接着分配下一个item。
3. `slabs_reassign`: 这个函数的作用是设置需要进行内存页转移的源和目的slab-class，并唤醒rebalance线程进行内存页转移操作。这个函数主要由LRU维护线程在调用`lru_maintainer_junngle`时调用，当然前提是开启auto_move功能。
4. `slab_rebalance_thread`: 这是rebalance线程，首先调用`slab_rebalance_start`确定要转移的源slab-class中的slab。然后调用`slab_rebalance_move`将要转移的slab中的item全部回收。最后如果回收完毕，则会调用`slab_rebalance_finish`将slab从源slab-class转移到目的slab-class中。

#### item 中的 CAS（Compare-And-Set）的作用
CAS(Compare-And-Set)通常在以下情况使用：多个并发请求以原子的方式更新memcahced中的某个key。
假设现在有两个并发请求A和B，两个请求都希望去增加memcache中的counter，可能的更新顺序如下：
```
A: counter = memcache.get(key) # Reads 42
B: counter = memcache.get(key) # Reads 42
A: memcache.set(key, counter+1) # Writes 43
B: memcache.set(key, counter+1) # Writes 43
```
如上所示，并没有将counter增加2。另外使用每次修改后再读取检查是否修改的方法也不可行，使用额外的锁也可能因为网络原因导致死锁。使用CAS可以解决这个问题：
```
client = memcache.Client();
while (true) {
  counter = client.gets(key);
  if (client.cas(key, counter+1)) {
    break;
  }
}
```
每次client更新数据时，首先使用gets获得相应的key, key里包含当前value的一个版本（或者是timestamp），每次key对应的value被更新，key中的版本则会更新。因此当client更新value时需要提交上次读取的key，如果memchach检测到提供的key和server中的key不相同时，说明期间value已经被其他客户端更新，此时则会返回false，不做任何更新。客户端接收到false结果后则会继续尝试，当然实际实现中需要限制尝试的次数，因为有可能系统接受的请求太多，导致某些客户的请求一直无法满足。回到上面A和B两个并发请求的执行顺序，如果使用cas，那么第一次只有请求A会执行成功，而请求B的cas操作将会返回false，请求B再尝试一次则会成功。当然要达到上述的效果，必须要求在memcache中针对某个key的cas是串行操作，而不是并行操作。
Ref: http://neopythonic.blogspot.com/2011/08/compare-and-set-in-memcache.html
