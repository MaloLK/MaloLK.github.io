title: Memory Barrier的理解
layout: post
date: 2016-09-25 00:26
categories: programming
tags: [体系结构, memory]
---
## 内存屏障（Memory Barrier）到底是什么？
对于单个CPU来说，只要所执行的内存操作是独立的，那么CPU则可能会对这些操作的顺序进行重排以达到较好的效率，而这种情况会在多个CPU的情况下出现问题。
```
	CPU 1    		CPU 2
	===============	===============
		{ A == 1; B == 2 }
		A = 3;		x = B;
		B = 4;		y = A;
```
操作`A = 3; B = 4;`对CPU1来说是两个独立的操作，CPU1可能会以任意顺序来执行这些序列，这对于正在访问`B、A`的CPU2来说，所得到的值是不确定的。为了保证程序按照预想的顺序执行，必须引入CPU内存屏障来抑制CPU的这种优化（乱序）行为。另外编译器也会为了优化而使得执行的指令顺序可能与我们预想的并不相同。

<!--more-->

## 内存屏障有哪些类型？
+ __Write (or store) 内存屏障__
Write-barrier 能够保证位于它之前的所有`STORE`操作能够在其后的所有`STORE`操作开始之前完成。
+ __Data dependency 内存屏障__
对于两个相依赖的`load`操作`B = load &A，C = load B`， barrier能够保证在取得A的地址之前，A的更新已经完毕，这样就能够防止当别的CPU更改A的值时，当前CPU_感知_的仍然是旧的A的值（具体的CPU之间的_感知_机制还需细究）。
+ __Read （or load）内存屏障__
Read-barrier 能够保证位于它之前的所有的`LOAD`操作能够在其后的所有`LOAD`操作开始之前完成。（有如下场景，`LOAD B；read_barrier()；LOAD A；`， read\_barrier能够保证当读取B时，对于B的最新值以及在B之前对其他变量做的更新都能被read\_barrier之后的操作_感知_到。如果在当前CPU读取B之前，别的CPU相继对A、B做了更新，那么read\_barrier的存在就能保证后续读取A时能够读到更新值，而不是感知到旧的A的值）。
+ __General 内存屏障__
General-barrier 则是`Write-`和`Read-barrier`的结合，表示出现在它之前的所有`LOAD`和`STORE`操作必须在出现在其后的`LOAD`和`STORE`操作开始之前完成。
+ __ACQUIRE operations__
表征的语义是在barriers的后面的所有的操作执行时也应该在其后面。
+ __Release operations__
表征的语义是在barriers的前面的所有的操作都应该在其前面执行。

## Linux内核的那些barriers
* __编译器内存屏障__
Compiler barrier在linux内核中由函数barrier()来体现，它是一种General Barrier。主要的作用如下：
    * 防止编译器在`reorder`时将barrier()之后的访问操作置于barrier()之前。比如在中断处理指令和被中断指令之间使用barrier()可以保证一定的访问顺序。
    * 在循环中，强制编译器在每次循环时都从内存重新导入用于条件判断的变量值。
barrier()的副作用是将编译器存储在所有寄存器中的值丢弃再重新读取，而linux kernel中有`READ_ONCE()`和`WRITE_ONCE()`（由`volatile cast`实现）等操作只对特定内存地址进行重新读取，对CPU的`order`没有影响。
GCC中程序可以使用以下语句来实现编译器的内存屏障，此屏障是一种General Barrier。`__asm__ __volatile__("" : : : "memory")`。其中的`__asm__`表示其后是一条内嵌的汇编语句，`__volatile__`则用来阻止编译器优化，`memory`则表示需要重新从内存读取。这条语句只能实现编译器的内存屏障，不能保证CPU不会对指令的乱序执行。
* __CPU 内存屏障__
Linux内核的8个基本CPU内存屏障。其中除了`DATA DEPENDENCY`之外，其他的CPU屏障函数实际上调用的是编译器的barrier(); smp_xxx()是为了给其他的CPU提供barrier宏，例如x86的rmb()是用了lfence指令，但其它CPU不能用这个指令。x86体系结构下，`STORE`和`STORE`之间的顺序在各个CPU之间是一致的，`LOAD`和`LOAD`之间的顺序在各个CPU之间是一致的，但是并不保证`STORE`和`LOAD`之间的顺序在各个CPU之间是一致的。而CPU内存屏障可以用来保证`STORE`和`LOAD`之间顺序的一致性。相对于Compiler Barrier 使用如下语句可以阻止CPU和编译器的乱序行为。
	```
		__asm__ __volatile__("mfence" : : : "memory");
	```
|  TYPE             |  MANDATORY              | SMP CONDITIONAL             |
| :---------------: | :---------------------: | :-------------------------: |
| GENERAL           |	mb()	              | smp_mb()                    |
| WRITE             |	wmb()	              | smp_wmb()                   |
| READ	            |	rmb()	              | smp_rmb()                   |
| DATA DEPENDENCY   | read_barrier_depends()  | smp_read_barrier_depends()  |

## 什么时候需要内存屏障
1. Interprocessor interaction.
2. Atomic operations.
3. Accessing devices.
4. Interrupts.

