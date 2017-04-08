title: leveldb-1.19源码阅读
layout: post
date: 2016-10-11 12:03
categories: source-reading
tags: [leveldb, database, C++]

---

### 1. leveldb源码阅读总结
阅读源码应该选择较低的版本，可以快速掌握整个功能的实现。阅读较高的版本就会考虑一些优化和设计上的完善，但是可能使得阅读源代码时不能快速理清思路。以下的源码剖析基于leveldb-1.19。
<!--more-->

* **编码的设计**
在对key-value进行存储时采用了这些优化存储效率的方式：使用LengthPrefixed（使用Varint32编码）的方式存储key-value、使用key值共享的方式缩短键值占用的空间、使用多个restart-point提供了key值共享方式的健壮性以及使用crc32进行校验和检验数据的正确性。使用slice代替string来传递key-value减少string的拷贝。
* **cache的设计**
leveldb提供两层的缓存，一层是对一个table进行缓存，更低一层是对于block进行缓存。在cache的实现中单个元素LRUHandle中使用柔性数组的方式来表示key值部分减少内存碎片加快访问速度。另外一个Cache通过hash成16（默认值）个ShardedLRUCache，每个ShardedLRUCache都含有各自的互斥锁来提供并发访问，通过分片的方式对请求进行load-balance可以减少多个线程同时访问一个ShardedLRUCache的概率因此可以提高cache的并发度。
* **内存池arena的设计**
内存池主要为memtable服务，在每次key-value编码和插入memtable中的skiplist时进行分配，这些动态内存一般都比较小因此使用内存池来分配。在memtable析构时arena将会释放分配的内存。
* **skiplist的读取无锁设计**
使用编译器memory-barrier来保证从内存读取，可以实现单写多读且不需加锁，但是多个线程写入时需要锁进行同步。
* **BloomFilter的设计**
在实际实现中使用一次hash代替k次hash的方法高效产生其余k-1个hash值。BloomFilter也是快速确定block中是否存在某个key值，如果判断为不存在则一定不存在，判断为存在只能说明可能存在，通过参数选取可以保证较低的误报率。
* **迭代器的设计**
leveldb中由小到大为每个数据集都实现了相应的迭代器且都继承自统一的抽象类Iterator，另外通过实现Iterator的包装类IteratorWrapper为key值做了缓存来减少虚函数的调用。
* **snapshots的设计**
leveldb为每一个加入的key-value对或者key-deletion-mark附上了相应的sequence-number，而snapshots相当于是捕捉某一时刻的sequence-number，捕捉完成之后，不管当前的sequence-number已经增长到了哪里，当读取snapshots所标志时刻的数据库时只关心seq比snapshots中的sequence-number小的key值，另外由于sequence的存在，即使多个user-key相同但是seq不同的key存在，只要选择其中seq最大的即可，seq越大表示对相关key的更新越新鲜。因此读取snapshots所标志的数据库中的key-value时，只会读取小于等于snapshots中sequence-number但是又是user-key相同的key中seq最大的那个。这就很好的阐释了“快照”的语义。
* **容错性设计**
在每次写入memtable之前都会将相同的内容写入log文件夹中，比如此时的memtable的内容还没导入到文件中，但是系统崩溃掉后memtable中内容将会消失，但是log文件已经记录下当前memtable中的内容因此可以再次恢复。
* **version的设计**
使用version的设计可以实现从磁盘中进行状态恢复，只要读取MANIFEST文件中记录的VersionEdit进行回放则可回到上次数据库关闭之前的状态。
* **进程独占的设计**
在每次打开数据库之前都试图获得文件锁，文件锁被设置为读写锁因此是进程互斥，从而保证同一时刻只能有一个进程打开并使用数据库。同样在销毁数据库时也是通过文件锁来保证当前进程独占的功能。


### 2. 框架

援引自网络。
![leveldb框架](/images/leveldb.JPG)
从图中可以看出每次write都会现将数据写入磁盘文件log中防止写入memtable时由于系统崩溃而出现数据丢失的情况，然后再写入内存memtable中，其中Immutable-memtable主要是用来将内存中的数据导入到Table文件，在leveldb-1.19版本中已经没有使用SSTTable文件而只是使用Table文件。另外manifest文件的作用是记录的是数据库中每次更新有哪些文件增加或删除以及一些序号如log\_number和next\_file\_number。current文件中的内容就是当前manifest的文件名。


### 3. 源码剖析(levedb-1.19)

#### 3.1 Util

**1. Enoding**

leveldb的编码使用小端格式进行存储，即低字节\-\>低地址、高字节\-\>高地址。值得关注的的是变长整形（varint32, varint64）的编码采用的方式是，只对无符号整数进行编码，所形成的编码中每个字节的最高一位若为1则表明后面的紧接着更高地址的字节，为0表示此字节为最后一个字节，每个字节中的低7位串接起来表示被编码的值。
很多地方用到了slice这个类型来代替string，原因有二：返回slice比返回string的代价更小，slice只有一个指针和一个表示长度的值; leveldb中的key和value允许'\0'的字符出现，因此也不能使用以'\0'结尾的字符串。使用slice时需要保证所指的数据必须在有效生命周期，slice不对所指资源进行管理。

**2. Cache**

