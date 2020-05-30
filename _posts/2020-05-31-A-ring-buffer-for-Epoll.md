---
layout: post
title: Epoll环形缓冲区
date: 2020-05-31 00:00:00 +0300
description: epoll为了使轮询I/O事件更加具有扩展性，协同地使用了一系列的系统调用。为此，它必须最小化使用每一个系统调用并且返回多个事件，所以调用数量也必须是最小化的。但是他的扩展性诉求并没有满足一些用户。Roman Penyaev从一系列的补丁看到此问题，并提出自己的解决方案:给内核增加另外一个环形缓冲区(ring-buffer) # Add post description (optional)
img: epoll/epoll_API.png # Add image post (optional)
tags: [Epoll, Linux, Kernel] # add tag
---
# Epoll环形缓冲区
##### 原作: Jonathan CorbetMay 30, 2019
##### 翻译: scott wilson 30, 2020
原文链接: [https://lwn.net/Articles/789603](https://lwn.net/Articles/789603)
## 概述
&emsp;&emsp;epoll为了使轮询I/O事件更加具有扩展性，协同地使用了一系列的系统调用。为此，它必须最小化使用每一个系统调用并且返回多个事件，所以调用数量也必须是最小化的。但是他的扩展性诉求并没有满足一些用户。Roman Penyaev从一系列的补丁看到此问题，并提出自己的解决方案:给内核增加另外一个环形缓冲区(ring-buffer)。
<br />
&emsp;&emsp;在Epoll至少一个I/O文件描述符处于就绪状态，否则 poll() 和 select()系统将一直处于等待状态。为了用户能够收到任何一个监听描述符改变的状态的时候，因此需要内核设置一个内部数据结构，Epoll获取分离的就绪和可能等待描述符，它是必须保持内部数据结构和描述符处理时间一样长。
<br />
&emsp;&emsp;程序启动时后，将会调用epoll_create1()创建文件描述符，调用将会偶然取代了epoll_create()，没有使用flags 参数会被替换，epoll_ctl()会添加一个文件描述符到被Epoll所监听epoll集合，调用epoll_wait()将会被阻塞直到任何一个监听的描述符上报了感兴趣的事件，对于应用程序来说，与poll()相比，该接口将会做更多的工作，在大量描述符被监听的情况下，它们之间有很大的不同。
<br />
&emsp;&emsp;话虽如此，仍然有做的更好的空间。尽管epoll比其他更具有效率，程序不得不使用系统调用获取下一个I/O处于就绪状态的文件描述符。在繁忙的系统中，如果获取一个新事件不需要通知内核，那么效率更加高效，这些地方都是大多数需要注意的。这就是Penyaev提交进来的补丁: 它创建一个环形缓冲区，处于应用程序和内核之间，只要有事件发生，它能够在两者之间进行传递事件。

## epoll_create() — 事不过三
&emsp;&emsp;这台机器上的应用程序想要使用,首先它必须告诉内核轮询将会使用环形缓冲队列和大小。当然,epoll_create1()没有一个参数用来表示使用大小的信息，所以它必须增加epoll_create2(),函数原型如下:
<br />
```
    int epoll_create2(int flags, size_t size);
```
为了事件通信，这里有一个新的标记EPOLL_USERPOLL,这个参数告诉内核使用环形缓冲区，size参数表示环形缓冲器持有有多少实体，size以２的幂增长，epoll监控文件描述符的数量有一个上限，当前补丁支持的最大数量为65536。
<br />
&emsp;&emsp;通常来说，使用epoll_ctl()添加文件描述符到轮询集合中,这里应用的时候有一些限制条件，在用户空间轮询有一些操作不能被兼容。特殊的说，EPOLLET 每个文件描述符都会请求边缘触发模式.一个文件描述符变为就绪状态是，也仅有一个文件描述符添加到环形缓冲区。持续添加事件在水平触发模式显然无法正常工作。EPOLLWAKEUP 标记(被用于事件处理的时候防止系统进行休眠)也无法正常工作。EPOLLEXCLUSIVE 标记也不支持。需要两个或者三个单独的mmap()调用才能将环形缓冲区映射到用户空间，第一个应该有０到最大一页的内存偏移.每一页都包含下面的数据结构。
<br />
```
struct epoll_uheader {
    u32 magic;          /* epoll user header magic */
    u32 header_length;  /* length of the header + items */
    u32 index_length;   /* length of the index ring, always pow2 */
    u32 max_items_nr;   /* max number of items */
    u32 head;           /* updated by userland */
    u32 tail;           /* updated by kernel */

    struct epoll_uitem items[];
};
```
&emsp;&emsp;header_length字段有点让困惑,它包含epoll_uheader结构和items 数组的长度.如该示例程序所示，预期的使用模式似乎是应用程序将映射标头结构，获取实际长度，取消映射刚刚映射的页面，然后使用header_length重新映射以获取完整的项目数组。可能有人希望这是环形缓冲区，但这里使用了一层间接层。 要获得实际的环形缓冲区，需要再次调用mmap（），同时将header_length作为偏移量并将index_length header字段作为长度。 结果将是一个整数索引数组，该整数索引将用作真正的环形缓冲区的项目数组。用于指示事件的实际items由以下结构表示：
```
struct epoll_uitem {
 __poll_t ready_events;
 __poll_t events;
 __u64 data;
};
```
&emsp;&emsp;在这里，事件似乎是调用epoll_ctl（）时请求的事件集，而ready_events是实际发生的事件集。 数据字段直接来自添加了此文件描述符的epoll_ctl（）调用。每当头和尾字段不同时，环形缓冲区就会消耗至少一个事件。 要使用事件，应用程序应从head的索引数组中读取条目; 应当在循环中执行此读取，直到在该循环中找到非零值为止。 显然，如果需要，需要循环，直到内核对该条目的写入可见为止。 读取的值几乎是项数组的索引。 它实际上是索引加一。 应该从条目中复制数据并将ready_events设置为零; 那么头索引应该增加。因此，经整理后的从环形缓冲区读取的代码将如下所示：
```
    while (header->tail == header->head)
        ;  /* Wait for an event to appear */
    while (index[header->head] == 0)
        ;  /* Wait for event to really appear */
    item = header->items + index[header->head] - 1;
    data = item->data;
    item->ready_events = 0;  /* Mark event consumed */
    header->head++;
```
在实践中，此代码可能使用C原子操作，而不是直接读写，并且必须以循环方式增加磁头。 但是希望这个想法很明确。在空的环形缓冲区上忙等待显然不是理想的。 如果应用程序发现自己无事可做，它仍然可以调用epoll_wait（）进行阻塞，直到发生某些事情为止。 但是，只有在将事件数组作为NULL传递且maxevents设置为零时，此调用才会成功; 换句话说，epoll_wait（）将阻止，但它本身不会将任何事件返回给调用者。 不过，它将有帮助地返回ESTALE以指示环形缓冲区中有可用事件。该补丁集是其第三版，目前似乎几乎没有反对。 这项工作尚未进入linux-next，但似乎可以为5.3合并窗口做好准备。
<br />
## 一些抱怨
&emsp;&emsp;弄清楚上面的接口需要进行大量的代码反向工程。这是一个相当复杂的新API，但是几乎没有文档记录。这将使它难以使用，但是缺乏文档也使得一开始很难审查API。令人怀疑的是，作者以外的任何人目前都没有编写任何代码来使用此API。开发社区在承诺使用此API之前是否会完全理解该API尚不清楚。
<br />
&emsp;&emsp;不过，也许最可悲的是，这将是内核中许多环形缓冲区接口中的另一个接口。其他包括perf事件，ftrace，io_uring，AF_XDP以及毫无疑问的其他立即出现的事件。这些接口中的每一个都是从头开始创建的，用户空间开发人员必须分别理解（并实现消费者）。如果内核为用户空间共享的环形缓冲区定义了一套标准，而不是每次都创建新的东西，这会很好吗？不能责怪当前补丁集失败。那艘船是前一段时间航行的。但这确实说明了如何设计Linux内核API的一个缺点。他们似乎注定永远无法融入一个连贯一致的整体。