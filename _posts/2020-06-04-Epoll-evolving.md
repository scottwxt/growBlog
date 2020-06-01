---
layout: post
title: 不断发展的Epoll
date: 2020-06-01 00:00:00 +0300
description: epoll为了使轮询I/O事件更加具有扩展性，协同地使用了一系列的系统调用。为此，它必须最小化使用每一个系统调用并且返回多个事件，所以调用数量也必须是最小化的。但是他的扩展性诉求并没有满足一些用户。Roman Penyaev从一系列的补丁看到此问题，并提出自己的解决方案:给内核增加另外一个环形缓冲区(ring-buffer) # Add post description (optional)
img: epoll/epoll_API.png # Add image post (optional)
tags: [Epoll, Linux, Kernel, System Call] # add tag
---
## 概述
&emsp;&emsp;Epoll作为Linux特定轮询处理大量地的活跃连接的系统调用集。它的API从2.5版本开发，直到就一直被使用，与和大多数接口一样，它总是有提升的空间，当前有两个补丁可以为这组系统调用增加功能。

## Epoll预览
&emsp;&emsp;Epoll的功能设计与select() 、poll()相似，但在使用大量文件描述符场景是更好的选择和更高的性能。当有新的连接被添加时，select() or poll()的每一次调用都涉及全部的文件描述符，所以内核必须轮询验证每一个文件描述符，检查I/O是否就绪，添加新的轮询进程到适当的等待队列，但实际上文件描述符改变的数量并不是很多，以至于有一些工作是没有必要重复做的,Epoll通过分离准备工作和等待就绪文件描述符来解决此问题。
<br />
&emsp;&emsp;进程开始使用API必须创建一个特殊的轮询文件描述符，以下的调用都可以完成这个工作:
```
    #include <sys/epoll.h>
    int epoll_create(int size);
    int epoll_create1(int flags);
```
&emsp;&emsp;这两个调用都保留Epoll的功能，他们都返回将被使用件描述符，epoll_create()调用的参数size将不再使用，epoll_create1()增加了flags参数，它用于生成描述符设置close-on-exec标记。
为了监控此文件描述符，下一步是把它增加到文件描述符集中，使用以下函数:
```
    int epoll_ctl(int efd, int op, int fd, struct epoll_event *event);
```

最后，等到其中一个文件描述符处于就绪状态。
```
    int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
    int epoll_pwait(int epfd, struct epoll_event *events, int maxevents, int timeout,
                    const sigset_t *sigmask);
```
具体函数意义为: 
epoll_wait()返回存储在events数组中最大数量，
timeout参数具体到毫秒，epoll_pwait()版本在调用时候也能允许设置阻塞与非阻塞标记，最后，请查看the man page可以找到相关细节。

## epoll_ctl_batch() and epoll_pwait1()
Fam Zheng提供的补丁集为Epoll引入了两个新的系统调用，第一个解决了需要更新epoll集合中的文件描述符的问题，调用epoll_ctl()只能对单个文件描述符进行添加，修改、删除，如果多个描述符需要更新，则需要多次调用epoll_ctl()才能完成工作。提议中的epoll_ctl_batch()解决单个调用就能处理多个描述符的问题。
```
    int epoll_ctl_batch(int epfd, int flags, int ncmds, struct epoll_ctl_cmd *cmds);
```
该cmds结构本质上是复制本应传递给epoll_ctl（）调用的所有参数。 
通过传递这些结构的数组，程序可以通过单个系统调用对多个文件描述符执行操作。
Fam的另一个更改是添加一个新的系统调用来执行实际的轮询：
```
    struct epoll_wait_params {
	int clockid;
	struct timespec timeout;
	sigset_t *sigmask;
	size_t sigsetsize;
    };

    int epoll_pwait1(int epfd, int flags,
                    struct epoll_event *events, int maxevents,
                    struct epoll_wait_params *params);
```
此版本的epoll_wait（）添加了新的flags参数，但未定义任何标志值，因此标志必须为零。 改组为params结构的参数主要是为了使应用程序可以更好地控制超时处理。 对于许多用例，事实证明epoll_wait（）可以理解的毫秒级分辨率超时过于粗糙。 新的系统调用定义了具有纳秒分辨率的超时，可以解决该问题。

## 对多线程更为友好
&emsp;&emsp;Akamai的Jason Baron遇到了一个不同的问题，只是提出了一种相对不常见的使用模式。通常，只有一个进程使用轮询功能来监视给定的文件描述符。但是，在Jason的用例中，可以有多个线程，每个线程都使用epoll跟踪同一组文件描述符。在此设置下，任何给定文件描述符上的事件都会导致所有等待的进程唤醒，即使其中只有一个能够实际处理该事件。 Jason希望避免出现这种敬群效应的问题。

&emsp;&emsp;他的回应是此补丁添加了两个新标志，这些标志通过epoll_ctl（）附加到文件描述符。其中第一个是EPOLLEXCLUSIVE，它要求仅唤醒一个进程来处理关联文件描述符上的事件。在内部，更改是在设置轮询时使用add_wait_queue_exclusive（）而不是add_wait_queue（）的简单问题。显然，所有轮询同一文件描述符的进程都必须使用独占模式才能将每个事件唤醒一次。
但是，这种更改并不能完全解决Jason的问题，因为它最终会响应每个事件而唤醒相同的过程。由于epoll存在的原因之一是允许在调用epoll_wait（）之间将进程留在所有特定于文件描述符的等待队列上，因此位于任何给定队列开头的进程将保留在那里。因此它将成为接收所有专有唤醒的人。但是，让多个进程轮询同一个文件描述符的全部目的是分散工作。每当破坏该目标时，都要唤醒相同的过程。为了解决这个问题，Jason添加了另一个名为EPOLLROUNDROBIN的标志，该标志使内核依次处理每个轮询过程。
对调度模式的支持以新的等待队列功能的形式添加到了调度程序中：
使用add_wait_queue_rr（）完成等待后，将仅唤醒一个进程，就像使用add_wait_queue_exclusive（）一样。但是，此外，等待队列开头的进程将移至尾部，因此直到所有其他进程轮到它时，它才会看到另一次唤醒。

&emsp;&emsp;Jason的补丁发布包括一个小基准程序的结果，该结果显示使用独占模式时执行时间减少了近50％。当发生大量唤醒时（许多面向网络的工作负载可能就是这种情况），由雷群引起的额外开销可能会减少。
&emsp;&emsp;这两个补丁都收到了大量评论评论，特别是自1月份首次发布以来，Fam的补丁有了很大的发展。编辑者的不科学印象是与API相关的补丁比以往任何时候都受到更多关注。那当然是一件好事； API是永久存在的，因此最好在将它们交付给用户并承诺永远支持它们之前正确使用它们。但是，这些补丁似乎已经准备就绪。它们可能会在下一个合并窗口中显示。

##### 原作: Jonathan Corbet February 16, 2015
##### 翻译: scott wilson Jane 1, 2020
原文链接: [https://lwn.net/Articles/633422/](https://lwn.net/Articles/633422/)