---
layout: post
title: "Direct IO和Buffered IO简单分析"
description: "Direct IO和Buffered IO简单分析"
category: "操作系统"
tags: ["direct io", "aio", "NGINX"]
---
{% include JB/setup %}

今天在看[《Computer Systems - A Programmer's Perspective》] [4] 时，发现两个名词不是很理解，如下：
-   write through  
-   write  back

查了一下，发现其实就是Buffered IO和Direct IO。  这两者有何区别呢？

打个比方：正常情况下磁盘上有个文件，如何操作它呢？

- 读取：硬盘->内核缓冲区->用户缓冲区
- 写回：用户缓冲区->内核缓冲区->硬盘

这里的内核缓冲区指的是page cache，说白了，也就是是DRAM。主要作用是它用于缓存文件内容，从而加快对磁盘上映像和数据的访问。

正常的系统调用read/write的流程是怎样的呢？
- 读取：硬盘->内核缓冲区->用户缓冲区;
- 写回：数据会从用户地址空间拷贝到操作系统内核地址空间的页缓存中去，这是write就会直接返回，操作系统会在恰当的时机写入磁盘，这就是传说中的Buffered IO。

然而对于[自缓存应用程序] [2]来说，缓存 I/O 明显不是一个好的选择。因此出现了DIRECT IO。然而想象一下，不经内核缓冲区，直接写磁盘，必然会引起阻塞。所以通常DIRECT　IO与AIO会一起出现。
>Linux 异步 I/O 是 Linux 2.6 中的一个标准特性，其本质思想就是进程发出数据传输请求之后，进程不会被阻塞，也不用等待任何操作完成，进程可以在数据传输的时候继续执行其他的操作。相对于同步访问文件的方式来说，异步访问文件的方式可以提高应用程序的效率，并且提高系统资源利用率。直接 I/O 经常会和异步访问文件的方式结合在一起使用。

对于nginx来说，是否开启AIO要看具体使用场景：
-  正常写入文件往往是写入内存就直接返回，因此目前nginx仅支持在读取文件时使用异步IO。
-  由于linux AIO仅支持DIRECT IO,AIO一定回去从磁盘上读取文件。所以从阻塞worker进程的角度上来说有一定的好处，但是对于单个请求来说速度是降低了的。
>异步文件I/O是把“双刃剑”，关键要看使用场景，如果大部分用户的请求落到文件缓冲中，那么不要使用异步 I/O，反之则可以试着使用异步I/O,看一下是否会为服务带来并发能力上的提升。 -- [<<深入理解NGINX>> 陶辉著] [1]
    
mmap  
mmap系统调用是将硬盘文件映射到用内存中，其实就是将page cache中的页直接映射到用户进程地址空间中，从而进程可以直接访问自身地址空间的虚拟地址来访问page cache中的页，从而省去了内核空间到用户空间的copy。

参考资料：  
[Linux 中直接 I/O 机制的介绍] [3]  
[<<深入理解NGINX>> 陶辉著] [1]
    
[1]: http://book.douban.com/subject/22793675/
[2]: http://www.ibm.com/developerworks/cn/linux/l-cn-directio/#major1
[3]: http://www.ibm.com/developerworks/cn/linux/l-cn-directio/
[4]: http://book.douban.com/subject/1896753/

