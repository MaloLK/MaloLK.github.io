---
layout: post
title: Memcached中的LRU和Hashtable
date: 2017-01-13 00:42
categories: programming
tags: [memcached, LRU, hashtable]
---

#### LRU 的组成
在最新的实现中，每个slabclass对应四个lru：HOT_LRU, WARM_LRU, COLD_LRU, NOEXP_LRU，管理LRU链表指针的数组的大小为256，而slabclass数组的大小为64，刚好是4倍的关系。，其中最新鲜的是HOT_LRU，如果处于活动的items在HOT_LRU或者WARM_LRU中，则不需移动，只有在处于COLD_LRU时才会将其移动到WARM_LRU中。另外，HOT_LRU和WARM_LRU中的items所占的内存的最大值为当前slabclass的32%。

##### memcached 中的延迟删除
实现flush_all的最直接的方式是一次将所有的items删除，但是这样花费的时间很长。memcached中使用延迟删除的方式，将当前执行flush_all命令的时间设置为oldest_live，在下次调用get等操作读取item时如果item最近一次访问的时间在oldest_live之前说明这个item已经被惰性删除了，因此不返回这个item而是将其删除。

##### 从 LRU 中分配一个 item
首先试着从cold_lru中回收过期的item, 然后从相应的slabclass中分配一个item，如果分配失败说明slabclass中已经没有空闲的item，在这种情况下再检查hot_lru和warm_lru中的item所占内存是否超过限制，如果超过限制则会将item移动到cold_lru，否则说明hot_lru和warm_lru中的item个数太少，如果item处于活跃状态则更新其在lru链表中的新鲜度。如果hot_lru和warm_lru中的item个数太少当前的item又不活跃则将其删除，这个过程不一定能够删除item，因此最后一次强制从cold_lru中删除item。如果经过上述的过程仍然不能够分配可用的item，系统则会认为是`Out of memory`。
接下值得注意的是，新分配的item所属的id（slabs_clsid, 这个名字起得比较混乱容易误解）不再仅仅是所属slabclass的id,而是所属lru的id。

##### item 在 LRU 中的操作
1. `do_item_update`: 在更新item的访问时间和在lru中的位置时会首先检查当前时间距离item的最近访问时间是否已经超过了预定的期限，如果是则更新。这样做的目的是防止频繁的更新操作耗费太多的时间，过多的get命令将会导致过多的更新操作，如果每次都更新（即使间隔时间很短），将会导致get请求变慢。
2. `do_item_get`（使用延迟删除的策略）: 首先在hashtable中查找这个item，如果找到则检查这个item是否已经被flush命令给“惰性删除了”，如果是则将其从hashtable和lru链表中删除，结束返回。如果没有被flushed掉则检查其是否已经过期，如果过期则将其从hashtable和lru链表中删除，结束返回。否则，将item置为`ITEM_FETECHED｜ITEM_ACTIVE`状态。
3. `do_item_touch`：这个操作调用上面的get操作，然后只是修改其过期时间。

##### 维护 LRU 的线程
1. `lru_maintainer_junngle`:　每调用一次则会检查当前的slabclass中所含的空闲的items是否太多，如果所有的空闲的items组成的内存超过2.5个page则认为空闲item太多，则将其中的一个slab转移到`global-page-pool`中。另外循环1000次，每次检查hot_lru和warm_lru中的item是否太多，如果太多则将其转移到cold_lru,如果cold_lru中的items太多，则将其删除归还给slab-class。这个函数的作用是不断的监测hot_lru和warm_lru的大小，如果过大则将多余的item转移到cold-lru中，检测cold-lru中的item是否处于活跃状态，如果处于活跃状态则将其转移到warm_lru中，否则将其删除。
2. `lru_maintainer_thread`: 主要的工作分为两个部分，第一部分为调用`lru_maintainer_junngle`控制lru中的item个数；另外一个作用调用`lru_maintainer_crawler_check`检查每个slab-class中的lru是否需要进行爬虫操作，如果是则将需要进行爬虫的slab-class的id等信息传递给`lru_crawler_start`用来启动爬虫功能。
3. `lru_crawler_start`：为需要进行爬虫操作的slab-class的每个lru链表后插入用于爬虫的item,　这个爬虫item的主要目的是记录当前在lru链表中的位置，并检查item是否已经过期或者已经被flushed，如果是则将其删除，这就是所谓的惰性删除。函数的最后是唤醒用于爬虫的线程`item_crawler_thread`。
4. `item_crawler_thread`: 对所有需要进行爬虫的lru依次进行爬虫，每次检查当前lru中的一个item，接着检查下一个lru。但是为了防止爬虫时间过长影响正常的处理，每处理一个item，则将相关的锁释放，源码中会每处理`crawls_persleep`个item就进入睡眠。当所有的lru都处理完毕，则爬虫线程将会进入睡眠，等待`lru_maintainer_thread`线程将其唤醒。

