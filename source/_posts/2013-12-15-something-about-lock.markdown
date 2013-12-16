---
layout: post
title: "something about lock"
date: 2013-12-15 21:27
comments: true
categories: 
---
老早就想梳理一下关于 Hardware && Lock 的知识，一直没有完成，比较惭愧。

这两方面的知识都比较多，现在本人还处于学习阶段，望指教。

最近我们组学习 SOSP13' Paper Everything You Always Wanted to Know about Synchronization but Were Afraid to Ask,
这篇 paper 主要讲 Hardware 对 Scalability 的影响 (文章涉及四个厂商的硬件，很好地总结了它们的特点，值得学习), 同时涉及到一些 Lock 的 Scalability 分析。

首先说一下硬件情况。
![enqueue](/images/hardware-perf.png)

本文主要讲一下几种锁 Lock 以及它们的优化原因。

基于 Test_And_Set 最简单的 Spin Lock ：
	type lock = (unlocked, locked)
  
	procedure acquire_lock (L : ^lock)
		delay : integer := 1
		while test_and_set (L) = locked     // returns old value
			pause (delay)                   // consume this many units of time, back-off
			delay := delay * 2

	procedure release_lock (L : ^lock)
		lock^ := unlocked
  
这种实现的 Spin Lock, 已经加入了 back-off.

Back-off  是为了干什么？
几个例子，如果今天早上外面公里上出现堵车情况，你最好不要马上出门。因为更多的人出门，只会造成公路更加拥挤。
既然说起来堵车这个比喻，我就在细说一下吧。
出现堵车的根本原因是，现在公路的运输能力不够。
在不改变公路的情况下，我们能做的就是充分把公路的运输能力挖掘出来。
最理想的情况就是，知道每个人要从哪里到哪里，然后安排好他们的出门时间，让整个公路通畅运行没有堵的情况出现，这样才能保证公路时刻都在以最大的效率运输车辆。
前面的 back-off 想做的就是这个。

具体到运行在某个硬件上的锁是这样的：在多核环境下，Lock 不管怎么实现，最终对归结为对某个内存的操作，由于多个核都可以看到这个内存，需要通过 Cache Coherence 协议保证对该变量访问和操作的一致性。
Cache Coherence 协议简单来说就是不同的核(可以这样理解，真正的硬件可能是 memory node controler)通过一系列的消息来保证不同核看到的数据 (cache line ) 是一致的。
有不同的 Cache Coherence 协议，一般我们都会在教科书上学习到基于四种状态 Modified Exclusive Shared Invalid (MESI) 的 Cache Coherence 协议。

在 Intel 机器上可能还会有一个 Forward 的状态,论文中说是为了
	This state is a special form of the shared state and indicates
	the only cache that will respond to a load request for that line (thus reducing bandwidth usage)。
AMD 多了个 Owner 的状态，论文中说是为了：
	The Opteron uses the MOESI protocol for cache coherence.
	The ‘O’ stands for the owned state, 
	which indicates that this cache line has been modified (by the owner) 
	but there might be more shared copies on other cores.
	This state allows a core to load a modified line of another core
	without the need to invalidate the modified line.
	The modified cache line simply changes to owned and the new core receives the line in shared state.

一个简单的场景。
许多核读了一个没有改变过的变量，这样每个核的 cache 中都会有一个处于 shared 状态的该变量。
而之后有个核 (例如 Core 1) 写了这个变量，它需要保证其他核看到正确的结果，由于之前其他核上的 cache 中已经有处于 share 状态的该变量，core 1 在完成写该变量的过程中需要 invalidate 其他核的该条 cache line。

如果明白上面场景，就可以明白为什么需要引入 back-off (即在拿不到锁后等一段时间)，这样做的目的是：如果你拿锁拿不到，证明有竞争，如果你这个时候再去试着拿锁，势必带来更多的 cache coherence message,
在 contention 十分重的情况下(极端情况就是每个核都在不断地操作这个变量)，这些 cache coherence message 会引起严重的性能问题。
(Non-scalable locks are dangerous 论文里面对这个问题做了很好的分析)。
在这种情况下，还不如你稍微等一下，在别人用完锁后你再去拿锁可以避免竞争，从而带来性能提升。

