---
layout: post
title:  "fork-exec 笔记"
date:   2021-09-25 00:00:00 +0800
categories: cs os
tag: fork exec
---

## fork

使用fork完，会创建一个子进程，内部处理逻辑，是让这俩进程共享代码段，同时，会新复制数据段和堆栈段（这里实际上没有做复制操作，只有在写新数据的时候，才会往不同地方写）
调用完fork，返回pid，对于父进程而言，pid>0，是子进程的pid。对于子进程而言，pid=0
调用完fork，对于父进程和子进程，会从下面一条语句开始执行。这里非常奇妙，相当于实时备份了一整个进程状态。

## exec

一个进程一旦调用exec系列函数，它本身就相当于死亡了，系统会把代码段替换成新的程序的代码，废弃原有的数据段和堆栈段，并为新程序分配新的 数据段和堆栈段。
但是，exec完，进程pid不会变，复用了父进程的pid，整个过程相当于把父进程覆盖掉了。一旦exec执行成功，原来父进程后面的代码，相当于没有任何用了，不会再执行了，强制切换到了新进程中去从头开始执行代码。如果exec执行失败，还是会返回错误信息的。

## 设计理念

接口设计的一个指导原则是“完整且最小”。“完整”的意思是，对外提供的接口必须能够满足任何使用要求。“最小”的意思是，接口功能无重复（无重叠），以能达到“正交化”水平为最佳。“完整且最小”的接口未必好用。有时候，为了使用方便或者性能或者其它种种原因，接口可以出现冗余；但冗余必然带来维护/学习等方面的代价。

linux的fork/exec就是一组典型的“完整且最小”的接口。“fork”用来产生一个新进程，这个进程默认会复制自身——于是类似apache这样用到“进程池”的场景得到支持。“fork”的“复制自身”操作又是“悬挂”的，如果紧接着调用exec，复制就会取消——于是启动另外一个进程的场景得到支持。“exec”则是“启动参数指定的程序，代替自身进程”。如果不配合fork使用，它是“当前进程结束，执行指定进程”；配合fork使用，就成了“当前进程启动另一个进程”。这样一来，各种使用场景就都得到了支持；再加上内部优化，写出“性能绝佳”的“多进程协作”程序就成了可能——于是linux甚至有相当一段时间都不支持线程，因为“fork的效率实在太好了，没必要支持线程”（另一个后遗症是，虽然现在linux内核有了线程支持，但线程和fork之间的关系极为复杂，以至于几乎只能在多进程/多线程两个方案中间选择其一）。
当然，为了便于使用，linux也提供了一个用起来简单一些的system调用。它和createProcess有点像，但内部仍然由fork+exec实现；此外，它执行时是阻塞的，同时还可能有很多信号之类的技术问题需要处理。现在，我们拿fork+exec和windows对比一下。
如果我们需要做一个“多进程协作”的网络服务框架，要求每个工作进程在处理了若干次服务请求后退出（这是个很讨巧的、保证系统7X24稳定性的经典设计）；但相应的，这就对进程创建效率提出了极高要求（哪怕按处理50个请求退出、且请求频率是相当初级的5000次/秒，也需要每秒创建100个进程；更不要说后来更为丧心病狂的10K、100K问题了）——这种设计在windows上也能保证性能吗？为什么？当然不行。因为windows相关API被封装的太“重”了。它用起来的确方便；但方便是有代价的。起码它不能“完整”的支持多进程协作的高性能服务框架……
linux的fork/exec方案也有缺陷。它的机制比较复杂，使得初学者难以理解；另外就是，当线程出现后，这个方案和多线程八字不合，混用的话有许多许多的坑等着你。反倒是windows，一旦有了线程支持，多进程互相监视保证稳定性、多线程同时执行提高效率，各种使用场景就全都被覆盖了——换句话说，现在它“完整”了，你改下方案就行。所以你看，过去毋庸置疑是linux的接口更灵活更强大（当然了，不是方便初学者的那种强大）；可一旦多了线程支持，windows方案反倒显得更“正交”了

> https://www.zhihu.com/question/66902460

## 参考

> https://keenjin.github.io/2019/05/linux_fork_and_exec/