#### Hashtable
hash表由一个指针数组构成，其中数组中的每个指针元素是一个链表。数组的大小是2的幂次，在初始化时需要指定数组的幂次。通过幂次分配指针数组。

##### hashtable 的扩展
memchached中要求hashtable中的元素的个数不能超过数组的大小的1.5倍，如果超过则需要扩展。扩展的任务由一个单独的线程来完成（`assoc_maintenance_thread`）。这个线程主要的工作流程如下：
1. 在初始阶段不需要扩展hashtable，此时该线程将会睡眠在条件变量`maintenance_cond`上。
2. 当其他的worker线程在不断的向hashtable插入元素时，元素个数超过hashtable的大小的1.5倍，此时会调用`assoc_start_expand`将`assoc_maintenance_thread`唤醒
3. 此时hashtable维护线程会将所有的其他线程给阻塞，然后调用`assoc_expand`将当前的`primary_hashtable`赋给`old_hashtable`，并为`primary_hashtable`设置新的指针数组。接着解除其他线程的阻塞。
4. **扩展过程每次只会转移少数几个桶，这样做也是为了防止迁移时间过长**。
5. 当迁移完毕时，hashtable维护线程则会重新进入睡眠状态。

##### Hashtable 迁移过程中的查找，插入和删除
- 插入操作：首先检查是否正在扩展hashtable，且所找key对应的bucket仍然在`old_hashtable`中，如果是则将其插入到`old_hashtable`中。**优先插入到`old_hashtable`中的原因是利于查找**，如果优先插入到`primary_hashtable`中，则需要检查两个hashtable。
- 查找和删除：也是先检查是否正在扩展hashtable，如果是则查看是否在`old_hashtable`中，否则再查看`primary_hashtable`。

##### Hashtable 中的锁机制
由上述可以知道，当hashtable开始进行扩展时，它会将所有其他会操作hashtable的线程暂停，此时将不会有不安全的操作发生。当将`primary_hashtable`和`old_hashtable`切换完毕后，对hashtable的访问将会自动判别item在哪个hashtable中。值得说的是，对hashtable的并发访问使用的是段锁，所谓段锁就是hashtable中的多个bucket的并发访问由一个锁访问。这里需要注意的是在进行迁移过程中，迁移线程转移一个bucket时是使用的是这个bucket的序号作为hash_value来获得段锁：`item_lock = item_trylock(expand_bucket)`。由于有可能当前的bucket中有其他的worker线程正在访问里面的item，所以需要迁移bucket时需要加锁，但是这里用的是桶的序号(expand_bucket)来获得锁，那么这个锁和使用桶内的item获得的锁是同一个锁吗？答案是：是的。原因是这样的：在item_get函数中使用的是item中key的hash值来获得锁，在源码中的设计是这样的，桶的个数是N个，段锁的个数是M个，且保证M小于N，由于item选择桶是根据取其hash值低N位，即`hash&(1 << N - 1)`，因此当使用bucket序号获取锁时相当于是`hash&(1 << N - 1) & (1 << M - 1)`，由于N > M,因此可以简化为`hash&(1 << M - 1)`,　跟使用item_get获取锁的方式是一样的，所以如果在迁移过程中，使用bucket序号获得对应的锁后，该bucket中的item将因暂时无法获得锁而阻塞，从而保证hashtable扩展的线程安全。
