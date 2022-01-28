---
title: 宏内核与微内核讨论之一
date: 2021-06-10 23:50:55
tags:
---

前段时间和群里的某个人有对关于微内核的争吵。我那时始终坚持微内核是未来，因为系统组件可自带仲裁，让线程有序调用。但那时候我也只觉得微内核的唯一价值在这了。那时候我就想，难不成只有微内核能做到安全？可维护性？以及方便使用多核处理器？ 现在我的观点有了改变，我尝试着把上面平常能听到提及微内核的上面那些优点，在宏内核上实现的可能性和性价比进行比较。或许，是混合内核更吃香？我目前一直在研究SMP 方面的算法。

对于微内核，各大营销号里经常见到的优点：安全。因为地址隔离，某组件崩溃了可重启；可维护性。是把内核要做的事情，分割到用户层的一个个组件；代码分离。这一点很少有人提及，但我觉得有一些道理，但我研究了之后，发现有不对；方便SMP 优化。因为可把每个组件自带仲裁机制，从而实现无锁，锁太难用了，代价也高，随着CPU 核心数的提升，锁将越来越拉跨。

当然，我要再加一点：系统调用少。一般至少只需要send,和recv 就可以完成所有的交互功能。但我觉得，对于这些微内核的优点，宏内核也许可以达到一样的效果？

对于安全性来说：因为地址隔离，而有了可重启的可能。其实我觉得，组件一旦panic 了，该出错一样会出错的，就算重启也没啥用。就比如系统文件，这种关键过程，重启了，就啥都没有了，而所谓的安全，我还得打一个问号。

只要是出错了，都会让用户体验不好，不管微内核和宏内核。至于宏内核panic 会出现黑屏或蓝屏，其实只要内核不黑屏就不一定就会黑屏，只不过对于内核来讲，这样做是至少安全点。所以我想，可不可以通过内核提供一个类似minix 里的一个"再生"组件，而达到宏内核的部分出错就重启那部分。

第二个要讲的就是微内核的可维护性。平常看到网上的文章  都说是把内核的代码移到user space。但我觉得，只要系统代码结构写得不够清晰干净，宏微都一样，至少为了实现同样的功能，宏微都需要一定量的代码。而对于宏内核来讲，只要内核代码结构写得清晰干净，是可以可维护性高的。

第三个就是对于SMP 的优化有优势，对于微内核而言。对于这点，我也打一个问号。是的，dragonfly bsd 实现了混合内核，它的目标是实现BSD 的SMP 可用性。让我有种感觉，对于多核处理器，是不是必须用至少部分要以类似微内核的形式呢？比如微内核组件自带仲裁功能，可以让用户请求有序进行，从而实现无锁，避免了锁竞争带来的CPU空转或是切换进程带来的额外switch context 开销。Dragonfly bsd 在介绍里就讲了它的memory allocator 是无锁的。所以我想，对于SMP，要尽可能避免锁，除非某些地方必须要用才行。但我觉得，宏内核就不能实现类似微的server 带仲裁功能？其实我觉得不一定，可以把内核的调度程序分层，类似dragonfly bsd 那样，具体情况，我目前也是在摸索，以后再给结论吧。

但我之前一直倾向于微内核是因为某些文章里给出的微内核的SMP 优化方便，这一点就足以吸引我。因为目前，摩尔定律已经不起作用了，只能靠堆核心来提高性能。游戏主机核心很多，至少可以印证对于需要高性能的一般应用而言，核心数多至少是性能的基准。

接着说微内核的SMP 好优化。其好在于对组件自带的仲裁、调度程序，可对user请求有序处理。但问题在于，微内核是需要IPC 通信的：之前看过mach 对比过宏内核的性能测试，微内核性能差在于因为要多次在user和kernel space 之间copy 消息，导致MMU 多次来回映射地址，这是一个重点性能点。

还有就是因为cache 不好命中，相比于宏内核，微内核会更频繁的在user 和kernel space 进程切换，导致cache miss 率高。所以我在思考 能否不像混合内核，纯宏内核也可以做高性能的SMP 操作系统。微内核就是因为页表切换和进程切换过于频繁， 之前的那些SMP 好处似乎也变得不那么好了。宏内核目前的一个弱点是SMP 不可扩展性。不知道目前linux 这种宏内核  有没有一个好的解决。至少在一年半前，是没有，现在不怎么关注了，也就不清楚linux 如何做了。我看了看dragonfly bsd 的介绍，dragonfly bsd 的内核是几乎没有瓶颈和锁竞争。对于部分需要锁的地方，也是尽量地简化，使其不容易发生死锁，当然，我没有安装使用dragonfly bsd，至于实际如何，我不清楚，但它给了我一些SMP 的思路。

###  很多人说微内核很多服务变成了用户态进程，可能重启，有容错性，我就想说一句你mm 服务进程和fs 服务进程出了bug 挂掉了你如何重启？其实只要关键性服务进程，没有一个能重启，服务进程之间都隐形的依赖性，fs 进程依赖于mm 进程,驱动程序可能依赖于mm 进程和fs 进程，有些是互相依赖，出了问题没有重启的机会。

###   现代芯片不适应微内核。看那个储存系统金字塔就知道，cpu 和内存之间cache 的miss 完全是因为程序的局部性，如果是smp 架构没有cache 的话对内存总线的争用更加严重，微内核靠消息通信运行导致大量的进程。上下文切换,势必导致性能低下，看图片上（鸿蒙架构图）内存管理都放在内核之外，这是反科学的，二愣子做法，微内核只是理论上的优美。