上面实现的 Spin Lock 最大的问题就是会有饿死的情况。(某个核不幸永远拿不到锁)

Ticket Spin Lock 可以保证公平:
	type lock = record
		next_ticket : unsigned integer := 0
		now_serving : unsigned integer := 0
  
	procedure acquire_lock (L : ^lock)
		my_ticket : unsigned integer := fetch_and_increment (&L->next_ticket)  
		// returns old value; arithmetic overflow is harmless
      
		loop
			pause (my_ticket - L->now_serving)
				// consume this many units of time
				// on most machines, subtraction works correctly despite overflow
			if L->now_serving = my_ticket
				return
  
	procedure release_lock (L : ^lock)
		L->now_serving := L->now_serving + 1

Ticket Spin Lock 很好地解决了饿死的问题。(每个人先去领个号，按号拿锁。一个人用完锁后通过共享变量通知下一个人)

此外，Back-off 在 Ticket Spin Lock 中可以更优雅的实现。

但是上面两篇文章都提到了 Spin Lock Scalability 不好。
原因是他们还是有共享变量 (那不到锁的人都会 spinning 在 now_serving，释放锁的开销比较大)。在核多的情况，会带来性能问题。

最好能够让每个核 spinning 在各自的 local 变量上， MCS Lock 做到了这一点。
	type qnode = record
		next : ^qnode
		locked : Boolean
	
	type lock = ^qnode      // initialized to nil
  
	// parameter I, below, points to a qnode record allocated
	// (in an enclosing scope) in shared memory locally-accessible
	// to the invoking processor
  
	procedure acquire_lock (L : ^lock, I : ^qnode)
		I->next := nil
		predecessor : ^qnode := fetch_and_store (L, I)
		if predecessor != nil      // queue was non-empty
			I->locked := true
			predecessor->next := I
			repeat while I->locked              // spin
  
	procedure release_lock (L : ^lock, I: ^qnode)
		if I->next = nil        // no known successor
			if compare_and_store (L, I, nil)
				return
				// compare_and_store returns true iff it stored
			repeat while I->next = nil          // spin
		I->next->locked := false

在上面实现的 MCS Lock 中，各个线程只会 spin 在自己的 local 变量  I:qnode  上。释放锁的人理想情况下只会给一个人发 invalidate 消息。
		
如果考虑到 NUNA 的特性，还可以继续提高 Lock 的 throughput. (同一个 NUMA Node 访问会比跨 NUMA Node 的访问快， 于是我们可以让锁具有偏向性，让位于一个 NUMA Node 上的 Core 更容易拿到锁)

![enqueue](/images/atomic-throughput.png)

![enqueue](/images/lock-throughput.png)

其实我们可以发现单个 Core 的 throughput 远大于多个 Core 的 throughput。原因是什么？ Contention 太中， cache coherence message 太多。

所以如果碰到性能问题，首先想到的不是去优化锁的性能，而是减少自身程序的 contention。
如果真的不行，在去考虑各种锁的实现，再不行的话，可以考虑采取其他同步机制了 RCU, Transaction Memory, Message Passing.

参考文档：

*  Everything You Always Wanted to Know about Synchronization but Were Afraid to Ask 
*  Non-scalable locks are dangerous
*  [http://www.cs.rochester.edu/research/synchronization/pseudocode/ss.html](http://www.cs.rochester.edu/research/synchronization/pseudocode/ss.html) 
*  [Spinlock and Contention](http://www.cs.tau.ac.il/~afek/Spin Locks and Contention.ppt) [Local Copy](/material/Spin Locks and Contention.ppt)

其中测试结果全部来自第一篇文章。
伪代码全部来自第三个参考文献。