---
layout: post
title: 关于linux中socket选项的理解
date: 2016-12-17 16:33:02
categories: programming
tags: [linux, network]
---

#### SO_REUSEADDR

主要有两个作用[戳这](http://stackoverflow.com/questions/14388706/socket-options-so-reuseaddr-and-so-reuseport-how-do-they-differ-do-they-mean-t)：
1. 对`wildcard`地址有影响，假如当前的`server`绑定的地址是0.0.0.0:21, 如果这个选项没有设置，那么此时同一台主机的另一个`server`绑定到特定地址相同端口，比如 192.168.0.1:21，就会发生错误EADDRINUSE。如果选项有设置，则不会出现这个错误。
2. 另外一个作用比较常见，在程序调试或者重新载入新的设置时一般需要立即重启`server`，但是之前被关闭的server可能正处于`TIME_WAIT`状态，此时如果没有设置这个选项，则会导致EADDRINUSE，因为处于`TIME_WAIT`的server仍旧绑定在相同的地址和端口上。因此实际使用时会设置这个选项利于调试。

#### SO_REUSEPORT

这个选项的作用[戳这](http://stackoverflow.com/questions/14388706/socket-options-so-reuseaddr-and-so-reuseport-how-do-they-differ-do-they-mean-t)是允许多个socket绑定相同的地址和端口，唯一的要求是所有绑定到这个地址和端口的socket都必须设置这个选项。这个功能常见的应用是实现负载均衡。
但是这个选项并不能完全替代`SO_REUSEADDR`，比如serverA绑定到ip:port，然后处于TIME_WAIT状态，此时serverB也绑定到相同的地址，只有两种情况才不会出现EADDRINUSE，一种是serverB设置了`SO_REUSEADDR`，另外一种情况是serverA和serverB都设置了`SO_REUSEPORT`。

#### socket连接
1. 设置tcp `keep-alive`时刻检查网络连接是否有效，比如当网络断掉时，可能无法完成四次握手导致server无法知道连接是否有效，而`keep-alive`会每隔一段实际来检查连接是否仍然有效。
2. 设置 `SOL_LINGER` 的作用是改变调用close时系统的操作，默认情况下是进行四次挥手结束连接，可能将发送缓冲区的数据发送给对方。
3. 设置tcp nodelay选项的原因是禁用Nagel's算法：
> The algorithm is: if data is smaller than a limit (usually MSS), wait until receiving ACK for previously sent packets and in the mean time accumulate data from user. Then send the accumulated data.
