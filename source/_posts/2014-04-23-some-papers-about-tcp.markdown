---
layout: post
title: "some papers about TCP"
date: 2014-04-23 20:06
comments: true
categories: 
---
一直对网络的知识比较感兴趣，最近组会轮到我读 Paper，刚好 NSDI'14 有篇关于 TCP Stack 的 Paper，就果断选择读它了。论文的 title 是： mTCP: A Highly Scalable User-level TCP Stack for Multicore Systems。其实 OSDI'12 也有一篇出发点差不多的文章，MegaPipe: A New Programming Interface for Scalable Network I/O。
两篇文章结合着看，会对整个 TCP Stack 的情况有更多的了解。

MegaPipe 比较详细地分析了 Linux TCP Stack 的性能情况，并通过修改 Kernel 解决一系列的问题。
而这篇 Paper 希望通过在 User-Level 实现完整的一套 TCP Stack来解决现有 Linux TCP Stack 的问题。

这两篇论文都是关注 short connection with small transaction, 虽然论文的有些测试可能在现实世界中很难找到，但是它们还是暴露了 Linux TCP Stack 的一些问题，这篇博文就简单总结一下文中提到的 Linux 的一些问题。

图一是来自 MegaPipe 的测试数据。
![enqueue](/images/net-eval.png)

图一说明了如果你的连接很短(每个连接只有请求)，你是很难把 10 Gpbs 带宽用掉的；此外，最好你的网络包大于 1KB，否则你也是很难把 10 Gpbs 的带宽用掉的。

### Contention on Accept Queue
这个是指：由于以前实现的 Socket 接口，只能允许一个端口被绑定一次，并且只能由绑定端口的进程去 Accept 新过来的请求连接。

对于拥有多核的 server，1）如果由一个 Accept 线程去接受请求，然后再分发请求给几个的工作线程，那么接受连接的线程可能成为最终的瓶颈（Google 已经出现了这样的极端情况）；
2）而如果由多个线程去接受请求，可能会造成 Work Load 不均衡的现象。
Google 员工 Tom Herbert 在发现这些现象后给 Kernel 提供了一个 Patch，他实现了一个新功能 SO_REUSEPORT （注意不是 SO_REUSEADDR，[两者差别](http://stackoverflow.com/questions/14388706/socket-options-so-reuseaddr-and-so-reuseport-how-do-they-differ-do-they-mean-t)）:

		int sfd = socket(domain, socktype, 0);

    	int optval = 1;
    	setsockopt(sfd, SOL_SOCKET, SO_REUSEPORT, &optval, sizeof(optval));

    	bind(sfd, (struct sockaddr *) &addr, addrlen);

通过 SO_REUSEPORT，多个线程／进程可以同时绑定一个端口多次，而 Kernel 会把这个端口的请求根据连接的情况（a hash based on the 4-tuple of the connection—that is, the peer IP address and port plus the local IP address and port）分发请求到不同的线程／进程。
现在的 SO_REUSEPORT 实现，在绑端口的线程／进程数目发生变化的时候，会有点[小问题](https://lwn.net/Articles/542629/)。

### Lake of Connect Affinity
主要原因是：执行内核中处理网络中断逻辑的核可能与执行应用程序的核不是同一个核，这样会造成很大的开销（Cache line 移动的开销）。
其实对于 Linux，我们可以通过绑核（把应用程序线程和 Kernel 相应中断处理逻辑绑到同一个的核上来解决整个问题），详见 Tuning 10Gb network cards on Linux。
EuroSys'12 文章 Affinity-Accept 有针对上面现象的很好的设计，有兴趣可以阅读一下。

### File Descriptor
这个是 POSIX 标准问题。POSIX 要求 Kernel 在分配 FD 的时候一定要是可用 FD 号中最小的。（已经有好几篇 OSDI 级别的文章用这个作为论文的 Motivation 了）
如果应用程序需要同时维护许多 Socket，这个要求会成为一个问题。

### VFS 抽象带来的开销
Virtual File System 给我们提供了一个统一的抽象。但是 Kernel 维护这个抽象不是没有代价的。MegaPipe 中说，当有许多短连接的时候，Kernel 维护全局可见的 VFS 对象会带来很大的同步开销。在多核环境下，开销比较大。

### System Call 的开销
我们都知道，处理网络请求需要通过 System Call 来完成，而 System Call/Context Switch 的开销是很大的。
关于 System Call／Contex Switch 的开销，可以看一下这个 [blog](http://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html)。

之前听实验室的大神说，如果是简单的 SysCall 可能就几百个 circle 就搞定了，而是复杂的就不好说了。
SysCall 会引起了 Contex Switch （可能更换 CR3，随之清 TLB，这开销就大了），如果不巧还有 Cache Pollution，那开销只会更大。
MigaPipe 和 mTCP 都是通过 Batch Processing 来分摊相应开销。


发现了问题所在，就能够提出解决方案了。
MigaPipe 通过修改 Kernel 解决问题，而 mTCP 则通过在 User 态实现一套完整的 TCP Stack 来实现。

为什么能够在 User 态操作硬件？这主要归功于 DPDK 这一类型的包处理库。对于这个完全不懂，需要进一步学习。DPDK 的网站是：www.dpdk.org。

对于 MigaPipe，mTCP，Affinity-Accept 的具体实现，简单一个博文是没有办法讲清楚的，有兴趣可以直接看 Paper 了。





 
 参考文档：

*  mTCP: A Highly Scalable User-level TCP Stack for Multicore Systems, NSDI'14 
*  MegaPipe: A New Programming Interface for Scalable Network I/O, OSDI'12
*  Improving network connection locality on multicore systems, EuroSys'12
*  Tuning 10Gb network cards on Linux
*  [Context Switch Overhead](http://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html)


