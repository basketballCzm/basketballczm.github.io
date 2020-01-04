---
title: "Go Sheduler"
date: 2020-01-04T16:21:44+08:00
draft: false
# description: "go-sheduler简介"
tags: [golang]
---

## go 调度器

### 介绍

Go 1.1的最重要的一个特性是引入了新的调度器，由Dmitry Vyukov贡献， 新的调度器极大的提高了程序的并行性能。

### go runtime需要一个怎样的调度器？

但是在查看新的调度器之前，我们需要了解为什么需要它。当操作系统可以为您调度线程时为什么需要创建用户进空间的线程（goroutine）?

POSIX线程API是现有Unix进程模型的逻辑扩展，因此，线程可以获得与进程相同的许多控件。线程有自己的信号掩码，可以分配CPU亲缘关系，可以放在cgroups中，可以查询它们使用哪些资源。所有这些控件都为Go程序使用goroutines所不需要的功能增加了开销，当程序中有100,000个线程时，这些开销就会迅速增加。

另一个问题是，基于Go模型操作系统无法做出明智的调度决策。例如，Go垃圾收集器要求在运行收集动作时停止所有线程，并且内存必须处于一致的状态。这涉及到等待正在运行的线程到达一个内存是一致的点。

当您在一个随机时间点调度许多线程时，很可能需要等待许多线程达到一致的状态。Go调度器只能在内存是一致的地方进行调度。这意味着，当我们停止垃圾收集动作时，我们不得不取等待正在CPU核上积极运行的线程。

通常有3个线程模型。一个是N:1，其中几个用户空间线程在一个OS线程上运行。这样做的优点是可以非常快速地切换上下文，但是不能利用多核系统。另一个是1:1，其中一个执行线程与一个操作系统线程匹配，它利用了机器上的所有核心，但是上下文切换很慢，因为上下文的切换也就是物理线程的切换，需要操作系统的介入。

Go试图通过使用M:N调度器来同时利用这两个方面。它将任意数量的goroutine调度到任意数量的OS线程上。您可以快速切换上下文，并利用系统中的所有核心。这种方法的主要缺点是它增加了调度器的复杂性。

### go-scheduler 组成

**为了完成调度任务，Go调度器使用3个主要实体**:

G: goroutine包含堆栈，指令指针和其他协程调度的重要信息，比如任何可能被阻塞的channel。在runtime的代码中它被称为G。

P: 逻辑处理器Processor，其代表调度的上下文，您可以将它看作是在单个线程上运行Go代码的调度程序的本地化版本。P是重要的对于让我们从老版本的GM模式的N:1转换到GPM模式的M:N（协程：物理线程）。在runtime中他被称为P，p的值可以被 `GOMAXPROCS()`函数所设置。

M: M是直接被OS调度管理的物理线程，它非常像标准的POSIX线程。在runtime中，它被称为M for Machine。

下面的图可以表示其3者者之间的关系：

