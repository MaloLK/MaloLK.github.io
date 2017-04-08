---
layout: post
title: Memcached守护进程的实现
date: 2017-01-10 00:42
categories: programming
tags: [memcached, C/C++, linux]
---

#### memcachd守护进程的实现
为了摆脱控制终端对进程的影响，一般都是将进程变为守护进程。

使用fork产生子进程，父进程使用_exit()退出，使用_exit()而不是exit是因为exit会进行额外的操作，比如反向执行使用`atexit`注册的函数，在linux中还会对标准输出和文件流进行flush操作。此时**父进程先于子进程结束**, 子进程将会变为孤儿进程，孤儿进程一般会将init进程作为父进程，与僵尸进程的区别是，**子进程先于父进程结束**，当子进程已经结束后，父进程没有使用wait等函数来获取终止子进程的信息以及清理子进程占用的资源（如回收进程列表信息）。

接下来在子进程中调用setsid使得子进程在新的会话(session)中运行。这里需要回顾一下进程组和会话的概念。
> 进程组：一个或多个进程的集合。拥有唯一的标识进程组 ID，每个进程组都有一个组长进程，该进程的进程号等于其进程组的 ID。**进程组 ID 不会因组长进程退出而受到影响**，fork 调用也不会改变进程组 ID。
> 会话：一个或多个进程组的集合。新建会话时，当前进程（会话中唯一的进程）成为会话首进程，也是当前进程组的组长进程，其进程号为会话 ID，同样也是该进程组的 ID。它通常是登录 shell，也可以是调用 setsid 新建会话的孤儿进程。对setsid的限制是不能由组长进程来调用setsid。
因此使用setsid后，子进程将会脱离原会话、原进程组的控制以及控制终端的影响。
值得注意的是，**由于当前子进程已经变成了新的会话的会话首进程，但是仍然可以重新申请打开控制终端，为了防止进程重新打开控制终端，可以再次进行fork并结束当前进程，此时的进程将不再是会话首进程，自然也就不能重新打开控制终端**。在memcached中，使用守护进程的方式有点不一样，在memcached的main函数中，首先将SIGHUP信号屏蔽，然后再调用daemonize函数，但是并没有再次进行fork，这种方式也是可行的。理由如下，首先在调用daemonize函数后此时的进程已经变为会话首进程，但是没有控制终端，如果这个进程打开一个终端设备，这个终端将会成为这个进程的控制终端，为了防止在这种情况下控制终端退出导致进程退出，屏蔽SIGHUP信号是必要的。

最后一步是将输入、输出以及错误等描述符重定向到/dev/null设备。重定向使用dup2。

步骤总结如下：
1. fork，父进程退出，子进程保留。
2. 子进程使用setsid开启新的会话，并成为会话首进程和组长进程。
3. 改变工作目录（可选）。
4. 重新设置文件掩码 (可选)。
5. 重定向标准输入、标准输出和标准错误到/dev/null。

```C
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#include "memcached.h"

int daemonize(int nochdir, int noclose)
{
    int fd;

    switch (fork()) {
    case -1:
        return (-1);
    case 0:
        // 子进程
        break;
    default:
        // 父进程成功退出让子进程单独运行
        _exit(EXIT_SUCCESS);
    }

    // 将程序在新的session中运行，子进程将会成为新的sesion的group leader，但是不能控制终端
    if (setsid() == -1)
        return (-1);

    if (nochdir == 0) {
        // 将当前的工作路径切换到根路径
        if(chdir("/") != 0) {
            perror("chdir");
            return (-1);
        }
    }

    // 将stdin, stdout 和 stderr　都导向 /dev/null
    if (noclose == 0 && (fd = open("/dev/null", O_RDWR, 0)) != -1) {
        if(dup2(fd, STDIN_FILENO) < 0) {
            perror("dup2 stdin");
            return (-1);
        }
        if(dup2(fd, STDOUT_FILENO) < 0) {
            perror("dup2 stdout");
            return (-1);
        }
        if(dup2(fd, STDERR_FILENO) < 0) {
            perror("dup2 stderr");
            return (-1);
        }

        if (fd > STDERR_FILENO) {
            // 节省文件描述符
            if(close(fd) < 0) {
                perror("close");
                return (-1);
            }
        }
    }
    return (0);
}
```

#### 从Linux shell 中启动守护进程
在实际使用中，如何在linux中最快速的将某个进程设置为守护进程？
使用`nohup`命令可以帮助我们将进程变为守护进程。如下：
```sh
nohup your-porcess &
```
当一个会话结束时，会向会话中的每个进程发送SIGHUP信号，这个信号会使得进程退出。因此这里使用nohup的意图就是阻止当前进程接收SIGHUP信号，另一个作用是关闭标准输入，即使这个进程运行在前台也接收不到输入，而标准输出和标准错误都将会被重定向到nohup.out中，这里可以自定义标准输出和标准错误的重定向。加上'&'的作用则是将其变为后台进程。

#### 参考
[Linux 守护进程的启动方法](http://www.ruanyifeng.com/blog/2016/02/linux-daemon.html)
[Linux 守护进程的实现](http://alfred-sun.github.io/blog/2015/06/18/daemon-implementation/)