* **LRUHandle**
cache中每个元素被封装为一个`LRUHandle`，存储相应的key、value。其中key的存储则是以长度`key_length`和柔性数组`key_data[1]`来定义。**为什么使用柔性数组**？这篇[文章](http://coolshell.cn/articles/11377.html)解释的很清楚。按照常规的方法就是使用指针来代替变长数组的地址，文章总结了使用柔性数组的好处:(1)方便内存释放，分配内存时只需要分配一次，自然释放内存也只需要free一次，使用指针则需要两次内存分配，而且需要用户能够记得释放指针成员所指的内存。（2）使用柔性数组分配内存是连续的，有利于提高访问速度，同时减少内存碎片。(#更新，不使用指针的原因还有一个，指针会占用结构体的内存，而数组名只是一个代表偏移地址的符号，不会占用结构体内存。)
其中的`next_hash`用于HashTable中的链表的指针操作，而`next`和`prev`则用于LRUCache中双链表的操作。
* **HandleTable**
以`LRUHandle*`为元素构建的一个HashTable，主要用于LRUCache的快速查找。关于其实现只是一个比较普通的实现，rehash的条件是每个buckets中欧能否链表的长度大于buckets的数目。需要关注的一点是，**链表的操作使用的是指针的指针来进行操作，使得代码变得非常简洁，但是使得可读性变差**。
* **LRUCache**
维持两个双链表，一个存储在Cache中的元素，一个用于存储正在使用的元素。LRU实现的原理是将最近使用的元素放在链表头`dummy-head`之前的位置，则越靠近链表头之前则越新鲜，反之越靠近链表头之后的位置则越旧。需要删除元素时则删除链表头之后的元素。同时还维持一个HandTable用于快速查找元素是否在cache中。
* **ShardedLRUCache**
维持一个LRUCache数组，默认大小为16。通过key的hash值的高位来选择将key，value对存储在哪个LRUCache中，**由于每个LRUCache中使用了互斥锁来保证线程安全，使用分片的方式可以减少线程之间因为锁竞争而降低并发性能**。

**3. Arena**
内存池的实现相比较于C++ STL中的实现，显得比较简单，而且内存利用率不高。使用内存池的目的还是为了减少小内存分配时new的消耗，以及减少内存碎片化。**另外一个值得注意的一个地方是内存池的内存用量`memory_usage`使用的是`AtomicPointer`,内部的实现使用了memory barrier来抑制编译器的优化，但是挡不住CPU的指令乱序执行，因此只能保证每次读取数据是从内存中直接读取以及写入数据时是直接写入内存，主要使用在skiplist的单写多读（即一个线程写同时多个线程读，如果是多个线程写则需要额外的同步机制）的设计中。

**4. Comparator**

`Class BytewiseComparatorImpl`是基于`Slice`的字母顺序排序的默认比较类。类中核心的成员函数为`FindShortestSeparator`和`FindShortSuccersor`。
* **FindShortestSeparator**
此函数实现的功能是找出两字符串中具有共同前缀子串的最短可区分字符串，函数实现的思路为首先找出两字符串中第一个不相等的字符（若没有，则返回长度较小的字符串），若此字符小于0XFF且加1后小于另一个字符串相应位置的字符，则将其加1,并将此字符之后的字符截断。这样做的目的，一方面是压缩key所占空间大小，另一方面通过最短的区分字符串来实现二分查找加快查询速度。
* **FindShortSuccersor**
函数的功能是获得比字符串start大的最短字符串，如给定字符串`start==“abcd”`, 则将start截短为`start=="b"`, 特殊情况是start全为OXFF，则不进行截断。主要功能是用来确定sstable中block的end-key。

**5. BloomFilter**
BloomFilter，使用一个比特数组来检测元素是否存在，每个元素经过k个不同的hash函数的到k个索引值，并将比特数组中这些位置全置为true，当检测某个元素是否存在时，则经过同样的方法获得k个索引值，只有比特数组中这k个位置都已经被置为true才能说此元素\_可能\_存在，若k个位置中的值存在false则可以\_肯定\_的说被检测元素肯定不存在。为保证上述性质（即k个位置中的值不全为true则可判断为不存在），BloomFilter不允许删除元素。而对于误报（即元素不存在，但是对应的k个位置中的值被其他存在的元素置位而判定为存在）的计算涉及到几个参数，m--比特数组的大小，n--元素的数量，k--元素被hash的次数，给定m和n则可以确定误报率最小的k值，`k=(m/n)*ln2`。具体的误报率公式为：`ln(p) = -k*ln(2)`。
leveldb中的实现，首先通过给定参数m/n的值确定hash次数k，再实际计算k次hash值时，首先使用hash函数计算元素的hash值，然后利用此hash值再计算k个不同的索引值，这是一种比较高效的做法，具体参见[Less Hashing, Same Performance: Building a Better Bloom Filter](https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf)。

**6. Logging**

里面的判断溢出方法比较简洁，摘录如下。
```
int delta = (c - '0');
static const uint64\_t kMaxUint64 = ~static\_cast<uint64\_t>(0);
if (v > kMaxUint64/10 ||
	(v == kMaxUint64/10 && delta > kMaxUint64%10)) {
        // Overflow
     return false;
}
v = (v * 10) + delta;
```
**在Posix\_Logger中值得借鉴的地方是，由于不能预先知道Logv接受的参数的长度,源码中首先使用栈上的空间来试图存储接受的参数，如果空间不够再动态分配一个较大的buffer来进行存储。栈上能够分配的最大内存是系统预先定义的，其中的内存分配也由系统完成，而且是一块连续的内存区域，而堆上的内存则是由链表进行管理，则堆上的内存是由很多个不连续的内存块链表组成，为什么说访问栈上的内存是比堆上的内存快？参考[这里](https://www.zhihu.com/question/30145142), 在系统配有高速缓存的情况下，栈上的内存是当前代码段最常访问的内存区域，因此其位于高速缓存的概率则非常高，因此访问就非常快，另外预先设置较大的栈的最大内存是没有用的，因为一般高速缓存的大小是比较有限的。**

**7. Random**
Random类生成伪随机数的方法使用的是**线性同余法**，具体的计算公式为`seed=(seed*A + C)%M`, 其中A、C、M都是常数（一般取为质数），当C=0时，则称为**乘同余法**。选用参数的原则是，M尽可能的大，对于32位的机器，M=2^31-1, A=7^5=16807时可以取得最佳效果。代码中将`product`对M的模运算转换为了位移运算和与运算，证明过程见[这](https://m.aliyun.com/yunqi/articles/2271?spm=0.0.0.0.q8EpvV)。

**8. Env**
* **PosixRandomAccessFile**
使用pread来实现文件的文件访问，由pread手册知道，pread和pwrite允许多个线程操作同一个文件描述符，且多个线程之间的文件偏移不受其他线程的影响。
使用mmap来实现。这篇[文章](http://www.cnblogs.com/huxiao-tee/p/4660352.html)比较好的解释了mmap的原理。mmap就是将进程的一段连续虚拟地址空间与磁盘文件地址进行映射，用户就可以使用指针来寻址文件中的内容而不需要调用read,write等系统调用，两个进程映射相同的文件则可实现进程之间通信。**mmap使用的进程虚拟地址空间在stack和heap之间的空闲区域**。linux内核使用vm\_area\_struct来存储一个虚拟内存区域的信息，整个进程虚拟内存空间使用多个vm\_area\_struct来表示其中各个不同的存储区域，该结构中包含一个vm\_ops指针用来引出对该结构体所对应的虚拟内存区域操作的所有系统调用函数。
>mmap的具体步骤：
>	- 进程启动映射
>调用用户层面的库函数`mmap(void*, size_t, int, int, int, off_t)`在stack和heap之间寻找一段地址连续的空间并建立相应的vm\_area\_struct。将此结构插入到虚拟地址区域链表中。
>	- 实现文件磁盘地址和进程虚拟空间地址的映射
>首先通过文件描述符在内核中找到与该文件对应的文件结构体，通过此文件结构体调用内核函数`int mmap(struct file *filp, struct vm_area_struct *vma)`， 内核mmap通过文件系统inode定位到文件的物理磁盘地址。
>	- `remap_pfn_range`建立页表实现虚拟区域地址和文件磁盘地址之间的映射。此时文件并没有读入到内存中。
>	- 进程读取引发缺页异常
>进程开始读写文件，由于文件内容没有写入内存会引发缺页异常。接着内核会发起调页过程，首先在交换缓存空间（swap cache）中寻找需要访问的内存页，如果没有则调用nopage函数将磁盘文件读取到内存中，之后进程对内存进行读写，系统会自动回写脏页（被修改的页面）到磁盘中。
>
>常规文件读写的具体步骤：
> - 进程发起文件读写，内核通过查找进程文件符表，定位到内核已打开文件集上的文件信息，从而找到此文件的inode。
> - inode在address_space上查找要请求的文件页是否已经缓存在页缓存中。如果存在，则直接返回这片文件页的内容。
> - 如果不存在，则通过inode定位到文件磁盘地址，将数据从磁盘复制到页缓存。之后再次发起读页面过程，进而将页缓存中的数据发给用户进程。
> **常规文件读写操作为了读写效率和保护磁盘会现将磁盘数据写入到内核空间的页缓存中，当用户进程读取时需要从内核空间的页缓存拷贝到用户进程空间的内存中，因此需要两次拷贝。而mmap只有在真正读写文件时引发缺页异常时直接将文件内容拷贝到进程空间对应的内存中，因此只需要一次拷贝。因此mmap的效率更高。**
> 
> 用户态和内核态的区别
>所谓用户态和内核态指的是进程拥有不同的权级，比如在linux中内核进程的权级为0, 而用户进程的权级为3, 权级不同则能够使用的系统资源就会有所不同，但是有时用户进程需要使用一些受到限制的资源则需要进行用户态到内核态的转换。转换的方式有以下三种。
> - 使用系统调用
>`mmap(void*, size_t, int, int, int, off_t)`, 这是一个库函数，该库函数将会产生一个中断，如linux的`init 80H`中断，通过系统预留的中断和相应的系统调用函数mmap对应的系统调用ID来在中断处理函数来调用系统调用`mmap(struct file, struct vm_area_struct*)`。当然内核在调用系统调用时肯定会检查该调用的合法性。
> - 使用异常
>比如缺页异常会触发内核调用相应的系统调用来完成磁盘到内存的拷贝。
> - 使用外围设备的中断
>当设备完成用户的请求后则会向CPU发送中断信号，这时CPU保存当前的环境处理中断调用中断处理函数。
>这三种方式本质上都是使用中断来进行用户态和内核态的切换。
* **PosixLockTable**
这个类使用`set`来保存已经被上锁的文件名，源码注释中说`fcntl(F_SETLK)`只能提供进程之间的保护，而对于同一个进程内多个线程对文件的同时访问则没有提供相应的保护，因为同一个进程的所有线程对文件锁是共享的。因此需要互斥锁来保证多个线程之间的同步。因此外部使用`LockFile`之间应该使用`PosixLockTable::Insert`来检测是否此文件的锁已经被其他线程锁住。
> The  threads  in  a  process  share locks.  In other words, a multi‐  threaded program can't use record locking  to  ensure  that  threads don't simultaneously access the same region of a file. (Reference fcntl@man)
* **PosixEnv::gettid()**
这个函数返回的线程ID使用的是`pthread_self`, 与创建该线程时返回的线程ID，但是这个ID只在当前进程中是独一无二的，且当线程退出后其线程ID会被重新使用，不利于程序调试时追踪线程的动作。可以考虑使用系统调用`::gettid()`，Linux特有，因此不具有可移植性。
* **PosixEnv::BGThread()**
在循环中不断取出队列中的任务执行，如果为空则进入睡眠状态，直到`PosixEnv::Schedule`插入新的任务使用条件变量将其唤醒。基本上是一个生产者消费者模式，使用互斥锁进行任务插入、提取的同步。

**9. SkipList**

源码中的跳表的实现与普通的实现没什么差别，值得注意的地方是如何实现线程安全，对于写入操作需要额外的同步机制，如使用互斥锁; 而对于并发的读操作，保证线程安全依赖于两点：源码中不对跳表中的元素进行删除操作，如果存在删除操作，读操作可能会读到被删除的元素; 使用内存屏障来保证指针修改的一致性（使用AtomicPointer）。能够影响并发读操作的只有`Insert`操作，写入操作分为更新高度和更改新节点与前后节点的链接。不管readers获得高度是之前的还是更新之后的，都可以对链表进行访问; 当对链表遍历时相应节点与新的节点的链接修改要么完成，要么没有完成，前者将会使得readers找不到新的节点，后者则可以使readers找到新节点，两种情况readers都可以获得**合适**的结果。为什么说是**合适**的结果？因为readers是相对于`Insert`操作是并发的，读不到新的节点只能说readers来早了，读到新节点则说明readers相对于`Insert`操作来迟了，这两种情况都是可接受的。除非人为的使用外部同步机制保障读取操作在`Insert`之后，这样肯定能读到新节点。（**更新：在skiplist中Node的分配内存地址是4byte或者8byte对齐，因此指针的赋值是原子操作。另外使用Atomic Pointer的作用可能是，在单写多读的情况下，只要写入完毕，正在读取的线程就会直接从内存中读取到最新的节点，这一点可以看skiplist的插入操作。但是反过来想，如果不使用Atomic Pointer又会怎么样？我的理解是SkipList::Insert中更新节点各级指针的顺序是从最低位置往最高位置，假如不使用AtomicPointer，那么这些节点中各级指针更新写入内存的次序可能与程序中执行的次序不相同，考虑这样一个场景，有一个写线程正在插入节点N，此时若有一个读线程捕获到了节点N的第i级指针，但是在后面没有找到要找的节点，因此开始从节点N的第i-1级节点继续寻找，但是可能第i-1级指针在内存中存放的地址还没有及时更新，因此会导致错误。这个可以仔细研究`Insert`操作中最终改变前缀节点的方式。**）

**10. IteratorWrapper**

对Iterator做了包装，私有数据对Iterator的key和valid都进行了缓存，注释中说这样做是为了避免调用虚函数以及cache locality。的确虚函数的调用涉及到实际对象信息的获得以及在相应的虚函数表(vtable)中查找对应的虚函数地址，与普通函数调用相比比较耗时。


#### 3.2 Table

![table](/images/table.jpg)
从图中可以知道table主要由data block、meta block、meta-index block、index block、footer组成。其中data block主要装载key-values，从最左边可以看到一个block的组成，每一个entry由shared(与前一个entry中的key相同前缀的长度)、unshared(除去与前一个key共有前缀长度的余下长度)、val\_len(value的长度)、key\_delta(key的非共享部分)，为了维持这种高效的压缩策略需要记住一些起始点（**为什么不只使用一个起始点而是使用多个起始点？出于效率和可靠性的考虑，在搜索key时可以根据起始点的key值进行二分法搜索获得key所在的起始段，另外如果依赖于某个起始点的bad-entry损坏了，那么其后的所有依赖于这个bad-entry也将无法解析，使用多个起始点的则可以保证在bad-entry后的下一个起始点之后的entry不会破坏**），这些起始点的key值不与前一个key值共享，block的最下方维持一个起始点数组和数组的长度。接下来是meta-block, 源码中只实现了filter-block作为meta-block来快速的检查某个key是否存在, filter-block的最底下1byte为一个base，用来表示哪些data-block将会被用来生成一个filter，比如正在生成第i个filter，那么包含的**偏移范围**为`[ i*base ... (i+1)*base-1 ]`， 任何data-block的**起始偏移**在这个范围内的都将被用来生成第i个filter，在1byte-base的上面是一个filter-offset数组以及数组长度，用来记录各个filter在filter-block中的位置。在解释table的各种index部分之前，介绍另外一个数据结构BlockHandle，这个结构中存储了对应block在table中的起始偏移位置以及长度。metaindex-block中的key为每个meta-block的名字，value为与此meta-block相关的BlockHandle。index-block中的key大于等于此block的最后一个key值但是小于下一个block的起始key值，方便key值查找（参考Comparator的实现）。接下来则是table的footer，里面包含metaindex-handle（指示meta-index block的BlockHandle）、index-handle（指示index block的BlockHandle）这两部分都使用变长编码，使用padding凑成40bytes, 最后一个固定的8bytes是一个magic number用来判断table是否被损坏，整个footer的大小是固定的48bytes。

**1. BlockBuilder**

这个模块用来对输入的key-value对进行编码，内部使用一个string来存储Block内容，当整个Block的大小超过规定大小时进行`Finish`操作，使用slice包装Block内容作为返回值。

**2. Block**

此模块主要用来访问block中的内容，Block中内嵌一个Iterator类Iter。
* **Iter**
**Seek(const Slice& target)**
由于block中的key是有序的，因此使用二分法进行搜索。首先通过二分法读取各个起始点的key值即可判断出key值在哪个起始段内，从所在起始段进行线性查找，直到找到大于等于target的值，或者遍历完整个起始段。
**ParseNextKey()**
根据Block中的entry格式进行解析。
**Prev()**
首先获取在当前entry之前的起始点，然后使用ParseNextKey来获取当前entry的前一个值。
* **FilterBlockBuilder**
此模块的作用主要是首先通过输入的block-offset确定filter-index， 既而对输入的key产生相应的filter，并按照filter-block的格式进行排列，使用slice来引用整个filter-block内容并通过Finish()传递给上层模块。使用时应该遵循StartBlock->AddKey->Finish的调用顺序，且在同一个filter-index中的data-block应该连续的输入key以产生对应的filter。
* **FilterBlockReader**
首先根据block的偏移计算对应于哪个filter，然后由此filter查看该key是否存在于这个block中。
* **Formate**
这个模块中主要实现BlockHandle、Footer的编码与解码。其中值得注意的是ReadBlock这个函数，在table中每个block后面还跟随着type（是否压缩）、crc（block中的data部分的crc校验）。首先分配整个block（包含type、crc）的内存空间用来存储整个block，接下来进行crc校验，若type显示为压缩还需要进行解压，将解压后的内存地址传送到result中，若没有压缩则直接将存储block的内存地址传送到result中。
* **Iterator**
其中的成员函数大部分是纯虛函数，主要是作为一种接口，其中已经实现的部分是注册对象析构时需要调用的回调函数，这些回调函数主要使用链表进行维护。
* **IteratorWrapper**
此类中包含一个Iterator*以及相应的key值和valide值，使用这个包装类主要是为了防止频繁调用虚函数以及获得较好的cache hit。
* **TableBuilder**
内含一个Rep类用来包含Block相关的参数。

**3. ChangeOptions**

* **ChangeOptions**
这个函数主要用来改变options，由于rep中的BlockBuilder中的options是指向TableBuilder中的Options的指针，因此在构建table的过程中options更新后会被立即应用到当前BlockBuilder中。options中的有些参数不允许修改，如Comparator。
* **Add**
Add实现将key-value对加入到table中，首先判断现在的位置是不是block的边界，若是则需要获得上一个key与当前key的分界值keyN，使用keyN和上一个block的BlockHandle作为index-block中的key-value对加入到index-block中。接着将key值加入到filter-block中，然后将当前的key-value对加入到data-block中，若当前的block大小已经达到规定的值则将block写入磁盘。
* **WriteBlock**
首先调用BlockBuilder::Finish()将当前block内容按照格式排列好后的结果使用slice传送出来，根据options中的压缩选项决定是否对block的数据进行压缩， 然后再调用WriteBlock进行处理。
* **WriteRawBlock**
设置当前block的offset以及长度并存储到BlockHandle中，**注意这个BlockHandle会在下一次调用TableBuilder::Add或者TableBuilder::Finish时才会被写入index-block中**。接着将当前block的内容append到指定文件，然后加入此block的trailer区域，包括压缩类型type和block内容的crc32编码。最后记录下一个block的起始偏移地址。
* **Flush**
调用WriteBlock将当前block内容写入文件，调用文件Flush操作将文件刷入磁盘。开始将此block加入filter中。
* **Finish**
首先将filter-block-builder产生的block内容写入文件，接着将filter-block所对应的名字作为key和其BlockHandle作为value写入metaindex-block中，通过writeBlock写入文件。接着写入index-block，此时最后一个data-block的BlockHandle还没有加入到index-block中，将其加入后再将index-block写入文件。最后加上Footer并写入文件，整个table写入文件完毕。

**4. TwoLevelIterator**

这个Iterator类能够对整个table中的blocks中的key-values进行遍历。其中index\_iter用来遍历index-block，data\_iter用来遍历单个block。具体的思路首先使用index\_iter获得key值所在block的偏移，即BlockHandle然后使用此BlockHandle来初始化data\_iter,再由data\_iter获得key的具体位置。对于空的block，获得的data\_iter是invalide的（参考block.cc中Iter::ParserNextKey的实现）。

**5. Table**

Table的带参构造函数是私有成员函数，类中提供了公有的static函数Open来打开文件，若打开文件正常且table的meta信息读取正常则构建一个table对象并通过参数中的二级指针返回调用方。（**这样设计的优点符合这样的语义，磁盘中的table是完好的则会构建table，否则不会构建相应的对象**）
* **BlockReader**
此函数根据index\_value(实际装载着待读Block的BlockHandle)来获取待读Block的Iterator。首先检查options中是否设置Block可缓存，若是则在缓存中查找该Block是否存在于缓存中，查找的key值为当前的cache\_id和Block的offset的组合，若存在则直接将缓存中的Block的地址取出，若不存在与缓存中，则从磁盘读取到内存并将其缓存。如果没有使用Block缓存则直接读取磁盘。接下来使用Block中的内嵌类Iter构造Iter对象iter。在返回之前首先注册Block析构时的函数，如果是没有缓存的则需要手动析构掉动态分配的内存，如果是有缓存则需要释放相应Block的引用。
* **InternalGet**
此函数的参数中含有一个函数指针用来处理查询到的key-value值，查询的过程分为两步，第一步使用filter查看是否存在于block中，若存在则读取对应的Block并返回相应的Iterator，查询key-value对，如果存在则使用函数指针处理key-value。
* **ApproximateOffsetOf**
首先获取index-block的Iterator iter-index并寻找key值所在的Block，如果iter-index是invalide则返回meta-index-block的偏移量，否则解析对应Block的BlockHandle，解析成功则返回Block的偏移量，否则则返回meta-index-block的偏移量。
* **MergingIterator**
此类将n个Iterator集合在一起进行遍历，通过统计当前各个iterator中的最小值和最大值来实现前进或后退操作，但是若某个key值存在于m个Iterator所指的范围则在遍历过程中会出现m次。


#### 3.3 Log
log-file中包含一连串的32KB的blocks，每个block包含一连串的records，records的格式如下：
```
   block := record* trailer?
   record :=
    checksum: uint32    // crc32c of type and data[] ; little-endian
    length: uint16      // little-endian
    type: uint8     // One of FULL, FIRST, MIDDLE, LAST
    data: uint8[length]

```
如果一个Block中剩余的空闲字节数小于等于6bytes则进行0字符填充，record从下一个Block存储。
**1. Writer**

主要功能是根据上面log-file的格式将record写入文件。
* **InitTypeCrc**
模块在初始化时会预先计算好type的crc校验，有利于提高计算type和data部分的校验和的速度。
* **AddRecord**
根据record的大小以及当前block剩余的空间决定record不同块的type并可能写入不同的Block。值得注意的是在写入record时，使用的`do while`的方式，因此即使record为空，也将会被写入一次。
* **EmitPhysicalRecord**
根据record的type和之后的payload计算crc校验和，并写入文件。

**2. Reader**

主要功能根据log-file格式读取record，在读取过程中产生的错误信息将会通过内部的Reporter类来传送故障信息。其中读取的文件是SequentialFile，即顺序读取。
* **SkipToInitialBlock**
根据读取的起始位置确定在哪个block中，并将文件的读取指针seek到相应的block起始位置。
* **ReadPhysicalRecord**
Reader中的backing\_store实际上是一个kBlockSize的缓冲区，buffer\_则是对这个缓冲区的引用，首先检查buffer\_长度是否还能装得下一个Header，若能则直接解析header，否则判断是否已经读到文件末尾，若没有且buffer中还有空余空间则表示上一个record已经读取完毕，接着从文件读取一个kBlockSize的数据存储到backing\_store中，过程中出现的错误情况都会经过reporter进行记录并返回相应的错误type。若一切顺利则返回所读record的slice以及对应的type。
* **ReadRecord**
首先调用SkipToInitialBlock将文件的读取位置seek到initial\_offset所在的block起始位置处，若在initial\_offset后的block部分无法装得下一个header则将文件读取位置设置为下一个block的起始位置。如果设置了resync标志，则会跳过当前type为kMiddleType的reocord部分直到读取到下一个record。 `prospective_record_offset` 记录的是每个record起始位置，最后此函数返回时会将这个值赋给`last_record_offset_`。


#### 3.4 Version
每一个Version用来记录每个level中的一些文件的信息，这些Version又组成一个VersionSet， 最新的Version被称为**current**, 比较旧的Version是为了保证一些Iterator访问的内容的一致性。在类中Version的构造函数和析构函数都是私有函数，因此其他用户不能直接构造Version对象，其内部声明了Compaction和VersionSet为其友元函数，因此Version对象的构造和析构应该是通过这两个类来调用。

**1. Version**

* **FileMetaData**
结构体中存有对应文件的引用个数，allowed\_seeks（**这个作用是什么？**从后面的UpdateStats的实现和被调用可以看出当一个文件被访问了allowed\_seeks次时将会作为下一次用来压缩的文件，这是leveldb的一种自动压缩机制）, file\_number, 文件大小，以及文件中key的范围。
* **~Version**
Version的析构函数，由于Version在VersionSet中是以双链表进行管理的，析构时需要从链表中删除。另外还需要对version所映射的各层files进行解引用，如果文件引用变为0，需要将文件对应的结构体（FileMetaData）删除。
* **FindFile**
根据输入key和files（files必须是有序的），利用二分法寻找大于等于key的files（使用file的最大key进行比较），返回文件在files中的序号。当key不在文件的所含键值范围之内时，则会返回files的个数。
* **SomeFileOverlapsRange**
根据输入的key的范围，检查是否文件列表中所含的键值范围与输入key的范围重叠。如果文件列表所涵盖的键值范围是有重叠的或者不是有序的，则需要对每个file判断是否与输入key的范围重叠；否则可以使用二分查找首先定位输入key的下限可能在哪个文件，再判断上限是否在该文件所含键值范围之内或者之后，则可判断是否有重叠。
* **LevelFileNumIterator**
这个Iterator用来遍历某个Version的某个level中所包含的文件信息， key()获得的是当前文件中的最大key值， value()则包含文件号(file\_number)和文件大小，各8bytes，以slice输出。内部的value\_buf\_则是16Bytes的value缓冲区，由于在const成员函数中修改因此使用mutable进行修饰，同时也说明这个Iterator在多线程环境中需要额外的同步机制。
* **GetFileIterator**
参数中的arg为TableCache, 输入的file\_value包含文件号（file\_number）和文件的大小，TableCache调用NewIterator转而使用FindTable找到相应的文件并根据该文件生成Table，再使用Table的NewIterator生成遍历此Table的Iterator。
* **NewConcatenatingIterator**
根据输入的level首先生成LevelFileNumIterator, 注意这种迭代器的value值为文件的文件号和文件大小，也是GetFileIterator的参数，使用NewTwoLevelIterator则可以获得对level中所有文件中的key-value对进行遍历，在此种迭代器没有调用各种Seek函数之前，对应的文件还没有打开，采用了一种lazy-opening方法。
* **AddIterators**
对于level-0的文件，所含的key的范围可能有重叠，每个文件需要一个单独的Iterator进行遍历；而对于大于0的level，文件之间是有序排列且不重叠，则使用NewConcatenatingIterator对当前level中的所有文件进行遍历就够了。
* **SaveValue**
作为TableCache::Get中的函数指针传入，如果得到的key相等，则查看value的类型，如果是kTypeValue则将value的值保存。
* **NewestFirst**
根据文件号判别文件的新鲜程度，文件号越大则越新鲜，因此在排序中是以文件号降序排列。
* **ForEachOverlapping**
输入的参数有一个处理函数func。首先在level-0中查找user-key,由于level-0中的文件中所涵盖的键值范围是无序的且有可能重叠，因此首先找到那些可能含有user-key的文件，并根据文件的新鲜程度进行排序，先从最新鲜的文件开始使用func进行处理，func返回false则返回，否则继续处理较旧的文件。如果level-0中的文件已经遍历完，则在其余level中由低到高进行搜寻user-key并使用func进行处理直到func返回false或者所有的文件已经遍历完。
* **Get**
实现思路与ForEachOverlapping类似，处理函数是SaveValue将获得value值进行保存。
* **UpdateStats**
将输入参数GetStats中包含的文件的allowed\_seeks递减，若小于等于0且现在没有需要压缩的文件，则将当前文件置为待压缩的文件。（**allowed\_seeks的作用见上一个粗体字**)
* **RecordReadSample**
调用ForEachOverlapping记录第一个包含internal-key的文件，如果两个或者两个以上的文件包含这个key则需要调用UpdateStats设置需要Compaction的文件以及level。
* **Unref**
对当前的Version进行解引用，如果当前的引用数已经降到0则delete当前对象即执行析构函数。
* **OverlapInLevel**
查看当前level中的文件所含有的键值范围是否与输入的key值范围重叠。
* **PickLevelForMemTableOutput**
选择哪一层适合放置MemTable的Compaction文件。config::kMaxMemCompactLevel的设置为2,因此只能在0～2层之间放置MemTable。源码中解释选择最大值为2的原因是，从level-0到level-1的压缩比较耗时同时可以避免过多的manifest文件操作，为什么不选择最大的层数？选择最大的层数的话可能会导致较低层数也会有相同key值范围的文件，比较浪费磁盘空间。选择层数的思路是：如果下一层有重叠的键值范围则选择当前层放置MemTable的Compaction文件，另外如果grandparent-level(即level+2)与指定键值范围重复的文件大小总和超过了最大值则也会选择当前level层。（**为什么会设置最大的kMaxGrandParentOverlapBytes这个限制？**）应该是出于Compaction效率的考虑，如果是放在level+1层，则和level+2进行合并则会比较耗费时间。
* **GetOverlappingInputs**
在level层中找到所有与输入key值范围重叠的文件。当level为0时，由于各个文件之间可能有重叠，如果新加的文件的键值范围则将输入key值范围进行扩展到当前文件的MinKey/MaxKey，**为什么需要扩展输入键值范围？**这个得看这个函数实际上在哪个环节上使用。当进行Compaction时，当给定键值范围，需要找出level中所有与其重叠的文件且没有被选中的文件所涵盖的键值范围与选中的文件所涵盖的键值不相重叠，这对于level>0的文件只需要依次检测每个文件是否与给定键值范围是否重叠，但是由于level-0的文件之间可能有重叠则需要将键值范围扩充为文件的键值范围。compaction的初衷就是缩小无效key-value对所占的空间，如果不扩充输入键值范围，那么有的键值范围将会在此次compaction进入level-1,但是在其他没有参与compaction的文件中可能存在相同的键值范围，则这些键值将继续留在level-0等待下一次compaction，这样就显得这次compaction不够干净。
* **DebugString**
打印Version中各level中每个文件的文件号、文件大小、最小key和最大key。

**2. VersionEdit**

用来管理需要压缩的键值、删除的文件和新的文件信息。
* **EncodeTo**
对VersionEdit进行序列化。
* **GetInternalKey**
输入字符串带有长度前缀，获取内部字符串内容后再解析出InternalKey。
* **GetLevel**
从字符串中解析使用varint32编码的level。
* **DecodeFrom**
对输入的字符串进行反序列化为VersionEdit，根据不同的tag进行不同成员的解析。
* **DebugString**
将VersionEdit中的成员数据转换为字符串。

**3. VersionSet**
使用双链表维持Version的管理。
* **Builder**
维护着每一层删除的文件和加入的文件，其中加入的文件的顺序是按照最小key值进行排序，如果最小key值相等则使用文件号进行排序。
*Apply*: 根据VersionEdit设置每一层进行compaction时应该从哪个key开始。接着将VersionEdit中各层应该删除的文件号加入删除的文件set中。接着将VersionEdit中的新文件加入进来，值得注意的是新文件的allowed\_seeks的设置，根据源码中的注释表达的语义是，该文件已经被seek了allowed\_seeks次所花的时间与将该文件compaction所花的时间是几乎相等的。compaction的时间是比较长的，但是随着文件的访问次数的增加，其时间与文件被访问的总时间相比将会显得不是那么大。
*SaveTo*： 将base\_中已有的文件和VersionEdit中新加入的文件进行合并，将合并的文件存入参数中的Version指针中。
*MaybeAddFile*: 如果在删除列表中则不加入输入文件；如果level>0，检测是否与当前level中已有文件的key值范围重叠; 一切正常则增加文件的引用并将其加入文件列表中。
* **AppendVersion**
将输入的v设置为当前的Version值current\_, 增加对v的引用并将其插入到双链表中。
* **Finalize**
计算Version中哪一层将会作为下一次的compaction。其中对于第0层文件数越多则越会在下一次被compacted，而其他层则是相对于本层允许的最大字节数，所有文件所含的字节数越大则越可能被compacted。区别对待第0层的理由是，每次读取第0层时需要将各文件进行联合，文件数越少则读取更快；另外对于写缓存比较大时，level-0的compaction次数越少越好。**这里的写缓存应该是memtable和immutable-memtable， 前者用于缓存用户直接写入的数据，后者用于compaction成Table文件**。
* **WriteSnapShot**
将当前Version, 即current\_以及compact点存储到VersionEdit对象中，并将VersionEdit进行序列化作为record写入log文件。
* **LogAndApply**
	1. 设置edit的各种参数，然后使用当前的version，即current\_， 和version\_edit合成下一个version v；
	2. 使用Finalize设置v应该接下来在哪一层进行compaction。若是第一次调用此函数，则会对当前的VersionSet进行快照写入，也就是使用WriteSnapShot对当前的current\_和compact点写入VersionEdit对象并进行序列化写入log文件。
	3. 由于写入log文件比较耗费时间，因此将lock进行解锁让其他线程获得锁。将输入的Edit进行序列化写入log文件中。
	4. 如果在第二步中创建了新的MANIFEST文件，则需要将CURRENT文件中所包含的文件名改为新的MANIFEST文件名。写完log文件后再尝试获得lock。
	5. 将Version v设置为当前Version， 即current\_指向v。
* **Recover**
	1. 从CURRENT文件中读取到当前的MANIFEST log文件， log文件中每一个record都是VersionEdit序列化后写入的。然后依次读取log文件中的VersionEdit并将VersionEdit应用到builder中。更新next\_file\_number，必须比所有的文件的文件号大。
	2. 将builder经过Finalize后保存到新的version中，并将此Version设置为当前的Version，更新VersionSet中的各种文件序号。
	3. 检查MANIFEST文件是否可以复用。
* **ApproximateOffsetOf**
查找输入key所在的位置相对于第一个文件的位置的偏移。
* **AddLiveFiles**
搜集所有version中各层所有文件的文件号。
* **MaxNextLevelOverlappingBytes**
统计当前version中，每一层的各个文件与下一层文件的重叠的文件大小的最大值。
* **GetRange**
获得输入所有文件的键值范围。
* **GetRange2**
调用GetRange计算覆盖两个输入文件集的key值范围。
* **MakeInputIterator**
输入为Compaction c，此函数是在c中两个input的文件列表上生成MergingIterator用于遍历所有的key值，其中对于level-0则为每个文件生成一个Iterator然后在参与merge生成MergingIterator。
* **PickCompaction**
触发Compaction的两种方式：#1某一层文件数过多或者文件大小超过所在level规定的大小，#2某个文件被seek的次数操作了预设值。源码实现时优先检查是否是前者，若是则寻找在compact\_pointer\_[level]之后的文件进行compaction，若没找到则选择该level中第一个文件。对于后面一种情况则直接使用file\_to\_compact\_中记录的文件作为compaction的输入文件。如果处于level-0还需要调用GetOverlappingInputs将所有与选定文件重叠的文件作为该level的compaction输入文件。
* **SetupOtherInputs**
设置Compaction的第二部分输入文件。
	1. 获得level+1中与level中文件相重叠的文件，然后检查是否可以将level中的文件个数扩展，而又不会导致level+1的文件数增加，这样做的好处是一次可以将level中更多的文件与level+1中的文件进行compaction操作，如果不这样做，那么每次level（> 0）中就只会有一个文件，因此需要compaction的次数就会更多，当然一次Compaction输出文件的大小得在预设值之内。
	2. 计算grandparent-level中与compaction中key值范围重叠的文件数, 在Compaction::ShouldStopBefore中会用到。
	3. 设置下一个compaction点为level层参与此次compaction文件中最大的key值。随后设置c中该level的compact点。
* **CompactRange**
根据输入的key值范围来选定level中输入的文件，同时对level>0的情况应该避免一次参与compact的文件过多，随后设置Compaction中level+1中输入的文件。

**4. Compaction**
压缩即将level\_中的一个文件与level\_+1中与这个文件所涵盖键值范围重叠的所有文件进行压缩生成新的level\_+1。
* **IsTrivialMove**
判断是否处于level+1的输入文件为0以及level的文件数是否只有一个，如果是表明level+1没有与处于level的输入文件重叠的键值，还得判断处于level的输入文件是否与grandparent（即level+2）的相重叠的文件大小总和是否超过规定的值，如果没有才表明只需要将处于level的输入文件移动到level+1层就可以。
* **AddInputDeletions**
将处于level和level+1的输入文件加入到VersionEdit的删除列表中。
* **IsBaseLevelForKey**
判断输入的key是否存在于大于等于level+2的层级中。
* **ShouldStopBefore**
统计输入key在grandparent-level中找到之前涵盖的文件大小总和，如果超过规定的值，表示需要在此key之前停止compaction，重新打开一个新的文件用于compaction后续的输出。这样保证compaction在level+1层产生的文件不与level+2的文件重叠太多。
* **ReleaseInputs**
若内部用于操作的version指针是有效的，则对相应version进行解引用。* **MemTable**
将析构函数设置为private，则用户无法直接使用，只能调用`Unref`来释放资源，这样做可以防止用户忘记`Unref`。MemTable中维持一个以key-value为基本元素的skiplist，以及相应的内存池(提供给key-value编码后的存储空间以及skiplist分配节点所需空间，这些内存在memtable引用数为0引发析构时随着arena析构而释放掉)，其中key的结构为`UserKey|(SequenceNumber<<8+ValueType)`，在实际key之间的比较中UserKey以升序排列（由小到大）而后续的SequenceNumber和ValueType以降序排列（由小到大）。
* **GetLengthPrefixedSlice**
每个entry中的key、value部分存储在skiplist中时是以内容长度+内容的形式存储，其中内容长度为Varint32的形式编码。
* **MemTableIterator**
实际上是对Skiplist::Iterator的简单包装。
* **Add**
根据`key_size|key bytes|value_size|value bytes`的格式进行编码存储到skiplist中。
* **Get**
首先在skiplist中寻找大于等于key的项，然后比对UserKey是否相同，**为什么不用比较SequenceNumber的大小？**若此时的UserKey部分是相同的，并且假如Skiplist中存在多个相同UserKey但SequenceNumber不同的entrys，那么由于Skiplist::Iterator::Seek获得的是大于等于要查找的值，而SequenceNumber越小在比较上是被认为越大（或者说属于相同UserKey的SequenceNumber是以降序排列的），因此此时获得的SequenceNumber是小于等于key中的SequenceNumber的，为什么这么设计得看SequenceNumber的用途。（从后面的WriteBatch中，MemTableInserter中的SequenceNumber是递增的，即对相同的UserKey的更新，seq越大表示更新越新，因此Get得到的是， 在小于等于key中sequence的范围内，对UserKey的最新更新）。接下来再查看ValueType是普通还是删除，若是普通则将后续带有length-prefixed的value解码输出，否则将状态置为NotFound状态并正常返回）。

**5. FileName**

leveldb中共有LogFile, DBLockFile, TableFile, DescriptorFile, CurrentFile, TempFile, InfoLogFile共7中file类型，通过dbname和各种类型file\_number可以获得对应的文件名，文件名中以dbname/作为前缀，然后加上一些后缀。其中DescriptorFile被标识为MANIFEST文件。
* **SetCurrentFile**
将CurrentFile**指向**与descriptor\_number相对应的DescriptorFile(其文件名的前缀为MANIFEST)，所谓**指向**就是将DescriptorFile的文件名写入CurrentFile中。在实现中首先将DescriptorFile的文件名写入TempFile，如果写入成功再将TempFile更名为CurrentFile，更名时旧的CurrentFile将会被删除，如果写入TempFile失败则不会影响CurrentFile的内容。

**6. TableCache**
内部维护一个Cache*, TableCache在初始化时通过输入的缓存容量entries数来初始化Cache*。
* **TableAndFile**
此结构体中包含两个分别指向RandomAccessFile和Table的指针。cache中存放的entry（即key-value对）为`key=file_number, value=TableAndFile*`。
* **DeleteEntry**
TableAndFile的析构函数，分别删除结构体中的table和file，此析构函数在cache删除相应entry时来调用删除entry的value（即TableAndFile*）部分。
* **UnrefEntry**
根据参数中的cache*和handle*对handle*所对应的entry进行Release操作，在Iterator析构时调用。
* **FindTable**
Cache中的key值是一个64bit的file\_number，通过此file\_number查询缓存中是否有相应的entry存在，若存在则将返回的Handle地址通过参数返回，否则开始读取相应的TableFileName，若读取失败则读取SSTTableFileName，读取成功后在此file上构建一个Table，根据获得的Table和File来构建TableAndFile并将其作为value, file\_number为key值存储在cache中，由于entry的value部分都是动态分配的，因此在插入到cache中将value对应的析构函数一并存储。在这里析构函数为DeleteEntry。
* **NewIterator**
由输入的file\_number和file\_size来定位对应的文件，并根据此文件建立table， 此table的地址通过参数返回，整个函数返回一个用来遍历整个文件的Iterator。在实现上使用FindTable获得相应的Handle*, 然后通过Handle*获取对应的Table*, 再由此Table*生成遍历Table的Iterator， 由于在FindTable中不管起初在缓存中找没找到对应的entry，Iterator的获得都会对cache中相应entry进行引用，因此在Iterator析构时还需要调用UnrefEntry对相应的entry进行解引用。
* **Get**
由FindTable获得Table*, 再调用Table中的InternalGet函数，该方法首先检查index-block是否可能存在相应的block，在根据block-handle信息在filter中查看该key是否可能存在于该block中，若是则读取该block并进行查找，若找到调用处理函数saver来处理key-value对。由于FindleTable会对相应的entry进行引用，因此最后需要cache对相应handle进行Realease操作进行解引用。
* **Evict**
删除cache中与file\_number对应的entry。

**6. WriteBatchInternal**

此类包含操作WriteBatch的静态方法，用来被Implementation使用而不是client使用。
**WriteBatch**
此类的作用是将多次对数据库的更新汇聚成对db的一次原子式的更新，且保证更新的顺序。对const的成员函数进行调用时，多线程不需要加锁（也就是说在多线程编程中尽量调用const方法减少锁的使用），非const成员函数则需要额外的同步机制保证正确性。WriteBatch存有一个记录这些更新的数据结构，使用string装载，格式如下。
```
WriteBatch::rep_ :=
    sequence: fixed64
    count: fixed32
    data: record[count]
    record :=
        kTypeValue varstring varstring         |
        kTypeDeletion varstring
    varstring :=
    len: varint32
    data: uint8[len]
```
**MemTableInserter**
该类只是调用了MemTable的Add方法将WriteBatch中的key-value插入到memtable中，包括记录更新的sequence-number。
**Iterate**
该函数的功能是遍历WriteBatch中的更新，并使用参数handler(实际上是MemTableInserter)将这些更新写入MemTable中。根据更新的类型（value或deletion）调用MemTableInserter的不同成员函数，最终调用的都是MemTable中的Add函数。

**7. Snapshot**
* **SnapshotImpl SnapshotList**
每个SnapshotImpl里面包是一个含一个序列号，且被SnapshotList管理，SnapshotList维护的是一个以SnapshotImpl为基本元素的双链表。dummyhead之前的快照是最新的，之后的是最旧的。


#### 3.5 DBImpl
* **NewDB**
	1. 创建VersionEdit记录log\_number, next_file\_number(从2开始)， last\_sequence。
	2. 创建MANIFEST文件，文件号从1开始，将VersionEdit序列化写入MANIFEST文件。
	3. 设置CURRENT文件指向新创建的MANIFEST文件。
* **DeleteObsoleteFiles**
删除不需要的文件。
	1. 如果出现后台线程错误，那么则无法确定新的Version是否已经形成，因此直接返回。
	2. 统计在所有Version中的文件。
	3. 统计当前文件夹下的所有文件名。读取文件名中的文件类型和文件号。若是kLogFile，则查看是否大于等于versions的log\_number或者等于prev\_log\_number, 否则将其删除，因为logfile的存在是为了保证正在写的memtable内容不会丢失，在log\_number之前的log文件已经被转换为Table文件因此可以将其删除;若是MANIEFST，查看是否大于等于当前的MAINIFEST文件号；若是kTableFile或者kTempFile,则查看是否在所有version管理的文件中；其他类型的文件都会保留。
	4. 如果判定当前文件是不需要的，且文件类型为kTableFile则将其从TableCache中删除，接着在当前目录下直接删除此文件的磁盘内容。
* **Recover**
恢复数据库。
	1. 创建文件锁提供进程互斥的功能，用来保护DB的状态。
	2. 如果CURRENT文件不存在则调用NewDB进行创建。
	3. 调用VersionSet::Recover来通过CURRENT文件指向的MANIFEST文件恢复versions。
	4. 获取当前目录下的所有文件以及所有version中管理的文件，并记录大于等于当前versions中的log\_number,将文件根据log\_number进行排序，依次调用RecoverLogFile来恢复log文件（即memtable文件）。由于这些logfile可能还没有被写入MANIFEST文件，因此需要手动将其文件号手动标注为已使用。
	5. 设置versions的last\_sequence为log\_file中的最大sequence。
* **RecoverLogFile**
	1. 通过log\_number读取log\_file。
	2. 依次读取log\_file中的record，每一条记录就是一个key-value对，写入memtable,如果memtable的大小超过了预设值write\_bffer\_size，则需要调用WriteLevel0Table将memtable写入level-0。
	3. 如果最后一个log\_file没有经过compaction则重用这个log-file，并将logfile\_number记为最后一个log\_file的文件号。
	4. 如果mem不为NULL，则将其写入level-0。
	5. 如果有文件发生变动则将save\_manifest置为true表示需要将新的更改保存到MANIFEST文件中。
* **BuildTable**
通过输入的memtable上的迭代器以及FileMetaData中的文件号创建一个Table文件。
* **WriteLevel0Table**
将memtable写入table文件。
	1. 获取最新的文件号，并以mem中的key-value为内容建立table文件。
	2. 调用PickLevelForMemTableOutput为新生成的文件选定所在的level。
	3. 将新增的文件以及对应的level保存到VersionEdit，用于接下来写入MANIFEST文件中。
	4. 将当前文件信息以及时间记录到CompactionStat中。
* **MakeRoomForWrite**
此模块的用意是memtable中的空间已经写满或者level-0的文件数太多得需要进行compaction。
	1. 首先判断是不是强制性的，如果不是强制性的可以进行将写入速度变慢。
	2. 如果允许变慢写的速度以及level-0中文件数已经达到了使得写速度变慢的文件数，则将锁放开睡眠1ms再获得锁。
	3. 如果是非强制的compaction且当前的mem\_使用的内存空间小于预先规定的大小则跳出循环表示memtable中还有额外的空间。
	4. 如果当前memtable文件已经达到了预设值且前一个memtable（即imm\_）还在进行compaction则需要等待imm\_压缩完成。
	5. 如果level-0中的文件数太多则停止写操作。
	6. 如果上面条件都不满足则试图切换到新的memtable文件,并新建新的log文件记录Write写入的数据，然后调用MaybeScheduleCompaction进行压缩。
* **MaybeScheduleCompaction**
判断当前状态下是否能够进行Compaction。
	1. 查看当前是否有后台线程在进行compaction
	2. 查看是否当前在删除db
	3. 查看当前后台线程是否出现错误
	4. 查看imm\_是否为空（表示没有memtable文件需要压缩）或者versions\_是否需要进行压缩
	5. 如果以上条件都不满足说明确实需要进行压缩，则使用后台线程执行BGWork。
* **BGWork**
此函数是类函数因此可以直接取得函数指针进行使用，里面调用DBImpl::BackgroundCall。
* **BackgroundCall**
判断当前状态是否在关闭db或者后台线程出现错误，如果一切顺利则调用BackgroundCompaction进行压缩工作。由于在一次压缩中可能会在某一层中产生太多的文件因此调用MaybeSchedualCompaction检查是否需要再进行压缩。压缩完毕则使用条件变量唤醒所有等待压缩完成的线程。
* **BackgroundCompaction**
进行具体的后台压缩工作。
	1. 如果imm\_有效表明有memtable文件需要压缩，则调用CompactMemTable压缩memtable文件。
	2. 检查是否是人为的设定压缩的level以及键值范围，若是则调用VersionSet::CompactRange设置Compaction在level的输入文件列表以及level+1的输入文件列表。如果返回的Compaction为空则表明没有文件参加压缩。
	3. 若没有人为设置则调用VersionSet::PickCompaction返回设置好两组input文件列表的Compaction。
	4. 如果返回的Compaction为空则表明没有文件参加压缩，否则
	5. 如果不是人为的设定compaction参数且当前compaction可以直接移动到下一层。则直接使用VersionEdit将处于level的input文件号搬移到level+1。否则进行下面的压缩工作。
	6. 调用DoCompactionWork执行实际的压缩工作。
	7. 调用CleanupCompaction将pending\_outputs\_中删除在这次压缩中新产生的文件。pending\_outputs中存储那些正在文件正在生成，及文件号已经分配，在调用DeleteObsoleteFiles的过程中不会删除里面的文件。
	8. 解除Compaction中作为输入文件的引用，调用DeleteObsoleteFiles删除不需要的文件。
* **DoCompactionWork**
执行实际的压缩工作。
	1. 设定当前的CompactionState的最小Snapshot值，如果快照列表为空则设置为上一个分配的sequence,即last-sequence， 否则设置为快照列表中最旧的快照。如果获得的key值的sequence是小于最小快照的seq则表明该key是不怎么重要的，可能有seq更大但user\_key相同的key将其覆盖。
	2. 获得Compaction中文件的迭代器
	3. 在对每个key值进行压缩之前都会检查是否有immutable memtable需要进行压缩，若是则调用CompactMemTable
	4. 检查是否当前的key值所标识的范围与grandparent-level的文件重叠的太多，若是则输出compaction后的Table文件。
	5. 判断当前的key是否是第一次出现（即user\_key相同的所有键值中sequence\_number最大）
	6. 如果当前key的sequence小于等于当前的最小快照所包含的sequence\_number，则后面具有相同user\_key但seq更小的key将会被丢弃。
	7. 如果当前的key的类型是deletion类型， seq又小于等于最小的快照（表示更低层的key有更大的seq），并且更高层没有这个key则表示这个删除标志是可以丢弃的。
	8. 如果当前key是不可以丢弃的则考虑将其加入TableBuilder中。如果TableBuilder是空的表示当前key是第一个加入的，因此将其设置为Table文件的最小key值。
	9. 如果加入这个key之后，Table文件的大小已经超过了预设值则输出压缩后的Table文件。然后对下一个key进行同样的操作。
	10.调用InstallCompactionResults在新的Version中记录删除的文件和加入新产生的文件。
* **InstallCompactionResults**
此模块的主要功能是使用VersionEdit记录需要删除的文件和新添加的文件，并将VersionEdit应用到当前的Version中产生新的Version。由于这些操作会改变VersionSet的内部操作，而每个version都是可以访问VersionSet因此需要加锁。
	1. 调用Compaction::AddInputDeletions将Compaction中用来压缩的文件记录在删除列表中。
	2. 将新产生的文件加入到对应level+1的添加列表中。
	3. 将VersionEdit记录到MANIFEST文件中并应用到当前Version生成下一个Version。
* **OpenCompactionOutputFile**
这个模块主要是为新产生的Table文件初始化TableBuilder并设置文件号。
	1. 从VersionSet中分配下一个文件号，将文件号存储在pending\_outputs，在下一次的VersionSet::LogAndApply时用来加入到新的version中。
	2. 创建输出文件并初始化TableBuilder
* **FinishCompactionOutputFile**
将CompactionState::builder中包含的key-value生成Table文件。
	1. Compaction中输入迭代器的状态无误后执行TableBuilder::Finish输出Table文件。
	2. 使用TableCache读取确认新生成的TableFile是否可用。
* **CompactMemTable**
将immutable MemTable文件转换为Table文件。
	1. 调用WriteLevel0Table将immu\_中的key-values对存储在新生成的Table文件中，将新生成的文件加入到VersionEdit。
	2. 将VersionEdit中记录的log\_number设置为当前db中的logfile\_number，调用VersionSet::LogAndApply将VersionEdit记录到MANIFEST文件中，然后将VersionSet的当前Version改为应用VersionEdit后生成的新的Version。
	3. 释放immu\_, 并将不需要的文件删除。
* **CompactRange**
根据输入的键值范围来对数据库进行compaction
	1. 确定与输入键值范围重叠的最大层数。
	2. 检测并等待imm\_压缩完成。
	3. 逐层调用TEST\_CompactRange
* **TEST\_CompactRange**
设置manual\_compaction的键值范围，调用MaybeScheduleCompaction进行压缩, 在实际选择压缩的层数时则使用manual中指定的层数而不会调用PickCompaction选择层数。
* **TEST\_CompactMemTable**
检测现在是否有imm\_进行压缩。
	1. 等待之前的imm\_的compaction完成
	2. 加锁后检查当前的imm\_是否为空, 因为在此刻有可能mem\_又已经填满并转换为imm\_,若imm\_不为空则等待后台线程将imm\_进行压缩完成。
* **IterState**
保存遍历mem\_和imm时的状态信息。
* **CleanupIteratorState**
解除Iterator对mem、imm、version的引用。
* **NewInternalIterator**
获取对当前状态的db的Iterator。
	1. 由于会对db中的imm\_、mem\_和version进行引用因此需要加锁。
	2. 由imm、mem\_和当前Version的Iterator产生对当前db的Iterator。
	3. 设置新产生的Iterator析构时的清理函数。
* **TEST\_NewInternalIterator**
调用NewInternalIterator返回当前db的迭代器。
* **TEST\_MaxNextLevelOverlappingBytes**
在锁保护的机制下计算单个文件与下一层重叠的最大字节数。
* **Get**
读取与key对应的value。
	1. 加锁获得当前的最后一个sequence\_number作为快照。
	2. 对db中当前的mem\_、imm\_和当前的version进行引用，防止被compaction后删除。接着解锁读取文件，**为什么现在解除锁是安全的？**由于这些memtable和version中的文件存在的唯一更改就是将其删除，但是由于当前线程已经获得对这些资源的引用，因此它们是不会被删除的，由于其他线程不会对其内容作出更改，因此在读取这些资源时解除锁是安全的。
	3. 使用last\_sequence\_number(当前状态下最大的seq)和key构造lookup\_key, 在寻找key时如果遇到多个user\_key部分相同的key则会选择seq尽量大的key（参考VersionSet::Get）。
	4. 依次查找mem\_、imm\_和当前version（按照新鲜程度递减的次序）。
	5. 如果是在当前version中查找到的，则记录被访问的文件的FileMetaData信息以及所在的level。
	6. 如果找到了对应的key值则调用VersionSet::UpadateStats来更新所访问的文件的seek次数，seek次数达到了预设的值就会被记录为将要进行compaction的文件。如果确实将被访问的文件设置为将要进行compaction的文件，则调用MaybeScheduleCompaction试图对其进行压缩。
	7. 释放mem\_、imm\_和当前version的引用。
* **RecordReadSample**
加锁调用Version::RecordReadSample, 实际上是找出第一次出现key的文件，由于此文件被seek了一次，调用Version::UpdateStats更新文件被seek的次数，如果次数超过了预设值则说明该文件已经被访问次数太多了调用一次compaction是可以接受的，因此启用MaybeScheduleCompaction试图压缩此文件。另外只有在存在两个文件或两个以上文件含有此key时才会调用UpdateStats来更新文件的seek次数。
* **NewIterator**
	1. 获得对当前的db的Iterator，可以访问mem\_、imm\_和version中的Table文件，并取得当前的最新sequence(也就是快照)和seed(用于生成随机数)。
	2. 根据获得的快照生成相应的迭代器。
* **Write**
	* Writer
	这个结构体中保存WriteBatch，以及是否立刻刷到磁盘和完成状态，以及一个用于同步的条件变量。
	1. 加入writers\_队尾，并等待其他writer完成知道当前writer变为队头。
	2. 调用MakeRoomForWrite检查memtable文件是否还有多余的空间，若没有则等待将memtable进行压缩。
	3. 调用BuildBatchGroup试图将多个writer中的WriteBatch一次写入。计算新的last\_sequence。
	4. 接下来将WriteBatch写入log文件(在recover时可用)和memtable中，在这之前可以解锁。**为什么可以解锁？**因为当前执行写的线程正在写入队列头的WriteBatch，其他所有的Writer都因为自己不是队列头因此被阻塞着，所以此时只有当前线程在写log文件和memtable。当然此时使用Get读取也是没有问题的，因为当前的version的last\_sequence还没有更新，因此即使有线程在读也读不到当前正在写入memtable中的key值，当然有人会担心在读取变量时伴随着其他线程写会出现寄存器值和内存值不一致的情况，这个得回到memtable的实现是基于skiplist实现，里面的插入和迭代器遍历都是可以保证直接从内存中读取和直接将更新写到内存。
	5. 将已经写了WriteBatch的writer从队列取出，并解放那些还在被阻塞但其实它的WriteBatch在本次写中已经被写了的线程。
	6. 如果队列不为空说明还有Writers在等待，则将队列的第一个writer唤醒。
* **BuildBatchGroup**
试图一次写入多个WriteBatch。
	1. 首先判断队列前面的WriteBatch的大小是不是太小，如果太小则限制一次写入的大小，否则将以(1 << 20)作为最大值来append排在队列后面的WriteBatch。
	2. 如果第一个WriteBatch不需要同步写到磁盘则当后面出现同步写磁盘的WriteBatch时停止append后面的WriteBatch。
	3. 经过多次append后WriteBatch的大小超过之前的限制也会停止append后面的WriteBatch。
* **GetProperty**
根据输入property中包含的选项，此函数则计算对应的数据或状态并返回。
* **GetApproximateSizes**
根据键值范围计算覆盖的字节数。
* **Put**
将key-value放入WriteBatch中转而调用write写入memtable。
* **Delete**
将key和删除标志放入WriteBatch中转而调用write写入memtable。
* **Open**
	1. 试着去recover（因为leveldb中使用的是读写锁，使用lockfile文件锁保证lockfile只能被当前进程访问，多个进程对文件进行读写可能会导致文件内容被破坏，因此可以保证每次只能有一个进程能够打开当前数据库），并将更改记录到VersionEdit中。
	2. 初始化logfile和memtable。
	3. 将VersionEdit写入MANIFEST文件并应用到当前Version中生成新的Version。
	4. 删除不需要的文件并试图进行压缩。
* **DestroyDB**
此函数是一个普通函数，可以直接包含头文件进行调用。
	1. 获得当前db中的文件锁防止多个进程删除db，读取当前文件夹中的所有文件，如果当前文件夹下没有文件则返回。
	2. 处理lockfile和目录，依次删除其他文件，然后解除文件锁，接着删除lockfile和目录。


#### 3.6 DBIter
memtable和sstable中存储的key值可能有多个，DBIter就是将多个具有相同user\_key的key值合并为一个key，且输出user\_key部分。内部含有一个Iterator iter\_，当向前移动时iter\_指向输出的那个key，向后移动时，iter\_所指位置之后是所有user\_key相等的key。构造函数中seed是作为随机数的种子。
* **ClearSavedValue**
当saved\_value\_的容量大于1MB时，使用swap将空字符串与saved\_value\_进行交换，否则直接调用string的clear函数。**这样做对于比较长的字符串的清空是不是更快？**
* **ParseKey**
从字符串中解析出InternalKey。其中bytes\_counter的作用是每读取bytes\_counter的key-value对，则对所读的key值所在的文件调用RecordReadSample查看是否该文件需要进行压缩。其中的bytes\_counter设置为一个以kReadBytesPeriod为平均值的随机数。这么做应该是为了使得SSTTable文件压缩比较均匀。
* **FindNextUserEntry**
此函数的功能是skip掉与当前user\_key相同的keys，**另外只关注那些InternalKey的seq部分比此迭代器所依赖的sequence大的key，那些拥有更大sequence的key是在迭代器所依赖的快照生成之后的加入进来的，由此可以知道所谓快照实现的原理仅是使用sequence划分一个可以访问的key值范围**。