![img](https://morsmachine.dk/in-motion.jpg)

在上面我们看到2个线程(M)，每个M都有一个context(P)和一个正在运行的goroutine(G)。为了运行G，一个M必须拥有一个P。

P的数量可以用环境变量GOMAXPROCS和runtime中的`GOMAXPROCS()`函数来设置，正常来说该值不会被改变。并且在go语言后面的版本中该值固定为CPU的逻辑核数。

灰色的协程没有被运行，但是它准备被调度，他们被排列在runqueues队列中， 该队列采用FIFO的进出方式。当协程执行go语句时，它被放到runqueues的结尾，P会运行一个协程直到到达调度时间点，然后P会从runqueue队列中取出G，然后设置栈和指令指针开始运行协程。

为了降低由多个P的情况下对于runqueue的互斥争用情况，每一个P都有自己的local runqueue。在之前的版本中go的调度器仅仅只有一个全局的runqueue，并且用一个互斥锁来保护它。线程经常会为了等待runqueue的互斥锁而被阻塞。当你想充分利用一个32核心机器上的计算资源时，这会变得更加糟糕。

只要所有上下文都有goroutines要运行，调度器就会在这种稳定状态下继续调度。然而，有两个场景可以改变这种情况。

#### syscall

你可能会想，为什么要有P呢?我们不能把runqueue放到线程M上，然后去掉上下文P吗?我们有上下文P的原因是，如果正在运行的线程由于某种原因需要阻塞，我们可以将它们传递给其他线程。

一个线程可能阻塞的例子是当我们调用一个系统调用时，线程可能会被阻塞在syscall上，因此在线程M阻塞时，应该传递上下文P，使得它能够继续调度其余G。

![img](https://morsmachine.dk/syscall.jpg)

在上面的图中，我们看到一个线程M0放弃了上下文P，以便另外一个线程M1能够运行该上下文。go调度器确保有足够的线程来运行上下文P。上面例子中的M1可能只是为了处理这个syscall而创建的，也可能来自线程的缓存(线程复用)。syscall线程M0将保持协程G0一直处理系统调用，G0尽管被阻塞了，但是从技术的角度来说它任然在执行。

当syscall返回时，为了运行返回的goroutine，线程必须尝试获取一个上下文P。正常的操作模式是从其他线程之一窃取上下文P。如果它不能窃取一个，它将把goroutine放到一个全局运行队列中，把自己放到线程缓存中，然后休眠。

上下文P的本地运行队列取出G取完后会到全局运行队列中去取。上下文P还会定期检查全局运行队列中的goroutines。否则，全局运行队列上的goroutines可能会因为饥饿而停止运行。

这种对系统调用的处理就是为什么Go程序能够运行多个线程，即使GOMAXPROCS为1。runtime来用goroutines通过线程进行系统调用，线程阻塞时，就将线程放在一边。

#### Stealing work

go调度系统稳定状态改变的另一种方式是当上下文P耗尽了要调度的goroutines时，如果上下文P的运行队列上的G数量不平衡，这将导致有的上下文P耗尽了它的运行队列，但是系统中仍然有工作要做。为了继续高效的运行代码，上下文P首先将从全局队列中获取G，如果当中没有G，它会从其余上下文P中取一半的G，这确保了每一个上下文P都是处于工作状态以及所有线程都在以最大容量G工作。

![img](https://morsmachine.dk/steal.jpg)

总结上面调度器为了达到平衡状态而做的两个动作。syscall可以总结当G导致M阻塞时，P向下抛弃M。steal work可以总结为P执行完所有G后，从全局运行队列和其余P中调度一半G。

### goroutine 调度

Go运行时会在下面的goroutine被阻塞的情况下运行另外一个goroutine。

1. blocking syscall(for example opening a file)
2. network input
3. channel operations
4. primitives in the sync package
5. The Go statement, although there is no guarantee that new goroutine will be scheduled immediately.
6. After being stopped for a garbage collection cycle.[link](https://codeburst.io/why-goroutines-are-not-lightweight-threads-7c460c1f155f)

### sysmon

sysmon是一个由runtime启动的M，也叫监控线程，它无需P也可以运行，它每20us~10ms唤醒一次，主要执行:

1. 释放闲置超过5分钟的span物理内存； scavenge heap
2. 如果超过2分钟没有垃圾回收，强制执行；forcegc
3. 将长时间未处理的netpoll结果添加到任务队列；netpool
4. 向长时间运行的G任务发出抢占调度；retake[golang-Cpu密集型任务调度抢占](http://xiaorui.cc/2018/06/04/golang%e5%af%86%e9%9b%86%e5%9c%ba%e6%99%af%e4%b8%8b%e5%8d%8f%e7%a8%8b%e8%b0%83%e5%ba%a6%e9%a5%a5%e9%a5%bf%e9%97%ae%e9%a2%98/)。这里发生抢占其实是一种伪抢占，本质上是sysmon中的retake触发morestack，然后调用newstack，然后gopreempt_m会重置g的状态，并且扔到本地runq中重新进行调度。
5. 收回因syscall长时间阻塞的P；

go调度器还有很多细节，比如cgo线程LockOSThread()函数以及与网络轮询器的集成。这些都能够在go的runtime库中能够找到。

#### 参考

[go-scheduler](https://morsmachine.dk/go-scheduler)

[go-scheduler1](https://wudaijun.com/2018/01/go-scheduler/)