## CPU缓存的影响
一般现在很多CPU内部都存在多个独立的cache，为了提高访问内存的速度，两个cache会并行工作，那么当前CPU更新数据的顺序由于多个cache操作的存在，会导致其他CPU所看到这些数据更新的顺序是不一样的。

#### CPU、缓存和内存的关系
```


	    <--- CPU --->         :       <----------- Memory ----------->
	                          :
	+--------+    +--------+  :   +--------+    +-----------+
	|        |    |        |  :   |        |    |           |    +--------+
	|  CPU   |    | Memory |  :   | CPU    |    |           |    |        |
	|  Core  |--->| Access |----->| Cache  |<-->|           |    |        |
	|        |    | Queue  |  :   |        |    |           |--->| Memory |
	|        |    |        |  :   |        |    |           |    |        |
	+--------+    +--------+  :   +--------+    |           |    |        |
	                          :                 | Cache     |    +--------+
	                          :                 | Coherency |
	                          :                 | Mechanism |    +--------+
	+--------+    +--------+  :   +--------+    |           |    |	    |
	|        |    |        |  :   |        |    |           |    |        |
	|  CPU   |    | Memory |  :   | CPU    |    |           |--->| Device |
	|  Core  |--->| Access |----->| Cache  |<-->|           |    |        |
	|        |    | Queue  |  :   |        |    |           |    |        |
	|        |    |        |  :   |        |    |           |    +--------+
	+--------+    +--------+  :   +--------+    +-----------+
	                          :
```
CPU内存将以它所认为合适的顺序来执行指令，其中有些`STORE or LOAD`操作指令则需要放到内存访问队列中进行处理，队列中这些指令的顺序也是由CPU来决定。内存屏障（Memory Barriers）的职责是控制CPU和内存之间的访问控制（比如说使用volatile关键字使得编译器每次直接从内存读取，而不是使用寄存器内的变量值），以及在多CPU的情况下，控制一个CPU能够感知另一个CPU对相同变量的更新顺序， 比如read_barrier()的作用。

#### 缓存一致性（Cache coherency）
1. __为什么需要缓存一致性？__
一般每个CPU核都自带私有的L1、L2缓存，多个核之间共享一个缓存L3，那么CPU1对全局变量`global_var`修改后将会首先把修改后的值刷入私有缓存L1、L2，另一个CPU2随后访问该全局变量时应该看到被修改后的值，这就需要协议来保证这种一致性。
2. __缓存伪共享的问题__？
缓存一致性协议能够操作的最小的单元为缓存行（Cache line），如果两个CPU所访问的两个变量x、y都为全局变量，且这两个变量在同一个缓存行上，当不同线程1修改变量x，线程2修改变量y，各个线程通过获取缓存行的所有权来更新全局变量，当线程1获得所有权时，线程2中对应的缓存行将会失效，必须通过L3重新获取缓存行内容，反之L2获得相应缓存行的所有权时，线程1也得这么做，对性能的影响很大，这种情况下L3没有真正的被各个线程共享。
可采取的解决方式是通过填充缓存行来使得变量x、y不在一个缓存行上，缺点是浪费内存。


## 一些实际应用
+ __关于关键字Volatile__
关键字vlolatile的主要作用是保证被修饰的变量每次读取时都会从内存中读取，而不是使用寄存器中现有的值，主要是为了使得别的线程对被修饰的变量更改时，当前线程能够获得最新值。
volatile的局限性在于不能保证编译器会将其周边的非volatile代码进行乱序处理，这样程序所执行的结果可能并不是我们想要的。比如下面这种情况, `flag`可能在`send_data`还没执行时就已经被置为`true`。`volatile`并不能保证所修饰的变量与其周围的代码之间的顺序。
	```
	volatile bool flag = false; // global var
	------thread 1
	...
	send_data();
	flag = true;
	...

	-------thread 2
	...
	while(!flag);
	recv_data();
	...

	```
使用CPU memory barrier不仅能够保证所修饰变量与周围代码之间的顺序，而且也提供了和volatile一样的立即可见性，即对变量的修改会直接写入缓存和内存，读取也是直接从缓存和内存来读取。比如C++11中的std::atomic<T>就能提供这两种特性。
+ __标准的同步原语（standard synchronization primitives）__
spinlocks, semaphores, RCU等都自带memory barriers。值得注意的是，大多数的原子操作都是没有包含memory barriers，比如atomic\_inc()和atomic\_add()。


## Reference
[并行编程--内存模型之缓存一致性](http://www.cnblogs.com/jiayy/p/3246133.html)
[LINUX KERNEL MEMORY BARRIERS](https://www.kernel.org/doc/Documentation/memory-barriers.txt)
[Why is volatile not considered useful in multithreaded C or C++ programming?](http://stackoverflow.com/questions/2484980/why-is-volatile-not-considered-useful-in-multithreaded-c-or-c-programming)
[这篇文章解释了为什么CPU会乱序](http://www.linuxjournal.com/article/8211?page=0,0)
[Memory Ordering at Compile Time](http://preshing.com/20120625/memory-ordering-at-compile-time/)




